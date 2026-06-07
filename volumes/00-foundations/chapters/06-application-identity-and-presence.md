# Chapter 6 — Application Identity and Presence

> *Every running process in the portfolio answers two questions
> the same way: "Who am I?" and "Where do my files live?"  This
> chapter is about the small machinery that makes the answers
> consistent.*

The previous chapter introduced the waiting primitives.  This
chapter introduces the *identity* primitives —
[`automojo-this-app`](../../../../automojo-this-app/source/packages/automojo/thisapp/),
the package whose `ThisApplication` singleton represents the
running process to itself.  Every higher tier of the portfolio
calls into `ThisApplication` to resolve configuration variables,
locate well-known directories, and emit identity metadata into
logs and traces.

The package began small but has grown consequential.  At its core
is one singleton; one function (`establish_app_persona`) that
*every* application calls early in startup; one global
context that carries identity values across the process.  As of the
runtime-foundation consolidation it *also* owns the machinery that
used to live in the former `automojo-runtime` package: the
`RuntimeVariables` descriptor set, the well-known-paths helpers, the
profile/job enumerations, and the activation sequence.  The old
`automojo-runtime` is being retired — it now holds only thin
re-export shims, and import paths shift from `automojo.runtime.X`
to `automojo.thisapp.X`.

This consolidation is what lets `automojo-testbase` sit *above* the
identity-and-presence tier without depending on it: testbase reads
the handful of core environment knobs it needs directly, and the
richer runtime machinery described later in this chapter is what an
*integrated* run (Volume 1) layers back on top via
`automojo-integration-runtime`.

## The two-phase pattern

The identity machinery has a deliberate two-phase startup:

1. **`establish_app_persona(...)`** — called once, before any
   other automojo machinery.  This sets the application's
   *persona*: name, logger name, log file basename, presence
   directory (where the app's files live on disk), settings
   file path.  After this call, identity is fixed for the
   process's lifetime.

2. **`ThisApplicationSingleton()`** — called any number of times
   afterward.  Returns the singleton `ThisApplication` instance.
   The first call constructs the instance from the persona set
   in phase 1.

The split is intentional.  Phase 1 is *configuration*; phase 2
is *use*.  The singleton refuses to construct itself if phase 1
hasn't run — it raises `SemanticError` (chapter 3's must-bubble
type) so the failure is loud and the caller is told exactly
what's wrong:

```python
errmsg = "You must call 'establish_app_persona' early in your application startup to define " \
         "your application's presence settings before attempting to use the `ThisApplication` " \
         "object or its corresponding singleton."
raise SemanticError(errmsg)
```

This is the *errors-never-silent* discipline applied to
initialisation order.  An application that forgets to establish
its persona doesn't get a `None` back — it gets a clear
`SemanticError` telling it to fix the startup order.

## `establish_app_persona` — the canonical entry point

The function lives in
[`persona.py`](../../../../automojo-this-app/source/packages/automojo/thisapp/persona.py).
The signature:

```python
def establish_app_persona(*, name: str, logger_name: str, log_basename: str,
                          presence_dir: Optional[str]=None,
                          log_directory: Optional[str]=None,
                          settings_file: Optional[str]=None,
                          extensions: Optional[str]=None,
                          service_name: Optional[str]=None,
                          **other):
```

Three required parameters, four optional.  Every parameter is
keyword-only (the leading `*`); this is the chapter-2 named-
parameter discipline.

Walk each parameter.

### `name` — the application's identity

The application's name in machine-readable form.  Lowercase
with hyphens for spaces.  Examples: `"automojo-testbase"`,
`"my-deployment-tool"`, `"nightly-runner"`.  Used to derive
default paths and to identify the process in logs.

### `logger_name` — the root logger name

The Python logger name the application uses as its root.  When
the application emits logs through `logging.getLogger("my-app")`,
`logger_name="my-app"`.  Subsystems use child loggers
(`logging.getLogger("my-app.subsystem")`); the root name lets the
logging configuration apply formatters and handlers
hierarchically.

