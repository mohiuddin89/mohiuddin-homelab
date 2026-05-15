# Mohiuddin's DevOps & Platform Engineering Homelab

> **Version:** v3.0
> **Status:** Phase 5 in progress
> **Goal:** Production-grade DevOps platform — GitOps, observability, CI/CD, IaC
> **Hardware:** Proxmox VE on i9-10850K, 64GB DDR4 — accessed via Tailscale

---

## Project Goals

1. **Hands-on production learning** — every tool installed manually, every concept understood deeply
2. **Production debugging muscle** — every phase ends with mandatory break-it drills
3. **Portfolio showcase** — runbooks, ADRs, architecture diagrams on GitHub
4. **Interview confidence** — real troubleshooting stories from actual failures

---

## Execution Philosophy — Manual First, Then IaC

**Core rule:**
> Homelab execution remains manual for learning clarity.
> Terraform becomes mandatory for AWS phases.
> Terraform learning may begin earlier but no Terraform is used to provision homelab infrastructure.
> AWS provisioning is Terraform-first after bootstrap exceptions.

### Why this ordering

- **Understand concrete before abstraction** — must know what Terraform automates by doing it manually first
- **Debugging muscle** — manual setup forces OS/networking root-cause work
- **Failure modes exposed** — kubeadm certs, kubelet, containerd touched directly
- **Interview signal** — "bare-metal K8s manually, then migrated to EKS via Terraform" >> "ran terraform apply"

### Manual = Required

| Layer | Why Manual |
|---|---|
| Proxmox + VMs | Hypervisor layer, click-driven for learning |
| Kubernetes (kubeadm) | Every control plane component visible |
| Add-ons (MetalLB, ingress, Longhorn) | Helm charts installed manually |
| CI/CD (GitLab, ArgoCD) | Web UI + config files |
| Observability stack | Helm install + manual configuration |
| OTel Demo workload | Custom Dockerfiles + manifest-based deploy |

### IaC = Required

| Layer | Tool |
|---|---|
| AWS resources (VPC, S3, Route53, SSM, IAM) | Terraform |
| EKS cluster + node groups | Terraform (terraform-aws-modules/eks) |
| AI Platform cloud resources | Terraform |

---

## Core Principles

- **Design before install** — architecture first, tool later
- **Manual first, IaC when justified** — Phase 14+ for provisioning
- **Verify every step** — no pass without validation
- **Break → debug → document** — every failure produces a runbook
- **Every component must answer:** Purpose, Traffic flow, Failure story, Recovery plan

---

## Infrastructure

### VM Layout

| VM | IP | Role | Spec |
|---|---|---|---|
| md-svc-1 | 10.20.0.50 | Root CA + Internal DNS | 2C / 2GB |
| md-cp-1 | 10.20.0.10 | K8s Control Plane | 2C / 4GB |
| md-w-1 | 10.20.0.21 | App workloads | 2C / 6GB |
| md-w-2 | 10.20.0.22 | App workloads + Ingress (co-located) | 2C / 6GB |
| md-ci-1 | 10.20.0.30 | GitLab CE + Runner + Registry | 2C / 8GB |

**Host:** Proxmox VE, i9-10850K, 64GB DDR4. Tailscale subnet router for remote access.

### Network Layout

| Network | CIDR | Purpose |
|---|---|---|
| Management | 192.168.68.0/24 | Proxmox UI, internet access |
| Lab (isolated) | 10.20.0.0/24 | All VMs, Kubernetes |
| Pod CIDR | 10.244.0.0/16 | Kubernetes pods (Calico) |
| Service CIDR | 10.96.0.0/12 | Kubernetes services |
| MetalLB pool | 10.20.0.200–210 | LoadBalancer IPs |

### Proxmox Bridges

- `vmbr0` — management network (physical NIC)
- `vmbr1` — isolated lab bridge (no physical NIC, VM-to-VM only)

### Critical Proxmox Config

```bash
# /etc/sysctl.d/99-ipforward.conf
net.ipv4.ip_forward=1

# nftables: FORWARD rules between vmbr0 ↔ vmbr1
# MASQUERADE on vmbr0 so lab VMs reach internet
```

