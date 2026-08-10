# ADR-063 — Demo-track dependency substitutions: declare, do not fix

**Status:** Proposed — blocked on ADR-062
**Track:** DEMO *(with a stated LTS reversal obligation)*
**Topic:** #6 Network Layer / #3 Erasure Coding
**Supersedes:** — *(amends ADR-021, ADR-001, ADR-003 for the demo track only)*
**Research source:** `internal/p2p/doc.go`, `internal/erasure/doc.go` (substitution records written
at build time); IC §11 (vendoring-disclosure rule); this document's N-02

## Context

N-02 above, in full. Two load-bearing external dependencies were replaced with stdlib
implementations because the build environment could not reach them, and both replacements are
recorded only in package `doc.go` files. IC §11 requires that even *vendoring* `go-libp2p` be
recorded in `architecture.md §4.1` with version and reason, and names silent vendoring as
prohibited. Substitution is larger than vendoring and is currently less documented.

The engineering was done well. Peer IDs are byte-exact against `peer.IDFromPublicKey`. The DHT key
validator implements IC §12's accept/reject rules byte-for-byte — which is the piece CI check 5
exists to guard. The 0-RTT deny-list is preserved under TLS session-ticket resumption, holding the
property IC §4 actually cares about. This ADR is not a criticism of that work; it is the missing
record of it.

### Options considered

| Option | Pros | Cons |
| --- | --- | --- |
| **Fix now — restore go-libp2p before the demo ships** | Demo and LTS share one stack; no reversal work later | Several hundred transitive modules across blocked vanity paths; multi-week; and it buys the *demo* nothing — the demo's bar is "it runs," and it already does. Directly contradicts the freeze |
| **Leave as-is, documented only in `doc.go`** | Zero work | Violates IC §11's own disclosure standard; a future reader of the frozen demo cannot tell which properties were proven and which were approximated; N-05's three checks stay silently mis-rationalised |
| **Declare in an ADR and in `architecture.md §4.1`; bind the LTS to reverse it — chosen** | The freeze stays honest; N-05's checks get correct rationales; the LTS inherits an explicit, scoped obligation instead of an inherited assumption | Requires the demo scope record (Session 18.3.1) to be written carefully — an under-written one is worse than none |

### Decision

**1. Both substitutions are ratified for the demo track**, and recorded in `architecture.md §4.1`
with the mechanism-by-mechanism table from N-02 above, not a summary sentence.

**2. Each substitution declares which property is preserved and which is approximated.** This is
the operative content:

| Contract | Preserved exactly | Approximated | LTS obligation |
| --- | --- | --- | --- |
| Peer ID (ADR-001) | **Yes** — byte-exact vs `peer.IDFromPublicKey` | — | None |
| DHT key validator (IC §12) | **Yes** — accept/reject rules byte-for-byte | Routing machinery (k-buckets, iterative lookup) is from-scratch | Restore `go-libp2p-kad-dht` behind the same validator |
| Transport auth (NFR-016) | Cryptographic authentication bound to peer identity | **Mechanism**: TLS 1.3/TCP, not QUIC v1 + Noise XX | Restore QUIC + Noise XX |
| 0-RTT policy (IC §4) | Prohibited protocols never ride a replayable shortcut | **Mechanism**: TLS session tickets, not QUIC 0-RTT | Restore under real QUIC |
| NAT traversal (IC §4.3) | Three-tier shape | **All three tiers**: no AutoNAT, no DCUtR, no Circuit Relay v2 reservations | Restore all three; NFR-006's 50 ms relay budget becomes measurable for the first time |
| Erasure coding (ADR-003) | RS(k,n) correctness at demo and prod parameters | **Performance** — hand-rolled, unbenchmarked against the ADR-009 5% CPU budget | Restore `klauspost/reedsolomon`; re-benchmark |

**3. N-05's three checks are restated, not weakened.** `TestDHTKeyValidatorPersists` keeps its
assertions and gains a comment stating that its stated trigger (a `go-libp2p` upgrade) does not
exist on this track and that the validator rules are the guarded property.
`TestTransportAuthentication` keeps its assertions and states the mechanism actually under test.

**4. The LTS reverses every row before any M20+ work depends on it.** This is Milestone 19's entire
subject.

#### Consequences

The frozen demo becomes self-describing: anyone who opens it in two years can tell exactly what it
proved. IC §11's disclosure standard is met. Milestone 19 gets a checklist instead of an
archaeology exercise. Cost: one ADR and one `architecture.md §4.1` table.

**Open constraints:**

- Whether the hand-rolled RS implementation is correct at **production** parameters RS(16,56) is
  untested — the demo only exercises RS(3,5). Not a demo concern; a real one for anyone tempted to
  reuse the code. Recorded so nobody is. Q-D-2.
- The from-scratch Kademlia has not been tested above 5 nodes. Its behaviour at the 10,000-node
  scale you asked about is unknown and should not be assumed. Q-D-3.

---
