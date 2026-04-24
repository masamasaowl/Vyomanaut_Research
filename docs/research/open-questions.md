# Open Questions

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