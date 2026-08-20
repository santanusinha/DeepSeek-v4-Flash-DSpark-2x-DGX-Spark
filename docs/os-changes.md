# OS-Level Changes for DeepSeek V4 Flash DSpark Cluster

**Date**: 2026-08-18
**Nodes**: `ai1` (head, 192.168.3.120 / RoCE 192.168.100.10), `ai2` (worker, 192.168.3.121 / RoCE 192.168.100.11)
**Hardware**: 2x DGX Spark (NVIDIA GB10, SM121, ARM64)

---

## 1. CPU Topology

Each DGX Spark has 20 logical CPUs (10 physical cores, SMT):

| CPU IDs | Max MHz | Role |
|---------|---------|------|
| 0–4     | 2808    | Slow cores (Cortex-A78C) |
| 5–9     | 3900    | Fast cores (Cortex-X925) |
| 10–14   | 2808    | Slow cores (SMT siblings of 0–4) |
| 15–19   | 3900    | Fast cores (SMT siblings of 5–9) |

**Strategy**: Pin system services and IRQ handling to slow cores (0–4, 10–14). Leave fast cores (5–9, 15–19) free for vLLM GPU orchestration and NCCL communication.

---

## 2. Changes Applied

### 2.1 RoCE Host Entries in /etc/hosts

**Purpose**: Enable hostname resolution for RoCE network between nodes.

**Head node** (`/etc/hosts`):
```
192.168.100.10 ai1
192.168.100.11 ai2
```

**Worker node** (`/etc/hosts`):
```
192.168.100.10 ai1
192.168.100.11 ai2
```

**Status**: Applied on both nodes. Worker was missing these entries initially; fixed on 2026-08-18.

**Reproduce**:
```bash
echo "192.168.100.10 ai1" | sudo tee -a /etc/hosts
echo "192.168.100.11 ai2" | sudo tee -a /etc/hosts
```

---

### 2.2 Systemd CPU Affinity Pinning

**Purpose**: Pin all systemd-managed services to slow cores so fast cores stay free for vLLM.

**File**: `/etc/systemd/system.conf.d/cpu-affinity.conf` (on both nodes)

```ini
[Manager]
CPUAffinity=0-4 10-14
```

**Effect**: All services started by systemd run on slow cores (0–4, 10–14). Fast cores (5–9, 15–19) remain available for vLLM worker processes and NCCL.

**Reproduce**:
```bash
sudo mkdir -p /etc/systemd/system.conf.d
echo -e '[Manager]\nCPUAffinity=0-4 10-14' | sudo tee /etc/systemd/system.conf.d/cpu-affinity.conf
sudo systemctl daemon-reexec
```

**Note**: `daemon-reexec` is required (not just `daemon-reload`) because `system.conf.d` is read at systemd init time.

---

### 2.3 IRQ Affinity Service

**Purpose**: Pin NVIDIA GPU and RoCE (mlx5/ConnectX) hardware interrupts to slow cores. Prevents IRQ handling from stealing fast-core cycles during inference.

**File**: `/etc/systemd/system/irq-affinity.service` (on both nodes)

```ini
[Unit]
Description=Set NVIDIA and RoCE IRQ affinity to slow cores
After=multi-user.target
Wants=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'for irq in $(grep -iE "nvidia|mlx5|connectx|roce" /proc/interrupts 2>/dev/null | awk -F: "{print \$1}" | tr -d " "); do echo "0-4,10-14" > /proc/irq/$irq/smp_affinity_list 2>/dev/null; done'

[Install]
WantedBy=multi-user.target
```

**Status**: Enabled and active on both nodes.

**Reproduce**:
```bash
sudo tee /etc/systemd/system/irq-affinity.service << 'EOF'
[Unit]
Description=Set NVIDIA and RoCE IRQ affinity to slow cores
After=multi-user.target
Wants=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'for irq in $(grep -iE "nvidia|mlx5|connectx|roce" /proc/interrupts 2>/dev/null | awk -F: "{print \$1}" | tr -d " "); do echo "0-4,10-14" > /proc/irq/$irq/smp_affinity_list 2>/dev/null; done'

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now irq-affinity.service
```

**Verify**:
```bash
systemctl is-enabled irq-affinity.service  # should print: enabled
systemctl is-active irq-affinity.service   # should print: active
```

---

### 2.4 Headless Boot (default.target)

