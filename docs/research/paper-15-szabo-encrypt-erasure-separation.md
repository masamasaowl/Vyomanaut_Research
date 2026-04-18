# Paper 15 — Separation and Optimization of Encryption and Erasure Coding in Decentralized Storage Systems

**Authors:** Marcell Szabó, Ákos Recse, Róbert Szabó, Dávid Balla, Markosz Maliosz — Budapest University of Technology and Economics / Ericsson Research
**Venue / Year:** Future Generation Computer Systems (FGCS), February 2025
**Topics:** #3, #9, #15
**ADRs produced:** none — ADR-019 and ADR-022 remain Proposed; see Decisions Influenced for why

---

## Problem Solved

Client-side erasure coding in OTT decentralized storage (Storj, Sia) forces every uploading device to transmit erasure-expanded data — typically 2.5–3× the original file size — over the access link before it reaches the operator's core network. For mobile users this wastes constrained radio spectrum and shortens battery life. The paper proposes separating encryption (kept on the client) from erasure coding (moved to an in-network operator proxy), reducing access-network traffic by approximately 20% while preserving zero-knowledge data privacy. For Vyomanaut, the paper's principal value is confirming that encrypt-then-code is the ordering that preserves zero-knowledge even when coding is delegated downstream — a necessary data point for resolving ADR-022 — while revealing that the full comparison with code-then-encrypt and the key-management-cost question remain open.

---

## Key Findings

**Traffic reduction:** Moving erasure coding to an operator proxy reduces uplink data volume from 254 MB to 101 MB for a 100 MB upload at a 4/10 redundancy scheme. The access link carries only the encrypted original; the expansion happens inside the network.

**Zero-knowledge preserved:** The proxy receives only ciphertext. Storage nodes receive only erasure-coded ciphertext chunks. No entity downstream of the client ever sees plaintext at any stage of the pipeline. This holds for both single-operator and federated multi-operator topologies.

**Proxy resource profile:** CPU usage scales roughly linearly with pieces generated; memory is nearly constant at ~2–2.5 GB regardless of redundancy scheme. Fewer proxies generating more pieces each is more CPU-efficient than many proxies generating few pieces.

**Encryption key granularity:** The paper states the key can be per-user or per-object (per-file), not per-chunk. With encrypt-then-code, one key encrypts the entire segment before coding; all n coded chunks share the same implicit key material.

**Large-scale simulation:** A 20% median traffic reduction is measured across Internet-scale AS topologies (500–1000 base nodes, 20,000–60,000 nodes total). The gain is highest on the second hop from the uplink; beyond hop 4, proxied and non-proxied traffic converge.

**ILP optimization:** Pre-planned "diamond" distribution paths approximate fair storage node selection within ~2% of uniform selection for small topologies and ~11% for large ones, using roughly 10% of all generated diamond paths.

---

## Trade-offs

| Chosen | Over | Consequence |
|---|---|---|
| Encrypt-then-code (encryption on client, coding in network) | Code-then-encrypt | Zero-knowledge preserved at proxy; proxy sees only ciphertext volume and metadata, never plaintext; one key per file/segment, not per chunk |
| Operator-proxy erasure coding | Client-side erasure coding | Reduces access-network load by ~20%; requires ISP cooperation — not available to OTT solutions |
| Shortest-path-tree proxy distribution | Random chunk forwarding | Lower transport cost; ILP solution runs in pre-planning phase only, not online |
| Single centralized global coordinator (Satellite-equivalent) | Decentralised coordination | Simple proxy orchestration; central DoS target — Storj's known weakness, unchanged here |

---

## Breaks in Our Case

- **Paper assumes ISP/operator cooperation — proxies reside inside the operator's transport network** ≠ **Vyomanaut V2 is an OTT solution with no operator relationships**
  → The proxy architecture is not applicable to V2. Client-side erasure coding remains the design. The paper's traffic reduction findings are a V3 consideration if mobile providers and operator partnerships are introduced.

- **Paper targets mobile users with constrained radio links** ≠ **V2 is desktop-only with ~100 Mbps symmetrical upload (ADR-009, ADR-010)**
  → The access-link bandwidth argument does not apply in V2. Desktop uplink is not the bottleneck; steady-state repair bandwidth is ~39 Kbps/peer (ADR-003), well inside the 100 Kbps background budget.

