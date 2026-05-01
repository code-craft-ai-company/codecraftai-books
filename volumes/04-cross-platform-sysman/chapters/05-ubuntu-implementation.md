# Chapter 5 — Ubuntu / Debian Implementation

> *Ubuntu came after RHEL.  It established the patterns RHEL
> grandfathered: the OS-prefixed class names (Convention 4),
> the shared `featurebase` imported from
> `automojo-sysman-common` (rather than locally defined), and
> the per-feature config extensions for tool quirks
> (`DEBIAN_FRONTEND=noninteractive`, the single-zone ufw
> abstraction).*

`automojo-sysman-linux-ubuntu` is the second sysman package.
Its existence is what *forced* the prefixing convention —
two packages with `PackageManager` classes was the
collision that prompted Convention 4.  Reading this chapter
is reading the *generalisation* of the patterns RHEL
established.

## Layout

```
automojo-sysman-linux-ubuntu/source/packages/automojo/sysman/linux/ubuntu/
├── __init__.py
├── packagemanagement.py
├── servicemanagement.py
├── firewallmanagement.py
├── networkconfiguration.py
├── bootloadermanagement.py    # covers EFI variants RHEL doesn't
├── ... (35 more feature modules)
└── (no local featurebase — imports from automojo-sysman-common)
```

40 Python files.  Notably *no local `featurebase.py`* — the
Ubuntu package imports `BaseFeatureManager`, `FeatureConfig`,
`FeatureActionResult`, and `ActionStatus` directly from
`automojo.sysman.common.featurebase`.

This is the **shared-base pattern** — the second-mover
advantage.  RHEL was first and has its own `featurebase` for
historical reasons (the common package didn't exist yet).
Ubuntu was second and uses the common package directly.
Every per-OS package added after Ubuntu (macOS, Windows,
UEFI) follows the shared-base pattern.

## OS-prefixed class names

Every public type in the Ubuntu package is prefixed with
`Ubuntu`:

```python
class UbuntuPackageManager(BaseFeatureManager): ...
class UbuntuPackageManagerConfig(FeatureConfig): ...
class UbuntuPackageActionResult(FeatureActionResult): ...
class UbuntuPackageActionStatus(Enum): ...
class UbuntuPackageInfo: ...

class UbuntuServiceManager(BaseFeatureManager): ...
class UbuntuServiceManagerConfig(FeatureConfig): ...
class UbuntuServiceActionResult(FeatureActionResult): ...
class UbuntuServiceScope(Enum): ...
```

The prefix is **mandatory** per Convention 4 — no
exceptions.  A reader importing from Ubuntu sees the prefix
in every name, which is what makes cross-package usage
unambiguous:

```python
from automojo.sysman.linux.redhat.packagemanagement import PackageManager
from automojo.sysman.linux.ubuntu.packagemanagement import UbuntuPackageManager

# Distinct names; no aliasing needed.
rhel_mgr = PackageManager(runner=rhel_runner, config=...)
ubuntu_mgr = UbuntuPackageManager(runner=ubuntu_runner, config=...)
```

The historical exception (RHEL retains unprefixed names) is
the *only* exception.  Every other package follows.

## Three worked features

### Package management with `apt-get`

**File:**
`packagemanagement.py:1–79`

```python
from automojo.sysman.common.featurebase import (
    ActionStatus,
    BaseFeatureManager,
    FeatureActionResult,
    FeatureConfig,
)


@dataclass
class UbuntuPackageManagerConfig(FeatureConfig):
    """
        Configuration for :class:`UbuntuPackageManager`.  Inherits
        ``dry_run`` and ``elevated`` from :class:`FeatureConfig`.

        :param apt_options: Optional list of additional flags appended to
                            every ``apt-get`` write argv.  Defaults to
                            ``None``.
        :param noninteractive_env: When True (the default), the runner's
                                   env for write verbs includes
                                   ``DEBIAN_FRONTEND=noninteractive``.
    """
    apt_options: Optional[list[str]] = None
    noninteractive_env: bool = True


class UbuntuPackageManager(BaseFeatureManager):
    """
        Ubuntu apt/dpkg package manager.  Every verb is dispatched through
        the injected :class:`ICommandRunner`.
    """

    result_class = UbuntuPackageActionResult
    status_class = UbuntuPackageActionStatus
    feature_name = "package-management"
```

Two Ubuntu-specific config extensions:

- **`apt_options`** — extra flags appended to every apt-get
  argv.  A test that wants to pass `--no-install-recommends`
  for stricter installs sets
  `config = UbuntuPackageManagerConfig(apt_options=["--no-install-recommends"])`.
- **`noninteractive_env`** — when True (the default),
  every write verb runs with the
  `DEBIAN_FRONTEND=noninteractive` env var set.  This
  prevents apt from prompting for user input during
  unattended runs (e.g., when a package's postinst script
  asks "do you want to keep your existing config file?").

The `noninteractive_env` flag is the *Ubuntu-specific
quirk* the contract leaves to the implementation.  RHEL's
dnf doesn't have this concept (dnf is non-interactive by
default with the `-y` flag).  The contract specifies the
verb's behaviour ("idempotent install at the requested
version"); the implementation handles the OS-specific noise
that gets in the way.

