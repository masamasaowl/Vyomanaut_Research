# Paper 05 — Storj: A Decentralised Cloud Storage Network Framework

**Authors:** Storj Labs, Inc. **Venue / Year:** Storj Labs Whitepaper v3.0 Oct 2018 / v3.1 Mar 2024 **Topics:** #1, #2, #3, #5, #8, #11, #13, #16, #17, #18 **ADRs produced:** none — all decisions from this paper are Accepted; see Decisions Influenced

---

## Problem Solved

Centralised cloud storage (Amazon S3, GCS) concentrates data in a handful of operators, creating single-points-of-failure, high per-GB costs, and an architecture misaligned with the distributed nature of the internet. Storj proposes a framework for a decentralised object store that encrypts data client-side, erasure-codes and distributes pieces across independently-operated nodes, and uses audits plus a reputation system to ensure storage nodes behave honestly without requiring trusted hardware or on-chain consensus. For Vyomanaut, the paper is the primary source for the four-parameter RS scheme, the four-subsystem reputation pipeline, the Berlekamp-Welch unlimited audit scheme, Neuman's proxy-based bandwidth accounting, and the held-earnings vetting mechanism — each of which maps directly to an accepted Vyomanaut ADR.

---

## Key Findings

**Four-parameter Reed-Solomon scheme — k, m, o, n (Section 4.7, Figure 4.4):**

Storj extends the standard RS(k,n) pair with two intermediate values:

- k — minimum pieces required for reconstruction (our s=16)
- m — minimum safe threshold: if available pieces fall below m, trigger repair immediately (our r0=8; Storj calls this r0 explicitly, citing Giroire et al.)
- o — optimal upload threshold: during upload/repair, cancel the slowest n−o uploads once o pieces are confirmed; durability is satisfied at o, not n
- n — total pieces generated, providing excess durability and long-tail cancellation headroom (our n=56)

The gap o−m is the buffer against temporary offline nodes without triggering repair. The gap m−k is the safety margin between repair trigger and reconstruction failure. n−o is the number of slow-node slots that can be cancelled during upload. This four-parameter framing clarifies why our r0=8 is calibrated differently from a simple "threshold" — it represents m−k=8 above the reconstruction floor, not a fraction of total pieces.

**Berlekamp-Welch unlimited audit scheme (Section 4.13):**

Storj's v2 used pre-generated Merkle challenges — a "limited" scheme in which a storage node can defeat audits by caching just the expected challenge-response pairs. v3 replaced this with a stripe-based unlimited scheme: the Satellite downloads the erasure shares of a randomly chosen stripe from all responsible nodes simultaneously, then runs the Berlekamp-Welch error correction algorithm across the responses. Any node returning a wrong share is identified. Because the audited stripe is selected at the time of audit (not pre-generated at upload), nodes cannot anticipate which stripe will be challenged. This is the same "unlimited proof of retrievability" design Vyomanaut uses: `SHA256(chunk_data || challenge_nonce)` where the nonce is microservice-generated at challenge time.

**Four-subsystem storage node reputation pipeline (Section 4.15):**

Storj's reputation system has four independent stages: (1) proof-of-work identity generation to limit Sybil attacks; (2) vetting — new nodes receive non-critical data under extra erasure redundancy until they have proved longevity over at least one payment period; (3) filtering — nodes that fail too many audits or have too much downtime are disqualified; (4) preference — surviving nodes are ranked by throughput, latency, uptime, and geographic diversity, with higher-ranked nodes receiving more new assignments via randomised selection (Power of Two Choices). Critically, preferential reputation only affects where new data is sent; existing data is never moved due to a node's preference score drop — moves require repair events. This is the exact pipeline adopted in ADR-005.

**Held-earnings vetting model (Section 3.7, Section 4.16):**

