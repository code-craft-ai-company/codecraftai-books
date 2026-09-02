# Chapter 3 — `featurebase` and the Per-OS Implementation Pattern

> *The contracts are markdown.  The implementations are
> Python.  The bridge is `featurebase` — a 366-line module in
> `codecraftai-sysman-common` that ships `BaseFeatureManager`,
> `FeatureConfig`, `FeatureActionResult`, and the `_dispatch`
> helper that enforces the contract invariants (dry-run,
> elevation, idempotency, result wrapping).*

The previous chapter walked the contracts.  This chapter
walks the *substrate* — the shared base classes every per-OS
implementation extends.  By the end, a reader should be able
to read any per-OS feature manager and know which behaviours
come from `featurebase` and which are implementation
specifics.

## Why a shared substrate

A reader who's read chapter 1 knows the discipline mechanism
is "shared base classes plus 11 conventions plus per-OS
tests."  This chapter is about the shared base classes.

If `featurebase` didn't exist, every per-OS manager would
re-implement:

- Dry-run handling — checking `config.dry_run` before
  invoking the runner.
- Elevation — passing `elevated=True` to the runner when
  `config.elevated` is set, except for reads.
- Result wrapping — building the `*ActionResult` instance
  with the standard fields (`status`, `command`,
  `exit_code`, `stdout`, `stderr`).
- Status mapping — translating the runner's `RunResult`
  into the contract's status enum.

Multiplied across ~50 features × 5 OSes = ~250 places where
that boilerplate could go subtly wrong.  Centralising the
boilerplate in a single helper means the boilerplate is
correct in ~250 places at once.  When a Phase-3 audit changes
how dry-run is reported, the change happens in
`_dispatch`; every implementation inherits the fix.

## The four canonical types

**File:**
`codecraftai-sysman-common/source/packages/ccai/sysman/common/featurebase.py:113–151`

```python
class ActionStatus(Enum):
    """
        Canonical outcome-status enum for every feature manager in
        this package.
    """
    SUCCESS = "success"
    FAILED  = "failed"
    SKIPPED = "skipped"
    DRY_RUN = "dry_run"


@dataclass
class FeatureConfig:
    """
        Canonical configuration base for every feature manager.
    """
    dry_run: bool = False
    elevated: bool = False


@dataclass
class FeatureActionResult:
    """
        Canonical per-operation result shape.
    """
    status: ActionStatus
    detail: Optional[str] = None
    exit_code: Optional[int] = None
    stdout: Optional[str] = None
    stderr: Optional[str] = None
    command: Optional[list[str]] = None
```

Three types, all canonical.  Per-OS managers subclass each:

- `MacosPackageActionStatus(ActionStatus)` — same four
  values, OS-prefixed name (Convention 4).
- `MacosPackageManagerConfig(FeatureConfig)` — adds
  `brew_user`, `cask_default`, etc.
- `MacosPackageActionResult(FeatureActionResult)` — adds
  `package_name`, `version`, `formula`.

The base types lock down the *common shape*; subclasses add
feature-specific fields.

### `ActionStatus`

Four members.  Every manager in every package returns one
of these four:

- `SUCCESS` — the operation completed as requested.
- `FAILED` — the operation could not be completed.
- `SKIPPED` — the post-state already matched (idempotent
  skip; the runner was not invoked).
- `DRY_RUN` — `dry_run=True`; the runner was not invoked,
  but the would-have-been argv was recorded.

Why these four?  Because they cover every meaningful
outcome of an operation that *might* mutate state:

- The operation *did* mutate state → `SUCCESS`.
- The operation *didn't* mutate state because of an error
  → `FAILED`.
- The operation *didn't* mutate state because it didn't
  need to → `SKIPPED`.
- The operation *didn't* mutate state because the caller
  asked it not to → `DRY_RUN`.

Test code reading the result can dispatch on the status
without parsing exit codes or stderr.  Tests that want to
be tolerant of "it was already configured this way" check
for `status in (ActionStatus.SUCCESS, ActionStatus.SKIPPED)`.
Tests that want strict success-or-fail check
`status == ActionStatus.SUCCESS` only.

### `FeatureConfig`

Two fields by default:

- `dry_run` — when True, the manager records the would-have-
  been command but never invokes the runner.
