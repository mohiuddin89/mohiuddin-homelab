# Mohiuddin's DevOps & Platform Engineering Homelab

> **Version:** v3.0 (Production-Grade)
> **Status:** Phase 6 in progress (GitOps — ArgoCD Application pending)
> **Goal:** Production-grade DevOps platform with full observability, GitOps, supply-chain-secured CI/CD, and AI Platform extension
> **Hardware:** Proxmox VE on i9-10850K, 64 GB DDR4 — accessed via Tailscale
> **Last updated:** 2026-05-12

---

## 🎯 Project Goals

1. **Hands-on production learning** — every tool installed manually, every concept understood deeply
2. **Production debugging muscle** — every phase ends with mandatory break-it drills
3. **Portfolio showcase** — GitHub runbooks, ADRs, architecture diagrams, screenshots
4. **Interview confidence** — real troubleshooting stories from actual failures
5. **AI Platform readiness** — foundation for LLM gateway, RAG pipeline, AI observability

---

## 📐 Execution Philosophy — Manual First, Then IaC

**Core rule:**

> Homelab execution remains manual for learning clarity.
> Terraform becomes mandatory for AWS phases.
> Terraform learning may begin earlier (concepts, syntax, small labs),
> but no Terraform is used to provision homelab infrastructure.
> AWS provisioning is Terraform-first after bootstrap exceptions.

### Why this ordering

- **Understand concrete before abstraction** — must know what Terraform automates by doing it manually first
- **Debugging muscle** — manual setup forces OS / networking root-cause work
- **Failure modes exposed** — kubeadm certs, kubelet, containerd touched directly
- **Judgment for IaC value** — know where it's overkill, where it earns its place
- **Interview signal** — "bare-metal K8s manually, then migrated to EKS via Terraform" >> "ran terraform apply"

### Manual = Required

| Layer | Why Manual |
|---|---|
| Proxmox + VMs | Hypervisor layer — click-driven for learning |
| Kubernetes (kubeadm) | Every control plane component visible |
| Add-ons (MetalLB, ingress, Longhorn) | Helm charts installed manually |
| CI/CD (GitLab, ArgoCD) | Web UI + config files |
| Observability stack | Helm install + manual configuration |
| OTel Demo workload | Custom Dockerfiles + manifest-based deploy |

### IaC = Required

| Layer | Tool |
|---|---|
| AWS resources (VPC, S3, Route53, SSM, IAM) | Terraform |
| EKS cluster + node groups | Terraform (`terraform-aws-modules/eks`) |
| AI Platform cloud resources | Terraform |

---

## 🧭 Core Principles

- **Design before install** — architecture first, tool later
- **Manual first, IaC when justified** — Phase 14+ for provisioning
- **Verify every step** — no pass without validation
- **Break → debug → document** — every failure produces a runbook
- **OTLP everywhere** — all telemetry uses OTLP protocol (no legacy Jaeger Thrift, no Zipkin)
- **Least privilege by default** — every ServiceAccount, Role, NetworkPolicy starts from deny
- **Signed-only deploys** — every container image signed with Cosign; ArgoCD verifies before sync
- **Every component must answer:** Purpose, Traffic flow, Failure story, Recovery plan

---

## 🖥️ Naming Convention

| Role | Name | IP |
|---|---|---|
| Proxmox host | pve | 192.168.68.200 |
| Control plane | md-cp-1 | 10.20.0.10 |
| Worker 1 | md-w-1 | 10.20.0.21 |
| Worker 2 (ingress co-located) | md-w-2 | 10.20.0.22 |
| CI server | md-ci-1 | 10.20.0.30 |
| Utility server (DNS + CA) | md-svc-1 | 10.20.0.50 |

---

## 🌐 Network Architecture

### Networks

| Network | CIDR | Purpose |
|---|---|---|
| Management | 192.168.68.0/24 | Proxmox UI, internet access |
| Lab (isolated) | 10.20.0.0/24 | All VMs, Kubernetes |
| Pod CIDR | 10.244.0.0/16 | Kubernetes pods (Calico) |
| Service CIDR | 10.96.0.0/12 | Kubernetes services |
| MetalLB pool | 10.20.0.200–210 | LoadBalancer IPs |

### Proxmox Bridges

- `vmbr0` → management network (physical NIC)
- `vmbr1` → isolated lab bridge (no physical NIC, lab VMs only)

### Critical Proxmox Config

```bash
# /etc/sysctl.d/99-ipforward.conf
net.ipv4.ip_forward=1

# nftables: FORWARD rules between vmbr0 ↔ vmbr1
# MASQUERADE on vmbr0 so lab VMs reach the internet via NAT
```

---

## 🔐 Remote Access — Tailscale

- pve → Tailscale subnet router
- Advertised routes: `192.168.68.0/24`, `10.20.0.0/24`
- Access from anywhere via Tailscale client

**Tradeoff documented:** Single subnet router. pve down = remote access lost. Acceptable at lab scale. Production equivalent: redundant subnet routers.

---

## 🖥️ VM Architecture

