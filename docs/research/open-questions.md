# Open Questions

One entry per question. Status is `open` or `answered`. When a later paper answers a question, update its status here and paste the answer — do not delete the entry.

Format: `### QNN-N — [short question]`, then **Raised by:**, **Status:**, **Blocked on:**, and (once answered) **Answer:**.

>**NOTE:** The original open-questions.md file was merged into section 10 of system-design/requirements.md, later when the project research was continued from Paper 42, it was decided to move back to a standalone version of the file. Due to these constant migrations the file is currently in a poorly formatted state. The documentation team is currently working to revive this file to it's original state. Till then it has been requested to all the teams to not make use of this file extensively.

---

### Q42-1 — Does a paid, escrow-penalized marketplace need a public leaderboard the way BOINC's unpaid volunteer model does?

**Raised by:** Paper 42 (BOINC)
**Status:** open
**Blocked on:** the Impact Analytics design (`ux-finding.md` §8) — real money may change whether ranking against other providers motivates or discourages participation. Not resolved by Paper 42, which only covers an unpaid context.

---

### Q43-1 — If Vyomanaut pursues an institution-led rollout, is the unit of trust the institution's IT department, or the individual machine's user with institutional awareness but no formal sign-off?

**Raised by:** Paper 43 (Condor)
**Status:** open
**Blocked on:** the business-development decision in `ux-finding.md` §5.2. Neither Condor's IT-led model nor BOINC's individual-opt-in model (Paper 42) precedents a hybrid directly.

---

### Q44-1 — What is a defensible, citable per-terabyte or per-server embodied-carbon figure, current enough to show a specific number to an individual provider?

**Raised by:** Paper 44 (Masanet et al.)
**Status:** open
**Blocked on:** a dedicated embodied-carbon source — Paper 44 only grounds the operational-energy half of the Impact Analytics claim, not the manufacturing/embodied half the actual pitch depends on.

---

### Q45-1 — Does Wails' in-process binding model constrain a future browser-based or mobile companion app?

**Raised by:** Paper 45 (Wails/Electron)
**Status:** open
**Blocked on:** scoping of any future mobile companion. The existing REST API (M11) is expected to serve that need independent of the desktop shell choice, but this has not been explicitly verified against Wails' architecture.

---

### Q46-1 — What fraction of Storj's total network capacity comes from its Commercial Storage Node Operator Program versus individual/home operators?

**Raised by:** Paper 46 (Storj node operator docs)
**Status:** open
**Blocked on:** Storj does not publish this split publicly, as far as this review found. Without it, the size of the "true home consumer" segment within Storj's existing network is an estimate, not a measured figure.

---

### Q47-1 — How much of Vyomanaut's reliability-score/payout-multiplier formula should be disclosed to providers, given that full opacity risks the trust breakdown this paper documents, and full disclosure risks the score being gamed?

**Raised by:** Paper 47 (Rosenblat & Stark)
**Status:** open
**Blocked on:** the forthcoming Impact Analytics / earnings-transparency design. Directly related to the already-accepted ADR-016 addendum (gross/multiplier/withheld columns), which supplies the data this decision would act on, but does not itself decide how much of the formula to show.

---

### Q48-1 — Is the fiat/UPI advantage over crypto-token wallets durable, or could improved wallet UX narrow this gap over time?

**Raised by:** Paper 48 (Voskobojnikov et al., crypto wallet UX)
**Status:** open
**Blocked on:** nothing actionable now — flagged as a watch item. The crypto industry's own 2025 account-abstraction efforts (e.g., EIP-7702) are attempting to close exactly this gap; revisit if wallet UX changes materially before any ADR treats the fiat/UPI advantage as permanent rather than current.

## Old version of Open Questions

Genuinely open questions only. Every entry states why it is open and how to close it.

Three tiers:

**Tier 1 — Build blockers** require a decision or design before the relevant module can be implemented.
**Tier 2 — Telemetry** can only be answered by observing the live V2 system.
**Tier 3 — V3 scope** are valid questions explicitly deferred to the next major version.

All Tier 1 questions are now resolved. Resolved questions have been moved to `answered-questions.md`. Dropped questions are recorded in `dropped-questions.md`.

---

## Tier 1 — Build Blockers

**All Tier 1 questions are resolved. There are no remaining build blockers.**

Every question that was in this tier has been closed analytically and the resulting
constraints have been written into the ADRs they govern. See `answered-questions.md`
for the closed entries and `adr-changes.md` for the specific ADR text changes.

---

## Tier 2 — Telemetry

These cannot be answered by research or analysis. They require observing the live V2 system.
None of them block the build. They produce tuning decisions after V2 beta launch.

---

**Q01-5** — *Peer selection* — Does reliability-proportional assignment create a runaway Matthew effect where the top 5% of providers receive > 50% of new chunk assignments?
Measure: track cumulative chunk assignment distribution vs reliability score percentile over the first 90 days. Trigger: if the top decile holds > 40% of assignments, add a concentration cap analogous to the ASN cap.