---

## Remote Access — Tailscale

- pve → Tailscale subnet router
- Advertised routes: `192.168.68.0/24`, `10.20.0.0/24`
- Access from anywhere via Tailscale client

**Tradeoff documented:** Single subnet router. pve down = remote access lost. Acceptable at lab scale.

---

## Kubernetes Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Proxmox VE — i9-10850K, 64GB DDR4 — Tailscale subnet router     │
│                                                                  │
│  ┌──────────────┐                                                │
│  │  md-svc-1    │  Root CA (cert-manager source) + DNS           │
│  │  10.20.0.50  │                                                │
│  └──────────────┘                                                │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Cluster (kubeadm v1.31.x)                     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │   │
│  │  │  md-cp-1    │  │   md-w-1    │  │   md-w-2    │        │   │
│  │  │ API Server  │  │  App Pods   │  │  App Pods   │        │   │
│  │  │ etcd        │  │             │  │  + Ingress  │        │   │
│  │  │ ArgoCD      │  │             │  │             │        │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘        │   │
│  │                                                            │   │
│  │  CNI: Calico  |  LB: MetalLB  |  Storage: Longhorn        │   │
│  │  TLS: cert-manager + Root CA on md-svc-1                  │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────┐                                                │
│  │  md-ci-1     │  GitLab CE + GitLab Runner + Container Registry │
│  │  10.20.0.30  │                                                │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
```

### Traffic Flow

```
User
   │
   ▼
MetalLB IP (10.20.0.200+)
   │
   ▼
ingress-nginx (HTTPS, TLS via cert-manager)
   │
   ├── argocd.lab      → ArgoCD
   ├── grafana.lab     → Grafana
   ├── gitlab.lab      → GitLab
   ├── prometheus.lab  → Prometheus
   ├── jaeger.lab      → Jaeger
   └── otel-demo.lab   → OTel Demo Frontend
```

### Namespaces

| Namespace | Sync Policy | Pod Security |
|---|---|---|
| dev | ArgoCD auto-sync | baseline |
| staging | Manual gate | baseline |
| prod | Manual sync only | restricted |
| kube-system | N/A | privileged |
| argocd | N/A | baseline |
| gitlab | N/A | baseline |
| observability | N/A | baseline |
| otel-demo | ArgoCD-managed | restricted |
| longhorn-system | N/A | privileged |
| metallb-system | N/A | baseline |
| ingress-nginx | N/A | baseline |

---

## Full Tool Stack

### Cluster Core

| Tool | Purpose | Install Method |
|---|---|---|
| kubeadm v1.31.x | Cluster bootstrap | Manual |
| containerd | Container runtime (systemd cgroup) | Manual |
| Calico | CNI + NetworkPolicy enforcement | kubectl apply |
| Longhorn | Distributed block storage (2-replica) | Helm |
| MetalLB | Bare-metal LoadBalancer (L2 mode) | Helm |
| ingress-nginx | HTTP/HTTPS routing | Helm |
| cert-manager | TLS automation (internal CA) | Helm |
| metrics-server | Resource metrics for kubectl top + HPA | kubectl apply |

### Security

| Tool | Purpose |
|---|---|
| Calico NetworkPolicy | Default-deny per namespace + explicit allow rules |
| Pod Security Standards | restricted/baseline/privileged per namespace label |
| Sealed Secrets | Encrypted secrets safe to commit to Git |
| RBAC | Least-privilege ServiceAccounts per workload |
| Internal Root CA | Self-signed CA on md-svc-1, cert-manager issues certs |

### CI/CD + GitOps

| Tool | Purpose |
|---|---|
| GitLab CE | Git server + CI/CD + Container Registry |
| GitLab Runner | Pipeline executor |
| Kaniko | Privilegeless in-cluster image builds |
| Trivy | CVE scanning — HIGH/CRITICAL fails pipeline |
| ArgoCD | GitOps engine |

### Observability

| Tool | Purpose |
|---|---|
| Prometheus + Alertmanager | Metrics collection + alerts |
| Grafana | Unified dashboard — metrics, logs, traces |
| Loki + Promtail | Log aggregation (DaemonSet per worker) |
| Jaeger | Distributed tracing |
| OTel Collector | Gateway-mode telemetry pipeline |

### Workload

| Tool | Purpose |
|---|---|
| OpenTelemetry Demo App | 22 real microservices — Go, Java, Python, .NET, Node.js, Ruby, Rust, C++, Elixir, Kotlin, PHP |

---

## CI/CD + GitOps Flow

```
Developer
   │ git push
   ▼
