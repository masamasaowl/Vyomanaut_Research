# Open Questions

Genuinely open questions only. Every entry states why it is open and how to close it.

Three tiers:

**Tier 1 — Build blockers** require a decision or design before the relevant module can be implemented.
**Tier 2 — Telemetry** can only be answered by observing the live V2 system.
**Tier 3 — V3 scope** are valid questions explicitly deferred to the next major version.

Answered questions have been moved to `answered-questions.md`.
Dropped questions (resolved by existing ADRs or no longer relevant) are recorded in `dropped-questions.md`.

---

## Tier 1 — Build Blockers

These must be resolved before implementing the module they govern.

---

**Q10-5** — *Repair protocol* — **Uneven repair burden at launch**
When the network is small (D << N² × lf), block distribution across providers is uneven. Which providers absorb the excess repair load, and how does this affect their BWavg relative to the Giroire prediction?
Blocked on: repair assignment algorithm design in ADR-004 implementation.
How to close: specify whether the repair scheduler assigns replacement chunks to the provider with the most free capacity, or distributes load proportionally to reliability score.

**Q12-2** — *Microservice cluster* — **Hinted handoff for the metadata store**
The (3,2,2) quorum (ADR-025) survives one replica failure without hinted handoff. Should hinted handoff be implemented anyway to preserve write availability during rolling restarts?
Blocked on: staging environment failure testing.
How to close: measure whether writes fail visibly during planned replica restarts in staging; if yes, implement hinted handoff; if no, the quorum alone is sufficient.

**Q14-2** — *P2P transfer* — **0-RTT vs 1-RTT latency penalty on audit reconnects**
ADR-021 disables 0-RTT for all audit interactions to prevent replay attacks (RFC 9000 §8.1). What is the actual p99 latency cost of forcing 1-RTT on every provider reconnect after a nightly absence?
Blocked on: prototype measurement against providers with realistic inter-challenge gaps.
How to close: benchmark 1-RTT reconnect latency from three Indian cloud regions to a simulated desktop provider; confirm it stays within the audit deadline margin.

**Q28-1** — *Availability service* — **Per-provider RTO formula — bootstrapping value**
ADR-006 specifies the timeout formula `RTO = AVG + 4×VAR` (TCP-style, Rhea et al.). What is the initial (bootstrapped) RTO for a new provider with no response-time history?
Blocked on: implementation of the availability service.
How to close: use the 30th-percentile RTT of the vetted provider pool as the initial estimate; store `(avg_rtt_ms, var_rtt_ms)` per provider in the reliability scoring DB; initialise new providers to pool median.

**Q28-2** — *DHT routing* — **Global sampling for proximity optimisation**
Paper 28 (Rhea) shows periodic global sampling — DHT lookups for random target prefixes, selecting the closest responder — gives a 24% lookup latency reduction at virtually no bandwidth cost. With all V2 providers in India, geographic proximity in the DHT is meaningful. Should libp2p Kademlia be configured to run this?
Blocked on: V2 launch inter-provider latency data.
How to close: measure median inter-provider DHT hop latency in the first 30 days of V2 operation; if median > 80 ms, enable global sampling in the libp2p kad-dht configuration.

**Q30-1** — *Infrastructure planning* — **Relay node count at V2 launch**
30% of providers will permanently require Circuit Relay v2 (Paper 30). How many Vyomanaut-operated relay nodes are required to serve V2's provider base without relay becoming a latency bottleneck or single point of failure?
Blocked on: expected V2 provider count at launch.
How to close: relay node count = ceil(0.30 × expected_providers × peak_concurrent_audits / relay_connection_limit). Follow the same (3,2,2) availability model as the microservice cluster (ADR-025) — minimum 3 relay nodes, 2 required for availability.

