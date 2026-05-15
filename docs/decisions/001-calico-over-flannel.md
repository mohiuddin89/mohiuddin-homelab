# ADR 001 — Calico over Flannel

## Status
Accepted

## Context
Kubernetes cluster requires a CNI plugin for pod networking.
Two primary options evaluated: Flannel and Calico.

## Decision
Use **Calico** as the CNI plugin.

## Reasons
- Flannel provides only basic pod networking — no NetworkPolicy support
- Calico provides both pod networking AND NetworkPolicy enforcement
- Phase 4 requires default-deny NetworkPolicy per namespace
- Without NetworkPolicy, security baseline cannot be implemented
- Calico is production-standard in on-prem and cloud-native environments

## Consequences
- Slightly higher resource usage than Flannel (~200MB vs ~50MB per node)
- More complex troubleshooting (Calico-specific commands: `calicoctl`, `kubectl describe networkpolicy`)
- Full NetworkPolicy enforcement available from Phase 4 onward
- Production skill: Calico is what real teams use

## Alternatives Rejected
- **Flannel** — no NetworkPolicy support, cannot implement Phase 4 security baseline