| VM | IP | Role | Spec |
|---|---|---|---|
| md-cp-1 | 10.20.0.10 | K8s Control Plane · etcd · apiserver | 2C / 4GB / 40GB |
| md-w-1 | 10.20.0.21 | App workloads | 2C / 6GB / 40GB |
| md-w-2 | 10.20.0.22 | App workloads + ingress | 2C / 6GB / 40GB |
| md-ci-1 | 10.20.0.30 | GitLab CE + Runner + Registry | 2C / 4GB / 40GB |
| md-svc-1 | 10.20.0.50 | Internal DNS + Root CA | 2C / 2GB / 30GB |

### Documented Tradeoffs

**Single control plane**
- Not HA — md-cp-1 down = cluster API unavailable
- Production equivalent: 3-node etcd + 2+ control planes behind LB
- Mitigation: etcd snapshot every 6h, documented restore procedure

**md-svc-1 SPOF (reduced after Phase 13)**

Pre-Phase 13 blast radius:
- DNS down → all `*.lab` unreachable
- Root CA down → cert renewals fail

Post-Phase 13:
- Backup target moved → external disk / NAS
- md-svc-1 scope reduced to DNS + CA only

**Ingress co-located on md-w-2**
- No dedicated ingress node (resource budget)
- ingress-nginx scheduled on md-w-2 via `nodeSelector` + `podAntiAffinity` rules
- Tradeoff: md-w-2 saturation impacts both app pods and ingress
- Production equivalent: dedicated ingress NodePool (demonstrated on EKS in Phase 16)

**In-cluster observability blind spot**
- All metrics, logs, traces in-cluster
- Cluster fully down = observability also down
- Accepted for homelab simplicity
- Production mitigation (future optional): external Prometheus on md-svc-1 for cluster-level heartbeat

---

## ☸️ Kubernetes Architecture

**Tool:** kubeadm (bare-metal, fully manual install) — v1.31.x (pinned)
**CNI:** Calico (NetworkPolicy enforcement)
**Runtime:** containerd (systemd cgroup driver)

### Traffic Flow

```
User (Browser / curl)
   │
   ▼
MetalLB IP (10.20.0.200+)        ← L2 mode, ARP-based
   │
   ▼
ingress-nginx (HTTPS via cert-manager)
   │
   ▼
Kubernetes Service (ClusterIP)
   │
   ▼
Pods (App / Grafana / ArgoCD / etc.)
```

### Namespaces & Sync Policy

| Namespace | Sync Policy | Pod Security |
|---|---|---|
| dev | ArgoCD auto-sync | baseline |
| staging | ArgoCD manual gate | baseline |
| prod | ArgoCD manual sync only | restricted |
| kube-system | N/A | privileged |
| argocd | N/A | baseline |
| gitlab | N/A | baseline |
| observability | N/A | baseline |
| otel-demo | ArgoCD-managed | restricted |
| longhorn-system | N/A | privileged |
| metallb-system | N/A | baseline |
| ingress-nginx | N/A | baseline |

### Add-ons Stack (all manual install)

| Tool | Purpose | Install Method |
|---|---|---|
| Calico | Networking + NetworkPolicy | `kubectl apply` manifest |
| MetalLB | LoadBalancer IP (L2 mode) | Helm, manual values |
| ingress-nginx | HTTP/HTTPS routing | Helm, manual values |
| metrics-server | `kubectl top`, HPA | `kubectl apply` manifest |
| Longhorn | StorageClass (dynamic PVC, replicated) | Helm, manual values |
| cert-manager | TLS automation (internal CA issuer) | Helm, manual values |
| ArgoCD | GitOps engine | `kubectl apply` manifest |
| External Secrets Operator | Secrets injection from backend | Helm, manual values |
| kube-prometheus-stack | Prometheus + Alertmanager + Grafana | Helm, manual values |
| Loki | Log aggregation | Helm, manual values |
| Jaeger | Distributed tracing backend | Helm, manual values |
| OTel Collector | Telemetry pipeline (gateway mode) | Helm, manual values |
| Velero | Cluster + PVC backup | Helm, manual values |

> **Rule:** All Helm commands manual. No Helmfile, no ArgoCD app-of-apps bootstrap, no Terraform Helm provider. Every values file read and edited by hand.

---

## 📦 Storage — Longhorn

- Dynamic PVC provisioning via StorageClass
- Replicated across worker nodes (md-w-1, md-w-2)
- **Replica count: 2** (not default 3) — resource budget tradeoff, documented in ADR 002
- Backup target: external disk mounted to Proxmox (Phase 13, not md-svc-1)
- Longhorn UI accessible via ingress for monitoring replica health

**Production-grade additions:**
- Recurring jobs: daily snapshot, weekly backup to external target
- Volume-specific replica count override for critical PVCs (Prometheus, GitLab data)

---

## 🔐 Security Design

### NetworkPolicy (Phase 4)

Default deny all ingress + egress per namespace. Explicit allow rules:

