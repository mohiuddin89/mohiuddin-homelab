# ADR 002 — Longhorn over local-path

## Status
Accepted

## Context
Kubernetes cluster requires a StorageClass for dynamic PVC provisioning.
Two options evaluated: local-path-provisioner and Longhorn.

## Decision
Use **Longhorn** as the distributed storage solution.

## Reasons
- local-path-provisioner stores data on a single node — no replication
- If that node fails, PVC data is lost permanently
- Longhorn replicates volumes across worker nodes (2-replica configured)
- Longhorn provides snapshot, backup, and restore capabilities
- Longhorn UI gives visibility into volume health and replica status
- Production storage always has replication — local-path is a dev tool

## Consequences
- Higher resource usage than local-path (~500MB cluster-wide)
- Requires minimum 2 worker nodes for replication
- Replica count set to 2 (not default 3) — lab resource tradeoff
- Production equivalent: 3 replicas with 3+ worker nodes
- Longhorn backup target: external disk (Phase 13)

## Alternatives Rejected
- **local-path-provisioner** — no replication, node-bound PV, data loss on node failure
