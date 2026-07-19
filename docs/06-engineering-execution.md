# 06 — Engineering Execution

## Purpose

Describe how this program is delivered. Architecture without execution is a diagram. This section covers team structure, delivery model, quality practices, DORA metrics, release strategy, and how engineering management is applied throughout.

---

## Team Structure

The program requires two delivery squads, one platform team, and a Tech Lead per squad. Team size is deliberately kept small enough to be accountable, large enough to deliver in parallel.

### Squad Structure

```
Program
├── Squad Kilo — Edge & Lane Management
│   ├── Tech Lead (1)
│   ├── Senior Engineers (2)
│   ├── Engineers (2)
│   └── QA Engineer (1, embedded)
│
├── Squad Lima — Core Platform Services
│   ├── Tech Lead (1)
│   ├── Senior Engineers (2)
│   ├── Engineers (2)
│   └── QA Engineer (1, embedded)
│
└── Platform Team
    ├── Platform Engineer (1) — CI/CD, observability, AKS
    └── Security Engineer (0.5 FTE, shared) — Zero Trust, secrets, SAST
```

### Squad Kilo — Edge & Lane Management

**Owns:** Edge Processing Unit, RFID reader integration, Lane Controller Service, Lane Management Service, Express Lane logic

**Focus:** Low-latency, high-reliability edge software. Engineers here need embedded systems awareness and strong distributed systems fundamentals.

### Squad Lima — Core Platform Services

**Owns:** Pre-Auth Service, Payment Service, Fraud Detection Service, Audit Service, API Gateway, NHAI Reporting Service

**Focus:** Financial accuracy, event-driven processing, external API integration. Engineers here need strong Java, Spring Boot, and data consistency experience.

### Platform Team

**Owns:** AKS cluster, CI/CD pipelines, Kafka cluster, observability stack, developer tooling

**Focus:** Everything that makes the two delivery squads fast. Measured by pipeline lead time and developer satisfaction.

---

## Delivery Model

### Sprint Cadence

- 2-week sprints, fixed-length
- Sprint 0 (weeks 1–2): Architecture review and ADR sign-off, API contract definitions, event schema design, local `docker compose` dev environment — development begins here, not after infra is ready
- Delivery sprints begin Sprint 1 (week 3) — cloud infrastructure is provisioned in parallel, not as a prerequisite
- No scope changes within a sprint without Tech Lead approval

### Definition of Done

A user story or task is done when:

1. Code reviewed by one Tech Lead or Engineering Lead
2. Unit tests written (target: 80% coverage on new code)
3. Integration tests pass in the test environment
4. SLO defined for any new user-facing capability
5. Observability dashboard updated or created
6. Runbook exists for any new alert

This definition is not negotiable. Partial implementations do not ship.

### Sprint Ceremonies

| Ceremony | Frequency | Duration | Purpose |
|---|---|---|---|
| Sprint Planning | Per sprint | 2 hours | Commit to sprint backlog |
| Daily Standup | Daily | 15 minutes | Blockers and synchronization |
| Architecture Review | Bi-weekly | 1 hour | ADR review, design decisions |
| Tech Lead Sync | Weekly | 30 minutes | Cross-squad dependency management |
| Retrospective | Per sprint | 1 hour | Process improvement |
| Demo | Per sprint | 30 minutes | Stakeholder visibility |

---

## Phased Delivery

### Phase 1 — Foundation (Weeks 1–6)

**Objective:** Development starts as soon as architecture review is approved. Infrastructure provisioning and service development run in parallel — engineers do not wait for cloud environments to begin writing code.

Two tracks run concurrently from Sprint 1 (Week 3):

#### Track A — Infrastructure (Platform Team)

| Week | Deliverable |
|---|---|
| 1–2 (Sprint 0) | Architecture review, ADR sign-off, API contracts defined, event schemas drafted |
| 3–4 | AKS cluster provisioned, namespaces and network policies configured |
| 3–4 | CI/CD pipelines live (Azure DevOps + ArgoCD), container registry ready |
| 4–5 | Kafka cluster operational (Azure Event Hubs), schema registry bootstrapped |
| 5–6 | OpenTelemetry collector deployed, Dynatrace connected, API Gateway with mTLS live |

#### Track B — Development (Squad Kilo + Squad Lima)

