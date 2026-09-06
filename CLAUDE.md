# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a custom ZMK (Zephyr Mechanical Keyboard) firmware configuration for a Lily58 split keyboard. The project creates a personalized keymap with custom combos, macros, and behaviors optimized for development workflows.

## Build System

This project uses GitHub Actions for automated building via `build.yaml`. The build matrix targets:
- Board: `nice_nano_v2` 
- Shields: `lily58_left`, `lily58_right`, `settings_reset`
- Additional snippet: `studio-rpc-usb-uart` for the left half

No local build commands are available - all builds happen through GitHub Actions when pushed to the repository.

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
- Layer 1: Symbols and numpad
- Layer 2: Window management, function keys, and external power (VCC rail) controls
- Layer 3: Mouse controls and media keys
- Layers 4-5: Reserved for future expansion

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

RGB underglow and the OLEDs are intentionally disabled. The shield declares a
36-LED WS2812 chain per half (`config/boards/shields/lily58/lily58.dtsi`), and
those chips draw ~0.5-1mA each even when dark, so the switched VCC rail must be
cut to stop the drain -- `CONFIG_ZMK_RGB_UNDERGLOW=n` alone is not enough.
`&ext_power EP_OFF` (Layer 2) cuts that rail; the state persists in settings.
Deep sleep (`CONFIG_ZMK_SLEEP=y`) also cuts it automatically after 15 minutes.

All power-related Kconfig lives in `config/lily58.conf`. The shield-level
`.conf` files under `config/boards/shields/lily58/` are deliberately kept free
of it so the two cannot drift out of sync.

## Hardware Configuration

- **Controller**: nice!nano v2 (nRF52840)
- **Keyboard**: Lily58 Pro split keyboard  
- **Features**: Rotary encoder support. RGB underglow and OLED displays are compiled out for battery life (see `config/lily58.conf`)
- **Wireless**: Bluetooth Low Energy with ZMK
- **Power**: Battery level reporting enabled for both halves