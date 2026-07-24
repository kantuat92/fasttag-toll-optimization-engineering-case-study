# 11 — Tech Muscle

---

## 1. Business & Requirements

- **Problem statement** — Why vehicles wait 3–5 minutes despite FASTag being automated. The structural issue: processing starts too late.
- **Business goals** — Reduce end-to-end processing to under 30 seconds. Improve throughput. Reduce manual intervention rate from ~15% to under 2%.
- **Functional requirements** — Pre-stage RFID reading at 150m, express FASTag lanes, backward compatibility with existing manual lanes.
- **Non-functional requirements** — P95 < 30s, 99.9% uptime, 99.99% audit completeness, zero double-charge events.
- **Assumptions** — 13 documented assumptions (see `03-assumptions.md`): NPCI P95 SLA, vehicle approach speed, read success rate, peak throughput targets.


---

## 2. Architecture

### Topics to Cover
- **Current vs proposed architecture** — Current: synchronous 5-step chain, everything in critical path, no offline capability, no observability. Proposed: 3-zone architecture, event-driven, edge-first decision making.
- **Service decomposition** — 5 bounded contexts: Lane Management, Payment, Fraud Detection, Audit, Operations. Each owns its data. No shared databases.
- **API Gateway** — Spring Cloud Gateway. Single entry point for all external consumers (operators, NHAI, monitoring). Handles authentication, rate limiting, routing. Sits behind Azure Front Door.
- **Microservices** — Pre-Auth Service, Payment Service, Fraud Detection Service, Audit Service, Lane Management Service, NHAI Reporting Service. Each independently deployable.
- **Event-driven architecture** — Apache Kafka as primary integration mechanism. Two-tier model: synchronous critical path (token lookup → barrier signal) and asynchronous event stream (settlement, audit, fraud, analytics).
- **Service communication** — Synchronous (HTTP/REST via API Gateway for external), asynchronous (Kafka events for internal), mTLS via Istio service mesh for all service-to-service calls.


---

## 3. Backend Engineering

- **Java 21** — Team expertise, LTS release, virtual thread support via Project Loom.
- **Spring Boot 3.x** — Auto-configuration, embedded server, Spring Cloud for service discovery and gateway.
- **REST APIs** — External-facing endpoints through API Gateway. Contract-first design. Spring Cloud Contract for consumer-driven contract tests.
- **Virtual Threads (Project Loom)** — Applicable to the Pre-Auth Service: NPCI call is a blocking I/O operation. Virtual threads allow high concurrency without the overhead of large thread pools. A single JVM can sustain thousands of concurrent NPCI calls with virtual threads where a traditional thread pool would exhaust at 200–500.
- **Validation** — Bean Validation (Jakarta Validation) at API boundary. All inputs validated at system boundaries. Internal service calls do not re-validate (trust internal contracts).
- **Idempotency** — 30-second deduplication window for RFID reads. Pre-auth token as idempotency key for settlement. Three-layer protection: application check, database unique constraint, NPCI client reference ID.
- **Error handling** — Structured error responses at API Gateway. Internal exceptions mapped to domain events (PreAuthorizationDeclined, FraudSignalRaised). No raw stack traces in API responses.


---

## 4. Data & Messaging


- **Kafka** — 6 topics, partitioned by lane_id (for ordering) or tag_id (for fraud correlation). Replication factor 3. Schema registry (Avro) with backward-compatibility enforcement. 7–90 day retention depending on topic. Azure Event Hubs as managed Kafka alternative.
- **Database design** — PostgreSQL 15 (HA). Separate schemas per bounded context. Append-only audit table (no UPDATE, no DELETE). Expand-contract migration pattern for zero-downtime schema changes. Write-ahead log for audit durability.
- **Transaction management** — `@Transactional` in Spring for settlement writes. Idempotency check + NPCI call + record update in a single transaction boundary. Optimistic locking (`@Version`) for concurrent pre-auth updates.
- **Event publishing** — Transactional outbox pattern: event written to outbox table in same transaction as domain state change. Separate relay process publishes to Kafka. Prevents lost events on application crash.
- **Dead Letter Queues** — Kafka dead letter topic per consumer group. Messages that fail after max retries are routed to DLQ. Operations team reviews DLQ manually. Re-processing is possible by replaying from original topic with corrected consumer logic.


