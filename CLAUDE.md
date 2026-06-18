# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Klipper configuration backup** for a Voron 2.4 (350mm, CoreXY) running on a BigTreeTech Octopus V1 (STM32F446), with a multi-tool **toolchanger** setup. There is no application code, build step, or test suite here — every file is a Klipper/Moonraker config (`.cfg`/`.conf`) that runs on the printer's host (a Raspberry Pi). The repo is pushed automatically by [klipper-backup](https://github.com/Staubgeborener/klipper-backup); the auto-commit messages (e.g. "New backup from ...") come from that script, not manual edits.

Everything lives under `printer_data/config/`.

## How config is wired together

Klipper loads a single `printer.cfg`, which pulls in everything else via `[include ...]` directives. Order matters: Klipper keeps the **last** value seen for any setting, so include order is significant.

- `printer.cfg` — entry point. Defines `[mcu]`, `[printer]` kinematics/limits, QGL, bed mesh, board pins, and ends with the auto-generated `SAVE_CONFIG` block (`#*# ...`) holding live-calibrated values (PID, bed mesh, input shaper, tool Z-offset). **Do not hand-edit the `SAVE_CONFIG` block** — Klipper regenerates it.
- `printer.cfg` includes, in order: `mainsail.cfg` → `toolchanger/readonly-configs/toolchanger-include.cfg` → `hardware/hardware.cfg` → `macros/macros.cfg`.
- `toolchanger-include.cfg` is provided by the **klipper-toolchanger-easy** plugin (installed at `~/klipper-toolchanger-easy`, see `moonraker.conf`) and is **not tracked in this repo**. `toolchanger/toolchanger-config.cfg` is the user override file — per the comments in it, it must be included *last* so its values win.

### Directory roles
- `hardware/` — fan-out includes from `hardware.cfg`: `heaters/` (extruder, bed), `leds/` (Stealthburner status LEDs), `steppers/` (a-b-steppers, z-steppers), `fans.cfg`, `sfs.cfg` (Smart Filament Sensor). `probe.cfg`/`btt-ebb36.cfg` exist but are commented out in `hardware.cfg`.
- `macros/` — fan-out from `macros.cfg`: `print_start.cfg`, `print_end.cfg`, `G32.cfg` (QGL+home), `preheat.cfg`, `nozzle_scrub.cfg` (`CLEAN_NOZZLE`), `load_filament.cfg`/`unload_filament.cfg`, `M600.cfg` (filament change), `backup_config.cfg` (git backup macros).
- `toolchanger/tools/` — one file per physical tool. `T0.cfg` is the active tool; `example_T0.cfg`/`example_T1.cfg` are templates (remove the `example_` prefix to activate). Each tool file defines its own CAN MCU (`[mcu EBBT0]`), `[extruder]`, fans, `[tool TN]` (dock park coords + per-tool input shaper), and `[tool_probe TN]` (per-tool Z-offset).

### Conventions for tool configs
When adding/editing a tool, the per-tool identifiers must all be renumbered consistently: `[mcu EBBTn]`, `[tool Tn]` `tool_number: n`, `extruder`/`extruder1`/..., `fan: Tn_partfan`, `[tool_probe Tn] tool: n`, and the `[gcode_macro Tn]` wrapper that calls `SELECT_TOOL T=n`. `params_park_x/y/z` are physical dock coordinates and are build-specific.

## Print start flow (`PRINT_START`)

The slicer calls `PRINT_START TOOL=.. TOOL_TEMP=.. BED_TEMP=.. CHAMBER=..`. Sequence: stop crash detection → home → `PREHEAT` → activate T0 → `CLEAN_NOZZLE` → `QUAD_GANTRY_LEVEL` → re-home Z → adaptive `BED_MESH_CALIBRATE` → switch to print tool if `TOOL>0` → heat hotend → clean → re-enable crash detection. Toolchanger crash detection (`STOP/START_CRASH_DETECTION`) and `tool_probe_endstop` are toolchanger-plugin features guarded by `{% if printer.tool_probe_endstop is defined %}`.

## Editing & deploying

There is nothing to build or run locally — these files only take effect on the printer. Validate changes by restarting Klipper on the host (`FIRMWARE_RESTART`, or via Mainsail). Secrets are kept out of git: `.env` and `secrets.conf` are gitignored, along with `*.bak`, timestamped `printer-*.cfg`, and `.zip`/`.csv` artifacts.

`backup_config.cfg` exposes an `UPDATE_GIT` macro (`update_git`) that runs `~/klipper-backup/script.sh` to commit and push this repo from the printer.
