# FASTag Toll Optimization — Engineering Proposal

> **Engineering Manager Submission | National Highways Authority Contract**

---

## Executive Summary

ABC Company has contracted with NHAI to reduce FASTag toll processing time from 3–5 minutes to under 30 seconds. This is not primarily a hardware problem. It is a **distributed systems problem** — synchronous processing, no pre-computation, and a single point of failure through external payment gateways.

This proposal describes a production-grade architecture that solves the latency problem through early RFID detection, an event-driven processing pipeline, and AI-assisted operations. It also describes how I would lead, staff, and deliver this program.

---

## The Problem in One Sentence

The toll gate waits for everything — tag read, payment authorization, and backend logging — in a sequential chain, while 20 vehicles queue behind a barrier that could have been pre-cleared 8 seconds ago.

---

## Proposed Architecture — High Level

```
[RFID Reader @ 150m]  →  [Edge Processing Unit]  →  [Event Bus]
         ↓                                                 ↓
  [Pre-Auth Service]    ←→    [Payment Gateway]    [Audit Log Service]
         ↓                                                 ↓
  [Lane Controller]     →     [Barrier Signal]     [Observability Platform]
```

**Core principle:** Move work left. Pre-read the tag, pre-authorize the payment, and make the gate decision in under 2 seconds of the vehicle arriving at the barrier.

**Fallback principle:** Every component degrades gracefully. If the payment gateway is unreachable, the system runs on cached balance and reconciles offline. The barrier never stays closed indefinitely due to a network failure.

---

## Why This Approach Works

| Problem | Root Cause | Solution |
|---|---|---|
| 3–5 min delay | Sequential processing at gate | Async pre-authorization at 150m |
| Manual fallback queues | Tag read failure with no retry | AI-predicted failure + redundant reader array |
| Payment gateway latency | Synchronous HTTP call at gate | Pre-auth with idempotent confirmation |
| No visibility | No structured observability | OpenTelemetry + Dynatrace integration |
| Single point of failure | No offline mode | Edge cache + async reconciliation |

---

## Architecture Principles

1. **Pre-compute, don't react** — Decisions at the gate are confirmations, not computations.
2. **Event-driven, not request-driven** — Gate events are published to a durable bus. Downstream systems (billing, audit, analytics) consume independently.
3. **Observability is not optional** — Every RFID read, payment call, and barrier action emits a trace span. No black boxes.
4. **AI augments, not replaces** — AI detects anomalies, predicts failures, and generates runbooks. Operators make decisions.
5. **Domain boundaries are explicit** — Lane Management, Payment, Fraud, Audit, and Operations are separate bounded contexts with defined contracts.
6. **Graceful degradation over hard failure** — The system has defined behavior for every failure mode before it ships.

---

## Engineering Principles

- Architecture decisions are documented as ADRs and version-controlled.
- No service goes to production without SLOs defined and dashboards wired.
- Every domain event has a schema and is registered in the event catalog.
- DORA metrics are tracked from Sprint 1.
- Incident postmortems are blameless and feed back into the engineering backlog.

---

## AI Vision

AI in this system earns its place by solving real operational problems:

| Capability | Problem Solved | Business Value |
|---|---|---|
| RFID Failure Prediction | Readers degrade before they fail visibly | Proactive maintenance, reduced manual lanes |
| Traffic Queue Forecasting | Lane assignment is reactive | Dynamic lane optimization, throughput +20% |
| Fraud Signal Detection | Tag cloning goes undetected | Revenue protection, regulatory compliance |
| AI Operations Assistant | On-call engineers spend 40min per incident tracing logs | RCA in under 5 minutes |
| GenAI Runbook Generation | Runbooks are stale or missing | Accurate, context-aware runbooks from telemetry |

AI is not a feature layer. It is wired into the observability and operations platform from day one.

---

## Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Edge Processing | Java 21 + Spring Boot | Team expertise, proven in high-throughput enterprise |
| Event Bus | Apache Kafka | Durable, replayable, high-throughput event streaming |
| API Gateway | Spring Cloud Gateway | Consistent with Spring ecosystem, enterprise-grade |
| Pre-Auth Service | Spring Boot + Redis Cache | Sub-10ms balance lookup, offline fallback |
| Container Orchestration | Azure Kubernetes Service (AKS) | Cloud-native, team familiarity, enterprise SLA |
| Observability | OpenTelemetry + Dynatrace | Distributed tracing, AI-assisted anomaly detection |
| AI/ML Platform | Azure ML + custom inference services | Fraud, failure prediction, traffic forecasting |
| Database | PostgreSQL (transactions), Redis (cache) | ACID for financial records, low-latency for gate decisions |
| CI/CD | Azure DevOps + Helm + ArgoCD | GitOps delivery, rollback-safe deployments |
| Security | Azure AD, mTLS, Zero Trust network policy | No implicit trust between services |

---

## Engineering Management Approach

This is a delivery program, not a proof of concept. Engineering management is as important as architecture.

**Team Structure:**
- 2 squads of 5 engineers each (Edge + Core Services)
- 1 Platform engineer (CI/CD, observability, shared tooling)
- 1 Tech Lead per squad (accountable for quality, not just velocity)
- Embedded QA from Sprint 1

**Delivery Model:**
- 2-week sprints, fixed-length
- Architecture reviews gated on ADR approval
- No feature is "done" without SLO defined and dashboard wired
- DORA metrics tracked and reviewed in retrospectives

**Technical Debt:**
- Tracked in the backlog with business impact stated
- Allocated 20% sprint capacity for debt and operational improvement
- Debt items above a defined risk threshold are escalated to Product Owner

---

## Repository Navigation

```
/docs                   — Engineering documents (business problem through conclusion)
/adr                    — Architecture Decision Records (all major decisions)
/diagrams               — Mermaid diagrams (C4 context, container, sequence, deployment)
/presentation           — Executive presentation (10 slides, markdown)
```

| Document | Description |
|---|---|
| [Business Problem](docs/01-business-problem.md) | Problem framing, stakeholder impact, business case |
| [Current State](docs/02-current-state.md) | Where we are today, root cause analysis |
| [Assumptions](docs/03-assumptions.md) | Stated assumptions and their impact on design |
| [Target Architecture](docs/04-target-architecture.md) | C4 architecture, design decisions, bounded contexts |
| [AI Enhancements](docs/05-ai-enhancements.md) | AI features with problem, approach, and business value |
| [Engineering Execution](docs/06-engineering-execution.md) | Delivery plan, team structure, DORA, sprint model |
| [Observability](docs/07-observability.md) | OpenTelemetry, SLOs, golden signals, AI-assisted RCA |
| [Risk Analysis](docs/08-risk-analysis.md) | Architectural and operational risks with mitigations |
| [Success Metrics](docs/09-success-metrics.md) | Business KPIs, engineering KPIs, SLIs/SLOs |
| [Conclusion](docs/10-conclusion.md) | Summary, trade-offs accepted, what comes next |

| ADR | Decision |
|---|---|
| [ADR-001](adr/ADR-001-Early-RFID-Detection.md) | Pre-stage RFID reading at 150m |
| [ADR-002](adr/ADR-002-Event-Driven-Processing.md) | Event-driven architecture over synchronous API chain |
| [ADR-003](adr/ADR-003-AI-Operations-Assistant.md) | AI-assisted operations and RCA |
| [ADR-004](adr/ADR-004-Observability-First.md) | OpenTelemetry as the observability standard |

---

## What This Repository Is Not

- It is not a complete implementation. It is an engineering proposal.
- It does not assume unlimited budget. Trade-offs are documented where cost was a factor.
- It does not promise a timeline without knowing the team size and current codebase state.
- It does not hide risks. Every significant risk has a mitigation and an owner.

---

## Author Context

This proposal reflects experience from:
- **FedEx** — High-throughput distributed systems, event-driven logistics at scale
- **JP Morgan** — Enterprise-grade financial transaction processing, audit, compliance
- **AI SRE** — Dynatrace Davis AI, automated anomaly detection, AI-assisted incident management
- **Platform Engineering** — Azure Kubernetes, GitOps delivery, developer experience programs
- **DDD** — Bounded context design in large, multi-team programs

The architecture choices are not theoretical. They reflect what works in production at scale.

---

*This document is part of a series. See [/docs](docs/) for the full engineering proposal.*
