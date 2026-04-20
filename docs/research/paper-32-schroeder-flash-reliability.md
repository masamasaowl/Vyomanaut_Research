# Paper 32 — Flash Reliability in Production: The Expected and the Unexpected

**Authors:** Bianca Schroeder — University of Toronto; Raghav Lagisetty, Arif Merchant — Google Inc.
**Venue / Year:** USENIX FAST 2016
**Topics:** #16
**ADRs produced:** none — ADR-023 gains concrete scrubbing and failure-rate constraints; Q25-1 and Q25-2 answered; see Decisions Influenced

---

## Problem Solved

Flash reliability was understood almost entirely through controlled lab experiments on a handful of chips under synthetic workloads. Manufacturers' datasheets offered only vague guarantees (PE cycle limits), and the standard metrics — RBER and UBER — were assumed to predict user-visible failures without empirical validation at scale. This paper provides the first large-scale field study of SSD reliability across millions of drive days, ten drive models, three flash technologies (MLC, eMLC, SLC), and 6 years of production use at Google. It finds that multiple core assumptions are wrong: RBER does not predict uncorrectable errors, UBER is meaningless, and error growth is linear not exponential. For Vyomanaut, the paper answers Q25-1 (whether proactive scrubbing is justified) and Q25-2 (whether sparse vLog placement is necessary) with concrete production-scale failure rate numbers.

---

## Key Findings

**Uncorrectable errors are the dominant non-transparent failure mode (Section 3.1):**
Between 20–63% of drives experience at least one uncorrectable error in their first four years in the field. Between 2–6 out of every 1,000 drive days are affected. Final read errors — read operations that fail even after retries, almost exclusively due to bit corruption beyond ECC capacity — account for nearly all of these, and are two orders of magnitude more frequent than any other non-transparent error type.

**Correctable errors are ubiquitous and not a concern (Section 3.2):**
Between 61–90% of drive days experience at least one correctable error. These are masked by the drive's ECC and have no user-visible impact. They are irrelevant as a reliability signal.

**RBER does not predict uncorrectable errors (Section 5.2):**
There is no correlation between a drive's raw bit error rate and its incidence of uncorrectable errors — neither across drive models, nor across individual drives within the same model, nor across drive months. The Spearman correlation coefficient between RBER and UEs is below 0.02 across all models. The implication is that the failure mechanisms producing correctable bit errors are structurally different from those producing uncorrectable errors. RBER is useful for comparing drives in controlled environments but provides no useful signal for predicting the failures that matter in production.

**UBER is not a meaningful field metric (Section 5.1):**
UBER (uncorrectable bit error rate) normalises uncorrectable errors by the number of bits read. The paper shows there is zero correlation between UE count and read operations — UEs are independent of read workload. Normalising by reads therefore artificially inflates UBER for low-read drives and deflates it for high-read drives. UBER is meaningless in the field.

**Error growth is linear, not exponential (Sections 4.2.1, 5.3):**
Both RBER and uncorrectable error probability grow with PE cycles, but the growth follows a linear rate rather than the exponential rate widely assumed from lab tests. There is no sudden spike after the vendor's PE cycle limit is exceeded — drives continue operating smoothly well past their rated end-of-life.

**Age has an independent effect on RBER beyond wear-out (Section 4.2.2):**
Even after controlling for PE cycles, drives that have been in the field longer show significantly higher RBER (Spearman correlation coefficient 0.2–0.4). Silicon aging mechanisms independent of program-erase cycles are active in the field.

**Prior errors are highly predictive of future uncorrectable errors (Section 5.6):**
A drive month immediately following an uncorrectable error has a ~30% chance of another uncorrectable error, versus a ~2% baseline probability in a random month. Prior erase errors, meta errors, and final write errors also increase future UE probability by more than 5×. This creates a clear signal for proactive action when errors are detected.

**Bad block clustering is severe (Section 6.1.1):**
Between 30–80% of drives develop bad blocks in the field. The median number is only 2–4, but the distribution has a heavy tail: once a drive develops more than 2 bad blocks, the median total bad block count jumps to ~200. This is consistent with chip failure — a single failing chip can produce hundreds of bad blocks.

**SLC drives are not more reliable than MLC drives in practice (Section 7):**
SLC drives have dramatically lower RBER than MLC drives. But their repair rates, replacement rates, and incidence of non-transparent errors are not lower. The higher RBER of MLC drives does not translate to more user-visible failures. The premium paid for SLC for its RBER advantage is not justified by field reliability data.