### `log_basename` — the log file's base name

The base name of the application's log files.  When the app
writes `my-app-2026-04-30.log` and `my-app-2026-04-30-debug.log`,
`log_basename="my-app"`.  The chapter-9 configuration loader
uses this to wire up the log file paths.

### `presence_dir` — where the app's files live

The directory where the application's files live on disk.  When
unspecified, defaults to `os.path.join("~", name.lower())` —
i.e., `~/my-deployment-tool/` for an app named
`"my-deployment-tool"`.

The presence directory is the *home base* for the application's
runtime artifacts: settings files, log files, captured profiles,
output directories, lock files.  Code that needs a writable
location asks `ThisApplication.presence_dir` and gets a
predictable path.

### `log_directory` — where logs go

Defaults to `os.path.join(presence_dir, "logs")` when
unspecified.  Override when logs need to live somewhere other
than the presence directory (a system log path, a separate
mounted volume).

### `settings_file` — the canonical settings file

Defaults to `os.path.join(presence_dir, "settings.config")`.

The function has a small convenience: if `settings_file` is a
bare filename (no path component), it's resolved relative to the
presence directory.  When a full path is given, it's expanded
through `os.path.expandvars` and `os.path.expanduser`.

### `extensions` — extension module list

A semicolon-separated string of extension module names.  Used by
the chapter-7 extension/factory machinery to discover plug-ins.

### `service_name` — the service identity

When the application runs as a system service, the service name.
Used by logging and observability to tag events with the
service identity.

## What `establish_app_persona` does at runtime

The function does three things:

1. **Validates** — confirms the singleton hasn't already been
   constructed.  If it has, raises `SemanticError` (you can't
   re-establish a persona).
2. **Stores the persona** — copies the parameters onto module-
   level `AppPersona` class attributes.
3. **Returns** — the function returns `None` (per chapter-2
   discipline, with explicit `return`).

The actual `ThisApplication` instance isn't constructed yet —
that happens lazily on the first `ThisApplicationSingleton()`
call.  The two-phase pattern keeps startup responsive: an
application that calls `establish_app_persona` early but doesn't
actually need the singleton until later doesn't pay the
construction cost upfront.

## `ThisApplication` — the singleton's surface

The
[`ThisApplication`](../../../../automojo-this-app/source/packages/automojo/thisapp/thisapplication.py)
class exposes seven properties (read-only views of the persona
fields) and four methods (the *active* surface for variable
resolution and overrides):

```python
class ThisApplication:
    # Properties (read-only)
    @property
    def name(self) -> str: ...

    @property
    def logger_name(self) -> str: ...

    @property
    def log_basename(self) -> str: ...

    @property
    def log_directory(self) -> str: ...

    @property
    def presence_dir(self) -> str: ...

    @property
    def settings_file(self) -> str: ...

    @property
    def service_name(self) -> str: ...

    # Methods
    def context_insert(self, path: str, obj: Any) -> None: ...
    def context_lookup(self, path: str, default=None, raise_error=False) -> Any: ...

    def get_variable_value(self, variable_desc: AppVarDesc, ...) -> Union[str, Any]: ...
    def override_variable_value(self, variable_desc: AppVarDesc, new_value: Any) -> None: ...

    def resolve_variable_set(self, variable_set: AppVarSet) -> None: ...
```

Each property is a *simple data accessor* — chapter 2's
property-as-pure-data discipline.  Each method does *one thing*
clearly named.

## The variable-descriptor model

The richest subsystem in `automojo-this-app` is the
*variable-descriptor* model.  It's the formal mechanism for
declaring which configuration values a subsystem cares about,
where each value can come from, and how it's converted to a
typed value.

Three classes participate:

### `AppVarDesc` — the descriptor

A descriptor declares *one variable*:

```python
AM_LOG_BASENAME = AppVarDesc(
    source=VariableSource.NONE,
)

AM_PRESENCE_DIR = AppVarDesc(
    source=VariableSource.NONE,
    context_path=ContextPaths.PRESENCE_HOME_DIRECTORY,
)

AM_INTERACTIVE_CONSOLE = AppVarDesc(
    default=False,
    converter=parse_bool,
)
```

