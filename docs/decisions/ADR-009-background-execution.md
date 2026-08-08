# ADR-009 — Background Execution: Desktop-Only, ≤5% CPU Budget

**Status:** Accepted — **CPU budget downgraded to an open constraint pending measurement**
**Topic:** #11 Background OS Execution
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 09, **62, 65**

> **Revision note (Papers 62, 65) — F-27.** The ≤5% budget was asserted from a single source
> (Bolosky 2000) which measured *ambient* corporate-desktop load rather than Vyomanaut's workload.
> Papers 62 and 65 supply the first erasure-coding throughput data in the corpus. Three changes:
> (1) the budget is **decomposed by role**, because the three costs land on three different machines
> at three different rates and one document was covering all of them; (2) the audit path is shown by
> direct calculation to be **cheap in CPU and expensive in disk**, which confirms F-40's framing and
> removes a worry that was never the real one; (3) the one genuinely unbudgeted item — repair-time
> decode and re-encode at `(16,40)` — is named, and the only available benchmark data points the
> wrong way.

---

## Context

Storage providers must run the Vyomanaut daemon in the background at all times. On mobile (iOS/Android), OS-enforced background execution limits severely restrict what background processes can do — they can be killed without warning, limiting both storage reliability and upload bandwidth. Desktop OSes have no such restrictions.

## Decision

V2 is desktop-only. No mobile providers in V2. Mobile is deferred to V3 after studying desktop V2 performance (see [ADR-010](ADR-010-desktop-only-v2.md)).

For the desktop daemon:

- CPU budget: ≤ 5–10% of CPU for background audits and transfers
- Disk I/O budget: operate within normal desktop I/O load
- The daemon must not degrade user experience; it runs at below-normal process priority

The 100 KB/s background upload bandwidth assumption (Blake & Rodrigues ([paper-06](../research/paper-06-blake-rodrigues.md))) is taken as consistent with Indian consumer broadband. *(F-25 remains open: consumer FTTH at this price point is typically **asymmetric**, and upload is the binding constraint for a provider. Also note the unresolved eightfold unit inconsistency across ADRs — 100 KB/s vs 100 Kbps — flagged in the interrogation and not settled here.)*

### The budget, decomposed by role

The single figure above was covering four distinct workloads on three distinct machines. Separating them is the substantive change in this revision.

| Workload | Runs on | Rate | Cost | Status |
| --- | --- | --- | --- | --- |
| RS(16,56) **encode** | data owner's client | once per uploaded byte | unmeasured | **Not a background budget item at all** — it is user-visible upload latency, and no NFR covers it |
| **SHA-256** over stored chunks | provider daemon | 70 GB/day (daily full audit, conservative tier) | 46.7 s/day with SHA-NI (**0.054%** of a core); 467 s/day scalar (**0.54%**) | ✅ comfortably inside budget |
| **ChaCha20-Poly1305** | provider daemon | proportional to transfer traffic | bounded by the 100 Kbps background budget | ✅ negligible |
| Wails / WebView2 GUI process | provider desktop | while the window is open | unmeasured | ⚠ not a daemon cost; only present when the user has the UI open |
| RS(16,56) **decode + re-encode** | repairing provider | 32 fragments reconstructed per repair event (ADR-004, `r0=8`) | **unmeasured, unbudgeted** | ⚠ **the real gap** |

**The audit path was never the problem.** Hashing 70 GB/day costs at most half a percent of one core even without SHA extensions. What the daily full audit costs a provider is **disk**: ~286,720 random seeks per day, roughly an hour of continuous drive activity on a 7200 RPM HDD (F-40). ADR-009 budgets CPU and nothing budgets audit disk I/O. **The ≤5% CPU figure gives false assurance about the audit path — it is the wrong resource.** Domain B is where this is fixed; ADR-009's job is to stop implying the question is answered.

### What the throughput literature actually supports

[Paper 65](../research/paper-65-bilp-bx-xor-scheduling.md) benchmarks Jerasure's Cauchy Reed–Solomon single-threaded, without SSE or AVX, over GF(2⁸), on an Intel i5-13600KF:

| `(k, m)` | Jerasure throughput (MB/s, derived from the paper's figures) |
| --- | --- |
| (4, 2) | 2,412 |
| (8, 4) | 376 |
| **(10, 4)** | **258** |

Throughput falls superlinearly in the coding work `k·m` — a log-log fit over all twelve tested configurations gives `T ≈ 40,335 · (k·m)^(−1.357)`, R² = 0.985 *within the tested range*.

[Paper 62](../research/paper-62-less-io-efficient-repair.md) adds a second data point from a different direction: plain RS at `(14,10)` reaches 2.8 GiB/s single-threaded with a modern optimised library, and sub-packetised LESS at α=4 reaches 1.6 GiB/s — a 43% cut for `α = 4`.

**Vyomanaut's `k·m = 640` is 16× the largest configuration in Paper 65's set.** Extrapolating the fit yields ~6 MB/s, and **that number must not be used** — a power-law extrapolated 16× beyond its support, from a paper whose own instruction-count and throughput axes disagree by 20×, is not an estimate. What the two papers jointly support is narrower and sufficient:

1. Absolute throughput of a production XOR-based RS library on commodity hardware without SIMD is in the hundreds of MB/s at datacentre parameters.
2. It falls faster than linearly in `k·m`.
3. Our `k·m` is 16× anything measured.

**Therefore: the ≤5% CPU budget has never been evaluated against the actual codec, and the direction of the only available evidence is unfavourable.** It is downgraded from an asserted property to an open constraint with a named measurement task.

## Consequences

**Positive:**

- Desktop daemons have no background execution limits — reliable uptime
- The audit path's CPU cost is now calculated, not assumed, and it is comfortably within budget
- The GUI process is correctly separated from the daemon budget

**Negative / trade-offs:**

- Excludes the large population of potential mobile providers until V3
- Daemon must be auto-started on OS boot — requires OS-level integration per platform *(see [ADR-047](ADR-047-windows-autostart-mechanism.md); F-72 notes the logon-trigger mechanism means the daemon does not run before login, which is a duty-cycle problem, not a CPU one)*
- **The ≤5% figure is no longer claimed as demonstrated.** It is a target.

**Open constraints:**

- iOS/Android background execution limits must be researched before mobile launch (Q09-5)
- **NEW (F-27, Q65-1) — measure the actual codec.** Required before the ≤5% claim can be restored:
  1. RS(16,56) encode throughput, Vyomanaut's own Go codec, on min-spec Indian desktop hardware, at `lf = 256 KB`, cold cache.
  2. RS(16,56) decode-plus-re-encode throughput for a 32-fragment repair event — the ADR-004 case.
  3. Wall-clock and CPU-share for one full repair event on a provider that is simultaneously serving audits.
  Only (2) and (3) bear on this ADR's budget; (1) belongs to an upload-latency NFR that does not yet exist.
- **NEW — no NFR covers audit disk I/O.** ADR-009 budgets CPU; F-40's ~1 h/day of seeking has no owner. → Domain B, ADR-023.
- **NEW — the codec choice is not fixed.** If ADR-026's council session adopts LESS, encode throughput falls ~43% (Paper 62) and the field widens to GF(2¹⁶). Any measurement taken now must be re-taken then.

## References

- [Paper 65 — BiLP-BX](../research/paper-65-bilp-bx-xor-scheduling.md): first erasure-coding throughput measurement in the corpus — 258 MB/s at `(10,4)`, single-threaded, no SIMD, on a current mid-range desktop; superlinear falloff in `k·m`; our `k·m` is 16× the tested maximum
- [Paper 62 — LESS](../research/paper-62-less-io-efficient-repair.md): 2.8 GiB/s for optimised RS at `(14,10)`; 43% encode-throughput cost of sub-packetisation at α=4
- [Paper 09 — Bolosky](../research/paper-09-bolosky-feasibility.md): median CPU 1–2% — **ambient corporate-desktop load in 2000, not Vyomanaut's workload.** Retained as the origin of the 5% figure and explicitly not as evidence for it *(F-27, and Domain D on the staleness of the underlying population)*
- [ADR-004](ADR-004-repair-protocol.md): `r0 = 8` — the source of the 32-fragment repair burst that is the unbudgeted item here
- [ADR-023](ADR-023-provider-storage-engine.md): audit read path; where the disk-I/O budget belongs
- [ADR-026](ADR-026-repair-bw-optimisation.md): a V3 code-family change would invalidate any codec measurement taken before it
- [ADR-010](ADR-010-desktop-only-v2.md): desktop-only decision