During the first nine months, Storj holds a portion of each storage node's earnings as an offset against the risk that the node departs with data still on it. If a storage node leaves prematurely, the Satellite reclaims the held payments to cover repair costs. This is the direct production precedent for Vyomanaut's escrow hold structure (ADR-024): earnings are withheld not as a pre-committed stake but as accumulated-but-unlocked collateral that is seized on departure.

**Neuman's proxy-based bandwidth accounting (Section 4.17):**

Rather than trusting self-reported transfer counts, Storj implements B. C. Neuman's restricted proxy protocol. The Satellite (account server) issues a signed bandwidth allocation to the Uplink (payer) for each operation. The Uplink incrementally extends this allocation to the storage node in small steps, signed and restricted to the bytes transferred so far. The storage node holds the largest restricted allocation it received and submits it for settlement. This creates a gradual fair-exchange protocol — neither party is exposed to more than a small delta of loss at any step. Vyomanaut's escrow-per-audit-passed (ADR-012) achieves the same fraud prevention through a different mechanism: the audit pass itself is the proof of service.

**Selected calculations (Section 7):**

Section 7.3 Table 7.2 provides the empirical decision table Vyomanaut used to calibrate initial erasure parameters. At MTTF=6 months with k=20, n=40, m=30: repair bandwidth ratio = 0.87, durability = 0.9999 (17 nines). At MTTF=12 months with the same k and n, ratio drops to 0.31. The paper also validates (via Section 7.2) that using Jeffrey's prior β(0.5,0.5), a storage node's estimated audit success probability exceeds 99% after just 80 consecutive successful audits. This informs the vetting period length in ADR-005.

**v3.1 change — Kademlia removed from node discovery (Section 4.6, Changelog):**

The March 2024 v3.1 update replaced Kademlia DHT node discovery with a direct node-to-Satellite check-in. Storage nodes now announce themselves directly to each Satellite in their trust list on join, and check in approximately hourly. The Satellite's per-Satellite node discovery cache is the sole source of truth for node addresses. This is a deliberate centralisation tradeoff for operational reliability. Vyomanaut retains Kademlia for chunk lookup (ADR-001) — the Satellite equivalent (coordination microservice) handles provider registration, but the DHT handles data-plane lookups. The lesson from Storj's removal is that pure Kademlia node discovery has operational fragility at scale without a fallback.

**Long-tail mitigation via over-encoding (Section 3.4.2, 4.7):**

The o parameter gives Storj over-encoding: n pieces are uploaded in parallel; as soon as o complete, the remaining n−o slowest uploads are cancelled. This eliminates wait time on the slowest nodes (analogous to MapReduce backup tasks). The same principle applies to downloads: any k of n pieces suffice, so the k fastest responses are used, cancelling the rest. This reduces P99 upload and download latency dramatically compared to waiting for all n.

**Hot file content delivery scaling (Section 6.1):**

A RS(k,n) encoding has the property that the first n pieces of a RS(k,n+x) encoding are identical to the original n pieces. Adding redundancy for a hot file requires only generating the additional x pieces — the original n storage nodes are unaffected. The Satellite can scale from (20,40) to (20,250) without touching existing data. This is the theoretical foundation for Vyomanaut's V3 Hot Storage Band (ADR-018).

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Satellite central coordinator (v3.0) | Kademlia DHT (v2) | Dropped decentralisation for operational reliability; faster node discovery; single point of failure for metadata and audits |
| Kademlia fully removed (v3.1) | Retained as fallback | Clean architecture; eliminates DHT maintenance complexity; providers must check in explicitly or go unrouted |
| Berlekamp-Welch unlimited audit over stripe | Pre-generated Merkle challenges (v2) | Eliminates challenge caching attack; requires downloading all n erasure shares per stripe per audit — higher audit bandwidth than a single-chunk Merkle proof |
| Not paying for ingress bandwidth | Paying per byte stored + egress | Disincentivises delete-and-restore behaviour; ingress is a cost borne by the data owner's upload; aligns payment with long-term storage commitment |
| Held earnings (9-month vetting hold) | Pre-committed collateral | No capital barrier to entry; providers earn during vetting; held fraction is lost on premature departure |
| Four-parameter RS (k, m, o, n) | Two-parameter RS (k, n) | Long-tail performance gain from the o threshold; repair sensitivity configurable independently of durability and upload performance |
| Power of Two Choices for node preference selection | Deterministic top-scorer selection | Prevents runaway Matthew effect; any qualified node has non-zero selection probability |
| Stripe-level erasure (small sub-pieces) | Segment-level erasure | Streaming and partial decoding without staging the full segment; audit bandwidth proportional to stripe size, not segment size |

