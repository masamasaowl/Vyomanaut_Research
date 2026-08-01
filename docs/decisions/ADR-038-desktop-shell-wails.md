# ADR-038 — Desktop Shell: Wails, Not Electron

**Status:** Proposed
**Topic:** #22 Desktop Application Shell Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 45 (Wails/Electron shell architecture)

---

## Context

The GUI mandate (`ux-finding.md` §7) requires a native-feeling desktop app on Windows first, matching the design bar of Docker Desktop and MongoDB Compass — both Electron apps. Vyomanaut's backend is already 100% Go (crypto, P2P, erasure, audit, payment, REST API all built through M11). The shell choice determines whether that backend is driven directly or through a second language/runtime, and was deliberately left undecided until this research phase completed.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Electron | Mature, first-class tray, matches reference apps' own choice, largest hiring pool | Bundles Chromium + Node (150–300MB), ongoing CVE-patching burden for the bundled runtime (confirmed in MongoDB Compass's own release notes), a second language alongside an already-complete Go backend |
| Tauri | Lighter than Electron, native webview | Rust backend — would introduce a second language the team has no existing exposure to, for no benefit Wails doesn't already provide given the backend is Go |
| **Wails v2** | Direct Go bindings to the existing backend, ~10–20MB, no bundled runtime to patch, Windows' WebView2 is Chromium-family so rendering matches Electron on the platform that matters first | No built-in system tray (workaround required, addressed in ADR-040) |

## Decision

Build both desktop apps (Data Owner, Provider) with Wails v2, binding directly to the existing `cmd/client`/`cmd/provider` Go logic — no rewrite of backend code, no second language introduced. Revisit this decision when Wails v3 (currently alpha, under active development) reaches a stable release, since the migration only removes the tray workaround in ADR-040 without restructuring anything else.

## Consequences

**Positive:** zero rewrite of existing, tested Go backend logic; smallest possible install size; no bundled-runtime security-patch treadmill; Windows-first rendering matches Electron's consistency guarantee without Electron's weight.

**Negative:** no built-in tray until v3 stabilizes (see ADR-040); smaller hiring pool than Electron's web-developer-everywhere talent base; per-OS webview rendering differences become a live concern only if/when macOS/Linux support is added.

**Open constraints:** must revisit if Wails v3's stabilization stalls materially, or if team composition shifts toward web-only hiring at a scale that makes Electron's talent-pool advantage decisive.

## References

- Paper 45 — Wails and Electron desktop shell documentation and issue trackers
- `desktop-application-foundations.md` §1 (original decision write-up this ADR formalizes)
