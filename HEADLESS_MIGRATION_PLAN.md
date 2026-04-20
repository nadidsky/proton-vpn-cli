# Proton VPN CLI Headless Migration Plan (Backward-Compatible)

## Objective
Enable reliable headless operation for `proton-vpn-cli` while preserving current CLI behavior, command UX, and supported packaging targets.

## Scope
- In scope: CLI/runtime/auth/tooling changes needed for headless operation.
- In scope: preserving current Debian/RPM packaging surfaces and existing command flags/flows.
- In scope: evaluating and introducing non-desktop-dependent runtime alternatives compatible with currently supported OSes.
- In scope: phased migration from desktop-coupled runtime dependencies to cross-OS headless-capable equivalents.
- Out of scope: full web product implementation (covered as an extension path).

## Current headless blockers (from current codebase)
1. Session DBus GUI guard in `proton/vpn/cli/__init__.py` (`MessageBus(...SESSION)` + GTK app name lookup).
2. Desktop-oriented credential dependencies (`proton-keyring-linux` integration path).
3. Interactive-only signin flow in `proton/vpn/cli/commands/account.py` (`getpass`, interactive 2FA prompt).
4. Runtime assumptions around desktop/session services in downstream connector stack (local agent + NetworkManager ecosystem).

## Backward-compatibility requirements
1. **No command removals** (`signin`, `connect`, `disconnect`, `status`, `config`, etc. remain unchanged).
2. **No behavior break for existing desktop users** (interactive flow remains default).
3. **No packaging regressions** (keep Debian + RPM build/installation paths).
4. **Preserve existing error semantics** where possible; only add new headless-specific guidance.
5. **Feature parity by default**: if a feature works today on supported systems, it must continue to work after migration.

## Design principles
- Add headless support as an additive runtime mode, not a replacement.
- Make desktop/session checks best-effort and non-fatal when not required.
- Separate auth input methods from core controller logic.
- Keep CLI surface stable; introduce new flags/env vars only where needed.

## Proposed phased implementation

### Phase 1: Runtime detection and safe fallback
**Goal:** Stop failing early when no graphical/session DBus is present.

Changes:
- Wrap GUI-concurrency DBus check with robust failure handling.
- If session DBus is unavailable, skip GUI process-name guard and continue.
- Keep current “CLI+GUI cannot run simultaneously” behavior when DBus is available.

Acceptance:
- Existing desktop behavior unchanged.
- CLI can run in environments without `DBUS_SESSION_BUS_ADDRESS`.

### Phase 2: Non-interactive authentication path
**Goal:** Support automation/server usage without TTY prompts.

Changes:
- Add optional non-interactive auth inputs (for example `--password-stdin`, `--password-env`, `--otp-env`).
- Keep current interactive `signin <username>` as default if no non-interactive inputs are provided.
- Add clear precedence/validation rules for auth input sources.

Acceptance:
- Existing users can continue interactive signin unchanged.
- CI/server users can sign in without prompt-based input.

### Phase 3: Credential backend abstraction
**Goal:** Avoid hard dependency on desktop keyring semantics.

Changes:
- Define a credential storage interface with multiple backends:
  - existing desktop keyring backend (default where available),
  - headless backend option (file-based encrypted store, OS key store, or token-only mode).
- Add explicit backend selection strategy:
  - auto detect, with override via env/config.

Acceptance:
- Desktop users keep current keyring path.
- Headless users can authenticate without a desktop keyring session.

### Phase 4: Connector/runtime hardening for headless
**Goal:** Validate end-to-end connect/disconnect/status flows without a GUI session.

Changes:
- Audit and remove assumptions that require GUI/session services for connector startup.
- Ensure local-agent and NetworkManager interactions are independent of desktop session bus.
- Evaluate and stage non-desktop-dependent connector/runtime alternatives that preserve behavior on all currently supported OSes.
- Improve failure messages when system services are missing (e.g., NM daemon not available).

Acceptance:
- `connect`, `disconnect`, `status` work in supported headless environments.
- Error paths clearly distinguish auth issues vs system service issues.
- At least one cross-OS, non-desktop-dependent runtime path is validated behind a compatibility-safe rollout plan.

### Phase 5: Packaging and tooling updates (no OS support loss)
**Goal:** Keep currently supported OS packaging while enabling headless installs.

Changes:
- Keep Debian/RPM package outputs.
- Reclassify desktop-only dependencies (if any) as optional where safe.
- Add package/runtime checks to avoid pulling GUI/session components when not needed.
- Document required system services per mode:
  - desktop mode,
  - headless/server mode.

Acceptance:
- Current install paths continue to work.
- Headless-targeted installs are possible without desktop stack requirements.

### Phase 6: Testing, CI, and rollout
**Goal:** Ship safely with compatibility guarantees.

Changes:
- Add tests for:
  - missing session DBus,
  - interactive + non-interactive signin compatibility,
  - credential backend selection,
  - headless connect/disconnect smoke scenarios.
- Add CI matrix dimensions for desktop-like and headless environments.
- Release behind staged feature flag if needed, then graduate to default auto mode.

Acceptance:
- No regressions in current unit tests.
- New headless scenarios are covered by automated tests.

## Code-grounded migration examples (one level deeper)

This section maps proposed changes directly to current code surfaces so the migration work is implementation-ready.

### 1) Session DBus guard fallback (`proton/vpn/cli/__init__.py`)
Current anchor:
- `_vpn_gui_running()` currently assumes session bus access (`MessageBus(...SESSION).connect()`).
- `app()` enforces GUI/CLI exclusivity by calling `_vpn_gui_running()`.

