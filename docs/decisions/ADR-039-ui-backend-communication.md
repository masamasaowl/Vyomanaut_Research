# ADR-039 — UI–Backend Communication: In-Process Bindings, No Local Port

**Status:** Proposed
**Topic:** #22 Desktop Application Shell Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 45; Paper 46 (Storj/Syncthing local-web-server precedent, reviewed in the earlier user-facing-application study)

---

## Context

Storj and Syncthing both front their daemon with a local web server the user opens in a browser tab, which then requires its own auth/CSRF hardening (both projects added this after the fact). Wails (ADR-038) offers a different option: the frontend calls Go functions directly, in-process, with nothing listening on a port. This decision fixes how the desktop app's UI talks to its own backend logic, distinct from the existing REST API (M11), which continues to serve external/automation callers.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Local HTTP server + browser tab (Storj/Syncthing pattern) | Framework-agnostic, works with any future web-based front end | Requires its own auth/CSRF hardening for a surface that has no reason to be network-reachable at all; not "a real app," per the GUI mandate |
| **In-process Go↔frontend bindings (Wails runtime calls)** | No open port, no auth surface to build for the UI itself, matches "real app not a browser tab" mandate | Ties the UI tightly to the Wails runtime rather than a swappable web front end |

## Decision

The desktop app's UI calls backend logic through Wails' in-process bindings — no local HTTP server is started for the UI's own use. The existing REST API (M11) is retained, unchanged, for scripting, automation, and any future mobile/browser-based companion — it is a separate surface, not replaced by this decision.

## Consequences

**Positive:** removes an entire class of local-attack-surface concerns (no loopback port to secure); no CSRF/auth layer needed purely for the desktop UI's own use.

**Negative:** the desktop UI cannot be trivially repointed at a remote instance the way a browser-tab-based UI could; any future need for that would require adding a network-facing path deliberately, not get one for free.

**Open constraints:** if a future browser-based or mobile companion needs its own local-network story, design it against the existing REST API rather than retrofitting a port onto the desktop shell (see Q45-1).

## References

- Paper 45 — Wails and Electron desktop shell documentation and issue trackers
- `desktop-application-foundations.md` §2