### Firewall management with `ufw`

**File:**
`firewallmanagement.py:167–176`

```python
class UbuntuFirewallManager(BaseFeatureManager):
    """
        Ubuntu ufw firewall manager.  Every verb is dispatched through
        the injected :class:`ICommandRunner`.
    """

    result_class = UbuntuFirewallActionResult
    status_class = UbuntuFirewallActionStatus
    feature_name = "firewall-management"
```

The contract's `firewall-management` interface includes
zone-based methods (`add_service(service, zone)`, etc.).
ufw, however, *has no zone concept*.

The Ubuntu manager handles this via Convention 5 (FAILED
with stable detail token):

```python
def add_service(self, service: str, zone: str) -> UbuntuFirewallActionResult:
    if zone != "default":
        return UbuntuFirewallActionResult(
            status=UbuntuFirewallActionStatus.FAILED,
            zone=zone,
            detail="ufw_no_zones",  # stable token
        )
    # ... ufw-specific implementation for the default zone ...
```

A caller passing any value other than `"default"` gets
`FAILED` with the stable token `ufw_no_zones`.  Tests
written against the contract know to accept this token on
Ubuntu; cross-OS tests that target the firewall feature on
both RHEL and Ubuntu either:

- Use `zone="default"` (works on both — RHEL's "default"
  is `public`).
- Have separate paths for the two OSes when zone-specific
  behaviour is needed.

Convention 5 is the framework's mechanism for handling this
class of divergence cleanly.

### Network configuration with `netplan`

**File:**
`networkconfiguration.py:1–80`

```python
"""
    Ubuntu network-configuration feature.

    Implements the cross-OS network-configuration contract for Ubuntu using
    the iproute2 family (``ip``), netplan (``netplan apply`` plus a
    declarative YAML drop-in under ``/etc/netplan/``), ``resolvectl`` for
    DNS, and ``hostnamectl`` for the static hostname.
"""


@dataclass
class UbuntuNetworkConfigurationManagerConfig(FeatureConfig):
    """
        :param netplan_dir: Directory holding netplan YAML drop-ins.
                            Defaults to ``/etc/netplan``.
        :param netplan_basename: Filename of the netplan drop-in
                                 maintained by this manager.  Defaults to
                                 ``99-automojo.yaml``.
    """
    netplan_dir: str = "/etc/netplan"
    netplan_basename: str = "99-automojo.yaml"
```

Persistent IPv4 addressing on Ubuntu happens through
**netplan YAML drop-in files**.  The Ubuntu manager:

