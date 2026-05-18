# ADR 009 — Custom Dockerfiles for All OTel Demo Services

## Status
Accepted

## Context
OpenTelemetry Demo provides upstream Docker images on Docker Hub.
Project needs to decide: use upstream images directly, or write
custom Dockerfiles for every service.

## Decision
Write **custom Dockerfiles** for all OTel Demo services instead
of using upstream images.

## Reasons
- Custom Dockerfiles give full ownership of the build chain
- Trivy CVE scan runs on every image in the CI pipeline —
  upstream images pulled directly bypass this security gate
- Multi-stage builds significantly reduce final image size
- Non-root user (USER 1000) required for Pod Security Standards
  restricted namespace enforcement
- HEALTHCHECK directive enables Kubernetes liveness and
  readiness probes
- Portfolio value: custom Dockerfiles across 22 microservices
  in Go, Java, Python, .NET, Node.js, Ruby, Rust, C++,
  Elixir, Kotlin, PHP demonstrates real build chain ownership
- Interview signal: "I wrote and maintained Dockerfiles for
  every service" vs "I used upstream images"

## Consequences
- Significant upfront effort in Phase 12a-docker and 12b-docker
- Each Dockerfile must be maintained when upstream changes
- All images stored in GitLab Container Registry on md-ci-1
- Full Trivy scan coverage on every image in the pipeline

## Alternatives Rejected
- **Upstream images directly** — no build chain ownership,
  no Trivy scan in pipeline, no control over base image,
  non-root user not guaranteed, PSS restricted may reject them
