# Chapter 8 — Runtime Variables and Well-Known Paths

> *Application identity is structural; runtime is operational.
> The runtime layer is what every running process inherits the
> moment its persona is established and its activation profile
> is selected.*

The previous chapter introduced the extension and factory
machinery.  This chapter introduces
[`automojo-runtime`](../../../../automojo-runtime/source/packages/automojo/runtime/),
the package that provides the *runtime* variable set and the
*well-known path* helpers every higher tier uses.  The package
sits one tier above `automojo-this-app` (which it imports for
the variable-descriptor model) and one tier below
`automojo-configuration` (which extends the runtime variable
set with configuration-specific descriptors).

The package's contents are organised into five modules:

- **`runtimevariables.py`** — `RuntimeVariablesDesc` and
  `RuntimeVariables` extending the chapter-6 persona variable
  set.
- **`paths.py`** — well-known directory helpers (cache,
  diagnostics, results, test-results, temp).
- **`activation.py`** — activation-profile commands that
  configure logging and behaviour for the running process.
- **`enumerations.py`** — `JobType`, `ActivationProfile`, and
  related enums.
- **`optionoverrides.py`** — programmatic override helpers for
  runtime variables.

## The runtime variable set

[`runtimevariables.py`](../../../../automojo-runtime/source/packages/automojo/runtime/runtimevariables.py)
extends `PersonaVariablesDesc` (chapter 6) with the variables
every running process needs for *operational* concerns —
distinct from chapter-6's *identity* concerns.

```python
class RuntimeVariablesDesc(PersonaVariablesDesc):

    AM_ACTIVATION_PROFILE = AppVarDesc(default=None)
    AM_AUTOMATION_POD = AppVarDesc(default=DefaultValue.NotSet)

    AM_BUILD_RELEASE = AppVarDesc(default=DefaultValue.NotSet,
                                   context_path=ContextPaths.BUILD_RELEASE)
    AM_BUILD_BRANCH = AppVarDesc(default=DefaultValue.NotSet,
                                  context_path=ContextPaths.BUILD_BRANCH)
    AM_BUILD_NAME = AppVarDesc(default=DefaultValue.NotSet,
                                context_path=ContextPaths.BUILD_NAME)
    AM_BUILD_URL = AppVarDesc(default=DefaultValue.NotSet,
                               context_path=ContextPaths.BUILD_URL)

    AM_RUN_ID = AppVarDesc(default=DefaultValue.NotSet,
                            context_path=ContextPaths.RUNID)

    AM_PIPELINE_ID = AppVarDesc(default=DefaultValue.NotSet,
                                 context_path=ContextPaths.PIPELINE_ID)
    AM_PIPELINE_NAME = AppVarDesc(default=DefaultValue.NotSet,
                                   context_path=ContextPaths.PIPELINE_NAME)

    AM_JOB_ID = AppVarDesc(default=DefaultValue.NotSet,
                            context_path=ContextPaths.JOB_ID)
    AM_JOB_LABEL = AppVarDesc(default=DefaultValue.NotSet,
                               context_path=ContextPaths.JOB_LABEL)
    AM_JOB_NAME = AppVarDesc(default=DefaultValue.NotSet,
                              context_path=ContextPaths.JOB_NAME)
    AM_JOB_SEED = AppVarDesc(default=default_job_seed,
                              context_path=ContextPaths.JOB_SEED)
    AM_JOB_TYPE = AppVarDesc(default=DefaultValue.NotSet,
                              context_path=ContextPaths.JOB_TYPE)

    AM_INTERACTIVE_CONSOLE = AppVarDesc(default=False, converter=parse_bool)

    AM_DEBUG_BREAKPOINTS = AppVarDesc(default=None,
                                       context_path=ContextPaths.DEBUG_BREAKPOINTS)
    AM_DEBUG_DEBUGGER = AppVarDesc(default=None,
                                    context_path=ContextPaths.DEBUG_DEBUGGER)

    # ... and many more
```

