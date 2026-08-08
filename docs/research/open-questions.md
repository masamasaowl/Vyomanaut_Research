# Open Questions

One entry per question. Every question here is genuinely open — resolved questions are moved to
[`answered-questions.md`](./answered-questions.md), not left here with a pasted-in answer, so this
file only ever contains work still to be done.

Format for individually-raised questions: `### QNN-N — [short question]`, then **Raised by:**,
**Status:**, **Blocked on:**. When a question resolves, cut it from this file and paste it —
with its answer and governing ADR — into `answered-questions.md`.

> **Provenance note:** this content was folded into `requirements.md` §10 for a period, and a
> second, unrelated numbering scheme (Tier 1/2/3) from an earlier stage of the project was mixed
> in alongside it. Both have now been reconciled into the single structure below.
> `requirements.md` §10 is a short pointer back to this file — it does not carry its own copy of
> these questions, so there is exactly one place to look.

---

## Status

**All Tier 1 build-blocker questions are resolved.** There are no open questions blocking the V2
build. The closed record is summarized in `requirements.md` §11.3 and given in full, with the
governing ADR for each, in [`answered-questions.md`](./answered-questions.md).

Everything below is one of three kinds of *non*-blocking open question, plus a set of newer
questions raised by later research (Papers 42–49) that haven't yet been triaged into those three
tiers.

---

## Product Open Questions

These require a business or product decision before private beta opens.

| # | Question | Owner | Due | Linked |
| --- | --- | --- | --- | --- |
| OQ-001 | What is the storage rate (paise per GB per month) that makes participation economically viable for providers at MTTF ≥ 180 days while keeping data owner costs below cloud alternatives? | Product | Before private beta | ADR-024, FR-013, Paper 40 |
| OQ-002 | What fraction of Indian home routers are behind symmetric NAT, requiring Circuit Relay v2 permanently? Global baseline is ~30% (Paper 30); Indian CGNAT prevalence may be higher. Measure via AutoNAT classification at provider registration (Q20-1). | Engineering | Private beta — 30 days post-launch | ADR-021, NFR-006 |
| OQ-003 | Does the dual-window partial hold trigger (0.20 drop in 7d vs 30d score) correctly catch degrading providers before the 72h threshold without penalising providers with legitimate weekend absences? (Q31-1) | Engineering | Private beta — 90 days post-launch | ADR-024 |
| OQ-004 | What RocksDB rate limiter value keeps p99 audit latency ≤ 200 ms under concurrent compaction on a consumer 7200 RPM HDD? Run benchmark protocol `requirements.md` §7.5 Q27-1 before GA. | Engineering | Before V2 GA | ADR-023 |
| OQ-005 | At what observed BWavg does Hitchhiker code adoption for V3 become justified? Decision gate: if V2 BWavg exceeds 60 Kbps/peer over the first 6 months, implement. (Q39-1) | Engineering | 6 months post-launch | ADR-026 |

---

## Engineering Open Questions — Telemetry

None of the following block the build. All require observing the live V2 system. Questions
marked **[design locked]** have their design decision already made; only the empirical
validation remains open.

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

- **Q23-1** — Is the ~10–15% write throughput penalty at lf=256 KB vs lf=512 KB observable on Indian desktop hardware? Measure: run Q27-1 benchmark protocol (`requirements.md` §7.5) at both entry sizes. If gap > 20%, re-evaluate lf — requires co-ordinated ADR-003 + ADR-004 update.
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

*(Q20-1 and Q39-1 are not listed here — both were promoted to the Product tier as OQ-002 and
OQ-005 respectively, since each now gates a business/infrastructure decision rather than only
informing a tuning parameter.)*

---

## V3-Deferred Questions

Valid questions explicitly out of scope for V2. Each references the V3 milestone it feeds.

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

### Q26-1 — At what provider count N does Hitchhiker's 25–45% repair-bandwidth reduction become economically necessary rather than optional, once Paper 36's 22× burst-variance multiplier is accounted for?

Measure: once V3 provider-count growth data exists, substitute the Hitchhiker-adjusted β into
Paper 10's Formula 1 and compare against the background bandwidth budget, using Paper 36's
correlated-burst variance rather than the mean. Not blocking ADR-026 — the code-family decision
(Hitchhiker, not Clay) doesn't depend on pinning down N precisely
---

## Additional Open Questions

Raised during the later literature-review pass (BOINC through BadgerDB). Not yet triaged into
the three tiers above — that's a call for whoever owns the next taxonomy pass, not assumed here.

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

