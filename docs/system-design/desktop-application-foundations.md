# Vyomanaut — Desktop Application Foundations

*Companion to `ux-finding.md`. These are decisions, not options — made so engineering has a fixed foundation to build on. Still not a spec or an ADR; each decision here should get its own ADR when implementation starts.*

---

## 1. Shell technology: Wails

**Decision: the desktop app is built with Wails, not Electron.**

Docker Desktop and MongoDB Compass — the two reference points for this project's GUI mandate — are both built on Electron, and it's worth being direct about why we're not following them there. Electron is the safe, proven choice when your backend is a separate thing entirely (Docker's is a virtual machine; MongoDB's is a database server you connect to over the network) and your team already thinks in JavaScript. Neither is true here. Vyomanaut's entire backend — the cryptography, the peer-to-peer network, the audit system, the payment logic, the API — is already written, and it's already Go. Wails takes that Go code and gives it a window, directly, with no second language and no second runtime sitting next to it. Electron would mean shipping a bundled Chromium and Node.js runtime (150–300MB, and a growing list of security patches to track and re-ship over the app's lifetime) purely to display a UI in front of a backend we've already built in a language that doesn't need any of that.

The one real gap Wails has — no built-in system tray — turns out not to be a blocker. It's a known, documented gap in the current stable release, with a proven workaround (run a small, separate tray library alongside the app) that's specifically confirmed working on Windows, our first platform. The next major version of Wails is already being built with a proper tray built in, and it's in active weekly development, not stalled — we'll move to it once it's stable, and the migration only removes a workaround, it doesn't restructure anything.

**What this decides, concretely:**

- Both apps (Data Owner, Provider) are Wails apps: a Go backend we already have, a web-based frontend (HTML/CSS/JS, framework choice below) rendered in the operating system's own built-in browser engine, packaged as one native application.
- No Node.js, no Electron, anywhere in the shipped product.
- The existing `cmd/client` and `cmd/provider` logic is what the app runs — the app is a new front door on the same house, not a rebuild of it.

## 2. How the app talks to the daemon

**Decision: direct, in-process calls — no local web server, no open port.**

Storj and Syncthing both run a small web server on the user's machine and expect them to open it in a browser tab. We're deliberately not doing that: a web server on a loopback port is one more thing to secure (both of those projects have had to add authentication specifically because a bare local server isn't automatically safe), and it doesn't match the "real app, not a browser tab" mandate in `ux-finding.md`. Wails avoids this entirely — the frontend calls Go functions directly, in the same process, the way a normal function call works. Nothing is listening on a port for the UI's sake.

The REST API already built for Vyomanaut (used by `cmd/client`/`cmd/provider` today) doesn't go away — it stays as the interface for scripting, automation, and any future mobile app. It's just not how the desktop app itself talks to its own backend.

## 3. Platform order: Windows first

**Decision, per the correction to this brief: Windows is where we build and ship first**, full stop — not a "build once, adjust for other platforms" plan. Most Indian desktops, in homes, offices, and university labs, run Windows, and the product needs to be installable and testable the moment it launches, on the machines people actually have.

This also happens to be the easiest platform for the shell choice we just made: Windows ships its own modern, auto-updating browser engine (WebView2), which Wails uses directly, and it's the same rendering engine family Electron bundles — so the "smooth, premium" bar in `ux-finding.md` is fully achievable on Windows without any of Electron's extra weight. On most Windows 11 machines it's already installed; where it isn't, the installer offers it automatically.

macOS and Linux come after, matching the design as closely as each platform's own browser engine allows — that's a real, separate piece of work each time, not an afterthought, and we're not pretending otherwise.

## 4. System tray, for real this time

**Decision: yes, on all platforms, starting with Windows — built with a small dedicated tray library running alongside the app, not inside Wails itself (Wails doesn't have its own yet).** This is a documented, working pattern, already proven in production apps, not an experiment.

The tray is the Provider app's main interface for routine use — status at a glance, pause, quit — matching the "tray-first" navigation model in `ux-finding.md`. The full window opens for anything deeper.

## 5. Packaging and signing, for Windows

- **Installer:** NSIS, built directly by Wails's own tooling (`wails build -nsis`). This also handles the WebView2 dependency automatically, so a user without it gets it installed as part of setup, not as a separate confusing step.
- **Code signing:** required from the first build we let anyone outside the team install, not added later. An unsigned installer gets flagged by Windows and antivirus tools by default — this is normal and expected for any new desktop app, not a Vyomanaut-specific problem, but it's a real trust barrier for exactly the non-technical end of our audience, and it compounds with the Electron-specific version of the same problem we already avoided in §1. Azure's newer code-signing service (Trusted Signing) is the practical way to do this without the cost and paperwork of a traditional multi-year certificate, and fits a small team's budget.
- **Automated builds:** a straightforward CI pipeline (Wails already has ready-made GitHub Actions tooling for this) should produce a signed, installable Windows build on every release from day one — "ready to test and integrate immediately after launch" is a pipeline decision as much as a product one, and it's solved, not research.

## 6. Design system — making "premium and minimal" concrete

A design mandate only helps engineering if it turns into specific, checkable choices. These are the starting primitives; a designer should refine the exact values, but the *kind* of choice is decided:

- **Typography:** one confident, modern system typeface (Windows' own Segoe UI Variable is a legitimate, free starting point — it was built for exactly this kind of clean, native-feeling look on Windows), a small number of sizes and weights used consistently, generous line height. No more than two typefaces in the entire app, ever.
- **Color:** a small, restrained palette — a neutral base (whites, greys, one dark mode equivalent) plus a single accent color used sparingly for actions and status. Status colors (healthy, degraded, error) are consistent everywhere they appear, tied directly to the error/status wording already defined for the CLI so the same event never looks or reads differently in two places.
- **Spacing and layout:** one consistent spacing scale (e.g., multiples of 4px) used everywhere, generous whitespace over dense information, no screen that tries to show everything at once. If a screen feels crowded, it's split into two screens, not shrunk to fit.
- **Motion:** short, purposeful transitions (opening a window, switching a tab, a status changing) — never decorative animation for its own sake. Motion should make state changes easier to follow, not slower to get through.
- **One component system, shared by both apps.** The Data Owner and Provider apps are different products with the same visual language — a shared internal component library (buttons, cards, status badges, the toast/error system already defined in the interface contracts) is what keeps them feeling like one company made both, and keeps a design change from having to happen twice.

## 7. What's still open

- The exact frontend framework for the Wails UI layer (React, Vue, Svelte, or plain HTML/CSS/JS) — a smaller, more reversible choice than the shell itself, and can be decided when the first screen is actually built.
- macOS- and Linux-specific packaging and signing steps — deferred until the Windows version is real and in front of users, per §3.
- The exact color palette, type scale, and spacing values — §6 fixes the *kind* of system, not the final numbers; that's a focused design pass, not an open research question.
