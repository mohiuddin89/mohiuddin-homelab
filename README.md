# Mohiuddin's DevOps & Platform Engineering Homelab

> **Production-grade Kubernetes platform** built from scratch — kubeadm cluster, GitOps (ArgoCD), CI/CD (GitLab), full observability (Prometheus + Grafana + Loki + Tempo), and OpenTelemetry instrumentation.

[![Status](https://img.shields.io/badge/status-active-brightgreen)]()
[![Phase](https://img.shields.io/badge/phase-4.5-blue)]()
[![Kubernetes](https://img.shields.io/badge/kubernetes-1.31.14-326CE5)]()

---

## 🎯 What This Is

A bare-metal Kubernetes homelab built on **Proxmox** (i9-10850K, 64GB DDR4) where I install every component manually to deeply understand production DevOps tooling — no managed services, no shortcuts.

**End goal:** Deploy the OpenTelemetry Demo microservices app with full GitOps workflow and trace-log-metrics correlation in Grafana.

---

## 🏗️ Architecture at a Glance

```
Proxmox Server (i9-10850K, 64GB DDR4)
├── md-svc-1 (10.20.0.50) — Root CA + DNS
├── md-cp-1  (10.20.0.10) — K8s Control Plane
├── md-w-1   (10.20.0.21) — K8s Worker 1
├── md-w-2   (10.20.0.22) — K8s Worker 2
└── md-ci-1  (10.20.0.30) — GitLab CI Server
```

**Stack:** kubeadm · Calico · Longhorn · MetalLB · ingress-nginx · cert-manager · ArgoCD · GitLab · Prometheus · Grafana · Loki · Tempo · OTel Collector

📖 **See [ARCHITECTURE.md](./ARCHITECTURE.md)** for full details, traffic flows, and design decisions.

---

## ✅ Progress

| Phase | Component | Status |
|---|---|---|
| 0 | VM baseline (SSH, swap, kernel, containerd) | ✅ |
| 1 | kubeadm cluster + Flannel CNI | ✅ |
| 1.5 | etcd backup + restore drill | ✅ |
| 2 | Helm + MetalLB + ingress-nginx + storage | ✅ |
| 3 | ArgoCD GitOps engine | ✅ |
| 4 | RBAC + Sealed Secrets + NetworkPolicy | ✅ |
| 4.5 | Calico + Longhorn + Root CA + cert-manager | 🔄 In Progress |
| 5 | GitLab CE + CI pipeline | ⏳ |
| 6 | Full GitOps loop (two-repo pattern) | ⏳ |
| 7-10 | Observability stack (Prometheus, Grafana, Loki, Tempo, OTel) | ⏳ |
| 11 | OpenTelemetry Demo deploy | ⏳ |
| 12 | Backup + disaster recovery drill | ⏳ |
| 13-15 | Terraform + AWS + EKS migration | ⏳ |

---

## 📚 Documentation

### Runbooks (per phase)
- [Phase 0 — Environment Baseline](./docs/runbooks/phase-0.md)
- [Phase 1 — kubeadm Cluster + etcd Backup](./docs/runbooks/phase-1.md)
- [Phase 2 — MetalLB, ingress-nginx, Storage](./docs/runbooks/phase-2.md)
- [Phase 3 — ArgoCD GitOps](./docs/runbooks/phase-3.md)
- [Phase 4 — Security Baseline](./docs/runbooks/phase-4.md)

### Interview Prep
- [Phase 1 Q&A — kubeadm + etcd](./docs/interview-prep/phase-1-qa.md)
- [Phase 2 Q&A — MetalLB + Ingress + Storage](./docs/interview-prep/phase-2-qa.md)
- [Phase 3 Q&A — ArgoCD + GitOps](./docs/interview-prep/phase-3-qa.md)
- [Phase 4 Q&A — Security Baseline](./docs/interview-prep/phase-4-qa.md)

---

## 💥 Production Mindset — Break-it Drills

Every phase ends with mandatory failure simulation. Phase pass condition is not "it worked" — it is **"I broke it, debugged it, fixed it, and documented it."**

Examples completed:
- **kube-apiserver kill** — moved static pod manifest, measured 30-second recovery
- **ingress-nginx pod kill** — Deployment ReplicaSet auto-recovery validated
- **NetworkPolicy enforcement** — default-deny verified, explicit allow rules tested

See [ARCHITECTURE.md](./ARCHITECTURE.md#-break-it-drills--per-phase) for the full drill plan across all phases.

---

## 🎓 What I Am Learning

- Kubernetes control plane internals (etcd, API server, scheduler, kubelet, static pods)
- Container networking (CNI plugins, NetworkPolicy, pod-to-pod traffic flow)
- GitOps principles and ArgoCD operator patterns
- Production CI/CD with image scanning and vulnerability gates
- OpenTelemetry instrumentation and observability correlation
- Linux networking, kernel parameters, systemd services
- Production debugging — reading logs, tracing failures, measuring recovery

---

## 🛠️ Built By

**Mohiuddin** — DevOps / Platform Engineer candidate (US market)
GitHub: [@mohiuddin89](https://github.com/mohiuddin89)

---

*This is a learning project built with strict production-grade standards — no shortcuts, every drill mandatory.*
