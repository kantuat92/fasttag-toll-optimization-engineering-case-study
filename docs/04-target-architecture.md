# 04 — Target Architecture

## Purpose

Describe the target system with enough specificity that a senior engineer can begin implementation and an architect can validate trade-offs. This is not a design specification — it is an architectural blueprint.

---

## Design Principles

Before describing what is built, it helps to state why it is built that way.

1. **Pre-compute, don't react.** The gate decision must be ready before the vehicle arrives. All work that can be done upstream must be done upstream.

2. **Event-driven over request-driven.** Domain events — tag read, payment authorized, barrier opened, transaction completed — are the primary integration mechanism. Services consume events independently. No service blocks another.

3. **Edge-first, cloud-backed.** Gate decisions are made at the edge. The cloud validates, reconciles, and provides intelligence. A cloud outage degrades reporting and analytics, not lane operations.

4. **Observability is structural.** Traces, metrics, and logs are emitted by design, not added later. Every service boundary emits a span.

5. **Fail safe, not fail open.** If a lane cannot determine authorization status, it falls back to a defined degraded mode (pre-authorized-or-manual), not an undefined failure state.

6. **Bounded contexts over shared databases.** Lane Management, Payment, Fraud, Audit, and Operations are separate domains with separate data stores. Integration is through events, not shared schema.

---

## System Context (C4 Level 1)

```
┌──────────────────────────────────────────────────────────────────┐
│                          FASTag Toll System                      │
│                                                                  │
│  ┌──────────────┐   ┌───────────────────┐   ┌────────────────┐  │
│  │  Pre-Stage   │   │   Core Processing │   │  Operations &  │  │
│  │  RFID Edge   │──▶│   Platform (AKS)  │──▶│  Intelligence  │  │
│  │  (Per Plaza) │   │                   │   │  Platform      │  │
│  └──────────────┘   └───────────────────┘   └────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
          │                      │                      │
          ▼                      ▼                      ▼
  [NHAI Highway]        [NPCI Payment Gateway]    [Toll Operators]
  [Vehicles]            [Issuing Banks]           [NHAI Reporting]
```

**External actors:**
- **Vehicles with FASTag** — the primary end-user
- **NPCI Payment Gateway** — external payment network
- **NHAI** — contract authority and data consumer
- **Issuing Banks** — balance holders
- **Toll Operators** — human operators managing exceptions

---

## Container Diagram (C4 Level 2)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TOLL PLAZA (EDGE)                                                          │
│                                                                             │
│  ┌──────────────────┐    ┌─────────────────────┐    ┌──────────────────┐   │
│  │  RFID Reader     │    │  Edge Processing    │    │  Lane Controller │   │
│  │  (150m upstream) │───▶│  Unit               │───▶│  Service         │   │
│  │  + Reader Array  │    │  (Spring Boot)      │    │  (Barrier ctrl)  │   │
│  └──────────────────┘    └─────────────────────┘    └──────────────────┘   │
│                                    │                                        │
│                          ┌─────────────────────┐                           │
│                          │  Local Redis Cache   │                           │
│                          │  (Balance + Token)   │                           │
│                          └─────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │ Kafka (TLS)
┌─────────────────────────────────────────────────────────────────────────────┐
│  CORE PLATFORM (Azure Kubernetes Service)                                   │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │  Pre-Auth      │  │  Payment       │  │  Fraud         │                │
│  │  Service       │  │  Service       │  │  Detection     │                │
│  │  (Spring Boot) │  │  (Spring Boot) │  │  Service       │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │  Audit         │  │  Lane Mgmt     │  │  API Gateway   │                │
│  │  Service       │  │  Service       │  │  (Spring Cloud │                │
│  │  (Spring Boot) │  │  (Spring Boot) │  │   Gateway)     │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                             │
│  ┌────────────────────────────────────────────────────────┐                │
│  │  Apache Kafka (Event Bus)                              │                │
│  │  Topics: tag.read / pre-auth.result / barrier.event   │                │
│  │          transaction.complete / fraud.signal           │                │
│  └────────────────────────────────────────────────────────┘                │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐                                    │
│  │  PostgreSQL    │  │  Redis Cluster │                                    │
│  │  (Transactions │  │  (Auth tokens  │                                    │
│  │   + Audit)     │  │   + cache)     │                                    │
│  └────────────────┘  └────────────────┘                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
┌─────────────────────────────────────────────────────────────────────────────┐
│  INTELLIGENCE & OPERATIONS PLATFORM                                         │
│                                                                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │  AI Ops        │  │  Observability │  │  NHAI          │                │
│  │  Assistant     │  │  (OTel +       │  │  Reporting     │                │
│  │  (Azure ML)    │  │   Dynatrace)   │  │  Service       │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Bounded Contexts

