<!-- This is a demo design map, future versions to include more detailed and reader-friendly documentation -->

Step 1: Provider downloads and installs daemon
→ Daemon generates Ed25519 key pair on first run
→ Key pair stored at ~/.vyomanaut/keys/ (encrypted with passphrase)

Step 2: Provider registers via app/web UI
→ POST /api/v1/provider/register
{ phone_number, otp, ed25519_public_key, declared_storage_gb, declared_region }
→ Microservice creates provider record with status = PENDING_VERIFICATION
→ Microservice initiates Razorpay Linked Account creation (async)

Step 3: KYC verification (OTP + phone number)
→ If OTP valid: status → PENDING_ONBOARDING
→ If OTP invalid: return 401; provider must retry

Step 4: Razorpay Linked Account cooling period (24 hours)
→ Microservice polls Razorpay until Linked Account is active
→ No chunk assignments are made during this window

Step 5: Daemon establishes first heartbeat (ADR-028)
→ POST /api/v1/provider/heartbeat with current multiaddr
→ Microservice verifies signature, stores multiaddr
→ status → VETTING

Step 6: First chunk assignment (vetting mode)
→ Assignment service selects a non-critical chunk
→ Extra erasure redundancy applied (per ADR-005 vetting design)
→ Provider receives chunk via libp2p P2P transfer
→ Provider stores chunk, sends signed upload receipt

Step 7: Vetting period (4–6 months)
→ Daily audit challenges issued
→ Escrow accrues with 60-day hold, 50% release cap (ADR-024)
→ Score accumulates in three rolling windows (ADR-008)

Step 8: Vetting completion
→ After 80+ consecutive audit passes (Paper 05 Jeffrey's prior threshold):
status → ACTIVE
→ Hold window reverts to 30 days, release cap removed (ADR-024)
→ Provider receives full chunk assignment rate

Failure states:

- Registration with duplicate phone number → 409 Conflict
- Razorpay Linked Account creation failure → retry up to 3 times, then manual review
- No heartbeat within 24h of registration → status → STALLED; provider must restart daemon
- Provider fails > 50% of vetting audits → status → FAILED_VETTING; escrow forfeited; re-registration requires new phone number