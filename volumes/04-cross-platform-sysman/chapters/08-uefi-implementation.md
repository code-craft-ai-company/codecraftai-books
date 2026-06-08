# Chapter 8 — UEFI Firmware Implementation

> *UEFI is the smallest sysman package — 5 features, one per
> firmware concern.  But it's structurally distinct: each
> manager detects the running host OS at construction and
> routes every verb through host-specific private methods.
> Single-tier dispatch where the per-OS packages are
> multi-tier.  This chapter walks the structure and the five
> features.*

`codecraftai-sysman-uefi` is *intentionally* the smallest
sysman package.  The entire package is 5 feature managers:
boot management, secure boot, EFI variables, capsule
updates, and ESP management.

The smallness is *the design*.  UEFI is a firmware-tier
concern that exists across all hosts (any modern x86 / ARM
machine has UEFI firmware).  Rather than ship five copies
of each feature (one per host OS), the package ships one
implementation per feature with *internal dispatch* on the
host OS.

## Layout

```
codecraftai-sysman-uefi/source/packages/ccai/sysman/uefi/
├── __init__.py
├── bootmanagement.py        # firmware boot entries + boot order
├── secureboot.py            # Secure Boot state + MOK (Linux)
├── efivariables.py          # raw NVRAM variable read/write
├── capsuleupdate.py         # firmware capsule application
└── espmanagement.py         # EFI System Partition management
```

5 modules.  No `featurebase.py` (imports from
`codecraftai-sysman-common`).  No registry.  Every type prefixed
with `Uefi`.

## The five features

### 1. Boot management

Firmware boot entries (the list of bootable installations
the firmware knows about) and the persistent boot order
(which entry boots by default).

Host-OS dispatch:

- **Linux**: `efibootmgr` (the canonical Linux UEFI tool).
- **macOS**: `bless` + `nvram` — Apple's UEFI doesn't have
  `efibootmgr`; the equivalent is `bless` for setting the
  system's startup disk plus `nvram` for raw EFI variable
  edits.
- **Windows**: `bcdedit` — the Boot Configuration Data
  editor.  Windows hides the firmware-level boot entries
  behind BCD; `bcdedit` is what edits them.

Each verb dispatches based on `_host_os`:

```python
class UefiBootManager(BaseFeatureManager):

    feature_name = "uefi-boot-management"

    def list_boot_entries(self) -> list[UefiBootEntry]:
        if self._host_os == "linux":
            return self._dispatch_linux_list_boot_entries()
        elif self._host_os == "darwin":
            return self._dispatch_darwin_list_boot_entries()
        elif self._host_os == "windows":
            return self._dispatch_windows_list_boot_entries()
        else:
            raise ConfigurationError(...)

    def _dispatch_linux_list_boot_entries(self) -> list[UefiBootEntry]:
        # ... efibootmgr -v ... parse output ...

    def _dispatch_darwin_list_boot_entries(self) -> list[UefiBootEntry]:
        # ... nvram -p ... parse output ...

    def _dispatch_windows_list_boot_entries(self) -> list[UefiBootEntry]:
        # ... bcdedit /enum firmware ... parse output ...
```

Three private methods, one per host OS.  The public verb
selects which to call.  The private methods do the
host-specific argv construction and output parsing.

### 2. Secure Boot

Secure Boot state inspection (is it on?  what mode?) and,
on Linux, MOK (Machine Owner Key) management for kernel
module signing.

Host-OS dispatch:

- **Linux**: `mokutil` + `efivar` — MOK enrolment and EFI
  variable inspection.
- **macOS**: `csrutil` + `bputil` — `csrutil` is the System
  Integrity Protection tool; `bputil` is the Apple Silicon
  "Bypass Utility" for boot policy.
- **Windows**: PowerShell `Confirm-SecureBootUEFI` cmdlet.

The MOK verbs (`enroll_mok_key`, `list_mok_keys`) are
*Linux-only*.  On macOS and Windows they return `FAILED`
with stable detail token (Convention 5):

- macOS: `unsupported_no_mok_concept_macos`.
- Windows: `unsupported_no_mok_concept_windows`.

The contract's verb is reachable from any host; the host's
firmware-level concept just doesn't exist.

### 3. EFI Variables

Raw NVRAM variable read / write / delete / list.  This is
the *low-level escape hatch* for firmware automation that
doesn't fit one of the higher-level features.

