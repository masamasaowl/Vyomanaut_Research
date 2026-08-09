# Paper 69 — Dynamic Provable Data Possession

**Authors:** C. Chris Erway (AppNeta), Alptekin Küpçü (Koç University), Charalampos Papamanthou (University of Maryland), Roberto Tamassia (Brown University)
**Venue / Year:** ACM Transactions on Information and System Security (TISSEC), Vol. 17, No. 4, Article 15, April 2015 (extended from ACM CCS 2009)
**Citations:** ~2,500+ — the founding paper of *dynamic* PDP
**Topics:** #2 Proof of Storage, #4 Repair Protocol
**ADRs produced:** none directly; extends ADR-059's evidence base, sharpens Q68-3
**Findings addressed:** F-01, F-32 (evidence); newly bears on the repair sub-problem inside F-01
**Reading list:** Domain A / **R-03** — "proof transfer on repair," the domain's own search string (`dynamic provable data possession update`) is this paper's title

---

## Problem Solved

Papers 66–68 (Ateniese, Juels–Kaliski, Shacham–Waters) all assume a **static, archival file**: tag once at upload, verify forever after, never touch the content again. Vyomanaut's repair protocol (ADR-004) makes that assumption false on a routine basis — every lost shard is reconstructed and re-hosted at the same stripe position, and ADR-059's own text admits the consequence directly: *"A shard that is reconstructed is new bytes at the same stripe position with no valid authenticators, and only the owner holds the keys to generate them, and the owner is offline by design. This is R-03 and it is unsolved in all three papers."*

Erway et al. define **Dynamic Provable Data Possession (DPDP)**: a formal extension of Ateniese's PDP model that adds authenticated **insert**, **modify**, and **delete** operations at the block level, each provably correct against the previously committed state, while preserving PDP's core guarantee — the verifier holds O(1) metadata and never touches the data. The mechanism is a new authenticated data structure, the **rank-based authenticated skip list**, chosen specifically because (unlike a Merkle tree) it supports efficient two-party updates without a rebalancing proof. This paper is the direct answer to R-03's own accept criterion — "auditability survives a shard moving to a replacement provider without the owner online" — and it is worth reading precisely because it names what survives that move and what does not.

---

## Key Findings

### Rank-based authenticated skip lists

A plain authenticated dictionary indexes blocks by search key. Using a block's position as its key breaks under insertion: inserting after block 40 of 100 would require renumbering blocks 41–100. Erway et al. instead store, at every node `v`, the **rank** `r(v)` — the count of leaves reachable below it — and define the hash label recursively over `(level, rank, children)` rather than over a fixed index. A verification path recomputes rank on the fly by walking from the root, so an insertion, deletion, or modification touches only `O(log n)` nodes along a single search path, and updates run in expected `O(log n)` time at both client and server.

### Two update-capable constructions, with a live trade-off between them

| | DPDP I (skip list) | DPDP II (RSA tree) |
| --- | --- | --- |
| Update time (server) | `O(log n)` | `O(n log n)` amortised |
| Query / proof time | `O(log n)` | `O(log n)` |
| Detection probability | `1 − (1−f)^C` | `1 − (1−f)^(C log n)` — strictly higher |
| Security model | Standard model | Standard model |

DPDP II's higher detection probability comes from an RSA-tree structure (Papamanthou et al. 2008) rebuilt periodically to keep its depth constant; the authors present it as a genuine alternative, not a strictly dominated option, for a deployment that values detection probability over cheap updates.

### PrepareUpdate / PerformUpdate / VerifyUpdate

The client (holder of the secret key and the current basis — the root hash) issues `PrepareUpdate`, specifying the operation and, for insert/modify, the new block content. The server runs `PerformUpdate`, physically applying the change and returning a proof. The client runs `VerifyUpdate`, checking the proof against the *previous* basis, and — only if it accepts — computes and adopts the *new* basis. **The update is authenticated end to end: a server that applies a different change than the one requested, or that applies it against a stale prior state, produces a proof that fails verification.** This is a genuine formal result, not a description of what a reasonable implementation would do.

### Detection probability is the same formula, and the same numbers

`Pr[detect] = 1 − (1 − f)^C`, identical to Ateniese's bound. At `f = 1%`, `C = 460` gives 99% confidence — the authors' own worked example, independently matching ADR-060's Ateniese-derived table. Measured on a 1 GB file at 16 KB blocks: 415 KB proof, 30 ms server computation, both essentially unaffected by dynamism — *"the price of dynamism is very low in practice."*

