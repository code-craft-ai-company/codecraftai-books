# Chapter 7 — Windows Implementation

> *Windows automation is PowerShell.  Every feature manager
> wraps PowerShell cmdlets, parses output via
> `ConvertTo-Json -Compress -Depth 5` for cross-version
> stability, and handles the elevation-blocked condition
> (Convention 6) when the calling process isn't elevated.
> winget's text-format fragility, profile-based firewall
> scoping, no-masking aliasing — Windows brings its own set
> of quirks the contract layer absorbs.*

`automojo-sysman-windows` ships 49 features and 45 test
files.  Like macOS, the package follows the shared-base
pattern with `Windows`-prefixed types throughout.  Unlike
macOS, the Windows package is *uniformly* PowerShell-driven
— almost every feature manager builds a PowerShell
expression as its argv.

## Layout

```
automojo-sysman-windows/source/packages/automojo/sysman/windows/
├── __init__.py
├── packagemanagement.py        # winget
├── servicemanagement.py        # Get/Start/Stop/Set-Service
├── firewallmanagement.py       # NetSecurity module
├── networkconfiguration.py     # Get-NetAdapter / Get-NetIPConfiguration
├── ... (45 more)
```

49 Python files.  Same shared-base pattern as macOS.

A note up front: many Windows feature managers use the same
underlying mechanism (PowerShell + `ConvertTo-Json`).  The
volume of cross-feature mechanism duplication is higher
than on Linux or macOS.  This is consistent with PowerShell
being the canonical Windows automation surface.

## PowerShell + `ConvertTo-Json` — the read pattern

**File:**
`servicemanagement.py:1–100`

```python
"""
    Windows service-management feature.

    Implements the cross-OS service-management contract for
    Windows using the PowerShell ``ServiceController`` cmdlets
    (``Start-Service``, ``Stop-Service``, ``Restart-Service``,
    ``Set-Service``, ``Get-Service``).

    Reads use ``Get-Service ... | ConvertTo-Json -Compress -Depth 5``
        for a single, robust serialization that round-trips
        across PowerShell 5.1 and PowerShell 7+.
    - ``Status`` and ``StartType`` come back as integers
        because ``ConvertTo-Json`` serializes the underlying
        .NET enum values as their numeric encoding.  See
        :func:`_decode_status_int` and
        :func:`_decode_start_type_int` for the mapping.
    - ``mask_service`` / ``unmask_service`` are aliased to
        ``disable`` / ``enable`` because the Windows Service
        Control Manager has no separate masking concept.
    - ``reload_service`` always returns ``FAILED`` because
        Windows services have no native reload signal;
        ``Restart-Service`` is the closest equivalent if a
        reconfigure-with-restart is acceptable.
    - ``reload_daemon`` is a SUCCESS no-op: the Windows
        Service Control Manager applies service definitions
        immediately.
"""
```

Three things define the Windows service manager (and
generalise to most Windows feature managers):

### `ConvertTo-Json -Compress -Depth 5` for reads

PowerShell objects don't serialise consistently across
versions.  Plain `Get-Service | Out-String` produces
human-readable text whose format differs between PowerShell
5.1 (Windows 10's default) and PowerShell 7+ (modern
Windows / Linux PowerShell).

`ConvertTo-Json` produces *machine-readable JSON* that's
stable.  The `-Compress` flag avoids whitespace; `-Depth 5`
ensures nested structures serialise (the default depth is
2, which truncates many objects).

The read pattern across the Windows package:

```powershell
Get-Service -Name nginx | ConvertTo-Json -Compress -Depth 5
```

The Python manager invokes this via
`self._runner.run(["powershell", "-NoProfile", "-Command", "..."])`,
captures stdout, and `json.loads` it.  Cross-version stable.

### Status / StartType as integers

PowerShell's `ConvertTo-Json` serialises .NET enums as
*integers* — the underlying numeric value rather than the
string name.

```python
# Status values for a service:
# 1 = Stopped
# 2 = StartPending
# 3 = StopPending
# 4 = Running
# 5 = ContinuePending
# 6 = PausePending
# 7 = Paused

def _decode_status_int(status_int: int) -> str:
    return {
        1: "stopped", 4: "running", 7: "paused",
        # ... transitional states ...
    }.get(status_int, "unknown")
```

The manager has decoder functions for every cmdlet that
returns enum-typed fields.  An engineer reading the manager
sees the decoder before the verb implementation; the
discipline keeps the integer semantics out of the verb
logic.

