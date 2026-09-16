# OrcaSlicer presets

Backup of the OrcaSlicer user presets for this printer.

- `machine/` — printer profiles (KLP1 uses `Kingroon KLP1 0.4 nozzle - JRL`, which calls `START_PRINT`/`END_PRINT`).
- `filament/` — filament profiles (PLA, Creality Hyper-PETG).
- `process/` — print/quality profiles.

`print_host` (the printer's LAN IP) has been **removed** from these copies. After importing, set your printer IP in Orca (Printer settings → Print host / physical printer).

## Restore

1. In OrcaSlicer: quit the app.
2. Copy these `.json` files into `~/Library/Application Support/OrcaSlicer/user/default/{machine,filament,process}/` (macOS). On Windows: `%APPDATA%/OrcaSlicer/user/default/...`.
3. Reopen Orca; select the `- JRL` printer preset.

> The KLP1 machine preset inherits from Orca's bundled vendor preset `Kingroon KLP1 0.4 nozzle` — that vendor preset ships with wrong start G-code and a 230×230 bed; always use the `- JRL` preset, not the vendor one.
