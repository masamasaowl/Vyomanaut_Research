# Paper 51 — Frontend Rendering-Runtime Comparison for Embedded Webviews: js-framework-benchmark and Wails' Official Template Documentation

**Authors:** Stefan Krause and contributors (js-framework-benchmark maintainers); Wails project maintainers (wailsapp/wails)
**Venue / Year:** js-framework-benchmark (github.com/krausest/js-framework-benchmark) — continuously maintained, results republished on new Chrome major versions; Wails official documentation (wails.io/docs, v2.13.0) | retrieved 2026
**Citations:** not applicable — a continuously-maintained open benchmark suite and official project documentation, not an academic paper
**Topics:** #22 Desktop Application Shell Architecture
**ADRs produced:** ADR-049 (Frontend Framework for the Wails UI Layer)

---

## Problem Solved

`ux-decisions.md` §9.1 commits to Wails as your desktop shell. §11 deliberately left the frontend framework choice inside that shell open, calling it smaller and more reversible than the shell decision itself, to be made once the first screen is actually built. That point has arrived: M11 (REST API) is closing out, and the M19 desktop-shell milestone `build_part3.md`'s forward-compatibility section already outlines needs a concrete stack to scope its sessions against. Left undecided, it also blocks a decision on how the single shared component system §9.5 calls for gets structured across your two separate Wails binaries — the Owner app and the Provider app.

---

## Key Findings

**Wails ships six official templates, each in JavaScript and TypeScript variants, selected at project generation.** Running `wails init -t <name>` gives you Svelte, React, Vue, Preact, Lit, or Vanilla, with a `-ts` suffix for the TypeScript version of each. All six are first-party, not community-maintained, so none of them carries extra maintenance risk relative to the others on that basis alone.

**Wails auto-generates TypeScript definitions from your Go bindings regardless of which of the six you pick, but only TypeScript frontends can consume them.** Every Go method and struct exposed to the frontend gets a matching `.d.ts` file generated at build time. A JavaScript frontend receives the exact same runtime bindings but with no compile-time checking against them — your compiler-enforced-completeness discipline on the Go side would stop at the language boundary rather than reaching through to the UI layer.

**js-framework-benchmark's methodology measures real DOM operations, not a synthetic proxy for them.** It automates Chrome, times the interval from click event to the end of the resulting paint event using Chrome's own timeline data, and covers a fixed set of operations — create/update/append/clear rows at 1,000 and 10,000 rows, plus startup time and memory at each step. It has run continuously since 2017 and republishes results on new Chrome major versions, which is why any single snapshot of its numbers ages quickly.

**The direction of the finding is consistent and repeated; the specific magnitude is not.** Every credible source consulted agrees on the ranking: compiled, no-virtual-DOM frameworks (Svelte, and Solid in the wider field) post smaller startup payloads, faster first paint, and lower per-update memory than virtual-DOM frameworks (React; Vue in its classic mode). Vue's newer Vapor Mode narrows this gap without eliminating it. But the absolute numbers different secondary sources report for the same claimed metric — for instance, a minimal Svelte app's gzipped bundle size — vary by as much as 5–8× from one blog writeup to another, because they measure different app shapes at different points in each framework's release cycle and aren't running the same benchmark build. The qualitative ranking is trustworthy; no specific KB or ms figure quoted anywhere in this note should be.

**Nearly every source, including the ones most favorable to Svelte's numbers, adds the same caveat: real application performance differences are smaller than the micro-benchmark differences suggest.** This is not a hedge that weakens the finding — it means the benchmark data should inform, not replace, the ecosystem and team-fit reasoning that decides which framework a team with no existing frontend codebase can actually ship fastest in.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Svelte 5 + TypeScript | React + TypeScript | Smaller runtime footprint and less code to reach the "snappy, premium" interaction feel `ux-decisions.md` §7 sets as the bar — at the cost of a materially smaller component and library ecosystem than React's, which matters less here because §9.5 already commits to a from-scratch design system rather than an off-the-shelf component kit React's ecosystem would otherwise make valuable |
| Svelte 5 + TypeScript | Vue 3 + TypeScript | Vue is close on performance once Vapor Mode is counted, and is broadly approachable — Svelte is chosen because it compiles reactivity away entirely rather than shipping a smaller-but-still-present reactive runtime, and because its component syntax carries less added ceremony over plain HTML/CSS/JS, which matters for a team with no existing frontend codebase in any of the three |
| Plain Svelte + Vite in library mode, for a shared `@vyomanaut/ui` package | A full SvelteKit application, or a heavier monorepo tool such as Turborepo or Nx | Wails' asset model only needs an `embed.FS`; a full application framework's routing and data-loading machinery is unused weight for two thin, mostly form-and-status interfaces — at the cost of wiring the workspace linkage by hand (a pnpm workspace) rather than getting it from a framework's own scaffolding |

---

## Breaks in Our Case

- **js-framework-benchmark's own periodic Chrome-version re-runs, and the secondary sources summarizing them, disagree with each other by as much as 5–8× on the same claimed metric**
  ≠ **an ADR needs a specific, implementable decision, not a number range that wide**
  → The decision below is grounded in the *direction* of the finding — compiled frameworks ship less code and win update-heavy benchmarks — not in any single absolute figure. If this is ever formally re-validated, the number to trust is the live table at github.com/krausest/js-framework-benchmark, run against your actual packaged build, not a cited blog post.

- **js-framework-benchmark measures a generic Chromium browser environment**
  ≠ **your app runs inside Windows' WebView2, a specific Chromium-family runtime already fixed by the Windows-first decision (Paper 45)**
  → The benchmark's results should transfer reasonably well, since WebView2 is Chromium-based, but this has not been measured directly inside a Wails-packaged WebView2 host. Flagged as Q51-1, not blocking — nothing in the data suggests the qualitative ranking would invert inside WebView2 specifically.

- **The benchmark's methodology assumes each implementation is idiomatic, written by someone already fluent in that framework**
  ≠ **your team has no existing Svelte, React, or Vue codebase — the starting skill level is close to zero for all three**
  → Read the performance data alongside the ecosystem and learning-curve reasoning in the Decision below, not instead of it. Performance alone does not tell you which framework this specific team ships fastest in.

---

## Decisions Influenced

- **[ADR-049](../decisions/ADR-049-svelte-frontend-framework.md) [#22 Desktop Application Shell Architecture]** `NEW`
  Svelte 5 + TypeScript, scaffolded from Wails' official `svelte-ts` template, with shared UI factored into a separate Svelte + Vite library package consumed by both app binaries.
  *Because:* it is the only one of Wails' six official templates that compiles reactivity away at build time rather than shipping a runtime, matching the same footprint reasoning already accepted for Wails over Electron (Paper 45); TypeScript is chosen specifically to consume the type definitions Wails already generates from your Go structs, rather than leaving that generated typing unused.

---

## Disagreements

- **A case can be made that React's ecosystem and hiring depth are worth the runtime cost regardless of app shape, independent of the benchmark numbers.**
  *Implication for us:* this paper does not fully resolve that for a team that may hire dedicated frontend engineers later — it establishes that, for the team and app shape you have today, the footprint and team-fit arguments favor Svelte. Paper 45 left the equivalent "ecosystem vs. footprint" question open for the shell choice itself; this is the same open question one layer up the stack, not a new one.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q51-1.
