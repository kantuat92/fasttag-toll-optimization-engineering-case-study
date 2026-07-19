# ADR-002 — Event-Driven Architecture Over Synchronous API Chain

**Status:** Accepted  
**Date:** 2025-07-19  
**Authors:** Engineering Manager / Principal Engineer  
**Reviewers:** Tech Lead (Squad Lima), Platform Engineer  

---

## Context

The current FASTag system processes every step sequentially via synchronous HTTP calls. Tag read → backend lookup → payment gateway → audit log → barrier signal. The barrier cannot open until every step completes. This is the architectural source of the 3–5 minute delay.

A naive fix might be: speed up the HTTP calls, add caching, and improve server capacity. This reduces latency within the existing model but does not eliminate the structural problem: **everything is in the critical path**.

The question this ADR addresses is: what is the right integration model for the services in the new system, and why?

---

## Decision

**Use Apache Kafka as the primary integration mechanism between services. Gate-critical operations (token lookup, barrier signal) remain synchronous. All non-critical processing (settlement, audit, fraud analysis, analytics) is asynchronous via events.**

This creates a two-tier architecture:

**Tier 1 — Synchronous critical path (< 200ms total):**
```
Edge RFID Read → Edge Redis Token Lookup → Lane Controller Barrier Signal
```

**Tier 2 — Async event stream (parallel to vehicle passage):**
```
TagRead (event) → Pre-Auth Service → Payment Service → Audit Service → Fraud Service
```

By the time the vehicle arrives at the barrier, Tier 2 is already largely complete (pre-auth is done). After the barrier opens, Tier 2 completes settlement and audit asynchronously. The vehicle does not wait for any of it.

---

## Why Kafka

Apache Kafka provides:

1. **Durability.** Events are persisted to disk. A service restart does not lose events — consumers replay from the last committed offset.
2. **Replayability.** If the Fraud Detection Service has a bug and misclassifies events, the topic can be replayed with the fixed service. This is not possible with direct API calls.
3. **Decoupling.** The Pre-Auth Service does not know about the Fraud Detection Service. Adding a new consumer (analytics, NHAI reporting) requires no changes to any existing service.
4. **Backpressure handling.** If downstream services cannot keep up with event throughput, consumer lag grows — it is visible and manageable. A synchronous chain under the same conditions times out or rejects requests.
5. **Ordering guarantees.** Kafka partitions by `lane_id`. All events from a single lane are ordered, which is required for correct transaction sequencing.

---

## Alternatives Considered

### Alternative 1: REST API Chain (Current Model, Optimized)

**Approach:** Keep synchronous HTTP calls but add connection pooling, timeouts, and circuit breakers.

**Rejected because:**
- Settlement and audit remain in the critical path regardless of optimization.
- The payment gateway's P99 latency (8+ seconds under load) cannot be improved by the application tier.
- A circuit breaker on the payment gateway call helps availability but does not reduce the 30-second processing target — it just fails faster.

---

### Alternative 2: Message Queue (Azure Service Bus)

**Approach:** Use Azure Service Bus as the messaging infrastructure rather than Kafka.

**Not rejected outright — it is a valid option for simpler message queue scenarios.**

**Chosen against because:**
- Kafka's log-based storage enables event replay for fraud analysis and reprocessing. Azure Service Bus deletes messages once consumed.
- Kafka's partition model maps naturally to lane-level ordering requirements.
- If the team already has Kafka experience, Azure Service Bus provides no capability advantage at this scale.
- Azure Event Hubs (Kafka-compatible protocol) is available as a managed Kafka option if operational overhead of self-managed Kafka is a concern.

---

### Alternative 3: Choreography via Event Mesh

**Approach:** Use a full event mesh (e.g., Solace) for richer routing, filtering, and topology management.

**Rejected because:**
- Adds significant operational complexity.
- Kafka already provides everything needed at this scale.
- Event mesh adds value in large organizations with dozens of disparate event producers and consumers. This system has a defined, bounded set of services.

---

### Alternative 4: Saga Pattern with Orchestration

**Approach:** A central orchestrator (Spring State Machine or Temporal) coordinates each step in the transaction.

