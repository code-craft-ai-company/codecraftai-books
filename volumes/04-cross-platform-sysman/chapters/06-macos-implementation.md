# Chapter 6 — macOS Implementation

> *macOS is the most quirky OS in the suite.  Homebrew
> refuses root.  launchd's "user services" run in
> `gui/<uid>` domains, not bare-user domains.  pf has no
> first-class zones — they're emulated via per-anchor files.
> The macOS hostname is a triple, not a string.  This chapter
> walks `codecraftai-sysman-macos` with each quirk explicit.*

`codecraftai-sysman-macos` ships 49 features and 45 test files.
The package follows the patterns Ubuntu established — shared
`featurebase`, OS-prefixed class names — but adds more
implementation-specific peculiarities than any other OS in
the suite.  Convention 11 (elevation-model inversion) was
*invented* for macOS.

## Layout

```
codecraftai-sysman-macos/source/packages/ccai/sysman/macos/
├── __init__.py
├── packagemanagement.py        # Homebrew (Convention 11)
├── servicemanagement.py        # launchctl + gui/<uid> domains
├── firewallmanagement.py       # pf with anchor-file emulation
├── networkconfiguration.py     # networksetup + scutil hostname triple
├── configurationmanagement.py  # plist via defaults
├── ... (44 more features)
```

49 Python files in the implementation directory.  Like
Ubuntu, no local `featurebase.py` — imports from
`ccai.sysman.common`.  All public types are
`Macos`-prefixed.

## Homebrew elevation inversion (Convention 11)

The most-discussed macOS quirk.  Homebrew **refuses to run
as root**.  If a Homebrew command is invoked under root, it
errors out:

```
Error: Running Homebrew as root is extremely dangerous and no
longer supported.
```

This breaks the standard sysman elevation model.  The
contract says "writes elevate when `config.elevated=True`."
Homebrew's writes can't elevate — they have to *de-elevate*.

The solution (Convention 11): pass `elevated=False`
unconditionally to the runner, and wrap the argv with
`sudo -u <brew_user>` when the calling process is root.

**File:**
`packagemanagement.py:159–246`

```python
class MacosPackageManager(BaseFeatureManager):
    """
        macOS Homebrew package manager.  Every verb is dispatched
        through the injected :class:`ICommandRunner`.

        **Homebrew never runs as root.**  Every argv assembled by
        this manager is dispatched with ``elevated=False`` via
        :meth:`_dispatch_brew`, regardless of ``config.elevated``.
        When the calling process is root the caller must populate
        :attr:`MacosPackageManagerConfig.brew_user` so the manager
        can wrap every argv with ``sudo -u <brew_user>`` and
        execute ``brew`` under a non-root identity.
    """

    result_class = MacosPackageActionResult
    status_class = MacosPackageActionStatus
    feature_name = "package-management"
```

The `brew_user` config field is the operator's *non-root
identity* on the host.  When the test process is running as
root (typical on CI hosts that need elevation for other
features), the manager:

1. Detects it's running as root.
2. Looks at `config.brew_user`.
3. Prepends `["sudo", "-u", brew_user]` to the brew argv.
4. Calls `self._runner.run(prefixed_argv, elevated=False)`.

The runner's `elevated=False` means the `sudo -u` is the
*only* privilege transition — root drops to `brew_user`,
runs brew as that user, returns.

When the test process is *not* running as root, the manager
just calls brew directly with `elevated=False`.

The discipline is documented and the manager validates:
running as root *without* `brew_user` set raises
`ConfigurationError` at construction time.  An operator who
forgets to configure the brew_user gets a clear error rather
than a confusing brew-refuses-root failure mid-run.

This is the canonical example of the **elevation-model
inversion** Convention 11 names.

## launchd domains for service management

**File:**
`servicemanagement.py:1–75`

```python
"""
    macOS service-management feature.

    Implements the cross-OS service-management contract for macOS
    using ``launchctl``.  Every verb is dispatched through an
    :class:`ICommandRunner` so the manager works identically under
    local and SSH execution.

    Scope mapping:
        ``MacosServiceScope.SYSTEM`` -> domain ``"system"``
        ``MacosServiceScope.USER``   -> domain ``f"gui/{config.target_uid}"``

    The ``user/<uid>`` background-user domain is also a valid
    macOS launchd domain target.  This pilot uses ``gui/<uid>``
    as the primary user-scope domain because GUI-session agents
    are by far the most common user-scoped target on macOS
    workstations.
"""
```

macOS's launchd uses *domains* to scope services:

- `system` — system-wide services (LaunchDaemons).
- `gui/<uid>` — services in a user's GUI session
  (LaunchAgents).
- `user/<uid>` — services in a user's background session.