```yaml
# ingress-nginx → app pods (HTTP/HTTPS routing)
# app pods → app pods (same namespace, named ports only)
# app pods → kube-system DNS (port 53 UDP/TCP)
# app pods → OTel Collector (OTLP gRPC :4317, OTLP HTTP :4318)
# app pods → external HTTPS egress (Phase 17+ AI API calls — explicit CIDR rule)
# monitoring → all namespaces (Prometheus cross-namespace scrape)
# argocd → all namespaces (GitOps sync via kubectl API)
```

> ⚠️ Without these explicit allows: Phase 11 OTel telemetry drops silently; Phase 17+ AI API calls block; Prometheus cross-namespace scrape fails. Debug this early.

### Pod Security Standards

Enforced via namespace labels (`pod-security.kubernetes.io/enforce`):

| Namespace | Level | Reason |
|---|---|---|
| prod / otel-demo | `restricted` | No root, no privilege escalation, drop all caps |
| staging / dev / argocd / observability / gitlab / ingress-nginx / metallb-system | `baseline` | No privileged containers, some capabilities allowed |
| kube-system / longhorn-system | `privileged` | System-level access required |

All `restricted` namespace pods must include:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault       # syscall filtering — kernel default filter
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

### LimitRange — Per Namespace

Every non-system namespace gets a `LimitRange`. Prevents a single unbounded pod from starving the cluster:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: otel-demo
spec:
  limits:
  - type: Container
    default:                # applied when pod does not set limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:         # applied when pod does not set requests
      cpu: "100m"
      memory: "128Mi"
    max:                    # hard ceiling
      cpu: "2"
      memory: "2Gi"
```

> **Why production-grade:** In production, every namespace has resource quotas and LimitRanges. Without them, a misconfigured pod can consume all node memory and evict critical pods.

### PodDisruptionBudget

For critical single-replica workloads, PDB prevents voluntary disruptions (node drain, rolling upgrade) from taking them down:

```yaml
# ingress-nginx — always keep at least 1 ready
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ingress-nginx-pdb
  namespace: ingress-nginx
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
```

Same pattern applied to ArgoCD server, Grafana, Prometheus.

### Secrets

- **Tool:** External Secrets Operator (ESO)
- **Phase 7 backend:** local Kubernetes secret store
- **Phase 15 backend:** AWS SSM Parameter Store (ESO migrates backend without app changes)
- **Rule:** No plaintext secrets in Git. Ever.
- **Rotation:** Automatic when backend rotates (ESO polls on refresh interval)

### TLS

- cert-manager + Root CA on md-svc-1
- HTTPS on all Ingress objects from Phase 4 onward
- Quarterly cert rotation drill (delete TLS secret → verify cert-manager auto-renews)
- Internal CA cert pinned in ArgoCD repo-server config

---

## 🌐 Internal DNS — md-svc-1

**Tool:** dnsmasq (lightweight, simple to backup and restore)

| Domain | Target |
|---|---|
| grafana.lab | MetalLB IP (ingress) |
| argocd.lab | MetalLB IP (ingress) |
| gitlab.lab | md-ci-1 direct (10.20.0.30) |
| prometheus.lab | MetalLB IP (ingress) |
| alertmanager.lab | MetalLB IP (ingress) |
| jaeger.lab | MetalLB IP (ingress) |
| otel-demo.lab | MetalLB IP (ingress) |

All lab VMs use md-svc-1 as DNS resolver (set via netplan `nameservers`).

---

## 🔄 CI/CD + GitOps Flow

```
Developer (git push / git tag v1.2.3)
   │
   ▼
GitLab Repo (md-ci-1)
   │
   ▼
GitLab CI Pipeline
   │
   ├── STAGE 1: build
   │     └── Kaniko (privilegeless image build)
   │           Tags: sha-<commit> AND semver vX.Y.Z if tagged
   │
   ├── STAGE 2: scan
   │     └── Trivy — fail on HIGH or CRITICAL CVEs
   │           SARIF report → GitLab Security tab
   │
   ├── STAGE 3: sign
   │     └── Cosign — sign image with keyless OIDC (GitLab CI identity)
   │           Signature stored in registry alongside image
   │
   ├── STAGE 4: push
   │     └── Push signed image → GitLab Container Registry
   │
   ├── STAGE 5: update-values
   │     └── CI updates image tag in apps-values repo (git commit via API)
   │
   └── STAGE 6: mirror
         └── git push --mirror → GitHub (offsite backup)
   │
   ▼
ArgoCD detects change in apps-values repo
   │
   ├── ArgoCD verifies Cosign signature before sync
   │
   ├── dev  → auto-sync
   └── prod → manual approval gate
   │
   ▼
Pod restarts with new signed, scanned image
```

### Two-Repo GitOps Pattern

- **`apps-source/`** — application code, Dockerfile, `.gitlab-ci.yml`
- **`apps-values/`** — K8s manifests / Helm values. ArgoCD watches this only.

Source code changes and deployment changes stay separated. Rollback = revert a commit in `apps-values/`. ArgoCD re-syncs automatically.

### Image Tagging Strategy (Semantic Versioning)

| Trigger | Tag Format | Use Case |
|---|---|---|
| Commit to `main` | `sha-<short-commit>` | Development, dev namespace deploy |
| Git tag `v1.2.3` | `v1.2.3` AND `latest` | Staging / production deploy |
| Feature branch | `branch-<name>-sha-<commit>` | PR testing only |

> **Why semver matters:** SHA tags are opaque. `v1.2.3` in a deployment manifest tells you exactly what changed and when. Incident response is faster when image tags are human-readable.

---

## 📊 Observability Stack

### Architecture

```
Applications (Pods)
   │
   ▼ OTLP gRPC :4317 (all signals)
