# Phase 0 Runbook — Environment Baseline

**Date:** 2026-04-30
**Status:** Complete
**Applies to:** md-cp-1, md-w-1, md-w-2, md-svc-1, md-ci-1

## Prerequisites

- 5 VMs provisioned in Proxmox
- SSH access via MobaXterm
- Tailscale connected to Proxmox server

## Step 1 — Verify VM Baseline

Run on each VM:

```bash
hostname && ip a | grep 10.20 && free -h && df -h /
```

Expected: hostname matches VM name, IP matches plan, swap = 0.

## Step 2 — Disable Swap Permanently

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

Verify:
```bash
free -h | grep Swap
# Expected: Swap: 0B 0B 0B
```

## Step 3 — Update /etc/hosts

Run on all 5 VMs:

```bash
cat <<EOF | sudo tee -a /etc/hosts
10.20.0.10 md-cp-1
10.20.0.21 md-w-1
10.20.0.22 md-w-2
10.20.0.50 md-svc-1
10.20.0.30 md-ci-1

# Phase 0 Runbook — Environment Baseline

**Date:** 2026-04-30
**Status:** Complete
**Applies to:** md-cp-1, md-w-1, md-w-2, md-svc-1, md-ci-1

## Prerequisites
- 5 VMs provisioned in Proxmox
- SSH access via MobaXterm
- Tailscale connected to Proxmox server

## Step 1 — Verify VM Baseline
Run on each VM:
hostname && ip a | grep 10.20 && free -h && df -h /
Expected: hostname matches VM name, IP matches plan, swap = 0.

## Step 2 — Disable Swap Permanently
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
Verify: free -h | grep Swap
Expected: Swap: 0B 0B 0B

## Step 3 — Update /etc/hosts
Add on all 5 VMs:
10.20.0.10 md-cp-1
10.20.0.21 md-w-1
10.20.0.22 md-w-2
10.20.0.50 md-svc-1
10.20.0.30 md-ci-1
Verify: ping -c 2 md-w-1

## Step 4 — SSH Hardening
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo systemctl restart ssh
Verify: sudo sshd -T | grep -E "passwordauthentication|permitrootlogin"
Expected: both = no
Note: If 50-cloud-init.conf does not exist, check sshd -T directly.

## Step 5 — Verify NTP
timedatectl status | grep -E "System clock|NTP service"
Expected: synchronized = yes, NTP service = active

## Step 6 — Kernel Modules (K8s nodes only)
sudo modprobe br_netfilter && sudo modprobe overlay
Persist: /etc/modules-load.d/k8s.conf with overlay and br_netfilter
Verify: lsmod | grep -E "br_netfilter|overlay"

## Step 7 — sysctl Parameters (K8s nodes only)
/etc/sysctl.d/k8s.conf:
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
Apply: sudo sysctl --system
Verify: sysctl net.ipv4.ip_forward = 1

## Step 8 — containerd (K8s nodes only)
sudo apt-get install -y containerd
containerd config default | sudo tee /etc/containerd/config.toml
Set SystemdCgroup = true
sudo systemctl restart containerd && sudo systemctl enable containerd
Verify: grep SystemdCgroup /etc/containerd/config.toml = true

## Step 9 — Reboot Test
sudo reboot
After reboot verify all settings persist.

## Troubleshooting
SSH lock out: Use Proxmox console, re-enable password auth, inject key, disable again.
bridge-nf not applying: modprobe br_netfilter first, then sysctl --system.
50-cloud-init.conf override: Edit that file directly, not sshd_config.
