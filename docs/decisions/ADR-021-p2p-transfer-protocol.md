# ADR-021 — P2P Transfer Protocol

**Status:** Proposed — blocked on reading-list Phase 1 #8 and #9
**Topic:** #12 P2P Transfer Protocol
**Supersedes:** —
**Superseded by:** —
**Research source:** TBD — libp2p spec + RFC 9000

---

## Context

All data movement between providers and between providers and data owners is pure P2P. The transfer protocol must handle NAT traversal (most providers are behind home routers), connection migration (providers may change IP), and multiplexed streams (multiple chunk transfers in parallel). S/Kademlia (Paper 03) suggested gRPC or QUIC between nodes.

## Blocked on

- **libp2p Specification & Architecture** (reading-list Phase 1 #8): Kademlia implementation in production + NAT traversal strategies
- **RFC 9000 — QUIC** (reading-list Phase 1 #9): connection migration behaviour; multiplexed streams; UDP-based transport

## Known constraints (from existing papers)

- BitTorrent confirmed: keep requests pipelined so the transport layer is always loaded (Q01-2)
- S/Kademlia: nodes communicate via gRPC or QUIC (self-decision, Paper 03)
- NAT traversal is required — most home desktop providers are behind NAT
- Connection migration is required — providers may have dynamic IPs
- QUIC's connection migration (based on connection ID, not IP/port tuple) is the primary candidate for handling IP changes without dropping transfers in progress

## Current candidate

libp2p for peer discovery and connection management. QUIC (RFC 9000) as the transport layer for data transfer. NAT traversal via libp2p's built-in hole-punching strategies.

## Decision

**Not yet made.** Fill this ADR after reading libp2p spec and RFC 9000.

## References

- [Paper 03 — S/Kademlia](../research/paper-03-skademlia.md): QUIC/gRPC suggestion
- [open-questions.md Q04-1](../research/open-questions.md): P2P replication protocol open question