Linux's systemd has a simpler model (`--user` flag).  The
contract specifies a `ServiceScope` enum (`SYSTEM` vs
`USER`); macOS implements it via launchctl's domain
targeting.

The Ubuntu manager maps `ServiceScope.USER` to systemctl's
`--user` flag.  The macOS manager maps it to
`gui/<uid>` because user-scoped services on macOS are
overwhelmingly GUI-session agents (menubar apps, login
items).  The `user/<uid>` background domain is supported by
launchd but rarely the right target; the macOS manager
chooses `gui/<uid>` as the default for `USER` scope and
documents the choice.

A test that wants the background domain explicitly can pass
a config with the alternative; the manager honours it.  The
default tracks the common case.

## pf firewall via anchor files

**File:**
`firewallmanagement.py:1–60`

```
Zone mapping: macOS pf has no first-class zone concept, but it
does have anchor namespaces.  This manager maps each contract
``zone`` to a pf anchor named ``ccai.<zone>``; the anchor's
rules live in ``/etc/pf.anchors/ccai.<zone>``.

Surface choices:
    * Each ``add_*`` / ``remove_*`` verb reads the current anchor
      file via ``cat``, computes the desired post-state, and
      either rewrites the file via ``tee`` (followed by a
      per-anchor ``pfctl -a`` reload) or returns ``SKIPPED``
      when the post-state already matches.
```

macOS's pf (Packet Filter) doesn't have zones the way
firewalld does.  pf has *anchors* — named rule sub-tables
that can be loaded, listed, flushed independently.

The macOS manager maps the contract's `zone` argument to
pf anchor names with a fixed prefix:

- Contract zone `"public"` → pf anchor `"ccai.public"`.
- Contract zone `"internal"` → pf anchor `"ccai.internal"`.

Anchor rules live in `/etc/pf.anchors/ccai.<zone>`.
The manager reads-modify-writes the file:

1. `cat /etc/pf.anchors/ccai.public` → current rules.
2. Compute the desired post-state.
3. If matches, return SKIPPED.
4. Otherwise `tee /etc/pf.anchors/ccai.public` →
   write the new rules.
5. `pfctl -a ccai.public -f /etc/pf.anchors/ccai.public`
   → reload that anchor.

The reload-per-anchor scoping prevents the manager's
changes from disturbing rules in *other* pf anchors that
non-ccai tools maintain.

A separate concern: pf needs `/etc/pf.conf` to *include*
the ccai.<zone> anchors; otherwise pf doesn't load
them.  The manager doesn't manage `/etc/pf.conf` — that's
the operator's setup task.  The chapter is honest that the
sysman tier can't substitute for initial system
configuration; it can only manage *changes* to the firewall
rules, given an already-configured pf.conf that includes
the anchors.

## networksetup + scutil for network configuration

**File:**
`networkconfiguration.py:1–45`

```python
"""
    Implements the cross-OS network-configuration contract for
    macOS using ``networksetup`` (the canonical persistent
    surface, which writes through System Configuration /
    ``configd``), ``ifconfig`` for runtime interface detail,
    BSD ``route`` for routes, and ``scutil`` for the macOS
    hostname triple (``HostName`` / ``ComputerName`` /
    ``LocalHostName``).
"""
```

macOS's network configuration is split across multiple
tools, each with a specific role:

- **`networksetup`** — the canonical *persistent*
  configuration tool.  Writes through the System
  Configuration framework (`configd`); changes survive
  reboot and are visible to the GUI Network Preferences
  pane.
- **`ifconfig`** — runtime interface state.  Read-mostly;
  changes don't persist.
- **BSD `route`** — runtime route table.  Persistent route
  management is *out of scope* on macOS; runtime routes
  are what the manager handles.
- **`scutil`** — the macOS *hostname triple*.

### The hostname triple

macOS has *three* hostnames:

- **`HostName`** — the BSD hostname (returned by
  `hostname` command).
- **`ComputerName`** — the friendly name shown in System
  Preferences.
- **`LocalHostName`** — the Bonjour mDNS name (e.g.,
  `MyMac.local`).

The contract has one `set_hostname(name)` verb.  The
macOS manager sets *all three* using `scutil`:

```python
def set_hostname(self, name: str) -> MacosNetworkActionResult:
    # ... query existing values ...
    # ... idempotency check ...

    cmds = [
        ["scutil", "--set", "HostName", name],
        ["scutil", "--set", "ComputerName", name],
        ["scutil", "--set", "LocalHostName", name],
    ]
    # ... dispatch each in sequence ...
```

The three are set together because they typically *should*
match.  An operator who needs them to differ (rare —
typically only on Bonjour-conscious lab gear) bypasses the
contract and uses `scutil` directly.

