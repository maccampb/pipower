# pipower — Raspberry Pi 5 RTC Scheduled Power Management
## Setup & Usage Guide — V2.5 — ckmmconsulting

### Changelog
| Version | Date | Notes |
|---------|------|-------|
| V2.5 | 2026-04-17 | BUGFIX: `pipower-shutdown.service` now calls `pipower _pre-shutdown` via `ExecStartPre` before `systemctl poweroff`. Fixes ExecStop not being called reliably when poweroff is initiated from within a systemd service. ExecStop retained as safety net for manual shutdowns. |
| V2.4 | 2026-04-17 | Added persistent log file `/var/log/pipower.log` written at every shutdown and boot. New `pipower log` command. `TimeoutStopSec=30` added to alarm service. |
| V2.3 | 2026-04-16 | BUGFIX: **Root cause found.** `Requires=pipower-shutdown.service` in timer `[Unit]` section caused systemd to start the service immediately on timer activation — triggering instant shutdown on `pipower enable`. Removed `Requires=` and redundant `Unit=` from timer. systemd auto-links timer→service by name. |
| V2.2 | 2026-04-16 | BUGFIX: Removed `OnBootSec=2min` from timer unit. `OnCalendar` and `OnBootSec` are OR'd in systemd — timer fires immediately if system has been up longer than `OnBootSec` when `enable` runs. `OnCalendar` alone with `Persistent=false` is correct. |
| V2.1 | 2026-04-16 | BUGFIX: `install` now migrates legacy configs missing `tested`/`test_epoch` fields; enforces test gate on install — refuses to re-enable shutdown timer if `tested=False` even when preserved config has `enabled=True`. Prevents shutdown loop on reinstall. |
| V2.0 | 2026-04-15 | Test-gate safety: `enable` blocked until `test-alarm --halt` cycle completes successfully. `_post-boot` detects wake confirmation on reboot. New commands: `reset-test`. `status` shows test state. |
| V1.9 | 2026-04-14 | Updated NTS section: `^-` for Cloudflare is expected with NRC `prefer` servers selected; `chronyc authdata` requires sudo; `nc -zv` port test added; NRC legacy MD5 auth documented; NRC does not support NTS RFC 8915. |
| V1.8 | 2026-04-13 | Removed `hwclock` from all verification and troubleshooting steps — not installed by default on Trixie (`util-linux-extra`). Replaced with `timedatectl status` and `chronyc` throughout. |
| V1.7 | 2026-04-13 | Fixed `hwclock` commands: use full path `/usr/sbin/hwclock` (not in sudo PATH on Trixie); split combined code blocks into separate commands |
| V1.6 | 2026-04-13 | Added `--verbose` (`-v`) and `--debug` (`-d`) global flags; `--debug` implies `--verbose` |
| V1.5 | 2026-04-13 | chrony: NRC stratum-2 servers (`time.nrc.ca`, `time.chu.nrc.ca`) added as preferred primary sources; `ca.pool.ntp.org` demoted to fallback |
| V1.4 | 2026-04-13 | Reverted to `/usr/local/bin/pipower` |
| V1.3 | 2026-04-13 | `/etc/profile.d` PATH entry (superseded) |
| V1.2 | 2026-04-13 | `/opt` install location (superseded) |
| V1.1 | 2026-04-13 | Added chrony NTP time synchronization |
| V1.0 | 2026-04-13 | Initial release |

---

## Overview

**pipower** manages automatic daily wake/sleep cycles on the Raspberry Pi 5 using its built-in PCF85063A real-time clock. The Pi shuts down at a configured time each evening and the RTC alarm powers it back on each morning.

### How it works

1. A **systemd timer** fires at the configured sleep time and initiates a system poweroff.
2. A **systemd alarm service** runs on every boot and intercepts any shutdown (scheduled or manual), writing the next wake epoch to `/sys/class/rtc/rtc0/wakealarm` just before the Pi halts.
3. The Pi enters PMIC STANDBY (~3mA draw from your external 5V supply).
4. When the alarm fires, the PMIC powers the Pi back on and it boots normally.

> **Important:** The Pi 5 does NOT self-power from its RTC battery. The RTC battery (J5 connector) keeps the clock running while off. **External 5V power must remain present during halt** for the wake alarm to restart the board. This is normal — a wall adapter or always-on UPS satisfies this requirement.

---

## Prerequisites

### 1. Battery (J5 connector)

