# CLAUDE.md

## Project overview
- Project: Proton VPN CLI for Linux (`proton-vpn-cli`)
- Language: Python (>=3.9)
- Primary app type: Click-based async CLI

## Where to look first
- CLI entrypoint and command wiring: `proton/vpn/cli/__init__.py`
- Command implementations: `proton/vpn/cli/commands/`
- Core orchestration and business flow: `proton/vpn/cli/core/controller.py`
- Unit tests: `tests/unit/`
- Packaging and dependency metadata: `setup.py`
- Test/lint config: `setup.cfg`
- Headless/web migration planning artifact: `HEADLESS_MIGRATION_PLAN.md`

## Developer workflow
1. Initialize submodules:
   - `git submodule update --init --recursive`
2. Create and activate a virtual environment:
   - `python3 -m venv venv`
   - `source venv/bin/activate`
3. Install dependencies:
   - `pip install -r requirements.txt`

## Validation commands
- Tests: `pytest`
- Lint: `flake8 proton tests`

## Coding and change guidelines
- Keep changes minimal and scoped to the request.
- Prefer extending existing command/core patterns over introducing new abstractions.
- Preserve Click UX conventions already used by existing commands.
- Add/adjust unit tests in `tests/unit/` for behavior changes.
- Do not change packaging/version files unless the task requires it.

## High-level architecture
- Click commands parse user input and dispatch to controller/core functionality.
- Core layer interacts with Proton VPN API/session/connector dependencies.
- Exceptions are normalized and rendered as user-facing CLI messages.
- Async helpers bridge coroutine-based operations with CLI command execution.

## Known repo constraints
- CLI and Proton VPN GUI should not run simultaneously.
- Some functionality is plan-tier dependent.
- Packaging support includes Debian and RPM metadata in-repo.
