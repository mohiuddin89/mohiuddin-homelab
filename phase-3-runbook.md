# Phase 3 — ArgoCD GitOps Engine

**Status:** ✅ Complete  
**Date:** 2026-05-04  
**Author:** Mohiuddin  
**Applies to:** md-cp-1 (all kubectl/argocd commands)

---

## Overview

This runbook documents installing ArgoCD as the GitOps engine for the cluster. ArgoCD watches Git repositories and automatically syncs the desired state declared in Git to the actual state in Kubernetes. This eliminates manual `kubectl apply` for deployments and makes Git the single source of truth for the cluster.

**Why GitOps:** In traditional CI/CD, a pipeline pushes changes directly to the cluster. In GitOps, the cluster pulls changes from Git. This means the cluster state is always auditable, rollbacks are a Git revert, and no pipeline needs direct cluster credentials.

---

## Prerequisites

- Phase 2 complete (ingress-nginx, MetalLB running)
- `argocd.lab` in Windows hosts file pointing to 10.20.0.200

---

## Step 1 — Install ArgoCD

**Why a dedicated namespace:** ArgoCD has many components — keeping them isolated in the `argocd` namespace makes it easier to manage, audit, and apply RBAC policies.

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait 2-3 minutes for all pods to start.

**Verify:**

```bash
kubectl get pods -n argocd
```

**Expected:** All pods in Running state:
- argocd-application-controller
- argocd-applicationset-controller
- argocd-dex-server
- argocd-notifications-controller
- argocd-redis
- argocd-repo-server
- argocd-server

---

## Troubleshooting — ApplicationSet CrashLoopBackOff

**Symptom:** `argocd-applicationset-controller` enters CrashLoopBackOff with error:
```
failed to get restmapping: no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"
```

**Cause:** The `ApplicationSet` CRD annotation exceeded the 262144 byte limit during install and was not created.

**Fix:**

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/applicationset-crd.yaml

kubectl rollout restart deployment argocd-applicationset-controller -n argocd
```

**Verify:**

```bash
kubectl get pods -n argocd | grep applicationset
# Expected: 1/1 Running
```

---

## Step 2 — Configure Ingress

**Why HTTPS passthrough:** ArgoCD server enforces HTTPS. The ingress must pass TLS traffic directly to the ArgoCD server without terminating it — otherwise the gRPC API used by the CLI breaks.

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
spec:
  ingressClassName: nginx
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
kubectl get ingress -n argocd
# Expected: ADDRESS = 10.20.0.200
```

---

## Step 3 — Retrieve Admin Password

**Why it is stored as a secret:** ArgoCD generates a random initial admin password and stores it as a Kubernetes secret. This is a one-time bootstrap credential — you should change it after first login in production.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Save the output — this is the initial admin password.

**Login credentials:**
- Username: `admin`
- Password: (output from above command)

---

## Step 4 — Install ArgoCD CLI

**Why CLI:** The ArgoCD UI is useful for visualization, but the CLI is essential for automation, scripting, and CI/CD integration. Production workflows use the CLI to create apps, trigger syncs, and check health.

```bash
curl -sSL -o /tmp/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 /tmp/argocd /usr/local/bin/argocd
```

**Verify:**

```bash
argocd version --client
```

---

## Step 5 — Login via CLI

**Why port-forward:** The Ingress SSL passthrough configuration causes gRPC CLI connections to return 404 when routing through ingress-nginx. Port-forwarding bypasses the ingress and connects directly to the ArgoCD server service, which resolves the issue.

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

argocd login localhost:8080 \
  --username admin \
  --password <your-password> \
  --insecure
```

**Expected:**
```
'admin:login' logged in successfully
```

---

## Step 6 — Deploy First GitOps Application

**Why this test:** Validates the complete GitOps flow end-to-end — ArgoCD connects to a public Git repo, reads Kubernetes manifests, and applies them to the cluster automatically.

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

**Verify:**

```bash
argocd app get guestbook
# Expected: Sync Status = Synced, Health Status = Healthy

kubectl get pods -n default
# Expected: guestbook-ui pod Running
```

**Cleanup after validation:**

```bash
argocd app delete guestbook --cascade
```

---

## ArgoCD Architecture — How It Works

```
Git Repository
     │
     │ (ArgoCD polls every 3 minutes or webhook)
     ▼
ArgoCD Repo Server
(clones repo, renders manifests)
     │
     ▼
ArgoCD Application Controller
(compares desired state in Git vs actual state in cluster)
     │
     ├── Synced: no action needed
     └── OutOfSync: apply changes to cluster
          │
          ▼
     Kubernetes API Server
     (creates/updates/deletes resources)
```

**Key concepts:**
- **Desired state:** What is in Git
- **Actual state:** What is running in the cluster
- **Sync:** The action of making actual state match desired state
- **Auto-sync:** ArgoCD syncs automatically when it detects drift

---

## Validation Checklist

| Check | Command | Expected |
|---|---|---|
| All ArgoCD pods running | `kubectl get pods -n argocd` | All 1/1 Running |
| Ingress configured | `kubectl get ingress -n argocd` | ADDRESS = 10.20.0.200 |
| UI accessible | browser `https://argocd.lab` | Login page |
| CLI login | `argocd login localhost:8080` | logged in successfully |
| GitOps deploy | guestbook app Synced + Healthy | ✅ |

---

*Phase 3 complete. Ready for Phase 4 — Security Baseline (Calico, NetworkPolicy, RBAC, Sealed Secrets).*