### Index uniqueness, again

Section 4.1 requires that tag indices never repeat, **including across a file's version history** — a modification or a delete-then-reinsert must not reuse a prior index. This is the same requirement Paper 66's Remark 3 states for the static case; DPDP makes it a load-bearing property of the update proof itself rather than an implementation footnote, since a reused index breaks the rank arithmetic the verification path depends on.

### Version control as a nested authenticated dictionary

Section 5.3 layers a second authenticated dictionary, keyed by revision number, between each file's directory entry and its block-level skip list — proofs chain through both layers, at `O(log n + log v)` for `v` versions. Critically: *"The server may implement its method of block storage independently from the dictionary structures used to authenticate data; it does not need to physically duplicate each block of data that appears in each new version."* Persistent authenticated dictionaries (Anagnostopoulos et al. 2001) let old internal nodes be reused across versions rather than replicated. This is the closest structural analogue in the paper to "a shard gets replaced at the same stripe position" — DPDP's own authors built it for exactly this shape of problem, just in the versioned-filesystem domain, not the erasure-coded-storage one.

---

## Substitution at Vyomanaut's parameters

### What DPDP actually answers about repair, and what it does not

DPDP formalises the **update-verification half** of Q68-3: given a claimed new block content at a known index, and a proof, the verifier can check in `O(log n)` that the update is the unique valid successor of the prior committed state — without downloading anything else. Applied to Vyomanaut: whichever party holds the authenticator keys (currently the microservice, per ADR-059) could, in principle, issue a verifiable `PrepareUpdate("replace block i with content C_new")` and check the resulting proof, giving a formal audit trail that a specific reconstructed block was accepted against a specific prior state — genuinely useful for the dispute-resolution problem F-22 also names.

**It does not answer the other half — who is allowed to compute the new tag in the first place.** DPDP's `PrepareUpdate` is run by whichever party holds the secret key, and computing a homomorphic tag over new content still requires that party to have both the key *and* the new content, at the same time, in the same place. That is the identical trilemma Q68-3 already states — give the repairing provider the keys (danger: it can now forge future proofs for that position), have the microservice see the reconstructed bytes (breaks ADR-021's pure-P2P repair model — *"The repair process itself must be P2P — no central entity fetches and re-encodes on behalf of others"* — and ADR-004's own repair flow contacts surviving fragment holders directly), or leave the shard unauditable until the owner returns (unbounded, and the owner is offline by design). DPDP does not close this trilemma. What it adds is a formal, checkable proof *of whichever choice is made*, rather than an unverified claim of one.

### The standard-model property does not survive Vyomanaut's scale either

DPDP I is provably secure in the standard model — no random oracle — which sounds like it resolves Q68-1's stated trade-off in ADR-059 (accept a random-oracle deviation, or move to BLS). It does not, for an arithmetic reason, not a security one: the standard-model proof holds because the **actual challenge (a set of block indices) is transmitted**, not derived from a compact seed. At Vyomanaut's audit rate — 2,867 sampled chunks × 256 blocks = 733,952 challenged blocks per provider per day — transmitting explicit block indices is worse than the 11.7 MB/day coefficient-vector cost that made ADR-059 adopt seed-derived queries in the first place. DPDP's standard-model guarantee and Vyomanaut's bandwidth budget are mutually exclusive at this scale, independent of which primitive is chosen. This closes off DPDP as an escape from Q68-1 rather than opening one.

### The versioning model is the right shape, at the wrong layer

DPDP's nested per-revision dictionary (Section 5.3) is designed for exactly "content at position X gets replaced, do not re-tag everything, keep a provable history." Applied one layer up — treating each **stripe position** (not each chunk) as a versioned object, with "repair" as a new version — the persistent-authenticated-dictionary technique would let the microservice avoid re-deriving history for the other 2,866 chunks on a provider when one shard is repaired. This does not change who can compute a new tag; it only means that *whoever legitimately does* leaves a cheap, chained proof behind rather than an isolated one. Worth carrying into any future council discussion of Q68-3's resolution, as a shape rather than a ready construction.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Rank-based skip list (DPDP I) | Authenticated red-black tree | Two-party rebalancing without complex authenticated-rotation proofs; the paper explicitly rejects the tree option on this ground |
| Rank information at every node | Search-key-per-block indexing | Insertion/deletion in `O(log n)` instead of `O(n)` renumbering; the mechanism this paper exists to contribute |
| Standard-model security | Random-oracle proof | Stronger guarantee in principle; requires transmitting the actual challenge, which is the property Vyomanaut's bandwidth budget cannot afford at scale |
| DPDP I (skip list) | DPDP II (RSA tree) | `O(log n)` update time instead of `O(n log n)`; lower detection probability per fixed `C`, though both use the same formula and the same achievable target |
| Persistent authenticated dictionaries for versioning | Full re-tag per version | Old internal nodes shared across versions instead of duplicated; adds a second dictionary layer and `O(log v)` term to every proof |

