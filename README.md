# voron-config

Configuration backup for a Voron 2.4 300, pushed automatically from the printer
by [klipper-backup](https://github.com/Staubgeborener/klipper-backup).

This is a **configuration-only** backup. It does not contain Klipper, Moonraker
or any of the plugins: install those first, or Klipper will not start. See
[Install order](#install-order) below.

## Layout

Everything lives under `printer_data/config/`, mirroring the printer:

| Path | Contents |
|---|---|
| `printer.cfg` | Header, includes, MCU definitions, kinematics, idle timeout |
| `hardware/` | What the machine is made of, and on which pins |
| `leds/` | The two neopixel chains, their effects, and the `STATUS_*` macros |
| `macros/` | What the machine does |
| `*.conf` | Moonraker and the third-party services |

## Hardware

| Component | Details |
|---|---|
| Printer | Voron 2.4, 300 × 300 (usable Z 270) |
| Mainboard | Fysetc Spider, STM32F446, 12 MHz crystal, USB |
| | `serial: /dev/serial/by-id/usb-Klipper_stm32f446xx_1E0027000F51303530323539-if00` |
| Toolhead | EBB36 over CAN — `canbus_uuid: a701c91bc20c` (WWBMG, dual sensor) |
| | Spare, currently unused: `95b4bab5914c` (EBB36 WWG2) |
| Probe | Cartographer V4 over CAN — `canbus_uuid: 623c5a948da6` |
| | Firmware `CARTOGRAPHER v4 6.2.0`, plugin `1.9.0` |
| Hotend | PT100 via MAX31865 (2-wire, 430 Ω ref) on `EBBCan:PA4`, SPI1 |
| Extruder | 50:10, `rotation_distance: 22.6789511` |
| Bed | Keenovo, `Generic 3950` on `PB0`, SSR on `PB4`, `max_power: 0.6` |
| Chamber | `Generic 3950` thermistor on `PC0` |
| Display | mini12864 (uc1701) |
| Case LEDs | Neopixel GRB ×36 on `PD3` |
| Toolhead LEDs | Neopixel GRBW ×3 on `EBBCan:PD3` (1 = logo, 2-3 = nozzle) |
| Bed fans | `fan_generic BedFans` on `PC8` |
| Filament sensors | `switch_sensor_1` on `EBBCan:PB6`, `switch_sensor_2` on `EBBCan:PB5` |
| Host | Raspberry Pi 4 (`VoronPrinter`), user `pi`, Debian 11 bullseye arm64 |

When rebuilding the Spider firmware: in `menuconfig` enable *extra low-level
configuration setup*, select the 12 MHz crystal and the USB interface. Flash to
`0x08000000`.

## Install order

Everything in the first list comes from [KIAUH](https://github.com/dw-0/kiauh):

1. Klipper, Moonraker, Mainsail, KlipperScreen, Crowsnest, Sonar

The rest are installed manually (or through the `update_manager` entries already
present in `moonraker.conf`):

2. [moonraker-timelapse](https://github.com/mainsail-crew/moonraker-timelapse) — **required**, creates `timelapse.cfg`
3. [KAMP](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) (`dev` branch) — **required**, creates the `KAMP/` symlink
4. [klipper-led_effect](https://github.com/julianschill/klipper-led_effect) — without it `leds/effects.cfg` will not load
5. [Klippain Shake&Tune](https://github.com/Frix-x/klippain-shaketune) — provides the `[shaketune]` section
6. Cartographer plugin: `cartographer3d-plugin` into `~/klippy-env`
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

Also excluded, via the `exclude` array in the `.env`: `ShakeTune_results/`,
`*.db`, `*.stdata`, `*.png`, `*.csv`, `*.zip`, `*.bak`, `*.bkp`, and the
automatic `printer-<timestamp>.cfg` copies.

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
