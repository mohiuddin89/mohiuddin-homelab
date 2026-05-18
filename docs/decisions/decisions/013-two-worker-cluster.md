# ADR 013 — Two-Worker Cluster, Ingress Co-located on md-w-2

## Status
Accepted

## Context
Cluster needs worker nodes for app workloads and an ingress
controller. Options: dedicated ingress node, or co-locate
ingress on an existing worker node.

## Decision
Use **2 worker nodes** (md-w-1, md-w-2) with ingress-nginx
co-located on md-w-2 via nodeSelector.

## Reasons
- Lab budget: 5 VMs total with dedicated roles
- A 3rd dedicated ingress worker reduces app workload capacity
- Longhorn requires minimum 2 workers for 2-replica storage
- ingress-nginx is scheduled on md-w-2 via nodeSelector —
  not random placement
- podAntiAffinity rules keep memory-heavy app pods off md-w-2
  where possible
- This constraint is documented, not hidden — intentional
  tradeoff with a known production equivalent
- EKS Phase 16 demonstrates the dedicated node pool pattern

## Consequences
- md-w-2 saturation can impact both app pods and ingress
- Monitor ingress-nginx resource usage from Phase 8 onward
- Longhorn 2-replica only (not default 3)
- Production equivalent: dedicated ingress NodePool + 3+
  app worker nodes (demonstrated on EKS in Phase 16)

## Alternatives Rejected
- **3rd dedicated ingress worker** — reduces total app
  workload capacity, exceeds lab VM budget
- **Ingress on control plane** — control plane should never
  run application workloads
