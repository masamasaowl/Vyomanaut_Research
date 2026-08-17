# Vyomanaut V2 — LTS Build Plan (Part 4)

**Track:** LTS. Nothing in this document applies to the demo, which is frozen at `demo-v1.0.0`.

**Relationship to Parts 1–3.** `build.md`, `build_part2.md` and `build_part3.md` are the **demo
track**, complete at Milestone 18. This document begins at Milestone 19 and continues the milestone
numbering, but it is a **clean start**: no demo functionality is carried forward as a decision. The
demo repository is inherited as a starting point only, and `docs/LTS-INHERITANCE.md` (Session
19.0.1) records exactly what that means package by package.

**Governing decisions:** ADR-062 (two-track split), ADR-063 (demo substitutions and the LTS reversal
obligation), ADR-065–068 (launch gates, metric grammar, alert↔runbook bijection, relay telemetry),
ADR-069 (scale simulation).

### The LTS milestone map

**Milestone 19 is fixed. Everything after it is provisional** and will be renumbered and reordered
according to what the demo runs (M18 Phase 18.2) and the `reading-list.md` research produce. The
ordering below is a dependency argument, not a schedule; the *sequence* is the claim, not the
numbers.

> **Revision note — August 2026 Design Council.** Three changes below, each with an ADR behind it.
> **(a) Confidentiality moves ahead of Proof of Storage.** ADR-076 (F-LTS-08) found that
> `cmd/microservice` reconstructs the AONT package on every repair — the operator holds plaintext
> by design, today, with no attacker. Council 2 further established that *no* assignment of the
> repair role under RS + AONT-RS leaves every party blind, which makes F-69 a structural result
> rather than a defect to relocate. Confidentiality is no longer a milestone that can wait behind
> the audit primitive. **(b) Repair & Erasure Optimisation is removed as a milestone.** ADR-076
> resolves Q26-4 from the code and ADR-026 closes as *no V3 repair-bandwidth optimisation*; what
> survives (small-object policy, redundancy adaptation) is not milestone-sized. **(c) Repair
> Topology enters as a new milestone** — ADR-076's `r0` gate and provider-side executor. The
> research preconditions column is new and is the reading list's half of the contract.

| Milestone | Theme | Depends on | Research precondition | Why it sits here |
| --- | --- | --- | --- | --- |
| **19 — LTS Foundation** | Real `go-libp2p`, real `klauspost/reedsolomon` | `demo-v1.0.0` | **None** — deliberately. Do R-61's wire-format audit here while the interfaces are open | Everything measurable depends on it. CGNAT fraction, relay slot capacity, DCUtR success, NFR-006's relay budget — all unmeasurable until the real stack is in place |
| *(next)* **Confidentiality** ⬆ | Domains P & K — F-69, F-34 | 19 | **R-27 → R-29** (P), **R-17** then **R-30/R-31** (K). R-17 is a feasibility gate: < 8 independent ASNs and cap-tuning is off the table | **Moved ahead of Proof of Storage.** F-LTS-08: the operator decodes every repaired segment today. Council 2's three-repairers result makes this structural, not a bug — and ADR-022-A withdrew the ASN cap's confidentiality claim, leaving the property with no mechanism at all |
| *(next)* **Repair Topology** `[NEW]` | ADR-076 — the `r0` gate, provider-side executor | 19 | None to *start*; Domain P's substitution tests are defined against the post-gate repair frequency, so it runs **alongside** Confidentiality | ADR-004's ratified trigger flow was never built and its P2P constraint is violated. The gate alone removes a 38× bandwidth penalty and converts repair from constant to rare, which is what makes Domain P's mechanisms worth evaluating |
| *(next)* **Proof of Storage** | ADR-059, ADR-060 | 19, Repair Topology | **R-47 gating** (Band 0, no search needed), **R-50 hard dependency** (price `drand` first), then R-48. Blocked on F-LTS-11 | **F-32: the audit currently cannot fail.** Q68-3 lost its cheapest option to ADR-076 — the microservice no longer holds the reconstructed bytes — so R-47 is now a precondition, not a background read |
| *(next)* **Population Re-basing** | Domain D `[STALE]` | 19 | **R-12 → R-15.** Sequence **before** Domain U, or U derives well-founded answers from a stale distribution | The 72 h departure threshold, 24 h polling and bandwidth budget rest on Bolosky 2000, Saroiu 2002, Blake & Rodrigues 2003. That population no longer exists |
| *(next)* **Assumption Retirement** `[NEW]` | Domains U, V, O, W | Population Re-basing (for U) | **R-65 → R-67** (U), **R-68 → R-70** (V), **R-56 → R-58, R-71** (O), **R-59, R-72** (W) | Fourteen parameters across nine ADRs are `[UNDERIVED]` (ADR-077). Domain U alone retires five of them from one body of theory. Runs in parallel from here; it is not a gate on anything, and everything is slightly wrong until it lands |
| *(next)* **Audit at Fleet Scale + Transparency Log** | Domains B & G | Proof of Storage | **R-23 → R-25**; R-50 already landed upstream | Fortress is a dependency for two open questions, not one. Q70-1 resolved **yes** — the beacon is not Fortress's price, it is the fix for a seed-grinding attack ADR-060 leaves open |
| *(next)* **Payments Hardening** | ADR-011/012/016/024/035/061 | Proof of Storage | **R-43 → R-46.** R-63 only if the mid-period handover count exceeds ~1% | Real Razorpay, real money. Deliberately after the audit primitive — releasing against an audit that cannot fail is a financial hole, not a bug. F-03 is closed (ADR-012-A), so the receipt schema is unblocked |
| *(next)* **Production Hardening** | *(relocated from the old M17)* ADR-025, ADR-027, ADR-048; IC §8 | 19 | **R-26, R-32, R-33** (L), **R-34 → R-37** (M). R-32 is the high-value item | Secrets-manager adapters, HA gossip, `cmd/relay`, real client-driven routing. Needs real libp2p to be meaningful |
| *(next)* **Observability & Launch Gates** | ADR-065, 066, 067, 068 | Production Hardening | **Domain O** — and note all four ADRs cite `This review` as their entire research source (ADR-077 §Context) | All four have subjects to instrument only once relays and the HA cluster exist |
| **28 — Desktop GUI Shell** | ADR-038–052 | Observability | ADR-038–052 cluster; low priority, correctly | **Confirmed at M28 per your ruling.** Eight phases, shared component library **before** the app shells |
| *(next)* **Scale Simulation** | ADR-069; `architecture.md §27.5/§27.7` | 19, Production Hardening, Observability | **R-54, R-55** (S). **Blocked on F-LTS-03** — the synthetic tier cannot compute `σ` without ADR-059's authenticators | 10,000+ nodes. Only meaningful after real libp2p, real relays, and telemetry to observe. A number produced without Domain S is a number that cannot be defended |
| *(next)* **GA Launch Readiness** | *(relocated from the old M18)* | all | **R-51 → R-53** (Q) — runs continuously across every milestone above, not just this one | Runbooks, benchmark suite, security verification checklist, full CI gate. ADR-070 found thirteen defects by running one seam once; F-LTS-07 and F-LTS-08 are two more of the same class |

