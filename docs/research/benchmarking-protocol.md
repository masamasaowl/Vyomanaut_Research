## Benchmark Protocols for Q16-1, Q18-1, Q27-1

Execute these on a **minimum-spec test machine** before the relevant subsystem is built:

- CPU: dual-core, ≤1.8 GHz, Intel Celeron / old Pentium (confirmed no AES-NI)
- RAM: 2 GB
- Storage: consumer 7200 RPM HDD (not SSD)
- OS: Ubuntu 22.04 LTS or equivalent

### Q16-1 — AONT Encoding Throughput (ChaCha20 path)

**Purpose:** Confirm that a 14 MB segment can be AONT-encoded within ≤200ms on minimum-spec hardware without AES-NI.

`Step 1: Verify AES-NI is absent.
  $ grep -o 'aes' /proc/cpuinfo | head -1
  If this returns nothing, AES-NI is absent. Proceed.
  If it returns 'aes', switch to a different machine.

Step 2: Install dependencies.
  $ pip install pycryptodome --break-system-packages
  (or use Go/Rust implementation — preferred for production accuracy)

Step 3: Implement AONT encode (ADR-022 Step 2–4).
  Input: 14 MB random plaintext (56 × 256 KB)
  Algorithm:
    K = os.urandom(32)                          # AONT key
    for i, word in enumerate(words):            # 16-byte words
        c_i = word XOR ChaCha20(K, counter=i)  # encrypt each word
    h = SHA256(c_0 || c_1 || ... || c_s)       # commit hash
    c_{s+1} = K XOR h                           # embed key

Step 4: Time 100 iterations.
  $ python3 -c "
  import time, os, hashlib
  from Crypto.Cipher import ChaCha20
  results = []
  for _ in range(100):
      plaintext = os.urandom(14 * 1024 * 1024)
      K = os.urandom(32)
      t0 = time.perf_counter()
      cipher = ChaCha20.new(key=K, nonce=b'\x00'*8)
      ciphertext = cipher.encrypt(plaintext)
      h = hashlib.sha256(ciphertext).digest()
      c_last = bytes(a^b for a,b in zip(K, h[:32]))
      results.append(time.perf_counter() - t0)
  results.sort()
  print(f'median={results[50]*1000:.1f}ms p95={results[95]*1000:.1f}ms p99={results[99]*1000:.1f}ms')
  "

Step 5: Evaluate.
  PASS: median ≤ 200ms, p99 ≤ 400ms — ChaCha20 path fits within ≤5% CPU budget.
  FAIL: If median > 200ms, investigate:
    (a) Is ChaCha20 vectorised (SSSE3/SSE4)? Check CPU flags for 'ssse3'.
    (b) If yes and still slow, Python overhead is the issue — repeat with Go or Rust.
    (c) If Go/Rust also fails: the 5% budget must be re-evaluated or segment size
        must be reviewed (but do not change lf without updating ADR-003, ADR-004).

Step 6: Record result in the build log with machine specs.`

### Q18-1 — Argon2id Session Start Latency

**Purpose:** Confirm that master secret derivation with t=3, m=64MB takes ≤500ms on minimum-spec hardware.

`Step 1: Install argon2-cffi.
  $ pip install argon2-cffi --break-system-packages

Step 2: Run benchmark.
  $ python3 -c "
  import time
  from argon2 import PasswordHasher
  ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=4,
                      hash_len=32, salt_len=16)
  times = []
  for _ in range(10):
      t0 = time.perf_counter()
      ph.hash('test_passphrase_representative_length_32ch')
      times.append(time.perf_counter() - t0)
  times.sort()
  print(f'median={times[5]*1000:.0f}ms min={times[0]*1000:.0f}ms max={times[-1]*1000:.0f}ms')
  "

Step 3: Evaluate.
  PASS: median ≤ 500ms — UX acceptable for session start (one-time per session).
  PARTIAL PASS: 500ms–1000ms — acceptable with a spinner UI, no parameter change needed.
  FAIL: > 1000ms — reduce parameters in order:
    First try: t=2, m=65536 (reduce time_cost from 3 to 2)
    If still failing: t=3, m=32768 (reduce memory from 64MB to 32MB)
    If still failing: t=2, m=32768
    Stop at the first combination that yields median ≤ 500ms.
    Update ADR-020 with the confirmed parameters and machine spec.

Step 4: Record result: chosen (t, m, p) values with machine specs.
        Note: these parameters must be the minimum acceptable for security.
        OWASP 2023 recommends t≥1, m≥64MB for interactive login. Do not go below m=32768.`

### Q27-1 — RocksDB Rate Limiter Calibration

**Purpose:** Find the highest compaction rate limiter that keeps p99 audit response latency ≤ 100ms during concurrent writes.

`Step 1: Set up a WiscKey prototype.
  - RocksDB instance: chunk_id (32 bytes) → (vlog_offset uint64, size uint32) = 44 bytes
  - vLog file: append-only, 256 KB entries
  - Rate limiter set to 10 MB/s initially

Step 2: Populate with 200,000 chunks (50 GB equivalent at 256 KB each).
  - Use synthetic data (random bytes per chunk is fine).
  - This populates the vLog and builds the RocksDB index.

Step 3: Run concurrent workload for 10 minutes.
  Thread A (write loop):   continuously append new 256 KB chunks to vLog + RocksDB
  Thread B (audit loop):   issue 1 random point lookup per second (audit challenge simulation)
                           measure time from RocksDB lookup to vLog read completion

Step 4: Collect metrics.
  - Thread B: record p50, p95, p99 audit response latency in ms.
  - Thread A: record write throughput in MB/s.

Step 5: Repeat with rate limiters: 5 MB/s, 10 MB/s, 20 MB/s, 40 MB/s, unlimited.

Step 6: Find the crossover.
  Target: highest rate limiter where p99 audit latency ≤ 100ms.
  Set ADR-023's default rate limiter to that value.
  Record result: (rate_limiter_MB_s, p99_audit_ms, write_throughput_MB_s) per test.

Step 7: Sanity check.
  At chosen rate limiter, write throughput should be ≥ 2 MB/s.
  A provider receiving new data at 5 Mbps upload = 0.625 MB/s — this is well within budget.`