**Flash drives have lower replacement rates than HDDs but higher uncorrectable error rates (Section 8):**
Annual HDD replacement rates are 2–9%; flash drives see only 4–10% replacement over 4 years. But more than 20% of flash drives develop uncorrectable errors in 4 years, compared to only 3.5% of HDDs developing bad sectors in 32 months.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Field study (millions of drive days) | Lab accelerated life tests | Real failure rates are higher than lab projections; linear not exponential growth; RBER is not predictive of UEs in the field |
| Per-model, per-drive time-series analysis | Aggregate fleet statistics | Reveals within-model variance is large; median-to-99th-percentile RBER spans an order of magnitude within a single model |
| Full error taxonomy (transparent + non-transparent) | UBER/RBER only | Exposes that RBER is decoupled from user-visible failures; final read errors are the dominant concern |
| Conservative PE cycle limits (set at ~1/3 of accelerated-test failure point) | Aggressive cycle limits | Drives operate smoothly past limits; no sharp degradation; vendors are very conservative |

---

## Breaks in Our Case

- **Paper studies enterprise-grade PCIe SSDs with custom firmware and ECC at Google's data center scale** ≠ **Vyomanaut providers run consumer-grade desktop hardware (SATA/NVMe SSDs, or HDDs) with commodity firmware**
→ The failure rates in this paper are measured on drives under sustained datacenter workloads and with sophisticated on-device ECC. Consumer SSDs have similar flash chips but less aggressive firmware ECC. The paper's 20–63% UE incidence over 4 years should be treated as an approximate lower bound for consumer hardware, not an upper bound. For Vyomanaut's HDD-bearing providers, the IRON paper (Paper 25) provides the more directly applicable latent sector error model.
- **Paper's drives are write-heavy datacenter workloads with hundreds to nearly 1,000 average PE cycles** ≠ **Vyomanaut providers run a write-once cold storage workload where PE cycle accumulation is extremely slow**
→ Vyomanaut's chunk storage workload writes a chunk once and reads it infrequently for audit challenges. PE cycle accumulation on a provider's SSD will be dominated by the OS, other applications, and WiscKey compaction — not by Vyomanaut chunk writes. The provider SSD is unlikely to accumulate more than a few hundred PE cycles in its Vyomanaut-relevant lifetime. The paper's wear-out data is largely irrelevant for our use case; age-based degradation (Section 4.2.2) and the time-dependent UE incidence rate are the relevant findings.
- **Paper finds RBER does not predict uncorrectable errors** ≠ **ADR-023 relies on the PoR challenge as the end-to-end detection mechanism for per-chunk corruption**
→ This finding reinforces ADR-023's design: monitoring RBER (even if we could — consumer drives don't typically expose it) would not give advance warning of chunk corruption. The PoR challenge response hash is the correct and only reliable detection mechanism. No additional monitoring signal improves on it.
- **Paper's bad block clustering shows that 2+ bad blocks predicts 200+ bad blocks (chip failure)** ≠ **Vyomanaut's audit system detects individual chunk failures, not drive-level bad block counts**
→ Vyomanaut does not have visibility into a provider's bad block count. The audit system will detect individual chunk corruption as FAIL results. The clustering finding implies that if a provider starts failing audits on multiple chunks simultaneously, this is a strong signal of impending chip failure — the provider should be treated as in rapid decline and their chunks should be aggressively re-replicated. This is a signal design issue for the reliability scorer (ADR-008), not a storage engine issue.
- **Paper finds prior errors are highly predictive of future UEs (~30% probability of UE in month following a UE)** ≠ **ADR-008's three-window rolling score treats audit failures as independent events**
→ The paper's finding that prior errors are strongly predictive of future errors is directly applicable to Vyomanaut's reliability scoring. A provider's audit FAIL events should not be treated as independent draws from a binomial distribution — a run of recent failures should trigger an accelerated scoring revision and possibly early repair. The current ADR-008 design (three rolling windows, last 24h weighted highest) partially captures this but does not explicitly model the elevated-failure-probability regime after a cluster of failures.

---

## Decisions Influenced

