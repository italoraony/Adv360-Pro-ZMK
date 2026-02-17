# CLAUDE.md - Kinesis Advantage 360 Pro ZMK Firmware

## Project Overview

This repository contains the ZMK firmware configuration for the **Kinesis Advantage 360 Pro** split ergonomic keyboard. It uses the [ZMK Firmware](https://zmk.dev/) framework built on [Zephyr RTOS](https://zephyrproject.org/) targeting the **Nordic nRF52840** SoC.

The keyboard is a split design with independent left (central/master) and right (peripheral/slave) halves communicating over BLE. Each half produces its own `.uf2` firmware file.

## Repository Structure

```
Adv360-Pro-ZMK/
├── .github/workflows/
│   └── build.yml              # GitHub Actions CI workflow
├── bin/
│   └── build.sh               # Docker build script (west build commands)
├── config/                    # Main ZMK configuration directory
│   ├── adv360.keymap          # Main keymap (GENERATED - shared by both halves)
│   ├── adv360_left.keymap     # Left half entry (includes adv360.keymap)
│   ├── adv360_right.keymap    # Right half entry (includes adv360.keymap)
│   ├── macros.dtsi            # Custom macro definitions
│   ├── west.yml               # West manifest (ZMK module sources)
│   ├── info.json              # Keyboard layout metadata for visualization
│   ├── keymap.json            # Keymap in JSON format
│   └── boards/arm/adv360/     # Board-specific hardware definitions
│       ├── adv360.dtsi        # Base device tree (shared hardware config)
│       ├── adv360_left.dts    # Left half device tree (GPIO matrix)
│       ├── adv360_right.dts   # Right half device tree (GPIO matrix)
│       ├── adv360.yaml        # Board metadata
│       ├── board.cmake        # Flash runner config (nrfjprog)
│       ├── Kconfig            # DCDC mode option
│       ├── Kconfig.board      # Board selection options
│       ├── Kconfig.defconfig  # Default Kconfig values
│       ├── adv360_left_defconfig   # Left half build config
│       └── adv360_right_defconfig  # Right half build config
├── firmware/                  # Build output directory (.uf2 files)
├── Dockerfile                 # Build environment container
├── Makefile                   # Local build automation (Podman/Docker)
├── settings-reset.uf2         # Factory reset firmware
└── README.md
```

## Build System

### Build Commands

**Local build (Docker/Podman):**
```bash
make          # Build both halves; outputs to firmware/
make clean    # Remove .uf2 files and Docker images
```

The Makefile auto-detects Podman (preferred) or Docker. It builds inside the `zmkfirmware/zmk-build-arm:stable` container image.

**What happens during build:**
1. Docker builds the container from `Dockerfile`
2. Container runs `west init -l config` then `west update` and `west zephyr-export`
3. `bin/build.sh` executes two west build commands:
   - `west build -s zmk/app -d build/left -b adv360_left -- -DZMK_CONFIG="${PWD}/config"`
   - `west build -s zmk/app -d build/right -b adv360_right -- -DZMK_CONFIG="${PWD}/config"`
4. Output: `{TIMESTAMP}-left.uf2` and `{TIMESTAMP}-right.uf2` in `firmware/`

**CI build (GitHub Actions):**
- Triggers on push, pull request, and manual dispatch
- Runs in `zmkfirmware/zmk-build-arm:stable` container
- Caches west modules keyed on `west.yml` hash
- Uploads `left.uf2` and `right.uf2` as artifact `firmware-{branch}`

### West Manifest

`config/west.yml` defines the ZMK source. This project uses a **custom fork**: `refil/zmk` on the `adv360-z3` branch (not the upstream `zmkfirmware/zmk`). This fork contains Adv360-specific patches.

## Keymap Architecture

### File Relationships

- `adv360_left.keymap` and `adv360_right.keymap` both `#include "adv360.keymap"`
- `adv360.keymap` includes `macros.dtsi` inside the `behaviors` node
- `adv360.keymap` is marked as **GENERATED** — it may be produced by the Kinesis keymap editor tool

### Layers (4 total)

| Layer | Name            | Index | Purpose                                    |
|-------|-----------------|-------|--------------------------------------------|
| 0     | `default_layer` | 0     | QWERTY with window management macros       |
| 1     | `layer_keypad`  | 1     | Numpad overlay on right half               |
| 2     | `layer_fn`      | 2     | F1-F12 function keys, numpad layer toggle  |
| 3     | `layer_mod`     | 3     | BT device selection, RGB/BL, bootloader    |

**Layer access:**
- `&mo 2` (momentary layer 2) on left/right thumb clusters in layer 0
- `&mo 3` (momentary layer 3) accessible from layers 0, 1, and 2
- `&tog 1` (toggle numpad) on layer 2

### Custom Behaviors

**Homerow mods (`&hm`):** Hold-tap behavior with tap-preferred flavor, 200ms tapping term, 175ms quick-tap. Used for modifier keys on home row positions.

### Macros (`config/macros.dtsi`)

| Macro                | Binding                  | Description                     |
|----------------------|--------------------------|---------------------------------|
| `macro_wndw_full`    | `Cmd+Alt+F`             | Window fullscreen               |
| `macro_wndw_center`  | `Cmd+Alt+C`             | Window center                   |
| `macro_wndw_left`    | `Cmd+Alt+Left`          | Window snap left                |
| `macro_wndw_right`   | `Cmd+Alt+Right`         | Window snap right               |
| `macro_quotes`       | `' ' ←`                 | Insert paired single quotes     |
| `macro_dquotes`      | `" " ←`                 | Insert paired double quotes     |
| `macro_braces`       | `{ } ←`                 | Insert paired braces            |
| `macro_parens`       | `( ) ←`                 | Insert paired parentheses       |
| `macro_brackets`     | `[ ] ←`                 | Insert paired brackets          |
| `macro_kinesis`      | Types "KINESIS"          | Character-by-character output   |

Paired-character macros insert both delimiters then move the cursor back one position.

## Hardware Configuration

### Board Details

- **MCU:** Nordic nRF52840 (ARM Cortex-M4, BLE 5.0)
- **USB VID/PID:** 0x29EA:0x0362 (Kinesis Corporation)
- **Key matrix:** 22 columns x 5 rows (11 columns per half)
- **Diode direction:** Column-to-row (`col2row`)
- **Split communication:** BLE (left = central, right = peripheral)
- **LEDs:** 3x WS2812B RGB underglow (GRB order, SPI-driven) + PWM backlight + blue status LED
- **Power:** Battery with ADC voltage measurement, external power control on GPIO0[13]
- **Flash layout:** Softdevice (152KB) → Code (776KB) → Storage/NVS (32KB) → Bootloader (48KB)
- **HID:** NKRO report type

### Key Differences Between Halves

| Setting              | Left                  | Right              |
|----------------------|-----------------------|--------------------|
| Split role           | Central (master)      | Peripheral (slave) |
| TX power boost       | +8dBm                 | Default            |
| Column offset        | 0                     | 11                 |
| BLE name             | "Adv360 Pro"          | "Adv360 Pro rt"    |

## Development Conventions

### Editing the Keymap

1. **Primary edit target:** `config/adv360.keymap` — this is the shared keymap for both halves
2. **Macros:** Define in `config/macros.dtsi` using `zmk,behavior-macro` compatible
3. **Do not edit** `adv360_left.keymap` or `adv360_right.keymap` directly — they only include the shared keymap
4. The keymap file is marked as generated; it may be overwritten by the Kinesis keymap editor. Manual edits should be made carefully
5. The key matrix has a specific positional layout — refer to `config/info.json` for key position coordinates

### Adding New Macros

Add new macros to `config/macros.dtsi` following this pattern:
```dts
macro_name: macro_name{
compatible = "zmk,behavior-macro";
label = "macro_name";
#binding-cells = <0>;
bindings = <&kp KEY1>, <&kp KEY2>;
};
```

Then reference as `&macro_name` in the keymap bindings.

### Adding New Layers

1. Add the layer block in `config/adv360.keymap` after the existing layers
2. Each layer must define bindings for all 76 keys (22 cols x 5 rows, minus some positions)
3. Use `&trans` for transparent (pass-through) keys and `&none` for disabled keys
4. Add layer access bindings (`&mo N` or `&tog N`) in existing layers

### Board/Hardware Changes

- Device tree files are in `config/boards/arm/adv360/`
- `adv360.dtsi` contains shared hardware config; `_left.dts` and `_right.dts` have per-half GPIO mappings
- Kconfig defaults are in `adv360_left_defconfig` and `adv360_right_defconfig`
- The `Kconfig.defconfig` file sets BLE, USB, SPI, and split options

### ZMK Key Code Reference

- Key codes: `dt-bindings/zmk/keys.h` (e.g., `N1`, `A`, `LSHFT`, `LALT`, `LCMD`)
- Bluetooth: `dt-bindings/zmk/bt.h` (e.g., `BT_SEL 0`, `BT_CLR`)
- RGB: `dt-bindings/zmk/rgb.h` (e.g., `RGB_TOG`, `RGB_MEFS_CMD 5`)
- Backlight: `dt-bindings/zmk/backlight.h` (e.g., `BL_TOG`, `BL_INC`, `BL_DEC`)
- Modifier combos use function syntax: `LG()` = Left GUI, `LA()` = Left Alt, `LS()` = Left Shift

## Flashing Firmware

1. Build produces `left.uf2` and `right.uf2`
2. Put each keyboard half into bootloader mode (accessible via layer 3 `&bootloader` binding)
3. The keyboard appears as a USB mass storage device
4. Copy the appropriate `.uf2` file to the device
5. Use `settings-reset.uf2` to factory reset if needed
6. See the [Kinesis Advantage360 Professional Quick Start Guide](https://kinesis-ergo.com/support/kb360pro/) for detailed flashing instructions

## CI/CD

- Every push and PR triggers the GitHub Actions build workflow
- Build artifacts are uploaded as `firmware-{branch}` and can be downloaded from the Actions tab
- The workflow caches west modules for faster subsequent builds
- Both Kconfig outputs are logged in the build for debugging configuration issues

## Important Notes

- This project uses a **custom ZMK fork** (`refil/zmk`, branch `adv360-z3`), not upstream ZMK. Do not change the `west.yml` remote without understanding the implications.
- The `.gitignore` only ignores `firmware/*.uf2` — built artifacts should not be committed.
- The `settings-reset.uf2` is a pre-built binary included in the repo for recovery purposes.
- Window management macros target macOS-style shortcuts (`Cmd+Alt+...`); adjust for other operating systems.