**Q05-1** — *Coordination* — What practical challenges does the central microservice pose at launch scale, and which satellite functions can be decentralised in V3?
Measure: microservice latency percentiles, failure incidents, and operational overhead over the first 6 months.

**Q05-4** — *Economic calibration* — What held-earnings percentage and vetting period length empirically achieve the target MTTF (180–380 days)?
Starting values in ADR-024: 30-day rolling hold, 50% release cap during vetting. Measure: provider survival curves by cohort; adjust multipliers if median provider tenure < 6 months.

**Q05-7** — *Storage overhead* — At what file size does per-segment pointer file metadata overhead become user-visible, and should small files be inlined?
Measure: track upload distribution at launch; compute metadata-to-data ratio by file size decile; set an inline threshold if the smallest decile shows > 5% metadata overhead.

**Q06-3** — *Polling interval* — Does t=24h need production tuning?
Measure: false-positive departure declarations (providers declared departed who return within 72h) over the first 3 months. Adjust if false-positive rate > 5%.

**Q08-1** — *Provider MTTF* — What is the actual MTTF of a financially-incentivised Indian desktop provider?
Measure: survival analysis on the first provider cohort; compare to Bhagwan's unincentivised floor (median session ~1–2h) and Bolosky's corporate desktop ceiling (MTTF 290–380 days).

**Q08-3** — *Repair budget* — At what provider count does a 20%/day turnover rate produce more repair bandwidth than the network's idle upload capacity can absorb?
Compute: `turnover_rate × repair_BW_per_event (chunk_size × parity_count)` vs `N × 100 KB/s`. Inputs are unknown until actual turnover rate is observed.

**Q14-1** — *NAT traversal* — What fraction of Indian home ISPs block UDP, forcing all QUIC connections to the TCP fallback?
Measure: log transport type per provider connection at registration; report UDP-block rate at 30 days.

**Q14-3** — *Audit receipt* — What is the false-positive rate of the JIT detector (`response_latency_ms`) during QUIC connection migration events?
Measure: correlate QUIC migration signals in the libp2p layer with audit response latency spikes over the first month; if false-positive rate > 1%, add a migration flag to the audit receipt schema.

**Q19-3** — *Erasure parameter validation* — Does the > 98% single-chunk failure rate (Facebook warehouse) hold in a P2P consumer desktop network with correlated failures?
Measure: log failure event sizes (how many chunks fail simultaneously per provider departure) over the first 6 months; if multi-chunk bursts > 2% of events, re-evaluate whether MSR repair bandwidth optimisation targets the right case.

**Q20-1** — *NAT geography* — What fraction of Indian desktop home-router deployments are behind symmetric NAT and require Circuit Relay v2 permanently?
Measure: log AutoNAT classification per provider at registration; report symmetric-NAT fraction at 30 days. Global baseline from Paper 30 is ~30%; India-specific may be higher due to CGNAT prevalence.

**Q20-3** — *Economic mechanism* — What held-earnings percentage empirically achieves MTTF 180–380 days from an initially unincentivised provider population?
Measure: survival curves for the first provider cohort; compare against Trautwein's unincentivised floor (87.6% of sessions under 8h).

**Q21-1** — *Vetting effectiveness* — What fraction of registered providers pass the 4–6 month vetting period vs are rejected or downgraded?
Measure: cohort analysis at 6 months; if reject rate > 40%, investigate whether registration gate or vetting criteria need adjustment.

**Q23-1** — *Fragment size* — Is the ~10–15% write throughput penalty at lf=256 KB vs lf=512 KB observable on Indian desktop hardware?
Measure: benchmark vLog append throughput at 256 KB and 512 KB on a minimum-spec desktop; if gap > 20%, re-evaluate lf (requires coordinated change to ADR-003, ADR-004).

**Q27-2** — *Disk reliability* — What is the empirical rate of within-provider burst failures (multiple chunks failing simultaneously)?
Measure: log the number of chunks simultaneously invalidated per provider departure/failure event; if multi-chunk bursts appear at > 1% of events, evaluate sparse vLog pre-allocation.

**Q29-1** — *Economic calibration* — Does graduated penalty (warn at 48h, partial hold at 60h) retain more providers near the 72h boundary than binary seizure?
Measure: survival curves for providers in the 48–72h absence zone.

**Q30-2** — *DCUtR retry count* — Should the libp2p DCUtR retry count be confirmed at 1 or returned to 3?
ADR-021 now specifies retry count = 1 based on Paper 30's 97.6% first-attempt success finding. Measure at V2 beta: audit challenge p99 latency under relay; if p99 exceeds 400 ms, re-evaluate.

