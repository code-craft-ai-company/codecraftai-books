# Chapter 10 — Building Cross-OS Test Parity

> *Nine chapters walked the contracts, the implementations,
> and the conventions.  This chapter is the volume's payoff:
> how the discipline turns into a green-across-OSes test
> campaign at scale.  Thousands of tests; five OSes; one
> uniform shape; predictable outcomes.*

The volume's earlier chapters established the *machinery*.
This chapter walks the *operational shape* — what an SDET's
day-to-day work looks like on a project that's adopted the
sysman tier.  The chapter is the volume's closer; by the
end, a reader should know how to set up, run, and triage a
cross-OS test campaign against the framework.

## The reference numbers

The `codecraftai-sysman-*` packages collectively ship:

- **5 reference implementations** (RHEL, Ubuntu, macOS,
  Windows, UEFI).
- **~245 manager classes** (the per-OS package counts vary;
  the rough total is 5 × ~50 features).
- **~270 test files** combined (RHEL 53 + Ubuntu 41 + macOS
  45 + Windows 45 + UEFI 175 — UEFI's count is
  disproportionate because of the 3-path dispatch matrix).
- **Several thousand individual test invocations** —
  parametrised tests in pytest's sense, one test file
  containing many test functions.

The volume's outline mentioned "3,656 / 3,656 testbase
tests passing."  That count is *test invocations*, not test
files.  A single test file might have 30 test functions,
each parametrised across 5–10 input combinations, totalling
hundreds of individual `assert` calls.

The discipline of getting that count to *pass* everywhere
across five OSes is what this chapter walks.

## The setup

A cross-OS test campaign typically requires:

### One CI runner per OS

The framework's tests aren't fully cross-host (you can't
run a Windows test against a Linux runner without a Windows
host somewhere).  A typical setup:

- One Linux runner for RHEL and Ubuntu tests.
- One macOS runner for macOS tests.
- One Windows runner for Windows tests.
- The Linux runner usually doubles as the UEFI test runner,
  using `override_host_os` (Convention 10) to exercise the
  Windows and macOS dispatch paths.

Volume 2 chapter 3's Pod model applies — each runner is a
*Pod*; the test campaign is *more Pods*, not more parallel
tests within a Pod.

### Pinned tool versions

`dnf` 4.x produces different output than `dnf` 5.x.  `winget`
1.0 differs from 1.5.  The test discipline *pins* tool
versions per OS (the per-OS spec documents the supported
range).  When a CI runner gets an unexpected tool version,
the parser tests catch it before the broader test suite
runs.

### The Aspects defaults

Per Volume 5, every sysman manager runs commands through an
`ICommandRunner` that takes `Aspects`.  Test code typically
uses defaults, but slow OSes (macOS waiting for `softwareupdate`,
Windows waiting for service start) need longer timeouts:

```python
SLOW_ASPECTS = AspectsCmd(
    completion_timeout=300,    # 5 minutes for slow operations
    completion_interval=15,
    action_pattern=ActionPattern.DO_UNTIL_SUCCESS,
)

result = service_mgr.start("nginx", aspects=SLOW_ASPECTS)
```

Tests that exercise long-running operations override the
defaults.  Tests that exercise quick operations use the
short defaults and fail fast on hangs.

## A worked test scenario

A test that exercises *the same feature on three OSes*:

```python
import pytest

from ccai.sysman.linux.redhat.packagemanagement import PackageManager as RhelPM
from ccai.sysman.linux.ubuntu.packagemanagement import UbuntuPackageManager
from ccai.sysman.macos.packagemanagement import MacosPackageManager

import os

OS = os.environ.get("TARGET_OS", "linux-redhat").lower()


@pytest.fixture
def package_mgr(runner):
    if OS == "linux-redhat":
        return RhelPM(runner=runner, config=None)
    elif OS == "linux-ubuntu":
        return UbuntuPackageManager(runner=runner, config=None)
    elif OS == "macos":
        return MacosPackageManager(runner=runner, config=brew_config())
    else:
        pytest.skip(f"Unsupported OS for this test: {OS}")


def test_install_nginx_idempotent(package_mgr):
    """
        Verify nginx installs idempotently across all supported OSes.
    """
    # Initial install — may already be present.
    first = package_mgr.install("nginx")
    assert first.status in (
        # contract-specified outcomes
        type(first.status).SUCCESS,
        type(first.status).SKIPPED,
    )

    # Second install — must be SKIPPED per contract.
    second = package_mgr.install("nginx")
    assert second.status == type(second.status).SKIPPED
```

Three things to notice:

1. **Per-OS manager selection.**  The fixture picks the
   right manager class based on `TARGET_OS`.  The test body
   doesn't care which.
2. **Test against the contract.**  The assertions check
   `status in (SUCCESS, SKIPPED)` — the contract's
   vocabulary, not the OS's.
3. **`type(first.status)`** — the status enum is per-OS
   (`PackageActionStatus` for RHEL,
   `UbuntuPackageActionStatus` for Ubuntu, etc.), but they
   all have the same value names.  The test uses
   `type(...)` to look up the enum dynamically.