OTel Collector (Deployment — gateway mode, single replica)
   │
   ├── Metrics  ──→ Prometheus (Prometheus pulls Collector :8889)
   ├── Logs     ──→ Loki :3100 (Collector pushes)
   └── Traces   ──→ Jaeger :4317 OTLP (Collector pushes via OTLP)
   │
   ▼
Grafana (unified UI — all 3 signals in one pane)
   └── Trace → Logs → Metrics correlation ← THE WOW MOMENT
```

### OTLP Everywhere

> **All telemetry uses OTLP protocol.** No legacy Jaeger Thrift (`:14250`), no Zipkin.
>
> - Apps → OTel Collector: OTLP gRPC `:4317`
> - OTel Collector → Jaeger: OTLP gRPC `:4317` (Jaeger 1.35+ has native OTLP receiver)
>
> Why: OTLP is the vendor-neutral standard. If Jaeger is later replaced by Tempo or a cloud backend, only the Collector exporter changes — apps stay unchanged.

### Push vs Pull Reference

| Signal | Model | Protocol | Endpoint |
|---|---|---|---|
| Metrics | Pull — Prometheus scrapes | Prometheus exposition | Collector `:8889` |
| Traces | Push — Collector pushes | OTLP gRPC | Jaeger `:4317` |
| Logs | Push — Collector pushes | Loki push API | Loki `:3100` |

> Collector does NOT push to Prometheus. Prometheus actively pulls from the Collector's metrics endpoint.

### OTel Collector — Production-Grade Config

```yaml
processors:
  memory_limiter:                # Drops data before OOM
    check_interval: 1s
    limit_mib: 400
    spike_limit_mib: 100
  batch:                         # Reduces export connections
    timeout: 5s
    send_batch_size: 512
  resource:                      # Standardize resource attributes
    attributes:
      - key: deployment.environment
        value: homelab
        action: insert

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### Collector Mode — Gateway First

**Initial (Phase 11):** Single gateway Collector (Deployment). One endpoint, one config, simpler debugging.

**Future optional upgrade:** Agent DaemonSet per node for host metrics + infrastructure logs → feeds gateway Collector for processing.

### Alertmanager Routing

Production observability requires alerts to actually reach someone. Phase 8 scope includes Slack routing:

```yaml
route:
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-homelab'
  routes:
    - match: { severity: critical }
      receiver: 'slack-homelab'
    - match: { severity: warning }
      receiver: 'slack-homelab'

receivers:
  - name: 'slack-homelab'
    slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'   # stored in ESO secret, not plaintext
        channel: '#homelab-alerts'
```

**Mandatory alerts to configure:**

- `KubePodCrashLooping` — pod restarting > 5 times
- `KubeNodeNotReady` — node goes NotReady
- `PrometheusTargetMissing` — scrape target disappears
- `LonghornVolumeRobustnessStatus` — volume replica degraded
- `etcdHighCommitDurations` — etcd performance warning
- Custom: OTel Collector memory usage > 80%

### DORA Metrics (Phase 11 Deliverable)

Four engineering metrics tracked in a Grafana dashboard:

| Metric | Source | How to Track |
|---|---|---|
| Deployment Frequency | ArgoCD sync events | ArgoCD metrics → Prometheus → Grafana |
| Lead Time for Change | GitLab CI pipeline duration | GitLab metrics → Prometheus |
| Change Failure Rate | ArgoCD sync failures + manual rollbacks | ArgoCD app health history |
| MTTR | Time between AlertFiring → ArgoCD sync success | Alertmanager → Prometheus ruler |

A DORA metrics dashboard is a strong portfolio piece for any platform engineering interview.

---

## 🔄 ArgoCD Design

### App-of-Apps Pattern

```
root-app (ArgoCD Application)
   │
   ├── otel-demo-app      → apps-values/otel-demo/
   ├── observability-app  → apps-values/observability/
   └── platform-app       → apps-values/platform/
```

Root app deployed once. All children managed by ArgoCD watching the values repo.

### Sync Policies

| Namespace | Auto-Sync | Self-Heal | Prune |
|---|---|---|---|
| dev | Yes | Yes | Yes |
| staging | No | No | No |
| prod | No | No | No |

> `selfHeal: true` reverts manual `kubectl` changes automatically. Good for dev, dangerous for prod without an approval gate.

### Image Signature Verification

ArgoCD only deploys images signed by the GitLab CI OIDC identity. Unsigned images fail sync → alert fires.

---

## 💾 Backup & Recovery

### Core Backup Targets

| Target | Tool | Frequency | Phase |
|---|---|---|---|
| etcd | `etcdctl snapshot save` | Every 6h | Phase 2.5 |
| GitLab | `gitlab-backup create` | Daily | Phase 13 |
| Longhorn PVCs | Longhorn recurring snapshot + backup | Daily | Phase 13 |
| Cluster PVC backup offsite | Velero + S3/NFS target | Weekly | Phase 13 |

