# Paper 45 — Wails and Electron: Desktop Shell Architecture Documentation and Issue Trackers

**Authors:** Wails project maintainers (wailsapp/wails); Electron project; Docker Inc.; MongoDB Inc.
**Venue / Year:** Official project documentation and GitHub issue trackers (wails.io, v3.wails.io, github.com/wailsapp/wails) | retrieved 2026, Wails v3 changelog entries current to within the prior month
**Citations:** not applicable — documentation and issue-tracker sources, not an academic paper
**Topics:** #20, #22
**ADRs produced:** none yet — primary basis for the shell-technology decision recorded in `desktop-application-foundations.md` §1

---

## Problem Solved

Your GUI mandate requires a real, native-feeling application on Windows first, matching the design bar of Docker Desktop and MongoDB Compass — both built on Electron. Your entire backend is already written in Go. This review tests whether following the reference products' technology choice is actually correct for your specific starting position, rather than assuming it generalizes.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Wails (Go-native, OS webview) | Electron (bundled Chromium + Node) | Single binary, ~10–20MB vs. 150–300MB, no second language/runtime — at the cost of no built-in system tray (v2) and per-OS webview rendering differences that only matter once macOS/Linux are in scope |
| A goroutine-based community tray pattern, confirmed on Windows | Waiting for Wails v3's native tray to leave alpha | Ships now, on the platform you're targeting first, at the cost of a workaround with documented (non-Windows-specific) rough edges |
| In-process Go↔frontend bindings | A local HTTP server the UI connects to (Storj/Syncthing pattern) | No open port, no CSRF/auth surface to build — at the cost of tying the UI tightly to one shell framework rather than a swappable web-based front end |

---

## Breaks in Our Case

- **Docker Desktop and MongoDB Compass both use Electron**
  ≠ **your backend is already 100% Go, with no pre-existing separate-language server to talk to**
  → Electron's justification in the reference products (bridging to a VM over a socket; a database over a network wire protocol) does not apply to you — Wails is chosen because your starting precondition differs, not because Electron is wrong in general.

- **Wails v2's system tray gap is real and long-standing**
  ≠ **the documented workaround is specifically confirmed working on Windows, your first platform**
  → The gap is a genuine cost, not a blocker for the platform sequencing already decided. It becomes a more serious question only if/when macOS support (where the same workaround has reported linking conflicts) is added.

- **MongoDB Compass's own release notes show a recurring pattern of Electron-version bumps specifically to patch bundled-runtime CVEs**
  ≠ **Wails ships no bundled Chromium/Node runtime, so this entire maintenance category does not exist in your release cycle**
  → Not a criticism of Compass's engineering — a cost category inherent to bundling a browser runtime, worth naming explicitly so a future engineer doesn't default to Electron out of habit without weighing it.

---

## Decisions Influenced

- **Shell technology decision (`desktop-application-foundations.md` §1)** `PRIMARY BASIS`
  Wails v2 now, migrating to v3 once stable.
  *Because:* every argument for Electron examined here is either inapplicable to your starting position (no existing server-language separation) or neutralized by the Windows-first decision — Windows' WebView2 is Chromium-family, the same engine family Electron bundles, and the tray workaround is specifically Windows-confirmed.

- **Windows packaging and code-signing plan (`desktop-application-foundations.md` §5)** `CONFIRMED FEASIBLE`
  The NSIS + WebView2-bootstrap + CI-action pipeline is documented, current, and already used in production by other Wails projects.
  *Because:* `wails build -nsis` and the official GitHub Actions tooling remove this from the list of open engineering risks for the upcoming build milestones.

- **Shell-technology revisit trigger (not yet an ADR — forthcoming)**
  Revisit the shell decision once Wails v3 exits alpha, since the migration only removes the goroutine tray workaround without restructuring anything else.
  *Because:* v3's changelog shows active, current development specifically on native tray and native-chrome polish (DPI scaling, frameless drag regions, Windows 11 Snap Layouts) — this is a planned, low-risk future revision, not an open risk today.

---

## Disagreements

- **A case can be made that Electron's larger developer ecosystem (any web developer, vs. Go+web-binding familiarity) is a real hiring/maintainability advantage, independent of runtime footprint or tray support.**
  *Implication for us:* this paper does not resolve that trade-off — it only establishes that the runtime-footprint and tray-support arguments, taken alone, favor Wails once Windows-first is fixed. Revisit alongside the shell decision if team composition or hiring plans change materially.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q45-1.
