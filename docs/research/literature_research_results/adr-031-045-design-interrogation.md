# Design Interrogation — ADR-031 to ADR-045

**Continues** [`adr-016-030-design-interrogation.md`](./adr-016-030-design-interrogation.md). Finding numbers continue from F-52.

| Route | Count |
| --- | --- |
| `[COUNCIL]` | 7 |
| `[RESEARCH]` | 3 |
| `[CORRECT]` | 11 |

**Headline:** this is a different kind of block. ADR-031/032/033/036 are hard engineering, and three of them are the best-tested decisions in the project — ADR-032 and ADR-033 each carry a numbered strength-test table run in production posture, and ADR-036 found a protocol that lets any peer delete arbitrary chunks. ADR-034/037/044/045 are careful about their own epistemic limits in a way earlier blocks were not.

The weaknesses are correspondingly different. Almost nothing here is a research gap — the ratio inverts to 11 corrections against 3 research items. What this block has instead is **drift between documents**, and one case where the implementation contract is right and three ADRs are wrong.

---

## Band A — Things that will produce wrong code

### F-53 `[CORRECT]` Segment size is wrong by 3.5× in three ADRs. Interface Contracts has it right.

ADR-022 heads its pipeline *"Encoding pipeline (per segment, max 14 MB = 56 × 256 KB)."* That arithmetic describes the **encoded output** — 56 shards of 256 KB. The **plaintext input** to the AONT is `s × lf = 16 × 262,144 = 4 MB`.

`interface-contracts.md` §5.1 states it correctly: *"segment is pre-padded by the caller to 4 MB minimum."*

The error propagates through three ADRs:

| Document | Says | Should say |
| --- | --- | --- |
| ADR-022 | "per segment, max 14 MB" | 4 MB plaintext → 14 MB encoded |
| ADR-019 | *"a 14 MB segment can be encoded within the ≤5% CPU budget"*; *"14 MB segment encodes in ~186 ms"* | 4 MB → ~53 ms at 75 MB/s |
| ADR-031 | "Max segment size \| 14 MB \| 1.25 MB" | 4 MB prod / 768 KB demo (both columns list the *encoded* size) |

**Why it matters beyond tidiness.** An implementer building the client SDK from ADR-022 splits files into 14 MB segments, RS(16,56)-encodes each, and gets shards of 875 KB — not 256 KB. That breaks the vLog entry size (262,212 bytes, ADR-023), the audit framing, the RocksDB `chunk_size` field, and every one of the constants ADR-031 lists under *"must never appear in NetworkProfile."* It is precisely the failure ADR-031's fixed-`ShardSize` rule exists to prevent, arriving through a different door.

The one consolation: IC is the document the build actually reads, and IC is correct. Fix the three ADRs to match it.

---

### F-54 `[COUNCIL]` NetworkProfile captures topology and timing. It omits almost every parameter its own ADRs flagged for tuning.

ADR-031 is a good piece of design — one struct, dependency-injected, `Mode` explicitly never used for branching, compiler-enforced tests that both profiles are fully specified. The stated goal is that switching to production is *"changing the active profile"* with *"no Go function changes."*

But the struct covers shard counts, thresholds, time windows and infrastructure flags. It does not cover **policy values** — and the policy values are the ones whose own ADRs say they are provisional:

| Parameter | Source | What its ADR says |
| --- | --- | --- |
| Scoring window **weights** | ADR-008 | *"a product decision… tuned empirically after launch"* (windows are in the profile; weights are not) |
| Release multiplier thresholds 0.95 / 0.80 / 0.65 | ADR-024 §3 | *"starting values… must be tuned empirically after V2 launch"* |
| `storage_rate_paise_per_GB_per_month` | ADR-024 §1 | *"a product pricing decision… must be set before any contract can be signed"* |
| JIT latency floor 0.3; flag counts 1/3 per 7d; 0.5× weight; 30d penalty | ADR-014 | no derivation given at all |
| Audit deadline ×1.5 multiplier | ADR-002 / ADR-014 | — |
| RocksDB `rate_limiter` (SSD 10 MB/s, HDD **TBD**) | ADR-023 | *"must be calibrated against actual provider hardware"* — the HDD value is explicitly unknown |
| Background-throttle p99 threshold 50 ms | ADR-025 | *"a starting value; must be tuned empirically"* |
| Client membership cache refresh 10 s | ADR-025 | — |
| Repair-ETA minimum sample size 20 | ADR-037 | *"arbitrary but non-trivial; tune post-launch"* |

