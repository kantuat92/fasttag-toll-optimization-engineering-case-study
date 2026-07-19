# 07 — Observability

## Purpose

Define the observability strategy for the FASTag optimization system. Observability is not a post-launch concern — it is a design constraint. A system that cannot be observed cannot be operated reliably, and cannot be improved based on evidence.

---

## Background

The current FASTag system has no structured telemetry. Failures are discovered when queues form. Root causes are identified by manual log inspection. There is no way to distinguish between a payment gateway issue, a reader hardware fault, a backend processing delay, or a high arrival rate.

The target system inverts this: every component emits structured telemetry, and the operations platform provides a complete picture of system health from the perspective of both the business (vehicles processed, revenue collected) and engineering (latency, error rates, saturation).

---

## Observability Stack

| Layer | Tool | Purpose |
|---|---|---|
| Instrumentation | OpenTelemetry SDK (Java) | Traces, metrics, logs from all services |
| Collector | OpenTelemetry Collector | Receive, process, export telemetry |
| APM / Tracing | Dynatrace | Distributed tracing, AI-assisted anomaly detection |
| Metrics | Dynatrace + Prometheus | Service and infrastructure metrics |
| Logs | Dynatrace Log Management | Structured log ingestion and correlation |
| Alerting | Dynatrace Davis AI | Automated anomaly detection and alerting |
| Dashboards | Dynatrace Dashboards | Business and engineering KPI views |
| SLO Tracking | Dynatrace SLO Management | SLI measurement and burn rate alerting |

Dynatrace Davis AI provides the correlation layer — it connects anomalies across traces, metrics, and logs to surface root causes rather than individual alerts.

---

## OpenTelemetry Implementation

All services are instrumented using the OpenTelemetry Java SDK. Instrumentation is automatic for Spring Boot (via the OTel Java agent) with manual span enrichment for business-critical paths.

### Automatic Instrumentation

The following are automatically traced via the OTel Java agent:

- Incoming HTTP requests (Spring MVC / WebFlux)
- Outgoing HTTP client calls (RestTemplate, WebClient)
- Kafka producer and consumer operations
- JDBC operations (PostgreSQL)
- Redis client operations

### Manual Span Enrichment

Critical business operations are enriched with semantic attributes:

```java
// Example: Pre-authorization span enrichment
Span span = tracer.spanBuilder("pre-auth.authorize")
    .setAttribute("tag.id", tagId)
    .setAttribute("plaza.id", plazaId)
    .setAttribute("lane.id", laneId)
    .setAttribute("pre-auth.token", tokenId)
    .startSpan();
```

Standard semantic attributes follow OpenTelemetry conventions. Custom business attributes use the `fasttag.*` namespace.

### Custom Attribute Namespace

| Attribute | Description |
|---|---|
| `fasttag.tag.id` | Anonymized FASTag identifier |
| `fasttag.plaza.id` | Toll plaza identifier |
| `fasttag.lane.id` | Specific lane within plaza |
| `fasttag.pre_auth.token` | Pre-authorization token ID |
| `fasttag.vehicle.class` | Vehicle classification |
| `fasttag.payment.amount` | Toll amount (for audit context) |
| `fasttag.reader.signal_strength` | RFID reader signal quality |

---

## The Four Golden Signals

Google's four golden signals are applied to each service. These are the minimum observability requirement for any service that goes to production.

### 1. Latency

| Signal | Measurement | SLO |
|---|---|---|
| End-to-end processing time | Time from TagRead event to BarrierOpened event | P95 < 30s |
| Pre-authorization time | NPCI API response time | P95 < 5s |
| Reader-to-event latency | Time from physical read to Kafka message | P99 < 200ms |
| Barrier signal latency | Token lookup to barrier open command | P99 < 100ms |

### 2. Traffic

| Signal | Measurement |
|---|---|
| Vehicle arrival rate | Events per minute per lane |
| Pre-auth request rate | Requests per minute to NPCI |
| Kafka consumer throughput | Messages processed per second |
| Operator override rate | Manual interventions per 100 transactions |

### 3. Errors

| Signal | Measurement | Alert Threshold |
|---|---|---|
| Tag read failure rate | Failed reads / total read attempts | > 1% over 5 minutes |
| Pre-auth failure rate | Failed pre-auths / total pre-auth requests | > 2% over 5 minutes |
| Payment settlement failure rate | Failed settlements / total settlements | > 0.5% over 5 minutes |
| Kafka consumer lag | Messages behind per topic/partition | > 5,000 messages |

### 4. Saturation

| Signal | Measurement | Alert Threshold |
|---|---|---|
| AKS node CPU | CPU utilization per node | > 80% sustained 5 minutes |
| AKS node memory | Memory utilization per node | > 85% |
| Kafka broker disk | Disk utilization per broker | > 75% |
| Redis memory | Memory utilization | > 80% |
| NPCI connection pool | Active connections / pool max | > 90% |

---

## Service Level Objectives

SLOs define what "good enough" means. They are agreed between engineering and the business before any service goes to production.

### Transaction Processing SLO

```
SLO:    99.0% of vehicle passages complete in under 30 seconds
Window: Rolling 28-day
SLI:    (count of passages completing in < 30s) / (total passages)
```

### Payment Authorization SLO

```
SLO:    99.5% of pre-authorization requests succeed within 8 seconds
Window: Rolling 28-day
SLI:    (count of pre-auths completing successfully in < 8s) / (total pre-auths)
```

### System Availability SLO

