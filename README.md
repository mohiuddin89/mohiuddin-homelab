# Production-Grade Kubernetes Homelab

> Bare-metal Kubernetes platform built manually on Proxmox — full GitOps, CI/CD, distributed tracing, and disaster recovery. Every phase ends with a mandatory break-it drill.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.31-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Calico](https://img.shields.io/badge/CNI-Calico-FF6B35)](https://www.tigera.io/project-calico/)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io)
[![GitLab CI](https://img.shields.io/badge/CI%2FCD-GitLab-FC6D26?logo=gitlab&logoColor=white)](https://gitlab.com)
[![Prometheus](https://img.shields.io/badge/Metrics-Prometheus-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io)
[![Jaeger](https://img.shields.io/badge/Tracing-Jaeger-66CFE3)](https://www.jaegertracing.io)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform&logoColor=white)](https://www.terraform.io)

---

## What This Is

A fully manual, production-grade Kubernetes platform on bare-metal Proxmox infrastructure. Every component installed by hand — kubeadm control plane, Calico CNI, Longhorn storage, GitLab CI, ArgoCD, full observability stack — to build deep understanding of how production systems actually work.

**The standard:** Every phase is not done until it is broken, debugged, recovered, and documented.

📐 Full architecture, ADRs, and design decisions: [ARCHITECTURE.md](./ARCHITECTURE.md)

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Proxmox VE — i9-10850K, 64GB DDR4 — Accessed via Tailscale      │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Cluster (kubeadm v1.31.x)                     │   │
│  │                                                           │   │
│  │  md-cp-1 (4GB)    md-w-1 (6GB)     md-w-2 (6GB)          │   │
│  │  ┌────────────┐   ┌────────────┐   ┌────────────┐        │   │
│  │  │ API Server │   │  App Pods  │   │  App Pods  │        │   │
│  │  │ etcd       │   │            │   │  + Ingress │        │   │
│  │  │ ArgoCD     │   │            │   │            │        │   │
│  │  └────────────┘   └────────────┘   └────────────┘        │   │
│  │                                                           │   │
│  │  CNI: Calico  |  LB: MetalLB  |  Storage: Longhorn       │   │
│  │  TLS: cert-manager + Internal Root CA                     │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  md-ci-1 (8GB): GitLab CE + Runner + Registry                    │
│  md-svc-1 (2GB): Internal DNS + Root CA                          │
└──────────────────────────────────────────────────────────────────┘
```

### Request Flow

```
User → MetalLB (10.20.0.200+) → ingress-nginx (HTTPS) → Service → Pod
```

### CI/CD + GitOps Flow

```
Developer
   │ git push
   ▼
GitLab (md-ci-1)
   │
   ├── Kaniko build (privilegeless)
   ├── Trivy scan (HIGH/CRITICAL = fail)
   └── Push image → GitLab Registry
   │
   ▼
Values repo (image tag update via CI)
   │
   ▼
ArgoCD detects change
   ├── dev  → auto-sync
   └── prod → manual gate
```

### Observability Flow

```
Application Pods
      │
      ▼
OTel Collector (gateway mode)
      ├── Metrics → Prometheus (:8889)
      ├── Logs    → Loki (:3100)
      └── Traces  → Jaeger (:14250)
                │
                ▼
           Grafana (trace ↔ logs ↔ metrics correlation)
```

---

## Build Status

| Phase | Focus | Status |
|---|---|---|
| **0** | Proxmox + Networking + Tailscale | ✅ Complete |
| **1** | Kubernetes (kubeadm) + containerd | ✅ Complete |
| **2.5** | etcd backup + verified restore drill | ✅ Complete |
| **3** | MetalLB + ingress-nginx + Longhorn | ✅ Complete |
| **4** | cert-manager + NetworkPolicy + PSS + RBAC | ✅ Complete |
| **5** | GitLab CI + Kaniko + Trivy + Custom Dockerfiles (OTel services) | 🔄 In Progress |
| **6** | Two-repo GitOps + ArgoCD | ⏳ Upcoming |
| **7** | External Secrets Operator | ⏳ Upcoming |
| **8** | Prometheus + Alertmanager + Grafana | ⏳ Upcoming |
| **9** | Loki + Promtail | ⏳ Upcoming |
| **10** | Jaeger | ⏳ Upcoming |
| **11** | OTel Collector gateway | ⏳ Upcoming |
| **12a** | OTel Demo core services (8 services) | ⏳ Upcoming |
| **12b** | OTel Demo extended (all 22 services) | ⏳ Upcoming |
| **13** | Backup + restore drill | ⏳ Upcoming |
| **14–16** | Terraform + AWS + EKS | ⏳ Upcoming |
| **17–22** | AI Platform (LiteLLM, Langfuse, RAG, Eval) | ⏳ Upcoming |

---

## Tech Stack

**Cluster Core**
kubeadm v1.31 · containerd · Calico (CNI + NetworkPolicy) · MetalLB (L2 mode) · ingress-nginx · Longhorn (2-replica) · cert-manager · metrics-server

**Security**
NetworkPolicy default-deny per namespace · Pod Security Standards (restricted/baseline/privileged) · External Secrets Operator · RBAC least-privilege · Internal Root CA

**CI/CD + GitOps**
GitLab CE · GitLab Runner · Kaniko (privilegeless builds) · Trivy (HIGH/CRITICAL gate) · ArgoCD · Two-repo GitOps pattern

**Observability**
Prometheus + Alertmanager · Grafana · Loki + Promtail · Jaeger · OTel Collector (gateway mode)

**Backup + Recovery**
etcd snapshots (6h cron) · Longhorn recurring snapshots · GitLab → GitHub mirror (offsite)

**Workload**
OpenTelemetry Demo App — 22 microservices: Go, Java, Python, .NET, Node.js, Ruby, Rust, C++, Elixir, Kotlin, PHP

**Cloud + IaC (Phase 14–16)**
Terraform · AWS VPC · Route53 · SSM Parameter Store · S3 · IAM · EKS · ALB Controller

---

## Production-Grade Practices

| Practice | Implementation |
|---|---|
| Manual-first learning | Every component installed by hand before any IaC abstraction |
| Defense in depth | NetworkPolicy + PSS + RBAC layered per namespace |
| GitOps purity | ArgoCD is the only entity that mutates cluster state in dev/prod |
| Disaster recovery culture | Every phase has break-it drills with measured RTO and committed runbooks |
| Manual before managed | kubeadm by hand first, then EKS via Terraform — understand before abstracting |

---

## Break-it Discipline

A phase is not complete until:

> **"I broke it, debugged it, fixed it, measured recovery time, and committed a runbook."**

| Phase | Drill | What It Proves |
|---|---|---|
| etcd backup | Delete namespace → restore from snapshot | Full cluster recovery + RTO |
| Security | Too-strict NetworkPolicy → debug | Traffic flow analysis |
| Storage | Force-delete Longhorn replica | Auto-rebuild monitoring |
| CI/CD | Push HIGH CVE image → pipeline blocks | Trivy gate enforcement |
| GitOps | `kubectl edit` while ArgoCD watches | Drift detection + auto-revert |
| Observability | OTel Collector OOM | memory_limiter behavior |
| Workload | Kill paymentservice mid-trace | Cascading failure in Jaeger |
| Backup | Delete namespace → restore | Full namespace RTO |

All drill outcomes: [`docs/runbooks/failures/`](./docs/runbooks/failures/)

---

## Repository Structure

```
mohiuddin-homelab/
├── README.md                    ← This file
├── ARCHITECTURE.md              ← Full design + ADRs + tradeoffs
├── docs/
│   ├── runbooks/                ← Per-phase operational runbooks
│   │   ├── phase-0.md
│   │   ├── phase-1.md
│   │   ├── phase-2.md
│   │   ├── phase-3.md
│   │   ├── phase-4.md
│   │   ├── phase-4.5.md
│   │   ├── phase-5-gitlab-ci.md
│   │   └── failures/            ← Break-it drill outcomes + RTO
│   └── decisions/               ← ADRs 001–013
├── manifests/                   ← Raw Kubernetes YAMLs
├── helm-values/                 ← Per-chart Helm values.yaml
├── dockerfiles/                 ← Per-service custom Dockerfiles
└── apps-values/                 ← GitOps source (ArgoCD watches)
```

---

## Key Architecture Decisions

13 ADRs document every significant technical choice:

| ADR | Decision | Reason |
|---|---|---|
| 001 | Calico over Flannel | NetworkPolicy enforcement required |
| 002 | Longhorn over local-path | Replication + snapshots |
| 003 | GitLab over Gitea | Full CI/CD parity |
| 004 | Jaeger over Tempo | Industry-standard, widely recognized |
| 005 | External Secrets Operator | Multi-backend: local → AWS SSM |
| 006 | kubeadm before EKS | Understand control plane before managed abstraction |
| 008 | Two-repo GitOps | Clean code/deploy separation |
| 011 | Kaniko over Docker-in-Docker | Privilegeless builds |

Full ADR catalog: [`docs/decisions/`](./docs/decisions/)

---

## Documented Tradeoffs

| Constraint | Reason | Production Equivalent |
|---|---|---|
| Single control plane | Lab resource budget | 3-node etcd + 2 control planes |
| 2-replica Longhorn | Worker count | 3-replica with 3+ workers |
| Self-signed Root CA | No public domain | Let's Encrypt + real domain |
| Ingress co-located on md-w-2 | Resource budget | Dedicated ingress node pool |
| In-cluster observability | Lab simplicity | External Prometheus for cluster health |

---

## Contact

**Mohiuddin** — DevOps / Platform Engineer
GitHub: [github.com/mohiuddin89](https://github.com/mohiuddin89)

---

*Built one phase at a time. Every component deployed manually, broken intentionally, recovered deliberately, and documented completely.*