Connect a compatible RTC battery to the **J5 (BAT)** connector to the right of the USB-C port. This is only for RTC timekeeping — supercaps or ML2032/LIR2032 rechargeable cells are preferred. Do **not** use primary (non-rechargeable) lithium cells unless you disable the trickle charger.

### 2. Bootloader EEPROM — POWER_OFF_ON_HALT

For the deepest power-off state (~3mA PMIC standby), set `POWER_OFF_ON_HALT=1` in the bootloader EEPROM. Wake-from-alarm will still work without this, but the halt power draw will be higher.

**Option A — raspi-config (recommended):**
```bash
sudo raspi-config
# → Advanced Options → Bootloader Version
# → select the option that sets Full power off / POWER_OFF_ON_HALT=1
# Reboot to apply
```

**Option B — manual:**
```bash
sudo -E rpi-eeprom-config --edit
# Add or change: POWER_OFF_ON_HALT=1
# Save and quit, then reboot
```

**Verify after reboot:**
```bash
rpi-eeprom-config | grep POWER_OFF_ON_HALT
# Should show: POWER_OFF_ON_HALT=1
```

### 3. System clock / chrony NTP

pipower depends on an accurate system clock for the scheduled shutdown timer, and on the RTC (hwclock) for the wake alarm. **chrony** is used as the NTP client to keep both in sync. See **Section 4 — chrony Setup** below for the full installation procedure.

> **Why chrony over systemd-timesyncd?**
> chrony is more accurate for embedded systems with intermittent network access. Its `rtcsync` directive writes the NTP-disciplined time back to the hardware RTC every 11 minutes, ensuring the wake alarm is always based on accurate time even after extended power-off periods.

---

## 4. chrony Setup

Complete this section before installing pipower. chrony must be running and synced before the scheduled shutdown timer is enabled.

### Step 4.1 — Remove fake-hwclock (Trixie)

Debian Trixie ships with `fake-hwclock` v0.14, which has a known bug that overwrites the real RTC time with a stale saved value on boot. Remove it before enabling chrony's RTC sync.

```bash
sudo systemctl disable --now fake-hwclock-load.service fake-hwclock-save.service 2>/dev/null
sudo apt remove --purge fake-hwclock -y
```

Verify it is gone:
```bash
dpkg -l fake-hwclock 2>&1 | grep -E '^(ii|rc)'
# Should show nothing or 'rc' (removed, config only)
```

### Step 4.2 — Install chrony

Installing chrony automatically disables and removes `systemd-timesyncd`.

```bash
sudo apt update
sudo apt install chrony -y
```

### Step 4.3 — Configure chrony

Replace the default `/etc/chrony/chrony.conf` with the following pipower-tuned configuration. This uses Canadian stratum 2 NTP sources and enables RTC synchronization.

```bash
sudo cp /etc/chrony/chrony.conf /etc/chrony/chrony.conf.bak
sudo tee /etc/chrony/chrony.conf > /dev/null << 'EOF'
# /etc/chrony/chrony.conf
# pipower / ckmmconsulting — Pi 5 NTP + RTC sync configuration
# V1.1 — 2026-04-13

# ── NTP Sources ──────────────────────────────────────────────────────────────
# NRC (National Research Council Canada) stratum-2 servers — primary sources
# Atomic-clock disciplined, official Canadian time, free public access
# Two geographically separate sites on independent networks
server time.nrc.ca iburst prefer
server time.chu.nrc.ca iburst prefer

# Canadian NTP pool — fallback if both NRC servers are unreachable
pool ca.pool.ntp.org iburst maxsources 3

# Cloudflare NTS-secured stratum 2 — authenticated fallback
server time.cloudflare.com iburst nts

# Google public NTP — additional fallback
server time1.google.com iburst
server time2.google.com iburst

# ── NTP source directory (DHCP-provided sources) ─────────────────────────────
sourcedir /run/chrony-dhcp
sourcedir /etc/chrony/sources.d

# ── Key file ─────────────────────────────────────────────────────────────────
keyfile /etc/chrony/chrony.keys

# ── Drift file ───────────────────────────────────────────────────────────────
# Stores the measured frequency error of the system clock between restarts.
# This allows chrony to apply a correction immediately on next boot,
# reducing the time to achieve sync after a power cycle.
driftfile /var/lib/chrony/chrony.drift

# NTS cookie dump (retains NTS session state across restarts)
ntsdumpdir /var/lib/chrony

# ── Logging ──────────────────────────────────────────────────────────────────
logdir /var/log/chrony
# log tracking measurements statistics

# ── Clock discipline ─────────────────────────────────────────────────────────
# Reject sources with estimated error above this threshold (ppm)
maxupdateskew 100.0

# Step the system clock (rather than slew) if offset > 1s, but only on
# the first 3 clock updates after startup. Important for cold-boot accuracy
# when the RTC has drifted (e.g. after a long power-off with depleted battery).
makestep 1.0 3

# ── RTC synchronization ──────────────────────────────────────────────────────
# Enables kernel synchronization of the hardware RTC every 11 minutes.
# chrony writes the NTP-disciplined system time back to /dev/rtc0.
# This keeps the RTC accurate for pipower wake alarms.
# NOTE: Cannot be used with the 'rtcfile' directive.
rtcsync

EOF
```

