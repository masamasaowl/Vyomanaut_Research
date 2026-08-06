# ADR-055 — Burst-Failure Emergency Eject to Data Owner

**Status:** Proposed
**Topic:** #4 Repair Protocol, #19 Adversarial Provider Behaviour
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 36, 38; closes an open constraint in ADR-004

---

## Context

ADR-004 already flags an unresolved problem at V3 scale: *"the repair scheduler must implement burst admission control"* for correlated failure events, citing Paper 38 (Nath), which shows that a repair backlog from a large simultaneous failure event degrades the repair pipeline's own effectiveness, and Paper 36 (Dalle), which shows correlated failures produce repair-demand variance roughly 22× higher than independent-failure models predict. ADR-007's existing emergency path only fires when a chunk's live fragment count drops to the reconstruction floor (`s=16`) — by definition, this is the last possible moment to act, and it still routes through ordinary P2P repair, the exact mechanism Nath shows can itself be overwhelmed by the event that caused the drop in the first place. There is no earlier, faster, non-repair-pipeline-dependent response to a genuine multi-region correlated outage.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Rely on ADR-007's existing floor trigger (`s=16`) only | No new mechanism | Fires at the last safe moment and depends on the same repair pipeline Nath/Dalle show degrades under the triggering event |
| Lower the floor trigger threshold (e.g., fire repair earlier, at 20 or 24 live fragments) | Simple, no new code path | Still routes through ordinary P2P repair — does not address the actual failure mode Nath identifies (repair backlog under burst load), only moves the same weak response earlier |
| **New burst-velocity trigger: direct-to-owner delivery, independent of the repair pipeline, fired on simultaneous failure count rather than absolute live-fragment count** | Provides a non-repair-dependent safety net specifically for the scenario Nath/Dalle show ordinary repair cannot handle; fires with meaningful margin before the reconstruction floor | New code path; needs its own false-positive tuning separate from ADR-004's existing thresholds |

## Decision

Add a second, independent trigger — **Emergency Eject** — that runs alongside, not instead of, ADR-007's existing floor-based repair trigger.

**Trigger condition:** for a given file, if the number of distinct shard-holding providers transitioning from last-known-healthy to unresponsive **within a rolling window equal to one heartbeat interval** reaches `θ = ⌈0.75 × ParityShards⌉` shards, Emergency Eject fires immediately for that file.

**Where the threshold comes from.** The user-proposed number for production (`TotalShards=56`, `ParityShards=40`) was 30 simultaneous failures. Rather than take that as a fixed constant, this ADR derives it: `θ = ⌈0.75 × 40⌉ = 30` — the proposed number falls out exactly as "75% of the parity budget consumed within one heartbeat window." This makes the rule **profile-portable** instead of a hardcoded constant that breaks in demo mode (`TotalShards=5` makes a literal "30" impossible). Demo profile: `⌈0.75 × 2⌉ = 2`, leaving exactly `DataShards=3` remaining — appropriately aggressive given demo's much thinner redundancy margin to begin with.

**Why 30 is not an arbitrary number, checked two ways:**

1. **ASN-cap consistency.** ADR-014's 20% ASN cap limits any single correlated group to ≤ 11 shards (`⌊56 × 0.20⌋`). Reaching 30 failures therefore requires **at least ⌈30/11⌉ = 3 distinct ASNs** failing within the same window — this cannot be a single ISP/datacenter incident under the cap that is already enforced for adversarial reasons (ADR-014); it is definitionally a multi-region event.
2. **Statistical implausibility under independent churn.** Using ADR-009's Bolosky-sourced MTTF range (180–380 days) and a 4-hour heartbeat-scale window, the probability of 30-of-56 shards failing simultaneously **by chance**, if failures were independent, is on the order of 10⁻⁵⁰ or smaller (numerically indistinguishable from zero at double precision) — see calculation in the accompanying research note. Thirty simultaneous failures is not a tail event under the independence assumption; it is definitionally evidence the independence assumption has broken, which is precisely the Dalle/Nath correlated-failure regime this ADR is built for.
3. **Margin over the floor.** At `θ=30`, 26 shards remain — 10 shards of margin over the `s=16` reconstruction floor in production. This fires with real lead time, not at the last possible moment, which is the entire point of building a mechanism that doesn't depend on the repair pipeline still working.

