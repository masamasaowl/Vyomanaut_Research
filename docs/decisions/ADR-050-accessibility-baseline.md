# ADR-050 — Set WCAG 2.1 AA as the Desktop App Accessibility Baseline

**Status:** Accepted
**Topic:** #24 Accessibility & Institutional ICT Compliance (new topic)
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 52 — Indian ICT Accessibility Mandates and WebView-Hosted Application Accessibility

---

## Context

`ux-decisions.md` §5.2 names universities, offices, and hospitals — a population that includes a real share of government and government-funded institutions — as the institutional route to the provider audience, and commits the product to being "good enough that either path is viable whenever the business decides to pursue it": the same pre-scrutiny posture §5.2 already takes toward institutional IT policy on peer-to-peer software (§10.2, ADR-042). India's RPwD Act 2016, IS 17802, and GIGW 3.0 make accessibility exactly this kind of pre-scrutiny requirement for a meaningful fraction of that audience. §9.5's design system is still at the "starting primitives, not final values" stage — the cheapest point at which to decide a baseline, and the most expensive point at which to discover one is missing.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| No formal baseline; address accessibility only if or when an institutional customer requests it | Zero upfront cost; matches the general "business development happens after" posture §5.2 takes toward institutional timing | Retrofitting contrast ratios, focus management, and ARIA authoring into an already-built component library is materially more expensive than building it in from the start; risks failing a diligence check silently, at exactly the point an institutional deal is closest to landing |
| Pursue formal IS 17802 Part 2 conformance certification now | Removes any doubt for a future institutional procurement review | A heavier, audit-style process aimed at an organization ready to be assessed; committing to it now front-loads a business-development-timed cost ahead of the actual business decision §5.2 defers |
| WCAG 2.1 Level AA as a built-in design-system baseline, with a direct screen-reader smoke test — chosen | Matches the technical standard both IS 17802 and GIGW actually reference; costs little at the primitives stage; the smoke test catches Wails/WebView2-specific gaps a markup-only approach would miss | Real, if modest, ongoing design discipline cost (contrast ratios, focus states, ARIA on every custom component); does not by itself constitute formal IS 17802 certification if an institutional customer later requires the audited version |

## Decision

Both apps' UI target WCAG 2.1 Level AA as a built-in property of the §9.5 design system, not a later pass: every custom component in the shared `@vyomanaut/ui` package (ADR-049) is authored with correct semantic HTML and ARIA roles and states, a minimum 4.5:1 contrast ratio for normal text (3:1 for large text and UI component boundaries), full keyboard operability with no keyboard traps, and a visible focus indicator on every interactive element. Before the M19 milestone's shared component library is considered done, the first built screens are tested directly with Windows Narrator (built into Windows, requiring no separate install) against the actual packaged Wails build, not inferred from markup correctness alone — this specifically checks for the class of Wails/WebView2 accessibility-tree bridging gap already found to have a real history (WebView2Feedback#2330; wailsapp/wails#4535). Formal IS 17802 Part 2 conformance certification is not pursued at this stage; it is deferred to whenever institutional go-to-market timing (§5.2, a business decision) makes it relevant.

## Consequences

**Positive:**

- Costs the least it will ever cost, since it is built into the design system's primitives instead of retrofitted after both apps' component libraries are fully built out
- Directly derisks the government and government-funded slice of §5.2's named institutional audience without requiring that audience to be engaged, or a market decision to be made, first
- Gives engineering a concrete, checkable bar — specific contrast ratios, keyboard operability, a named smoke-test tool — rather than an abstract "be accessible" instruction

**Negative / trade-offs:**

- Adds real, ongoing authoring discipline to every custom component for the life of the design system, not a one-time cost
- The Narrator smoke test only exercises the Windows/WebView2 path — it does not verify macOS (VoiceOver/WebKit) or Linux (Orca/WebKitGTK, where a real, currently open Wails accessibility gap already exists), consistent with §7's own Windows-first sequencing but not a substitute for the equivalent check when those platforms are built

**Open constraints:**

- This baseline assumes ordinary WCAG technique (semantic HTML, ARIA, focus management) is sufficient once the Wails/WebView2 accessibility-tree bridge is confirmed working for this specific build — not yet independently verified end-to-end. Tracked as Q52-1; not blocking, since the smoke test above is exactly the verification step, scheduled into M19 rather than left informal.
- Holds only as a baseline, not a certified conformance claim. If an institutional customer's procurement process requires an audited IS 17802 Part 2 conformance statement, that is a separate, heavier engagement to scope when it is actually asked for, not implied by this ADR.

## References

- [Paper 52 — Indian ICT Accessibility Mandates and WebView-Hosted Application Accessibility](../research/paper-52-accessibility-mandates-webview-at-support.md)
- [ADR-049 — Adopt Svelte + TypeScript for the Wails UI Layer](ADR-049-svelte-frontend-framework.md) — the design-system package this baseline is built into
- [ADR-038 — Desktop Shell: Wails](ADR-038-desktop-shell-wails.md) — the WebView2 hosting layer this baseline's smoke test verifies
- `ux-decisions.md` §5.2, §9.5, §9.7
