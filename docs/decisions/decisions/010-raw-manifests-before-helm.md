# ADR 010 — Raw Manifests Before Helm Migration

## Status
Accepted

## Context
Phase 12 deploys the OTel Demo workload to Kubernetes.
Decision: deploy directly with Helm, or start with raw
Kubernetes manifests then migrate to Helm.

## Decision
Deploy with **raw manifests first** (12a-raw, 12b-raw),
then migrate to **Helm** (12a-helm, 12b-helm).

## Reasons
- Raw manifests expose every Kubernetes primitive directly:
  Deployment, Service, ConfigMap, PVC, ServiceAccount,
  NetworkPolicy, HPA — no abstraction layer hiding anything
- Writing manifests by hand builds muscle memory for what
  Helm generates under the hood
- Debugging raw manifests is simpler — no Helm templating
  layer, no chart values to trace through
- After raw deploy works, Helm migration shows exactly what
  Helm abstracts — the contrast is the learning
- Doing Helm first loses this contrast entirely
- Production engineers must read and write both

## Execution Order
