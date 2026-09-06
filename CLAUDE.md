# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a custom ZMK (Zephyr Mechanical Keyboard) firmware configuration for a Lily58 split keyboard. The project creates a personalized keymap with custom combos, macros, and behaviors optimized for development workflows.

## Build System

This project uses GitHub Actions for automated building via `build.yaml`. The build matrix targets:
- Board: `nice_nano_v2`
- Shields: `lily58_left`, `lily58_right`, `settings_reset`
- Additional snippet: `studio-rpc-usb-uart` for the left half
- A diagnostic `lily58_left-usblogging` target (see "Debugging with USB logs")

No local build commands are available - all builds happen through GitHub Actions
when pushed to the repository. The workflow triggers on changes to `config/**`,
`build.yaml`, and the workflow file itself.

### ZMK version pinning (read before changing)

Two things must stay in lockstep:

- `config/west.yml` pins ZMK to `v0.2.1`
- `.github/workflows/build.yml` pins the reusable workflow to
  `build-user-config.yml@v0.2.1`

Do not bump one without the other. The `@main` workflow runs
`west boards --format "{qualifiers}"`, which requires Zephyr 4.1's hardware
model v2; against v0.2.1's Zephyr 3.5 it dies with `KeyError: 'qualifiers'` and
the build fails even though the firmware itself compiles.

Moving to ZMK `main` also requires renaming the board from `nice_nano_v2` to
`nice_nano//zmk` (HWMv2 board variant), and removing any `label =` property
from devicetree nodes, since Zephyr 4.x dropped it from device bindings.

We deliberately stayed on `v0.2.1`: there is still no tagged ZMK release on
Zephyr 4.1 (`v0.3.0` is Zephyr 3.5), so `main` means tracking a moving branch.
The one measurable power win from newer ZMK is backported below. Revisit when a
release ships on Zephyr 4.1.

## Debugging with USB logs

`build.yaml` builds a `lily58_left-usblogging` artifact using the
`zmk-usb-logging` snippet. Flash it to the left half, plug in USB, and open the
CDC console (`/dev/ttyACM0`, 115200) to see ZMK's own log output -- keymap
bindings firing, `zmk_ble_prof_select`, `update_advertising`, endpoint
selection. This is far faster than inferring behaviour from the config.

It is a diagnostic image, not a daily driver: it sets `-DCONFIG_ZMK_STUDIO=n`
because Studio's UART RPC transport needs the `zmk,studio-rpc-uart` chosen node
that only the `studio-rpc-usb-uart` snippet provides, and a target may carry
only one snippet.

## Architecture

### Core Configuration Files

- **Main keymap**: `config/lily58.keymap` - The active keymap with custom layouts, combos, and macros
- **Reference keymap**: `config/boards/shields/lily58/lily58.keymap` - Default Lily58 keymap for reference
- **Build configuration**: `build.yaml` - GitHub Actions build matrix
- **West manifest**: `config/west.yml` - Points to ZMK firmware repository
- **Shield definition**: `config/boards/shields/lily58/lily58.zmk.yml` - Hardware configuration

### Key Features in Main Keymap

**Custom Behaviors:**
- `hold_or_tap` behavior with 200ms tapping term for modifier keys
- Home row mods on ASDF and JKL; keys

**Extensive Combos (19 total):**
- Tmux shortcuts (next window, split panes)
- Task switcher (Cmd+Tab forward/backward)  
- Common shortcuts (copy, paste, undo, cut)
- Navigation (backspace, enter, tab, shift+tab)
- Grave accent replacement

**Custom Macros:**
- Tmux workflow automation
- Enhanced task switching with macro_press for sustained modifiers

**Layer Structure:**
- Layer 0: Base QWERTY with home row mods
- Layer 1: Symbols and numpad, plus BLE profile (`&bt`) and output-endpoint
  (`&out`) selection on the top row
- Layer 2: Window management, function keys, and external power (VCC rail) controls
- Layer 3: Mouse controls and media keys
- Layers 4-5: Reserved for future expansion

## Bluetooth

The top row of Layer 1 has both `&bt` and `&out` bindings, and they do
different things:

