# KLP1 Calibration Manual

Printer: Kingroon KLP1. Firmware: Klipper. Interface: Fluidd (Mainsail is also installed). Slicer: OrcaSlicer.

This manual has four parts. Do them in this order. Each part uses the results of the parts before it.

| Part | Content | When |
| --- | --- | --- |
| Part M | Mechanics and base setup (M1 to M3) | One time, and after a hardware change |
| Part A | Printer calibration (A1 to A5) | One time, and after a hardware change or a move |
| Part B | Filament calibration (B0 to B8) | For each filament |
| Part C | Final checks (C1, C2), optional | One time, after Part B for your main filament |

Software cannot correct loose hardware. The filament calibration gives wrong values if the printer is not calibrated.

---

## Safety

- The nozzle reaches 220 C or more. The bed reaches 60 C or more. Do not touch them.
- `SAVE_CONFIG` restarts Klipper. Do not run `SAVE_CONFIG` during a print.
- Move the nozzle down in small steps near the bed. A large step pushes the nozzle into the build plate.
- Turn off the heaters after each calibration. The printer keeps the heaters on until the idle timeout.

---

# Part M. Mechanics and base setup

Do these steps before Part A. A loose belt or a wrong extruder value makes every later result wrong.

## M1. Check the mechanics

Do this step one time. Do this step again after you change a belt, the toolhead, or a motor, and after the printer falls or moves in a car.

1. Turn off the motors. Run `M84`.
2. Push the toolhead and the bed by hand in each direction. They must move without play and without a click.
3. Tighten the frame screws that you can reach.
4. Heat the nozzle to 230 C. Tighten the nozzle with a small turn. Do not use force. A hot nozzle seals against the heat break. A cold nozzle can leak.
5. Check the belt tension. The KLP1 is a CoreXY printer. Each belt moves both axes, so the two belts must have the same tension.
    1. Run `G28`.
    2. Run `TEST_RESONANCES AXIS=1,1`. This test moves only the belt of `stepper_x` (belt A).
    3. Run `TEST_RESONANCES AXIS=1,-1`. This test moves only the belt of `stepper_y` (belt B).
    4. Each test writes a CSV file. The console shows the file path. Compare the main peak of the two files.
    5. The two peaks must be within 2 Hz of each other. If the peaks are different, tighten the belt with the lower peak. The tension screws are on the outside of the printer.
    6. Do the tests again after each change.

Tension changes with the square of the frequency. A peak of 39 Hz against 52 Hz means that the first belt has only 55 % of the tension of the second belt.

## M2. Tune the heaters (PID_CALIBRATE)

Do this step after you change the heater, the thermistor, the hotend, or the toolhead.

1. Let the hotend and the bed cool to room temperature.
2. Run `PID_CALIBRATE HEATER=extruder TARGET=220`. Use your usual print temperature.
3. Wait for the test to finish. The test takes about 5 minutes.
4. Run `PID_CALIBRATE HEATER=heater_bed TARGET=60`.
5. Wait for the test to finish. The bed test takes longer.
6. Run `SAVE_CONFIG`. Klipper restarts.

A stable temperature gives a stable extrusion. The temperature graph must show a flat line during a print. A wave of more than 1 C shows a bad PID tune.

## M3. Calibrate the extruder (rotation_distance)

Do this step after you change the extruder or the extruder gears. The flow rate (B3) builds on this value.

1. Heat the nozzle to the print temperature of the loaded filament.
2. Mark the filament 120 mm above the entry of the extruder.
3. Run `M83`. After a restart Klipper uses absolute extrusion, and `G1 E` moves do not move as expected.
4. Run `G1 E100 F60` one time only. The extruder pushes 100 mm at 1 mm/s. Wait for the move to stop. The move takes 100 s.
5. Measure the distance from the extruder entry to the mark.
6. Calculate the extruded length: 120 minus the measured distance.
7. Calculate the new value: new `rotation_distance` = old `rotation_distance` × extruded length / 100.
8. Write the new value in `printer.cfg`, section `[extruder]`. `SAVE_CONFIG` does not save this value.
9. Restart Klipper. Do the test again. The extruded length must be between 99 mm and 101 mm.

Example: the old value is 23. The measured distance is 22 mm. The extruded length is 98 mm. The new value is 23 × 98 / 100 = 22.54.

Send the `G1` command one time only. A second command adds 100 mm more and the result is wrong.

---

# Part A. Printer calibration

Do these steps in this order. Each step uses the result of the step before it.

## A1. Level the bed (BED_TRAM)

Do this step after you move the printer. Do this step also when the mesh shows a large tilt.

1. Clean the build plate with isopropyl alcohol.
2. Open the Fluidd console.
3. Run `BED_TRAM BED_TEMP=60`.
4. Wait for the probe to measure the four screw positions.
5. Read the report in the console. The report gives a turn direction and an amount for each screw.
6. Turn each screw as the report says.
7. Run `BED_TRAM BED_TEMP=60` again.
8. Repeat steps 5 to 7 until each screw shows less than 5 minutes of adjustment.

