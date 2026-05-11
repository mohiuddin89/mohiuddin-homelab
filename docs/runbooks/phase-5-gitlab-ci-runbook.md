# Phase 5 — GitLab CI + Kaniko Build Pipeline

> **Runbook version:** 1.0  
> **Status:** Complete  
> **Environment:** Proxmox homelab — md-ci-1 (10.20.0.30), Ubuntu 22.04  
> **Scope:** GitLab CE installation, Runner configuration, Kaniko image build, Container Registry push  

---

## Why This Phase Matters

In production environments, every code change must pass through a gated CI pipeline before it ever reaches a container registry or a Kubernetes cluster. This phase establishes that gate:

- **GitLab CE** acts as the source of truth for code and CI configuration
- **GitLab Runner** executes pipeline jobs in isolation
- **Kaniko** builds container images without requiring Docker daemon access or privileged containers — a hard security requirement in most production clusters
- **GitLab Container Registry** stores versioned, immutable image artifacts

Without this foundation, GitOps (Phase 6) cannot function — ArgoCD has nothing to pull if images are not built and pushed by a trusted pipeline.

---

## Architecture Overview

```
Developer (git push)
        │
        ▼
GitLab CE (md-ci-1, 10.20.0.30)
        │
        ▼
GitLab Runner (docker executor)
        │
        ├── Stage: test
        │     └── test-job (alpine container)
        │
        └── Stage: build
              └── build-job (gcr.io/kaniko-project/executor:debug)
                    └── Push → registry.gitlab.lab/root/<project>:<commit-sha>
```

**Key design decision:** The Runner uses the **docker executor**, not the shell executor. This means each job runs inside an isolated container — the job cannot access or modify the host system. This mirrors production CI patterns where runners are ephemeral and sandboxed.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| VM | md-ci-1, Ubuntu 22.04, 4+ GB RAM, 40 GB disk |
| Network | 10.20.0.0/24, static IP 10.20.0.30 |
| TLS | Self-signed certificate with SAN for `gitlab.lab` and `registry.gitlab.lab` |
| DNS | `/etc/hosts` entries on all nodes that need to reach GitLab |
| GitLab CE | Installed and accessible at `https://gitlab.lab` |

---

## Step 1 — Verify GitLab CE is Running

**Why:** Before configuring the runner, confirm GitLab itself is healthy. A degraded GitLab will cause misleading runner errors.

```bash
sudo gitlab-ctl status
```

**Expected output:** All services (nginx, puma, postgresql, redis, gitaly) show `run`.

```bash
# Verify HTTPS is working
curl -k https://10.20.0.30/api/v4/version
```

**Expected output:** JSON with `version` and `revision` fields.

---

## Step 2 — Verify /etc/hosts on md-ci-1

**Why:** The lab has no DNS server. GitLab Runner and Kaniko both make HTTPS calls to `gitlab.lab` and `registry.gitlab.lab`. Without `/etc/hosts` entries, these calls fail with `no such host`.

```bash
cat /etc/hosts | grep gitlab
```

**Expected output:**
```
10.20.0.30 gitlab.lab
10.20.0.30 registry.gitlab.lab
```

If missing, add them:
```bash
echo "10.20.0.30 gitlab.lab" | sudo tee -a /etc/hosts
echo "10.20.0.30 registry.gitlab.lab" | sudo tee -a /etc/hosts
```

---

## Step 3 — Install Docker Engine

**Why:** The GitLab Runner docker executor requires Docker to spawn job containers. Without Docker, the runner can only use the shell executor — which does not support the `image:` keyword in `.gitlab-ci.yml` and cannot run Kaniko.

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Verify:
```bash
sudo docker version
```

**Expected output:** Client and Server both showing version 29.x or later.

Add the `gitlab-runner` user to the `docker` group so the runner can control Docker without `sudo`:

```bash
sudo usermod -aG docker gitlab-runner
```

Verify the runner can reach Docker:
```bash
sudo -u gitlab-runner docker info | head -5
```

**Expected output:** Docker client information without permission errors.

---

## Step 4 — Register GitLab Runner with Docker Executor

**Why:** The runner must be registered against the GitLab instance with the correct executor type. The executor type determines how job isolation works. Docker executor = each job runs in a fresh container. Shell executor = jobs run directly on the host OS (insecure, not production-grade).

**Important (GitLab 17+):** The new runner authentication token workflow does not accept `--tag-list`, `--locked`, or `--run-untagged` at registration time. These are configured in the GitLab UI after registration.

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.lab" \
  --token "<RUNNER_AUTHENTICATION_TOKEN>" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "lab-runner" \
  --tls-ca-file "/etc/gitlab/ssl/gitlab.lab.crt"
```

Verify registration:
```bash
sudo gitlab-runner verify
```

**Expected output:** `Verifying runner... is valid`

---

## Step 5 — Configure extra_hosts in Runner Config

**Why:** When the Runner spawns a job container (e.g., the Kaniko container), that container runs in Docker's bridge network. It does not inherit the host's `/etc/hosts`. Without explicit host injection, Kaniko cannot resolve `gitlab.lab` or `registry.gitlab.lab` — causing `no such host` errors during image push.

Edit `/etc/gitlab-runner/config.toml`:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

Add `extra_hosts` inside the `[runners.docker]` block:

```toml
[runners.docker]
  tls_verify = false
  image = "alpine:latest"
  extra_hosts = ["gitlab.lab:10.20.0.30", "registry.gitlab.lab:10.20.0.30"]
  privileged = false
  disable_entrypoint_overwrite = false
  oom_kill_disable = false
  disable_cache = false
  volumes = ["/cache"]
  shm_size = 0
