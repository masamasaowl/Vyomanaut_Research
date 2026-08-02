# Paper 46 — Storj Node Operator Documentation and Community Forum

**Authors:** Storj Labs (storj.dev documentation); Storj community (forum.storj.io)
**Venue / Year:** storj.dev/node and related pages; forum.storj.io | retrieved 2026, continuously maintained
**Citations:** not applicable — documentation and community-forum sources, not an academic paper
**Topics:** #21, #5
**ADRs produced:** none — corrects the provider-persona assumption behind `ux-decisions.md` §4

---

## Problem Solved

Paper 05 (the Storj v3 whitepaper) is your primary source for Storj's *protocol design*. It says nothing about who actually operates a Storj node in practice — a whitepaper describes intended design, not observed adoption. Your original provider persona ("home desktop/NAS operators without technical expertise," `requirements.md`) needed checking against how Storj — the closest real, running, comparable network — is actually operated today. This review of Storj's own operator-facing documentation and public forum supplies that evidence.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Storj's setup path requires manual port forwarding, dynamic DNS, and a recommended dedicated device (NAS/Pi) | A zero-configuration setup flow | Lower engineering burden for Storj, but selects for a technical, home-lab operator base regardless of how the product is marketed |

---

## Breaks in Our Case

- **Your original persona describes providers as lacking technical expertise**
  ≠ **the closest real, running, directly comparable network's actual operator base — evidenced by its own required setup steps and its own community's vocabulary (Ceph clusters, LACP bonding, dynamic DNS, UPS planning) — skews toward home-lab and small-hosting technical hobbyists**
  → This is the strongest single piece of evidence behind the "prosumer provider" vs. "aspirational mainstream provider" split adopted in `ux-decisions.md` §4. It does not prove a non-technical provider is impossible for you — only that achieving one requires removing the setup complexity Storj's documentation treats as baseline.

- **Storj's protocol design (Paper 05) is agnostic to operator skill level**
  ≠ **Storj's actual deployment requirements (this paper) are not**
  → The gap is entirely in the setup/operational layer, not the protocol — encouraging evidence that if your provider app removes port-forwarding/dynamic-DNS/dedicated-hardware requirements (NAT traversal handled transparently by your own P2P layer), a materially less technical operator base is achievable without changing your storage protocol at all.

---

## Decisions Influenced

- **Provider persona (`ux-decisions.md` §4)** `PRIMARY EVIDENCE, ALREADY APPLIED`
  Split the provider persona into a realistic v1 ("prosumer") and an aspirational v2+ ("mainstream") segment, rather than assuming `requirements.md`'s original wording describes who will actually show up.
  *Because:* Storj's own setup documentation and community forum are the closest available evidence of what a comparable network's operator base actually looks like in practice.

- **Provider app setup-flow requirement (not yet an ADR — forthcoming)**
  If you want a less technical operator base than Storj's, the app's setup flow must not require the operator to understand port forwarding or configure dynamic DNS themselves.
  *Because:* Storj's documentation shows that requiring this step, even as a "recommended" setup detail, selects for a technical operator base regardless of marketing — your own NAT-traversal handling in `internal/p2p/nat.go` already targets removing this requirement and should be treated as load-bearing for the persona goal, not incidental.

---

## Disagreements

- **It could be argued Storj's technical operator base is a marketing failure, not an inherent property of decentralized storage as a category** — different messaging alone might have reached a broader audience without changing the setup requirements.
  *Implication for us:* the evidence here does not settle that, because Storj's setup *documentation* — not just its marketing — imposes real technical steps. The more defensible reading, adopted in `ux-decisions.md`, is that both matter, but the setup-requirement evidence is the stronger and more directly actionable of the two for your own app design.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q46-1.
