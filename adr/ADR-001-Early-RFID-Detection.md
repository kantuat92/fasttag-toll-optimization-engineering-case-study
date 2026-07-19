# ADR-001 — Pre-Stage RFID Detection at 150m

**Status:** Accepted  
**Date:** 2025-07-19  
**Authors:** Engineering Manager / Principal Engineer  
**Reviewers:** Tech Lead (Squad Kilo), Platform Engineer  

---

## Context

The current FASTag system positions RFID readers at the toll barrier. Vehicles must stop or near-stop within 30–50cm of the reader for a reliable tag read. Processing — tag lookup, payment authorization, audit — begins only after the vehicle arrives at the gate.

This sequential model is the primary reason the 3–5 minute delay exists. By the time the vehicle reaches the barrier, zero work has been done.

ABC Company's contract proposal explicitly includes "Pre-Stage FastTag Reading: implementing scanners that read tags before the vehicle reaches the toll gate." This ADR documents why this approach is chosen, what trade-offs it introduces, and what constraints it places on the design.

---

## Decision

**Deploy a reader array at 100–150m upstream of each toll barrier, in addition to the existing gate-level reader.**

Processing begins the moment the tag is read at the upstream position. The gate-level reader remains as a fallback for vehicles whose tags were not read upstream.

At approach speeds of 30–40 km/h (the typical speed within a toll plaza zone), 150m provides **13–18 seconds of processing time** before the vehicle reaches the barrier. This is the window in which payment pre-authorization is completed.

---

## Alternatives Considered

### Alternative 1: Optimize Existing Gate-Level Processing

**Approach:** Keep the reader at the gate but reduce processing latency through faster servers, connection pooling, and payment gateway SLA enforcement.

**Rejected because:**
- Payment gateway P99 latency is 8+ seconds under load. No amount of server optimization controls external API behavior.
- Even with optimal backend performance, the vehicle still waits at the barrier for the gateway to respond.
- This approach addresses symptoms (slow processing) not the structural problem (processing starts too late).

---

### Alternative 2: Long-Range RFID at the Barrier

**Approach:** Replace gate-level readers with long-range RFID antennas capable of reading at 10–30m distance.

**Rejected because:**
- Long-range RFID significantly increases signal interference in multi-lane environments. A reader that can detect tags at 30m will also detect tags in adjacent lanes.
- Lane assignment becomes ambiguous without directional antennas and vehicle position correlation.
- Provides 2–3 seconds of pre-processing time at best — insufficient for a payment gateway round-trip.

---

### Alternative 3: RFID Reader at 50m with Edge Pre-Authorization

**Approach:** Place readers at 50m rather than 150m. Accept a tighter processing window.

**Not rejected — it is the fallback position.**

If site constraints prevent 150m placement, 50m is the minimum viable position. At 40 km/h approach speed, 50m provides approximately 4.5 seconds. Payment pre-authorization must complete in under 3 seconds to leave margin. This requires a pre-cached balance check rather than a live gateway call. The design accommodates this through the balance cache fallback.

---

## Consequences

### Positive

- **Processing time removed from gate critical path.** The barrier open signal is triggered by a token lookup (< 100ms), not a payment gateway response.
- **Tag read quality improves.** Vehicles at 150m are still in motion but at highway approach speed — signal conditions are more consistent than at near-stop distance.
- **Redundancy.** Gate-level reader remains as secondary. Any vehicle missed by the upstream reader is caught at the gate (slower, but functional).

### Negative

- **Infrastructure cost.** Additional RFID hardware at each lane on each plaza. Estimated: 2 upstream readers per lane (redundant pair) + mounting infrastructure.
- **Vehicle-to-tag matching complexity.** At 150m, multiple vehicles may be in the reader range. Loop sensors and lane geometry must be used to correlate a tag read with the specific vehicle in the specific lane.
- **Edge processing unit required.** The upstream read initiates processing at the edge, which requires local compute. This is addressed in the container architecture ([04 — Target Architecture](../docs/04-target-architecture.md)).

### Neutral

- **Gate-level reader is retained.** It serves as a fallback, not the primary read point. No existing hardware is removed.

---

## Implementation Constraints

1. **Loop sensors must be deployed at both the upstream position and the gate.** Tag read without a loop sensor trigger is not acted upon — this prevents false pre-authorizations from nearby vehicles in adjacent lanes.

2. **Pre-authorization is keyed by `tag_id + lane_id + plaza_id + event_window`.** Tokens are lane-specific. A pre-authorization on Lane 3 cannot open the barrier on Lane 4.

3. **Pre-authorization token TTL is 120 seconds.** If the vehicle does not arrive at the barrier within 120 seconds of the upstream read (e.g., the vehicle stops or changes lanes), the token expires and must be re-acquired.

4. **Reader health telemetry must be emitted from day one.** The AI reader failure prediction model (ADR-003) depends on structured telemetry. Readers without telemetry cannot be monitored.

---

## Future Considerations

- **Camera-based vehicle identification** could supplement RFID correlation for lane disambiguation in high-density scenarios. This is outside current scope but the event schema is designed to accommodate a `vehicle.detected` event from a vision system.
- **5G private network at toll plazas** would reduce the upstream-to-cloud latency, potentially enabling 50m placement to meet the 30-second target without cache fallback. Worth evaluating at Phase 4.

---

## References

- [04 — Target Architecture](../docs/04-target-architecture.md)
- [03 — Assumptions: A-02](../docs/03-assumptions.md)
- [08 — Risk Analysis: AR-03](../docs/08-risk-analysis.md)