**Q31-1** — *Scoring calibration* — What threshold drop in the 7d vs 30d score should trigger the partial escrow hold?
ADR-024 starting value: 0.20 drop. Measure: examine provider score trajectories before silent departures in V2 beta; the threshold should catch > 80% of departing providers before the 72h boundary.

**Q33-1** — *Economic calibration* — What holding percentage and holding period maximise provider retention while maintaining deterrence?
ADR-024 starting values: 30-day rolling hold, 50% cap during vetting. Measure: cohort analysis.

**Q34-2** — *Repair budget* — What fraction of a provider's declared upload bandwidth is consumed by repair events in steady state?
Measure: log per-provider repair transfer volume over the first 3 months; compare to Giroire BWavg prediction.

**Q35-2** — *Razorpay fees* — Is the per-transaction payout fee model sustainable at V2 scale?
Measure: aggregate payout fee expense at 1,000 providers, 3,000 providers, and 10,000 providers; confirm with Razorpay account manager that fee schedules are stable.

**Q36-1** — *Failure correlation model parameters* — What are the bi-exponential failure model parameters (α, ρ₁, ρ₂) for Indian ISP failure events?
The analytical framework is established (Paper 36 Refined Fluid Model). V3 provisioning target is set at BWavg × 10 as a conservative upper bound. The exact parameters require V2 telemetry.
Measure: after 6 months of V2 operation, fit G(α, ρ₁, ρ₂) to observed provider failure events grouped by ASN. Use the resulting σ_correlated to refine the V3 provisioning target.

**Q38-1** — *Erasure coding empirical validation* — Does the 20% ASN cap provably keep RS(16,56) in the superior region under observed Indian ISP correlated failures?
The analytical answer is yes (closed as Q38-2 — see answered-questions.md). This question now asks for empirical confirmation using real failure data from V2.
Measure: after 6 months, compute the observed maximum correlated failure event size (shards from the same ASN failing together); confirm it remains below the 11-shard analytical bound.

**Q39-1** — *Hitchhiker adoption gate* — Does observed V2 BWavg exceed the 60 Kbps/peer threshold that justifies Hitchhiker implementation?
The decision gate is set at 60 Kbps/peer (54% above Giroire prediction). This is a telemetry gate, not a research question. See ADR-026 for the adoption criteria.
Measure: mean and variance of per-provider repair transfer volume over the first 6 months.

**Q40-1** — *Nash equilibrium stability* — At what provider count N does bav/bc − 1 ≥ 2.0?
Computable once the storage rate (paise/GB/month) is set. At any positive storage rate and N > 1 the condition is satisfied. The question is about the specific N at which the margin becomes comfortable.

**Q40-2** — *Economic heterogeneity* — What is the empirical distribution of marginal storage cost among Indian home desktop providers?
Measure: provider survey at V2 beta registration; estimate marginal cost from declared free disk space and hardware tier.

---

## Tier 3 — V3 Scope

Valid questions explicitly deferred. Each entry notes what V3 milestone they feed.

---

**Q01-4** — *Peer selection* — Geographic proximity as a viable assignment criterion. Feeds: Coral DSHT implementation.

**Q05-3** — *Mobile providers* — At what mobile MTTF does the business model break? Feeds: V3 mobile provider tier design.

**Q07-4** — *Reputation* — Can correlated failure detection be distributed using EigenTrust on repair-event interactions? Feeds: V3 distributed reputation.

**Q08-4** — *Polling* — Should polling interval t be adaptive based on score history? Feeds: V3 reliability scoring model.

**Q08-5** — *Scoring* — Can the scorer distinguish a diurnally-absent provider from a permanently-departed one before t expires? Feeds: V3 reliability scoring model.

**Q09-4** — *Mobile* — Actual free-space fraction on modern Indian smartphones. Feeds: V3 mobile provider tier.

**Q09-5** — *Mobile* — Mobile-specific 72h departure threshold. Feeds: V3 mobile provider tier.

**Q09-6** — *Mobile* — Maximum safe lazy-update window at mobile MTTF ~30 days. Feeds: V3 erasure parameters for mobile.

**Q13-2** — *Repair* — Should GossipSub be used for repair event propagation to surviving fragment holders? Feeds: V3 repair scheduler if microservice throughput becomes a bottleneck.

**Q15-1** — *Mobile encryption* — Can client-side RS erasure coding on a mobile device fit within battery and CPU budgets? Feeds: V3 mobile encoding pipeline.

**Q19-1** — *Erasure coding* — ECWide locality in a P2P network with no rack topology. Feeds: V3 wide-stripe optimisation.

**Q24-1** — *Reputation* — Minimum repair interactions per provider pair before EigenTrust PSM score is statistically meaningful. Feeds: V3 distributed reputation.

**Q24-2** — *Decentralisation* — What replaces the microservice pre-trusted anchor in V3? Feeds: V3 decentralised coordination.

