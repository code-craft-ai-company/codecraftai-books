# Volume 4 — Code-Anchor Index

## codecraftai-sysman-common (chapters 1–3)

### The contracts

- `codecraftai-sysman-common/specs/contracts/README.md` — master index (361 lines).
- `codecraftai-sysman-common/specs/contracts/README.md:75–364` — the **11 Implementation Conventions** (load-bearing for ch.9).
- `codecraftai-sysman-common/specs/contracts/README.md:385–510` — the 50-contract catalogue.
- 50 directories under `specs/contracts/`, each with a `contract.md`.  Examples:
  - `package-management/contract.md`
  - `service-management/contract.md`
  - `network-configuration/contract.md`
  - `firewall-management/contract.md`
  - `encryption-management/contract.md`
  - (45 more)

### Three contract layers

- **Layer 1** (1 contract): `command-execution`.
- **Layer 2** (1 contract): `maintenance-reporting`.
- **Layer 3** (48 contracts): firewall, service, package, network, encryption, etc.

### `featurebase.py` (the shared base)

- `codecraftai-sysman-common/source/packages/ccai/sysman/common/featurebase.py:113–151` — `ActionStatus`, `FeatureConfig`, `FeatureActionResult` canonical types.
- `featurebase.py:154–205` — `BaseFeatureManager` declaration with `result_class`, `status_class`, `feature_name` overridable class variables.
- `featurebase.py:229–300` — `_dispatch(*, command, detail, stdin, operation, elevated_override, **extra)` — the helper enforcing dry-run, elevation, result wrapping.
- `featurebase.py` (366 lines total).

### Discipline anchors

- `codecraftai-sysman-common/AGENTS.md:15–26` — high-level project rules.
- `codecraftai-sysman-common/userguide/10-02-naming-conventions.rst` — OS-prefixed type names.
- `codecraftai-sysman-common/specs/design.md` — package scope (18 lines).

## codecraftai-sysman-linux-redhat (chapter 4)

### Layout

- `codecraftai-sysman-linux-redhat/source/packages/ccai/sysman/linux/redhat/` — 47 `.py` files (excluding `__init__.py`).

### Feature registry

- `featureregistry.py:146–469` — registers **46 features** across three layers.
- `featureregistry.py` uses **lazy imports** for circular-dependency avoidance.

### Representative features

- `packagemanagement.py:143–179` — `PackageManager(BaseFeatureManager)`; uses `dnf` + `rpm`.
- `servicemanagement.py:100–178` — `ServiceManager(BaseFeatureManager)` with `start` / `stop` / `restart`.
- `firewallmanagement.py:63–80` — `FirewallManager(BaseFeatureManager)` with firewalld zone support.

### Tests

- `source/testroots/testbase/ccai/tests/sysman/linux/redhat/` — 53 test files.

### RHEL-specific features (not in Ubuntu)

- `cockpit-management`, `insights-management`, `selinux-management`, `subscription-management`, `tuned-management`.
- RHEL retains *unprefixed* class names (`PackageManager`, not `RhelPackageManager`) for historical reasons.

## codecraftai-sysman-linux-ubuntu (chapter 5)

### Layout

- `codecraftai-sysman-linux-ubuntu/source/packages/ccai/sysman/linux/ubuntu/` — 40 `.py` files.
- Imports `featurebase` from `ccai.sysman.common` (the shared-base pattern).

### Representative features

- `packagemanagement.py:1–79` — `UbuntuPackageManager(BaseFeatureManager)`; uses `apt-get` with `DEBIAN_FRONTEND=noninteractive`.
- `firewallmanagement.py:167–176` — `UbuntuFirewallManager(BaseFeatureManager)`; ufw with single-zone (`"default"`) constraint.
- `networkconfiguration.py:1–80` — `UbuntuNetworkConfigurationManager(BaseFeatureManager)`; netplan + resolvectl + hostnamectl.

### Tests

- 41 test files at `source/testroots/testbase/`.

### Ubuntu-specific differences from RHEL

- `noninteractive_env: bool = True` config field for unattended apt-get.
- ufw single-zone vs firewalld multi-zone.
- netplan YAML drop-ins vs NetworkManager KeyFile.
- `UbuntuServiceScope` enum (SYSTEM vs USER) — distinguishes user-scope services.

## codecraftai-sysman-macos (chapter 6)

### Layout

- `codecraftai-sysman-macos/source/packages/ccai/sysman/macos/` — 49 `.py` files.

### Representative features

