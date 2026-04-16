# pipower

**Version:** V2.3  
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
| Alarm guard | Intercepts every shutdown to set the wake epoch |
| Test gate | Requires a verified halt→wake cycle before enabling |
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
pipower --version   # confirm: pipower V2.3
```

**Step 2 — Install systemd units**
```bash
sudo pipower install
```

**Step 3 — Set your schedule**
```bash
sudo pipower set --wake 06:30 --sleep 22:00 --timezone America/Toronto
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

## Configuration

Config file: `/etc/pipower/config.json`

```json
{
  "wake_time":   "06:30",
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
| `/etc/systemd/system/pipower-alarm.service` | Alarm guard — active on every boot |
| `/etc/systemd/system/pipower-shutdown.service` | Triggered by timer |
| `/etc/systemd/system/pipower-shutdown.timer` | Fires at sleep_time |

## NTP / time sync

chrony is used for time synchronization with the following source hierarchy:

| Source | Type | Notes |
|--------|------|-------|
| `time.nrc.ca` | Stratum 2 | NRC main lab — cesium atomic clock, preferred |
| `time.chu.nrc.ca` | Stratum 2 | NRC CHU site — separate network, preferred |
| `ca.pool.ntp.org` | Pool | Canadian fallback |
| `time.cloudflare.com` | Stratum 2 | NTS authenticated fallback (TCP 4460) |

See `pipower_setup_guide_V2.3_ckmmconsulting.md` for full chrony configuration.

## Troubleshooting

**Pi shuts down immediately after `pipower enable`**  
This should not occur in V2.3. If it does, mount the bootfs and add to
`/boot/firmware/cmdline.txt` (single line, space-separated):
```
systemd.mask=pipower-shutdown.timer systemd.mask=pipower-shutdown.service
```
Boot, SSH in, redeploy, then remove the masks and reboot.

**`pipower enable` returns error: not tested yet**  
Run the test cycle first:
```bash
sudo pipower test-alarm --minutes 5 --halt
```

**RTC alarm not being set**  
Check the alarm service journal from the last shutdown:
```bash
journalctl -u pipower-alarm.service -b -1
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
| V2.3 | 2026-04-16 | BUGFIX: `Requires=` in timer unit caused instant shutdown on enable. Removed `Requires=` and redundant `Unit=` — systemd auto-links timer→service by name. |
| V2.2 | 2026-04-16 | BUGFIX: Removed `OnBootSec=2min` — OR'd with `OnCalendar` causing immediate fire |
| V2.1 | 2026-04-16 | BUGFIX: `install` safety gate — refuses to re-enable timer if `tested=False` |
| V2.0 | 2026-04-15 | Test-gate safety: `enable` blocked until halt→wake test confirmed |
| V1.9 | 2026-04-15 | Code documentation pass |
| V1.8 | 2026-04-15 | BUGFIX: Removed timezone from `OnCalendar`; added `OnBootSec` guard |
| V1.7 | 2026-04-15 | Comprehensive inline documentation |
| V1.6 | 2026-04-13 | Added `--verbose` and `--debug` flags |
| V1.5 | 2026-04-13 | NRC stratum-2 NTP sources as preferred primary |
| V1.4 | 2026-04-13 | Install location reverted to `/usr/local/bin` |
| V1.1 | 2026-04-13 | chrony NTP synchronization |
| V1.0 | 2026-04-13 | Initial release |