The test runs *unchanged* on three CI runners.  Each runner
sets `TARGET_OS` differently.  The test's *assertions* are
the contract's promise.

## What gets tested in a contract suite

A representative test suite for one feature (e.g.,
`PackageManager`) covers:

### Command-construction tests

For each write verb, assert the constructed argv matches
the expected pattern:

```python
def test_install_constructs_dnf_argv():
    config = FeatureConfig(dry_run=True)
    mgr = RhelPM(runner=mock_runner, config=config)

    result = mgr.install("nginx", version="1.21.6")

    assert result.command == ["dnf", "install", "-y", "nginx-1.21.6"]
    assert mock_runner.last_call is None  # dry-run; runner not invoked
```

Multiplied across every write verb across every feature
across every OS, this is hundreds of tests.  Each is small
and fast (no real subprocess calls).

### Idempotency tests

For each write verb that mutates state, verify a second
call returns SKIPPED:

```python
def test_install_skipped_when_already_installed(monkeypatch):
    mock_runner.queue_query_response(installed_version="1.21.6")
    mgr = RhelPM(runner=mock_runner, config=...)

    result = mgr.install("nginx", version="1.21.6")

    assert result.status == PackageActionStatus.SKIPPED
    assert mock_runner.last_install_call is None  # no real install attempted
```

The test mocks the *query* response (saying nginx 1.21.6 is
installed), then asserts that calling `install` doesn't
result in a runner call.  The idempotency check passed.

### Dry-run tests

For each write verb, verify `dry_run=True` skips the runner:

```python
def test_install_dry_run(mock_runner):
    config = FeatureConfig(dry_run=True)
    mgr = RhelPM(runner=mock_runner, config=config)

    result = mgr.install("nginx")

    assert result.status == PackageActionStatus.DRY_RUN
    assert mock_runner.calls == []  # no calls made
    assert result.command == ["dnf", "install", "-y", "nginx"]
```

### Elevation tests

Verify writes elevate (when `elevated=True`) and reads
don't:

```python
def test_install_elevates(mock_runner):
    config = FeatureConfig(elevated=True)
    mgr = RhelPM(runner=mock_runner, config=config)

    mgr.install("nginx")

    last_call = mock_runner.calls[-1]
    assert last_call.elevated is True


def test_query_does_not_elevate(mock_runner):
    config = FeatureConfig(elevated=True)  # caller asks for elevation
    mgr = RhelPM(runner=mock_runner, config=config)

    mgr.query("nginx")

    last_call = mock_runner.calls[-1]
    assert last_call.elevated is False  # but reads ignore it
```

### Output-parsing tests

Verify query results parse correctly from representative
output:

```python
def test_list_installed_parses_dnf_output(mock_runner):
    mock_runner.queue_response(stdout=DNF_LIST_OUTPUT_FIXTURE)
    mgr = RhelPM(runner=mock_runner, config=...)

    packages = mgr.list_installed()

    assert len(packages) == 247
    nginx = next(p for p in packages if p.name == "nginx")
    assert nginx.version == "1.21.6"
```

Fixtures (`DNF_LIST_OUTPUT_FIXTURE`) are real captured
output from a representative system.  They're checked into
the test suite.

### Error-handling tests

Verify the manager handles failures correctly:

```python
def test_install_failed_when_package_doesnt_exist(mock_runner):
    mock_runner.queue_response(exit_code=1, stderr="No match for argument: bogus")
    mgr = RhelPM(runner=mock_runner, config=...)

    result = mgr.install("bogus-package")

    assert result.status == PackageActionStatus.FAILED
    assert result.exit_code == 1
    assert "No match" in result.stderr
```

## How the volume of tests is *manageable*

A reader doing the math sees the test count is large.  ~50
features × 8 minimum-required tests per feature × 5 OSes =
2000 tests *minimum*.  Real packages have more (each verb
has command-construction + idempotency + dry-run +
elevation + output-parsing tests).

Three things make this manageable:

### The conventions are enforced by tests indirectly

A manager that follows the conventions passes the tests by
construction.  A manager that violates a convention fails a
test.  Engineers don't write per-convention tests; they
write per-feature tests, and the conventions surface in the
patterns.

### Mock runners run fast

A test that uses a mock runner doesn't invoke `dnf` or
`apt-get` or `Get-Service`.  The test runs in milliseconds.
A package's full unit-test suite typically completes in
under a minute even at 500+ tests.

### Test parameters are reusable

Many tests are parametrised across feature variants.  A
single test function plus pytest's `parametrize` runs the
same logic against five different verbs, ten different
input combinations, three different status outcomes — one
test source becomes dozens of test invocations.

## Integration tests against real OSes

The unit tests use mock runners.  For *integration*
testing — actually invoking dnf / apt-get / brew / winget —
the framework's discipline is:

- Run integration tests *separately* from unit tests.
- Run them periodically (nightly), not per-PR.
- Use real CI runners (RHEL VM, Ubuntu VM, macOS host,
  Windows VM).
