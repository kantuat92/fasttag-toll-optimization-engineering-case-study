# ADR-004 — Observability-First with OpenTelemetry and Dynatrace

**Status:** Accepted  
**Date:** 2025-07-19  
**Authors:** Engineering Manager / Principal Engineer  
**Reviewers:** Platform Engineer, Tech Lead (Squad Kilo), Tech Lead (Squad Lima)  

---

## Context

The current FASTag system produces no structured telemetry. Failures are discovered when operators observe queues. Root causes are identified by manually searching logs on individual servers. There is no end-to-end visibility across the reader, backend, and payment gateway.

The new system is significantly more distributed: edge processing units, Kafka brokers, six cloud services, a payment gateway, and an AI platform all participate in a single vehicle passage. Without observability, this complexity makes the system harder to operate, not easier.

This ADR establishes observability as a first-class architectural concern — not a post-launch addition — and specifies the standards and tooling that all services must comply with.

---

## Decision

**All services must emit structured telemetry (traces, metrics, logs) via the OpenTelemetry SDK from Sprint 1. Dynatrace is the observability backend for collection, correlation, alerting, and dashboards. No service may be promoted to the production environment without an active observability dashboard and at least one defined SLO.**

This is an architectural constraint, not a guideline.

---

## Why OpenTelemetry as the Instrumentation Standard

OpenTelemetry (OTel) is the CNCF-graduated observability standard. It provides:

1. **Vendor neutrality.** OTel separates instrumentation (what data is emitted) from collection (where it goes). If Dynatrace is replaced by another backend in the future, no application code changes — only the collector configuration changes.

2. **Consistent semantic conventions.** OTel defines standard attribute names for HTTP calls, database queries, messaging systems, and more. This means traces from the Pre-Auth Service and the Lane Controller use the same attribute names for the same concepts — correlation is automatic.

3. **Automatic instrumentation for the Spring Boot + Java stack.** The OTel Java agent instruments Spring MVC, WebClient, JDBC, Kafka producer/consumer, and Redis client operations without any code change. Business-critical paths are additionally enriched with custom span attributes.

4. **Future-proof.** OTel is the direction the industry is moving. Tools built on proprietary SDKs require re-instrumentation when the backend changes.

---

## Why Dynatrace as the Observability Backend

The team has direct operational experience with Dynatrace in prior enterprise programs (AI SRE context from FedEx and JP Morgan programs). Specific capabilities that justify the choice:

1. **Dynatrace Intelligence.** Automated anomaly detection that correlates across infrastructure, APM, and logs. In practice, Dynatrace Intelligence reduces alert noise by correlating 5–10 individual threshold alerts into a single "problem" with a ranked root cause. This is measurably better than maintaining threshold alert rules manually.

2. **Full-stack topology.** Dynatrace automatically builds a service dependency map from trace data. For a distributed system with 8+ services, this topology view is invaluable during incident triage.

3. **SLO management.** Dynatrace tracks SLOs against live telemetry and provides burn rate alerts. This removes the need to build custom SLO tracking infrastructure.

4. **Log correlation.** Dynatrace correlates structured logs with trace spans via `trace_id`. A slow trace can be clicked through to the exact log lines from that execution, without a separate log query.

5. **Enterprise support and SLA.** For a financial transaction system that operates 24/7, vendor-backed support with an SLA is preferable to self-managed open-source tooling.

---

## Alternatives Considered

### Alternative 1: Prometheus + Grafana + Jaeger Stack

**Approach:** Self-managed open-source observability stack.

**Not rejected for all organizations — rejected for this program specifically.**

**Reasons:**
- Self-managing Prometheus, Grafana, Jaeger, and Loki at production scale requires dedicated platform engineering effort that this team does not have.
- No built-in AI-assisted anomaly detection. Threshold-based alerting in Prometheus requires manual tuning and produces higher alert noise.
- No out-of-the-box SLO management.
- Log-trace correlation requires manual setup across multiple tools.

This stack is appropriate for teams with dedicated observability engineering resources. For a delivery program of this size, managed tooling provides better ROI.

---

### Alternative 2: Azure Monitor + Application Insights

**Approach:** Native Azure observability stack.

**Partially accepted:** Azure Monitor is used for infrastructure-level metrics (AKS node health, Azure Service Bus metrics). Application-level observability uses Dynatrace.

**Not chosen as the primary APM because:**
- Application Insights does not have Dynatrace's AI-assisted root cause analysis capability.
- Distributed tracing across edge units and cloud services is more mature in Dynatrace.
- The team's prior operational experience is with Dynatrace, reducing the learning curve.

