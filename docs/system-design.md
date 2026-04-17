# System Design

## Functional Requirements

What the system must do:

1. A service for users, companies, and data owners to store data
2. The data is stored inside the idle memory of phones, laptops, and devices
3. Data owner pays; storage provider earns (escrow)
4. Storage provider cannot access the stored data
5. Data must be available to the data owner at all times
6. The storage provider can alter their storage contribution
7. Storage provider can exit the network; data owner can delete the data
8. The service never stores or reads the given data

---

## Research Topics

19 active topics. Every paper and every ADR maps back to one or more of these numbers.

| # | Topic | Core question |
|---|---|---|
| #1 | Coordination Architecture | Central vs DHT balance; what stays decentralised and why; Satellite design |
| #2 | Proof of Storage | Audit mechanism: full PoRep vs random Merkle challenge vs lightweight heartbeat |
| #3 | Erasure Coding | EC scheme selection (RS / MSR / LRC); (k, m) parameters; cold vs warm tradeoffs |
| #4 | Replication / Repair Protocol | Who triggers repair; peer-to-peer chunk transfer during repair; repair bandwidth budget |
| #5 | Peer Selection Algorithm | Selection criteria: latency, uptime, geography, free space; initial assignment |
| #6 | Availability & Polling | Polling interval vs audit gap attack surface; heartbeat failure detection |
| #7 | Provider Exit State Mechanism | Accidental / announced / promised exit; grace period; migration orchestration |
| #8 | Reliability Scoring Model | EigenTrust vs rolling audit rate vs hybrid; bootstrapping new providers; decay function |
| #9 | Client-Side Encryption | Encryption scheme; key derivation per chunk vs per file; forward secrecy |
| #10 | Key Management Strategy | Owner key loss scenario; key rotation; delegation to service for republication |
| #11 | Background OS Execution | iOS/Android background execution limits; provider tier model (desktop vs mobile) |
| #12 | P2P Transfer Protocol | QUIC vs WebRTC; NAT traversal; connection migration for mobile providers |
| #13 | Escrow & Payment Basis | Per GB stored vs per audit passed vs per GB transferred; penalty structure |
| #14 | Consistency Model | Which operations need strong consistency vs eventual; invariant confluence framework |
| #15 | Encryption-Erasure Interaction | Encrypt-then-code vs code-then-encrypt; key management cost of each choice |
| #16 | Provider-Side Storage Engine | How providers store chunks locally; LSM-tree vs object store; EC tiering |
| #17 | Repair Bandwidth Optimisation | MSR/LRC regenerating codes; repair parallelisation; helper node selection |
| #18 | Economic Mechanism Design | Game-theoretic incentive compatibility; credit systems; SLA enforcement |
| #19 | Adversarial Provider Behaviour | Provider deletes data between audits; false audit responses; collusion modelling |