- **ADR-023 [#16 Provider-Side Storage Engine]** `CONFIRMED — SCRUBBING DESIGN CLARIFIED`
The `content_hash` per vLog entry (verified on every read before computing the PoR response) is the correct and sufficient per-chunk integrity mechanism. The paper confirms that monitoring RBER would add no predictive value for uncorrectable errors. ADR-023's design — detect corruption when a chunk is read for an audit challenge, report immediately as `audit_result = FAIL` — handles all the failure modes the paper describes (final read errors, bad blocks, chip failures) correctly. An uncorrectable error on a chunk read produces a FAIL audit result, triggering repair via ADR-004.
For Q25-1 (proactive scrubbing): the paper's finding that prior errors are highly predictive of future uncorrectable errors (30% probability after a recent UE) provides a concrete signal for when proactive scrubbing is worthwhile. Continuous background scrubbing is justified only after a provider has recently failed an audit. In the steady state — before any error signals — scheduled PoR audit challenges at the 24-hour polling interval are sufficient detection coverage. Full continuous disk scrubbing independent of audit failures adds I/O load without proportionate reliability benefit for a provider with no error history.
*Because:* Table 2 shows 2–6 UE-affected drive days per 1,000 drive days, so the base rate is low enough that reactive scrubbing (triggered by the first audit failure) is the right design. Continuous scrubbing has a cost that scales with provider disk size but a benefit that only materialises for the minority of drives with active error histories.
- **ADR-023 [#16 Provider-Side Storage Engine]** `Q25-2 ANSWERED — vLog PLACEMENT DEFERRED`
The paper's bad block spatial clustering finding (hundreds of bad blocks following 2–4 initial bad blocks, attributable to chip failure) motivates spreading chunk placement across non-adjacent disk regions. However, at the granularity relevant to Vyomanaut — individual 256 KB chunks — the key insight is that chip failure affects a localised disk region. Vyomanaut's vLog appends chunks sequentially by upload time; chunks from many different files are interleaved. A chip failure that corrupts a contiguous region of the vLog will corrupt shards from many different files. Since Vyomanaut holds only 1 of 56 shards from each file per provider, the RS(16,56) redundancy absorbs this: even if a chip failure corrupts 10 sequential vLog entries (10 chunks from 10 different files), each of those files retains 55 other shards across other providers. The within-provider burst failure is therefore contained within the distributed redundancy budget. Sparse vLog pre-allocation to separate chunks from different files adds engineering complexity without meaningful reliability improvement given the RS parameters. This defers Q25-2 to V3 pending empirical measurement — but the paper provides the theoretical justification that sequential vLog layout is acceptable.
*Because:* The heavy-tail bad block clustering (Figure 8) operates at chip granularity (~hundreds of blocks covering a contiguous region), not at file granularity. Vyomanaut's chunk-level RS redundancy is explicitly designed to absorb correlated within-provider failures at this scale.
- **ADR-008 [#8 Reliability Scoring]** `STRENGTHENED — FAILURE CLUSTERING SIGNAL`
The finding that a drive month with a prior uncorrectable error has a 30% probability of another UE (vs 2% baseline) supports adding a short-term failure-clustering detector to the reliability scorer. The current three-window design (24h highest weighted, 7d, 30d) already partially captures this — a cluster of recent failures dominates the 24h window. The paper provides the quantitative basis: a provider showing more than one audit FAIL within a 7-day window should be treated as being in an elevated-failure regime, not as experiencing isolated incidents. The scoring weights should be tuned to reflect this: the 30× failure probability increase after recent UEs means a cluster of recent audit FAILs is much more predictive of future FAILs than the scoring model's linear weighting might imply.
*Because:* Figure 7 directly quantifies the conditional probability of a UE given prior errors. The 15×–30× increase in UE probability after a recent UE is the largest single predictor available without hardware-level monitoring.

---

## Disagreements

- **Meza et al. (Facebook, SIGMETRICS 2015):** Observed significant infant mortality in flash drives; the Google paper finds no evidence of infant mortality within the PE cycle ranges studied.
*Implication for us:* The discrepancy is likely due to burn-in testing differences between companies. For consumer hardware (no burn-in), infant mortality is plausible. Vyomanaut's 4–6 month vetting period (ADR-005) naturally filters out infant-mortality providers before they receive full chunk assignments — exactly the mitigation the Facebook finding motivates.
- **Lab-based RBER predictions (Grupp et al., Cai et al.):** Report RBER values close to 1e-06 for MLC drives near their PE cycle limit. The paper finds field RBER around 3e-08 to 8e-08 for similar drives — 1–2 orders of magnitude lower.
*Implication for us:* Lab tests overestimate field failure rates. For Vyomanaut, this means the erasure parameters (s=16, r=40) calibrated from field-realistic failure assumptions (ADR-003) are conservative, not optimistic.
- **Common assumption that SLC drives are more reliable:** The paper directly refutes this for user-visible failures. SLC drives have better RBER but equivalent repair, replacement, and non-transparent error rates.
*Implication for us:* Provider hardware tier selection should not use flash type as a proxy for reliability. MTTF, audit pass rate, and response latency during vetting (ADR-005) are the correct signals.

---

## Open Questions

See open-questions.md — Q25-1 and Q25-2 are now answered. No new questions added.