**Four orderings worth defending explicitly.** Payments sits after the audit primitive because
releasing escrow on the strength of an audit that cannot fail is a financial exposure, not merely an
incomplete feature. Scale simulation sits near the end because a simulation run before real libp2p
measures the substitute — and a number that looks like data but is not is worse than no number.
**Confidentiality now sits first among the research milestones** because it is the only one where
the failure is happening continuously in the current implementation rather than waiting to happen:
every repair event hands the operator a decodable package. And **Population Re-basing precedes
Assumption Retirement** because Domain U's timeout derivations consume an absence distribution —
deriving `t = 24 h` rigorously from Bolosky 2000 would replace a guess with a well-founded answer to
the wrong question.

**One milestone was removed.** Repair & Erasure Optimisation is gone: ADR-076 settles Q26-4 from the
code, ADR-026 closes as *no V3 repair-bandwidth optimisation*, and Papers 62 and 64 leave the
drafting queue. What survives — R-11 (small objects) and R-64 (redundancy adaptation) — folds into
whichever milestone touches stripe construction.

---

## Milestone 19 — LTS Foundation: Dependency Reality

**Deliverable:** an LTS repository whose network and erasure layers are the ones the architecture
documents describe — real `go-libp2p`, real `klauspost/reedsolomon` — behind unchanged interfaces,
with every substituted claim in ADR-063 either restored or explicitly re-scoped.

**Dependency:** `demo-v1.0.0` → **M19** → the Proof of Storage milestone.
**Reference:** ADR-062, ADR-063 (reversal table), ADR-021, ADR-001, ADR-003, IC §4, §5.2, §5.4, §12,
ARCH §13; the Design Council verdict on Q-L-1.
**Sessions:** 7.

**Precondition on the environment — one command, per the council verdict:**

```bash
curl -sI https://proxy.golang.org/github.com/libp2p/go-libp2p/@v/list | head -1
```

If this returns `200`, the restriction described in `internal/p2p/doc.go` was a property of the
authoring sandbox and does not apply. If it does not, characterise the restriction before designing
around it. **The council's verdict is that CI is the authoritative LTS build environment**, so a
restricted developer machine is a developer-experience problem, not a milestone blocker.

---

### Phase 19.0 — Fork and inherit

#### Session 19.0.1 — Fork, classify, and set the LTS CI policy

**PRECONDITIONS** — `demo-v1.0.0` tagged. ADR-062 accepted. Egress check above run and recorded.

**TASK**

1. Fork `Vyomanaut_LTS` from `demo-v1.0.0`. **Delete `vendor/` in the first commit** — council
   recommendation 2: `vendor/` is prohibited on the LTS track.
2. Create `docs/LTS-INHERITANCE.md` classifying every `internal/` and `cmd/` package into exactly
   one of: **Carried**, **Reversal-pending** (ADR-063's table), **Demo-gated** (production-gated,
   not extended — `cluster.SoloMembership`, `envSecretsClient`, `payment.MockProvider`), **Discard**.
3. Two new LTS CI checks, both from the council verdict:
   - **no `vendor/` directory exists**
   - **no unverified module checksums** — every module proxy-fetched with a `sum.golang.org`
     checksum; hand-repackaged module zips are prohibited on this track.
4. Raise CI check 10's ADR ceiling to the LTS series and add ADR-062 §3's track-tag check: every ADR
   from 062 onward carries a `**Track:**` line.
5. Copy `docs/DEMO.md` in read-only as `docs/inherited/DEMO.md` — the record of what the starting
   point did and did not prove, including its module-provenance disclosure.
**FILES** — `docs/LTS-INHERITANCE.md`, `docs/inherited/DEMO.md`, `scripts/ci/lts_policy.sh` (new),
`scripts/ci/grep_checks.sh` (edit), `.github/workflows/ci.yml` (edit)

**VERIFY**