### Step 4.4 — Restart and verify chrony

```bash
sudo systemctl restart chrony
sudo systemctl enable chrony

# Check chrony service status
sudo systemctl status chrony

# View time sources (wait ~30 seconds after restart for initial sync)
chronyc sources -v
```

Expected `sources -v` output (the `^*` line is the selected source):
```
MS Name/IP address         Stratum Poll Reach LastRx Last sample
=========================================================================
^* ntp1.example.ca               2   6   377    42   +123us[ +456us] +/- 5ms
^+ ntp2.example.ca               2   6   377    43    -12us[  -10us] +/- 4ms
^+ time.cloudflare.com           2   6   377    44    +89us[  +91us] +/- 3ms
```

Verify RTC is being updated:
```bash
# Check tracking summary — confirms chrony is synced and its source
chronyc tracking

# Confirm system clock and RTC agree
timedatectl status
# Good output looks like:
#   RTC time:                 (matches Universal time)
#   System clock synchronized: yes
#   NTP service:              active
#   RTC in local TZ:          no
```

> **Note:** `hwclock` is in the `util-linux-extra` package which is not installed by default on Debian Trixie. It is not needed — `timedatectl status` and `chronyc tracking` provide all the verification needed.

Force an immediate clock step and RTC sync (useful after initial setup):
```bash
sudo chronyc makestep
```

### Step 4.5 — NTS verification (Cloudflare source)

**First, test that TCP port 4460 is reachable** (required for NTS key exchange):
```bash
nc -zv time.cloudflare.com 4460
# Expected: Connection to time.cloudflare.com 4460 port [tcp/ntske] succeeded!
```

If the port is open, verify NTS is active in chrony (requires sudo):
```bash
chronyc sources -v | grep cloudflare
# Expected: ^- time.cloudflare.com  (^- means fallback, not selected — this is correct)
# Note: Cloudflare shows ^- not ^* because the NRC servers have the prefer flag.
# ^- is healthy — it means reachable and in the candidate pool.

sudo chronyc authdata
# Expected: a row for time.cloudflare.com showing NTS key ID and cookie count
```

> **Note:** `chronyc authdata` requires `sudo` — running without it returns `501 Not authorised`.

> **Note on NRC and NTS:** The NRC servers do **not** support NTS (RFC 8915). The NRC's
> authenticated NTP offering uses legacy MD5 symmetric key authentication, which requires
> contacting NRC directly (MSS-SMETime@nrc-cnrc.gc.ca) for a keyID and password. For
> pipower's use case this is unnecessary — unauthenticated NTP from atomic clock sources
> is fully sufficient. Cloudflare NTS covers the authenticated fallback role.

If TCP 4460 is blocked by your network, disable NTS on the Cloudflare entry:
```bash
sudo sed -i 's/server time.cloudflare.com iburst nts/server time.cloudflare.com iburst/' /etc/chrony/chrony.conf
sudo systemctl restart chrony
```

---

## 5. pipower Installation

### Prerequisites checklist before starting

- [ ] RTC battery connected to J5 connector
- [ ] EEPROM `POWER_OFF_ON_HALT=1` set (see Section 2)
- [ ] chrony installed and synced (see Section 4)
- [ ] `timedatectl status` shows correct timezone and `System clock synchronized: yes`
- [ ] Pi has a stable internet connection
- [ ] You have SSH access and can wait ~5 minutes for the test cycle

---

### Step 1 — Copy the script to its install location

```bash
sudo cp pipower /usr/local/bin/pipower
sudo chmod +x /usr/local/bin/pipower
```

Verify:
```bash
pipower --version
# Should print: pipower V2.3
```

---

### Step 2 — Run install

```bash
sudo pipower install
```

