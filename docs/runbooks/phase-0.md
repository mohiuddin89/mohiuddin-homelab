# Phase 0 — Environment Baseline

**Status:** ✅ Complete  
**Date:** 2026-04-30  
**Author:** Mohiuddin  
**Applies to:** md-cp-1, md-w-1, md-w-2, md-svc-1, md-ci-1  

---

## Overview

This runbook documents the baseline configuration applied to all 5 VMs in the Proxmox homelab before Kubernetes installation. The goal is to ensure every node meets production-grade prerequisites — correct networking, security hardening, kernel configuration, and container runtime setup.

**Why this matters:** If any of these steps are skipped or misconfigured, `kubeadm init` will fail, pods will not communicate, or the cluster will break silently after a reboot.

---

## Infrastructure

| VM | IP | Role | RAM | CPU | Disk |
|---|---|---|---|---|---|
| md-svc-1 | 10.20.0.50 | DNS + CA | 2 GB | 2 | 30 GB |
| md-cp-1 | 10.20.0.10 | K8s Control Plane | 4 GB | 2 | 40 GB |
| md-w-1 | 10.20.0.21 | K8s Worker 1 | 6 GB | 2 | 40 GB |
| md-w-2 | 10.20.0.22 | K8s Worker 2 | 6 GB | 2 | 40 GB |
| md-ci-1 | 10.20.0.30 | GitLab CI | 2 GB | 2 | 40 GB |

**Network:** 10.20.0.0/24 (Proxmox internal)  
**Remote access:** Tailscale + MobaXterm SSH  

---

## Step 1 — Verify VM Baseline

**Why:** Confirm each VM has the correct hostname, IP, RAM, and disk as provisioned in Proxmox. Catch any mismatch before proceeding.

Run on each VM:

```bash
hostname && ip a | grep 10.20 && free -h && df -h /
```

**Expected output (example for md-cp-1):**
```
md-cp-1
inet 10.20.0.10/24
Mem: 3.8Gi
/dev/sda1  38G
```

---

## Step 2 — Disable Swap Permanently

**Why:** Kubernetes requires swap to be off. If swap is on, `kubeadm init` will fail with an error. The `/etc/fstab` edit ensures swap stays off after reboot.

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

**Verify:**
```bash
free -h | grep Swap
```

**Expected:**
```
Swap:  0B  0B  0B
```

> **Note:** md-svc-1 had swap enabled (2GB) — this was the only VM that required this fix. All K8s nodes already had swap off.

---

## Step 3 — Update /etc/hosts

**Why:** Kubernetes components and kubeadm join commands use hostnames internally. Without `/etc/hosts` entries, hostname resolution fails and cluster join breaks.

Run on all 5 VMs:

```bash
sudo tee -a /etc/hosts <<EOF
10.20.0.10 md-cp-1
10.20.0.21 md-w-1
10.20.0.22 md-w-2
10.20.0.50 md-svc-1
10.20.0.30 md-ci-1
EOF
```

**Verify:**
```bash
ping -c 2 md-w-1
```

**Expected:** 0% packet loss

---

## Step 4 — SSH Hardening

**Why:** Password-based SSH is a security risk. Key-based authentication is the production standard — only someone with the private key can login.

```bash
sudo sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo systemctl restart ssh
```

**Verify:**
```bash
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin"
```

**Expected:**
```
passwordauthentication no
permitrootlogin no
```

> **Important:** On Ubuntu cloud images, `/etc/ssh/sshd_config.d/50-cloud-init.conf` overrides the main `sshd_config`. Always edit this file — editing only `sshd_config` will have no effect.

---

## Step 5 — Verify NTP Sync

**Why:** Kubernetes TLS certificates and etcd depend on accurate time across all nodes. If clocks drift, certificate validation fails and etcd rejects writes.

```bash
timedatectl status | grep -E "System clock|NTP service|Time zone"
```

**Expected:**
```
Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: yes
NTP service: active
```

---

## Step 6 — Load Kernel Modules (K8s nodes only)

