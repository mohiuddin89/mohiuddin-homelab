# ADR 005 — External Secrets Operator over Sealed Secrets

## Status
Accepted

## Context
Project requires secrets management that is safe to commit to Git
and supports multiple backends as the project evolves from homelab
to AWS cloud (Phase 15).

## Decision
Use **External Secrets Operator (ESO)** for secrets management.

## Reasons
- ESO supports multiple backends without changing application config:
  Phase 7 = local Kubernetes secret store,
  Phase 15 = AWS SSM Parameter Store
- No plaintext secrets in Git — ESO fetches from backend at runtime
- Automatic secret rotation when backend value changes
- Production-standard pattern used in real cloud-native teams
- Clean migration: same ExternalSecret CRD, only SecretStore changes

## Consequences
- Phase 7: local Kubernetes secret store backend (no external dependency)
- Phase 15: AWS SSM Parameter Store (no app changes required)
- ESO controller must be running in cluster
- SecretStore and ExternalSecret CRDs created per namespace

## Alternatives Rejected
- **Sealed Secrets** — no multi-backend path, manual re-sealing on
  key rotation, does not integrate with AWS SSM
- **Plaintext in Git** — never acceptable
