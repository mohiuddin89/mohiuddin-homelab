# Phase 1 — kubeadm Cluster Bootstrap + etcd Backup

**Status:** ✅ Complete  
**Date:** 2026-05-04  
**Author:** Mohiuddin  
**Applies to:** md-cp-1 (control plane), md-w-1, md-w-2 (workers)

---

## Overview

This runbook documents bootstrapping a 3-node Kubernetes cluster using kubeadm v1.31.14, configuring Flannel CNI for pod networking, setting up automated etcd backups, and validating cluster resilience via a break-it drill.

**Why kubeadm:** kubeadm bootstraps a production-grade cluster manually — exposing every component (API server, etcd, scheduler, controller-manager) so you understand what Kubernetes actually does under the hood. Managed services like EKS hide all of this.

---

## Infrastructure

| VM | IP | Role |
|---|---|---|
| md-cp-1 | 10.20.0.10 | Kubernetes Control Plane |
| md-w-1 | 10.20.0.21 | Kubernetes Worker 1 |
| md-w-2 | 10.20.0.22 | Kubernetes Worker 2 |

**Kubernetes version:** v1.31.14  
**CNI:** Flannel  
**Pod CIDR:** 10.244.0.0/16  
**Service CIDR:** 10.96.0.0/12 (default)

---

## Phase 1 — Cluster Bootstrap

### Step 1 — Install Dependencies

**Why:** kubeadm requires `conntrack` for network connection tracking. Without it, preflight checks fail and installation is blocked.

Run on **all 3 nodes:**

```bash
sudo apt-get install -y conntrack
```

> **Gotcha:** This package is not automatically pulled in with kubeadm. Missing it causes `[ERROR FileExisting-conntrack]` at both `kubeadm init` and `kubeadm join`. Install it on all nodes before proceeding.

### Step 2 — Add Kubernetes APT Repository

Run on **all 3 nodes:**

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Step 3 — Install kubeadm, kubelet, kubectl

**Why version pin:** Kubernetes components must be the same version across control plane and workers. `apt-mark hold` prevents accidental upgrades that could break the cluster.

Run on **all 3 nodes:**

```bash
sudo apt-get update && sudo apt-get install -y \
  kubelet=1.31.* \
  kubeadm=1.31.* \
  kubectl=1.31.*

sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

**Verify:**

```bash
kubeadm version && kubectl version --client && kubelet --version
```

**Expected:** All show `v1.31.14`

### Step 4 — Bootstrap Control Plane

**Why these flags:**
- `--pod-network-cidr=10.244.0.0/16` — required by Flannel CNI
- `--apiserver-advertise-address=10.20.0.10` — tells API server which IP to listen on
- `--node-name=md-cp-1` — sets explicit node name

Run on **md-cp-1 only:**

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=10.20.0.10 \
  --node-name=md-cp-1
```

This takes 2-3 minutes. At the end you will see a `kubeadm join` command — **save it.**

### Step 5 — Configure kubectl Access

**Why:** kubectl reads cluster credentials from `~/.kube/config`. Without this, every kubectl command fails with "no config file found."

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Verify:**

```bash
kubectl get nodes
```

**Expected:** `md-cp-1 NotReady control-plane` (NotReady is normal — CNI not installed yet)

### Step 6 — Install Flannel CNI

**Why:** Without a CNI plugin, pods cannot communicate with each other. Flannel creates a flat network across all nodes using VXLAN overlay. Node status stays `NotReady` until CNI is installed.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Wait 30 seconds, then verify:

```bash
kubectl get nodes
```

**Expected:** `md-cp-1 Ready control-plane`

### Step 7 — Join Worker Nodes

Run the saved `kubeadm join` command on **md-w-1 and md-w-2:**

```bash
sudo kubeadm join 10.20.0.10:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**Verify from md-cp-1:**

```bash
kubectl get nodes
```

**Expected:**
```
NAME      STATUS   ROLES           AGE   VERSION
md-cp-1   Ready    control-plane   Xm    v1.31.14
md-w-1    Ready    <none>          Xm    v1.31.14
md-w-2    Ready    <none>          Xm    v1.31.14
```

### Step 8 — Label Worker Nodes

**Why:** Node labels allow workloads to be scheduled on specific nodes using `nodeSelector`. This is used later to pin Ingress to md-w-1 and Observability to md-w-2.

```bash
kubectl label node md-w-1 node-role.kubernetes.io/worker=worker
kubectl label node md-w-2 node-role.kubernetes.io/worker=worker
```

### Step 9 — Verify All System Pods

```bash
kubectl get pods -A
```

**Expected:** All pods in `Running` state:
- kube-flannel (3 pods — one per node)
- coredns (2 pods)
- etcd, kube-apiserver, kube-controller-manager, kube-scheduler (1 each on md-cp-1)
- kube-proxy (3 pods — one per node)

---

## Phase 1.5 — etcd Backup + Restore Drill

### Why etcd Backup is Critical

This cluster has a single control plane node. etcd is the only source of truth for the entire cluster state — all deployments, secrets, configmaps, RBAC rules, and node registrations live here. If etcd is lost without a backup, the cluster cannot be recovered. The only option would be to rebuild from scratch.

**Rule:** A backup that has never been restored is not a backup. Always run a restore drill.

### Step 1 — Install etcdctl

```bash
sudo apt-get install -y etcd-client
```

**Verify:**

```bash
etcdctl version
```

### Step 2 — Take Manual Snapshot

**Why cert flags:** etcd is TLS-protected. The snapshot command must authenticate using the etcd CA, server cert, and key stored in `/etc/kubernetes/pki/etcd/`.

```bash
sudo mkdir -p /var/lib/etcd-backups