Every one currently requires a code change and redeploy to tune. That is the inverse of what the profile struct is for. ADR-031's own inclusion criterion appears to have been *"does this differ between demo and prod"* rather than *"is this a value we expect to change"* — and the second is the one that determines whether tuning is a config edit or a release.

Worth noting the ADR gets the boundary right in the other direction: `ShardSize`, cipher identities, the nonce length, `ASNCapFraction`, and the single-writer vLog requirement are all correctly listed as never-variable.

---

### F-55 `[CORRECT]` The mode guard rails miss the two failure modes most likely to actually occur.

ADR-031's guards are: `prod` + env-var seed → fatal; `demo` + live Razorpay endpoint → fatal. Both are good.

Neither covers:

1. **Demo mode against the production database.** Phase 0 specifies *"two separate databases (demo_db, prod_db)"* but no guard enforces the pairing. A demo process with a prod `DATABASE_URL` writes demo-parameter rows — 2-minute audit periods, 5-pass vetting graduations — into the production ledger. Nothing stops it, and the append-only invariants (ADR-032) mean those rows can never be deleted, only detached with their partition (ADR-033).

2. **Mode is not on the wire.** `VYOMANAUT_MODE` is per-process. A provider daemon at `--mode=demo` can register with a production microservice; the microservice has no field telling it otherwise. Most consequential parameters are microservice-side so the blast radius is limited — but `HeartbeatInterval` is daemon-side, and demo is 30 s against prod's 4 h. That is a **480× rate increase** from a single misconfigured daemon, against an endpoint whose own ADR already lists heartbeat storms as an open constraint (ADR-028).

Both are cheap: assert a mode marker row in the database at migration time and check it at startup; add `mode` to the heartbeat payload and reject mismatches at registration.

---

### F-56 `[CORRECT]` ADR-034 specifies a 28-row table and then validates against 43 codes.

The Decision says *"one row per `ErrorCode` constant"*, describes *"186 `WriteError()` call sites spanning **28 distinct ErrorCode values**"*, and marks the artifact as a *"(28-row table)"*.

The Consequences say build-out is *"nontrivial copywriting work across **43 codes**"*, and Validation requires *"Every `error_code` actually returned by the live API (**43 today**)"* to have an entry or exercise the fallback.

Fifteen codes are unaccounted for. If 28 is the distinct count at `WriteError()` sites and 43 is the count of declared constants, both numbers can be true — but then the table is specified against the wrong one, because a code that is declared and returned anywhere needs copy.

**And the artifact has three homes in one ADR.** Decision §1: *"lives as `interface-contracts.md` §14."* Also §1: a companion file `interface-contracts-section-14.md`. Affected: *"New artifact: `docs/system-design/ux-copy.md`."* ADR-037 and ADR-045 both then cite "IC §14" as canonical, so IC §14 is presumably the intent — but as written, three documents claim to be the single source of truth for the single source of truth.

---

### F-57 `[CORRECT]` The UPI `tr` field is not unique per transaction, and the one float in the payment path is unflagged.

ADR-035 constructs:

```
upi://pay?pa={vpa}&pn=Vyomanaut&am={amount_rupees}&cu=INR&tr={contract_id}
```

