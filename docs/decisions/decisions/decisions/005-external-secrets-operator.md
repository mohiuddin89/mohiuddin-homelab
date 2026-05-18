# ADR 005 — External Secrets Operator

## Status
Accepted

## Context
Project requires secrets management that is safe to use with Git
and supports multiple backends as the project grows.

## Decision
Use **External Secrets Operator (ESO)** for secrets management.

## Reasons
- ESO supports multiple backends: local Kubernetes store (Phase 7),
  AWS SSM Parameter Store (Phase 15) — same app config, backend changes
- No plaintext secrets in Git — ESO fetches from backend at runtime
- Clean migration path: homelab uses local backend, EKS phase uses AWS SSM
- Production-standard pattern: secrets live in a secret store, not in Git
- Automatic secret rotation when backend value changes

## Consequences
- Phase 7: local Kubernetes secret store backend
- Phase 15: AWS SSM Parameter Store backend (no app changes required)
- ESO controller running in cluster
- SecretStore and ExternalSecret CRDs must be created per namespace

## Alternatives Rejected
- **Sealed Secrets** — no multi-backend path, manual re-sealing required on rotation
- **Plaintext in Git** — never acceptable, security violation
