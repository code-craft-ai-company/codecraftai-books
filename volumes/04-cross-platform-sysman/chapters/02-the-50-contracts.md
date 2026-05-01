# Chapter 2 — The 50 Cross-OS Contracts

> *The contracts are markdown.  Fifty directories under
> `automojo-sysman-common/specs/contracts/`, each containing
> one `contract.md` file with a fixed structure: Scope, Verbs,
> Invariants, Data Models, Channel/Source Mapping, Elevation,
> Execution Model, Naming, Aspect Composition, Tests, Out of
> Scope, Implementation Drift.  This chapter walks the
> structure and three worked examples.*

The previous chapter argued that a contract layer is the
*right shape* for cross-OS automation.  This chapter walks
the contracts themselves.  By the end, a reader should be
able to open any of the 50 contracts and find what they need
within the structure.

## Where the contracts live

```
automojo-sysman-common/
└── specs/
    └── contracts/
        ├── README.md                          # 361-line master index
        ├── package-management/contract.md
        ├── service-management/contract.md
        ├── network-configuration/contract.md
        ├── firewall-management/contract.md
        ├── encryption-management/contract.md
        ├── ... (45 more)
```

50 directories.  Each directory is a feature area.  Each
holds exactly one `contract.md` file.  No feature has its
contract spread across multiple files; no directory has
contracts for multiple features.

The README at the directory root is the master index.  Its
catalogue (lines 385–510) lists every contract with:

- **Feature name** — e.g. `package-management`.
- **Contract path** — relative path to `contract.md`.
- **Scope parameter** — what the contract covers in one
  sentence.

A reader looking for a specific feature opens the README,
finds the row, follows the path.

## The three layers

Per chapter 1, the 50 contracts split into three layers:

### Layer 1 — Infrastructure (1 contract)

- **`command-execution`** — the `ICommandRunner` contract
  that every other feature depends on.  Specifies how
  commands are dispatched, how results are returned, how
  elevation is requested, how dry-run is signalled.

Every feature manager runs commands through an
`ICommandRunner` (per Convention 1: file I/O is also routed
through this same seam).  The `command-execution` contract
locks the `ICommandRunner` interface so all higher contracts
can rely on it.

### Layer 2 — Data (1 contract)

- **`maintenance-reporting`** — cross-feature reporting
  vocabulary.  When a manager performs a write that's worth
  recording in an audit log (a service start, a package
  install, a firewall rule add), the report follows this
  contract's shape.

The contract exists separately from the feature contracts
because *every* feature reports through the same shape.
Centralising the report model keeps consumers (audit log
writers, observability dashboards, time-travel debuggers
from Volume 8) consistent.

### Layer 3 — Business logic (48 contracts)

The bulk of the suite.  Each contract is one feature area:

```
acl-management              kdumpmanagement
audit-management            kernel-parameters
authentication-management   localization-management
backup-restore              log-management
boot-management             mail-transport-management
bootloader-management       maintenance-reporting (also Layer 2)
certificate-issuance        network-configuration
certificate-trust-management nfs-server-management
cockpit-management          package-management
configuration-management    precision-time-management
container-management        process-management
dns-server-management       remote-log-forwarding
encryption-management       repository-management
environment-configuration   resource-control-management
filesystem-management       samba-server-management
filesystem-quotas           scheduled-tasks
firewall-management         security-compliance
hardware-inventory          selinux-management
identity-management         service-management
insights-management         ssh-server-configuration
                            storage-management
                            subscription-management
                            tuned-management
                            user-group-management
                            virtualization-management
```

Some Layer-3 contracts are RHEL-specific
(`cockpit-management`, `insights-management`,
`selinux-management`, `subscription-management`,
`tuned-management`).  Others are universal across most OSes
(`package-management`, `service-management`,
`network-configuration`).  Per chapter 1's Convention 5
discussion: a feature without a per-OS implementation
returns `FAILED` with a stable token; the contract still
exists.

## The contract structure

Every `contract.md` follows the same EARS-inspired structure
with named sections.  A reader who's seen one contract knows
where to find each piece in any other.

### `# <Feature> Contract` and the boilerplate

Every contract opens identically:

```markdown
# Package-Management Contract

This is the cross-OS abstract contract for the
**package-management** feature.  Every `automojo-sysman-*`
package that ships a package manager MUST honor this contract.
```

The opening identifies the feature and reminds the reader
that *every* implementation must honour the contract.  The
*MUST* is RFC 2119 capitalisation — non-negotiable.

### `## Scope`

The Scope section is the contract's *abstract*.  A bulleted
list of *what the feature does*, written abstractly so it
applies to every OS:

```markdown
## Scope

The package-management feature provides idempotent,
auditable control over installed software packages.  It
covers:

- Installing a package at the latest version or at a pinned
  version.
- Removing an installed package.
- Upgrading a package to its latest version.
- Upgrading all installed packages.
- Querying a package's installed state.
- Listing all installed packages.
- Listing packages with available upgrades.
- Refreshing package-manager metadata.
- Installing a package from a local file.
- Dry-run-aware writes.
- Elevation for writes; reads never elevate.

The feature is executed exclusively through an `ICommandRunner`.
```

The section is two things:

- A *checklist* of what an implementation must support.
- A *boundary* — anything not in this list is not part of
  the contract (deferred to "Out of Scope" later).

The closing line — *"executed exclusively through an
`ICommandRunner`"* — is Convention 1 lock-in: file I/O,
read I/O, write I/O all routed through the runner seam.

### `## Verbs`

The Verbs section is a *table*.  Every verb (method) the
implementation must expose, with its signature and *kind*
(Read or Write):

```markdown
## Verbs

Every OS implementation MUST expose a `PackageManager` class
(the OS-prefixed variant — see §Naming) whose public surface
is the following set of verbs.

| Verb | Signature | Kind |
|------|-----------|------|
| `install_package` | `(name: str, version: str \| None) → PackageActionResult` | Write |
| `remove_package` | `(name: str) → PackageActionResult` | Write |
| `upgrade_package` | `(name: str) → PackageActionResult` | Write |
| `upgrade_all` | `() → PackageActionResult` | Write |
| `is_installed` | `(name: str) → bool` | Read |
| `get_package_info` | `(name: str) → PackageInfo \| None` | Read |
| `list_installed` | `() → list[PackageInfo]` | Read |
| `list_upgradable` | `() → list[PackageInfo]` | Read |
| `refresh_metadata` | `() → PackageActionResult` | Write |
| `install_local` | `(path: str) → PackageActionResult` | Write |
```

Three things matter in the table:

- **The verb name is exact.**  The implementation's method
  must have this name.  Renaming or aliasing breaks the
  contract.
- **The signature is exact.**  Argument types, argument
  order, return type — all locked.  An OS implementation that
  needs an extra argument adds it as a *config* on the
  manager (per Convention 8), not as a verb argument.
- **The Kind is exact.**  Reads never elevate; writes
  elevate per the manager's config.  A "Read" verb must not
  modify state; a "Write" verb returns a `*ActionResult`.

### `## Method-level invariants (writes and reads)`

The invariants section says *what's true before, during, and
after each verb runs*:

```markdown
## Method-level invariants

### Writes

- Every write MUST honour `dry_run`.  When `dry_run=True`,
  the manager MUST NOT invoke the runner; it MUST return
  `*ActionResult(status=DRY_RUN, command=[...the would-have-
  been-argv...])`.
- Every write MUST honour `elevated`.  When `elevated=True`,
  the manager MUST pass `elevated=True` to the runner.
- Every write MUST be idempotent.  When the post-state
  matches the pre-state, the manager MUST return
  `*ActionResult(status=SKIPPED, ...)` without invoking the
  runner.
- Every write MUST record the command argv on the result
  (per Convention 2).

### Reads

- Reads MUST NOT elevate.  The runner is invoked with
  `elevated=False` regardless of `config.elevated`.
- Reads MUST NOT modify state.
- Reads MAY return `None` / empty / synthetic record on
  resource-not-found (per Convention 7); the per-OS spec
  pins down which.
```

The *MUST* / *MUST NOT* / *MAY* keywords are RFC 2119
again.  The contract is precise about which behaviours are
required and which are permitted to vary per OS.

### `## Data Models`

Enums and dataclasses the verbs return:

```markdown
## Data Models

### `PackageActionStatus` (Enum)

| Value | Meaning |
|-------|---------|
| `SUCCESS` | The package was installed / removed / upgraded as requested. |
| `FAILED`  | The operation could not be completed. |
| `SKIPPED` | The post-state already matched (idempotent skip). |
| `DRY_RUN` | `dry_run=True`; the runner was not invoked. |

### `PackageActionResult` (Dataclass)

| Field | Type | Description |
|-------|------|-------------|
| `status` | `PackageActionStatus` | The outcome status. |
| `package_name` | `str` | The package name acted on. |
| `version` | `str \| None` | The version acted on (or installed). |
| `command` | `list[str] \| None` | The argv that was (or would have been) invoked. |
| ... | ... | (inherited fields from `FeatureActionResult`) |

### `PackageInfo` (Dataclass — query result)

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Package name. |
| `version` | `str` | Installed version. |
| `architecture` | `str \| None` | Architecture (when known). |
| ... | ... | (per-OS extensions allowed) |
```

