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
from dbus_fast.aio import MessageBus
from dbus_fast import BusType
from dbus_fast.errors import DBusError

async def _vpn_gui_running() -> bool:
    try:
        bus = await MessageBus(bus_type=BusType.SESSION).connect()
    except (DBusError, OSError):  # session bus not available in headless mode
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
import getpass
import os
import sys

@click.argument("username")
@click.option("--password-stdin", is_flag=True)
@click.option("--password-env", type=str)
@click.option("--otp-env", type=str)
async def signin(ctx, username, password_stdin, password_env, otp_env):
    get_password = select_password_provider(password_stdin, password_env)
    get_2fa = select_otp_provider(otp_env)  # falls back to interactive prompt
    await controller.login(username, get_password, get_2fa)

def select_password_provider(password_stdin, password_env):
    if password_stdin:
        return lambda: _read_stdin_password()
    if password_env:
        return lambda: os.environ[password_env]
    return getpass.getpass

def _read_stdin_password():
    value = sys.stdin.readline().rstrip("\n")
    if not value:
        raise click.ClickException("No password received on stdin")
    return value

def select_otp_provider(otp_env):
    if otp_env:
        return lambda: os.environ[otp_env]
    return lambda: getpass.getpass("2FA Token: ")
```

Security note: this is a migration sketch. Production implementation should use hardened secret input handling, avoid logging/echoing sensitive values, and document automation risks for stdin/env-based credentials.

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
    backend: Literal["keyring", "pass", "file+age", "tpm2", "memory"]
    secret_dir: Path | None = None
    tpm2_key_handle: str | None = None

    @model_validator(mode="after")
    def validate_backend_requirements(self):
        if self.backend in {"pass", "file+age"} and self.secret_dir is None:
            raise ValueError("secret_dir is required for file-backed secret stores")
        if self.backend == "tpm2" and self.tpm2_key_handle is None:
            raise ValueError("tpm2_key_handle is required for TPM2-backed store")
        return self

class SecretEnvelope(BaseModel):
    account: str
    ciphertext_b64: str
    wrapped_key_ref: str = Field(
        description="backend reference: keyring item id, pass entry, age key id, tpm2 handle, or memory session id"
    )
```

