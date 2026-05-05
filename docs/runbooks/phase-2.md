# Phase 2 — MetalLB, ingress-nginx, and Storage

**Status:** ✅ Complete  
**Date:** 2026-05-04  
**Author:** Mohiuddin  
**Applies to:** md-cp-1 (all kubectl commands), cluster-wide components

---

## Overview

This runbook documents setting up the platform layer for a bare-metal Kubernetes cluster — LoadBalancer IP allocation via MetalLB, HTTP routing via ingress-nginx, and dynamic persistent storage via local-path-provisioner. These three components are the foundation every application in the cluster depends on.

**Why this matters:** In cloud environments (EKS, GKE, AKS), LoadBalancers, Ingress controllers, and StorageClasses are provisioned automatically. On bare-metal, none of these exist by default. This phase replicates what cloud providers give you for free, manually — which gives deep understanding of what managed services abstract away.

---

## Prerequisites

- 3-node Kubernetes cluster running (Phase 1 complete)
- All nodes in Ready state
- kubectl configured on md-cp-1

---

## Step 1 — Install Helm

**Why:** Helm is the Kubernetes package manager. It templates complex multi-resource applications (like MetalLB with its CRDs, controllers, and speakers) into a single installable unit with configurable values. Without Helm, each component would require manually applying dozens of YAML manifests.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Verify:**

```bash
helm version
# Expected: v3.20.2
```

---

## Step 2 — Install MetalLB

**Why MetalLB:** Kubernetes Service type `LoadBalancer` requires an external load balancer to assign an IP. Cloud providers handle this automatically. On bare-metal, without MetalLB, LoadBalancer Services stay in `<pending>` state forever — no external IP is ever assigned. MetalLB watches for LoadBalancer Services and assigns IPs from a configured pool.

**Why L2 mode:** MetalLB supports two modes — L2 (ARP-based) and BGP. L2 is simpler and works without any router configuration. MetalLB's speaker pod responds to ARP requests for the assigned IP, making the host (Proxmox node) forward traffic to the correct Kubernetes node.

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace
```

**Verify:**

```bash
kubectl get pods -n metallb-system
# Expected: controller = 1/1 Running, speaker pods = 4/4 Running (one per node)
```

---

## Step 3 — Configure MetalLB IP Pool

**Why a dedicated IP range:** MetalLB needs a pool of IPs it can assign to LoadBalancer Services. These IPs must be in the same subnet as the nodes (10.20.0.0/24) but must not conflict with existing node IPs or DHCP assignments. Range 10.20.0.200-210 is reserved for this purpose.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.20.0.200-10.20.0.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - lab-pool
EOF
```

**Verify:**

```bash
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

**Test — Deploy a temporary LoadBalancer Service:**

```bash
kubectl create deployment nginx-test --image=nginx
kubectl expose deployment nginx-test --port=80 --type=LoadBalancer
kubectl get svc nginx-test
# Expected: EXTERNAL-IP = 10.20.0.200
```

**Cleanup:**

```bash
kubectl delete deployment nginx-test
kubectl delete svc nginx-test
```

---

## Step 4 — Install ingress-nginx

**Why ingress-nginx:** A LoadBalancer Service gives one IP per Service. With multiple applications (ArgoCD, Grafana, Gitea, OTel Demo), that would consume multiple IPs and require users to remember port numbers. Ingress allows hostname-based routing — a single IP (10.20.0.200) routes to different Services based on the `Host` header. `argocd.lab` → ArgoCD, `grafana.lab` → Grafana, etc.

**Why replicaCount=1:** This is a lab environment. Running a single replica saves memory. Production deployments use 2+ replicas for availability.

**Why nodeSelector:** Pinning ingress-nginx to worker nodes ensures it does not run on the control plane, which is reserved for Kubernetes system components.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=1 \
  --set controller.nodeSelector."node-role\.kubernetes\.io/worker"=worker
```

**Verify:**

```bash
kubectl get pods -n ingress-nginx
# Expected: ingress-nginx-controller = 1/1 Running

kubectl get svc -n ingress-nginx
# Expected: ingress-nginx-controller TYPE=LoadBalancer EXTERNAL-IP=10.20.0.200
```

---

## Step 5 — Configure Windows hosts File

**Why:** Lab DNS resolution without a dedicated DNS server. The Windows host file maps hostnames to IP addresses locally. All lab services share the same IP (10.20.0.200) — ingress-nginx routes to the correct backend based on the hostname.

Open Notepad as Administrator, edit `C:\Windows\System32\drivers\etc\hosts`, add:

```
10.20.0.200  argocd.lab
10.20.0.200  grafana.lab
10.20.0.200  gitea.lab
10.20.0.200  otel-demo.lab
10.20.0.200  prometheus.lab
```

**Verify from cluster:**