GitLab Repo (md-ci-1)
   │
   ▼
GitLab CI Pipeline
   │
   ├── build image (Kaniko, privilegeless)
   ├── Trivy scan (fail on HIGH/CRITICAL)
   └── push → GitLab Container Registry
   │
   ▼
Values Repo (image tag update via CI)
   │
   ▼
ArgoCD detects change
   │
   ├── dev  → auto-sync
   └── prod → manual gate
   │
   ▼
Pod restarts with new image
```

### Two-Repo GitOps Pattern

- **`mohiuddin-otel-homelab/`** — application code, Dockerfiles, `.gitlab-ci.yml`
- **`apps-values/`** — Kubernetes manifests and Helm values. ArgoCD watches this only.

CI in source repo builds image → updates image tag in apps-values → ArgoCD detects → syncs.

### Key Decisions

- GitLab CI + Runner only (no Jenkins)
- GitLab Container Registry on md-ci-1
- Kaniko for privilegeless image builds
- Trivy scan — fails pipeline on HIGH/CRITICAL
- Two-repo GitOps pattern

---

## Observability Stack

```
Applications (Pods)
   │
   ▼
OTel Collector (Deployment — gateway mode)
   │
   ├── Metrics → Prometheus (pull :8889)
   ├── Logs    → Loki (:3100)
   └── Traces  → Jaeger (:14250)
   │
   ▼
Grafana (unified UI)
   └── Trace → Logs correlation ← THE WOW MOMENT
```

### Push vs Pull

| Signal | Model | Endpoint |
|---|---|---|
| Metrics | Pull — Prometheus scrapes Collector | :8889 |
| Logs | Push — Collector → Loki | :3100 |
| Traces | Push — Collector → Jaeger | :14250 |

> Collector does NOT push to Prometheus. Prometheus pulls from Collector.

### OTel Collector Mode

**Initial (Phase 11):** Single gateway Collector (Deployment). Simpler debugging, single config point.

**Future (optional):** Agent DaemonSet per node → gateway aggregation (two-tier pattern).

---

## Security Architecture

### NetworkPolicy (Phase 4)

```yaml
# Default deny all ingress + egress per namespace

