# ADR 004 — Jaeger over Tempo

## Status
Accepted

## Context
Project requires a distributed tracing backend to store and query
traces from the OpenTelemetry Demo application.

## Decision
Use **Jaeger** as the distributed tracing backend.

## Reasons
- Jaeger is industry-standard, widely recognized in production environments
- Strong community adoption — most DevOps/Platform Engineer job descriptions mention Jaeger
- Well-documented integration with OTel Collector
- Jaeger UI is familiar to most engineering teams
- Interview value: Jaeger experience directly transferable

## Consequences
- OTel Collector sends traces to Jaeger via Thrift protocol (:14250)
- Grafana datasource configured to query Jaeger
- Single instance (all-in-one mode) — not HA (lab tradeoff)
- Production equivalent: Jaeger with Elasticsearch/Cassandra backend

## Alternatives Rejected
- **Tempo** — less industry recognition at time of decision
