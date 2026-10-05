# KLP1 Session Handoff

Snapshot: 2026-10-04, evening. This document lists every change made to the printer from 2026-09-16 to 2026-10-04, the problems found, their causes, and the fixes. Use it to continue the work in a new session.

Related documents:

- [README](../README.md): macro table, slicer G-code, restore steps, change log.
- [Printer system setup](printer-system-setup.md): operating system changes (clock, package sources, update) with the commands to redo them.
- [Calibration manual](calibration-manual.md): printer and filament calibration order.

---

## 1. Current state

**Jammed again, root cause found.** On 2026-10-03 and again on 2026-10-04 (with the new toolhead) a filament change (`M600`) jammed the extruder. Both prints `Letras_3D_PLA` were lost. The user found a hardened blob in the extruder's gear chamber, above the heatbreak. The cause was the hot unload ([5.12](#512-filament-jams-during-filament-change-solved-2026-10-04)). The cold-pull unload is deployed and loaded. The user must clear the blob from the extruder before the next print.

**Repository.** The local folder, the GitHub repository and the files on the printer match. The repository is public. It contains no secrets and no LAN addresses.

---

## 2. Open items (do these next)

| # | Item | Why |
| --- | --- | --- |
| 1 | Run Orca's max volumetric speed test (manual B2) for `Creality EN-PLA Red`. | Its preset allows 75 mm³/s, far above what this hotend melts. Fast moves would under-extrude. It also has `slow_down_layer_time` 0, which turns off the slow-down for small layers. |
| 2 | Watch the next filament changes with the cold pull ([5.12](#512-filament-jams-during-filament-change-solved-2026-10-04)). | The fix replaces the hot unload. Check that the tip comes out hard, thin and without a blob. If the cooldown is too slow, a faster hot variant (tip forming) is possible, but it needs the toolhead dimensions (nozzle tip to top of heater block, to top of heatsink, to the gears). |
| 3 | Calibrate flow and pressure advance per filament (calibration manual B3, B4) if not done yet. | New nozzle and extruder. |
| 3b | Balance the belts ([5.19](#519-belt-balance-and-fresh-shaper-data-2026-10-05)), then `CALIBRATE_SHAPER` and save. | Belt A measures about 55 % of belt B's tension. Balanced belts can raise the visible-feature acceleration from 3000 toward about 4500. |
| 4 | Check that `RESUME RETURN=next` leaves no mark after a filament change ([5.9](#59-resume-left-a-mark-on-the-part)). | Tune `resume_retract` (2.0 mm) if needed. |
| 5 | Test `TEST_MOTION` above 100 mm/s. | Only 50 and 100 mm/s were run. Arcs above about 300 mm/s can overload the host (`Timer too close`). |
| 6 | Decide on restart and shutdown macros. | The user asked. The answer: Fluidd already has host reboot and shutdown in its power menu. Macro buttons need the `gcode_shell_command` extension plus a sudoers rule. Not done. |

The KLP1 Orca preset now sets Pause G-code `PAUSE RETURN=next` and color-change retraction 0.6 mm (2026-10-04). Files sliced before that use `M601` (skipped) or 2 mm. Re-slice them.

---

## 3. Access and environment

| Item | Value |
| --- | --- |
| Printer | Kingroon KLP1, MKS Pi host, MKS_THR toolhead board on serial |
| Software | Klipper v0.11.0-122 (2023-02), Moonraker, **Fluidd** (not Mainsail), KlipperScreen, crowsnest, Obico |
| Host OS | Armbian 22.05 on Debian 10 buster, kernel 5.16.20-rockchip64 |
| SSH | user `mks`, key `~/.ssh/id_ed25519` on the Mac (no password). `sudo` needs the `mks` password, which only the user types. |
| Config path | `/home/mks/printer_data/config` |
| Moonraker API | port 7125 on the printer, open to the LAN without login |
| Repository | `JRLanger/klp1-config` on GitHub, **public** |
| Local folder | `/Users/jrlanger/Documents/Claude/Projects/KLP1` |
| Slicer | OrcaSlicer, printer preset `Kingroon KLP1 0.4 nozzle - JRL`. Presets live in `~/Library/Application Support/OrcaSlicer/user/default/`. A copy with the IP removed is in `orca/`. |
| Time zone | America/Sao_Paulo (UTC−3) |

---

## 4. Working method

This method worked for every change. Use it again.

1. Edit the file in the local repository.
2. Render-test changed macros with Klipper's own Jinja environment on the printer: `~/klippy-env/bin/python`, `jinja2.Environment('{%', '%}', '{', '}')`, a mock `printer` object, and sample `params`. This catches template bugs before they move the machine.
3. Copy the old file on the printer to `<file>.bak.<timestamp>`, then copy the new file with `scp`.
4. Run `FIRMWARE_RESTART` through Moonraker. Then check `/printer/info`: `state` must be `ready`. **Never restart during a print.** A restart cancels it and turns the heaters off.
5. Check the loaded values through `/printer/objects/query?configfile=settings`. Note: this view drops `;` comments, so search for code, not comments.
6. Commit with a body that states the cause and the fix, then push.

The permission system asks for approval before Claude writes to the printer over SSH. Reads do not need approval.

---

## 5. Problems and fixes, by topic

### 5.1 Repository and secrets (2026-09-16)

- Scanned all files. The only secret was the Obico `auth_token` in `moonraker-obico.cfg`. That file is in `.gitignore`.
- `.gitignore` also excludes `moonraker.secrets`, `*.bak`, the `printer-*.cfg` backups from `SAVE_CONFIG`, and `.DS_Store`.
- The first `.gitignore` had an inline `#` comment on a pattern line. Git reads that as part of the pattern, so the token file was staged. It was removed before the first commit and never reached the history.
- The repository was created private, then made public at the user's request. LAN addresses were removed from the OrcaSlicer backup.

### 5.2 Macro rewrite and the Mainsail override (2026-09-16)

- The user supplied a rewritten `macros.cfg`: two-stage `PAUSE`/`RESUME`, safe `END_PRINT`/`CANCEL_PRINT`, and `G29`/`G30`/`G40` renamed to `CALIBRATE_Z_OFFSET`/`CALIBRATE_MESH`/`CALIBRATE_SHAPER`.
- `mainsail.cfg` on the printer is a **link** to `~/mainsail-config/client.cfg`, which is read-only. It also defines `PAUSE`, `RESUME` and `CANCEL_PRINT`.
- No conflict: `printer.cfg` includes `macros.cfg` after `mainsail.cfg`, and the later definition replaces the earlier one. The first plan, to comment out the macros in `mainsail.cfg`, was not needed and was reverted.
- The first backup of `mainsail.cfg` in the repository was a trimmed copy. It now matches the real file.

### 5.3 Beeper (2026-09-16)

- `[output_pin beeper]` (PC5) got `pwm: True`. `M300 S<Hz> P<ms>` added.
- Volume is about the same at 1.5–4.5 kHz. The pause alert is 3 × 1000 ms at 1 kHz.

### 5.4 Filament change `M600` (2026-09-16)

- A guided pop-up (`action:prompt`, needs `[respond]`) did not show in Fluidd v1.24.1. The console received the messages, but Fluidd drew nothing. KlipperScreen never shows these pop-ups.
- Final design: `M600` = `PAUSE` + unload + console message. The user loads with `LOAD_FILAMENT`, then presses `RESUME` twice. `[respond]` stays in `printer.cfg` for Mainsail.

### 5.5 Configuration review (2026-09-16)

| Problem | Fix |
| --- | --- |
| The runout `runout_gcode` parked and reheated, which conflicted with the new `PAUSE` | Rewritten. See [5.12](#512-filament-jams-during-filament-change-solved-2026-10-04) for the current version. |
| `CALIBRATE_Z_OFFSET` probed with the bed mesh active | `BED_MESH_CLEAR` after `G28`. |
| `max_accel: 20000`, but homing reset it to 5000 | `max_accel: 6000`. `homing_override.cfg` restores the configured value. |
| Every `G28` loaded mesh `default` | Removed. `START_PRINT` handles the mesh. |
| `min_extrude_temp: 60`, bed `max_temp: 200` | 80 and 120. |

Left as is: `fluidd.cfg` (not included anywhere), Moonraker open to the LAN, X/Y `run_current` 1.0 A.

### 5.6 Materials (2026-09-16)

- `_MAT` holds the temperatures: PLA 220/90, PETG 240/110, ABS 250/120 (melt/cold-pull).
- `START_PRINT` stores `MATERIAL` in `_MAT.current`. A bare `M600` from the slicer uses it. Orca passes `MATERIAL=[filament_type]`.

### 5.7 OrcaSlicer (2026-09-16 to 2026-10-02)

- The vendor preset `Kingroon KLP1 0.4 nozzle` has wrong start G-code and a 230 × 230 bed. Use the `- JRL` preset only.
- Start: `START_PRINT BED_TEMP=[bed_temperature_initial_layer_single] EXTRUDER_TEMP=[nozzle_temperature_initial_layer] MATERIAL=[filament_type]`. End: `END_PRINT`. Change filament: `M600`. Pause: `PAUSE RETURN=next` (set 2026-10-04). Color-change retraction: 0.6 mm. If a field shows the reset arrow and is reset, Orca falls back to the vendor values (`M601`, 2 mm).
- Pressure advance is off in Orca. The firmware has `pressure_advance: 0.02` as a fallback. The calibration manual explains how per-filament values override it.
- The preset was renamed from `- Copy` to `- JRL` by editing the JSON files while Orca was closed. Orca does not allow a user preset to take a vendor preset's name.
- Measured over 16 prints: Orca's time estimate is within 3 % (median). Long waits come from pauses and the start sequence.

### 5.8 Resume oozed on the part (2026-09-16)

- `PAUSE` stores the print position (`rx`, `ry`, `rz`).
- The exact return: purge and wipe at the corner, retract, travel above the part at a safe height, lower straight down, then prime.

### 5.9 Resume left a mark on the part (2026-10-02)

- Cause 1: the resume primed at the old pause point.
- Cause 2: Klipper's built-in `RESUME` runs `RESTORE_GCODE_STATE NAME=PAUSE_STATE MOVE=1`, which always drives back to the pause point.
- Fix: `PAUSE RETURN=next`. The resume purges at the corner, leaves the filament retracted by `resume_retract` (2.0 mm), and saves `PAUSE_STATE` again at the current position. The G-code file then travels to its next start, lowers and primes. Orca writes this travel after `M600` and after its pause G-code.
- A plain `PAUSE` (Fluidd button, runout) keeps the exact return because it can happen in the middle of a line.

### 5.10 Motion test `TEST_MOTION` (2026-10-02)

- Visits the center and corners, then runs circles, a square, an X pattern, and the bed down and up. Speed, acceleration, loops and margin are parameters.
- Bug 1: `I{-r}` rendered as `I85.0`. In Klipper's Jinja, `{-` is the whitespace-trim marker, not a minus sign. Write `I-{r}`.
- Bug 2: Fluidd's macro form sends empty values (`ACCEL=`). Treat an empty string as "use the default".

### 5.11 Adaptive mesh (2026-10-02)

- Klipper v0.11 has no `ADAPTIVE=1`. `_ADAPTIVE_MESH` reads the `EXCLUDE_OBJECT` outlines and probes that area plus 10 mm, at full-mesh point spacing (3 × 3 to 5 × 5). Files without object data get a full mesh.
- The mesh goes to profile `adaptive`. After each print Fluidd offers `SAVE_CONFIG`. Ignore it. A save on 2026-10-03 stored the `adaptive` profile in `printer.cfg`, which is harmless.
- `START_PRINT ADAPTIVE=0` loads the saved `JRLanger` mesh instead.

### 5.12 Filament jams during filament change (solved 2026-10-04)

The same jam happened 5 times. Rounds 1–3 assumed heat creep in the heatbreak. That was wrong. Round 4 found the real cause.

**Root cause:** the hot unload (`MODE=change`, added 2026-09-18) pulled the filament out at 25–40 mm/s with the nozzle at 220 °C. The soft tip reached the extruder gears in about 2 s, before it could harden. The gears squashed it into a blob wider than the path above them, and the blob hardened in the gear chamber. Heating the nozzle cannot free it, because the gear chamber stays cold. The user found the blob there after the jam of 2026-10-04.

Evidence:

- Logs: 15 filament changes with the hot unload between 2026-10-01 and 10-04. 10 resumed and 5 did not, which matches the 5 jams the user reported. The jams started when the user changed to the current PLA. The hot unload worked about 2 times in 3, so one good change proves nothing.
- The jam repeated with a new toolhead, so the old toolhead was not the cause.
- Kingroon's stock unload never pulls a hot tip through the gears. It snaps 27 mm at 150 mm/s, cools to 62 °C, then pulls at 5 mm/s.
- Multi-material tip forming (SuperSlicer, Happy Hare) does the same in principle: fast separation, then the tip hardens in the cold zone before the eject.

| Round | Date | Cause found | Fix |
| --- | --- | --- | --- |
| 0 | 2026-09-18 | `M600` used the cold pull: cool to 90 °C, then reheat | `UNLOAD_FILAMENT MODE=change` (fast and hot). `clean` stays for maintenance. Part fan at 100 % during cold-pull cooling. |
| 1 | 2026-10-02 | Orca retracted 2 mm, then `PAUSE` retracted 3 mm more and cooled with the fan at 100 %. The tip froze in the heatbreak. | `M600` calls `PAUSE RETURN=next RETRACT=0 COOL=0`. Pause retract 3 → 1 mm. Fan off during the unload. Standby only after the filament is out. Runout pauses without cooling and unloads hot. |
| 2 | 2026-10-02 | The unload waited about 15 s for 215 → 220 °C to settle, with the tip in the heatbreak | Start at once when within 20 °C of the target. Pull 100 mm in total (`LENGTH=`). |
| 3 | 2026-10-03 | The sequence ran correctly (210–224 °C, 8 s, no errors) but the filament did not move. It was already stuck. | Toolhead replaced ([5.16](#516-toolhead-replaced-2026-10-04)). |
| 4 | 2026-10-04 | Jam again with the new toolhead. Blob found in the gear chamber, above the heatbreak. | `UNLOAD_FILAMENT` is always a cold pull: E+10 at 5 mm/s, E−15 at 80 mm/s, E−12 at 18 mm/s (tip parked 27 mm up, below the gears), part fan 100 %, cool to `PULL_TEMP` (PLA 90 °C), then pull the rest at 5 mm/s. `MODE` removed. |

### 5.13 Unload button ran a cold pull during a pause (2026-10-03)

- The `UNLOAD_FILAMENT` button sends no `MODE`, so it ran the cold pull mid-print. It left the nozzle at 90 °C and cancelled the hotend-off timer.
- Fix (loaded 2026-10-03 at power-on): while paused, the default mode is `change`. While idle, it is `clean`. After any unload during a pause, the nozzle returns to standby and the 30-minute hotend-off timer restarts.
- Superseded 2026-10-04: `MODE` is gone and every unload is a cold pull. The standby and timer behavior after an unload during a pause stays.

### 5.14 Idle timeout

No change. `PAUSE` sets 12 h. The global value is 10 h. The hotend turns off after 30 min of pause on purpose. `RESUME` heats it again.

### 5.15 Operating system (2026-10-02)

Full details and commands: [printer system setup](printer-system-setup.md).

- The clock was 9 days behind and set to Hong Kong time. Causes: no clock battery, the MKS boot script reports success even when `ntpdate` fails, and `systemd-timesyncd` cancelled `ntp` at boot. Fix: time zone `America/Sao_Paulo`, `systemd-timesyncd` masked, `ntp` enabled.
- Debian buster moved to `archive.debian.org`. The sources were changed and 156 packages updated. Kernel, device tree, bootloader and `armbian-bsp-cli-mkspi` were kept. Never run `apt autoremove` on this printer.
- The vnStat `eth0` database was reset.
- 4 boot failures are harmless: `networking` (CAN), `makerbase-net-mods`, `haveged`, `smartd`.

### 5.16 Toolhead replaced (2026-10-04)

The user replaced the whole toolhead (hotend, extruder, probe, fans, `MKS_THR` board with accelerometer) with a spare of the same model. The board connected without config changes.

| Step | Result |
| --- | --- |
| Fans and probe check | OK |
| Hotend PID at 220 °C | Kp 32.379, Ki 4.693, Kd 55.852 |
| Extruder `rotation_distance` | Stays 23. A first test gave 11.5 because `G1 E50` ran once, not twice, so the measurement was halved. A check at 11.5 extruded about 100 mm for 50 mm commanded. Reverted to 23 and confirmed 50 mm = 50 mm. |
| Z offset | 1.310 → 0.450 |
| `JRLanger` mesh | New 5 × 5 mesh, range about 0.28 mm. It rises about 0.15 mm from left to right. |
| Input shaper | Values unchanged (X mzv 49.2 Hz, Y 2hump_ei 53.0 Hz). |

Lesson: after `FIRMWARE_RESTART` Klipper is in absolute extrusion mode (`M82`). Send `M83` before manual `G1 E` moves, or a repeated `G1 E50` does nothing. Check the console history (`/server/gcode_store`) before acting on a measurement.

### 5.17 Probe and homing speeds (2026-10-05)

The probe is an inductive proximity sensor (metal only, 2 mm range), mounted about 1 mm above the nozzle tip. It is wired to the `MKS_THR` toolhead board, and the Z motor is on the main board. Klipper allows up to 25 ms between a trigger on one board and the motor stop on the other, so the bed can travel speed × 0.025 s past the trigger. The nozzle is only 0.45 mm above the bed at the trigger point (the Z offset), so 10 mm/s is the safe maximum (0.25 mm worst case). Measured round-trip time to both boards is 1–4 ms, so real overtravel is much smaller.

Evidence before the change:

- 14 meshes in the logs (301 points × 3 samples): median spread 0.0013 mm, worst 0.010 mm, no retries. About 5 s per point.
- The average of 2 samples differs from the median of 3 by at most 0.0044 mm (mean 0.0001 mm) on the same points.
- `PROBE_ACCURACY` with 20 samples at the bed center, bed 60 °C and nozzle 150 °C: 5 mm/s gave a standard deviation of 0.0018 mm, and 10 mm/s gave 0.0010 mm. One Z step is 0.00125 mm. The 5 mm/s run drifted down 0.004 mm while the sensor warmed up.
- Each probe touch costs about 0.4 s of fixed overhead at any speed, so fewer samples saves more time than a faster probe.
- Homing took 17–61 s. The long cases start with the bed parked at 200 mm after a print, then a 5 mm/s approach.

| Setting | Before | After |
| --- | --- | --- |
| `[probe] speed` | 5 | 10 |
| `[probe] lift_speed` | not set (5) | 15 |
| `[probe] samples` | 3 | 2 |
| `[probe] samples_tolerance` | 0.05 | 0.02 |
| `[bed_mesh] speed`, `horizontal_move_z` | 50, 5 | 150, 3 |
| `[screws_tilt_adjust] speed`, `horizontal_move_z` | 2000 (capped at 500), 5 | 150, 3 |
| `[stepper_z] homing_speed`, `homing_retract_dist` | 5, 5 (default) | 10, 2 |
| `homing_override.cfg` final `G1 Z10` | F100 | F600 |

Not changed: X/Y sensorless homing at 50 mm/s with `driver_SGTHRS: 110`. Klipper suggests 20 mm/s and 2 s pauses as a starting point, but speed and sensitivity are tuned as a pair. Change them only if homing fails. The first `G1 Z5 F100` in the homing override stays slow because Z is unknown at that point.

If the sensor is ever re-mounted and the Z offset drops below about 0.3 mm, set the probe speed back to 5 mm/s.

### 5.18 Speed and acceleration optimization (2026-10-05)

Goal: the fastest print without a visible quality loss. Visible features keep their settings. Hidden features run faster.

Findings:

- The `- JRL` process preset set no speeds of its own. All values came from Kingroon's generic `fdm_process_common`. It sets inner walls to 300 mm/s² while outer walls use 3000, which looks like a vendor typo (their KP3S profile uses 700).
- Input shaper limits, from Klipper's own calculation: X mzv 49.2 Hz allows 7100 mm/s², Y 2hump_ei 53.0 Hz allows 3100 mm/s². Visible features stay at 3000.
- The PLA max flow of 75 mm³/s did not come from the flow test. `SpeedTestStructure` ramps from 6 to 20 mm³/s only. The furthest run stopped at 18 mm³/s (91 mm/s). An earlier failure at about 75 mm/s equals about 15 mm³/s, so the speed was probably typed into the flow field.
- A planner simulation of the last print (213 min of motion, Orca estimate 220 min) put 30 % of the time in inner walls.

| Setting (Orca) | Before | After |
| --- | --- | --- |
| Inner wall / sparse infill / internal solid infill acceleration | 300 / 3000 / 3000 | 6000 |
| Inner wall / sparse infill / internal solid infill speed | 100 | 200 (capped by the flow limit, about 185–200) |
| Travel | 150 mm/s at 3000 | 300 mm/s at 6000 |
| Internal solid infill pattern | monotonic | rectilinear |
| Max flow: EN-PLA / Generic PLA copy / Hyper-PETG | 75 / 12 / 30 | 15 / 15 / 12 |
| EN-PLA min layer time | 0 s | 4 s |
| Machine limits X, Y, extruding, travel / retracting, E | 10000, 10000, 5000, 9000 / 5000, 5000 | 6000 / 2000 |

Unchanged on purpose: outer walls (80 mm/s, 3000), top surfaces (60 mm/s, 3000), first layer, bridges, overhangs, gap fill, ironing. Applied to both the `- JRL` and `- Toy Cube` process presets. The printer config did not change.

Checks: `TEST_MOTION SPEED=300 ACCEL=6000 CIRCLES=0 LOOPS=2` finished with no errors. Circles were skipped because fast arcs can overload the host. Simulated time for the last print: 213 → 149 min of motion (−30 %). A flow limit of 18 instead of 15 would save only 1 more minute.

Orca stores the rectilinear pattern as `rectilinear` in version 2.4. Its system profiles still use the older `zig-zag`.

### 5.19 Belt balance and fresh shaper data (2026-10-05)

The KLP1 is a CoreXY: both belts move both axes. A diagonal move turns only one motor, so it loads only one belt. `TEST_RESONANCES AXIS=1,1` measures belt A (the `stepper_x` motor) and `AXIS=1,-1` measures belt B (`stepper_y`).

| Diagonal | Main peak | Other peaks |
| --- | --- | --- |
| Belt A (1,1) | 38.9 Hz, broad and low | 56.9, 64.4 Hz |
| Belt B (1,−1) | 52.4 Hz, one strong peak (3× belt A) | 67.3, 40.4 Hz |

The curves correlate 0.69. If the main peaks are the same mode, belt A has about 55 % of belt B's tension (tension ∝ frequency²). The belt ends clamp at the toolhead, so the toolhead swap on 2026-10-04 re-tensioned both belts by hand. Uneven belts split each axis's resonance, which explains why the old calibration chose 2hump_ei for Y.

Fresh `CALIBRATE_SHAPER` with the current belts (not saved, by agreement with the user):

| Axis | Saved | New recommendation | Max accel saved → new |
| --- | --- | --- | --- |
| X | mzv 49.2 Hz | mzv 54.2 Hz (0.0 % vibration) | 7100 → 8700 |
| Y | 2hump_ei 53.0 Hz | mzv 40.2 Hz (0.8 % vibration) | 3100 → 4800 |

On today's data the saved shapers still leave 0.0 % residual vibration on both axes, so no ringing now. They only cap acceleration lower. Next: tighten belt A until the diagonal peaks match within about 2 Hz, then measure again, save the shaper, and raise Orca's outer wall and top surface acceleration to the new limit.

---

## 6. Rules learned

- `mainsail.cfg` is a read-only link. Override its macros in `macros.cfg`.
- Never write `{-` in a macro for a negative value. Write `-{value}`.
- Treat empty parameters as unset. Fluidd's macro form sends every field.
- Klipper's built-in `RESUME` returns to the pause point unless `PAUSE_STATE` is saved again.
- Unknown G-code commands do not stop a print. Klipper only prints `Unknown command`.
- `max_extrude_only_distance` is 100 mm. Split long extruder moves into moves of 45 mm or less.
- `SAVE_CONFIG` restarts Klipper. Never run it during a print.
- Klipper v0.11 limits: no `ADAPTIVE=1`. The `BED_MESH_CALIBRATE` overrides (`MESH_MIN`, `MESH_MAX`, `PROBE_COUNT`) reset after each call.
- Klipper accepts any `SET_VELOCITY_LIMIT ACCEL=` from the slicer, even above `max_accel`. Orca's machine limits are the only cap, so keep them equal to the printer's real limits.
- Never pull a hot, soft tip through the extruder gears. Let it harden in the cold zone first (cold pull). A hot pull squashes the tip into a blob that locks the gear chamber.

---

## 7. Files

| File | Content |
| --- | --- |
| `printer.cfg` | Hardware, limits, runout sensor, beeper, `[respond]`, `SAVE_CONFIG` block (z_offset 0.450, extruder PID, meshes `default`, `JRLanger`, `adaptive`) |
| `macros.cfg` | All macros |
| `homing_override.cfg` | Sensorless X/Y homing, probe Z. Restores accel from the config. |
| `mainsail.cfg` | Snapshot of the read-only link target |
| `MKS_THR.cfg` | Toolhead board, part fan, hotend fan |
| `orca/` | OrcaSlicer presets, IP removed |
| `documents/` | Calibration manual, printer system setup, this handoff |
| Not in the repository | `moonraker-obico.cfg` (token), `printer-*.cfg` and `*.bak.*` backups on the printer |
