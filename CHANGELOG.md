## 2026-05-30

### Loadcell mesh-probe reliability fixes
- Fixed intermittent first-mesh-point probe failure ("Probe triggered prior to movement" with sample_count=0)
- Three host-side fixes in `puppy_bootloader.py`, all addressing asymmetry between Z-home and bed-mesh probe paths:
  - `run_probe`: added `toolhead.wait_moves()` before tare so the loadcell samples a settled toolhead instead of mid-deceleration
  - `multi_probe_begin`: post-coil-on settle 0.2s → 1.0s so the loadcell FIFO refills with steady-state samples before tare
  - `LoadcellEndstop.home_wait`: disarm via `probe_stop_cmd` instead of re-arming with `oid=0xff` — preserves MCU diagnostic state (count/load/min/max) for failure capture instead of wiping it
- Root cause confirmed by 3-agent code read: trsync framework and MCU loadcell module are both clean; bug was host-side Z-home/mesh path asymmetry
- Multiple clean bed-mesh runs validated post-deploy

### Loadcell probe DIAG instrumentation
- Added `DIAG probe-start` (X/Y/Z/threshold/multi-mode) and `DIAG probe-FAIL` (MCU count/load/min/max/errors + endpos) logging in `run_probe`
- Added `DIAG home_wait NO-HIT` log for non-trigger trsync results
- Provides actionable data in `klippy.log` when probe failures recur

### Anti-ooze: END_PRINT auto-retract + cancel-path retract
- `END_PRINT` macro now does a hot-guarded `G1 E-20 F1800` deep retract to empty the melt zone while the nozzle is still at print temp
- Hot path emits `RESPOND MSG="END_PRINT: deep retract E-20 (nozzle XXXC)"` so users see the action fire; skip path (nozzle below `min_extrude_temp`) emits a corresponding skip message
- New `_MELT_RETRACT_ON_CANCEL` macro mirrors the same retract on print cancel
- `_CLIENT_VARIABLE` macro registers `_MELT_RETRACT_ON_CANCEL` as Fluidd's `user_cancel_macro` hook so cancel-from-pause fires the retract
- Reduces dock-ooze residue and lowers refill-on-next-print burden

### Babystep capture via SAVE_TOOL_Z_OFFSET wrapper
- New `[gcode_macro SET_GCODE_OFFSET]` wraps Klipper's stock `SET_GCODE_OFFSET`
- On a babystep `Z_ADJUST`, calls `SAVE_TOOL_Z_OFFSET` to stage the active tool's `tN_z_offset` immediately
- Stage survives print complete / cancel / soft-error; lost only on hard klippy shutdown (then `SAVE_CONFIG` before risky prints)
- Toolchange and homing `Z_ADJUST` calls pass through untouched

### max_extrude_cross_section bumped
- `5.0 → 50.0` (in `[extruder]`) to accommodate slicer-emitted purge moves that exceed the default 4×nozzle² limit (would otherwise error mid-print on the first prime line)

---

## Upgrading from an earlier version

The install **procedure** itself has not changed (USB BBF flash, MCU build, Python module deploy are all the same). However existing users updating to this version need to take these steps:

### 1. Update Python modules
Re-run the install script's Python deploy step, OR manually copy:
```bash
cp klippy/puppy_bootloader.py /home/pi/klipper/klippy/extras/puppy_bootloader.py
sudo systemctl restart klipper
```

