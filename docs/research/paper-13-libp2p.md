# Paper 13 — libp2p: Modular Peer-to-Peer Networking Stack

**Authors:** Protocol Labs (primary maintainers); community specification
**Venue / Year:** Specification — continuously maintained; reference implementation 2016–present
**Topics:** #1, #12
**ADRs produced:** None closed — [ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) partially resolved (still blocked on RFC 9000)

---

## Problem Solved

Raw TCP and UDP sockets force every P2P application to re-implement peer discovery, NAT traversal, connection security, stream multiplexing, and peer identity from scratch. libp2p is a modular networking stack extracted from IPFS that solves all five problems through composable, swappable components. For Vyomanaut V2, it answers how provider daemons establish and maintain direct P2P connections across NAT, what the production-grade Kademlia implementation looks like, and what the correct concurrency and bucket-size parameters are for our DHT — closing Q02-3 and partially closing Q04-1.

---

## Key Findings

**Peer identity:** Every node has a Peer ID derived from a public key — `multihash(public_key)`. Ed25519 is the preferred key type. This is the same principle as S/Kademlia's cryptographic node IDs (Paper 03) but implemented in production across IPFS, Filecoin, and Ethereum's devp2p.

**Multiaddress (multiaddr):** Addresses are composable protocol stacks: `/ip4/1.2.3.4/udp/4001/quic-v1/p2p/12D3Koo...`. A single peer can advertise multiple multiaddrs (IPv4, IPv6, relay) simultaneously. The dialer selects the best one.

**Transport layer:** QUIC v1 (RFC 9000) is the primary transport — it provides connection migration (survives IP changes), built-in TLS 1.3, and native stream multiplexing. TCP with Noise protocol + yamux multiplexer is the fallback. WebRTC and WebTransport are additional transports for browser contexts (irrelevant for V2 desktop daemon).

**Connection security:** TCP connections are secured with the Noise protocol (XX handshake). QUIC connections use TLS 1.3 internally. Both authenticate the remote Peer ID during handshake — a provider's identity is cryptographically verified at the transport layer.

**Stream multiplexing:** Each connection carries many independent bidirectional streams. yamux handles this for TCP; QUIC handles it natively. BitTorrent's pipelining requirement (Q01-2 — keep requests lined up) is satisfied: multiple chunk transfer streams run in parallel over one QUIC connection.

**Kademlia DHT (kad-dht):** libp2p ships a production Kademlia implementation. Key parameters: k=20 (k-bucket size), alpha=3 (parallel lookup concurrency). The DHT supports two modes — server mode (full participant, stores and routes) and client mode (queries only, does not serve as a routing intermediary). Desktop providers run in server mode; mobile would run client mode (V3 concern).

**NAT traversal — three-tier approach:**
1. AutoNAT protocol: peers ask relay nodes to probe their external reachability. Classifies the provider as publicly reachable, behind cone NAT (hole-punchable), or behind symmetric NAT (relay required).
2. DCUtR (Direct Connection Upgrade through Relay): hole-punching protocol for TCP and QUIC. A relay coordinates simultaneous dial attempts from both sides. Success rate is high for cone NAT (common on home routers). Takes one relay round-trip plus the hole-punch attempt.
3. Circuit Relay v2: fallback for symmetric NAT. All traffic proxied through an operator-run relay node. Introduces relay-node latency on every message — critical for audit deadlines (see Breaks).

**Identify protocol:** On connection establishment, peers exchange listen addresses, supported protocol IDs, Peer ID, and agent version. This is the mechanism by which a new provider announces its capabilities to the network without a separate registration call.

**Connection manager:** libp2p maintains a connection table with configurable low/high watermarks. When connections exceed the high watermark, the least-recently-used connections are trimmed. For V2 desktop daemons with hundreds of peers, default values (high=900, low=600) are appropriate.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| QUIC v1 as primary transport | TCP only | Connection migration survives IP changes; built-in mux and security; UDP may be blocked by some ISPs/firewalls — TCP fallback required |
| Noise XX handshake for TCP | TLS 1.3 for TCP | Simpler handshake; mutual authentication; not as widely recognised by ops teams as TLS |
| k=20 bucket size | k=8 (Kademlia default in paper) | Larger k improves resilience against churn at the cost of more memory and more messages per lookup |
| alpha=3 parallel lookups | Sequential lookups | Faster convergence; 3× message amplification vs sequential |
| DCUtR hole punching | Always-relay | Removes relay bottleneck for most home routers; fails for symmetric NAT; requires a rendezvous relay node to coordinate |
| Client mode for light nodes | Full DHT participation | Light nodes don't store routing entries; reduces overhead for low-uptime peers — applies to mobile V3 |

---

## Breaks in Our Case

