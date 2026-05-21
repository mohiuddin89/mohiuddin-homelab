# Phase 5 — GitLab CI: Kaniko + Trivy + 6-Stage Pipeline

> **Status:** Complete  
> **VM:** md-ci-1 (10.20.0.30, 8GB RAM)  
> **GitLab repo:** mohiuddin-otel-homelab  
> **Pipeline:** lint → build → scan → size-check → health-test → push

---

## Why This Phase Matters

Phase 5 establishes the CI foundation for all future workload deployments. Every image that reaches the cluster must pass through this pipeline — linted, built without Docker privileges, scanned for CVEs, size-checked, health-tested, and only then pushed to the registry. This is the security gate between code and cluster.

---

## Architecture

```
Developer (git push)
    │
    ▼
GitLab (md-ci-1: 10.20.0.30)
    │
    ▼
GitLab Runner (docker executor, privileged=true, socket binding)
    │
    ├── lint (hadolint + language linter)
    ├── build (Kaniko — privilegeless image build)
    ├── scan (Trivy — HIGH/CRITICAL = fail)
    ├── size-check (image size enforcement)
    ├── health-test (docker run via socket binding)
    └── push (Kaniko — push to GitLab registry)
    │
    ▼
GitLab Container Registry (registry.gitlab.lab)
```

---

## Services Configured

| Service | Language | Base Image | Registry Tag |
|---|---|---|---|
| frontend | Next.js (TypeScript) | node:22-alpine | `frontend:<sha>` |
| productcatalog | Go 1.25 | golang:1.25-alpine + distroless/static:nonroot | `productcatalog:<sha>` |

---

## Pipeline Stages

### Stage 1 — lint

**Jobs:** `frontend:hadolint`, `frontend:lint-node`, `product-catalog:hadolint`, `product-catalog:lint-go`

- hadolint enforces Dockerfile best practices (DL3018, DL3006, etc.)
- eslint for TypeScript, golangci-lint for Go
- Fails fast before expensive build stage

### Stage 2 — build

**Jobs:** `frontend:build`, `product-catalog:build`

- Kaniko executor (`gcr.io/kaniko-project/executor:debug`)
- `--no-push --tar-path $CI_PROJECT_DIR/<service>.tar` — builds image to tar, no registry push yet
- `--skip-tls-verify` — lab self-signed cert
- Artifact uploaded to GitLab for downstream stages

**Why Kaniko, not Docker-in-Docker:**
- DinD requires `privileged: true` and kernel namespace access
- Kaniko builds in userspace — no Docker daemon, no root required
- Production security teams reject DinD for exactly this reason

### Stage 3 — scan

**Jobs:** `frontend:scan`, `product-catalog:scan`

- Trivy scans the tar artifact from build stage
- `--exit-code 1 --severity HIGH,CRITICAL` — pipeline fails on any HIGH or CRITICAL CVE
- `--ignorefile .trivyignore` — documented exceptions only (transitive deps, justified)
- Known CVEs documented in `.trivyignore` with justification and tracking links

**CVEs encountered and resolved:**

| CVE | Package | Resolution |
|---|---|---|
| CVE-2026-31789 | libssl3 (debian) | Switched runtime from distroless/debian to node:22-alpine |
| CVE-2026-28387/88/89/90 | libssl3 (debian) | Same — alpine has no debian packages |
| CVE-2026-44289/90/91/93 | protobufjs (transitive) | Added to .trivyignore — OTel upstream dependency |
| CVE-2026-33671 | picomatch (npm internal) | Added to .trivyignore — npm tooling, not app runtime |

### Stage 4 — size-check

**Jobs:** `frontend:size-check`, `product-catalog:size-check`

- Frontend: `du -m` on tar → fail if > 500MB
- Product catalog: fail if > 100MB
- Prevents accidentally bloated images from reaching registry

### Stage 5 — health-test

**Jobs:** `frontend:health-test`, `product-catalog:health-test`

- `docker load` the tar artifact
- `docker run -d` with required env vars
- `sleep` + `docker logs` + `docker stop`
- Proves the image actually starts — not just that it builds

**Docker socket binding:**
- Runner uses `DOCKER_HOST: unix:///var/run/docker.sock`
- Host Docker daemon on md-ci-1 used directly
- `volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]` in config.toml
- Pre-run cleanup: `docker rm -f <name> 2>/dev/null || true` prevents container name conflicts

### Stage 6 — push

**Jobs:** `frontend:push`, `product-catalog:push`

- Kaniko re-builds and pushes to `registry.gitlab.lab`
- Image tag: `$CI_COMMIT_SHORT_SHA` (e.g., `frontend:12407e1d`)
- `--skip-tls-verify` — lab self-signed registry cert

---

## GitLab Runner Configuration

**File:** `/etc/gitlab-runner/config.toml`

```toml
[runners.docker]
  extra_hosts = ["gitlab.lab:10.20.0.30", "registry.gitlab.lab:10.20.0.30"]
  tls_verify = false
  image = "alpine:latest"
  privileged = true
  volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
```

**Why privileged = true:**
- Required for Docker socket binding to function
- Acceptable on dedicated CI VM (md-ci-1)
- Not used on Kubernetes worker nodes

---

## DNS Configuration

md-ci-1 `/etc/hosts` requires:
```
10.20.0.30 gitlab.lab registry.gitlab.lab
```

**Important:** This entry is lost on VM reboot. Re-add if pipeline shows DNS errors:
```bash
echo "10.20.0.30 gitlab.lab registry.gitlab.lab" | sudo tee -a /etc/hosts
```