### Aliasing where the OS doesn't support a concept

The Windows Service Control Manager has no separate
"masking" concept the way Linux systemd does.  systemd's
`mask` is a stronger form of `disable` — a masked service
can't be started even by transient activation.  Windows
SCM doesn't distinguish.

The Windows manager handles this via Convention 5
(supported here as a *naming alias* rather than a FAILED
return):

```python
def mask_service(self, name: str, scope: ServiceScope) -> ServiceActionResult:
    """
        Aliased to disable_service on Windows; SCM has no separate
        masking primitive.
    """
    return self.disable_service(name, scope)


def unmask_service(self, name: str, scope: ServiceScope) -> ServiceActionResult:
    """
        Aliased to enable_service on Windows.
    """
    return self.enable_service(name, scope)
```

A test that calls `mask_service` on Windows gets the same
behaviour as `disable_service`.  The contract verb is
honoured (a result is returned with `status=SUCCESS`); the
underlying OS just doesn't have the granularity systemd
does.

`reload_service` on Windows returns `FAILED` with stable
detail token `unsupported_no_reload_signal`.  Per Convention
5; the contract verb is reachable; the OS doesn't support
it.

## winget's locale-fragile parsing

**File:**
`packagemanagement.py:1–50`

```python
"""
    Windows package-management feature.

    Output-parsing fragility:
        ``winget`` produces text output for ``list`` /
        ``show`` / ``upgrade`` rather than a stable
        machine-readable form.  The column boundaries and
        dash-separator rows differ across ``winget`` versions
        and Windows locales (the column headers and the dash
        row are localised).  This module's parsing strategy is:

            1. Locate the dash-separator row (``-`` runs
               interleaved with whitespace) which appears once
               after the header row and never inside the data
               rows.
            2. Use the column starts inferred from the header
               row to slice each subsequent data row by
               character position; fall back to whitespace
               splitting when a row is shorter than expected.
            3. Skip lines that are obvious progress / footer noise.
"""
```

winget is the Windows Package Manager.  Unlike `Get-Service`
(which produces JSON-friendly objects), winget produces
*text* output for `list` and `show` commands.  Worse, the
text format is *localised* — column headers appear in the
user's language; the dash-separator row varies in width.

The Windows package manager's parsing strategy:

1. Find the dash row.  The first row of all-dashes (with
   spaces between dash runs marking column boundaries) is
   the column-position anchor.
2. From the header row above the dashes, compute each
   column's start position.
3. Slice each data row by character position.
4. Fall back to whitespace splitting when a row is shorter
   than the header would suggest.

The code is a small parser explicitly designed for
fragility tolerance.  Tests cover representative output
from multiple winget versions and locales.

A note in the chapter's honesty section: *future winget
versions are expected to add a JSON output mode*.  When
that ships, the manager will switch.  Until then, the
column-position parser is the pragmatic answer.

## NetSecurity module for firewall

**File:**
`firewallmanagement.py:1–90`

```python
"""
    Windows firewall-management feature.

    Implements the cross-OS firewall-management contract for
    Windows using the PowerShell ``NetSecurity`` module
    (``New-NetFirewallRule``, ``Remove-NetFirewallRule``,
    ``Get-NetFirewallRule``, ``Set-NetFirewallProfile``,
    ``Get-NetFirewallProfile``, ``Get-NetConnectionProfile``).

    Zone mapping:
        The contract's ``zone`` argument maps directly onto a
        Windows Firewall profile name: one of ``"Domain"``,
        ``"Private"``, ``"Public"``, or ``"Any"``.  Any other
        value yields a ``FAILED`` result with
        ``message="invalid_zone"`` BEFORE the runner is
        invoked.

    Verb behaviour highlights:
        * ``set_default_zone`` is a SKIPPED no-op when the
          zone is valid.  Windows has no "default profile"
          concept -- profiles activate based on the network
          category that the OS assigns to the current
          connection.
        * ``query_state`` reports ``running=True`` iff every
          profile's ``Enabled`` field is True; ``default_zone``
          is taken from ``Get-NetConnectionProfile``'s
          ``NetworkCategory`` for the active connection,
          which maps directly onto a profile name.
"""
```

Windows Firewall has *profiles* (Domain, Private, Public,
Any) rather than zones.  The contract's `zone` argument
maps directly to a Windows profile name.

A few interesting differences from firewalld and ufw:

### No default zone

