# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a ZMK firmware configuration for the **Silakka54**, a 54-key wireless split keyboard based on the Lily58 layout. The keyboard uses nice!nano v2 boards (nRF52840) and has 4 fewer keys than the standard Lily58 (no encoder positions and reduced thumb cluster).

**Key architectural decision**: Uses the official `lily58` shield instead of a custom shield, with the 4 missing key positions mapped to `&none` in the keymap. This allows leveraging ZMK's built-in Lily58 physical layout support for ZMK Studio.

## Target Platform

**This keyboard is primarily used on macOS for programming.**

When making keymap changes, consider macOS conventions:
- **CMD (GUI) key is primary**: macOS uses CMD for most shortcuts (CMD+C, CMD+V, CMD+Tab, etc.), not CTRL
- **Home row mods placement**: Current layout has GUI on home row A and L positions for easy CMD access
- **GLOBE key**: Mapped on bottom left for macOS emoji/symbol picker (bottom-left thumb position in default layer)
- **Navigation**: macOS uses CMD+arrows for line/word navigation, Option+arrows for word jumping
- **Shortcuts to prioritize**: CMD+Space (Spotlight), CMD+Tab (app switching), CMD+Q (quit), CMD+W (close window)

Programming-specific considerations:
- **Symbol access**: Brackets `[]`, braces `{}`, parens `()`, and common operators should be easily accessible
- **Current symbol layout**: Lower layer has symbols on right side (right hand), including brackets/braces/parens
- **Navigation keys**: Arrow keys on raise layer (right side: left/down/up/right on J/K/L/;)
- **Numbers**: Available on both default layer (top row) and raise layer for easy access
- **Common programming keys**: Backtick, minus, equals, backslash all accessible without excessive layer switching

## Building Firmware

### GitHub Actions (Primary Method)
Firmware builds automatically via GitHub Actions on every push/PR. The workflow is defined in `.github/workflows/build.yml` and uses ZMK's reusable `build-user-config.yml@v0.3` workflow.

To trigger a build:
```bash
git push
```

Then download artifacts from the Actions tab on GitHub. The build produces:
- `lily58_left-nice_nano_v2-zmk.uf2` (left half)
- `lily58_right-nice_nano_v2-zmk.uf2` (right half)
- `settings_reset-nice_nano_v2-zmk.uf2` (for pairing reset)

### Local Building
For local builds, follow [ZMK documentation](https://zmk.dev/docs/development/setup) for toolchain setup. This is rarely needed since GitHub Actions handles builds.

## Key Configuration Files

### `build.yaml`
Defines the build matrix for GitHub Actions:
- Left half: `nice_nano_v2` + `lily58_left` + `studio-rpc-usb-uart` snippet
- Right half: `nice_nano_v2` + `lily58_right`
- Settings reset: `nice_nano_v2` + `settings_reset`

**Important**: The left half includes `studio-rpc-usb-uart` snippet for ZMK Studio USB connectivity.

### `config/west.yml`
Specifies ZMK firmware dependency:
- Remote: `zmkfirmware/zmk`
- Revision: `v0.3`

To update ZMK version, change the `revision` field and push.

### `config/lily58.conf`
Firmware feature configuration:
- `CONFIG_ZMK_STUDIO=y` - Enables real-time keymap editing via USB
- `CONFIG_ZMK_STUDIO_LOCKING=n` - No unlock required for ZMK Studio
- `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` - Boost Bluetooth transmission power
- Debounce: 1ms press, 10ms release (eager debouncing)

### `config/lily58.keymap`
Main keymap definition with devicetree syntax:

**Custom Behaviors**:
- `hm` (homerow mods): Hold-tap behavior with 200ms tap term, 175ms quick-tap, 150ms prior-idle requirement, balanced flavor

**Layers**:
1. **default_layer**: Base layer with home row mods on ASDF/JKL; (GUI/Alt/Ctrl)
2. **lower_layer**: Bluetooth controls, function keys, symbols (accessed via `&mo 1`)
3. **raise_layer**: Navigation arrows, function keys, numbers (accessed via `&mo 2`)
4. **extra_1/2/3**: Reserved layers for future use

**Critical constraints**:
- Physical layout has 54 keys total
- 4 positions must be mapped to `&none`: the 2 inner thumb keys and 2 encoder positions
- Standard Lily58 has 58 keys, so the keymap uses 6 columns but marks 4 positions as `&none`

## Modifying the Keymap

When editing `config/lily58.keymap`:

1. **Preserve `&none` positions**: The 4 key positions marked `&none` correspond to physical keys not present on Silakka54. Do not change these to actual keycodes unless the hardware changes.

2. **Layer structure**: Each layer must have exactly the same binding structure:
   ```
   bindings = <
   &kp ESC ... (6 keys) ...              ... (6 keys) ... &kp GRAVE
   &kp TAB ... (6 keys) ...              ... (6 keys) ... &kp MINUS
   &kp LCTRL ... (6 keys) ...            ... (6 keys) ... &kp SQT
   &kp LSHFT ... (6 keys) ... &none &none ... (6 keys) ... &kp RSHFT
                    ... &none &none ... (3 keys)
   >;
   ```

3. **Home row mods syntax**: Use `&hm <modifier> <key>` for hold-tap behavior, e.g., `&hm LGUI A`

4. **Testing changes**: After modifying, push to GitHub and wait for Actions build, or use ZMK Studio for live testing without reflashing.

## Bluetooth Configuration

All Bluetooth controls are on the **lower_layer** (hold left middle thumb key):

- **Profile management**: `&bt BT_SEL 0-4` to select profiles, `&bt BT_CLR` to clear current profile
- **Output selection**: `&out OUT_USB` / `&out OUT_BLE` to force output mode
- **Clear all**: `&bt BT_CLR_ALL` to wipe all profile pairings

The keyboard supports 5 Bluetooth profiles for multi-device pairing.

## ZMK Studio Integration

This firmware has ZMK Studio enabled for real-time keymap editing:
- Connect left half via USB
- Open [ZMK Studio](https://zmk.studio) in Chrome/Edge
- Changes save immediately to keyboard's persistent storage
- **Note**: The 4 `&none` positions will appear in ZMK Studio UI but are non-functional

## Common Tasks

### Update ZMK version
Edit `config/west.yml` and change `revision: v0.3` to desired version/branch, then push.

### Add a new layer
1. Add layer definition in `config/lily58.keymap` after `raise_layer`
2. Ensure it has correct binding structure (54 positions with 4 `&none`)
3. Add layer toggle/momentary binding in other layers (e.g., `&mo 3` for layer index 3)

### Modify behavior parameters
Edit the behavior definition in the `behaviors` node:
```c
hm: homerow_mods {
    compatible = "zmk,behavior-hold-tap";
    tapping-term-ms = <200>;     // Hold threshold
    quick-tap-ms = <175>;        // Repeat tap window
    require-prior-idle-ms = <150>; // Idle before hold
    flavor = "balanced";         // Hold-tap timing flavor
    bindings = <&kp>, <&kp>;
};
```

### Debug connection issues
1. Flash `settings_reset-nice_nano_v2-zmk.uf2` to both halves
2. Re-flash firmware
3. Use Lower layer BT controls to re-pair devices

## Git Workflow

- Main branch: `main`
- Current branch: Feature branches like `cc/layout0`
- Commit messages should describe keymap/config changes clearly
- Each push triggers automatic firmware build via GitHub Actions