---

## Breaks in Our Case

- **The client that issues updates is the same client that holds the data and can regenerate any tag at will** ≠ **Vyomanaut's authenticator-key holder (the microservice) never has the reconstructed shard content unless it violates ADR-021's pure-P2P repair model**
  → DPDP's update protocol assumes away the exact problem Q68-3 is about. Any adaptation must add a separate answer to "who computes the tag," which this paper does not supply.

- **One client, one server, one file, updated by its owner** ≠ **repair is performed by a third party — a surviving fragment holder or the newly assigned provider — neither of whom is the data owner nor (currently) an authenticator-key holder**
  → The `PrepareUpdate` role does not map cleanly onto any existing Vyomanaut party. Assigning it to the microservice requires the microservice to either see plaintext-adjacent ciphertext bytes (breaking ADR-021) or trust a claimed tag from the repairing party without being able to verify it independently — DPDP's proof structure only helps once *someone* has legitimately computed the update.

- **Standard-model security, achieved by transmitting the real challenge** ≠ **a 100 Kbps background bandwidth budget that already rejected transmitting Shacham–Waters' coefficient vector**
  → DPDP's core security advantage over the random-oracle deviation ADR-059 already accepts is unusable at Vyomanaut's challenge volume. This is a genuine, quantified rejection, not a preference.

- **A single authenticated structure per file, held entirely at one server** ≠ **one file is 56 independently-held shards, and DPDP's skip list authenticates blocks *within* one party's copy, not *across* parties holding different erasure-coded fragments of the same content**
  → DPDP does not natively express "prove shard 23 of stripe X was correctly regenerated from shards {1..16, 24..30} by RS(16,56) decode." That is a repair-correctness proof, a different primitive than a storage-possession proof, and it is not in this paper or in Papers 66–68.

---

## Decisions Influenced

- **[ADR-059](../decisions/ADR-059-homomorphic-authenticator-audit.md) [#2 Proof of Storage]** `PROPOSED — evidence added, no status change`
  Adds this paper to the research source list. Sharpens the Q68-3 open constraint: DPDP formalises authenticated-update verification but does not solve the tag-generation-authority trilemma; the three "shapes of answer" already listed in ADR-059 remain the actual decision space. Adds a fourth, weaker option to that list — a persistent-authenticated-dictionary-style provable update log wrapped around whichever of the three shapes the council chooses — as an accountability layer, not a replacement resolution.
  *Because:* R-03 is the domain's own stated blocker, and this paper is the canonical source for it; the council should see what it does and does not close before ruling.

- **No new ADR produced.** Per the project's own rule, an ADR is drafted only for a decision this paper closes or revises. This paper narrows Q68-3's option space; it does not close it. The corresponding narrowing is recorded in ADR-059's Open constraints and in the open-questions tracker, not as a standalone decision.

---

## Disagreements

- **Against DPDP II's implicit invitation to trade update cost for detection probability.** DPDP II's `O(n log n)` amortised update cost is incompatible with Vyomanaut's repair frequency and 100 Kbps background budget regardless of its detection advantage — not evaluated further here, recorded for completeness since the reading list's own R-03 entry does not distinguish between the two constructions.

- **Against reading DPDP's standard-model result as a way out of Q68-1.** It is tempting, on a first pass, to read "DPDP is standard-model secure, and ADR-059's problem is a random-oracle deviation" as DPDP being the fix. The arithmetic in Substitution above shows this is not available at Vyomanaut's scale — the standard-model property and the seed-derived-challenge requirement are mutually exclusive here, not sequentially achievable.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q69-1 (new). Bears on Q68-1 and Q68-3, both updated with this paper's evidence.