sudo ETCDCTL_API=3 etcdctl snapshot save \
  /var/lib/etcd-backups/snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Step 3 — Verify Snapshot Integrity

**Why:** A corrupt snapshot is useless. Always verify immediately after taking a backup.

```bash
sudo ETCDCTL_API=3 etcdctl snapshot status \
  /var/lib/etcd-backups/snapshot-<timestamp>.db \
  --write-out=table
```

**Expected:**
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 68eeb0ca |     1975 |       1054 |     2.1 MB |
+----------+----------+------------+------------+
```

### Step 4 — Automated Backup Script

```bash
sudo tee /usr/local/bin/etcd-backup.sh <<'EOF'
#!/bin/bash
BACKUP_DIR=/var/lib/etcd-backups
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

ETCDCTL_API=3 etcdctl snapshot save $BACKUP_DIR/snapshot-$TIMESTAMP.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Retain last 7 days only
find $BACKUP_DIR -name "snapshot-*.db" -mtime +7 -delete
EOF

sudo chmod +x /usr/local/bin/etcd-backup.sh
```

### Step 5 — Schedule Cron Job

**Why every 6 hours:** This limits maximum data loss (RPO) to 6 hours. For production, you would tune this based on business requirements.

```bash
(sudo crontab -l 2>/dev/null; echo "0 */6 * * * /usr/local/bin/etcd-backup.sh") | sudo crontab -
```

**Verify:**

```bash
sudo crontab -l
# Expected: 0 */6 * * * /usr/local/bin/etcd-backup.sh
```

---

## Break-it Drill — kube-apiserver Failure

### Scenario

Simulate kube-apiserver crash and measure recovery time.

### Why kube-apiserver is a Static Pod

Unlike regular pods, control plane components (apiserver, etcd, scheduler, controller-manager) are **static pods** — kubelet reads their manifests directly from `/etc/kubernetes/manifests/` and keeps them running. There is no Deployment or ReplicaSet managing them. If the manifest file is removed, kubelet stops the pod. If restored, kubelet restarts it automatically.

### Drill Steps

**Step 1 — Kill kube-apiserver:**

```bash
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml
```

**Step 2 — Confirm cluster is unreachable:**

```bash
kubectl get nodes
# Expected: "The connection to the server 10.20.0.10:6443 was refused"
```

**Step 3 — Restore:**

```bash
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Step 4 — Verify recovery:**

```bash
kubectl get nodes
# Expected: All nodes Ready
```

### Results

| Metric | Value |
|---|---|
| Time to failure detection | Immediate — kubectl refused |
| Recovery action | Move manifest file back |
| Recovery time | ~30 seconds |
| Data loss | None — etcd was untouched |

---

## Troubleshooting

### kubeadm init/join fails with conntrack error

**Symptom:** `[ERROR FileExisting-conntrack]: conntrack not found in system path`  
**Fix:** `sudo apt-get install -y conntrack` on the affected node

### Node stuck in NotReady after join

**Symptom:** Worker node joined but stays `NotReady`  
**Cause:** Flannel pod not yet scheduled on that node  
**Fix:** Wait 60 seconds. Check: `kubectl get pods -n kube-flannel`

### kubectl returns "connection refused"

**Symptom:** `The connection to the server 10.20.0.10:6443 was refused`  
**Cause:** kube-apiserver not running  
**Check:** `sudo ls /etc/kubernetes/manifests/` — verify kube-apiserver.yaml exists  
**Fix:** If missing, restore from backup or re-run kubeadm init

---

## Validation Checklist

| Check | Command | Expected |
|---|---|---|
| All nodes Ready | `kubectl get nodes` | 3 nodes, all Ready |
| All system pods Running | `kubectl get pods -A` | All Running |
| etcd snapshot exists | `ls /var/lib/etcd-backups/` | snapshot file present |
| Snapshot integrity | `etcdctl snapshot status` | valid hash + key count |
| Cron job active | `sudo crontab -l` | 0 */6 entry present |
| Break-it recovery | kube-apiserver restore | cluster back in ~30s |

---

*Phase 1 + 1.5 complete. Ready for Phase 2 — MetalLB + ingress-nginx + storage.*
