# 08 — Risk Analysis

## Purpose

Identify architectural, operational, and delivery risks before they become incidents. A risk that is named and mitigated is manageable. A risk that is discovered in production is a crisis.

---

## Risk Classification

Risks are classified by likelihood and impact. The classification drives the response.

| Likelihood \ Impact | Low | Medium | High |
|---|---|---|---|
| **High** | Monitor | Mitigate | Escalate immediately |
| **Medium** | Accept | Mitigate | Mitigate |
| **Low** | Accept | Monitor | Mitigate |

---

## Architectural Risks

### AR-01: Payment Gateway as a Single Point of Failure

**Likelihood:** High  
**Impact:** High  
**Risk:** NPCI is an external, third-party service. Its availability and latency are outside ABC Company's control. A payment gateway outage or significant latency spike directly impacts vehicle processing time.

**Current State:** The existing system fails completely when the gateway is unavailable.

**Mitigation:**
1. Pre-authorization decouples the gate decision from the payment confirmation. The barrier opens on a confirmed fund hold, not a confirmed deduction.
2. Balance cache at the edge provides a fallback when the gateway is unreachable. Transactions are queued for reconciliation.
3. Circuit breaker pattern on the NPCI client: after 5 consecutive failures within 30 seconds, the service switches to cache-fallback mode and stops hammering the gateway.
4. NPCI SLA and maintenance window schedule is integrated into the operations calendar.

**Residual Risk:** Cache-fallback mode carries a small risk of authorizing a transaction against a stale balance. This is bounded by the cache TTL and the reconciliation process. Acceptable given the alternative (lane shutdown).

---

### AR-02: Edge-Cloud Split Consistency

**Likelihood:** Medium  
**Impact:** Medium  
**Risk:** Pre-authorization tokens are issued by the cloud Pre-Auth Service and consumed by the edge Lane Controller. If the cloud writes a token to Redis and the edge cache does not receive it before the vehicle arrives at the barrier, the gate will not open despite a valid authorization.

**Mitigation:**
1. Pre-authorization happens at T-13 seconds (vehicle 150m from gate). The write-to-read window is ~13 seconds — well within Redis replication latency.
2. Fallback: edge unit can make a direct query to the Pre-Auth Service if local cache miss occurs. This adds ~200ms but does not fail the transaction.
3. Token TTL is 120 seconds — enough time for cache synchronization and vehicle arrival.

**Residual Risk:** Network partition between edge and cloud during the 13-second window. In this case, the fallback query also fails, and the vehicle routes to manual lane. Unlikely but possible.

---

### AR-03: RFID Reader Placement and Vehicle Matching

**Likelihood:** Medium  
**Impact:** Medium  
**Risk:** A reader at 150m reads tags as vehicles approach. If multiple vehicles are close together (convoy, traffic jam), a tag read may be associated with the wrong vehicle, triggering a pre-authorization for a vehicle that is not yet at the gate.

**Mitigation:**
1. Pre-authorization is lane-specific. The token is keyed by `tag_id + lane_id + timestamp`. Even if a tag is read early, it will only open the barrier on that specific lane.
2. Loop sensors at both the upstream position and the gate correlate vehicle presence with tag reads. A tag read without a corresponding loop sensor trigger at the upstream position is flagged.
3. Token TTL of 120 seconds bounds the window — if the vehicle does not arrive within 120 seconds, the pre-authorization expires and must be re-acquired.

**Residual Risk:** Tag read from a closely following vehicle could pre-authorize the wrong transaction. This is a known challenge in multi-vehicle RFID scenarios and is an active research area. Vehicle-to-tag correlation via loop sensor reduces but does not eliminate this risk.

---

### AR-04: Double Charge on Retry or Replay

**Likelihood:** Low  
**Impact:** High  
**Risk:** If a settlement event is processed twice — due to a Kafka consumer restart, a network retry, or an edge event replay — a vehicle could be charged twice for the same passage.

**Mitigation:**
1. All payment operations are idempotent. The Pre-Auth token is the idempotency key. A settlement request with a token that has already been settled returns success without re-processing.
2. Kafka consumer uses exactly-once semantics (EOS) for settlement events.
3. The Audit Service detects duplicate events (same token, same barrier event within 120 seconds) and flags them before they reach the reconciliation layer.

**Residual Risk:** Extremely low. Idempotency + EOS + duplicate detection provides three independent safeguards.

---

### AR-05: Kafka Cluster Failure

**Likelihood:** Low  
**Impact:** High  
**Risk:** Kafka is the primary integration bus. A Kafka cluster failure would break event-driven processing across all services.

**Mitigation:**
1. Kafka deployed with replication factor 3 across AKS nodes in separate availability zones.
2. Services have a synchronous fallback path for critical operations (pre-auth can be triggered via direct API call if Kafka is unavailable).
3. Kafka monitoring via Dynatrace — broker health alerts fire before consumer lag becomes critical.
4. Azure Managed Kafka (Event Hubs with Kafka protocol) considered as a fallback to avoid self-managing the broker in production.

---

### AR-06: Data Schema Evolution

**Likelihood:** Medium  
**Impact:** Medium  
**Risk:** Event schemas evolve over time. A schema change that is not backward-compatible breaks consumers that have not yet been updated.