The system is decomposed into five bounded contexts following Domain-Driven Design principles. Each context owns its data, its domain logic, and its integration contracts.

### 1. Lane Management Context

**Responsibility:** Everything that happens at the physical toll lane.

**Aggregates:** Lane, RFID Reader, Barrier, Vehicle Passage

**Key events published:**
- `TagRead` — tag ID detected at upstream reader
- `BarrierOpened` — barrier raised following authorization
- `BarrierClosed` — barrier lowered, vehicle through
- `ManualOverrideTriggered` — operator intervened

**Invariants:**
- A barrier may only open if a valid pre-authorization token exists
- A reader read is idempotent within a 30-second window for the same tag ID
- Manual override always creates an audit event

---

### 2. Payment Context

**Responsibility:** Pre-authorization and settlement with NPCI.

**Aggregates:** PreAuthorizationRequest, Settlement, Reconciliation

**Key events published:**
- `PreAuthorizationApproved` — funds held, token issued
- `PreAuthorizationDeclined` — insufficient balance or invalid tag
- `SettlementCompleted` — post-crossing deduction confirmed

**Invariants:**
- Every pre-authorization token is globally unique (UUID v4)
- A token expires after 120 seconds if not consumed by a barrier event
- Settlement is idempotent — replaying a settlement event has no effect if already processed

---

### 3. Fraud Detection Context

**Responsibility:** Identify fraudulent or anomalous tag usage.

**Aggregates:** TagActivity, FraudSignal

**Key events published:**
- `FraudSignalRaised` — anomalous pattern detected
- `TagBlacklisted` — tag flagged for manual review

**Key detection signals:**
- Geographic impossibility (same tag at two plazas too close in time)
- Velocity anomaly (tag used far more than expected vehicle type)
- Balance anomaly (pre-auth repeatedly declined then suddenly approved)

---

### 4. Audit Context

**Responsibility:** Durable, tamper-evident record of every transaction.

This context consumes events. It does not publish commands or influence gate decisions.

**Storage:** Append-only PostgreSQL write (no updates, no deletes). Long-term archive to Azure Blob Storage.

---

### 5. Operations Context

**Responsibility:** System health, incident detection, AI-assisted operations.

**Aggregates:** ReaderHealth, LaneStatus, IncidentRecord

Consumes telemetry from all other contexts. Publishes alerts and runbook suggestions to the operator dashboard.

---

## Pre-Authorization Flow (The Critical Path)

This is the sequence that achieves the sub-30-second target.

```
T-13s: Vehicle detected by upstream RFID reader (150m from gate)
         → Edge Processing Unit reads tag ID
         → Publishes TagRead event to Kafka

T-12s: Pre-Auth Service consumes TagRead
         → Checks local Redis for recent balance cache hit
         → Sends pre-authorization request to NPCI (async, 2–3 second SLA)

T-9s:  NPCI returns pre-authorization token + fund hold confirmation
         → Pre-Auth Service publishes PreAuthorizationApproved event
         → Token stored in Redis (TTL 120s, keyed by tag ID + plaza ID)

T-2s:  Vehicle arrives at barrier
         → Lane Controller queries edge cache for valid token
         → Token found → barrier open signal (< 100ms)

T-0s:  Barrier opens
         → BarrierOpened event published
         → Vehicle passes

T+2s:  Settlement event published
         → Payment Service submits capture to NPCI
         → Audit Service records complete transaction
```