---

## 5. External Integrations

### Topics to Cover
- **NPCI (National Payments Corporation of India)** — External payment gateway. Pre-authorization (fund hold) + capture (settlement) two-phase model. P95 SLA assumption: 5 seconds. Circuit breaker wraps all NPCI calls. Cached balance fallback when NPCI is unreachable.
- **Issuer Bank** — Behind NPCI. Balance deduction happens at issuing bank level. The system interacts only with NPCI; bank integration is NPCI's responsibility.
- **FASTag Issuer System** — Provides tag registration data, vehicle class, account status. Queried during tag validation. Cached locally. Blacklist updates propagated to edge Redis.
- **Fraud & Risk Engine** — Internal: Fraud Detection Service consuming Kafka events, checking geographic impossibility, velocity anomaly, balance anomaly. External integration point for future: NHAI's central fraud database or an external risk scoring API.


---

## 6. Reliability

### Topics to Cover
- **Circuit Breaker** — Resilience4j wrapping NPCI calls. States: Closed (normal), Open (failing, reject immediately), Half-Open (test recovery). Configured: 50% failure rate in 10-call sliding window → Open. 30-second wait → Half-Open. 3 permitted calls → Closed if successful.
- **Retry** — Exponential backoff with jitter for NPCI retries (max 2 retries, base 500ms, jitter ±100ms). No retry on permanent failures (insufficient balance, invalid tag). Retry on transient failures (timeout, network error).
- **Timeout** — NPCI call timeout: 5 seconds (per SLA assumption). Redis timeout: 50ms. Kafka publish timeout: 1 second. All timeouts explicit — no framework defaults.
- **Health checks** — Spring Boot Actuator `/health` endpoints for Kubernetes liveness and readiness probes. Readiness check includes: Redis connectivity, Kafka producer connectivity, NPCI reachability. Liveness check: JVM heap, thread pool health.
- **Monitoring** — Dynatrace Davis AI for anomaly detection. Four golden signals per service: latency, traffic, errors, saturation. SLO dashboards. Alert tiers: P1 (immediate page), P2 (15-min review), P3 (Slack), P4 (next business day).
- **Logging** — Structured JSON logs. Correlated with traces via trace_id and span_id. Log levels: INFO for business events, WARN for recoverable errors, ERROR for unrecoverable failures. No PII in logs. Tag IDs hashed in production logs.
- **Distributed tracing** — OpenTelemetry Java agent (auto-instrumentation) + manual span enrichment for business-critical paths. Full vehicle passage trace: RFID read → edge publish → pre-auth → token lookup → barrier open → settlement → audit. `fasttag.*` custom attribute namespace.


---

## 7. Scalability

- **Horizontal scaling** — All microservices are stateless. State is in Redis or Kafka. Adding pods increases throughput linearly. Pre-Auth Service scales independently of Payment Service. Each service scales based on its own load signal.
- **Stateless services** — No in-process session state. Pre-auth tokens are in Redis, not JVM memory. Any pod can handle any request. Kubernetes can reschedule pods without losing state.
- **Load balancing** — Azure Front Door for external traffic. Azure Load Balancer for AKS ingress. Spring Cloud Gateway for internal API routing. Istio service mesh for service-to-service load balancing with traffic policies.
- **Kubernetes** — Azure Kubernetes Service (AKS). Namespace-per-service (network isolation). Pod anti-affinity rules (no two instances of the same service on the same node). Pod Disruption Budgets for safe rolling deployments.
- **Autoscaling** — Kubernetes Horizontal Pod Autoscaler (HPA) configured per service. Pre-Auth Service: scales on Kafka consumer lag (KEDA — Kubernetes Event-Driven Autoscaling). Payment Service: scales on CPU. All services have min/max replica bounds.



---

## 8. Security