Clarifications:
- `file+age` denotes one backend strategy (filesystem storage encrypted with age: https://github.com/FiloSottile/age), not two independent backend selectors.
- `pass` refers to the standard Unix password manager (`pass`) as the secret material storage backend.
- For `tpm2`, store sealed blobs/metadata in `secret_dir` as well; validate `secret_dir` when blob persistence is enabled.
- `tpm2_key_handle` should be documented as a stable TPM2 object identifier (for example persistent handle format used by local tooling).
- `memory` backend should be restricted to ephemeral sessions (for example CI smoke tests), with explicit warning that secrets are lost on restart and must never be persisted.

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
- OpenVPN path: `openvpn` + service orchestration (systemd where available; OpenRC/runit/supervisord alternatives otherwise).
- Resolver fallback (non-systemd): write managed `resolv.conf` via `openresolv`/`resolvconf` integration where `resolvectl` is unavailable.
- Resolver ownership must be coordinated with active network managers (`dhclient`, NetworkManager, or equivalent) to avoid DNS race conditions.

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

## Security officer review gates (secure + future-proof)
Before rollout, security sign-off should require the following controls:

1. **Secret lifecycle controls**
   - No plaintext credential persistence in logs, config, process args, or crash dumps.
   - In-memory secret lifetime minimized (zeroize buffers where practical).
   - File-backed secrets require strict permissions (owner-only) and at-rest encryption.

2. **Auth path hardening**
   - Non-interactive auth sources (`stdin`, env, token flows) must be mutually exclusive or explicitly prioritized.
   - Mandatory redaction policy for all sensitive CLI output and telemetry.
   - Brute-force/rate-limit handling aligned with upstream API policy.

3. **Backend trust model**
   - Backend-specific threat model documented (`keyring`, `pass`, `file+age`, `tpm2`, `memory`).
   - `memory` backend explicitly non-persistent and disabled by default outside controlled environments.
   - `tpm2` path includes key rotation and recovery procedure documentation.

4. **Supply-chain and crypto agility**
   - Pin and monitor security advisories for crypto/key-management dependencies.
   - Keep backend interface versioned so storage/crypto implementation can evolve without CLI breaking changes.
   - Require migration tooling for future cryptographic algorithm upgrades.

5. **Operational safety**
   - Safe failure defaults (failed secret backend init must not silently downgrade to insecure storage).
   - Resolver/service reconfiguration must be transactional with rollback on failure.
   - Security incident runbook documented for credential leak, backend corruption, and auth abuse scenarios.

Security acceptance criteria:
- Security review checklist completed and approved for each backend/mode.
- Negative security tests pass (secret leakage, unsafe fallback, privilege boundary checks).
- No critical/high unresolved security findings at release gate.

## TDS expert review: required test definition and pass criteria
The TDS review should produce/own this minimum test matrix for the headless migration:

1. **Compatibility tests**
   - Existing desktop flows unchanged (`signin`, `connect`, `status`, `disconnect`, `signout`).
   - Help/usage text and existing flags remain backward compatible.
   - Packaging install/upgrade parity on supported Debian/RPM targets.

2. **Headless runtime tests**
   - No `DBUS_SESSION_BUS_ADDRESS` scenario: CLI starts and headless commands execute.
   - Missing GUI/session services do not block non-GUI operations.
   - Connector selection behaves deterministically by mode/capability.

3. **Auth and secret-storage tests**
   - Interactive signin and each non-interactive auth mode pass independently.
   - Invalid/ambiguous auth input combinations fail with safe, clear errors.
   - Backend matrix tests (`keyring`, `pass`, `file+age`, `tpm2`, `memory`) validate persistence/security expectations.

4. **Network/runtime backend tests**
   - NetworkManager path regression tests.
   - Native tool path tests (WireGuard/OpenVPN) including resolver ownership conflicts.
   - Service-manager variation tests (systemd + non-systemd supervisor path).

5. **Security and resilience tests**
   - Secret redaction tests for stdout/stderr/logging.
   - Failure-injection tests for backend init, resolver apply, and partial connect/disconnect.
   - Rollback integrity tests (no orphaned routes/firewall/DNS state after failure).

6. **CI gating policy**
   - Define required vs optional jobs per phase (PR vs nightly).
   - Block merge on: compatibility regressions, security test failures, or unresolved flaky-critical tests.
   - Require trend monitoring for reliability (pass rate, flake rate, mean recovery time).

TDS acceptance criteria:
- Test specification reviewed and approved by TDS owner.
- Automated coverage exists for all critical paths above.
- Release candidate must pass the agreed required-gate suite across supported OS/package variants.

## Web UI and API extension assurance gates (frontend + backend)
For the requested web control-plane extension, add two dedicated ownership gates with concrete implementation examples.

### Frontend developer assurance (consistency + UX)
Required outcomes:
- One source of truth for command/field semantics so web forms match CLI behavior and validation.
- Stable UX patterns for connect/disconnect/status/auth flows across desktop and headless contexts.
- Accessibility, localization, and error-message parity with CLI guidance.

Code-grounded example (generate UI-safe schemas from CLI-facing models):
```python
# shared/contracts.py
from pydantic import BaseModel, Field
from typing import Literal

class ConnectRequest(BaseModel):
    profile: str = Field(
        min_length=1,
        description="Target selector accepted by CLI semantics (for example profile name, country code, or explicit server name)",
    )
    protocol: Literal["wireguard", "openvpn"] = "wireguard"
    netshield: Literal["off", "malware", "ads_malware"] = Field(
        default="off",
        description="NetShield mode: off, malware-only, or ads+malware",
    )
```

```ts
// web/ui consumes generated JSON schema for form validation + labels
// (schema generation done in build step from shared/contracts.py)
```

Frontend acceptance criteria:
- UX spec approved by frontend owner for all core flows.
- Web validation rules remain synchronized with backend contract versions.
- Usability/a11y regressions block release.

### Backend developer assurance (CLI parity + maintainable API evolution)
Required outcomes:
- API operations map to existing controller seams so web functionality stays complete vs CLI.
- API versioning/deprecation rules prevent breaking clients unexpectedly.
- Contract tests guarantee parity between API calls and CLI command outcomes.

Code-grounded example (service layer calling existing controller paths):
```python
# proton/vpn/cli/web/service.py
class VpnService:
    def __init__(self, controller):
        self._controller = controller

    async def connect(self, request):
        settings = self._controller.get_connection_settings()
        # map request -> existing controller logic, do not duplicate business rules
        server_name = request.profile
        # Note: current controller signature uses `servername` (legacy naming).
        return await self._controller.connect(servername=server_name, connection_type=None)
```

Backend acceptance criteria:
- Endpoint-to-controller mapping documented for each core capability.
- Deprecation policy enforced (`/v1` preserved while `/v2` rolls out).
- Contract + parity tests pass for all CLI-equivalent actions.

## Full REST compatibility and evolution policy
- Provide complete REST coverage for CLI-complete operations: auth/session, connect/disconnect/reconnect, status, settings, server selection, and signout.
- Use explicit API versioning (`/api/v1/...`) and additive-change-first policy.
- Breaking changes require:
  1) new version namespace,
  2) compatibility shim window,
  3) migration notes and automated compatibility tests.