**Purpose**: Disable graphical desktop to free GPU memory and CPU cycles for inference.

**Change**: `default.target` symlinked from `graphical.target` to `multi-user.target`.

**Status**: Applied on both nodes.

**Reproduce**:
```bash
sudo systemctl set-default multi-user.target
```

**Revert** (if desktop needed):
```bash
sudo systemctl set-default graphical.target
```

---

## 3. Pre-existing NVIDIA Defaults (NOT our changes)

These ship with the DGX Spark image. Do not modify.

| Item | Location |
|------|----------|
| Docker systemd override | `/etc/systemd/system/docker.service.d/*.conf` — `After=nvidia-gpu-reset.target` |
| nvidia-persistenced | `systemctl is-enabled nvidia-persistenced` → enabled |
| NUMA balancing disable | `nvidia-disable-numa-balancing.service` — sets `kernel.numa_balancing=0` |
| Init-on-alloc disable | `nvidia-disable-init-on-alloc.service` |
| ARP sysctl defaults | `/etc/sysctl.d/20-nvidia-defaults.conf` — `arp_announce=2`, `arp_ignore=1` |
| modprobe: r8169 alias | `/etc/modprobe.d/nvidia-spark-r8169.conf` |
| modprobe: cppc_cpufreq | `/etc/modprobe.d/cppc_cpufreq.conf` — `auto_sel_mode=1` |
| modprobe: algif_aead disable | `/etc/modprobe.d/disable-algif_aead.conf` — CVE-2026-31431 |
| modprobe: nvidia-drm modeset | `/etc/modprobe.d/zz-nvidia-drm-override.conf` — `modeset=0` |

---

## 4. Rollback Procedure

To undo all custom OS changes on a node:

```bash
# 1. Remove CPU affinity pinning
sudo rm /etc/systemd/system.conf.d/cpu-affinity.conf
sudo systemctl daemon-reexec

# 2. Remove IRQ affinity service
sudo systemctl disable --now irq-affinity.service
sudo rm /etc/systemd/system/irq-affinity.service
sudo systemctl daemon-reload

# 3. Restore graphical boot (optional)
sudo systemctl set-default graphical.target

# 4. Remove RoCE host entries (optional)
sudo sed -i '/^192.168.100.10 ai1$/d; /^192.168.100.11 ai2$/d' /etc/hosts

# 5. Reboot to apply all changes
sudo reboot
```

---

## 5. Verification Commands

```bash
# Check CPU affinity
cat /etc/systemd/system.conf.d/cpu-affinity.conf

# Check IRQ affinity service
systemctl is-enabled irq-affinity.service
systemctl is-active irq-affinity.service

# Check IRQ pinning at runtime
grep -iE "nvidia|mlx5|roce" /proc/interrupts | while read line; do
  irq=$(echo "$line" | awk -F: '{print $1}' | tr -d ' ')
  echo "IRQ $irq: $(cat /proc/irq/$irq/smp_affinity_list 2>/dev/null)"
done

# Check default target
systemctl get-default

# Check RoCE host entries
grep -E "192.168.100" /etc/hosts
```
---

## 6. Swap and Memory Pressure Reduction

**Date**: 2026-08-20
**Nodes**: Both `ai1` (head) and `ai2` (worker)

### Problem

The DGX Spark GB10 uses **unified memory** — GPU and CPU share the same 120 GiB physical RAM. Under load, the kernel's page reclaimer was aggressively swapping VLLM process pages to disk, causing latency spikes when tool calls or idle periods triggered page-faults back from swap.

**Evidence** (before fix):
- System memory: 113 GiB used out of 121 GiB (93%)
- VLLM Worker_TP0 process: **~870 MiB swapped**
- VLLM EngineCore process: **~250 MiB swapped**
- Total swap usage: **~2.7 GiB** out of 16 GiB
- Default `vm.swappiness=60` was too aggressive for a dedicated inference node

### Fix: Reduce Kernel Swap Aggressiveness

**File**: `/etc/sysctl.d/99-vm-tune.conf` (on both nodes)

```ini
# VM tuning for VLLM inference workloads
# DGX Spark has unified memory (GPU/CPU share 120 GiB RAM).
# Default swappiness=60 causes the kernel to swap VLLM pages
# under memory pressure, causing latency spikes on tool calls.

# Near-zero swap (only under extreme OOM pressure)
vm.swappiness=1

# Aggressively reclaim page cache before even considering swapping
# process pages. This keeps VLLM resident by sacrificing cached files.
vm.vfs_cache_pressure=200
```

