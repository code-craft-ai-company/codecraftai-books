# Chapter 1 — Why a Contract Layer

> *Cross-OS automation has two shapes that fail.  One is
> "OS-by-OS feature flags" — `if platform.system() == "Linux":`
> branches scattered through every test.  The other is "least
> common denominator" — strip away the platform-specific
> concepts until the API is too anaemic to be useful.  The
> sysman tier rejects both with a *contract layer*: 50 cross-OS
> contracts that lock the *semantics* of each feature without
> committing to OS-specific *mechanism*.*

This is the volume's foundational chapter.  By the end, a
reader should know *why* a separately-versioned contract layer
exists and what it commits to.  Chapter 2 walks the contracts
themselves; chapter 3 walks the substrate (`featurebase`)
that turns the contracts into runnable code.

## The two failure modes

Most automation suites that span multiple OSes choose between
two unworkable patterns.

### Pattern A — OS-by-OS feature flags

```python
def install_package(name: str) -> bool:
    if platform.system() == "Linux":
        if shutil.which("dnf"):
            return _run_dnf_install(name)
        elif shutil.which("apt-get"):
            return _run_apt_install(name)
        else:
            raise RuntimeError("unknown Linux package manager")
    elif platform.system() == "Darwin":
        return _run_brew_install(name)
    elif platform.system() == "Windows":
        return _run_winget_install(name)
    else:
        raise NotImplementedError(platform.system())
```

The pattern looks reasonable in one place.  It's a disaster
across thousands of test cases.  The `if platform.system()`
ladders proliferate.  The error-handling diverges.  The
return shapes diverge — `_run_dnf_install` returns a tuple,
`_run_brew_install` returns a status enum, `_run_winget_install`
returns a parsed dict.  Test code that calls
`install_package` has no idea what to expect; either every
caller learns every OS's quirks, or the function lies by
forcing a least-common-denominator return.

The cost shows up as drift.  Six months in:

- One OS's branch tests for a different return type than
  another's because the developer who wrote that branch had
  different assumptions.
- One OS's branch silently swallows errors that another's
  bubbles, and a test passes on macOS while failing on
  Linux for the same scenario.
- One OS's branch handles the "package already installed"
  case as success; another raises.  Test code rewrites the
  semantics every time.

### Pattern B — Least common denominator

```python
def install_package(name: str) -> bool:
    """
        Returns True if the package install command exited cleanly.
    """
    cmd = _platform_install_command(name)
    return subprocess.run(cmd).returncode == 0
```

The pattern looks clean.  It's also *useless*.  "Did the
install command exit cleanly" is not the question the test
is asking.  The test wants to know:

- Was the package already installed (skipped) or was it
  freshly installed (succeeded)?
- If a version was specified, did the right version land?
- If installation failed, was it a transient network blip
  or a permanent error like "package doesn't exist in any
  configured repository"?
- Was elevation required, and was it granted?
- Did the install have side effects that need to be undone
  if the test fails partway through?

A least-common-denominator API answers none of these.  Every
call site has to *re-derive* the answer from raw stdout /
stderr / exit codes — and the call site doesn't know the
OS-specific output formats.

## The third option — a contract layer

The sysman tier picks neither pattern.  It picks a *contract
layer*: a set of cross-OS specifications that define the
*semantics* of each feature without committing to OS-specific
*mechanism*.

Concretely:

- Every feature has a markdown contract at
  `automojo-sysman-common/specs/contracts/<feature>/contract.md`.
- The contract specifies *verb signatures* (what methods
  exist, what arguments they take, what they return).
- The contract specifies *invariants* (what must be true
  before / during / after the call).
- The contract specifies *result vocabulary* (what
  `ActionStatus` values are returned, when each applies).
- The contract specifies *elevation rules* (which verbs
  elevate, which don't).
- The contract leaves *mechanism* to per-OS implementations.

A test calling `package_manager.install("nginx")` knows it
will get back a result with a status of `SUCCESS`, `SKIPPED`,
`FAILED`, or `DRY_RUN` — and knows what each means.  The
test code is *the same* across OSes.  The implementations
underneath differ wildly (dnf invocations vs. apt-get vs.
brew), but the test sees a uniform surface.

## The 50-contract substrate

The portfolio ships **50 contracts** — one per feature area.
The full list lives at
`automojo-sysman-common/specs/contracts/README.md:385–510`,
catalogued in three layers:

- **Layer 1 (Infrastructure)** — 1 contract: `command-execution`.
  The substrate for everything above.
- **Layer 2 (Data)** — 1 contract: `maintenance-reporting`.
  Cross-feature reporting.
- **Layer 3 (Business Logic)** — 48 contracts: firewall,
  service, package, network, encryption, certificate,
  user/group, kernel parameters, virtualization, container,
  ACLs, audit, authentication, backup, DNS server, mail
  transport, NFS, repository, samba, security compliance,
  storage, etc.

The chapter doesn't list all 50 by name (the catalogue does
that).  What matters is the *shape* of the layering.  A
Layer-3 contract like `package-management` calls into Layer
1's `command-execution` contract for actually running the
underlying command.  Layer 2's `maintenance-reporting`
sits alongside, capturing the cross-feature concerns of
"what changed and when."