## A2. Measure the bed mesh (CALIBRATE_MESH)

Do this step after every tram. Do this step also when you change the build plate.

1. Run `CALIBRATE_MESH BED_TEMP=60 SOAK=10`.
2. Wait for the bed to heat and soak. The soak takes 10 minutes.
3. Wait for the probe to measure the mesh.
4. Run `SAVE_CONFIG`. Klipper restarts.
5. Open the Fluidd Tune page. Find the bed mesh. Check the total range of the mesh.

A range below 0.2 mm is good. A range above 0.4 mm means the bed needs a tram. Go back to step A1.

## A3. Set the Z offset (CALIBRATE_Z_OFFSET)

Do this step after a nozzle change. Do this step also when the first layer fails across the full bed.

1. Set the nozzle to 150 C. Clean the nozzle tip with a brass brush.
2. Run `CALIBRATE_Z_OFFSET BED_TEMP=60`.
3. Wait for the printer to probe and to move to the bed center.
4. Put a sheet of paper under the nozzle.
5. Click the -1 or -0.1 button until the nozzle is near the paper.
6. Move the paper with your hand after each step.
7. Change to the -0.05 or -0.025 button when the nozzle touches the paper.
8. Stop when the paper moves with a small drag. The nozzle must scratch the paper but not hold it.
9. Click ACCEPT.
10. Click SAVE_CONFIG. Klipper restarts.

To stop the procedure without a change, click ABORT.

## A4. Fine-tune the Z offset on a print

The paper test gives an approximate value. A real first layer gives the exact value.

1. Print a single-layer square of 50 x 50 mm.
2. Open the Fluidd dashboard. Find the Z offset adjustment in the Toolhead panel.
3. Click - in steps of 0.01 mm when the lines show gaps or round edges.
4. Click + in steps of 0.01 mm when the surface is rough, transparent, or smeared.
5. Stop when the lines touch each other and the surface is smooth.
6. Click the save icon next to the Z offset. Fluidd runs `Z_OFFSET_APPLY_PROBE`. If you cannot find the icon, run `Z_OFFSET_APPLY_PROBE` in the console.
7. Run `SAVE_CONFIG` after the print ends.

## A5. Calibrate the input shaper (CALIBRATE_SHAPER)

Do this step one time. Do this step again after you change a belt or a motor.

1. Connect the accelerometer.
2. Run `CALIBRATE_SHAPER`.
3. Wait for the test. The printer shakes each axis.
4. Read the suggested shaper and frequency for each axis in the console.
5. Run `SAVE_CONFIG`. Klipper restarts.

Do not run this test with the part fan on. The fan adds noise to the measurement.

---

# Part B. Filament calibration

Do Part B for each new material and each new brand. Save the result as a filament profile in OrcaSlicer.

Use one profile name for one spool type. Example: `Voolt PLA Black`.

## B0. Dry the filament

A new spool can hold water. Water causes stringing, bubbles, and weak parts.

| Material | Temperature | Time |
| --- | --- | --- |
| PLA | 45 to 50 C | 4 to 6 h |
| PETG | 65 C | 4 to 6 h |
| TPU | 50 C | 4 to 6 h |

Dry the filament before the other steps. A wet spool gives wrong values.

## B1. Temperature tower

1. Open OrcaSlicer. Select the filament profile.
2. Click Calibration. Click Temperature.
3. Set the start and end temperature for the material.
4. Print the tower.
5. Break each block with your hand. Find the block with the best layer strength.
6. Look at the surface of each block. Find the block with the least stringing.
7. Select the lowest temperature that gives both results.
8. Write the temperature in the filament profile.

## B2. Max volumetric speed

1. Click Calibration. Click Max flowrate.
2. Print the test model.
3. Find the height where the surface loses material.
4. Read the flow value for that height.
5. Multiply the value by 0.85.
6. Write the result in the filament profile.

## B3. Flow rate

1. Click Calibration. Click Flow rate. Click Pass 1.
2. Print the test.
3. Look at the top surface of each block. Find the block with no gaps and no ridges.
4. Write the flow ratio of that block in the filament profile.
5. Click Calibration. Click Flow rate. Click Pass 2.
6. Print the test. Select the best block again.
7. Write the new flow ratio in the filament profile.

Judge the top surface only. The side walls do not show the flow error.

## B4. Pressure advance

1. Click Calibration. Click Pressure advance.
2. Select the line method or the pattern method.
3. Print the test.
4. Find the line with the most equal width at the start and at the end.
5. Read the pressure advance value for that line.
6. Write the value in the filament profile. Turn on the pressure advance option.

Set pressure advance per filament in OrcaSlicer, not per print. `printer.cfg` keeps a small default (`pressure_advance: 0.02`) as a fallback for prints started without the slicer. When a filament profile has pressure advance turned on, OrcaSlicer sends `SET_PRESSURE_ADVANCE` at the start of the print and that value overrides the firmware default at runtime. There is no conflict: the last value sent wins.

Turn on pressure advance in every filament profile. The value from `SET_PRESSURE_ADVANCE` stays active until the next restart. If one profile leaves it off, the value from the last print can carry over. One value per profile avoids this.