### 2. Copy the new macros into your existing `printer.cfg`
(Skip this step if you're replacing your `printer.cfg` wholesale with the new template — but back up your `SAVE_CONFIG` block first, it contains your calibration!)

From the new `config/printer.cfg`, copy these blocks into your local printer.cfg:
- **`[gcode_macro SET_GCODE_OFFSET]`** — babystep capture (wraps Klipper's stock `SET_GCODE_OFFSET` and stages `tN_z_offset` on `Z_ADJUST`)
- **`[gcode_macro _CLIENT_VARIABLE]`** — Fluidd cancel-macro hook (registers the cancel-retract)
- **`[gcode_macro _MELT_RETRACT_ON_CANCEL]`** — cancel-path deep retract
- **`[gcode_macro END_PRINT]`** — REPLACE your existing END_PRINT block with the new version (adds hot-guarded deep retract + RESPOND messages)

### 3. Bump `max_extrude_cross_section`
In your `[extruder]` block, change `max_extrude_cross_section: 5.0` → `max_extrude_cross_section: 50.0`.
Required if your slicer emits purge moves wider than 4× nozzle area (most modern slicers do for prime lines). No reason not to update.

### 4. Restart klipper
```bash
sudo systemctl restart klipper
```
Or `RESTART` from the gcode console. Verify klippy comes back to "Ready" state with no config errors.

---

## Troubleshooting recipe: occasional hard X/Y home tap → missed tool pick

If you see your X or Y axis occasionally do a "hard tap" at homing (sometimes skipping a step), and that misalignment causes a downstream tool-pick failure ("Tool not detected as picked after wiggle"), the following values were validated on the developer's XL as a starting point you can try:

- `[tmc2130 stepper_x]` and `[tmc2130 stepper_y]`:
  - `homing_speed: 55` (default is 52)
  - `second_homing_speed: 55` (default is 52)
  - `driver_SGT: 0` (default is 1)
- `[homing_override]` macro: change `{% set HOME_CUR = 0.750 %}` → `{% set HOME_CUR = 0.70 %}`

**Rationale:** Higher `homing_speed` puts TMC2130 StallGuard back inside its velocity window where the trigger is more decisive. Lower `HOME_CUR` softens the impact at trip.

**These values are per-printer.** StallGuard sensitivity depends on mechanical variance between XLs (belt tension, frame stiffness, motor batch tolerances, even dock magnet strength). The above is a known-good starting point — your printer's sweet spot may differ. If `0.70` is too low and you start missing home detection entirely, step up by 0.02 at a time. If `55` mm/s feels too fast and causes overshoot, try 54. Do not adopt these values unless you're already experiencing the hard-tap problem.

---

# Changelog

All notable changes to KlipperXL are documented here.

---

## 2026-04-20

### LED Stroboscope for belt tension / VFA visualization
- New `[strobe_test]` module + MCU-side `neopixel_spi.c` — strobes the side LED strip white channel at configurable frequency using TIM13 autonomous timer (no host involvement during strobe)
- New commands:
  - `STROBE_START FREQ=75` — default 75 Hz, 14% duty
  - `STROBE_START FREQ=90 DUTY=35` — custom frequency (0-255 Hz) and duty (0-255)
  - `STROBE_STOP` — stop strobe
- Prusa reference range: 60-130 Hz, 13.7% duty (35/255) — matches stock XL belt tension tool
- Implementation detail: WS2812 data pin (PG14) is shared with SPI6 display. MCU atomically switches pin AF mode → GPIO output and back via PE9 mux, avoiding SPI race conditions
- Config:
  ```
  [strobe_test]
  data_pin: PG14
  mux_pin: PE9
  ```
- New files: `klippy/extras/strobe_test.py`, `src/neopixel_spi.c`; `src/Makefile` adds `neopixel_spi.c` alongside `neopixel.c`
- Requires MCU firmware rebuild + flash (bundled in this release)
- Added helper macros `STROBE_BELT` / `STROBE_OFF` in `print_macros.cfg` — button-friendly wrappers that stop running LED effects on strobe start and restore `effect_idle` on strobe stop
- MCU-side fix: `command_neopixel_spi_strobe` with freq=0 now restores PG14 to GPIO output mode (MODER=01) and sets the PE9 mux HIGH after the final OFF frame, so the standard Klipper neopixel driver can drive the LED strip immediately after strobe stops (previously required a FIRMWARE_RESTART to recover the LEDs)
- Install script (`INSTALL_ON_PI.sh`) now deploys both files and patches `src/Makefile` to build `neopixel_spi.c` automatically

---

## 2026-04-17

### Install guide start G-code fix
- Added `TOOL=[initial_extruder]` to the OrcaSlicer Machine Start G-code example (Section 9.2)
- Without this parameter, `START_PRINT` always defaulted to `TOOL=0` regardless of which tool the slicer assigned — caused prints to use T0 even when another tool was selected
- The `KlipperXL.json` and `KlipperXL_profile.ini` profiles already included `TOOL=[initial_extruder]` — only the manually-pasted install guide line was missing it

### Time Lapse G-code section added
- New "Time Lapse G-code" subsection under 9.3 with `TIMELAPSE_TAKE_FRAME` — pairs with moonraker-timelapse installs

### Between Objects G-code heading clarified
- Heading now notes that OrcaSlicer labels this field "Printing by object G-code" and it is only emitted when Print Sequence = By Object

---

## 2026-04-15

### PCA9557 commands renamed to IOEXP (fixes [#11](https://github.com/racoutlaw/KlipperXL/issues/11))
- Renamed `PCA9557_RELEASE_DWARFS`, `PCA9557_RESET_DWARFS`, `PCA9557_STATUS`, and `PCA9557_SET_PIN` to `IOEXP_*` counterparts
- Klipper's gcode parser strips digits from command names, so `PCA9557_STATUS` was being dispatched as unknown command `PCA9557` and was never callable from the console
- Added `ENCLOSURE_FAN_POWER_ON` / `ENCLOSURE_FAN_POWER_OFF` user-facing commands that toggle PCA9557 pin 6 (12V rail to the enclosure fan header) without requiring users to know pin numbers
- Replaced cached-output-state `_set_pin()` with read-modify-write on the hardware register — the PCA9557 is shared with modbus_stm32f4.c firmware and puppy_bootloader, so cached state could go stale and a naive write could clobber Dwarf reset pins

### Dead modbus I2C scaffolding removed (fixes [#16](https://github.com/racoutlaw/KlipperXL/issues/16))
- Removed 144 lines of unused bit-banged I2C code from `src/modbus_stm32f4.c` — had been triggering `-Wunused-function` warnings since the initial KlipperXL commit
- Functions removed: `i2c_delay`, `i2c_sda_high/low/read`, `i2c_scl_high/low`, `i2c_init`, `i2c_start`, `i2c_stop`, `i2c_write_byte`, `pca9557_write`, `pca9557_release_all`, `pca9557_reset_all`, `pca9557_raw_write`
- Replaced during initial development by hardware I2C (`hw_i2c_init`, `hw_i2c_write`) — old functions were never called
- Verified zero warnings after cleanup; resulting `klipper.bin` is functionally equivalent (removed code was never executed)

---

## 2026-04-14

### Force-flash button sequence documented
- Section 6.2 now explains how to trigger the Prusa bootloader's firmware-update menu: press the selector knob on power-up, or reset + double-press the knob to force the update menu
- Credit @Ro3Deee ([#13](https://github.com/racoutlaw/KlipperXL/issues/13)) for pointing out that some users couldn't get the bootloader to see `klipper.bbf` without the button sequence

### USB BBF is now the default flash method
- Restructured `INSTALLATION_GUIDE.md` to make USB drive flashing the primary path (Section 6)
- DFU flashing moved to Section 12 as an advanced / recovery-only method
- Rationale: USB BBF requires no opening the printer, no BOOT0 jumper, preserves the Prusa bootloader, and future updates are drag-and-drop to a USB stick
- Section 3.4 now instructs users to set bootloader offset to **128KiB with STM32 bootloader (0x08020200)** by default
- New Section 3.6: packaging the firmware as `.bbf` with `pack_fw.py`
- Section 1 (Prerequisites) now lists the FAT32 USB drive and broken-appendix requirement up front
- Troubleshooting updated with USB BBF-first error guidance
- Thanks to @Ro3Deee for raising the issue

---

## 2026-04-05

### Build & Install Fixes
- Fixed missing `build-essential` dependency that caused "cpp: No such file or directory" error during firmware build
- Install script now checks for `cpp` before building and tells you exactly what to install
- Fixed install script path bug that couldn't find source files when run from the scripts directory
- Guide now uses `git clone` instead of referencing a tarball package that was never published
- Guide uses `make menuconfig` instead of a pre-built .config file that was incompatible with current Klipper versions (caused "Multiple definitions for command 'identify'" error)
- All commands now run directly on the Pi — no separate build computer needed
- Added `tool_offsets.py` to the Python module deployment (was missing)
- Fixed section order: Python modules and config files deploy before MCU flash so you can prep the Pi without the printer connected
- Added note that KIAUH already clones Klipper (no duplicate clone step)
- Added section for fixing config includes (Mainsail vs Fluidd, LED effects)

### Side LED Strip Fix
- Corrected LED configuration from `chain_count: 1` / `color_order: RGBW` to `chain_count: 2` / `color_order: GRB`
- Prusa XL uses two daisy-chained WS2812 drivers per strip: Driver 1 handles RGB color, Driver 2 handles white on its green channel
- Confirmed from Prusa firmware source code

### USB Drive Firmware Flashing (New)
- Added advanced guide for flashing KlipperXL via USB drive instead of DFU
- Preserves the Prusa bootloader — no BOOT0 jumper needed for future updates
- Klipper compiled at 0x08020200 offset, packaged as .bbf using Prusa's pack_fw.py tool
- Bootloader progress bar stops at ~50% when Klipper takes over (normal — Klipper doesn't drive the display, just restart the Klipper service)
- Requires broken appendix (safety seal) to bypass signature verification
- Included pack_fw.py in the repo for packaging

### Recovery & Stock Restore
- Complete recovery instructions for both flash methods
- With bootloader preserved: just put stock Prusa .bbf on USB drive and power cycle
- Without bootloader: DFU flash bootloader first, then stock firmware via USB
- Included bootloader-xl-2.5.0.bin and verified stock XL_firmware_6.4.0.bbf in the recovery directory
- Emergency DFU noboot recovery option as last resort

### Dirty State Fix
- Added `tool_offsets.py` to the git exclude list so Klipper update manager stays clean

All changes tested with a complete fresh install from SD card flash through DWARF_STATUS confirmation.