- `elevated` — when True, write verbs pass `elevated=True`
  to the runner.  Reads ignore this and pass `elevated=False`
  unconditionally.

Per-OS managers subclass to add fields.  The
`MacosPackageManagerConfig` adds `brew_user`.  The
`UbuntuPackageManagerConfig` adds `apt_options` and
`noninteractive_env`.  The `WindowsServiceManagerConfig`
adds nothing — Windows services use the canonical fields
directly.

The two-field minimum is *the* contract config.  Anything
else is per-OS extension.

### `FeatureActionResult`

Six fields:

- `status` — the `ActionStatus` value.
- `detail` — optional human-readable extra context.
- `exit_code` — the runner's exit code (None for SKIPPED
  / DRY_RUN where the runner wasn't invoked).
- `stdout` — captured stdout (or None).
- `stderr` — captured stderr (or None).
- `command` — the argv that was (or would have been)
  invoked.  Per Convention 2: every result records this.

Per-OS subclasses add fields.  `PackageActionResult`
(unprefixed in RHEL) adds `package_name` and `version`.
`UbuntuFirewallActionResult` adds `zone`.  Each addition is
documented in the per-OS spec.

The `command` field is the framework's audit trail.
Diagnostic captures (Volume 2 chapter 6) include action
results in their reports; an engineer reading a failure
record sees the exact argv that was run.

## `BaseFeatureManager`

**File:** `featurebase.py:154–205`

```python
class BaseFeatureManager:
    """
        Shared base for every feature manager.

        Subclasses override the class variables ``result_class`` and
        (rarely) ``status_class`` to plug in feature-specific result
        and status types.  Methods that need to populate extra result
        fields either pass them through as ``**extra`` to
        :meth:`_dispatch` / :meth:`_wrap_result`, or override
        :meth:`_wrap_result` to inject them explicitly.
    """

    # Feature-specific result dataclass.  MUST subclass
    # FeatureActionResult so the standard fields are accepted as
    # keyword arguments.
    result_class: type = FeatureActionResult

    # Feature-specific status enum.  Defaults to ActionStatus;
    # subclasses override only if they extend the base enum.
    status_class: type = ActionStatus

    # Feature name used for future auto-reporting.  Subclasses set
    # this to the canonical spec-folder name.
    feature_name: Optional[str] = None

    def __init__(
        self,
        runner: ICommandRunner,
        config: Optional[FeatureConfig] = None,
    ) -> None:
        """
            :param runner: The command runner used by every mutating
                           operation.
            :param config: Manager configuration.  Defaults to the
                           base FeatureConfig when omitted.
        """
        if config is None:
            config = FeatureConfig()

        self._runner: ICommandRunner = runner
        self._config: FeatureConfig = config

        override_attr = getattr(config, "override_host_os", None)
        normalised_override = normalise_override_host_os(override=override_attr)
        if normalised_override is not None:
            self._host_os: str = normalised_override
        else:
            self._host_os = detect_host_os()
        return
```

Three things to notice:

### Three class variables for subclass override

- **`result_class`** — the subclass-specific result type.
  Default `FeatureActionResult`; per-OS managers override
  with `MacosPackageActionResult`, etc.
- **`status_class`** — the subclass-specific status enum.
  Default `ActionStatus`; OS-prefixed variants set this
  per Convention 4.
- **`feature_name`** — the spec-folder name (e.g.
  `"package-management"`).  Used by the (forthcoming) auto-
  reporting hook from Layer 2's `maintenance-reporting`
  contract.

Subclasses set these as class attributes:

```python
class UbuntuPackageManager(BaseFeatureManager):
    result_class = UbuntuPackageActionResult
    status_class = UbuntuPackageActionStatus
    feature_name = "package-management"

    def install(self, name: str, version: Optional[str] = None) -> UbuntuPackageActionResult:
        # ... implementation ...
```

The subclass doesn't override `__init__`; it inherits the
runner / config / host-OS handling from the base.

### `runner` and `config` parameters

Every manager takes:

- A *command runner* (an `ICommandRunner` instance).  This
  is the seam from Convention 1: every command — and every
  file I/O operation — flows through this runner.
  Test code injects a mock runner; production code injects
  a real one (typically `LocalCommandAgent` from Volume 5
  chapter 3 for local execution, or `SshAgent` for remote).