| Week | Deliverable |
|---|---|
| 1–2 (Sprint 0) | Local dev environment: `docker compose up` runs Kafka, Redis, PostgreSQL locally |
| 3–4 | Edge Processing Unit — RFID event ingestion and Kafka publish (running locally) |
| 3–4 | Pre-Auth Service skeleton — TagRead consumer, NPCI client stub, token store logic |
| 4–5 | Lane Controller Service — token lookup, barrier signal interface (simulator) |
| 5–6 | Services integrated with cloud infrastructure as it becomes available |

**The rule:** if a cloud service is not yet provisioned, the squad uses a local Docker equivalent and integrates when cloud is ready. Development is never blocked by infrastructure timelines.

**Exit criteria:** A synthetic vehicle passage generates a complete distributed trace from RFID read to simulated barrier signal, running on the cloud environment.

---

### Phase 2 — Core Flow (Weeks 7–14)

**Objective:** End-to-end vehicle passage in a controlled environment.

Deliverables:
- Pre-Auth Service integrated with NPCI (staging environment)
- Payment Service settlement flow operational
- Fraud Detection Service — geographic impossibility check active
- Audit Service consuming all transaction events
- Lane Management Service — basic lane status management
- Operator dashboard — lane status, payment health, alert feed
- Load testing at 2× target throughput

**Exit criteria:** A vehicle at test plaza completes the full passage in under 30 seconds, with a complete audit trail and distributed trace.

---

### Phase 3 — AI and Operations (Weeks 15–20)

**Objective:** Operational intelligence layer, express lane support, hardening.

Deliverables:
- AI Operations Assistant integrated (Anthropic Claude API)
- Reader failure prediction — rule-based first, ML model in parallel
- Traffic queue forecasting model — Prophet baseline
- GenAI runbook generation pipeline
- Express lane logic with dynamic activation
- Chaos engineering exercises (Chaos Mesh on AKS)
- Disaster recovery runbooks tested

**Exit criteria:** AI Operations Assistant correctly identifies injected failure root cause in 3/3 chaos exercises. Express lanes tested under simulated peak load.

---

### Phase 4 — Pilot Deployment (Weeks 21–26)

**Objective:** Production deployment at a single controlled plaza. Validate architecture under real traffic before any expansion.

Deliverables:
- Single plaza, single lane live in production (Stage 0 of geographic rollout)
- SLO dashboards live in production
- Incident management process and on-call rotation operational
- Operator training completed at pilot plaza
- 30-day observation period before expanding to additional lanes

**Exit criteria:** Pilot lane sustains P95 < 30 seconds for 30 consecutive days, manual intervention rate < 2%, zero data loss events.

---

## Geographic Rollout Strategy

Software delivery ends at Phase 4. Geographic expansion is a separate, risk-managed program that begins only after Phase 4 exit criteria are met.

Each stage has a defined exit gate. No stage begins until the previous stage's gate is passed. This is not a schedule — it is a capability gate.

---

### Stage 0 — Single Plaza, Single Lane (Weeks 21–26)

**Scope:** One lane at one pilot plaza on a high-traffic NH corridor.

**Objective:** Validate the full stack under real traffic. Confirm that architecture assumptions hold in production — reader placement, NPCI latency, edge cache behavior, operator workflow.

**What is being learned:**
- Does pre-authorization complete within the 13-second window under real traffic conditions?
- What is the actual NPCI P95 latency in production?
- What failure modes appear that were not covered in testing?
- Are operator runbooks accurate and actionable?

**Exit gate:**
- P95 processing time < 30 seconds sustained for 30 consecutive days
- Manual intervention rate < 2%
- Zero double-charge events
- All P1/P2 incidents have completed postmortems with backlog items

---

### Stage 1 — Full Plaza (Weeks 27–32)

**Scope:** All lanes at the pilot plaza. If a second plaza is on the same site, include it.

**Objective:** Validate multi-lane operations. Confirm that Kafka partition-per-lane design holds under concurrent load across 8–12 lanes. Validate express lane dynamic activation.

