# ADR-085 — `cmd/` Cross-Package Orchestration: Documented IC §11 Exception, Not a Package Move

**Status:** Proposed — recommendation for Aryan's review; not yet ratified
**Track:** BOTH
**Topic:** #1 Coordination Architecture
**Supersedes:** — *(clarifies IC §11's scope; IC §9 itself is unchanged)*
**Superseded by:** —
**Research source:** Milestone 12 audit corrections session — QA/Audit team's M12 SRE review, Finding 3 ("Systemic 'business logic in `cmd/`' — direct conflict with IC §11, and it's not just M12")

---

## Context

IC §11 states, without qualification, under "Forbidden Code Patterns": *"`cmd/` is wiring only — flag parsing, dependency construction, signal handling. Any function with testable behaviour belongs in `internal/`. An engineer who needs to test a `cmd/` function should move it to an `internal/` package first."*

IC §9 states, equally without qualification: *"The microservice entrypoint wires `internal/audit`, `internal/scoring`, `internal/repair`, and `internal/payment` together; none of these four packages imports any of the others directly."*

These two rules are in direct, structural tension for exactly one kind of code: logic that needs to call into two or more of those four mutually-import-restricted packages together. IC §9 forbids any single `internal/` package from being that caller. IC §11 forbids `cmd/` from being that caller either. Nobody is left to write it.

This is not hypothetical. The M12 audit's Finding 3 documents the actual, live consequence: `cmd/microservice` is ~2,095 non-test lines across 10 files, with dedicated, substantial test files (`audit_dispatch_test.go`, `main_test.go`) testing real, non-wiring logic — wire-frame parsing (`writeChallengeRequest`/`readChallengeResponse`), signature-verification dispatch (`adjudicateResponse`), DB-query-driven peer resolution (`resolveProviderPeer`), materialized-view SQL generation (`scores_view.go`), key-loading/validation (`keys.go`). `cmd/provider` (Milestone 13) independently arrived at the identical pattern: four `handler_*.go` files (~1,712 non-test lines) each with a dedicated test file, implementing the server side of the same wire protocols. This is now the established convention for **both** of the project's daemon entrypoints — a two-milestone-deep precedent, not a one-off lapse in a single session.

`cmd/microservice/secrets_client.go` already identified and explicitly flagged an analogous tension (IC §8 vs IC §9, over where the cluster audit secret's shared-fetch logic may live) in its own header comment. This IC §9-vs-§11 tension was not similarly flagged anywhere before the M12 audit — evidence it was missed by both the Session 12.1.1/12.1.2 implementer and by every prior audit pass, not that it was deliberately accepted and is safe to leave silently unenforced.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| **(A)** Amend IC §11 with an explicit, narrowly-scoped exception for cross-package orchestration code that IC §9 itself requires to live in `cmd/` — **chosen** | Zero code movement; zero risk to two already-shipped, already-tested milestones (M12, M13) while M16–18 is under active, unrelated development in adjacent packages; the exception is narrow enough to still flag genuinely misplaced logic (business logic unrelated to orchestrating the four IC §9 packages) in future audits; resolves the tension honestly — IC §11 stops silently being unenforced against otherwise-compliant code | IC §11's "wiring only" rule becomes conditional rather than absolute; requires precise scoping language so it doesn't become a loophole for arbitrary logic to migrate into `cmd/` instead of `internal/` |
| **(B)** Relocate the orchestration logic into a new `internal/orchestration`-style package that `cmd/microservice` and `cmd/provider` call into | Restores IC §11 to an unqualified rule; keeps `cmd/` genuinely wiring-only, matching its literal text | A real refactor touching ~3,800 combined lines across two already-shipped, already-tested milestones (M12, M13) — this session's own scope is M12 audit corrections, not an M12+M13 refactor; undertaking it now, while M16–18 is independently changing adjacent code (per this session's own operating context), meaningfully raises the risk of merge conflicts and regressions in code this session has no mandate to otherwise touch; `internal/orchestration` would itself need to import all four of `internal/audit`/`internal/scoring`/`internal/repair`/`internal/payment` — which is exactly what IC §9 forbids of any `internal/` package today, so this option requires amending IC §9 as well, not just moving code, doubling its actual scope of change |
| **(C)** Leave both rules exactly as written, unenforced against the current code | No document change, no code change | Not a real option — the audit's own framing is correct: leaving IC §11 silently unenforced against a now-two-milestone-deep, ~3,800-line precedent means the rule exists on paper only, and the gap will only get more expensive to close the longer it's left (M14+ likely adds more of the same pattern to any future daemon entrypoint) |

## Decision

**Amend IC §11 with an explicit, scoped exception (Option A).** The rule now reads, in full:

> `cmd/` is wiring only — flag parsing, dependency construction, signal handling. Any function with testable behaviour belongs in `internal/`. An engineer who needs to test a `cmd/` function should move it to an `internal/` package first.
>
> **Exception (ADR-085):** code in a `cmd/` entrypoint that exists *specifically* to orchestrate two or more `internal/` packages IC §9 forbids from importing each other directly (currently: `internal/audit`, `internal/scoring`, `internal/repair`, `internal/payment`) is permitted to carry testable behaviour, and to be tested in place, in `cmd/`. This exception does **not** extend to logic that is merely *located* in the same file as such orchestration but does not itself require cross-package coordination — that logic still belongs in `internal/` per the rule above, and "it's already in `cmd/` for unrelated reasons" is not a basis for leaving it there. A future audit finding logic in `cmd/` that could be tested and used correctly from a single `internal/` package, without needing IC §9-restricted cross-package access, should still cite this rule as violated.

**Rationale for choosing (A) over (B):** this is a case where a structurally "purer" fix (B) is the wrong call for *this* session specifically, not in principle. The current code is not badly engineered — the audit's own Phase 2 assessment calls it "well-organized, well-tested, and honestly documented throughout." The actual cost of the tension is that IC §11 currently makes a promise about `cmd/` the codebase doesn't keep, not that the codebase itself is broken. Option A closes that promise-gap today, with no execution risk, while leaving the door open for Option B later — most naturally as its own dedicated session once M16–18's active changes have settled, at which point `internal/orchestration` could be introduced as a genuine refactor with its own regression coverage, rather than as a rushed side effect of an M12 corrections pass with no mandate to touch M13's already-shipped `cmd/provider` code.

**This decision is Claude's own engineering judgment, made without Aryan's explicit direction on this specific question, and is flagged here — per this session's own working pattern — for Aryan to accept or decline.** If declined, Option B or Option C-with-a-different-resolution remain open; this ADR's Status should be updated accordingly rather than silently superseded.

## Consequences

**Closed:** IC §11 no longer silently contradicts ~3,800 lines of already-shipped, already-tested code across two milestones. Future audits have a precise, citable rule to check `cmd/` code against — either it's genuine IC §9-restricted cross-package orchestration (permitted) or it isn't (still forbidden, per IC §11's original text).

**Not closed by this ADR:** no code moves. `cmd/microservice` and `cmd/provider` are unchanged by this decision alone — this ADR only changes what IC §11 *permits* them to already be doing, not what they *are* doing. If a future audit finds `cmd/`-resident logic that does NOT meet this exception's scope (i.e., doesn't require IC §9-restricted cross-package access), it should still be flagged and moved to `internal/`, exactly as IC §11 already requires.

**Follow-up, not undertaken here:** Option B (a dedicated `internal/orchestration` refactor) remains a reasonable future direction if the `cmd/`-resident orchestration surface continues to grow past M13's current footprint — worth revisiting once M16–18's independent changes have stabilized, as its own scoped session with its own regression plan, not folded into a corrections pass for an unrelated milestone.

```

---

## Verification performed

Every change above was implemented and verified against a live, from-scratch environment matching the audit's own stated setup as closely as this session could reproduce — real Go 1.26.5 (not a substitute version), real PostgreSQL 16 with the demo schema applied via the project's own migration generator, real `vyomanaut_app`/`vyomanaut_migrator` roles.

| Check | Result |
|---|---|
| `go build` (full non-RocksDB dependency graph: `cmd/microservice`, `internal/api`, `internal/audit`, `internal/p2p`, `internal/repair`, `internal/payment`, `internal/scoring`, `internal/config`, `internal/crypto`, `internal/client/*`) | Clean |
| `go vet` (same scope) | Clean |
| `gofmt -l` (same scope) | Clean — no output |
| `go test -race -count=1` (same scope) | **All packages pass** |
| `scripts/ci/migration_check.sh` | **30/30 PASS** |
| `scripts/ci/grep_checks.sh` | **7/7 PASS** |

`internal/storage` and `internal/vettingchunk` were excluded from this session's build/test runs — both fail to build in this sandbox on a pre-existing RocksDB CGo header/library version mismatch unrelated to any change here, matching the audit's own explicitly stated scope ("clean for `cmd/microservice` and its full non-RocksDB dependency graph"). No file in either package was touched by this session.

New regression tests added (test-first, each confirmed failing against unpatched code before the fix, passing after):
- `internal/p2p/dht_test.go`: `TestFindPeerUnknownReturnsErrPeerNotInRoutingTable`, `TestFindPeerAfterBootstrapReturnsSeedAddr`, `TestFindPeerAfterInboundPutProviderRecord`
- `cmd/microservice/adapters_test.go` (new file): `TestResolveProviderPeerUsesDHTFallbackWhenStale`, `TestResolveProviderPeerFallsBackToStoredAddressWhenDHTHasNothing`, `TestResolveProviderPeerIgnoresDHTWhenNotStale`, `TestResolveProviderPeerNilDHTFallsBackSafely`
- `cmd/microservice/audit_dispatch_test.go`: `TestAdjudicateResponseFailStatusesAreFailEvenWithBadSignature`, `TestAdjudicateResponseFailStatusWithValidSignature` (replacing the now-stale `TestAdjudicateResponseNonOKStatusesAreFail`, split into this pair plus `TestAdjudicateResponseNoSigStatusesAreFailWithoutDB` since Finding 4 changed which status codes need a live DB)

---

## Deliberate non-changes (things this session found but did NOT change, and why)

- **`cmd/provider`'s own DHT gap (`Store: nil`, marked `GAP` in that file) and the absence of any DHT bootstrap-seed mechanism for the provider network** — real, pre-existing, cross-milestone issues that materially affect how much practical benefit Finding 2's fix delivers today. Left untouched: both live in M13-owned code, out of scope for an M12 corrections session, and touching them wasn't requested. Flagged in `main.go`'s diff and in the "Important" note above so it isn't silently lost.
- **Option B for Finding 3 (moving `cmd/`-resident orchestration logic into a new `internal/orchestration` package)** — considered and rejected in favor of the documented-exception approach (ADR-085, Option A) specifically because it would require touching ~3,800 lines across two already-shipped milestones while M16–18 is independently active. ADR-085 leaves this option open for a dedicated future session.
- **`repair.RepairTransport`'s interface itself** — untouched; `repairTransportAdapter` continues to satisfy it exactly as before, now with an added `dht` field.
- **No `go.mod`/`go.sum` changes are included above.** This sandbox needed temporary `replace` directives (GitHub-mirror routes for `golang.org/x/*`, `google.golang.org/protobuf`, `gopkg.in/yaml.v3`, `gopkg.in/check.v1`) purely to fetch dependencies through this environment's network allowlist — the same class of workaround the audit's own environment note describes ("network-mirror replace directives... to route around blocked vanity-import domains"). These are a sandbox artifact only, not a real fix, and are excluded from the diffs above.