Host-OS dispatch:

- **Linux**: `efivar` (or direct
  `/sys/firmware/efi/efivars` filesystem reads).
- **macOS**: `nvram` — Apple's command for NVRAM access.
- **Windows**: PowerShell `[UEFI]` namespace (the
  `Get-SecureBootUEFI` family of cmdlets).

A test that needs to inspect or modify a vendor-specific
NVRAM variable uses the EFI-variables feature.  Most tests
don't need this; the higher-level features
(boot-management, secure-boot) call into EFI variables
internally.

### 4. Capsule Updates

Firmware capsule discovery and application — installing
new firmware versions.

Host-OS dispatch:

- **Linux**: `fwupdmgr` — the LVFS-driven firmware update
  daemon.
- **macOS**: `softwareupdate` — Apple's update tool, with
  special filtering for firmware updates.
- **Windows**: PowerShell `PSWindowsUpdate` module —
  filtered to firmware-class updates.

Capsule updates are *high-stakes* — a failed firmware
update can brick a device.  The contract specifies:

- `list_available_capsules()` (Read) — what firmware
  updates are pending.
- `apply_capsule_file(path)` (Write) — apply a specific
  capsule.  Returns `SUCCESS, requires_reboot=True` per
  Convention 3 (firmware updates apply on reboot).

Implementation drift on macOS is significant: Apple's
`softwareupdate` doesn't accept arbitrary capsule files;
firmware updates flow through the integrated software-update
mechanism.  The macOS implementation of `apply_capsule_file`
returns `FAILED` with stable token
`unsupported_arbitrary_capsule_file_macos`.

### 5. ESP Management

EFI System Partition discovery, mount, and file I/O.  The
ESP is the FAT-formatted partition that holds the
boot-loader files (`grub.efi`, `BootMgfw.efi`,
`bootmgfw.efi.signed`, etc.).

Host-OS dispatch:

- **Linux**: `findmnt` + `mount` for discovery and
  mounting.
- **macOS**: `diskutil` for ESP discovery; the ESP is
  typically auto-mounted at `/Volumes/EFI` on Apple Silicon.
- **Windows**: `mountvol` + PowerShell `Get-Partition` for
  partition table inspection and mounting.

Once mounted, file I/O follows Convention 1 — through the
runner using OS-native file utilities (`cat`, `tee`, etc.
on Linux/macOS; `Get-Content`, `Set-Content` on Windows).

## The single-tier dispatch architecture

A reader comparing UEFI to the per-OS packages (chapters
4–7) will notice the structural difference.  The per-OS
packages are *multi-tier*:

```
codecraftai-sysman-linux-redhat → RHEL-specific managers
codecraftai-sysman-linux-ubuntu → Ubuntu-specific managers
codecraftai-sysman-macos        → macOS-specific managers
codecraftai-sysman-windows      → Windows-specific managers
```

Five packages × N features = many manager classes.

UEFI is *single-tier*:

```
codecraftai-sysman-uefi → 5 managers, each dispatches internally
```

One package, 5 features, internal host-OS dispatch.  The
choice maps to the *nature* of the feature area.  System
management (the per-OS tier) is genuinely OS-specific —
a `dnf install` and a `winget install` are different
*operations*, not just different commands for the same
operation.  Firmware management (UEFI tier) is
*host-OS-agnostic at the firmware level* — the firmware
itself doesn't care whether the OS is Linux or Windows;
only the OS-side tools that talk to the firmware differ.

So:

- Per-OS packages: per-OS managers, one per package.
- UEFI: one manager per feature, dispatching internally.

## The `override_host_os` field — Convention 10

UEFI managers carry a config field documented under
Convention 10:

```python
@dataclass
class UefiBootManagerConfig(FeatureConfig):
    """
        :param override_host_os: When set, forces the dispatch path
                                 to ``"linux"``, ``"darwin"``, or
                                 ``"windows"`` (case-insensitive)
                                 instead of calling
                                 :func:`platform.system`.  Reserved
                                 for testing.
        :param efibootmgr_path: Path / argv-name of ``efibootmgr``.
                                Override only for testing or
                                non-standard installations.
        :param bcdedit_path: Path / argv-name of ``bcdedit``.
        :param bless_path: Path / argv-name of ``bless``.
        :param nvram_path: Path / argv-name of ``nvram``.
    """
    override_host_os: Optional[str] = None
    efibootmgr_path: str = _DEFAULT_EFIBOOTMGR
    bcdedit_path: str = _DEFAULT_BCDEDIT
    bless_path: str = _DEFAULT_BLESS
    nvram_path: str = _DEFAULT_NVRAM
```