1. Reads the existing drop-in file (if it exists).
2. Computes the desired post-state as a YAML dict.
3. If post-state matches existing, return SKIPPED.
4. Otherwise, render the YAML and write the file via
   `tee` (per Convention 1's file-I/O-through-runner rule).
5. Run `netplan apply` to make the change live.

The `netplan_dir` and `netplan_basename` config fields let
tests target a custom drop-in path (useful for testing in
CI environments where `/etc/netplan/` should be left
alone).

Compare to RHEL's network-configuration: RHEL uses
NetworkManager's KeyFile format under
`/etc/NetworkManager/system-connections/`.  Same contract;
different tool; different file layout; same idempotency
discipline.

The `resolvectl` and `hostnamectl` tools handle DNS and
hostname respectively — also different from RHEL's
NetworkManager-based DNS handling.  Each manager does what
the OS's idiom requires.

## What's NOT in Ubuntu

A reader comparing the Ubuntu and RHEL packages will notice:

### RHEL-only features absent from Ubuntu

- **`selinux-management`** — Ubuntu uses AppArmor, not
  SELinux.  The Ubuntu package doesn't ship an AppArmor
  manager (no contract for AppArmor exists yet).
- **`subscription-management`** — RHEL-specific (Red Hat
  Subscription Manager).
- **`cockpit-management`** — RHEL-specific.
- **`insights-management`** — RHEL-specific.
- **`tuned-management`** — RHEL-specific.
- **`command-execution`** and **`command-runner-architecture`**
  — these are in RHEL's registry as Layer-1 features, but
  Ubuntu inherits the runner from `automojo-sysman-common`
  and doesn't re-register.

### Ubuntu-specific feature: `bootloader-management`

The Ubuntu package ships `bootloader-management` (covering
GRUB2 with EFI variants).  RHEL's package has a simpler
`boot-management` feature; the Ubuntu version is more
sophisticated because Ubuntu's bootloader story (Secure
Boot, EFI, BIOS modes) varies more than RHEL's typical
deployment.

The asymmetry is *not* a contract violation.  The contracts
are a *menu*; not every OS implements every feature.  What
the contracts ensure is that *when* an OS implements a
feature, the implementation honours the contract.

## The shared-base benefit visible

A worth-pointing-out detail: when the
`automojo-sysman-common` package's `featurebase` is
updated (a Phase-3 audit fix, a new convention added), the
Ubuntu package picks up the change *automatically* on its
next dependency resolution.  The RHEL package's local
`featurebase` is a copy that has to be updated *manually*
(or the package has to migrate to the shared base, which is
a separate maintenance task).

This is the trade-off that prompted the shared-base pattern.
The cost: Ubuntu can't customise its base.  The benefit:
common improvements arrive uniformly.  Ubuntu's choice
(shared base) is a vote in favour of consistency over
flexibility.

## Tests

41 test files at
`source/testroots/testbase/automojo/tests/sysman/linux/ubuntu/`.

Slightly fewer than RHEL's 53, reflecting:

- Fewer features (40 vs 46).
- Some features have simpler test surfaces (ufw's
  single-zone reduces the test matrix for firewall vs
  firewalld's multiple zones).

The test patterns are the same as RHEL's: command-construction,
idempotency, dry-run, elevation, output-parsing,
error-handling.  The tests run against mock runners in unit
mode; integration tests against real Ubuntu hosts run
periodically.

## Cross-OS feature surface — shared and divergent

A summary of where Ubuntu and RHEL converge and diverge on
features they *both* implement:

| Feature | RHEL tool | Ubuntu tool | Convergence |
|---|---|---|---|
| package-management | dnf + rpm | apt-get + dpkg | Same contract; different tools; different version-pinning syntax |
| service-management | systemctl | systemctl | Both systemd; Ubuntu adds `UbuntuServiceScope` enum for SYSTEM/USER distinction |
| firewall-management | firewalld (zones) | ufw (single "default" zone) | Same contract; Ubuntu uses Convention 5 for non-default zones |
| network-configuration | NetworkManager KeyFile | netplan YAML | Both write persistent files, run a "reload" command |
| storage-management | parted, mkfs, LVM tools | parted, mkfs, LVM tools | Largely identical (Linux-wide tools) |
| user-group-management | useradd, groupadd, usermod | useradd, groupadd, usermod | Identical |
| repository-management | yum/dnf .repo files | apt sources.list.d | Same concept; very different file formats |
| ssh-server-configuration | sshd_config edit + restart | sshd_config edit + restart | Identical |

The convergence pattern: when the *concept* is shared
(package install, service start, user add), the contract
abstracts over both OSes cleanly.  When the *concept*
diverges (firewall zones, network config files), the
contract specifies the abstract verb and Convention 5
handles the cases where one OS can't honour the abstraction.

## What's next

Chapter 6 walks the macOS implementation
(`automojo-sysman-macos`) — 49 features, 45 test files,
and the most distinctive set of OS-specific quirks: Homebrew
elevation inversion (Convention 11), launchd's `gui/<uid>`
domain mapping for user services, pf's per-anchor file
layout, and the macOS hostname triple
(`HostName` / `ComputerName` / `LocalHostName`).
