# Dropped Questions

These questions were removed from the tracker. Each entry states the specific reason.
A question is dropped only when it meets one of three criteria:

1. **Resolved by ADR** — an existing accepted ADR already contains the answer and there is nothing left to decide.
2. **No longer relevant** — the question was about a design path that was rejected; the question has no residue in the current architecture.
3. **Duplicate** — the question is covered more precisely by another open question.

These are not archived for reference; they are recorded here so future contributors understand why they are not in the tracker.

---

**Q04-3** — "How to use IPNS for file routing if the data owner is not part of the network?"
*Dropped: no longer relevant.* This question assumed the data owner needed a DHT presence to route file lookups. The current design uses the pointer file (held by the data owner) and the microservice's chunk location table for retrieval. IPNS is not part of the architecture. The question has no residue.

**Q05-5** — "What is the minimum k' for Berlekamp-Welch random stripe audits?"
*Dropped: about a rejected mechanism.* Berlekamp-Welch auditing was considered (Paper 05) and rejected in favour of PoR Merkle challenges (ADR-002). This question is about optimising the audit mechanism that was not chosen. No residue.

**Q05-8** — "Should per-segment pricing be adopted to cover metadata overhead?"
*Dropped: resolved by ADR.* ADR-024 explicitly states per-segment pricing is not adopted in V2 with reasoning: per-chunk metadata overhead (~44 bytes per chunk in the WiscKey index) is negligible at anticipated volumes. Revisit if segment count grows to the point where metadata cost is measurable. There is nothing to decide; the ADR closes the question.

**Q12-1** — "At what microservice cluster size does hand-rolled gossip become harder to operate than etcd/Consul?"
*Dropped: resolved by ADR.* ADR-025 already contains the answer: "if the cluster ever grows beyond 5 replicas, evaluate etcd." The threshold is specified. There is no open decision.

**Q16-2** — "If the pointer file is the sole retrieval credential, what is the recovery path if lost?"
*Dropped: resolved by ADR.* ADR-020 fully specifies: (1) microservice stores the encrypted pointer file ciphertext; (2) data owner re-derives the pointer file encryption key from master_secret + file_id via HKDF and decrypts the stored ciphertext; (3) if master_secret is also lost, a BIP-39 mnemonic generated at account creation is the offline recovery path; (4) if both are lost, permanent data loss — disclosed at onboarding. The question is closed.

**Q17-2** — "Should the pointer file nonce counter be stored in the daemon's local DB or in the pointer file itself? What happens after device restore with a stale counter?"
*Dropped: resolved by ADR.* ADR-019 and ADR-020 together specify: nonce counter is stored in the owner's local key store alongside the encrypted signing key. On device restore from backup with a stale counter, the daemon enforces incrementing the counter past the last known value before re-encrypting any pointer file. The recovery procedure is specified. Nothing remains open.

**Q10-4** — "How does splitting alpha (failure rate) into temporary + permanent components change the optimal r?"
*Dropped: resolved by research reasoning.* Paper 21 (Saroiu) provides the worst-case α_temporary floor (median unincentivised session ~60 minutes). In Vyomanaut's financially-incentivised model, α_temporary is orders of magnitude lower, making the conservative Giroire assumption (treating all departures as permanent) safely overestimate repair bandwidth. The resulting r=40 is robust to any alpha split. ADR-003 is accepted. The formal Dimakis proof that was originally cited as blocking this question is not needed — the conservative direction is safe and the ADR is locked. Nothing remains to decide.