**Q29-2** — *Erasure + pricing* — Should hot-band uploads specify different erasure parameters at upload time? Feeds: V3 hot band (ADR-018) pricing and ADR-026 multi-tier design.

**Q31-2** — *Reputation* — PSM ratings from repair interactions at V3 scale. Feeds: V3 distributed reputation.

**Q33-2** — *Economic mechanism* — Multi-task incentive weight function for V3 hot band. Feeds: V3 hot band economic design.

**Q34-1** — *Storage engine* — Within-vLog hot/cold tiering for frequently-accessed vs archival chunks. Feeds: V3 provider daemon storage tiering.

**Q36-2** — *Repair* — Periodic chunk rebalancing to homogenise provider disk fill ratios. Feeds: V3 repair scheduler + Dalle variance reduction.

**Q37-1** — *Trust model* — At what provider concentration does microservice capture become a realistic threat? V3 Transparent Merkle Log (ADR-015) is the architectural response.

**Q37-2** — *Audit scalability* — Minimum pau for probabilistic audit sampling while maintaining SHELBY Theorem 1 Condition (i). Feeds: V3 audit scheduler design.


## Open questions from requirements.md section 10

Questions are in three tiers: **Product** (business decisions blocking private beta),
**Telemetry** (can only be answered by observing the live V2 system; none block the build),
and **V3 Deferred** (valid questions explicitly out of scope for V2).

All Tier 1 build-blocker questions are resolved. See §11.3 for the closed record.

---

### 10.1 Product Open Questions

These require a business or product decision before private beta opens.

| # | Question | Owner | Due | Linked |
| --- | --- | --- | --- | --- |
| OQ-001 | What is the storage rate (paise per GB per month) that makes participation economically viable for providers at MTTF ≥ 180 days while keeping data owner costs below cloud alternatives? | Product | Before private beta | ADR-024, FR-013, Paper 40 |
| OQ-002 | What fraction of Indian home routers are behind symmetric NAT, requiring Circuit Relay v2 permanently? Global baseline is ~30% (Paper 30); Indian CGNAT prevalence may be higher. Measure via AutoNAT classification at provider registration (Q20-1). | Engineering | Private beta — 30 days post-launch | ADR-021, NFR-006 |
| OQ-003 | Does the dual-window partial hold trigger (0.20 drop in 7d vs 30d score) correctly catch degrading providers before the 72h threshold without penalising providers with legitimate weekend absences? (Q31-1) | Engineering | Private beta — 90 days post-launch | ADR-024 |
| OQ-004 | What RocksDB rate limiter value keeps p99 audit latency ≤ 200 ms under concurrent compaction on a consumer 7200 RPM HDD? Run benchmark protocol §7.5 Q27-1 before GA. | Engineering | Before V2 GA | ADR-023 |
| OQ-005 | At what observed BWavg does Hitchhiker code adoption for V3 become justified? Decision gate: if V2 BWavg exceeds 60 Kbps/peer over the first 6 months, implement. (Q39-1) | Engineering | 6 months post-launch | ADR-026 |

### Q42-1 — Does a paid, escrow-penalized marketplace need a public leaderboard the way BOINC's unpaid volunteer model does?

**Raised by:** Paper 42 (BOINC)
**Status:** open
**Blocked on:** the Impact Analytics design (`ux-decisions.md` §8) — real money may change whether ranking against other providers motivates or discourages participation. Not resolved by Paper 42, which only covers an unpaid context.

---

### Q43-1 — If Vyomanaut pursues an institution-led rollout, is the unit of trust the institution's IT department, or the individual machine's user with institutional awareness but no formal sign-off?

**Raised by:** Paper 43 (Condor)
**Status:** open
**Blocked on:** the business-development decision in `ux-decisions.md` §5.2. Neither Condor's IT-led model nor BOINC's individual-opt-in model (Paper 42) precedents a hybrid directly.

---

### Q44-1 — What is a defensible, citable per-terabyte or per-server embodied-carbon figure, current enough to show a specific number to an individual provider?

**Raised by:** Paper 44 (Masanet et al.)
**Status:** open
**Blocked on:** a dedicated embodied-carbon source — Paper 44 only grounds the operational-energy half of the Impact Analytics claim, not the manufacturing/embodied half the actual pitch depends on.

---

### Q45-1 — Does Wails' in-process binding model constrain a future browser-based or mobile companion app?

**Raised by:** Paper 45 (Wails/Electron)
**Status:** open
**Blocked on:** scoping of any future mobile companion. The existing REST API (M11) is expected to serve that need independent of the desktop shell choice, but this has not been explicitly verified against Wails' architecture.

---

### Q46-1 — What fraction of Storj's total network capacity comes from its Commercial Storage Node Operator Program versus individual/home operators?

