# ADR 007 — Single Control Plane

## Status
Accepted

## Context
Production Kubernetes requires HA control plane (3+ etcd nodes,
2+ control plane nodes). Lab hardware has resource constraints
with 5 VMs total.

## Decision
Use a **single control plane node** (md-cp-1) for the homelab.

## Reasons
- 3-node etcd + 2 control planes would require 3 additional VMs
- Lab budget: 5 VMs total, each with a dedicated role
- Single control plane exposes the same components as HA —
  just without redundancy. Learning value is identical
- etcd snapshot every 6 hours (Phase 1.5) mitigates data loss
- Full restore drill documented with measured RTO

## Consequences
- md-cp-1 down = cluster API unavailable (no HA)
- All workloads continue running on workers during control
  plane downtime — only API calls are blocked
- etcd backup is mandatory, not optional
- RTO from etcd restore: ~10 minutes (documented in phase-1.md)
- Production equivalent: 3-node etcd + 2+ control planes
  behind a load balancer

## Alternatives Rejected
- **3-node HA control plane** — requires 3 additional VMs,
  exceeds lab resource budget, no additional learning value
  for the skills being targeted