```bash
FORKED_AT_THE_DEMO_TAG:
  $ git merge-base --is-ancestor $(git rev-list -n1 demo-v1.0.0) HEAD && echo PASS
  EXPECT: PASS
 
VENDOR_PROHIBITED_ON_LTS:                   # council recommendation 2
  $ test -d vendor && echo FAIL || echo PASS
  EXPECT: PASS
  $ grep -c "vendor" scripts/ci/lts_policy.sh
  EXPECT: >= 1
 
NO_HAND_BUILT_MODULE_CHECKSUMS:             # council recommendation 4
  $ GOFLAGS=-mod=mod GONOSUMDB= GONOSUMCHECK= go mod verify
  EXPECT: all modules verified
  $ bash scripts/ci/lts_policy.sh
  EXPECT: exit 0
 
EVERY_PACKAGE_CLASSIFIED:
  $ for d in internal/*/ cmd/*/; do grep -q "$(basename $d)" docs/LTS-INHERITANCE.md \
      || echo "UNCLASSIFIED $d"; done
  EXPECT: no UNCLASSIFIED lines
 
FOUR_CLASSES_USED:
  $ grep -cE "Carried|Reversal-pending|Demo-gated|Discard" docs/LTS-INHERITANCE.md
  EXPECT: >= 4
 
TRACK_TAG_CHECK_REGISTERED:                 # ADR-062 §3
  $ grep -c "TestEveryADRHasTrackTag" scripts/ci/grep_checks.sh
  EXPECT: >= 1
  $ bash scripts/ci/grep_checks.sh
  EXPECT: exit 0
 
DEMO_RECORD_PRESERVED:
  $ test -f docs/inherited/DEMO.md && grep -c "hand-repackaged" docs/inherited/DEMO.md
  EXPECT: >= 1
```

---

### Phase 19.1 — Restore the P2P stack

#### Session 19.1.1 — `go-libp2p` behind the `Host` contract

**PRECONDITIONS** — 19.0.1 complete.

**TASK**

1. Add `github.com/libp2p/go-libp2p` and `github.com/multiformats/go-multiaddr`.
2. Reimplement `internal/p2p/host.go` against `libp2p.New` with **QUIC v1 transport and Noise XX**
   (ADR-021, IC §4) — restoring the mechanism NFR-016 names.
3. **The exported surface does not change.** `PeerID`, `ProtocolID`, `Multiaddr`, `Stream`,
   `StreamHandler`, `Connect`, `NewStream`, `SetStreamHandler` keep their signatures.
   `internal/p2p/doc.go` claims these were designed to be drop-in-replaceable; this session is the
   test of that claim. **If any signature must change, stop and escalate** — it means the demo's
   interfaces diverged from libp2p's in a way nobody recorded, which is a finding, not a refactor.
4. Restore the real 0-RTT policy: `zeroRTTProhibited`'s three protocol IDs enforced against QUIC
   0-RTT rather than TLS session tickets. **The deny-list itself is unchanged** —
   `/vyomanaut/chunk-upload/1.0.0` stays deliberately absent (IC §4.1).
5. Delete `peerid.go`'s hand-rolled derivation; use `peer.IDFromPublicKey`. Add a test asserting the
   new output is byte-identical to the demo's for a fixed key — which validates the demo's
   spec-exactness claim retroactively.
**FILES** — `internal/p2p/host.go` (rewrite), `internal/p2p/peerid.go` (rewrite),
`internal/p2p/types.go` (edit), `go.mod`, `internal/p2p/host_test.go`

**VERIFY**

```bash
LIBP2P_IS_A_REAL_DEPENDENCY:
  $ grep -c "github.com/libp2p/go-libp2p" go.mod
  EXPECT: >= 1
  $ go build ./internal/p2p/
  EXPECT: exit 0
 
QUIC_AND_NOISE_RESTORED:                    # NFR-016, ADR-021
  $ grep -cE "libp2pquic|quic.NewTransport|transport/quic" internal/p2p/host.go
  EXPECT: >= 1
  $ grep -cE "noise.New|transport/noise" internal/p2p/host.go
  EXPECT: >= 1
 
STDLIB_TLS_TCP_SUBSTITUTION_GONE:
  $ grep -cE "crypto/tls|crypto/x509|VerifyPeerCertificate" internal/p2p/host.go
  EXPECT: 0
 
EXPORTED_SURFACE_UNCHANGED:                 # doc.go's drop-in claim, tested
  $ git diff demo-v1.0.0 -- internal/p2p/types.go \
      | grep -cE "^-.*type (PeerID|ProtocolID|Multiaddr|Stream|StreamHandler)"
  EXPECT: 0
 
ZERO_RTT_DENYLIST_PRESERVED_EXACTLY:        # IC §4
  $ grep -cE "audit-challenge|repair-download|vetting-gc" internal/p2p/host.go
  EXPECT: >= 3
  $ grep -c "chunk-upload" internal/p2p/host.go
  EXPECT: 0
 
PEERID_BYTE_IDENTICAL_TO_DEMO:
  $ go test -run TestPeerIDMatchesFrozenDemoVector ./internal/p2p/
  EXPECT: exit 0
 
UNIT_TESTS:
  $ go test -race -v ./internal/p2p/
  EXPECT: exit 0; tests include:
    TestHostUsesQUICTransport
    TestNoiseXXHandshakeAuthenticatesPeerID
    TestZeroRTTRefusedForProhibitedProtocols
    TestZeroRTTPermittedForChunkUpload
    TestPeerIDMatchesFrozenDemoVector
 
VET:
  $ go vet ./internal/p2p/
  EXPECT: exit 0; zero output
```

---

#### Session 19.1.2 — `go-libp2p-kad-dht` behind the IC §12 validator

**PRECONDITIONS** — 19.1.1 complete.

**TASK**

1. Add `github.com/libp2p/go-libp2p-kad-dht` and `go-libp2p-record`.
2. Replace the from-scratch Kademlia at `k=16`, `α=3`, Server mode (ARCH §13).
3. **Register the existing `validateDHTKey` as a `record.Validator`, unchanged.** ADR-063 records
   that it implements IC §12's accept/reject rules byte-for-byte; this session re-hosts it, it does
   not rewrite it. `TestDHTKeyValidatorPersists` becomes meaningful for the first time — its
   documented trigger (a `go-libp2p` upgrade) finally has a subject.
4. Preserve IC §12.2's republication contract and §12.3's stale-address fallback.
**FILES** — `internal/p2p/dht.go` (rewrite), `internal/p2p/dht_namespace.go` (edit), `go.mod`,
`internal/p2p/dht_test.go`

