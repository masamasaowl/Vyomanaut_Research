# ADR-041 — Windows Packaging and Code-Signing Pipeline

**Status:** Proposed
**Topic:** #22 Desktop Application Shell Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 45

---

## Context

`ux-decisions.md` §4/§7 requires the product to be "ready to test and integrate immediately after launch," on Windows first. An unsigned installer is flagged by Windows and antivirus tools by default — Paper 45 confirms this is described as normal specifically for new Electron apps, and applies equally to any new, unsigned Windows installer regardless of shell technology. This decision fixes the packaging and signing pipeline before the first build ships to anyone outside the team.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Ship unsigned, add signing later | Fastest to a first build | Every early install gets flagged by Defender/antivirus — a real trust barrier for exactly the less-technical end of the target audience, who are least equipped to know the warning doesn't mean the software is unsafe |
| Traditional multi-year code-signing certificate | Well-established | Expensive, slow to obtain, overkill for a small team's first release cadence |
| **NSIS installer (Wails' built-in `-nsis` build target) + Azure Trusted Signing** | Official, automatable (`wails build -nsis`, ready-made CI Action), WebView2 dependency handled automatically, Trusted Signing is affordable and fast to set up for a small team | Newly-signed software still builds reputation with Windows SmartScreen gradually — signing reduces but does not eliminate first-run friction immediately |

## Decision

Package Windows releases with Wails' built-in NSIS installer target, which also bootstraps the WebView2 runtime automatically for machines that don't have it. Sign every installer, starting with the first build given to anyone outside the team, using Azure Trusted Signing. Automate the build+sign pipeline in CI from the start, using Wails' existing GitHub Actions tooling, so every tagged release produces a signed, installable artifact without manual steps.

## Consequences

**Positive:** removes a real, evidenced adoption barrier (installer-flagging) before it costs any real users; automatable from day one, not a future hardening pass.

**Negative:** Azure Trusted Signing has its own account/setup overhead, and SmartScreen reputation still needs to build up over the first period of releases regardless of signing.

**Open constraints:** macOS notarization and Linux packaging are explicitly out of scope for this ADR — each is its own decision when those platforms are scoped (`ux-decisions.md` §7).

## References

- Paper 45 — Wails and Electron desktop shell documentation and issue trackers
- `ux-decisions.md` §9.4
