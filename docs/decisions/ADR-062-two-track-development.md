# ADR-062 — Two-track development: Demo freeze at M18, LTS from M19

**Status:** Proposed — blocked on your confirmation of the repository topology in §Decision 2
**Topic:** #0 Project Governance *(new topic)*
**Supersedes:** — *(governs `build.md`'s dependency graph from M17 onward)*
**Research source:** Karma's ruling, August 2026; completeness sweep of `Vyomanaut_V2` @ working copy

## Context

Every planning document in the repository assumes a single artifact progressing M0 → M18 to
production launch. `build.md`'s dependency graph ends `M17 → M18`; MVP §8.5 gates runbooks on
private beta; `requirements.md §7.4` calls four benchmarks V2 launch blockers. That assumption is
now false in a way no document records: there are **two** artifacts with different bars, different
lifetimes, and — critically — **different definitions of "done."**

Without a ratified decision, three concrete failures follow. Demo work gets held to LTS standards
and never ships (M17's secrets adapters are the live example — pure production hardening sitting
directly in front of the demo's finish line). LTS work gets started against demo compromises and
inherits them silently (the P2P substitution, N-02, is the live example). And a stashed demo gets
demonstrated years later with nobody able to say what it did and did not prove.

### Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **No ratified split — one track, demo as a checkpoint tag** | Zero governance overhead; one document set | The bar is undefined at the tag. Every subsequent milestone has to re-litigate "is this demo-grade or LTS-grade?" — which is exactly the question that produced M17 blocking the demo |
| **Branch within one repository** | Shared history; cherry-picking fixes is trivial | The demo must be *frozen*, and a live branch in an active repo is not frozen. CI check 10's ADR ceiling, the `NetworkProfile` prod half, and the migration prod profile all drift under the demo's feet |
| **Separate repository forked at the demo tag — chosen** | The freeze is real; the demo's ADR ceiling, dependency set and CI gate are fixed at fork time; LTS is free to make breaking changes without a compatibility argument | Fixes do not flow automatically. Accepted deliberately: after the freeze, the demo should not receive fixes — a demo that keeps changing is not a stash |

#### Decision

**1. Two tracks, with different completion bars, stated once and referenced thereafter.**

| | Demo | LTS |
| --- | --- | --- |
| Milestones | M0 – M18 | M19 – (M30, est.) |
| Profile | `demo` **only** | `demo` + `prod` |
| Interface | CLI only | CLI + two Wails desktop apps |
| Bar | **It runs, and a file demonstrably moves between nodes** | Outperforms every competitor; thousands of desktops |
| Audit primitive | ADR-002 hash-challenge, permanently | ADR-059 / ADR-060 |
| Dependencies | Substitutions permitted and declared (ADR-063) | Real `go-libp2p`, real `klauspost/reedsolomon` |
| Payments | `MockProvider` only | Razorpay/UPI live |
| Deployment | Single machine (`--sim-count`) or a LAN of desktops | HA cluster, relays, secrets manager |
| Lifetime | Frozen at the M18 tag; **no post-freeze fixes** | Iterated indefinitely |

**2. Repository topology.** `Vyomanaut_V2` is tagged `demo-v1.0.0` at M18 sign-off and archived
read-only. The LTS continues in a new repository, `Vyomanaut_LTS`, forked from that tag.
`Vyomanaut_Research` is **shared** — one research corpus, one ADR series, one set of system-design
documents — with every ADR and every requirement carrying an explicit track tag.

**3. Track tagging is mandatory and machine-checkable.** Every ADR from ADR-062 onward carries a
`**Track:** DEMO | LTS | BOTH` line directly under `**Status:**`. Every new NFR carries a track
column. An untagged ADR fails CI check 10's extended form.

**4. Milestone 17 is reclassified `LTS — deferred`** (N-03). The Demo dependency graph becomes
**M16 → M18**.

**5. No demo functionality enters M19.** The LTS inherits the *repository* at the demo tag as a
starting point, not the demo's *decisions*. Specifically: every substitution in ADR-063 is reversed
before any M20+ work depends on it, and the demo's `SoloMembership` / `envSecretsClient` /
`MockProvider` paths are production-gated, not extended.

**6. Frozen ADR ceiling.** CI check 10's ceiling is fixed at **ADR-064** in the demo repository at
fork time. LTS ADRs (065+) referenced in demo source are a check-10 failure — which is the desired
behaviour: it makes accidental back-porting loud.

#### Consequences

The demo ships. M17's production hardening stops blocking it. Every future "is this demo or LTS?"
question resolves against a table instead of a debate. The cost is a second repository and the
discipline of tagging every ADR — both cheap relative to the cost of the alternative, which the
project has already paid once in M17.

**Open constraints:**

- Whether `Vyomanaut_Research` should also fork is left open. My recommendation is **no** — the
  research corpus is the project's most valuable asset and forking it would split the ADR series
  irrecoverably. Track tags do the same job at a fraction of the cost. Q-D-1.
- The demo tag's reproducibility depends on `go.sum` pinning and on the two hand-repackaged
  `golang.org/x/*` module zips described in `internal/p2p/doc.go`. If those are not vendored at
  freeze time, the demo is not rebuildable from a clean machine. **Addressed in Session 18.4.1** —
  flagging it here because it is the one way the freeze can silently fail.

---