**Mechanism.** On trigger, the repair service performs the same download-and-RS-decode sequence it already uses for ordinary repair (recovering the AONT-encoded package from the ≥16 surviving shards), but instead of re-distributing new shards to replacement providers via P2P, it delivers the reconstructed, still-encrypted package **directly to the data owner** via the existing microservice-to-client channel. This does **not** violate the zero-knowledge design: the AONT layer's key material is embedded in the package itself and does not require the owner's master secret to decode at the AONT layer, but the file itself remains protected by the owner's outer file-key encryption (ADR-020) — the microservice can reconstruct exactly what it could already reconstruct for ordinary repair, and sends it to the one party who can actually decrypt it further. No new trust boundary is crossed.

**Relationship to ordinary repair:** Emergency Eject does not replace ADR-004's lazy repair or ADR-007's floor trigger. If shard count later recovers or stabilizes above the floor, ordinary repair proceeds as normal to restore full redundancy; Emergency Eject is a parallel safety net for the owner's copy, not a replacement repair strategy.

## Consequences

**Positive:**

- Closes the open constraint ADR-004 explicitly flags, using the exact literature (Nath, Dalle) that ADR-004 already cites as the reason it's needed
- Fires with quantified margin (10 shards / ~18% of total) before the reconstruction floor, using a mechanism independent of the repair pipeline that Nath shows degrades under the same conditions that cause the burst
- Threshold is derived, not guessed, and generalizes across profiles (demo vs. prod) instead of breaking at a hardcoded 30

**Negative / trade-offs:**

- New code path with its own operational cost: every file affected by a qualifying burst triggers an owner delivery, which is bandwidth the repair pipeline would not otherwise spend during a period when network conditions are already degraded
- Owner-side delivery assumes the owner's client is reachable; if the owner is also offline, this ADR does not specify a fallback beyond retry — see open constraints
- False-positive risk: a large, non-catastrophic but genuinely coincidental burst (e.g., a scheduled maintenance window across a hosting provider) would still fire this mechanism; this ADR does not distinguish "catastrophic" from "large planned" bursts

**Open constraints:**

- Distributed correlated-failure detection without a central coordinator remains deferred to Phase 2B (Q07-4, already open in ADR-014) — this ADR's trigger runs centrally in the microservice, consistent with the current architecture, and does not attempt the harder distributed version
- Retry/backoff policy for an unreachable data owner at trigger time is not specified here
- The rolling-window burst-detection query (distinct providers transitioning to unresponsive within one heartbeat interval, per file) is a new query pattern against the audit/heartbeat data; its cost at V3 scale (100k+ providers) is not benchmarked in this research pass

## References

- [Paper 36 — Dalle et al.](../research/paper-36-dalle-failure-correlation.md): correlated-failure variance ~22× independent-model predictions; motivates treating burst events as structurally different from steady-state churn
- [Paper 38 — Nath et al.](../research/paper-38-nath-correlated-failures.md): repair backlog under correlated failure degrades repair convergence; source of the open constraint this ADR closes; RS(16,56) diminishing-returns regime under real correlated failure
- [ADR-004](ADR-004-repair-protocol.md): the ADR whose open constraint this decision closes
- [ADR-007](ADR-007-provider-exit-states.md): existing floor-based emergency trigger, unchanged, runs alongside this
- [ADR-014](ADR-014-adversarial-defences.md): source of the 20% ASN cap used in the threshold's ASN-consistency check
- [ADR-020](ADR-020-key-management.md): file-key encryption layer that keeps owner-delivered data safe without violating zero-knowledge

---
