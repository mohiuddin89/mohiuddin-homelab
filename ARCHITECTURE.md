# Mohiuddin's DevOps & Platform Engineering Homelab

> **Version:** v3.0
> **Status:** Phase 5 in progress — 7-stage CI pipeline with real OTel services
> **Goal:** Production-grade DevOps platform — GitOps, observability, supply chain security, IaC
> **Hardware:** Proxmox VE on i9-10850K, 64GB DDR4 — accessed via Tailscale

---

## Project Goals

1. **Hands-on production learning** — every tool installed manually, every concept understood deeply before automation
2. **Production debugging muscle** — every phase ends with mandatory break-it drills and documented RTO
3. **Portfolio showcase** — runbooks, ADRs, architecture diagrams, and drill outcomes on GitHub
4. **Interview confidence** — real troubleshooting stories from actual failures

---

## Core Principles

- **Manual first, IaC when justified** — homelab is all manual; AWS phases use Terraform
- **OTLP everywhere** — all telemetry uses OTLP protocol, no legacy formats
- **Signed-only deploys** — every image signed with Cosign; ArgoCD verifies before sync
- **Least privilege by default** — every SA, Role, and NetworkPolicy starts from deny
- **Break → debug → document** — every failure produces a runbook entry
- **No shortcuts** — phase pass criteria enforced strictly before moving forward

---

## Infrastructure

### VM Layout

| VM | IP | Role | RAM | CPU | Disk |
|---|---|---|---|---|---|
| md-svc-1 | 10.20.0.50 | Root CA + Internal DNS | 2 GB | 2 | 30 GB |
| md-cp-1 | 10.20.0.10 | K8s Control Plane | 4 GB | 2 | 40 GB |
| md-w-1 | 10.20.0.21 | K8s Worker 1 | 6 GB | 2 | 40 GB |
| md-w-2 | 10.20.0.22 | K8s Worker 2 (+ Ingress) | 6 GB | 2 | 40 GB |
| md-ci-1 | 10.20.0.30 | GitLab CE + Runner + Registry | 8 GB | 2 | 40 GB |

**Host:** Proxmox VE, i9-10850K, 64GB DDR4. Tailscale subnet router for remote access.

### Network Layout

| Network | CIDR | Purpose |
|---|---|---|
| Lab network | 10.20.0.0/24 | VM-to-VM communication |
| Pod CIDR | 10.244.0.0/16 | Kubernetes pods (Calico) |
| Service CIDR | 10.96.0.0/12 | Kubernetes services |
| MetalLB pool | 10.20.0.200–210 | LoadBalancer IPs |

### Proxmox Bridges

- `vmbr0` — management network (physical NIC, internet access)
- `vmbr1` — isolated lab bridge (no physical NIC, VM-to-VM only)

---

## Kubernetes Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Proxmox VE — i9-10850K, 64GB DDR4 — Tailscale subnet router     │
│                                                                  │
│  ┌──────────────┐                                                │
│  │  md-svc-1    │  Root CA (cert-manager ClusterIssuer source)   │
│  │  10.20.0.50  │  Internal DNS (dnsmasq)                        │
│  └──────────────┘                                                │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Cluster (kubeadm v1.31.14)                    │   │
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
│  │  10.20.0.30  │  Cosign signing (keyless OIDC)                 │
│  └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
```

### Traffic Flow

```
User (Browser)
    │
    ▼
MetalLB (10.20.0.200+) — L2 mode, ARP-based
    │
    ▼
ingress-nginx (HTTPS, TLS via cert-manager)
    │
    ├── argocd.lab      → ArgoCD UI
    ├── grafana.lab     → Grafana
    ├── gitlab.lab      → GitLab
    ├── jaeger.lab      → Jaeger UI
    ├── otel-demo.lab   → OTel Demo Frontend
    └── prometheus.lab  → Prometheus