- Publish machine-readable OpenAPI spec and run spec-drift checks in CI.

## Local API exposure security (`127.0.0.10` + `/etc/hosts`)
If binding a local port (initially `127.0.0.10`) and adding `protonvpn` host alias:

1. **Bind and trust boundary**
   - Bind only to loopback, never wildcard interfaces.
   - Enforce Host-header allowlist (`protonvpn`, `127.0.0.10`, `localhost` as explicitly configured).
   - Disable proxy trust by default to prevent header spoofing.

2. **Authentication and authorization**
   - Require local auth token (or mTLS over loopback) for all mutating endpoints.
   - Use least-privilege scopes (read status vs modify connection state).
   - CSRF protections required for browser-based session auth.

3. **`/etc/hosts` integrity**
   - Write only exact managed line (`127.0.0.10 protonvpn`) with idempotent updater.
   - Require elevated privileges with explicit user confirmation.
   - Keep backup + rollback; never remove unrelated host entries.

4. **Operational controls**
   - Audit log all state-changing API calls.
   - Rate-limit sensitive endpoints.
   - Default deny on auth failure and lock down debug endpoints in production mode.

Security acceptance criteria:
- No unauthenticated state-changing endpoint is exposed on the local API.
- Host alias updates are idempotent, reversible, and integrity-checked.
- Pen-test/local-threat-model review passes before default enablement.

## MCP endpoint for local AI integration (optional, gated)
Add an optional MCP endpoint only after REST security gates are green.

Code-grounded example (tool layer delegates to same service/API contract):
```python
class McpTools:
    async def vpn_status(self):
        return await self._service.status()

    async def vpn_connect(self, profile: str):
        return await self._service.connect(profile=profile)
```

Required controls:
- Reuse same authz scopes as REST API (no privileged bypass path).
- Tool allowlist (no arbitrary command execution).
- Full audit trail for AI-triggered actions with caller identity/session correlation.
- Admin-configurable kill switch for MCP endpoint.

## Suggested delivery order
1. Phase 1 (safe DBus fallback)
2. Phase 2 (non-interactive signin)
3. Phase 3 (credential backend abstraction)
4. Phase 4 (connector hardening)
5. Phase 5 + 6 (packaging, CI matrix, rollout)
