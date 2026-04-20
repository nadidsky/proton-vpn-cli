# Copilot Instructions

## Repository quick init
This repository contains the Proton VPN Linux CLI.

Start with:
- `README.md` for product scope and contributor setup
- `PROJECT_CARTOGRAPHY.md` for codebase map
- `proton/vpn/cli/__init__.py` for CLI entrypoint/command registration
- `proton/vpn/cli/core/controller.py` for orchestration logic
- `tests/unit/` for expected behavior
- `HEADLESS_MIGRATION_PLAN.md` for headless/web migration scope and review gates

## Tech and structure
- Python CLI using Click + async helpers
- Main code under `proton/vpn/cli/`
- Unit tests under `tests/unit/`
- Packaging metadata in `setup.py`, `debian/`, and `rpmbuild/`

## Preferred change strategy
- Make small, targeted changes.
- Reuse existing command/controller patterns.
- Keep CLI UX and error messaging consistent with neighboring commands.
- Avoid introducing new dependencies unless required.

## Validation
Use existing repository commands:
- `pytest`
- `flake8 proton tests`

If environment dependencies are unavailable, report that explicitly and avoid fabricating results.

## Files and concerns
- Command behavior: `proton/vpn/cli/commands/*.py`
- Core flow and API orchestration: `proton/vpn/cli/core/*.py`
- User-visible command routing/help: `proton/vpn/cli/__init__.py`
- Tests to update for behavior changes: `tests/unit/**/*.py`
