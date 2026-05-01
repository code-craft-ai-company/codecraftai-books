# Chapter 4 — RHEL Implementation

> *RHEL is the historical reference implementation.  46 features
> in a registry, 53 test files, the unprefixed type names that
> predate Convention 4.  This chapter walks the package's
> structure, three representative features (`dnf`,
> `systemctl`, `firewalld`), and the layered spec architecture
> that distinguishes RHEL from Ubuntu's shared-base pattern.*

`automojo-sysman-linux-redhat` is the first sysman package
shipped in the portfolio.  Many of the patterns the other
four reference implementations follow were *invented* here —
the registry, the layered spec architecture, the
contract-driven development discipline.  Reading this
chapter is reading the *origin* of the sysman tier's
discipline.

## Layout

```
automojo-sysman-linux-redhat/source/packages/automojo/sysman/linux/redhat/
├── __init__.py
├── featurebase.py            # local extension to the common base
├── featureregistry.py        # the 46-feature registry
├── packagemanagement.py
├── servicemanagement.py
├── firewallmanagement.py
├── networkconfiguration.py
├── selinuxmanagement.py
├── subscriptionmanagement.py
├── cockpitmanagement.py
├── insightsmanagement.py
├── tunedmanagement.py
├── ... (37 more feature modules)
└── tests/                    # 53 test files in source/testroots/...
```

47 Python files in the implementation directory (excluding
`__init__.py`).  Each represents either a feature manager or
shared infrastructure (`featurebase`, `featureregistry`).
The structure is *flat* — no deep nesting.  Every feature is
a single file at the top level of the package.

A reader looking for "the RHEL package manager" finds it at
`packagemanagement.py`.  A reader looking for "the RHEL
SELinux feature" finds it at `selinuxmanagement.py`.  The
naming is uniform; the files map 1:1 to the feature names in
the contract layer.

## The 46-feature registry

**File:** `featureregistry.py:146–469`

The RHEL package's feature registry is a comprehensive
catalogue of every feature the package implements.  It binds
canonical feature names to their implementing classes.

```python
# Excerpt — featureregistry.py uses lazy imports to avoid
# circular dependency issues.

FEATURE_DESCRIPTORS = [
    FeatureDescriptor(
        name="command-execution",
        layer=1,
        module="automojo.sysman.linux.redhat.commandexecution",
        class_name="LocalCommandRunner",
        summary="Run single commands, ordered sequences, and shell scripts.",
    ),
    FeatureDescriptor(
        name="package-management",
        layer=3,
        module="automojo.sysman.linux.redhat.packagemanagement",
        class_name="PackageManager",
        summary="RPM install/remove/update/query via dnf + rpm.",
    ),
    FeatureDescriptor(
        name="service-management",
        layer=3,
        module="automojo.sysman.linux.redhat.servicemanagement",
        class_name="ServiceManager",
        summary="systemd service lifecycle + journal queries.",
    ),
    FeatureDescriptor(
        name="firewall-management",
        layer=3,
        module="automojo.sysman.linux.redhat.firewallmanagement",
        class_name="FirewallManager",
        summary="firewalld zones, services, ports, rich rules.",
    ),
    # ... 42 more entries ...
]
```

A representative selection of the 46 features:

- **Layer 1 (1)**: `command-execution` (`LocalCommandRunner`).
- **Layer 2 (1)**: `command-runner-architecture`.
- **Layer 3 (44)**: `acl-management`, `audit-management`,
  `authentication-management`, `backup-restore`,
  `boot-management`, `certificate-issuance`,
  `certificate-trust-management`, `cockpit-management`,
  `configuration-management`, `container-management`,
  `dns-server-management`, `encryption-management`,
  `environment-configuration`, `filesystem-management`,
  `filesystem-quotas`, `firewall-management`,
  `hardware-inventory`, `identity-management`,
  `insights-management`, `kdump-management`,
  `kernel-parameters`, `localization-management`,
  `log-management`, `mail-transport-management`,
  `network-configuration`, `nfs-server-management`,
  `package-management`, `precision-time-management`,
  `process-management`, `remote-log-forwarding`,
  `repository-management`, `resource-control-management`,
  `samba-server-management`, `scheduled-tasks`,
  `security-compliance`, `selinux-management`,
  `service-management`, `ssh-server-configuration`,
  `storage-management`, `subscription-management`,
  `tuned-management`, `user-group-management`,
  `virtualization-management`.

A few of these are RHEL-specific — `cockpit-management`
(RHEL's web admin console), `insights-management` (Red Hat
Insights client), `selinux-management` (RHEL has SELinux;
Ubuntu uses AppArmor), `subscription-management` (Red Hat
Subscription Manager), `tuned-management` (the tuned
performance daemon).

The registry uses **lazy imports** — each `FeatureDescriptor`
records the module path and class name as strings; the
actual class is resolved on `descriptor.resolve()` rather
than at registry import time.  This avoids circular
imports (a feature whose implementation imports from another
feature) and reduces the cost of importing the registry
when only a handful of features are needed.

## The unprefixed naming exception

