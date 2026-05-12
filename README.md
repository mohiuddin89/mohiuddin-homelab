# Production-Grade Kubernetes Homelab

> Bare-metal Kubernetes cluster built manually on Proxmox with full GitOps, observability, and supply-chain-secured CI/CD — designed to mirror production engineering practices end-to-end.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.31-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Calico](https://img.shields.io/badge/CNI-Calico-FF6B35)](https://www.tigera.io/project-calico/)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io)
[![GitLab](https://img.shields.io/badge/CI%2FCD-GitLab-FC6D26?logo=gitlab&logoColor=white)](https://gitlab.com)
[![Prometheus](https://img.shields.io/badge/Metrics-Prometheus-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io)
[![Jaeger](https://img.shields.io/badge/Traces-Jaeger-66CFE3?logo=jaeger&logoColor=white)](https://www.jaegertracing.io)
[![Cosign](https://img.shields.io/badge/Image%20Signing-Cosign-2F88FF)](https://www.sigstore.dev)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

---

## 📋 Overview

A fully manual, production-grade Kubernetes platform built on bare-metal Proxmox infrastructure. Every component is installed by hand to build deep understanding of underlying systems — from `kubeadm` bootstrap through GitOps deployment, image signing, distributed tracing, and disaster recovery.

**The goal:** demonstrate end-to-end platform engineering — not "I deployed an app," but **"I built the platform that deploys the app, secures it, monitors it, breaks it, and recovers it."**

Every phase ends with a mandatory **break-it drill**: kill a component, debug the failure, measure recovery time, write a runbook.

📐 **Full architecture and design rationale:** [ARCHITECTURE.md](./ARCHITECTURE.md)

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Proxmox VE — i9-10850K, 64GB DDR4 — Accessed via Tailscale      │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Kubernetes Cluster (kubeadm v1.31.x)                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │  │
│  │  │  md-cp-1    │  │   md-w-1    │  │   md-w-2    │         │  │
│  │  │ API Server  │  │ App Pods    │  │ App Pods +  │         │  │
│  │  │ etcd        │  │             │  │ Ingress     │         │  │
│  │  │ ArgoCD      │  │             │  │             │         │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │  │
│  │                                                            │  │
│  │  CNI: Calico  │  LB: MetalLB  │  Storage: Longhorn         │  │
│  │  Ingress: ingress-nginx  │  TLS: cert-manager + Root CA    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  md-ci-1: GitLab CE + Runner + Container Registry + Cosign       │
│  md-svc-1: Internal DNS (dnsmasq) + Root CA                      │
└──────────────────────────────────────────────────────────────────┘
```

### Traffic Flow

```
User → MetalLB IP → ingress-nginx (HTTPS) → Kubernetes Service → Pod
```

### CI/CD + GitOps Flow

```
git push  →  GitLab CI  →  Kaniko build  →  Trivy scan  →  Cosign sign
                                                                │
                                                                ▼
                                            GitLab Registry (signed image)
                                                                │
                                                                ▼
                                        CI updates apps-values repo
                                                                │
                                                                ▼
                                        ArgoCD verifies signature → Deploy
                                                ├── dev: auto-sync
                                                └── prod: manual gate
```

### Observability Flow (OTLP Everywhere)

```
Application Pods
        │
        ▼  OTLP gRPC :4317
OpenTelemetry Collector (gateway mode)
        ├── Metrics ─→ Prometheus
        ├── Logs    ─→ Loki
        └── Traces  ─→ Jaeger (OTLP)
                │
                ▼
            Grafana (unified UI — trace ↔ logs ↔ metrics correlation)
```

---

## 📊 Current Status

| Phase | Focus | Status |
|---|---|---|
| **0** | Proxmox + Networking + Tailscale | ✅ Complete |
| **1** | VM baseline + SSH hardening | ✅ Complete |
| **2** | Kubernetes (kubeadm) + Calico CNI | ✅ Complete |
| **2.5** | etcd backup + restore drill | ✅ Complete |
| **3** | MetalLB + ingress-nginx + Longhorn | ✅ Complete |
| **4** | cert-manager + NetworkPolicy + PSS + LimitRange + RBAC + ESO | ✅ Complete |
| **5** | GitLab CI + Kaniko + Trivy + **Cosign signing** | ✅ Complete |
| **6** | ArgoCD app-of-apps + two-repo GitOps | 🔄 **In progress** (apps-source + apps-values done; ArgoCD Application pending) |
| **7** | External Secrets Operator (local backend) | ⏳ Upcoming |
| **8** | Prometheus + Alertmanager (Slack routing) + Grafana | ⏳ Upcoming |
| **9** | Loki + Promtail | ⏳ Upcoming |
| **10** | Jaeger (OTLP receiver) | ⏳ Upcoming |
| **11** | OTel Collector + **DORA metrics dashboard** | ⏳ Upcoming |
| **12** | OpenTelemetry Demo workload + **HPA** | ⏳ Upcoming |
| **13** | Velero backup + full disaster recovery drill | ⏳ Upcoming |
| **14–16** | Terraform + AWS + EKS migration | ⏳ Upcoming |
| **17–22** | AI Platform (LiteLLM, Langfuse, RAG, Eval, Guardrails) | ⏳ Upcoming |

---

## 🛠️ Tech Stack

### Cluster Core
Kubernetes 1.31 (kubeadm) · containerd · Calico CNI · MetalLB · ingress-nginx · Longhorn · cert-manager · metrics-server

### Security
NetworkPolicy (default-deny + explicit allow list) · Pod Security Standards (restricted / baseline / privileged) · LimitRange per namespace · PodDisruptionBudget · External Secrets Operator · Root CA + cert-manager · RBAC (least privilege) · seccompProfile RuntimeDefault

### CI/CD + GitOps
GitLab CE · GitLab Runner · Kaniko (privilegeless builds) · Trivy (CVE scanning, HIGH/CRITICAL gate) · Cosign (supply-chain signing) · ArgoCD (signature verification before deploy) · Two-repo GitOps · Semantic versioning + SHA tags

### Observability
Prometheus + Alertmanager · Grafana · Loki + Promtail · Jaeger (OTLP) · OpenTelemetry Collector (gateway mode) · DORA metrics dashboard · Slack alert routing

### Backup + Recovery
etcd snapshots (6h cron) · Velero (namespace-level PVC backup) · Longhorn recurring snapshots · GitLab → GitHub mirror (offsite code backup)

### Workload
OpenTelemetry Demo (16+ microservices) with custom Dockerfiles + HPA on high-traffic services

### Cloud + IaC (Phase 14+)
Terraform · AWS VPC · Route53 · SSM Parameter Store · S3 · IAM · EKS · ALB Controller

### AI Platform (Phase 17+)
LiteLLM gateway · Langfuse tracing · pgvector + RAG pipeline · Promptfoo/Ragas evaluation · Prompt registry (Git-versioned)

---

## ✨ Production-Grade Practices Demonstrated

This is not "spin up a cluster and call it done." Every choice maps to a real production engineering practice:

- **Manual-first learning** — kubeadm, Helm values, CNI installation all done by hand before any IaC abstraction
- **Supply chain security** — every container image signed with Cosign; ArgoCD refuses unsigned deployments
- **Defense in depth** — NetworkPolicy default-deny, PSS restricted, LimitRange, PDB, seccompProfile
- **OTLP-everywhere observability** — vendor-neutral telemetry pipeline; trace ↔ logs ↔ metrics correlation
- **GitOps purity** — two-repo pattern; ArgoCD is the only thing that changes cluster state in dev/staging/prod
- **DORA metrics** — deployment frequency, lead time, change failure rate, MTTR tracked in Grafana
- **Disaster recovery culture** — every phase has break-it drills with measured RTO and documented runbooks
- **Cost discipline** — EKS phase is intentionally short (`terraform destroy` same day) to demonstrate cloud comparison without sustained spend
- **Architecture Decision Records** — 17 ADRs explain *why* each tool was chosen and what was rejected

---

## 💥 Break-it Discipline

Phase pass criteria is not "it worked." It is:

> **"I broke it, debugged it, fixed it, measured recovery time, and committed a runbook."**

Every phase ends with one or more drills:

| Phase | Example Drill | What It Teaches |
|---|---|---|
| etcd backup | Delete a namespace → restore from snapshot | Full etcd recovery procedure + RTO |
| Networking | Apply too-strict NetworkPolicy → debug | Calico policy trace, traffic flow analysis |
| Storage | Force-delete Longhorn replica | Auto-rebuild behavior, replica health |
| CI/CD | Push image with HIGH CVE | Trivy gate validation |
| GitOps | `kubectl edit` while ArgoCD watches | Drift detection + selfHeal |
| Observability | OTel Collector OOM | `memory_limiter` processor behavior |
| Workload | Kill paymentservice mid-trace | Cascading failure analysis in Jaeger |
| Backup | Delete entire namespace | Velero restore + RTO measurement |

All drill outcomes are documented in [`docs/runbooks/failures/`](./docs/runbooks/failures/).

---

## 📁 Repository Structure

```
mohiuddin-homelab/
├── README.md                     ← You are here
├── ARCHITECTURE.md               ← Full design + ADRs
├── docs/
│   ├── runbooks/                 ← Per-phase how-to (English)
│   │   ├── phase-0-prereq.md
│   │   ├── phase-1-kubeadm.md
│   │   ├── phase-2.5-etcd-backup.md
│   │   └── failures/             ← Break-it drill outcomes + RTO logs
│   ├── decisions/                ← Architecture Decision Records (017+)
│   ├── interview-prep/           ← Q&A per phase
│   └── diagrams/                 ← PNG/SVG architecture diagrams
├── manifests/                    ← Raw K8s YAMLs (Phase 12 raw layer)
├── helm-values/                  ← Per-chart values.yaml files
├── apps-values/                  ← GitOps source repo (ArgoCD watches)
├── dockerfiles/                  ← Custom Dockerfiles per OTel Demo service
└── terraform/                    ← Phase 14+ (bootstrap/ + envs/lab/)
```

---

## 📐 Key Architecture Decisions

A subset of the [17 ADRs](./docs/decisions/) — full reasoning is in each linked document:

| ADR | Decision | Reason |
|---|---|---|
| 001 | Calico over Flannel | NetworkPolicy enforcement required |
| 002 | Longhorn over local-path | Replication + snapshots + Velero CSI integration |
| 003 | GitLab over Gitea | Full CI/CD + production parity |
| 006 | kubeadm before EKS | Control plane internals before managed abstraction |
| 008 | Two-repo GitOps | Clean code/deploy separation; rollback = values revert |
| 011 | Kaniko over Docker-in-Docker | Privilegeless builds, production-safe |
| 014 | Velero over manual `kubectl cp` | Application-consistent snapshots, namespace restore |
| 015 | Cosign image signing | Supply chain security; signed-only deploys |
| 016 | Semver + SHA tags | Human-readable deployment history |
| 017 | LimitRange per namespace | Prevents unbounded resource consumption |

---

## ⚠️ Documented Tradeoffs

Every shortcut is intentional and has a documented production equivalent:

| Tradeoff | Reason | Production Equivalent |
|---|---|---|
| Single control plane | Lab resource constraint | 3-node etcd + 2+ control planes behind LB |
| Ingress co-located on worker | No dedicated ingress node | Dedicated ingress NodePool (demonstrated on EKS) |
| 2-replica Longhorn | Resource pressure | 3-replica (requires 3+ workers) |
| Self-signed Root CA | No public domain | Let's Encrypt + real domain |
| In-cluster observability | Simplicity | External monitoring for cluster-level health |
| Single Tailscale subnet router | One Proxmox host | Redundant subnet routers |

---

## 🎯 What This Project Demonstrates

For hiring managers and recruiters:

- **Platform engineering depth** — not just app deployment, but the platform that runs it
- **Production-grade discipline** — every phase produces a runbook, ADR, and recovery measurement
- **Modern toolchain** — GitOps, OTLP observability, supply chain security, IaC, AI platform foundations
- **Real troubleshooting experience** — every break-it drill is a future interview story
- **Bare-metal Kubernetes expertise** — kubeadm-installed cluster comparable to on-prem production environments
- **Cloud comparison** — same workload deployed on EKS via Terraform demonstrates managed-vs-unmanaged tradeoffs

---

## 📞 Contact

**Mohiuddin** — DevOps / Platform Engineer
GitHub: [github.com/mohiuddin89](https://github.com/mohiuddin89)

---

## 📄 License

[MIT](./LICENSE) — feel free to fork and adapt for your own learning journey.

---

*This homelab is built one phase at a time, one break-it drill at a time. Every commit reflects something that was deployed, tested, broken, debugged, and documented.*
