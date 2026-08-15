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
