# Research Topic List v2 — Dependency-Ordered

**Replaces `reading-list.md` in full.** This is not a continuation of the old phases — it is a fresh
prioritization built from three sources: (1) every ADR's `Status` and `Open constraints` section,
(2) the `Build Dependency Graph` in `build_part3.md`, (3) the four V3 functionality ADRs already
in `Proposed` status (053, 055, 056, 057).

## Ordering rule

The build DAG (M0→M18) governs *Go import order*, not *research order*. Two packages can be
import-independent (e.g. `internal/storage` only imports `internal/config`, per IC §9) while one's
on-disk *parameters* are still downstream of the other's decisions. `internal/storage`'s vLog entry
size (262,212 bytes) and fixed chunk size (262,144 bytes) are consequences of `internal/erasure`'s
shard-size decision, even though the packages don't import each other. This list orders topics by
that parameter dependency, not the import graph — where the two disagree, parameter dependency wins.

Each row is tagged: **[GAP]** — no prior paper touches this at all. **[WEAK]** — an Accepted ADR
with a documented open constraint. **[V3]** — one of the four active V3 functionality proposals.

---

## Tier 1 — Crypto stack (`internal/crypto`, #9/#10/#15)

No upstream dependency. Reviewed, not re-opened as a literature target this pass — ADR-019, 020,
022 are Accepted with no unresolved literature question, only empirical/spec work:

| Item | Type | Source | Action needed |
| --- | --- | --- | --- |
| Provider-side keystore KDF | spec gap, not lit | ADR-058 (Proposed) | Write the daemon-side analogue of ADR-020's owner KDF. Internal consistency work, not a search target. |
| Argon2id (t,m,p) hardware benchmarking | empirical, not lit | ADR-020 | Needs a real min-spec Indian desktop, not papers. |

---

## Tier 2 — Erasure Coding (`internal/erasure`, #3) — RECOMMENDED STARTING POINT

Depends on Tier 1 (AONT wrapping decisions in ADR-022 feed what erasure coding actually operates on).

| # | Topic | Tag | ADR | Why |
| --- | --- | --- | --- | --- |
| 1 | Hot/Cold storage band erasure parameters | **[GAP][V3]** | ADR-018 | Hot band explicitly deferred to V3; ADR-018's own open constraint says this "must be researched separately in Phase 2A before Hot band launch." Zero papers cover it today. |
| 2 | Upload straggler mitigation / optimality parameter | **[WEAK]** | ADR-003 | Storj's `o`-parameter cancel-slowest-uploads mechanism is explicitly flagged "not specified for V2; add as a V3 enhancement." |

---

## Tier 3 — Storage Engine (`internal/storage`, #16) — "Wisckey"

Depends on Tier 2: vLog entry format and fixed chunk size are direct consequences of the erasure
shard-size decision. Researching this before Tier 2 risks redoing the work if Hot-band research
changes the shard size or `lf` constant.

| # | Topic | Tag | ADR | Why |
| --- | --- | --- | --- | --- |
| 3 | vLog garbage collection: fine/coarse-grained reclaim | **[WEAK]** | ADR-023 | Already substantiated this session — `build.md` Session 5.1.5's algorithm (full-vLog-scan copying GC) diverges from ADR-023's own prose (tail-based, `fallocate(PUNCH_HOLE)`). VGKV and ZoomDB abstracts already triaged as ACCEPT / WATCH. |
| 4 | RocksDB vs. BadgerDB engine unification | **[WEAK]** | ADR-046 | Q49-1 open: does Badger's advantage at 256 KB values on Windows justify retiring RocksDB on Linux/macOS too. Explicitly deferred to post-launch measurement, not blocking. |

---

## Tier 4 — P2P Network Layer (`internal/p2p`, #12/#21)

Depends on Tier 3 — there must be something to transfer before transfer protocol research pays off.

| # | Topic | Tag | ADR | Why |
| --- | --- | --- | --- | --- |
| 5 | NAT traversal real-world success rate | **[WEAK]** | ADR-043 | **This replaces "libp2p QUIC NAT traversal 2026."** That search failed because it targets ADR-021, which is Accepted — libp2p+QUIC is a settled engineering choice, not an open research question. The actual gap is ADR-043's: "the actual failure rate of transparent NAT traversal across real Indian ISP/router configurations is not yet measured." |
| 6 | QUIC transport empirical gaps | **[WEAK]** | ADR-021 | Relay overhead vs. audit deadline, UDP-block rate at Indian ISPs, 0-RTT reconnect penalty, connection-migration false-positive rate — four open constraints, all empirical-leaning but literature on CGNAT prevalence / UDP-blocking measurement studies would narrow the range before in-house measurement. |

---

## Tier 5 — Audit & Coordination (#2/#1)

Depends on Tier 4 — audit challenges travel over the P2P transport.