- **libp2p Kademlia stores arbitrary values (CIDs, provider records) under content IDs** ≠ **our pseudonymous HMAC-derived chunk lookup keys (Q07-5)**
  → libp2p kad-dht must be configured with a custom key validator that accepts our HMAC(chunk_hash, file_owner_key) keys. The default CID-based namespace must not be exposed — it would leak file identity.

- **libp2p Peer IDs are derived from public keys by default** ≠ **our registration-gated model where the account UUID is the node identity (ADR-001)**
  → Our node IDs are derived from registration UUIDs, not from key-generation alone. libp2p requires a Peer ID from a key pair — the key pair should be generated at daemon installation and the public key registered with the microservice at join time. The key pair IS the node's identity; the registration is the admission gate.

- **Circuit Relay v2 proxies all traffic through a relay node** ≠ **our audit response deadline of (chunk_size / upload_speed) × 1.5**
  → A provider behind symmetric NAT that can only be reached via relay will have all audit traffic relayed. At 256 KB chunk size and 5 Mbps upload, the deadline is ~614 ms. A round trip through a relay adds 50–200 ms of relay overhead depending on geographic distance. This is tight but likely acceptable for most Indian ISP configurations. Needs measurement (Q13-1).

- **libp2p kad-dht default k=20 exceeds our ADR-001 parameter of k=8,16 from S/Kademlia** ≠ **our disjoint path design requires k=2×d where d=4,8**
  → libp2p's k=20 is a superset of our requirement. We can configure the Kademlia instance with k=16 to match ADR-001. Changing k at runtime after the network has launched is disruptive; set k=16 from genesis.

- **libp2p assumes persistent storage for DHT records (24 h TTL before republication)** ≠ **our availability service must actively republish provider records every 24 h (Paper 02 break)**
  → This is already addressed in ADR-001 — the availability service handles republication. libp2p's DHT republication can be disabled and driven externally by the availability microservice.

---

## Decisions Influenced

- **[ADR-021](../decisions/ADR-021-p2p-transfer-protocol.md) [#12 P2P Transfer — PARTIALLY RESOLVED]**
  libp2p is confirmed as the networking framework. QUIC v1 (RFC 9000) is confirmed as the primary transport. NAT traversal strategy is now concrete: AutoNAT → DCUtR → Circuit Relay v2 fallback. yamux is the multiplexer for TCP fallback connections. Peer IDs are Ed25519 key pairs registered with the microservice at join.
  ADR-021 remains Proposed until RFC 9000 is read — the QUIC-specific behaviour (connection migration mechanics, stream limits, loss recovery) must be confirmed before the full transfer protocol is locked.
  *Because:* libp2p ships a production QUIC implementation and the three-tier NAT traversal design is the industry standard. These are not choices to be designed from scratch.

- **[ADR-001](../decisions/ADR-001-coordination-architecture.md) [#1 Coordination]** `STRENGTHENED`
  libp2p's kad-dht production parameters confirm our choices: k=16 (within libp2p's configurable range), alpha=3 parallel lookups. The custom key validator mechanism confirms that pseudonymous HMAC chunk keys are achievable in libp2p's DHT without forking the library.

- **[ADR-005](../decisions/ADR-005-peer-selection.md) [#5 Peer Selection]** `STRENGTHENED`
  The Identify protocol provides the mechanism for providers to announce their capabilities (listen addresses, supported protocol versions) at connection time. This feeds directly into the vetting subsystem — a provider that fails to complete a Noise or QUIC handshake is automatically excluded before the application layer is reached.

  **[Q02-3 answered]:** libp2p Kademlia bucket size k=20 (we configure to k=16), alpha=3 parallel lookups. Branching factor in the binary XOR tree is 2; convergence is O(log n) hops with alpha=3 concurrency giving O(log n / alpha) round trips in practice.

  **[Q04-1 partially answered]:** libp2p + QUIC is the P2P transfer framework. Fully resolves when RFC 9000 is read (ADR-021).

---

## Disagreements

- **S/Kademlia (Paper 03):** recommended gRPC as the node-to-node transport.
  *Implication for us:* QUIC directly supersedes this. gRPC typically runs over HTTP/2 over TCP; QUIC provides the same multiplexing with connection migration and lower handshake latency. No disagreement in principle — QUIC is the evolution gRPC would use.

- **Paper 04 (IPFS):** IPFS's BitSwap uses want-lists over libp2p streams as the data exchange protocol.
  *Implication for us:* We do not use BitSwap. Our data transfer is escrow-motivated direct transfer (confirmed by ADR-011, ADR-012). We reuse libp2p's connection and stream layer only; the application protocol above the stream is our own chunk transfer protocol.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q13-1 and Q13-2.
