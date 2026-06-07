# Chapter 14 — Configuration Profiles as a Discipline

> *A profile is a name plus a `MergeMap`.  This chapter proposes the
> convention that turns the existing `codecraftai-configuration`
> machinery into a user-facing profile system.*

The previous chapter established the four-environment ladder and
named the discipline that lets a single codebase scale across it.
This chapter is the mechanism: what a profile actually is, how
profiles compose, how selection happens at process start, what
piece of new tooling the discipline asks the reader to build, and
where each piece sits relative to the shipped portfolio
primitives.

The shipped substrate is half of the answer.  The shipped
`AM_*_NAMES` family already drives the per-category configuration
loading.  The shipped `MergeMap` already supports the layered-with-
overlay semantics profile composition needs.  The shipped
`override_variable_value` already provides the runtime override
primitive.  What's missing is the *user-facing convention* that
binds them — a profile YAML format, a process-start selection
mechanism, an introspection CLI.  This chapter proposes the
convention and shows how to wire it on top of the existing
machinery.

## What a profile is

A profile is a **named bundle of category selections** plus an
optional **overlay of per-category value patches**.  In the
simplest form, a profile is a YAML document like:

```yaml
# config/profiles/local-dev.yaml
profile:
    name: local-dev
    description: Single-host developer workstation

    select:
        landscape: local-dev
        topology: local-dev
        runtime: local-dev
        credentials: local-dev
        dataprofiles: local-dev
```

Six fields under `select:`, one per category — five for the
shipped `AM_CONFIG_*_NAMES` family, plus an entry for the
out-of-band Specification category from chapter 4.  Each field's
value is a name (or comma-separated list of names) that the
loader resolves against the corresponding category's directory.

The profile says: *for the local-dev environment, load the
landscape named `local-dev`, the topology named `local-dev`, the
runtime named `local-dev`*, and so on.  When the orchestration
starts with `AM_PROFILE=local-dev`, the profile resolver reads
this file and translates each `select:` entry into the
corresponding `AM_*_NAMES` setting.  After that translation, the
shipped configuration machinery takes over unchanged — the
discipline is *additive* on top of what already works.

## Profile composition: `extends`

A second profile, `staging-east`, can inherit from a `base`
profile and override only what differs:

```yaml
# config/profiles/base.yaml
profile:
    name: base
    description: Conservative defaults for any environment

    select:
        landscape: base
        topology: default-flat
        runtime: default
        credentials: base
        dataprofiles: base
```

```yaml
# config/profiles/staging-east.yaml
profile:
    extends: base
    name: staging-east
    description: Staging east region — production-shaped

    select:
        landscape: staging-east
        topology: staging-east-leader-follower
        runtime: staging-east
        credentials: staging-east
        dataprofiles: staging-east
```

The `extends:` field is the parent profile name.  Resolution walks
the chain top-down: load `base`, then layer `staging-east`'s
`select:` entries on top.  Any field absent from the child
inherits from the parent; any field present in the child shadows
the parent's value.

The chain has no fixed depth.  A `staging-east-canary` profile
extends `staging-east`; a `staging-east-canary-slow-cell` profile
extends `staging-east-canary`; the resolution still terminates
because the chain is acyclic and bounded.  The resolver detects
cycles at load time and raises a clear `ConfigurationError` naming
the cycle.

```yaml
# config/profiles/staging-east-canary.yaml
profile:
    extends: staging-east
    name: staging-east-canary
    description: Smaller-pod canary slice of staging-east

    select:
        landscape: staging-east-canary    # smaller landscape
        runtime: staging-east-canary      # tighter timeouts
        # topology, credentials, dataprofiles all inherited
```

## Profile composition: `overlay`

Profile composition is for selecting *which* configuration to
load.  Sometimes a profile needs to **patch values within an
already-selected configuration** without forking the configuration
file.  The `overlay:` field is for those cases:

```yaml
# config/profiles/staging-east-debug.yaml
profile:
    extends: staging-east
    name: staging-east-debug
    description: Staging-east with verbose logging for incident response

    # Same selections as staging-east; overlay patches specific values
    overlay:
        runtime:
            logging:
                levels:
                    console: DEBUG    # was INFO; bumped for this run
                    logfile: TRACE    # was DEBUG; bumped for this run
            timeouts:
                default_seconds: 60   # was 30; doubled for slow debugging
```

The overlay is a partial document with the same shape as the
named configuration it patches.  At resolution time the
profile system loads `runtime: staging-east` (inherited from
`staging-east`) and *layers the overlay on top* via the same
`MergeMap` machinery the shipped categories already use.  The
result is `staging-east`'s runtime with three keys replaced; the
unchanged keys flow through unchanged.

Overlays are powerful and come with a cost: an overlay diverges
the active runtime from any *named* runtime, which makes the
configuration harder to identify in retrospective analysis.  The
discipline:

- **Use `select:` first.**  When the change you want exists as a
  named configuration somewhere, select it.  A `runtime:
  staging-east-verbose` is more discoverable than an overlay that
  bumps a few log levels.
- **Use `overlay:` for one-off and incident-response cases.**  A
  profile that bumps a timeout for a specific debugging session
  doesn't need to become a permanent configuration file.  The
  overlay says "this run is special" without polluting the named
  catalogue.
- **Keep overlays small.**  An overlay that patches twenty fields
  has crossed the line into "this is a different runtime";
  promote it to a named runtime configuration.

## The proposed `AM_PROFILE` selector

The shipped `AM_*_NAMES` family selects per category.  A profile
is a *bundled* selection, so the natural extension is one more
environment variable that names the profile to load:

```bash
AM_PROFILE=staging-east ccai testbase test run tests/
```

When `AM_PROFILE` is set at process start, the profile resolver
runs *before* the existing configuration loading.  Its job is
mechanical:

1. Read `config/profiles/${AM_PROFILE}.yaml`.
2. Walk the `extends:` chain to assemble the final `select:` map.
3. For each entry in the merged `select:`, set the corresponding
   `AM_CONFIG_*_NAMES` variable by calling
   [`override_variable_value`](../../../codecraftai-this-app/source/packages/ccai/thisapp/thisapplication.py)
   on the appropriate descriptor.
4. For each entry in the merged `overlay:`, register the patch
   with the configuration loader so that it gets composed onto the
   loaded configuration via `MergeMap` after the named
   configuration is loaded.
5. Set `ContextPaths.ACTIVE_PROFILE` (a proposed context-path
   addition; see below) to the resolved profile name.

After step 5, control hands off to the existing configuration
loaders.  Each `resolve_*_configuration` function in
[`configurationmaps.py`](../../../codecraftai-configuration/source/packages/ccai/configuration/configurationmaps.py)
runs unchanged — it reads the `AM_CONFIG_*_NAMES` variables the
profile resolver just set, loads the named files, and inserts the
results into the shared application context.

The seam is small: the profile resolver is a thin layer that
*translates* a profile name into a concrete set of `AM_*_NAMES`
values plus a small overlay-patch registration.  It doesn't
replace the loaders; it *parameterises* them.

## The proposed `ACTIVE_PROFILE` context path

The shipped
[`ContextPaths`](../../../codecraftai-this-app/source/packages/ccai/thisapp/contextpaths.py)
enumeration has slots for `JOB_ID`, `JOB_NAME`, `PIPELINE_ID`,
`RUN_ID`, and similar process-identity fields.  It does not have
a slot for the active profile name.  Adding one is a one-line
change:

```python
class ContextPaths(str, Enum):
    # ... existing entries ...
    ACTIVE_PROFILE = "/runtime/active_profile"
```

The profile resolver inserts the resolved profile name at this
context path during step 5 of the resolution flow.  Code that
needs to know which profile is active reads it via the
`ThisApplicationSingleton().context.lookup(ContextPaths.ACTIVE_PROFILE)`
pattern the rest of the platform already uses.