**VERIFY**

```bash
REAL_KAD_DHT_DEPENDENCY:
  $ grep -c "go-libp2p-kad-dht" go.mod
  EXPECT: >= 1
 
VALIDATOR_LOGIC_UNCHANGED_FROM_DEMO:        # the piece that was already correct
  $ git diff demo-v1.0.0 -- internal/p2p/dht.go | grep -cE "^-.*func validateDHTKey"
  EXPECT: 0
 
VALIDATOR_REGISTERED_WITH_LIBP2P:
  $ grep -cE "record.Validator|Validator:.*validateDHTKey" internal/p2p/dht.go
  EXPECT: >= 1
 
KADEMLIA_PARAMS_PER_ARCH13:
  $ grep -cE "BucketSize\(16\)|Concurrency\(3\)|ModeServer" internal/p2p/dht.go
  EXPECT: >= 3
 
FROM_SCRATCH_ROUTING_GONE:
  $ grep -ciE "k-bucket|kbucket\[|iterativeLookup" internal/p2p/dht.go
  EXPECT: 0
 
CI_CHECK_5_NOW_HAS_ITS_SUBJECT:             # N-05 closed on the LTS track
  $ go test -run TestDHTKeyValidatorPersists ./internal/p2p/
  EXPECT: exit 0
  $ grep -c "TRACK: LTS" internal/p2p/dht_test.go
  EXPECT: 0                                  # the demo-track restatement is removed; the real rationale applies
 
REPUBLICATION_AND_FALLBACK_PRESERVED:       # IC §12.2, §12.3
  $ go test -race -v -run "TestDHTRepublication|TestStaleAddressFallback" ./internal/p2p/
  EXPECT: exit 0
```

---

#### Session 19.1.3 — AutoNAT, DCUtR, Circuit Relay v2

**PRECONDITIONS** — 19.1.1, 19.1.2 complete.

**TASK**

1. Replace `nat.go`'s from-scratch three-tier stack with libp2p's: `p2p/host/autonat`,
   `p2p/protocol/holepunch` (DCUtR), `p2p/protocol/circuitv2/client`.
2. Restore **Circuit Relay v2 reservations** per IC §4.3 — the mechanism NFR-006's 50 ms budget is
   defined against, and which the demo never had.
3. `cmd/relay` — a real relay binary, replacing the `deployments/dev/docker-compose.yml` placeholder.
4. **Measure, for the first time:** relay reservation RTT overhead (NFR-006, ≤ 50 ms from Indian
   cloud nodes), DCUtR hole-punch success rate, and the **observed reservation slot limit per node**
   — which `architecture.md §27.5` currently *assumes* is libp2p's default 128, and on which the
   entire relay capacity model rests.
5. Compare against the five-desktop rig's **O-6** result: the demo's reachability probe correctly
   avoided the relay tier on a flat subnet. Confirm AutoNAT reaches the same conclusion.
**FILES** — `internal/p2p/nat.go` (rewrite), `cmd/relay/main.go`,
`deployments/dev/docker-compose.yml` (edit), `docs/measurements/m19-nat-baseline.md`

**VERIFY**

```bash
ALL_THREE_LIBP2P_NAT_COMPONENTS:
  $ grep -cE "host/autonat|protocol/holepunch|circuitv2/client" internal/p2p/nat.go
  EXPECT: >= 3
 
FROM_SCRATCH_NAT_GONE:
  $ grep -ciE "simultaneous.?open|self.?reachability probe|vyomanaut-only relay" internal/p2p/nat.go
  EXPECT: 0
 
RELAY_BINARY_REAL_NOT_PLACEHOLDER:
  $ go build ./cmd/relay/
  EXPECT: exit 0
  $ grep -c "placeholder" deployments/dev/docker-compose.yml
  EXPECT: 0
 
NFR006_MEASURED_FOR_THE_FIRST_TIME:
  $ python3 -c "import re;t=open('docs/measurements/m19-nat-baseline.md').read(); \
      m=re.search(r'relay overhead[^0-9]*([0-9.]+) ?ms',t,re.I); \
      print('PASS' if m and float(m.group(1))<=50 else 'CHECK: '+(m.group(1) if m else 'unmeasured'))"
  EXPECT: PASS
 
RESERVATION_SLOT_LIMIT_MEASURED_NOT_ASSUMED:    # architecture.md §27.5 assumes 128
  $ grep -cE "reservation slots|slot limit" docs/measurements/m19-nat-baseline.md
  EXPECT: >= 1
 
HOLEPUNCH_SUCCESS_RATE_RECORDED:
  $ grep -ciE "hole.?punch success" docs/measurements/m19-nat-baseline.md
  EXPECT: >= 1
 
FLAT_SUBNET_BEHAVIOUR_MATCHES_DEMO_O6:
  $ grep -ciE "flat subnet|direct.*no relay|O-6" docs/measurements/m19-nat-baseline.md
  EXPECT: >= 1
 
UNIT_TESTS:
  $ go test -race -v ./internal/p2p/ ./cmd/relay/
  EXPECT: exit 0; tests include:
    TestAutoNATDetectsReachability
    TestCircuitRelayV2ReservationSucceeds
    TestDCUtRUpgradeToDirectConnection
```

---

### Phase 19.2 — Restore erasure coding

#### Session 19.2.1 — `klauspost/reedsolomon`

**PRECONDITIONS** — 19.0.1 complete. Parallelisable with Phase 19.1.

**TASK**

1. Add `github.com/klauspost/reedsolomon`; replace `rs_internal.go`'s hand-rolled encoder behind
   the existing `Engine` contract (IC §5.2). The exported surface does not change.
2. **Differential correctness against the demo implementation (closes Q-L-2 by hard test, per your
   instruction 4):** for both profiles, encode identical input with the demo's encoder and with
   `klauspost`, and assert byte-identical shards. If they differ, **the frozen demo produced shards
   no standard decoder can read** — a finding about the stash, which `docs/inherited/DEMO.md` must
   then record as a correction.
