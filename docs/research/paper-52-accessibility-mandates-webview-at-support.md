# Paper 52 — Indian ICT Accessibility Mandates and WebView-Hosted Application Accessibility: RPwD Act 2016, IS 17802, GIGW 3.0, and WebView2/WebKitGTK Assistive-Technology Support

**Authors:** Government of India, Ministry of Law and Justice (RPwD Act 2016); Bureau of Indian Standards / Ministry of Electronics and Information Technology (IS 17802); National Informatics Centre (GIGW 3.0); Microsoft (WebView2 and Windows App SDK documentation and issue tracker); Wails project maintainers and community (wailsapp/wails)
**Venue / Year:** Government of India legislation and national standards (RPwD Act enacted 2016; IS 17802 notified 2023; GIGW 3.0 published 2023); Microsoft Learn and the WebView2Feedback GitHub issue tracker; wailsapp/wails GitHub Discussions | retrieved 2026
**Citations:** not applicable — legislation, a national standard, and vendor/project documentation and issue trackers, not an academic paper
**Topics:** #24 Accessibility & Institutional ICT Compliance (new topic), #22 Desktop Application Shell Architecture
**ADRs produced:** ADR-050 (Desktop App Accessibility Baseline); introduces Topic #24

---

## Problem Solved

`ux-decisions.md` §5.2 names universities, offices, and hospitals as the institutional route to the provider audience, and commits the product to being "good enough that either path is viable whenever the business decides to pursue it" — the same pre-scrutiny posture §5.2 already takes toward institutional IT policy on peer-to-peer software (§10.2, ADR-042). A meaningful share of that named audience — government and government-funded universities and hospitals — is legally required to check software against an accessibility standard before adopting it. §9.5's design system is still at the "starting primitives, not final values" stage: the cheapest point at which to build an accessibility baseline in, and the most expensive point at which to discover later that one is missing.

---

## Key Findings

**India's ICT accessibility mandate is not merely a website concern, and applies to a meaningful share of the named institutional audience.** The Rights of Persons with Disabilities Act 2016, Sections 40–46, requires government bodies to formulate and enforce ICT accessibility standards. IS 17802, notified by the Bureau of Indian Standards in 2023 and based on the European EN 301 549 standard, covers the full ICT product category — software and hardware, not just websites and apps. GIGW 3.0, the Guidelines for Indian Government Websites and Apps, makes WCAG 2.1 Level AA compliance mandatory for public-sector digital platforms.

**WCAG is the operative technical target regardless of which specific Indian instrument applies.** IS 17802 is itself built on EN 301 549, which incorporates WCAG's success criteria, and GIGW references WCAG 2.1 AA directly. There is one technical bar to build against, not three separate ones.

**Because both apps' UI is Svelte rendered inside a native webview (ADR-049), the correct implementation technique is ordinary web accessibility technique, with no separate Wails-specific accessibility API required.** Semantic HTML, correct ARIA roles and states, keyboard operability, visible focus indicators, and sufficient color contrast are the actual unit of work — the same unit of work any WCAG-conformant website requires, not something Wails adds a new layer on top of.

**The bridge from correct markup to an assistive technology actually consuming it is not automatically true in this stack, and has a real, documented history of gaps at exactly the layers this project depends on.** Microsoft's own Windows App SDK 1.5 release notes confirm a fix for a WebView2 bug where a window containing only a WebView2 control did not correctly set initial keyboard focus into the control, "leaving it unusable by keyboard and accessibility tools" — the same bug tracked as WebView2Feedback issue #2330, filed against Windows App SDK and WebView2 in 2022. That fix landed in Windows App SDK specifically; Wails hosts WebView2 through its own bindings, not through Windows App SDK/WinUI, so whether the fix's code path covers Wails' own hosting approach is not established by this finding alone. Separately, and more directly relevant to Wails itself, a currently open wailsapp/wails GitHub discussion (#4535) documents a live bug: a screen reader (Orca) does not announce any content in a Wails app on Linux unless the window is first clicked with a mouse, even when the window has keyboard focus. An open pull request aims to fix the underlying startup-focus behavior. The same discussion states the general architecture plainly: Wails "relies on the underlying WebKitGTK and the frontend's accessibility markup for integration with AT-SPI and Orca" — confirming there is no separate Wails accessibility API layer, and that the quality of the bridge depends on both the specific native webview and the frontend's own markup being correct.

