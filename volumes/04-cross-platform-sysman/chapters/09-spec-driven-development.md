# Chapter 9 — Spec-Driven Development for System Administration

> *Eight chapters walked the contracts and the
> implementations.  This chapter walks the **discipline** that
> keeps them aligned: the 11 Implementation Conventions
> documented at
> `automojo-sysman-common/specs/contracts/README.md:75–364`.
> The conventions are what prevent drift across 5 OSes ×
> ~50 features without a code generator.*

The volume's previous chapters referenced the conventions
piecemeal — Convention 4 (OS-prefixed names), Convention 5
(FAILED with stable token), Convention 11 (Homebrew
elevation inversion).  This chapter walks all eleven as a
*unified* discipline.  By the end, a reader writing a new
per-OS implementation knows the rules in detail.

The conventions emerged from real implementation work —
specifically the Phase-3 cross-OS push that brought Ubuntu,
macOS, and Windows online alongside the original RHEL
package.  Each convention is a *crystallisation* of a
problem the implementation work uncovered.

## Why conventions instead of code

A reader from a heavily-tooled engineering culture might
ask: *if these are uniform rules across all packages, why
not enforce them in code via metaclasses, decorators, or a
linter?*

Three reasons:

### Many of the conventions are *naming*

Convention 4 (OS-prefixed type names) is enforceable by a
linter.  But the discipline isn't *just* the prefix — it's
that the prefix appears consistently across types
(`UbuntuPackageManager`, `UbuntuPackageActionResult`,
`UbuntuPackageActionStatus`).  A linter that just checks
class-name prefixes misses the cross-type alignment.

### Many of the conventions are *trade-offs*

Convention 7 (reads degrade gracefully — return None /
empty / synthetic record vs. raise) gives implementations
*choice*.  The discipline is that the implementation picks
*one* and is *consistent*.  Code can't enforce "be
consistent in your choice."

### The conventions evolve

Phase-3 added Convention 6 (elevation_blocked).  A future
phase might add Convention 12.  Hard-coding the conventions
in framework code makes them harder to evolve; documenting
them in prose lets them grow.

## The eleven conventions

### Convention 1 — `ICommandRunner` is the file-I/O seam

> *File I/O MAY use `IFileOperator` if the host package
> provides one; the established pattern is to route file I/O
> through `ICommandRunner` using OS-native file utilities.*

The framework has *no separate file-operator abstraction*.
Reading, writing, removing files all goes through the same
`ICommandRunner` that runs commands.  On Linux, that means
invoking `cat` / `tee` / `rm` / `mv`.  On Windows, it means
invoking `Get-Content` / `Set-Content` / `Remove-Item`.

The convention exists because:

- A separate `IFileOperator` would duplicate concerns the
  runner already handles (elevation, dry-run, audit
  recording).
- Routing file I/O through the runner means the *audit
  trail* (every action records its argv per Convention 2)
  works for file operations too.
- Tests that mock the runner automatically also mock the
  file operations.

A worked example.  When a feature needs to write a config
file:

```python
# Wrong — bypasses the runner.
with open("/etc/myservice.conf", "w") as f:
    f.write(content)

# Right — routes through the runner per Convention 1.
self._runner.run(
    ["tee", "/etc/myservice.conf"],
    elevated=True,
    stdin=content,
)
```

The runner-routed version:

- Goes through the manager's elevation logic.
- Captures the argv in the action result.
- Honours dry-run (the runner doesn't invoke when
  `dry_run=True`).
- Routes through SSH when the runner is an `SshAgent` (so
  the file write happens on the *remote* host, not the
  controller's filesystem).

### Convention 2 — `command: list[str] | None` field on action results

> *Every per-OS `*ActionResult` dataclass carries a
> `command` field that records the argv that was (or would
> have been) executed.*

The `command` field on every result enables:

- **Audit logging** — what changed and how.
- **Dry-run inspection** — see the would-have-been argv.
- **Test verification** — assert on the constructed command.
- **Forensic debugging** — when a test fails, the command
  field tells you exactly what ran.

`featurebase._dispatch` populates this automatically.  A
manager that doesn't use `_dispatch` (chapter 4's
`PackageManager` example) has to populate it manually:

