# ADR-075 — Repair-Replacement Provider Selection Has No Eligible Candidate at the Demo's Fixed 5-Provider Topology

**Status:** Accepted — Option A, headroom derived per scenario below
**Track:** DEMO
**Topic:** #4 Repair Protocol, #16 Simulation & Scale (ADR-069)
**Supersedes:** —
**Superseded by:** —
**Research source:** mvp.md §7.1, §7.2, §7.13 (and §7.10 — see *Bonus finding* below); ADR-014 (20% ASN cap); `internal/repair/assignment.go`; live verification, Session 16.1.1 continuation (F-16-6); five-seat Design Council verdict (this session)

---

## Context

`TestDemoTimeline` kills exactly one of five simulated providers (`testSimCount = 5`, matching `MinDistinctASNs = 5` and `TotalShards = 5`) and expects the resulting repair job to complete. It does not, live:

```
[REPAIR] job d128dc32-...: repair.ExecuteRepairJob: select replacement:
repair: no ASN-cap-eligible replacement provider found after bounded retries
```

This is not a code bug. `internal/repair/assignment.go`'s `SelectReplacementProvider` is doing exactly what FR-045/ADR-014 specify: draw an `ACTIVE` candidate not already excluded, verify it would not push any ASN's share of the segment's shards above `floor(TotalShards × ASNCapFraction)`, and give up (`ErrNoEligibleReplacement`) after bounded retries if none qualifies. The problem is arithmetic, not logic, and mvp.md already contains — in two places, read separately rather than together — everything needed to see it in advance:

- **§7.1** (correcting an earlier internal contradiction in the pre-ADR analysis): `floor(5 × 0.20) = 1` shard per ASN. Placing 5 shards under a 1-shard-per-ASN cap requires exactly 5 distinct ASNs. `MinDistinctASNs = 5` was set for exactly this reason.
- **§7.13**: "ASN cap check at upload: MAX shards per ASN = floor(5 × 0.20) = 1... With 5 ASNs and 5 shards, this is **exactly** satisfied." (§7.13's own word.)
- **§7.2**: "Repair places 1 new shard on a replacement provider — restores count to 4 (still above floor). ✓" — asserted without asking where that provider comes from.

§7.13's own "exactly satisfied" is the tell: original assignment already consumes the entire available cap headroom across all five ASNs, with zero margin anywhere in the system. This is the same shape of internal contradiction §7.1 itself was written to catch — just between §7.1/§7.13 and §7.2, rather than within a single paragraph.

## Design Council Verdict (summary)

A five-seat council was convened on the four options below (full transcript held in session; not reproduced here). Consensus: **Option B is ruled out cleanly.** At demo scale, `k = DataShards = 3` and `maxPerASN = 1`; two colluding ASNs currently hold 2 shards — *below* the 3-shard AONT-RS disclosure threshold — while at production scale two colluding ASNs already hold 22 against a 16-shard threshold (F-34: the cap already fails to prevent this at prod scale, even under normal operation). Loosening the cap for repair-replacement specifically would push a compromised ASN pair from 2 shards to 3 — exactly the disclosure threshold, with one fewer ASN needing to collude — eroding demo's currently-*better*-than-production margin down toward production's already-flagged-as-inadequate one, at the single moment a shard's placement is hardest to audit after the fact. No seat defended B.

Option C was found weaker than its original framing: a permanently-unsatisfiable replacement-selection job needs its own give-up/dead-letter handling in `internal/repair` to avoid spinning indefinitely — not the "zero code change" the initial framing claimed — and it still concedes `build_part3.md`'s O-5 as currently worded. Option D collapses into a variant of A once a standby provider's own audit-freshness is taken seriously (it needs the same real vetting/audit lifecycle a 6th "real" provider gets by default) — same footprint as A with extra bookkeeping and a weaker story.

**Recommendation adopted: Option A, with the headroom explicitly derived rather than picked ad hoc** — the peer-review pass flagged that no seat had checked whether a single spare (6 total) is sufficient for *both* the one-departure and two-departure test scenarios. It is not, for the two-departure case. The derivation below settles it.

## The Derivation

**Governing quantity:** a spare provider/ASN is *reusable across a file's other segments* (the ASN cap is enforced per `segment_id` — a spare already holding segment A's replacement is still fully cap-eligible for segment B, since it holds nothing there) but is *never freed for reuse by a later, different departure* once consumed — it becomes a regular holder for that segment, subject to the same cap as every other provider. Departing providers are never redrawn as candidates (status leaves `ACTIVE` permanently), and a departed provider's now-vacated ASN contributes nothing back, since no new provider ever occupies it in a fixed-size test/rig. Therefore, for a fixed provider pool with no rejoining:

```
N_spares_needed = number of DISTINCT providers concurrently departed
                  (independent of how many segments the file spans)
```

**Verifying the segment count mattered.** `testUploadBytes = 1,000,000` bytes was assumed by both tests' own comments to fit in one segment. It does not: `plaintextSegmentSize(DemoProfile) = DataShards×ShardSize − aontOverheadBytes = 3×262144 − 48 = 786,384` bytes (`internal/client/upload/orchestrator.go`), and `ceilDiv(1,000,000, 786,384) = 2` segments — confirmed directly against the orchestrator's own segmentation code, not inferred. This means `TestDemoTimeline`'s single kill produces 2 repair jobs (one per segment, both from the same departed provider — matching the "2 repair job(s) have status=FAILED" observed live) and `TestViabilityRepairSucceedsWithTwoOfFiveOffline`'s two kills produce 4 (2 providers × 2 segments — matching the 4 job IDs observed live). Working through the two-departure, two-segment case explicitly confirms the governing quantity above: with 2 spares (P5/ASN5, P6/ASN6), segment A's two missing shards each draw one spare; segment B's two missing shards (same 2 departed providers, same file) draw the *same* 2 spares again, since neither has yet been assigned anything in segment B specifically. One spare, worked the same way, is insufficient for the two-departure case — the second segment-A replacement job has no eligible candidate left once the first has consumed the only spare.

