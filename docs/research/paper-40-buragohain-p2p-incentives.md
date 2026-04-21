## Paper 40 — A Game Theoretic Framework for Incentives in P2P Systems

**Authors:** Chiranjeeb Buragohain, Divyakant Agrawal, Subhash Suri — University of California, Santa Barbara
**Venue / Year:** IEEE P2P 2003
**Topics:** #18, #8
**ADRs produced:** none — ADR-024 confirmed with one constraint clarified; ADR-008 confirmed; see Decisions Influenced

---

### Problem Solved

Free-riding in P2P systems — where 25% of Gnutella users share no files and 50% of sessions last under an hour — destroys system welfare because rational peers have no incentive to contribute when contribution is costly and consumption is free. Prior work either proposed monetary micropayments (impractical infrastructure) or reputation schemes (hard to quantify and game-resistant). This paper formalises P2P contribution as a non-cooperative game among rational players, models differential service as the incentive mechanism (a peer's probability of being served scales with their contribution), and proves two Nash equilibria exist when the benefit-to-cost ratio exceeds a critical threshold — with the higher equilibrium being uniquely stable under natural learning dynamics. For Vyomanaut, the paper validates that Vyomanaut's per-audit-passed payment model satisfies the conditions for the stable, high-contribution Nash equilibrium, and quantifies the critical benefit threshold that the storage rate must clear to avoid system collapse.

---

### Key Findings

**Two Nash equilibria when benefit b ≥ bc (Section 3.1):**
For a symmetric two-player game with contribution di and service probability p(d) = d^α / (1 + d^α), the Nash equilibria are at d*_lo < 1 (unstable) and d*_hi > 1 (stable). The critical benefit is bc = 4 for α = 1. Below bc, the only equilibrium is zero contribution — the system collapses. Above bc, the stable equilibrium scales monotonically with b, and the homogeneous N-player result simply replaces b with b(N−1).

**The high equilibrium is uniquely stable under Cournot learning (Section 3.3):**
Starting from any initial contribution above d*_lo, the iterative best-response process converges to d*_hi. Starting near zero, it converges to zero. This means a P2P system with above-threshold benefits is self-reinforcing: as long as the system bootstraps above the unstable equilibrium, it remains at the high-contribution stable point indefinitely.

**System robustness scales with size and benefit (Section 4.1.3):**
For bav/bc − 1 = 2.0, the system survives until 2/3 of peers depart before collapsing. At lower benefit levels the collapse threshold is lower. This is the opposite of centralized systems — P2P robustness increases with scale because each additional peer raises the per-peer benefit b(N−1).

**Heterogeneous results track homogeneous predictions within 1.5% (Section 4.1.2):**
For 500–1000 peers with bij drawn from a Gamma distribution, the average Nash equilibrium contribution d*_av differs from the closed-form homogeneous prediction by under 1.5%. The qualitative structure — two equilibria, one stable — is preserved under heterogeneity, varying α, peer churn, and uncooperative peers. The analytical result is robust.

**Uncooperative peers bias the equilibrium toward their contribution level (Section 4.1.3):**
A fraction of peers contributing a fixed low value pulls the Nash equilibrium downward proportionally. 100% uncooperative peers → equilibrium equals their contribution. This quantifies the infection risk: if too many providers defect from honest storage in Vyomanaut, the equilibrium contribution rate degrades proportionally — which is the formal motivation for audit-pass-contingent payment rather than unconditional payment.

**Contribution misreporting requires a neighbor audit scheme (Section 5.1):**
The paper identifies self-reported contribution as a manipulation vector and proposes that peers audit each other's disk space and uptime. This is the peer-audit approach that Proposition 1 of Paper 37 (SHELBY) proves collapses to universal dishonesty — the formal refutation of this paper's neighbor audit suggestion.

---

### Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Differential service (p(d) scales with contribution) | Monetary micropayments | No accounting infrastructure; incentive is built into the protocol; contribution must be observable and non-forgeable |
| Continuous contribution metric (disk-space × time) | Binary contribute/free-ride | Smooth Nash equilibrium computation; misreporting risk requires external audit |
| Pure strategy Nash equilibrium | Mixed strategy | Contribution levels are deterministic at equilibrium; mixed strategy would mean probabilistic contribution, which is inadmissible for contracted storage |
| Homogeneous analytical model validated against heterogeneous simulation | Full heterogeneous analytical solution | Closed-form result usable for parameter setting; heterogeneous deviation < 1.5%, making the approximation safe |

---

### Breaks in Our Case

- **Paper assumes peers directly observe each other's contribution (disk space, uptime) to implement p(di)** ≠ **Vyomanaut's audit pass rate is measured by the microservice, not by peer-to-peer observation**
→ The differential service probability p(di) maps to Vyomanaut's per-audit-pass payout: providers who pass more audits earn more, which is the functional equivalent of higher p(di). The microservice plays the role of the trusted measurer of di. No peer-to-peer contribution observation is needed or desired — the microservice's audit pass count is the contribution metric.
- **Paper's contribution metric is cumulative disk-space × time (a stock variable)** ≠ **Vyomanaut measures contribution as a flow — audit passes per period**
→ The per-audit-passed payout is mathematically equivalent to integrating disk-space × time only when audits are issued uniformly (one per 24h per chunk, ADR-006). Since our audit schedule is uniform, the two metrics coincide in expectation. The mapping is exact under Vyomanaut's audit design.
- **Paper's differential service is implemented as probabilistic request acceptance p(di)** ≠ **Vyomanaut's incentive is monetary — providers are paid per audit pass, not given differential access**
→ The game-theoretic structure is identical: utility = expected benefit − cost, where expected benefit is proportional to contribution (audit pass rate × payout per pass). Replacing p(di) × download benefit with payout(di) × audit earnings is a direct substitution with the same Nash equilibrium structure. The monetary version is strictly stronger as a deterrent because the payoff is transferable (real rupees) rather than just service access.
- **Paper assumes self-reporting of contribution is the manipulation vector** ≠ **Vyomanaut's contribution is measured independently by the microservice via PoR challenges that cannot be faked without the chunk (ADR-002)**
→ The misreporting attack the paper identifies is closed. A provider cannot inflate their audit pass rate without actually storing the data — SHA256(chunk_data || challenge_nonce) is not computable without chunk_data. The neighbor audit scheme the paper proposes as a defense is therefore unnecessary; the PoR design already guarantees non-forgeable contribution measurement.
- **Paper models voluntary P2P file-sharing with free entry and exit** ≠ **Vyomanaut has a 4–6 month vetting period, escrow seizure on exit, and non-transferable held earnings (ADR-024)**
→ The paper's system has bc as the sole entry barrier. Vyomanaut adds three additional barriers: temporal (vetting), financial (30-day escrow hold), and reputational (score history). These raise the effective bc for strategic defection far above the paper's critical value, pushing the stable equilibrium even higher. Vyomanaut's design is strictly more conservative than the paper's sufficient conditions require.

---

### Decisions Influenced

- **ADR-024 [#18 Economic Mechanism]** `CONFIRMED — NASH CONDITIONS SATISFIED`
The paper provides the game-theoretic validation that ADR-024 lacks. The per-audit-passed payout satisfies the paper's sufficient conditions for the stable high-contribution Nash equilibrium: (1) benefit b(N−1) exceeds bc (for any N > 1 provider where the storage rate is set above cost, the benefit-to-cost ratio exceeds the critical threshold), (2) the contribution metric (audit pass count) is non-forgeable via PoR, and (3) the iterative best-response process starting from any above-threshold initial condition converges to d*_hi. The paper also validates that robustness increases with provider count N — each new provider raises the per-peer benefit b(N−1), which pushes d*_hi higher and the collapse threshold further away. Vyomanaut's network gets more stable as it grows, not less.

One constraint added: the storage rate paise/GB/month must be set such that bc < b_actual for the median provider. The critical threshold bc = 4 (for α=1) means the ratio of total benefit per unit contribution to marginal cost must exceed 4. In Vyomanaut terms: (monthly_earnings / marginal_storage_cost) × (N−1) > 4. Since N will be hundreds at launch and marginal storage cost on a modern desktop is near zero, this condition is trivially satisfied at any positive storage rate. The participation constraint (rst ≥ cst from ADR-037/Paper 37) is the binding condition, not the Nash threshold.
*Because:* Section 3.1 proves the two-equilibrium structure and the uniqueness of the stable high equilibrium under Cournot learning. Section 4.1.2 validates the result holds for heterogeneous benefit matrices within 1.5% of the closed-form prediction.

- **ADR-008 [#8 Reliability Scoring]** `CONFIRMED`
The paper's uncooperative peer analysis (Section 4.1.3) shows that a fraction of low-contributing peers pulls the equilibrium contribution downward proportionally. This is the formal motivation for the three-window rolling score (ADR-008): providers whose audit pass rate falls signal exactly this pulling-down effect. Removing or downgrading low-scoring providers from chunk assignment (the Preference subsystem, ADR-005) is the correct response — it raises the effective bav of the remaining pool, pushing the equilibrium back toward d*_hi. The reliability scorer is not just a payment tool; it is the mechanism that enforces the stable Nash equilibrium by excluding peers whose contribution pulls others toward the unstable low equilibrium.
*Because:* Figure 8 shows that as the fraction of failed or absent peers increases, the average Nash equilibrium contribution degrades proportionally — confirming that purging low-contribution providers raises system welfare for all remaining providers.
- **ADR-005 [#5 Peer Selection — Preference Subsystem]** `CONFIRMED`
The stable Nash equilibrium d*_hi is the desirable equilibrium and the unstable d*_lo is the point below which the system collapses. The vetting period (4–6 months) serves as a bootstrap mechanism that ensures new providers demonstrate above-threshold contribution before receiving full chunk assignments. This is the practical implementation of ensuring the system starts above d*_lo: no provider with insufficient audit history receives the chunk volume that would make defection profitable.
Because: Section 3.3 shows the iteration converges to d*_hi only when initial conditions exceed the unstable fixed point. The vetting period provides exactly this guarantee by requiring observed contribution before trusting a new provider with full assignment.

---

### Disagreements

- **Paper's neighbor audit scheme (Section 5.1) as the misreporting defense:** The paper proposes that peers audit each other's disk space and uptime to prevent self-reporting manipulation. Paper 37 (SHELBY, Proposition 1) proves this collapses to universal dishonesty as a Nash equilibrium — peer audits without a trusted backstop are ineffective.
*Implication for us:* The microservice-as-auditor design (ADR-008) is not just an architectural convenience — it is the game-theoretically correct response to this paper's identified weakness. The neighbor audit the paper proposes would fail; our PoR-based microservice audit is the necessary and sufficient fix.
- **KaZaA's participation level as a contribution metric:** The paper cites KaZaA's uploads/downloads ratio as an alternative to disk-space × time. This metric rewards popularity rather than availability.
*Implication for us:* Audit passes are a superior metric for contracted storage because they measure continuous availability rather than retrieval popularity. A provider storing rarely-accessed archive data should earn equally to one storing popular data — the audit-pass metric achieves this.

---

### Open Questions

See open-questions.md — questions Q40-1 and Q40-2.