## The five reference implementations

Five OSes are implemented to the contract:

1. **`automojo-sysman-linux-redhat`** — RHEL family (RHEL,
   CentOS Stream, Fedora).  46 features.  53 test files.
2. **`automojo-sysman-linux-ubuntu`** — Ubuntu / Debian
   family.  40 features.  41 test files.
3. **`automojo-sysman-macos`** — macOS (Apple Silicon and
   Intel).  49 features.  45 test files.
4. **`automojo-sysman-windows`** — Windows 10/11 / Server.
   49 features.  45 test files.
5. **`automojo-sysman-uefi`** — UEFI firmware.  Intentionally
   smaller — 5 features only.

Why five?  Because these five OSes cover the bulk of
production automation targets.  The contract layer doesn't
*forbid* implementations for other OSes — adding FreeBSD,
Solaris, or AIX would follow the same pattern — but the
maintenance burden of each implementation is real and the
five shipped are what the portfolio actively maintains.

A note on the test counts.  The volume's outline mentions
much higher numbers (e.g., "814 tests" for Ubuntu, "917 tests"
for Windows).  Those are *test invocation* counts (individual
`test_*` functions) rather than test *file* counts.  A single
test file typically holds a dozen or more test functions.
The chapter uses file counts in its summaries because they're
verifiable; readers should expect actual test invocations to
be 10–20× the file count.

## Why these five are not the same

A reader might naively expect that 50 contracts × 5 OSes
should produce 250 features × 5 implementations.  The
reality is messier:

- **RHEL has features Ubuntu doesn't** — `selinux-management`
  (RHEL has SELinux; Ubuntu uses AppArmor),
  `subscription-management` (Red Hat Subscription Manager),
  `cockpit-management`, `insights-management`,
  `tuned-management`.
- **Ubuntu has features RHEL doesn't** — `bootloader-management`
  (covers EFI variants beyond RHEL's simpler boot story).
- **macOS doesn't ship some features** — Linux-specific
  concepts (`selinux-management`, `firewalld`) have no macOS
  analogue.
- **Windows has unique concepts** — Active Directory
  integration, Windows-specific service control, DSC.

The contract layer accommodates this through **Convention 5**
(chapter 9 walks the eleven conventions): when a feature has
no scriptable implementation on a given OS, the manager
returns `*ActionResult` with `status=FAILED` and a stable
detail token like `unsupported_on_host_os_darwin`.  This
*is* the contract — a feature that's intentionally
unsupported on an OS still has a manager that *responds
predictably*.

## What the contract layer commits to

A reader should understand what the contract layer locks
down and what it leaves loose.

### Locked

- **Verb names.**  Every OS's `PackageManager` exposes
  `install`, `remove`, `query`, `list_installed`, etc. with
  exactly the names the contract specifies.
- **Verb signatures.**  Argument order, types, defaults.
- **Result types.**  Every `*ActionResult` subclass has
  `status`, `detail`, `exit_code`, `stdout`, `stderr`,
  `command` fields (per Convention 2).
- **Status vocabulary.**  `SUCCESS`, `FAILED`, `SKIPPED`,
  `DRY_RUN` — same four values across every feature on
  every OS.
- **Elevation rules.**  Reads never elevate; writes elevate
  per the manager's config (with the Homebrew / Convention
  11 inversion as the well-known exception).

### Loose

- **Mechanism.**  How a manager actually invokes the
  underlying tool.  RHEL's `PackageManager.install` runs
  `dnf install -y`; Ubuntu's runs `apt-get install -y`;
  macOS's runs `brew install` (with `sudo -u`).
- **Output parsing.**  Every OS's tool produces different
  text; per-OS code parses its own.
- **Per-OS quirks.**  Apt's `DEBIAN_FRONTEND=noninteractive`,
  Homebrew's root refusal, Windows' winget table-format
  fragility — all per-OS implementation concerns.

The split is what makes the layer work.  A test author
writing against the contract sees a uniform surface.  An
implementor handling a single OS focuses on that OS's
specifics.  Neither has to know about the other's concerns.

## What the contract layer is *not*

Three misconceptions worth clearing up.