The *contract specifies one verb*; the *implementation*
maps it to three commands; the *result* aggregates the
outcomes.  This is the "manager handles multi-step
operations" pattern from chapter 3.

## Configuration management — plist routing

**File:**
`configurationmanagement.py:1–80`

```python
"""
    Implements the cross-OS configuration-management contract for
    macOS by routing each call by the source path:

    * Apple plist files (path ends in ``.plist`` OR lives under
      ``/Library/Preferences/`` or ``~/Library/Preferences/``)
      are manipulated with ``defaults``.
    * Every other path is treated as a plain shell-style
      ``KEY=VALUE`` configuration file and is manipulated with
      ``cat`` / ``tee``.

    Plist-type inference (write):
        ``"true"``/``"false"`` -> ``-bool``; all-digit strings ->
        ``-int``; everything else -> ``-string``.  Floats, arrays,
        and nested dicts must be written via
        :meth:`reload_configuration` + an externally rendered
        plist; the limitation is documented in
        ``specs/3-configuration-management/tasks.md``.
"""
```

macOS configuration files come in two shapes:

- **plist files** (Property List XML or binary) under
  `/Library/Preferences/` and similar.  Manipulated by
  `defaults` command.
- **shell-style `KEY=VALUE` files** for shell environments,
  cron tables, etc.  Manipulated by `cat` / `tee`.

The macOS configuration manager *routes by path*: ends in
`.plist` or lives in a Preferences directory → `defaults`;
otherwise → plain text.  A test setting a configuration
key writes the same `set_value(path, key, value)` call;
the manager picks the right tool.

**Plist type inference** is the macOS-specific quirk.
`defaults write` requires a type flag (`-string`, `-bool`,
`-int`, `-float`, etc.).  The manager infers from the value:

- `"true"` / `"false"` → `-bool true` / `-bool false`.
- All-digits → `-int`.
- Otherwise → `-string`.

Floats, arrays, and nested dicts are *out of scope* for the
macOS configuration manager.  The implementation drift
section of the contract (chapter 2) documents this; tests
that need rich plist values use `reload_configuration`
with an externally rendered plist file.

The boundary is honest.  `defaults` works well for simple
key-value plist edits.  Complex plist structures need a
different tool (`PlistBuddy`, custom XML rendering); the
contract acknowledges the limit rather than pretending the
manager handles every case.

## Other macOS-specific features

A short tour of the other distinctive macOS features:

### `launchctl` user-domain reload

`reload_daemon` on Linux means `systemctl daemon-reload`.
On macOS, it means `launchctl bootstrap`/`bootout` cycles
to refresh the launchd domain's known unit list.  The
implementation differs because launchd's reload semantics
are different — same contract verb, different mechanics.

### `csrutil` for system integrity protection

The macOS `security-compliance` feature exposes SIP
(System Integrity Protection) state via `csrutil status`.
SIP is a recovery-mode-only setting (you can't disable it
from a running OS), so the manager's
`get_security_state()` reads but `set_security_state()`
returns FAILED with a stable token —
`unsupported_runtime_change_macos`.

### `softwareupdate` for system updates

The macOS `package-management` feature is *Homebrew*.  But
*system* updates (macOS itself, security patches) go
through `softwareupdate`.  The macOS package adds a
separate manager for this — `MacosSystemUpdateManager` —
that's not in the cross-OS contract surface (no
`software-update` contract exists; system updates vary too
much across OSes to abstract).

This is honest scoping.  Some OS-specific concerns don't
generalise; they ship as OS-specific features without
contract participation.

## Tests

45 test files at `source/testroots/testbase/`.

The test count is comparable to Ubuntu's 41 and Windows's
45.  The macOS tests cover the same patterns
(command-construction, idempotency, dry-run, elevation,
output-parsing) plus *more* per-feature edge cases because
of the OS-specific quirks:

- Homebrew root-detection tests verify the
  `brew_user` requirement and the `sudo -u` wrapping.
- launchctl domain-mapping tests verify
  `MacosServiceScope.USER` produces the correct
  `gui/<uid>` argv.
- pf anchor-file tests verify the read-modify-write
  flow against mock file contents.
- scutil hostname-triple tests verify all three names get
  set in the correct order.

The mock-runner pattern from chapter 4 still applies; tests
don't actually invoke brew / launchctl / pfctl / scutil.
They verify the manager constructs the right argvs.

## What's next

Chapter 7 walks the Windows implementation
(`codecraftai-sysman-windows`) — 49 features, 45 test files,
the PowerShell-driven mechanism throughout (cmdlets +
`ConvertTo-Json`), winget's locale-fragile output parsing,
the elevation-blocked detection (Convention 6), and the
profile-based firewall model that maps the contract's
`zone` to Windows Firewall profiles (Domain / Private /
Public / Any).
