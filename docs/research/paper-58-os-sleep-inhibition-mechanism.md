# Paper 58 - OS-Level Sleep and Lid-Close Inhibition Mechanisms (Windows, macOS, Linux)

**Authors:** Microsoft (PowerToys documentation, Win32 API reference); Apple (caffeinate/IOKit documentation); freedesktop.org (systemd-logind documentation)
**Venue / Year:** vendor/project documentation, continuously maintained; retrieved 2026
**Citations:** not applicable - primary vendor documentation and public issue trackers, not academic sources
**Topics:** #11 Background Execution
**ADRs produced:** ADR-057

---

## Problem Solved

The proposed background-operations table (Windows: "Do Nothing on Lid Close" + PowerToys Awake; macOS: Amphetamine or caffeinate; Linux: systemd-logind HandleLidSwitch=ignore) needed checking against current, primary-source behavior before being adopted, because it mixes two mechanisms that solve different problems on at least one platform. ADR-009 covers auto-start-on-boot for the V2 desktop provider but says nothing about lid-close behavior specifically - this is genuinely unaddressed ground, not a gap in an existing decision.

---

## Trade-offs

| Chosen (recommendation below) | Over | Consequence |
|---|---|---|
| Daemon calls native OS sleep-inhibition APIs directly, no third-party utility required | Asking the user to install PowerToys/Amphetamine/edit config files | More implementation work inside the Go daemon (platform-specific code for three OSes); removes a setup step and a dependency from the provider onboarding flow, directly relevant to the intake-friction goal running through the rest of this research pass |

---

## Breaks in Our Case

- **The proposed table assumes PowerToys Awake handles lid-close sleep on Windows**
  != **it does not.** Two open issues on the PowerToys GitHub tracker (#13785, #16422) confirm this is a known, unimplemented feature request - Awake prevents idle-timeout sleep and can keep the display on, but explicitly does not override the lid-close action. The mechanism that actually controls lid-close behavior on Windows is the native "choose what closing the lid does" setting (powercfg /setacvalueindex ... lidaction 0, or the equivalent Group Policy for managed machines).
  -> PowerToys Awake is not part of the correct solution for the stated problem (laptop closes, keep running). It solves a different, adjacent problem (prevent idle sleep with the lid open). Drop it from the design; it adds an unnecessary dependency without addressing lid-close at all.

- **The proposed table treats these as user-facing utilities the provider installs**
  != **all three platforms expose the underlying mechanism as an API or command the daemon can call directly, without any user-facing installation step**
  -> Windows: SetThreadExecutionState(ES_CONTINUOUS | ES_SYSTEM_REQUIRED | ES_AWAYMODE_REQUIRED) from within the daemon process (Win32 API, reachable from Go via golang.org/x/sys/windows), plus setting lid-action to "do nothing" via powercfg at install time. macOS: caffeinate is a built-in OS binary (no Amphetamine needed) - the daemon can spawn caffeinate -s -i as a managed child process, or call IOKit's IOPMAssertionCreateWithName directly. Linux: systemd-logind exposes sleep/idle inhibitor locks over D-Bus (org.freedesktop.login1.Manager.Inhibit), which are per-application and don't require editing /etc/systemd/logind.conf system-wide.
  -> The better design removes the third-party-utility dependency from the proposed table entirely and has the daemon manage its own wake-lock, consistent with the intake-friction goal elsewhere in this research pass: fewer manual setup steps a provider has to get right.

- **systemd-logind's HandleLidSwitch=ignore (the proposed Linux mechanism) is a system-wide config file edit**
  != **inhibitor locks achieve the same effect per-application, without requiring root access or a system-wide policy change that affects every application on the machine, not just the provider daemon**
  -> Inhibitor locks are the better default; HandleLidSwitch=ignore remains a reasonable fallback for users who want it system-wide, but should not be the primary recommendation.

- **A real operational risk not mentioned in the original proposal:** forcing a laptop to stay awake indefinitely while running on battery, not just plugged in, risks a full battery drain during unattended operation
  -> The daemon's wake-lock behavior should apply only while the machine is on AC power by default (matching how powercfg's AC/DC value split already works, and how Bolosky's desktop-uptime findings, already cited in ADR-009, favor plugged-in machines as the more reliable provider population anyway). Battery-powered wake-lock should be an explicit opt-in, not the default.

---

## Decisions Influenced

- **[ADR-057](../decisions/ADR-057-native-sleep-inhibition.md) [#11 Background Execution]** `ACCEPTED - EXTENDS ADR-009`
  The provider daemon manages its own sleep/lid-close inhibition natively per platform, AC-power-only by default, with no third-party utility dependency.
  *Because:* the proposed utility-based approach either doesn't solve the stated problem (PowerToys Awake and lid-close) or introduces an avoidable setup dependency (Amphetamine, manual logind.conf edits) where a native API call would do.

---

## Disagreements

- It could be argued that shelling out to caffeinate/powercfg as subprocesses is simpler and less risky than direct API calls (IOPMAssertionCreateWithName, SetThreadExecutionState) that require more careful lifecycle management (the assertion/handle must be released cleanly on daemon shutdown or the wake-lock leaks).
  *Implication for us:* ADR-057 takes this seriously and recommends the subprocess/command approach for macOS and Windows's lid-action setting specifically (simpler, well-tested, harder to leak), reserving direct API calls for the parts (Windows SetThreadExecutionState, Linux D-Bus inhibitor) where no equivalent simple command exists.

---

## Open Questions

See [open-questions.md](open-questions.md) — question Q58-1.