```bash
curl http://10.20.0.200
# Expected: 404 Not Found (nginx) — ingress is responding, no rules configured yet
```

> **Note:** If accessing from Windows via Tailscale, the 10.20.0.0/24 subnet may not be routed through Tailscale. Use curl from md-cp-1 for cluster-internal verification.

---

## Step 6 — Install local-path-provisioner

**Why:** Kubernetes PersistentVolumeClaims require a StorageClass to dynamically provision volumes. Without a provisioner, every application needing storage requires a manually pre-created PersistentVolume. local-path-provisioner automatically creates hostPath volumes on the node where the pod is scheduled.

**Why local-path over Longhorn:** Longhorn is a distributed storage system that replicates data across nodes. It uses approximately 500MB+ RAM per node. local-path-provisioner uses approximately 30MB. For a lab environment where data replication is not required, local-path is the correct tradeoff.

**Tradeoff accepted:** local-path volumes are node-bound. If a pod moves to a different node, it cannot access the volume from the previous node. This is acceptable in a lab but not in production.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

**Set as default StorageClass:**

```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**Verify:**

```bash
kubectl get storageclass
# Expected: local-path (default)
```

---

## Step 7 — Test PVC Provisioning

**Why test:** Deploying applications with PVCs without first validating the StorageClass wastes debugging time. A 2-minute test here prevents hours of troubleshooting later.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc test-pvc
# Expected: Pending (WaitForFirstConsumer — normal until pod mounts it)
```

**Mount with a test pod:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: test-vol
  volumes:
  - name: test-vol
    persistentVolumeClaim:
      claimName: test-pvc
EOF

kubectl get pvc test-pvc && kubectl get pod test-pod
# Expected: PVC = Bound, Pod = Running
```

**Cleanup:**

```bash
kubectl delete pod test-pod && kubectl delete pvc test-pvc
```

---

## Break-it Drill — ingress-nginx Pod Failure

### Scenario

Simulate ingress-nginx controller crash and measure Kubernetes self-healing time.

### Key Difference from Static Pod Drill

Unlike kube-apiserver (a static pod managed directly by kubelet), ingress-nginx runs as a **Deployment**. When the pod is deleted, the ReplicaSet controller — running on the control plane — detects that the desired replica count (1) is not met and schedules a new pod automatically. No manual intervention required.

### Steps

```bash
# Get current pod name
kubectl get pods -n ingress-nginx

# Delete the pod
kubectl delete pod <ingress-nginx-controller-pod-name> -n ingress-nginx

# Watch recovery
kubectl get pods -n ingress-nginx -w
```

### Results

| Metric | Value |
|---|---|
| Failure type | Pod deleted manually |
| Recovery mechanism | Deployment ReplicaSet auto-restart |
| Recovery time | ~30 seconds |
| Manual intervention | None required |

---

## Troubleshooting

### MetalLB speaker pods stuck in Init state

**Symptom:** Speaker pods show `Init:0/3` for more than 2 minutes  
**Cause:** Init containers pulling images slowly  
**Fix:** Wait — image pull completes and pods transition to Running

### LoadBalancer Service stuck in `<pending>` EXTERNAL-IP

**Symptom:** `kubectl get svc` shows `<pending>` under EXTERNAL-IP  
**Cause:** MetalLB L2Advertisement not configured, or IP pool exhausted  
**Fix:** Verify `kubectl get ipaddresspool -n metallb-system` shows the pool

### PVC stuck in Pending after pod creation

**Symptom:** PVC remains Pending even after pod is running  
**Cause:** local-path-provisioner pod not running, or wrong StorageClass  
**Fix:** Check `kubectl get pods -n local-path-storage` and `kubectl get storageclass`

### Worker node kubectl connection refused

**Symptom:** `kubectl` on worker node returns `connection refused to localhost:8080`  
**Cause:** kubeconfig not configured on worker nodes — only control plane has it by default  
**Fix:** Run all kubectl commands from md-cp-1, or copy kubeconfig to worker nodes

---

## Validation Checklist

| Check | Command | Expected |
|---|---|---|
| MetalLB pods | `kubectl get pods -n metallb-system` | All Running |
| IP pool configured | `kubectl get ipaddresspool -n metallb-system` | lab-pool with 10.20.0.200-210 |
| ingress-nginx running | `kubectl get pods -n ingress-nginx` | 1/1 Running |
| ingress-nginx IP | `kubectl get svc -n ingress-nginx` | EXTERNAL-IP = 10.20.0.200 |
| StorageClass default | `kubectl get storageclass` | local-path (default) |
| PVC test | PVC = Bound after pod mount | ✅ |
| Break-it recovery | Pod deleted → new pod in ~30s | ✅ |

---

*Phase 2 complete. Ready for Phase 3 — ArgoCD GitOps engine.*
