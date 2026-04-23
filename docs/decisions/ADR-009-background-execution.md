# ADR-009 — Background Execution: Desktop-Only, ≤5% CPU Budget

**Status:** Accepted
**Topic:** #11 Background OS Execution
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 09

---

## Context

Storage providers must run the Vyomanaut daemon in the background at all times. On mobile (iOS/Android), OS-enforced background execution limits severely restrict what background processes can do — they can be killed without warning, limiting both storage reliability and upload bandwidth. Desktop OSes have no such restrictions.

## Decision

V2 is desktop-only. No mobile providers in V2. Mobile is deferred to V3 after studying desktop V2 performance (see [ADR-010](ADR-010-desktop-only-v2.md)).

For the desktop daemon:
- CPU budget: ≤ 5–10% of CPU for background audits and transfers
- Disk I/O budget: operate within normal desktop I/O load (Bolosky median ([paper-09](../research/paper-09-bolosky-feasibility.md)): 1–2% CPU, 1–2% disk load)
- The daemon must not degrade user experience; it runs at below-normal process priority

The 100 KB/s background upload bandwidth assumption (Blake & Rodrigues ([paper-06](../research/paper-06-blake-rodrigues.md))) is consistent with modern Indian ISP promises of 100 Mbps symmetrical at ₹600/month.

## Consequences

**Positive:**
- Desktop daemons have no background execution limits — reliable uptime
- 5–10% CPU is well within budget; user experience is not impacted

**Negative / trade-offs:**
- Excludes the large population of potential mobile providers until V3
- Daemon must be auto-started on OS boot — requires OS-level integration per platform (Windows service, macOS LaunchDaemon, Linux systemd unit)

**Open constraints:**
- iOS/Android background execution limits must be researched before mobile launch (Q09-5, reading-list Phase 1 #11)

## References

- [Paper 09 — Bolosky](../research/paper-09-bolosky-feasibility.md): median CPU 1–2%; 5–10% budget is safe
- [ADR-010](ADR-010-desktop-only-v2.md): desktop-only decision
