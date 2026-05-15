# Production-Grade Kubernetes Homelab

> Bare-metal Kubernetes platform built manually on Proxmox — full GitOps, supply-chain-secured CI/CD, distributed tracing, and disaster recovery. Every phase ends with a mandatory break-it drill.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.31-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io)
[![GitLab CI](https://img.shields.io/badge/CI%2FCD-GitLab-FC6D26?logo=gitlab&logoColor=white)](https://gitlab.com)
[![Prometheus](https://img.shields.io/badge/Metrics-Prometheus-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io)
[![Cosign](https://img.shields.io/badge/Signing-Cosign-2F88FF)](https://www.sigstore.dev)
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
│  │  Kubernetes Cluster (kubeadm v1.31.14)                    │   │
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
git push → GitLab CI → Kaniko build → Trivy scan → Cosign sign
                                                         │
                                               GitLab Registry
                                                         │
                                          CI updates apps-values repo
                                                         │
                                    ArgoCD verifies signature → syncs
                                          ├── dev:  auto-sync
                                          └── prod: manual gate
```

### Observability Flow (OTLP Everywhere)
```
Application Pods
      │  OTLP gRPC :4317
      ▼
OTel Collector (gateway mode)
      ├── Metrics → Prometheus
      ├── Logs    → Loki
      └── Traces  → Jaeger
                │
                ▼
           Grafana (trace ↔ logs ↔ metrics correlation)
```

---

## Build Status

| Phase | Focus | Status |
|---|---|---|
| **0** | Proxmox + Networking + Tailscale subnet router | ✅ Complete |
| **1** | Kubernetes cluster (kubeadm) + containerd | ✅ Complete |
| **1.5** | etcd backup + verified restore drill | ✅ Complete |
| **2** | MetalLB + ingress-nginx + Longhorn storage | ✅ Complete |
| **3** | ArgoCD GitOps engine | ✅ Complete |
| **4** | Security: NetworkPolicy + PSS + RBAC + Sealed Secrets | ✅ Complete |
| **4.5** | Calico CNI + cert-manager + Internal TLS | ✅ Complete |
| **5** | GitLab CI: 7-stage pipeline + Kaniko + Trivy + Cosign | 🔄 In Progress |
| **6** | Full GitOps: two-repo + ArgoCD signature verification | ⏳ Upcoming |
| **7–10** | Prometheus + Alertmanager + Grafana + Loki + Jaeger + OTel Collector | ⏳ Upcoming |
| **11** | OpenTelemetry Demo: 22 microservices full deploy + HPA | ⏳ Upcoming |
| **12** | Velero backup + namespace-level disaster recovery drill | ⏳ Upcoming |
| **13–15** | Terraform + AWS VPC + EKS migration + comparison | ⏳ Upcoming |

---

## Tech Stack

**Cluster Core**
kubeadm v1.31 · containerd · Calico (CNI + NetworkPolicy) · MetalLB (L2 mode) · ingress-nginx · Longhorn (replicated storage) · cert-manager · metrics-server

**Security**
NetworkPolicy default-deny per namespace · Pod Security Standards (restricted/baseline/privileged) · LimitRange per namespace · PodDisruptionBudget · Sealed Secrets · RBAC least-privilege · Internal Root CA · seccompProfile RuntimeDefault

**CI/CD + GitOps**
GitLab CE · GitLab Runner · Kaniko (privilegeless builds) · Trivy (HIGH/CRITICAL gate) · Cosign (keyless OIDC signing) · ArgoCD (signature verification before deploy) · Two-repo GitOps pattern · Semantic versioning + SHA tags

**Observability**
Prometheus + Alertmanager (Slack routing) · Grafana · Loki + Promtail · Jaeger (distributed tracing) · OpenTelemetry Collector (gateway mode) · DORA metrics dashboard

**Backup + Recovery**
etcd snapshots (6h cron) · Velero (namespace-level PVC backup + restore) · Longhorn recurring snapshots · GitLab to GitHub mirror (offsite code backup)

**Workload**
OpenTelemetry Demo App — 22 microservices: Go, Java, Python, .NET, Node.js, Ruby, Rust, C++, Elixir, Kotlin, PHP

**Cloud + IaC (Phase 13–15)**
Terraform · AWS VPC (3-AZ) · Route53 · SSM Parameter Store · S3 · IAM · EKS · ALB Controller

---

## Production-Grade Practices

| Practice | Implementation |
|---|---|
| Supply chain security | Every image signed with Cosign — ArgoCD refuses unsigned deployments |
| Defense in depth | NetworkPolicy + PSS restricted + LimitRange + PDB + seccompProfile layered |
| OTLP-everywhere observability | Vendor-neutral telemetry pipeline — trace, logs, metrics correlated in Grafana |
| GitOps purity | ArgoCD is the only entity that mutates cluster state in dev/staging/prod |
| Disaster recovery culture | Every phase has break-it drills with measured RTO and committed runbooks |
| Manual-first, IaC-when-justified | kubeadm by hand first, then EKS via Terraform — understand before abstracting |
| DORA metrics | Deployment frequency, lead time, change failure rate, MTTR tracked in Grafana |
| Cost discipline | EKS phase intentionally short — terraform destroy same day after comparison |

---

## Break-it Discipline

A phase is not complete until:

> **"I broke it, debugged it, fixed it, measured recovery time, and committed a runbook."**

| Phase | Drill | What It Proves |
|---|---|---|
| etcd backup | Delete namespace → restore from snapshot | Full cluster recovery capability + RTO |
| Security | Too-strict NetworkPolicy → debug with Calico | Traffic flow analysis muscle |
| Storage | Force-delete Longhorn replica | Auto-rebuild, replica health monitoring |
| CI/CD | Push HIGH CVE image → pipeline blocks | Trivy gate enforcement |
| GitOps | `kubectl edit` while ArgoCD watches | Drift detection + auto-revert (selfHeal) |
| Observability | OTel Collector OOM kill | memory_limiter behavior, data loss window |
| Workload | Kill paymentservice mid-trace | Cascading failure analysis in Jaeger |
| Backup | Delete entire namespace → Velero restore | Namespace-level RTO measurement |

All drill outcomes: [`docs/runbooks/`](./docs/runbooks/)

---

## Repository Structure

```
mohiuddin-homelab/
├── README.md                    ← This file
├── ARCHITECTURE.md              ← Full design, ADRs, tradeoffs
├── docs/
│   ├── runbooks/                ← Per-phase operational runbooks
│   │   ├── phase-0.md           ← VM baseline + Proxmox networking
│   │   ├── phase-1.md           ← kubeadm Kubernetes cluster
│   │   ├── phase-2.md           ← MetalLB + Longhorn + Ingress
│   │   ├── phase-3.md           ← ArgoCD GitOps
│   │   ├── phase-4.md           ← Security baseline
│   │   ├── phase-4.5.md         ← Calico + cert-manager + TLS
│   │   └── phase-5-gitlab-ci.md ← GitLab CI 7-stage pipeline
│   ├── decisions/               ← Architecture Decision Records (ADR 001–016)
│   └── break-it-drills/         ← Drill outcomes + RTO measurements
├── manifests/                   ← Raw Kubernetes YAMLs
├── helm-values/                 ← Per-chart Helm values
├── apps-values/                 ← GitOps source (ArgoCD watches this)
└── terraform/                   ← Phase 13+ AWS infrastructure
```

---

## Key Architecture Decisions

16 ADRs document every significant technical choice. Selected highlights:

| ADR | Decision | Rejected | Reason |
|---|---|---|---|
| 001 | Calico | Flannel | NetworkPolicy enforcement required from Phase 4 |
| 002 | Longhorn | local-path | Replication + snapshots + Velero CSI integration |
| 003 | GitLab CE | Gitea | Full CI/CD parity: runner, registry, SAST, MR gates |
| 006 | kubeadm first | EKS first | Understand control plane internals before managed abstraction |
| 008 | Two-repo GitOps | Single repo | Clean separation: code changes vs deployment changes |
| 011 | Kaniko | Docker-in-Docker | Privilegeless builds — no root required in CI runner |
| 015 | Cosign signing | No signing | Supply chain security — unsigned images never reach cluster |

Full ADR catalog: [`docs/decisions/`](./docs/decisions/)

---

## Documented Tradeoffs

Every constraint is intentional and has a production-equivalent documented:

| Constraint | Reason | Production Equivalent |
|---|---|---|
| Single control plane | Lab resource budget | 3-node etcd + 2 control planes behind load balancer |
| 2-replica Longhorn | Worker node count | 3-replica with 3+ worker nodes |
| Self-signed Root CA | No public domain | Let's Encrypt + real domain |
| Ingress co-located on worker | Resource budget | Dedicated ingress node pool (shown on EKS in Phase 15) |
| In-cluster observability | Lab simplicity | External Prometheus for cluster-level heartbeat |
| GitLab single instance | Resource budget | GitLab HA or GitLab.com |

---

## Contact

**Mohiuddin** — DevOps / Platform Engineer
GitHub: [github.com/mohiuddin89](https://github.com/mohiuddin89)

---

*Built one phase at a time. Every component deployed manually, broken intentionally, recovered deliberately, and documented completely.*