# Explicit allow rules:
- ingress-nginx → app pods
- app pods → database
- app pods → DNS (kube-system, port 53)
- app pods → OTel Collector (telemetry)
- app pods → external HTTPS (Phase 17+ AI API calls)
- monitoring → all namespaces (Prometheus cross-namespace scrape)
```

### Pod Security Standards

| Namespace | Level |
|---|---|
| prod / otel-demo | `restricted` |
| staging / dev / argocd / observability / gitlab / ingress-nginx | `baseline` |
| kube-system / longhorn-system | `privileged` |

### Secrets

**Tool:** External Secrets Operator (ESO)

- Phase 7 backend: local Kubernetes secret store
- Phase 15 backend: AWS SSM Parameter Store
- Rule: No plaintext secrets in Git

### TLS

- cert-manager + internal Root CA on md-svc-1
- HTTPS on all Ingress objects from Phase 4 onward
- Quarterly cert rotation drill

---

## Storage — Longhorn

- Dynamic PVC provisioning
- Replicated across worker nodes
- **Replica count: 2** (not default 3 — resource budget)
- Backup target: external disk (Phase 13, not md-svc-1)

---

## Internal DNS (md-svc-1)

**Tool:** dnsmasq

| Domain | Target |
|---|---|
| grafana.lab | MetalLB IP (ingress) |
| argocd.lab | MetalLB IP (ingress) |
| gitlab.lab | md-ci-1 direct |
| prometheus.lab | MetalLB IP (ingress) |
| jaeger.lab | MetalLB IP (ingress) |
| otel-demo.lab | MetalLB IP (ingress) |

---

## Backup & Recovery

### Core Backup Targets

| Target | Tool | Frequency |
|---|---|---|
| etcd | `etcdctl snapshot save` | Every 6h |
| GitLab | Built-in backup | Daily |
| Longhorn PVCs | Longhorn snapshot | Daily |

### Additional Scope

- **md-svc-1:** DNS config + CA private key → backup off-cluster
- **Git repos:** Mirror to GitHub (`git push --mirror`) — not dependent on md-ci-1
- **Rule:** Backup ≠ useful. Restore drill mandatory in Phase 13.

---

## Break-it Model (Per Phase)

```
☐ Kill a core component pod / service
☐ Measure recovery time
☐ Simulate one realistic failure
☐ Write 1-page runbook: detect → debug → recover
☐ Commit to docs/runbooks/failures/ in Git
```

Phase pass condition: NOT "it worked." It is "I broke it, debugged it, fixed it, and documented it."

### Phase 0 — VM Baseline
| Drill | Procedure | What to Learn |
|---|---|---|
| VM reboot | Reboot all VMs | swap stays off, kernel modules persist, containerd auto-starts |
| SSH lockout | Remove key before setting new one | Proxmox console recovery |
| sysctl misconfig | Set wrong value | How kernel parameters affect networking |

### Phase 1 — kubeadm Cluster
| Drill | Procedure | What to Learn |
|---|---|---|
| API server kill | Move kube-apiserver.yaml | Static pod recovery |
| Worker reboot | Reboot md-w-1 | Pod rescheduling, NotReady detection |
| etcd corruption | Corrupt data dir | Restore from snapshot, RTO |
| CNI delete | Remove Calico | Pod networking failure |

### Phase 2.5 — etcd Backup
| Drill | Procedure | What to Learn |
|---|---|---|
| Restore drill | Delete namespace, restore | Full restore + data loss window |
| Snapshot corruption | Corrupt snapshot file | Integrity verification |
| Cron failure | Stop cron | Monitor backup jobs |

### Phase 3 — Access + Storage
| Drill | Procedure | What to Learn |
|---|---|---|
| ingress pod kill | Delete ingress-nginx pod | Deployment self-healing |
| MetalLB speaker fail | Stop speaker on one node | L2 ARP failover timing |
| PVC orphan | Delete PVC while pod uses it | Volume lifecycle |
| Storage full | Fill PVC to 100% | Pod eviction behavior |

### Phase 4 — Security
| Drill | Procedure | What to Learn |
|---|---|---|
| Wrong NetworkPolicy | Block legitimate traffic | Calico policy trace debug |
| RBAC misuse | Forbidden action with limited SA | API server auth/authz response |
| Sealed Secret tamper | Modify encrypted bytes | Decryption failure logs |
| Lost CA key | Delete sealed-secrets-controller secret | Key backup importance |

### Phase 5 — GitLab CI
| Drill | Procedure | What to Learn |
|---|---|---|
| Runner offline | Stop GitLab Runner | Pipeline queue behavior |
| Build OOM | Memory-intensive build | Pod limits, OOMKilled events |
| Registry full | Fill registry storage | Push failures, garbage collection |
| Bad Dockerfile | Push HIGH CVE image | Trivy gate rejection |

### Phase 6 — GitOps Loop
| Drill | Procedure | What to Learn |
|---|---|---|
| Invalid YAML | Push bad manifest to apps-values | ArgoCD sync failure |
| Image pull fail | Reference non-existent tag | ImagePullBackOff, rollback |
| Manual change | kubectl edit while ArgoCD watches | Drift detection, auto-revert |
| ArgoCD self-destruct | Delete ArgoCD app for ArgoCD | App-of-apps recovery |

### Phase 8–11 — Observability
| Drill | Procedure | What to Learn |
|---|---|---|
| Prometheus disk full | Fill metrics storage | Retention, oldest data drop |
| Grafana DB lost | Delete Grafana PVC | Dashboard recovery from Git |
| Loki crash | Kill Loki pods | Log loss window |
| Collector OOM | Massive trace volume | memory_limiter behavior |

### Phase 12 — OTel Demo
| Drill | Procedure | What to Learn |
|---|---|---|
| Service crash chain | Kill paymentservice | Cascading failures in Jaeger |
| Resource exhaustion | loadgenerator max rate | RAM ceiling, scheduling |
| Network partition | NetworkPolicy breaks one service | Distributed system debug |
| Latency injection | sleep() in one service | Performance debugging in Jaeger |

### Phase 13 — Backup + Recovery
| Drill | Procedure | What to Learn |
|---|---|---|
| Full namespace loss | Delete otel-demo | ArgoCD redeploy from Git |
| Volume snapshot restore | Modify + restore Longhorn snapshot | Snapshot workflow |
| Control plane down | Power off md-cp-1, restore | Full cluster recovery |

### Phase 14–16 — AWS + Terraform
| Drill | Procedure | What to Learn |
|---|---|---|
| Terraform state edit | Manually corrupt tfstate | State lock importance |
| Wrong destroy | Run destroy on wrong resource | prevent_destroy lifecycle |
| EKS node drain | Cordon all nodes | Cluster autoscaler |
| IAM permission removed | Remove role permissions | IRSA auth failure |

---

## Phase Plan

### Part 1 — Core Infrastructure

| Phase | Focus | Status |
|---|---|---|
| **0** | Network + Proxmox + Tailscale | ✅ Done |
| **1** | VM baseline + SSH hardening | ✅ Done |
| **2** | Kubernetes (kubeadm) + Calico CNI | ✅ Done |
| **2.5** | etcd backup + restore drill | ✅ Done |
| **3** | MetalLB + ingress-nginx + Longhorn | ✅ Done |
| **4** | cert-manager + NetworkPolicy + PSS + RBAC | ✅ Done |

### Part 2 — Platform Layer

| Phase | Focus | Status |
|---|---|---|
| **5** | GitLab CE + Runner + Kaniko + Trivy + Custom Dockerfiles (OTel Demo services) | 🔄 In Progress |
| **6** | ArgoCD + two-repo GitOps pattern | ⏳ Upcoming |
| **7** | External Secrets Operator (local backend) | ⏳ Upcoming |

### Part 3 — Observability

| Phase | Focus | Status |
|---|---|---|
| **8** | Prometheus + kube-state-metrics + Alertmanager | ⏳ Upcoming |
| **9** | Loki + Promtail | ⏳ Upcoming |
| **10** | Jaeger | ⏳ Upcoming |
| **11** | OTel Collector (gateway mode) | ⏳ Upcoming |

### Part 4 — Workload + Backup

| Phase | Focus | Status |
|---|---|---|
| **12a** | Core services: frontend, frontendproxy, productcatalog, cart, checkout, payment, shipping, currency | ⏳ Upcoming |
| **12b** | Extended: recommendation, ad, email, accounting, frauddetection, quote, imageprovider, loadgenerator, flagd | ⏳ Upcoming |
| **13** | Full backup + restore drill + backup target off md-svc-1 | ⏳ Upcoming |

### Phase 12 Execution Order

```
Phase 12a:
  12a-docker → Dockerfile rewrite + build + test (core 8 services)
  12a-raw    → Raw manifest deploy — learn K8s primitives
  12a-helm   → Helm migration — learn abstraction value