---

### Alternative 3: Datadog

**Approach:** Datadog as the observability backend.

**Technically valid alternative.** Not chosen because:
- Dynatrace has stronger AI-assisted anomaly detection (Davis AI vs. Watchdog).
- Team has direct operational experience with Dynatrace, reducing implementation risk.
- Dynatrace licensing is already under evaluation for enterprise procurement.

If Dynatrace licensing does not proceed, Datadog is the first alternative.

---

## Implementation Standards

### Mandatory for All Services

Every service that runs in any environment beyond a developer laptop must:

1. **Export traces via OpenTelemetry to the OTel Collector.** The collector is deployed as a DaemonSet on AKS and as a sidecar on edge units.

2. **Emit structured logs in JSON format** with `trace_id` and `span_id` included on every log line during an active span.

3. **Expose a `/health` endpoint** (Spring Boot Actuator) and a `/metrics` endpoint in Prometheus format. Both are scraped by the OTel Collector.

4. **Attach mandatory span attributes:**
   - `fasttag.plaza.id`
   - `fasttag.lane.id`
   - `service.name`
   - `service.version`

5. **Define at least one SLO** in the Dynatrace SLO configuration before promotion to production.

6. **Have a named owner in the Dynatrace service catalog.** No orphan services.

### Trace Sampling

- Development and staging: 100% trace sampling
- Production: 10% head-based sampling, 100% for error traces and traces exceeding P95 latency threshold
- Tail-based sampling is evaluated for Phase 3 if sampling overhead becomes measurable

### Alert Minimum Requirements

Every alert that fires on-call must have:
- A named owner (squad-level)
- A linked runbook in the knowledge base
- A defined SLO it protects
- A review date (alerts older than 90 days without a review are automatically set to warning-only)

---

## Consequences

### Positive

- End-to-end trace visibility from RFID read at edge to settlement confirmation at cloud, including external payment gateway calls.
- Incident triage time reduced from hours to minutes — engineers navigate from alert to root cause via trace rather than manual log correlation.
- SLO compliance is automatically tracked; error budget burn rate is visible in real time.
- New engineers can understand the system behavior by examining live traces — not just reading documentation.

### Negative

- Dynatrace licensing cost. For a system of this size, annual licensing is in the tens of thousands of dollars. This is justified by the reduction in incident resolution time and the reduction in manual operations overhead.
- OTel agent adds a small CPU and memory overhead to each service (approximately 2–5% CPU, < 100MB RAM). This is factored into AKS resource requests.
- The observability stack itself must be monitored. Collector failure means telemetry gaps. Collector health is monitored via Azure Monitor.

### Constraints This Decision Places on the Design

- Services cannot use proprietary instrumentation SDKs (Dynatrace OneAgent auto-instrumentation is acceptable as a complement, not a replacement for OTel).
- Log format is standardized across all services — no service-specific log formats.
- SLO definitions must be agreed before a service enters production. This adds a planning step to the Definition of Done.

---

## Rollout

- **Sprint 1:** OTel Collector deployed, Dynatrace connected, edge unit telemetry pipeline validated
- **Sprint 2:** All services in Squad Lima emit traces and metrics
- **Sprint 3:** All services in Squad Kilo emit traces and metrics
- **Sprint 4:** First SLOs defined and dashboards live
- **Phase 3:** Dynatrace Davis AI tuned on production data; alerting reviewed and refined

Observability is not "added in Sprint 5." It is a delivery criterion from Sprint 1.

---

## Future Considerations

- **Continuous profiling.** Dynatrace supports continuous profiling in production. Useful for identifying CPU-intensive code paths at scale. Evaluate after 3 months of production data.
- **OpenTelemetry Metrics for custom business KPIs.** The OTel Metrics SDK can be used to emit business metrics (vehicles processed, revenue collected per hour) directly from services. This removes the dependency on a separate BI pipeline for operational dashboards.
- **OpenTelemetry Protocol (OTLP) to multiple backends.** If the architecture evolves to require multiple observability backends (e.g., Dynatrace for APM, ClickHouse for long-term analytics), the OTel Collector fan-out configuration supports this without application changes.

---

## References

- [07 — Observability](../docs/07-observability.md)
- [05 — AI Enhancements](../docs/05-ai-enhancements.md)
- [ADR-003 — AI Operations Assistant](ADR-003-AI-Operations-Assistant.md)