### Velero — Production PVC Backup

**Why Velero over manual `kubectl cp`:**
- Application-consistent snapshots (Longhorn CSI hooks)
- Namespace-level backup + restore in one command
- Real disaster recovery — rebuild a whole namespace from offsite
- Interview-relevant: "I used Velero for cluster-level disaster recovery drills"

```bash
# Backup entire namespace:
velero backup create otel-demo-backup --include-namespaces otel-demo

# Restore drill:
kubectl delete namespace otel-demo
velero restore create --from-backup otel-demo-backup

# Measure RTO: time from delete → fully running = MTTR baseline
```

### Additional Backup Scope

**md-svc-1 (critical):**
- Root CA private key → encrypted (GPG) → external disk AND private GitHub repo
- dnsmasq config → Git (version controlled)

**GitLab:**
- `--mirror` push to GitHub after every pipeline (automated CI Stage 6)
- md-ci-1 destroyed scenario: restore GitLab → pull code from GitHub mirror

**ArgoCD:**
- Application manifests are in Git (source of truth)
- Repo credentials → ESO managed secret

> ⚠️ **Backup ≠ useful until restore drill complete.** Phase 13 pass condition: delete namespace, restore from backup, measure RTO.

---

## ⚠️ Break-it Model — Per Phase

Phase pass condition is NOT "it worked."

It is: **"I broke it, debugged it, fixed it, measured recovery time, and committed a runbook."**

```
For every phase:
☐ Kill a core component pod / service
☐ Measure recovery time (start timer on break, stop when healthy)
☐ Simulate one realistic failure scenario
☐ Write 1-page runbook: detect → debug → recover → prevent
☐ Commit runbook to docs/runbooks/failures/ in Git
```

### Phase-by-Phase Break-it Drills

#### Phase 0–1 — Kubernetes Foundation
| Drill | What to Learn |
|---|---|
| Reboot all VMs | swap stays off, kernel modules persist, containerd auto-starts |
| Move `kube-apiserver.yaml` static pod manifest | static pod recovery, kubectl behavior with API down |
| Reboot a worker | pod rescheduling, NotReady detection, recovery time |
| Corrupt etcd data dir | restore from snapshot, measure RTO |
| Delete Calico pods | pod networking failure, CNI recovery |

#### Phase 2.5 — etcd Backup
| Drill | What to Learn |
|---|---|
| Delete a namespace, restore from etcd snapshot | full etcd restore procedure |
| Corrupt snapshot file, run status check | why integrity verification matters |
| Stop cron service, observe missing backup | monitoring backup jobs |

#### Phase 3 — Access + Storage
| Drill | What to Learn |
|---|---|
| Delete ingress-nginx pod | deployment self-healing speed |
| Stop MetalLB speaker on one node | L2 ARP failover timing |
| Fill PVC to 100% | pod eviction behavior, storage alerts |
| Force-delete Longhorn replica | auto-rebuild, replica health dashboard |

#### Phase 4 — Security
| Drill | What to Learn |
|---|---|
| Apply NetworkPolicy that breaks app | traffic flow debugging, Calico policy trace |
| Forbidden RBAC action with limited SA | API server auth/authz response |
| Tamper Sealed Secret bytes | decryption failure, controller logs |
| Hit LimitRange ceiling | admission controller behavior |
| `kubectl drain` with PDB | drain blocking, PDB interaction |

#### Phase 5 — GitLab CI
| Drill | What to Learn |
|---|---|
| Runner goes offline | pipeline queue, CI failure alert |
| Trivy blocks HIGH CVE | pipeline gate validation, remediation loop |
| Cosign signing fails | image signing failure, unsigned blocked by ArgoCD |
| Registry disk full | push failure, garbage collection |

#### Phase 6–7 — GitOps + Secrets
| Drill | What to Learn |
|---|---|
| Invalid YAML pushed to apps-values | ArgoCD sync failure, error inspection |
| Non-existent image tag referenced | ImagePullBackOff, ArgoCD health status |
| Manual `kubectl edit` while ArgoCD watches | drift detection, auto-revert (selfHeal) |
| ESO backend unreachable | pod startup failure, secret mount behavior |

#### Phase 8–11 — Observability
| Drill | What to Learn |
|---|---|
| Prometheus storage fills | retention behavior, oldest data drop |
| Delete Grafana DB PVC | dashboard recovery from Git backup |
| OTel Collector OOM | `memory_limiter` behavior, dropped spans |
| Misconfigure Alertmanager webhook | alert routing debug |
| Remove Prometheus label used by dashboard | dashboard breakage, label discovery |
| Kill Loki | log loss window, query failures |

#### Phase 12 — OTel Demo Workload
| Drill | What to Learn |
|---|---|
| Kill paymentservice | cascading trace errors, timeout propagation |
| Run loadgenerator at max rate | cluster RAM ceiling, scheduling pressure |
| NetworkPolicy blocks one service | distributed system debugging via telemetry |
| Inject `sleep()` latency in one service | Jaeger trace waterfall analysis |
| HPA scale event | observe HPA trigger, pod schedule time |