### Q48-1 — Is the fiat/UPI advantage over crypto-token wallets durable, or could improved wallet UX narrow this gap over time?

**Raised by:** Paper 48 (Voskobojnikov et al., crypto wallet UX)
**Status:** open
**Blocked on:** nothing actionable now — flagged as a watch item. The crypto industry's own 2025 account-abstraction efforts (e.g., EIP-7702) are attempting to close exactly this gap; revisit if wallet UX changes materially before any ADR treats the fiat/UPI advantage as permanent rather than current.

---

### Q49-1 — Does BadgerDB's write/read throughput advantage over general-purpose RocksDB at 256 KB values hold on Vyomanaut's actual Linux/macOS provider hardware, to a degree that justifies retiring the RocksDB/vLog path everywhere rather than only on Windows?

**Raised by:** Paper 49 (BadgerDB)
**Status:** open
**Blocked on:** empirical benchmarking against real provider hardware post-launch. Not blocking any current milestone — ADR-046 scopes BadgerDB to the Windows build only, where it resolves a build-blocking gap; the Linux/macOS RocksDB path (ADR-023) is unaffected and already CI-proven. This question is about whether to later retire a second, redundant engine implementation, not about unblocking a build.

---

### Q51-1 — Does Svelte's runtime-footprint advantage over React/Vue, established by js-framework-benchmark in a generic Chromium environment, hold when measured directly inside a Wails-packaged WebView2 host?

**Raised by:** Paper 51 (js-framework-benchmark / Wails template docs)
**Status:** open
**Blocked on:** nothing actionable now — WebView2 is itself Chromium-family, so the qualitative ranking is expected to transfer. This is a watch item to re-verify empirically once M19 has a real build to profile, not a pre-decision blocker.

---

### Q52-1 — Does the Wails/WebView2 accessibility-tree bridge work correctly end-to-end (screen reader announces content, keyboard focus lands in the webview on window activation) in an actual packaged Windows build, not just in the underlying pieces individually?