**Result:**

| Scenario | Concurrent departures | Segments (confirmed) | Spares needed | Total providers/ASNs |
| --- | --- | --- | --- | --- |
| `TestDemoTimeline` | 1 | 2 | 1 | **6** |
| `TestViabilityRepairSucceedsWithTwoOfFiveOffline` | 2 | 2 | 2 | **7** |
| Five-desktop physical rig (M18, step 10–11, one kill) | 1 | — (real file, size TBD) | 1 | **6** |

## Decision

**Option A**, headroom set to **7** for the shared test harness (`scripts/test/demo_timeline_test.go`'s `testSimCount`/`testSimASNCount`, both file-level constants shared across all five tests in the file — the `providerFleet` struct's `cmds`/`logPaths` arrays are sized by `testSimCount` at compile time, so per-test values would require converting those from fixed-size arrays to slices, a larger structural change not undertaken here). 7 is the binding case (`TestViabilityRepairSucceedsWithTwoOfFiveOffline`); it is also correct, just not minimal, for `TestDemoTimeline` (needs 6) and the three tests that don't exercise repair at all (unaffected either way — their readiness assertions check `RequiredValue`, unchanged at 5, not `CurrentValue`).

`MinActiveProviders`, `MinDistinctASNs`, and `TotalShards` in `config.DemoProfile` are **unchanged at 5**. `internal/repair/assignment.go` is **unchanged** — no core logic was touched; this is a topology/test-harness fix, not a patch.

**Not yet acted on, explicitly:**

- **`build_part3.md`'s five-desktop physical rig** needs a 6th machine (`DESK-06`) to satisfy O-5 as currently worded, for the single-departure procedure it actually runs — not 7; the rig only ever exercises one kill (step 10). This ADR does not update `build_part3.md`'s machine table, network diagram, or procedure text — that touches real hardware acquisition/lab setup and is left for its own pass.
- **mvp.md §7** needs a new line item for "does repair-replacement have a resource it needs," the checklist gap the Design Council's Outsider seat named, so a future parameter change (e.g. `ASNCapFraction`) gets checked against this automatically.
- **mvp.md §7.13**'s "exactly satisfied" framing and §7.2's replacement-provider claim should be corrected or cross-referenced against each other directly, now that this ADR has done so.

**Bonus finding, found during the derivation, not blocking:** mvp.md §7.10 ("max 1.25 MB per segment," worked example: "a 5 MB demo file → 4 segments") does not match `plaintextSegmentSize`'s actual formula. At `DataShards=3, ShardSize=262144`, one segment holds 786,384 bytes (~0.77 MB), not 1.25 MB — a 5 MB file would need 7 segments under the real formula, not 4. §7.10 appears to predate the current `DataShards`/`ShardSize` values and was never revisited. Both `demo_timeline_test.go` comments that assumed "1 segment" for `testUploadBytes` were corrected in the same pass as this ADR's code change (documentation-only; the tests' `pollRepairCompleted` assertions already use "at least," so this was never a false-pass risk). §7.10 itself is left for the same future mvp.md pass as the two items above.

## Consequences

**Applied in this pass:**

- `scripts/test/demo_timeline_test.go`: `testSimCount`/`testSimASNCount` raised from 5 to 7, with the full derivation recorded inline. Two stale "one segment" comments corrected to reflect the confirmed 2-segment reality.
- No change to `internal/repair/assignment.go`, `internal/repair/executor.go`, or any core repair logic.

**Still open (see *Not yet acted on* above):** the physical rig's machine count, and the mvp.md §7 documentation sweep (§7.1/§7.2/§7.13 cross-reference, §7.10 correction, new checklist item).

## References

- mvp.md §7.1 — CORRECTED: ASN Cap vs `MinDistinctASNs` (the original internal-contradiction precedent this finding mirrors)
- mvp.md §7.2 — RS(3,5) reconstruction math (the "restores count to 4" claim that assumes an unexamined replacement-provider source)
- mvp.md §7.10 — Segment size limitation (bonus finding: stale against `plaintextSegmentSize`'s real formula)
- mvp.md §7.13 — Synthetic ASNs and the readiness gate ("exactly satisfied" — the zero-margin observation this ADR generalises to repair)
- `internal/client/upload/orchestrator.go` — `plaintextSegmentSize`, `aontOverheadBytes` (the segment-count derivation)
- ADR-014 — 20% ASN cap (FR-045)
- ADR-069 — Three-tier scale simulation topology ("`--sim-count` was designed for the five-node demo and does not generalise" — the flag's own documented scope)
- `build_part3.md` — "The five-desktop rig" (machine table, procedure, O-5 — needs its own update, not made here)
- F-32 / F-34 (project findings log, referenced via memory) — AONT-RS repair-time plaintext disclosure threshold and the ASN cap's confidentiality role, decisive against Option B in the Design Council verdict
- ADR-074 — Repair Pipeline Live-Verification Findings (F-16-6 was found alongside F-16-5 in the same live-verification session; the two are independent findings in the same pipeline)