**What is being learned:**
- Do lane-level failures isolate correctly? (One lane's reader fault should not affect adjacent lanes.)
- Does the AI queue forecasting model produce accurate lane activation recommendations at plaza scale?
- What is the operational burden on the plaza operator team?

**Operational changes at this stage:**
- Full on-call rotation covering the pilot plaza
- Plaza-level SLO dashboard visible to NHAI stakeholders
- Operator feedback loop established — operators submit friction reports weekly

**Exit gate:**
- All lanes at pilot plaza sustaining Stage 0 targets
- Peak-hour throughput ≥ 60 vehicles/hour/lane for 5 consecutive peak days
- Operator satisfaction score > 3.5/5 (post-training survey)
- No cross-lane incidents caused by system failure

---

### Stage 2 — District Corridor (Weeks 33–44)

**Scope:** All plazas on one NH corridor — for example, NH-48 (Delhi–Gurugram–Jaipur) or NH-44 (Delhi–Hyderabad). Typically 8–15 plazas.

**Objective:** Validate multi-plaza operations: centralized observability across plazas, fraud detection at geographic scale (tag cloning across plazas), NHAI reporting aggregation, and the distributed edge unit management model.

**What is being learned:**
- Does the geographic impossibility fraud detection work correctly across plazas on the same corridor?
- Does the AI reader failure prediction model generalize across different reader models and environmental conditions?
- Can the Platform Team manage deployments across 8–15 edge units without proportional headcount increase?
- What does NHAI corridor-level reporting look like in practice?

**Infrastructure changes at this stage:**
- ArgoCD manages edge unit deployments at corridor scale — deployment targeting by `plaza.corridor` label
- Dynatrace hierarchy: Plaza → Corridor → National (dashboard aggregation)
- Fraud Detection Service cross-plaza event correlation enabled in production
- NHAI corridor reporting dashboard live

**Rollout sequence within Stage 2:**
1. Deploy to 2 additional plazas, observe for 1 week
2. Deploy to remaining plazas in the corridor in batches of 2–3
3. Full corridor observation for 2 weeks before Stage 3 gate review

**Exit gate:**
- All corridor plazas sustaining Stage 0 targets simultaneously
- At least one fraud signal detected and correctly handled (or confirmed clean with evidence)
- Edge unit deployment automation validated: deploy to all corridor plazas in < 2 hours
- NHAI corridor reporting signed off by NHAI stakeholder

---

### Stage 3 — National Rollout (Weeks 45+)

**Scope:** Remaining NH corridors, prioritised by traffic volume and NHAI contract terms.

**Objective:** Scale the operating model, not the architecture. The architecture is validated by Stage 2. Stage 3 is about operational repeatability — can the team onboard a new corridor in 2 weeks with no regression on existing corridors?

**Rollout sequencing:**
- Corridors prioritised by NHAI traffic volume ranking (highest-volume first — maximum business impact)
- Each corridor follows a 2-week activation: deploy → observe → sign-off
- No more than 3 corridors activated simultaneously (operational bandwidth constraint)

**Operational model at national scale:**
- Tier 1 support: Plaza operators (trained per corridor activation)
- Tier 2 support: ABC Company operations team (on-call, MTTR < 30 min)
- Tier 3 support: Engineering squads (P1 escalation only)
- AI Ops Assistant in production across all plazas — RCA burden on Tier 2 is the primary scaling mechanism

**Governance at national scale:**
- Monthly architecture review: performance trends, emerging failure patterns, model retraining schedule
- Quarterly capacity review: Kafka partition growth, AKS node scaling, NPCI SLA compliance
- NHAI monthly report: corridor-level throughput, SLO compliance, incident summary

**Exit criteria (program completion):**
- All contracted NH corridors live and sustaining targets
- National SLO compliance report delivered to NHAI
- Operations fully handed over to steady-state team with no engineering squad dependency for routine operations

---

### Rollout Summary

| Stage | Scope | Duration | Key Validation |
|---|---|---|---|
| Stage 0 | 1 lane, 1 plaza | Weeks 21–26 | Architecture under real traffic |
| Stage 1 | All lanes, 1 plaza | Weeks 27–32 | Multi-lane isolation and peak load |
| Stage 2 | 1 full NH corridor (8–15 plazas) | Weeks 33–44 | Multi-plaza ops, fraud at scale |
| Stage 3 | All contracted NH corridors | Weeks 45+ | Operational repeatability, national scale |

**The principle across all stages:** each stage proves the next stage is safe to attempt. No stage is entered on a schedule alone — it is entered when the exit gate of the previous stage is cleared.

---

## Release Strategy

### GitOps Model

All deployments are managed via ArgoCD from Git. No manual deployments to production. A deployment is a Git commit.

```
feature/* → develop → staging → production
```

- Feature branches: developer-level work
- `develop`: integration, automated tests run on every push
- `staging`: production-equivalent environment, performance tests
- `production`: controlled by ArgoCD, requires pipeline green + manual approval gate

### Deployment Strategy

- **Services:** Rolling deployment on AKS (zero downtime)
- **Database schema changes:** Expand-contract pattern (no destructive migrations in deployment path)
- **Kafka schema changes:** Schema registry with backward compatibility enforcement
- **Edge units:** Blue-green deployment with traffic shift via lane controller configuration

### Rollback Strategy

Every deployment is rollback-safe by construction:

- ArgoCD maintains deployment history — rollback is a single command
- Database migrations are reversible (expand-contract enforced in code review)
- Kafka consumer offset can be reset for replay if processing error is detected
- Feature flags for any capability that cannot be rolled back at the deployment level

---

## Code Quality and Review

### Code Review Standards

- Every PR requires at least one peer review and one Tech Lead review before merge
- PRs larger than 400 lines of changed code are flagged for decomposition
- Security-sensitive code (payment, auth, secrets) requires Security Engineer review
- Architecture changes require an ADR before a PR is opened

### Static Analysis

- SonarQube on every build: quality gate blocks merge on critical or blocker issues
- Checkstyle for Java code style consistency
- OWASP Dependency Check for known vulnerabilities in dependencies
- Trivy for container image vulnerability scanning

### Testing Strategy

| Level | Tool | Target Coverage |
|---|---|---|
| Unit tests | JUnit 5 + Mockito | 80% on new code |
| Integration tests | Spring Boot Test + Testcontainers | All external integrations |
| Contract tests | Spring Cloud Contract | All service-to-service APIs |
| Performance tests | Gatling | P95 < 30s at 2× peak load |
| Chaos tests | Chaos Mesh | Defined failure scenarios, Phase 3 |

---

## DORA Metrics

DORA metrics are tracked from Sprint 1. They are reviewed in every retrospective. They are not vanity metrics — they are the signal that tells the team if its delivery process is healthy.

| Metric | Baseline (Target by Phase 2) | Target (Steady State) |
|---|---|---|
| Deployment Frequency | Weekly | Multiple per week |
| Lead Time for Changes | < 5 days | < 2 days |
| Change Failure Rate | < 10% | < 5% |
| Mean Time to Recovery | < 2 hours | < 30 minutes |

These targets are achievable with a proper CI/CD pipeline, observability, and rollback-safe deployments. They are not aspirational — they are the outcome of the practices described above.

---

## Technical Debt Management

Technical debt is inevitable. The goal is to make it visible and managed, not to eliminate it.

- Every debt item is logged in the backlog with a business impact statement
- 20% of each sprint capacity is reserved for debt reduction and operational improvement
- Debt items above a defined severity threshold are escalated to the Product Owner with a cost-of-carry estimate
- No item is marked as debt if it can be fixed in the current sprint — it is just fixed

### Debt Categories

| Category | Example | Response |
|---|---|---|
| Architecture debt | Shared database between two services | Scheduled refactor, ADR required |
| Code debt | Duplicated validation logic | Fix within 2 sprints |
| Test debt | Missing integration test coverage | Fix within current sprint |
| Dependency debt | Library with known CVE | Fix within current sprint (security) |

---

## Developer Experience

Engineering productivity is an engineering management responsibility. Slow builds, flaky tests, and unclear environments reduce output and increase attrition.

- Local development environment setup: single command (`docker compose up`)
- Build time target: < 5 minutes for full build + unit tests
- Test environment refresh: automated, daily
- On-call rotation: no engineer on call more than 1 week in 4
- Documentation: every service has a README with local run instructions, API contract, and event schema

---

## Mentoring and Growth

- Each junior engineer is paired with a senior engineer in a formal pairing rotation
- Monthly 1:1s between Engineering Manager and every team member
- Brown-bag sessions: bi-weekly, tech leads rotate topics
- External conference budget: 1 conference per engineer per year
- Promotion criteria are documented and reviewed annually

---

## Engineering Health Checks

Monthly engineering health review covers:

- DORA metrics trend
- SLO compliance rate
- On-call incident count and burden
- Open tech debt item count
- Team member satisfaction (quarterly pulse survey)
- Build pipeline reliability

Deterioration in any of these is treated as an engineering management priority, not an acceptable steady state.

---

## Related Documents

- [04 — Target Architecture](04-target-architecture.md)
- [07 — Observability](07-observability.md)
- [09 — Success Metrics](09-success-metrics.md)