Windows doesn't have a "default profile."  Which profile is
active depends on the *network category* the OS assigns to
the current connection — automatically, based on the
network's properties.  An office network gets `Domain`;
home Wi-Fi gets `Private`; coffee shop Wi-Fi gets `Public`.

The contract's `set_default_zone` verb is therefore a
no-op on Windows — there's nothing to set.  The manager
returns `SKIPPED` to acknowledge the call without
modifying state.

### Rule prefixing for round-trip queries

```python
# When add_service is called for service "ssh" in zone "Private":
ps_cmd = """
New-NetFirewallRule
    -DisplayName 'automojo-svc-ssh'
    -Profile Private
    -LocalPort 22 -Protocol TCP
    -Action Allow
"""
```

Rules created by the manager are prefixed:

- `automojo-svc-<service>` for service-based rules.
- `automojo-port-<port>-<protocol>` for explicit port
  rules.

The prefix lets the manager:

- Query "is this service enabled?" by listing rules whose
  DisplayName starts with `automojo-svc-`.
- Avoid conflicts with rules created by Group Policy or
  the user via the GUI.

A bundled service-name table maps service names to ports:
ssh (22/tcp), http (80/tcp), https (443/tcp), smtp
(25/tcp), dns (53/udp), smb (445/tcp), rdp (3389/tcp).

## Convention 6 — the elevation-blocked check

**File:** referenced from
`automojo-sysman-common/specs/contracts/README.md:185–200`

Convention 6: *the Windows `LocalCommandRunner` pre-detects
elevation status*.  When a manager passes `elevated=True`
but the calling process isn't elevated, the runner returns
a `RunResult` with:

- `status = FAILED`.
- An additional `elevation_blocked = True` field.
- A clear stderr message explaining the cause.

The runner *doesn't* attempt to elevate (Windows UAC isn't
something the runner can drive non-interactively).  It
*signals* the manager that the elevation request can't be
honoured.

The manager catches the `elevation_blocked` flag and:

- Returns the standard `FAILED` result to the caller.
- Adds `elevation_blocked=True` to the manager's
  `*ActionResult` so the test can detect the cause.

A test running unelevated Windows can:

```python
result = service_mgr.start("nginx")
if result.elevation_blocked:
    pytest.skip("This test requires elevated PowerShell.")
```

The check is what makes Windows tests run cleanly under
both elevated and unelevated processes — the test discovers
its own constraints.

## Other Windows quirks

A few more distinctive Windows features:

### Hostname rename requires reboot (Convention 3)

The Windows hostname-management feature renames the
computer using `Rename-Computer`.  The change *doesn't take
effect until reboot*.  The manager returns
`status=SUCCESS, requires_reboot=True` per Convention 3;
test code can decide whether to schedule a reboot or treat
the change as pending.

### Registry access for some features

Some Windows features that don't have first-class PowerShell
cmdlets reach into the registry:

```python
# environment-configuration writes user/system env vars
# via PowerShell's [Environment] static class:
ps_cmd = (
    "[Environment]::SetEnvironmentVariable("
    "    'MY_VAR', 'value', 'Machine')"
)
```

The manager builds the `[Environment]` invocation rather
than poking the registry directly.  PowerShell's class
abstracts the underlying registry mechanics.

### winget's unattended flags

Every write verb's argv includes:

```
--silent --accept-package-agreements --accept-source-agreements
```

These flags suppress winget's interactive prompts (license
acceptance, source addition).  Without them, an unattended
test run hangs waiting for the operator to press a key.
The manager always applies them; tests don't have to
remember.

## Tests

45 test files at `source/testroots/testbase/`.

Test patterns mirror RHEL / Ubuntu / macOS:
command-construction, idempotency, dry-run, elevation
(including elevation_blocked), output parsing.  Windows
tests have a few additional categories:

- **JSON serialisation tests** — verify that `_decode_*_int`
  decoders handle all enum values.
- **winget output-parsing tests** — sample outputs from
  multiple versions / locales.
- **Profile-based firewall tests** — verify each of
  Domain / Private / Public / Any maps correctly.

Mock-runner pattern again — tests don't actually invoke
PowerShell; they construct the expected argv and verify
the manager produces it.

## What's next

Chapter 8 walks the UEFI implementation
(`automojo-sysman-uefi`) — 5 features intentionally minimal,
single-tier dispatch where each manager detects the host OS
once and routes verbs through host-specific private methods,
the `override_host_os` config field as testability
infrastructure (Convention 10), and the firmware-tier scope
decision that explains why UEFI is so much smaller than the
per-OS packages.
