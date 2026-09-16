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

Note: `mainsail.cfg` on the printer is a **symlink** to `~/mainsail-config/client.cfg` (read-only stock Mainsail macros). It defines `PAUSE`/`RESUME`/`CANCEL_PRINT`, but `printer.cfg` includes `macros.cfg` **after** it, so the custom versions in `macros.cfg` override the stock ones.

## Macros

Defined in [`macros.cfg`](macros.cfg). Macros starting with `_` are internal helpers.

| Macro | Parameters (default) | Description |
|-------|----------------------|-------------|
| `START_PRINT` | `BED_TEMP` (60), `EXTRUDER_TEMP` (220), `MESH` ("JRLanger") | Preheat nozzle to 150°C, heat bed, home, load mesh profile, heat nozzle, draw purge line. Warns if slicer passes no temps. |
| `END_PRINT` | — | Heaters + fan off, small retract, raise Z, park center-back, disable steppers. |
| `CANCEL_PRINT` | — | Heaters + fan off, reset pause timers, restore idle timeout, cancel, retract, raise Z, park back-left, motors off. |
| `PAUSE` | `Z` (z_lift, default 50) | Retract, drop bed, park at back, cool nozzle to standby, optional beep, idle timeout 12h. |
| `RESUME` | — | **Two-stage.** 1st call: reheat + purge (then clean nozzle). 2nd call: restore position, prime, continue print. |
| `_PAUSE_CFG` | (variables) | Settings holder for PAUSE/RESUME: park pos, z_lift, retract, standby_drop, heater_off_after, purge, beep. Edit values here. |
| `_PAUSE_PURGE` | `LENGTH` (30, max 45) | Manual purge at the purge position while paused. |
| `CALIBRATE_Z_OFFSET` | `BED_TEMP` (optional) | Home, move to bed center, run `PROBE_CALIBRATE`. Adjust with `TESTZ`, `ACCEPT`, then `SAVE_CONFIG`. |
| `CALIBRATE_MESH` | `BED_TEMP` (60), `SOAK` (0 min), `PROFILE` ("JRLanger") | Heat bed, optional soak, home, probe mesh, save profile. |
| `CALIBRATE_SHAPER` | — | Query accelerometer, home, `SHAPER_CALIBRATE`. Review results, then `SAVE_CONFIG` manually. |
| `BED_TRAM` | `BED_TEMP` (optional) | Home, heat bed if given, `SCREWS_TILT_CALCULATE`, drop bed for screw access. |
| `LOAD_FILAMENT` | `TEMP` (220), `LENGTH` (50) | Heat, load filament (split into safe chunks). |
| `UNLOAD_FILAMENT` | `TEMP` (220) | Heat, soften tip, staged retract, disable extruder stepper. |
| `DISPLAY_MESSAGE` | `MESSAGE` | Print `MESSAGE` to the console; helper for other macros. |

> Pause beeps are enabled: `[output_pin beeper]` (PC5) runs in `pwm: True` mode and `macros.cfg` defines `M300 S<freq> P<ms>`. The pause alert is 3 × 1000ms beeps at 1kHz.

## PAUSE / RESUME flow

- **PAUSE** — retracts, drops the bed (Z lift, clamped below Z max), parks at the back, cools the nozzle to standby (print temp minus `standby_drop`), beeps (if enabled), and extends idle timeout to 12h. Hotend fully off after `heater_off_after` seconds.
- **RESUME (1st press)** — reheats the nozzle and purges at the purge position. Clean the nozzle, then press RESUME again. `_PAUSE_PURGE` can be run for more purge.
- **RESUME (2nd press)** — restores idle timeout, returns to the parked height and print position, primes, and continues the print.

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

1. `BED_TRAM` — tram the bed (optionally `BED_TRAM BED_TEMP=60`).
2. `CALIBRATE_MESH BED_TEMP=60`
3. `SAVE_CONFIG`
4. `CALIBRATE_Z_OFFSET`
5. `SAVE_CONFIG`
6. `CALIBRATE_SHAPER`
7. `SAVE_CONFIG`

## How to restore

1. Copy these files back into `/home/mks/printer_data/config` on the printer (SSH user `mks`).
2. Re-create any excluded secrets file (e.g. `moonraker-obico.cfg` with the Obico `auth_token`).
3. `mainsail.cfg` is a symlink on the printer — do not overwrite the symlink; the tracked copy is only a content snapshot.
4. In Mainsail, click **Save & Restart** (or `FIRMWARE_RESTART`).

## Change log

- **2026-09-16** — Enabled pause beeps: `[output_pin beeper]` set to `pwm: True`, added `M300` macro, pause alert = 3 × 1000ms beeps. Verified live.
- **2026-09-16** — `macros.cfg` rewritten and deployed to the printer: two-stage PAUSE/RESUME (retract, bed-drop, back-park, standby cooling, beep; resume reheat/purge then continue), safe `END_PRINT`/`CANCEL_PRINT`, renamed `G29`/`G30`/`G40` to `CALIBRATE_Z_OFFSET`/`CALIBRATE_MESH`/`CALIBRATE_SHAPER`. Verified live (`FIRMWARE_RESTART` → ready). `mainsail.cfg` refreshed to its true symlink-target content.
- **2026-09-16** — Initial backup of current machine state. Obico `auth_token` excluded via `.gitignore`.