```

---

## Full Tool Stack

### Cluster Core

| Tool | Purpose | Install Method |
|---|---|---|
| kubeadm v1.31.14 | Cluster bootstrap | Manual |
| containerd | Container runtime (systemd cgroup) | Manual |
| Calico | CNI + NetworkPolicy enforcement | kubectl apply |
| Longhorn | Distributed block storage (2-replica) | Helm |
| MetalLB | Bare-metal LoadBalancer (L2 mode) | Helm |
| ingress-nginx | HTTP/HTTPS routing | Helm |
| cert-manager | TLS automation (internal CA) | Helm |
| metrics-server | Resource metrics for HPA + kubectl top | kubectl apply |

### Security

| Tool | Purpose |
|---|---|
| Calico NetworkPolicy | Default-deny per namespace + explicit allow rules |
| Pod Security Standards | restricted/baseline/privileged per namespace label |
| LimitRange | Default + max resource limits per namespace |
| PodDisruptionBudget | Voluntary disruption protection (ingress, ArgoCD) |
| External Secrets Operator | Secrets injection from backend (local → AWS SSM) |
| RBAC | Least-privilege ServiceAccounts per workload |
| Internal Root CA | Self-signed CA on md-svc-1, cert-manager issues certs |
| seccompProfile RuntimeDefault | Syscall filtering on restricted namespaces |

### CI/CD + GitOps

| Tool | Purpose |
|---|---|
| GitLab CE | Git server + CI/CD + Container Registry |
| GitLab Runner | Pipeline executor (Kubernetes executor on md-ci-1) |
| Kaniko | Privilegeless in-cluster image builds (ADR 011) |
| Trivy | CVE scanning — HIGH/CRITICAL fails pipeline |
| Cosign | Image signing (supply chain security) |
| ArgoCD | GitOps engine — signature verification before deploy |

### Observability

| Tool | Purpose |
|---|---|
| Prometheus + Alertmanager | Metrics collection + Slack routing |
| Grafana | Unified dashboard — metrics, logs, traces |
| Loki + Promtail | Log aggregation (DaemonSet per worker) |
| Jaeger | Distributed tracing (industry-standard) |
| OTel Collector | Gateway-mode telemetry pipeline (OTLP in, multi-exporter out) |

### Workload

| Tool | Purpose |
|---|---|
| OpenTelemetry Demo App | 22 real microservices — Go, Java, Python, .NET, Node.js, Ruby, Rust, C++, Elixir, Kotlin, PHP |

---

## CI/CD Pipeline — 7 Stages

Phase 5 builds a production CI pipeline for real OpenTelemetry Demo services.

```
git push (to mohiuddin-otel-homelab)
    │
    ▼
STAGE 1 — lint
    ├── hadolint (Dockerfile best practices)
    └── language linter (golangci-lint / eslint / ruff per service)

STAGE 2 — build
    └── Kaniko (privilegeless, no Docker socket)
        Tags: sha-<commit> + v1.2.3 if git tag present

STAGE 3 — scan
    └── Trivy — fail pipeline on HIGH or CRITICAL CVE
        Output: SARIF report to GitLab Security tab

STAGE 4 — size-check
    └── Image size limit enforced (Go services ≤ 50MB, frontend ≤ 200MB)

STAGE 5 — health-test
    └── docker run image + curl health endpoint
        Proves image actually starts — not just builds

STAGE 6 — sign
    └── Cosign keyless OIDC (GitLab CI job identity)
        Signature stored in registry alongside image

STAGE 7 — push
    └── Signed image pushed to GitLab Container Registry
        ArgoCD verifies signature before deploying