```python
result = PackageActionResult(
    status=PackageActionStatus.SUCCESS,
    package_name=name,
    version=installed_version,
    command=command_copy,  # Convention 2
    exit_code=run_result.exit_code,
    stdout=run_result.stdout,
    stderr=run_result.stderr,
)
```

The field is `Optional[list[str]]` because `SKIPPED`
results don't have a command (no command was needed).

### Convention 3 — `requires_reboot: bool` on action results that mutate boot-time-only state

> *Features that mutate boot-time-only state (Windows
> hostname rename, UEFI capsule-update) carry a
> `requires_reboot: bool = False` field set to `True` when
> the underlying tool reports the change was staged but a
> reboot is required.*

Some operations *succeed* in the sense that the tool
accepted the change, but the change *doesn't take effect
until reboot*.  The result needs to carry both signals:
"yes, this succeeded" and "but the system needs to reboot."

Worked example: Windows hostname rename.

```python
# WindowsHostnameManager.rename_computer

def rename_computer(self, new_name: str) -> WindowsHostnameActionResult:
    # ... build PowerShell Rename-Computer argv ...
    result = self._dispatch(
        command=ps_argv,
        new_name=new_name,
    )
    # Rename-Computer reports success but the new name
    # isn't visible until reboot.
    if result.status == WindowsHostnameActionStatus.SUCCESS:
        result.requires_reboot = True
    return result
```

The test calling `rename_computer` checks
`result.requires_reboot` and either schedules the reboot or
flags the test as needing a manual restart.

### Convention 4 — OS-prefixed concrete type names

Already covered extensively (chapters 4 and 5).  Recap:

- Per-OS implementations prefix every public type:
  `UbuntuPackageManager`, `MacosFirewallManager`,
  `WindowsServiceManager`.
- RHEL retains unprefixed names (`PackageManager`, etc.)
  for historical reasons; this is the documented exception.
- The prefix applies to managers, configs, results,
  statuses, info classes — every public type the package
  exposes.

### Convention 5 — `FAILED` with a stable detail token for unsupported `(host_os, verb)` combos

> *When a feature has no scriptable implementation on a
> given host OS, the manager returns `*ActionResult` with
> `status=FAILED` and `detail` set to a machine-readable
> token like `unsupported_on_host_os_darwin`.*

This replaces the older "raise `NotImplementedError`"
pattern.  The convention's value:

- The contract verb is *reachable* from any host.
- The result is *machine-discoverable* — test code checks
  `result.detail` for known tokens.
- The error path is *uniform* with other failures —
  `status=FAILED` with a structured `detail`.

Stable tokens documented in the contracts.  Examples:

| Token | Meaning |
|---|---|
| `unsupported_on_host_os_darwin` | The feature has no macOS implementation. |
| `unsupported_on_host_os_windows` | No Windows implementation. |
| `ufw_no_zones` | ufw doesn't support multiple zones. |
| `unsupported_no_mok_concept_macos` | macOS has no MOK equivalent. |
| `unsupported_runtime_change_macos` | The change requires recovery mode. |

A test checking for *any* unsupported case can match the
`unsupported_*` prefix.  Tests checking for a *specific*
case match the full token.

### Convention 6 — `elevation_blocked: bool` on Windows `RunResult`

Already covered in chapter 7.  Recap:

- Windows can't drive UAC non-interactively.
- The Windows runner pre-detects elevation status.
- When a manager requests `elevated=True` and the calling
  process isn't elevated, the runner returns a
  `RunResult` with `elevation_blocked=True`.
- Manager code propagates the flag onto the
  `*ActionResult`.