A reader following Volume 4 chapter 2's Convention 4 will
notice something: the RHEL classes are *not OS-prefixed*.

```python
# RHEL — unprefixed (the documented exception)
class PackageManager(BaseFeatureManager):
    ...

# Ubuntu — prefixed
class UbuntuPackageManager(BaseFeatureManager):
    ...

# macOS — prefixed
class MacosPackageManager(BaseFeatureManager):
    ...
```

The contract documentation (Convention 4 from chapter 9)
acknowledges this exception:

> *RHEL retains unprefixed names (`PackageManager`,
> `PackageActionResult`) for historical reasons; this is the
> documented exception.*

The reason is *project history*.  RHEL was the first
implementation; the suite's prefixing convention emerged
later when Ubuntu / macOS / Windows were added and the team
needed to disambiguate.  Renaming RHEL's classes across the
test suite would touch hundreds of test files for an
aesthetic improvement.  The project chose to acknowledge
the inconsistency rather than pay the rename cost.

A reader importing from RHEL writes:

```python
from automojo.sysman.linux.redhat.packagemanagement import (
    PackageManager,
    PackageActionResult,
)
```

A reader importing from Ubuntu writes:

```python
from automojo.sysman.linux.ubuntu.packagemanagement import (
    UbuntuPackageManager,
    UbuntuPackageActionResult,
)
```

The module paths disambiguate at the import level even
though the class names don't.  Tests that use both packages
(rare) typically alias one to avoid a class-name collision:

```python
from automojo.sysman.linux.redhat.packagemanagement import (
    PackageManager as RhelPackageManager,
)
from automojo.sysman.linux.ubuntu.packagemanagement import (
    UbuntuPackageManager,
)
```

## Three worked features

### Package management with `dnf`

**File:** `packagemanagement.py:143–179`

```python
class PackageManager(BaseFeatureManager):
    """
        High-level facade for Red Hat package management.  Keeps its
        own :class:`PackageActionResult` shape (``package_name``
        required) so does not use the base class's ``_dispatch``;
        inherits only the standard ``__init__`` / ``config`` /
        ``runner`` surface.
    """

    feature_name = "package-management"

    def install(self, name: str, version: Optional[str] = None) -> PackageActionResult:
        """
            Install a package, optionally at a specified version.
        """
        existing = self.query(name)
        if existing is not None:
            if version is None or existing.version == version:
                already_installed = PackageActionResult(
                    status=PackageActionStatus.SKIPPED,
                    package_name=name,
                    version=existing.version,
                )
                return already_installed

        target = name
        if version is not None:
            target = "{}-{}".format(name, version)
        command = [_DNF, "install", "-y", target]

        # ... call self._runner.run(command, elevated=...) directly ...
        # ... map exit code to status ...
        # ... return PackageActionResult ...
```

Two notes:

- **The class doesn't use `_dispatch`.**  The docstring
  explicitly says: *"does not use the base class's
  `_dispatch`."*  This is one of the cases chapter 3
  mentioned: features whose status mapping isn't simple
  exit-code-zero-vs-non-zero (dnf has `check-update`
  returning 100 for "updates available") implement their
  own dispatch.  RHEL's package manager is one of those.
- **Idempotency check first, then construct command.**  The
  `query` call (a Read verb) returns `None` if the package
  isn't installed, or a `PackageInfo` with the installed
  version.  If installed and version matches (or version
  is unspecified), return `SKIPPED` without invoking dnf.
  If not, build the argv and proceed.

The `target = "{}-{}".format(name, version)` line is the
RHEL-specific quirk.  dnf supports the `package-version`
syntax to pin a specific version.  Ubuntu's apt uses
`package=version` instead.  Per-OS code choice; visible at
this layer only.

### Service management with `systemctl`

**File:** `servicemanagement.py:100–178`

```python
class ServiceManager(BaseFeatureManager):

    def start(self, name: str) -> ServiceActionResult:
        """
            Start a service unit.  Skipped when the unit is already active.
        """
        info = self.query(name)
        if info.active_state == "active":
            return ServiceActionResult(
                status=ServiceActionStatus.SKIPPED,
                service_name=name,
            )
        return self._dispatch(
            command=[_SYSTEMCTL, "start", name],
            detail=name,
            operation="start",
            service_name=name,
        )

    def stop(self, name: str) -> ServiceActionResult:
        """
            Stop a service unit.  Skipped when already inactive.
        """
        info = self.query(name)
        if info.active_state == "inactive":
            return ServiceActionResult(
                status=ServiceActionStatus.SKIPPED,
                service_name=name,
            )
        return self._dispatch(
            command=[_SYSTEMCTL, "stop", name],
            detail=name,
            operation="stop",
            service_name=name,
        )
```

In contrast to `PackageManager`, the service manager *does*
use `_dispatch`.  systemctl's exit codes are
exit-code-zero-vs-non-zero clean — zero on success, non-zero
on failure — so the base helper's mapping suffices.

The pattern is uniform across the verbs:

1. Query the current state.
2. If it matches the target state, return `SKIPPED`.
3. Otherwise call `_dispatch` with the systemctl argv.
4. The dispatcher invokes the runner, maps the result, and
   wraps in `ServiceActionResult`.

### Firewall management with `firewalld`

**File:** `firewallmanagement.py:63–80`

```python
class FirewallManager(BaseFeatureManager):
    """
        Facade for firewalld management through the command runner.
    """

    result_class = FirewallActionResult
    feature_name = "firewall-management"

    def add_service(self, service: str, zone: str) -> FirewallActionResult:
        """
            Add ``service`` to ``zone`` (permanent configuration).
        """
        info = self.query_zone(zone)
        if service in info.services:
            return FirewallActionResult(
                status=FirewallActionStatus.SKIPPED,
                zone=zone, detail=service,
            )
        # ... build firewall-cmd argv, dispatch ...
```

Three notes:

- **`result_class = FirewallActionResult`** — overrides the
  default `FeatureActionResult` so `_wrap_result` builds the
  firewall-specific class with `zone` field.
- **Zone vocabulary** — firewalld's *zones* (public,
  internal, trusted, etc.) are first-class concepts.  RHEL's
  manager exposes them directly.  Ubuntu's ufw has no zones
  (chapter 5 walks the divergence); the contract abstracts
  the concept and Ubuntu collapses it.
- **`info.services`** — the query result includes a list of
  enabled services in the zone.  The idempotency check is "is
  this service already in this zone's services list?"

The `firewall-cmd` argv typically uses the `--permanent`
flag to make changes survive reload — a firewalld-specific
detail visible only at this layer.

## The layered spec architecture

`featureregistry.py:146–469` defines the three layers
explicitly:

```python
# Layer 1 — Infrastructure
FeatureDescriptor(name="command-execution", layer=1, ...),
FeatureDescriptor(name="command-runner-architecture", layer=1, ...),

# Layer 2 — Data
FeatureDescriptor(name="maintenance-reporting", layer=2, ...),

# Layer 3 — Domain features (44 entries)
FeatureDescriptor(name="acl-management", layer=3, ...),
FeatureDescriptor(name="audit-management", layer=3, ...),
# ... etc.
```

The layers are not just metadata.  They imply a *dependency
direction*:

- Layer 1 features have no dependencies on the rest of the
  registry.  `command-execution`'s `LocalCommandRunner` is
  the substrate everything else builds on.
- Layer 2 features depend on Layer 1.
- Layer 3 features depend on Layers 1 and 2.

A test that wants to use a Layer-3 feature (e.g.,
`PackageManager`) constructs the manager with a Layer-1
runner.  The framework's typical setup:

```python
from automojo.sysman.linux.redhat.commandexecution import LocalCommandRunner
from automojo.sysman.linux.redhat.packagemanagement import PackageManager
from automojo.sysman.linux.redhat.featurebase import FeatureConfig

runner = LocalCommandRunner()
manager = PackageManager(runner=runner, config=FeatureConfig(elevated=True))
result = manager.install("nginx")
```

The dependency injection is explicit — the test code passes
the runner to the manager.  The layered architecture is
*expressed* in the construction pattern.

The spec files themselves follow the same layering.  Under
`automojo-sysman-linux-redhat/specs/`, each layer has its
own subdirectory (`L1/`, `L2/`, `L3/`), and within each
layer, each feature has its own subdirectory with
`design.md`, `requirements.md`, and `tasks.md`.  The spec
files are the per-OS counterpart to the contracts — where
the contract specifies the abstract verb signatures, the
spec specifies the OS-specific tool mappings, output
formats, and edge cases.

## Tests

53 test files at
`source/testroots/testbase/automojo/tests/sysman/linux/redhat/`.

The test structure mirrors the source structure:

```
tests/
├── test_packagemanagement.py
├── test_servicemanagement.py
├── test_firewallmanagement.py
├── test_featureregistry.py
└── ... (49 more)
```

Each test file targets one feature manager.  Tests typically
include:

- Command-construction tests (verify the argv is right
  for each verb).
- Idempotency tests (verify SKIPPED on matching state).
- Dry-run tests (verify the runner isn't invoked).
- Elevation tests (writes elevate; reads don't).
- Output-parsing tests (verify query results are parsed
  correctly).
- Error-handling tests (FAILED on non-zero exit).

The tests run against a *mock runner* — they don't actually
invoke dnf or systemctl.  This makes the tests fast and
hermetic; they verify the manager's *logic* (what argv it
builds, what status it returns for which inputs) without
depending on a live RHEL system.

A separate integration test suite (run periodically against
a real RHEL host) validates that the constructed argvs
actually do what the contract promises.  But the unit tests
in `source/testroots/...` are the bulk of the test count.

## What's next

Chapter 5 walks the Ubuntu implementation
(`automojo-sysman-linux-ubuntu`) — 40 features, 41 test
files, the *prefixed* class names from Convention 4, and
the shared-base pattern that imports `featurebase` from
`automojo-sysman-common` rather than defining it locally.
The Ubuntu package shows what the sysman discipline looks
like *after* the patterns RHEL established.