```
SLO:    99.9% uptime (< 44 minutes downtime per month)
Window: Rolling 28-day
SLI:    (minutes of healthy operation) / (total minutes)
Healthy defined as: ≥ 1 lane operational per plaza AND payment processing available
```

### Audit Completeness SLO

```
SLO:    99.99% of completed barrier passages have a corresponding audit record within 60 seconds
Window: Rolling 28-day
SLI:    (audit records created within 60s of passage) / (total passages)
```

### Error Budget Policy

- Error budget is calculated weekly
- When error budget is 50% consumed, Tech Lead is notified
- When error budget is 75% consumed, Engineering Manager reviews and may pause feature work
- When error budget is exhausted, no new features ship until budget recovers

---

## Distributed Tracing

Every vehicle passage generates a distributed trace that spans the full processing path:

```
[TRACE: vehicle-passage-{uuid}]
  │
  ├── rfid-reader.read (Edge, 2ms)
  │     tag.id, signal.strength, reader.id
  │
  ├── edge-processor.publish (Edge, 5ms)
  │     kafka.topic: tag.read, kafka.offset
  │
  ├── pre-auth-service.authorize (Cloud, 2,800ms)
  │     npci.request.id, pre-auth.token, cache.hit: false
  │   ├── redis.get (2ms)
  │   └── npci.pre-authorize (2,750ms)
  │
  ├── lane-controller.token-lookup (Edge, 8ms)
  │     token.found: true, token.age_ms: 9,200
  │
  ├── lane-controller.barrier-open (Edge, 45ms)
  │     barrier.response_ms: 42
  │
  ├── payment-service.settle (Cloud, async)
  │     settlement.status: completed, amount: 65.00
  │
  └── audit-service.record (Cloud, async)
        audit.record.id, event.count: 6
```

This trace allows a single lookup to answer: "Why was this vehicle's passage slow?"

---

## AI-Assisted Root Cause Analysis

Dynatrace Davis AI continuously analyzes telemetry and correlates anomalies across signals. When a problem event is detected:

1. Davis AI identifies the affected entities (services, hosts, processes)
2. Correlates the problem with recent deployment events, configuration changes, and infrastructure events
3. Ranks probable root causes by confidence
4. Creates a problem card in the operator dashboard

For complex incidents that span multiple services, the AI Operations Assistant (see [05 — AI Enhancements](05-ai-enhancements.md)) supplements Davis AI with business context — connecting "pre-auth failure rate spiked" to "NPCI scheduled maintenance window noted in the configuration change log."

---

## Business KPI Dashboard

The operator dashboard exposes business metrics alongside engineering metrics. Business stakeholders can view:

| KPI | Description | Refresh Rate |
|---|---|---|
| Vehicles processed (per plaza) | Completed passages in last hour | 1 minute |
| Average processing time (P95) | Gate time across all lanes | 1 minute |
| Manual override rate | % of passages requiring operator | 5 minutes |
| Revenue collected | Toll amount settled (last 24h) | 5 minutes |
| Failed transaction count | Pre-auths declined or timed out | 1 minute |
| Lane availability | % of lanes operational | 30 seconds |

---

## Alerting Strategy

Alerts are tiered by impact and urgency. Alert fatigue is an active risk — every alert must require an action.

### Alert Tiers

| Tier | Definition | Response | Notification |
|---|---|---|---|
| P1 — Critical | Multiple lanes down OR payment processing failed | Immediate on-call page | PagerDuty + SMS |
| P2 — High | Single lane down OR error rate > threshold | On-call review within 15 min | PagerDuty |
| P3 — Medium | SLO burn rate elevated | Review within 1 hour | Slack |
| P4 — Low | Trend anomaly, no current impact | Review next business day | Dashboard only |

### Alert Noise Policy

- Every alert must have a runbook
- Alerts that fire more than 3 times per week without action are reviewed for suppression or threshold adjustment
- Flapping alerts (fire/resolve rapidly) are investigated for root cause — flapping is a symptom of either an unstable system or a badly calibrated alert

---

## Log Strategy

Logs are structured (JSON), correlated with traces via `trace_id` and `span_id`, and shipped to Dynatrace via the OTel collector.

**What is logged:**
- Service startup and configuration
- All external API calls (request and response status, latency)
- Error conditions with full context
- Business events (tag read, pre-auth result, barrier event) at INFO level

**What is not logged:**
- Tag IDs in plaintext in production logs (anonymized or hashed)
- Payment amounts in detail logs (only in audit service)
- High-frequency health check calls (filtered at collector)

Log retention: 30 days in Dynatrace hot storage, 1 year in Azure Blob cold archive.

---

## Incident Management

### Incident Lifecycle

```
Alert fires → On-call acknowledges (< 5 min)
           → Initial assessment (P1/P2/P3)
           → AI Ops Assistant consulted
           → Mitigation applied
           → Incident resolved
           → Postmortem scheduled (P1/P2 mandatory, P3 optional)
```

### Postmortem Policy

- P1 incidents: Postmortem within 48 hours, published to team
- P2 incidents: Postmortem within 1 week, published to team
- Postmortems are blameless — they identify systemic causes, not individual errors
- Every postmortem generates at least one backlog item

---

## Related Documents

- [04 — Target Architecture](04-target-architecture.md)
- [05 — AI Enhancements](05-ai-enhancements.md)
- [09 — Success Metrics](09-success-metrics.md)
- [ADR-004 — Observability First](../adr/ADR-004-Observability-First.md)