---

## Breaks in Our Case

- **Storj uses a Satellite (centralised coordination) with no Kademlia after v3.1** ≠ **Vyomanaut uses a hybrid microservice + Kademlia DHT for chunk lookup (ADR-001)** → Storj's centralisation choice confirms that pure P2P node discovery has operational costs. Vyomanaut retains Kademlia for data-plane chunk lookups (FIND_VALUE) but uses the microservice for provider registration and audit scheduling — a hybrid that captures the reliability benefit Storj sought without fully abandoning decentralised lookup.
- **Storj's Berlekamp-Welch audit downloads all n erasure shares of a stripe simultaneously** ≠ **Vyomanaut's audit challenge is a single point lookup: SHA256(chunk_data || challenge_nonce) from one provider** → Storj's scheme catches data degradation across the full erasure group in one audit event; Vyomanaut audits one provider at a time. Vyomanaut's approach is lower bandwidth per audit event but requires multiple independent audit rounds to detect multi-provider degradation. The Berlekamp-Welch approach is not adopted because it requires contacting all 56 providers per stripe — disproportionate overhead for a write-once cold storage workload.
- **Storj assumes MTTF of 6–12 months for NAS operators** ≠ **Vyomanaut targets desktop providers with MTTF 180–380 days (6–12.6 months) — compatible range** → The overlap is intentional. Storj's Table 7.2 parameters at MTTF=6 months directly informed the starting calibration for ADR-003. However, Storj's table uses k=20 while Vyomanaut uses s=16, requiring Giroire's full formula (Paper 10) to rederive r and r0 at our specific k.
- **Storj's o parameter enables aggressive upload cancellation at the 52nd-of-80 piece level** ≠ **Vyomanaut does not specify an upload optimality threshold o separately from n** → The long-tail upload mitigation the o parameter provides is architecturally valuable but deferred for V2. Our upload path currently targets all 56 providers and considers the upload complete when all 56 confirm. Introducing an o threshold (e.g. o=52) would reduce P99 upload latency but requires the microservice to know when enough pieces have landed and issue cancel signals to the remaining providers.
- **Storj pays storage nodes for egress bandwidth in addition to data at rest** ≠ **Vyomanaut pays per audit pass (ADR-012), decoupled from retrieval** → Storj's per-egress payment ties revenue directly to data retrieval demand, rewarding popular data more. Vyomanaut's per-audit payment rewards continuous availability regardless of retrieval frequency — correct for cold archive storage where data may go months without retrieval. The difference in payment model was deliberate: Swarm's per-bandwidth payment failed in production (SoK, Paper 07), motivating our audit-pass design.
- **Storj's vetting period is 9 months with held earnings** ≠ **Vyomanaut's vetting period is 4–6 months (ADR-005)**→ Storj's 9-month hold is calibrated for their payment period and NAS-operator MTTF. Vyomanaut's shorter 4–6 month window reflects the same principle but calibrated to Indian desktop-provider onboarding friction: a 9-month hold before full assignment would deter participation in the target market. The held-earnings fraction during vetting (50% per ADR-024) compensates for the shorter window.
- **Storj's garbage collection uses per-Satellite Bloom filters sent to storage nodes** ≠ **Vyomanaut's WiscKey storage engine uses GC on the vLog triggered by explicit deletion events (ADR-023)** → Storj's approach handles the case where delete messages are missed by offline nodes; nodes poll for Bloom filters to find what is no longer tracked. Vyomanaut's write-once cold storage model produces far fewer deletions — GC is a rare maintenance sweep, not a continuous background process. The Bloom filter GC model is not adopted; a simpler append-only vLog with tail-scan recovery (ADR-023) is sufficient.