- Test code checks `result.elevation_blocked` and skips
  with a clear reason.

### Convention 7 — Reads degrade gracefully

> *Read methods MAY raise `*QueryError` OR return a None /
> empty / synthetic record.  Whichever the per-OS spec
> picks, it MUST be consistent and documented.*

The convention gives implementations *flexibility* on
read-failure semantics but *requires consistency* within an
implementation.

Two acceptable patterns:

#### Raise-on-error

```python
def get_package_info(self, name: str) -> PackageInfo:
    """
        Returns PackageInfo for the named package.

        :raises PackageQueryError: if the package isn't installed.
    """
    info = self._query(name)
    if info is None:
        raise PackageQueryError(f"Package '{name}' is not installed")
    return info
```

#### Return-None-on-not-found

```python
def get_package_info(self, name: str) -> Optional[PackageInfo]:
    """
        Returns PackageInfo for the named package, or None if
        not installed.
    """
    info = self._query(name)
    return info  # may be None
```

The **discipline** is that *all reads in a single
implementation* pick the same pattern.  A package whose
some reads raise and some return None is broken.

The per-OS spec documents which pattern the package uses.
Test code is written against the documented choice.

### Convention 8 — Per-OS backend enums

> *Features ship a single abstract concept on the contract
> (e.g., `BackupKind`, `VirtBackend`) that diverges into
> OS-specific enums in practice.*

Some features have an *abstract concept* (a backup kind,
a virtualisation backend) that has *different concrete
options* per OS.  The contract defines the abstract concept;
the per-OS implementation defines its own enum.

Worked example: `virtualization-management` ships with a
`VirtBackend` abstract enum on the contract, but each per-OS
implementation has its own:

```python
class UbuntuVirtBackend(Enum):
    LIBVIRT_KVM = "libvirt-kvm"
    LIBVIRT_LXC = "libvirt-lxc"
    LXC = "lxc"
    QEMU = "qemu"


class MacosVirtBackend(Enum):
    HYPERVISOR_FRAMEWORK = "hypervisor-framework"
    VIRTUALIZATION_FRAMEWORK = "virtualization-framework"
    PARALLELS = "parallels"
    VMWARE_FUSION = "vmware-fusion"


class WindowsVirtBackend(Enum):
    HYPER_V = "hyper-v"
    VMWARE_WORKSTATION = "vmware-workstation"
    VIRTUALBOX = "virtualbox"
```

Cross-OS test code that needs to handle the
`VirtBackend` argument has per-OS dispatch:

```python
if isinstance(backend, UbuntuVirtBackend):
    # Ubuntu-specific handling
    ...
elif isinstance(backend, MacosVirtBackend):
    # macOS-specific handling
    ...
```

The convention acknowledges that *some abstractions don't
collapse cleanly* and provides a structured way to handle
them.

### Convention 9 — Concrete-implementation naming overrides

> *The contract uses a clean abstract name (`SystemIdentity`,
> `MailTransportConfig`) while the per-OS implementation
> uses a name that fits its package
> (`UbuntuHostIdentity`, `UbuntuMailTransportConfig`).*

Convention 4 prefixes the *implementation* types.  Convention
9 covers the case where the *contract* type name doesn't
fit the per-OS package's terminology.

Worked example: the contract has a type called
`SystemIdentity` describing the host's identity profile.
The Ubuntu implementation calls its corresponding type
`UbuntuHostIdentity` because Ubuntu's terminology calls the
machine a *host*, not a *system*.

The convention allows this divergence with *cross-OS
adapter layers* translating at the boundary.  A test that
needs to use the contract's `SystemIdentity` against
multiple OSes either:

- Writes per-OS code that uses each OS's preferred name.
- Uses an adapter function that converts between the
  contract type and the implementation type.

### Convention 10 — UEFI managers carry `override_host_os`

Already covered in chapter 8.  Recap:

- UEFI managers detect host OS at construction.
- The `override_host_os: str | None = None` config field
  forces a specific dispatch path for testing.
- This is testability infrastructure unique to UEFI's
  single-tier dispatch architecture.

### Convention 11 — Elevation-model inversion

Already covered in chapters 6 and 9.  Recap:

- Some tools (Homebrew on macOS) refuse to run as root.
- The manager passes `elevated=False` unconditionally.
- When the calling process *is* root, the manager wraps
  the argv with `sudo -u <non_root_user>`.
- The pattern is documented per-feature in the
  Implementation Drift section of the contract.

## Credential-delivery conventions

In addition to the eleven implementation conventions,
the contracts document **credential-delivery conventions**
at lines 299–328 of the README.

The three patterns, in order of preference:

### 1. Stdin (preferred)

```python
result = self._dispatch(
    command=["luksFormat", device_path],
    stdin=passphrase,  # secret stays out of argv
    detail=device_path,
)
```

Secrets flow through stdin.  The `RunResult` doesn't carry
the stdin data, so the `command` field never contains the
secret.  The audit trail records the argv (`["luksFormat",
"/dev/sdb"]`) without leaking the passphrase.

### 2. In-process secure-string (Windows)

```powershell
$securePassword = ConvertTo-SecureString -String $env:PWD -AsPlainText -Force
Set-LocalUser -Name $username -Password $securePassword
```

PowerShell has SecureString — an in-memory representation
that's not paged to disk and not visible in the process
listing.  Windows managers route credentials through
SecureString when the underlying cmdlet supports it.

### 3. Argv with redaction (last resort)

```python
# Some tools accept credentials only on argv.
command = ["smbpasswd", "-a", username, "-p", password]
result = self._dispatch(
    command=command,
    detail=username,
    redact_argv_indices=[4],  # mark the password slot for redaction
)
```

When a tool only accepts credentials on argv, the manager
*marks* the secret-bearing positions so the action
result's `command` field has the secret redacted before
recording.  The framework's argv-redaction logic replaces
the marked slots with `"[REDACTED]"` in the recorded
command.

The convention prefers stdin → SecureString → argv-with-
redaction.  Argv-with-redaction is acknowledged as the
fallback for tools that won't accept the better patterns.

## How the conventions are *enforced*

The conventions aren't enforced by tooling.  They're
enforced by:

### Code review

Every PR that touches a sysman package goes through review.
Reviewers cross-reference the change against the
conventions document.  A convention violation is a review
comment.

### Per-OS test coverage

The contract's "Tests" section prescribes minimum tests.
Tests catch convention violations indirectly:

- A manager that doesn't populate `command` (Convention 2)
  fails its dry-run test (the test asserts
  `result.command == [...]`).
- A manager that doesn't honour `dry_run` (Convention 1's
  scope) fails its dry-run test.
- A manager that elevates reads (against the contract)
  fails its elevation test.

Tests catch the *symptoms* of convention violations even
when the convention isn't tested directly.

### Cross-OS comparisons

When two OSes implement the same feature, comparing their
tests catches drift.  An RHEL `PackageManager.install` test
that's structurally different from a `UbuntuPackageManager.install`
test is a sign one of them is wrong.

## What the conventions give you

A reader who understands the eleven conventions can:

- Read any sysman feature manager and know what to expect.
- Add a new feature to an existing OS by following the
  conventions.
- Add a new OS implementation by following the conventions
  + the contracts.
- Spot convention violations during code review.
- Write tests that exercise the convention's behaviour.

The conventions *are* the discipline.  The framework's
50 contracts × 5 OSes = ~250 implementations stay aligned
because the conventions are documented, reviewed, and
tested.

## What's next

Chapter 10 closes the volume by walking cross-OS test
parity — how the suite achieves consistent green-across-OSes
test runs at the scale of thousands of tests.  The chapter
explains how the contracts + featurebase + 11 conventions
all compose to make a multi-OS test campaign tractable.