- `packagemanagement.py:159–246` — `MacosPackageManager(BaseFeatureManager)`; uses Homebrew with elevation inversion (Convention 11).
- `servicemanagement.py:1–75` — uses `launchctl` with `gui/<uid>` domain mapping.
- `firewallmanagement.py:1–60` — pf via per-anchor files at `/etc/pf.anchors/ccai.<zone>`.
- `networkconfiguration.py:1–45` — `networksetup` + `ifconfig` + BSD `route` + `scutil`.
- `configurationmanagement.py:1–80` — plist via `defaults` + plaintext via `cat`/`tee`.

### Tests

- 45 test files.

### macOS-specific quirks

- **Homebrew refuses root** — wraps with `sudo -u <brew_user>` while keeping `elevated=False`.
- **launchd domains** — `gui/<uid>` for GUI sessions; `system` for system scope.
- **pf anchors** — zones map to `/etc/pf.anchors/ccai.<zone>` files.
- **Hostname triple** — `HostName` / `ComputerName` / `LocalHostName` set via `scutil`.

## codecraftai-sysman-windows (chapter 7)

### Layout

- `codecraftai-sysman-windows/source/packages/ccai/sysman/windows/` — 49 `.py` files.

### Representative features

- `packagemanagement.py:1–50` — `WindowsPackageManager(BaseFeatureManager)`; `winget` with locale-fragile column-position parsing.
- `servicemanagement.py:1–100` — PowerShell `Get-Service` + `ConvertTo-Json -Compress -Depth 5` for stable cross-version parsing.
- `firewallmanagement.py:1–90` — `New-NetFirewallRule` with `codecraftai-svc-<service>` / `codecraftai-port-<port>-<protocol>` rule prefixes.

### Tests

- 45 test files.

### Windows-specific quirks

- **PowerShell JSON serialization** — Status / StartType serialised as integer enum values (Status: 1–7, StartType: 0–5).
- **No masking primitive** — `mask_service` / `unmask_service` aliased to disable / enable.
- **Profile-based firewall** — zones = profiles (Domain, Private, Public, Any); no "default" concept.
- **No reload signal** — `reload_service` returns FAILED; `reload_daemon` is a no-op.
- **`elevation_blocked: bool`** on `RunResult` (Convention 6) — the runner pre-detects elevation status.

## codecraftai-sysman-uefi (chapter 8)

### Layout

- `codecraftai-sysman-uefi/source/packages/ccai/sysman/uefi/` — **5 modules** (intentional).

### The five features

1. `bootmanagement.py` — efibootmgr (Linux), bless+nvram (macOS), bcdedit (Windows).
2. `secureboot.py` — mokutil+efivar (Linux), csrutil+bputil (macOS), Confirm-SecureBootUEFI (Windows).
3. `efivariables.py` — efivar (Linux), nvram (macOS), `[UEFI]` namespace (Windows).
4. `capsuleupdate.py` — fwupdmgr (Linux), softwareupdate (macOS), PSWindowsUpdate (Windows).
5. `espmanagement.py` — findmnt/mount (Linux), diskutil (macOS), mountvol/Get-Partition (Windows).

### UEFI-specific architecture

- **Single-tier dispatch.**  Each UEFI manager detects `platform.system()` once at construction and routes verbs through host-specific `_dispatch_<linux|darwin|windows>_<verb>` methods.
- **`override_host_os` config field** (Convention 10) — testability infrastructure letting any workstation drive any backend.
- 175 test files (cross-host dispatch matrix demands multiplied coverage).

## The 11 Implementation Conventions (chapter 9)

From `codecraftai-sysman-common/specs/contracts/README.md:75–364`:

1. **`ICommandRunner` is the file-I/O seam** — no separate `IFileOperator`; route through `cat` / `tee` / `Get-Content` / `Set-Content`.
2. **`command: list[str] | None` on action results** — every `*ActionResult` records the argv that ran (or would have run for dry-run).
3. **`requires_reboot: bool`** — for boot-time-only state mutations (Windows hostname rename, UEFI capsule).
4. **OS-prefixed concrete type names** — `UbuntuPackageManager`, `MacosPackageManager`, `WindowsPackageManager`.  RHEL is the unprefixed historical exception.
5. **`FAILED` with stable detail token** for unsupported `(host_os, verb)` combos — e.g. `unsupported_on_host_os_darwin`.  Replaces the older `raise NotImplementedError` pattern.
6. **`elevation_blocked: bool` on Windows `RunResult`** — pre-detects elevation status.
7. **Reads degrade gracefully** — return None / empty / synthetic record vs raise; consistent and documented per-OS.
8. **Per-OS backend enums** — abstract `BackupKind` on the contract; OS-specific enums in practice.
9. **Concrete-implementation naming overrides** — clean abstract names on the contract; OS-fitted names in implementations.
10. **UEFI managers carry `override_host_os`** — testability infrastructure.
11. **Elevation-model inversion** when the underlying tool refuses root — Homebrew on macOS being the canonical case.