#### Phase 13 — Backup + Recovery
| Drill | What to Learn |
|---|---|
| `velero backup` → delete namespace → restore | full namespace recovery, RTO measurement |
| Longhorn snapshot → modify data → restore | snapshot workflow, data rollback |
| Stop DNS + CA on md-svc-1 | blast radius, certificate behavior, recovery |
| Destroy GitLab, restore from backup | platform recovery procedure |

#### Phase 14–16 — AWS + Terraform
| Drill | What to Learn |
|---|---|
| Manually edit Terraform state | state lock importance, drift detection |
| `terraform destroy` wrong resource | `prevent_destroy`, targeted destroy |
| Drain EKS node group | cluster autoscaler, pod rescheduling |
| Remove IAM permission from node role | pod auth failure debugging (IRSA) |

---

## 🚀 Phase Plan

### 🟦 PART 1 — Core Infrastructure (Manual)

| Phase | Focus | Key Deliverable | Status |
|---|---|---|---|
| **0** | Proxmox + Networking | vmbr1 bridge, NAT masquerade, Tailscale subnet router | ✅ Done |
| **1** | VM Baseline | Ubuntu template, SSH hardening, static IPs, sysctl | ✅ Done |
| **2** | Kubernetes | kubeadm init, Calico CNI, all nodes Ready | ✅ Done |
| **2.5** | etcd Backup | Cron snapshot, restore drill, integrity verify | ✅ Done |
| **3** | Access + Storage | MetalLB L2, ingress-nginx, Longhorn 2-replica, PVC test | ✅ Done |
| **4** | Security | cert-manager + HTTPS, NetworkPolicy, PSS, LimitRange, PDB, RBAC, ESO install | ✅ Done |

### 🟩 PART 2 — Platform Layer (Manual)

| Phase | Focus | Status |
|---|---|---|
| **5** | GitLab CE + Runner + Kaniko + Trivy + Cosign + custom Dockerfiles | ✅ Done (apps-source image built: `registry.gitlab.lab/root/apps-source:ba4a6dd3`) |
| **6** | ArgoCD app-of-apps + two-repo GitOps + signature verification | 🔄 In progress (apps-values repo done, **GitLab Deploy Token + ArgoCD Application pending**) |
| **7** | External Secrets Operator (local backend → SSM in Phase 15) | ⏳ Upcoming |

### 🟨 PART 3 — Observability (Manual)

| Phase | Focus | Status |
|---|---|---|
| **8** | Prometheus + Alertmanager (Slack) + Grafana + kube-state-metrics | ⏳ Upcoming |
| **9** | Loki single-binary + Promtail DaemonSet | ⏳ Upcoming |
| **10** | Jaeger (OTLP receiver `:4317`) + Grafana datasource | ⏳ Upcoming |
| **11** | OTel Collector gateway + DORA metrics dashboard | ⏳ Upcoming |

### 🟧 PART 4 — Workload + HPA + Backup (Manual)

| Phase | Focus | Status |
|---|---|---|
| **12a** | OTel Demo core (frontend, productcatalog, cart, checkout, payment, shipping, currency) — raw manifests → Helm | ⏳ Upcoming |
| **12b** | OTel Demo extended (recommendation, ad, email, accounting, frauddetection, etc.) + HPA on high-traffic services | ⏳ Upcoming |
| **13** | Velero + full namespace restore drill + RTO documented | ⏳ Upcoming |

#### Phase 12 Execution Order

```
Phase 12a:
  12a-docker  → Dockerfile rewrite (core 8 services) — build + Trivy scan + Cosign sign
  12a-raw     → Raw manifest deploy (no Helm) — learn K8s primitives
  12a-helm    → Helm migration — abstraction value vs raw manifests

Phase 12b:
  12b-docker  → Dockerfile rewrite (extended services)
  12b-raw     → Raw manifest deploy
  12b-helm    → Helm (full workload) + HPA config
```

#### HPA Example (Phase 12b)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: otel-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Apply HPA to: `frontend`, `productcatalogservice`, `checkoutservice`. Observe scaling events with loadgenerator running.

### 🟥 PART 5 — AWS + Terraform

| Phase | Focus |
|---|---|
| **14** | Terraform foundation — bootstrap stack (S3 + DynamoDB local state → migrate to S3 backend), module structure |
| **15** | AWS core services — VPC (3 AZ), Route53, SSM Parameter Store, S3 backup archive, IAM (all via Terraform) |
| **16** | EKS migration — EKS + node groups, ALB Controller, ESO→SSM, OTel Demo redeploy via same ArgoCD repos, comparison doc, `terraform destroy` |

### 🟪 PART 6 — AI Platform Extension

| Phase | Focus |
|---|---|
| **17** | LiteLLM gateway (Claude, OpenAI backends, future local model) |
| **18** | LiteLLM metrics + Langfuse tracing + Grafana cost/latency dashboard |
| **19** | Prompt registry — Git-versioned prompts, ArgoCD sync to K8s ConfigMap |
| **20** | Evaluation harness — Promptfoo/Ragas in GitLab CI |
| **21** | RAG pipeline — pgvector + ingestion CronJob + retrieval FastAPI + eval-validated |
| **22** | Guardrails — prompt injection detection, PII filter, output validation |