**Rejected as primary pattern (retained for payment saga specifically):**
- A central orchestrator introduces a new point of failure in the processing chain.
- For the gate decision path, the latency cost of orchestrator coordination is unacceptable.
- The payment pre-authorization + settlement flow uses a saga pattern (two-phase: pre-auth + capture), which is appropriate for financial correctness. This is implemented within the Payment Service, not as a system-level orchestrator.

---

## Event Schema

All events are defined in Avro and registered in Confluent Schema Registry. Backward-compatible changes only (new optional fields, never removals or type changes).

### Core Events

```json
// TagRead
{
  "event_type": "TagRead",
  "event_id": "uuid-v4",
  "tag_id": "anonymized-hash",
  "plaza_id": "PLAZA-NH48-023",
  "lane_id": "LANE-04",
  "reader_id": "READER-UPSTREAM-04",
  "timestamp_utc": "2025-07-19T08:32:11.453Z",
  "signal_strength_dbm": -42,
  "retry_count": 0
}

// PreAuthorizationApproved
{
  "event_type": "PreAuthorizationApproved",
  "event_id": "uuid-v4",
  "tag_read_event_id": "uuid-v4",
  "pre_auth_token": "uuid-v4",
  "token_expires_at": "2025-07-19T08:32:11.453Z",
  "plaza_id": "PLAZA-NH48-023",
  "lane_id": "LANE-04",
  "toll_amount": 65.00,
  "currency": "INR"
}

// BarrierOpened
{
  "event_type": "BarrierOpened",
  "event_id": "uuid-v4",
  "pre_auth_token": "uuid-v4",
  "plaza_id": "PLAZA-NH48-023",
  "lane_id": "LANE-04",
  "opened_at": "2025-07-19T08:32:23.812Z",
  "processing_time_ms": 12359,
  "authorization_method": "PRE_AUTH"
}
```

---

## Topic Design

| Topic | Partition Key | Retention | Consumers |
|---|---|---|---|
| `fasttag.tag.read` | `lane_id` | 7 days | Pre-Auth Service, Fraud Service |
| `fasttag.preauth.result` | `lane_id` | 7 days | Lane Controller, Payment Service |
| `fasttag.barrier.event` | `lane_id` | 30 days | Payment Service, Audit Service, Fraud Service |
| `fasttag.transaction.complete` | `tag_id` (hashed) | 90 days | Audit Service, NHAI Reporting |
| `fasttag.fraud.signal` | `tag_id` (hashed) | 90 days | Operations Platform, NHAI |
| `fasttag.reader.health` | `reader_id` | 7 days | AI Prediction Service, Operations |

---

## Consequences

### Positive

- Non-critical processing (settlement, audit, fraud, analytics) does not affect gate latency.
- System components can be deployed, scaled, and updated independently.
- Event replay enables reprocessing for fraud analysis or bug fixes without data loss.
- New capabilities (NHAI analytics, MLPC fraud model, future services) can be added by subscribing to existing topics — zero changes to producers.

### Negative

- Eventual consistency. The audit record for a transaction is created seconds after the barrier opens, not atomically with it. This is acceptable (and expected) in an event-driven model.
- Kafka adds operational complexity: broker management, consumer group coordination, offset management. Mitigated by considering Azure Event Hubs (Kafka protocol) as the managed option.
- Debugging is harder than synchronous chains. Distributed tracing (ADR-004) compensates for this.

### Constraints This Decision Places on the Design

- All services must be idempotent consumers. An event may be delivered more than once; processing it twice must produce the same result.
- Schema evolution is constrained to backward-compatible changes.
- Consumer group assignments must be documented and managed — two services accidentally consuming the same consumer group will each miss half the events.

---

## Future Considerations

- **Event sourcing for the Audit Context.** Currently the Audit Service writes derived records. Event sourcing (treating the event log as the source of truth) would eliminate the audit database entirely. This is worth evaluating at Phase 4 when audit volume is understood.
- **Stream processing (Apache Flink or Kafka Streams).** Real-time analytics — queue length prediction, revenue dashboards — may benefit from stream processing over raw Kafka consumption.

---

## References

- [04 — Target Architecture](../docs/04-target-architecture.md)
- [08 — Risk Analysis: AR-05, AR-06](../docs/08-risk-analysis.md)
- [ADR-001 — Early RFID Detection](ADR-001-Early-RFID-Detection.md)