3. **Test RS(16,56) for the first time.** The demo only ever exercised RS(3,5) (Q-D-2).
4. Benchmark against ADR-009's 5% background CPU budget, which the hand-rolled encoder was never
   measured against.
**FILES** — `internal/erasure/engine.go` (edit), `internal/erasure/params.go` (edit),
`internal/erasure/rs_internal.go` (delete), `internal/erasure/erasure_test.go`, `go.mod`

**VERIFY**

```bash
REAL_REEDSOLOMON_DEPENDENCY:
  $ grep -c "klauspost/reedsolomon" go.mod
  EXPECT: >= 1
  $ test -f internal/erasure/rs_internal.go && echo FAIL || echo PASS
  EXPECT: PASS
 
EXPORTED_SURFACE_UNCHANGED:
  $ git diff demo-v1.0.0 -- internal/erasure/ | grep -cE "^-.*func \(.*Engine\) (Encode|Decode)"
  EXPECT: 0
 
DIFFERENTIAL_MATCH_AGAINST_DEMO_ENCODER:    # Q-L-2, resolved by test
  $ go test -run TestShardsMatchFrozenDemoVectors ./internal/erasure/
  EXPECT: exit 0
 
PRODUCTION_PARAMS_EXERCISED:                # Q-D-2 — RS(16,56) never tested before
  $ go test -run TestEncodeDecode16_56 ./internal/erasure/
  EXPECT: exit 0
 
CPU_BUDGET_MEASURED:                        # ADR-009's 5% budget
  $ go test -bench=BenchmarkEncode -benchtime=10x ./internal/erasure/ | tee docs/measurements/m19-rs.bench
  EXPECT: exit 0
 
UNIT_TESTS:
  $ go test -race -v ./internal/erasure/
  EXPECT: exit 0; tests include:
    TestShardsMatchFrozenDemoVectors
    TestEncodeDecode16_56
    TestEncodeDecode3_5
    TestDecodeSucceedsWithExactlyKShards
    TestDecodeFailsWithKMinusOneShards
```

---

### Phase 19.3 — Conformance and record

#### Session 19.3.1 — Stack conformance gate

**PRECONDITIONS** — 19.1.1–19.1.3, 19.2.1 complete.

**TASK** — verify every row of ADR-063's reversal table is closed and no substitution remains
anywhere in the tree. This is the milestone's gate.

**FILES** — `internal/p2p/conformance_test.go`, `scripts/ci/no_substitutions.sh` (new)

**VERIFY**

```bash
NO_SUBSTITUTION_MARKERS_ANYWHERE:
  $ grep -rciE "substitut|from-scratch|stdlib-only|could not be completed in this build environment" internal/ cmd/ \
      | grep -v ":0" | wc -l
  EXPECT: 0
 
EVERY_ADR063_ROW_CLOSED:
  $ bash scripts/ci/no_substitutions.sh
  EXPECT: exit 0
 
NFR016_MECHANISM_NOW_AS_SPECIFIED:
  $ go test -run TestTransportAuthentication ./internal/p2p/
  EXPECT: exit 0
  $ grep -cE "QUIC|Noise" internal/p2p/conformance_test.go
  EXPECT: >= 2
 
ALL_FOUR_DEPENDENCIES_PRESENT:
  $ grep -cE "go-libp2p|go-libp2p-kad-dht|go-multiaddr|klauspost/reedsolomon" go.mod
  EXPECT: >= 4
 
DEMO_GATED_PATHS_STILL_GATED:               # ADR-062 §5 — not extended, not deleted
  $ grep -cE "SoloMembership|envSecretsClient|MockProvider" docs/LTS-INHERITANCE.md
  EXPECT: >= 3
 
FULL_SUITE_GREEN_ON_REAL_STACK:
  $ go build ./... && go vet ./... && go test ./... -race && bash scripts/ci/lts_policy.sh
  EXPECT: exit 0
```

---

#### Session 19.3.2 — Record the dependency set; supersede ADR-063

**PRECONDITIONS** — 19.3.1 green.

**TASK**

1. Write `architecture.md §4.1`'s dependency table with real versions and reasons — the record
   IC §11 requires and which the demo never had.
2. Mark **ADR-063 `Superseded`** by this milestone; its reversal table becomes the completion
   evidence, row by row.
3. Update `docs/LTS-INHERITANCE.md`: every `Reversal-pending` entry moves to `Carried`.
4. Remove the demo-track restated rationales from `internal/p2p/*_test.go` (N-05) — on this track
   the original rationales are true again.
5. Sign off: `milestone: M19 LTS foundation — real dependency stack`.
**FILES** — `docs/system-design/architecture.md` (edit §4.1),
`docs/decisions/ADR-063-*.md` (status), `docs/LTS-INHERITANCE.md` (edit),
`internal/p2p/dht_test.go`, `internal/p2p/host_test.go`

**VERIFY**

```bash
ARCH_4_1_RECORDS_VERSIONS:                  # IC §11's own requirement, finally met
  $ grep -cE "go-libp2p v[0-9]|kad-dht v[0-9]|reedsolomon v[0-9]" docs/system-design/architecture.md
  EXPECT: >= 3
 
ADR063_SUPERSEDED:
  $ grep -c "^\*\*Status:\*\* Superseded" docs/decisions/ADR-063-*.md
  EXPECT: 1
 
NO_REVERSAL_PENDING_REMAINS:
  $ grep -c "Reversal-pending" docs/LTS-INHERITANCE.md
  EXPECT: 0
 
DEMO_TRACK_RESTATEMENTS_REMOVED:            # N-05 — no longer applicable on LTS
  $ grep -crE "TRACK: LTS|does not exist on the demo track" internal/p2p/ | grep -v ":0" | wc -l
  EXPECT: 0
 
FULL_GATE_GREEN:
  $ go build ./... && go vet ./... && go test ./... -race \
      && bash scripts/ci/grep_checks.sh && bash scripts/ci/no_substitutions.sh \
      && bash scripts/ci/lts_policy.sh
  EXPECT: exit 0
 
SIGNOFF:
  $ git log -1 --pretty=%s
  EXPECT: milestone: M19 LTS foundation — real dependency stack
```

