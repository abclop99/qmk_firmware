# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

QMK is keyboard firmware supporting 3700+ keyboards across AVR and ARM (ChibiOS) microcontrollers. Almost all interaction goes through the `qmk` Python CLI; the actual firmware is built by a layered GNU Make system.

## Common Commands

Run from the repo root. The `qmk` CLI auto-detects the repo.

```bash
# Build / flash firmware for one keyboard+keymap
qmk compile -kb <keyboard> -km <keymap>      # e.g. -kb planck/rev6 -km default
qmk flash   -kb <keyboard> -km <keymap>      # compile + write to a connected board
make <keyboard>:<keymap>                     # equivalent to qmk compile
make <keyboard>:<keymap>:flash               # equivalent to qmk flash
make <keyboard>:all                          # build every keymap for a keyboard

# Scaffolding
qmk new-keyboard                             # interactive new-keyboard wizard
qmk new-keymap -kb <keyboard>                # new keymap from the default
qmk doctor                                   # diagnose toolchain / environment

# Validate & format keyboard data (info.json / keyboard.json)
qmk lint -kb <keyboard> [--strict]           # validate against data/schemas/keyboard.jsonschema
qmk format-json -i path/to/info.json         # format + validate JSON in place

# Format source
qmk format-c       [--core-only] [-a]        # clang-format (.clang-format at repo root)
qmk format-python  [-a]                       # yapf (config in setup.cfg)
make format-core                              # format core C changed vs origin

# C unit tests (GoogleTest) — live under tests/ and quantum/*/tests/
make test:all                                # build + run every unit test
make test:<name>                             # one suite, e.g. make test:caps_word
make list-tests
# (test names cannot contain '-')

# Python CLI tests (nose2; start-dir = lib/python/qmk/tests)
qmk pytest

# Misc
make clean                                   # remove .build
make list-keyboards
```

Firmware output lands in `.build/`. `make` never runs the top-level Makefile in parallel, but `-jN` parallelizes the sub-make for a single target.

## Architecture

The firmware is a stack of layers, from hardware up to user config. When changing behavior, find the layer that owns it rather than special-casing in a keyboard:

- **`tmk_core/`** — the original TMK foundation QMK is built on. Mostly USB/host protocols (`tmk_core/protocol/`: LUFA, V-USB, ChibiOS USB, etc.).
- **`quantum/`** — the QMK core. The main loop, the keycode/action **processing pipeline** (`action.c`, `action_*.c`, `process_keycode/`), and nearly all features (combos, tap dance, leader, caps word, RGB matrix/light, audio, mouse keys, …). `quantum.h` is the umbrella header keymaps include via `QMK_KEYBOARD_H`.
- **`platforms/`** — hardware abstraction for `avr`, `chibios` (ARM), and `test`. Provides `gpio.h`, `timer.h`, `wait.h`, `eeprom.h`, `suspend.c`, etc. **Always use these abstractions** — no direct register/GPIO/I2C/SPI access, `wait_ms()` not `_delay_ms()`, `timer_read()` not raw timers.
- **`drivers/`** — peripheral drivers (oled, led, sensors, encoder, haptic, bluetooth, flash, …) built on the platform abstractions.
- **`keyboards/`** — one directory tree per keyboard. Revisions/variants nest as subfolders. A buildable target has a `keyboard.json`; an intermediate/shared level uses `info.json` (same schema, but not directly buildable). Plus `rules.mk`, `config.h`, `<kb>.c`/`<kb>.h`, and `keymaps/`.
- **`layouts/`** (`default/`, `community/`) — keymaps and layout macros shared across keyboards with a common physical layout (e.g. `tkl_ansi`, `ortho_4x4`).
- **`users/`** — userspace C shared across a contributor's own keymaps, keyed by GitHub username.
- **`lib/python/qmk/`** — the `qmk` CLI, built on the `milc` framework. Each subcommand is a module in `lib/python/qmk/cli/`.
- **`data/`** — the heart of data-driven config: `data/schemas/` (JSON schemas), `data/mappings/` (`info_config.json`, `info_rules.json` map JSON keys → generated `config.h` defines / `rules.mk` vars; `keyboard_aliases.hjson` tracks moved keyboards), `data/templates/`, `data/constants/`.
- **`builddefs/`** — the Make build system glue: `build_keyboard.mk`, `common_features.mk`, `generic_features.mk`, `build_test.mk`, etc.
- **`modules/`** — community modules.

### Processing pipeline (the thing most features touch)
Matrix scan → debounce → matrix position mapped to keycode via the keyboard's `LAYOUT` macro and the active keymap layers → **`process_record` chain** → HID report sent to host. Features hook the chain in sequence; flow is gated by `bool`-returning callbacks at each level: a feature/keyboard/keymap returns `false` to consume the event and stop further processing, `true` to continue.

### The `_kb` / `_user` override pattern
Core features define `__attribute__((weak))` default functions. Each overridable hook comes in two layers: keyboards override `*_kb()` (which should call the corresponding `*_user()` and respect its return), keymaps override `*_user()`. New core callbacks must follow this pattern and return `bool`.

### Data-driven configuration
`keyboard.json`/`info.json` is the **preferred source of truth** for hardware config. The build generates `config.h` defines and `rules.mk` variables from it via `data/mappings/`. Do not duplicate in `config.h`/`rules.mk` anything expressible in JSON, and do not re-define values that already match platform defaults. Layout macros live in `info.json`, not in `<kb>.h`. After editing JSON, run `qmk format-json -i` and `qmk lint`.

## Conventions

C style (full list in `docs/coding_conventions_c.md`; `.clang-format` enforces whitespace, not braces):
- 4-space soft tabs; modified One True Brace Style; **always** include optional braces (`if (c) { return false; }`).
- `#pragma once` in headers, never include guards.
- C-style `/* */` comments preferred; explain *why*, skip the obvious.
- Wrap LAYOUT macros and other alignment-sensitive blocks in `// clang-format off` / `on`.
- GPL2+-compatible license header (SPDX form) on every new `.c`/`.h`. Assignment-only `rules.mk` files are exempt.
- All new files and directories must be lowercase.

Python: yapf + flake8 (`setup.cfg`); E501 long lines are allowed.

## Contributing / branch targeting (upstream PRs)

Branch choice is **not** based on what feels related — it's strict:
- **New keyboard additions (new folder under `keyboards/`) → `master`.**
- **Everything else → `develop`**: core changes, keyboard updates/refactors, keyboard *moves* (also update `data/mappings/keyboard_aliases.hjson`), and data-driven migrations.

PRs should be the smallest set of changes for a single change (don't touch multiple keyboards at once), and should not be filed from your fork's own `master`. Personal keymaps are no longer accepted; default keymaps must stay pristine (no custom keycodes / advanced features). VIA must not be enabled in keymaps. New hardware support needs a build/test target, typically under `keyboards/handwired/onekey`. See `.github/copilot-instructions.md` and https://docs.qmk.fm/pr_checklist for the full checklist.
