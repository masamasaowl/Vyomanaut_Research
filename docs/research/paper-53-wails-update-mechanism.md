# Paper 53 — Application Update Mechanisms for Wails Desktop Apps: Wails v3's Built-In Updater, and the Squirrel/Tailscale Precedent

**Authors:** Wails project maintainers and community (wailsapp/wails, wailsapp/updater-demo); Tailscale Inc. (product documentation and public issue tracker)
**Venue / Year:** Wails v3 official documentation and tutorials (v3.wails.io), GitHub Releases and issue tracker (wailsapp/wails); Tailscale Docs and public GitHub issue tracker (tailscale/tailscale) | retrieved 2026
**Citations:** not applicable — vendor/project documentation, release notes, and issue trackers, not an academic paper
**Topics:** #22 Desktop Application Shell Architecture
**ADRs produced:** ADR-051 (Adopt a Two-Phase Application Update Mechanism)

---

## Problem Solved

Neither app has any way to receive an update once installed. This has been true since §9.1 committed to Wails and was never addressed — not because it was overlooked, but because ux-decisions.md never claimed to cover it. An unpatchable desktop app is a standing security liability for as long as the gap exists, not a missing convenience feature, and it compounds with §9.4's code-signing decision: a signed installer that can never deliver a fix is signed for a version that only gets more outdated.

---

## Key Findings

**Wails v3 has moved from alpha to beta since ADR-038 was written, but the project's own release notes explicitly advise against switching yet.** The v3.0.0-beta.0 release notes state plainly that Wails v2 remains the current stable release, and that teams should "port deliberately, test the result, and keep your v2 application in place until the new one is ready." ADR-038's watch condition ("once Wails v3... reaches a stable release") has not fired — alpha-to-beta is progress, not the trigger itself.

**Wails v3 ships a real, documented built-in updater, but the specific example it's demonstrated against is still on a branch, not a tagged release.** The official tutorial walks through an updater that checks GitHub Releases on demand or on a timer, downloads the correct OS/architecture asset, verifies a SHA-256 digest and optionally an Ed25519 signature, shows release notes in a built-in window, and swaps the running binary in place before relaunching — all without a separate helper executable. It also supports Appcast and keygen.sh as alternative providers, and binary delta updates via bsdiff to shrink download sizes. The reference repository this tutorial downloads from, however, describes itself as targeting "v3/examples/updater — the feature is still on a branch," meaning the updater's actual merge status into a tagged v3 release is not confirmed by this documentation alone.

**Wails v2 has no equivalent, and its asset model works against building one from scratch.** A 2022 issue requesting self-update support in v2 (pointing at the third-party `minio/selfupdate` library as a starting point) was never resolved into a built-in feature. A separate community discussion confirms why a lightweight version isn't straightforward either: Wails embeds frontend assets into the binary via `embed.FS`, so updating only the frontend without touching the binary "is not how Wails works" today.

**Electron's ecosystem solved this years ago, but that path is closed here by §9.1.** Squirrel — originally built for GitHub's own desktop apps — became the standard auto-update mechanism Electron apps use via `electron-updater`/`electron-builder`. It's the most mainstream precedent for this exact problem, but irrelevant to a decision that already ruled out Node and Electron entirely.

**Tailscale — this project's own repeatedly-cited design-quality precedent — did not have a Windows auto-updater for years, and solved it without a custom binary-swap tool.** A 2020 feature request from Tailscale's own tracker states the gap directly: "We need a Windows auto-updater. Windows updates the slowest of all our platforms." A follow-up 2022 issue names a precondition for building one: moving off the NSIS installer entirely, to "100% MSI." Tailscale's current, published mechanism doesn't do a custom binary swap: it reuses whatever mechanism originally installed the client — the MSI installer on Windows, the platform package manager on Linux, the app store on mobile and App-Store macOS — silently re-invoked to apply the update. One of Tailscale's own engineers described the intended Windows approach, in that same original issue thread, as running the installer "in headless mode, the same way that Chocolatey does it, with `/S`."

