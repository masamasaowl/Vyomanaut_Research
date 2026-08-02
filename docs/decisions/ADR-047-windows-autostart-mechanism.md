# ADR-047 — Provider Daemon Autostart on Windows: Per-User Task Scheduler Logon Trigger

**Status:** Proposed
**Topic:** #21 Desktop/Institutional Compute Harvesting Precedent, #22 Desktop Application Shell Architecture
**Supersedes:** —
**Superseded by:** —
**Research source:** Paper 50 (Windows background-app autostart mechanisms)

---

## Context

Three existing decisions do not agree with each other. FR-031 requires the provider daemon to auto-start via "the platform-appropriate mechanism (**Windows Service**, macOS LaunchDaemon, Linux systemd unit)." ADR-040 makes the system tray the Provider app's *primary* interface for routine use. ADR-042 requires the Provider app to request no elevated or administrator privilege at any point — installation, auto-start registration, or runtime operation.

A genuine Windows Service cannot satisfy both of the other two at once: installing one requires administrator rights via the Service Control Manager (violating ADR-042), and it runs in Session 0 — architecturally isolated from the interactive desktop since Windows Vista, specifically so a service cannot present UI to an interactive session (violating ADR-040, since a Session-0 process cannot show a tray icon at all). This is a hard OS boundary, not a configuration gap.

ADR-042 previously cited a scoping of this problem as "already scoped in `desktop-application-foundations.md` §3." That citation was inaccurate — §3 of that document (now folded into `ux-decisions.md`) covers platform build-and-ship *order*, not per-OS autostart mechanisms. No prior ADR actually decided this. This ADR does.

Paper 50 also corrects a precedent this project has cited elsewhere: Tailscale, held up as the design bar for Vyomanaut's desktop apps, does *not* achieve unprivileged Windows autostart. Its Windows client runs `tailscaled.exe` as a genuine, admin-installed Windows Service (Session 0), with a separate `tailscale-ipn.exe` process in the interactive session rendering the tray icon and talking to the service over local IPC — a real, working pattern for the Session-0/tray conflict, but one that requires exactly the elevated install ADR-042 rules out. Tailscale accepts that cost because its actual job (installing and driving a virtual network adapter) requires kernel-level access no standard user account has, under any design. Vyomanaut's provider daemon does ordinary outbound, user-mode P2P networking (ADR-021) — it has no equivalent requirement, and therefore no equivalent reason to pay Tailscale's cost.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| Genuine Windows Service, as FR-031 literally states | Simple to reason about; standard "background daemon" pattern in the abstract | Requires admin to install (violates ADR-042); Session 0 isolation makes ADR-040's tray-as-primary-interface architecturally impossible |
| Split design: admin-installed Session-0 service + interactive companion process (the actual Tailscale pattern) | Production-proven; would technically satisfy ADR-040's tray requirement | Still requires an elevated install step (violates ADR-042 as written); adds a SYSTEM-level component whose entire job is spawning a process into a user's session — exactly the kind of component a security-conscious institutional IT reviewer (`ux-decisions.md` §5.2) scrutinizes hardest, working against the same trust this design is meant to build |
| UWP/MSIX `StartupTask` API | Most Microsoft-endorsed modern pattern; user-visible toggle in Settings → Apps → Startup | Requires MSIX packaging, reopening ADR-041's NSIS packaging decision to gain a startup mechanism — a larger trade-off than this ADR is scoped to make |
| `HKCU\...\Run` registry key or Startup-folder shortcut | Simplest possible per-user, non-elevated mechanism; well-precedented (Dropbox-era pattern) | No retry-on-failure, no startup delay, no queryable run history — strictly less capability than the next option, at the same privilege cost |
| **Per-user Task Scheduler logon trigger, registered by the installer under the installing user's own account** | No elevation required when scoped to the creating user's own logon (not "all users," no stored credentials, not "highest privileges"); retry-on-failure, startup delay, and run history are available at no extra privilege cost | Slightly more setup code than a bare registry key (COM Task Scheduler API or `schtasks`, rather than a single registry write) |

