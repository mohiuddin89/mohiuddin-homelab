# ADR 008 — Two-Repo GitOps Pattern

## Status
Accepted

## Context
GitOps requires ArgoCD to watch a Git repository for desired
cluster state. Decision: one repo for everything, or separate
repos for application source code and deployment config.

## Decision
Use **two separate repos**:
- `mohiuddin-otel-homelab` — application source code + CI
- `apps-values` — Kubernetes manifests + Helm values (ArgoCD
  watches this only)

## Reasons
- Source code changes and deployment changes have different
  lifecycles and different audiences
- ArgoCD watches only deployment config — not application
  source code. Every source commit does not trigger a sync
- Rollback is simple: revert one commit in apps-values,
  ArgoCD re-syncs automatically
- CI pipeline updates apps-values after successful build —
  clean automated handoff from CI to GitOps
- Different access controls: developers push to source repo,
  ops controls apps-values
- This is how real GitOps teams operate in production

## Flow