**Effect**:
- `vm.swappiness=1` — kernel only swaps under near-OOM conditions (was 60)
- `vm.vfs_cache_pressure=200` — kernel reclaims page cache at double the normal rate, **preferring to evict cached files rather than swap VLLM process pages**

**Why these values**:
- `swappiness=1` instead of `0` because `0` means "no swap until OOM" which can trigger the OOM killer on a transient spike. `1` allows a tiny amount of swap as a safety valve.
- `vfs_cache_pressure=200` instead of the earlier `50` because we want the kernel to **prefer evicting page cache over swapping VLLM**. Page cache just gets re-read from disk; swapped VLLM pages cause multi-second latency spikes.

**Reproduce**:
```bash
sudo tee /etc/sysctl.d/99-vm-tune.conf << 'EOF'
# VM tuning for VLLM inference workloads
vm.swappiness=1
vm.vfs_cache_pressure=200
EOF
sudo sysctl -w vm.swappiness=1
sudo sysctl -w vm.vfs_cache_pressure=200
```

**Verify**:
```bash
cat /proc/sys/vm/swappiness           # should print: 1
cat /proc/sys/vm/vfs_cache_pressure   # should print: 200
```

**Monitor swap drain**:
```bash
# Check VLLM process swap (should trend toward 0)
for pid in $(pgrep -f VLLM); do
  name=$(cat /proc/$pid/comm 2>/dev/null)
  swap=$(cat /proc/$pid/status 2>/dev/null | grep VmSwap | awk '{print $2}')
  echo "PID $pid ($name): Swap=${swap}kB"
done

# Overall swap usage
swapon --show
```

**Revert**:
```bash
sudo rm /etc/sysctl.d/99-vm-tune.conf
sudo sysctl -w vm.swappiness=60
sudo sysctl -w vm.vfs_cache_pressure=100
```

**Reproduce**:
```bash
sudo tee /etc/sysctl.d/99-swap-tune.conf << 'EOF'
# Reduce swap aggressiveness for VLLM inference workloads
vm.swappiness=10
vm.vfs_cache_pressure=50
EOF
sudo sysctl -w vm.swappiness=10
sudo sysctl -w vm.vfs_cache_pressure=50
```

**Verify**:
```bash
cat /proc/sys/vm/swappiness       # should print: 10
cat /proc/sys/vm/vfs_cache_pressure  # should print: 50
```

**Monitor swap usage**:
```bash
# Check VLLM process swap
for pid in $(pgrep -f VLLM); do
  name=$(cat /proc/$pid/comm 2>/dev/null)
  swap=$(cat /proc/$pid/status 2>/dev/null | grep VmSwap | awk '{print $2}')
  echo "PID $pid ($name): Swap=${swap}kB"
done

# Overall swap usage
swapon --show
```

**Revert**:
```bash
sudo rm /etc/sysctl.d/99-swap-tune.conf
sudo sysctl -w vm.swappiness=60
sudo sysctl -w vm.vfs_cache_pressure=100
```

---

## 7. Verification Commands (Updated)

```bash
# Check CPU affinity
cat /etc/systemd/system.conf.d/cpu-affinity.conf

# Check IRQ affinity service
systemctl is-enabled irq-affinity.service
systemctl is-active irq-affinity.service

# Check IRQ pinning at runtime
grep -iE "nvidia|mlx5|roce" /proc/interrupts | while read line; do
  irq=$(echo "$line" | awk -F: '{print $1}' | tr -d ' ')
  echo "IRQ $irq: $(cat /proc/irq/$irq/smp_affinity_list 2>/dev/null)"
done

# Check default target
systemctl get-default

# Check RoCE host entries
grep -E "192.168.100" /etc/hosts

# Check swap tuning
cat /proc/sys/vm/swappiness
cat /proc/sys/vm/vfs_cache_pressure

# Check VLLM process swap
for pid in $(pgrep -f VLLM); do
  name=$(cat /proc/$pid/comm 2>/dev/null)
  swap=$(cat /proc/$pid/status 2>/dev/null | grep VmSwap | awk '{print $2}')
  echo "PID $pid ($name): Swap=${swap}kB"
done
```