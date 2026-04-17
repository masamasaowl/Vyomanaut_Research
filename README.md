# Vyomanaut Research

> Distributed storage network powered by over a billion smartphones

V1 is archived at [masamasaowl/Vyomanaut](https://github.com/masamasaowl/Vyomanaut).
V1 failed due to:
1. Lack of research in architecture
2. Structural compromises made during build
3. Inefficient transfer speed, storage, and peer discovery

V2 is a research-first rebuild. This repository contains the system design and the full research log behind it.

---

## Repository map

| Path | What it contains |
|---|---|
| [`docs/system-design.md`](docs/system-design.md) | Functional and non-functional requirements |
| [`docs/research/briefing-protocol.md`](docs/research/briefing-protocol.md) | How to read and document papers in this project |
| [`docs/research/reading-list.md`](docs/research/reading-list.md) | All reading phases — Phase 0 through Phase 2B |
| [`docs/research/open-questions.md`](docs/research/open-questions.md) | Live tracker of open questions across all papers |
| [`docs/research/`](docs/research/) | Individual paper summaries (paper-01 through paper-11) |
| [`docs/decisions/`](docs/decisions/) | Architecture Decision Records (ADR-001 onward) |
| [`docs/references/style-guide.md`](docs/references/style-guide.md) | Writing conventions for this repository |
| [`docs/references/SKILL.md`](docs/references/SKILL.md) | AI skill for documenting new papers |

---

<!-- ## Current architecture (V2)

| Property | Decision |
|---|---|
| Coordination | Hardened microservices + Kademlia DHT for chunk lookup |
| Data transfer | Pure P2P via libp2p + QUIC |
| Payments | Fiat escrow — Razorpay/Cashfree, UPI, India-first |
| Storage | Zero-knowledge — client-side encryption, service never sees plaintext |
| Erasure coding | Reed-Solomon: s=16, r=40, r0=8, lf=256 KB |
| Audit trail | Signed receipt exchange + Transparent Merkle Log |
| Providers | Desktop-only for V2 (MTTF 180–380 days); mobile deferred to V3 |
| Repair | Lazy repair, 72 h trigger threshold |

All decisions are individually documented in [`docs/decisions/`](docs/decisions/). -->
