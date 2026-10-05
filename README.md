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
| `START_PRINT` | `BED_TEMP` (60), `EXTRUDER_TEMP` (220), `MATERIAL` (PLA), `ADAPTIVE` (1), `MESH` ("JRLanger") | Preheat nozzle to 150°C, heat bed, home, mesh, heat nozzle, purge. `ADAPTIVE=1` probes only the area the parts cover; `ADAPTIVE=0` loads the saved `MESH` profile. |
| `END_PRINT` | — | Heaters + fan off, small retract, raise Z, park center-back, disable steppers. |
| `CANCEL_PRINT` | — | Heaters + fan off, reset pause timers, restore idle timeout, cancel, retract, raise Z, park back-left, motors off. |
| `PAUSE` | `Z` (z_lift, default 50), `RETURN` (exact) | Retract, drop bed, park at back, cool nozzle to standby, beep, idle timeout 12h. `RETURN=next` = on resume, hand back to the file at the corner instead of returning to the pause spot (for slicer pauses; the file travels to its next start). |
| `RESUME` | — | **Two-stage.** 1st call: reheat + purge (then clean nozzle). 2nd call: restore position, prime, continue print. |
| `_PAUSE_CFG` | (variables) | Settings holder for PAUSE/RESUME: park pos, z_lift, retract, standby_drop, heater_off_after, purge, beep. Edit values here. |
| `_PAUSE_PURGE` | `LENGTH` (30, max 45) | Manual purge at the purge position while paused. |
| `CALIBRATE_Z_OFFSET` | `BED_TEMP` (optional) | Home, move to bed center, run `PROBE_CALIBRATE`. Adjust with `TESTZ`, `ACCEPT`, then `SAVE_CONFIG`. |
| `CALIBRATE_MESH` | `BED_TEMP` (60), `SOAK` (0 min), `PROFILE` ("JRLanger") | Heat bed, optional soak, home, probe mesh, save profile. |
| `_ADAPTIVE_MESH` | `MARGIN` (10) | Probes the print area (from the file's object outlines) plus a margin, at full-mesh density (3x3 to 5x5). Full bed if the file has no object data. Used by `START_PRINT`. |
| `CALIBRATE_SHAPER` | — | Query accelerometer, home, `SHAPER_CALIBRATE`. Review results, then `SAVE_CONFIG` manually. |
| `BED_TRAM` | `BED_TEMP` (optional) | Home, heat bed if given, `SCREWS_TILT_CALCULATE`, drop bed for screw access. |
| `M600` | `MATERIAL` (PLA) | Filament change: pause, eject, then load new + two-stage resume (see below). |
| `LOAD_FILAMENT` | `MATERIAL` (PLA), `TEMP`, `LENGTH` (50) | Heat to material temp, load filament (split into safe chunks). |
| `UNLOAD_FILAMENT` | `MATERIAL` (PLA), `TEMP`, `PULL_TEMP`, `LENGTH` (100) | Cold pull: melts a fresh tip, snaps it 27 mm up into the cold zone, cools the nozzle to `PULL_TEMP` with the part fan (about 1 min), then pulls slowly. A hot pull jams the gears (see change log 2026-10-04). While paused, the nozzle returns to standby afterwards and the hotend-off timer restarts. Pulls `LENGTH` mm total. |
| `_MAT` | (variable) | Per-material temp table (`hot`/`pull`). Edit temps here in one place. |

Material presets (edit in `_MAT`):

| MATERIAL | hot (load/unload) | pull (cold pull) |
|----------|-------------------|------------------|
| PLA | 220 | 90 |
| PETG | 240 | 110 |
| ABS | 250 | 120 |

Usage: `LOAD_FILAMENT MATERIAL=PETG`, `UNLOAD_FILAMENT MATERIAL=ABS`, `M600 MATERIAL=PETG`. Override per call with `TEMP=`/`PULL_TEMP=`. Unknown/no material falls back to PLA.
| `TEST_MOTION` | `SPEED` (100), `ACCEL`, `LOOPS` (1), `MARGIN` (15), `CIRCLES` (3), `Z` (1), `Z_SPEED` (15) | Motion test, no heating: center, four corners, circles, square, X pattern, then bed down/up. Blocked during a print. |
| `DISPLAY_MESSAGE` | `MESSAGE` | Print `MESSAGE` to the console; helper for other macros. |

> Pause beeps are enabled: `[output_pin beeper]` (PC5) runs in `pwm: True` mode and `macros.cfg` defines `M300 S<freq> P<ms>`. The pause alert is 3 × 1000ms beeps at 1kHz.

## PAUSE / RESUME flow

- **PAUSE** — retracts, drops the bed (Z lift, clamped below Z max), parks at the back, cools the nozzle to standby (print temp minus `standby_drop`), beeps (if enabled), and extends idle timeout to 12h. Hotend fully off after `heater_off_after` seconds.
- **RESUME (1st press)** — reheats the nozzle and purges at the purge position. Clean the nozzle, then press RESUME again. `_PAUSE_PURGE` can be run for more purge.
- **RESUME (2nd press)** — restores idle timeout, returns to the parked height and print position, primes, and continues the print.

## Filament change (M600)

`M600` (sent by the slicer at a filament/color change, or run manually). Fluidd-native flow — no popup:

1. `M600` → pauses, parks, ejects old filament with a cold pull (about 1¾ min: the nozzle cools to the pull temperature, then returns to standby). Wait for the message before you pull the filament out.
2. Insert new filament → click the **`LOAD_FILAMENT`** macro button.
3. Press **Resume** → reheats + purges. Clean the nozzle. (`_PAUSE_PURGE` for more purge.)
4. Press **Resume** again → continues the print.

> `printer.cfg` has a `[respond]` section (added for a guided popup). Fluidd doesn't render `action:prompt` dialogs, so `M600` uses the button flow above. The popup would work in Mainsail if you switch UIs; `[respond]` is left in place for that.

## Slicer G-code

**Start G-code:**

```
START_PRINT BED_TEMP=[bed_temperature_initial_layer_single] EXTRUDER_TEMP=[nozzle_temperature_initial_layer] MATERIAL=[filament_type]
```

`MATERIAL` is stashed by `START_PRINT`, so a bare slicer-inserted `M600` (color change) auto-uses the print's material for the cold pull. Override any time with `M600 MATERIAL=ABS`.

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

## Slicer

OrcaSlicer presets are backed up in [`orca/`](orca/) (LAN IP scrubbed). Start G-code = `START_PRINT BED_TEMP=... EXTRUDER_TEMP=...`, End = `END_PRINT`, Pause = `PAUSE`. Use the `Kingroon KLP1 0.4 nozzle - JRL` preset, not the vendor default.

## Documents

- [Calibration manual](documents/calibration-manual.md) — printer calibration order, per-spool filament calibration, and a command reference.
- [Printer system setup](documents/printer-system-setup.md) — changes to the printer's operating system (clock and time zone, package sources, system update), why each was needed, and how to redo them after a reflash.
- [Session handoff](documents/session-handoff.md) — current state, open items, and every problem, cause and fix from 2026-09-16 to 2026-10-03.

## Change log

- **2026-10-05** — Bed checked after moving the printer. `BED_TRAM`: corners within 0.015 mm, no screw adjustment needed. New `JRLanger` mesh saved: range 0.134 mm (was 0.28). The 25-point mesh took 55 s with the faster probe settings (was about 2 min).
- **2026-10-05** — Hidden features to 8000 mm/s²: `TEST_MOTION SPEED=300 ACCEL=8000` ran clean. `max_accel_to_decel` 6000 → 8000, so Klipper does not throttle short zig-zag moves when Orca requests 8000 (`max_accel` stays 6000 for macros and probing). Orca: inner wall, sparse infill, internal solid infill and travel acceleration 8000; machine limits X, Y, extruding and travel 8000.
- **2026-10-05** — Belts balanced and input shaper recalibrated. Diagonal resonance tests showed belt A at 38.9 Hz against belt B at 52.4 Hz (about 55 % tension). After re-tensioning and moving the printer to a sturdier surface, both peak at 53.9 Hz. New shaper saved: X mzv 56.4 Hz, Y mzv 42.6 Hz (was mzv 49.2 / 2hump_ei 53.0). Max acceleration without excess smoothing rises from 7100 / 3100 to 9400 / 5300 mm/s².
- **2026-10-05** — Faster Orca profile without touching visible features. The KLP1 process inherited Kingroon's generic base, with inner walls at 300 mm/s² (outer walls 3000). Hidden features now run at 6000 mm/s²: inner walls, sparse and solid infill at up to 200 mm/s (capped by the filament flow limit), travel 300 mm/s. Outer walls, top surfaces, first layer, bridges, gap fill and ironing are unchanged. PLA max flow 75 → 15 mm³/s: the flow test ramps only from 6 to 20 mm³/s and failed between 15 and 18. PETG 30 → 12. EN-PLA min layer time 0 → 4 s. Orca machine limits now match the printer (6000, retract 2000). `TEST_MOTION SPEED=300 ACCEL=6000` passed. Simulated on the last print: 213 → 149 min of motion (−30 %).
- **2026-10-05** — Faster probing and homing, checked with `PROBE_ACCURACY` (20 samples at the bed center: standard deviation 0.0018 mm at 5 mm/s and 0.0010 mm at 10 mm/s; one Z step is 0.00125 mm). Probe speed 5 → 10 mm/s, lift 15 mm/s, 2 samples per point instead of 3, tolerance 0.02 mm. Mesh travel 150 mm/s at 3 mm height. `BED_TRAM` travel 150 mm/s (was 2000). Z homing first pass 10 mm/s with a 2 mm back-off; the precise 2 mm/s pass stays. The final lift after homing runs at 10 mm/s. Expected: full mesh about 2 min → 1 min, homing after a print about 60 s → 30 s.
- **2026-10-04** — Filament-change jams, root cause: the hot unload (`MODE=change`) pulled the soft tip into the extruder's gear chamber, where it squashed into a blob and locked the gears. Found by the user inside the extruder body, above the heatbreak. It happened with a new toolhead too. `UNLOAD_FILAMENT` is now always a cold pull, like the Kingroon stock macro: snap the tip 27 mm into the cold zone, cool to the pull temperature, then pull at 5 mm/s. `MODE` removed. `M600` and the runout sensor use it. Costs about 1¾ min per change.
- **2026-10-04** — Replaced the toolhead with a spare after the heat-creep jams. Recalibrated: hotend PID (Kp 32.379, Ki 4.693, Kd 55.852), Z offset 1.310 → 0.450, `JRLanger` mesh. Extruder `rotation_distance` stays 23 (verified: 50 mm commanded = 50 mm moved).
- **2026-10-03** — `UNLOAD_FILAMENT` defaults to the fast `change` mode while paused (the Fluidd button ran a cold pull mid-print). After any unload during a pause the nozzle returns to standby and the hotend-off timer restarts. Synced the `SAVE_CONFIG` block from the printer: z_offset 1.060 → 1.310 (intentional) and the saved `adaptive` mesh.
- **2026-10-02** — Added printer system setup document (clock and time zone fix, Debian archive sources, system update, vnStat reset, harmless boot errors).
- **2026-10-02** — Unload jam, round 2: the hot unload still waited ~15 s for the nozzle to settle at unload temp while Orca's 2 mm retraction sat in the heatbreak. `change` mode now starts at once when already within 20 °C. Pull lengthened from 60 to 100 mm (`LENGTH=`) so the tip clears the gears instead of being pulled out by hand.
- **2026-10-02** — Fix filament-change jams (heat creep): `M600` and runout no longer retract far and cool before unloading. They park hot and unload hot right away (fan off, one continuous 60 mm pull), then cool to standby. Default pause retract 3 → 1 mm. Runout uses a hot unload instead of a cold pull. Hotend-off timer re-armed after the unload.
- **2026-10-02** — Resume no longer marks the part after filament changes: `M600` (and Orca layer pauses with Pause G-code `PAUSE RETURN=next`) purge at the corner, stay retracted, and let the file travel to its next start. Plain `PAUSE` (button, runout) keeps the exact return. Also re-saves Klipper's `PAUSE_STATE`, whose built-in resume otherwise drives back to the pause spot.
- **2026-10-02** — Adaptive bed mesh: `START_PRINT` now probes only the print area every print (`_ADAPTIVE_MESH`, profile `adaptive`, not saved). `ADAPTIVE=0` keeps the old behavior (load the saved `JRLanger` mesh).
- **2026-10-02** — Added `TEST_MOTION` macro (variable-speed motion test: corners, circles, square, X pattern, Z travel).
- **2026-09-18** — `M600` faster: `UNLOAD_FILAMENT` got a `MODE` (`change` = fast hot eject, stays warm for immediate `LOAD`; `clean` = cold-pull, still the default). Removes the cool-to-90 → reheat thermal thrash on filament changes.
- **2026-09-17** — Added calibration manual (printer calibration order, filament calibration per spool, command reference).

- **2026-09-16** — `START_PRINT` now stashes `MATERIAL`; bare `M600`/`LOAD`/`UNLOAD` default to it. Orca start G-code passes `MATERIAL=[filament_type]`.
- **2026-09-16** — Backed up OrcaSlicer presets to `orca/`; renamed KLP1 machine preset to `- JRL`.
- **2026-09-16** — Parametrized `LOAD_FILAMENT`/`UNLOAD_FILAMENT`/`M600` by `MATERIAL` (PLA/PETG/ABS) via a `_MAT` temp table; per-call `TEMP`/`PULL_TEMP` overrides.
- **2026-09-16** — Added `M600` filament change (pause + eject + two-stage resume via Fluidd macro/Resume buttons). Added `[respond]` to printer.cfg (unused by Fluidd; kept for Mainsail popups).
- **2026-09-16** — Enabled pause beeps: `[output_pin beeper]` set to `pwm: True`, added `M300` macro, pause alert = 3 × 1000ms beeps. Verified live.
- **2026-09-16** — `macros.cfg` rewritten and deployed to the printer: two-stage PAUSE/RESUME (retract, bed-drop, back-park, standby cooling, beep; resume reheat/purge then continue), safe `END_PRINT`/`CANCEL_PRINT`, renamed `G29`/`G30`/`G40` to `CALIBRATE_Z_OFFSET`/`CALIBRATE_MESH`/`CALIBRATE_SHAPER`. Verified live (`FIRMWARE_RESTART` → ready). `mainsail.cfg` refreshed to its true symlink-target content.
- **2026-09-16** — Initial backup of current machine state. Obico `auth_token` excluded via `.gitignore`.