Each descriptor carries:

- **`source`** — where the variable's value originates.  Common
  values: `VariableSource.ENVIRONMENT` (read from `os.environ`),
  `VariableSource.SETTINGS` (read from the app's settings file),
  `VariableSource.NONE` (resolved programmatically).
- **`default`** — the value to use when the source doesn't
  provide one.
- **`converter`** — a function that takes the raw string and
  returns the typed value.  The chapter-9 loader's
  `parse_bool`, `split_and_normalize_name_list`, etc., are the
  shipped converters.
- **`context_path`** — when the variable is also published to
  the shared application context, the well-known path it
  publishes to.  This is the `ContextPaths` enum we'll meet
  shortly.

The descriptor is *just data* — no behaviour, no resolution
logic.  The behaviour lives on `ThisApplication`.

### `AppVarSet` — a group of descriptors

A set of related descriptors live in a class that subclasses
`AppVarSet`:

```python
class PersonaVariablesDesc(AppVarSet):

    AM_NAME = AppVarDesc(source=VariableSource.NONE,
                          context_path=ContextPaths.PRESENCE_NAME)
    AM_LOGGER_NAME = AppVarDesc(source=VariableSource.NONE)
    AM_LOG_BASENAME = AppVarDesc(source=VariableSource.NONE)
    AM_PRESENCE_DIR = AppVarDesc(source=VariableSource.NONE,
                                  context_path=ContextPaths.PRESENCE_HOME_DIRECTORY)
    AM_LOG_DIRECTORY = AppVarDesc(source=VariableSource.NONE)
    AM_SETTINGS_FILENAME = AppVarDesc(source=VariableSource.NONE)
    AM_EXTENSIONS = AppVarDesc(source=VariableSource.NONE,
                                context_path=ContextPaths.PRESENCE_EXTENSIONS)
    AM_SERVICE_NAME = AppVarDesc(source=VariableSource.NONE)
```

Eight related descriptors, one class.  The class is *iterable* —
`AppVarSet.items()` yields the descriptors in declaration order
— so resolution code can walk the whole set without naming each
descriptor individually.

The volume's other variable sets follow the same pattern.
`ConfigVariablesDesc` (in `automojo-configuration`) declares the
configuration-related descriptors.  `RuntimeVariablesDesc` —
formerly in `automojo-runtime`, now in
[`automojo.thisapp.runtimevariables`](../../../../automojo-this-app/source/packages/automojo/thisapp/runtimevariables.py)
— declares the runtime-related descriptors and itself subclasses
`PersonaVariablesDesc`.  Each subsystem owns its own variable set;
the sets compose at the `ThisApplication` level.

### `AppVarAccessor` — typed read-through

For convenient access at the call site, a parallel class
provides typed read-through accessors:

```python
class PersonaVariables:
    AM_NAME = AppVarAccessor(PersonaVariablesDesc.AM_NAME)
    AM_LOGGER_NAME = AppVarAccessor(PersonaVariablesDesc.AM_LOGGER_NAME)
    AM_LOG_BASENAME = AppVarAccessor(PersonaVariablesDesc.AM_LOG_BASENAME)
    # ...
```

A consumer calls `PersonaVariables.AM_NAME` and gets the
resolved value.  The accessor object internally calls
`ThisApplicationSingleton().get_variable_value(descriptor)`.
The two-class pattern (Desc + Accessor) is the chapter's
ergonomic concession: `Desc.AM_NAME` is the *descriptor* (used
when calling `get_variable_value`); `Accessor.AM_NAME` is the
*resolved value* (used at the call site).

## Variable resolution

The active surface is `ThisApplication.get_variable_value(desc)`.
The resolution order:

1. **Override** — has the variable been overridden via
   `override_variable_value`?  If yes, return the override.
2. **Environment** — does the variable's source include
   environment variables, and is one set?  If yes, return the
   typed-converted environment value.
3. **Settings** — does the variable's source include the
   settings file, and is the variable present in it?  If yes,
   return the typed-converted settings value.
4. **Default** — is there an explicit default on the descriptor?
   If yes, return it.
5. **Else** — return `None` (or the `default` parameter if
   the caller provided one).

The order is deterministic and inspectable — given a descriptor,
you can predict what value it'll resolve to without running the
code.  This is the chapter-1 *explicit-is-better-than-implicit*
discipline applied to configuration resolution.

## Override semantics

`override_variable_value(desc, new_value)` is the runtime
override mechanism.  It writes the new value into an internal
override map; the next `get_variable_value(desc)` returns it
ahead of every other source.

Concretely, the implementation does *more* than just store the
value:

- For descriptors with `VariableSource.ENVIRONMENT` source, the
  override *also* writes to `os.environ` so child processes
  inherit the new value.  This is the chapter-21 (Volume 3)
  bootstrap-name-flow mechanism in action — the orchestration's
  override of `AM_LANDSCAPE_NAMES` reaches subprocess jobs
  through the environment variable.
- For *some* descriptors, the override *also* hot-swaps a
  matching field on `ThisApplication`.  Specifically:
  `AM_SERVICE_NAME` updates `_service_name`; `AM_LOG_BASENAME`
  updates `_log_basename`; `AM_LOG_DIRECTORY` updates
  `_log_directory`; `AM_LOGGER_NAME` updates `_logger_name`.
  This was the in-flight refactor flagged during the portfolio
  hygiene sweep in the prior session.

The pattern: the override is *immediate* in this process and
*propagated* to subprocess processes through `os.environ`.  Two
levels; same value; one entry point.

## The shared context

Beyond variable resolution, `ThisApplication` carries a *context*
— a hierarchical key-value store accessible by *path strings*:

```python
ctx = ThisApplicationSingleton().context

ctx.insert("/environment/landscape/names", ["staging-east"])
landscape_names = ctx.lookup("/environment/landscape/names")
```

The context is shared across the process; it's the *serializable*
identity of the running application.  Subsystems write
configuration data into it; consumers read it out.

The well-known paths live in
[`ContextPaths`](../../../../automojo-this-app/source/packages/automojo/thisapp/contextpaths.py)
— a string enum:

```python
class ContextPaths(str, Enum):

    BUILD_RELEASE = "/environment/build/release"
    BUILD_BRANCH = "/environment/build/branch"
    BUILD_NAME = "/environment/build/name"

    CONFIG_LANDSCAPE_NAMES = "/environment/landscape/names"
    CONFIG_LANDSCAPE_URIS = "/environment/landscape/uris"
    CONFIG_LANDSCAPE_FILES = "/environment/landscape/files"

    CONFIG_RUNTIME_NAMES = "/environment/runtime/names"
    CONFIG_RUNTIME_URIS = "/environment/runtime/uris"

    CONFIG_TOPOLOGY_NAMES = "/environment/topology/names"
    # ...

    JOB_ID = "/environment/job/id"
    JOB_NAME = "/environment/job/name"
    JOB_LABEL = "/environment/job/label"
    JOB_SEED = "/environment/job/seed"
    # ...
```

Each enum entry is *both a name and a path string*.  Code that
needs to publish or consume a value uses the enum:

```python
ctx.insert(ContextPaths.JOB_ID, run_uuid)
job_id = ctx.lookup(ContextPaths.JOB_ID)
```

The string-enum dual-citizenship makes the path *greppable* and
*rename-safe*.  When a path moves (rare), the enum value changes
in one place and every consumer that uses the enum is
automatically updated.

## The variable-descriptor + context-path composition

A variable descriptor with a `context_path` field automatically
publishes its resolved value to the context at that path:

```python
AM_PRESENCE_DIR = AppVarDesc(
    source=VariableSource.NONE,
    context_path=ContextPaths.PRESENCE_HOME_DIRECTORY,
)
```

When the variable is resolved (at startup, through
`resolve_variable_set`), its value is *also* written to
`/presence/home-directory` in the context.  The descriptor is
the *source of truth*; the context path is the *publication
endpoint*.

This composition is what enables Volume 3's bootstrap-name flow
to work cleanly.  The orchestration sets `AM_LANDSCAPE_NAMES`;
the variable resolution writes the value into both `os.environ`
and the context at `/environment/landscape/names`; subsystems
that need to *find* the active landscape names read the context
path; subsystems that need to *override* call
`override_variable_value` and trust the propagation.

## The absorbed runtime foundation

Everything above describes the *persona* machinery — the half of
`automojo-this-app` that was always here.  The other half arrived
with the runtime-foundation consolidation: the modules that used to
constitute the standalone `automojo-runtime` package now live under
`automojo.thisapp`, and the application-identity-and-presence tier is
their home.  They build directly on the descriptor model this
chapter just introduced.

> **As-built note.**  The old `automojo-runtime` package still
> exists, but only as a set of thin re-export shims so that
> in-flight callers importing `automojo.runtime.X` keep working
> while they migrate to `automojo.thisapp.X`.  New code imports from
> `automojo.thisapp` directly.

### `RuntimeVariablesDesc` / `RuntimeVariables` — the runtime descriptor set

[`runtimevariables.py`](../../../../automojo-this-app/source/packages/automojo/thisapp/runtimevariables.py)
declares `RuntimeVariablesDesc`, which *subclasses*
`PersonaVariablesDesc` — so a single descriptor set carries both the
persona fields and the runtime fields.  It is a large set: build
metadata (`AM_BUILD_RELEASE`, `AM_BUILD_BRANCH`, `AM_BUILD_NAME`,
`AM_BUILD_URL`), pipeline identity (`AM_PIPELINE_ID`,
`AM_PIPELINE_NAME`, `AM_PIPELINE_INSTANCE`), job identity
(`AM_JOB_ID`, `AM_JOB_NAME`, `AM_JOB_TYPE`, `AM_JOB_SEED`, …),
diagnostics knobs, logging levels, the output directory, the
test-result path templates, and more.  Each descriptor follows the
same `AppVarDesc` contract — `default`, `converter`, `context_path`
— so every runtime value resolves and publishes through the exact
pipeline the persona variables use.

The companion accessor class `RuntimeVariables` (subclass of
`PersonaVariables`) exposes the typed read-through views.  Call-site
code reads `RuntimeVariables.AM_JOB_TYPE` and gets the resolved,
typed value; resolution code passes
`RuntimeVariablesDesc.AM_JOB_TYPE` to `get_variable_value` /
`override_variable_value`.

A small free function, `resolve_runtime_variables()`, performs the
one-time resolution of the whole runtime set: it coerces the
start-time and shared-output-directory values, then calls
`this_app.resolve_variable_set(RuntimeVariablesDesc)` so every
descriptor with a `context_path` publishes into the shared context.
It guards against double resolution with a module-level
`RUNTIME_VARIABLES_RESOLVED` flag.

### Well-known paths

[`paths.py`](../../../../automojo-this-app/source/packages/automojo/thisapp/paths.py)
turns the resolved runtime variables into concrete on-disk
locations.  The shipped helpers include:

- `get_presence_directory()` — the application's home base, derived
  from `RuntimeVariables.AM_PRESENCE_DIR`.
- `get_path_for_output()` — the timestamped directory where results
  and artifacts are deposited.
- `get_path_for_testresults()` — the testresults directory.
- `get_path_for_testcase_by_products(test_id)` — a per-test-case
  by-products folder under the testresults directory.
- `get_path_for_diagnostics(label)` and `get_path_for_artifacts(label)`
  — labelled subfolders of the output directory.
- `get_summary_html_template_source()`,
  `get_summary_static_resource_dest_dir()`,
  `get_summary_static_resource_src_dir()` — the HTML-summary template
  and static-resource locations.

Several of these are *activation-gated*: helpers like
`get_path_for_diagnostics`, `get_path_for_testresults`, and
`get_path_for_testcase_by_products` raise `SemanticError` when
`RuntimeVariables.AM_JOB_TYPE == JobType.Unknown` — i.e. when the
process was *not* activated as a job.  This is the same
errors-never-silent discipline applied to the directory layout: a
plain, non-activated process is told plainly that the presence-folder
results tree does not exist for it.

> **Consequence for testbase.**  Plain `automojo-testbase` runs
> *without* activation — it records to stdout, has no presence
> folder, and never touches `RuntimeVariables` or this-app.  The
> presence-folder results layout, JSOS-to-disk streaming, HTML
> summary, and CI/build metadata are all the responsibility of
> `IntegratedTestJob` in `automojo-integration-runtime`, which
> activates the process as a job and *then* delegates to these
> path helpers.

### Profile and job enumerations

[`enumerations.py`](../../../../automojo-this-app/source/packages/automojo/thisapp/enumerations.py)
defines two string enums that govern *how* a process is activated:

```python
class ActivationProfile(str, Enum):
    Command = "command"
    Console = "console"
    Job = "job"
    Service = "service"

class JobType(str, Enum):
    Unknown = "Unknown"
    Build = "Build"
    Infrastructure = "Infrastructure"
    Orchestration = "Orchestration"
    TestRun = "TestRun"
```

`ActivationProfile` selects the *shape* of the process — a one-shot
command, an interactive console, a long-running service, or a job
that collects by-products into an output folder.  `JobType`
sub-classifies a job and drives which results subtree the activation
sequence builds (`Build`, `Infrastructure`, `Orchestration`,
`TestRun`).

### The activation sequence

[`activation.py`](../../../../automojo-this-app/source/packages/automojo/thisapp/activation.py)
ties the enumerations, the runtime variables, and the well-known
paths together.  It exposes one function per profile —
`activate_profile_command()`, `activate_profile_console()`,
`activate_profile_service(...)`, and `activate_profile_job(job_type)`
— each of which:

1. **Guards against double activation** — if
   `RuntimeVariables.AM_ACTIVATION_PROFILE` is already set, it
   refuses (a `SemanticError` for most profiles), because a process
   has exactly one persona-shape.
2. **Records the profile** — overrides
   `RuntimeVariablesDesc.AM_ACTIVATION_PROFILE` with the chosen
   `ActivationProfile`.
3. **Runs the common activation** — `_activate_profile_common`
   resolves the start-time-stamped output directory appropriate to
   the `JobType`, assigns a job id if one was not supplied, and
   overrides `AM_OUTPUT_DIRECTORY` accordingly.
4. **Activates logging** — wires up file, console, or rotating
   service logging through the `automojo.xmodules.xlogging` helpers.

`activate_profile_job(JobType.TestRun)` is the entry point an
integrated test run uses.  In `automojo-integration-runtime`, the
activation init (`establish_app_persona` →
`resolve_runtime_variables` → `activate_profile_job(JobType.TestRun)`)
is what populates the presence-folder output context and the
`RuntimeVariables`-sourced run metadata that `IntegratedTestJob`
records against.  This chapter's tier supplies the mechanism; that
package supplies the policy of *when* to invoke it.

## Cross-references

The identity machinery is used by every higher tier:

- **Volume 0 chapter 9 (Configuration loading)** uses
  `AppVarDesc` and `AppVarAccessor` as the typed descriptor for
  every configuration variable; `ConfigVariablesDesc` extends
  `PersonaVariablesDesc`.
- **Volume 0 chapter 8 (Runtime variables and well-known
  paths)** goes deeper on the runtime-related variables
  (`AM_RUN_ID`, `AM_INTERACTIVE_CONSOLE`, etc.) and the
  activation sequence — all of which, per the consolidation, now
  live in `automojo.thisapp` alongside the persona machinery rather
  than in a separate `automojo-runtime` package.
- **Volume 3 chapter 14 (Configuration profiles as a
  discipline)** proposes the `ContextPaths.ACTIVE_PROFILE`
  addition.  The chapter is using the same machinery this
  chapter introduces.
- **Volume 3 chapter 16 (Reproducing CI failures locally)**
  captures the resolved variables — the snapshot is a JSON
  serialisation of `AppVarSet.items()` results across all
  variable sets.

The pattern is consistent throughout the portfolio: identity is
managed by `ThisApplication`; values are typed through
descriptors; publication is through well-known context paths;
overrides are explicit and traceable.

## What this machinery is *not*

Three deliberate non-features.

### Not a generic dependency-injection container

`ThisApplication` carries identity and configuration variables.
It does *not* carry arbitrary application objects.  When the
application has a database connection, a cache client, an HTTP
session — those live in their own singletons or are passed
explicitly to functions.  The identity machinery is not the
right home for them.

### Not multi-application

The `ThisApplication` singleton is per-process.  When two
applications share a process (test harnesses, plugin
ecosystems), only one persona can be established.  When
isolation between two applications matters, they need separate
processes.

This is consistent with the philosophy's *simple-is-better-than-
complex* principle.  A multi-application machinery would be
heavier; the per-process model is sufficient for almost every
use case.

### Not stateful across restarts

The `ThisApplication` state lives in process memory.  When the
process restarts, the singleton is reconstructed from the
persona — but the override map and the runtime context are
fresh.  When state needs to *persist*, the application writes
it to the settings file or to disk.

The chapter-16 capture-and-replay pattern is the explicit
mechanism for cross-process state recovery.  The discipline does
not pretend that runtime overrides survive process boundaries
without that machinery.

## Summary

- **`automojo-this-app`** provides the identity and
  presence machinery every running process in the portfolio
  uses.  Two-phase startup: `establish_app_persona(...)` first
  (sets identity), then `ThisApplicationSingleton()` (gets the
  configured singleton).  Calling the singleton without
  establishing the persona raises `SemanticError`.
- The persona has **seven fields**: `name`, `logger_name`,
  `log_basename`, `presence_dir`, `log_directory`,
  `settings_file`, `extensions`, `service_name`.  Three are
  required; the rest have sensible defaults derived from
  `name`.
- **`ThisApplication`** exposes seven read-only properties (the
  persona fields) plus four methods (`context_insert`,
  `context_lookup`, `get_variable_value`,
  `override_variable_value`, `resolve_variable_set`).
- The **variable-descriptor model** has three classes:
  `AppVarDesc` (the data), `AppVarSet` (a group of related
  descriptors), `AppVarAccessor` (typed read-through at the
  call site).  Each subsystem ships its own descriptor set.
- **Variable resolution** has a deterministic order: override →
  environment → settings → default.  Inspectable; predictable.
- **Override propagation**: `override_variable_value` writes to
  the override map *and* to `os.environ` (for subprocess
  inheritance) *and* hot-swaps matching fields on
  `ThisApplication` for a small set of identity-critical
  variables.
- The **shared context** is a path-keyed key-value store
  accessible through `ThisApplication.context`.  Well-known
  paths live in `ContextPaths`, a string enum that's both a
  name and a path string for grep-and-rename safety.
- **Variable descriptors with `context_path`** automatically
  publish their resolved values to the context, making the
  resolution pipeline observable end-to-end.
- The tier now also **owns the absorbed runtime foundation** (moved
  here from the retired `automojo-runtime`):
  `RuntimeVariablesDesc` / `RuntimeVariables` and
  `resolve_runtime_variables` (runtime descriptor set), the
  well-known-paths helpers in `paths.py` (several activation-gated
  via `JobType.Unknown`), the `ActivationProfile` / `JobType`
  enumerations, and the `activate_profile_*` sequence in
  `activation.py`.  Plain testbase runs *without* this layer;
  `automojo-integration-runtime` layers it back on for integrated
  runs.
- The machinery is *not* a generic dependency-injection
  container, *not* multi-application, *not* persistent across
  restarts.  Each scoping choice is deliberate.

The next chapter introduces the extension and factory machinery
— the precedence-ordered factory model that lets the portfolio
plug new behaviour into existing extension points without
modifying the consumers of those extensions.
