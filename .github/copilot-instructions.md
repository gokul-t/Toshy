# Copilot instructions for Toshy

## Project overview

Toshy is a Linux utility that remaps keyboard input to make Linux behave more like macOS. The codebase is not a traditional app with a single build pipeline; it is a combination of:

- `setup_toshy.py`: the canonical installer/bootstrap script for Linux distros and user services
- `default-toshy-config/`: the default keymapper config and config templates
- `toshy_common/`: shared environment detection, keyboard layout analysis, service logic, and runtime helpers
- `toshy_gui/`: tray/GUI preferences UI
- `systemd-user-service-units/`, `kwin-dbus-service/`, `cosmic-dbus-service/`, `wlroots-dbus-service/`: distro/desktop-specific runtime integrations and D-Bus helpers
- `vendors/xwaykeyz-*`: vendored upstream keymapper dependency used by Toshy; treat these as dependency code, not the main editing surface unless the task is explicitly about the vendored xwaykeyz work
- `tests/`: verification scripts for modmap, key detection, and screenshot/keymap parsing

This project is highly environment-sensitive: a change can affect keyboard remaps, desktop integration, Wayland/X11 behavior, and distro-specific package setup. Keep fixes focused on the runtime and integration points involved.

## Purpose of this fork

The `user_custom` branch is a backup of personal Toshy configuration customizations from the installed `~/.config/toshy/toshy_config.py`, maintained alongside upstream Toshy changes. The corresponding repository template is `default-toshy-config/toshy_config.py`. Treat the installed config as the source of truth for personal edits: sync only the intended content between matching upgrade-safe slice markers into the template before committing. During an upgrade, the installer copies the template and merges matching slices from the existing installed config into it, so template-only changes do not necessarily update an existing install. Never replace either full config with the other or add local preferences/database files to the repository.

## Build, test, and lint commands

There is no normal compile step for the repo. The main project action is installation/setup:

- `./setup_toshy.py install` — install Toshy for the current distro and user environment
- `./setup_toshy.py list-distros` — list known distro profiles
- `./setup_toshy.py install --override-distro <name>` — use a distro override when needed

For runtime validation and debugging, the project provides user commands after install:

- `toshy-services-status` — show service state
- `toshy-services-log` — inspect service logs
- `toshy-debug` — run verbose debugging output in the terminal

Testing:

- `python3 -m pytest tests/test_modmap_hygiene.py -q` — run a single representative test file
- `python3 -m pytest tests/test_modmap_integrity.py -q`
- `python3 -m pytest tests/test_sc_det_keynames.py -q`
- `python3 -m pytest tests/test_screenshot_keymap_builder.py -q`

Use the same pattern to run any one test file or any selected test node. If the repo's Python environment is not yet installed, bootstrap it with the project setup flow first.

Linting:

- `ruff check .` — the repo includes `ruff.toml` and uses Ruff for linting
- `ruff format --check .` — if formatting is relevant to the change

## Key conventions and repo-specific patterns

- Keep changes compatible with Linux desktop environments and distro-specific setup logic; this repo is partly an installer and runtime integration project, not just a library.
- Prefer modifying the existing installer/runtime data flow (`setup_toshy.py`, `toshy_common`, desktop service helpers) over adding ad hoc one-off scripts.
- The default config files are important source of truth for keyboard remaps and preference behavior; if a fix affects shortcuts or modifier behavior, inspect the config and the relevant handler before editing unrelated code.
- The project relies heavily on environment detection. If a change is about application-specific shortcuts, check the detection logic and the active desktop integration path before patching the remap itself.
- Desktop-environment-specific code is split by environment (KWin, COSMIC, wlroots, GNOME, etc.). When touching a behavior that depends on a particular DE/WM, keep the fix local to that integration and avoid broad cross-platform assumptions.
- The vendored `xwaykeyz` code under `vendors/` is a dependency subtree; do not treat it as generic application code unless the task specifically concerns that upstream keymapper.
- Test files under `tests/` are focused on keyboard mapping and layout parsing; if you change keyboard detection, modmap behavior, or shortcut parsing, check the corresponding test coverage first.
- Do not run the installer as root. `setup_toshy.py` explicitly refuses to run with elevated privileges.

## Practical workflow

- Start by checking the relevant integration point: installer, config, desktop service, or shared runtime module.
- For user-facing keyboard behavior changes, verify the active environment and app-window detection path first.
- Keep fix scope narrow and aligned with the project’s runtime/install model rather than building unrelated abstraction layers.
- When editing install or service logic, prefer patterns already used by the project’s shell, systemd unit, and D-Bus service files over introducing new orchestration methods.

## High-level architecture

The project has a layered runtime model:

1. Installer (`setup_toshy.py`) prepares native packages, Python environment, user services, and desktop integration hooks.
2. Runtime detection (`toshy_common/`) decides what desktop environment, keyboard type, and layout are active.
3. Keymapper integration uses the vendored xwaykeyz stack plus app/window awareness from desktop-specific D-Bus or protocol adapters.
4. User-facing preferences and debugging are exposed through `toshy_gui/` and the `toshy-*` shell commands.
5. The generated config and service units are the operational heart of the app; they live outside the Python package entrypoints and are designed to be hot-reloaded or restarted as needed.

This is a “live system integration” project more than a standard library project, so architecture is best understood by tracing installer -> runtime detection -> desktop integration -> keymapper config.