**NSIS — the installer §9.4 already committed to — supports exactly that same silent, headless install pattern natively, via its own `/S` command-line flag.** This is the same technique Tailscale's engineers referenced for their own eventual Windows updater, and it requires no new packaging technology beyond what §9.4 already builds.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| A two-phase plan: silent NSIS reinstall now, migrate to Wails v3's built-in updater once it's stable and confirmed merged | Building nothing until Wails v3 stabilizes | Closes the unpatchable-app gap starting now, on the installer technology §9.4 already committed to, instead of leaving both apps without any update path for however long v3 remains in beta — at the cost of maintaining a second, simpler mechanism that gets retired later, not carried forward indefinitely |
| Modeling the interim mechanism on Tailscale's "reuse the existing installer" pattern | Building a custom Squirrel-style binary-swap updater for Wails v2 now | Reuses infrastructure §9.4 already built (the signed NSIS installer) instead of standing up a second release-artifact pipeline and a from-scratch binary-swap tool that Wails v2's `embed.FS` model doesn't cleanly support in the first place |
| Checking GitHub Releases as the version source in both phases | Standing up a dedicated update server | Zero additional infrastructure, matching the same reasoning §9.4 already uses for CI/release tooling, and gives the interim and future mechanisms the same artifact source to point at rather than two |

---

## Breaks in Our Case

- **Wails v3's tutorial and documentation present the built-in updater as a working, ready feature**
  ≠ **the reference repository it's demonstrated against describes the feature itself as still living on a branch, not a tagged v3 release**
  → Don't schedule the v3 migration off the documentation alone — confirm the updater has actually landed in a tagged release before committing an engineering session to it. Tracked as Q53-1.

- **Tailscale's published auto-update mechanism assumes an installer that can be silently re-run without a UAC elevation prompt interrupting it**
  ≠ **§9.4 doesn't currently specify whether Vyomanaut's own NSIS installer runs per-user or requires elevation on first install — only that the provider daemon's own autostart, a separate mechanism, is per-user (§10.2, ADR-047)**
  → The installer itself needs to be directly confirmed silent-reinstallable without a UAC prompt before this plan is real, not assumed from the daemon's own no-elevation decision. Also tracked as Q53-1.

- **Wails v2's own issue tracker treated self-updating as a nice-to-have feature request, filed and left open**
  ≠ **an unpatchable desktop app is a standing security liability for as long as the gap exists, not a convenience gap**
  → This is the direct argument for building the interim NSIS-based mechanism now rather than comfortably waiting for v3, even knowing v3's approach will eventually be strictly better.

---

## Decisions Influenced

- **[ADR-051](../decisions/ADR-051-two-phase-update-mechanism.md) [#22 Desktop Application Shell Architecture]** `NEW`
  Phase 1 (now, Wails v2/NSIS): the app periodically checks the GitHub Releases API for a newer tagged version, downloads the corresponding signed NSIS installer, verifies it against a published SHA-256 checksum, and silently re-runs it via NSIS's `/S` flag before prompting the user to relaunch. Phase 2 (once Wails v3 is stable and its updater is confirmed merged into a tagged release): migrate to it directly and retire Phase 1.
  *Because:* Tailscale — this project's own design-quality precedent — took years to ship a Windows updater and ultimately solved it by reusing its existing installer mechanism rather than building a custom binary-swap tool, the same shape of solution §9.4's already-chosen NSIS installer supports today without waiting on Wails v3 to stabilize.

---

## Disagreements

- **A case can be made that building a second, temporary update mechanism is wasted effort that gets thrown away once v3's updater is stable**, and that it would be simpler to skip Phase 1 entirely and rely on manual reinstalls or in-app "a new version is available, download it here" messaging until then.
  *Implication for us:* this paper takes the position that an actually-silent, actually-automatic mechanism is worth the bounded throwaway cost given the standing-security-liability argument above — but it does not independently establish how long Wails v3 will realistically remain in beta. If that turns out to be weeks rather than months, the throwaway-cost argument weakens and a simpler manual-notification-only interim approach becomes more defensible.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q53-1.