**Raised by:** Paper 46 (Storj node operator docs)
**Status:** open
**Blocked on:** Storj does not publish this split publicly, as far as this review found. Without it, the size of the "true home consumer" segment within Storj's existing network is an estimate, not a measured figure.

---

### Q47-1 — How much of Vyomanaut's reliability-score/payout-multiplier formula should be disclosed to providers, given that full opacity risks the trust breakdown this paper documents, and full disclosure risks the score being gamed?

**Raised by:** Paper 47 (Rosenblat & Stark)
**Status:** resolved by ADR-045 — disclose gross amount, multiplier applied, and a plain-language reason category; withhold the exact formula and thresholds.
**Blocked on:** — (the reason-category list itself remains to be finalised alongside the scoring implementation, per ADR-045's "Open constraints")

### Q48-1 — Is the fiat/UPI advantage over crypto-token wallets durable, or could improved wallet UX narrow this gap over time?

Raised by: Paper 48 (Voskobojnikov et al., crypto wallet UX)
Status: open
Blocked on: nothing actionable now — flagged as a watch item. The crypto industry's own 2025 account-abstraction efforts (e.g., EIP-7702) are attempting to close exactly this gap; revisit if wallet UX changes materially before any ADR treats the fiat/UPI advantage as permanent rather than current

---

### Q49-1 — Does BadgerDB's write/read throughput advantage over general-purpose RocksDB at 256 KB values hold on Vyomanaut's actual Linux/macOS provider hardware, to a degree that justifies retiring the RocksDB/vLog path everywhere rather than only on Windows?

**Raised by:** Paper 49 (BadgerDB)
**Status:** open
**Blocked on:** empirical benchmarking against real provider hardware post-launch. Not blocking any current milestone — ADR-046 scopes BadgerDB to the Windows build only, where it resolves a build-blocking gap; the Linux/macOS RocksDB path (ADR-023) is unaffected and already CI-proven. This question is about whether to later retire a second, redundant engine implementation, not about unblocking a build.

---

### 10.2 Engineering Open Questions — Telemetry

None of the following block the build. All require observing the live V2 system.
Questions marked **[monitoring note]** have their design decision locked; only the
empirical validation remains open.

**Provider economics and survival**

- **Q05-4** — What held-earnings percentage and vetting period length empirically achieve MTTF 180–380 days? Starting values in ADR-024: 30-day rolling hold, 50% release cap during vetting. Measure: provider survival curves by cohort; adjust multipliers if median provider tenure < 6 months.
- **Q08-1** — What is the actual MTTF of a financially-incentivised Indian desktop provider? Measure: survival analysis on the first provider cohort; compare to Bhagwan's unincentivised floor (median session ~1–2h) and Bolosky's corporate desktop ceiling (MTTF 290–380 days). Feeds: ADR-003 MTTF assumption validation.
- **Q08-3** — At what provider count does a 20%/day turnover rate produce more repair bandwidth than idle upload capacity can absorb? Framework: `turnover_rate × Qpeek / (N × 100 KB/s)`. The repair window table in `architecture.md §27.3` shows the window exceeds 12h at N < 500; the actual turnover rate is unknown until V2. Measure: log per-departure transfer volume against provider count.
- **Q20-3** — What held-earnings percentage empirically achieves MTTF 180–380 days from an initially unincentivised population? Measure: survival curves for the first provider cohort; compare against Trautwein's unincentivised floor (87.6% of sessions under 8h).
- **Q21-1** — What fraction of registered providers pass the 4–6 month vetting period vs are rejected or depart? Measure: cohort analysis at 6 months. If reject rate > 40%, investigate registration gate or vetting criteria.
- **Q29-1** — Does graduated penalty (warn at 48h, partial hold at 60h) retain more providers near the 72h boundary than binary seizure? Measure: survival curves for providers in the 48–72h absence zone. Feeds: ADR-007 potential refinement.
- **Q33-1** — What holding percentage and holding period maximise provider retention while maintaining deterrence? ADR-024 starting values: 30-day rolling hold, 50% cap during vetting. Measure: cohort analysis.
- **Q40-1** *(method resolved)* — At what provider count N does the Nash equilibrium stability condition bav/bc − 1 ≥ 2.0 hold comfortably? **Analysis:** the condition is satisfied at any positive storage rate and N > 1. The question is the specific N at which the margin becomes operationally comfortable. Measure: compute bav/bc after OQ-001 storage rate is set; validate against first-cohort economics.
- **Q40-2** — What is the empirical distribution of marginal storage cost among Indian home desktop providers? Measure: provider survey at V2 beta registration; estimate marginal cost from declared free disk space and hardware tier. Feeds: OQ-001 rate-setting.

**Network and transport**

- **Q14-1** — What fraction of Indian home ISPs block UDP, forcing all QUIC connections to the TCP fallback? Measure: log transport type per provider connection at registration; report UDP-block rate at 30 days.
- **Q14-3** — What is the false-positive rate of the JIT detector (`response_latency_ms < deadline × 0.3`) during QUIC connection migration events? Measure: correlate QUIC migration signals with latency spikes over the first month. If false-positive rate > 1%, add a migration flag to the audit receipt schema.
- **Q30-2** *(design locked — validation pending)* — **The DCUtR retry count is confirmed at 1** in `architecture.md §13` (ADR-021), based on Paper 30's 97.6% first-attempt success rate. Validation: monitor audit challenge p99 latency under relay at private beta. If p99 exceeds 400 ms, re-evaluate to retry count = 2.

**Scoring and audit**

- **Q06-3** — Does t=24h need production tuning? Measure: false-positive departure declarations (providers declared departed who return within 72h) over the first 3 months. Adjust if false-positive rate > 5%.
- **Q31-1** — What threshold drop in the 7d vs 30d score should trigger the dual-window hold? ADR-024 starting value: 0.20 drop. Measure: examine provider score trajectories before silent departures; the threshold should catch > 80% of departing providers before the 72h boundary.
- **Q34-2** — What fraction of a provider's declared upload bandwidth is consumed by repair in steady state? Measure: log per-provider repair transfer volume over the first 3 months; compare to Giroire BWavg prediction.

**Storage and disk**

- **Q23-1** — Is the ~10–15% write throughput penalty at lf=256 KB vs lf=512 KB observable on Indian desktop hardware? Measure: run Q27-1 benchmark protocol (§7.5) at both entry sizes. If gap > 20%, re-evaluate lf — requires co-ordinated ADR-003 + ADR-004 update.
- **Q27-2** — What is the empirical rate of within-provider burst failures (multiple chunks failing simultaneously)? Measure: log the count of chunks simultaneously invalidated per provider failure event. If multi-chunk bursts > 1% of events, evaluate sparse vLog pre-allocation.

**Failure correlation**

- **Q19-3** — Does the > 98% single-chunk failure rate (Facebook warehouse observation) hold in a P2P consumer desktop network with correlated failures? Measure: log failure event sizes over the first 6 months. If multi-chunk bursts > 2% of events, re-evaluate whether MSR repair bandwidth targets the right case.
- **Q36-1** — What are the bi-exponential failure model parameters (α, ρ₁, ρ₂) for Indian ISP failure events? Measure: after 6 months, fit G(α, ρ₁, ρ₂) to observed provider failure events grouped by ASN. Use resulting σ_correlated to refine V3 provisioning target.
- **Q38-1** — Does the 20% ASN cap empirically keep RS(16,56) in the analytically superior region under real Indian ISP correlated failures? Measure: after 6 months, compute the observed maximum correlated failure event size; confirm it remains below the 11-shard analytical bound.

**Payment**

- **Q35-2** — Is the Razorpay per-transaction payout fee model sustainable at V2 scale? Measure: aggregate payout fee expense at 1,000, 3,000, and 10,000 providers; confirm with Razorpay account manager.

**Provider onboarding**

- **Q01-5** — Does reliability-proportional assignment create a runaway Matthew effect where the top 5% of providers receive > 50% of new chunk assignments? Power of Two Choices in ADR-005 is the structural mitigation. Measure: track cumulative chunk assignment distribution vs reliability score percentile over the first 90 days. Trigger: if the top decile holds > 40% of assignments, add a concentration cap.
- **Q05-1** — What practical challenges does the central microservice pose at launch scale, and which satellite functions can be decentralised in V3? Measure: microservice latency percentiles, failure incidents, and operational overhead over the first 6 months.
- **Q05-7** — At what file size does per-segment pointer file metadata overhead become user-visible? Measure: track upload distribution at launch; compute metadata-to-data ratio by file size decile; set an inline threshold if the smallest decile shows > 5% metadata overhead.

---

### 10.3 V3-Deferred Questions

Valid questions explicitly out of scope for V2. Each references the V3 milestone they feed.

| ID | Domain | Question summary | Feeds |
| --- | --- | --- | --- |
| Q01-4 | Peer selection | Geographic proximity as an assignment criterion | Coral DSHT implementation |
| Q05-3 | Mobile providers | At what mobile MTTF does the business model break? | V3 mobile provider tier design |
| Q07-4 | Reputation | Can correlated failure detection be distributed using EigenTrust on repair-event interactions? | V3 distributed reputation |
| Q08-4 | Scoring | Should polling interval t be adaptive based on score history? | V3 reliability scoring model |
| Q08-5 | Scoring | Can the scorer distinguish a diurnally-absent provider from a permanently-departed one before t expires? | V3 reliability scoring model |
| Q09-4–6 | Mobile | Free-space fraction on Indian smartphones; mobile departure threshold; max safe lazy-update window at MTTF ~30 days | V3 mobile provider tier |
| Q13-2 | Repair | Should GossipSub be used for repair event propagation? | V3 repair scheduler |
| Q15-1 | Mobile encryption | Can client-side RS erasure coding on mobile fit within battery and CPU budgets? | V3 mobile encoding pipeline |
| Q19-1 | Erasure coding | ECWide locality in a P2P network with no rack topology | V3 wide-stripe optimisation |
| Q24-1, Q24-2 | Reputation | Minimum repair interactions before EigenTrust score is meaningful; replacement for microservice pre-trusted anchor | V3 distributed reputation and coordination |
| Q29-2 | Erasure + pricing | Should hot-band uploads specify different erasure parameters at upload time? | V3 hot band pricing and ADR-026 multi-tier design |
| Q31-2, Q33-2 | Reputation / economics | PSM ratings from repair interactions; multi-task incentive weight function | V3 distributed reputation; V3 hot band economics |
| Q34-1 | Storage engine | Within-vLog hot/cold tiering for frequently-accessed vs archival chunks | V3 provider daemon storage tiering |
| Q36-2 | Repair | Periodic chunk rebalancing to homogenise provider disk fill ratios | V3 repair scheduler + Dalle variance reduction |
| Q37-1, Q37-2 | Trust / audit scalability | At what provider concentration does microservice capture become a realistic threat? Minimum audit frequency for probabilistic sampling under SHELBY Theorem 1 | V3 Transparent Merkle Log; V3 audit scheduler |
| Q39-1 (V3 path) | Repair BW | Hitchhiker code adoption after V2 telemetry gate (OQ-005) triggers | V3 ADR-026 implementation |

---

### 11.3 Answered Questions

The following questions were open during the research phase and are now closed.
Answers are locked in the referenced ADRs and must not be re-opened without a
superseding ADR. The full resolution record is in `docs/research/answered-questions.md`.

#### Coordination and DHT

| Question | Answer | ADR |
| --- | --- | --- |
| How to avoid the tracker as a single point of failure? | Kademlia DHT replaces the tracker for all peer and chunk discovery. | ADR-001 |
| How to pseudonymise chunk IDs in the DHT without breaking FIND_VALUE? | DHT lookup key = `HMAC-SHA256(chunk_hash, file_owner_key)` where `file_owner_key = HKDF(master_secret, "vyomanaut-dht-v1", file_id)`. The DHT never sees chunk_hash or file_id. | ADR-001 |
| How to set DHT branching factor and concurrency? | k-bucket size k=16, α=3 parallel lookups. O(log n / 3) round trips. | ADR-001, ADR-021 |
| What replaces blockchain as the neutral audit trail? | Write-once append-only audit log. Both provider and microservice sign each receipt with Ed25519. INSERT-only Postgres (row security policy). V3 upgrade: Transparent Merkle Log. | ADR-015 |
| What practical attacks does DHT pseudonymisation close? | Closes DSN Challenge 3 from the SoK survey — a monitoring node recording all DHT traffic cannot correlate lookups to files without the owner's master secret. | ADR-001 |

#### Erasure Coding and Repair

| Question | Answer | ADR |
| --- | --- | --- |
| What is Qpeek at N=1,000, 50 GB/provider? | ~793 GB per failure event. At 100 Kbps/peer aggregate, repair completes in ~8 hours — within the 12-hour safety window. | ADR-003, ADR-004 |
| What is BWavg at target parameters? | ~39 Kbps/peer at MTTF=300 days, N=1,000, 50 GB/peer (Giroire Formula 1). | ADR-003 |
| At what correlated failure rate does RS(16,56) become worse than a simpler scheme? | Never, under the 20% ASN cap. The reversal condition (Paper 38) requires correlated failure size to approach r=40. The ASN cap bounds maximum correlated failure at ~11 shards, leaving 44 survivors — 28 above the reconstruction floor. | ADR-003, ADR-014 |
| What is the optimal lazy repair strategy? | Single r0=8 (desktop-only V2 collapses the tier model). Reduces bandwidth ~38× vs eager repair. | ADR-004 |
| Is hinted handoff needed for the (3,2,2) quorum? | No. If replica A is down during a write, replicas B and C both ACK (W=2 satisfied). Anti-entropy gossip reconciles A's state within seconds of its return. | ADR-025 |

#### NAT Traversal and Transport

| Question | Answer | ADR |
| --- | --- | --- |
| Does Circuit Relay v2 violate the audit response deadline? | No. Relay RTT < 50 ms from Indian cloud regions. Two relay legs = < 100 ms overhead, within the 614 ms deadline at 5 Mbps. | ADR-021 |
| What is the latency cost of forcing 1-RTT for audit reconnects? | 5–90 ms depending on NAT type (5 ms same-city, ~40 ms cross-city, +50 ms for relay-dependent). Worst case 90 ms, well within the 614 ms deadline. 0-RTT remains disabled for audit interactions. | ADR-021 |
| How many relay nodes are required at launch? | 3 nodes (Mumbai AZ1, Mumbai AZ2, Chennai/Hyderabad), 128 concurrent reservations each = 384 slots. 4.3× headroom at 300 initial providers. Scale to 4th node when provider count exceeds 570 (45% CGNAT assumption) or 850 (30% baseline). | ADR-021 |
| Q20-1 — What fraction of Indian home routers are behind symmetric NAT? | Design-time assumption: 45% (conservative upper bound). Sources: Paper 30 measures ~30% globally; Indian ISPs have broader CGNAT deployment. Relay infrastructure is sized for 45% to be conservative. Scale trigger: provision 4th relay node before provider count exceeds 570 (45% assumption) or 850 (30% baseline). Empirical validation via AutoNAT classification telemetry post-launch does not block the build; it informs the scale trigger. If observed relay-dependent fraction <30%, the 4th node can be deferred. If >45%, provision the 4th node immediately. | ADR-021, Paper 30, architecture.md §27.2 |

**DCUtR retry count — confirmed at 1.** Paper 30 shows 97.6% first-attempt success rate from 4.4M traversal attempts. Setting retry count to 1 (not the libp2p default of 3) is correct and is confirmed in `architecture.md §13`. Validation: monitor audit challenge p99 latency under relay at private beta. (ADR-021)

#### Encryption and Key Management

| Question | Answer | ADR |
| --- | --- | --- |
| Does code-then-encrypt with per-chunk keys improve on AONT-RS? | No. AONT-RS achieves 2^256 computational security with zero external key management. Code-then-encrypt with 56 keys per file offers no meaningful security improvement. | ADR-022 |
| Should the Ed25519 signing key be derived from master_secret or stored separately? | Store separately, encrypted under a key derived from master_secret (HKDF `"vyomanaut-keystore-v1"`). A compromised signing key can then be rotated without rotating master_secret or re-encrypting any data. | ADR-020 |
| What fraction of Indian desktop providers lack AES-NI? | Planning estimate: 10–15% lack AES-NI (Celeron N-series, Atom). Both cipher paths must be production-quality. CPUID detection at daemon startup is non-negotiable. | ADR-019 |

#### Erasure Code Selection

| Question | Answer | ADR |
| --- | --- | --- |
| Are Clay / MSR codes feasible at (n=56, k=16)? | No. Sub-packetisation α = 40^16 ≈ 10^25 — computationally intractable (Paper 22). MSR and Clay codes are rejected. Hitchhiker (α=2) is the only viable V3 candidate if BWavg telemetry gate triggers. | ADR-026 |
| Are LRC codes viable? | No. Non-MDS; local group co-locality cannot be guaranteed in a consumer P2P network; repair benefit collapses to RS-level under Indian ISP conditions. | ADR-026 |

#### Storage Engine

| Question | Answer | ADR |
| --- | --- | --- |
| Fixed or variable vLog entry size? | Fixed (262,212 bytes = 256 KB chunk + headers). GC tail advancement requires only arithmetic, no parsing. | ADR-023 |
| Is proactive continuous disk scrubbing justified? | No. The base UE rate is 2–6 per 1,000 drive days (Schroeder et al.). Scrubbing is reactive: triggered by the first audit FAIL. | ADR-023 |
| Does the chunk index fall in the hot, warm, or cold data regime? | Moot at 256 KB values. WiscKey eliminates value movement from compaction — write amplification ≈ 1.0 regardless of access regime. | ADR-023 |

#### Economic Mechanism and Payment

| Question | Answer | ADR |
| --- | --- | --- |
| How should Razorpay `on_hold_until` be set to target first-3-business-days release? | Embed a static `rbi_bank_holidays_YYYY` table (updated each December). Monthly release job runs on the 23rd: set `on_hold_until` to the last working day of the current month. Route releases on the next business day, landing within the first 1–3 days of the following month. | ADR-024 |
| How should the service-denial attack be monitored? | Three layers: (1) structural — RS(16,56) requires > 40 simultaneous refusals; ASN cap limits any group to ~11. (2) scoring signal — 3 independent data owner retrieval failure reports from the same provider within 72h rolling window → 0.3× audit FAIL weight for 24h window. (3) V3 upgrade — repair-event interactions provide microservice-visible retrieval evidence. | ADR-014, ADR-008 |

#### Audit Scalability

| Question | Answer | ADR |
|---|---|---|
| At what provider count does daily full audit become infeasible? | ~100,000 providers × 10,000 chunks (planning estimate). At N=10,000 × 10,000 chunks, challenge rate is 1,157/sec — well within Postgres capacity. At 100,000 × 10,000 = ~11,574/sec, sharding or probabilistic sampling is needed. SHELBY Theorem 1 must be re-verified if sampling is introduced. | ADR-002 |