---

## LTS — Production Hardening

**Provenance note (M17 Session 17.3.1).** This section is the content that used to live under
"Milestone 17 — Production Hardening" before that milestone number was repurposed for Demo
Completion (`build_part3.md`, commit `9c5262d "Restructure M17 entirely"`). That rewrite deleted the
old body without first copying it here — the gap Session 17.3.1 exists to close (N-03) — so this
text was recovered from `docs/system-design/build_part3.md` at revision `9c5262d^` via `git log`/
`git show`, not reconstructed from memory or re-derived. It is reproduced verbatim below, including
its internal `Phase 17.1`/`Session 17.1.1`-style numbering, which is now historical — a hangover from
when this content was Milestone 17 — and does **not** denote a current LTS milestone number.
Deliberately no new number is assigned here (per instruction 8): numbering for this milestone
follows the demo run outcomes, same as every other `(next)` row in this document's milestone map.

**Deliverable:** secrets-manager adapters for Vault/AWS SSM/GCP Secret Manager (IC §8), a real
three-replica gossip cluster with ADR-048's authenticated inter-replica sync, `cmd/relay`, and real
client-driven routing — replacing every demo-mode stub the same category of component uses today.

**Reference:** IC §4.3 (circuit relay), IC §8 (secrets manager — path/rotation contract),
architecture.md §13 (P2P transfer layer — relay tiers, node placement), §18 (coordination
microservice — gossip, quorum, routing, background throttling), §24 (deployment topology), §27.5
(network/DHT scaling); ADR-048 (gossip authentication)

---

**Status:** *(corrected per A-12)* All deployment-topology detail this milestone needs is
present in `architecture.md` §13, §18, §24, §27.5 — nothing here is blocked on missing
documentation. The one genuine gap found during this review was the absence of an
authentication contract for inter-replica gossip sync (A-9); it is resolved by
**ADR-048** (approved, drafted in full alongside this document) and is folded into Session
17.2.1 below as a marked addendum, in the same base-task-plus-addendum shape Milestones
13/14 already use for ADR-036.

**Reference:** IC §4.3 (circuit relay), IC §8 (secrets manager — path/rotation contract),
architecture.md §13 (P2P transfer layer — relay tiers, node placement), §18 (coordination
microservice — gossip, quorum, routing, background throttling), §24 (deployment topology),
§27.5 (network/DHT scaling); ADR-048 (gossip authentication — new)

---

### Phase 17.1 — Secrets Manager Adapters

**Reference:** IC §8, `mvp.md` §6.2 (IR-03)

#### Session 17.1.1 — Implement secrets manager adapters

**TASK:** Implement `SecretsManagerClient` (IC §8 interface — `GetSecret(ctx, path)
([]byte, error)`) for three backends:

- HashiCorp Vault (`internal/secrets/vault.go`)
- AWS SSM Parameter Store (`internal/secrets/aws_ssm.go`)
- GCP Secret Manager (`internal/secrets/gcp_secret.go`)
Each adapter reads the secret at path `/vyomanaut/audit-secret/v{N}` (IC §8 path
convention). Each must handle the 24-hour rotation overlap window: read both `v{N}` and
`v{N+1}` when both exist (IC §8 rotation contract). Selection among adapters is via
`VYOMANAUT_SECRETS_BACKEND`. **Forward-compatibility note (A-10/A-11):** these same three
adapters, unchanged, are what Session 17.2.1 below reads the new
`/vyomanaut/cluster-replica-pubkeys/v{N}` path through — no fourth adapter, no interface
change; write the path handling generically (accept any `/vyomanaut/{topic}/v{N}` shape)
rather than hardcoding `audit-secret` into the adapter logic itself, so the new path Session
17.2.1 needs costs nothing extra here.

**FILES** — `internal/secrets/vault.go`, `aws_ssm.go`, `gcp_secret.go`, `secrets_test.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/secrets/
  EXPECT: exit 0
 
IMPORT_CONSTRAINTS:                    # A-3 — internal/secrets is a leaf, like crypto/erasure
  $ grep -cE "Vyomanaut_V2/internal/(audit|scoring|repair|payment|storage|cluster|client)" internal/secrets/*.go
  EXPECT: 0
 
ALL_THREE_BACKENDS_IMPLEMENT_GETSECRET:
  $ grep -cE "func.*GetSecret\(ctx" internal/secrets/vault.go internal/secrets/aws_ssm.go internal/secrets/gcp_secret.go
  EXPECT: 3
 
PATH_CONVENTION_NOT_HARDCODED_TO_AUDIT_SECRET:      # forward-compat for ADR-048's new path
  $ grep -nE '"/vyomanaut/audit-secret' internal/secrets/vault.go internal/secrets/aws_ssm.go internal/secrets/gcp_secret.go
  EXPECT: no matches inside GetSecret itself — the path is a caller-supplied argument, not a package-level constant
 
ROTATION_OVERLAP_READS_BOTH_VERSIONS:
  $ grep -cE "v\{N\}|vN|version.*\+.*1|N\+1" internal/secrets/vault.go
  EXPECT: >= 1
 
BACKEND_SELECTED_VIA_ENV_VAR:
  $ grep -c "VYOMANAUT_SECRETS_BACKEND" internal/secrets/*.go
  EXPECT: >= 1
 
SENTINEL_ERRORS_PRESENT:
  $ grep -cE "ErrSecretNotFound|ErrSecretManagerUnavailable|ErrSecretExpired" internal/secrets/*.go
  EXPECT: >= 3
 
UNIT_TESTS:
  $ go test -v -run TestSecretsManager ./internal/secrets/
  EXPECT: exit 0; tests include:
    TestVaultGetSecretReturnsDecodedBytes
    TestAWSSSMGetSecretReturnsDecodedBytes
    TestGCPSecretManagerGetSecretReturnsDecodedBytes
    TestRotationOverlapReadsBothVAndVPlusOne
    TestBackendSelectionRespectsEnvVar
    TestUnreachableManagerReturnsErrSecretManagerUnavailable
 
VET:
  $ go vet ./internal/secrets/
  EXPECT: exit 0; zero output
```