**No source found confirms, one way or the other, that Wails' specific WebView2 hosting on Windows has this class of bug fixed or open.** The Windows App SDK fix and the Wails/Linux bug are the two closest, most relevant data points available, and they point in opposite directions on confidence — one shows Microsoft actively fixing this exact class of issue, the other shows it currently unresolved in Wails on a different platform. Treat this as unverified for the Windows/WebView2 path specifically until tested directly.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| WCAG 2.1 Level AA as a design-system-wide baseline, built in from the first component | Treating accessibility as a post-launch or institutional-request-only addition | Costs real, upfront design discipline (contrast ratios, focus states, ARIA authoring on every custom component) before any screen is fully built — far cheaper than retrofitting it into a shipped design system, and removes a real diligence-check risk for the government/government-funded slice of §5.2's institutional audience, at the cost of some design iteration speed early on |
| A direct Windows Narrator smoke test on the first built screens, before the shared component library is considered done | Assuming WCAG-correct markup is sufficient because the underlying pieces (WebView2, ARIA) are each individually capable | Catches a Wails-hosting-specific gap — of exactly the kind already found live in a Wails/Linux discussion — before it's buried under months of component work, at the cost of one deliberate manual test pass added to the M19 milestone |
| A baseline aimed at WCAG 2.1 AA conformance, not formal IS 17802 Part 2 certification, at this stage | Pursuing a certified conformance audit now | Matches what GIGW and institutional procurement actually check for in practice, without taking on a formal certification process ahead of engaging any actual institutional customer — certification, if ever needed, is timed to the business decision §5.2 already defers, not front-loaded into engineering now |

---

## Breaks in Our Case

- **GIGW and most public discussion of Indian ICT accessibility law is written primarily with government websites and mobile apps in mind**
  ≠ **Vyomanaut is a native desktop application, installed and run outside a browser**
  → IS 17802's own scope — software and hardware, not just websites, inherited from EN 301 549 — is what actually covers this case. GIGW's WCAG 2.1 AA reference point remains the practical technical target to build against even though GIGW's own stated scope is narrower than the whole standard.

- **WCAG assumes the accessibility tree bridge between correct HTML/ARIA markup and the assistive technology consuming it just works, because that's true of a standards-compliant browser**
  ≠ **that bridge is the native webview's and Wails' job here, not a browser's, and it has a real, confirmed history of gaps — a Microsoft-confirmed WebView2 startup-focus bug (fixed in Windows App SDK 1.5, applicability to Wails' own hosting unverified) and a currently open Wails-specific screen-reader focus bug on Linux**
  → Correct markup is necessary but not sufficient. The Windows-first build needs its own direct verification — a Narrator smoke test against the actual packaged app — not an inference from either the WCAG spec or a fix that landed in a different Microsoft framework.

- **IS 17802's Part 2 conformance process assumes an organization ready to be formally audited against it**
  ≠ **Vyomanaut has not engaged any institutional customer yet, and §5.2 explicitly treats institutional go-to-market timing as a business decision, not a product one**
  → Building to the WCAG 2.1 AA technical bar now is cheap and reversible in the useful direction; committing to formal IS 17802 certification now would front-load a business-development-timed cost ahead of the actual business decision §5.2 defers.

---

## Decisions Influenced

- **[ADR-050](../decisions/ADR-050-accessibility-baseline.md) [#24 Accessibility & Institutional ICT Compliance]** `NEW`
  WCAG 2.1 Level AA as the target conformance level for both apps' UI, built into the §9.5 design system's primitives from the start, with a direct Windows Narrator smoke test added to the M19 milestone before the shared component library is considered done. Formal IS 17802 certification deferred to institutional go-to-market timing.
  *Because:* it is the technical standard both IS 17802 and GIGW actually reference, it is cheap to build in now and expensive to retrofit later, and the Wails/WebView2 accessibility bridge has a real, confirmed history of gaps that markup correctness alone does not catch.

---

## Disagreements

- **A case can be made that WCAG AA is more rigor than a pre-institutional-engagement product needs right now**, and that the smarter sequencing is to build the design system freely and retrofit accessibility once, or if, an institutional customer specifically asks.
  *Implication for us:* this paper does not resolve that trade-off on general principle — it takes the position that, because §9.5's design system is still at the primitives stage, the marginal cost of building this in now is close to zero, which is the actual argument for doing it now, not a claim that accessibility work is always worth its cost regardless of timing. If the design system were already fully built, this paper's conclusion would not automatically transfer to a retrofit decision.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q52-1.
