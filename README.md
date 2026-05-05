#ARCHITECTURE.md
# Mohiuddin's DevOps & Platform Engineering Homelab

> **Version:** v2.0 (Production-Grade with Break-it Drills)
> **Status:** Active — Phase 4.5 in progress
> **Goal:** Production-grade DevOps platform with full observability, GitOps, and CI/CD
> **Hardware:** Proxmox bare-metal server (i9-10850K, 64GB DDR4) via Tailscale

---

## 🎯 Project Goals

1. **Hands-on production learning** — every tool installed manually, every concept understood deeply
2. **Production debugging muscle** — every phase ends with mandatory break-it drills
3. **Portfolio showcase** — GitHub runbooks, ADRs, architecture diagrams, screenshots
4. **Interview confidence** — real troubleshooting stories from break-it drills

---

## 🖥️ Infrastructure

### VM Layout

| VM | IP | Role | RAM | CPU | Disk |
|---|---|---|---|---|---|
| md-svc-1 | 10.20.0.50 | Root CA + DNS | 2 GB | 2 | 30 GB |
| md-cp-1 | 10.20.0.10 | K8s Control Plane | 4 GB | 2 | 40 GB |
| md-w-1 | 10.20.0.21 | K8s Worker 1 | 6 GB | 2 | 40 GB |
| md-w-2 | 10.20.0.22 | K8s Worker 2 | 6 GB | 2 | 40 GB |
| md-ci-1 | 10.20.0.30 | GitLab CI Server | 4 GB | 2 | 40 GB |

### Network Layout

| Network | CIDR | Purpose |
|---|---|---|
| Lab network | 10.20.0.0/24 | VM-to-VM communication |
| Pod CIDR | 10.244.0.0/16 | Kubernetes pods |
| Service CIDR | 10.96.0.0/12 | Kubernetes services |
| MetalLB pool | 10.20.0.200-210 | LoadBalancer IPs |

---

## 🏗️ Full Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Proxmox Server (i9-10850K, 64GB DDR4) via Tailscale                │
│                                                                     │
│  ┌──────────────┐                                                   │
│  │  md-svc-1    │  Root CA + DNS                                    │
│  │  10.20.0.50  │  cert-manager ClusterIssuer source                │
│  └──────────────┘                                                   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Kubernetes Cluster (kubeadm v1.31.14)                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │    │
│  │  │  md-cp-1    │  │   md-w-1    │  │   md-w-2    │          │    │
│  │  │ API Server  │  │ App Pods    │  │ Observability│         │    │
│  │  │ etcd        │  │ Ingress     │  │ Pods         │         │    │
│  │  │ ArgoCD      │  │             │  │              │         │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │    │
│  │                                                             │    │
│  │  CNI: Calico │ LB: MetalLB │ Storage: Longhorn              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌──────────────┐                                                   │
│  │  md-ci-1     │  GitLab CE + Runner + Registry                    │
│  │  10.20.0.30  │                                                   │
│  └──────────────┘                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Full Tool Stack

### Cluster Core
| Tool | Purpose |
|---|---|
| kubeadm | Cluster bootstrap |
| containerd | Container runtime |
| Calico | CNI + NetworkPolicy |
| Longhorn | Distributed storage |
| MetalLB | Bare-metal LoadBalancer |
| ingress-nginx | HTTP/HTTPS routing |
| cert-manager | TLS automation |
| metrics-server | Resource metrics |

### Security
| Tool | Purpose |
|---|---|
| Calico NetworkPolicy | Pod-to-pod traffic control |
| Sealed Secrets | Encrypted secrets in Git |
| Pod Security Standards | Pod-level security |
| RBAC | Least-privilege access |
| Root CA + cert-manager | Internal TLS infrastructure |

### GitOps + CI/CD
| Tool | Purpose |
|---|---|
| GitLab CE | Git server + CI/CD + Registry |
| GitLab Runner | CI pipeline executor |
| Kaniko | Privilegeless image builds |
| Trivy | Vulnerability scanning |
| ArgoCD | GitOps engine |

