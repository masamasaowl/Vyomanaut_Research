# Paper 14 — QUIC: A UDP-Based Multiplexed and Secure Transport

**Authors:** J. Iyengar (Fastly), M. Thomson (Mozilla)
**Venue / Year:** IETF RFC 9000 | May 2021
**Topics:** #12, #1
**ADRs produced:** none (ADR-021 partially unblocked — libp2p blocker remains)

---

## Problem Solved

TCP was designed in 1974. Every modern transport feature — TLS encryption, connection resumption,
multiplexed streams, NAT traversal keepalives — was bolted on afterward, producing compounding
latency from separate handshakes and head-of-line blocking that stalls all streams on a single
lost packet. QUIC rebuilds the transport layer over UDP with TLS 1.3 baked in, connection
identity decoupled from IP address, and independent stream delivery by design.
For Vyomanaut, the critical properties are connection migration (providers have dynamic IPs),
multiplexed parallel chunk streams (no HOL blocking), and 1-RTT reconnection after nightly
provider absence.

---

## Key Findings

**Connection identity via Connection IDs (CIDs), not IP:port tuples:**
A QUIC connection is named by a Connection ID exchanged during the handshake. When a provider's
IP changes — dynamic DHCP lease, router restart, ISP address rotation — the connection survives.
The server sends NEW_CONNECTION_ID frames in advance; the migrating endpoint switches to one,
sends a PATH_CHALLENGE, and only commits to the new path once PATH_RESPONSE is received. In-flight
chunk transfers are not dropped.

**Multiplexed streams with no head-of-line blocking:**
A single QUIC connection carries arbitrarily many independent streams. Each stream has its own
flow control window (MAX_STREAM_DATA). A lost UDP packet carrying stream-3 data does not block
delivery of stream-7 data — the two streams dequeue independently. For parallel chunk transfer
(pipelining confirmed by Q01-2), this eliminates the primary TCP bottleneck: a single retransmit
does not stall all in-flight chunks.

**Integrated TLS 1.3 — combined handshake:**
QUIC and TLS 1.3 share a single handshake round trip. A new connection pays 1-RTT before
application data flows. Session resumption via TLS session tickets enables 0-RTT: the client
can send application data in the first packet of a resumed session. 0-RTT data is encrypted with
a resumption key from the previous session, not the current handshake — making it vulnerable to
replay unless the server defends explicitly (see Breaks).

**Explicit ACK ranges — no cumulative acknowledgement:**
QUIC ACK frames list explicit packet number ranges. A receiver that receives packets 1, 2, 4
sends ACK {1-2, 4}. The sender knows exactly which packets are lost and retransmits only those.
This eliminates TCP's SACKs-as-extension complexity and reduces per-packet overhead in a
high-churn network.

**Packet number spaces — three independent spaces:**
QUIC uses three packet number spaces (Initial, Handshake, Application Data), each with its own
keys and numbering. Key compromise of one space does not affect the others. For Vyomanaut, the
Application Data space is the only one used after the handshake completes; Initial and Handshake
keys are discarded after connection setup.

**Flow control at two levels:**
Connection-level: MAX_DATA cap on total bytes across all streams.
Stream-level: MAX_STREAM_DATA cap per stream.
Both are credit-based: a sender can only transmit up to the receiver's advertised maximum. This
is directly useful for throttling repair bandwidth — a provider under CPU pressure can shrink its
stream-level credit to signal backpressure without closing the connection.

**NAT binding maintenance via PING frames:**
UDP NAT bindings in home routers time out in 30–120 seconds of inactivity. QUIC specifies a
max_idle_timeout transport parameter; connections below this threshold send PING frames to
maintain the NAT binding. For Vyomanaut providers idle between audit challenges (interval up to
24 h), persistent connection maintenance is not feasible — each audit challenge round-trip will
require a fresh connection or session resumption.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| UDP as base transport | TCP | Avoids OS kernel TCP stack; enables connection migration; loses automatic OS-level ordering |
| Connection IDs for identity | IP:port tuple identity | Providers can migrate IPs without breaking transfers; CIDs must be kept confidential to prevent linkability tracking |
| 0-RTT session resumption | Full 1-RTT handshake on every reconnect | Sub-millisecond reconnect for returning providers; replay vulnerability on 0-RTT data that must be mitigated per-operation |
| Stream-level independence | TCP multiplexing (HOL-blocked) | Each chunk stream fails or succeeds independently; complexity shifts from transport to stream management in the application |
| Explicit ACK ranges | Cumulative ACK | Precise loss detection; larger ACK frames under high packet loss |
| Integrated TLS 1.3 | Separate TLS negotiation | One fewer round trip; TLS keys are now QUIC's responsibility — a QUIC bug can break both transport and cryptographic security |