- **Paper relies on a centralised global coordinator (Satellite) for proxy orchestration** ≠ **our hybrid microservice + Kademlia DHT**
  → The proxy selection function maps to our coordinator microservice. Not a contradiction in principle, but we have no ISP proxies to orchestrate. The lesson is architectural compatibility, not direct applicability.

- **Paper validates encrypt-then-code but does not evaluate code-then-encrypt** ≠ **ADR-022 requires a direct comparison of both orderings including key management cost**
  → The paper confirms one option is viable and zero-knowledge-safe. It does not answer whether code-then-encrypt offers different security guarantees, or what the key management cost difference is between the two. ADR-022 stays Proposed.

- **Paper discusses per-file or per-user keys only** ≠ **our open question on per-chunk key derivation for stronger isolation**
  → If a provider holds s=16 fragments and the single file key, they could reconstruct the plaintext. Whether per-chunk key derivation prevents this is not addressed. ADR-019 stays Proposed.

---

## Decisions Influenced

- **[ADR-022](../decisions/ADR-022-encryption-erasure-order.md) [#15 Encryption-Erasure Interaction]** — `PARTIALLY INFORMED, REMAINS PROPOSED`
  Encrypt-then-code is confirmed as the ordering that preserves zero-knowledge: the paper's entire architecture encrypts on the client and codes downstream. Proxies and storage nodes never see plaintext regardless of where coding occurs. This eliminates one concern about encrypt-then-code — that delegating coding would break zero-knowledge — it does not. However, the paper does not compare with code-then-encrypt or measure the key management cost difference between the two orderings. ADR-022 cannot be accepted on this paper alone; the comparative analysis is still missing.
  *Because:* Only one ordering is modelled. The paper's research question was traffic reduction, not security comparison of ordering choices.

- **[ADR-019](../decisions/ADR-019-client-side-encryption.md) [#9 Client-Side Encryption]** — `PARTIALLY INFORMED, REMAINS PROPOSED`
  Client-side encryption before any data leaves the device is confirmed as the correct architectural choice. The paper's security model is built entirely on this guarantee, and it holds across single-operator and federated topologies. The specific encryption primitive (AES-GCM, ChaCha20-Poly1305, key derivation function) is not specified. ADR-019 stays Proposed pending that specification.
  *Because:* Paper focuses on where coding occurs, not which cryptographic primitive is used.

- **[ADR-003](../decisions/ADR-003-erasure-coding.md) [#3 Erasure Coding]** — `CONFIRMED`
  Reed-Solomon erasure coding remains the correct scheme. The paper uses RS as its baseline throughout. No evidence that an alternative EC scheme is required for client-side coding in a desktop V2 context. Parameters s=16, r=40, r0=8, lf=256 KB remain unchanged.
  *Because:* Paper uses RS without identifying any deficiency at our parameter scale.

- **[ADR-010](../decisions/ADR-010-desktop-only-v2.md) [#5 Peer Selection]** — `CONFIRMED`
  The proxy model's primary value is for mobile users with constrained access links. The paper independently supports the decision to defer mobile to V3: the erasure-coding-on-client cost is specifically a mobile problem. Desktop providers with 100 Mbps symmetrical links face no equivalent constraint.
  *Because:* Paper's entire motivation is mobile access link exhaustion — which does not exist at desktop V2 scale.

---

## Disagreements

- **Proxy model vs zero-knowledge purists:** The paper claims zero-knowledge is fully preserved because the proxy only sees ciphertext. This is correct for data content but the proxy necessarily sees file metadata (object ID, size, timing of uploads). A sufficiently determined proxy operator can perform traffic analysis. For Vyomanaut V2 this is irrelevant since there are no proxies, but it is a limit of the model to note if V3 operator partnerships are considered.

- **Storj's stripe-level erasure (the paper's baseline) vs our chunk-level approach:** Storj codes at the stripe level within a segment before bundling into pieces. The paper inherits this design. Our concern with stripe-level coding (from Paper 05 disagreements, cited by Szabó et al. themselves) is not addressed here — the paper focuses on where coding occurs, not at what granularity.

- **ILP pre-planning is not online:** The proxy placement and diamond-path optimisation algorithms run offline and are not suitable for per-upload decisions at scale. The 20% traffic reduction assumes pre-planned paths. Dynamic storage node selection within V3 would require a simpler heuristic rather than the full ILP.

---

## Open Questions

See [open-questions.md](open-questions.md) — questions Q15-1 through Q15-2.
