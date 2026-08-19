# ADR-084 — The Demo Presentation Surface, and the Operator Blindness Invariant

**Status:** Accepted — seven decisions ratified (D-1…D-7), scoped as **Milestone 17 Extended**
**Track:** DEMO
**Topic:** #16 Simulation & Scale, #12 Desktop Shell & UX, #8 Key Management (D-3), #4 Repair Protocol (D-4)
**Supersedes:** —
**Superseded by:** —
**Research source:** the eleven founding functional requirements (project inception); `build_part3.md`
Milestone 17 deliverable statement; live read of `cmd/provider/main.go`, `cmd/client/`,
`internal/api/router.go`, `internal/repair/departure.go`, `internal/storage/`,
`scripts/test/demo_timeline_test.go`, `scripts/test/demo_cli_test.go` (this session); ADR-020,
ADR-029, ADR-030, ADR-031, ADR-035, ADR-046, ADR-055, ADR-061, ADR-063, ADR-064, ADR-071, ADR-075,
ADR-077; MVP §7, §8.3; IC §11, §14; NFR-038, NFR-044

---

## Context

Vyomanaut began with eleven functional requirements written before any architecture existed. They
are reproduced verbatim in §Appendix A. Every ADR since has been downstream of them, and none has
restated them. The demo track exists to satisfy them.

M17's own deliverable statement in `build_part3.md` reads:

> **Deliverable:** a demo a human can run.

`cmd/client` is built, `scripts/test/` is complete, and four integration tests are green. The
milestone nonetheless does not meet its own deliverable, for a reason its VERIFY blocks could not
have caught: **no human can currently onboard a provider.** `cmd/provider/main.go`'s own comment
records the mechanism:

> A provider daemon has no legitimate way to obtain this itself […] so this is supplied externally,
> by whatever orchestrates the daemon and legitimately holds that access (`demo_timeline_test.go`).

The only entity in the system capable of onboarding a provider is the integration harness, holding
database credentials and brute-forcing 10⁶ preimages against `otp_codes.code_hash`
(`demo_timeline_test.go:493`). Requirement 2 is satisfied by test infrastructure and by nothing else.

Reading the live tree against all eleven requirements produced four findings. They are recorded here
because each one changes what must be built, and the first changes what the demo track can honestly
claim.

### F-D-1 — Requirement 7 is unproven. No test retrieves a file after repair

Requirement 7 promises: *"When the provider goes offline, repair takes place, **ensuring that the
file can be retrieved at any time by the data owner**."*

Two tests exercise departure and repair:

| Test | Terminates at |
| --- | --- |
| `TestDemoTimeline` (`demo_timeline_test.go:1011`) | `pollRepairCompleted(t, ctx, db, 1, 5*time.Minute)` |
| `TestViabilityRepairSucceedsWithTwoOfFiveOffline` (`:1129`) | `pollRepairCompleted(t, ctx, db, 2, 5*time.Minute)` |

Both stop at *"the repair pipeline marked its jobs `COMPLETED`."* Neither retrieves the file
afterwards; neither compares bytes.
`TestDemoCLIRetrievedBytesIdenticalToUploaded` does compare bytes, but against a fleet where nothing
was ever killed — it proves the undamaged path only.

The suite therefore proves *repair jobs reach a terminal database state*. It does not prove *the file
survives a provider loss*. These are different claims and only the second is the product's promise.
A repair that writes a corrupt shard, writes to the wrong `chunk_id`, or completes its bookkeeping
without completing its transfer passes both tests and fails the product.

This is not a hypothetical defect class. F-16-3 (repair-download frame size mismatch) and F-16-4
(`findSurvivingHolders` never selecting `chunk_id`) were live-verified bugs in exactly this pipeline,
found only by reading both sides of a wire protocol. The assertion that would have caught them
independently does not exist in any test.

**F-D-1 is the most serious finding on the demo track and this ADR's primary justification.**

### F-D-2 — Requirement 2 is not human-runnable

