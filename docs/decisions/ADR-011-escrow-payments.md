# ADR-011 — Fiat Escrow via UPI/Razorpay; India-First at Launch

**Status:** Accepted
**Topic:** #13 Escrow & Payment Basis (provider)
**Supersedes:** —
**Superseded by:** —
**Research source:** Papers 05, 07

---

## Context

All surveyed DSNs (Filecoin, Storj token, Swarm SWAP) use cryptocurrency for payments. Every one of them introduced friction that hurt adoption or failed in production. The system needs a payment method that works for Indian providers and data owners without requiring crypto wallets.

## Options Considered

| Option | Pros | Cons |
|---|---|---|
| Cryptocurrency (Storj token model) | Trustless; globally accessible | Price volatility; high onboarding friction; failed in production (Swarm) |
| Stripe Connect | Global; mature | Per-transaction fee; days to settle; requires Stripe account |
| **UPI via Razorpay or Cashfree** | Zero per-transaction fee; settles in seconds; universally available in India; no account friction | India-only; requires abstraction layer for future expansion |

## Decision

Fiat escrow exclusively at V2 launch. Payment via UPI using Razorpay or Cashfree as the payment gateway.

**Why India-first:**
- Ensures geographic proximity among providers, reducing transfer latency
- No currency conversion needed — all payments in INR
- UPI has zero per-transaction merchant fee
- UPI settles in seconds; Stripe Connect can take days
- UPI is available to every Indian with a bank account; Stripe requires additional account setup

**Abstraction layer (required from day one):**
The payment system is built behind a `PaymentProvider` interface:
```
initiateEscrow(data_owner_id, amount, contract_id)
releaseEscrow(provider_id, amount, audit_period)
penalise(provider_id, amount, reason)
getBalance(entity_id)
```
This allows Stripe or other gateways to be added without rewriting the payment logic.

**Escrow flow:**
Data owner deposits funds into escrow at contract start. The escrow holds funds until audit results release them to providers. The payment microservice handles all releases with idempotency keys to prevent double-payment (see [ADR-016](ADR-016-payment-db-schema.md)).

## Consequences

**Positive:**
- Zero per-transaction fee makes micro-payments economically viable
- Instant UPI settlement reduces provider cash flow friction
- Abstraction layer future-proofs the system for international expansion

**Negative / trade-offs:**
- India-only at launch limits the geographic scope of the provider network
- The payment microservice becomes a critical dependency — if it is down, no escrow releases can occur

**Open constraints:**
- International expansion requires the PaymentProvider interface to be implemented for a new gateway (Stripe Connect is the likely first candidate)

## References

- [Paper 07 — SoK](../research/paper-07-sok-dsn.md): all surveyed DSNs use crypto; Swarm SWAP failed in production
- [Paper 05 — Storj](../research/paper-05-storj.md): Storj token payment volatility