Three reasons to put the profile name in `ContextPaths` rather
than leaving it as just an environment variable:

- **Distributed orchestrations** (Volume 2's territory) propagate
  the application context to remote workers as part of their
  setup.  Putting the profile name in the context means remote
  workers see the same profile name without needing to read the
  coordinator's `os.environ`.
- **Logging frameworks** that read context paths automatically
  pick up the profile name and include it in every log event.
- **The "never log a credential" defences from chapter 11** scan
  context-path values for sensitive content; having the profile
  name in a known location makes the audit trail richer without
  expanding the search surface.

## The proposed `ccai profile show` CLI

The single most-asked question once the profile system is in
place is: *what is the resolved configuration of the active
profile?*  The shipped portfolio has the building blocks for the
answer (`AppVarSet.items()` enumerates active variables and their
resolved values) but no CLI command exposes it.  The discipline
proposes one.

The proposed command:

```bash
ccai profile show
```

Output (for `AM_PROFILE=local-dev`):

```
Profile:        local-dev
Description:    Single-host developer workstation

Selections:
    landscape:      local-dev          (config/landscape/landscape-local-dev.yaml)
    topology:       local-dev          (config/topology/topology-local-dev.yaml)
    runtime:        local-dev          (config/runtime/runtime-local-dev.yaml)
    credentials:    local-dev          (config/credentials/credentials.yaml#local-dev)
    dataprofiles:   local-dev          (config/dataprofiles/dataprofiles-local-dev.yaml)

Overlays:
    (none)

Resolved variables:
    AM_CONFIG_LANDSCAPE_NAMES        local-dev
    AM_CONFIG_TOPOLOGY_NAMES         local-dev
    AM_CONFIG_RUNTIME_NAMES          local-dev
    AM_CONFIG_CREDENTIAL_NAMES       local-dev
    AM_CONFIG_DATAPROFILE_NAMES      local-dev
    AM_CONFIG_USE_LANDSCAPE          true
    AM_CONFIG_USE_TOPOLOGY           true
    AM_CONFIG_USE_RUNTIME            true
    AM_CONFIG_USE_CREDENTIALS        true
    AM_CONFIG_USE_DATAPROFILES       true
    AM_LOG_LEVEL_CONSOLE             DEBUG
    AM_LOG_LEVEL_LOGFILE             DEBUG
    ...

Resolved files:
    landscape:      /Users/dev/proj/config/landscape/landscape-local-dev.yaml
    topology:       /Users/dev/proj/config/topology/topology-local-dev.yaml
    runtime:        /Users/dev/proj/config/runtime/runtime-local-dev.yaml
    credentials:    /Users/dev/proj/config/credentials/credentials.yaml
    dataprofiles:   /Users/dev/proj/config/dataprofiles/dataprofiles-local-dev.yaml
```

Three sections matter:

- **Selections** — what categories the profile selects, plus the
  resolved file path for each.  The reader can see at a glance
  which configurations are in scope for this run.
- **Resolved variables** — every active `AppVarDesc` and its
  current value (with credential material elided per the
  "never log" rule).  This is the answer to "what is the
  effective parameterisation of this run?"
- **Resolved files** — the concrete paths the loader is reading.
  Useful for the chapter-15 use case ("which file do I edit to
  change this?").

Implementation sketch:

```python
class ProfileShowCommand(Command):
    def execute(self, context):
        this_app = ThisApplicationSingleton()
        active_profile = this_app.context.lookup(
            ContextPaths.ACTIVE_PROFILE,
            default="<no profile active>",
        )

        print(f"Profile:        {active_profile}")
        print(f"Description:    {self._get_description(active_profile)}")
        print()

        # ... dump selections, overlays, variables, files
```

The command lives in the `ccai` CLI tree alongside the existing
`ccai testbase`, `ccai am-interop` commands.  The
implementation is roughly a hundred lines and reuses
`AppVarSet.items()` plus the chapter-9 redacting filter to
produce the variables section safely.

## A worked four-file profile set

To illustrate composition end-to-end, here are four profile files
that together describe a real four-environment ladder:

### `base.yaml`

```yaml
# config/profiles/base.yaml
profile:
    name: base
    description: Conservative defaults that every environment inherits

    select:
        landscape: base
        topology: default-flat
        runtime: default
        credentials: base
        dataprofiles: base
```

`base` is the safety net.  It selects conservative configurations
that will run *somewhere* but not optimally anywhere.  It exists
so that every other profile can `extends: base` without having to
copy a complete `select:` block.

### `local-dev.yaml`

```yaml
# config/profiles/local-dev.yaml
profile:
    extends: base
    name: local-dev
    description: Single-host developer workstation

    select:
        landscape: local-dev
        topology: local-dev
        runtime: local-dev
        credentials: local-dev
        dataprofiles: local-dev
```

`local-dev` overrides every selection.  This is normal — the
local-dev environment is so different from production that
inheriting much from `base` would be misleading.  The profile
still inherits the *shape* of the schema (`select:` field names,
implicit `overlay:` absence) from `base` for consistency.

### `staging-east.yaml`

```yaml
# config/profiles/staging-east.yaml
profile:
    extends: base
    name: staging-east
    description: Staging east region — production-shaped pod and services

    select:
        landscape: staging-east
        topology: staging-east-leader-follower
        runtime: staging-east
        credentials: staging-east
        dataprofiles: staging-east
```

`staging-east` is the production-shaped profile.  Its selections
are all `staging-east` except for the topology (which carries the
relationship-specific suffix `leader-follower`).

### `staging-east-canary.yaml`

```yaml
# config/profiles/staging-east-canary.yaml
profile:
    extends: staging-east
    name: staging-east-canary
    description: Canary slice — three runners not nine, slower pace

    select:
        landscape: staging-east-canary    # smaller landscape file
        runtime: staging-east-canary      # tighter parallelism

    overlay:
        runtime:
            parallelism:
                max_workers: 2            # belt-and-braces; runtime
                                          # already says 2 but explicit
                                          # for canary visibility
            logging:
                levels:
                    console: DEBUG        # canaries log verbosely
```

`staging-east-canary` extends `staging-east`, overrides two
selections (smaller landscape, tighter runtime), and adds an
overlay that bumps console logging to DEBUG and explicitly pins
the parallelism cap.  Topology, credentials, and dataprofiles
inherit unchanged from the parent.

### Resolution trace

When `AM_PROFILE=staging-east-canary` is set, the profile
resolver walks:

```
staging-east-canary  (read)
       │
       ├── extends: staging-east
       │           │
       │           └── extends: base
       │                       │
       │                       └── (terminal)
       │
       ▼
   merged select:
       landscape:      staging-east-canary   (canary override)
       topology:       staging-east-leader-follower (from staging-east)
       runtime:        staging-east-canary   (canary override)
       credentials:    staging-east          (from staging-east)
       dataprofiles:   staging-east          (from staging-east)
   merged overlay:
       runtime: { ... }  (from staging-east-canary)
```

The merged result is what gets translated into `AM_CONFIG_*_NAMES`
overrides.  The shipped resolution machinery loads the named
configurations.  The overlay is registered with the runtime
loader, which composes it onto the loaded `runtime-staging-east-canary`
via `MergeMap` overlay semantics.

## How the profile resolver runs

The resolver is roughly forty lines of Python plus a YAML loader.
Sketch:

```python
class ProfileResolver:
    def __init__(self, profiles_dir: str):
        self._dir = profiles_dir
        return

    def resolve(self, profile_name: str) -> dict:
        """
            Walk the extends chain, merge selections and overlays.
            Return a dict with 'select' (final selections per category)
            and 'overlay' (composed overlays).
        """
        chain = self._walk_chain(profile_name, seen=set())

        # Merge top-down: base is leftmost, deepest descendant is rightmost.
        merged_select = {}
        merged_overlay = MergeMap()

        for profile in chain:
            for category, name in profile.get("select", {}).items():
                merged_select[category] = name
            overlay = profile.get("overlay")
            if overlay is not None:
                merged_overlay.maps.append(overlay)

        return {"select": merged_select, "overlay": merged_overlay}

    def apply(self, resolved: dict) -> None:
        """
            Translate resolved selections into AM_*_NAMES overrides
            and register the overlay with the configuration loader.
        """
        this_app = ThisApplicationSingleton()
        category_to_descriptor = {
            "landscape":    ConfigVariablesDesc.AM_CONFIG_LANDSCAPE_NAMES,
            "topology":     ConfigVariablesDesc.AM_CONFIG_TOPOLOGY_NAMES,
            "runtime":      ConfigVariablesDesc.AM_CONFIG_RUNTIME_NAMES,
            "credentials":  ConfigVariablesDesc.AM_CONFIG_CREDENTIAL_NAMES,
            "dataprofiles": ConfigVariablesDesc.AM_CONFIG_DATAPROFILE_NAMES,
        }
        for category, name in resolved["select"].items():
            descriptor = category_to_descriptor.get(category)
            if descriptor is not None:
                this_app.override_variable_value(descriptor, name)

        # Register overlay with the loader.  The loader's
        # resolve_*_configuration runs after this and picks up the
        # overlay maps to layer onto the loaded named config.
        register_profile_overlay(resolved["overlay"])

        # Set the ACTIVE_PROFILE context path.
        this_app.context.insert(
            ContextPaths.ACTIVE_PROFILE,
            resolved["name"],
        )
        return

    def _walk_chain(self, profile_name: str, seen: set) -> list:
        if profile_name in seen:
            raise ConfigurationError(
                f"Cycle detected in profile chain at {profile_name}"
            )
        seen = seen | {profile_name}

        with open(os.path.join(self._dir, f"{profile_name}.yaml")) as fh:
            profile = yaml.safe_load(fh)["profile"]

        parent_name = profile.get("extends")
        if parent_name is not None:
            chain = self._walk_chain(parent_name, seen)
            chain.append(profile)
            return chain
        return [profile]
```

The resolver is small because it does one thing.  The shipped
machinery does all the heavy lifting downstream.

## Wiring at process start

The profile resolver runs as part of orchestration startup,
before any of the existing `resolve_*_configuration` functions:

```python
def bootstrap_runtime() -> None:
    profile_name = os.environ.get("AM_PROFILE")
    if profile_name is not None:
        resolver = ProfileResolver(profiles_dir="config/profiles")
        resolved = resolver.resolve(profile_name)
        resolver.apply(resolved)

    # Now the existing machinery runs unchanged.
    establish_configuration_settings()
    resolve_configuration_variables()
    resolve_configuration_maps()
    return
```

When `AM_PROFILE` is unset, the bootstrap skips profile
resolution entirely.  The shipped machinery runs against whatever
`AM_*_NAMES` are set in the environment directly — exactly the
current shipped behaviour.  The profile system is *opt-in*; teams
not using it pay no cost.

## Profile selection at the CLI

A second selection mechanism — the CLI flag — is convenient for
ad-hoc runs:

```bash
ccai testbase test run --profile staging-east tests/
```

The `--profile` flag is sugar for `AM_PROFILE=staging-east`.
Implementation: every `ccai` CLI command's argument-parsing
intercepts `--profile` before any command-specific parsing,
sets `os.environ["AM_PROFILE"]`, and then runs the rest of the
command normally.  The shipped
[`arcli`](../../../codecraftai-arcli/source/packages/ccai/arcli/)
machinery already supports global flags of this shape; adding
`--profile` is a small extension to the CLI tree's root.

The flag and the environment variable are equivalent.  Teams
using container-orchestration platforms typically prefer the
environment variable (it survives across `exec` boundaries
without explicit propagation); developers running ad-hoc commands
prefer the flag.  Both work.

## Profile diff: the natural follow-on tool

Once `ccai profile show` exists, the next-most-asked question
is: *what is the difference between two profiles?*  The proposed
`ccai profile diff` command answers it:

```bash
ccai profile diff staging-east staging-east-canary
```

Output:

```
Profile:        staging-east       →  staging-east-canary
Description:    Staging east region    Canary slice — 3 runners

Selections:
                                    landscape       staging-east → staging-east-canary
                                    runtime         staging-east → staging-east-canary
                                    (others unchanged)

Overlays:
                                    runtime.parallelism.max_workers (added: 2)
                                    runtime.logging.levels.console  (added: DEBUG)
```

Implementation reuses the resolver: resolve both profiles, walk
the merged selections side-by-side, walk the merged overlays
side-by-side, emit a diff.  Roughly a hundred lines.  Worth it
because *the most common operational question about profiles is
"what changed between two of them?"*

## What the discipline doesn't yet handle

Three things the proposed profile system *defers* rather than
solves.  Each is the subject of a later chapter or a future
extension.

### Captured profiles for replay

The resolver translates a profile name into a set of `AM_*_NAMES`
values plus an overlay.  Capturing the *resolved* state of a
particular run — every variable and its value, including those
chosen at startup — and replaying it later is chapter 16's
subject.  The resolver exposes the data the capture mechanism
needs; the capture mechanism itself is built on top.

### Per-user profile overrides

A team often has *team profiles* (the four-file set above) plus
*personal profiles* a developer keeps in their dotfiles.  The
pattern that works is to layer a personal profile on top of a
team profile via the same `extends:` mechanism, with personal
profiles living outside the repository.  The proposed resolver
supports this directly — profile files can come from multiple
search paths, with personal paths taking precedence.

### Cross-team profile sharing

When two teams share infrastructure, they often share *parts* of
profiles (the same Vault, the same staging hostnames) without
sharing whole profiles.  The pattern is to factor shared selections
into a `shared` profile that both teams `extends:`.  The proposed
resolver supports this with no special machinery; the discipline
just needs the team to draw the line.

## Summary

- A **profile** is a named bundle of category selections plus an
  optional overlay of value patches.  Profiles compose via
  `extends:`; selection happens via `select:`; granular value
  overrides happen via `overlay:`.
- The discipline is *additive* on top of the shipped portfolio.
  The profile resolver translates a profile name into
  `AM_*_NAMES` overrides and overlay registrations; the existing
  configuration-loading machinery runs downstream unchanged.
- Profile selection at process start uses the proposed
  `AM_PROFILE` environment variable (equivalently a `--profile`
  CLI flag).  When `AM_PROFILE` is unset, the system runs with
  the current shipped behaviour — the discipline is opt-in.
- The proposed `ContextPaths.ACTIVE_PROFILE` slot holds the
  resolved profile name.  Distributed orchestrations propagate it
  to remote workers; logging frameworks pick it up automatically.
- The proposed `ccai profile show` CLI command answers the
  most-asked operational question — *what is the resolved
  configuration of this run?*  The proposed `ccai profile
  diff` command answers the second-most-asked question — *what
  changed between two profiles?*
- The four-file worked example (`base`, `local-dev`,
  `staging-east`, `staging-east-canary`) shows additive overlay
  resolution end-to-end.  Cycles are detected at load time;
  resolution is bounded.
- The discipline defers captured-profile replay (chapter 16),
  per-user profile overrides (search-path extension), and
  cross-team profile sharing (multi-team `extends:` patterns) to
  later chapters and natural follow-ons.

The next chapter walks through every piece of "tribal knowledge"
that makes a fresh checkout unusable today (`.env.local`,
`direnv`, `make setup`, the README's "you might also need to..."
section) and shows how each piece migrates into a typed
configuration under the profile discipline.
