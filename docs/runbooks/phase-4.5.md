# Phase 4.5 — Production Foundation (metrics-server, Root CA, cert-manager, ArgoCD hardening)

**Status:** 🔄 Partially Complete  
**Date:** 2026-05-06  
**Author:** Mohiuddin  
**Applies to:** md-cp-1 (control plane), md-svc-1 (Root CA)

---

## Overview

This runbook documents hardening the cluster with production-grade components before moving to CI/CD. Phase 4.5 fills the gaps left by the initial setup — resource metrics, TLS infrastructure, proper ArgoCD security, and namespace-level pod security enforcement.

**Why Phase 4.5 exists:** Phase 0-4 built the foundation. Phase 4.5 makes it production-grade before applications arrive. Security and observability infrastructure must be in place before workloads, not retrofitted after.

---

## Prerequisites

- Phase 0-4 complete
- ArgoCD running
- cert-manager CRDs not yet installed
- md-svc-1 accessible via SSH

---

## Step 1 — metrics-server

**Why:** Without metrics-server, `kubectl top` returns no metrics, and HPA (Horizontal Pod Autoscaler) cannot function. It is a required addon for any production cluster.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Fix for kubeadm clusters:** kubeadm uses self-signed certificates for kubelet. metrics-server rejects these by default. Add `--kubelet-insecure-tls` flag:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Verify:**

```bash
kubectl get pods -n kube-system | grep metrics-server
# Expected: 1/1 Running

kubectl top nodes
# Expected: CPU and memory usage per node
```

> **Why --kubelet-insecure-tls:** In production with proper PKI, you would configure metrics-server to trust the cluster CA instead. For a lab with self-signed kubelet certs, this flag is the correct tradeoff — documented as a known limitation.

---

## Step 2 — ArgoCD Admin Password Change

**Why:** The initial admin password is stored in a Kubernetes secret and is trivially retrievable by anyone with cluster access. Changing it and deleting the initial secret is mandatory before exposing ArgoCD to any network.

```bash
# Start port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Login with initial password
argocd login localhost:8080 --username admin --password <initial-password> --insecure

# Change password
argocd account update-password \
  --current-password <initial-password> \
  --new-password 'DevOps@Homelab2026!'

# Delete initial secret
kubectl delete secret argocd-initial-admin-secret -n argocd
```

**Verify:**

```bash
argocd login localhost:8080 --username admin --password 'DevOps@Homelab2026!' --insecure
# Expected: 'admin:login' logged in successfully
```

---

## Step 3 — Pod Security Standards

**Why:** Pod Security Standards (PSS) enforce security constraints at the namespace level. Without PSS labels, any pod can run as root, mount host paths, or use privileged containers. Baseline level blocks the most dangerous capabilities while allowing common workloads.

```bash
# Apply baseline PSS to platform namespaces
kubectl label namespace argocd \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline

kubectl label namespace ingress-nginx \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline

kubectl label namespace metallb-system \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=baseline
```

**Verify:**

```bash
kubectl get namespace argocd ingress-nginx metallb-system --show-labels | grep pod-security
```

---

## Step 4 — Root CA Setup (md-svc-1)

**Why:** A self-signed Root CA allows all cluster services to use TLS with certificates trusted by the lab environment. This is the equivalent of an internal PKI in production.

Run on **md-svc-1:**

```bash
sudo apt-get install -y openssl
sudo mkdir -p /etc/ssl/lab-ca

# Generate 4096-bit RSA private key
sudo openssl genrsa -out /etc/ssl/lab-ca/ca.key 4096

# Generate self-signed root certificate (10 year validity)
sudo openssl req -new -x509 -days 3650 \
  -key /etc/ssl/lab-ca/ca.key \
  -out /etc/ssl/lab-ca/ca.crt \
  -subj "/C=US/ST=Lab/L=Lab/O=Mohiuddin-Homelab/CN=Mohiuddin-Lab-CA"
```

**Verify:**

```bash
sudo openssl x509 -in /etc/ssl/lab-ca/ca.crt -text -noout | grep -E "Issuer|Not After|Not Before"
```

**Copy CA files to md-cp-1:**

```bash
# On md-svc-1 — make key readable temporarily
sudo cp /etc/ssl/lab-ca/ca.key /tmp/ca.key
sudo chmod 644 /tmp/ca.key

# On md-cp-1 — copy files
scp mohiuddin@10.20.0.50:/etc/ssl/lab-ca/ca.crt /tmp/lab-ca.crt
scp mohiuddin@10.20.0.50:/tmp/ca.key /tmp/lab-ca.key
```

---

## Step 5 — cert-manager Install

**Why:** cert-manager automates TLS certificate lifecycle — issuance, renewal, and rotation. Without it, certificates must be manually created and rotated. cert-manager integrates with Kubernetes Ingress resources to automatically provision TLS.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