- `&bt BT_SEL 0..4` picks which BLE *profile* (host) is active.
- `&out OUT_BLE / OUT_USB / OUT_TOG` picks the output *endpoint*.

ZMK defaults the endpoint to USB whenever USB is connected, so with the cable
plugged in the `&bt` keys will appear to do nothing until the output is moved
to BLE with `&out OUT_BLE`. This is the usual cause of "the BT keys don't work
and it only types over USB".

If BLE is still broken after that, reset the stored bonds: flash the
`settings_reset` firmware to BOTH halves, then flash the normal left/right
firmware to both, then re-pair with the host.

## Development Workflow

When modifying keymaps:

1. Edit `config/lily58.keymap` for changes
2. Commit and push to trigger GitHub Actions build
3. Download firmware from Actions artifacts
4. Flash to keyboard using ZMK's bootloader mode

## Key Customizations

The keymap is heavily customized for macOS development with:
- Home row modifiers for ergonomic typing
- Tmux integration for terminal multiplexing
- Window management shortcuts for macOS tiling
- Task switching optimizations
- Mouse and scroll wheel emulation on Layer 3

## Power / Battery

RGB underglow is compiled out. The OLEDs are ON by default.

The shield declares a 36-LED WS2812 chain per half
(`config/boards/shields/lily58/lily58.dtsi`), and those chips draw ~0.5-1mA
each even when dark, so `CONFIG_ZMK_RGB_UNDERGLOW=n` alone does not stop the
drain -- only cutting the switched VCC rail does. On a nice!nano that same rail
also powers the OLED, so the display and the LED quiescent draw are a package
deal: keeping the display means keeping the rail up while the board is awake.

Other power settings:
- `CONFIG_SSD1306_DEFAULT_CONTRAST=40` (~16%) dims the OLED. Zephyr's ssd1306
  driver writes this to the contrast register (0x81) at init, which sets segment
  drive current. Range 0-255; Zephyr's default is 128, so stock is ~50%, not max.
  26 is ~10%, 51 is ~20%.
- `CONFIG_BOARD_ENABLE_DCDC_HV=n` backports ZMK commit 8059e67, which landed in
  v0.3.0. Upstream measured with a Nordic PPK2 that the nRF52840 high-voltage
  DC-DC stage *increases* draw on a nice!nano v2 at a typical ~4V battery and
  3.3V rail. v0.2.1 still defaults it on.

What limits the cost:
- Deep sleep (`CONFIG_ZMK_SLEEP=y`) drops the whole rail after 15 minutes idle.
- `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE=y` blanks the OLED after 60s. This turns
  off pixels only -- the rail, and the LED draw, stay up until deep sleep.
- Only `&ext_power EP_ON` remains bound (Layer 2), as a recovery key.

EP_OFF and EP_TOG were deliberately removed. Cutting the rail is a one-way
trip for the OLED: ZMK's display module (`app/src/display/main.c`) only calls
`display_blanking_on/off()` on activity changes and has no `pm_device` resume
or re-init path -- `initialized` is a one-shot flag set at boot. So when the
rail is cut the SSD1306 loses its whole register state, and EP_ON restores
power to a chip that is never re-initialised. The screen stays blank until a
reboot. Deep sleep is unaffected, because ZMK uses PM_STATE_SOFT_OFF and
nRF52 SYSTEM OFF wakes via reset, which re-runs display init.

If the display is dark and ext_power was previously turned off, the saved
state lives in settings: press EP_ON and then reset the board, or flash the
`settings_reset` firmware.

All power-related Kconfig lives in `config/lily58.conf`. The shield-level
`.conf` files under `config/boards/shields/lily58/` are deliberately kept free
of it so the two cannot drift out of sync.

## Hardware Configuration

- **Controller**: nice!nano v2 (nRF52840)
- **Keyboard**: Lily58 Pro split keyboard  
- **Features**: Rotary encoder support, OLED displays (layer + battery widgets). RGB underglow is compiled out for battery life (see `config/lily58.conf`)
- **Wireless**: Bluetooth Low Energy with ZMK
- **Power**: Battery level reporting enabled for both halves