---

## 🏗️ Terraform Strategy (Phase 14+)

### What Gets Terraform'd

| Resource | Terraform? | Reason |
|---|---|---|
| Proxmox VMs | No | Manual for learning |
| K8s cluster (homelab) | No | kubeadm manual |
| Helm charts (homelab) | No | Manual values |
| AWS VPC, subnets, SGs | Yes | Cloud primitive |
| AWS S3, Route53, SSM | Yes | Cloud services |
| AWS IAM | Yes | Security critical |
| EKS cluster + node groups | Yes | Full lifecycle |
| EKS add-ons (ALB, ESO) | Yes | `helm_release` via TF |
| App workloads on EKS | ArgoCD | GitOps-managed |

### Remote State Bootstrap Pattern

```
Step 1 — Bootstrap (local state, no backend):
  cd bootstrap/
  terraform init       # uses local .tfstate
  terraform apply
  # outputs: s3 bucket, dynamodb table

Step 2 — Main env (S3 backend):
  cd envs/lab/
  # backend.tf references bootstrap outputs
  terraform init       # migrates local state to S3
  terraform plan
```

### Repo Structure

```
mohiuddin-homelab-terraform/
├── bootstrap/              # S3 + DynamoDB (local state, run once)
│   └── main.tf
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── route53/
│   └── secrets/
├── envs/
│   └── lab/
│       ├── backend.tf
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
└── .gitlab-ci.yml          # plan on MR, apply on main (manual gate)
```

### CI/CD for Terraform

- MR → `terraform plan` (output visible in MR)
- Main branch merge → manual `terraform apply` gate (not auto)
- `terraform destroy` = separate manual pipeline stage, never auto

### Cost Discipline

- EKS ~$73/month + EC2 → destroy after Phase 16
- NAT Gateway ~$33/month → single AZ for practice
- Default rule: Phase 16 done = `terraform destroy` same day

---

## 🤖 AI Platform Design (Part 6)

### Gateway Architecture

```
Application / Service
   │
   ▼
LiteLLM Gateway (K8s Deployment)
   │
   ├── Claude API (Anthropic)
   ├── OpenAI API
   └── Local model (Ollama / vLLM — Phase 23+)
   │
   ▼
Response
   │
   ▼
Langfuse (Tracing + Cost tracking) ← Phase 18
   │
   ▼
Grafana Dashboard (cost per request, latency P95, error rate)
```

### RAG Pipeline

```
Source documents (homelab runbooks, personal notes, ADRs)
   │
   ▼
Ingestion CronJob (chunking + cleaning)
   │
   ▼
Embedding Job (sentence-transformers or OpenAI embeddings)
   │
   ▼
pgvector (Postgres in-cluster)
   │
   ▼
Retrieval FastAPI (K8s Deployment)
   │
   ▼
LiteLLM → LLM response
   │
   ▼
Eval harness (Ragas / Promptfoo) → recall, faithfulness, latency
```

### GPU Strategy

Phase 17–22: API-only (Claude, OpenAI). No GPU required.

Trigger for local inference (Phase 23+):
- Sustained API cost increase
- Offline / private inference needed
- Fine-tuning experiments

GPU upgrade (e.g., RTX 3090 24GB) deferred until cost-justified.

---

## ☁️ AWS Service Map (Phase 14–16)

| Service | Purpose | Phase |
|---|---|---|
| S3 (tfstate) | Terraform remote backend | 14 |
| DynamoDB | State lock | 14 |
| IAM | Terraform execution role | 14 |
| VPC | 3-AZ network for EKS | 15 |
| Route53 | DNS | 15 |
| SSM Parameter Store | Secrets backend for ESO | 15 |
| S3 (archive) | Offsite backup target | 15 |
| EKS | Managed K8s comparison | 16 |
| ALB Controller | Ingress on EKS (vs ingress-nginx) | 16 |

---

## ⏱️ Realistic Timeline

| Part | Phase Range | Focus | Duration |
|---|---|---|---|
| Part 1 | Phase 0–4 | Core Infrastructure | ~2 months |
| Part 2 | Phase 5–7 | Platform Layer | ~1.5 months |
| Part 3 | Phase 8–11 | Full Observability | ~1.5 months |
| Part 4 | Phase 12–13 | Workload + HPA + Backup | ~2–3 months |
| Part 5 | Phase 14–16 | AWS + Terraform + EKS | ~1.5–2 months |
| Part 6 | Phase 17–22 | AI Platform | ~3–4 months |
| **Total** | | | **~12–15 months** |

**Biggest risk:** execution drop, scope creep, phase skip. Not design quality.

---

## 📝 Architecture Decision Records