| # | Topic | Tag | ADR | Why |
| --- | --- | --- | --- | --- |
| 7 | Audit/PoR scalability past ~100k providers | **[WEAK][V3]** | ADR-002 | Postgres INSERT throughput ceiling (~5-10k rows/sec) hit before V3 target scale. Also Q41-1: should retrieval-refusal (service-denial) feed the reliability scorer — open, unresolved, cross-references Tier 6. |
| 8 | DHT proximity optimization at geographic scale | **[WEAK]** | ADR-001 | 24% lookup-latency reduction available (Rhea et al.) but not enabled — V2's India-only network is homogeneous enough not to need it yet. Low priority unless international expansion is on the table. |

---

## Tier 6 — Reliability Scoring (#8) + the correlated-failure cluster — HIGHEST LEVERAGE GAP

Depends on Tier 5 — scoring consumes audit outcomes.

| # | Topic | Tag | ADR | Why |
| --- | --- | --- | --- | --- |
| 9 | Distributed correlated-failure detection without a central coordinator | **[GAP]** | ADR-001, ADR-008, ADR-014, ADR-055 (all four) | The same open question (Q07-4, "deferred to Phase 2B") recurs across four separate ADRs and has never actually been researched. This is the single highest-leverage gap in the whole project — closing it unblocks four stalled constraints at once, including the V3 emergency-eject trigger design. |
| 10 | Adaptive polling interval from score history | **[GAP][V3]** | ADR-008 | Explicitly named a V3 enhancement, zero papers. |

---

## Tier 7 — Repair & Payment (#4/#17/#19, #13/#18)

Depends on Tier 6 — both consume reliability scores.

| # | Topic | Tag | ADR | Why |
| --- | --- | --- | --- | --- |
| 11 | Repair burst admission control at V3 scale | **[WEAK][V3]** | ADR-004, ADR-055 | Without it, a correlated failure event (ASN outage) produces a repair backlog that degrades audit performance. Directly feeds the V3 emergency-eject trigger (ADR-055) — shares the Tier 6 correlated-failure gap. |
| 12 | Sybil-farming cost bound for progressive earnings ramp | **[WEAK][V3]** | ADR-054 | Cost bound is asserted qualitatively ("bounded by phone/KYC identity cost") but never computed in currency terms. Must be computed before launch. |

---

## Tier 8 — Provider Daemon (#11, #10/#21 gap)

Depends on Tier 7 — the daemon runs repair and payment logic locally.

| # | Topic | Tag | ADR | Why |
|---|---|---|---|---|
| 13 | Background execution continuity across Windows/macOS/Linux | **[WEAK][V3]** | ADR-057 | Privilege requirements for `powercfg`/D-Bus inhibitor calls not checked against ADR-042's least-privilege design; no fallback specified for managed-device policy blocking (e.g. corporate laptop Group Policy). |

---

## Tier 9 — Vetting (#5)

Depends on Tier 8 — vetting duration logic runs inside the daemon.

| # | Topic | Tag | ADR | Why |
|---|---|---|---|---|
| 14 | Stealth vetting: post-graduation monitoring + Sybil exposure | **[WEAK][V3]** | ADR-053 | Monitoring policy for forced-ceiling providers undefined; Sybil-farming exposure shared with Tier 7 item 12, not independently closed. |

---

## Tier 10 — Cross-cutting: UX, desktop shell, environmental (#20/#22/#23/#24)

Parallel track, depends on REST API (M11) + Daemon (M13) APIs being stable, not on Tiers 1-9 directly.

| # | Topic | Tag | ADR | Why |
| --- | --- | --- | --- | --- |
| 15 | ISP data-plan sync — OS-level network accounting | **[WEAK][V3]** | ADR-056 | "Buildable now" internally; per-platform (Windows/macOS/Linux) network accounting method unspecified. |
| 16 | Defensible embodied-carbon figure for impact claims | **[GAP]** | ADR-044 | Q44-1: no quantitative version of the environmental-impact feature can ship without this. Genuinely uncovered — worth checking whether recent embodied-carbon / data-center-vs-P2P-storage energy literature exists. |
| — | Desktop shell (Wails v3 stabilization), accessibility conformance | low priority | ADR-038, ADR-050 | Mostly engineering verification (smoke tests, WebView2 accessibility-tree bridge), not literature gaps. Revisit only if Wails v3 stalls materially. |

---

## Summary: recommended sequence

**1 → 2 (Tier 2, Hot/Cold + upload straggler) → 3-4 (Tier 3, Wisckey GC) → 5-6 (Tier 4, NAT/QUIC) →
7-8 (Tier 5) → 9-10 (Tier 6, correlated-failure cluster) → 11-12 (Tier 7) → 13 (Tier 8) → 14 (Tier 9)
→ 15-16 (Tier 10).**
