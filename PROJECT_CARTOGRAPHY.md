# Project Cartography

## Purpose
`proton-vpn-cli` is the official Linux command-line interface for Proton VPN.

## Top-level map
- `proton/vpn/cli/` — application source code
- `tests/unit/` — unit tests for commands and core behavior
- `debian/` — Debian packaging metadata
- `rpmbuild/` — RPM/Fedora packaging scaffolding
- `scripts/` — release/changelog helper scripts
- `versions.yml` — canonical version/changelog input
- `setup.py`, `setup.cfg`, `requirements.txt` — packaging, lint/test config, dev dependencies

## Runtime entrypoints
- Console script entrypoint (`setup.py`): `protonvpn=proton.vpn.cli:main`
- Main CLI composition: `proton/vpn/cli/__init__.py`
  - Defines Click root command group and command registration
  - Applies async wrapper and top-level exception handling
  - Guards concurrent GUI+CLI usage via DBus name check

## Main source areas
- `proton/vpn/cli/commands/`
  - `account.py` — sign-in/sign-out/account info commands
  - `server.py` — connect/disconnect/status/server listing flows
  - `settings.py` — feature settings config commands
  - `location_discovery.py` — countries/cities lookup commands
  - `command_utils.py` and `feature_setting_definitions.py` — command helpers and setting metadata
- `proton/vpn/cli/core/`
  - `controller.py` — core orchestration over API/client behavior
  - `exceptions.py` / `exception_handler.py` / `click_exception_handler.py` — error models and translation
  - `run_async.py`, `wait_for_current_tasks.py` — async execution helpers

## Testing layout
- `tests/unit/commands/` — command-level behavior and help output tests
- `tests/unit/test_controller.py` — controller orchestration tests
- `tests/unit/test_exception_handler.py` — exception handling tests
- Pytest config in `setup.cfg`:
  - test paths: `tests/unit`
  - coverage target: `proton`

## Build and packaging notes
- Python package metadata and dependencies are declared in `setup.py`
- Debian artifacts are under `debian/`
- RPM spec template is under `rpmbuild/SPECS/package.spec.template`
- Version automation relies on `versions.yml`

## Operational constraints called out in repository docs
- CLI cannot run simultaneously with Proton VPN GUI app
- Headless setups are currently unsupported
- Split tunneling is not yet available
