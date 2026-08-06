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