**Q35-1** — *Payment implementation* — **Razorpay Route `on_hold_until` timestamp for first-3-business-days release**
ADR-024 says "release within first 3 business days of each month." Route releases on the *next business day* after the `on_hold_until` timestamp. The correct timestamp must account for Indian bank holidays.
Blocked on: access to the RBI holiday calendar API or static holiday list.
How to close: integrate the RBI holiday calendar (available from RBI's website as a machine-readable list); set `on_hold_until` to midnight on the last working day of each month so the next business day falls within the first 3 days of the following month.

**Q36-1** — *V3 infrastructure sizing* — **Peak-to-mean repair bandwidth ratio under correlated failures**
Paper 36 (Dalle et al.) shows the real repair bandwidth standard deviation is 22× the independent model's prediction. The Giroire BWavg ≈ 39 Kbps/peer is the mean only. At V3 scale (10,000+ providers), what is the actual peak provisioning target?
Blocked on: V2 launch provider failure telemetry to fit the bi-exponential model parameters.
How to close: after 6 months of V2 operation, fit the Refined Fluid Model (G(α, ρ₁, ρ₂)) to observed provider failure events grouped by ASN; use the resulting standard deviation to set V3 bandwidth headroom.

**Q38-2** — *Erasure coding safety margin* — **At what correlated failure rate does RS(16,56) become worse than a simpler scheme?**
Paper 38 (Nath) proves that at high correlated failure rates, large-m erasure schemes can be worse than smaller-m schemes. Our (m=16, n=56) is the widest stripe in any studied system. Does the 20% ASN cap keep us safely in the superior region?
Blocked on: applying Nath's bi-exponential model numerics to our specific parameters and cap.
How to close: compute the availability curves for RS(16,56) and RS(4,56) under the bi-exponential failure model from Paper 38 using our cap-bounded maximum correlated failure size (20% × 56 = 11 shards); confirm RS(16,56) remains superior at this truncated failure size.

**Q39-1** — *V3 repair bandwidth* — **Combined Hitchhiker + lazy repair reduction**
Paper 39 (Silberstein) shows lazy recovery and bandwidth-efficient codes are orthogonal and combinable — Xorbas+LAZY achieves ~2× additional reduction beyond either alone. At V3 scale, combining Hitchhiker's 25–45% reduction with ADR-004's lazy repair should compound. Does the combined saving justify Hitchhiker's implementation complexity?
Blocked on: V2 launch telemetry confirming actual repair bandwidth at production provider MTTF, and ADR-026 candidate decision.
How to close: after 6 months of V2 data, compute observed BWavg vs Giroire prediction; if actual > predicted by > 20%, evaluate Hitchhiker implementation for V3.

**Q41-1** — *Adversarial defences (ADR-014 gap)* — **Service-denial monitoring design**
Paper 41 (Vakilinia) formally identifies service-denial as a fifth adversarial class: a provider stores data, passes all audits, but refuses client retrieval. ADR-014 does not enumerate this. The structural mitigation (RS(16,56) requires > 40 simultaneous refusals for an effective denial) is sound, but retrieval failures are not currently surfaced to the reliability scorer.
Should retrieval failures be reported to the scorer? If yes, what evidence format prevents a malicious data owner from false-reporting a failure to punish a provider?
Blocked on: retrieval path design in ADR-021 and the audit receipt schema (ADR-017).
How to close: (1) add an optional `retrieval_failure_report` to the signed audit receipt, (2) require the report to include a provider-signed denial response (proves the provider refused, rather than a network failure), (3) weight repeated retrieval failures in the 24h scoring window.

**Q41-2** — *Audit scalability* — **At what provider count does daily full-audit become infeasible?**
At V2 scale, the microservice challenges every chunk on every provider daily. At V3 scale (10,000+ providers × thousands of chunks each), this may exceed microservice throughput.
Blocked on: microservice audit dispatch throughput telemetry at V2 launch.
How to close: measure audit challenge dispatch rate at V2 beta; extrapolate to V3 scale; set the threshold at which probabilistic sampling (pau < 1) becomes necessary; verify the SHELBY Theorem 1 Condition (i) still holds at the reduced pau.

---

## Tier 2 — Telemetry

These cannot be answered by research. They require observing the live V2 system.

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
Measure: log AutoNAT classification per provider at registration; report symmetric-NAT fraction at 30 days (global baseline from Paper 30 is ~30%; India-specific may be higher due to CGNAT prevalence).

**Q20-3** — *Economic mechanism* — What held-earnings percentage empirically achieves MTTF 180–380 days from an initially unincentivised provider population?
Measure: survival curves for the first provider cohort; compare against Trautwein's unincentivised floor (87.6% of sessions under 8h).

**Q21-1** — *Vetting effectiveness* — What fraction of registered providers pass the 4–6 month vetting period vs are rejected or downgraded?
Measure: cohort analysis at 6 months; if reject rate > 40%, investigate whether registration gate or vetting criteria need adjustment.

**Q23-1** — *Fragment size* — Is the ~10–15% write throughput penalty at lf=256 KB vs lf=512 KB observable on Indian desktop hardware?
Measure: benchmark vLog append throughput at 256 KB and 512 KB on a minimum-spec desktop; if gap > 20%, re-evaluate lf (requires coordinated change to ADR-003, ADR-004).

**Q27-2** — *Disk reliability* — What is the empirical rate of within-provider burst failures (multiple chunks failing simultaneously)?
Measure: log the number of chunks simultaneously invalidated per provider departure/failure event; if multi-chunk bursts appear at > 1% of events, evaluate sparse vLog pre-allocation.

**Q29-1** — *Economic calibration* — Does graduated penalty (warn at 48h, partial hold at 60h) retain more providers near the 72h boundary than binary seizure?
Measure: survival curves for providers in the 48–72h absence zone; compare pre-ADR-024-update cohort (if any) with post-update cohort.

**Q30-2** — *DCUtR retry count* — Should the libp2p DCUtR retry count be reduced from 3 to 1?
Paper 30 provides the theoretical basis (97.6% first-attempt success) but the right answer for Vyomanaut's specific audit deadline needs measurement.
Measure: audit challenge p99 latency under relay, with 1-retry vs 3-retry DCUtR; if p99 stays below 400 ms with 1-retry, apply the change.

**Q31-1** — *Scoring calibration* — What threshold drop in the 7d vs 30d score should trigger the partial escrow hold?
ADR-024 starting value: 0.20 drop. Measure: examine provider score trajectories before silent departures in V2 beta; the threshold should catch > 80% of departing providers before the 72h boundary.

**Q33-1** — *Economic calibration* — What holding percentage and holding period maximise provider retention while maintaining deterrence?
ADR-024 starting values: 30-day rolling hold, 50% cap during vetting. Measure: cohort analysis comparing providers at different holding percentages if A/B testing is feasible.

**Q34-2** — *Repair budget* — What fraction of a provider's declared upload bandwidth is consumed by repair events in steady state?
Measure: log per-provider repair transfer volume over the first 3 months; compare to Giroire BWavg prediction.

**Q35-2** — *Razorpay fees* — Is the per-transaction payout fee model sustainable at V2 scale?
Measure: aggregate payout fee expense at 1,000 providers, 3,000 providers, and 10,000 providers; confirm with Razorpay account manager that fee schedules are stable.

**Q38-1** — *Failure correlation* — What are the bi-exponential model parameters (α, ρ₁, ρ₂) for Indian ISP failure events?
Measure: after 6 months of V2 operation, fit the bi-exponential distribution to observed provider failure events grouped by ASN; use to validate that the 20% ASN cap keeps RS(16,56) in the superior region.

**Q40-1** — *Nash equilibrium* — At what provider count N does bav/bc − 1 ≥ 2.0, placing the network in the robustly stable regime?
Computable once the storage rate (paise/GB/month) is set in ADR-024. Formula: `(monthly_earnings / marginal_storage_cost) × (N−1) > 4`. At any positive storage rate and N > 1 this is trivially satisfied; the question is about the specific N at which the stability margin becomes comfortable.

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