---

## Breaks in Our Case

- **QUIC assumes UDP is reachable** ≠ **some Indian home ISPs and corporate networks block non-DNS UDP**
  → A subset of providers will be behind UDP-blocking middleboxes. libp2p must provide a fallback
    (WebTransport over HTTPS/3, or TCP-based relay) without requiring provider configuration.
    The impact on effective provider count must be measured at launch.

- **0-RTT data is replay-vulnerable by spec design** ≠ **our audit challenge-response must be non-replayable**
  → 0-RTT must be disabled or treated as unsafe for any operation that carries an audit response
    or a signed receipt. Reconnection for chunk data transfers (no auth consequence) may safely
    use 0-RTT. The latency cost of forcing 1-RTT for audit interactions is unknown (see Q12-2).

- **NAT binding timeout (30–120 s) is shorter than our audit interval (24 h)** ≠ **providers cannot hold a
  persistent QUIC connection across a full polling period**
  → Each audit challenge must begin with a fresh QUIC connection or 0-RTT/1-RTT resumption.
    The microservice initiates the challenge; the provider daemon must accept incoming QUIC
    connections, which requires either a stable public IP or NAT hole-punching via libp2p.

- **QUIC's connection migration latency spike is indistinguishable from a slow response** ≠ **ADR-017
  uses response_latency_ms as a JIT retrieval detector**
  → A provider migrating IP during an audit challenge will show elevated latency. Without a
    migration signal from the provider, the JIT detector will produce a false positive. The
    audit receipt schema must account for this (see Q12-3).

---

## Decisions Influenced

- **[ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) [#12 P2P Transfer Protocol]** — RFC 9000 blocker removed.
  QUIC is confirmed as the transport layer for all provider-to-provider and provider-to-client
  chunk transfers. Streams map directly to individual chunk pipelines; connection migration
  handles dynamic IPs at zero application-layer cost. ADR-021 remains Proposed — the libp2p
  spec blocker (NAT traversal strategy, hole-punching protocol, relay fallback) is still open.
  *Because:* RFC 9000 Sections 9 (connection migration), 2 (streams), and 5 (0-RTT) directly
  resolve the transport-layer questions in ADR-021.

- **[ADR-017](../decisions/ADR-017-audit-receipt-schema.md) [#2 Proof of Storage — receipt schema]**
  The `response_latency_ms` field used as a JIT retrieval detector (ADR-014 Defence 3) may spike
  during a legitimate QUIC connection migration. A migration flag or path-validation timestamp
  should be considered as an optional field in a future schema version. No schema change is
  required now — the false-positive rate from migration events will be empirically low for
  desktop providers with stable home connections.
  *Because:* RFC 9000 Section 9.2 (path validation latency) introduces a known source of latency
  that is unrelated to data availability.

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]** — QUIC CID
  confidentiality must be enforced at the libp2p layer to prevent a monitoring node from linking
  multiple connections to the same provider by observing CIDs in flight. RFC 9000 Section 9.5
  specifies that CIDs should be rotated to prevent linkability. This requirement must be carried
  into the libp2p configuration.
  *Because:* DHT lookups must not expose file identity (Q07-5); CID linkability is a parallel
  traffic-analysis vector.

---

## Disagreements

- **HTTP/3 (RFC 9114):** Builds application semantics on top of QUIC and is the primary consumer
  of RFC 9000 in practice. HTTP/3's stream mapping (one request = one stream) is a useful model
  for Vyomanaut's chunk transfer design, but the full HTTP/3 framing overhead is unnecessary
  for a binary chunk protocol.
  *Implication for us:* Use raw QUIC streams directly via libp2p rather than HTTP/3. libp2p's
  QUIC transport does this by default.

- **TCP with TLS 1.3:** Achieves near-equivalent security to QUIC for stable connections, and
  avoids UDP-blocking middlebox risk. Some deployments (Akamai, internal benchmarks) show TCP
  matching QUIC latency on low-loss links.
  *Implication for us:* QUIC's advantage for Vyomanaut is specifically connection migration and
  stream independence — not raw throughput. If UDP is blocked at >10% of provider sites, the
  libp2p fallback to TCP must be well-tested and not treated as a degraded path.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q14-1 through Q14-3.