---

## Decisions Influenced

- [**ADR-003](https://www.notion.so/decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** `CONFIRMED` The four-parameter RS scheme (k, m, o, n) from Section 4.7 is the production model that Giroire's formulas (Paper 10) formalise. Storj's starting values (k=29, n=80, m=35, o=52 in earlier versions; Table 7.2 at k=20, n=40, m=30 for MTTF=6mo) provided the initial calibration reference from which Vyomanaut derived s=16, r=40, r0=8 via Giroire's full LossRate formula. The paper confirms that Reed-Solomon is the correct code for production-scale decentralised storage, that systematic RS is used universally, and that the lazy repair threshold (m in Storj's notation) is distinct from the reconstruction floor (k). Parameters s=16, r=40, r0=8, lf=256 KB remain unchanged. *Because:* Section 7.3.4 concludes that n should be as large as network conditions allow to minimise repair bandwidth ratio while maintaining durability — exactly the reasoning behind Vyomanaut's wide-stripe n=56.
- [**ADR-002](https://www.notion.so/decisions/ADR-002-proof-of-storage.md) [#2 Proof of Storage]** `CONFIRMED` The v3 shift from limited Merkle challenges to unlimited stripe-based Berlekamp-Welch auditing (Section 4.13) confirms that pre-generated challenges are exploitable by rational storage nodes. Vyomanaut's microservice-generated nonce in `SHA256(chunk_data || challenge_nonce)` is the minimal unlimited scheme — the challenge cannot be precomputed because the nonce is not known before the challenge is issued. The paper independently validates the design principle behind ADR-014 Defence 3 (randomised challenge timing).*Because:* Section 4.13 explains exactly how a storage node can defeat limited-scheme audits by caching challenge-response pairs, and confirms the shift to per-challenge-time nonce generation as the fix.
- [**ADR-005](https://www.notion.so/decisions/ADR-005-peer-selection.md) [#5 Peer Selection]** `CONFIRMED` The four-subsystem reputation pipeline in Section 4.15 — identity gate, vetting, filtering, preference — is adopted verbatim as Vyomanaut's provider selection design. The Power of Two Choices for preference selection prevents runaway concentration. The 80-audit threshold from Section 7.2 (Jeffrey's prior reaching 99% confidence) informs the vetting period length. The explicit statement that preference scores only affect new assignments (not existing data) is the correct principle: preferential scoring cannot cause involuntary data movement. *Because:* Section 4.15 provides the production-validated pipeline that Vyomanaut's ADR-005 implements, including the bootstrapping mechanism for new nodes (optimistic random assignment during vetting with extra erasure redundancy).
- [**ADR-010](https://www.notion.so/decisions/ADR-010-desktop-only-v2.md) [#5 No Mobile V2]** `CONFIRMED` Section 2.5 and the churn analysis in Section 7.3 confirm that MTTF of "several months" is the minimum for acceptable network durability. Mobile providers with MTTF ~1–2 months (Blake & Rodrigues, Paper 06) would require dramatically higher r and n, increasing repair bandwidth to unsustainable levels for all other providers. Storj targets NAS and desktop operators specifically for this reason.*Because:* Table 7.2 shows that at MTTF=1 month, k=20, n=40, m=35: repair bandwidth ratio reaches 9.36 — nine times the steady-state data volume per month. At MTTF=6 months the same parameters give 0.87. Mobile inclusion at 1-month MTTF is not economically viable.
- [**ADR-011](https://www.notion.so/decisions/ADR-011-escrow-payments.md) [#13 Fiat Escrow]** `CONFIRMED` Storj's Neuman proxy-based bandwidth accounting (Section 4.17) is the production precedent for metering resource usage without trusting self-reports. Vyomanaut's audit-pass payment achieves the same fraud resistance through a stronger mechanism: the provider cannot compute a valid PoR response without the actual data. The paper's held-earnings vetting model (Section 4.16, first 9 months) directly validates Vyomanaut's escrow hold design in ADR-024. However, Storj uses STORJ token payments; Vyomanaut replaces this with Razorpay/UPI fiat to eliminate crypto friction for Indian providers. *Because:* Section 4.16 explicitly states that storage nodes "will not be paid for the initial transfer of data" — incentivising retention over churn, and that the Satellite "reclaims held payments" on premature departure — the direct model for Vyomanaut's escrow seizure on silent departure.
- [**ADR-012](https://www.notion.so/decisions/ADR-012-payment-basis.md) [#13 Payment per Audit Passed]** `CONFIRMED` Storj pays per-GB-stored-per-month plus per-GB-egress, not per ingress. The explicit exclusion of ingress payment prevents the delete-and-restore attack (Section 4.3: "storage nodes are not paid for the initial transfer of data"). Vyomanaut's per-audit payment is equivalent to per-storage-presence payment, decoupled from retrieval. The paper confirms that tying payment to continuous presence (storage at rest) rather than to transfer events is the correct model for a cold archive. *Because:* Section 4.16 documents the exact failure mode Storj's previous version suffered — paying for ingress led to nodes deleting data to be paid for re-storing it — motivating the ingress exclusion.
- [**ADR-018](https://www.notion.so/decisions/ADR-018-hot-cold-storage-bands.md) [#5 Hot/Cold Storage Bands]** `CONFIRMED` Section 6.1 describes the hot file scaling mechanism: because RS(k,n) and RS(k,n+x) share the same first n pieces, redundancy can be scaled up for popular content without modifying existing storage nodes. This is the theoretical basis for Vyomanaut's V3 Hot Storage Band, where files requiring CDN-like performance can be over-replicated on demand. The paper explicitly identifies this as a planned enhancement for "CDN" use cases. *Because:* Section 6.1's RS monotonicity property proves that hot-band scaling requires only generating additional pieces, not re-encoding existing data — operationally feasible at V3 scale.

---

## Disagreements

- **Storj's "Handling Churn in a DHT" (Rhea et al., Paper 28, cited as [7]):** The paper cites Rhea et al. showing median session time in P2P systems ranging from hours to minutes as justification for high churn expectations. Vyomanaut's financially-incentivised providers are expected to maintain MTTF 180–380 days — orders of magnitude above Rhea's unincentivised baseline. The churn data cited by Storj represents the unincentivised floor, not the target for contracted storage.
- **Storj v3.1 removed Kademlia entirely:** The whitepaper notes this in the changelog as a simplification. In our case, Kademlia DHT serves a different function — chunk-address lookup (FIND_VALUE), not provider-address lookup. Storj removed DHT because they found that Satellite node-discovery caches were operationally simpler and more reliable. Vyomanaut retains Kademlia for data-plane lookups where decentralisation is architecturally important, while using the microservice for control-plane registration.
- **Satellite as the trust anchor:** Storj's Satellite stores file metadata, pays providers, audits, and manages accounts for users who trust it. Users who distrust a Satellite operator cannot independently verify their data's integrity — the paper acknowledges this limitation and defers public verifiability to future work (Section 6.2). Vyomanaut's V3 Transparent Merkle Log (ADR-015) addresses the same gap through a different mechanism: a public Merkle tree over all audit receipts, verifiable without trusting the microservice operator.

---

## Open Questions

See [open-questions.md](https://www.notion.so/open-questions.md) — questions Q05-1 through Q05-8.