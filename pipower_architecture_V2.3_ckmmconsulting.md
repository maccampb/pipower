# pipower — Architecture & System Overview
**Version:** V2.3
**Date:** 2026-04-16
**Author:** ckmmconsulting

---

## Changelog

| Version | Date | Notes |
|---------|------|-------|
| V2.3 | 2026-04-16 | BUGFIX: **Root cause of immediate shutdown.** `Requires=pipower-shutdown.service` in timer `[Unit]` caused systemd to activate (start) the service immediately on timer activation, not on schedule. Removed `Requires=` and redundant `Unit=` directive — systemd auto-links `pipower-shutdown.timer` → `pipower-shutdown.service` by matching base names. |
| V2.2 | 2026-04-16 | BUGFIX: `OnBootSec=2min` removed from `pipower-shutdown.timer`. systemd OR's `OnCalendar` and `OnBootSec` — causes immediate fire if system already up longer than `OnBootSec` when timer is enabled. `OnCalendar` + `Persistent=false` only is correct. |
| V2.1 | 2026-04-16 | BUGFIX: `cmd_install` legacy config migration and safety gate. Migrates pre-V2.0 configs missing `tested`/`test_epoch` fields. Refuses to re-enable shutdown timer on install if `tested=False`. Prevents shutdown loop on reinstall over legacy config. |
| V2.0 | 2026-04-15 | Version alignment: script promoted to V2.0 to match documentation suite. |
| V1.9 | 2026-04-15 | Test-gate safety mechanism: `_post-boot` boot detection, `tested` and `test_epoch` config fields, `enable` gated on `tested=True`, `reset-test` command. `alarm_service_content()` ExecStart updated from `/bin/true` to `_post-boot`. Boot sequence and CLI tables updated. |
| V1.8 | 2026-04-15 | BUGFIX: OnCalendar timezone string removed (caused immediate timer firing on some systemd versions). `OnBootSec=2min` guard added. Config schema and shutdown_timer_content() updated. |
| V1.7 | 2026-04-14 | Updated NTP source table: Cloudflare `^-` status clarified; NRC legacy MD5 auth documented; NRC does not support NTS RFC 8915; NTS port 4460 confirmed open; `sudo` required for `chronyc authdata`. |
| V1.6 | 2026-04-13 | Added global `--verbose` (`-v`) and `--debug` (`-d`) flags. Updated CLI command table. |
| V1.5 | 2026-04-13 | Updated NTP source table and timekeeping chain: NRC stratum-2 servers (`time.nrc.ca`, `time.chu.nrc.ca`) added as preferred primary sources; `ca.pool.ntp.org` demoted to fallback. |
| V1.4 | 2026-04-13 | Reverted to `/usr/local/bin/pipower` |
| V1.3 | 2026-04-13 | `/etc/profile.d` PATH entry (superseded) |
| V1.2 | 2026-04-13 | `/opt/pipower/pipower` install location (superseded) |
| V1.1 | 2026-04-13 | Added Section 3 — Time Synchronization Architecture |
| V1.0 | 2026-04-13 | Initial release |

---

## 1. Purpose

**pipower** automates daily wake/sleep power cycles on a Raspberry Pi 5 using the board's built-in real-time clock (RTC). It eliminates the need for manual power intervention by:

- Shutting the Pi down at a configured time each evening via a systemd timer.
- Setting an RTC wake alarm just before halt so the PMIC restarts the board at a configured morning time.
- Exposing all scheduling through a single CLI tool backed by a JSON config file.

---

## 2. Hardware Foundation

### 2.1 Raspberry Pi 5 RTC

The Pi 5 includes an onboard **PCF85063A** real-time clock connected to the PMIC (RP1 power management IC) via a **firmware mailbox interface** — not directly via I2C as on older models. The Linux kernel exposes the RTC through the standard sysfs interface at:

```
/sys/class/rtc/rtc0/wakealarm
```

