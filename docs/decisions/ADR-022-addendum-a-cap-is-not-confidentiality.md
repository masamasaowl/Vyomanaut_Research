# ADR-022 Addendum A — The ASN Cap Is Not a Confidentiality Control

**Status:** Accepted
**Track:** LTS
**Topic:** #3 Confidentiality / #5 Peer Selection
**Amends:** ADR-022 (the encryption/erasure *order* stands; the claim made for the ASN cap does not)
**Research source:** Design Council session 4, August 2026

---

## The claim being withdrawn

ADR-022 states that the 20% ASN cap *"keeps the threshold safe."* It does not, and F-34 quantified
why: the cap permits `⌊56 × 0.20⌋ = 11` shards per ASN, so two colluding ASNs hold **22** against
AONT-RS's disclosure threshold of **16**. The durability margin is 29 shards; the confidentiality
margin is **−6**.

Council 4 established the more general point. **A placement cap cannot repair a collusion threshold.**
A cap constrains how shards *may be distributed*; collusion is what holders do *afterwards*.
Cap-tuning raises the attacker's price and does not restore the property.

The arithmetic bounds even that mitigation. To make two-ASN collusion insufficient:
`2 × ⌊56f⌋ < 16` → `⌊56f⌋ ≤ 7` → `f < 0.143`, requiring **≥ 8 genuinely independent ASNs** to place
56 shards at all. And at `f = 0.143`, **three** colluding ASNs still reach 21 ≥ 16. Cap-tuning buys
exactly one coalition size.

## Three findings that make it worse

1. **The exposure is maximal at launch.** ADR-029 gates bootstrap at ≥56 providers across ≥5 ASNs.
   `5 × 20% = 100%` — the cap is exactly saturated, and any two ASNs is 40% of the network. The
   system is least confidential on day one and grows *into* safety.
2. **The cap is enforced against a self-declared field.** ADR-070's open-items list records that real
   ASN detection is unimplemented and every registration supplies `demo_asn`. A Sybil operating 16
   identities declaring five ASNs defeats the cap at the cost of 16 phone numbers. **The control does
   not currently function at all.** (F-LTS-10)
3. **CGNAT undercuts even correct detection.** A carrier ASN may cover millions of subscribers, so
   ASN diversity can overstate failure-domain *and* collusion-domain diversity simultaneously.

## Decision

1. **Remove the confidentiality claim from ADR-022** and from every other document where the ASN cap
   is cited as protecting the disclosure threshold. The cap remains a **durability** control under
   ADR-014, which is what it was sized for and where it is correct.
2. **Ship a compiler-enforced invariant, documented as making the gap loud rather than closed.**
   Following `TestProfileShardSizeIsConstant`'s precedent:

   ```go
   // TestProfileConfidentialityMarginHolds
   // FAILS TODAY on the production profile: 2*⌊56*0.20⌋ = 22 >= DataShards = 16.
   // This is intentional. It fails until ADR-022's successor lands a real fix.
   if 2*int(float64(p.TotalShards)*p.ASNCapFraction) >= p.DataShards { t.Fatal(...) }
   ```

   F-67 is structural — `DataShards` and `⌊TotalShards × ASNCapFraction⌋` are independent
   `NetworkProfile` fields with no expressed relationship — so a one-off fix returns with the next
   profile. This makes the relationship permanent and machine-checked.
3. **F-34 is re-ranked below F-69.** Under ADR-076's F-LTS-08 finding, the operator assembles 16
   shards on every repair *by design*. Two-ASN collusion is the second-worst case. Both are fixed by
   the same structural work (Domains P and K), which is further reason to move Confidentiality
   earlier in the milestone map.
4. **R-17 becomes the feasibility gate**, promoted from Domain E to Domain K and from Band 2 to
   Band 1: how many genuinely independent ASNs an Indian consumer provider pool can span, treating
   CGNAT. **If the answer is < 8, cap-tuning is off the table entirely and R-30 is the only
   instrument.** R-17 absorbs F-LTS-10.
5. **Settle the product claim before any owner-facing copy exists.** "Zero-knowledge" and "the
   service never sees plaintext" are absolute claims; a placement cap is a probabilistic,
   operator-enforced mitigation against one adversary shape. Recommend stating **computational
   confidentiality resting on client-held key material** — still a strong claim, and the one that is
   true.

## Consequences

One test that fails on the production profile today and should. One claim withdrawn from ADR-022 and
its dependents. R-17 changes domain and band. Domains P and K become the Confidentiality milestone's
content, and F-34's fix is deferred to them rather than attempted through cap tuning.

**Open constraints:** real ASN detection remains unimplemented — this addendum does not schedule it,
and until it exists the invariant test above is the only honest signal about the cap's state.

## References

- ADR-014 — where the ASN cap correctly lives, as a durability control
- ADR-022 — amended here; the encryption/erasure order is unaffected
- ADR-029 — the bootstrap gate at which exposure is maximal
- ADR-070 — records that real ASN detection is unimplemented
- ADR-076 — F-LTS-08, which re-ranks F-34 below F-69
- `reading-list.md` §5 Domain K — R-17, R-30, R-31
