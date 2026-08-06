# ADR-057 - Native Sleep and Lid-Close Inhibition, AC-Power Only by Default

**Status:** Proposed
**Topic:** #11 Background Execution
**Supersedes:** -- (extends ADR-009; does not change auto-start-on-boot behavior)
**Superseded by:** --
**Research source:** Paper 58

---

## Context

ADR-009 covers desktop-only background execution and auto-start-on-boot, but says nothing about what happens when a laptop's lid closes. This matters specifically for V3's primary target provider base -- engineering students on laptops that get closed and opened frequently, unlike the always-on desktop assumption ADR-009 and ADR-005's MTTF figures (Bolosky) were built around. Without an explicit mechanism, closing the lid suspends the daemon and the provider silently drops out of every audit until the lid reopens, directly hurting the reliability score the rest of this system is built to measure fairly.

## Options Considered

| Option | Pros | Cons |
| --- | --- | --- |
| No mechanism; rely on the provider to change their own OS power settings | Zero engineering effort | Silently excludes the exact provider population V3 is targeting; most users never touch default power settings |
| User-installed third-party utility (PowerToys Awake, Amphetamine) plus manual OS setting changes, as originally proposed | Uses existing, well-known tools | PowerToys Awake does not actually address lid-close (Paper 58); still requires the provider to correctly perform multiple manual setup steps, working against the intake-friction goal elsewhere in this research pass |
| **Daemon-managed native wake-lock per platform, AC-power only by default, no third-party dependency** | Zero manual setup steps for the provider; uses the actual correct mechanism per platform; matches the intake-friction goal directly | More implementation work up front (three platform-specific code paths); requires careful lifecycle management so the wake-lock doesn't leak past daemon shutdown |

## Decision

The provider daemon manages sleep/lid-close inhibition itself, natively, per platform, with no user-facing utility installation required:

- **Windows:** at install time, set lid-close action to "do nothing" while on AC power only, via `powercfg /setacvalueindex <scheme> SUB_BUTTONS LIDACTION 0` (leaving the DC/battery value at the user's existing default). While running, the daemon additionally calls `SetThreadExecutionState(ES_CONTINUOUS | ES_SYSTEM_REQUIRED | ES_AWAYMODE_REQUIRED)` to prevent idle-timeout sleep during active operation.
- **macOS:** the daemon manages a `caffeinate -s -i` child process for the duration it needs the machine awake (`-s` = prevent sleep on AC power, deliberately omitting `-d`/display-related flags since a headless provider daemon does not need the screen kept on). No installation of Amphetamine or any third-party app.
- **Linux:** the daemon acquires a `systemd-logind` inhibitor lock over D-Bus (`org.freedesktop.login1.Manager.Inhibit`, types `sleep` and `handle-lid-switch`) for its own process, rather than editing `/etc/systemd/logind.conf` system-wide. This is scoped to the daemon and does not change lid-close behavior for any other application on the machine.
- **AC-power-only by default, on all three platforms.** Wake-lock while running on battery is an explicit opt-in the provider can enable, not the default -- unattended full battery drain is a real risk this ADR is not willing to impose silently.
- **Lifecycle:** the wake-lock/inhibitor must be released cleanly on daemon shutdown (including crash-recovery paths) to avoid leaving the machine permanently unable to sleep if the daemon exits abnormally.

This extends, and does not modify, ADR-009's auto-start-on-boot and CPU-budget behavior.

## Consequences

**Positive:**

- Removes lid-close as a silent, unexplained cause of dropped audits and reliability-score damage for the laptop-heavy V3 provider base
- No manual setup step, no third-party utility -- consistent with the intake-friction goal running through the rest of this research pass
- Corrects a real error in the originally proposed table (PowerToys Awake does not do what it was proposed for) before it reached implementation

**Negative / trade-offs:**

- Three separate platform-specific implementations to build and maintain, versus the single "tell the user to install X" approach originally proposed
- Windows and macOS approaches rely on subprocess/registry-level calls that need testing across OS versions; not guaranteed stable indefinitely as vendors change these subsystems
- AC-only default means a provider running purely on battery for extended periods (no adapter available) does not get the reliability benefit unless they explicitly opt in

**Open constraints:**

- Privilege requirements for `powercfg` and D-Bus inhibitor calls have not been checked against ADR-042's least-privilege design for the provider app (see open question below)
- No fallback is specified for platforms/configurations where the native mechanism is blocked by managed-device policy (e.g., a corporate/university-managed laptop with Group Policy overriding lid-action) -- the daemon should detect this failure case rather than silently assume the wake-lock succeeded, but detection logic is not specified here

## References

- [Paper 58 - OS-level sleep-inhibition mechanisms](../research/paper-58-os-sleep-inhibition-mechanisms.md): source of the corrected mechanism-per-platform findings
- [ADR-009](ADR-009-background-execution.md): the ADR this decision extends
- [ADR-042](ADR-042-provider-app-least-privilege.md): least-privilege design this decision's privilege requirements must be checked against

---
