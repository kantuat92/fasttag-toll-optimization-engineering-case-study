# FASTag Toll Optimization
## Engineering Proposal — Executive Presentation

> 10 slides · Prepared for NHAI / ABC Company review panel

---

---

## Slide 01 — The Problem Is Not What It Looks Like

**FASTag is electronic. It is not fast.**

The automation is shallow: RFID replaced cash, but the same sequential processing model remained.

```
Vehicle arrives → Read tag → Call payment gateway → Write audit log → Open barrier
```

Every step blocks the next.  
The payment gateway alone takes **1.5–45 seconds** depending on load.  
The barrier waits for all of it.

**This is not a hardware problem. It is an architecture problem.**

---

**One number:**

> A single payment gateway timeout (3–5% of transactions)  
> causes the queue behind it to grow by 10–15 vehicles before the lane recovers.

---

---

## Slide 02 — Root Causes (Not Symptoms)

| # | Root Cause | Impact |
|---|---|---|
| 1 | Reader positioned at the barrier | Zero pre-processing time |
| 2 | All steps synchronous, single chain | Payment gateway in critical path |
| 3 | No offline capability | Gateway outage = lane shutdown |
| 4 | Single RFID reader per lane | Hardware failure = manual queue |
| 5 | Audit logging in critical path | +150ms every transaction, no benefit |
| 6 | No observability | Failures discovered by watching queues form |

**Fixing any one of these helps. Fixing all six achieves the 30-second target.**

---

---

## Slide 03 — The Core Idea: Move Work Left

**Today:** All processing happens at the barrier. The vehicle waits.

**Proposed:** Processing starts 150m before the barrier. The vehicle does not wait.

```
TODAY:
  Vehicle stops at gate → [3–5 minutes of processing] → Gate opens

PROPOSED:
  Vehicle 150m away → [pre-authorization completes in 3–5s] →
  Vehicle arrives at gate → [token lookup: 100ms] → Gate opens
```

**The gate decision becomes a cache lookup, not a payment transaction.**

---

**The shift:**

> Pre-authorization at 150m gives 13 seconds of processing time  
> before the vehicle reaches the barrier.  
> At P95 NPCI latency, this is sufficient.  
> The barrier opens within 2 seconds of vehicle arrival.

---

---

## Slide 04 — Architecture in 60 Seconds

```
[RFID Reader @ 150m]
        │ TagRead event
        ▼
[Edge Processing Unit] ──── [Local Redis Cache]
        │                          │
        │ Kafka                    │ token stored here
        ▼                          │
[Pre-Auth Service] ──── NPCI ─────┘
        │ PreAuthApproved
        ▼
[Kafka Event Bus]
        │
        ├──▶ [Payment Service] ──── settlement (async)
        ├──▶ [Audit Service]   ──── audit record (async)
        ├──▶ [Fraud Service]   ──── anomaly scoring (async)
        │
        ▼
[Vehicle arrives at barrier]
[Lane Controller] ──── Redis token lookup (100ms) ──── Barrier opens ✓
```

**Three principles:**
1. Pre-compute before the vehicle arrives
2. Async everything that is not the gate decision
3. Edge cache enables offline operation

---

---

## Slide 05 — Failure Modes Are Designed, Not Discovered

| Failure | Current State | Proposed State |
|---|---|---|
| Payment gateway unreachable | Lane shutdown | Cache fallback + async reconciliation |
| RFID reader hardware fault | Operator discovers when queue forms | AI predicts failure 24–48h ahead |
| Central cloud unreachable | All lanes affected | Edge autonomy — local tokens continue |
| Tag balance insufficient | Vehicle blocks lane 2–5 min | Declined at 150m — routed to manual before barrier |
| Double charge on retry | No protection | Idempotent payment API + dedup at audit layer |

**Every failure has a defined behaviour before the system ships.**

Graceful degradation is an architectural requirement, not an operational afterthought.

---

---

## Slide 06 — AI That Earns Its Place

AI is applied where it solves a specific problem that rules-based systems handle poorly.

| Capability | Problem | Not Covered By Rules Because |
|---|---|---|
| **Reader Failure Prediction** | Readers degrade silently | Degradation pattern is multi-signal, temporal |
| **Traffic Queue Forecasting** | Lane assignment is reactive | Historical + contextual pattern, not threshold |
| **Fraud Detection** | Tag cloning undetected | Geographic impossibility + velocity pattern |
| **AI Ops Assistant** | RCA takes 40+ minutes | Cross-signal correlation across 8+ services |
| **GenAI Runbooks** | Runbooks are stale or missing | Generated from live system state, not memory |