### Topics to Cover
- **Authentication** — Azure Active Directory for operator authentication. Service-to-service: mTLS certificates managed by Azure Key Vault. No service trusts another by default (Zero Trust).
- **Authorization** — Role-based: Toll Operator (lane management, manual override), NHAI Reporting (read-only), Platform Engineer (observability), Engineering Manager (all). API Gateway enforces authorization before routing.
- **API security** — All external endpoints require JWT (Azure AD token). Rate limiting at API Gateway (per-client, per-endpoint). Request validation before forwarding to services. No raw stack traces in error responses.
- **Encryption** — TLS 1.3 for all in-transit traffic. Payment tokens encrypted at rest (AES-256). Tag IDs hashed in analytics pipelines (SHA-256 with per-deployment salt). All secrets in Azure Key Vault with rotation policy. No hardcoded credentials.
- **Audit logging** — Every transaction permanently recorded. Every manual override recorded with operator ID and reason code. Audit table is append-only — no update or delete at the application layer. Long-term archive to Azure Blob cold storage (1 year).

---

## 9. Engineering Lead

- **Assumptions and requirement clarification** — 13 named assumptions before architecture begins. Each assumption has an explicit impact-if-wrong statement. Assumptions validated with business stakeholders before design is finalized.
- **Trade-off analysis** — Eventual consistency accepted in exchange for low latency (audit record created after barrier opens, not atomically with it). Cached balance fallback accepted in exchange for availability during NPCI outage. Lane-specific tokens accepted in exchange for fraud protection (prevents token re-use across lanes).
- **Operational considerations** — Runbook required for every alert. On-call rotation: no engineer on call more than 1 week in 4. MTTR target: under 30 minutes. AI Ops Assistant reduces Tier 2 escalation burden. Blameless postmortems for all P1/P2 incidents.
- **Rollout strategy** — 4 stages: Stage 0 (1 lane, 1 plaza, 6 weeks pilot), Stage 1 (all lanes, 1 plaza), Stage 2 (1 full NH corridor, 8–15 plazas), Stage 3 (all contracted corridors). Each stage has an explicit exit gate — no stage begins on schedule alone, only on exit criteria met.
- **Risks and mitigation** — 12 named risks in `08-risk-analysis.md`. Key risks: NPCI SLA assumptions wrong (mitigated by cached balance fallback), edge unit deployment complexity at scale (mitigated by ArgoCD GitOps), tag read accuracy in real traffic (mitigated by gate-level reader redundancy), team ramp-up on distributed systems (mitigated by embedded QA and architecture review cadence).
- **Metrics to measure success**:
  - Average processing time (target: under 30 seconds)
  - P95 / P99 barrier-open latency (target: P95 < 30s, measured per lane)
  - Barrier-open success rate on first attempt (target: > 98%)
  - Payment success rate (target: > 99.5%)
  - Manual intervention rate (target: < 2%, current ~15%)
  - Express lane throughput (vehicles per hour per lane at peak)
  - Reader failure detection lead time (AI prediction: alert before failure, not after)
  - DORA metrics: Deployment Frequency, Lead Time for Changes, Change Failure Rate, MTTR

---

## Document Cross-Reference

| Topic Area | Primary Document | ADR | Diagram |
|---|---|---|---|
| Business & Requirements | `01-business-problem.md`, `03-assumptions.md` | — | — |
| Architecture | `04-target-architecture.md` | ADR-001, ADR-002 | container.png, future-state.png |
| Backend Engineering | `04-target-architecture.md`, `06-engineering-execution.md` | ADR-002 | component.png |
| Data & Messaging | `04-target-architecture.md` | ADR-002 | sequence.png |
| External Integrations | `04-target-architecture.md`, `03-assumptions.md` | ADR-002 | sequence.png |
| Reliability | `04-target-architecture.md`, `08-risk-analysis.md` | ADR-002 | — |
| Scalability | `06-engineering-execution.md`, `09-success-metrics.md` | ADR-004 | deployment.png |
| Security | `04-target-architecture.md` | ADR-004 | deployment.png |
| Engineering Lead | `00-engineering-analysis.md`, `06-engineering-execution.md`, `08-risk-analysis.md`, `09-success-metrics.md`, `10-conclusion.md` | All | All |

---

## Related Documents

- [00 — Engineering Analysis](00-engineering-analysis.md)
- [04 — Target Architecture](04-target-architecture.md)
- [06 — Engineering Execution](06-engineering-execution.md)
- [08 — Risk Analysis](08-risk-analysis.md)
- [09 — Success Metrics](09-success-metrics.md)
