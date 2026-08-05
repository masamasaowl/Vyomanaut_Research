# ADR-049 — Adopt Svelte + TypeScript for the Wails UI Layer

**Status:** Accepted
**Topic:** #22 Desktop Application Shell Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 51 — Frontend Rendering-Runtime Comparison for Embedded Webviews

---

## Context

`ux-decisions.md` §9.1 commits to Wails as the desktop shell for both the Owner and Provider apps. §11 explicitly deferred the frontend framework choice inside that shell, describing it as smaller and more reversible than the shell decision itself, to be made once the first screen is actually built. That point has arrived: M11 (REST API) is closing out, and the M19 desktop-shell milestone `build_part3.md`'s forward-compatibility section already outlines needs a concrete frontend stack to scope its sessions against. Left undecided, it also blocks a decision on how the single shared component system §9.5 requires gets structured across two separate Wails binaries rather than duplicated between them.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| React + TypeScript | Largest ecosystem and hiring pool of any option; official Wails `react-ts` template | Ships the most runtime code of the mainstream options (virtual DOM diffing); the ecosystem's biggest practical advantage — large pre-built component libraries — goes largely unused given §9.5's from-scratch design system mandate |
| Vue 3 + TypeScript | Closer to Svelte on performance once Vapor Mode is counted; large, approachable ecosystem; official Wails `vue-ts` template | Still ships a reactive runtime rather than compiling it away; Vapor Mode is newer and less battle-tested than Vue's classic virtual-DOM mode |
| Svelte 5 + TypeScript — chosen | Compiles reactivity away at build time instead of shipping a runtime; smallest and fastest option on every credible reading of the benchmark data (Paper 51); syntax closest to plain HTML/CSS/JS, lowest added ceremony for a team with no existing frontend codebase; official Wails `svelte-ts` template | Smallest component and library ecosystem of the three; smallest hiring pool if the frontend team grows |
| Vanilla TypeScript, no framework | Zero framework runtime at all; official Wails `vanilla-ts` template | No component model and no reactivity primitives — would mean hand-building the state and re-render plumbing a framework provides, for two apps with genuine interactive state (live status, forms, wizards); works against the team's own stated preference for compiler-enforced completeness over hand-rolled correctness |

## Decision

The Owner and Provider apps' Wails frontends are built with Svelte 5 and TypeScript, scaffolded from Wails' official `svelte-ts` template (`wails init -t svelte-ts`). The single shared component system `ux-decisions.md` §9.5 requires lives in a separate Svelte + Vite package built in library mode, `@vyomanaut/ui`, consumed by both app frontends through a pnpm workspace linking the two Wails frontend directories to the shared package. Each Wails binary continues to embed only its own compiled frontend output via `embed.FS`, per Wails' asset model — the shared package is a build-time dependency, not a runtime one, so this does not change how the two binaries ship and version independently.

## Consequences

**Positive:**

- Reuses the runtime-footprint and team-fit reasoning already accepted for Wails over Electron (Paper 45), rather than introducing a new, unrelated decision axis for this layer
- TypeScript throughout means Wails' auto-generated Go-struct bindings are consumed with the same compile-time completeness guarantee already enforced on the backend, instead of stopping at the language boundary
- A single `@vyomanaut/ui` package makes "one component system, two apps" mechanically true rather than a style-guide convention the two codebases could still drift apart from

**Negative / trade-offs:**

- Smallest ecosystem of the three mainstream options considered — a future contributor already fluent in React or Vue has a real, if modest, ramp-up cost on this codebase specifically
- The performance advantage this decision leans on is directional, not independently re-measured inside a Wails-packaged WebView2 host specifically — see Open constraints

**Open constraints:**

- This decision assumes the benchmark's qualitative ranking — compiled frameworks carry less runtime overhead than virtual-DOM frameworks — holds inside WebView2, not just in a generic Chromium test environment. Unverified; tracked as Q51-1; not blocking, since WebView2 is itself Chromium-family and nothing in the data suggests the ranking would invert there.
- Holds only while both apps remain thin, mostly form-and-status interfaces per §7 and §9. If either app's interactivity grows enough to need SvelteKit-level routing or data-loading machinery, the "plain Svelte + Vite, not SvelteKit" framing here should be revisited deliberately, not assumed to still be sufficient by default.

## References

- [Paper 51 — Frontend Rendering-Runtime Comparison for Embedded Webviews](../research/paper-51-frontend-framework-benchmark.md)
- [ADR-038 — Desktop Shell: Wails](ADR-038-desktop-shell-wails.md) — same runtime-footprint and team-fit reasoning, one level up the stack
- `ux-decisions.md` §9.1, §9.5, §11
