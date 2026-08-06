# ADR-052 — Adopt Paraglide JS and en-IN Number Formatting

**Status:** Accepted
**Topic:** #20 Client-Facing UX & Copy
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 54 — Locale-Ready Copy and Number Formatting for a Copy-Table-Driven UI

---

## Context

`interface-contracts.md` §14 (the user-facing copy contract, ADR-034) hasn't merged yet, and `ux-decisions.md` §11 correctly defers the English-vs-multilingual V1 scope question as a business decision, not a product one. Neither of those facts settles two adjacent technical questions engineering needs an answer to now: how the copy table gets consumed by Svelte components once it merges, and whether correct rupee-amount display for an Indian audience is itself contingent on that later multilingual decision. §9.5 through §9.7 already established the pattern of settling this kind of foundation cheaply now rather than expensively later; this extends the same pattern to copy and number formatting.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Hardcode strings directly in Svelte components now; introduce a message system later if a second language is ever needed | No new tooling or authoring convention today | Retrofitting a message system across two fully built component libraries is materially more expensive than building it in from the start — the same cost asymmetry §9.7 already used to justify an early accessibility baseline |
| A runtime i18n library (svelte-i18n, i18next) | Larger ecosystem, more prior art in general web development | Ships a runtime lookup layer instead of compiling messages away, working against the same footprint reasoning ADR-049 already used to choose Svelte itself |
| Paraglide JS, wired to plain Svelte + Vite (ADR-049), plus native `Intl.NumberFormat("en-IN", ...)` for all numbers and currency — chosen | Compiler-based, typesafe, tree-shakable — matches ADR-049's own compiler-over-runtime philosophy; works in a plain Vite app without requiring SvelteKit; the number-formatting half is a zero-dependency, browser-native correctness fix independent of any future language decision | Paraglide's own documentation and default tooling assume a SvelteKit project; only its core message-compiler functionality is used here, with its routing/URL features left unused |

## Decision

All user-facing text in both the Owner and Provider apps is authored as keyed Paraglide JS messages, sourced from `interface-contracts.md` §14's copy table once it merges, rather than as hardcoded inline strings in Svelte components — this applies regardless of whatever the still-deferred V1 language-scope decision (§11) ends up being, since the authoring convention is what's being decided here, not the launch language. Separately and unconditionally, every number and currency amount displayed in either app's UI uses `Intl.NumberFormat("en-IN", ...)` — with `{ style: "currency", currency: "INR" }` for rupee amounts — rather than hand-rolled formatting or default locale formatting. The number-formatting half of this decision does not wait on §14, on Paraglide's integration, or on the language-scope business decision; it ships as soon as any screen displays a number.

## Consequences

**Positive:**

- Extends the same "cheap now, expensive later" reasoning §9.7 already applied to accessibility, one layer over into copy and formatting
- Keeps the frontend stack internally consistent: Paraglide's compile-time approach mirrors ADR-049's own reasoning for choosing Svelte over a virtual-DOM framework
- The `Intl.NumberFormat("en-IN", ...)` half is a correctness fix the team gets essentially for free, and it ships immediately rather than waiting on any other decision in this document

**Negative / trade-offs:**

- Introduces a compiled message-function convention, and the build step that goes with it, before `interface-contracts.md` §14 has actually merged and before there's much real UI copy to author through it
- Paraglide's tooling and most of its public documentation assume a SvelteKit project; integrating its Vite plugin into Wails' own build pipeline (`wails build`, which itself wraps Vite) hasn't been verified in practice

**Open constraints:**

- This decision assumes the Paraglide JS Vite plugin integrates cleanly with Wails' build pipeline for a plain Svelte + Vite project (ADR-049), not a SvelteKit one. Not yet verified end-to-end. Tracked as Q54-1; not blocking adoption of the pattern, since a working plain-Vite integration is well within Paraglide's documented scope — only the specific interaction with `wails build`'s own Vite wrapping is unconfirmed.

## References

- [Paper 54 — Locale-Ready Copy and Number Formatting for a Copy-Table-Driven UI](../research/paper-54-paraglide-en-in-formatting.md)
- [ADR-049 — Adopt Svelte + TypeScript for the Wails UI Layer](ADR-049-svelte-frontend-framework.md) — the plain Svelte + Vite target this decision integrates with
- [ADR-034 — User-Facing Error Copy](ADR-034-user-facing-error-copy.md) — the `interface-contracts.md` §14 copy table this decision consumes
- `ux-decisions.md` §5.2, §8, §9.5, §9.9