**Mitigation:**
1. Confluent Schema Registry enforces schema compatibility on every producer publish.
2. Backward-compatible changes only (new optional fields, no removals).
3. Breaking changes require a two-phase rollout: deploy consumers first, then producers.
4. Schema changes require a PR with a Tech Lead review before deployment.

---

## Operational Risks

### OR-01: Edge Unit Hardware Failure

**Likelihood:** Medium  
**Impact:** High  
**Risk:** Edge Processing Units are deployed in outdoor environments and subject to environmental stress (heat, vibration, power fluctuations). Hardware failure at an edge unit takes down all lanes at that plaza.

**Mitigation:**
1. Edge units deployed in redundant pairs at each plaza (active-standby).
2. Heartbeat monitoring via OpenTelemetry — a missed heartbeat triggers an alert within 60 seconds.
3. Edge units use UPS (uninterruptible power supply) with minimum 30-minute runtime.
4. Hardware failure runbook defines replacement procedure with target restoration time of < 2 hours.

---

### OR-02: Operator Skill Gap

**Likelihood:** Medium  
**Impact:** Medium  
**Risk:** Toll plaza operators manage the new system with significantly more complexity than the current system. Incorrect operator actions (wrong override, misconfigured lane) could cause cascading failures.

**Mitigation:**
1. Operator dashboard is designed for minimal cognitive load — actions are confirmed before execution.
2. AI Operations Assistant provides context-aware guidance during incidents.
3. Operator training program delivered before plaza rollout, refreshed quarterly.
4. All operator actions are audit-logged with a 30-second undo window for non-financial actions.

---

### OR-03: NHAI Integration Delays

**Likelihood:** Medium  
**Impact:** Medium  
**Risk:** Integration with NHAI's central reporting system depends on NHAI providing a stable API. Government API projects frequently experience delays.

**Mitigation:**
1. NHAI integration is designed as a separate, non-blocking service. Core toll processing does not depend on it.
2. A stub implementation handles NHAI integration during development. Switch to the real API when available.
3. Data is stored locally and batched for transmission when the NHAI API becomes available.

---

## Delivery Risks

### DR-01: NPCI Staging Environment Availability

**Likelihood:** Medium  
**Impact:** High  
**Risk:** Pre-authorization flow cannot be fully tested without access to an NPCI staging environment. If NPCI does not provide a test environment, integration testing is blocked.

**Mitigation:**
1. A mock NPCI service is built in Sprint 1 and used throughout development and staging.
2. NPCI staging access is requested at project initiation — this is a dependency that must be tracked.
3. If NPCI staging is unavailable at Phase 2 exit, production deployment is delayed until it is resolved. This is a hard dependency.

---

### DR-02: Physical Hardware Installation Delay

**Likelihood:** Medium  
**Impact:** High  
**Risk:** Software can be built in parallel with hardware procurement, but the pre-stage RFID solution cannot be tested end-to-end until readers are physically installed at 150m.

**Mitigation:**
1. Test plaza is identified in Sprint 0 and hardware procurement started immediately.
2. Software testing uses RFID simulators that replicate reader behavior.
3. Phase 2 completion gate requires physical hardware validation at the test plaza.

---

### DR-03: Team Ramp-Up Time

**Likelihood:** Low  
**Impact:** Medium  
**Risk:** If engineers are new to the domain (FASTag, NPCI, RFID), there is a ramp-up period that reduces initial velocity.

**Mitigation:**
1. Domain knowledge sessions in Sprint 0 — FASTag architecture, NPCI integration, toll operations.
2. Tech Leads have prior payment or transport domain experience (hiring criteria).
3. Sprint 0 includes reading existing NHAI documentation and NPCI API specifications.

---

## Risk Register Summary

| ID | Risk | Likelihood | Impact | Status |
|---|---|---|---|---|
| AR-01 | Payment gateway SPOF | High | High | Mitigated (pre-auth + cache fallback) |
| AR-02 | Edge-cloud consistency | Medium | Medium | Mitigated (TTL + fallback query) |
| AR-03 | Vehicle matching at upstream reader | Medium | Medium | Mitigated (loop sensor correlation) |
| AR-04 | Double charge on replay | Low | High | Mitigated (idempotency + EOS) |
| AR-05 | Kafka cluster failure | Low | High | Mitigated (replication + fallback) |
| AR-06 | Schema evolution breakage | Medium | Medium | Mitigated (Schema Registry + policy) |
| OR-01 | Edge unit hardware failure | Medium | High | Mitigated (HA pairs + monitoring) |
| OR-02 | Operator skill gap | Medium | Medium | Mitigated (training + UX design) |
| OR-03 | NHAI integration delay | Medium | Medium | Accepted (non-blocking architecture) |
| DR-01 | NPCI staging unavailable | Medium | High | Tracked (mock + escalation path) |
| DR-02 | Hardware installation delay | Medium | High | Tracked (Sprint 0 procurement) |
| DR-03 | Team ramp-up | Low | Medium | Mitigated (Sprint 0 knowledge program) |

---

## Related Documents

- [03 — Assumptions](03-assumptions.md)
- [04 — Target Architecture](04-target-architecture.md)
- [07 — Observability](07-observability.md)
- [09 — Success Metrics](09-success-metrics.md)