**`tr` is the transaction reference** in the UPI deep-link grammar — expected unique per payment, used for reconciliation on both sides. `contract_id` is not unique per deposit: FR-006/FR-014 exist precisely because an owner tops up a *shortfall*, which implies repeat deposits against the same contract. Reusing `tr` across deposits breaks reconciliation and some PSP apps reject apparent duplicates. This wants a per-deposit identifier, which the project already has a discipline for (ADR-016's `idempotency_key`).

**`am={amount_rupees}`** is the only place in the entire payment path where an amount leaves integer paise. ADR-016 is emphatic: *"No floating-point anywhere in the payment path."* The UPI grammar requires decimal rupees to two places, so the conversion is unavoidable — but it must be an integer-formatting rule (`paise/100` and `paise%100` formatted separately), never a float division. Given the project treats this as a forbidden pattern everywhere else, the one legitimate exception should be called out explicitly in the ADR rather than left to the implementer.

Also: ADR-035's header reads `Superseded by: ADR-038 — context only… the intent_url decision… is not superseded`. That is a context correction, not a supersession, and it puts an **Accepted** ADR in a `Superseded by` relationship with a **Proposed** one. Make it a note.

---

## Band B — Conflicts and unexamined consequences

### F-58 `[COUNCIL]` ADR-033 defers exactly the capacity problem that F-02 says arrives at launch.

ADR-033 reasons carefully and lands on a proportionate answer: partition from day one (satisfying the day-one mandate), but ship only a `DEFAULT` partition and a manual maintenance function, because *"V2 launches at hundreds of providers — far below this limit"* and the alarming figures are *"the V3, 100k-provider projection."*

That framing comes from `capacity.md`, which projects from **1,000 providers × 10,000 chunks = 10M rows/day ≈ 116 INSERT/s**. Ten thousand chunks at 256 KB is **2.5 GB per provider** — the same unrealistically small provider that produced F-02.

Recomputed against the ~70 GB tier the implementation actually assigns:

| Scenario | Rows/day | INSERT/s | ×2 for two-phase |
| --- | --- | --- | --- |
| capacity.md's assumption (2.5 GB) | 10 M | 119 | 237 |
| 1,000 providers × 70 GB | 287 M | 3,319 | **6,637** |
| 1,000 providers × 200 GB | 819 M | 9,481 | **18,963** |

capacity.md's own analysis makes this worse, not better: it notes the ~5,000–10,000/s figure is *"derived from generic benchmark literature and not from the actual Vyomanaut schema,"* and that the real schema carries two 64-byte signatures, a 33-byte unique index, RLS policy evaluation, and a two-phase INSERT→UPDATE. So the true ceiling is likely **below** 5,000/s, against an effective load of 6,600/s at a thousand modest providers.

The decisions ADR-033 defers — `pg_partman` versus a manual function, DEFAULT-only versus pre-created monthly partitions, no scheduled maintenance job — are cheap now and expensive at volume. They should be re-taken against the corrected arithmetic, not against capacity.md's.

---

### F-59 `[CORRECT]` The nonce guard table is the bottleneck capacity.md named, shrunk by retention rather than removed.

ADR-033's `audit_receipt_nonces` table is a genuinely elegant solution: partitioning forces the unique key to include the partition column, which would silently weaken replay protection, so a small non-partitioned table preserves global uniqueness. Test #3 proves it works.

The ADR describes it as *"small"* on the strength of *"bounded retention — the migrator prunes rows past the window."* But the window is set by ADR-015 (re-submission accepted for **48 hours** after `server_ts`) and ADR-027 (24-hour rotation overlap). At the corrected audit volume:

| Retention | Rows @ 1,000 × 70 GB | B-tree index (~56 B/entry) |
| --- | --- | --- |
| 48 h | 573 M | **~30 GB** |
| 72 h | 860 M | **~45 GB** |

capacity.md flagged the global nonce index as *the* growth bottleneck. Retention bounds it, but 30 GB of hot index at a thousand providers is not "small," and every audit INSERT probes it. Worth stating the retention arithmetic explicitly in the ADR so the number is visible rather than asserted.

---

### F-60 `[COUNCIL]` ADR-037 declines to show an ETA that ADR-004 already claims to know.

ADR-037's reasoning is honest and mostly right: no completed-repair history exists pre-launch, and *"a confidently wrong ETA is arguably worse than showing none."* Refusing to fabricate a number is good practice.

But ADR-004 states, as a durability argument, that reconstruction *"completes in ≈ 8 h — within the 12-hour reconstruction window θ,"* derived analytically from Giroire's Qpeek. That is a numeric ETA, produced from a model, with no observed history.

So the same quantity is simultaneously:
- precise enough to underwrite the 12-hour safety window and the LossRate guarantee, and
- too speculative to display to a data owner whose file is degraded.

Both positions cannot hold. Either the analytical model is trustworthy — in which case ADR-037 should seed its estimate from it as a prior and refine with observation, rather than waiting for 20 completed jobs per tier — or it is not, in which case F-08's finding bites harder than recorded: ADR-004's window claim rests on a model the product does not trust enough to show anyone.

F-08 already showed the 8-hour figure does not reproduce under either bandwidth-unit reading (2.4 h or 18.9 h). ADR-037 is, without meaning to, evidence that nobody believes it.

---

### F-61 `[COUNCIL]` ADR-045's disclosure policy is self-defeating: the amounts reveal the thresholds.

ADR-045 draws a careful line — disclose the gross amount, the released amount, and a plain-language reason category; withhold *"the underlying formula, exact score, and exact thresholds."* The Rosenblat & Stark sourcing is apt and the reasoning about gaming is real.

But the disclosed set determines the withheld set:

- `released / gross` **is** the multiplier. A provider sees 0.75 or 0.50 exactly.
- ADR-024's ladder has four rungs. Two or three observed releases reveal the full multiplier set.
- The proposed reason-category copy — ADR-016's addendum offers *"₹X was withheld because your reliability score dropped below 95%"* — states a threshold verbatim.

After a handful of months, or one provider forum post, the ladder `{0.95→1.00, 0.80→0.75, 0.65→0.50, <0.65→0.00}` is public. The policy withholds the formula but not the thing a gaming provider would actually optimise against, which is the threshold ladder.

That may be acceptable — a public ladder is arguably *better*, since it lets an honest provider know what target to hit, and F-31 notes the scoring function has never had an adversarial analysis anyway. But it should be a decision rather than an accident. Either accept the ladder is public and design the scoring function to be robust to that, or disclose only direction ("your score fell") without amounts.

---

### F-62 `[COUNCIL]` The desktop app is the daemon, and a GUI user cannot see which mode they are in.

ADR-039 correctly rejects the Storj/Syncthing local-web-server pattern: in-process Wails bindings, no loopback port, no CSRF surface. Good call.

The consequence is that the GUI process *is* the daemon process. So `VYOMANAUT_MODE` is fixed at launch (ADR-031: read once, *"no API endpoint or runtime signal alters it"*), and the only mode indicator ADR-031 specifies is a startup log line:

```
[STARTUP] Vyomanaut daemon v0.1.0 — mode=DEMO — do not use for real data
```

A GUI user never sees stdout. For an application that moves real money and stores data the user believes is durable, "am I in demo mode" must be visible in the interface — and ADR-031's `demo` guard against the live Razorpay endpoint means a user in demo mode who thinks they are in production will find their deposits going nowhere.

Cheap fix, and it belongs in ADR-031 or ADR-039 before either app is built: a persistent, non-dismissible mode banner in demo, and the mode surfaced in an about/status pane always.

---

### F-63 `[CORRECT]` ADR-042's "no negatives identified" is already contradicted by two later ADRs.

ADR-042: *"**Negative:** none identified — no current provider-app requirement needs elevated privilege."*

- **ADR-057** (native sleep inhibition) needs `powercfg` / D-Bus inhibitor calls. The previous reading list already flagged that their privilege requirements were never checked against ADR-042's least-privilege design, with no fallback specified for Group-Policy-managed devices.
- **ADR-023** requires storage-device-type detection to select the RocksDB rate limiter. `/sys/block/{dev}/queue/rotational` is unprivileged on Linux; the Windows equivalent (`IOCTL_STORAGE_QUERY_PROPERTY`) is not uniformly so.

The decision is right and should stand — the Condor sourcing is one of the better single-paper arguments in the corpus, and ADR-047's per-user Task Scheduler resolution shows the constraint being honoured under pressure. But "none identified" should read "none identified at time of writing," with ADR-057 named as the open case, because a blanket claim invites the next feature to quietly assume it still holds.

---

### F-64 `[RESEARCH]` The environmental claim accounts for one side of the ledger.

ADR-044 is the best epistemic hygiene in the corpus. Refusing the popular "data centres are an exploding energy problem" framing because Paper 44's own finding contradicts it, and shipping qualitative language until a defensible number exists, is exactly right.

But the scoped claim — *"avoids the embodied carbon and material cost of manufacturing new dedicated storage hardware and building new purpose-built cooling infrastructure"* — is a gross figure, not a net one.

Vyomanaut increases utilisation of consumer drives that would otherwise sit mostly idle. Daily full audit alone is roughly **an hour a day of continuous random seeking** on a 70 GB HDD provider (F-40), plus sustained vLog writes. That accelerates drive wear and brings forward a replacement purchase — which has its own embodied carbon, on hardware manufactured in smaller, less efficient units than datacentre drives.

The honest accounting is *avoided datacentre hardware minus accelerated consumer hardware replacement*. That is exactly the calculation a journalist checking the claim would run, and ADR-044's whole purpose is to survive that check.

**This changes what R-16 should search for.** The reading list currently scopes it to "defensible embodied-carbon figure for impact claims." It should be scoped to both sides: embodied carbon per unit of storage hardware, *and* the relationship between duty cycle and drive lifetime for consumer-class drives. The second half also feeds F-40 and Domain O.

---

### F-65 `[RESEARCH]` Eight consecutive ADRs on one source each — and the tray decision is the thin one.

ADR-038 through ADR-045 each cite exactly one research source. Four of them (038, 039, 040, 041) cite the same one: *"Paper 45 — Wails and Electron desktop shell documentation and issue trackers"* — which is vendor documentation and issue trackers, not a study.

Most of these decisions are probably right and cheap to revisit. A Go backend taking a Go shell is close to self-evident; in-process bindings over a loopback port is a straightforward security win; signing installers before external distribution is table stakes.

**ADR-040 is the exception worth flagging.** `ux-decisions.md` §7 makes the tray *the Provider app's primary interface for routine use*. Wails v2 has no tray API. The chosen path is a community library in a goroutine alongside the webview, justified as *"proven in at least one shipped production app,"* with acknowledged *"thread-safety, no first-class support"* risk, and an exit condition — Wails v3 stabilising — that has no committed date, no owner, and no check interval anywhere in the corpus.

That is the primary interface of one of two shipping applications, resting on a workaround with a single anecdotal precedent and an unowned trigger.

**Reading-list action:** re-tag this cluster from *"mostly engineering verification, not literature gaps"* to *"decisions already made on single-source evidence."* The priority stays low — but low-priority-because-well-evidenced and low-priority-because-cheap-to-reverse are different claims, and only the second is true here.

---

### F-66 `[CORRECT]` ADR-036 fixes two protocols and names a pattern without inventorying the surface.

ADR-036 is the most valuable find in this block: `/vyomanaut/vetting-gc/1.0.0` had **no authorization field or check whatsoever**, and chunk IDs are DHT-discoverable, so any peer could instruct permanent deletion of arbitrary real chunks. The reasoning against ADR-030's original defence — that the 0-RTT prohibition protects against replay, not forgery, *"a different threat entirely"* — is exactly right.

The ADR states its ambition as *"a general authenticated-mutation pattern"* covering *"future mutating protocols too."* It then fixes two protocols and stops.

There is no inventory of the current libp2p sub-protocol surface with its authorization posture. `/vyomanaut/chunk-upload/1.0.0` is named in ADR-030 and appears nowhere in ADR-036 — can an arbitrary peer push chunks to a provider and consume their declared storage? That is resource exhaustion against a provider whose earnings are capped by capacity (ADR-030 §Storage cap). Perhaps it is authorised elsewhere in IC; the point is that ADR-036 is where a reader would look and the answer isn't there.

A one-page table — protocol, direction, side effect, authorisation, freshness — would close this. Audit task, not research.

**Live trap:** ADR-036's own renumbering note records ~15 stale "ADR-032" references across the M-OBS/13/14 review that must be updated *before that review drives a build session*. Flagged, not fixed.

---

### F-67 `[COUNCIL]` F-34's confidentiality gap is now structural in the profile, not a one-off.

The confidentiality threshold is `DataShards`; the per-group placement allowance is `floor(TotalShards × ASNCapFraction)`. Both are profile fields, so the ratio is reproduced by construction in every profile:

| Profile | Plaintext threshold | Shards per ASN | Colluding groups needed |
| --- | --- | --- | --- |
| Production (16/56, 0.20) | 16 | 11 | **2** |
| Demo (3/5, 0.20) | 3 | 1 | 3 |

Demo does not matter — the data is mock. What matters is that ADR-031 lists `ASNCapFraction` and `VettingCapFraction` under *"parameters that must never appear in NetworkProfile"* as fixed policy, while `DataShards` varies freely. Nothing in the struct expresses the relationship between them, so any future profile — a V3 tier, a regional variant, a larger demo — silently re-derives its own collusion threshold.

Whatever the council decides on F-34, the fix has to land as an explicit profile invariant (a `ConfidentialityThreshold` distinct from `DataShards`, or a compiler-enforced test relating the two), alongside the existing `TestProfileShardSizeIsConstant`. Otherwise the same defect returns with the next profile.

---

### F-68 `[CORRECT]` RLS gives tamper-*resistance* against the app, not tamper-*evidence* against the operator.

ADR-032 is the strongest security work in the project. Test #10 — grant the app `DELETE`, retry, observe `DELETE 0` because `FORCE` RLS makes the rows invisible — is the right way to test a control: prove the mechanism, not the absence of a grant.

One qualification. `vyomanaut_migrator` must hold `BYPASSRLS` (or superuser) to run `REFRESH MATERIALIZED VIEW` against `FORCE`-RLS base tables. So the maintenance identity can delete audit receipts by design. That is unavoidable at the database layer — which is precisely why F-23's hash-chaining matters, and why F-22's V2 trust story does not close.

ADR-015's *"append-only Postgres table is resistant to tampering"* and data-model.md's tamper-evident language should be read as: **resistant to the request path, not evident to a third party.** That is a real and worthwhile property. It is not the property a provider needs in a dispute, and Domain G exists to supply the difference.

---

## What holds up

Genuinely strong work in this block, and worth naming because the finding list above is dense:

- **ADR-032** and **ADR-033** each ship a numbered strength-test table run in production posture — non-superuser migrator, not the trivially-passing superuser case. ADR-032 caught and discarded its own first draft (the defensive `ALTER ROLE`, which fails under a least-privilege migrator) via test #1. That is how this should work.
- **ADR-033's nonce guard table** solves a real bind — partitioning forces the unique key to include the partition column, which would silently weaken replay protection — with a small non-partitioned table, and proves it with test #3. Rejecting `pg_partman` on stated dependency discipline rather than convenience is the right instinct.
- **ADR-036** found an unauthenticated deletion protocol and argued precisely why the previous defence (0-RTT prohibition) addressed a different threat.
- **ADR-031 §5** shows its arithmetic: `floor(5 × 0.20) = 1` shard per ASN, therefore 5 shards need 5 distinct ASNs, therefore `MinDistinctASNs = 5` and not 2. This is exactly the substitution-shown discipline F-43 asked for, appearing spontaneously.
- **ADR-037** and **ADR-044** both decline to ship a number they cannot defend, and say why. ADR-044 additionally rejects the framing its own marketing would prefer because the cited evidence contradicts it.
- **ADR-034's fallback rule** — unmapped codes render a generic message and log a warning — lets the copy table ship incomplete without blocking M13/M15. Good incremental design.

---

## Routing summary

**To the council:**

1. **F-58 / F-59** — re-take ADR-033's deferred capacity decisions against corrected audit arithmetic. Cheap now, expensive later. Pairs with the F-02/F-03 session already queued.
2. **F-54** — what belongs in `NetworkProfile`. Change the inclusion criterion from "differs between modes" to "expected to change."
3. **F-67** — express the confidentiality threshold as a profile invariant, whatever F-34 resolves to.
4. **F-60** — is the analytical repair model trustworthy or not. Same answer must serve ADR-004 and ADR-037.
5. **F-61** — accept a public threshold ladder and design for it, or disclose less.
6. **F-62** — mode visibility in the GUI, before either app is built.
7. **F-66** — commission the protocol authorisation inventory.

**Corrections (no ruling needed):** F-53, F-55, F-56, F-57, F-63, F-64 (partial), F-68. Plus the ~15 stale ADR-032 references named in ADR-036's renumbering note, which block the M-OBS/13/14 build sessions.

**To the reading list:** F-64 (rescope R-16 to both sides of the carbon ledger), F-65 (re-tag the ADR-038–045 cluster). No new domains — this block is drift and arithmetic, not missing literature.