GitLab nginx artifact size limit — `/etc/gitlab/gitlab.rb`:
```ruby
nginx['client_max_body_size'] = '2g'
```

Apply with: `sudo gitlab-ctl reconfigure`

---

## Dockerfile Standards

All service Dockerfiles follow:

1. **Multi-stage build** — builder stage separate from runtime stage
2. **Alpine or distroless runtime** — no debian-based runtime (CVE surface reduction)
3. **Non-root user** — `USER node` (frontend) or `USER nonroot:nonroot` (productcatalog)
4. **No apk/apt installs in runtime stage** — runtime layer is immutable
5. **EXPOSE** — port documented in Dockerfile

**Frontend (node:22-alpine):**
- Builder: npm ci + next build
- Deps: npm ci --omit=dev
- Runtime: node:22-alpine, USER node, port 8080

**Product catalog (golang:1.25-alpine → distroless/static:nonroot):**
- Builder: go mod download + CGO_ENABLED=0 go build
- Runtime: distroless/static:nonroot, USER nonroot:nonroot, port 3550

---

## Validation Commands

```bash
# Verify GitLab Runner is running
sudo gitlab-runner status

# Verify Docker socket binding
sudo grep "volumes" /etc/gitlab-runner/config.toml

# Verify nginx artifact size limit
sudo grep "client_max_body_size" /var/opt/gitlab/nginx/conf/service_conf/gitlab-rails.conf

# Verify images in registry (via GitLab UI)
# https://gitlab.lab/root/mohiuddin-otel-homelab/container_registry

# Pull and inspect a built image
docker pull registry.gitlab.lab/root/mohiuddin-otel-homelab/frontend:<sha>
docker inspect registry.gitlab.lab/root/mohiuddin-otel-homelab/frontend:<sha>
```

---

## Break-it Drills

### Drill 1 — Runner Offline

**Procedure:**
```bash
sudo gitlab-runner stop
# Push a commit
git commit --allow-empty -m "drill: runner offline"
git push origin main
```

**Observation:** Pipeline immediately enters `Pending` state. All 14 jobs queued but not executing. GitLab UI shows yellow `Pending` badge.

**Recovery:**
```bash
sudo gitlab-runner start
```

**Result:** Pipeline resumed and completed normally.

**Metrics:**
- Queue time (runner offline): 889 seconds (~15 minutes)
- Pipeline execution time after recovery: 10 minutes 55 seconds
- RTO: ~26 minutes end-to-end

**Production alert recommendation:** Alert if pipeline pending > 10 minutes — indicates runner health issue.

### Drill 2 — Bad Dockerfile / Trivy Gate Validation

**Background:** This drill was executed organically during development when the runtime base image contained HIGH and CRITICAL CVEs.

**Observation:**
- Runtime stage using `gcr.io/distroless/nodejs22-debian12:nonroot` (Debian-based)
- Trivy detected: `libssl3` CRITICAL CVE-2026-31789, multiple HIGH CVEs
- `frontend:scan` job failed with exit code 1
- Pipeline blocked at scan stage — push job never triggered
- Image never reached registry

**Recovery:**
- Switched runtime from `distroless/nodejs22-debian12` to `node:22-alpine`
- Alpine has no Debian libssl3 packages — CVEs eliminated at source
- Documented transitive protobufjs CVEs in `.trivyignore` with justification

**Key learning:** Trivy gate works as designed. A vulnerable image cannot reach the registry. The fix is to upgrade the base image, not to bypass the gate.

---

## Troubleshooting

### Pipeline stuck in Pending
```
Cause: GitLab Runner not running
Fix: sudo gitlab-runner start
Verify: sudo gitlab-runner status
```

### Artifact upload 413 Request Entity Too Large
```
Cause: GitLab nginx default body size limit
Fix: nginx['client_max_body_size'] = '2g' in /etc/gitlab/gitlab.rb
Apply: sudo gitlab-ctl reconfigure
```

### Cannot connect to Docker daemon
```
Cause: Docker socket not mounted in runner
Fix: Add volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
     to [runners.docker] in /etc/gitlab-runner/config.toml
Apply: sudo gitlab-runner restart
```

### Container name already in use
```
Cause: Previous health-test container not cleaned up
Fix: docker rm -f frontend-test productcatalog-test 2>/dev/null
     Add cleanup line before docker run in health-test job
```

### DNS resolution failed for gitlab.lab
```
Cause: /etc/hosts entry lost after VM reboot
Fix: echo "10.20.0.30 gitlab.lab registry.gitlab.lab" | sudo tee -a /etc/hosts
```

### Trivy scan blocked by CVEs
```
Cause: HIGH/CRITICAL CVEs in image
Fix options:
  1. Upgrade base image to patched version
  2. Remove vulnerable package if not needed
  3. Document in .trivyignore with justification (last resort)
```

---

## Phase 5 Pass Criteria — Checklist

- [x] GitLab CE running on md-ci-1
- [x] GitLab Runner registered with docker executor
- [x] Custom Dockerfile for frontend (multi-stage, alpine, non-root)
- [x] Custom Dockerfile for productcatalog (multi-stage, distroless, non-root)
- [x] 6-stage pipeline: lint → build → scan → size-check → health-test → push
- [x] Kaniko builds (privilegeless) working
- [x] Trivy CVE gate (HIGH/CRITICAL blocks push) verified
- [x] Images pushed to GitLab Container Registry
- [x] Pipeline #38 all 6 stages passed
- [x] Drill 1: Runner offline — RTO documented (~26 minutes)
- [x] Drill 2: Trivy CVE gate — verified blocking behavior

---

*Phase 5 complete. Next: Phase 6 — ArgoCD GitOps two-repo pattern.*
