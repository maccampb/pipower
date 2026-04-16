# pipower V2.3

Raspberry Pi 5 RTC scheduled power management — ckmmconsulting

Setup daily wake/sleep power on/off cycles  using the Pi 5 built-in PCF85063A RTC.

- Originally designed to wakeup and sleep a MagicMirror running on a Raspberry Pi 5

- RTC battery is not required.

- Allows setting one wake and one sleep time per day. 

- Operates from command line interface

- Provides a controlled alternative to shutdown command to ensure wakeup is set

- survives reboots

There is a companion MMM-pipower module that shows the status of the power management

## Files
- `pipower` — CLI tool (install to `/usr/local/bin/pipower`)
- `pipower_setup_guide_V2.3_ckmmconsulting.md` — Installation and usage guide
- `pipower_architecture_V2.3_ckmmconsulting.md` — Architecture and system overview

## Quick start
See the setup guide for full instructions.
