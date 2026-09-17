# KLP1 Calibration Manual

Printer: Kingroon KLP1. Firmware: Klipper. Interface: Mainsail. Slicer: OrcaSlicer.

This manual has two parts. Part A is the printer calibration. Part B is the filament calibration. Do Part A first. The filament calibration gives wrong values if the printer is not calibrated.

---

## Safety

- The nozzle reaches 220 C or more. The bed reaches 60 C or more. Do not touch them.
- `SAVE_CONFIG` restarts Klipper. Do not run `SAVE_CONFIG` during a print.
- Move the nozzle down in small steps near the bed. A large step pushes the nozzle into the build plate.
- Turn off the heaters after each calibration. The printer keeps the heaters on until the idle timeout.

---

# Part A. Printer calibration

Do these steps in this order. Each step uses the result of the step before it.

## A1. Level the bed (BED_TRAM)

Do this step after you move the printer. Do this step also when the mesh shows a large tilt.

1. Clean the build plate with isopropyl alcohol.
2. Open the Mainsail console.
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
5. Open the Mainsail heightmap page. Check the total range of the mesh.

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
2. Open the Mainsail dashboard. Find the Z offset panel.
3. Click - in steps of 0.01 mm when the lines show gaps or round edges.
4. Click + in steps of 0.01 mm when the surface is rough, transparent, or smeared.
5. Stop when the lines touch each other and the surface is smooth.
6. Click Save in the panel. The panel runs `Z_OFFSET_APPLY_PROBE`.
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

Do not write a `pressure_advance` value in `printer.cfg`. OrcaSlicer sends the value for each filament.

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

# How much to calibrate again

| Situation | Steps to do |
| --- | --- |
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
| `BED_TRAM BED_TEMP=60` | Probe above each bed screw. Report the adjustment. |
| `CALIBRATE_MESH BED_TEMP=60 SOAK=10` | Heat, soak, home, probe the mesh, save the profile. |
| `CALIBRATE_Z_OFFSET BED_TEMP=60` | Start the interactive Z offset procedure. |
| `CALIBRATE_SHAPER` | Measure the resonances. Suggest an input shaper. |
| `TESTZ Z=-0.05` | Move the nozzle down 0.05 mm in the Z offset procedure. |
| `ACCEPT` | End the Z offset procedure and keep the value. |
| `ABORT` | End the Z offset procedure without a change. |
| `SAVE_CONFIG` | Write the result to `printer.cfg`. Klipper restarts. |
| `Z_OFFSET_APPLY_PROBE` | Add the babystep value to the saved Z offset. |