### It is not a shared abstraction layer

The contract layer is not "an abstract base class everyone
inherits from."  It is *prose* — markdown contracts that
specify the API in English plus tables.  The shared base
class (`BaseFeatureManager` from chapter 3) implements *some*
of the contract uniformly, but the contract itself is text.

The reason: per-OS implementations vary so much in *how*
they implement the contract that a code-level abstraction
would either be too thin (forcing duplication) or too
opinionated (forcing per-OS quirks into a shape that doesn't
fit).  Prose contracts let the contract specify *what* and
let each OS's code decide *how*.

### It is not generated

There is *no code generator* for the per-OS implementations.
Every per-OS file is hand-written to follow the contract.
The substrate (`featurebase` — chapter 3) provides shared
helpers, but the manager classes themselves are
hand-authored.

The reason: the OS-specific quirks (Homebrew root refusal,
winget output parsing, launchd domain semantics) are
genuinely irreducible.  A code generator that tried to
produce them would either (a) not produce them and force
hand-edits, defeating the generator, or (b) become a
prose-to-code translator that's harder to read than the
direct hand-written code.

The chapter is honest: hand-writing 50 contracts × 5
implementations is *expensive*.  The portfolio justifies the
cost by treating sysman as a *foundational* tier — once it's
correct, every higher-tier test relies on it.  The cost is
amortised across the rest of the series.

### It is not bidirectional

The contracts describe *what the framework provides to test
code*, not *what test code provides to the framework*.  A
test that wants to install a package calls
`PackageManager.install(...)`; the contract specifies that
call.  The reverse — "what does test code have to do for the
framework to call its hooks?" — is a different concern
covered by the test framework (Volume 1).

## How the discipline holds up

Three mechanisms keep the per-OS implementations aligned with
the contracts:

### Shared base classes

Every per-OS manager inherits from `BaseFeatureManager` (or
its `featurebase` family).  The base class implements the
common scaffolding (`__init__`, `_dispatch`,
`_wrap_result`) so that *anything that uses these helpers*
follows the contract by construction.  Chapter 3 walks the
base.

### The 11 Implementation Conventions

Documented at
`automojo-sysman-common/specs/contracts/README.md:75–364`,
the eleven conventions cover the cross-OS patterns that
emerged from the Phase-3 implementation work.  Things like:

- **Convention 1**: `ICommandRunner` is the file-I/O seam.
  No separate `IFileOperator`.
- **Convention 4**: OS-prefixed concrete type names
  (`UbuntuPackageManager`, etc.).
- **Convention 11**: Elevation-model inversion when the
  underlying tool refuses root (Homebrew).

Chapter 9 walks all eleven in detail.

### Per-OS test coverage

Every contract's "Tests" section prescribes minimum coverage
(command-construction tests per write verb, idempotency
tests, dry-run tests, elevation tests).  Per-OS packages
implement these tests.  When a per-OS implementation drifts
from the contract, the tests catch it.

The discipline isn't enforced by automated specification
tooling.  It's enforced by the combination of shared bases,
documented conventions, and thorough tests.  An engineer
trying to skip the discipline runs into one of the three.

## What this volume covers

The remaining nine chapters walk the layer in detail:

- **Chapter 2** — the 50 contracts themselves.  How they're
  organised, the EARS-inspired structure, three worked
  examples (package-management, service-management,
  network-configuration).
- **Chapter 3** — `featurebase` and the per-OS implementation
  pattern.  `BaseFeatureManager`, `FeatureConfig`,
  `FeatureActionResult`, the `_dispatch` helper.
- **Chapters 4–8** — the five reference implementations,
  one per OS.  RHEL, Ubuntu, macOS, Windows, UEFI.  Each
  chapter walks the package's structure, key features, and
  OS-specific quirks.
- **Chapter 9** — the 11 Implementation Conventions and the
  spec-driven development discipline.
- **Chapter 10** — building cross-OS test parity.  How the
  tests compose into a green-across-OSes campaign.

By the end of the volume, a reader should be able to:

- Read any contract and predict the per-OS implementation
  shape.
- Read any per-OS feature and identify which contract it
  implements.
- Add a new feature to an existing OS by writing the
  contract first and the implementation second.
- Add a new OS to the suite by implementing the existing
  contracts following the conventions.

The discipline is the volume's value.  Five OSes × 50
features is a maintenance bill many automation suites would
walk away from.  The portfolio pays the bill because the
contract layer makes the maintenance *predictable* — every
test against the sysman tier behaves the same way regardless
of OS.

## What's next

Chapter 2 walks the 50 contracts: their structure, what
they specify, and three worked examples that show the
contract format end-to-end.