### Observability
| Tool | Purpose |
|---|---|
| Prometheus + Alertmanager | Metrics + alerts |
| Grafana | Unified dashboard |
| Loki + Promtail | Log aggregation |
| Tempo | Distributed tracing |
| OTel Collector | Telemetry pipeline |

### Workload
| Tool | Purpose |
|---|---|
| OpenTelemetry Demo | Real microservices app |

---

## 🌐 Traffic Flow

```
User (Browser)
    │
    ▼
10.20.0.200 (MetalLB LoadBalancer IP)
    │
    ▼
ingress-nginx (HTTPS, TLS via cert-manager)
    │
    ├── argocd.lab      → ArgoCD
    ├── grafana.lab     → Grafana
    ├── gitlab.lab      → GitLab
    ├── otel-demo.lab   → OTel Demo Frontend
    └── prometheus.lab  → Prometheus
```

---

## 🔄 CI/CD + GitOps Flow

```
Developer (git push)
    │
    ▼
GitLab (md-ci-1)
    │
    ├── Checkout code
    ├── Kaniko build (no privilege)
    ├── Trivy scan (HIGH/CRITICAL = fail)
    └── Push image → GitLab Registry
    │
    ▼
CI updates image tag in apps-values repo
    │
    ▼
ArgoCD detects change
    │
    ├── dev → auto-sync
    └── prod → manual approval
    │
    ▼
Pod restarts with new image
```

---

## 📊 Observability Flow

```
OTel Demo Apps
    │
    ▼ OTLP (gRPC :4317)
OTel Collector (gateway)
    │
    ├── Metrics → Prometheus
    ├── Logs    → Loki
    └── Traces  → Tempo
    │
    ▼
Grafana (unified UI)
    └── Trace → Logs correlation ← THE WOW MOMENT
```

---

## 🔐 Security Architecture

```
Pod Security Standards:
├── kube-system     → privileged
├── argocd          → baseline
├── observability   → baseline
└── otel-demo       → restricted

NetworkPolicy:
├── Default deny (per namespace)
├── Allow: ingress-nginx → otel-demo
├── Allow: otel-demo apps → otel-demo apps
├── Allow: all → OTel Collector
└── Allow: monitoring → all (Prometheus scrape)

TLS:
└── Root CA (md-svc-1) → cert-manager → all Ingress HTTPS

Secrets:
└── Sealed Secrets only — no plaintext in Git
```

---

## 💥 Break-it Drills — Per Phase

> **Rule:** Phase pass condition is NOT "it worked." It is "I broke it, debugged it, fixed it, and documented it."

### Phase 0 — Infrastructure Baseline
| Drill | Procedure | What to Learn |
|---|---|---|
| VM reboot | Reboot all VMs | Verify swap stays off, kernel modules persist, containerd auto-starts |
| SSH lock out | Disable password auth before key inject | Proxmox console recovery, key injection workflow |
| sysctl misconfig | Set wrong sysctl, observe pod failure | How kernel parameters affect networking |

### Phase 1 — kubeadm Cluster
| Drill | Procedure | What to Learn |
|---|---|---|
| API server kill | Move kube-apiserver.yaml manifest | Static pod recovery, kubectl behavior when API down |
| Worker node reboot | Reboot md-w-1 | Pod rescheduling, node NotReady detection |
| etcd corruption | Stop etcd, corrupt data dir | Restore from snapshot, RTO measurement |
| CNI delete | Remove Flannel/Calico mid-cluster | Pod networking failure, CNI dependency |

### Phase 1.5 — etcd Backup
| Drill | Procedure | What to Learn |
|---|---|---|
| Restore drill | Delete a namespace, restore etcd from snapshot | Full etcd restore procedure, data loss measurement |
| Backup verification | Corrupt a snapshot file, run status check | Why integrity verification matters |
| Cron failure | Stop cron service, check missing backups | Monitoring backup jobs |

