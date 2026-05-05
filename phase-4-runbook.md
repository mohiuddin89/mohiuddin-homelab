# Phase 4 — Security Baseline

**Status:** ✅ Complete  
**Date:** 2026-05-05  
**Author:** Mohiuddin  
**Applies to:** md-cp-1 (control plane), md-w-1, md-w-2 (workers)

---

## Overview

This runbook documents implementing a security baseline for the Kubernetes cluster — CNI migration attempt with NetworkPolicy validation, RBAC with least-privilege ServiceAccounts, and encrypted secret management with Sealed Secrets.

**Why security baseline before applications:** Deploying applications without security controls means any compromised pod can access any other pod, read any secret, and make any API call. Security controls must be in place before workloads arrive, not retrofitted after.

---

## Prerequisites

- Phase 3 complete (ArgoCD running)
- Fresh etcd backup before CNI migration

---

## Step 1 — Pre-Migration etcd Backup

**Why:** CNI migration is a high-risk operation. If something goes wrong, etcd backup is the recovery path.

```bash
sudo /usr/local/bin/etcd-backup.sh
ls -lh /var/lib/etcd-backups/
```

Expected: Fresh snapshot file with current timestamp.

---

## Step 2 — Calico CNI Migration Attempt

**Why Calico over Flannel:** Flannel provides basic pod networking but does not support Kubernetes NetworkPolicy. Calico provides both pod networking and NetworkPolicy enforcement — enabling fine-grained traffic control between pods.

### Remove Flannel

```bash
kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Install Calico

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml
```

**Verify:**

```bash
kubectl get pods -n kube-system | grep calico
# Expected: calico-node (one per node) + calico-kube-controllers = all Running
```

---

## NetworkPolicy Validation (Same-Node)

### Deploy Test Pods

```bash
kubectl create namespace netpol-test

kubectl run server --image=nginx --namespace=netpol-test --labels="app=server"
kubectl run client --image=busybox --namespace=netpol-test --labels="app=client" -- sleep 3600
```

### Apply Default-Deny Policy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netpol-test
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

**Test — traffic should be blocked:**

```bash
kubectl exec -n netpol-test client -- wget -qO- --timeout=5 http://<server-ip>
# Expected: timeout — NetworkPolicy blocking traffic
```

### Apply Explicit Allow Rules

```bash
# Allow client → server ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-server
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 80
EOF

# Allow client egress to server
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-egress
  namespace: netpol-test
spec:
  podSelector:
    matchLabels:
      app: client
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: server
    ports:
    - protocol: TCP
      port: 80
EOF
```

**Test — same-node traffic should be allowed:**

```bash
kubectl exec -n netpol-test client -- wget -qO- --timeout=5 http://<server-ip>
# Expected: nginx welcome page HTML
```

**Result:** Same-node NetworkPolicy enforcement validated ✅

---

## Calico Rollback Decision

**Issue encountered:** Cross-node pod communication failed with Calico in the Proxmox environment. Calico uses IP-in-IP (IPIP) tunneling by default for cross-node traffic. Proxmox's virtual network bridge blocked IPIP protocol packets between VMs. Switching to VXLAN mode also did not resolve the issue.

**Decision:** Roll back to Flannel for cluster stability. NetworkPolicy concepts were validated on same-node traffic.

**ADR note:** In a production bare-metal environment, Calico cross-node networking requires either:
- Enabling promiscuous mode on the hypervisor network bridge
- Using BGP mode (requires router configuration)
- Using Calico with VXLAN on a network that permits UDP 4789

### Rollback Steps

```bash
# Remove Calico
kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml

# Clean up Calico CNI config from worker nodes (CRITICAL)
# Run on md-w-1 AND md-w-2:
sudo rm -f /etc/cni/net.d/10-calico.conflist
sudo rm -f /etc/cni/net.d/calico-kubeconfig
sudo systemctl restart kubelet

# Reinstall Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

> **Critical gotcha:** Deleting Calico via kubectl removes Kubernetes resources but leaves the CNI binary and config on worker nodes. New pods get stuck in `ContainerCreating` with error `plugin type="calico" failed: Unauthorized` because the CNI plugin binary calls the deleted Calico controller. Always clean up `/etc/cni/net.d/` on worker nodes after CNI removal.

---

## Step 3 — RBAC — Least Privilege ServiceAccount

**Why least privilege:** Every pod that needs to query the Kubernetes API should use a ServiceAccount with only the permissions it needs. A monitoring tool needs to read pod metrics — it does not need to create or delete resources. If that pod is compromised, the blast radius is limited to read-only access.

### Create ServiceAccount

```bash
kubectl create serviceaccount monitoring-reader -n kube-system
```

### Create ClusterRole (read-only)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints", "nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
EOF
```