## Decision

**The provider daemon's Windows autostart mechanism is a Task Scheduler task, registered by the installer at install time under the installing user's own account, with a logon trigger scoped to that user only.** This is not a Windows Service, and the installer never requests elevation to set it up.

Task configuration:

- **Trigger:** "At log on" of the installing user specifically (`LogonTrigger.UserId` set to that user, not left empty for "any user") — the case Microsoft's own documentation confirms does not require elevation to create.
- **Action:** launch the provider daemon's own entry point (the Wails-wrapped `cmd/provider`, per ADR-038), not a separate service binary.
- **Security options:** "Run only when user is logged on" (no stored credentials); **not** "Run whether user is logged on or not"; **not** "Run with highest privileges." Both of the latter require elevation and are explicitly not used.
- **Reliability options:** a short startup delay (on the order of 30–60 seconds, to let the network stack come up before the daemon tries to reach the microservice) and a bounded restart-on-failure policy (e.g., restart after 1 minute, up to 3 attempts) — both configurable within Task Scheduler's own options, at no additional privilege cost.
- **Scope:** this ADR covers the **Provider app only**. The Data Owner app is used on demand, not as an always-on background service, and has no autostart requirement — its absence from this ADR is deliberate, not an oversight.
- **Uninstall:** the uninstaller removes the scheduled task by name; no task is left behind after uninstall.

The tray process and the daemon logic are the same process (per ADR-038's Wails architecture — no separate service/companion split), started directly in the user's interactive session by the logon trigger. This is what makes ADR-040's tray-as-primary-interface possible in the first place: there is no Session 0 boundary to cross, because nothing here runs in Session 0.

## Consequences

**Positive:** satisfies ADR-042 (zero elevation, at any point) and ADR-040 (tray runs in the interactive session it needs to render in) simultaneously; gains retry-on-failure and a startup delay "for free" relative to the simplest possible mechanism (a registry key); closes a citation that previously pointed at content that didn't exist.

**Negative / trade-offs:** the daemon does not run before any user logs on, and stops when the user logs off — acceptable for this product, since a machine with nobody logged in has no interactive desktop for the tray to serve anyway, and the daemon's job (serving stored chunks, responding to audits) does not require a login session to make sense of running only when the machine is actually in use by someone; slightly more setup code than a bare registry write, using the COM Task Scheduler API (`ITaskService`) or `schtasks.exe` invoked by the installer.

**Open constraints:**

- FR-031's wording needs correcting beyond just the Windows clause: "macOS LaunchDaemon" and "Linux systemd unit" name the *system-wide* form of each platform's mechanism, the same category error "Windows Service" made. The per-user equivalents (macOS LaunchAgent; Linux systemd *user* unit) are the ones actually consistent with ADR-042's "standard-user constraints" framing — corrected text provided alongside this ADR. Deciding the macOS/Linux specifics in full is out of scope here (Windows ships first).
- The restart-after-failure count and startup delay above are starting values, not tuned — consistent with this research phase's other starting-value decisions (e.g., ADR-023's rate limiter), pending real-world observation once the Windows provider daemon is in testing.

## References

- Paper 50 — Windows background-app autostart mechanisms
- [ADR-038](./ADR-038-desktop-shell-wails.md) — Wails architecture; daemon and tray are one process, not a service/companion split
- [ADR-040](./ADR-040-system-tray-strategy.md) — tray as the Provider app's primary interface
- [ADR-042](./ADR-042-provider-app-least-privilege.md) — no elevation at any point, including auto-start registration
- [ADR-021](./ADR-021-p2p-transfer-protocol.md) — provider daemon networking is user-mode P2P, not a kernel-level driver requirement (the reason Tailscale's precedent does not transfer)
