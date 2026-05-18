# ADR 006 — kubeadm Before EKS

## Status
Accepted

## Context
Project needs a Kubernetes cluster. Two options evaluated:
start with managed EKS, or install manually with kubeadm first
then migrate to EKS in Phase 16.

## Decision
Install Kubernetes manually with **kubeadm first**, migrate to
EKS via Terraform in Phase 16.

## Reasons
- kubeadm exposes every control plane component directly:
  etcd, kube-apiserver, kube-scheduler, kube-controller-manager
- Manual install forces understanding of kubelet, containerd,
  CNI plugins, TLS certificates, and static pods
- Break-it drills (kill API server, corrupt etcd, remove CNI)
  are only possible on kubeadm — EKS hides all of this
- Phase 16 EKS comparison is only meaningful after kubeadm
  experience — you know exactly what AWS is automating for you
- Interview signal: "I bootstrapped a cluster with kubeadm,
  then migrated to EKS via Terraform — here is what changed"
  is far stronger than "I used EKS from the start"

## Consequences
- More setup time (Phase 0-2) compared to EKS click-through
- etcd backup is mandatory (single control plane, no HA)
- Full control plane debugging experience gained
- Phase 16 EKS comparison has direct learning value

## Alternatives Rejected
- **EKS from start** — hides control plane internals, no deep
  learning value, break-it drills not possible