### Bind Role to ServiceAccount

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-reader-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: monitoring-reader
subjects:
- kind: ServiceAccount
  name: monitoring-reader
  namespace: kube-system
EOF
```

### Validate RBAC

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:kube-system:monitoring-reader -n default
# Expected: yes

kubectl auth can-i delete pods \
  --as=system:serviceaccount:kube-system:monitoring-reader -n default
# Expected: no
```

---

## Step 4 — Sealed Secrets

**Why Sealed Secrets:** Kubernetes Secrets are only base64 encoded — not encrypted. Anyone with Git access can decode them. Sealed Secrets uses asymmetric encryption — the cluster holds the private key, and anyone can encrypt using the public key. Encrypted secrets are safe to commit to Git.

**Why Sealed Secrets over External Secrets Operator:** External Secrets Operator requires an external secrets backend (AWS SSM, Vault, etc.). For a lab environment, Sealed Secrets is self-contained — no external dependencies.

### Install Sealed Secrets Controller

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller
```

**Verify:**

```bash
kubectl get pods -n kube-system | grep sealed
# Expected: sealed-secrets-controller = 1/1 Running
```

### Install kubeseal CLI

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | grep tag_name | cut -d'"' -f4 | cut -c2-)
curl -sSL -o /tmp/kubeseal "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xzf /tmp/kubeseal -C /tmp/
sudo install -m 755 /tmp/kubeseal /usr/local/bin/kubeseal
```

### Encrypt a Secret

```bash
kubectl create secret generic test-secret \
  --from-literal=password=mysecretpassword \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-test-secret.yaml

cat sealed-test-secret.yaml
# Output: SealedSecret with encrypted data — safe to commit to Git
```

### Apply and Verify Decryption

```bash
kubectl apply -f sealed-test-secret.yaml

kubectl get secret test-secret -n default \
  -o jsonpath='{.data.password}' | base64 -d && echo
# Expected: mysecretpassword
```

---

## Troubleshooting

### New pods stuck in ContainerCreating after CNI removal

**Symptom:** `plugin type="calico" failed: error getting ClusterInformation: Unauthorized`  
**Cause:** Calico CNI binary and config remain on worker nodes after `kubectl delete`  
**Fix:**
```bash
# On each worker node:
sudo rm -f /etc/cni/net.d/10-calico.conflist
sudo rm -f /etc/cni/net.d/calico-kubeconfig
sudo systemctl restart kubelet
```

### Calico cross-node communication failure in Proxmox

**Symptom:** Pods on different nodes cannot communicate  
**Cause:** Proxmox VM network bridge blocks IPIP/VXLAN encapsulated traffic  
**Fix options:**
- Enable promiscuous mode on Proxmox network bridge
- Use Calico BGP mode with router support
- Roll back to Flannel (chosen for this lab)

### kubeseal download returns "Not: command not found"

**Symptom:** kubeseal binary is a text file, not an executable  
**Cause:** Direct binary URL returned an error page instead of the binary  
**Fix:** Use GitHub API to detect latest version, download tar.gz, extract

---

## Validation Checklist

| Check | Command | Expected |
|---|---|---|
| Flannel running | `kubectl get pods -n kube-flannel` | All Running |
| Nodes Ready | `kubectl get nodes` | All Ready |
| NetworkPolicy enforcement | wget with/without policy | blocked/allowed |
| RBAC list pods | `kubectl auth can-i list pods --as=...` | yes |
| RBAC delete pods | `kubectl auth can-i delete pods --as=...` | no |
| Sealed Secrets running | `kubectl get pods -n kube-system \| grep sealed` | Running |
| Secret encrypt+decrypt | kubeseal → apply → get secret | original value |

---

*Phase 4 complete. Ready for Phase 5 — GitLab CI (md-ci-1 RAM upgrade to 4GB required before proceeding).*