**What AI does not do:**
- Open or close barriers
- Process payments
- Blacklist tags without operator confirmation

Every AI output is **advisory**. Operators make decisions.

---

---

## Slide 07 — Observability Is Structural

**The current system has no telemetry. Failures are visible when queues form.**

The proposed system emits structured telemetry from every component.

```
RFID Read → Edge Processor → Kafka → Pre-Auth → NPCI → Redis → Lane Controller
   │              │            │          │         │        │          │
   └──────────────┴────────────┴──────────┴─────────┴────────┴──────────┘
                          OpenTelemetry Trace
                          (one trace per vehicle passage)
```

**A single trace answers:** "Why was this vehicle's passage slow?"  
Without correlation. Without manual log digging.

**Stack:** OpenTelemetry SDK → OTel Collector → Dynatrace (Davis AI, SLOs, Dashboards)

**No service goes to production without:**
- SLO defined
- Dashboard wired
- Alert with runbook

---

---

## Slide 08 — How We Deliver

**Two tracks run in sequence: build, then expand.**

### Track 1 — Engineering Delivery (Weeks 1–26)

| Phase | Weeks | Deliverable |
|---|---|---|
| Foundation | 1–6 | AKS, Kafka, CI/CD, observability — before first feature |
| Core Flow | 7–14 | End-to-end passage at test plaza in < 30 seconds |
| AI & Hardening | 15–20 | AI Ops Assistant, express lanes, chaos testing |
| Pilot | 21–26 | Single lane live in production, 30-day gate |

### Track 2 — Geographic Rollout (Weeks 21+)

Each stage has an exit gate. No stage begins on a schedule — it begins when the previous gate is cleared.

| Stage | Scope | Gate Before Proceeding |
|---|---|---|
| **Stage 0** | 1 lane, 1 plaza | P95 < 30s for 30 consecutive days |
| **Stage 1** | All lanes, full plaza | Peak throughput ≥ 60 vehicles/hr/lane |
| **Stage 2** | 1 NH corridor (8–15 plazas) | Multi-plaza ops + fraud detection validated |
| **Stage 3** | All contracted NH corridors | Operational repeatability, no engineering dependency |

---

**Non-negotiable delivery practices:**
- DORA metrics tracked from Sprint 1
- Definition of Done: SLO defined + dashboard wired + runbook exists
- No deployments without automated rollback path
- Blameless postmortems within 48 hours of P1

---

**DORA targets (steady state):**

| Metric | Target |
|---|---|
| Deployment frequency | Multiple per week |
| Lead time for changes | < 2 days |
| Change failure rate | < 5% |
| MTTR | < 30 minutes |

---

---

## Slide 09 — Success Looks Like This

**6 months post-pilot:**

| Metric | Current | Target |
|---|---|---|
| Processing time (P95) | 3–5 minutes | **< 30 seconds** |
| Manual intervention rate | ~15% | **< 2%** |
| Lane throughput (peak) | ~12 vehicles/hr | **≥ 60 vehicles/hr** |
| Unplanned lane downtime | Unknown | **< 30 min/month** |
| System availability | Unknown | **99.9%** |
| Incident RCA time | 30–60 minutes | **< 5 minutes** |

**How we know we've succeeded:**  
SLOs measured continuously. Error budget tracked weekly.  
Not a milestone survey. Not a demo.

---

---

## Slide 10 — Why This Approach

Three architectural decisions drive the outcome:

**1. Pre-stage reading at 150m**  
Moves the gate decision from "payment complete?" to "token exists?".  
A cache lookup is 100ms. A payment gateway call is 1,500–45,000ms.

**2. Event-driven processing**  
Decouples gate decisions from settlement, audit, fraud, and analytics.  
Each concern is independent, scalable, and deployable separately.

**3. Observability first**  
The current system is opaque. The new system is measured.  
You cannot improve what you cannot observe. You cannot operate what you cannot trace.

---

**What this is not:**

- It is not a proof of concept. It is a production architecture.
- It is not a technology refresh. Every choice is justified by a specific operational problem.
- It is not novel. The patterns are proven. The discipline is in applying them correctly.

---

*Full technical detail: [docs/](../docs/) · Architecture decisions: [adr/](../adr/) · Diagrams: [diagrams/](../diagrams/)*
