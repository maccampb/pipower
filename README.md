# pipower

**Version:** V2.5  
**Author:** ckmmconsulting

Raspberry Pi 5 RTC scheduled power management CLI. Manages daily wake/sleep
power cycles using the Pi 5 built-in PCF85063A real-time clock — shutting the
Pi down at a configured time each evening and waking it automatically each
morning.

## What it does

| Feature | Description |
|---------|-------------|
| Scheduled shutdown | systemd timer fires at configured sleep time |
| RTC wake alarm | Programs PMIC to restore power at configured wake time |
| Alarm guard | ExecStop intercepts every manual shutdown to set wake epoch |
| Explicit alarm set | pipower-shutdown.service sets alarm via ExecStartPre before poweroff |
| Test gate | Requires a verified halt→wake cycle before enabling |
| Persistent log | Every shutdown and boot logged to `/var/log/pipower.log` |
| chrony sync | NRC atomic clock sources keep RTC accurate |
| CLI management | Single command manages all configuration and state |

## Prerequisites

- Raspberry Pi 5 (PCF85063A RTC required)
- Raspberry Pi OS Trixie (Debian 13) or Bookworm (Debian 12)
- Python 3.11+
- chrony installed and synced (`sudo apt install chrony`)
- RTC battery connected to J5 (BAT) connector
- `POWER_OFF_ON_HALT=1` set in bootloader EEPROM
- External 5V supply present during halt (required for PMIC wake)

## Installation

**Step 1 — Copy the script**
```bash
sudo cp pipower /usr/local/bin/pipower
sudo chmod +x /usr/local/bin/pipower
pipower --version   # confirm: pipower V2.5
```

**Step 2 — Install systemd units**
```bash
sudo pipower install
```

**Step 3 — Set your schedule**
```bash
sudo pipower set --wake 07:00 --sleep 22:00 --timezone America/Toronto
```

**Step 4 — Run the test cycle (mandatory)**
```bash
sudo pipower test-alarm --minutes 5 --halt
# Pi halts and wakes in ~5 minutes
# SSH back in and confirm:
sudo pipower status   # must show: Tested: Yes ✓
```

**Step 5 — Enable**
```bash
sudo pipower enable
```

**Step 6 — Verify**
```bash
sudo pipower status
```

## Commands

### Global flags

| Flag | Short | Description |
|------|-------|-------------|
| `--verbose` | `-v` | Step-by-step progress output |
| `--debug` | `-d` | Low-level detail — sysfs, subprocess, config. Implies `--verbose`. |

Flags must appear **before** the subcommand: `sudo pipower -v status`

### Command reference

| Command | Description |
|---------|-------------|
| `sudo pipower install` | Install systemd units, create config, check prerequisites |
| `sudo pipower uninstall` | Remove systemd units (config preserved) |
| `sudo pipower set --wake HH:MM --sleep HH:MM [--timezone TZ]` | Update configuration |
| `sudo pipower test-alarm --minutes N [--halt]` | Set test alarm; `--halt` required to gate `enable` |
| `sudo pipower enable` | Enable scheduled cycle **(requires test first)** |
| `sudo pipower disable` | Disable scheduled shutdown (alarm guard stays active) |
| `sudo pipower status` | Show configuration, test status, timer, and RTC alarm |
| `sudo pipower shutdown` | Set alarm then poweroff |
| `sudo pipower reset-test` | Clear tested flag to require re-testing |
| `pipower log [--lines N]` | Show persistent shutdown/boot log (no sudo needed) |

## Configuration

Config file: `/etc/pipower/config.json`

```json
{
  "wake_time":   "07:00",
  "sleep_time":  "22:00",
  "enabled":     true,
  "timezone":    "America/Toronto",
  "tested":      true,
  "test_epoch":  null
}
```

Edit with `sudo pipower set` — do not edit manually while enabled.

## File inventory

| Path | Description |
|------|-------------|
| `/usr/local/bin/pipower` | CLI script |
| `/etc/pipower/config.json` | Runtime configuration |
| `/var/log/pipower.log` | Persistent shutdown/boot event log |
| `/etc/systemd/system/pipower-alarm.service` | Alarm guard — active on every boot |
| `/etc/systemd/system/pipower-shutdown.service` | Sets alarm via ExecStartPre, then poweroff |
| `/etc/systemd/system/pipower-shutdown.timer` | Fires at sleep_time |