Expected output confirms:
- `[OK] RTC wake alarm interface` — RTC driver loaded
- `[OK] EEPROM: POWER_OFF_ON_HALT=1` — PMIC standby enabled
- Three systemd units installed
- `pipower-alarm.service` enabled and active
- Shutdown timer **NOT** started (requires test first)

---

### Step 3 — Set your schedule and timezone

```bash
sudo pipower set --wake 06:30 --sleep 22:00 --timezone America/Toronto
```

Confirm:
```bash
sudo pipower status
# Tested: No — run: sudo pipower test-alarm --minutes 5 --halt
# Enabled: No
```

---

### Step 4 — Run the test cycle

This is mandatory before `enable` will work. The test proves the full halt→wake mechanism functions on your hardware.

```bash
sudo pipower test-alarm --minutes 5 --halt
```

The Pi will:
1. Set the RTC alarm 5 minutes from now
2. Store the test epoch in config
3. Count down 5 seconds (Ctrl+C cancels)
4. Power off

**Wait ~5 minutes.** The Pi should power back on automatically.

SSH back in and check:
```bash
sudo pipower status
# Look for: Tested: Yes ✓
```

If status shows `Tested: Pending`, wait 30 seconds and check again — `_post-boot` runs early in the boot sequence and may not have completed yet.

If status still shows `Tested: No` after a full reboot, the wake mechanism did not work. Check:
```bash
journalctl -u pipower-alarm.service -b -1
dmesg | grep -i rtc
```

**Do not proceed to Step 5 until `Tested: Yes ✓` is confirmed.**

---

### Step 5 — Enable the schedule

```bash
sudo pipower enable
```

Expected output:
```
Scheduled shutdown enabled.
  Shutdown at: 22:00 America/Toronto
  Wake at:     06:30 America/Toronto
```

---

### Step 6 — Final verification

```bash
sudo pipower status
```

Expected output:
```
────────────────────────────────────────────────────────
  pipower V2.3 — Status
────────────────────────────────────────────────────────
  Config:       /etc/pipower/config.json
  Wake time:    06:30 America/Toronto
  Sleep time:   22:00 America/Toronto
  Enabled:      Yes
  Tested:       Yes ✓
  Current time: 2026-04-16 16:00:00 EDT

  Alarm service (pipower-alarm.service): active
  Shutdown timer (pipower-shutdown.timer): active
  Next shutdown:  2026-04-16 22:00:00 EDT

  RTC alarm:    not set (set at next shutdown)
  EEPROM:       POWER_OFF_ON_HALT=1 [OK]
────────────────────────────────────────────────────────
```

All six lines must be correct before considering the installation complete:
- `Enabled: Yes`
- `Tested: Yes ✓`
- `Alarm service: active`
- `Shutdown timer: active`
- `Next shutdown` showing tonight at sleep_time
- `EEPROM: POWER_OFF_ON_HALT=1 [OK]`

---

### Recovery procedure (if Pi shuts down immediately after enable)

This should not happen with V2.3 but if it does:

1. Mount the Pi's bootfs from another machine
2. Edit `/boot/firmware/cmdline.txt` — add to the end of the single line:
   ```
   systemd.mask=pipower-shutdown.timer systemd.mask=pipower-shutdown.service
   ```
3. Boot the Pi, SSH in
4. Deploy the latest script: `sudo cp pipower /usr/local/bin/pipower`
5. Run `sudo pipower uninstall` then `sudo pipower install`
6. Edit `cmdline.txt` again and remove the two `systemd.mask=` entries
7. Reboot and repeat from Step 3

---

## Command Reference

### Global Flags

These flags apply to all commands and must be placed **before** the subcommand:

| Flag | Short | Description |
|------|-------|-------------|
| `--verbose` | `-v` | Print step-by-step progress as each command executes |
| `--debug` | `-d` | Print low-level detail: variable values, sysfs reads/writes, subprocess calls, config contents. Implies `--verbose`. |

```bash
sudo pipower -v status
sudo pipower -d install
sudo pipower -d test-alarm --minutes 3 --halt
```

### Commands

| Command | Description |
|---------|-------------|
| `sudo pipower install` | Install systemd units, create config, check prerequisites |
| `sudo pipower uninstall` | Remove systemd units (config preserved) |
| `sudo pipower set --wake HH:MM --sleep HH:MM [--timezone TZ]` | Update configuration |
| `sudo pipower test-alarm --minutes N [--halt]` | Set test alarm; `--halt` triggers full test cycle and gates `enable` |
| `sudo pipower enable` | Enable scheduled cycle **(requires successful test first)** |
| `sudo pipower disable` | Disable scheduled shutdown (alarm guard stays active) |
| `sudo pipower status` | Show configuration, test status, timer, and RTC alarm state |
| `sudo pipower shutdown` | Set alarm then poweroff (use instead of `sudo shutdown -h now`) |
| `sudo pipower reset-test` | Clear tested flag to require re-testing |
| `pipower log [--lines N]` | Show persistent shutdown/boot log (no sudo needed) |

