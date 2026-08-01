# ADR-042 — Provider App Runs at Standard User Privilege

**Status:** Proposed
**Topic:** #21 Desktop/Institutional Compute Harvesting Precedent
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 43 (Condor)

---

## Context

Condor's original 1988 architecture required its central manager to hold root/elevated privileges specifically to impersonate submitting users on remote machines — a design an independent 1997 patent filing (US 5,978,829) names as Condor's principal security weakness, since a coding flaw in a privileged server "may allow someone other than the authorized user to gain access to other users' privileges." Vyomanaut's provider app has no equivalent need to impersonate another user or process, but the permission model has not yet been explicitly fixed as a requirement, which risks it being decided ad hoc during implementation.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Request elevated/administrator privileges for convenience (e.g., simpler auto-start registration, broader file-system access) | Marginally simpler implementation in a few places | Directly repeats the documented weakness in Condor's original design; a real red flag for institutional IT review (§5.2's documented policy risk) and for individual users asked to grant admin rights to a new app |
| **Standard user privilege only, for all provider-app functionality** | No cross-user impersonation risk; easier institutional security review; matches ordinary desktop-app expectations | Auto-start registration and any OS-integration features must be implemented within standard-user constraints (per-OS mechanisms already scoped in `desktop-application-foundations.md` §3 — systemd user service, launchd user agent, Windows Task Scheduler/Run-key at the user level) |

## Decision

The Provider app requests no elevated or administrator privileges for any of its functionality, at any point — installation, auto-start registration, or runtime operation. All OS integration is implemented using standard-user-level mechanisms already scoped for auto-start (`desktop-application-foundations.md` §3).

## Consequences

**Positive:** removes a documented, named class of security risk before it's designed in; simplifies institutional IT security review (relevant given the real policy barriers documented in `ux-finding.md` §5.2); matches ordinary user expectations for a desktop app that isn't a system utility.

**Negative:** none identified — no current provider-app requirement needs elevated privilege.

**Open constraints:** if a future feature genuinely requires elevated privilege, it must be justified against this ADR explicitly, not added by default.

## References

- Paper 43 — Condor: A Hunter of Idle Workstations
- US Patent 5,978,829 (cited within Paper 43 as the source of the specific critique)