Phase 12b:
  12b-docker → Dockerfile rewrite + build + test (extended services)
  12b-raw    → Raw manifest deploy
  12b-helm   → Helm migration (full workload)
```

### Part 5 — AWS + Terraform

| Phase | Focus | Status |
|---|---|---|
| **14** | Terraform foundation: S3 backend + DynamoDB + IAM + module structure | ⏳ Upcoming |
| **15** | AWS core: VPC + Route53 + SSM + S3 backup (all via Terraform) | ⏳ Upcoming |
| **16** | EKS: cluster + node groups + ALB + ESO→SSM + OTel Demo redeploy + comparison | ⏳ Upcoming |

### Part 6 — AI Platform

| Phase | Focus | Status |
|---|---|---|
| **17** | LiteLLM gateway (Claude + OpenAI backends) | ⏳ Upcoming |
| **18** | Cost + latency observability (LiteLLM metrics + Langfuse + Grafana) | ⏳ Upcoming |
| **19** | Prompt registry (Git-versioned, ArgoCD sync → K8s ConfigMap) | ⏳ Upcoming |
| **20** | Evaluation harness (Promptfoo/Ragas in GitLab CI) | ⏳ Upcoming |
| **21** | RAG pipeline (pgvector + ingestion + retrieval + eval-validated) | ⏳ Upcoming |
| **22** | Guardrails (prompt injection detection, PII filter, output validation) | ⏳ Upcoming |

---

## Terraform Strategy (Phase 14+)

### What Gets Terraform'd

| Resource | Terraform? |
|---|---|
| Proxmox VMs | ❌ Manual |
| K8s cluster (homelab) | ❌ Manual |
| Helm charts (homelab) | ❌ Manual |
| AWS VPC, subnets, SG | ✅ Yes |
| AWS S3, Route53, SSM | ✅ Yes |
| AWS IAM roles/policies | ✅ Yes |
| EKS cluster + node groups | ✅ Yes |
| EKS add-ons (ALB, ESO) | ✅ Yes |
| App workloads on EKS | ArgoCD (GitOps) |

### Remote State Bootstrap Pattern

```bash
# Step 1 — Bootstrap (local state)
cd bootstrap/
terraform init      # local state
terraform apply     # creates S3 bucket + DynamoDB table