---

## Q17-1: Hardware Survey Method

**The question:** What fraction of Indian desktop providers will have AES-NI?

**Why it matters:** ADR-019 specifies two cipher paths. If >95% of providers have AES-NI, the ChaCha20 path becomes a low-priority fallback. If <80%, both paths must be production-quality from day one.

**How to answer it — three methods, in order of effort:**

**Method 1 — CPU model correlation (fastest, do this first)**

India's desktop/laptop market data shows CPU distribution by model families. Cross-reference with Intel/AMD AES-NI support charts:

- Intel Core i3/i5/i7/i9 ≥ 2011 (Sandy Bridge and newer): AES-NI present
- Intel Pentium G-series ≥ 2011: AES-NI present (G630 and above)
- Intel Celeron G-series ≥ 2012: AES-NI present (G530 and above)
- Intel Atom (Bay Trail, Silvermont): AES-NI absent or unreliable
- AMD Ryzen all generations: AES-NI present
- AMD FX (Bulldozer/Piledriver 2011–2014): AES-NI present
- Very old AMD (pre-2011): AES-NI absent

**Practical estimate for Indian V2 desktop providers (2025):**
The typical Indian home desktop or laptop used as a Vyomanaut provider will be a machine purchased in 2015–2023. Budget laptops at ₹20,000–₹50,000 price point almost universally run Intel Core i3 (Sandy Bridge or later) or AMD Ryzen 3. AES-NI prevalence: **estimated 85–90%.** The remaining 10–15% are very old or very budget machines (Celeron N-series, Atom).

**Method 2 — Steam Hardware Survey (quantitative)**

Steam's hardware survey shows CPU model distribution globally and by region. India is available as a filter. Steps:

1. Visit: https://store.steampowered.com/hwsurvey/
2. Filter: India region
3. Note CPU family distribution (Core i3/i5/i7 vs Celeron/Atom percentages)
4. Cross-reference Intel's AES-NI support matrix: https://ark.intel.com/

**Method 3 — Provider onboarding telemetry (definitive)**

At V2 beta launch, add a one-time hardware detection step to the provider daemon:

go

`func hasAESNI() bool {
    // x86: check CPUID EAX=1, ECX bit 25
    // ARM: check via getauxval(AT_HWCAP) & HWCAP_AES
}`

Report the result to the microservice at registration. After 100 beta registrations, the answer is known definitively. This is the production answer.

**Action:** Implement the CPUID check from day one. The code is trivial (5 lines in Go/Rust), adds zero latency, and gives you the ground truth from real providers.

**Conclusion for code strategy:** Both paths must be production-quality. The ChaCha20 path is not a fallback — it is the primary path for a non-trivial fraction of providers. The AES-NI detection at daemon startup (already specified in ADR-019) is mandatory, not optional.

---

## Q18-2: Ed25519 Signing Key — Derived vs Stored

**Decision:** Store the signing key separately in the local key store, encrypted under a key derived from the master secret. Do not derive the signing key from the master secret directly.

**Analysis:**

|  | Derive from master secret | Store separately (chosen) |
| --- | --- | --- |
| Recovery | Perfect — passphrase recovers everything | Key store is a second backup target |
| Rotation | Cannot rotate without rotating master secret → re-derives all file keys → re-encrypts all pointer files → catastrophic | Rotate at any time, re-sign only new pointer files going forward |
| Compromise response | Compromised signing key = must rotate master secret = full re-encryption of all data | Compromised signing key = generate new key, update microservice record, re-sign new files lazily |
| Key store loss | Not applicable | Lose key store + know passphrase → generate new signing key, re-sign all pointer files from microservice ciphertext |

The compromise scenario is the deciding factor. A stolen signing key allows a forged pointer file. If the signing key is derived from the master secret, the only response is to rotate the master secret — which is equivalent to re-keying the entire account. If the signing key is stored separately, a new one can be generated and the old one invalidated at the microservice without touching any data or file keys.

**Implementation spec:**

go

`// At daemon installation:
signingKey = Ed25519.generate()
storageKey = HKDF(master_secret, "vyomanaut-keystore-v1")
encryptedSigningKey = AEAD_CHACHA20_POLY1305(storageKey, signingKey)
// Store encryptedSigningKey in local key store file alongside the nonce counter.

// At session start:
master_secret = Argon2id(passphrase, owner_id)
storageKey = HKDF(master_secret, "vyomanaut-keystore-v1")
signingKey = AEAD_CHACHA20_POLY1305.decrypt(storageKey, encryptedSigningKey)
// Use signingKey for this session.`

The key store now contains: `{encrypted_signing_key, pointer_file_nonce_counter}`. Both require the master secret to access. This is a single file, not multiple backup targets — it is already required for the nonce counter (Q17-2 from ADR-019).

---