## B5. Retraction

1. Click Calibration. Click Retraction test.
2. Print the test model.
3. Find the lowest retraction length with no strings between the towers.
4. Write the value in the filament overrides of the profile.

The KLP1 has a direct drive extruder. Expect a value between 0.4 mm and 1.0 mm. A value above 2 mm can cause a clog.

## B6. Cooling and overhangs (optional)

1. Print an overhang test model or a bridge test model.
2. Increase the fan speed when the overhangs curl or sag.
3. Decrease the fan speed when the layers separate.
4. Write the fan values in the filament profile.

## B7. Tolerance (optional)

Do this step for parts that must fit together.

1. Click Calibration. Click Tolerance test.
2. Print the test model.
3. Find the fit that you want.
4. Write the compensation value in the process profile. Use the X-Y hole compensation field and the X-Y contour compensation field.

## B8. Z offset per material

PETG sticks harder than PLA. A low nozzle smears the first layer.

1. Open the filament profile. Find the start G-code field.
2. Add the line `SET_GCODE_OFFSET Z_ADJUST=0.03`.
3. Change the value between 0.02 and 0.05 after a test print.

Do not change the global Z offset for one material.

---

# Part C. Final checks (optional)

Do Part C after Part B for your main filament. These checks need a good flow rate and pressure advance.

## C1. Outer wall speed (VFA test)

VFA means vertical fine artifacts. At some speeds the motors make small vertical lines on the walls. This test finds the speeds to avoid.

1. Open OrcaSlicer. Click Calibration. Click VFA. Some versions show VFA under More.
2. Print the test tower. Each band of the tower prints at a different speed.
3. Look at the walls under a side light. Find the bands with no vertical lines.
4. Set the outer wall speed of each process profile to a speed from a clean band.

Do the test again after you change a motor, a belt, or the input shaper.

## C2. Skew correction

Do this step only for parts that must have exact dimensions or must fit together. A skewed printer prints a square as a parallelogram.

1. Run `SET_SKEW CLEAR=1`. The test print must not use an old correction.
2. Print a skew calibration square. Search for "Klipper skew calibration" on Printables.
3. Measure the diagonal from corner A to corner C, the diagonal from corner B to corner D, and the side from corner A to corner D. Use a caliper.
4. Run `SET_SKEW XY=AC,BD,AD`. Replace `AC`, `BD` and `AD` with your measurements in mm.
5. Run `SKEW_PROFILE SAVE=CaliSkew`.
6. Run `SAVE_CONFIG`. Klipper restarts.
7. Add `SKEW_PROFILE LOAD=CaliSkew` to `START_PRINT` after the homing.
8. Print the square again. The two diagonals must be within 0.1 mm of each other.

`printer.cfg` must have a `[skew_correction]` section for these commands. The section is not in `printer.cfg` yet. Add it before step 1.

---

# How much to calibrate again

| Situation | Steps to do |
| --- | --- |
| New toolhead or new hotend | M1, M2, M3, A3, A4, A5, B2, B3, B4 |
| New extruder or extruder gears | M3, B3, B4 |
| Belt change or new belt tension | M1 (belt check), A5, C1 |
| New material or new brand | B0 to B5 |
| New color, same brand and material | B1, B3, B4 |
| New spool, same brand, material, and color | B0, then one flow test (B3 pass 2) |
| New nozzle size or new nozzle type | A3, A4, B2, B3, B4 |
| Printer moved to a new place | A1, A2 |
| New build plate | A2, A3, A4 |

---

# Command reference

| Command | Action |
| --- | --- |
| `PID_CALIBRATE HEATER=extruder TARGET=220` | Tune the hotend heater. Use `HEATER=heater_bed TARGET=60` for the bed. |
| `TEST_RESONANCES AXIS=1,1` | Measure belt A. Use `AXIS=1,-1` for belt B. |
| `M83` | Use relative extrusion for manual `G1 E` moves. |
| `BED_TRAM BED_TEMP=60` | Probe above each bed screw. Report the adjustment. |
| `CALIBRATE_MESH BED_TEMP=60 SOAK=10` | Heat, soak, home, probe the mesh, save the profile. |
| `CALIBRATE_Z_OFFSET BED_TEMP=60` | Start the interactive Z offset procedure. |
| `CALIBRATE_SHAPER` | Measure the resonances. Suggest an input shaper. |
| `TESTZ Z=-0.05` | Move the nozzle down 0.05 mm in the Z offset procedure. |
| `ACCEPT` | End the Z offset procedure and keep the value. |
| `ABORT` | End the Z offset procedure without a change. |
| `SAVE_CONFIG` | Write the result to `printer.cfg`. Klipper restarts. |
| `Z_OFFSET_APPLY_PROBE` | Add the babystep value to the saved Z offset. |
| `SET_SKEW XY=AC,BD,AD` | Set the skew correction from three measurements. Needs `[skew_correction]`. |
| `SKEW_PROFILE SAVE=CaliSkew` | Save the skew correction. Use `LOAD=CaliSkew` to load it. |