## Shutdown sequence

```
pipower-shutdown.timer fires at sleep_time
    │
    ▼
pipower-shutdown.service
    ExecStartPre: pipower _pre-shutdown   ← alarm set here (primary path)
    ExecStart:    systemctl poweroff
    │
    ▼
systemd shutdown
    pipower-alarm.service ExecStop: pipower _pre-shutdown  ← safety net
    │
    ▼
PMIC STANDBY (~3mA) — alarm armed
    │  [hours pass]
    ▼
RTC alarm fires → PMIC restores power → Pi boots at wake_time
```

## Diagnostics

**Check shutdown/boot history:**
```bash
pipower log
# Shows timestamped SHUTDOWN_START, ALARM_SET, and BOOT entries
```

**Check persistent journal (if enabled):**
```bash
journalctl -u pipower-alarm.service -b -1
```

**Enable persistent journal (recommended):**
```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

## NTP / time sync

chrony is used for time synchronization with the following source hierarchy:

| Source | Type | Notes |
|--------|------|-------|
| `time.nrc.ca` | Stratum 2 | NRC main lab — cesium atomic clock, preferred |
| `time.chu.nrc.ca` | Stratum 2 | NRC CHU site — separate network, preferred |
| `ca.pool.ntp.org` | Pool | Canadian fallback |
| `time.cloudflare.com` | Stratum 2 | NTS authenticated fallback (TCP 4460) |

See `pipower_setup_guide_V2.5_ckmmconsulting.md` for full chrony configuration.

## Troubleshooting

**Pi shuts down immediately after `pipower enable`**  
Should not occur in V2.5. If it does, mount the bootfs and add to
`/boot/firmware/cmdline.txt` (single line, space-separated):
```
systemd.mask=pipower-shutdown.timer systemd.mask=pipower-shutdown.service
```
Boot, SSH in, redeploy, remove the masks and reboot.

**`pipower enable` returns error: not tested yet**  
Run the test cycle first:
```bash
sudo pipower test-alarm --minutes 5 --halt
```

**Pi did not wake at scheduled time**  
Check the log first:
```bash
pipower log
# ALARM_SET present = alarm was written, PMIC/power issue
# ALARM_SET missing = _pre-shutdown did not run
```

**Wake alarm interface missing**
```bash
dmesg | grep -i rtc
# Should show: rpi-rtc soc:rpi_rtc: registered as rtc0
```

**chrony not syncing**
```bash
chronyc sources -v
timedatectl status
```

## Changelog

| Version | Date | Notes |
|---------|------|-------|
| V2.5 | 2026-04-17 | BUGFIX: `pipower-shutdown.service` now calls `pipower _pre-shutdown` via `ExecStartPre` before `systemctl poweroff`. Fixes ExecStop not being called reliably when poweroff is initiated from within a systemd service. |
| V2.4 | 2026-04-17 | Persistent log file `/var/log/pipower.log`. `pipower log` command. `TimeoutStopSec=30` on alarm service. |
| V2.3 | 2026-04-16 | BUGFIX: `Requires=` in timer unit caused instant shutdown on enable. |
| V2.2 | 2026-04-16 | BUGFIX: Removed `OnBootSec=2min` — OR'd with `OnCalendar` causing immediate fire. |
| V2.1 | 2026-04-16 | BUGFIX: `install` safety gate — refuses to re-enable timer if `tested=False`. |
| V2.0 | 2026-04-15 | Test-gate safety: `enable` blocked until halt→wake test confirmed. |
| V1.9 | 2026-04-15 | Code documentation pass. |
| V1.8 | 2026-04-15 | BUGFIX: Removed timezone from `OnCalendar`. |
| V1.7 | 2026-04-15 | Comprehensive inline documentation. |
| V1.6 | 2026-04-13 | Added `--verbose` and `--debug` flags. |
| V1.5 | 2026-04-13 | NRC stratum-2 NTP sources as preferred primary. |
| V1.4 | 2026-04-13 | Install location reverted to `/usr/local/bin`. |
| V1.1 | 2026-04-13 | chrony NTP synchronization. |
| V1.0 | 2026-04-13 | Initial release. |
