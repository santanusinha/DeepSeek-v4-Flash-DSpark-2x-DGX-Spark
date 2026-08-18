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
