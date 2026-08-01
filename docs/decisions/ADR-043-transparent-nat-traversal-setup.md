# ADR-043 — Provider Setup Requires No Manual Port-Forwarding or Dynamic DNS

**Status:** Proposed
**Topic:** #21 Desktop/Institutional Compute Harvesting Precedent
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 46 (Storj node operator docs)

---

## Context

Storj's own node-operator documentation requires manual port forwarding and recommends a dynamic-DNS service for operators without a static IP — steps that, per Paper 46, select for a technical home-lab operator base regardless of how the product is marketed. The provider persona in `ux-finding.md` §4.2 explicitly targets a less technical operator than Storj's actual base. This is only achievable if Vyomanaut's setup flow does not require the same manual networking steps.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Follow Storj's precedent: require manual port-forwarding and recommend dynamic DNS | Simpler daemon-side networking logic | Directly reproduces the exact setup complexity Paper 46 identifies as selecting for a technical operator base — undermines the persona goal in `ux-finding.md` §4.2 before the app is even built |
| **Transparent NAT traversal handled entirely by the daemon, no manual networking configuration exposed in setup** | Matches the "prosumer, not home-lab" persona; consistent with the existing `internal/p2p/nat.go` NAT-traversal work already in the codebase | Requires the setup flow to honestly communicate on the rare occasion NAT traversal fails (symmetric NAT, restrictive institutional firewalls) rather than presenting a false "you're all set" |

## Decision

The Provider app's setup flow does not ask the operator to configure port forwarding, dynamic DNS, or any other manual networking step. NAT traversal is handled transparently by the daemon's existing P2P layer. If traversal fails for a given network (e.g., symmetric NAT), the app must say so honestly and explain the likely cause in plain language (per the IC §14 copy contract), rather than silently failing or asking the operator to self-diagnose a networking problem the persona goal assumes they can't.

## Consequences

**Positive:** directly closes the gap Paper 46 identifies between Storj's actual operator base and Vyomanaut's target persona; no new protocol work required — this is a setup-flow and error-handling decision, not a networking-layer one.

**Negative:** NAT-traversal failure cases must be designed for explicitly in the setup flow (a new UX surface), rather than assumed away by pushing the problem onto the operator as Storj does.

**Open constraints:** the actual failure rate of transparent NAT traversal across real Indian ISP/router configurations is not yet measured — flagged as a research item before the setup-flow error states can be finalized.

## References

- Paper 46 — Storj node operator documentation and community forum
