# 03 — Assumptions

## Purpose

State every assumption explicitly. Unstated assumptions are the most common cause of architecture that cannot be built, or that is built correctly for the wrong environment.

Where an assumption carries architectural risk, the mitigation is noted.

---

## Why This Section Exists

The case study leaves several constraints undefined. Assumptions have been made to enable specific, justifiable design decisions. Each assumption is stated, its impact on design is explained, and the consequence of the assumption being wrong is documented.

If any assumption is invalidated during discovery, the relevant design decisions must be revisited.

---

## Infrastructure and Network

### A-01: Toll sites have 4G/LTE connectivity as the primary network link

**Assumed:** Yes, with fiber as an optional upgrade path at high-traffic plazas.

**Design impact:** The edge processing unit at each toll plaza must operate autonomously during connectivity loss. All gate decisions must be executable locally without a round-trip to the central cloud.

**If wrong:** If sites are fiber-only, the offline-first design adds unnecessary complexity but remains valid. If sites have no reliable connectivity, a store-and-forward model with longer reconciliation windows is required.

---

### A-02: Physical installation of RFID readers at 100–150m upstream of the gate is feasible

**Assumed:** Yes. The case study explicitly references pre-stage reading as an approved proposal.

**Design impact:** The entire pre-authorization model depends on having 6–10 seconds of processing time between tag read and vehicle arrival at the barrier. At highway approach speeds (30–40 km/h in toll zones), 150m provides approximately 13 seconds — sufficient for payment authorization if the gateway SLA is met.

**If wrong:** If readers can only be placed at 50m, processing time drops to 4–5 seconds. Pre-authorization must complete within 3 seconds, requiring aggressive gateway timeouts and a pre-cached balance fallback.

---

### A-03: Each toll plaza has a local compute unit capable of running containerized workloads

**Assumed:** Yes. An edge unit (equivalent to an industrial PC or ruggedized server) runs the Edge Processing Service, lane controller interface, and local cache.

**Design impact:** This enables the offline-first model. Without local compute, every decision requires a cloud round-trip.

**If wrong:** If edge compute is unavailable, the pre-authorization model still works but loses its offline fallback. Availability degrades from 99.9% to approximately that of the cloud connectivity link.

---

## Payment and Financial

### A-04: NPCI payment gateway provides an async pre-authorization API

**Assumed:** Yes. Most modern payment gateways support a two-phase commit: pre-authorize (hold funds) followed by capture (confirm deduction). If NPCI does not natively support this, a balance cache with subsequent reconciliation achieves the same outcome.

**Design impact:** Pre-authorization allows the barrier to open on a confirmed fund hold rather than a confirmed deduction. This eliminates gateway round-trip from the critical path at the barrier.

**If wrong:** If NPCI requires a synchronous single-step deduction, the system falls back to pre-cached balance checks at the gate. Reconciliation happens asynchronously. Some risk of balance drift exists and must be managed.

---

### A-05: Double-charge prevention is a hard requirement

**Assumed:** Yes. NHAI's financial agreement with NPCI and regulatory requirements make this non-negotiable.

**Design impact:** Every payment event must be idempotent. Pre-authorization tokens must be unique per vehicle-per-gate-per-crossing event. The audit service must detect and flag duplicate events before they reach the reconciliation layer.

---

### A-06: FASTag balances are stored at the issuing bank, not locally

**Assumed:** Yes. The tag stores a tag ID only. Balance is a bank-side record. This is standard NPCI architecture.

**Design impact:** Balance caching at the edge is an approximation, not a source of truth. Edge cache is used for fast pre-checks (is there likely sufficient balance?) and the pre-authorization step confirms with the bank.

---

## Traffic and Scale

### A-07: Peak traffic is approximately 500–800 vehicles per hour per plaza

**Assumed:** Based on NHAI published data for high-traffic NH corridors (NH-48, NH-44). Peak periods are 0700–1000 and 1700–2000.

**Design impact:** The pre-authorization service must handle sustained throughput of ~0.15 events per second per lane, with burst capacity for 10× that figure during convoy passages.

---

### A-08: Lane count per plaza ranges from 4 to 12 lanes

**Assumed:** Yes. This affects the event bus partition strategy and the lane controller topology.

**Design impact:** Kafka topics partitioned per lane. Each lane has an independent processing chain. A failure in one lane does not impact others.

---

## Fraud and Security

### A-09: Tag cloning is a known fraud vector

**Assumed:** Yes. NHAI has publicly documented cases of cloned FASTag use. A tag clone is a physical copy of a valid RFID tag used on a different vehicle.

**Design impact:** The Fraud Detection Service must correlate tag reads across geographic locations. A tag appearing at two geographically separated plazas within an impossible travel time is a fraud signal.

**Confidence:** High — this is a documented operational problem.

---

### A-10: Internal services communicate over a private network (not public internet)

**Assumed:** Yes. All services within the system — edge units, central services, payment proxy — communicate over a private Azure Virtual Network with no public endpoints.

**Design impact:** mTLS between services provides authentication and encryption without relying on network-layer security alone. Zero Trust policy is applied at the service level.

---

## Operations and Compliance

### A-11: Every toll transaction requires a durable, tamper-evident audit record

**Assumed:** Yes. Financial regulations and NHAI contract terms require this.

**Design impact:** Audit records are written to an append-only event log (Kafka + long-retention Azure Blob archive). The audit service is a separate consumer — it does not participate in the gate decision path.

---

### A-12: The system must integrate with NHAI's central reporting system via a defined API

**Assumed:** Yes. NHAI maintains a central data warehouse for highway analytics and revenue reporting.

**Design impact:** A reporting service publishes aggregated transaction summaries to NHAI's API on a scheduled basis. Real-time streaming is not assumed unless NHAI specifies it.

---

### A-13: The engineering team does not have access to existing TMS source code

**Assumed:** Yes, until proven otherwise. The TMS is treated as an external system with a defined integration interface.

**Design impact:** Integration is via API contract only. No internal TMS coupling. If source access is provided, the integration strategy can be revised.

---

## Summary of High-Impact Assumptions

| ID | Assumption | Risk if Wrong | Mitigation |
|---|---|---|---|
| A-02 | 150m reader placement is feasible | Pre-auth window shrinks, 3s target becomes hard | Pre-cached balance fallback |
| A-04 | NPCI supports pre-authorization | Gate decision blocks on payment | Balance cache + async reconciliation |
| A-03 | Edge compute is available | No offline fallback | Cloud-first mode with degraded SLA |
| A-09 | Tag cloning is active fraud vector | Under-investment in fraud detection | Validate with NHAI operations data |

---

## Related Documents

- [01 — Business Problem](01-business-problem.md)
- [04 — Target Architecture](04-target-architecture.md)
- [08 — Risk Analysis](08-risk-analysis.md)
