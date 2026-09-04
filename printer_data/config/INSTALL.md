# Voron 2.4 300 — Restauración desde cero

Este repo es un backup **solo de configuración**. No incluye Klipper, Moonraker ni los
plugins: hay que instalarlos antes de restaurar, o Klipper no arrancará.

## Hardware

| Componente | Detalle |
|---|---|
| Impresora | Voron 2.4, 300 × 300 (Z útil 270) |
| Placa base | Fysetc Spider, STM32F446, cristal 12 MHz, USB |
| | `serial: /dev/serial/by-id/usb-Klipper_stm32f446xx_1E0027000F51303530323539-if00` |
| Toolhead | EBB36 sobre CAN — `canbus_uuid: a701c91bc20c` (WWBMG, doble sensor) |
| | Repuesto en desuso: `95b4bab5914c` (EBB36 WWG2) |
| Sonda | Cartographer V4 sobre CAN — `canbus_uuid: 623c5a948da6` |
| | Firmware `CARTOGRAPHER v4 6.2.0`, plugin `1.9.0` |
| Hotend | Termistor PT100 MAX31865 (2 hilos, 430 Ω ref) en `EBBCan:PA4`, SPI1 |
| Extrusor | 50:10, `rotation_distance: 22.6789511` |
| Cama | Keenovo, `Generic 3950` en `PB0`, SSR en `PB4`, `max_power: 0.6` |
| Cámara térmica | Termistor `Generic 3950` en `PC0` |
| Pantalla | mini12864 (uc1701) |
| LEDs cámara | Neopixel GRB ×36 en `PD3` |
| LEDs toolhead | Neopixel GRBW ×3 en `EBBCan:PD3` (1 = logo, 2-3 = boquilla) |
| Bed fans | `fan_generic BedFans` en `PC8` |
| Sensores filamento | `switch_sensor_1` en `EBBCan:PB6`, `switch_sensor_2` en `EBBCan:PB5` |
| Host | Raspberry Pi (`VoronPrinter`), usuario `pi` |

Al recompilar el firmware del Spider: `menuconfig` → activar *extra low-level
configuration setup*, cristal de 12 MHz, interfaz USB. Se flashea a `0x08000000`.

## Orden de instalación

Todo lo de la primera lista se instala con [KIAUH](https://github.com/dw-0/kiauh):

1. Klipper, Moonraker, Mainsail, KlipperScreen, Crowsnest, Sonar

Y estos a mano (o desde el `update_manager` de `moonraker.conf`, que ya los trae):

2. [moonraker-timelapse](https://github.com/mainsail-crew/moonraker-timelapse) — **imprescindible**, crea `timelapse.cfg`
3. [KAMP](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging) (rama `dev`) — **imprescindible**, crea el symlink `KAMP/`
4. [klipper-led_effect](https://github.com/julianschill/klipper-led_effect) — sin él no carga `config/led_effects.cfg`
5. [Klippain Shake&Tune](https://github.com/Frix-x/klippain-shaketune) — sección `[shaketune]`
6. Plugin Cartographer: `cartographer3d-plugin` en `~/klippy-env`
7. [Katapult](https://github.com/Arksine/katapult) — bootloader CAN
8. [mobileraker_companion](https://github.com/Clon1998/mobileraker_companion)
9. [klipper-backup](https://github.com/Staubgeborener/klipper-backup) — este backup
10. [Spoolman](https://github.com/Donkie/Spoolman) **standalone** (no Docker) en
    `~/Spoolman`, servicio systemd `Spoolman.service`, escuchando en el 7912

Sobre Spoolman, que tiene un par de trampas:

- Su instalador usa `uv`, que **se descarga su propio Python 3.10+**. El 3.9 de
  bullseye no cumple el `requires-python` de Spoolman, pero no hace falta ni
  actualizar Debian ni compilar nada: `uv` lo resuelve solo en aarch64.
- Los datos de bobinas viven en `~/spoolman/data` (heredado de la instalación
  anterior en Docker), apuntados con `SPOOLMAN_DIR_DATA` en `~/Spoolman/.env`.
  Ese `.env` está en `persistent_files` del `update_manager`: si se pierde,
  Spoolman arranca con una base de datos vacía.
- El instalador añade `Spoolman` a `~/printer_data/moonraker.asvc` sin salto de
  línea previo, y lo pega a la entrada anterior dejándola inválida. Revisar ese
  archivo después de instalar.

## Restaurar

```bash
git clone https://github.com/pmelero/voron-config.git /tmp/restore
cp -r /tmp/restore/printer_data/config/* ~/printer_data/config/
sudo systemctl restart klipper
```

Después, en Mainsail, revisa el bloque `SAVE_CONFIG` al final de `printer.cfg`:
trae los PID de cama y hotend, el `bed_mesh` y los modelos `scan`/`touch` de
Cartographer. Si el hardware es el mismo, sirven tal cual; si cambiaste boquilla o
sonda, recalíbralos.

## Lo que este backup NO contiene

`klipper-backup` **ignora los enlaces simbólicos** a propósito (dice
`Skipping symbolic link` al ejecutarse). Estos dos existen en la carpeta de config
pero no están en el repo, y los recrean sus instaladores:

- `KAMP` → `~/Klipper-Adaptive-Meshing-Purging/Configuration`
- `timelapse.cfg` → `~/moonraker-timelapse/klipper_macro/timelapse.cfg`

Por eso los `[include]` que apuntan a ellos usan comodín (`timelapse*.cfg`,
`Line_Purge*.cfg`): Klipper trata un include con comodín sin coincidencias como
vacío en vez de como error, así que la impresora **arranca igualmente** aunque los
plugins todavía no estén instalados, y puedes repararla desde la interfaz.

Tampoco se respaldan, por estar en `exclude` del `.env`:
`ShakeTune_results/`, `*.db`, `*.stdata`, `*.png`, `*.csv`, `*.zip`, `*.bak`,
`*.bkp`, y las copias automáticas `printer-<fecha>.cfg`.

## Cómo funciona el backup

Definido en `~/klipper-backup/.env` (contiene el token de GitHub, **nunca se sube**).

- `backupPaths=("printer_data/config/*")` — solo se respalda esa carpeta
- El `.gitignore` del repo **se regenera** en cada ejecución desde el array `exclude`
  del `.env`. Editarlo a mano no sirve de nada: hay que tocar el `.env`.
- El repo de trabajo es `~/config_backup`, y su árbol se vacía después de cada push.
  La fuente de la verdad es siempre `~/printer_data/config/`.

Tres disparadores:

| Disparador | Qué es |
|---|---|
| `klipper-backup-filewatch.service` | `inotifywait` sobre la config; respalda en cada cambio (salvo imprimiendo) |
| `klipper-backup-on-boot.service` | Un backup al arrancar |
| cron `0 */4 * * *` | Cada 4 horas |

Manual: `UPDATE_GIT` desde la consola de Klipper, o `bash ~/klipper-backup/script.sh`.

**El flujo es unidireccional en cuanto a contenido**: los archivos van de
`~/printer_data/config/` a GitHub por `rsync`, nunca al revés. Editar el repo en
GitHub o en un clon no cambia nada en la impresora.