```

### Two-Repo GitOps Pattern

- **`mohiuddin-otel-homelab/`** — application code, Dockerfiles, `.gitlab-ci.yml`
- **`apps-values/`** — Kubernetes manifests and Helm values. ArgoCD watches this only.

CI in source repo builds image → updates image tag in apps-values → ArgoCD detects → verifies signature → syncs.

Rollback = revert one commit in apps-values. ArgoCD re-syncs automatically.

### Image Tagging Strategy

| Trigger | Tag | Use |
|---|---|---|
| Every commit to `main` | `sha-<short-commit>` | Dev namespace auto-deploy |
| Git tag `v1.2.3` | `v1.2.3` + `latest` | Staging / production deploy |
| Feature branch | `branch-<name>-sha-<commit>` | PR testing only |

---

## GitOps — ArgoCD Design

### Sync Policies

| Namespace | Auto-Sync | Self-Heal | Prune |
|---|---|---|---|
| dev | Yes | Yes | Yes |
| staging | No | No | No |
| prod | No | No | No |

`selfHeal: true` means ArgoCD reverts manual `kubectl` changes automatically. Enabled only in dev.

### App-of-Apps Pattern

```
root-app (ArgoCD Application)
    ├── otel-demo-app      → apps-values/otel-demo/
    ├── observability-app  → apps-values/observability/
    └── platform-app       → apps-values/platform/
```

---

## Observability Architecture

### OTLP Everywhere

All telemetry uses OTLP protocol. No legacy Jaeger Thrift (`:14250`), no Zipkin.

- Apps → Collector: OTLP gRPC `:4317`
- Collector → Jaeger: OTLP gRPC :4317
- Collector → Loki: Loki push API `:3100`
- Prometheus → Collector: pull scrape `:8889`

```
Application Pods (OTLP gRPC :4317)
    │
    ▼
OTel Collector (gateway Deployment, single replica)
    │
    ├── Metrics ──→ Prometheus (Prometheus pulls :8889)
    ├── Logs    ──→ Loki (Collector pushes :3100)
    └── Traces  ──→ Jaeger (Collector pushes OTLP :4317)
    │
    ▼
Grafana (unified UI — THE WOW MOMENT: trace → logs → metrics)
```

### Alertmanager Routing

```yaml
# Slack routing — Webhook stored in Sealed Secret, not plaintext
route:
  receiver: 'slack-homelab'
  group_by: ['alertname', 'namespace']
```

Mandatory alerts: KubePodCrashLooping, KubeNodeNotReady, PrometheusTargetMissing,
LonghornVolumeRobustnessStatus, etcdHighCommitDurations, OTel Collector memory >80%.

---

## Security Architecture

### NetworkPolicy (Default Deny)

```
Per namespace: deny all ingress + egress (default)