**Total gate wait time: < 1 second** (barrier signal on token lookup, not payment confirmation)

---

## Failure Modes and Degraded Operation

### Mode 1: Payment Gateway Unreachable

- Edge cache holds last-known balance
- Pre-auth uses cached balance for decisions above a safety threshold
- Transactions queue for reconciliation when gateway recovers
- Operator dashboard flags the gateway outage within 30 seconds

### Mode 2: RFID Reader Failure (Upstream)

- AI model flags reader health degradation before failure (see [05 — AI Enhancements](05-ai-enhancements.md))
- Barrier-level reader activates as secondary
- Processing falls back to gate-level authorization (slower, but functional)
- Incident auto-created in operations platform

### Mode 3: Central Platform Unreachable

- Edge unit enters autonomous mode
- Pre-authorized tokens cached locally continue to function
- New tags without cached tokens route to manual lane
- All transactions queued for cloud reconciliation on reconnect

### Mode 4: Cloud Platform Failure (AKS)

- AKS multi-node deployment with pod anti-affinity ensures no single node failure takes down a service
- Database failover to read replica within 30 seconds (PostgreSQL HA)
- Kafka replication factor 3 — no data loss on broker failure

---

## Security Architecture

### Zero Trust

No service trusts any other service by default.

- All service-to-service calls use mTLS with certificates managed by Azure Key Vault
- API Gateway enforces authentication on all external-facing endpoints
- Azure Active Directory for operator authentication
- No hardcoded credentials — all secrets in Azure Key Vault with rotation policy

### Network Segmentation

```
Internet → [Azure Front Door] → [API Gateway] → [Service Mesh (Istio)]
                                                        │
                                    ┌───────────────────┼───────────────────┐
                                    │                   │                   │
                              [Pre-Auth NS]      [Payment NS]        [Fraud NS]
                              [Audit NS]         [Lane Mgmt NS]      [Ops NS]
```

Each service runs in its own Kubernetes namespace with network policies that deny all traffic not explicitly allowed.

### Data Protection

- Payment tokens encrypted at rest (AES-256)
- Tag IDs hashed in analytics pipelines (SHA-256 with per-deployment salt)
- Audit records are write-only — no update or delete operations permitted at the application layer

---

## Technology Decisions Summary

| Component | Technology | Decision Reference |
|---|---|---|
| Edge processing | Java 21 + Spring Boot 3.x | Team expertise, runtime efficiency |
| Event bus | Apache Kafka | [ADR-002](../adr/ADR-002-Event-Driven-Processing.md) |
| API Gateway | Spring Cloud Gateway | Consistent Spring ecosystem |
| Pre-auth cache | Redis 7.x (Cluster) | Sub-millisecond token lookup |
| Persistent storage | PostgreSQL 15 (HA) | ACID compliance for financials |
| Container platform | Azure Kubernetes Service | [ADR-004](../adr/ADR-004-Observability-First.md) |
| Observability | OpenTelemetry + Dynatrace | [ADR-004](../adr/ADR-004-Observability-First.md) |
| Service mesh | Istio | mTLS, traffic policy, observability |
| CI/CD | Azure DevOps + ArgoCD | GitOps, rollback-safe |

---

## Related Documents

- [05 — AI Enhancements](05-ai-enhancements.md)
- [07 — Observability](07-observability.md)
- [ADR-001](../adr/ADR-001-Early-RFID-Detection.md)
- [ADR-002](../adr/ADR-002-Event-Driven-Processing.md)
- [Diagrams](../diagrams/)
