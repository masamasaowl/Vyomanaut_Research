# ADR-040 — System Tray: Goroutine Pattern Now, Native Tray After Wails v3

**Status:** Proposed
**Topic:** #22 Desktop Application Shell Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 45

---

## Context

`ux-finding.md` §7's navigation model makes the tray the Provider app's primary interface for routine use (status glance, pause, quit), matching the Tailscale precedent reviewed earlier. Wails v2 (ADR-038) has no built-in tray API — a long-standing, explicitly acknowledged gap. A documented community workaround exists and is specifically confirmed working on Windows, the platform this ships on first.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Wait for Wails v3's native tray to stabilize before shipping any tray UI | No workaround code to maintain | Blocks the Provider app's primary navigation model on an alpha framework's release timeline, with no committed date |
| **Goroutine-based tray (`getlantern/systray` or `fyne.io` fork, run alongside the Wails runtime)** | Ships now, confirmed working on Windows, proven in at least one shipped production app | Workaround, not a first-class API; reported rough edges are specifically in cross-platform/macOS-linking contexts, not the Windows-only case shipping first |

## Decision

Implement the Provider app's tray using a dedicated tray library running in its own goroutine, independent of the Wails webview, with tray-click toggling the main window's visibility. Ship this on Windows now. Migrate to Wails v3's native `application.SystemTray` API once v3 reaches a stable release — this migration removes the workaround only; it does not restructure the tray-first navigation model itself.

## Consequences

**Positive:** the Provider app's primary navigation model is not blocked on an alpha framework's timeline; the pattern is proven, not experimental, on the platform shipping first.

**Negative:** carries workaround-level risk (thread-safety, no first-class support) until the v3 migration; the specific rough edges reported for other platforms mean this decision should be re-evaluated, not assumed safe, before macOS support is added.

**Open constraints:** track Wails v3's stabilization explicitly (see ADR-038) as the trigger for retiring this workaround.

## References

- Paper 45 — Wails and Electron desktop shell documentation and issue trackers
- `desktop-application-foundations.md` §4