**Wait 60 seconds, then verify:**

```bash
kubectl get pods -n cert-manager
# Expected: cert-manager, cert-manager-cainjector, cert-manager-webhook = all 1/1 Running
```

---

## Step 6 — ClusterIssuer from Root CA

**Why:** A ClusterIssuer tells cert-manager how to sign certificates. Using our Root CA means all issued certificates are trusted by any client that trusts the Root CA.

```bash
# Create secret with CA credentials
kubectl create secret tls lab-ca-secret \
  --cert=/tmp/lab-ca.crt \
  --key=/tmp/lab-ca.key \
  -n cert-manager

# Create ClusterIssuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: lab-ca-issuer
spec:
  ca:
    secretName: lab-ca-secret
EOF
```

**Verify:**

```bash
kubectl get clusterissuer lab-ca-issuer
# Expected: READY = True
```

---

## Step 7 — ArgoCD HTTPS with cert-manager

**Why:** ArgoCD should only be accessible over HTTPS. cert-manager automatically provisions and renews the certificate.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    cert-manager.io/cluster-issuer: "lab-ca-issuer"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - argocd.lab
    secretName: argocd-tls
  rules:
  - host: argocd.lab
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
EOF
```

**Verify:**

```bash
kubectl get certificate -n argocd
# Expected: argocd-tls READY = True
```

---

## Step 8 — DNS Configuration

**Why:** All VMs need to resolve lab hostnames. Without this, ArgoCD CLI and inter-service communication fails.

Run on **all VMs:**

```bash
echo "10.20.0.200 argocd.lab grafana.lab gitlab.lab otel-demo.lab prometheus.lab" | sudo tee -a /etc/hosts
```

---

## Step 9 — ArgoCD CLI Direct Login (No Port-Forward)

```bash
argocd login argocd.lab \
  --username admin \
  --password 'DevOps@Homelab2026!' \
  --insecure
```

**Expected:** `'admin:login' logged in successfully`

---

## Step 10 — ArgoCD RBAC + Project

```bash
argocd proj create homelab \
  --description "Mohiuddin Homelab Project" \
  --dest https://kubernetes.default.svc,* \
  --src "*"
```

**Verify:**

```bash
argocd proj list
# Expected: homelab project listed
```

---

## Step 11 — Trust Root CA on Windows

1. Download `lab-ca.crt` from md-cp-1 via MobaXterm
2. Double-click the `.crt` file
3. Click **Install Certificate** → **Local Machine** → Next
4. Select **Place all certificates in the following store**
5. Browse → **Trusted Root Certification Authorities**
6. Next → Finish → Yes

**Verify:** Open `https://argocd.lab` in browser — no certificate warning.

---

## Pending (Requires Tuhin — Proxmox promiscuous mode)

| Task | Reason blocked |
|---|---|
| Calico CNI | Cross-node networking needs promiscuous mode |
| Longhorn storage | Replica replication needs promiscuous mode |
| local-path removal | Replace with Longhorn after it is installed |
| md-ci-1 RAM 4GB | Required for GitLab CE |

---

## Troubleshooting

### metrics-server pods not ready

**Symptom:** `0/1 Running` indefinitely  
**Cause:** kubelet serving cert not trusted  
**Fix:** Add `--kubelet-insecure-tls` arg to metrics-server deployment

### argocd.lab DNS resolution failure

**Symptom:** `no such host` error in argocd CLI  
**Cause:** `/etc/hosts` missing lab hostname entries  
**Fix:** Add `10.20.0.200 argocd.lab` to `/etc/hosts` on all VMs

### scp permission denied for ca.key

**Symptom:** `Permission denied` when copying ca.key  
**Cause:** ca.key is root-owned (600 permissions)  
**Fix:** `sudo cp /etc/ssl/lab-ca/ca.key /tmp/ca.key && sudo chmod 644 /tmp/ca.key`

---

## Validation Checklist

| Check | Command | Expected |
|---|---|---|
| metrics-server running | `kubectl get pods -n kube-system \| grep metrics` | 1/1 Running |
| kubectl top works | `kubectl top nodes` | CPU/RAM per node |
| PSS labels set | `kubectl get ns argocd --show-labels` | pod-security labels present |
| ClusterIssuer ready | `kubectl get clusterissuer` | READY = True |
| ArgoCD certificate | `kubectl get certificate -n argocd` | READY = True |
| ArgoCD HTTPS | browser `https://argocd.lab` | No certificate warning |
| ArgoCD CLI login | `argocd login argocd.lab` | logged in successfully |
| ArgoCD project | `argocd proj list` | homelab project listed |

---

*Phase 4.5 partially complete. Calico + Longhorn pending Proxmox promiscuous mode from Tuhin.*