```

Restart the runner to apply:
```bash
sudo gitlab-runner restart
```

Verify the config loaded:
```bash
sudo gitlab-runner verify
```

---

## Step 6 — Verify Instance Runner is Enabled for the Project

**Why:** GitLab instance runners are not automatically available to all projects. Each project must explicitly opt in. If this toggle is off, jobs will sit in `Pending` state indefinitely with the message "this job is stuck because the project doesn't have any runners."

Navigate to:
```
https://gitlab.lab/<username>/<project>/-/settings/ci_cd
```

Under **Runners → Instance → Turn on instance runners for this project** — ensure the toggle is **ON**.

---

## Step 7 — .gitlab-ci.yml Structure

**Why this structure:** The `test` stage runs first and acts as a quality gate. If tests fail, the `build` stage is skipped entirely — no broken images are ever pushed to the registry. This is a core CI principle: fail fast, fail early.

```yaml
stages:
  - test
  - build

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

test-job:
  stage: test
  tags:
    - lab
  script:
    - echo "Running tests..."
    - echo "Test passed!"
    - date

build-job:
  stage: build
  tags:
    - lab
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor
      --context $CI_PROJECT_DIR
      --dockerfile $CI_PROJECT_DIR/Dockerfile
      --destination $IMAGE_TAG
      --skip-tls-verify
```

**Why Kaniko instead of `docker build`:**
- Kaniko runs as a standard container — no Docker socket mount required
- No `privileged: true` needed — reduces attack surface
- Industry standard for in-cluster image builds (used in most production GitLab/Kubernetes setups)

**Why `--skip-tls-verify`:**
- Lab uses a self-signed certificate
- In production, this flag is removed and the CA cert is properly trusted

**Why `$CI_COMMIT_SHORT_SHA` as the image tag:**
- Every commit produces a unique, traceable image tag
- Enables precise rollback — you always know exactly which commit produced which image
- ArgoCD in Phase 6 will use this tag to track deployments

---

## Step 8 — Validation Checklist

Run these after every pipeline execution:

```bash
# 1. Confirm runner is online
sudo gitlab-runner verify

# 2. Confirm runner service is running
sudo gitlab-runner status

# 3. Check recent pipeline logs
# → GitLab UI: CI/CD → Pipelines → latest pipeline → both jobs green

# 4. Confirm image exists in registry
# → GitLab UI: Packages & Registries → Container Registry
# → You should see the image tagged with the commit SHA
```

---

## Troubleshooting

### Job stuck in Pending for >5 minutes
**Symptom:** `This job is stuck because the project doesn't have any runners online assigned to it.`  
**Cause 1:** Instance runner not enabled for project → enable in project CI/CD settings  
**Cause 2:** Runner service not running → `sudo gitlab-runner status`  
**Cause 3:** Runner not registered → `sudo gitlab-runner verify`  

### `mkdir: cannot create directory '/kaniko': Permission denied`
**Symptom:** build-job fails at first line  
**Cause:** Runner is using shell executor, not docker executor  
**Fix:** Re-register runner with `--executor "docker"`  

### `dial tcp: lookup gitlab.lab: no such host` (in runner verify)
**Cause:** `/etc/hosts` on md-ci-1 missing `gitlab.lab` entry  
**Fix:** `echo "10.20.0.30 gitlab.lab" | sudo tee -a /etc/hosts`  

### `dial tcp: lookup gitlab.lab: no such host` (inside build-job container)
**Cause:** Docker containers don't inherit host `/etc/hosts`  
**Fix:** Add `extra_hosts = ["gitlab.lab:10.20.0.30", "registry.gitlab.lab:10.20.0.30"]` to `config.toml`  

### `lookup registry.gitlab.lab: no such host`
**Cause:** `extra_hosts` only has `gitlab.lab` but not `registry.gitlab.lab`  
**Fix:** Add both entries to `extra_hosts` in `config.toml`  

---

## Security Notes

| Item | Lab Setting | Production Equivalent |
|---|---|---|
| `--skip-tls-verify` | Enabled (self-signed CA) | Disabled — proper CA cert trusted |
| `privileged: false` | ✅ Enforced | ✅ Always enforced with Kaniko |
| Runner isolation | Docker container per job | Kubernetes pod per job (Phase 6+) |
| Registry auth | CI variables (auto-injected) | Sealed Secrets or Vault |
| Runner token | Stored in config.toml | Injected via secret management |

---

## Key Files Modified

| File | Location | Purpose |
|---|---|---|
| `/etc/hosts` | md-ci-1 | DNS resolution for gitlab.lab + registry.gitlab.lab |
| `/etc/gitlab-runner/config.toml` | md-ci-1 | Runner executor, docker settings, extra_hosts |
| `.gitlab-ci.yml` | Git repo root | Pipeline definition |
| `Dockerfile` | Git repo root | Image build instructions |

---

## Phase Pass Criteria

- [x] GitLab CE running and accessible via HTTPS
- [x] GitLab Runner registered with docker executor
- [x] `gitlab-runner verify` returns `is valid`
- [x] test-job passes
- [x] build-job (Kaniko) passes
- [x] Image visible in GitLab Container Registry with commit SHA tag
- [x] Troubleshooting documented (DNS, executor type, extra_hosts)

---


*Phase 5 complete. Next: Phase 6 — Full GitOps loop with two-repo pattern and ArgoCD sync.*