Migration example:
```python
async def _vpn_gui_running() -> bool:
    try:
        bus = await MessageBus(bus_type=BusType.SESSION).connect()
    except Exception:  # session bus not available in headless mode
        return False

    reply = await bus.call(...)
    if reply.message_type == MessageType.ERROR:
        return False
    return GTK_APP_ID in reply.body[0]
```

Compatibility effect:
- Desktop behavior stays the same when session DBus exists.
- Headless/server usage no longer fails before command execution.

### 2) Non-interactive signin path (`proton/vpn/cli/commands/account.py`, `core/controller.py`)
Current anchor:
- `signin()` uses `getpass.getpass` and interactive 2FA prompt.
- `Controller.login()` already accepts callables (`get_password`, `get_2fa`), which is a good seam.

Migration example:
```python
@click.option("--password-stdin", is_flag=True)
@click.option("--password-env", type=str)
@click.option("--otp-env", type=str)
async def signin(ctx, username, password_stdin, password_env, otp_env):
    get_password = select_password_provider(password_stdin, password_env)
    get_2fa = select_otp_provider(otp_env)  # falls back to interactive prompt
    await controller.login(username, get_password, get_2fa)
```

Compatibility effect:
- Existing `protonvpn signin <username>` remains unchanged.
- Automation gains non-interactive inputs without changing controller contract.

### 3) Credential storage with validated schema (keyring + headless secure backends)
Current anchor:
- Packaging currently hard-depends on `proton-keyring-linux` (`setup.py`, `debian/control`, `rpmbuild/SPECS/package.spec.template`).

Migration example (Pydantic-backed backend config and secret envelope):
```python
from pydantic import BaseModel, Field, model_validator
from typing import Literal
from pathlib import Path

class SecretBackendConfig(BaseModel):
    backend: Literal["keyring", "pass", "file+age", "TPM2", "memory"]
    secret_dir: Path | None = None
    tpm2_key_handle: str | None = None

    @model_validator(mode="after")
    def validate_backend_requirements(self):
        if self.backend in {"pass", "file+age"} and self.secret_dir is None:
            raise ValueError("secret_dir is required for file-backed secret stores")
        if self.backend == "TPM2" and self.tpm2_key_handle is None:
            raise ValueError("tpm2_key_handle is required for TPM2-backed store")
        return self

class SecretEnvelope(BaseModel):
    account: str
    ciphertext_b64: str
    wrapped_key_ref: str = Field(description="keyring id, TPM handle, or age key id")
```

Implementation notes:
- Keep `keyring` as default on desktop.
- Add headless-compatible backends (for example TPM-backed wrapping, `pass`, or encrypted file store with strict permissions).
- Keep clear migration/rollback path via config/env backend selector.

### 4) NetworkManager-to-native runtime alternatives (`core/controller.py` connector seam)
Current anchor:
- `Controller.get_vpn_connector()` delegates to `self._api.get_vpn_connector()`.
- `connect()`/`disconnect()` already use connector abstraction and state events.

Migration example:
```python
class ConnectorBackend(Protocol):
    async def connect(self, server): ...
    async def disconnect(self): ...
    @property
    def current_state(self): ...

class NetworkManagerBackend(ConnectorBackend): ...
class WireGuardNativeBackend(ConnectorBackend): ...  # uses wg/ip/resolvectl
class OpenVPNServiceBackend(ConnectorBackend): ...   # uses openvpn/systemd unit

def select_backend(mode, os_caps) -> ConnectorBackend:
    if mode == "desktop":
        return NetworkManagerBackend()
    if os_caps.has_wg_tools:
        return WireGuardNativeBackend()
    return OpenVPNServiceBackend()
```

Native OS tool examples for headless mode:
- WireGuard path: `wg`, `ip`, `resolvectl` (or distro resolver equivalent), `nft`/`iptables`.
- OpenVPN path: `openvpn` + systemd service orchestration.

Compatibility effect:
- Existing NetworkManager path remains default where already supported.
- Non-desktop path is additive and selected by mode/capability detection.

### 5) Packaging evolution without OS support loss
Current anchor:
- Dependency declarations in `setup.py`, Debian control, and RPM spec currently assume keyring dependency in all installs.

Migration example:
- Keep current package outputs.
- Add mode-aware optional dependencies where safe (desktop extras vs headless extras).
- Add install-time/runtime checks that fail with actionable messages if required native tools are missing.

Acceptance additions for this section:
- Every migration item references a concrete current file/function.
- Each new backend path has a fallback preserving current desktop behavior.

## CLI/tooling changes summary
- **Keep:** all existing commands and common usage patterns.
- **Add:** optional non-interactive auth flags/env vars, optional credential backend selector.
- **Adjust internally:** GUI-concurrency check becomes tolerant of absent session DBus.
- **Docs:** update README/command help with “Desktop mode vs Headless mode” guidance.

## OS/platform continuity strategy
To keep existing supported OSes while moving away from headful-only assumptions:
1. Preserve existing Debian and RPM packaging pipelines.
2. Maintain existing dependencies for desktop mode unless explicitly optionalized.
3. Introduce mode-aware dependency/runtime checks rather than global removals.
4. Validate both modes in CI before release.

## Risks and mitigations
- **Risk:** auth/security regression from new non-interactive paths.  
  **Mitigation:** strict input-source validation, redaction, and secure storage defaults.
- **Risk:** hidden dependency on session services in connector stack.  
  **Mitigation:** targeted integration tests with session bus absent.
- **Risk:** packaging drift across distributions.  
  **Mitigation:** keep Debian/RPM parity checks and mode-specific install tests.

## Suggested delivery order
1. Phase 1 (safe DBus fallback)
2. Phase 2 (non-interactive signin)
3. Phase 3 (credential backend abstraction)
4. Phase 4 (connector hardening)
5. Phase 5 + 6 (packaging, CI matrix, rollout)