Provider registration requires `--registration-bearer-token`, obtainable only by completing OTP
verification. In demo mode `NoopOtpSender` sends nothing, and `otp_codes` stores a SHA-256 hash, not
the code. There is no path from "a person wants to volunteer a desktop" to "that desktop is
`ACTIVE`" that does not pass through database credentials.

### F-D-3 — Normal (single-instance) provider mode is exercised by no test

Every integration test starts providers with `--sim-count=7 --sim-only-index=i`. The normal-mode
branch at `cmd/provider/main.go:269` — the branch a physical desktop takes — has zero coverage.
`listenPort` is additionally fixed at 4001 with no flag, so two providers cannot share a host. M18
plans a five-to-six-desktop physical run; that run would be the first execution of this code path.

### F-D-4 — Multi-machine peer-to-peer transfer cannot work today. The advertised address is hardcoded to loopback

The provider binds its listener correctly to all interfaces:

```go
listenAddr := fmt.Sprintf("0.0.0.0:%d", cfg.listenPort)   // main.go:578
```

but advertises itself to the network as loopback, in both places it publishes an address:

```go
multiaddr := fmt.Sprintf("/ip4/127.0.0.1/tcp/%d/p2p/%s", cfg.listenPort, peerID)         // main.go:501 — registration
localMultiaddr, _ := p2p.ParseMultiaddr(fmt.Sprintf("/ip4/127.0.0.1/tcp/%d/p2p/%s", …))  // main.go:663 — heartbeat/DHT
```

`internal/client/upload/transfer.go` dials `last_known_multiaddrs`. On separate desktops every dialer
would therefore connect to `127.0.0.1` — its own machine — and never reach the provider. This was
invisible because every test to date runs all peers in one process on one host, where loopback
happens to be correct.

The finding is cheap to fix and structurally important: it is the single change standing between the
current demo and genuine machine-to-machine data transfer. See §Answers, Q2.

---

## Decision

Seven decisions, ratified. D-2, D-3 and D-4 carry design content and are elaborated below.

| # | Decision | Ratified as |
| --- | --- | --- |
| D-1 | Operator binary name | **`cmd/operator`** — consistent with the existing role-named binaries `client` / `provider` / `microservice`. The operator is the fourth role in Vyomanaut's own threat model. |
| D-2 | Console implementation | **A real TUI**, Bubble Tea + Lip Gloss + Bubbles, vendored. See §D-2. |
| D-3 | OTP delivery for human onboarding | **`FileOtpSender`** — a real `OtpSender` implementation writing a gateway delivery log. Not a database read, not a demo-mode endpoint. See §D-3. |
| D-4 | Departure threshold | **Overridable at runtime, demo-mode only, with a derived safety floor**, plus a six-case departure edge-case matrix. Profile constants unchanged. See §D-4. |
| D-5 | Scope and numbering | **Milestone 17 Extended (M17-E)**, phases 17.4–17.8. M17's deliverable is unmet; this closes it rather than extending the plan. |
| D-6 | Documentation order | **ADR first, always.** This document precedes Session 17.4.1. `mvp.md` §8.3's subcommand tables are amended in Session 17.8.2, not ad hoc. |
| D-7 | Physical rig invocation | **Normal mode**, not `--sim-only-index` per desktop. This is why F-D-3 and F-D-4 must close before M18. |

### D-2 — The console is a real TUI, and the dependency cost is paid by vendoring

The earlier recommendation was stdlib-only ANSI, on the grounds that M18 is a freeze and `go.sum`
already carries hand-repackaged checksums requiring disclosure. That recommendation is withdrawn. It
optimised for freeze hygiene at the cost of the artifact the freeze exists to preserve.

**Chosen:** `charmbracelet/bubbletea` (Elm-architecture runtime), `charmbracelet/lipgloss` (layout and
style), `charmbracelet/bubbles` (viewport, table, spinner primitives).

Rationale, in order of weight:

1. **Correct redraw semantics.** Bubble Tea diffs frames and repaints only changed cells. A
   hand-rolled full-screen ANSI repaint at 1 Hz flickers on every terminal that does not
   synchronise output, and flicker on a projector is the difference between a credible system and a
   script. Our tick rate is 1 s against a renderer capable of 60 fps; performance headroom is not in
   question.
2. **Windows works.** Bubble Tea handles Windows Terminal and conhost virtual-terminal sequences,
   terminal resize, and alternate-screen entry/exit across platforms. Hand-rolled escape sequences
   do not, and the physical rig is Windows-first (ADR-010, ADR-041). This decision materially
   improves the answer to Q1.
3. **Pure Go, no cgo.** Adds no C toolchain requirement — which matters given the provider's
   existing cgo dependency (§Answers, Q1).

**The freeze cost is paid, not avoided.** `go mod vendor` before the M18 tag. M18 already requires a
vendored, tagged, archived freeze; vendoring makes the artifact self-contained and, as a side
effect, *reduces* the hand-repackaged-`go.sum` exposure rather than adding to it, because the
vendored tree no longer resolves checksums at build time. Disclosure in `docs/DEMO.md` remains
required either way.

**The console is designed for Vyomanaut, not adapted from a template.** Its panels are the system's
own invariants, each rendered against the profile constant that governs it — never a hardcoded
number:

- **Readiness gate** — the five ADR-029 conditions as satisfied/unsatisfied bars against
  `MinActiveProviders`, `MinDistinctASNs`, `MinMetroRegions`, `MinRelayNodes`, `MinCooledAccounts`.
- **Provider fleet** — one row per provider: peer ID, ASN, status, heartbeat age with a live
  countdown to the effective departure threshold, declared GB, chunks held, `Score7d`/`Score30d`.
- **ASN cap occupancy** — shards per ASN against `floor(TotalShards × ASNCapFraction)`. At demo scale
  that is `floor(5 × 0.20) = 1`, and ADR-075's finding is that the demo topology sits at *exactly*
  full occupancy. A panel that shows headroom of zero is a panel that explains the seventh provider.
- **Repair** — queue depth by priority, in-flight jobs, completions, and the `r0` threshold
  (`LazyRepairR0 = 1`) against available shard counts. See §Consequences on F-LTS-07.
- **Audit** — challenges, passes, failures, pass rate, and time to next `AuditPeriodDuration` tick.
- **Escrow and release** — charged paise, released paise, per-provider split, next
  `ChargeComputationInterval` / `ReleaseComputationInterval` tick. `int64` throughout, one formatter,
  no floating point (NFR-038).
- **Event feed** — a rolling log of node entry, status transition, transfer, repair, audit, and
  payment events.

### D-2a — Invariant I-DEMO-1: operator blindness

Requirement 4's parenthetical — *"but never see the originally uploaded data"* — is a confidentiality
claim about the operator role. It is promoted here from prose to a CI-enforceable structural
invariant, in the manner ADR-077 established for `[UNDERIVED]` / `[VENDOR-DEFAULT]`:

> **I-DEMO-1.** `cmd/operator` shall have no import path, direct or transitive, to any decoding
> primitive.

```bash
go list -deps ./cmd/operator \
  | grep -cE 'internal/(crypto/aont|erasure|client/(retrieve|upload))'
EXPECT: 0
```