The status enum names are *the same across every contract*
— `SUCCESS`, `FAILED`, `SKIPPED`, `DRY_RUN`.  The
dataclass names are *per-feature* (`PackageActionResult`,
`ServiceActionResult`, etc.) but they all subclass
`FeatureActionResult` so the standard fields are uniform.

### `## Channel/Source Mapping`

Where channels (e.g., command-execution channels) and
sources (configuration files) come from for each verb:

```markdown
## Channel/Source Mapping

| Verb | Source | Channel |
|------|--------|---------|
| `install_package` | (none) | runner |
| `list_installed` | (none) | runner |
| `install_local` | `path` argument | runner |
| ... | ... | ... |
```

Useful for understanding what each verb reads / writes at
the system level.

### `## Elevation`

The contract restates the elevation rules per verb:

```markdown
## Elevation

Reads NEVER elevate.  Writes elevate when `config.elevated=True`,
EXCEPT:

- Implementations of `install_package` on macOS via Homebrew
  MUST pass `elevated=False` to the runner and wrap the
  argv with `sudo -u <brew_user>` instead.  See Convention 11.
```

The "EXCEPT" carve-outs are the contract's *acknowledgement*
of OS-specific quirks that affect the contract's surface.
Convention 11 (Homebrew root refusal) is the canonical
example.

### `## Execution Model`

Whether verbs are synchronous or asynchronous, in-process
or via subprocess:

```markdown
## Execution Model

Every verb is **synchronous** from the caller's perspective.
The verb returns when the underlying operation has
completed (or failed, or hit a timeout).  Implementations
MAY use threads internally for parallelism but MUST NOT
return before the operation is observable on the system.

Every verb runs through `ICommandRunner` for the actual
command execution.  Implementations MUST NOT call
`subprocess.run` or equivalent directly.
```

The synchronous-from-the-caller commitment matches the
single-actor determinism model from Volume 2 chapter 2.

### `## Naming`

The OS-prefixed type-name convention (Convention 4):

```markdown
## Naming

Every OS implementation MUST expose its types with an
OS-prefixed name:

- Ubuntu: `UbuntuPackageManager`, `UbuntuPackageActionResult`,
  `UbuntuPackageActionStatus`, `UbuntuPackageInfo`.
- macOS: `MacosPackageManager`, ...
- Windows: `WindowsPackageManager`, ...
- UEFI: `UefiPackageManager` (when applicable).

RHEL retains unprefixed names (`PackageManager`,
`PackageActionResult`) for historical reasons; this is the
documented exception.
```

The historical exception for RHEL is honest — when the
suite was first developed, RHEL was the reference and the
prefixing convention came later.  Renaming RHEL's classes
across the test suite is a maintenance cost the project
hasn't paid; the contract acknowledges the inconsistency
rather than pretending it isn't there.

### `## Aspect Composition`

How the verb interacts with Volume 5's `Aspects` pattern:

```markdown
## Aspect Composition

Every write verb that runs through the runner MAY accept
an optional `aspects: AspectsCmd` parameter.  When provided,
the runner uses the supplied aspects; when omitted, the
runner uses its default aspects.

A verb that takes longer-than-default to complete (e.g.,
`upgrade_all` on a system with hundreds of packages) SHOULD
document the appropriate aspects shape in its per-OS spec.
```

This is the contract acknowledging Volume 5's Measured
Interoperability discipline.  The package-management verbs
go through the runner; the runner takes aspects; therefore
package-management implicitly inherits the persistence
behaviours from chapter 5.

### `## Tests`

What every implementation must test (per chapter 1's
discipline mechanisms):

```markdown
## Tests

Each OS implementation MUST include at minimum:

1. One command-construction test per write verb.
2. One idempotency test per write proving `SKIPPED` on
   matching state.
3. One `is_installed` / `get_package_info` test for an
   installed and a missing package.
4. One `list_installed` parser test covering representative
   output.
5. One `list_upgradable` parser test covering representative
   output, including the OS's "updates available" exit-code
   idioms (e.g., `dnf check-update` returning 100).
6. One `install_local` test proving `PackageFileNotFoundError`
   is raised for a missing path and the runner is not invoked.
7. One dry-run test per write verb proving the runner is
   not invoked.
8. One elevation test proving writes pass `elevated=True`
   and reads do not, when configured.
```

The test list is a *minimum bar*.  Per-OS implementations
add more — output-format-specific parser tests, edge cases
that arise from the OS's tool quirks, integration tests
against the real tool.

### `## Out of Scope`

What the contract *deliberately doesn't* cover:

```markdown
## Out of Scope

The package-management contract does NOT cover:

- Repository configuration (covered by `repository-management`).
- Package signing / verification (covered by
  `certificate-trust-management`).
- Service activation after install (covered by
  `service-management`).
- Automatic upgrade scheduling (covered by `scheduled-tasks`).
```

The "Out of Scope" section is what keeps the contract
focused.  Tests that need to verify "the package was
installed AND its service is running" use *both*
`PackageManager` and `ServiceManager`, calling each one's
verbs.  The contracts compose; no contract tries to do
everything.

### `## Implementation Drift`

Acknowledged places where per-OS implementations *can't*
honour the contract literally:

```markdown
## Implementation Drift

- macOS Homebrew has no concept of "system-wide install"
  — packages install per-user.  Implementations MUST
  document this and the manager's `brew_user` config field.
  Tests SHOULD use `brew_user` to verify the user-context
  inversion (Convention 11).

- Windows winget produces text output rather than a stable
  machine-readable format.  Per-OS spec MUST document the
  parsing strategy (column-position from header dashes; see
  `automojo-sysman-windows/.../packagemanagement.py`).

- Some Linux distributions ship `apt` and `dnf`
  simultaneously (e.g., Fedora's `dnf` plus optional
  `apt`).  Implementations target one tool only; the
  package's name (`automojo-sysman-linux-redhat` vs
  `-ubuntu`) determines which.
```

The "Implementation Drift" section is the contract's
*honesty*.  Real OSes don't all support every concept the
contract describes; the section calls out where the
implementations have to diverge and what tests should look
for.

## Three worked examples

To make the structure concrete, the chapter walks three
contracts end-to-end.

### Example 1 — `package-management`

The example used throughout the chapter so far.  Highlights:

- 10 verbs (4 reads, 6 writes).
- Three data models: `PackageActionStatus`,
  `PackageActionResult`, `PackageInfo`.
- Implementation Drift acknowledges Homebrew's per-user
  install model and winget's text fragility.
- 8 minimum tests per implementation.

### Example 2 — `service-management`

The contract is at
`automojo-sysman-common/specs/contracts/service-management/contract.md`.

Distinctive features:

- 11 verbs covering service lifecycle (start, stop, restart,
  reload, enable, disable, mask, unmask, query, list,
  reload_daemon).
- A `ServiceScope` enum (`SYSTEM` vs `USER`) — matters on
  Linux (systemctl --user) and macOS (launchctl `gui/<uid>`).
- "Reload" semantics differ across OSes: Linux systemd has
  `systemctl reload`; Windows has no reload concept and
  returns `FAILED` per Convention 5.
- Implementation Drift notes:
  - Windows has no separate "masking" — `mask_service` is
    aliased to `disable_service`.
  - macOS launchd uses domains (`system`, `gui/<uid>`)
    rather than systemd's `--user` flag.

### Example 3 — `network-configuration`

The contract is at
`automojo-sysman-common/specs/contracts/network-configuration/contract.md`.

Distinctive features:

- 11 verbs covering interfaces, addressing, routes, DNS,
  hostname.
- `InterfaceInfo` carries fields like `name`, `type`,
  `mac_address`, `addresses`, `mtu`, `state`.
- Implementation Drift is *substantial*: Linux uses netplan
  YAML (Ubuntu) or NetworkManager KeyFile (RHEL); macOS
  uses `networksetup` writing through System Configuration;
  Windows uses PowerShell with `Get-NetAdapter`.
- Per-OS configurations differ in *where persistence
  lives*: Linux files under `/etc/`, macOS in
  `configd`/`scutil` plists, Windows in WMI / registry.
- The contract acknowledges that "persistent route
  management" is out of scope on macOS (BSD `route` is
  runtime-only).

## What the contracts give you

A reader who's read this chapter should now:

- Know where the 50 contracts live and how they're
  organised.
- Recognise the standard sections in any contract (Scope,
  Verbs, Invariants, Data Models, Channel/Source Mapping,
  Elevation, Execution Model, Naming, Aspect Composition,
  Tests, Out of Scope, Implementation Drift).
- Be able to open any contract and find specific
  information quickly.
- See how Convention 5 (FAILED with stable token) and
  Convention 11 (Homebrew elevation inversion) fit into
  the contracts as Implementation Drift.

The contracts are the *documentation* of cross-OS
automation.  Chapter 3 walks the *substrate* that turns the
contracts into runnable code.

## What's next

Chapter 3 walks `featurebase` — `BaseFeatureManager`,
`FeatureConfig`, `FeatureActionResult`, and the `_dispatch`
helper that enforces the contract invariants (dry-run,
elevation, idempotency).  The substrate is what every per-OS
implementation builds on top of.