- A *config* (a subclass of `FeatureConfig`).  Carries
  `dry_run`, `elevated`, and per-OS extensions.

The runner is the *mechanism*; the config is the
*behaviour parameter*.  Manager code calls `self._runner.run(
command, elevated=...)` for the underlying execution.

### Host-OS detection

```python
override_attr = getattr(config, "override_host_os", None)
normalised_override = normalise_override_host_os(override=override_attr)
if normalised_override is not None:
    self._host_os: str = normalised_override
else:
    self._host_os = detect_host_os()
```

The base supports an `override_host_os` config field
(Convention 10).  When the config has this attribute set
(typical for UEFI managers, optional for others), the
manager treats the host as if it were the named OS.  When
absent, the manager calls `detect_host_os()` (which calls
`platform.system()` under the hood).

The override is *testability infrastructure*.  A UEFI
test running on a developer's macOS laptop sets
`override_host_os="linux"` to drive the Linux dispatch
paths through the manager — the manager runs the same code
paths it would on a Linux host, against mock command output.
Volume 4 chapter 8 walks UEFI's use of this in detail.

## The `_dispatch` helper

**File:** `featurebase.py:229–300`

```python
def _dispatch(
    self,
    *,
    command: list[str],
    detail: Optional[str] = None,
    stdin: Optional[str] = None,
    operation: Optional[str] = None,
    elevated_override: Optional[bool] = None,
    **extra,
) -> FeatureActionResult:
    """
        Dispatch a mutating command through the runner, honouring
        dry-run, and map the result to the feature's result type.
    """
    command_copy = list(command)

    # Step 1 — dry-run short-circuit.
    if self._config.dry_run:
        dry_result = self._wrap_result(
            status=self.status_class.DRY_RUN,
            detail=detail,
            command=command_copy,
            **extra,
        )
        return dry_result

    # Step 2 — resolve elevation.
    if elevated_override is None:
        run_elevated = self._config.elevated
    else:
        run_elevated = elevated_override

    # Step 3 — invoke the runner.
    run_result: RunResult = self._runner.run(
        command_copy,
        elevated=run_elevated,
        stdin=stdin,
    )

    # Step 4 — map runner result to manager status.
    if run_result.exit_code == 0:
        mapped_status = self.status_class.SUCCESS
    else:
        mapped_status = self.status_class.FAILED

    # Step 5 — wrap result.
    final_result = self._wrap_result(
        status=mapped_status,
        detail=detail,
        exit_code=run_result.exit_code,
        stdout=run_result.stdout,
        stderr=run_result.stderr,
        command=command_copy,
        **extra,
    )
    return final_result
```

Five steps that *every write verb* delegates to.  The
discipline is locked into one place:

### Step 1 — Dry-run short-circuit

`if self._config.dry_run: return _wrap_result(status=DRY_RUN,
command=command_copy, ...)`.  The runner is *not invoked*.
The would-have-been argv is recorded on the result.  The
caller sees `status == DRY_RUN`.

A test that wants to verify a write verb constructs the
right argv without actually running it sets
`config = FeatureConfig(dry_run=True)`, calls the verb,
and asserts on `result.command`.

### Step 2 — Resolve elevation

The base resolves elevation in priority order:

1. If the caller passed `elevated_override=True` or
   `elevated_override=False`, use that.
2. Otherwise, use `self._config.elevated`.

Reads typically pass `elevated_override=False` to lock
elevation off regardless of config.  Writes typically don't
override and let the config decide.  The Homebrew elevation
inversion (Convention 11) passes `elevated_override=False`
explicitly, regardless of `config.elevated`, because Homebrew
refuses root.

### Step 3 — Invoke the runner

```python
run_result: RunResult = self._runner.run(
    command_copy,
    elevated=run_elevated,
    stdin=stdin,
)
```

Single call to the runner.  The runner is responsible for
the actual subprocess / SSH / WinRM / serial dispatch; the
manager is responsible for choosing the argv and elevation
flag.

The `stdin` parameter is for credential delivery.  Per
Convention 11's credential-delivery rules, secrets *should*
flow via stdin (the runner's `RunResult` doesn't carry the
stdin data, so secrets stay out of the audit trail).  For
example, `luksFormat` routes the passphrase via stdin; the
recorded `command` argv has no secret material.