The descriptors fall into five thematic groups:

### Activation and pod identity

- **`AM_ACTIVATION_PROFILE`** — names the activation profile the
  process is running under (Console, Service, Test, Worker —
  the `ActivationProfile` enum from `enumerations.py`).
- **`AM_AUTOMATION_POD`** — the apod identifier (Volume 3
  chapter 5's apod from the landscape).

### Build provenance

- `AM_BUILD_RELEASE`, `AM_BUILD_BRANCH`, `AM_BUILD_FLAVOR`,
  `AM_BUILD_NAME`, `AM_BUILD_URL` — the build identity of the
  code currently running.  Carried into traces and result
  reports for retrospective analysis.

### Run, pipeline, job identity

- `AM_RUN_ID` — UUID for the current run.
- `AM_PIPELINE_ID` / `AM_PIPELINE_NAME` /
  `AM_PIPELINE_INSTANCE` — CI pipeline metadata.
- `AM_JOB_ID` / `AM_JOB_LABEL` / `AM_JOB_NAME` /
  `AM_JOB_SEED` / `AM_JOB_TYPE` — the running job's identity.

### Debugging knobs

- `AM_INTERACTIVE_CONSOLE` — boolean indicating whether the
  process has an attached interactive console.
- `AM_DEBUG_BREAKPOINTS` — set of named breakpoints the process
  should honour (the `breakpoints:` field from the chapter-7
  runtime YAML in Volume 3).
- `AM_DEBUG_DEBUGGER` — which debugger to attach
  (`debugpy`, `pydevd`, `none`).

### Diagnostics policy

- `AM_DIAGNOSTICS_POSTRUN` / `AM_DIAGNOSTICS_PRERUN` — whether
  to capture diagnostics around runs.
- `AM_DIAGNOSTICS_TRACEBACK_POLICY_OVERRIDE` — override the
  default traceback policy (the chapter-1 errors-never-silent
  discipline applied to traceback verbosity).

### `AM_JOB_SEED` — the reproducibility seed

The `AM_JOB_SEED` descriptor uses a *factory function* as its
default, not a literal value:

```python
def default_job_seed():
    rtnval = str(uuid4())
    return rtnval

AM_JOB_SEED = AppVarDesc(default=default_job_seed,
                         context_path=ContextPaths.JOB_SEED)
```

The factory returns a fresh UUID on every resolution.  When the
caller doesn't override the seed, every run gets a new UUID;
when the caller *does* override (passing a captured seed for
replay — the chapter-16 of Volume 3 capture-and-replay pattern),
the override pipeline takes precedence.

This is the descriptor model's *graceful fallback* for values
that need to be unique per run but reproducible when
explicitly fixed.

## `RuntimeVariables` — the typed accessor

For consumer convenience, the package ships
`RuntimeVariables` — an `AppVarAccessor`-based class that
parallels `RuntimeVariablesDesc`:

```python
class RuntimeVariables(PersonaVariables):

    AM_ACTIVATION_PROFILE = AppVarAccessor(RuntimeVariablesDesc.AM_ACTIVATION_PROFILE)
    AM_RUN_ID = AppVarAccessor(RuntimeVariablesDesc.AM_RUN_ID)
    AM_JOB_SEED = AppVarAccessor(RuntimeVariablesDesc.AM_JOB_SEED)
    # ... and so on
```

A consumer reads `RuntimeVariables.AM_RUN_ID` and gets the
resolved UUID.  The accessor calls
`ThisApplicationSingleton().get_variable_value(descriptor)` under
the hood; the call site doesn't need to know.

This is the *typed-read-through* pattern from chapter 6
applied at scale.  Forty-some descriptors; forty-some accessors;
the call sites are uniform.

## Activation profiles

[`enumerations.py`](../../../../automojo-runtime/source/packages/automojo/runtime/enumerations.py)
defines the `ActivationProfile` enum with four values:

- **`Command`** — the process is running as a CLI command.
  Logs go to console + file; user-interactive.
- **`Service`** — the process is a long-running service.  Logs
  go to syslog or service-aware log handler; non-interactive.
- **`TestRun`** — the process is a test runner.  Logs go to
  per-run output directories with structured file names.
- **`Worker`** — the process is a distributed-test worker
  (Volume 2 territory).  Logs go to a per-worker file.

[`activation.py`](../../../../automojo-runtime/source/packages/automojo/runtime/activation.py)
provides four `activate_profile_*` functions, one per profile.
Each function:

1. **Guards** against duplicate activation — calls
   `RuntimeVariables.AM_ACTIVATION_PROFILE` and raises
   `SemanticError` if already set.  A process may activate
   exactly one profile.
2. **Sets the active profile** through `override_variable_value`.
3. **Configures logging** appropriate to the profile (file path,
   format, level, handler type).
4. **Returns** with the process ready to run.

```python
def activate_profile_command():

    if RuntimeVariables.AM_ACTIVATION_PROFILE is not None:
        errmsg = "An attempt was made to activate multiple environment activation profiles. profile={}".format(
            RuntimeVariables.AM_ACTIVATION_PROFILE
        )
        raise SemanticError(errmsg)

    this_app = ThisApplicationSingleton()

    this_app.override_variable_value(
        RuntimeVariablesDesc.AM_ACTIVATION_PROFILE,
        ActivationProfile.Command,
    )

    _activate_profile_common(this_app)

    log_directory = RuntimeVariables.AM_LOG_DIRECTORY
    log_basename = RuntimeVariables.AM_LOG_BASENAME

    logging_location = LoggingLocation(log_directory, log_basename)

    activate_standard_file_logging(logging_location)
    activate_interactive_console_logging()

    return
```

The pattern is consistent across the four activation functions —
guard against duplicate activation; set the profile; common
setup; profile-specific logging.

The application's startup code calls *one* of the activate
functions:

```python
from automojo.thisapp.persona import establish_app_persona
from automojo.runtime.activation import activate_profile_command

establish_app_persona(
    name="my-tool",
    logger_name="my-tool",
    log_basename="my-tool",
)
activate_profile_command()

# ... rest of the application ...
```

Two lines of startup; both required; both clearly named.  The
chapter-1 *explicit-is-better-than-implicit* discipline applied
to process initialization.

## Well-known paths

[`paths.py`](../../../../automojo-runtime/source/packages/automojo/runtime/paths.py)
provides helpers for the directories every running process needs.
Each helper returns an *absolute path*, creating the directory
if it doesn't exist.

Module-level globals cache the resolved paths:

```python
DIR_PRESENCE_DIRECTORY = None
DIR_CACHE_DIRECTORY = None
DIR_DIAGNOSTICS_DIRECTORY = None
DIR_RESULTS_DIRECTORY = None
DIR_SHARED_STORE_DIRECTORY = None
DIR_TESTRESULTS_DIRECTORY = None
DIR_TESTCASE_BYPRODUCTS_DIRECTORY = {}
```

Each helper checks its global, computes the path on first call,
caches it, and returns.  Subsequent calls return the cached
value.  The pattern keeps path resolution fast without exposing
the caching mechanism to callers.

The shipped helpers:

- **`get_temporary_file(prefix, suffix)`** — wraps
  `tempfile.NamedTemporaryFile` with conventional naming.  Used
  by Volume 3's `ShellScript` task (chapter 19) to create the
  per-task temp script.
- **`get_cache_directory()`** — returns the per-process cache
  directory (typically `<presence_dir>/cache/`).  Used for
  intermediate artifacts that persist across runs.
- **`get_diagnostics_directory()`** — returns the per-run
  diagnostics directory.  Used by the chapter-5 final-attempt
  pattern when a wait fails and dumps state.
- **`get_results_directory()`** — returns the per-run results
  directory.
- **`get_testresults_directory()`** — returns the per-test-run
  results directory (a subdirectory of results).
- **`get_testcase_byproducts_directory(testcase_id)`** —
  per-test-case scratch space.
- **`collect_python_modules(search_dir, max_depth)`** — utility
  for module discovery; used by the chapter-7 factory machinery
  for namespace scanning.

The directory layout is conventional:

```
<presence_dir>/
├── cache/
├── diagnostics/
│   └── <run_id>/
├── logs/
│   └── <log_basename>-<timestamp>.log
├── results/
│   └── <run_id>/
│       ├── testresults/
│       └── ... (per-job results)
├── settings.config
└── ... (other artifacts)
```

A consumer that needs to write a file asks the helper for the
right directory; the directory is created if needed; the
consumer writes the file.  No path-construction code in
consumers; no per-application directory layout decisions.

## Option overrides

[`optionoverrides.py`](../../../../automojo-runtime/source/packages/automojo/runtime/optionoverrides.py)
provides programmatic override helpers for runtime variables.
The pattern: a CLI tool's `--debug` flag sets
`AM_DEBUG_DEBUGGER`; a CLI tool's `--seed` flag sets
`AM_JOB_SEED`; the helpers wrap the `override_variable_value`
calls in named functions:

```python
def override_debug_debugger(debugger: str):
    ThisApplicationSingleton().override_variable_value(
        RuntimeVariablesDesc.AM_DEBUG_DEBUGGER,
        debugger,
    )
    return

def override_job_seed(seed: str):
    ThisApplicationSingleton().override_variable_value(
        RuntimeVariablesDesc.AM_JOB_SEED,
        seed,
    )
    return
```

The CLI calls:

```python
@click.option("--debug-debugger", default=None, help="...")
@click.option("--seed", default=None, help="...")
def run_command(debug_debugger, seed):
    if debug_debugger is not None:
        override_debug_debugger(debug_debugger)
    if seed is not None:
        override_job_seed(seed)
    # ...
```

The CLI doesn't reach into the descriptor model directly — it
calls a *named override function* per knob.  When a knob is
renamed, the override function's signature stays stable; the
CLI doesn't change.  When a new knob is added, a new override
function joins the module; the CLI gets a new option that
calls it.

The pattern is the chapter-2 *named-parameter-passing*
discipline applied at the override layer.

## How the runtime layer is consumed

Three consumption patterns recur.

### Pattern 1: read a runtime variable

```python
from automojo.runtime.runtimevariables import RuntimeVariables

run_id = RuntimeVariables.AM_RUN_ID
log_dir = RuntimeVariables.AM_LOG_DIRECTORY
job_seed = RuntimeVariables.AM_JOB_SEED
```

The accessor reads through the resolution pipeline (override →
environment → settings → default).  Cached after first read
within the resolution.

### Pattern 2: get a well-known path

```python
from automojo.runtime.paths import (
    get_cache_directory,
    get_diagnostics_directory,
    get_results_directory,
)

cache_dir = get_cache_directory()
diagnostics_dir = get_diagnostics_directory()
results_dir = get_results_directory()
```

The helper creates the directory if needed; subsequent calls
return the cached path.

### Pattern 3: activate the runtime profile

```python
from automojo.runtime.activation import activate_profile_command

activate_profile_command()
```

One call.  The activation function does the rest — sets the
profile, configures logging, prepares the runtime environment.

## What the runtime layer is *not*

Three deliberate non-features.

### Not a job runner

`AM_JOB_*` variables identify the job, but the runtime layer
doesn't *run* jobs.  Job execution is Volume 1's territory
(`automojo-testbase`'s test job runner) and Volume 2's
(distributed test jobs).  The runtime layer provides identity
for whatever job is running; the running is elsewhere.

### Not a CI vendor adapter

`AM_PIPELINE_*` variables identify the CI pipeline, but the
runtime layer doesn't read GitLab CI's `$CI_PIPELINE_ID` or
GitHub Actions' `${{ github.run_id }}` automatically.  The
*adapter* is in the project — typically a small helper at
startup that reads the vendor's env vars and sets the runtime
variables:

```python
def import_vendor_pipeline_identity():
    if "CI_PIPELINE_ID" in os.environ:
        # GitLab CI
        override_pipeline_id(os.environ["CI_PIPELINE_ID"])
        override_pipeline_name(os.environ.get("CI_PIPELINE_NAME", "unnamed"))
    elif "GITHUB_RUN_ID" in os.environ:
        # GitHub Actions
        override_pipeline_id(os.environ["GITHUB_RUN_ID"])
        override_pipeline_name(os.environ.get("GITHUB_WORKFLOW", "unnamed"))
    # ... etc.
```

The adapter lives in the consuming application; the runtime
layer is vendor-neutral.

### Not multi-tenant

`AM_RUN_ID`, `AM_JOB_ID`, etc., are *per-process* identifiers.
A process running multiple jobs concurrently (a worker pool)
has *one* `AM_JOB_ID` at any time — typically updated as the
worker picks up each job.  When the runtime needs to track
multiple concurrent jobs, the consumer manages that explicitly;
the runtime layer's defaults assume sequential per-process
work.

## Cross-references

The runtime layer is consumed by:

- **Volume 0 chapter 9 (Configuration loading)** —
  `ConfigVariablesDesc` extends `RuntimeVariablesDesc`,
  inheriting all runtime variables and adding configuration-
  specific descriptors on top.
- **Volume 1 (Testing)** — the testbase test runner reads
  `AM_JOB_*` for job identity, `AM_DEBUG_*` for debugger
  attachment, `AM_INTERACTIVE_CONSOLE` for output formatting.
- **Volume 3 (Orchestration)** — the chapter-21
  bootstrap-name flow propagates `AM_*_NAMES` from the runtime
  layer through to subprocess jobs.

The pattern is consistent across the portfolio: every running
process inherits the runtime layer; every higher tier extends
it with subsystem-specific descriptors; consumers read through
the typed accessors.

## Summary

- **`automojo-runtime`** sits one tier above `automojo-this-app`
  and provides the *operational* runtime variable set plus
  well-known path helpers plus activation-profile commands.
- **`RuntimeVariablesDesc`** extends `PersonaVariablesDesc`
  (chapter 6) with ~40 descriptors organised into five themes:
  activation/pod identity, build provenance, run/pipeline/job
  identity, debugging knobs, diagnostics policy.
- **`AM_JOB_SEED`** uses a factory-function default
  (`default_job_seed`) to produce a fresh UUID on every
  unspecified resolution.  Captured-replay (Volume 3 chapter
  16) overrides through the standard pipeline.
- **`RuntimeVariables`** is the typed accessor class — one
  accessor per descriptor, each invoking
  `get_variable_value` through the chapter-6 resolution
  pipeline.
- **Four activation profiles** — Command, Service, TestRun,
  Worker.  Each is activated by a single call to
  `activate_profile_*`; duplicate activation raises
  `SemanticError`.  Each configures logging appropriately.
- **Well-known path helpers** — `get_cache_directory`,
  `get_diagnostics_directory`, `get_results_directory`,
  `get_testresults_directory`,
  `get_testcase_byproducts_directory`,
  `get_temporary_file`.  Each creates the directory on first
  call and caches the path.
- **Option overrides** — named functions that wrap
  `override_variable_value` calls per runtime variable.
  The CLI doesn't reach into the descriptor model; it calls
  named override functions.
- The runtime layer is *not* a job runner, *not* a CI vendor
  adapter (the adapter lives in the project), *not* multi-tenant.

The next chapter introduces `automojo-configuration` — the
configuration-loading machinery that builds on the runtime
variable model and provides the multi-source, multi-format,
optionally-encrypted file loader the rest of the portfolio
relies on.
