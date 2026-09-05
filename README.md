# voron-config

Configuration backup for a Voron 2.4 300, pushed from the printer by
[klipper-backup](https://github.com/Staubgeborener/klipper-backup).

**Config only** — no Klipper, no Moonraker, no plugins. Install those first
([Install order](#install-order)) or Klipper will not start.

**Content flows one way**: printer → GitHub. Editing this repo changes nothing on
the machine.

Anything a config file already explains in its own comments is not repeated here.

## Layout

Everything mirrors `printer_data/config/`. The split is by question answered:
`hardware/` says what the machine is, `macros/` what it does, `leds/` owns one
subsystem end to end.

```
printer.cfg              includes, MCUs, kinematics, idle timeout, SAVE_CONFIG
├── hardware/
│   ├── steppers.cfg     XY + four Z motors and their TMC2209 drivers
│   ├── extruder.cfg     extruder motor, PT1000 via MAX31865, pressure advance
│   ├── bed.cfg          heater_bed and its thermistor
│   ├── probe.cfg        Cartographer MCU, the probe, and [bed_mesh]
│   ├── homing.cfg       safe_z_home and quad_gantry_level
│   ├── fans.cfg         hotend, part cooling, driver, bay and bed fans
│   ├── sensors.cfg      chamber and MCU thermistors, filament switches
│   ├── display.cfg      [display_status] only — the screen is not Klipper's
│   └── input_shaper.cfg accelerometer, resonance tester, shaper, Shake&Tune
├── leds/
│   ├── hardware.cfg     the two neopixel chains
│   ├── effects.cfg      the [led_effect] definitions
│   └── macros.cfg       LED_* manual control, STATUS_* print states
├── macros/
│   ├── variables.cfg    _MACHINE_VARS — shared geometry
│   ├── print_start_end.cfg  PRINT_START and PRINT_END
│   ├── print_control.cfg    PAUSE / RESUME / CANCEL_PRINT / HOME / parking
│   ├── clean_nozzle.cfg     _CLEAN_NOZZLE engine + CLEAN_* wrappers
│   ├── filament.cfg         Spoolman, load/unload, M600, sensor enable
│   ├── bedfans.cfg          bed fan logic and the M190/M140 overrides
│   └── test_speed.cfg       TEST_SPEED
├── KAMP_Settings.cfg    third-party, stays at the root
├── update_git.cfg       UPDATE_GIT, the manual backup trigger
└── *.conf               Moonraker, crowsnest, mobileraker
```

## Conventions

- Klipper object names are `lowercase_with_underscores`, macros are `UPPERCASE`,
  and a leading underscore means internal — Mainsail hides those. Macros carry a
  `description:`; it is what Mainsail shows on the button.
- Coordinates more than one macro must agree on live in `_MACHINE_VARS`
  ([macros/variables.cfg](printer_data/config/macros/variables.cfg)), never in
  whichever macro needed them first:
  `{% set mv = printer["gcode_macro _MACHINE_VARS"] %}`.
- `_CLEAN_NOZZLE` is the engine and takes every parameter; the `CLEAN_*` macros
  are one-line presets. Add a wrapper rather than editing the engine.
- **The two filament sensors differ on purpose.** `extruder_entry` sits before
  the gears, so the stub is still gripped when it trips: it runs `M600`.
  `extruder_exit` sits after them, where an unload would spin on air: it only
  pauses. Both check `print_stats` first, so neither can pause an idle printer.
- **Bed heating is overridden.** [macros/bedfans.cfg](printer_data/config/macros/bedfans.cfg)
  renames `M190`, `M140`, `SET_HEATER_TEMPERATURE` and `TURN_OFF_HEATERS` so the
  bed fans follow the bed. If bed temperature commands behave oddly, that is why.
- **Nothing carries between prints.** `_RESET_PRINTER_TO_KLIPPER_CONFIG` puts
  speed factor, flow, velocity limits, pressure advance, Z offset and mesh back
  to what `printer.cfg` declares. `PRINT_START`, `PRINT_END` and `CANCEL_PRINT`
  all call it.
- **Heatsoak by time, not temperature.** The chamber is passive and
  `TEMPERATURE_WAIT` cannot be interrupted, so `CHAMBER=<degrees>` can block a
  print forever. Use `SOAK=<minutes>` unless the target is one you have measured:

  ```gcode
  PRINT_START BED=105 EXTRUDER=255 SOAK=15
  ```

## Hardware

| Component | Details |
|---|---|
| Printer | Voron 2.4, 300 × 300 (usable Z 270) |
| Mainboard | Fysetc Spider 2.2, STM32F446, 12 MHz crystal, plain USB |
| | `serial: /dev/serial/by-id/usb-Klipper_stm32f446xx_1E0027000F51303530323539-if00` |
| CAN bridge | BTT U2C V2.1, candleLight firmware, `can0` at 1 Mbit/s, USB id `1d50:606f` |
| | **Both CAN devices below hang off this, not off the Spider** |
| Toolhead | A4T, Crossbow X carriage, CNC XOL mount |
| | EBB36 over CAN — `canbus_uuid: a701c91bc20c` (dual filament sensor) |
| | Spare, unused: `95b4bab5914c` (EBB36 WWG2) |
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
| Filament sensors | `extruder_entry` on `EBBCan:PB6`, `extruder_exit` on `EBBCan:PB5` |
| Host | Raspberry Pi 4 (`VoronPrinter`), user `pi`, Debian 11 bullseye arm64 |

Rebuilding the Spider firmware: in `menuconfig` enable *extra low-level
configuration setup*, select the 12 MHz crystal and the USB interface, flash to
`0x08000000`. The U2C is flashed separately with candleLight (source kept in
`~/candleLight_fw`); Klipper has no section for it, it only provides `can0`.

## Printed mods

Listed because a rebuild has to reprint them, and because the panel and door mods
are what fix the panel thicknesses.

| Mod | Author | What it is |
|---|---|---|
| [THE FILTER, for Voron 2.4](https://www.printables.com/model/334276-the-filter-for-voron-24) | nateb16 (copy by AKinferno) | Carbon filter and the bed fans in one housing, front, under the bed |
| [Clicky-Clacky Door](https://github.com/tanaes/whopping_Voron_mods/tree/main/clickyclacky_door) | whopping | One-piece front door, lift-off hinges, magnet latch, foam seal |
| [BTT Pi TFT43 V2.0 case — clicky-clack mod](https://www.printables.com/model/1255057-btt-pi-tft43-v20-voron-24-touch-screen-case-clicky) | Ninefifty | PITFT43 housing under the front 2020, cut to clear the door |
| [Voron 2.4 Filament Latch](https://www.printables.com/model/172368-voron-24-filament-latch-or-any-2020-extrusion) | Richard M | Panel clips. **3 mm** top and back, **5 mm** sides |
| [Voron Rollback Stands](https://www.printables.com/model/408015-voron-rollback-stands) | Ken226 | Rear lower corners; the printer tips back to open the electronics bay |
| [VoronBFI — Beefy Front Idlers](https://github.com/clee/VoronBFI) | clee | Front idlers that turn belt tension into layer compression |
| [Voron Disco LED Mount Tray](https://www.printables.com/model/795364-voron-discodaylight-on-a-stick-led-mount-tray-with) | Krhom | Tray and snap-on diffuser for the case LED sticks |
| [V2.4 handle](https://mods.vorondesign.com/details/xa84lhUN5aMX4nmfZquaQ) | 1-0-R | Carrying handles, 6 mm clearance for the panel clamp |

**THE FILTER is not cosmetic.** Its 5015 fans are `fan_generic bed_fans` on `PC8`
and the entire reason [macros/bedfans.cfg](printer_data/config/macros/bedfans.cfg)
exists — they blow across the underside of the bed, which is also the only thing
that warms the passive chamber. Drop the mod and that file goes with it.

## Install order

From [KIAUH](https://github.com/dw-0/kiauh): Klipper, Moonraker, Mainsail,
KlipperScreen, Crowsnest, Sonar.

Then, manually or through the `update_manager` entries already in
`moonraker.conf`:

| Plugin | Why it is needed |
|---|---|
| [moonraker-timelapse](https://github.com/mainsail-crew/moonraker-timelapse) | **Required** — creates `timelapse.cfg` |
| [KAMP](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) (`dev`) | **Required** — creates the `KAMP/` symlink |
| [klipper-led_effect](https://github.com/julianschill/klipper-led_effect) | `leds/effects.cfg` will not load without it |
| [Klippain Shake&Tune](https://github.com/Frix-x/klippain-shaketune) | provides `[shaketune]` |
| [Cartographer plugin](https://docs.cartographer3d.com/cartographer-probe/installation-and-setup/software-configuration/klipper-setup) | `cartographer3d-plugin` into `~/klippy-env` |
| [Katapult](https://github.com/Arksine/katapult) | CAN bootloader |
| [mobileraker_companion](https://github.com/Clon1998/mobileraker_companion) | phone app bridge |
| [klipper-backup](https://github.com/Staubgeborener/klipper-backup) | this backup |
| [Spoolman](https://github.com/Donkie/Spoolman) | **standalone**, not Docker, in `~/Spoolman`, `Spoolman.service` on 7912 |

Spoolman gotchas:

- Its installer uses `uv`, which **downloads its own Python 3.10+**. Bullseye's
  3.9 does not satisfy `requires-python`, but nothing needs upgrading or
  compiling — `uv` handles it on aarch64.
- Data lives in `~/.local/share/spoolman` with the nightly backups and logs. The
  `.env` carries only `SPOOLMAN_HOST` and `SPOOLMAN_PORT`; `SPOOLMAN_DIR_DATA`
  was left out deliberately, so losing `.env` cannot start Spoolman against an
  empty database.
- The installer appends `Spoolman` to `~/printer_data/moonraker.asvc` **without a
  newline**, gluing it to the previous entry and invalidating both. Check it.

## Restoring

```bash
git clone https://github.com/pmelero/voron-config.git /tmp/restore
cp -r /tmp/restore/printer_data/config/* ~/printer_data/config/
sudo systemctl restart klipper
```

Then check the `SAVE_CONFIG` block at the end of `printer.cfg`: it carries the
bed and hotend PID, the bed mesh and the Cartographer scan and touch models.
Unchanged hardware can use them as-is; a new nozzle or probe means recalibrating.

## What is not in this backup

`klipper-backup` skips symlinks by design. These two live in the config folder,
are not in the repo, and their own installers recreate them:

- `KAMP` → `~/Klipper-Adaptive-Meshing-Purging/Configuration`
- `timelapse.cfg` → `~/moonraker-timelapse/klipper_macro/timelapse.cfg`

That is why their `[include]` lines use wildcards (`timelapse*.cfg`,
`Line_Purge*.cfg`): Klipper treats a glob that matches nothing as empty rather
than as an error, so the printer **still boots** before those plugins are
reinstalled and you can repair it from the web UI.

Also excluded, via the `exclude` array in the `.env`: `*.db`, `*.stdata`, `*.csv`,
`*.zip`, `*.bak`, `*.bkp`, the automatic `printer-<timestamp>.cfg` copies, and two
plugin configs that are not worth restoring — `sonar.conf` (byte-for-byte the
installer default, for a service that is disabled) and `KlipperScreen.conf`
(37 of 38 lines auto-generated, rewritten on every touch of the screen).

Kept on purpose, because they carry settings that are not obvious to rebuild:
`crowsnest.conf` (locked webcam focus, 30 fps), `mobileraker.conf` (timezone,
snapshot rotation, printer name) and `KAMP_Settings.cfg` (hand-edited
`probe_dock_enable: False`, and it holds the `Line_Purge*.cfg` include).

**Shake&Tune**: the `.png` graphs are backed up, the ~4 MB `.stdata` captures are
not. Retention is `number_of_results_to_keep` in `hardware/input_shaper.cfg` —
read the comment there, it counts files rather than calibration runs, and `0`
wipes the folder instead of meaning "unlimited". Retention does not bound this
repo: every graph ever committed stays in git history regardless.

## How the backup works

Configured in `~/klipper-backup/.env`, which holds the GitHub token and is never
pushed. Only `printer_data/config/*` is backed up, and the repo's `.gitignore` is
**regenerated from the `.env` on every run** — editing it by hand achieves
nothing.

The working repo is `~/config_backup`, emptied after each push; the source of
truth is always `~/printer_data/config/`. **`README.md` is the exception**: it
lives only in `~/config_backup`, never under `printer_data/config/`. The script
preserves it and pulls before each run, so editing it in a clone and pushing
works — but it reaches GitHub from the printer only on a backup something else
triggers.

| Trigger | When |
|---|---|
| `klipper-backup-filewatch.service` | every config change, unless printing |
| `klipper-backup-on-boot.service` | at boot |
| cron `0 */4 * * *` | every 4 hours |
| `UPDATE_GIT`, or `bash ~/klipper-backup/script.sh` | by hand |

Avoid running it by hand right after saving a file: filewatch is already running
its own backup and the two collide on `.git/index.lock`. Harmless — wait a few
seconds and retry.