Writing a **UTC epoch integer** to this file programs the alarm. On halt, if the PMIC is in STANDBY mode, it monitors this alarm and restores power when the epoch is reached.

### 2.2 RTC Battery (J5 / BAT Connector)

A battery connected to the **J5 (BAT)** header maintains RTC timekeeping during power loss. This is a **timekeeping-only** function — the battery does not power the Pi or execute the wake.

Supported cell types:
- ML2032 / LIR2032 rechargeable lithium (preferred — trickle charger compatible)
- Supercapacitor (short retention, no trickle charge concern)
- **Do not use:** primary CR2032 lithium (non-rechargeable, incompatible with trickle charger)

### 2.3 External 5V Power Requirement

**The external 5V supply must remain present during halt.** In PMIC STANDBY mode the board draws ~3mA from the 5V rail. The RTC alarm instructs the PMIC to restore full power — the PMIC cannot do this without the supply present.

> This is a normal operating requirement. A standard wall adapter or always-on UPS satisfies it.

### 2.4 PMIC STANDBY vs. Default Halt

| State | Power draw | Wake sources | EEPROM setting |
|-------|-----------|--------------|----------------|
| Default halt | Higher (3V3/5V rails partially active) | GPIO3, power button, RTC alarm | `POWER_OFF_ON_HALT=0` (default) |
| PMIC STANDBY | ~3mA | Power button, RTC alarm | `POWER_OFF_ON_HALT=1` |

Setting `POWER_OFF_ON_HALT=1` in the bootloader EEPROM is **recommended** for minimal power draw. Wake-from-alarm functions in both modes.

---

## 3. Time Synchronization Architecture

Accurate timekeeping is critical to pipower correctness. The shutdown timer relies on the system clock; the wake alarm relies on the RTC. **chrony** bridges both, keeping the system clock disciplined to stratum 2 NTP sources and writing that accuracy back to the RTC every 11 minutes.

### 3.1 Timekeeping Chain

```
  UTC(NRC) — cesium atomic clock ensemble, Ottawa
          │
          ▼
  NRC stratum-2 servers (preferred)
  ┌─────────────────────────────────────────────┐
  │  time.nrc.ca      (NRC main lab, Ottawa)    │
  │  time.chu.nrc.ca  (CHU radio, separate net) │
  └─────────────────────────────────────────────┘
          │
          │  fallback if NRC unreachable
          ▼
  Additional stratum-2 sources
  ┌──────────────────────────────────────────────┐
  │  ca.pool.ntp.org   (pool, 3 sources)         │
  │  time.cloudflare.com  (NTS, authenticated)   │
  │  time1/2.google.com   (fallback)             │
  └──────────────────────────────────────────────┘
          │  NTP (UDP 123) / NTS (TCP 4460)
          ▼
  chronyd (system clock discipline)
  ├── Drift file: /var/lib/chrony/chrony.drift
  │
  ├── makestep 1.0 3
  │
  └── rtcsync (every 11 minutes)
          │
          ▼
  PCF85063A RTC (/dev/rtc0)
  ├── Maintains time during PMIC STANDBY (halt)
  ├── Powered by J5 battery during external 5V removal
  └── Read at boot by kernel → sets system clock
          │
          ▼
  pipower wake alarm epoch
```

### 3.2 NTP Source Selection

| Source | Type | Stratum | Auth | Notes |
|--------|------|---------|------|-------|
| `time.nrc.ca` | Single | 2 | None (public) | NRC main lab — atomic clock, UTC(NRC). `prefer` flag set. |
| `time.chu.nrc.ca` | Single | 2 | None (public) | NRC CHU site — separate network, same accuracy. `prefer` flag set. |
| `ca.pool.ntp.org` | Pool (3 sources) | 2 | None | Canadian geographic routing — fallback if both NRC servers unreachable |
| `time.cloudflare.com` | Single | 2 | **NTS (RFC 8915)** | Authenticated fallback. TCP 4460 confirmed open. Shows `^-` (fallback, healthy) because NRC `prefer` servers are selected. |
| `time1.google.com` | Single | 2 | None | Additional fallback |
| `time2.google.com` | Single | 2 | None | Additional fallback |