`override_host_os` is *testability infrastructure*.  When
set, it forces the dispatch to a specific OS regardless of
what `platform.system()` returns.

A worked example: an SDET on a macOS laptop wants to write
tests for the *Windows* dispatch path of `UefiBootManager`.
Without the override, the manager would dispatch to the
macOS path (`bless` / `nvram`) on the laptop.  With the
override, the manager dispatches to the Windows path
(`bcdedit`).  The test runs `bcdedit`-shaped argvs against
a mock runner; the manager's Windows path is exercised
without needing a Windows host.

```python
windows_config = UefiBootManagerConfig(
    override_host_os="windows",
    bcdedit_path="/path/to/mock-bcdedit",
)

mgr = UefiBootManager(runner=mock_runner, config=windows_config)
result = mgr.list_boot_entries()  # invokes _dispatch_windows_list_boot_entries
```

The same pattern works for testing the Linux and macOS
dispatch paths from any developer machine.  Without the
override, cross-host testing of UEFI would require three
test environments; with the override, one is enough.

The pattern is *unique to UEFI* in the sysman tier.  The
per-OS packages don't need it because their managers
*are* OS-specific — there's no "Linux dispatch path inside
the macOS package."  UEFI's internal dispatch is what makes
the override meaningful.

## Why UEFI is intentionally smaller

A reader might ask: *if UEFI tooling is more complex than
package management (it is), why does the UEFI package have
fewer features?*

The answer is **scope discipline**.  The 5 features are the
ones that:

- Generalise across all firmware (every UEFI host has boot
  entries, secure-boot state, NVRAM, and an ESP).
- Have well-defined cross-OS tooling.
- Are *operationally meaningful* for automation.

What the package *deliberately doesn't* cover:

- **Vendor-specific firmware features.**  Dell PowerEdge
  servers have firmware features Lenovo ThinkSystems don't.
  The UEFI package doesn't try to abstract these; tests
  that need them use vendor-specific tools directly.
- **Firmware development.**  Tools for *creating* UEFI
  firmware (EDK2 build, signing capsules) are not in scope.
  The package is for *operating* firmware, not authoring it.
- **Post-boot OS firmware drivers.**  The OS's interaction
  with firmware after boot (DXE drivers, runtime services)
  is the OS's concern, not the framework's.

The package's `README.md` explicitly states:

> *`codecraftai-sysman-uefi` is the firmware-management tier.
> It covers the cross-host firmware concerns that
> automation can manipulate from a running OS.  Vendor-
> specific firmware features, firmware development, and
> post-boot driver interactions are out of scope.*

The chapter is honest about the boundary.  An automation
campaign that needs vendor-specific firmware features
writes its own helpers; the framework covers what
generalises.

## Tests

The UEFI package has a *higher* test count than its feature
count would suggest — 175 test files.  Why so many for 5
features?

Because the cross-host dispatch matrix multiplies the
coverage:

- Each feature has 3 dispatch paths (Linux / macOS /
  Windows).
- Each path has 5–10 verbs.
- Each verb has multiple tests (command-construction,
  idempotency, dry-run, output-parsing, edge cases).

5 features × 3 paths × ~10 verbs × ~3 tests = ~450
parametrised tests (the file count of 175 reflects how
they're grouped).

The `override_host_os` field is what makes this test count
practical.  Without it, testing the Windows dispatch path
would require a Windows host; testing the macOS path a Mac;
testing the Linux path a Linux box.  With it, the entire
matrix runs on whatever host the CI happens to use — a Linux
runner, a macOS laptop, a Windows VM.

## What's next

Chapter 9 walks the **11 Implementation Conventions** in
full.  Chapters 1–8 have referenced them piecemeal —
Convention 4 (OS-prefixed names), Convention 5 (FAILED with
stable token), Convention 11 (Homebrew elevation
inversion), Convention 10 (`override_host_os`), Convention
6 (elevation_blocked).  Chapter 9 walks all eleven as a
unified discipline, with a worked example for each.

Chapter 10 then closes the volume by walking cross-OS test
parity — how the suite achieves consistent green-across-OSes
test runs at the scale of thousands of tests.
