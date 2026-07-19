# 05 — AI Enhancements

## Purpose

Describe where AI adds measurable value in this system, how each capability is built, and what the engineering and business impact is.

AI is applied where it solves a specific operational problem that rules-based systems handle poorly. It is not applied as a feature addition or to meet a mandate.

---

## Guiding Principle

Every AI feature in this system must satisfy three conditions before it is built:

1. There is a specific, named problem it solves
2. The problem cannot be solved as well with a simpler rules-based approach
3. The cost of being wrong is bounded and the fallback is defined

---

## AI Capability Map

| Capability | Problem | Where It Runs |
|---|---|---|
| RFID Reader Failure Prediction | Readers degrade silently before failing | Operations Platform |
| Traffic Queue Forecasting | Lane assignment is reactive, not planned | Lane Management Service |
| Fraud Detection | Tag cloning and velocity fraud go undetected | Fraud Detection Service |
| AI Operations Assistant | Incident RCA takes 40+ minutes of manual log triage | Operations Platform |
| GenAI Runbook Generation | Runbooks are stale, missing, or inaccurate | Operations Platform |
| AI-Assisted Postmortem | Post-incident analysis is incomplete and slow | Operations Platform |

---

## 1. RFID Reader Failure Prediction

### Problem

RFID readers degrade before they fail. A reader with a dirty antenna, a loose connection, or firmware instability will show increasing read retry rates, decreasing signal strength variance, and intermittent timeouts — days before it stops working completely.

Today, the only signal is a fully failed reader. By then, the lane is already down.

### Approach

A time-series anomaly detection model runs over the telemetry stream from each RFID reader:

- **Features:** Read success rate (rolling 5-min window), signal strength distribution, retry count, read latency P95, temperature (if sensor available)
- **Model:** LSTM-based anomaly detection (trained on historical failure data from comparable deployments). Initial deployment uses a simpler isolation forest until sufficient training data exists.
- **Threshold:** When degradation probability exceeds 0.7, a `ReaderHealthAlert` event is published
- **Output:** Maintenance ticket auto-created in the operations dashboard with predicted time-to-failure range

### Training Data Strategy

Reader telemetry is collected from day one of deployment via OpenTelemetry. The first 90 days use rule-based anomaly detection (e.g., >5% retry rate increase over 24h). Model training begins when 90 days of labeled degradation events are available.

This avoids the cold-start problem of deploying a model with no training data.

### Business Value

- Maintenance is scheduled during off-peak hours rather than during peak congestion
- Estimated reduction in unplanned lane downtime: 70%
- Each avoided unplanned outage prevents approximately 40 minutes of disruption

### Engineering Impact

- Reader telemetry must be structured and emitted via OpenTelemetry from day one
- Model inference runs as a sidecar to the Operations Platform — no latency impact on gate processing
- False positive rate must be bounded: a maintenance alert for a healthy reader costs one technician visit. A missed failure costs 40 minutes of lane downtime. The cost asymmetry means the model should be tuned toward recall, not precision.

---

## 2. Traffic Queue Forecasting

### Problem

Lane assignment at toll plazas today is static or manually adjusted by operators. During peak hours, ETC lanes fill while manual lanes remain underused, or vice versa. Express lanes are opened reactively — after queues form.

### Approach

A time-series forecasting model predicts vehicle arrival rate per lane over the next 15 and 30 minutes:

- **Features:** Time of day, day of week, historical arrival rates (same hour, same day), upstream traffic sensor data (if available from NHAI), weather conditions, local event data
- **Model:** Facebook Prophet for baseline forecasting (interpretable, handles seasonality well). Replaced with LightGBM ensemble if accuracy requirements are not met after initial deployment.
- **Output:** Predicted vehicles/minute per lane, published to Lane Management Service every 5 minutes
- **Action:** Lane Management Service adjusts express lane activation thresholds based on forecast

### Business Value

- Dynamic lane pre-activation reduces queue formation
- Estimated throughput improvement during peak hours: 15–25%
- Operators receive a 15-minute advance warning before congestion is predicted