---

### Phase 17.2 — HA Microservice & Relay Nodes

#### Session 17.2.1 — Implement gossip cluster (`internal/cluster`) *(base task unchanged; security addendum added — A-9/ADR-048)*

**Reference:** ARCH §18; ADR-048 (new)

**TASK — base (matches ARCH §18 as currently written):**

Create `internal/cluster/gossip.go` implementing the three-replica gossip membership per
`architecture.md` §18. The `GossipCluster` struct must expose:

- `HealthyCount() int` — returns the count of peers with a last-seen timestamp within
  `profile.GossipHealthyWindow` (ADR-048 §6 — a named profile field, not the bare `5
  seconds` literal the original spec used).
- `MemberAddresses() []url.URL` — returns the current membership list for client-driven
  routing.
- A `reconcile()` loop running at 1-second intervals: select one randomly-chosen peer,
  exchange membership histories via `POST /internal/membership/sync` carrying a vector
  clock.
- Two pre-configured seed node addresses read from `VYOMANAUT_SEED_NODE_1` and
  `VYOMANAUT_SEED_NODE_2`; these prevent partition on restart (architecture.md §18).
Quorum check: `HealthyCount() >= 2` satisfies the (3,2,2) write quorum (architecture.md
§18). A read or write that cannot reach 2 replicas must return `ErrQuorumUnavailable`.

Add `internal/cluster/router.go` implementing `ResponsibleReplica(opType string) *url.URL`
per the client-driven routing description in M12 Session 12.1.1.

Add `internal/cluster/mock_cluster.go` (build tag `test`) providing `MockClusterMembership`
that returns configurable healthy counts for unit testing without a live cluster.

Wire into `cmd/microservice/main.go`: after guard rails pass, initialise `GossipCluster`,
wait for 2-peer ack, then start the readiness evaluator.

**⛔ SECURITY ADDENDUM — apply after ADR-048 accepted (A-9):** the base task above matches
`architecture.md §18` as currently written, but §18 itself never specifies authentication
for `POST /internal/membership/sync`, and `interface-contracts.md` §2's own rule treats
this link as out of scope without an ADR — closed by ADR-048, now approved and drafted.
Once accepted:

1. **Replica identity at startup.** Each replica generates (or loads, if already present)
   an Ed25519 key pair, persisted locally the same way `internal/p2p/identity.go` persists
   a provider's identity (ADR-048 §2). Load the other two replicas' public keys from
   `/vyomanaut/cluster-replica-pubkeys/v{N}` via the existing `SecretsManagerClient` (Session
   17.1.1 — no new interface), cached on the same 5-minute TTL as the audit secret.
2. **Extend `MembershipSyncRequest`/`-Response`** with `request_ts_ms`(8B) and
   `replica_sig`(64B) — Ed25519 by the sender's own replica key over
   `SHA-256("vyomanaut-gossip-sync-v1" ‖ replica_id ‖ vector_clock ‖ request_ts_ms)`
   (ADR-048 §4). Sign **both** the outgoing request and the outgoing response — this is a
   bidirectional exchange, and a forged response is exactly as dangerous as a forged
   request (ADR-048 §4).
3. **Handler ordering, before any vector-clock merge** (ADR-048 §5 — authz-before-mutation,
   the same ordering pattern already enforced in `cmd/provider/handler_vetting_gc.go` for
   ADR-036): (a) `replica_id` ∈ the cached two-key set → else discard silently; (b)
   `|now − request_ts_ms| ≤ profile.AuthRequestFreshnessWindow` (reused from ADR-036, not a
   new field) → else discard as stale; (c) `replica_sig` verifies → else discard. Only then
   merge.
4. Add the `MS ↔ MS` self-edge to `interface-contracts.md` §2's diagram and cross-reference
   table (ADR-048 §1 / A-10) — this is a documentation PR, not a code change, but it is a
   precondition IC §2 itself states before this session's code path is in scope at all.
**FILES** — `internal/cluster/gossip.go`, `router.go`, `mock_cluster.go` (build tag `test`),
`internal/cluster/identity.go` (new — addendum only), `gossip_test.go`

**VERIFY**