### Step 4 — Map runner result to manager status

```python
if run_result.exit_code == 0:
    mapped_status = self.status_class.SUCCESS
else:
    mapped_status = self.status_class.FAILED
```

The default mapping is *exit-code-driven*: zero means
success, non-zero means failure.

A few features need a different mapping.  For example,
`dnf check-update` returns 100 to signal "updates are
available."  In that case, the per-OS manager *doesn't*
delegate to `_dispatch` for the affected verb; it calls
the runner directly and applies the feature-specific status
mapping.  The base helper covers the common case; the
exceptional case is per-OS.

### Step 5 — Wrap result

```python
final_result = self._wrap_result(
    status=mapped_status,
    detail=detail,
    exit_code=run_result.exit_code,
    stdout=run_result.stdout,
    stderr=run_result.stderr,
    command=command_copy,
    **extra,
)
return final_result
```

`_wrap_result` constructs the `*ActionResult` instance with
the standard fields plus any feature-specific extras
passed via `**extra`.

The per-OS manager calls `_dispatch(command=[...], detail="nginx",
package_name="nginx", version="1.21")` and the
`package_name` / `version` extras flow through to the
result.  No subclass needs to override `_wrap_result` for
the common case; only features with unusual result shapes
override.

## Idempotency — the manager's job

A point worth being explicit about: `_dispatch` does NOT
implement idempotency on its own.  Per the contracts, every
write must be idempotent — but the *check* for idempotency
is per-feature.

A typical write-verb implementation:

```python
def install(self, name: str, version: Optional[str] = None) -> PackageActionResult:
    # Step 1 — check idempotency.
    existing = self.query(name)
    if existing is not None:
        if version is None or existing.version == version:
            return PackageActionResult(
                status=PackageActionStatus.SKIPPED,
                package_name=name,
                version=existing.version,
            )

    # Step 2 — construct the command.
    target = name
    if version is not None:
        target = "{}-{}".format(name, version)
    command = [_DNF, "install", "-y", target]

    # Step 3 — delegate to _dispatch for execution.
    return self._dispatch(
        command=command,
        detail=name,
        operation="install",
        package_name=name,
        version=version,
    )
```

Three steps:

1. **Idempotency check** — the manager queries the current
   state via `self.query(name)` (a Read verb).  If the
   target state already matches, return `SKIPPED` without
   calling `_dispatch`.
2. **Command construction** — the manager builds the
   OS-specific argv.  Per-OS variation lives here.
3. **`_dispatch` delegation** — pass the argv to the helper
   for actual execution and result wrapping.