**Source priority:** The two NRC servers carry the `prefer` flag — chrony selects them whenever they are reachable and agree. Cloudflare appears as `^-` (candidate, not selected) in `chronyc sources` output; this is correct and expected behaviour, not an error.

**NRC authentication:** The NRC does not support NTS (RFC 8915). Their authenticated NTP offering uses legacy NTPv4 MD5 symmetric key authentication, available free of charge by contacting NRC directly (MSS-SMETime@nrc-cnrc.gc.ca). This is unnecessary for pipower's use case — unauthenticated NTP from atomic clock sources is fully sufficient.

**NTS (Network Time Security):** The Cloudflare source uses RFC 8915 NTS. TCP port 4460 is used for key exchange (ntske). Verified open via `nc -zv time.cloudflare.com 4460`. NTS auth status can be confirmed with `sudo chronyc authdata` (sudo required — plain `chronyc authdata` returns `501 Not authorised`).

**Stratum note:** The NRC stratum-2 servers are disciplined by cesium atomic clocks traceable to UTC(NRC). pipower's chrony instance operates at stratum 3.

### 3.3 RTC Synchronization via rtcsync

The `rtcsync` directive in `chrony.conf` instructs the Linux kernel to write the NTP-disciplined system time to the hardware RTC (`/dev/rtc0`) approximately every 11 minutes using `adjtimex()`. This is a kernel-level operation — not a direct chrony write — and is incompatible with the `rtcfile` directive.

**Effect on pipower:**
- The wake alarm epoch written to `/sys/class/rtc/rtc0/wakealarm` is computed from the system clock at shutdown time.
- Because chrony keeps the system clock within milliseconds of UTC, and writes that accuracy to the RTC every 11 minutes, the wake alarm epoch is reliable even after extended halt periods.
- If the RTC battery is depleted and the RTC loses time during halt, the drift file allows chrony to apply a frequency correction immediately on next boot, reducing recovery time.

### 3.4 fake-hwclock Removal

