# 01 — Business Problem

## Purpose

Frame the business problem with sufficient precision that architectural decisions can be traced back to it. A vague problem statement leads to architecture that solves the wrong thing.

---

## Background

The National Highways Authority of India (NHAI) mandated FASTag as the sole electronic toll collection mechanism across national highways. The intent was to eliminate cash queues, reduce transaction time, and build a foundation for data-driven highway management.

ABC Company has been contracted to optimize toll collection across several high-traffic corridors. The mandate is specific: reduce end-to-end processing time from the current 3–5 minutes to **under 30 seconds**, measured from when a vehicle enters the toll lane to when the barrier opens.

---

## The Problem

FASTag is electronic, but it is not fast. The automation is shallow — the system replaced cash with RFID, but kept the same sequential, gate-blocked processing model.

Vehicles stop. The reader activates. A synchronous chain of network calls fires. The barrier waits for all of them to complete. Nothing is done in advance. If any link in the chain fails, a human operator intervenes — blocking every vehicle behind.

**This is not a hardware limitation. It is an architecture problem.**

---

## Current Processing Model

The existing system processes every step sequentially at the barrier — tag read, payment gateway call, audit write, then barrier signal. Nothing is done before the vehicle arrives. Every failure cascades to the vehicles queued behind.

The full root cause analysis is in [02 — Current State](02-current-state.md). The relevant point here is that the 3–5 minute delay is structural, not incidental. It cannot be fixed by faster hardware or a better network connection alone.

---

## Business Impact

### Revenue Impact

- A 3-minute average delay with 30-second theoretical capacity means **6× throughput reduction per lane**.
- Peak-hour congestion spills onto access roads, reducing effective capacity further.
- Manual lane fallback incurs cash handling cost, fraud risk, and reconciliation overhead.

### User Experience

- FASTag was mandated to improve driver experience. A 3-minute wait in an "automated" lane erodes public trust in the system.
- Repeat violations (insufficient balance, failed reads) create disproportionate delays — one failure can hold 10–15 vehicles.

### Operational Cost

- Manual intervention staff must be stationed at every lane as a permanent fallback.
- RFID reader failures are discovered reactively — when vehicles start queuing, not before.
- No visibility into where delays originate means every incident requires manual triage.

### Regulatory and Contractual Risk

- SLA with NHAI requires demonstrable improvement in throughput and processing time.
- Repeated failures or congestion events carry penalty clauses.
- Audit trail requirements for every financial transaction are non-negotiable.

---

## Stakeholders

| Stakeholder | Interest | Concern |
|---|---|---|
| NHAI | Highway throughput, SLA compliance | Congestion, public trust |
| ABC Company | Contract delivery, margin | Cost overrun, delivery risk |
| Commuters | Fast, reliable passage | Delay, double charge, failed reads |
| Banks / NPCI | Transaction success | Fraud, reconciliation failures |
| Toll Operators | Manageable workload | System complexity, manual overload |
| Engineering Team | Clear requirements, buildable system | Scope creep, unclear NFRs |

---

## Scope

This proposal covers:

- Software architecture for FASTag lane optimization
- Pre-stage RFID detection and pre-authorization
- Event-driven processing pipeline
- Express lane management
- AI-assisted operations
- Observability and incident management

This proposal explicitly **does not** cover:

- NHAI backend systems integration (treated as external dependency)
- Physical hardware selection (assumed to be within ABC Company's hardware procurement scope)
- Billing and reconciliation with bank networks beyond the toll transaction

---

## Success Definition

| Metric | Current | Target |
|---|---|---|
| End-to-end processing time | 3–5 minutes | < 30 seconds |
| Manual intervention rate | ~15% of transactions | < 2% |
| Tag read failure rate | Unknown (no observability) | < 0.5% |
| Throughput per lane (peak hour) | ~12 vehicles/hour | ~60 vehicles/hour |
| System availability | Unknown | 99.9% |

---

## Related Documents

- [00 — Engineering Analysis](00-engineering-analysis.md)
- [02 — Current State](02-current-state.md)
- [03 — Assumptions](03-assumptions.md)
- [ADR-001 — Early RFID Detection](../adr/ADR-001-Early-RFID-Detection.md)