### Phase 2 — Storage + LoadBalancer + Ingress
| Drill | Procedure | What to Learn |
|---|---|---|
| ingress-nginx pod kill | Delete ingress controller pod | Deployment self-healing, traffic disruption window |
| MetalLB speaker fail | Stop speaker on one node | L2 mode failover, ARP retransmission timing |
| PVC orphan | Delete PVC while pod uses it | Volume lifecycle, pod stuck termination |
| Storage full | Fill PVC to 100% | Pod behavior, eviction triggers |

### Phase 4 — Security
| Drill | Procedure | What to Learn |
|---|---|---|
| Wrong NetworkPolicy | Apply policy that blocks legit traffic | Debug with kubectl describe, traffic flow analysis |
| RBAC misuse | Try forbidden action with limited SA | API server auth/authz response, audit log reading |
| Sealed Secret tampering | Modify encrypted bytes manually | Decryption failure, controller logs |
| Lost private key | Delete sealed-secrets-controller secret | Disaster scenario, key backup importance |

### Phase 4.5 — Calico + Longhorn + TLS (Production Foundation)
| Drill | Procedure | What to Learn |
|---|---|---|
| Calico node down | Stop calico-node on one worker | Pod networking failure on that node |
| Longhorn replica fail | Force-delete a replica volume | Auto-rebuild, replica health monitoring |
| Longhorn node drain | Drain a node with attached volumes | Volume migration, RWO behavior |
| Cert expiry | Manually set cert to expired | cert-manager auto-renewal trigger |
| Wrong DNS | Break /etc/resolv.conf in a pod | DNS troubleshooting, CoreDNS logs |

### Phase 5 — GitLab CI
| Drill | Procedure | What to Learn |
|---|---|---|
| Runner offline | Stop GitLab Runner | Pipeline queue behavior, CI failure detection |
| Build OOM | Trigger build with massive memory | Pod limits, OOMKilled events |
| Registry full | Fill GitLab registry storage | Push failures, garbage collection |
| Bad Dockerfile | Push intentionally broken image | Trivy scan rejection, pipeline gate validation |

### Phase 6 — Full GitOps Loop
| Drill | Procedure | What to Learn |
|---|---|---|
| Bad commit to apps-values | Push invalid YAML | ArgoCD sync failure, error inspection |
| Image pull fail | Reference non-existent tag | Pod ImagePullBackOff, deployment rollback |
| Manual cluster change | kubectl edit while ArgoCD watches | ArgoCD drift detection, auto-revert behavior |
| ArgoCD self-destruction | Delete ArgoCD app for ArgoCD itself | Self-healing, app-of-apps recovery |

### Phase 7-10 — Observability
| Drill | Procedure | What to Learn |
|---|---|---|
| Prometheus disk full | Fill metrics storage | Retention policy, oldest data drop |
| Grafana DB corruption | Delete grafana DB pvc | Dashboard recovery from Git backup |
| Loki ingester crash | Kill Loki pods | Log loss window, replication strategy |
| OTel Collector OOM | Send massive trace volume | memory_limiter behavior, dropped data |
| Missing label | Remove a label that dashboards depend on | Dashboard breakage, label discovery |

### Phase 11 — OTel Demo
| Drill | Procedure | What to Learn |
|---|---|---|
| Service crash chain | Kill paymentservice | Cascading failures, trace error propagation |
| Resource exhaustion | Run loadgenerator at max | Pod scheduling, cluster RAM ceiling |
| Network partition | Apply NetworkPolicy that breaks one service | Distributed system debugging |
| Slow trace | Inject latency in one service | Performance debugging in Tempo |

### Phase 12 — Backup + Recovery
| Drill | Procedure | What to Learn |
|---|---|---|
| Full namespace loss | Delete entire otel-demo namespace | ArgoCD redeploy from Git, data loss measurement |
| Volume snapshot restore | Snapshot, modify, restore | Longhorn snapshot workflow |
| Disaster simulation | Power off control plane, restore | Full cluster recovery procedure |

