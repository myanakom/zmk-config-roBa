# roBa ZMK Config - Claude Code Instructions

## Architecture Overview

roBa is a split keyboard using two Seeed XIAO BLE (nRF52840) boards.

- **Right side (roBa_R)**: Central role. Has PMW3610 trackball (SPI) + EC11 encoder.
- **Left side (roBa_L)**: Peripheral role. Has EC11 encoder only.
- BLE split communication uses a fixed UUID (`ZMK_SPLIT_BT_SERVICE_UUID`), NOT keyboard name.
- This fork uses ZMK `main` branch with built-in PMW3610 driver (`CONFIG_INPUT_PMW3610`).
  Upstream (kumamuk-git) uses `v0.3-branch` with external driver (`CONFIG_PMW3610`). Do not confuse these.
- ZMK peripheral does NOT work as a USB keyboard. This is expected behavior.

## Critical Config Files

| File | Role |
|---|---|
| `config/boards/shields/roBa/roBa_L.conf` | Left (peripheral) build config |
| `config/boards/shields/roBa/roBa_R.conf` | Right (central) build config |
| `config/boards/shields/roBa/Kconfig.defconfig` | Split role definitions |
| `config/boards/shields/roBa/roBa.dtsi` | Shared hardware (matrix, encoders) |
| `config/boards/shields/roBa/roBa_L.overlay` | Left side GPIO columns, encoder |
| `config/boards/shields/roBa/roBa_R.overlay` | Right side GPIO columns, trackball SPI |
| `config/roBa.keymap` | Shared keymap for both sides |
| `build.yaml` | GitHub Actions build matrix |

## Past Incident: Left Side Keys Not Working (2026-02)

### Root Cause

`CONFIG_ZMK_POINTING=y` was missing from `roBa_L.conf`.

The keymap (`roBa.keymap`) uses `&mkp LCLK`, `&mkp RCLK`, `&mkp MB3` bindings.
These require `CONFIG_ZMK_POINTING=y` to be enabled. Without it, the left side
firmware **fails to build entirely** because the `&mkp` behavior nodes do not exist.

As a result, the left side `.uf2` file was never generated, and the user was
flashing stale or non-existent firmware.

### Wrong Hypotheses Tried (Do NOT Repeat)

1. **Removing battery proxy settings from roBa_L.conf** - These are harmless on the peripheral side.
2. **Changing ZMK_KEYBOARD_NAME to match between sides** - BLE split discovery uses a fixed UUID, not the keyboard name. Different names per side are correct.
3. **Removing CONFIG_ZMK_BLE_EXPERIMENTAL_CONN** - Unrelated to peripheral discovery.
4. **Removing CONFIG_EC11 from roBa_R.conf** - EC11 is needed on both sides.

### Prevention Checklist

When modifying this repository, ALWAYS verify:

1. **If the keymap uses `&mkp` or any pointing-related bindings, BOTH sides MUST have `CONFIG_ZMK_POINTING=y`.**
   Check: `grep -c '&mkp\|&mmv\|&msc' config/roBa.keymap` - if > 0, both `.conf` files need pointing enabled.

2. **Both `.conf` files must produce a successful build.** If a side's `.uf2` is missing from GitHub Actions artifacts, the build failed silently.

3. **Do not remove settings without understanding their purpose.** Check ZMK docs or the upstream repo (kumamuk-git/zmk-config-roBa) before removing any config line.

4. **Compare with upstream before making changes.** The upstream repo at `https://github.com/kumamuk-git/zmk-config-roBa` is the reference. Always check it when diagnosing issues.

5. **Kconfig.defconfig must define `ZMK_SPLIT=y` for BOTH sides, and `ZMK_SPLIT_ROLE_CENTRAL=y` only for the right side.** Each side should have its own `if SHIELD_ROBA_R` / `if SHIELD_ROBA_L` block.
