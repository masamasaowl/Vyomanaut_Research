# Paper 54 — Locale-Ready Copy and Number Formatting for a Copy-Table-Driven UI: Paraglide JS and the Indian Numbering System

**Authors:** Opral / the Paraglide JS project (opral/paraglide-js); SvelteKit documentation team (Paraglide JS's official SvelteKit integration); ECMA-402 / MDN Web Docs (Intl.NumberFormat)
**Venue / Year:** Paraglide JS official documentation (paraglidejs.com) and GitHub repository, retrieved 2026; MDN Web Docs, Intl.NumberFormat reference, retrieved 2026
**Citations:** not applicable — library documentation and a web-platform standard reference, not an academic paper
**Topics:** #20 Client-Facing UX & Copy, #22 Desktop Application Shell Architecture
**ADRs produced:** ADR-052 (Adopt Paraglide JS and en-IN Number Formatting)

---

## Problem Solved

`interface-contracts.md` §14 (the user-facing copy contract, ADR-034) hasn't merged yet, and `ux-decisions.md` §11 defers the English-vs-multilingual V1 scope question to a later business decision — correctly, since it's not a product question the engineering foundation needs to answer today. But two adjacent technical questions are not business decisions and were never actually raised: how the copy table gets wired into Svelte components once it does merge, and whether displaying rupee amounts correctly for an Indian audience is itself contingent on that later multilingual decision at all. Both are cheap to settle now, at the same "design-system primitives" stage §9.5, §9.6, and §9.7 were each settled at, and expensive to retrofit across two full component libraries' worth of hardcoded strings later.

---

## Key Findings

**Paraglide JS is SvelteKit's officially recommended i18n integration, and it also works in plain Vite projects without SvelteKit.** It's a compiler-based library: message definitions compile into typed, tree-shakable functions rather than runtime key-value lookups, which the project's own documentation credits with up to 70% smaller i18n bundle sizes than runtime-based alternatives like i18next or svelte-i18n. Its own framing states it directly: "Compiler-first i18n for React, TanStack Start, SvelteKit, and any Vite app" — the "any Vite app" case is the relevant one here, since ADR-049 already chose plain Svelte plus Vite in library mode, not SvelteKit.

**Paraglide's SvelteKit-specific features — localized URLs, routing, SSR reload behavior — don't apply to this project and aren't needed for it to work.** Its documented SvelteKit guide covers things like triggering a full reload on locale switches and syncing localized URLs across client-side navigation; none of that is relevant to a Wails-embedded app with no routing or server-rendering. The part that matters here is narrower and simpler: message definitions in, typed message functions out.

**The compiler-over-runtime approach matches the same reasoning ADR-049 already used to pick Svelte itself.** Both push work to build time rather than shipping a heavier runtime — the same footprint argument, applied one layer further.

**Separately, and not gated on any multilingual decision at all: JavaScript's native `Intl.NumberFormat` already knows the Indian numbering system, at zero additional dependency cost.** The Indian numbering system groups digits differently from the international convention — thousands first, then pairs of digits (lakh, crore) rather than groups of three throughout. `Intl.NumberFormat("en-IN")` produces this grouping natively (confirmed directly in MDN's own reference examples: formatting 123456.789 under `en-IN` yields `1,23,456.789`, against `123,456.789` for a Western locale), and `Intl.NumberFormat("en-IN", { style: "currency", currency: "INR" })` produces a correctly grouped, correctly symbolled rupee amount directly — no manual regex formatting, no separate library. This is a browser/webview platform feature already available inside WebView2's Chromium engine, not something that needs to be added.

**This means Indian number formatting is a today-correctness issue, not a someday-localization issue.** An app displaying "₹1,234,567" instead of "₹12,34,567" to the exact Indian audience §5.2 and §8 are built for reads as subtly foreign, independent of whether the surrounding text is in English, Hindi, or anything else. It should not be bundled with, or deferred alongside, the language-scope business decision §11 already correctly defers.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Paraglide JS, wired to plain Svelte + Vite | A runtime i18n library (svelte-i18n, i18next) | Matches ADR-049's own compiler-over-runtime reasoning and ships less code, at the cost of a compile step that regenerates message functions whenever `interface-contracts.md` §14's copy table changes — a real but small addition to the build, not a new category of tooling |
| Authoring all user-facing strings as keyed Paraglide messages now, even while V1 ships English-only | Writing plain hardcoded strings into Svelte components now, and extracting them into a message system later if a second language is ever needed | Costs a small amount of upfront structure (every string goes through a message function instead of being typed inline) in exchange for not having to retrofit that structure across two full component libraries after the fact — the same "cheap now, expensive later" shape §9.7 already used for accessibility |
| Native `Intl.NumberFormat("en-IN", ...)` for all displayed numbers and rupee amounts | Hand-rolled formatting logic, or default/browser-locale-based formatting | Zero new dependencies, correct by construction, and — unlike the copy-table question — not something that needs to wait for any future decision at all |

---

## Breaks in Our Case

- **Paraglide JS's own documentation and most public discussion of it assume a SvelteKit project, with routing and locale-prefixed URLs as part of the value proposition**
  ≠ **ADR-049 already committed to plain Svelte plus Vite, with no SvelteKit, no routing, and no server-rendering anywhere in either app**
  → Only the compiler/typed-message-function core of Paraglide is relevant; its SvelteKit-specific routing and URL features are simply unused, not a blocker and not a reason to reconsider ADR-049.

- **`interface-contracts.md` §14, the copy table this decision wires into, hasn't merged yet**
  ≠ **Paraglide needs an actual message source file to compile against**
  → This doesn't block adopting the pattern now — an empty or minimal `en.json` message file can exist and grow as §14 merges — but the two pieces of outstanding work are naturally sequenced together: merging §14 is, in practice, authoring the first locale's message file.

- **The Indian-numbering-system finding sits right next to the localization question in this same paper**
  ≠ **it isn't actually the same question, and bundling it with the deferred language-scope business decision would incorrectly defer a plain correctness fix along with a genuine business timing question**
  → Called out explicitly in the Decision below as unconditional — it ships regardless of what §11's language-scope decision ends up being or when it's made.

---

## Decisions Influenced

- **[ADR-052](../decisions/ADR-052-paraglide-and-en-in-formatting.md) [#20 Client-Facing UX & Copy]** `NEW`
  All user-facing text in both apps is authored as keyed Paraglide JS messages sourced from `interface-contracts.md` §14's copy table, not hardcoded inline strings — regardless of the still-deferred V1 language scope decision. All displayed numbers and currency amounts use `Intl.NumberFormat("en-IN", ...)`, unconditionally.
  *Because:* the marginal cost of authoring through a message system is small at the design-system-primitives stage and large after two component libraries are fully built out — the same argument §9.7 already used for accessibility — and Indian number formatting is a correctness question for the audience already being built for, not a future-localization question at all.

---

## Disagreements

- **A case can be made that adopting a compiled message-function system before §14 has even merged is premature**, since it introduces build tooling and a new authoring convention ahead of the content it's meant to serve.
  *Implication for us:* this paper takes the position that the cost of introducing that convention now is small precisely because there's so little committed UI copy yet to retrofit — the same timing argument used throughout §9.6–§9.9. It doesn't establish that this would still be the right call if a substantial amount of UI copy already existed hardcoded across both apps; at that point the retrofit-cost comparison would need to be redone, not assumed to still favor early adoption.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q54-1.
