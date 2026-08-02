# Paper 50 — Windows Background-App Autostart: Official Documentation and Production Precedent

**Authors:** Microsoft Learn documentation (Task Scheduler, Windows App SDK/UWP `StartupTask`, Session 0 isolation); Tailscale Inc. (tailscale/tailscale issue tracker, IPN service architecture)
**Venue / Year:** Official documentation and issue trackers, retrieved 2026
**Citations:** not applicable — documentation and issue-tracker sources, not an academic paper
**Topics:** #21 Desktop/Institutional Compute Harvesting Precedent, #22 Desktop Application Shell Architecture
**ADRs produced:** ADR-047 (Windows provider-daemon autostart mechanism)

---

## Problem Solved

Three existing decisions do not agree with each other. FR-031 requires the daemon auto-start "using the platform-appropriate mechanism (**Windows Service**, macOS LaunchDaemon, Linux systemd unit)." ADR-040 makes the system tray the Provider app's *primary* interface. ADR-042 requires no elevated or administrator privilege at any point, including auto-start registration. A genuine Windows Service needs administrator rights to install via the Service Control Manager, and runs in Session 0 — architecturally isolated from the interactive desktop, unable to show a tray icon at all. Implementing FR-031 literally would produce something ADR-040 cannot be built on top of. This review surveys the actual Windows-native alternatives, rather than assuming the first one considered (a per-user registry entry) is the only option, per the instruction to look for more solutions before falling back to it.

---

## Key Findings

**A standard user can register their own logon-triggered scheduled task without elevation — but only under specific conditions.** Microsoft's own Task Scheduler documentation and community troubleshooting sources are consistent on this: a logon trigger scoped to *the creating user's own account* (not "all users," not a different user, no stored credentials, not "run with highest privileges") does not require administrator rights to create. Elevation is required specifically for: registering a task for another user or all users, "run whether user is logged on or not" (which requires storing credentials as LSA secrets), and "run with highest privileges." None of these apply to a per-user provider daemon registering itself, at install time, to start when *that* user logs on.

**Task Scheduler's advantages over a bare registry Run key are real, not cosmetic.** Beyond the trigger itself, Task Scheduler exposes retry-on-failure, a delay-after-logon option (useful for letting the network stack settle before the daemon tries to reach the microservice), and a queryable run history — all useful for a background daemon whose entire value proposition is uptime. A `HKCU\...\Run` key or a Startup-folder shortcut is simpler to implement but gives none of this, and offers nothing a Task Scheduler logon trigger doesn't already provide at equivalent (non-elevated) privilege.

**The modern, most Microsoft-endorsed pattern — UWP/MSIX `StartupTask` — was considered and is not the right fit here, for a packaging reason, not a privilege reason.** Windows' `StartupTask` API gives the user OS-level visibility and control (Settings → Apps → Startup) without touching the registry or Task Scheduler directly. It requires the app be packaged as MSIX with a `StartupTask` extension declared in its manifest. ADR-041 already decided Windows packaging is Wails' built-in NSIS installer, which produces a conventional unpackaged Win32 executable, not an MSIX package. Adopting `StartupTask` would mean reopening ADR-041's packaging decision to gain a startup mechanism, not simply choosing a startup mechanism — a larger, separate trade-off than this ADR is scoped to make.

**Tailscale is not actually the no-elevation precedent it first appears to be, and citing it as one would be citing it inaccurately.** Tailscale's own issue tracker contains an open, unresolved feature request titled "Windows: Install/run Tailscale in user mode (without admin privileges)" — confirming that, as shipped, Tailscale's Windows client requires administrator rights. Its architecture is a split: `tailscaled.exe` runs as a genuine Windows Service (SYSTEM, Session 0), while a separate process, `tailscale-ipn.exe`, runs in the interactive user session and renders the tray icon, talking to the service over local IPC. This is a real, working, production-proven pattern for *exactly* the Session-0-isolation problem ADR-040 would otherwise hit — but Tailscale needs it because its actual job (installing and driving a virtual network adapter, WinTUN) requires kernel-level access no standard user account has, under any design. Vyomanaut's provider daemon does ordinary outbound, user-mode P2P networking (already working via QUIC/libp2p, ADR-021) — it has no equivalent requirement for elevated access, so it has no equivalent reason to accept Tailscale's elevated-install cost. The applicable precedent for a background app that genuinely needs no special OS access is the Dropbox/Syncthing pattern: ordinary per-user autostart, no service, no elevation, tray runs directly in the user's own session because the whole process already lives there.