### Engineering Impact

- Historical traffic data must be stored and accessible to the model training pipeline
- Azure ML pipelines handle retraining on a weekly schedule
- Lane Management Service consumes forecasts as advisory signals — lane decisions are always overridable by operators

---

## 3. Fraud Detection

### Problem

Tag cloning — copying a valid RFID tag's ID to a blank tag and using it on a different vehicle — is a known fraud vector in the FASTag ecosystem. The current system has no mechanism to detect it. A cloned tag transacts normally until the legitimate tag owner notices unexplained deductions.

A secondary fraud pattern is insider manipulation — manual override triggered without a legitimate failure, waiving toll charges.

### Approach

Two complementary detection mechanisms:

**Geographic Impossibility Detection:**
- Consume `TagRead` events across all plazas in real time
- If the same tag ID appears at two geographically separated plazas within a time window that makes travel impossible (distance / max highway speed), a `FraudSignalRaised` event is published
- This is a rule-based check, not ML — it is simple, fast, and has a near-zero false positive rate

**Velocity and Pattern Anomaly (ML):**
- Features: Tag usage frequency, time-of-day patterns, vehicle class vs. plaza type, operator override rate per lane
- Model: Gradient boosting classifier trained on labeled fraud cases
- Output: Fraud probability score per transaction, thresholded to raise signals above 0.8
- High-score transactions flagged for asynchronous review — not blocked in real time (to avoid false positives creating lane delays)

### Business Value

- Revenue protection from cloned tag usage
- Insider fraud detection reduces risk of regulatory non-compliance
- Fraud signals fed back to NHAI for blacklist updates

### Engineering Impact

- Fraud detection is asynchronous — it does not sit in the gate decision path
- A `FraudSignalRaised` event triggers a tag suspension workflow that the operator must confirm before a tag is blacklisted
- False positive review process must be defined before deployment — the system should not auto-blacklist

---

## 4. AI Operations Assistant

### Problem

When an incident occurs — a lane goes down, payment authorizations start failing, manual override rates spike — the on-call engineer opens a terminal and begins manually correlating logs, metrics, and traces. This typically takes 30–60 minutes to identify the root cause.

In a system processing 500+ vehicles per hour per plaza, a 30-minute RCA delay is operationally unacceptable.

### Approach

An AI Operations Assistant is embedded in the operator dashboard. It has access to:
- OpenTelemetry traces (via Dynatrace API)
- Kafka consumer lag metrics
- Payment gateway response time histogram
- Reader health telemetry
- Recent deployment events
- Alert history

When an alert fires, the assistant:

1. Correlates recent anomalies across telemetry signals
2. Identifies the most likely root cause with supporting evidence
3. Presents a ranked list of causes with confidence levels
4. Suggests the next diagnostic step

The assistant is powered by a Claude API integration (Anthropic SDK, Java) with a structured context window containing the relevant telemetry summary and a system prompt trained on incident history.

**This is not a chatbot.** It is a structured reasoning tool that operates on real observability data. The operator makes the decision; the assistant surfaces the evidence.

### Example Interaction

```
[ALERT] Payment pre-authorization failure rate: 18% (threshold: 2%)

AI Assistant Analysis:
──────────────────────────────────────────────────────
Root Cause Confidence:
  85% — NPCI gateway latency spike (P99: 8,400ms, baseline: 420ms)
  10% — Network degradation at AZ-East-2 (packet loss: 0.2%)
   5% — Pre-Auth Service pod OOM (2 restarts in last 10 min)

Supporting Evidence:
  • NPCI gateway P99 latency increased from 420ms → 8,400ms at 14:32 UTC
  • Pre-Auth Service consumer lag: +12,000 messages over 8 minutes
  • No deployment events in last 2 hours
  • Payment gateway status page: [check recommended]

Suggested Next Step:
  Check NPCI status page. If gateway degraded, activate balance-cache
  fallback mode via: kubectl set env deploy/pre-auth-service CACHE_FALLBACK=true
──────────────────────────────────────────────────────
```