**Why:**  
- `overlay` — required for container filesystem layering (containerd uses this)  
- `br_netfilter` — required for iptables to see bridged traffic between pods  

Without these, pod-to-pod networking will silently fail.

```bash
sudo modprobe br_netfilter
sudo modprobe overlay
```

Persist across reboots:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

**Verify:**
```bash
lsmod | grep -E "br_netfilter|overlay"
```

**Expected:**
```
overlay
br_netfilter
bridge   (loaded as dependency of br_netfilter)
```

---

## Step 7 — sysctl Parameters (K8s nodes only)

**Why:**  
- `ip_forward` — allows packets to be forwarded between network interfaces (pod → pod across nodes)  
- `bridge-nf-call-iptables` — ensures iptables rules apply to bridged traffic (required for NetworkPolicy and kube-proxy)  

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**Verify:**
```bash
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

**Expected:** both = `1`

> **Gotcha:** If `br_netfilter` is not loaded when sysctl runs, the bridge-nf settings will silently not apply. Always load the module first.

---

## Step 8 — Install and Configure containerd (K8s nodes only)

**Why:** containerd is the container runtime used by Kubernetes. Docker is not required. `SystemdCgroup = true` is critical — without it, kubelet and containerd use different cgroup drivers, causing pod startup failures.

```bash
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Verify:**
```bash
grep "SystemdCgroup" /etc/containerd/config.toml
sudo systemctl status containerd | grep -E "Active|Loaded"
```

**Expected:**
```
SystemdCgroup = true
Loaded: enabled
Active: active (running)
```

---

## Step 9 — Reboot Test

**Why:** Settings like kernel modules, sysctl values, and swap-off must survive a reboot. If they don't persist, the cluster will break the next time a node restarts.

```bash
sudo reboot
```

After reboot, run full verification:

```bash
free -h | grep Swap && \
lsmod | grep br_netfilter && \
sysctl net.ipv4.ip_forward && \
sudo systemctl status containerd | grep Active
```

**Expected:** All values match. Swap = 0, br_netfilter loaded, ip_forward = 1, containerd active.

---

## Troubleshooting

### SSH lock out after hardening

**Symptom:** `Permission denied (publickey)` after disabling password auth  
**Cause:** SSH public key was not injected before disabling password authentication  
**Fix:**  
1. Access VM via Proxmox console (direct TTY login)  
2. Re-enable password auth: `sudo vi /etc/ssh/sshd_config.d/50-cloud-init.conf` → set `yes`  
3. Restart SSH: `sudo systemctl restart ssh`  
4. Inject public key from another VM: `ssh-copy-id -i ~/.ssh/id_ed25519.pub user@IP`  
5. Disable password auth again  

### bridge-nf sysctl not applying

**Symptom:** `sysctl --system` runs but `bridge-nf-call-iptables` stays 0  
**Cause:** `br_netfilter` module not loaded when sysctl runs  
**Fix:** `sudo modprobe br_netfilter && sudo sysctl --system`

### 50-cloud-init.conf overriding sshd_config

**Symptom:** `sshd_config` shows `PasswordAuthentication no` but `sshd -T` still shows `yes`  
**Cause:** `/etc/ssh/sshd_config.d/50-cloud-init.conf` takes precedence  
**Fix:** Edit `/etc/ssh/sshd_config.d/50-cloud-init.conf` directly

---

## Validation Checklist

| Check | Command | Expected |
|---|---|---|
| Swap off | `free -h \| grep Swap` | `0B 0B 0B` |
| Hosts file | `ping -c 2 md-w-1` | 0% packet loss |
| SSH hardened | `sshd -T \| grep passwordauth` | `no` |
| NTP synced | `timedatectl status` | `synchronized: yes` |
| Kernel modules | `lsmod \| grep br_netfilter` | loaded |
| ip_forward | `sysctl net.ipv4.ip_forward` | `1` |
| containerd | `systemctl status containerd` | `active (running)` |
| Reboot test | all above after reboot | all pass |

---

*Phase 0 complete. Ready for Phase 1 — kubeadm cluster bootstrap.*