```bash
COMPILE:
  $ go build ./internal/cluster/
  EXPECT: exit 0
 
IMPORT_CONSTRAINTS:                    # A-3's proposed row for internal/cluster
  $ grep -cE "Vyomanaut_V2/internal/(audit|scoring|repair|payment|storage)" internal/cluster/*.go
  EXPECT: 0
 
HEALTHYCOUNT_USES_NAMED_PROFILE_FIELD_NOT_LITERAL_5S:
  $ grep -c "profile.GossipHealthyWindow" internal/cluster/gossip.go
  EXPECT: >= 1
  $ grep -cE '5 \* time\.Second\b' internal/cluster/gossip.go
  EXPECT: 0
 
RECONCILE_1S_INTERVAL_AND_SEED_NODES:
  $ grep -c "time.NewTicker(1 \* time.Second)\|1 \* time.Second" internal/cluster/gossip.go
  EXPECT: >= 1
  $ grep -cE "VYOMANAUT_SEED_NODE_1|VYOMANAUT_SEED_NODE_2" internal/cluster/gossip.go
  EXPECT: 2
 
QUORUM_THRESHOLD_IS_TWO:
  $ grep -c "HealthyCount() >= 2" internal/cluster/*.go
  EXPECT: >= 1
  $ grep -c "ErrQuorumUnavailable" internal/cluster/*.go
  EXPECT: >= 1
 
MOCK_CLUSTER_BUILD_TAGGED_TEST_ONLY:
  $ head -1 internal/cluster/mock_cluster.go | grep -c "go:build test"
  EXPECT: 1
 
UNIT_TESTS_BASE:
  $ go test -v -run "TestGossip|TestRouter" ./internal/cluster/
  EXPECT: exit 0; tests include:
    TestHealthyCountUsesProfileWindow
    TestReconcileTicksEverySecond
    TestQuorumUnavailableBelowTwoHealthy
    TestResponsibleReplicaClientDrivenRouting
    TestMockClusterMembershipConfigurableHealthyCounts
 
# ── VERIFY (enable after ADR-048 accepted) — the security-critical checks ──
REPLICA_IDENTITY_LOADED_AT_STARTUP:
  $ grep -c "ed25519.GenerateKey\|ed25519.NewKeyFromSeed" internal/cluster/identity.go
  EXPECT: >= 1
  $ grep -c "cluster-replica-pubkeys" internal/cluster/identity.go
  EXPECT: >= 1
 
SYNC_REQUEST_AND_RESPONSE_BOTH_SIGNED:
  $ grep -c "replica_sig\|ReplicaSig" internal/cluster/gossip.go
  EXPECT: >= 2                        # request AND response paths both reference it
 
AUTHZ_BEFORE_ANY_MERGE:
  $ awk '/replica_id.*cached|verifyReplicaID/{a=NR} /mergeVectorClock|MergeMembership/{m=NR} END{print (a>0 && m>0 && a<m)?"PASS":"FAIL"}' internal/cluster/gossip.go
  EXPECT: PASS
 
FRESHNESS_WINDOW_REUSES_ADR036_FIELD_NOT_A_NEW_ONE:
  $ grep -c "profile.AuthRequestFreshnessWindow" internal/cluster/gossip.go
  EXPECT: >= 1
  $ grep -cE "GossipFreshnessWindow|SyncFreshnessWindow" internal/config/*.go
  EXPECT: 0                           # must not introduce a second, redundant freshness field
 
UNIT_TESTS_ADR048:
  $ go test -v -run "TestGossipRejectsUnknownReplica|TestGossipRejectsStaleSync|TestGossipRejectsForgedSig|TestGossipVerifiesResponseSignatureToo" ./internal/cluster/
  EXPECT: exit 0
 
VET:
  $ go vet ./internal/cluster/
  EXPECT: exit 0; zero output
```

#### Session 17.2.2 — Relay node binary and deployment configuration

**Reference:** architecture.md §13, §24, §27.5

**TASK:** Create `cmd/relay/main.go` as the relay node binary. The relay runs a libp2p host
with Circuit Relay v2 enabled and no DHT, chunk storage, or audit logic. Configuration:

- 128 concurrent relay reservations per node (architecture.md §13, §27.5).
- Reservation TTL: 30 minutes (libp2p default).
- Relay multiaddrs are reported via `GET /relay/status` → `{"reservation_count": N,
  "capacity": 128}`.
- Metrics: expose `vyomanaut_relay_reservations_active` gauge at `/metrics` (naming per IC
  §10's `vyomanaut_{subsystem}_{name}_{unit}` convention — `relay` as subsystem).
Create `deployments/production/relay/docker-compose.yml` for the three-node relay
deployment:

- Node 1: Mumbai AZ1 (`ap-south-1a`)
- Node 2: Mumbai AZ2 (`ap-south-1b`)
- Node 3: Chennai/Hyderabad (`ap-south-2` or `ap-southeast-1`)
- Minimum spec per node: 1 vCPU, 1 GB RAM, 1 Gbps network (architecture.md §24).
**FILES** — `cmd/relay/main.go`, `deployments/production/relay/docker-compose.yml`

**VERIFY**

```bash
COMPILE:
  $ go build ./cmd/relay/
  EXPECT: exit 0
 
NO_DHT_STORAGE_OR_AUDIT_LOGIC:                # relay must be minimal per its own spec
  $ grep -cE "Vyomanaut_V2/internal/(storage|audit|scoring|repair|payment)" cmd/relay/main.go
  EXPECT: 0
  $ grep -ciE "dht\." cmd/relay/main.go
  EXPECT: 0
 
RESERVATION_CAPACITY_128:
  $ grep -c "128" cmd/relay/main.go
  EXPECT: >= 1
 
RESERVATION_TTL_30MIN:
  $ grep -cE "30 \* time.Minute|1800" cmd/relay/main.go
  EXPECT: >= 1
 
STATUS_ENDPOINT_SHAPE:
  $ grep -c "reservation_count\|/relay/status" cmd/relay/main.go
  EXPECT: >= 2
 
METRIC_NAME_FOLLOWS_IC10_CONVENTION:
  $ grep -c "vyomanaut_relay_reservations_active" cmd/relay/main.go
  EXPECT: >= 1
 
DOCKER_COMPOSE_THREE_AZ_NODES:
  $ grep -cE "ap-south-1a|ap-south-1b|ap-south-2|ap-southeast-1" deployments/production/relay/docker-compose.yml
  EXPECT: >= 3
 
UNIT_TESTS:
  $ go test -v -run TestRelay ./cmd/relay/
  EXPECT: exit 0; tests include:
    TestRelayEnforces128ReservationCap
    TestRelayStatusEndpointReturnsCorrectShape
    TestRelayExportsReservationsActiveGauge
    TestRelayHostHasNoDHTServerMode
 
VET:
  $ go vet ./cmd/relay/
  EXPECT: exit 0; zero output
```

---