Explicit allows:
- ingress-nginx  → app pods (HTTP/HTTPS)
- app pods       → kube-system DNS (port 53)
- app pods       → OTel Collector (:4317, :4318)
- monitoring     → all namespaces (Prometheus scrape)
- argocd         → all namespaces (kubectl API)
```

### Pod Security Standards

| Namespace | Level | Reason |
|---|---|---|
| otel-demo, prod | `restricted` | No root, drop all caps, seccompProfile |
| argocd, observability, gitlab, ingress-nginx | `baseline` | No privileged containers |
| kube-system, longhorn-system | `privileged` | System-level access required |

### Secrets

All secrets managed via External Secrets Operator (ESO). Local Kubernetes backend for homelab; AWS SSM backend for Phase 15. No plaintext secrets in any repo.

---

## Namespace Plan

| Namespace | Contents | PSS Level |
|---|---|---|
| kube-system | Core K8s, Longhorn, cert-manager | privileged |
| metallb-system | MetalLB controller + speaker | baseline |
| ingress-nginx | ingress controller | baseline |
| argocd | ArgoCD | baseline |
| gitlab | GitLab CE + Runner | baseline |
| observability | Prometheus, Grafana, Loki, Jaeger, OTel | baseline |
| otel-demo | 22 OTel Demo microservices | restricted |
| longhorn-system | Longhorn storage system | privileged |

---

## Phase Plan

### Part 1 — Core Infrastructure

| Phase | Focus | Status |
|---|---|---|
| **0** | Proxmox + VM baseline + Tailscale + SSH hardening | ✅ Complete |
| **1** | kubeadm Kubernetes cluster + containerd | ✅ Complete |
| **1.5** | etcd backup + cron + verified restore drill | ✅ Complete |
| **2** | MetalLB + ingress-nginx + Longhorn + StorageClass | ✅ Complete |
| **3** | ArgoCD install + first app sync | ✅ Complete |
| **4** | NetworkPolicy + PSS + RBAC + External Secrets Operator | ✅ Complete |
| **4.5** | Calico CNI + cert-manager + Internal Root CA + HTTPS | ✅ Complete |

### Part 2 — CI/CD Platform

| Phase | Focus | Status |
|---|---|---|
| **5** | GitLab CE + 7-stage pipeline + Cosign + real OTel services | 🔄 In Progress |
| **6** | Two-repo GitOps + ArgoCD signature verification + app-of-apps | ⏳ Upcoming |

### Part 3 — Observability

| Phase | Focus | Status |
|---|---|---|
| **7** | Prometheus + Alertmanager (Slack) + Grafana + DORA metrics | ⏳ Upcoming |
| **8** | Loki + Promtail DaemonSet + LogQL | ⏳ Upcoming |
| **9** | Jaeger (all-in-one) + Grafana datasource | ⏳ Upcoming |
| **10** | OTel Collector gateway + full pipeline | ⏳ Upcoming |

### Part 4 — Workload + Backup

| Phase | Focus | Status |
|---|---|---|
| **11** | OTel Demo 22 services + HPA + full telemetry correlation | ⏳ Upcoming |
| **12** | Velero backup + namespace restore drill + RTO measurement | ⏳ Upcoming |

### Part 5 — Cloud + IaC

| Phase | Focus | Status |
|---|---|---|
| **13** | Terraform foundation + S3 backend + module structure | ⏳ Upcoming |
| **14** | AWS: VPC + Route53 + SSM + S3 + IAM via Terraform | ⏳ Upcoming |
| **15** | EKS + ALB Controller + same workload + kubeadm vs EKS comparison | ⏳ Upcoming |

---

## Break-it Drills — Per Phase

> Phase pass condition: NOT "it worked." It is "I broke it, debugged it, fixed it, documented RTO."

### Phase 0 — VM Baseline
| Drill | Procedure | What to Learn |
|---|---|---|
| VM reboot | Reboot all VMs | swap stays off, kernel modules persist, containerd auto-starts |
| SSH lockout | Remove key before setting new one | Proxmox console recovery workflow |
| sysctl misconfig | Set wrong value, observe pod failure | Kernel parameters affect pod networking |

### Phase 1 — kubeadm Cluster
| Drill | Procedure | What to Learn |
|---|---|---|
| API server kill | Move kube-apiserver.yaml out of /etc/kubernetes/manifests | Static pod recovery, kubectl behavior |
| Worker reboot | Reboot md-w-1 | Pod rescheduling, NotReady detection, recovery time |
| etcd corruption | Corrupt data dir, restore from snapshot | Full etcd restore procedure, RTO |
| CNI delete | Remove Calico mid-cluster | Pod networking failure, CNI dependency |

### Phase 1.5 — etcd Backup
| Drill | Procedure | What to Learn |
|---|---|---|
| Restore drill | Delete namespace, restore from snapshot | Full restore procedure + data loss window |
| Snapshot corruption | Corrupt snapshot file, run status check | Integrity verification importance |
| Cron failure | Stop cron, observe missing backup | Monitor backup jobs |

### Phase 2 — Storage + LoadBalancer + Ingress
| Drill | Procedure | What to Learn |
|---|---|---|
| ingress pod kill | Delete ingress-nginx pod | Deployment self-healing speed |
| MetalLB speaker fail | Stop speaker on one node | L2 ARP failover timing |
| PVC orphan | Delete PVC while pod uses it | Volume lifecycle, stuck termination |
| Storage full | Fill PVC to 100% | Pod eviction behavior |

### Phase 4 — Security
| Drill | Procedure | What to Learn |
|---|---|---|
| Wrong NetworkPolicy | Block legitimate traffic | Debug with Calico policy trace |
| RBAC misuse | Forbidden action with limited SA | API server auth/authz response |
| Sealed Secret tamper | Modify encrypted bytes | Controller decryption failure logs |
| Lost private key | Delete sealed-secrets-controller secret | Key backup importance |

### Phase 4.5 — Calico + Longhorn + TLS
| Drill | Procedure | What to Learn |
|---|---|---|
| Calico node down | Stop calico-node on one worker | Pod networking failure on that node |
| Longhorn replica fail | Force-delete a replica | Auto-rebuild, health monitoring |
| Longhorn node drain | Drain node with attached volumes | Volume migration, RWO behavior |
| Cert expiry | Set cert to expired manually | cert-manager auto-renewal trigger |
| Wrong DNS | Break /etc/resolv.conf in pod | CoreDNS troubleshooting |

### Phase 5 — GitLab CI
| Drill | Procedure | What to Learn |
|---|---|---|
| Runner offline | Stop GitLab Runner | Pipeline queue behavior, CI failure alert |
| Build OOM | Trigger memory-intensive build | Pod limits, OOMKilled events |
| Registry full | Fill GitLab registry storage | Push failures, garbage collection |
| Bad Dockerfile | Push image with HIGH CVE | Trivy gate rejection, pipeline enforcement |

### Phase 6 — Full GitOps Loop
| Drill | Procedure | What to Learn |
|---|---|---|
| Invalid YAML | Push bad manifest to apps-values | ArgoCD sync failure, error inspection |
| Image pull fail | Reference non-existent tag | ImagePullBackOff, deployment rollback |
| Manual change | kubectl edit while ArgoCD watches | Drift detection, auto-revert |
| ArgoCD self-destruct | Delete ArgoCD app for ArgoCD | App-of-apps self-healing |

### Phase 7–10 — Observability
| Drill | Procedure | What to Learn |
|---|---|---|
| Prometheus disk full | Fill metrics storage | Retention policy, oldest data drop |
| Grafana DB lost | Delete Grafana PVC | Dashboard recovery from Git backup |
| Loki crash | Kill Loki pods | Log loss window, replication behavior |
| Collector OOM | Send massive trace volume | memory_limiter processor behavior |

### Phase 11 — OTel Demo
| Drill | Procedure | What to Learn |
|---|---|---|
| Service crash chain | Kill paymentservice | Cascading failures in Jaeger trace |
| Resource exhaustion | loadgenerator at max rate | Cluster RAM ceiling, HPA trigger |
| Network partition | NetworkPolicy breaks one service | Distributed system debugging |
| Latency injection | sleep() in one service | Performance debugging in Jaeger waterfall |

### Phase 12 — Backup + Recovery
| Drill | Procedure | What to Learn |
|---|---|---|
| Full namespace loss | Delete otel-demo namespace | Velero restore, RTO measurement |
| Volume snapshot restore | Modify data, restore from Longhorn snapshot | Snapshot workflow |
| Control plane down | Power off md-cp-1, restore | Full cluster recovery procedure |

### Phase 13–15 — AWS + Terraform
| Drill | Procedure | What to Learn |
|---|---|---|
| Terraform state edit | Manually corrupt tfstate | State lock importance, recovery |
| Wrong destroy | Run destroy on wrong resources | prevent_destroy, targeted destroy |
| EKS node drain | Cordon all nodes | Cluster autoscaler, rescheduling |
| IAM permission removed | Remove role permissions | IRSA auth failure debugging |

---

## Architecture Decision Records

| ADR | Decision | Rejected Alternative | Reason |
|---|---|---|---|
| 001 | Calico over Flannel | Flannel | NetworkPolicy enforcement required |
| 002 | Longhorn over local-path | local-path, NFS | Replication + snapshots + Velero CSI |
| 003 | GitLab CE over Gitea | Gitea | Full CI/CD: runner, registry, SAST, MR gates |
| 004 | Jaeger over Tempo | Tempo | Industry-standard, widely recognized |
| 005 | External Secrets Operator | Sealed Secrets | Multi-backend path: local → AWS SSM without app changes |
| 006 | kubeadm before EKS | EKS from start | Understand control plane before managed abstraction |
| 007 | Single control plane | 3-node HA | Lab resource constraint; etcd snapshot mitigates |
| 008 | Two-repo GitOps | Single monorepo | Clean code/deploy separation; rollback = values revert |
| 009 | Custom Dockerfiles per service | Upstream images | Build chain ownership + Trivy scan every image |
| 010 | Raw manifests before Helm | Helm from start | K8s primitives understanding before abstraction |
| 011 | Kaniko over Docker-in-Docker | DinD | Privilegeless builds — no root in CI runner |
| 012 | Gateway-only OTel Collector | Agent + gateway | Simpler debugging; agent mode deferred |
| 013 | 2-worker cluster | 3-worker | Resource budget; production pattern shown on EKS |

---

## Known Tradeoffs

| Tradeoff | Reason | Production Equivalent |
|---|---|---|
| Single control plane | Lab resource constraint | 3-node etcd + 2 control planes behind load balancer |
| 2-replica Longhorn | Only 2 worker nodes | 3-replica with 3+ workers |
| Self-signed Root CA | No public domain | Let's Encrypt with real domain |
| Ingress co-located on worker | Resource budget | Dedicated ingress node pool (shown on EKS Phase 15) |
| In-cluster observability | Lab simplicity | External Prometheus for cluster-level heartbeat |
| GitLab single instance | Resource budget | GitLab HA or GitLab.com |
| Tailscale single router | One Proxmox host | Redundant subnet routers |

---

## GitHub Repository Structure

```
mohiuddin-homelab/
├── README.md                        ← Architecture overview + status
├── ARCHITECTURE.md                  ← This document
├── docs/
│   ├── runbooks/                    ← Per-phase operational runbooks
│   │   ├── phase-0.md
│   │   ├── phase-1.md
│   │   ├── phase-2.md
│   │   ├── phase-3.md
│   │   ├── phase-4.md
│   │   ├── phase-4.5.md
│   │   ├── phase-5-gitlab-ci.md
│   │   └── failures/                ← Break-it drill outcomes + RTO
│   ├── decisions/                   ← ADR 001–013
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
│   └── interview-prep/              ← Q&A per phase (local study only)
├── manifests/                       ← Raw Kubernetes YAMLs
├── helm-values/                     ← Per-chart Helm values.yaml
├── dockerfiles/                     ← Per-service custom Dockerfiles
├── apps-values/                     ← GitOps source (ArgoCD watches)
│   ├── otel-demo/
│   ├── observability/
│   └── platform/
└── terraform/                       ← Phase 14+
    ├── bootstrap/                   ← S3 + DynamoDB (local state, run once)
    ├── modules/
    │   ├── vpc/
    │   ├── eks/
    │   └── secrets/
    └── envs/lab/
```

---

## Phase Pass Criteria

A phase is NOT complete until ALL of these are done:

- Tool deployed and working
- Verification commands run successfully and output documented
- Break-it drill executed and recovery time measured
- Runbook written in professional English — pushed to GitHub (`docs/runbooks/`)
- Drill outcome written — pushed to GitHub (`docs/runbooks/failures/`)
- ADR written if an architectural decision was made — pushed to GitHub (`docs/decisions/`)
- Screenshot taken for portfolio where applicable

**No shortcuts. No phase skipping. Every drill mandatory.**

---

*Production-grade homelab — debugging muscle built drill by drill.*
*Built by Mohiuddin — DevOps / Platform Engineer candidate.*
