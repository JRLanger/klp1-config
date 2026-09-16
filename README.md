# KLP1 — Kingroon KLP1 Klipper Config

Backup of the Klipper configuration for a Kingroon KLP1.

## Printer

| Item | Value |
|------|-------|
| Printer | Kingroon KLP1 |
| Firmware | Klipper |
| Web UI | Mainsail |
| OS image | MKS Armbian |
| SSH user | `mks` |
| Config path | `/home/mks/printer_data/config` |

## Macros

Documented as they currently exist in [`macros.cfg`](macros.cfg). (Behavior has **not** been rewritten yet — this is the original machine state.)

| Macro | Parameters (default) | Description |
|-------|----------------------|-------------|
| `START_PRINT` | `BED_TEMP` (60), `EXTRUDER_TEMP` (220) | Heat bed, home if needed, heat hotend, draw purge line, start print. |
| `END_PRINT` | — | Retract if hot, wipe, raise Z, heaters + fan off, move to center-back (X107 Y200 Z200), disable steppers. |
| `PAUSE` | `Z` (10), `E` (1) | Save state, Z-hop, retract `E`, park front (X10 Y10), hotend off, idle timeout 12h. Single-stage. |
| `RESUME` | `E` (2.5) | Reheat to saved temp, restore park position, prime `E` + lower Z, resume. |
| `CANCEL_PRINT` | — | Heaters off, cancel base, raise Z 10mm, move X10, disable motors, fan off. |
| `G29` | — | Home if needed, move to X105 Y105, run `PROBE_CALIBRATE` (Z offset / probe calibration). |
| `G30` | — | Home if needed, clear + run `BED_MESH_CALIBRATE`, save/load bed mesh profile `JRLanger`. |
| `G40` | — | Query accelerometer, reset input shaper, home if needed, fan full, `SHAPER_CALIBRATE`, `SAVE_CONFIG`. |
| `SHAPER_CALIBRATE` | — | Wraps `RESHAPER_CALIBRATE` with `FREQ_START=5 FREQ_END=100`. |
| `BED_TRAM` | `BED_TEMP` (optional) | Home, heat bed if `BED_TEMP` given, `SCREWS_TILT_CALCULATE`, drop bed for screw access. |
| `LOAD_FILAMENT` | — | Heat to 220°C, extrude 100mm slowly. |
| `UNLOAD_FILAMENT` | — | Heat to 220°C, purge/retract, cool to 62°C, final 50mm retract, motors off. |
| `DISPLAY_MESSAGE` | `MESSAGE` | Print `MESSAGE` to the console; helper for other macros. |

## PAUSE / RESUME flow

Current behavior (single-stage):

- **PAUSE** — saves gcode state, Z-hops by `Z` (default 10mm, capped at Z max), retracts `E` (default 1mm), parks the toolhead at the **front** (X10 Y10), turns the hotend off, and extends idle timeout to 12h.
- **RESUME** — reheats the hotend to the saved target, restores the parked position, primes `E` (default 2.5mm) while lowering Z back down, then continues the print.

> Note: this is not the two-stage bed-drop / back-park / beep / standby flow — that rewrite has not been done.

## Slicer G-code

**Start G-code:**

```
START_PRINT BED_TEMP=[bed_temperature_initial_layer_single] EXTRUDER_TEMP=[nozzle_temperature_initial_layer]
```

**End G-code:**

```
END_PRINT
```

## Calibration order

Using the macro names as they exist today:

1. `BED_TRAM` — tram the bed with the screw-tilt probe (optionally `BED_TRAM BED_TEMP=60`).
2. `G30` — bed mesh calibrate (saves/loads profile `JRLanger`).
3. `SAVE_CONFIG`
4. `G29` — Z offset / probe calibrate.
5. `SAVE_CONFIG`
6. `G40` — input shaper calibrate (runs `SAVE_CONFIG` internally).
7. `SAVE_CONFIG` (if not already saved by `G40`).

## How to restore

1. Copy these files back into `/home/mks/printer_data/config` on the printer (SSH user `mks`).
2. Re-create any excluded secrets file (e.g. `moonraker-obico.cfg` with the Obico `auth_token`).
3. In Mainsail, click **Save & Restart**.

## Change log

- **2026-09-16** — Initial backup of the current machine state. `macros.cfg` reviewed and documented as-is (no behavior changes). Obico `auth_token` excluded from the repo via `.gitignore`.
