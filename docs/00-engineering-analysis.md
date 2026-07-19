# Engineering Analysis & Solution Approach

> **Author:** Mani Kumar Sabbarapu
>
> **Purpose:** Document the engineering thought process followed before proposing an architecture.

---

# Executive Summary

As the Engineering Lead, my first objective was **not to design a solution**, but to understand the problem that ABC Company is trying to solve.

Before making architectural decisions, I wanted to determine whether the delays were caused by the FASTag technology itself or by the broader toll collection ecosystem.

This document captures the analysis performed before beginning the architecture, design, development, and rollout phases.

---

# Problem Understanding

The case study states that:

- FASTag is already an automated payment system.
- Vehicles still experience delays of **3–5 minutes**.
- The business objective is to reduce the total processing time to **under 30 seconds**.

At first glance, this raises an important engineering question.

> **If FASTag is already automated, why are vehicles still waiting?**

Rather than immediately proposing a new architecture, I analysed the existing workflow to identify where the actual bottlenecks might exist.

---

# Question 1 – Is FASTag Really the Problem?

FASTag is fundamentally an RFID-based identification mechanism.

Its primary responsibility is simple:

1. Read RFID tag
2. Identify the vehicle
3. Send tag information for validation

RFID technology typically performs tag detection within milliseconds.

This immediately suggested that the RFID tag itself was unlikely to be responsible for several minutes of delay.

---

# Question 2 – Where Does the Remaining Time Go?

The vehicle processing consists of multiple independent steps.

```text
Vehicle Arrival
        │
        ▼
RFID Detection
        │
        ▼
Tag Validation
        │
        ▼
Payment Authorization
        │
        ▼
Fraud Verification
        │
        ▼
Barrier Control
        │
        ▼
Vehicle Exit
```

Each stage introduces its own latency.

The total waiting time is therefore the cumulative result of multiple systems rather than a single component.

---

# Question 3 – Can FASTag Itself Be Improved?

This was my next consideration.

Possible improvements include:

- Higher quality RFID readers
- Better antenna positioning
- Improved tag quality
- Longer read range

While these improvements may increase read accuracy, they do not eliminate delays caused by:

- Backend processing
- Network communication
- Payment gateways
- Operational workflows
- Manual intervention

Improving RFID technology alone would therefore provide only incremental benefits.

It would not realistically reduce processing time from several minutes to under 30 seconds.

---

# Question 4 – Is This a Software Problem?

Not entirely.

The transaction involves several independent domains.

- Physical infrastructure
- RFID hardware
- Network connectivity
- Backend systems
- Payment providers
- Barrier control
- Human operators
- Vehicle movement

This indicates that the problem is not purely a software optimisation exercise.

It is an end-to-end systems engineering challenge.

---

# Question 5 – Is There Enough Information?

Before designing any solution, I identified several assumptions that should normally be validated with business stakeholders.

Examples include:

### Traffic Characteristics

- Vehicles per hour
- Peak traffic periods
- Lane utilisation

### RFID Infrastructure

- Current read success rate
- Retry frequency
- Reader placement
- Hardware limitations

### Payment Processing

- Average payment latency
- Gateway SLA
- Failure rates

### Operations

- Frequency of manual intervention
- Top operational failure scenarios

### Success Criteria

The case study specifies a target of **under 30 seconds**, but additional clarification would normally be required.

For example:

- Average processing time?
- P95 latency?
- Maximum acceptable latency?
- Peak traffic expectations?

These assumptions are documented separately in **03-assumptions.md**.

---

# Analysis of ABC Company's Proposal

ABC Company proposes three initiatives:

- Pre-Stage FASTag Reading
- Dedicated Express Lanes
- Continued support for existing manual and FASTag lanes

After analysing the problem, I concluded that these proposals address the right engineering challenge.

## Why Pre-Stage RFID Reading Makes Sense

Initially, I questioned why the proposal focused on reading the tag earlier instead of improving FASTag itself.

The answer became clear after analysing the transaction lifecycle.

The objective is **not to make RFID faster**.

The objective is to **start the processing earlier**.

By reading the FASTag before the vehicle reaches the barrier, the system gains additional time to complete:

- Tag validation
- Payment processing
- Fraud checks
- Backend communication

As a result, much of the required work is completed before the vehicle reaches the toll gate.

The barrier operation becomes the final confirmation step rather than the starting point of the transaction.

This approach optimises the overall customer journey rather than a single technology component.

---

## Why Express Lanes Are Appropriate

Dedicated FASTag lanes reduce operational complexity by separating automated traffic from manual transactions.

This enables:

- Consistent traffic flow
- Reduced queue interference
- Better throughput
- Predictable processing times

The proposal therefore addresses both technical and operational efficiency.

---

## Why Existing Lanes Should Continue

Replacing all toll infrastructure would introduce unnecessary operational risk.

Maintaining compatibility with existing manual and FASTag lanes provides:

- Incremental rollout
- Reduced business disruption
- Lower implementation risk
- Easier rollback strategy
- Better operational resilience

This aligns with good engineering and change management practices.

---

# Engineering Conclusion

After completing the analysis, I concluded that the primary challenge is **not FASTag technology itself**.

The challenge is the cumulative latency introduced across the complete toll transaction lifecycle.

ABC Company's proposal correctly focuses on optimising **when processing begins** rather than attempting to redesign FASTag.

This provides a pragmatic foundation for reducing end-to-end processing time while preserving compatibility with the existing ecosystem.

With this understanding established, the next step is to design an architecture that:

- Moves processing earlier in the vehicle journey
- Minimises synchronous operations at the barrier
- Improves operational visibility
- Enables phased rollout with minimal disruption

The subsequent documents describe the proposed target architecture, engineering design, AI enhancements, execution strategy, and operational considerations.