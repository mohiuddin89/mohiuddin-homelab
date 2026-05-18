# ADR 011 — Kaniko over Docker-in-Docker

## Status
Accepted

## Context
GitLab CI needs to build container images inside the pipeline.
Two options evaluated: Docker-in-Docker (DinD) or Kaniko.

## Decision
Use **Kaniko** for all CI image builds.

## Reasons
- Docker-in-Docker requires privileged containers — a serious
  security vulnerability in shared Kubernetes environments
- Privileged containers bypass Pod Security Standards
  (restricted and baseline levels both block privileged mode)
- Kaniko builds images without Docker daemon, without root,
  without any privileged access
- Kaniko runs as a standard unprivileged pod — safe in any
  Kubernetes cluster
- Production security teams reject DinD for exactly this
  reason — Kaniko is the industry-standard replacement
- No Docker socket mount required — eliminates host escape
  attack vector

## How Kaniko Works
