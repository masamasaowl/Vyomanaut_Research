# ADR-051 — Adopt a Two-Phase Application Update Mechanism

**Status:** Accepted
**Topic:** #22 Desktop Application Shell Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 53 — Application Update Mechanisms for Wails Desktop Apps

---

## Context

Neither the Data Owner nor the Provider app has any way to receive an update once installed — a gap ux-decisions.md never previously addressed, and a real, standing security liability rather than a missing convenience. §9.4 already committed to a signed NSIS installer for Windows; a signed installer that can never deliver a fix is signed for a version that only gets more outdated. Wails v3 will eventually solve this properly with a built-in updater, but v3 is still beta, and the framework's own release notes recommend keeping v2 applications in place until v3 is actually ready.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Wait for Wails v3's built-in updater; ship nothing until then | Avoids building a mechanism that gets thrown away later; lands directly on the more capable, framework-supported long-term solution | Leaves both apps unpatchable for however long v3 remains in beta — the framework's own guidance is to keep v2 apps in place until v3 is ready, which could be a long wait; a real, standing security liability, not just a convenience gap |
| Build a custom Squirrel-style binary-swap updater for Wails v2 now | Full control; no dependency on v3 stabilizing | Wails v2's `embed.FS` asset model doesn't cleanly support this (confirmed by the project's own community discussion on the topic); duplicates most of what v3's built-in updater already does properly, only to be retired once v3 ships |
| Two-phase: silent NSIS reinstall now, migrate to Wails v3's built-in updater once stable and confirmed merged — chosen | Ships real update delivery immediately, on infrastructure §9.4 already built; matches Tailscale's own eventual, published solution to the identical Windows-updater problem; retires cleanly once v3's updater is confirmed ready | Two mechanisms exist across the transition; Phase 1's silent-reinstall approach needs its own verification that it doesn't trigger a UAC elevation prompt |

## Decision

**Phase 1, effective now, on Wails v2/NSIS:** the app checks the GitHub Releases API for the latest tagged release on a periodic interval. If a newer version is found, it downloads the corresponding NSIS installer asset, verifies it against a published SHA-256 checksum (the same SHA256SUMS-sidecar convention Wails' own v3 updater tooling uses, kept consistent across both phases), and silently re-runs the installer via NSIS's `/S` flag, then prompts the user to relaunch rather than forcing it — consistent with §8's "inform, don't just act invisibly" posture.

**Phase 2, once Wails v3 reaches a stable release and its built-in updater is confirmed merged into a tagged release, not just present on a development branch:** migrate to it directly — GitHub Releases as the provider, SHA-256 plus Ed25519 signature verification, in-place binary swap and relaunch, bsdiff delta patches for smaller downloads — and retire the Phase 1 mechanism at that point rather than running both indefinitely.

## Consequences

**Positive:**

- Closes a real, standing security gap — an app with no way to receive patches — starting now, not after an indefinite wait on Wails v3
- Reuses the exact signed-installer pipeline §9.4 already committed to, rather than standing up a second release-artifact format
- Follows a solution shape with real production precedent (Tailscale's own eventual answer to the identical Windows-updater problem), not a from-scratch design
- Gives Phase 2 a clean, well-scoped trigger condition — v3 stable and its updater confirmed merged — rather than an open-ended "eventually"

**Negative / trade-offs:**

- Two update mechanisms exist across the transition, and Phase 1 is engineering effort that gets fully retired once Phase 2 lands — a real, if bounded, throwaway cost
- Phase 1's silent reinstall has not yet been confirmed to run without triggering a UAC elevation prompt, which would undermine its "silent" premise if it does

**Open constraints:**

- Phase 1 depends on the NSIS installer being runnable via `/S` without a UAC prompt appearing — not yet verified against Vyomanaut's own installer configuration. Tracked as Q53-1; needs resolving before Phase 1 ships, not just before it's considered complete.
- Phase 2's trigger condition depends on Wails v3's updater actually landing in a tagged release, not just appearing in the documentation and a branch-hosted example. Also tracked as Q53-1.

## References

- [Paper 53 — Application Update Mechanisms for Wails Desktop Apps](../research/paper-53-wails-update-mechanisms.md)
- [ADR-038 — Desktop Shell: Wails](ADR-038-desktop-shell-wails.md) — the v3-stability question Phase 2's trigger condition depends on
- `ux-decisions.md` §6 (Tailscale precedent), §8 (inform-don't-just-act posture), §9.4 (NSIS installer), §9.8