# Step 2 — Main env (S3 backend)
cd envs/lab/
terraform init      # migrates to S3 backend
terraform plan
```

### Repo Structure

```
mohiuddin-homelab-terraform/
├── bootstrap/           ← S3 + DynamoDB (local state, run once)
│   └── main.tf
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── route53/
│   └── secrets/
├── envs/
│   └── lab/
│       ├── backend.tf   ← S3 backend (after bootstrap)
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
└── .gitlab-ci.yml       ← plan on MR, apply on main (manual gate)
```

### Cost Discipline

- EKS ~$73/month → `terraform destroy` same day after Phase 16 complete
- NAT Gateway ~$33/month → single AZ or on-demand for practice

---

## Architecture Decision Records

| ADR | Decision | Reason |
|---|---|---|
| 001 | Calico over Flannel | NetworkPolicy enforcement required |
| 002 | Longhorn over local-path | Replication + snapshots |
| 003 | GitLab over Gitea | Full CI/CD + production parity |
| 004 | Jaeger over Tempo | Industry-standard, widely recognized |
| 005 | External Secrets Operator | Multi-backend: local → AWS SSM without app changes |
| 006 | kubeadm before EKS | Control plane internals understanding before managed abstraction |
| 007 | Single control plane | Lab resource tradeoff; etcd snapshot mitigates |
| 008 | Two-repo GitOps | Clean code/deploy separation; rollback = values revert |
| 009 | Custom Dockerfiles for all OTel Demo services | Build chain ownership + Trivy scan coverage |
| 010 | Raw manifests before Helm migration | K8s primitives understanding before abstraction |
| 011 | Kaniko over Docker-in-Docker | Privilegeless image builds |
| 012 | Gateway-only OTel Collector (initial) | Simpler debugging; agent+gateway deferred |
| 013 | 2-worker cluster, ingress co-located on md-w-2 | Resource budget; production parity deferred to EKS |

---

## Known Tradeoffs

| Tradeoff | Reason | Production Equivalent |
|---|---|---|
| Single control plane | Lab resource budget | 3-node etcd + 2 control planes behind LB |
| 2-replica Longhorn | Worker count | 3-replica with 3+ workers |
| Self-signed Root CA | No public domain | Let's Encrypt with real domain |
| Ingress co-located on md-w-2 | Resource budget | Dedicated ingress node pool |
| In-cluster observability | Lab simplicity | External Prometheus for cluster-level health |
| GitLab single instance | Resource budget | GitLab HA or GitLab.com |
| Tailscale single subnet router | One Proxmox host | Redundant subnet routers |
| md-svc-1 SPOF (DNS + CA) | Lab simplicity | Reduced after Phase 13 (DNS + CA only) |

---

## GitHub Repository Structure

```
mohiuddin-homelab/
├── README.md                    ← Architecture overview + screenshots
├── ARCHITECTURE.md              ← This document
├── docs/
│   ├── runbooks/                ← Per-phase runbooks
│   │   ├── phase-0.md
│   │   ├── phase-1.md
│   │   ├── phase-2.md
│   │   ├── phase-3.md
│   │   ├── phase-4.md
│   │   ├── phase-4.5.md
│   │   ├── phase-5-gitlab-ci.md
│   │   └── failures/            ← Break-it drill outcomes + RTO
│   ├── decisions/               ← ADRs 001–013
│   │   ├── 001-calico-over-flannel.md
│   │   ├── 002-longhorn-over-localpath.md
│   │   ├── 003-gitlab-over-gitea.md
│   │   ├── 004-jaeger-over-tempo.md
│   │   ├── 005-external-secrets-operator.md
│   │   ├── 006-kubeadm-before-eks.md
│   │   ├── 007-single-control-plane.md
│   │   ├── 008-two-repo-gitops.md
│   │   ├── 009-custom-dockerfiles.md
│   │   ├── 010-raw-manifests-before-helm.md
│   │   ├── 011-kaniko-over-dind.md
│   │   ├── 012-gateway-otel-collector.md
│   │   └── 013-two-worker-cluster.md
│   └── interview-prep/          ← Q&A per phase (local study only)
├── manifests/                   ← Raw Kubernetes YAMLs
├── helm-values/                 ← Per-chart Helm values.yaml
├── dockerfiles/                 ← Per-service custom Dockerfiles
├── apps-values/                 ← GitOps source (ArgoCD watches)
│   ├── otel-demo/
│   ├── observability/
│   └── platform/
└── terraform/                   ← Phase 14+
    ├── bootstrap/
    ├── modules/
    └── envs/lab/
