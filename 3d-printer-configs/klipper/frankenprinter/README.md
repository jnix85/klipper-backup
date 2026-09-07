# Frankenprinter

Ender 3 V3 Neo chassis, heavily modified. Klipper on MainsailOS.

| | |
|---|---|
| Board | BIGTREETECH SKR Mini E3 V3.0 (STM32G0B1) |
| Extruder | Creality Sprite, direct drive |
| Probe | CR Touch |
| Z | Dual motors, parallel on the single Z driver |
| Host | MainsailOS on Raspberry Pi |
| Display | KlipperScreen (stock 4.3" DWIN panel is unusable under Klipper) |

## Firmware

Built on the Pi against the running Klipper version. `make menuconfig`:

| Option | Value |
|---|---|
| Enable extra low-level options | yes |
| Micro-controller architecture | STMicroelectronics STM32 |
| Processor model | STM32G0B1 |
| Bootloader offset | 8KiB bootloader |
| Clock Reference | 8 MHz crystal |
| Communication interface | USB (on PA11/PA12) |

```sh
cd ~/klipper && make clean && make
```

Flash by copying `out/klipper.bin` to a microSD as `firmware.bin` — FAT32,
4096-byte allocation unit, 32 GB or smaller — then power-cycle the printer
with the card inserted. Success is the file coming back renamed
`FIRMWARE.CUR`. Rebuild and re-flash after every `git pull` of Klipper; host
and MCU versions must match.

## Values that must be measured on this machine

`printer.cfg` marks these with `<<`. They are the ones that cannot be copied
from anyone else's config:

- **`[mcu] serial`** — from `ls /dev/serial/by-id/*`.
- **`[bltouch] x_offset` / `y_offset`** — depends on the Sprite mount, not on
  the printer model. Jog the nozzle over a mark on the bed and note the
  coordinates, then jog until the probe pin is over the same mark. The
  difference is the offset. Left of nozzle is negative X, toward front is
  negative Y.
- **`[safe_z_home] home_xy_position`** and **`[bed_mesh]` bounds** — derived
  arithmetic from the offsets above. Redo both after measuring.
- **`[extruder] sensor_type`** — cold printer should read room temperature
  within a couple of degrees. If not, try `EPCOS 100K B57560G104F`.
- **Motor directions** — home X and Y alone first; add or remove `!` on
  `dir_pin` for any axis that runs away from its endstop.

## Commissioning order

Smoke test before the first `G28`. Each step catches a failure that would
otherwise become a crash into the bed.

1. Klipper connects. Errors land in `~/printer_data/logs/klippy.log`.
2. Both temperatures read ambient, cold.
3. `QUERY_ENDSTOPS` untouched, then again while pressing X and Y by hand —
   states must flip.
4. `BLTOUCH_DEBUG COMMAND=pin_down` / `pin_up` — pin extends and retracts.
5. `QUERY_PROBE` with the pin down, push it up by hand, query again.
6. `G28 X` then `G28 Y` only.
7. `G28`, hand on the power switch.

Then calibrate, in this order:

1. `SCREWS_TILT_CALCULATE` — mechanical level first; the mesh corrects
   residual error, not a tilted bed.
2. `PID_CALIBRATE HEATER=extruder TARGET=220` (part fan off)
3. `PID_CALIBRATE HEATER=heater_bed TARGET=60`
4. `PROBE_CALIBRATE` → paper test → `SAVE_CONFIG`
5. `BED_MESH_CALIBRATE` at printing temperature
6. Extruder `rotation_distance` — mark 120 mm of filament, extrude 100 mm at
   5 mm/s, measure the remainder. `new = old * (actual / 100)`.
7. Calibration cube for X/Y scale
8. `TEST_RESONANCES` for input shaper
9. `TUNING_TOWER` for pressure advance

## Pi setup

This repo is the source of truth. On the Pi, `~/printer_data/config` is a
symlink into a checkout of it, so everything Klipper and Moonraker write —
including the `SAVE_CONFIG` block with probe `z_offset`, PID values and the
bed mesh — lands in a working tree instead of being invisible.

```sh
# on the Pi, as the printer user
git clone git@github.com:jnix85/3d-printer-configs.git ~/3d-printer-configs
~/3d-printer-configs/scripts/setup-pi.sh --dry-run   # review first
~/3d-printer-configs/scripts/setup-pi.sh
```

The script backs up any existing config directory rather than deleting it,
and carries across MainsailOS-generated files the repo does not track
(`mainsail.cfg`, `moonraker.conf`, `KlipperScreen.conf`) so
`[include mainsail.cfg]` keeps resolving.

### Day-to-day

Two machines write to this repo, so the rule is *pull before you edit*, on
both sides.

**On the Pi, after calibrating.** `SAVE_CONFIG` restarts Klipper and writes
its block to the end of `printer.cfg`. Capture it:

```sh
~/3d-printer-configs/scripts/save-calibration.sh --dry-run   # review
~/3d-printer-configs/scripts/save-calibration.sh
```

It pulls with `--rebase --autostash` first, derives a commit message from
what actually changed (`z_offset`, PID, bed mesh, `rotation_distance`), then
commits and pushes.

**On the workstation, before editing.** `git pull` first, or you will be
editing a `printer.cfg` that no longer matches the machine.

**Apply workstation edits to the printer:**

```sh
cd ~/3d-printer-configs && git pull && sudo systemctl restart klipper
```

### If printer.cfg conflicts

Hand edits live at the top of the file; Klipper's generated values live in
the `#*# <---------------------- SAVE_CONFIG ---------------------->` block
at the bottom. A conflict almost always means the two ends moved
independently — resolve by keeping **both** halves.

Never hand-edit inside the `SAVE_CONFIG` block. Klipper rewrites it whole,
so edits there are silently discarded on the next save. To change a value in
it, either delete the whole block and recalibrate, or set the value with the
matching command (`PROBE_CALIBRATE`, `PID_CALIBRATE`, `BED_MESH_CALIBRATE`).

## Notes on this build

**Dual Z runs on one driver.** The board has exactly four drivers, so both Z
motors share the Z output through a splitter. Klipper sees one Z axis and the
config is correct as written — but there is no `z_tilt` or independent gantry
levelling without adding a driver on the expansion header. Raised
`run_current` to 0.8 to compensate for the shared current; check driver heat
after the first long print.

**CR Touch, not BLTouch.** Configured under `[bltouch]` because that is the
only section Klipper offers for this protocol. `sensor_pin: ^PC14` — the
caret matters.

## Board pinout

| Function | Step | Dir | Enable | Endstop | UART addr |
|---|---|---|---|---|---|
| Stepper X | PB13 | PB12 | PB14 | PC0 | 0 |
| Stepper Y | PB10 | PB2 | PB11 | PC1 | 2 |
| Stepper Z | PB0 | PC5 | PB1 | PC2 | 1 |
| Extruder | PB3 | PB4 | PD1 | — | 3 |

| Peripheral | Pin |
|---|---|
| Hotend heater / thermistor | PC8 / PA0 |
| Bed heater / thermistor | PC9 / PC4 |
| Part cooling fan (FAN0) | PC6 |
| Heatbreak fan (FAN1) | PC7 |
| Controller fan (FAN2) | PB15 |
| Probe signal / servo | ^PC14 / PA1 |
| Filament runout | PC15 |
| NeoPixel | PA8 |
| TMC UART / TX | PC11 / PC10 |
