# 09 — Success Metrics

## Purpose

Define what success looks like in measurable terms. Metrics that are not defined before delivery become arguments after delivery. Every metric here has a definition, a measurement method, and a target.

---

## Measurement Principles

1. Metrics are defined before development begins, not after.
2. Every metric has a clear owner.
3. Metrics are measured continuously, not at milestones.
4. A metric without a baseline is not a metric — it is a guess.
5. Too many metrics dilute focus. A small set of meaningful metrics beats a large dashboard nobody reads.

---

## Business KPIs

These are the outcomes the business cares about. They are reported to NHAI and ABC Company leadership.

### BK-01: End-to-End Processing Time

**Definition:** Time elapsed from when a vehicle is detected at the upstream RFID reader to when the barrier arm fully opens.

**Measurement:** Calculated from `TagRead` event timestamp to `BarrierOpened` event timestamp, per passage. Reported as P50, P95, P99.

**Baseline:** 3–5 minutes (current state, estimated)

**Target:**
- P50: < 10 seconds
- P95: < 30 seconds
- P99: < 60 seconds (includes exceptional cases: insufficient balance, manual override)

**Owner:** Engineering Manager

---

### BK-02: Manual Intervention Rate

**Definition:** Percentage of vehicle passages that require a human operator to intervene (manual override, cash fallback, dispute resolution).

**Measurement:** `ManualOverrideTriggered` events / total `BarrierOpened` events over rolling 24 hours.

**Baseline:** ~12–18% (estimated from operational reports)

**Target:** < 2%

**Owner:** Lane Management Squad (Squad Kilo)

---

### BK-03: Lane Throughput (Peak Hour)

**Definition:** Number of vehicles processed per lane per hour during the peak traffic window (0700–1000, 1700–2000).

**Measurement:** Count of `BarrierOpened` events per lane per hour during peak windows.

**Baseline:** ~12 vehicles/hour/lane (estimated)

**Target:** ≥ 60 vehicles/hour/lane

**Owner:** Engineering Manager + Product Owner

---

### BK-04: Payment Success Rate

**Definition:** Percentage of pre-authorization requests that result in a successful fund hold.

**Measurement:** `PreAuthorizationApproved` events / total `TagRead` events (excluding blacklisted tags) over rolling 24 hours.

**Baseline:** Unknown (no observability exists)

**Target:** ≥ 98.5%

**Note:** Failures include insufficient balance (legitimate) and gateway errors (operational). These are tracked separately.

**Owner:** Core Platform Squad (Squad Lima)

---

### BK-05: Tag Read Success Rate (First Attempt)

**Definition:** Percentage of vehicles whose FASTag is successfully read on the first attempt by the upstream reader.

**Measurement:** Successful reads / total read attempts where `retry_count = 0` over rolling 24 hours.

**Baseline:** ~92–95% (estimated; current reader is at gate level, not upstream)

**Target:** ≥ 99.5%

**Owner:** Squad Kilo

---

### BK-06: System Availability

**Definition:** Percentage of time during which at least one lane per plaza is processing FASTag transactions normally.

**Measurement:** Minutes of healthy operation / total minutes in the measurement window.

**Target:** 99.9% (permits ~44 minutes downtime per month)

**Owner:** Platform Team

---

## Engineering KPIs

These are the indicators that tell the engineering team whether its delivery process is healthy.

### EK-01: Deployment Frequency

**Definition:** Number of production deployments per week across all services.

**Target:** ≥ 3 per week (multiple times per week)

**Measurement:** ArgoCD deployment history

---

### EK-02: Lead Time for Changes

**Definition:** Time from a code commit being merged to `main` to it running in production.

**Target:** < 2 days (P95)

**Measurement:** Azure DevOps pipeline metrics + ArgoCD sync time

---

### EK-03: Change Failure Rate

**Definition:** Percentage of deployments that require a hotfix, rollback, or patch within 24 hours of deployment.

**Target:** < 5%

**Measurement:** Count of rollback or hotfix deployments / total deployments

---

### EK-04: Mean Time to Recovery

**Definition:** Average time from a P1 or P2 incident being detected to the system returning to normal operation.

**Target:** < 30 minutes

**Measurement:** Incident management system (PagerDuty): time from alert fired to incident resolved

---

### EK-05: Test Coverage (New Code)

**Definition:** Line coverage of unit tests on code committed in the current sprint.

**Target:** ≥ 80%

**Measurement:** SonarQube coverage report, enforced in CI pipeline

---

### EK-06: Build Pipeline Success Rate

**Definition:** Percentage of CI pipeline runs that complete successfully (no test failure, no quality gate failure).

**Target:** ≥ 95%

**Measurement:** Azure DevOps pipeline dashboard

**Note:** A consistently low pipeline success rate is a signal of poor test quality or insufficient pre-commit validation, not a badge of honor.

---

## SLOs and Error Budget Tracking

### SLO Summary

| SLO | Target | Window |
|---|---|---|
| Passage completion in < 30s | 99.0% | Rolling 28 days |
| Pre-auth success in < 8s | 99.5% | Rolling 28 days |
| System availability | 99.9% | Rolling 28 days |
| Audit record completeness | 99.99% | Rolling 28 days |

### Error Budget Interpretation

The error budget is the amount of time or transactions the system is allowed to underperform against its SLO.

**Example — Passage Completion SLO:**
- SLO: 99.0% of passages complete in < 30s over 28 days
- Total passages in 28 days (estimated): ~2,000,000
- Error budget: 20,000 passages that may exceed 30s without breaching the SLO

When the error budget is consumed faster than the 28-day window allows, the team shifts from feature delivery to reliability work.

---

## Phase-Gated Targets

Success targets are tiered to the delivery phases described in [06 — Engineering Execution](06-engineering-execution.md).

| Metric | Phase 2 Target | Phase 3 Target | Steady State |
|---|---|---|---|
| Processing time P95 | < 45s (controlled env) | < 30s (test plaza) | < 30s (production) |
| Manual intervention rate | < 5% | < 3% | < 2% |
| System availability | N/A (test env) | 99.5% (pilot) | 99.9% |
| DORA: Deployment freq | Weekly | 2× week | 3× week |
| DORA: MTTR | < 2 hours | < 1 hour | < 30 min |

---

## What Is Not Measured (and Why)

### Lines of Code Written

Lines of code is not a productivity metric. A team that writes less code to solve the same problem is doing better engineering.

### Number of Features Shipped

Features that do not improve a business KPI are waste. The metric is outcome, not output.

### Test Coverage as a Standalone Goal

80% coverage is a minimum floor, not a target to maximize. 95% coverage with meaningless tests provides less confidence than 80% coverage with well-designed integration tests.

---

## Reporting Cadence

| Audience | Metrics | Frequency |
|---|---|---|
| NHAI | BK-01, BK-02, BK-03, BK-04 | Monthly |
| ABC Leadership | All BKs + EK-01 to EK-04 | Bi-weekly |
| Engineering Manager | All metrics | Weekly |
| Squads | EK metrics + SLO burn rate | Every sprint |
| On-call Engineers | Golden signals + SLO burn rate | Real-time (dashboard) |

---

## Related Documents

- [07 — Observability](07-observability.md)
- [06 — Engineering Execution](06-engineering-execution.md)
- [08 — Risk Analysis](08-risk-analysis.md)