```

---

## Phase Pass Criteria

A phase is NOT complete until ALL of these are done:

- Tool deployed and working
- Verification commands run successfully
- Break-it drill executed and recovery time measured
- Runbook written in professional English — pushed to GitHub (`docs/runbooks/`)
- Drill outcome documented — pushed to GitHub (`docs/runbooks/failures/`)
- ADR written if architectural decision made — pushed to GitHub (`docs/decisions/`)
- Screenshot taken for portfolio where applicable

**No shortcuts. No phase skipping. Every drill mandatory.**

---

## Execution Rules

1. One phase complete before next begins
2. Phase 0–13: Homelab execution = manual
3. Phase 14+: AWS provisioning = Terraform only
4. Every phase ends with break-it checklist
5. Every failure → runbook in Git
6. Overbuilding = enemy
7. Phase skip = plan broken

---

## Timeline

| Part | Phase Range | Focus | Duration |
|---|---|---|---|
| Part 1 | Phase 0–4 | Core Infrastructure | ~2 months |
| Part 2 | Phase 5–7 | Platform Layer | ~1.5 months |
| Part 3 | Phase 8–11 | Observability | ~1.5 months |
| Part 4 | Phase 12–13 | Workload + Backup | ~2–3 months |
| Part 5 | Phase 14–16 | AWS + Terraform + EKS | ~1.5–2 months |
| Part 6 | Phase 17–22 | AI Platform | ~3–4 months |
| **Total** | | | **~12–15 months** |

---

*Production-grade homelab — debugging muscle built drill by drill.*
*Built by Mohiuddin — DevOps / Platform Engineer candidate.*