Step 1's logic is per-feature.  Some features have trivial
idempotency (a service that's already running is `SKIPPED`).
Others have nuanced idempotency (a firewall rule is a
member of a zone; checking membership is OS-specific).  The
contract specifies *what* idempotency means; the manager's
code implements *how* to check it for the OS.

## What `_dispatch` *doesn't* do

A short list of things `_dispatch` deliberately leaves to
the manager:

### Custom status mapping

Verbs that need exit-code mappings other than zero=SUCCESS
(`dnf check-update` returning 100, etc.) must implement
their own status logic.  `_dispatch` covers the simple case;
custom cases bypass it.

### Output parsing

`_dispatch` captures stdout and stderr but doesn't parse
them.  The manager parses (e.g.,
`UbuntuPackageManager.list_installed` parses `dpkg-query`
output into `PackageInfo` records).

### Per-OS pre-call setup

E.g., Ubuntu's package manager sets
`DEBIAN_FRONTEND=noninteractive` in the env before calling
apt-get.  That setup happens in the manager's verb code
before `_dispatch`; `_dispatch` doesn't know about it.

### Multi-step operations

A verb that needs multiple commands in sequence (e.g.,
"create LVM physical volume, then volume group, then logical
volume") makes multiple `_dispatch` calls and aggregates the
results into one `*ActionResult`.

## When the substrate becomes the implementation

`featurebase` is deliberately thin: it locks the invariants
and leaves every verb to the per-OS manager.  Two feature
bases in the same package sit one level up from it, and they
answer opposite questions about how much can be shared.

**File:**
`codecraftai-sysman-common/source/packages/ccai/sysman/common/vpnclientmanagementbase.py`

`VPNClientManagementBase` is *abstract*.  It declares the
seven `vpn-client-management` verbs as `@abstractmethod`s and
ships the helpers every OS needs (`_validate_provider`,
`_short_circuit_dry_run`, `_should_elevate`,
`_dispatch_write`).  The verbs stay per-OS because installing
Tailscale means an apt repository on Ubuntu, a dnf
repository on RHEL, a Homebrew cask on macOS, and an MSI on
Windows.  The base fixes the *shape*; the OS supplies the
*behaviour*.

**File:**
`codecraftai-sysman-common/source/packages/ccai/sysman/common/pythonenvironmentmanagementbase.py`

`PythonEnvironmentManagementBase` is *concrete*.  Every
`python-environment-management` verb — `create_environment`,
`install_packages`, `deploy`, the read surface — is
implemented in the base, because `python -m venv` and
`python -m pip` are the same program on every OS.  The one
abstract method is `_build_layout()`, which returns a
`PythonEnvironmentLayout` from the sibling module
`pythonenvironmentlayout.py`:

```python
class PythonEnvironmentLayout(ABC):
    def default_interpreter_argv(self) -> list[str]: ...   # ["python3"] / ["py", "-3"]
    def venv_python_path(self, path: str) -> str: ...      # bin/python / Scripts\python.exe
    def build_directory_exists_argv(self, path) -> list[str]: ...   # test -d / Test-Path
    def build_remove_tree_argv(self, path) -> list[str]: ...        # rm -rf / Remove-Item
```

A per-OS subclass is then about 200 lines: pick the layout,
alias the shared types with the OS prefix, and add
`ensure_prerequisites`, which delegates to that OS's existing
package manager.

Two details of the concrete base are worth noticing because
they show where `_dispatch` stops being enough:

- It calls the runner directly in its own `_dispatch_write`
  rather than through `BaseFeatureManager._dispatch`, because
  it needs the raw `RunResult` to read the Windows
  `elevation_blocked` extension (Convention 6) and surface it
  as `detail="elevation_blocked"`.  The base `_dispatch`
  discards the raw result after mapping status.  This is the
  "custom status mapping" escape hatch from the previous
  section, used for a cross-OS reason rather than a per-OS
  one.
- Its idempotency probe for `install_packages` is a
  conservative PEP 508 evaluator (`_requirement_satisfied`)
  that refuses to guess: pre-release versions, URL and
  editable requirements, and non-numeric ordering comparisons
  all answer "not satisfied" and let pip decide.  A false
  "not satisfied" costs one redundant pip call; a false
  "satisfied" would silently skip a dependency.

The rule the two bases illustrate: share the *shape* by
default, and share the *behaviour* only when the underlying
tool is provably identical across OSes.  The VPN base is the
default; the python-environment base is the justified
exception.

## What this chapter set up

A reader who's read this chapter should:

- Recognise `BaseFeatureManager`, `FeatureConfig`,
  `FeatureActionResult`, `ActionStatus` as the framework's
  canonical types for the sysman tier.
- Be able to read any per-OS manager and identify which
  behaviours come from the base (dispatch, dry-run,
  elevation, result wrapping) and which are
  implementation-specific (idempotency check, command
  construction, output parsing).
- Understand `override_host_os` as testability
  infrastructure (Convention 10).
- See the `_dispatch` helper as the *concrete locking
  mechanism* for the contract invariants — every write
  verb that uses it follows the contract by construction.
- Tell an abstract feature base (`VPNClientManagementBase`)
  from a concrete one (`PythonEnvironmentManagementBase`) and
  know which question — "is the tool the same on every OS?"
  — decides between them.

The next five chapters walk the per-OS implementations, one
per OS.  Each chapter shows what the OS contributes
specifically — RHEL's 47-feature registry, Ubuntu's
shared-base pattern, macOS's quirks, Windows' PowerShell
JSON serialisation, UEFI's single-tier dispatch.

## What's next

Chapter 4 walks the RHEL implementation
(`codecraftai-sysman-linux-redhat`) — the historical reference
that established the patterns.  RHEL retains unprefixed
class names (the documented Convention 4 exception) and
ships a 47-feature registry that's the closest thing to a
"complete" sysman implementation in the portfolio.
