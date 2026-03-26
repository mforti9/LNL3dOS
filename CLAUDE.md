# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LNL3dOS is the production Klipper firmware for the LNL3D Vulcan series of dual-extrusion IDEX (Independent Dual Extrusion) 3D printers. The Vulcan D3 is the primary unit, based on the Tenlog D3 platform which ships in two hardware variants: an 8-bit version with a Mega2560 MCU and a 32-bit version with an HC32F460 MCU. The Klipper host is a BigTreeTech (BTT) Pad7 running a CB1 or CB2 compute module. The firmware also supports Vulcan D5 and D6 models (based on Tenlog D5/D6).

## Architecture

### Config Include Hierarchy

`printer.cfg` is the root entry point. It uses Klipper's `[include]` directive to compose a configuration from modular files:

```
printer.cfg
├── mainsail.cfg              # Mainsail UI integration
├── paths.cfg                 # Virtual SD card path
├── LNL3dOS/boards/*.cfg      # MCU pin definitions (pick one)
├── LNL3dOS/printers/*.cfg    # Bed size, axis limits, screw positions (pick one)
├── LNL3dOS/extruders/*.cfg   # Drive ratio & pressure advance (pick one)
├── LNL3dOS/hotends/*.cfg     # Temperature limits (pick one)
├── LNL3dOS/sensors/*.cfg     # Probes & accelerometers (pick as needed)
├── LNL3dOS/utilities/*.cfg   # Optional features (enable as needed)
└── LNL3dOS/utilities/macros.cfg  # Central macro hub, includes:
    ├── idex.cfg              # IDEX mode management
    ├── PrintStartEnd.cfg     # PRINT_START / END_PRINT / CANCEL_PRINT
    ├── LoadUnloadFilament.cfg
    ├── LNLOSVariables.cfg    # Persistent variable storage system
    ├── pidutilities.cfg      # PID calibration macros
    ├── testspeed.cfg
    ├── clickyprobesafety.cfg # Probe dock safety
    └── scripts/shellmacros.cfg
```

The `printer.cfg` pattern is "pick one from each category" — users uncomment exactly one board, one printer, one extruder type, and one hotend, then enable utilities and sensors as needed.

### Persistent Variable System

`LNLOSVariables.cfg` defines the `LNLOS` macro containing ~23 variables (probe positions, IDEX offsets, PID targets, maintenance counters). These are persisted to `variables.cfg` via Klipper's `[save_variables]`. Key macros: `_SAVE_LNLOS`, `_LOAD_LNLOS`, `_ECHO_LNLOS`.

### IDEX Mode System

`idex.cfg` implements three modes stored in the `idex_mode` variable:
- **Mode 0** — Dual Material: independent extruders with T0/T1 tool changes
- **Mode 1** — Copy: both toolheads print the same pattern at an X offset
- **Mode 2** — Mirror: toolheads print mirrored patterns

Mode is saved persistently and restored at print start via `_RESTORE_IDEX_MODE`.

### Installer System

Two installer scripts in `LNL3dOS/scripts/`:
- `LNLOSSoloInstaller.sh` — installs config for a single printer instance
- `LNLOSBatchInstaller.sh` — installs for all `printer_[0-9]*_data` directories

Both use `sed` to rewrite paths in `printer.cfg`, `moonraker.conf`, `paths.cfg`, and `shellmacros.cfg` for the target printer's data directory. The installer can also be triggered from Klipper console via the `_run_installer` gcode macro.

## Hardware

### Platform Heritage

The Vulcan printers are based on the Tenlog platform (TL-D3, TL-D5, TL-D6). Config files still use the Tenlog naming convention (e.g. `TL-D3.cfg`) for the printer definitions.

### Print Area by Model

| | Vulcan D3 (TL-D3) | Vulcan D5 (TL-D5) | Vulcan D6 (TL-D6) |
|---|---|---|---|
| **Print Area** | 305×310mm | 605×495mm | 605×620mm |
| **Z Height** | 350mm | 600mm | 600mm |

### MCU Boards

- **Mega2560** (`boards/Mega2560.cfg`) — 8-bit legacy variant of the Tenlog D3. Arduino pin naming (`ar*`).
- **HC32F460** (`boards/HC32F460.cfg`) — 32-bit variant. The primary/modern board. STM-style pin naming (`PA*`, `PB*`, etc.).
- **Pad7CB1** (`boards/Pad7CB1.cfg`) — Secondary MCU config for the BTT Pad7's onboard CB1/CB2 compute module, communicates via `/tmp/klipper_host_mcu`. Only one printer instance per Pad7 can use this.

### Other Hardware Options

**Extruders:** Standard (1:1 direct drive) or BMG (50:17 geared)
**Hotends:** PTFE-lined (≤251°C) or High-temp (≤301°C)
**Probes:** EuclidProbe (dock-based), BLTouch (servo-based)

## Key Macros

- `PRINT_START BED_TEMP=<> EXTRUDER_TEMP=<>` — full pre-print sequence (home, heat, IDEX restore, mesh)
- `END_PRINT` — retract, park carriages, cool, disable steppers
- `T0` / `T1` — tool change with offset application (dual material mode)
- `ACTIVATE_COPY_MODE` / `ACTIVATE_MIRROR_MODE` / `ACTIVATE_DUAL_MATERIAL_MODE`
- `LOAD_FILAMENT` / `UNLOAD_FILAMENT` — with temperature and speed params
- `PID_ALL` / `PID_EXTRUDERS` / `PID_BED` — heater calibration
- `PROBE_PICKUP` / `PROBE_DROPOFF` — dock probe management
- `_TEST_SPEED SPEED=<> ACCEL=<> ITERATIONS=<>` — mechanical validation

## Important Conventions

- **Jinja2 templating** is used extensively in gcode macros — variables accessed via `printer["gcode_macro LNLOS"].<var>` or `printer.save_variables.variables.<var>`
- **The SAVE_CONFIG block** at the bottom of `printer.cfg` is auto-generated by Klipper — never edit below the `#*# <---------------------- SAVE_CONFIG ---------------------->` marker
- **Path placeholders** like `{printer_data_path}`, `{serial_port}`, `{klippy_port}` in config files are replaced by the installer scripts during setup
- **moonraker.conf** manages the update system — LNLOS self-updates via git from `https://github.com/LNL3D/LNL3dOS.git`
- **KlipperScreen customizations** in `.KlipperScreen/` are custom Python panels (zcalibrate, extrude, move) deployed to `~biqu/KlipperScreen/`
- Multi-printer setups use distinct Moonraker ports (7125, 7126, 7127) and USB serial paths mapped per physical port position