### Business Value

- RCA time reduced from 30–60 minutes to under 5 minutes
- On-call burden reduced — less expertise required to navigate the first 10 minutes of an incident
- Incident patterns feed back into the AI training corpus

### Engineering Impact

- Claude API key and prompt templates stored in Azure Key Vault
- Telemetry summarization runs on a scheduled Kubernetes Job triggered by alert events
- Context window must be bounded — telemetry is pre-filtered to relevant signals before LLM call
- Human-in-the-loop always: assistant suggestions are advisory, not automated actions

---

## 5. GenAI Runbook Generation

### Problem

Operational runbooks are written once and rarely updated. Within 6 months of a system change, a runbook is likely to reference services, commands, or configurations that no longer exist. On-call engineers either work from outdated runbooks or have no runbook at all.

### Approach

GenAI runbook generation uses the live system state as its source of truth:

- Triggered when a new alert definition is created or an existing runbook is older than 90 days
- Context includes: service topology (from service mesh metadata), recent alerts and their resolutions, OpenTelemetry service map, deployment manifest
- Output: A structured markdown runbook with diagnostic steps, rollback procedures, and escalation criteria
- Runbooks are stored in the repository and require human review before activation

The Anthropic Claude API (via Java SDK) generates the runbook. The prompt is structured to produce specific, actionable steps — not generic advice.

### Business Value

- Runbooks stay current with minimal manual effort
- New team members can operate the system safely from day one
- Reduces the "knowledge in people's heads" problem

### Engineering Impact

- Runbook generation is a background process — no latency impact on operations
- Generated runbooks are pull-request-gated before they replace existing ones
- Prompt templates are version-controlled alongside code

---

## 6. AI-Assisted Postmortem

### Problem

Postmortems are time-consuming to write and often incomplete. The on-call engineer reconstructs the incident timeline from memory and partial logs, hours or days after the event.

### Approach

After an incident is closed, the AI postmortem assistant:

1. Reconstructs the incident timeline from distributed traces and alert history
2. Identifies contributing factors from telemetry correlation
3. Generates a draft postmortem in the team's standard format
4. Highlights gaps in observability that made diagnosis harder

The draft is reviewed and amended by the engineering team before publication. The assistant writes the first draft; the engineers write the conclusions.

### Business Value

- Postmortem quality is consistent regardless of author
- Timeline reconstruction is accurate because it comes from telemetry, not memory
- Observability gaps are systematically identified and tracked

---

## AI Platform Architecture

```
┌────────────────────────────────────────────────────────────┐
│  AI / ML Platform (Azure ML + Custom Inference Services)   │
│                                                            │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ Reader Failure  │  │ Queue Forecast  │                 │
│  │ Prediction      │  │ Model           │                 │
│  │ (LSTM / IForest)│  │ (Prophet / LGBM)│                 │
│  └─────────────────┘  └─────────────────┘                 │
│                                                            │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ Fraud Detection │  │ AI Ops          │                 │
│  │ (GBM Classifier)│  │ Assistant       │                 │
│  │ + Geo Rules     │  │ (Claude API)    │                 │
│  └─────────────────┘  └─────────────────┘                 │
│                                                            │
│  Training Pipeline: Azure ML Pipelines (weekly schedule)  │
│  Model Registry: Azure ML Model Registry                  │
│  Feature Store: Redis (real-time) + PostgreSQL (batch)    │
└────────────────────────────────────────────────────────────┘
```

---

## What AI Does Not Do in This System

- AI does not open or close barriers autonomously
- AI does not blacklist tags without operator confirmation
- AI does not modify lane assignments without operator review
- AI does not process payment transactions

Every AI output is advisory. Operators retain final authority over all gate-level decisions.

---

## Related Documents

- [04 — Target Architecture](04-target-architecture.md)
- [07 — Observability](07-observability.md)
- [ADR-003 — AI Operations Assistant](../adr/ADR-003-AI-Operations-Assistant.md)