**Session 0 isolation is a hard OS boundary, not a configuration choice.** Since Windows Vista, services run in a session isolated from any interactive desktop specifically so a service cannot present UI to, or receive input from, an interactive user session — a deliberate security boundary, not an oversight. No configuration flag reverses this for a standard Windows Service. The only production pattern that combines a Session-0 service with a visible tray icon is the split-process design Tailscale uses — and that pattern's precondition (an elevated, admin-installed service half) is exactly what ADR-042 rules out.

---

## Trade-offs

| Chosen | Over | Consequence |
| --- | --- | --- |
| Per-user Task Scheduler logon trigger, registered by the installer running as the current user | A `HKCU` Run key or Startup-folder shortcut | Gains retry-on-failure, startup delay, and run history at effectively the same (non-elevated) privilege cost — a strictly better version of the same category of mechanism |
| Per-user Task Scheduler logon trigger | UWP/MSIX `StartupTask` | Avoids reopening ADR-041's NSIS packaging decision, at the cost of not getting the OS-native Settings → Apps → Startup visibility MSIX apps get for free |
| Ordinary per-user process (Dropbox/Syncthing pattern) | A split Session-0-service + interactive-companion design (Tailscale pattern) | Avoids the elevated install step and the added attack surface of a SYSTEM-level component spawning processes into a user's session, at the cost of not being able to run before any user logs on — acceptable, since a provider daemon with nobody logged in has no desktop session to be useful to anyway |

---

## Breaks in Our Case

- **`ux-decisions.md`'s own precedent-apps section cites Tailscale as the design bar to hold Vyomanaut to**
  ≠ **Tailscale's Windows client requires administrator privileges and a genuine Windows Service, which ADR-042 explicitly rules out**
  → The design-quality bar Tailscale sets (make a hard technical problem feel effortless) still holds. The specific *architecture* it uses to show a tray icon from a privileged service does not transfer, because the reason it needs that architecture (kernel network-driver access) does not apply to Vyomanaut. Citing Tailscale as precedent for the autostart *mechanism* specifically would have been an inaccurate reading of what Tailscale actually does.

- **FR-031 names "Windows Service" as if it were interchangeable with "macOS LaunchDaemon" and "Linux systemd unit" in the same requirement row**
  ≠ **A macOS LaunchDaemon and a Linux systemd *system* unit also typically run outside the interactive user session (root-owned, akin to Session 0) — but a LaunchAgent (per-user) and a systemd *user* unit are the actual per-user equivalents already implied by ADR-042's "standard-user constraints" framing**
  → FR-031's wording conflates the elevated, system-wide form of each platform's mechanism with the per-user form. The fix below applies the same correction Windows needs to the requirement's general framing, not just its Windows clause.

---

## Decisions Influenced

- **ADR-047 (Windows provider-daemon autostart mechanism)** `NEW`
  Per-user Task Scheduler logon trigger, registered at install time under the installing user's own account, no elevation, no stored credentials.
  *Because:* it is the only option surveyed that satisfies ADR-042 (no elevation, ever) and ADR-040 (tray must run in the interactive session) simultaneously, while giving strictly more reliability tooling than the simplest possible per-user mechanism (a Run key).

- **FR-031 correction (`requirements.md`)** `NEW — text provided separately`
  "Windows Service" replaced with the per-user Task Scheduler mechanism; "macOS LaunchDaemon" corrected to "macOS LaunchAgent" and "Linux systemd unit" corrected to "Linux systemd user unit," for the same per-user-vs-system-wide reason.

---

## Disagreements

- **A case can be made that a genuine Windows Service is still worth the elevated-install cost, for reliability reasons — a service can be configured to restart automatically on crash via the Service Control Manager's recovery options, which a Task Scheduler task cannot do in quite the same way.**
  *Implication for us:* Task Scheduler does have its own retry/restart options (on a schedule, or "restart every N minutes if the task fails," up to a configurable count) which cover most of the same need without elevation. The gap is real but narrow, and does not outweigh reopening ADR-042 for every provider on every install. Worth revisiting only if real-world crash-recovery data post-launch shows Task Scheduler's retry model is insufficient.

---

## Open Questions

None raised beyond what ADR-047 resolves directly.

**Note on citation convention:** nearly every paper in this repo (Papers 1–48) ends with "See `open-questions.md`" — that file does not exist anywhere in the repository, and `architecture.md`/`failure-analysis.md` cite it several times more as a place to log new questions going forward. The real open-questions ledger, at least for Papers 45 onward, is `requirements.md` §10.1 ("Product Open Questions") — Q45-1 through Q49-1 all live there. This paper and Paper 49 cite that location directly. Whether `open-questions.md` should be created retroactively for Papers 1–44's citations, or those citations corrected to point at `requirements.md` instead, is a repo-wide cleanup this review did not attempt — flagging it here rather than silently deciding it.
