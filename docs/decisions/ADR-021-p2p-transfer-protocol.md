# ADR-021 — P2P Transfer Protocol

**Status:** Accepted
**Topic:** #12 P2P Transfer Protocol
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 14 (RFC 9000), Paper 13 (libp2p spec)

---

## Context

All data movement between providers, and between providers and data owners, is pure P2P. The
transfer protocol must handle NAT traversal (most desktop providers are behind home routers),
connection migration (providers have dynamic IPs), multiplexed parallel chunk streams, and
cryptographic peer identity. S/Kademlia (Paper 03) suggested gRPC or QUIC. Paper 13 (libp2p)
and Paper 14 (RFC 9000) together answer all open questions.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Raw QUIC + custom peer discovery | Minimal stack; full control | NAT traversal, hole-punching, relay, and identity must all be built from scratch |
| gRPC over HTTP/2 over TCP | Mature; familiar tooling | No connection migration; HOL-blocking on TCP; no built-in NAT traversal |
| **libp2p + QUIC v1 (RFC 9000)** | Production-proven in IPFS, Filecoin, Ethereum devp2p; full NAT traversal stack; Kademlia DHT included; stream multiplexing built in | Additional dependency; requires custom DHT key validator and CID rotation configuration |

## Decision

**Framework:** libp2p for all peer discovery, connection management, NAT traversal, and stream
multiplexing.

**Primary transport:** QUIC v1 (RFC 9000).
- Each chunk transfer maps to one unidirectional QUIC stream on an existing connection.
- No head-of-line blocking: a lost packet on one chunk stream does not stall other streams.
- Connection migration via Connection IDs: when a provider's IP changes (DHCP renewal, router
  restart), in-flight transfers survive. PATH_CHALLENGE / PATH_RESPONSE validates the new path
  before the old path is abandoned.
- Integrated TLS 1.3: no separate security handshake.

**Fallback transport:** TCP + Noise XX handshake + yamux multiplexer, for providers behind
UDP-blocking middleboxes. libp2p selects this path automatically via multiaddr dialling. yamux
provides stream-level independence on TCP connections, matching QUIC stream semantics.

**NAT traversal — three-tier approach:**

| Tier | Protocol | Applies to | Cost |
|---|---|---|---|
| 1 | AutoNAT | All providers on connect | Classifies: public / cone-NAT / symmetric-NAT |
| 2 | DCUtR hole-punching | Cone-NAT (common on Indian home routers) | One relay round-trip + simultaneous dial |
| 3 | Circuit Relay v2 | Symmetric-NAT only | All traffic proxied; adds 50–200 ms per message |

Relay nodes are Vyomanaut-operated and co-located in Indian cloud regions to bound relay overhead.
Relay overhead vs audit response deadline must be measured before accepting symmetric-NAT
providers (Q13-1).

**Peer identity:**
Each provider daemon generates an Ed25519 key pair at installation. The public key is registered
with the coordination microservice at join time. The libp2p Peer ID is `multihash(public_key)`,
consistent with S/Kademlia's cryptographic node ID principle (Paper 03) and satisfying ADR-001's
registration-gated admission model. The key pair is the node's cryptographic identity; the
registration microservice is the admission gate.

**Kademlia DHT configuration:**

| Parameter | Value | Source |
|---|---|---|
| k-bucket size | 16 | ADR-001 (S/Kademlia disjoint path design); set at genesis |
| Parallel lookups (alpha) | 3 | libp2p default; O(log n / 3) round trips in practice |
| DHT mode | Server (all V2 desktop providers) | Full participant: stores and routes |
| Key namespace | Custom HMAC validator | `HMAC(chunk_hash, file_owner_key)` — disables default CID namespace |
| Republication | Driven by availability microservice | libp2p internal republication disabled |

The custom key validator replaces libp2p's default CID-based namespace. Only the file owner
can map a DHT lookup key back to the underlying chunk (Q07-5). This must not be overridden by
a libp2p version upgrade — pin the kad-dht namespace configuration in the daemon initialisation
path.

**0-RTT policy:**
0-RTT session resumption is disabled for all operations carrying an audit challenge response or
a signed receipt. The replay vulnerability (RFC 9000 §8.1) cannot be tolerated for these
interactions. 0-RTT may be enabled for pure chunk data transfers where replaying the stream
causes no auth or payment consequence. Latency cost of 1-RTT enforcement on audit reconnects
must be benchmarked (Q14-2).

**Pipelining:**
Multiple chunk transfer streams run in parallel over one QUIC connection, satisfying Q01-2.
libp2p connection manager watermarks: high=900, low=600 (appropriate for V2 desktop scale).

**Application protocol above the stream:**
BitSwap is not used. All data transfer is escrow-motivated direct transfer (ADR-011, ADR-012).
libp2p's connection and stream layer is reused; the application protocol above the stream is
Vyomanaut's own binary chunk transfer protocol. HTTP/3 framing overhead is not adopted.

## Consequences

**Positive:**
- NAT traversal handled by libp2p with no provider configuration required; three-tier fallback
  covers the full NAT landscape
- Connection migration at zero application cost: provider IP changes do not interrupt chunk
  transfers or require retry logic above the transport layer
- Stream independence eliminates TCP HOL blocking for parallel chunk pipelines
- Cryptographic peer identity verified at the transport handshake before any application data
  flows — a provider cannot impersonate another

**Negative / trade-offs:**
- libp2p is a significant external dependency; transport-layer bugs are outside project control
- Symmetric-NAT providers relying on Circuit Relay v2 add relay latency to every audit
  interaction; Vyomanaut must operate and maintain relay infrastructure
- Custom HMAC key validator is a DHT fork-point: must be explicitly preserved across libp2p
  version upgrades
- 0-RTT disabled for audit interactions adds one round trip per reconnect on every
  post-absence audit challenge

**Open constraints:**
- Relay overhead vs audit response deadline: measure before accepting symmetric-NAT providers
  in production (Q13-1)
- UDP-block rate at Indian ISPs: unknown; fallback TCP path must be tested under blocked-UDP
  conditions before V2 launch (Q14-1)
- 0-RTT latency penalty for audit reconnects must be benchmarked (Q14-2)
- Migration event vs JIT detection: QUIC connection migration causes response_latency_ms spike
  in audit receipts; false-positive rate must be measured empirically (Q14-3)

## References

- [Paper 14 — RFC 9000](../research/paper-14-quic-rfc9000.md): connection migration; stream multiplexing; 0-RTT replay risk; CID rotation
- [Paper 13 — libp2p](../research/paper-13-libp2p.md): NAT traversal stack; Peer ID; Kademlia config; Identify protocol; connection manager
- [Paper 03 — S/Kademlia](../research/paper-03-skademlia.md): cryptographic node IDs; k=16 disjoint path design
- [ADR-001](ADR-001-coordination-architecture.md): k=16, alpha=3; HMAC key validator; availability service republication
- [ADR-005](ADR-005-peer-selection.md): Identify protocol feeds the vetting subsystem
- [ADR-014](ADR-014-adversarial-defences.md): response_latency_ms JIT detector; migration false-positive risk
- [ADR-017](ADR-017-audit-receipt-schema.md): response_latency_ms field definition