### Phase 13-15 — AWS + EKS
| Drill | Procedure | What to Learn |
|---|---|---|
| Terraform state corruption | Manually edit tfstate | State lock importance, recovery |
| Wrong terraform destroy | Run destroy on prod-like resources | Targeted destroy, prevent_destroy lifecycle |
| EKS node group fail | Cordon all nodes | Pod rescheduling, cluster autoscaler |
| IAM permission drift | Remove role permissions | Pod auth failure debugging |

---

## 📦 Namespace Plan

| Namespace | Contents | PSS Level |
|---|---|---|
| kube-system | Core K8s, Longhorn, cert-manager | privileged |
| metallb-system | MetalLB | baseline |
| ingress-nginx | ingress controller | baseline |
| argocd | ArgoCD | baseline |
| gitlab | GitLab CE + Runner | baseline |
| observability | Prometheus, Grafana, Loki, Tempo, OTel | baseline |
| otel-demo | Demo microservices | restricted |
| longhorn-system | Longhorn storage | privileged |

---

## 🚀 Phase Plan

### ✅ Done
- Phase 0 — VM baseline
- Phase 1 — kubeadm cluster
- Phase 1.5 — etcd backup
- Phase 2 — Helm, MetalLB, ingress-nginx, local-path
- Phase 3 — ArgoCD
- Phase 4 — RBAC, Sealed Secrets, NetworkPolicy

### 🔄 In Progress
- Phase 4.5 — Calico, Longhorn, Root CA, cert-manager, HTTPS

### ⏳ Upcoming
- Phase 5 — GitLab CE + CI pipeline
- Phase 6 — Full GitOps loop, two-repo
- Phase 7-10 — Prometheus, Grafana, Loki, Tempo, OTel Collector
- Phase 11 — OTel Demo full deploy
- Phase 12 — Backup + restore drill
- Phase 13-15 — Terraform, AWS, EKS

---

## 📝 Architecture Decision Records (ADRs)

| ADR | Decision | Reason |
|---|---|---|
| 001 | Calico over Flannel | NetworkPolicy enforcement |
| 002 | Longhorn over local-path | Replication + snapshots |
| 003 | GitLab over Gitea | Full CI/CD + production parity |
| 004 | Tempo over Jaeger | Grafana correlation |
| 005 | Sealed Secrets over ESO | No external backend |
| 006 | kubeadm before EKS | Control plane internals |
| 007 | Single control plane | Lab tradeoff |
| 008 | Two-repo GitOps | Clean code/deploy separation |

---

## ⚠️ Known Tradeoffs (Interview-Ready)

| Tradeoff | Reason | Production Alternative |
|---|---|---|
| Single control plane | Lab resource | 3-node etcd + 2 control plane |
| Self-signed CA | No public domain | Let's Encrypt |
| Longhorn 2 workers | Min replication | 3+ workers for full HA |
| GitLab single instance | Resource | GitLab HA |

---

## 📁 GitHub Repository Structure

```
mohiuddin-homelab/
├── README.md                    ← Architecture overview + screenshots
├── ARCHITECTURE.md              ← This document
├── docs/
│   ├── runbooks/                ← Per-phase runbooks
│   ├── decisions/               ← ADRs
│   ├── interview-prep/          ← Q&A per phase
│   └── break-it-drills/         ← Drill outcomes + lessons
├── manifests/                   ← Raw YAMLs
├── helm-values/                 ← Helm values
├── apps-values/                 ← GitOps repo
└── terraform/                   ← Phase 13+
```

---

## 🎯 Phase Pass Criteria

A phase is NOT complete until ALL of these are done:

✅ Tool deployed and working  
✅ Verification commands run successfully  
✅ Break-it drill executed  
✅ Recovery time measured  
✅ Runbook written (English) — pushed to GitHub  
✅ Interview Q&A written (English + Bangla)  
✅ ADR written if architectural decision made  
✅ Screenshot taken for portfolio (where applicable)  

**No shortcuts. No phase skipping. Every drill mandatory.**

---

*Production-grade homelab — debugging muscle built drill by drill.*  
*Built by Mohiuddin — DevOps / Platform Engineer candidate.*
