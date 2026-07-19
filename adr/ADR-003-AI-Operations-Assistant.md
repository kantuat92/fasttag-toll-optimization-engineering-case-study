# ADR-003 — AI-Assisted Operations and Root Cause Analysis

**Status:** Accepted  
**Date:** 2025-07-19  
**Authors:** Engineering Manager / Principal Engineer  
**Reviewers:** Tech Lead (Squad Lima), Platform Engineer  

---

## Context

Toll plaza operations run 24 hours a day, 7 days a week. Incidents — payment gateway degradation, reader hardware faults, upstream network loss — occur outside business hours and require on-call engineers to diagnose and resolve within minutes.

In the current system, incident response means manually searching logs, correlating timestamps, and relying on institutional knowledge of which logs to check and in what order. This takes 30–60 minutes for a non-obvious root cause.

In the new system, distributed processing across edge units, Kafka, and cloud services produces rich telemetry — but also more data to correlate during an incident. Without tooling, more observability can mean more noise, not faster resolution.

The question this ADR addresses is: how should AI be applied to operations to reduce incident resolution time, and what constraints must be placed on that AI to keep it safe in a financial transaction system?

---

## Decision

**Implement an AI Operations Assistant that provides advisory root cause analysis during incidents, powered by the Anthropic Claude API (claude-Opus-4.8), integrated with the operations platform's observability telemetry.**

The assistant:
- Is triggered automatically when a P1 or P2 alert fires
- Receives a structured context window containing relevant telemetry (pre-filtered, not raw)
- Returns a ranked list of probable root causes with supporting evidence
- Suggests a next diagnostic step

The assistant **does not:**
- Execute any action autonomously
- Send alerts or notifications directly
- Modify system configuration
- Open or close barriers

Every AI output is advisory. The on-call engineer makes all decisions.

---

## Why Generative AI for This Problem

Rule-based alerting (Dynatrace Inteliigence, threshold-based alerts) handles well-known failure patterns effectively. It detects "NPCI gateway latency exceeded threshold" reliably.

Where it fails is in **cross-signal correlation for novel failure patterns**. A problem that manifests as "high manual override rate" may be caused by a reader hardware issue, a payment gateway degradation, a Kafka consumer lag, or an application bug — each with a different resolution. A rules engine would fire an alert; it cannot reason about which of four possible causes is most likely given the current state of all four signals simultaneously.

A large language model with access to structured telemetry context can:
- Apply reasoning across multiple signals simultaneously
- Incorporate knowledge of the system's architecture (from its system prompt)
- Produce a ranked, evidence-backed hypothesis rather than a single alert

This is a narrow but valuable application of generative AI in operations.

---

## Alternatives Considered

### Alternative 1: Enhanced Dynatrace  Intelligence Only

**Approach:** Rely entirely on Dynatrace Intelligence AI for anomaly detection and correlation.

**Partially accepted:** Dynatrace Intelligence AI is the primary anomaly detection layer. It handles known failure patterns and provides the initial problem event.

**Not sufficient alone because:** Dynatrace Intelligence AI correlates infrastructure-level signals well. It does not have knowledge of the FASTag domain — it cannot reason about "reader health degradation + payment gateway slowness = likely cascade, check NPCI status page first." The AI Operations Assistant adds this domain reasoning layer on top of Dynatrace Intelligence AI.

---

### Alternative 2: On-Call Runbook Only

**Approach:** Write detailed runbooks covering all known failure scenarios. Train on-call engineers to follow them.

**Partially accepted:** Runbooks exist and are required for all P1/P2 scenarios. They cover known failure modes.

**Not sufficient alone because:** Novel failures — scenarios not covered by a runbook — are the ones that take 60 minutes to resolve. The AI assistant helps with precisely these cases by reasoning from telemetry rather than matching to a known pattern.

---

### Alternative 3: Custom ML Model for RCA

**Approach:** Train a classification model on historical incident data to predict root cause.

**Rejected because:**
- Insufficient incident history at project start to train a meaningful model
- Classification models require labeled training data for every failure type they must handle
- Novel failure modes (which are the hard cases) are by definition not in the training set
- LLMs can reason about novel situations using their pre-trained knowledge of distributed systems; a narrow classifier cannot

---

### Alternative 4: Open-Source LLM (Self-Hosted)

**Approach:** Deploy an open-source model (LLaMA, Mistral) on Azure ML for data residency control.

**Not rejected — it is the migration path if data residency requirements change.**