Debian Trixie ships `fake-hwclock` v0.14, which has a confirmed upstream bug (Debian bug #1093227) causing `fake-hwclock-load` to overwrite the RTC-sourced system time with a stale saved value on boot. This directly undermines the pipower wake cycle timing. The package must be removed when using a real RTC with chrony.

**Affected services:** `fake-hwclock-load.service`, `fake-hwclock-save.service`

### 3.5 Boot Sequence with chrony

```
Power on (PMIC restores from alarm)
      │
      ▼
Kernel reads PCF85063A RTC → sets system clock
  dmesg: rpi-rtc soc:rpi_rtc: setting system clock to...
      │
      ▼
systemd starts
      │
      ▼
chronyd starts
  ├── Loads drift file → applies frequency correction
  ├── makestep: if offset > 1s, step clock immediately
  └── Begins polling NTP sources (iburst = fast initial sync)
      │
      ▼
pipower-alarm.service starts
  ExecStart: pipower _post-boot
  ├── Checks config for test_epoch
  ├── If set: compares boot time to test_epoch (±5 min tolerance)
  │     Match → sets tested=True, clears test_epoch
  │     No match → normal boot, no action
  └── Service enters 'active' state (RemainAfterExit=yes)
      │
      ▼
pipower-shutdown.timer armed (only if enabled=True AND tested=True)
      │
      ▼
Normal operation
  └── chrony slews system clock continuously
      └── rtcsync writes to RTC every ~11 minutes
```

---

## 4. System Architecture

### 3.1 Component Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Raspberry Pi 5                        │
│                                                         │
│  ┌─────────────┐   sudo    ┌──────────────────────────┐ │
│  │    User     │──────────▶│   pipower CLI            │ │
│  │  (SSH/TTY)  │           │  /usr/local/bin/pipower  │ │
│  └─────────────┘           └──────────┬───────────────┘ │
│                                       │                  │
│                       ┌──────────────▼───────────────┐  │
│                       │  /etc/pipower/config.json    │  │
│                       │  wake_time, sleep_time,      │  │
│                       │  timezone, enabled           │  │
│                       └──────────────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │               systemd                            │   │
│  │                                                  │   │
│  │  pipower-shutdown.timer ──────▶ fires at         │   │
│  │       (OnCalendar, TZ-aware)    sleep_time       │   │
│  │              │                                   │   │
│  │              ▼                                   │   │
│  │  pipower-shutdown.service ────▶ systemctl        │   │
│  │                                 poweroff         │   │
│  │                                      │           │   │
│  │                                      ▼           │   │
│  │  pipower-alarm.service ───────▶ ExecStop fires   │   │
│  │    (active since boot,           (every shutdown) │   │
│  │     Before=shutdown.target)           │           │   │
│  │                                      │           │   │
│  └──────────────────────────────────────┼───────────┘   │
│                                         │               │
│                           ┌─────────────▼─────────────┐ │
│                           │  /sys/class/rtc/rtc0/     │ │
│                           │  wakealarm                │ │
│                           │  (UTC epoch written here) │ │
│                           └─────────────┬─────────────┘ │
│                                         │               │
│  ┌──────────────────────────────────────▼─────────────┐ │
│  │                  PCF85063A RTC                     │ │
│  │          (firmware mailbox interface)              │ │
│  └──────────────────────────────────────┬─────────────┘ │
│                                         │               │
│  ┌──────────────────────────────────────▼─────────────┐ │
│  │                PMIC (RP1)                          │ │
│  │   STANDBY mode — monitors alarm — restores power  │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  External 5V ──────────────────────────────────────▶   │
│  (must remain present during halt)                      │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Shutdown Sequence

```
sleep_time reached
      │
      ▼
pipower-shutdown.timer fires
      │
      ▼
pipower-shutdown.service starts
      │  ExecStart: systemctl poweroff
      ▼
systemd begins poweroff target
      │
      ▼
pipower-alarm.service stop triggered
      │  ExecStop: pipower _pre-shutdown
      │    - load config
      │    - compute next wake_time epoch (UTC, DST-aware)
      │    - write 0 to wakealarm (clear)
      │    - write epoch to wakealarm
      ▼
system halts → PMIC enters STANDBY (~3mA)
      │
      │  [time passes]
      │
      ▼
RTC alarm fires at wake_time epoch
      │
      ▼
PMIC restores full power → Pi boots normally
      │
      ▼
pipower-alarm.service starts
      │  ExecStart: _post-boot (checks test_epoch, sets tested=True on match)
      │  Ready to guard next shutdown
      ▼
Normal system operation
```

### 3.3 Manual Shutdown Path

When using `sudo pipower shutdown` (recommended over raw `shutdown -h now`):

```
sudo pipower shutdown
      │
      ▼
pipower CLI:
  - load config
  - compute next wake epoch
  - write to wakealarm
  - confirm to user
  - 3-second countdown (Ctrl+C cancellable)
  │
  ▼
systemctl poweroff
  │
  ▼
pipower-alarm.service ExecStop also fires
  (double-sets alarm — harmless, same epoch value)
  │
  ▼
system halts
```

> Note: Even a raw `sudo shutdown -h now` or `sudo halt` will trigger `pipower-alarm.service ExecStop` and set the alarm, as the service is ordered `Before=shutdown.target`. Using `sudo pipower shutdown` is preferred as it provides explicit confirmation.

---

## 5. Systemd Units

### 4.1 `pipower-alarm.service`

| Property | Value |
|----------|-------|
| Type | `oneshot` |
| RemainAfterExit | `yes` |
| ExecStart | `/bin/true` (no-op — service stays "active") |
| ExecStop | `/usr/local/bin/pipower _pre-shutdown` |
| After | `multi-user.target network-online.target` |
| Before | `shutdown.target poweroff.target halt.target reboot.target` |
| DefaultDependencies | `no` |
| WantedBy | `multi-user.target` |

**Role:** Persistent guardian. Active from boot. `ExecStop` intercepts every shutdown to set the RTC alarm.

### 4.2 `pipower-shutdown.timer`

| Property | Value |
|----------|-------|
| OnCalendar | `*-*-* {sleep_time}:00 {timezone}` |
| Persistent | `false` |
| Unit | `pipower-shutdown.service` |
| WantedBy | `timers.target` |

**Role:** Triggers the scheduled poweroff at the configured sleep time. Timezone is embedded in the `OnCalendar` expression (requires systemd ≥ 248, present in Debian Trixie / systemd 256+).

### 4.3 `pipower-shutdown.service`

| Property | Value |
|----------|-------|
| Type | `oneshot` |
| ExecStart | `/bin/systemctl poweroff` |
| DefaultDependencies | `no` |
| After | `pipower-alarm.service` |

**Role:** Minimal service triggered by the timer. Initiates system poweroff.

---

## 6. CLI Design

### 6.1 Global Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--verbose` | `-v` | Step-by-step progress output, prefixed `→` |
| `--debug` | `-d` | Low-level detail prefixed `[DEBUG]`: variable values, sysfs r/w, subprocess args, config content. Implies `--verbose`. |

Flags are parsed by the top-level `argparse` parser before subcommand dispatch and configure a module-level `Logger` object (`LOG`). All helpers and command functions call `LOG.step()` and `LOG.debug()` throughout, so output is interleaved in execution order. Debug mode also prints version, command name, Python version, and UID at startup.

### 6.2 Command Summary

| Command | Requires root | Description |
|---------|--------------|-------------|
| `install` | Yes | Create config, install units, check prerequisites |
| `uninstall` | Yes | Remove systemd units (config preserved) |
| `set` | Yes | Update wake/sleep time and/or timezone |
| `enable` | Yes | Enable shutdown timer **(requires tested=True)** |
| `disable` | Yes | Disable shutdown timer (alarm service stays active) |
| `status` | No | Display config, test status, timer, and RTC alarm state |
| `shutdown` | Yes | Set alarm then poweroff (preferred over raw shutdown) |
| `test-alarm` | Yes | Set test alarm N minutes out, optionally halt; `--halt` sets test_epoch and gates `enable` on wake confirmation |
| `reset-test` | Yes | Clear tested flag to require re-testing |
| `_post-boot` | Yes | Internal — called by systemd ExecStart on every boot |
| `_pre-shutdown` | Yes | Internal — called by systemd ExecStop on every shutdown |

### 6.3 Wake Time Calculation

The `_pre-shutdown` handler computes the **next occurrence** of `wake_time` in the configured timezone:

```
1. Get current datetime in configured timezone (ZoneInfo — DST-aware)
2. Build candidate = today at wake_time in that timezone
3. If candidate ≤ now: add 1 day
4. Convert to UTC epoch (int)
5. Write to /sys/class/rtc/rtc0/wakealarm
```

This logic handles DST transitions correctly because Python's `ZoneInfo` performs the UTC conversion accounting for the wall-clock offset at the candidate datetime, not the current offset.

**Edge cases handled:**

| Scenario | Behaviour |
|----------|-----------|
| Shutdown at 22:00, wake at 06:30 | Alarm set for tomorrow 06:30 local |
| Shutdown at 05:00 (manual), wake at 06:30 | Alarm set for today 06:30 local |
| DST spring-forward night | 06:30 local correctly maps to new UTC offset |
| DST fall-back night | 06:30 local correctly maps to new UTC offset |

---

## 7. Configuration

### 6.1 File Location

```
/etc/pipower/config.json
```

### 6.2 Schema

```json
{
  "_version": "V1.9",
  "_description": "pipower configuration - edit with: sudo pipower set ...",
  "wake_time": "06:30",
  "sleep_time": "22:00",
  "enabled": true,
  "timezone": "America/Toronto",
  "tested": true,
  "test_epoch": null
}
```

| Field | Type | Description |
|-------|------|-------------|
| `_version` | string | Config schema version (informational) |
| `wake_time` | string | HH:MM (24h) — time to power on each morning |
| `sleep_time` | string | HH:MM (24h) — time to power off each evening |
| `enabled` | bool | Whether the shutdown timer is active |
| `timezone` | string | IANA timezone name (e.g. `America/Toronto`) |
| `tested` | bool | True after a successful `test-alarm --halt` wake cycle |
| `test_epoch` | int/null | Wake epoch stored before test halt; cleared by `_post-boot` on confirmed wake |

### 6.3 Editing

Always use the CLI to edit config while the service is running:

```bash
sudo pipower set --wake 06:30 --sleep 22:00 --timezone America/Toronto
```

The CLI regenerates and reloads the systemd timer automatically after any change.

---

## 8. Prerequisites & Dependencies

### 8.1 Software

| Requirement | Detail |
|-------------|--------|
| OS | Debian Trixie (Debian 13) or Raspberry Pi OS Bookworm (Debian 12) |
| Python | 3.11+ (for `zoneinfo` stdlib module) |
| systemd | ≥ 248 (for timezone in `OnCalendar`; Trixie ships 256+) |
| `chrony` | NTP client + RTC sync via `rtcsync`; replaces `systemd-timesyncd` |
| `rpi-eeprom-config` | For EEPROM prerequisite check (from `rpi-eeprom` package) |
| `fake-hwclock` | Must be **removed** on Trixie (bug #1093227 corrupts RTC-sourced time) |

No third-party Python packages required. All dependencies are stdlib.

### 8.2 Hardware

| Requirement | Detail |
|-------------|--------|
| Board | Raspberry Pi 5 |
| RTC battery | ML2032 / LIR2032 / supercap on J5 (optional but recommended) |
| 5V supply | Must remain present during halt for wake to function |

### 7.3 EEPROM Bootloader

Set `POWER_OFF_ON_HALT=1` for deep PMIC standby (~3mA). Not strictly required for wake functionality but strongly recommended. Applied via `raspi-config` or `rpi-eeprom-config --edit`. Requires one reboot to take effect.

---

## 9. File Inventory

| Path | Description |
|------|-------------|
| `/usr/local/bin/pipower` | CLI script — locally-administered tool, system dependencies |
| `/etc/pipower/config.json` | Runtime configuration |
| `/etc/systemd/system/pipower-alarm.service` | Alarm guard service |
| `/etc/systemd/system/pipower-shutdown.service` | Scheduled shutdown service |
| `/etc/systemd/system/pipower-shutdown.timer` | Shutdown timer |

---

## 10. Operational Notes

### 9.1 Diagnostics

```bash
# Check alarm service is running
systemctl status pipower-alarm.service

# Check timer next fire time
systemctl status pipower-shutdown.timer

# View current RTC alarm (UTC epoch, empty after boot until next shutdown)
cat /sys/class/rtc/rtc0/wakealarm

# Check logs from last shutdown (alarm set confirmation)
journalctl -b -1 -u pipower-alarm.service

# Verify RTC is registered
dmesg | grep -i rtc
```

### 9.2 Known Limitations (V1.0)

- Daily schedule only — same wake/sleep times every day of the week.
- No per-day or weekend/weekday differentiation.
- No alerting if alarm set fails at shutdown (check journalctl).
- External 5V dependency — not suitable for battery-only deployments without additional hardware (e.g. UPS HAT or relay-switched supply).

---

## 11. Future Development (Candidate Features)

| Item | Priority |
|------|----------|
| Per-weekday schedule overrides | Medium |
| Webhook / email notification on alarm set failure | Medium |
| `pipower log` command — last N shutdown/wake events | Low |
| Systemd journal integration for structured alarm logging | Low |
| Battery-backed UPS HAT integration hooks | Low |
