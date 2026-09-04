# voron-config

Configuration backup for a Voron 2.4 300, pushed automatically from the printer
by [klipper-backup](https://github.com/Staubgeborener/klipper-backup).

This is a **configuration-only** backup. It does not contain Klipper, Moonraker
or any of the plugins: install those first, or Klipper will not start. See
[Install order](#install-order) below.

## Layout

Everything lives under `printer_data/config/`, mirroring the printer. The split
is by *question answered*: `hardware/` says what the machine is, `macros/` says
what it does, `leds/` owns one subsystem end to end.

```
printer.cfg              includes, MCUs, kinematics, idle timeout, SAVE_CONFIG
├── hardware/            what the machine is made of, and on which pins
│   ├── steppers.cfg     XY + the four Z motors, and their TMC2209 drivers
│   ├── extruder.cfg     extruder motor, MAX31865/PT100 hotend, pressure advance
│   ├── bed.cfg          heater_bed and its thermistor
│   ├── probe.cfg        Cartographer MCU, the probe itself, and [bed_mesh]
│   ├── homing.cfg       safe_z_home and quad_gantry_level
│   ├── fans.cfg         hotend, part cooling, driver, bay and bed fans
│   ├── sensors.cfg      chamber and MCU thermistors, filament switches
│   ├── display.cfg      [display_status] only - the screen is not Klipper's
│   └── input_shaper.cfg accelerometer, resonance tester, shaper, Shake&Tune
├── leds/                the whole LED subsystem, ~400 lines
│   ├── hardware.cfg     the two neopixel chains
│   ├── effects.cfg      27 [led_effect] definitions
│   └── macros.cfg       LED_* manual control and STATUS_* print states
├── macros/              what the machine does
│   ├── variables.cfg    _MACHINE_VARS - shared geometry, read below
│   ├── print_control.cfg PAUSE / RESUME / CANCEL_PRINT / HOME / parking
│   ├── print_start_end.cfg PRINT_START and PRINT_END
│   ├── clean_nozzle.cfg _CLEAN_NOZZLE engine + the CLEAN_* wrappers
│   ├── filament.cfg     Spoolman, load/unload, M600, sensor enable
│   ├── bedfans.cfg      bed fan logic and the M190/M140 overrides
│   └── test_speed.cfg   TEST_SPEED
├── KAMP_Settings.cfg    third-party, stays at the root (see Install order)
├── update_git.cfg       UPDATE_GIT, the manual backup trigger
└── *.conf               Moonraker, crowsnest, mobileraker
```

### Conventions

**Naming.** Klipper object names are lowercase with underscores
(`bed_fans`, `spider_mcu`, `sb_leds`). Macros are UPPERCASE. A leading
underscore means internal — Mainsail hides those, and they are not meant to be
called by hand: `_MACHINE_VARS`, `_CLEAN_NOZZLE`, `_TOOLHEAD_PARK_PAUSE_CANCEL`,
`_BEDFANVARS`. Every macro carries a `description:`, which is what Mainsail
shows on the button.

**Shared geometry.** Coordinates that more than one macro has to agree on live
in `_MACHINE_VARS` ([macros/variables.cfg](printer_data/config/macros/variables.cfg)),
never in whichever macro happened to need them first:

```jinja
{% set mv = printer["gcode_macro _MACHINE_VARS"] %}
G0 X{mv.park_x} Y{mv.park_y} F6000
```

`PRINT_START`, `PRINT_END`, `_TOOLHEAD_PARK_PAUSE_CANCEL` and `_CLEAN_NOZZLE`
all read from it. Note that `park_*` and `bucket_*` hold the same coordinates
today but are deliberately separate entries: parking somewhere else should not
move the purge bucket.

**Engine plus wrappers.** `_CLEAN_NOZZLE` takes every parameter (wipe count,
purge length, load, retract, dwell, park) and does the work. The five public
`CLEAN_*` macros are one-line wrappers that call it with a preset. Add a new
cleaning behaviour by adding a wrapper, not by editing the engine.

**The two filament sensors do different things, on purpose.**
`extruder_entry` sits before the extruder gears, so when it trips the tail of
the filament is still gripped: it runs `M600`, which pauses and then unloads
that stub cleanly. `extruder_exit` sits after the gears, so by the time it
trips there is nothing left for them to grip and an unload would spin on air —
it only pauses, and you load new filament by hand. Both are disabled together
by `FILAMENT_SENSORS_ENABLE`, which `PAUSE` calls before anything else, so the
retraction inside `M600` cannot trip `extruder_exit` and cascade.

**Overrides.** [macros/bedfans.cfg](printer_data/config/macros/bedfans.cfg)
renames and replaces `M190`, `M140`, `SET_HEATER_TEMPERATURE` and
`TURN_OFF_HEATERS` so bed heating also drives the bed fans. If bed temperature
commands ever behave oddly, that file is the reason.

**Nothing carries over between prints.** `_RESET_PRINTER_TO_KLIPPER_CONFIG`
([macros/print_control.cfg](printer_data/config/macros/print_control.cfg))
puts the speed factor, flow factor, velocity limits, pressure advance, Z offset
and bed mesh back to what `printer.cfg` declares. `PRINT_START`, `PRINT_END`
and `CANCEL_PRINT` all call it, so a `M220 S150` typed into the web UI or a
`SET_VELOCITY_LIMIT` left behind by an interrupted `TEST_SPEED` cannot silently
follow you into the next job.

**Heatsoak by time, not by temperature.** The chamber here is passive — only
the bed and the bed fans heat it — so `PRINT_START` takes `SOAK=<minutes>`,
which always terminates. `CHAMBER=<degrees>` still exists and waits on the
chamber thermistor, but `TEMPERATURE_WAIT` cannot be interrupted: give it a
temperature this machine cannot reach and the print blocks forever with no way
out but cancelling. Use `CHAMBER=` only for a target you have actually
measured; otherwise use `SOAK=`. A typical ABS start line is:

```gcode
PRINT_START BED=105 EXTRUDER=255 SOAK=15
```

## Hardware

| Component | Details |
|---|---|
| Printer | Voron 2.4, 300 × 300 (usable Z 270) |
| Mainboard | Fysetc Spider 2.2, STM32F446, 12 MHz crystal, over plain USB |
| | `serial: /dev/serial/by-id/usb-Klipper_stm32f446xx_1E0027000F51303530323539-if00` |
| CAN bridge | BTT U2C V2.1, candleLight firmware, `can0` at 1 Mbit/s |
| | USB id `1d50:606f`. **Both CAN devices below hang off this, not off the Spider** |
| Toolhead | A4T, Crossbow X carriage, CNC XOL mount |
| | EBB36 over CAN — `canbus_uuid: a701c91bc20c` (dual filament sensor) |
| | Spare, currently unused: `95b4bab5914c` (EBB36 WWG2) |
| Hotend | Dragon Ace, **PT1000** via the EBB36's MAX31865 (2-wire) on `EBBCan:PA4`, SPI1 |
| Extruder | WWBMG Dual, 50:10, `rotation_distance: 22.6789511` |
| Probe | Cartographer V4 over CAN — `canbus_uuid: 623c5a948da6` |
| | Firmware `CARTOGRAPHER v4 6.2.0`, plugin `1.9.0` |
| Bed | Keenovo, `Generic 3950` on `PB0`, SSR on `PB4`, `max_power: 0.6` |
| Chamber | `Generic 3950` thermistor on `PC0` |
| Display | BTT PITFT43 V2.0 on the Pi (`/dev/fb0`), driven by KlipperScreen |
| Camera | Logitech C920 (`046d:082d`), focus locked in `crowsnest.conf` |
| Case LEDs | Neopixel GRB ×36 on `PD3` |
| Toolhead LEDs | Neopixel GRBW ×3 on `EBBCan:PD3` (1 = logo, 2-3 = nozzle) |
| Bed fans | `fan_generic bed_fans` on `PC8` |
| Filament sensors | `extruder_entry` on `EBBCan:PB6` (before the gears), `extruder_exit` on `EBBCan:PB5` (after them) |
| Host | Raspberry Pi 4 (`VoronPrinter`), user `pi`, Debian 11 bullseye arm64 |

When rebuilding the Spider firmware: in `menuconfig` enable *extra low-level
configuration setup*, select the 12 MHz crystal and the USB interface. Flash to
`0x08000000`.

The U2C is flashed separately with candleLight firmware; the source is kept in
`~/candleLight_fw` on the host. Klipper has no config section for it - it just
provides the `can0` interface that the EBB36 and the Cartographer talk over.

The hotend PT1000 is declared as `rtd_nominal_r: 1000` / `rtd_reference_r: 4300`.
Klipper only ever uses those two as a ratio, so 430/100 - what this config said
before - produced identical readings. The current values are simply the honest
ones for the sensor that is actually fitted; do not "fix" them back.

There is deliberately **no `[display]` section**. The PITFT43 is a Raspberry Pi
framebuffer, not a Klipper display, so `hardware/display.cfg` declares only
`[display_status]` - which is what provides `M117` and `SET_DISPLAY_TEXT`. See
the comment in that file before removing it.

## Printed mods

None of these change the config, with one exception noted below. They are listed
because a rebuild has to reprint them, and because the panel and door mods are
what fix the panel thicknesses.

| Mod | Author | What it is |
|---|---|---|
| [THE FILTER, for Voron 2.4](https://www.printables.com/model/334276-the-filter-for-voron-24) | nateb16 (Printables copy by AKinferno) | Activated-carbon filter and the bed fans in one housing, mounted at the front under the bed |
| [Clicky-Clacky Door](https://github.com/tanaes/whopping_Voron_mods/tree/main/clickyclacky_door) | whopping | One-piece front door on lift-off hinges, magnet latch, closes onto sealing foam |
| [BTT Pi TFT43 V2.0 Voron 2.4 touch screen case - clicky-clack mod](https://www.printables.com/model/1255057-btt-pi-tft43-v20-voron-24-touch-screen-case-clicky) | Ninefifty | Housing for the PITFT43 under the front 2020 rail, cut to clear the door above |
| [Voron 2.4 Filament Latch (or any 2020 extrusion)](https://www.printables.com/model/172368-voron-24-filament-latch-or-any-2020-extrusion) | Richard M | The panel clips. **3 mm** panels top and back, **5 mm** on the sides |
| [Voron Rollback Stands](https://www.printables.com/model/408015-voron-rollback-stands) | Ken226 | Replaces the rear lower corner and midspan clips; the printer tips onto its back to open the electronics bay |
| [VoronBFI — Beefy Front Idlers](https://github.com/clee/VoronBFI) | clee | Front idlers that turn belt tension into layer compression instead of pulling the printed layers apart |
| [Voron Disco/Daylight on a Stick LED Mount Tray With Snap-On Diffuser](https://www.printables.com/model/795364-voron-discodaylight-on-a-stick-led-mount-tray-with) | Krhom | Tray and snap-on diffuser on 2020 extrusion for the case LED sticks |
| [V2.4 handle](https://mods.vorondesign.com/details/xa84lhUN5aMX4nmfZquaQ) | 1-0-R | Carrying handles on the top extrusions, 6 mm clearance so the panel clamp still fits |

**THE FILTER is the exception.** It is the hardware behind `fan_generic bed_fans`
on `PC8` and behind all of
[macros/bedfans.cfg](printer_data/config/macros/bedfans.cfg): the same 5015 fans
that pull chamber air through the charcoal also blow across the underside of the
bed, which is why bed heating and the fans are tied together in the `M190`,
`M140`, `SET_HEATER_TEMPERATURE` and `TURN_OFF_HEATERS` overrides. It is also the
reason the passive chamber warms at all — see *Heatsoak by time, not by
temperature* above. Drop the mod and that file goes with it, not just the fan.

## Install order

Everything in the first list comes from [KIAUH](https://github.com/dw-0/kiauh):

1. Klipper, Moonraker, Mainsail, KlipperScreen, Crowsnest, Sonar

The rest are installed manually (or through the `update_manager` entries already
present in `moonraker.conf`):

2. [moonraker-timelapse](https://github.com/mainsail-crew/moonraker-timelapse) — **required**, creates `timelapse.cfg`
3. [KAMP](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) (`dev` branch) — **required**, creates the `KAMP/` symlink
4. [klipper-led_effect](https://github.com/julianschill/klipper-led_effect) — without it `leds/effects.cfg` will not load
5. [Klippain Shake&Tune](https://github.com/Frix-x/klippain-shaketune) — provides the `[shaketune]` section
6. [Cartographer plugin](https://docs.cartographer3d.com/cartographer-probe/installation-and-setup/software-configuration/klipper-setup) — `cartographer3d-plugin` into `~/klippy-env`
7. [Katapult](https://github.com/Arksine/katapult) — CAN bootloader
8. [mobileraker_companion](https://github.com/Clon1998/mobileraker_companion)
9. [klipper-backup](https://github.com/Staubgeborener/klipper-backup) — this backup
10. [Spoolman](https://github.com/Donkie/Spoolman) **standalone** (not Docker) in
    `~/Spoolman`, systemd service `Spoolman.service`, listening on 7912

Spoolman has a few gotchas worth knowing about:

- Its installer uses `uv`, which **downloads its own Python 3.10+**. The 3.9 that
  ships with bullseye does not satisfy Spoolman's `requires-python`, but there is
  no need to upgrade Debian or compile anything: `uv` handles it on aarch64.
- Spool data lives in `~/.local/share/spoolman` (the default path), together with
  the nightly automatic backups and the logs. The `.env` only carries
  `SPOOLMAN_HOST` and `SPOOLMAN_PORT` — `SPOOLMAN_DIR_DATA` was deliberately left
  out so that losing `.env` cannot start Spoolman against an empty database.
- The installer appends `Spoolman` to `~/printer_data/moonraker.asvc` without a
  preceding newline, gluing it onto the previous entry and invalidating both.
  Check that file after installing.

## Restoring

```bash
git clone https://github.com/pmelero/voron-config.git /tmp/restore
cp -r /tmp/restore/printer_data/config/* ~/printer_data/config/
sudo systemctl restart klipper
```

Afterwards, check the `SAVE_CONFIG` block at the end of `printer.cfg`: it carries
the bed and hotend PID values, the bed mesh, and the Cartographer scan and touch
models. If the hardware is unchanged they can be used as-is; if you swapped the
nozzle or the probe, recalibrate.

## What this backup does NOT contain

`klipper-backup` **skips symbolic links** by design (it prints
`Skipping symbolic link` as it runs). These two live in the config folder but are
not in the repo, and their own installers recreate them:

- `KAMP` → `~/Klipper-Adaptive-Meshing-Purging/Configuration`
- `timelapse.cfg` → `~/moonraker-timelapse/klipper_macro/timelapse.cfg`

That is why the `[include]` lines pointing at them use wildcards
(`timelapse*.cfg`, `Line_Purge*.cfg`). Klipper treats an include glob that
matches nothing as empty rather than as an error, so the printer **still boots**
when those plugins are not installed yet and you can repair it from the web UI.

Also excluded, via the `exclude` array in the `.env`: `*.db`, `*.stdata`,
`*.csv`, `*.zip`, `*.bak`, `*.bkp`, and the automatic `printer-<timestamp>.cfg`
copies.

### Shake&Tune results

The `.png` graphs under `printer_data/config/ShakeTune_results/` **are** backed
up. The `.stdata` raw captures sitting next to them are **not**: graphs average
~670 KB, raw captures ~4 MB, and the raw ones are only useful for reprocessing a
run or for reporting a bug upstream.

How many are kept is `number_of_results_to_keep` in the `[shaketune]` section of
`hardware/input_shaper.cfg`. It counts **PNG files per subfolder, not
calibration runs** — an input shaper run writes two graphs (X and Y), so the
current value of 200 keeps 100 of those, while `belts` writes one graph per run
and so keeps 200. Deleting a graph also deletes its matching `.stdata`. There is
no "unlimited" value: `0` wipes the folder rather than keeping everything, so
"keep them all" means choosing a number you will never reach. At ~670 KB per
graph, 200 per subfolder is roughly 134 MB of SD card per subfolder at full
saturation.

Note that retention does not bound the repository. Every graph that is ever
committed stays in git history regardless; `number_of_results_to_keep` only
controls how many remain visible in the current tree and on the SD card.

### Third-party configs: which are backed up and which are not

Two plugin config files are deliberately excluded, because nothing in them is
worth restoring:

- `sonar.conf` — byte-for-byte the installer default (only comment wording
  differs from `~/sonar/resources/sonar.conf`), for a service that is
  `enable: false` anyway.
- `KlipperScreen.conf` — 37 of its 38 lines are the auto-generated `#~#` block
  and none are hand-written. KlipperScreen rewrites it every time something is
  toggled on the touchscreen, which made it one of the noisiest files in the
  history. Restoring it just means re-hiding a few macros from the UI.

The other third-party configs **are** backed up, and should stay that way — they
carry real settings that are not obvious to reconstruct:

- `crowsnest.conf` — `v4l2ctl: focus_automatic_continuous=0,focus_absolute=75`
  (locked focus; the default has this commented out) and `max_fps: 30` instead
  of the default 15. Lose it and the webcam goes back to hunting focus mid-print.
- `mobileraker.conf` — timezone, `snapshot_rotation: 180`, and the printer name.
- `KAMP_Settings.cfg` — hand-edited (`probe_dock_enable: False`), and it carries
  the `Line_Purge*.cfg` wildcard include.

## How the backup works

Configured in `~/klipper-backup/.env`, which holds the GitHub token and is
**never** pushed.

- `backupPaths=("printer_data/config/*")` — only that folder is backed up.
- The repo's `.gitignore` is **regenerated on every run** from the `exclude`
  array in the `.env`. Editing it by hand achieves nothing; edit the `.env`.
- The working repo is `~/config_backup`, and its tree is emptied after each push.
  The source of truth is always `~/printer_data/config/`.
- **This file is the exception.** `README.md` lives only in `~/config_backup`,
  never under `printer_data/config/`. The backup script preserves it explicitly
  (`find ... ! -name 'README.md'`) and only recreates it when missing, so edit it
  there — a copy placed next to `printer.cfg` would not be this file.
  Because it sits outside `printer_data/config/`, filewatch does not see it:
  editing it triggers no backup of its own. It only reaches GitHub on the next
  backup that something else triggers, or when
  `bash ~/klipper-backup/script.sh` is run by hand.

Three triggers:

| Trigger | What it does |
|---|---|
| `klipper-backup-filewatch.service` | `inotifywait` on the config; backs up on every change, unless printing |
| `klipper-backup-on-boot.service` | One backup at boot |
| cron `0 */4 * * *` | Every 4 hours |

Manually: `UPDATE_GIT` from the Klipper console, or `bash ~/klipper-backup/script.sh`.
Avoid running it by hand right after saving a file — filewatch will already be
running its own backup and the two collide on `.git/index.lock`. Harmless, just
wait a few seconds and retry.

**Content flows one way only**: files go from `~/printer_data/config/` to GitHub
via `rsync`, never the other way. Editing the repo on GitHub or in a clone
changes nothing on the printer.
