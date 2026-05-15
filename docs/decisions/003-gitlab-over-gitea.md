# ADR 003 — GitLab over Gitea

## Status
Accepted

## Context
Project requires a self-hosted Git server with CI/CD capabilities,
container registry, and runner support.

## Decision
Use **GitLab CE** as the Git + CI/CD platform.

## Reasons
- GitLab CE provides: Git server, CI/CD runner, container registry,
  merge request gates, SAST scanning — all in one platform
- Gitea provides Git + basic CI but lacks production-grade CI features
- GitLab CI syntax is industry-standard — directly transferable to
  GitLab.com and mirrors GitHub Actions concepts
- GitLab Container Registry on md-ci-1 — no external registry needed
- Kaniko + Trivy integration well-documented with GitLab CI
- Interview value: "I ran GitLab CE on bare-metal" is a strong signal

## Consequences
- Higher resource usage (~4–8GB RAM vs Gitea ~300MB)
- md-ci-1 dedicated VM required (8GB RAM)
- Single instance — not HA (lab tradeoff)
- Production equivalent: GitLab HA or GitLab.com

## Alternatives Rejected
- **Gitea** — insufficient CI/CD features for production pipeline parity
- **Jenkins** — separate tool, additional complexity, not GitLab-integrated
