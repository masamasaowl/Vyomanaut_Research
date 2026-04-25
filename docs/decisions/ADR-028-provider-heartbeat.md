# ADR-028 — Provider Heartbeat: 4-Hour Address Re-announcement to Microservice

**Status:** Accepted
**Topic:** #6 Availability & Polling, #1 Coordination Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** ADR-006 (polling interval), ADR-021 (libp2p + QUIC), Paper 20 — IPFS
Measurement (DHT record republication parameters), Paper 28 — Rhea et al. (failure detection
and per-peer RTO), Paper 30 — Trautwein et al. (NAT traversal; 70% hole-punch success)

---

## Context

[ADR-021](./ADR-021-p2p-transfer-protocol.md) specifies QUIC connection migration for handling IP changes on **active connections**.
But audit challenges are cold-dialed every 24 hours — there is no active connection to migrate.

A provider with a dynamic DHCP lease (standard on Indian residential ISPs: BSNL and Airtel
home broadband rotate leases on 24-hour cycles) will have a different IP at challenge time
than the multiaddr stored in the DHT or the microservice's last known address.

The libp2p Identify protocol (specified in [Paper 13](../research/paper-13-libp2p.md) / ADR-021) fires only on connection establishment. It does not push address updates proactively. A provider whose IP changed overnight has no mechanism to update their DHT multiaddr until they initiate a new outbound
connection — which does not happen in the cold-challenge model.

**Failure scenario without this ADR:**
A provider is online and storing data correctly. Their ISP rotates their DHCP address at 03:00.
The microservice sends the 24-hour challenge to the stale address at 07:00. The challenge times
out. The provider's 24h window score decrements. Over a week of nightly IP rotations, the 7-day
score falls below the partial hold threshold in [ADR-024](./ADR-024-economic-mechanism.md). The provider begins losing earnings
despite being consistently available. They perceive this as a payment bug and churn. Financial
friction, the mechanism that reduces churn, is being triggered by an address management failure.

At Indian ISP scale, this is not an edge case: [Paper 30](../research/paper-30-trautwein-dcutr-nat.md) (Trautwein) measured that ~30% of P2P
peers require relay precisely because their addresses are not reliably reachable from outside.
Among Indian residential providers specifically, CGNAT prevalence is high (Q20-1). The DHT
republication interval (12 hours, [ADR-006](./ADR-006-polling-interval.md)) is insufficient to guarantee the microservice has a
fresh address before the 24-hour challenge fires.

---

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Rely on DHT republication (12h interval) | No change required | 12h window between republication and challenge means stale address is possible; DHT republication does not guarantee the microservice picks up the update |
| Force outbound connection from provider to microservice every 24h | Simple | Still a 24h window; does not solve the problem for DHCP rotations within the polling cycle |
| **4-hour signed heartbeat to microservice control plane** | Freshest known address is always < 4h stale; microservice uses this for challenge dispatch; decoupled from DHT | Adds a persistent outbound daemon connection requirement; slight bandwidth overhead |

---

## Decision

### 1. Heartbeat protocol

The provider daemon sends a signed heartbeat to the microservice control-plane REST endpoint
every **4 hours**:

```
POST /api/v1/provider/heartbeat

{
  "provider_id":        "UUID",
  "current_multiaddrs": ["/ip4/1.2.3.4/udp/4001/quic-v1/p2p/12D3Koo...",
                          "/ip4/192.168.1.2/udp/4001/quic-v1/p2p/12D3Koo..."],
  "timestamp":          "2026-04-25T07:00:00Z",
  "daemon_version":     "0.1.0",
  "provider_sig":       "<Ed25519 over all fields above, hex>"
}
```

The microservice:
1. Verifies the Ed25519 signature against the provider's registered public key
2. Updates `providers.last_known_multiaddrs` (JSON array) and `providers.last_heartbeat_ts`
3. Responds with HTTP 200 and a signed acknowledgement:
   ```json
   { "received_at": "2026-04-25T07:00:01Z", "microservice_sig": "..." }
   ```

Multiple multiaddrs (IPv4, IPv6, relay) are accepted; the microservice attempts them in order
during challenge dispatch.

### 2. Challenge dispatch uses heartbeat address, not DHT

The audit challenge scheduler uses `providers.last_known_multiaddrs` from Postgres as the
**primary** address source. DHT lookup is a **fallback** path, used only when:
- No heartbeat has been received within 8 hours, AND
- A challenge must be issued (repair pre-warning or scheduled audit)

This decouples challenge reliability from DHT convergence latency. It also means that
providers behind symmetric NAT who rely on Circuit Relay v2 ([ADR-021](./ADR-021-p2p-transfer-protocol.md)) can report their relay
address in the heartbeat, ensuring the microservice always dials via the correct relay.

### 3. Heartbeat interval rationale