**Chosen against at this stage because:**
- Self-hosting adds significant infrastructure and operational overhead
- Claude API latency (< 3 seconds for structured prompts) is acceptable for incident response
- Telemetry data sent to the API is pre-filtered: tag IDs are anonymized, payment amounts are not included. Data sent contains only system metrics and timestamps.
- If regulatory requirements demand full data residency, this is the defined migration path.

---

## Implementation Design

### Context Window Construction

The assistant receives a structured JSON context window assembled from live telemetry. The context is pre-filtered to reduce token cost and latency:

```json
{
  "alert": {
    "name": "PreAuthFailureRateHigh",
    "fired_at": "2025-07-19T14:32:00Z",
    "current_value": "18%",
    "threshold": "2%"
  },
  "recent_metrics_5min": {
    "npci_gateway_p99_ms": 8400,
    "npci_gateway_p99_baseline_ms": 420,
    "pre_auth_service_consumer_lag": 12000,
    "pre_auth_service_pod_restarts": 2,
    "kafka_broker_health": "healthy",
    "redis_hit_rate": 0.94
  },
  "recent_deployments": [],
  "recent_alerts_1h": ["PreAuthServicePodOOM"],
  "system_context": "Pre-Auth Service calls NPCI gateway synchronously. Consumer lag indicates queued events. Pod restarts suggest memory pressure."
}
```

### Prompt Design

```
System: You are an operations assistant for the FASTag toll processing system. 
You receive structured telemetry and return a ranked list of probable root causes 
with supporting evidence. Be specific. Reference the exact metric values provided. 
Do not speculate beyond the data. If the data is insufficient to determine root cause, 
say so and specify what additional data would help.

User: [context window JSON]
```

### Output Format

The assistant returns structured output that the operator dashboard renders:

```json
{
  "analysis": {
    "confidence_ranked_causes": [
      {
        "cause": "NPCI payment gateway degradation",
        "confidence": 0.85,
        "evidence": ["npci_gateway_p99 increased from 420ms to 8400ms at 14:32", "Pre-auth consumer lag +12000 messages consistent with slow gateway responses"],
        "next_step": "Check NPCI status page. If confirmed, activate cache-fallback mode."
      },
      {
        "cause": "Pre-Auth Service memory pressure causing GC pauses",
        "confidence": 0.10,
        "evidence": ["2 pod restarts in last 10 minutes (OOM)", "Could contribute to processing delay"],
        "next_step": "Check Pre-Auth Service pod memory metrics. If OOM confirmed, increase memory limit."
      }
    ],
    "data_gaps": ["NPCI external status not available in telemetry — manual check required"]
  }
}
```

---

## Guardrails

1. **Read-only context.** The assistant has no write access to any system. It cannot execute commands, modify configuration, or trigger deployments.

2. **Anonymized telemetry.** Tag IDs are hashed before inclusion in the context window. Payment amounts are excluded. Only system metrics and timestamps are sent.

3. **Human confirmation required.** Every suggested action requires explicit operator confirmation in the dashboard. The dashboard renders suggestions as "SUGGESTED: [action] — Confirm?" not as completed actions.

4. **Fallback on API failure.** If the Claude API is unreachable, the operations dashboard falls back to standard alert + runbook mode. The AI assistant is an enhancement, not a dependency.

5. **Cost controls.** Each context window is capped at 2,000 tokens. API calls are rate-limited to 20 per hour. Estimated monthly cost: < $50 USD at typical incident rates.

---

## Consequences

### Positive

- Incident RCA time reduced from 30–60 minutes to under 5 minutes for covered scenarios
- On-call burden reduced — less expertise required to navigate the first 10 minutes of a novel incident
- Incident context is always available in structured form (the context window becomes the incident record)

### Negative

- Dependency on external API for incident support (mitigated by fallback mode)
- Risk of over-reliance: engineers may defer to AI recommendations without independent verification. Training and runbooks must make clear that the AI is advisory.
- API cost is bounded but not zero.

---

## Future Considerations

- **Fine-tuning on incident history.** After 6 months of production incidents, consider fine-tuning a model specifically on FASTag incident patterns. This improves confidence ranking accuracy for domain-specific failure modes.
- **Postmortem automation.** Extend the assistant to draft postmortem documents from the incident context window and resolution steps (see [05 — AI Enhancements](../docs/05-ai-enhancements.md)).
- **Migration to self-hosted model.** If data residency requirements change, self-hosted deployment on Azure ML is the defined migration path. The integration interface (structured JSON in/out) is model-agnostic.

---

## References

- [05 — AI Enhancements](../docs/05-ai-enhancements.md)
- [07 — Observability](../docs/07-observability.md)
- [ADR-004 — Observability First](ADR-004-Observability-First.md)