| ADR | Decision | Reason |
|---|---|---|
| 001 | Calico over Flannel | NetworkPolicy enforcement required |
| 002 | Longhorn over local-path | Replication + snapshots + Velero CSI integration |
| 003 | GitLab over Gitea | Full CI/CD + production parity (runner, registry, SAST, MR gates) |
| 004 | Jaeger over Tempo | Industry-recognized; OTLP receiver (not legacy Thrift) |
| 005 | External Secrets Operator | Multi-backend path: local → AWS SSM without app changes |
| 006 | kubeadm before EKS | Control plane internals before managed abstraction |
| 007 | Single control plane | Lab resource tradeoff; etcd snapshot mitigates |
| 008 | Two-repo GitOps | Clean code/deploy separation; rollback = values revert |
| 009 | Custom Dockerfiles for all OTel Demo services | Build chain ownership + Trivy scan coverage |
| 010 | Raw manifests before Helm migration | K8s primitives understanding before abstraction |
| 011 | Kaniko over Docker-in-Docker | Privilegeless builds, production-safe |
| 012 | Gateway-only OTel Collector (initial) | Simpler debugging; agent+gateway deferred |
| 013 | 2-worker cluster, ingress co-located on md-w-2 | Resource budget; production parity demonstrated on EKS |
| 014 | Velero over manual kubectl cp | Application-consistent snapshots, namespace restore |
| 015 | Cosign image signing | Supply chain security; signed-only deploys |
| 016 | Semver + SHA tags | Human-readable deployment history |
| 017 | LimitRange per namespace | Prevents unbounded pod resource consumption |

---

## 🎯 Phase Pass Criteria

A phase is NOT complete until ALL of the following are done:

- ✅ Tool deployed and working
- ✅ Verification commands run and output documented
- ✅ Break-it drill executed and recovery time measured
- ✅ Runbook written in professional English → pushed to GitHub (`docs/runbooks/`)
- ✅ Interview Q&A written (English on GitHub, Bangla for personal study)
- ✅ ADR written if architectural decision made
- ✅ Screenshot or terminal output captured for portfolio

**No shortcuts. No phase skipping. Every drill mandatory.**

---

## ⚠️ Known Tradeoffs — Interview Ready

| Tradeoff | Reason | Production Equivalent |
|---|---|---|
| Single control plane | Lab resource constraint | 3-node etcd + 2+ control planes behind LB |
| Ingress co-located on md-w-2 | No dedicated ingress node | Dedicated ingress NodePool (EKS Phase 16) |
| 2-replica Longhorn | Resource pressure | 3-replica (requires 3+ workers) |
| Self-signed Root CA | No public domain | Let's Encrypt + real domain |
| md-svc-1 DNS SPOF | Lab simplicity | CoreDNS redundancy or managed DNS |
| GitLab single instance | Resource budget | GitLab HA or GitLab.com |
| In-cluster observability | Simplicity | External Prometheus + alerting for cluster-down case |
| Tailscale single router | Single Proxmox host | Redundant subnet routers |

---

## 📁 GitHub Repository Structure

```
mohiuddin-homelab/
├── README.md                     ← Architecture overview + screenshots + "what I learned"
├── ARCHITECTURE.md               ← This document
├── docs/
│   ├── runbooks/
│   │   ├── phase-0-prereq.md
│   │   ├── phase-1-kubeadm.md
│   │   ├── phase-2.5-etcd-backup.md
│   │   └── failures/             ← Break-it drill outcomes + RTO logs
│   ├── decisions/                ← ADRs 001–017
│   ├── interview-prep/           ← Q&A per phase (English + Bangla)
│   └── diagrams/                 ← PNG/SVG architecture diagrams
├── manifests/                    ← Raw K8s YAMLs (Phase 12 raw layer)
├── helm-values/                  ← Per-chart values.yaml
├── apps-values/                  ← GitOps source (ArgoCD watches)
│   ├── otel-demo/
│   ├── observability/
│   └── platform/
├── dockerfiles/                  ← Custom Dockerfiles per OTel Demo service
└── terraform/                    ← Phase 14+ (bootstrap/ + envs/lab/)
```

---

## 🏁 Final Verdict

### What This Architecture Demonstrates

- **End-to-end production workflow:** git push → signed image → GitOps sync → running pod → traced + logged + alerted
- **Security depth:** NetworkPolicy + PSS + LimitRange + PDB + Cosign + ESO + TLS everywhere
- **Observability maturity:** metrics + logs + traces correlated in Grafana + Alertmanager routing + DORA metrics
- **Infrastructure discipline:** manual first → IaC when justified → same workload on kubeadm and EKS for comparison
- **Recovery culture:** every phase has break-it drills + runbooks + RTO measurements

### Execution Rules (Non-Negotiable)

1. One phase complete before next begins
2. Phase 0–13: homelab execution = manual. Terraform learning allowed (non-production isolated practice)
3. Phase 14+: AWS provisioning = Terraform only. Allowed manual exceptions: bootstrap stack + emergency recovery
4. Every phase ends with break-it drill
5. Every failure → runbook committed to Git
6. Overbuilding = enemy
7. Phase skip = plan broken

---

*Document v3.0 · Production-grade homelab — debugging muscle built drill by drill.*
*Built by Mohiuddin — DevOps / Platform Engineer candidate.*