**4 hours** is chosen because:
- Indian residential DHCP leases rotate on 24-hour cycles. A 4-hour interval guarantees a
  refresh within 4 hours of any IP change — at worst a 4-hour stale window before the next
  heartbeat, always resolved before the 24-hour audit challenge fires.
- Bandwidth overhead: heartbeat payload ≈ 200 bytes × 6 per day = 1.2 KB/day per provider.
  At N=10,000 providers this is 12 MB/day of microservice ingress — negligible.
- Battery / CPU impact on desktop: 6 tiny HTTPS POST requests per day. Zero measurable impact.

### 4. Missed heartbeat handling

| Consecutive missed heartbeats | Time elapsed | Action |
|---|---|---|
| 1 | 4–8 hours | No action. Within nightly absence window. |
| 2 | 8–12 hours | Mark `providers.multiaddr_stale = true`. Fall back to DHT lookup on next challenge. |
| 3 | 12–16 hours | Attempt DHT lookup for current address. Log address resolution failure if DHT also stale. |
| 18+ | 72 hours | Standard ADR-007 departure threshold: declare silent departure, trigger repair. |

The 72-hour boundary is unchanged from [ADR-006](./ADR-006-polling-interval.md) and [ADR-007](./ADR-007-provider-exit-states.md). Missed heartbeats produce a
degraded address resolution path, not a premature departure declaration.

### 5. Heartbeat and the reliability score

Missed heartbeats do **not** directly decrement the reliability score (ADR-008). Only audit
challenge results (PASS / FAIL / TIMEOUT) affect the score. A provider with a stale address
will accumulate TIMEOUT results if the microservice cannot reach them — the score reflects
actual challenge failures, not the heartbeat state.

This preserves the design intent of ADR-008: the score is driven by audit evidence, not
connectivity signals. The heartbeat is an address management mechanism, not a presence
verification mechanism.

### 6. Schema change

```sql
-- Add to providers table:
last_known_multiaddrs  JSONB         NOT NULL DEFAULT '[]',
last_heartbeat_ts      TIMESTAMPTZ,
multiaddr_stale        BOOLEAN       NOT NULL DEFAULT false
```

The `last_known_multiaddrs` column is an ordered JSON array. The assignment service always
tries the first (most recently confirmed) address first during challenge dispatch.

---

## Consequences

**Positive:**
- Eliminates false TIMEOUT audit results caused by stale multiaddrs from DHCP rotations
- Challenge dispatch is now decoupled from DHT convergence latency
- Relay addresses for symmetric-NAT providers are explicitly tracked in the control plane
- The heartbeat provides a lightweight liveness signal that the operations team can monitor
  without waiting for the 24-hour audit cycle

**Negative / trade-offs:**
- The daemon must maintain a persistent 4-hour timer and an outbound HTTPS connection to the
  microservice control-plane endpoint — this is a small but real operational requirement
- Providers who disable the daemon for more than 8 hours will have stale multiaddr state;
  this is correct behaviour (they are absent) but requires the fallback DHT path to be
  implemented and tested
- A 4-hour stale window remains: a provider whose IP changes in the 3h 59m since their last
  heartbeat may still receive one stale challenge. This is acceptable — one TIMEOUT per IP
  rotation is not a meaningful score impact at the 24h/7d/30d window scale of ADR-008.

**Open constraints:**
- The microservice control-plane heartbeat endpoint must be load-balanced separately from the
  audit challenge endpoints to prevent heartbeat storms (all providers reconnecting
  simultaneously after a microservice restart) from affecting audit throughput
- Heartbeat interval (4 hours) is a starting value; it may be reduced to 2 hours if Q20-1
  telemetry shows that CGNAT rotation frequency in Indian ISPs is shorter than 24 hours

---

## References

- [ADR-006](ADR-006-polling-interval.md): 24-hour polling interval; DHT republication at 12h — the mismatch this ADR resolves
- [ADR-007](ADR-007-provider-exit-states.md): 72-hour departure threshold — the missed-heartbeat escalation path terminates here
- [ADR-021](ADR-021-p2p-transfer-protocol.md): libp2p Identify protocol; QUIC connection migration; three-tier NAT traversal; relay addresses
- [ADR-008](ADR-008-reliability-scoring.md): reliability score is audit-evidence-driven, not heartbeat-driven — preserved by this ADR
- [Paper 20 — IPFS Measurement](../research/paper-20-trautwein-ipfs.md): 12h republish / 24h expiry DHT parameters; fire-and-forget DHT RPCs have non-trivial failure rates — motivates not relying on DHT for challenge dispatch
- [Paper 28 — Rhea et al.](../research/paper-28-rhea-dht-churn.md): per-peer TCP-style RTO for failure detection; addresses the micro-level timeout used once a heartbeat has given the microservice a fresh address to dial
- [Paper 30 — Trautwein et al.](../research/paper-30-trautwein-dcutr-nat.md): ~30% of peers require relay; Indian CGNAT prevalence expected to push relay fraction above global baseline (Q20-1) — relay addresses must be explicitly tracked per provider