**Raised by:** Paper 52 (RPwD Act 2016 / IS 17802 / GIGW 3.0; WebView2Feedback#2330; wailsapp/wails#4535)
**Status:** open
**Blocked on:** a direct Windows Narrator smoke test against a real packaged build — scheduled as part of ADR-050's baseline, due before the M19 shared component library is considered done. Not a pre-launch blocker on its own: the underlying WebView2 focus bug is already confirmed fixed upstream (Windows App SDK 1.5), and only the Wails-specific hosting path remains unverified.

---

### Q53-1 — Two independent verifications gate ADR-051's plan: (1) does Vyomanaut's NSIS installer complete via the `/S` silent flag without a UAC elevation prompt, and (2) has Wails v3's built-in updater actually landed in a tagged release, not just a development branch?

**Raised by:** Paper 53 (Wails v3 updater docs; Tailscale issue tracker; wailsapp/updater-demo)
**Status:** open
**Blocked on:** (a) a direct test of Vyomanaut's own NSIS installer configuration run silently via `/S` — gates Phase 1, needed before it ships, not just before it's considered complete; (b) confirming the Wails v3 updater has merged into a tagged release rather than remaining on a branch — gates Phase 2's migration timing. Neither blocks the other.

---

### Q54-1 — Does the Paraglide JS Vite plugin integrate cleanly with Wails' own build pipeline (`wails build`, which wraps Vite) for a plain Svelte + Vite project, rather than the SvelteKit setup its documentation assumes?

**Raised by:** Paper 54 (Paraglide JS documentation; ADR-049's plain Svelte + Vite decision)
**Status:** open
**Blocked on:** a direct build test once M19 has a real Wails + Svelte project scaffolded — not blocking adoption of the pattern now, since Paraglide's core compiler is documented to work in "any Vite app," only its interaction with Wails' specific Vite wrapping is unconfirmed.

---

- **Q56-1** — What is the maximum extractable payout per fake identity during a full 90-day forced-ceiling vetting run, in currency terms, given real per-audit rates and declared-storage-driven vetting-chunk caps? Blocked on: implementation-level rate constants not sourced in this research pass.

- **Q56-2** — What identity-creation cost (phone/KYC friction) is required to keep expected Sybil-farming value below Q56-1's bound? Blocked on: Q56-1, and real registration-fraud data not available pre-launch.

---

- **Q57-1** — At what point should Vyomanaut re-evaluate true ISP-API integration if/when DEPA's telecom-sector Consent Manager rollout goes live? Blocked on: a regulatory event outside Vyomanaut's control; no fixed timeline exists as of this research pass.

- **Q57-2** — What monitoring policy applies to a provider forced to graduate at the 90-day ceiling with sub-threshold confidence? Blocked on: ADR-008 scoring implementation decision, not yet made.

- **Q57-3** — Does the current ADR-005 implementation already contain an early-exit-on-audit-count path, or is the 120–180 day window applied as a fixed calendar duration regardless of performance? Blocked on: implementation review — the cash-burn comparison in ADR-054 depends on this answer.

---

- **Q58-1** - Does daemon-managed wake-lock require elevated/administrator privileges on any of the three platforms, and does that conflict with ADR-042's least-privilege design for the provider app? Blocked on: implementation-level testing against ADR-042, not performed in this research pass.

---

### Q59-1 — Should Vyomanaut adopt automatic, continuously re-evaluated Hot/Cold band re-classification (HALO/Zebra's dynamic model), or keep the current one-time, data-owner-declared band choice indefinitely?

**Raised by:** Paper 59 (HALO), Paper 60 (Zebra)
**Status:** open — architectural decision, not a research gap
**Blocked on:** a product/scope call, not further literature. Dynamic re-tiering requires new infrastructure (per-file access-rate metering at the coordination microservice, a re-tiering trigger, and accepting the re-encode cost of moving between bands whenever hotness drifts) that does not exist today and was never scoped into V2 or the current V3 plan. Static declaration is simpler to ship and matches the existing enrolment flow (ADR-018); dynamic re-tiering would deliver better long-run fit to real access patterns at meaningful added complexity. Recommend deferring to V4+ unless data owners report the static choice mismatching their actual usage in practice.

---

### Q61-1 — Does Vyomanaut's client retrieval path (data owner or auditor reading chunks back) currently request more than the minimum `k=16` shards in parallel, or does it request exactly 16 and block on the slowest response?

**Raised by:** Paper 61 (POCache)
**Status:** open
**Blocked on:** implementation review of the client SDK / retrieval protocol (not yet reached in the dependency-ordered research plan — filed under Topic #12, Tier 4). If retrieval already over-requests, RS(16,56)'s 40-shard surplus likely already delivers most of POCache's straggler-tolerance benefit for free, with no new caching infrastructure required. If not, this is a low-cost latency win worth scoping before considering any caching-tier approach.

---

### Q62-1 — Under ADR-004's `r0 = 8`, does Vyomanaut ever perform a single-fragment repair — and if so, what fraction of repair events are they?

**Raised by:** Paper 62 (LESS), Paper 64 (Hitchhiker)
**Status:** open — **blocks ADR-026's design-council session**
**Blocked on:** first, resolving the ADR-004 trigger-flow ambiguity (Q26-4): steps 3–4 gate all repair on `available count ≤ 24`, but the scheduler-priority paragraph treats permanent-departure jobs as a distinct queued class, which reads as an independent trigger. If everything gates on `≤ 24`, single-fragment repair events are ~0% by construction and every candidate family in ADR-026 delivers a 0% saving — the code-family question then has no value and ADR-026 should close as *no V3 repair-BW optimisation*. If permanent departure enqueues repair independently, single-fragment events exist and the question becomes what fraction, which needs a churn model (Domain D, currently stale).

Note that this is a **specification** question first and a measurement question second. The specification half is answerable now, from ADR-004's own text plus a decision; only the fraction needs data.

---

### Q62-2 — Do MDS-feasible LESS coding coefficients exist at `n−k = 40`, and in which Galois field?

**Raised by:** Paper 62 (LESS)
**Status:** open — conditional on Q62-1 resolving in favour of a code-family change
**Blocked on:** a brute-force search that no published work has run. LESS's sufficient condition is `2^w ≥ nα + (n−k−1)·C(n−1,k)`; at `(56,16)`, `C(55,16) ≈ 2.97 × 10¹³`, so the condition demands `w ≥ 50`. The condition is only sufficient and the authors report smaller fields working in practice, but their table of feasible primitive elements covers only `n−k ≤ 4` and `2 ≤ α ≤ 4`. Independently, `α = 4` requires 224 distinct non-zero coefficients against GF(2⁸)'s 255, so GF(2¹⁶) is the realistic floor — which is a codec change, not a parameter change, and would invalidate any throughput measurement taken under Q65-1 before it.

Do not start this search before Q62-1 is settled. It is expensive and it is worthless if the answer to Q62-1 is that single-fragment repair never happens.

---

### Q63-1 — What is the departure-persistency of RS(16,56) under *correlated* rather than independent departure at `N ∈ [56, 500]`?

**Raised by:** Paper 63 (Friedman, Kapelko & Marchwicki)
**Status:** open — this is F-28, now with an exact baseline to measure the gap against
**Blocked on:** literature the corpus does not have (Domain E: R-16 correlated-failure durability at small scale, R-18 AS-level outage duration and blast radius). Paper 63 assumes departures are independent and uniformly random and does not relax it. The exact independent-case answer at the ADR-029 gate is 41 of 56 departures; the worst-case ASN substitution in ADR-029 Addendum A gives survival of any 3 of 5 simultaneous ASN losses. The distance between those two framings is the whole of F-28, and it is now bounded rather than unknown.

---

### Q63-2 — Does the persistency analysis extend to a mixed Hot/Cold redundancy population, and if not, what replaces it?

**Raised by:** Paper 63 §5.1
**Status:** open — becomes live only if ADR-018 ships two bands with different `(k, r)`
**Blocked on:** unsolved in the source. The authors sketch the non-uniform case — `D_i` documents under `REC(p_i, p_i+q_i, r_i)` — and state that closed-form and asymptotic results for the mixture need further theoretical work. ADR-018's Hot/Cold band model is exactly that case. Whatever `k_hot` is eventually derived (itself a calculation awaiting SLA targets, not a research item), the departure-persistency analysis will not extend to the two-band system without new work. Not blocking: ADR-018 is unshipped and Q59-1 (whether Vyomanaut wants band re-tiering at all) is upstream of this.

---

### Q64-1 — Does Hitchhiker's mandatory-helper structure interact badly with ADR-005 assignment and ADR-014's ASN cap?

**Raised by:** Paper 64 (Hitchhiker)
**Status:** open — conditional on Q62-1, low priority
**Blocked on:** nobody has looked. Hitchhiker's three-step decode requires the second sub-stripe of the *first* parity unit for every data-unit reconstruction, and the second sub-stripe of the `(j+1)`-th parity for a data unit in set `j`. Vyomanaut's placement currently treats all 56 shard positions as interchangeable; under Hitchhiker, specific parity positions become mandatory helpers for specific data positions. If the holder of parity 1 is unreachable — a routine event under ADR-021's NAT and relay constraints — the optimised path is unavailable and repair falls back to plain RS. Whether the 20% ASN cap's placement freedom is sufficient to keep mandatory helpers reliably reachable has not been analysed. Only matters if Hitchhiker is selected, which on current evidence is unlikely (LESS outranks it at our parameters, and both are zeroed by `r0 = 8`).

---

### Q65-1 — What is RS(16,56)'s actual encode and decode throughput on min-spec Indian desktop hardware?

**Raised by:** Paper 65 (BiLP-BX), Paper 62 (LESS)
**Status:** open — **F-27's remaining half; blocks restoring ADR-009's ≤5% CPU claim**
**Blocked on:** a measurement nobody has taken. Paper 65 gives the first data point in the corpus — 258 MB/s for production Cauchy-RS at `(k=10, m=4)`, single-threaded, no SIMD, on an Intel i5-13600KF — and shows throughput falling superlinearly in `k·m`. Vyomanaut's `k·m = 640` is **16× the largest configuration ever benchmarked in that set**, so extrapolation is not available: a power-law fit taken 16× beyond its support, from a paper whose own instruction-count and throughput axes disagree by a factor of 20, is not an estimate.

Three measurements are required, and only the last two bear on ADR-009's budget:

1. RS(16,56) encode throughput, Vyomanaut's own Go codec, min-spec Indian desktop, `lf = 256 KB`, cold cache. *(This is client-side upload latency and belongs to an NFR that does not yet exist.)*
2. Decode-plus-re-encode throughput for a **32-fragment repair event** — the ADR-004 case, and the only genuinely unbudgeted item in ADR-009.
3. Wall-clock and CPU share for one full repair event on a provider simultaneously serving audits.
Two things are already settled by calculation and need no measurement: SHA-256 over 70 GB/day costs at most 0.54% of one core (0.054% with SHA-NI), so the audit path is **cheap in CPU and expensive in disk** — confirming F-40's framing and removing a worry that was never the real one. Any measurement taken now must be re-taken if ADR-026's council session adopts LESS, which cuts encode throughput ~43% and widens the field to GF(2¹⁶).

---