This is the difference between asserting the operator cannot read the data and demonstrating that no
code path exists by which it could. It is paired with `operator shards <file_id>`, which prints the
operator's complete view of one file — chunk IDs, holding providers, ASNs, shard indices, sizes — and
renders `display_name_ciphertext` **as hex, labelled as ciphertext**. Because that column is AEAD
ciphertext in the schema (`migrations/generator.go:528`, *"Microservice stores blindly; cannot read
the filename (ADR-020)"*), the demonstration is honest: the operator genuinely does not know the
filename.

The same invariant is imposed on `cmd/provider`, for the same reason at a different threshold — a
provider holds one shard against `DataShards = 3`:

```bash
go list -deps ./cmd/provider \
  | grep -cE 'internal/(crypto/aont|erasure|client/retrieve)'
EXPECT: 0
```

### D-3 — OTP delivery: a file-backed gateway, not a database read

The brief was *"more presentable and accurate in terms of how the actual authentication would take
place; design it for simplicity and security."* Three candidates were considered.

| Candidate | Verdict |
| --- | --- |
| Demo-mode endpoint returning the plaintext code | **Rejected.** Adds an authentication-bypass path to the microservice gated only on `profile.Mode == "demo"`. Mode-gated auth bypasses are precisely the category of thing that survives into production by accident, and it must then be deliberately removed at M19. |
| `operator otp <phone>` reading `otp_codes` and brute-forcing the hash | **Rejected.** Structurally inaccurate: it depicts authentication as a database lookup, which is the opposite of how OTP works. Also slow and inelegant on stage. |
| **`FileOtpSender` — a real `OtpSender` implementation** | **Chosen.** |

`internal/api/otp.go` already defines the seam:

```go
type OtpSender interface {
    SendOTP(ctx context.Context, phoneNumber, code string) error
}
```

with exactly one implementation, `NoopOtpSender`, wired at `cmd/microservice/main.go:314`. The
decision is to write a second implementation and swap that one line.

`FileOtpSender` appends one line per send to a delivery log on the microservice host — mode `0600`,
never served over HTTP, path from `--otp-delivery-log` (default `<data-dir>/otp-delivery.log`):

```
2026-08-19T11:04:22Z  +919876530001  PROVIDER_REGISTER  418362
```

That is what an SMS gateway's delivery log looks like. `operator otp <phone>` tails it and prints the
most recent undelivered code.

Why this is both the most accurate and the most secure of the three:

- **Accurate.** In production the operator contracts an SMS gateway that holds the plaintext code
  transiently and the database never does. `FileOtpSender` preserves that exact division:
  `otp_codes.code_hash` stays hash-only and the schema is not weakened for a demo. Swapping in MSG91
  or Twilio at LTS deletes this implementation and changes nothing else.
- **Secure.** No new endpoint, no new authority, no mode-gated branch inside request handling. The
  code path exists only in the constructor at `main.go:314`.
- **Presentable.** The volunteer runs `provider onboard --phone +91…` and sees *"OTP sent — ask the
  network operator for your code."* The operator reads six digits off the console and says them
  aloud. The volunteer types them. The audience watches two parties with separate authority
  cooperate to admit a node — which is what actually happens in production, and is better theatre
  than any silent bypass.

### D-4 — Departure is overridable, floored, and exercised at six moments

The instruction is that a provider must be able to exit at any point in the timeline, and the system
must be unaffected. Two separable problems.

**(a) Detection latency.** `DemoProfile.DepartureThreshold` is 10 minutes and is consumed in exactly
one place — `findDepartureCandidates` (`internal/repair/departure.go:136`). That single injection
point makes a runtime override genuinely low-complexity, which was the condition attached to this
decision.

- New flag on `cmd/microservice`: `--departure-threshold`.
- **Fatal in production mode.** `profile.Mode != "demo"` → refuse to start.
- Default is `profile.DepartureThreshold` unchanged, so every existing test — including
  `TestViabilityActiveTransitionAtTenMinutes`, which asserts against the 10-minute value — passes
  untouched. `DemoProfile` is not edited. mvp §7.3's vetting arithmetic and M16's live-verified
  timeline remain valid.
- **A derived safety floor, enforced fatally at startup.** Shortening the threshold below the
  heartbeat cadence causes a *live* provider's normal jitter to be read as departure — which marks
  it `DEPARTED`, freezes it, and seizes its escrow (`processDeparture`). Punishing an honest provider
  for a demo's impatience is not an acceptable failure mode, so the floor is checked, not
  documented:

  ```
  floor = max( 2 × (HeartbeatInterval + HeartbeatJitter),   2 × DeparturePollingInterval )
        = max( 2 × (30s + 5s),                              2 × 30s )
        = max( 70s,                                          60s )
        = 70s
  ```

  Two missed heartbeats is the minimum evidence that distinguishes silence from jitter; two polling
  intervals is the minimum that keeps detection granularity below the threshold it measures.
  **Demo runs use 90 s**, giving margin above the floor while keeping requirement 7's ungraceful path
  inside a live audience's patience.

**(b) Departing at an arbitrary moment.** Latency is not the interesting part; *when* is. This ADR
adopts a six-case departure matrix, each case a named test in Phase 17.7:

| Case | Moment of departure | Expected behaviour | Risk |
| --- | --- | --- | --- |
| E-1 | While `VETTING`, before any real upload | Synthetic vetting chunks soft-deleted, **zero** repair jobs (FR-065) | low — covered path |
| E-2 | **Mid-upload**, after `/upload/assign`, before all shards transferred | Upload fails cleanly with an IC §14 code, or completes across the surviving set; **never** a half-registered file | **high** |
| E-3 | After upload, before first audit | Repair from `k = 3` survivors; retrieve succeeds | medium |
| E-4 | **Mid-repair — the replacement provider itself departs** | Repair job re-queued to a new replacement (ADR-075 headroom), not stuck `IN_PROGRESS` | **high** |
| E-5 | **Mid-retrieval** | Retrieval still gathers `k = 3` from the remaining holders | **high** |
| E-6 | Two concurrently, `s = 3` exactly | Emergency floor holds (ADR-055); both repairs complete; retrieve succeeds | medium |

E-2, E-4 and E-5 are expected to surface defects. That is the purpose of the matrix, not a risk to
be managed around it — the same reasoning that produced F-16-1 through F-16-6. Every case terminates
in a byte-identity assertion, per F-D-1.

---

## Consequences

### Positive

- The eleven founding requirements each acquire a named integration test (`TestReqD01`…`TestReqD11`),
  registered in `scripts/ci/grep_checks.sh`, so deleting one becomes a CI failure rather than a silent
  regression.
- F-D-1 closes. The demo track can, for the first time, claim data survival rather than pipeline
  completion.
- F-D-2, F-D-3 and F-D-4 close before the physical rig meets them, rather than during it.
- Requirement 4's confidentiality claim becomes structural (I-DEMO-1) instead of asserted.
- `FileOtpSender` leaves the microservice's attack surface unchanged and is deleted, not disabled,
  at LTS.
- Vendoring for D-2 hardens the M18 freeze independently of the console.

### Negative and accepted

- **Three new modules** enter `go.mod` (plus their transitive set) at a freeze milestone. Accepted;
  mitigated by vendoring and by D-2's stated rationale.
- **`ChunkStore` gains a method.** `provider inspect` must enumerate held chunks, and the interface
  has only `AppendChunk`/`LookupChunk`/`DeleteChunk`/`RecoverFromCrash`/`RunGC`/`Close`. Adding
  `ListChunks() ([][32]byte, error)` requires an implementation on **both** engines — RocksDB
  iterator (`engine_rocksdb.go`) and Badger iterator (`engine_badger.go`). This is the only change
  in M17-E that touches a core interface and it is flagged accordingly.
- **The ungraceful departure path still costs 90 seconds of wall clock**, floored by arithmetic that
  cannot be argued down without risking false departures. Phase 17.8's runbook spends it visibly:
  start the kill, run the audit and payment acts in the foreground, let the console's countdown
  carry the tension.
- **E-2, E-4 and E-5 may fail on first execution.** Budget for findings.

### Deferred, and named so it is not mistaken for done

- **F-LTS-07 stands.** The system performs eager repair; the `r0` gate specified in ADR-004 was never
  built. The console displays `LazyRepairR0` against live shard counts, which makes the gap
  *visible* — it does not close it. LTS work.
- **F-LTS-08 stands.** The microservice reconstructs the AONT package on every repair, violating
  ADR-004 and ADR-019. Not touched here.
- The Domain P confidentiality-margin result and Domain K's `N_eff = 5.161` finding are unaffected by
  this ADR and remain LTS-track.

---

## Milestone 17 Extended — phase structure

Phases are ordered so that each closes a finding before the phase depending on it begins, and so
every requirement has an owning phase.

| Phase | Title | Requirements | Closes |
| --- | --- | --- | --- |
| **17.4** | Provider self-service and real addressing | 2, 11 | F-D-2, F-D-3, F-D-4 |
| **17.5** | Local storage proof | 5, 11 | — |
| **17.6** | The operator console | 3, 4, 6, 9, 10 | I-DEMO-1 |
| **17.7** | Departure under load | 7, 8 | **F-D-1**, D-4 matrix |
| **17.8** | The runnable demo | all eleven | M17 deliverable |

**17.4** — `provider` subcommand split (`onboard` / `run` / `inspect` / `earnings` / `depart`) with
bare-flag invocation preserved for the harness; `FileOtpSender` and `--otp-delivery-log`;
`--listen-port`; `--advertise-addr` with LAN autodetect (F-D-4).

**17.5** — `ChunkStore.ListChunks` on both engines; `provider inspect` with hexdump and Shannon
entropy per chunk beside the original file's entropy; declared allocation and NFR-044 ceiling
displayed; `cmd/provider` blindness check.

**17.6** — `cmd/operator` with `watch` (the seven panels), `shards`, `otp`, `audit`, `payout`;
I-DEMO-1 in CI; no new server endpoints; every threshold read from the profile.

**17.7** — `--departure-threshold` with the derived floor; the six-case matrix E-1…E-6, each ending
in byte identity; `TestReqD07` closing F-D-1.

**17.8** — `scripts/demo/` runner in **normal mode** on distinct ports; `docs/DEMO.md` with eleven
acts; `demo_requirements_test.go` with `TestReqD01`…`TestReqD11`; `mvp.md` §8.3 amended (D-6);
`go mod vendor`; ADR-063 substitution and `go.sum` disclosure.

Session-level `FILES:` / `VERIFY:` blocks follow in the M17-E build document. Note the ordering
dependency: **17.4 must land before 17.7**, because the E-2/E-4/E-5 cases need `provider depart` and
a real advertised address to be meaningful.

---

## Answers to two standing questions

### Q1 — Is the demo runnable on any OS after M18?

**Conditionally yes, and the conditions are uneven in a way worth knowing before the rig is built.**

| Component | Windows | Linux / macOS |
| --- | --- | --- |
| `cmd/microservice` | pure Go (`lib/pq`); needs Postgres 16 reachable | same |
| `cmd/client` | pure Go | pure Go |
| `cmd/operator` (new) | pure Go, Bubble Tea handles VT sequences | pure Go |
| **`cmd/provider`** | **pure Go — BadgerDB, `CGO_ENABLED=0`** | **cgo — `grocksdb` needs `librocksdb` + a C toolchain** |

`NewChunkStore` dispatches on build tag: `engine_badger.go` is `//go:build windows`,
`engine_rocksdb.go` is `//go:build linux || darwin` and imports `github.com/linxGnu/grocksdb`, a cgo
binding. The consequence is counter-intuitive: **the Windows provider is the easier of the two to
run.** ADR-046 chose Badger for Windows because RocksDB is painful there; the effect is that a fresh
Windows desktop needs only the binary, while a fresh Linux desktop needs `librocksdb-dev`, `gcc`, and
`CGO_ENABLED=1` before `go build` succeeds.

Two further frictions: `scripts/demo/*.sh` is bash and needs PowerShell twins or WSL on Windows; and
`internal/storage/rotational.go` is `//go:build linux` with an `_other.go` fallback, so HDD/SSD
detection silently degrades off Linux (cosmetic, affects a log line).

**Recommendation, requiring an ADR-046 Addendum B rather than a unilateral change here:** relax
`engine_badger.go`'s constraint from `//go:build windows` to a `badger` build tag defaulting to
Windows, so a Linux demo can opt into the pure-Go engine with `-tags badger`. That would make the
entire demo `CGO_ENABLED=0` on every platform and reduce "can you run the demo" to "do you have Go
and Postgres." One session's work, and it materially changes how easily this can be handed to
someone. It is not folded into M17-E because it touches a storage-engine decision that has its own
ADR.

Short answer for M18 as currently scoped: **Windows, yes, trivially. Linux and macOS, yes, after a
toolchain install.** With Addendum B, yes everywhere with no toolchain at all.

### Q2 — Can providers join as separate desktops, making real peer-to-peer transfer happen?

**Not today — that is F-D-4 — and it is one flag away.**

The listener already binds `0.0.0.0`. Only the *advertised* address is wrong: hardcoded
`/ip4/127.0.0.1/…` at `main.go:501` and `:663`. Every dialer therefore connects to its own machine.
Phase 17.4 adds `--advertise-addr` with LAN-IP autodetection and uses it in both places.

**What becomes true after that fix, and it is worth stating precisely, because it is more than it
sounds like:**

- Real TCP sockets between real machines, carrying real shard bytes.
- Real TLS 1.3 with self-signed Ed25519 certificates verified against the expected Peer ID
  (`internal/p2p/host.go`) — genuine cryptographic peer authentication, not a stub.
- The microservice coordinates placement and **never touches shard bytes**. Data transfer is
  peer-to-peer in the meaningful sense: client → provider, and provider → provider during repair.

That is a true and defensible claim to make on a six-desktop LAN, and it is the heart of what
Vyomanaut promises.

**What remains untrue until M19, stated plainly so the claim is not overreached:**

- **No NAT traversal.** AutoNAT, DCUtR and Circuit Relay v2 are placeholders (`internal/p2p/nat.go`).
  Providers behind different home routers will not connect.
- **No real DHT over the network.** `internal/p2p/dht.go` is a substitute; Kademlia routing across
  hosts is not exercised.
- **No QUIC, no libp2p multiplexing, no stream negotiation.** TLS-over-TCP is ADR-063's honest
  substitute for QUIC v1 + Noise XX.
- **`relayAddrs` is a placeholder**, and `DemoProfile.MinRelayNodes = 0` — the readiness gate does
  not require relays because none exist.

So the honest ladder is: **same-LAN peer-to-peer is one flag away and belongs in M17-E.
Internet-scale peer-to-peer is M19**, where `build_part4.md` restores real `go-libp2p` with
AutoNAT/DCUtR/Circuit Relay v2 and the full conformance gate. The demo is not a mock of P2P that LTS
will replace; it is real P2P with the routing layer substituted, and M19 restores the routing layer.

`docs/DEMO.md` must state this boundary explicitly. A demonstration that lets an audience infer
internet-scale P2P from a LAN demonstration is a demonstration that has misled them, and this project
does not do that.

---

## Appendix A — the eleven founding functional requirements, verbatim

1. The interface allows a data owner to upload a file present on his desktop.
2. On the same desktop (or later on a separate machine), another user can volunteer as a provider
   (up to 7 providers in total).
3. The uploaded file gets encrypted and distributed among the providers present on the network.
4. A terminal can monitor a new node entry and also the log of the entire network health,
   performance, and transfers taking place (but never see the originally uploaded data).
5. The providers can store the file on their local machine and see encrypted data upon opening the
   file.
6. A mechanism to mark the provider online, as a heartbeat.
7. When the provider goes offline, repair takes place, ensuring that the file can be retrieved at any
   time by the data owner.
8. The data owner can fetch the file back, and the file has no changes.
9. A test to the provider verifies their storage and proves it can be retrieved easily. *(optional)*
10. A demo number acts as the payment sent by the data owner and gets equally split amongst the
    providers for the set duration of time. *(optional)*
11. The provider can choose how much storage he wishes to allocate. *(optional)*

*The three optional requirements are treated as mandatory for M17-E. Requirement 9 is the audit
system that already exists, requirement 10 is ADR-061's flat split that already exists, and
requirement 11 is `--declared-storage-gb` that already exists — in all three cases only the surface
is missing, and a demo that omits them understates what has been built.*
