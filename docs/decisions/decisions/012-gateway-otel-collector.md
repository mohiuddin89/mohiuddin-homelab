# ADR 012 — Gateway-Only OTel Collector (Initial)

## Status
Accepted

## Context
OpenTelemetry Collector can be deployed in multiple modes:
- Agent mode: DaemonSet (one pod per node)
- Gateway mode: Deployment (single centralized pod)
- Combined: agent per node feeding a central gateway

## Decision
Deploy OTel Collector in **gateway mode only** (single
Deployment) as the initial setup in Phase 11.

## Reasons
- Single config file, single pod — simpler to debug
- One scrape target for Prometheus
- One OTLP endpoint for all applications to send telemetry
- Easier to understand end-to-end data flow at learning stage
- Agent + gateway split adds operational complexity before
  the basics are working and understood
- Phase 11 goal: get telemetry flowing end-to-end first,
  then optimize

## Architecture