---

## Systemd Units Installed

Three units are installed to `/etc/systemd/system/`:

### `pipower-alarm.service`
Always active. Uses `ExecStop` to set the RTC wake alarm before **any** shutdown — whether triggered by the timer, `sudo pipower shutdown`, or a manual `sudo shutdown -h now`. This is the key safety net that ensures the alarm is always set.

### `pipower-shutdown.timer`
Fires at the configured sleep time using `OnCalendar` with full timezone support (systemd 248+). Only active when pipower is enabled.

### `pipower-shutdown.service`
Triggered by the timer. Calls `systemctl poweroff`, which triggers `pipower-alarm.service`'s ExecStop before the system halts.

---

## Config File

Located at `/etc/pipower/config.json`:

```json
{
  "_version": "V1.9",
  "wake_time": "06:30",
  "sleep_time": "22:00",
  "enabled": true,
  "timezone": "America/Toronto",
  "tested": true,
  "test_epoch": null
}
```

Edit with `sudo pipower set ...` — do not edit manually while enabled.

---

## Troubleshooting

### chrony not syncing

```bash
# Check service and journal
sudo systemctl status chrony
journalctl -u chrony -n 50

# Force a step correction if clock is badly out
sudo chronyc makestep

# Check sources — ^? means unreachable, ^x means falseticker
# ^- means fallback (healthy), ^* means selected (healthy)
chronyc sources -v

# Test NTS port for Cloudflare (if chronyc authdata shows no keys)
nc -zv time.cloudflare.com 4460
# If this fails, remove 'nts' from the cloudflare line in chrony.conf

# Check NTS auth status (requires sudo)
sudo chronyc authdata
```

### RTC not being updated by chrony

```bash
# Confirm rtcsync is in config
grep rtcsync /etc/chrony/chrony.conf

# Check kernel messages about RTC sync
dmesg | grep -i rtc

# Confirm RTC and system clock agree via timedatectl
timedatectl status
# RTC time should match Universal time
```

> **Note:** If you need `hwclock` directly, install it with `sudo apt install util-linux-extra`. It is not required for normal pipower operation.

### Clock wrong after long power-off

The `makestep 1.0 3` directive handles this by stepping the clock on the first 3 syncs after boot. If the RTC battery is depleted and the RTC shows epoch (1970), chrony will still correct the time once network is available. Force a one-off step correction of any size:

```bash
sudo chronyc makestep
```

### Wake alarm interface missing

```bash
dmesg | grep -i rtc
# Should show: rpi-rtc soc:rpi_rtc: registered as rtc0
```
If not present, the RTC driver may not have loaded. Check `/boot/firmware/config.txt` for conflicting overlays.

### Pi doesn't wake up
1. Confirm 5V external power is present during halt.
2. Check EEPROM: `rpi-eeprom-config | grep POWER_OFF_ON_HALT`
3. Test alarm immediately after setting: `cat /sys/class/rtc/rtc0/wakealarm` — should be non-zero before halt.
4. Run `sudo pipower test-alarm --minutes 3 --halt` to isolate the issue.

### Timer not firing at expected time
Check DST handling: `timedatectl` should show the correct local timezone. The OnCalendar entry uses your configured timezone directly, so DST transitions are handled automatically.

### Manual `sudo shutdown -h now` won't set alarm
Use `sudo pipower shutdown` instead. The `pipower-alarm.service ExecStop` *should* catch all shutdowns, but using the dedicated command is safer and provides confirmation output.

---

## Uninstalling

```bash
# Remove systemd units (config and script preserved)
sudo pipower uninstall

# Optionally remove config
sudo rm -rf /etc/pipower

# Optionally remove the script
sudo rm /usr/local/bin/pipower
```

---

## Known Limitations (V1.1)

- Daily schedule only (fixed wake and sleep times, same every day).
- No weekend/weekday differentiation.
- Requires external 5V to remain powered during halt.
- No notification mechanism if alarm set fails (check `journalctl -u pipower-alarm.service`).
- chrony requires network connectivity to maintain accuracy; accuracy degrades during extended offline periods (mitigated by the drift file and RTC battery).