- Treat integration-test failures as *infrastructure
  issues* — the test logic is unit-tested; the integration
  test verifies the OS hasn't changed underneath.

This is consistent with Volume 1 chapter 1's discipline:
unit tests verify the *framework's logic*; integration tests
verify the *system under test's behaviour*.  Both are
needed; they have different cadences and different failure
implications.

## Triaging a cross-OS failure

A failure pattern an SDET runs into: "the test passes on
RHEL and Ubuntu but fails on macOS."  Triage steps:

### Step 1 — Read the failure record

The failed test's `*ActionResult` carries:

- `status` — `FAILED`.
- `command` — the argv that ran (Convention 2).
- `stdout` / `stderr` — what the tool produced.
- `exit_code` — the tool's exit.
- `detail` — a stable token if Convention 5 applies.

A common failure: macOS's `result.detail ==
"unsupported_arbitrary_capsule_file_macos"`.  The test was
trying to apply a UEFI capsule via
`apply_capsule_file`; macOS doesn't support arbitrary
capsule files.  The test should skip on macOS, not fail.

### Step 2 — Check Convention 5 tokens

A failure with a stable `unsupported_*` token usually
indicates the test should skip on that OS:

```python
result = capsule_mgr.apply_capsule_file(path)
if result.status == ActionStatus.FAILED and result.detail.startswith("unsupported_"):
    pytest.skip(f"Feature unsupported on this OS: {result.detail}")
assert result.status == ActionStatus.SUCCESS
```

The test code becomes more sophisticated — it checks for
the stable token and skips gracefully.  Multiplied across
the test suite, this discipline keeps the test count *high*
(comprehensive coverage) without forcing every test to pass
on every OS.

### Step 3 — Check Convention 6 (Windows elevation)

If the failure is on Windows with
`result.elevation_blocked == True`, the test was
running under unelevated PowerShell.  Either:

- Run the test under elevated PowerShell.
- Skip the test when not elevated.

```python
result = mgr.install("nginx")
if getattr(result, "elevation_blocked", False):
    pytest.skip("This test requires elevated PowerShell.")
```

### Step 4 — Check macOS elevation inversion

If the failure is on macOS with `brew` errors mentioning
"running as root," the test wasn't configured with
`brew_user`.  Convention 11 requires the operator to set
this.

```python
config = MacosPackageManagerConfig(
    brew_user=os.environ.get("BREW_USER", "ci-runner"),
)
mgr = MacosPackageManager(runner=runner, config=config)
```

### Step 5 — Check the per-OS spec for known limitations

Each per-OS spec's "Implementation Drift" section
documents acknowledged divergences.  A test failing because
of a documented divergence is a *test bug* (didn't
account for the divergence), not a *manager bug*.

## What this discipline produces

The volume's earlier chapters described a discipline.  This
chapter shows what the discipline *produces*:

### Predictable test failures

Failures fall into three buckets:

1. **Real bugs in the manager** — fix the manager.
2. **Tool / OS-version drift** — fix the parser or update
   the supported-version range.
3. **Convention misuse** — fix the test (skip on
   unsupported, configure brew_user, etc.).

There's no fourth bucket of "I have no idea why this
failed."  The framework's structured output (Convention 2,
Convention 5, Convention 6) gives the failed test enough
information to triage.

### Reproducible cross-OS test runs

Same test code, same expected outcomes (modulo OS-specific
skips), green across all supported OSes.  An SDET who
writes a feature test against the contract gets cross-OS
coverage *for free* — the contract's verbs, status
vocabulary, and conventions take care of the per-OS
variation.

### Maintainable scale

50 features × 5 OSes is a lot of code.  The discipline
makes it *maintainable*:

- The contracts document the cross-OS API.
- The conventions document the cross-OS shape.
- The tests verify the conventions are followed.
- Code review catches convention violations.

A new engineer onboarding to the sysman tier reads the
conventions, picks a feature, follows the patterns.  No
deep tribal knowledge required.

## End of Volume 4

The volume started by asking why a contract layer exists
(chapter 1) and ended by walking a green-across-OSes test
campaign (this chapter).

The five reference implementations (RHEL, Ubuntu, macOS,
Windows, UEFI) demonstrate the discipline across genuinely
different operating systems.  Each implementation has its
quirks (Homebrew elevation, Windows PowerShell, UEFI single-
tier dispatch), and the contract layer absorbs them
without forcing test code to know about them.

The chapters that come next are in other volumes.  Volume
2's distributed automation invokes sysman feature execution
through bridge tasks.  Volume 3's orchestration triggers
sysman campaigns.  Volume 5's interop layer is the wire
that sysman managers use.  Volume 6's landscape-and-topology
is the environment sysman operates against.

Volume 4's contribution to the series is the *cross-OS
discipline* — the proof that genuinely different operating
systems can be automated through a single contract layer
when the layer is *honest about divergences* (Convention 5,
Convention 11) and *uniform in shape* (Conventions 2, 4, 6,
7).  Where Volume 5 said "talk to remote systems
consistently," Volume 4 says "ask remote systems to do
things consistently."
