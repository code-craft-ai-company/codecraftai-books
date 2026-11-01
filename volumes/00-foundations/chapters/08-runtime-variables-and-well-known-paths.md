# Chapter 8 — Runtime Variables and Well-Known Paths

> *Application identity is structural; runtime is operational.
> The runtime layer is what every running process inherits the
> moment its persona is established and its activation profile
> is selected.*

The previous chapter introduced the extension and factory
machinery.  This chapter introduces the *runtime* variable set
and the *well-known path* helpers every higher tier uses.  These
modules now live inside
[`codecraftai-this-app`](https://github.com/code-craft-ai-company/codecraftai-this-app/tree/main/source/packages/ccai/thisapp),
under the `ccai.thisapp` namespace alongside the chapter-6
persona machinery.  (Historically they were a separate
`codecraftai-runtime` package; that package is being retired and now
holds thin re-export shims that forward `ccai.runtime.X` to
`ccai.thisapp.X`.  Update any `ccai.runtime.*` imports to
`ccai.thisapp.*`.)

> **As-built note.**  The runtime variable set, the path helpers,
> the activation profiles, and the enumerations were all folded
> into `codecraftai-this-app` so that the variable-descriptor model
> and the operational variables ship from one tier.  This also let
> `codecraftai-testbase` become near-standalone: testbase does *not*
> import `RuntimeVariables` or these path helpers at all.  See the
> sidebar later in this chapter, "What testbase actually uses".

The runtime-related contents of `ccai.thisapp` are organised
into these modules:

- **`runtimevariables.py`** — `RuntimeVariablesDesc` and
  `RuntimeVariables` extending the chapter-6 persona variable
  set, plus `DefaultValue`, the factory defaults, and
  `resolve_runtime_variables`.
- **`paths.py`** — well-known directory helpers (cache,
  presence, output, results, diagnostics, test-results,
  by-products, temp, summary-template).
- **`activation.py`** — activation-profile commands that
  configure logging and behaviour for the running process.
- **`enumerations.py`** — `JobType` and `ActivationProfile`.

The programmatic override helpers (`optionoverrides.py`) have not
yet been moved into `ccai.thisapp`; they currently still live
in `codecraftai-runtime` (see the section on option overrides
below).

## The runtime variable set

[`runtimevariables.py`](https://github.com/code-craft-ai-company/codecraftai-this-app/blob/main/source/packages/ccai/thisapp/runtimevariables.py)
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

    AM_INTERACTIVE_CONSOLE = AppVarDesc(default=False)

    AM_DEBUG_BREAKPOINTS = AppVarDesc(default=None,
                                       context_path=ContextPaths.DEBUG_BREAKPOINTS)
    AM_DEBUG_DEBUGGER = AppVarDesc(default=None,
                                    context_path=ContextPaths.DEBUG_DEBUGGER)

    # ... and many more
```

The descriptors fall into five thematic groups:

### Activation and pod identity

- **`AM_ACTIVATION_PROFILE`** — names the activation profile the
  process is running under (Command, Console, Job, Service —
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

[`enumerations.py`](https://github.com/code-craft-ai-company/codecraftai-this-app/blob/main/source/packages/ccai/thisapp/enumerations.py)
defines the `ActivationProfile` enum with four values:

- **`Command`** — the process is running as a CLI command.
  Logs go to a standard file plus the console; user-interactive.
- **`Console`** — the process is an interactive console.  Logs
  go to a file (away from the screen) so log output does not
  disrupt user interaction; the output directory is a temp
  directory.
- **`Job`** — the process runs as a job that collects its
  by-products (logs, results) under an output folder.  A job
  carries a `JobType` (see below) — Build, Infrastructure,
  Orchestration, or TestRun.
- **`Service`** — the process is a long-running service.  Logs
  go to a rotating file handler; non-interactive.

The companion `JobType` enum names the kinds of job: `Unknown`
(the default until activation sets it), `Build`,
`Infrastructure`, `Orchestration`, and `TestRun`.  As the path
section below shows, the test-reporting path helpers are
*activation-gated* on `AM_JOB_TYPE` being something other than
`Unknown`.

[`activation.py`](https://github.com/code-craft-ai-company/codecraftai-this-app/blob/main/source/packages/ccai/thisapp/activation.py)
provides four `activate_profile_*` functions — one per profile:
`activate_profile_command`, `activate_profile_console`,
`activate_profile_service`, and `activate_profile_job(job_type)`
(the job activation takes the `JobType` to run under).  Each
function:

1. **Guards** against duplicate activation — reads
   `RuntimeVariables.AM_ACTIVATION_PROFILE` and raises if already
   set.  A process may activate exactly one profile.
2. **Sets the active profile** through `override_variable_value`.
3. **Runs common setup** (`_activate_profile_common`) that
   resolves the output directory appropriate to the job type.
4. **Configures logging** appropriate to the profile (file path,
   format, level, handler type).
5. **Returns** with the process ready to run.

```python
def activate_profile_command():

    # Guard against attemps to activate more than one, activation profile.
    if RuntimeVariables.AM_ACTIVATION_PROFILE is not None:
        errmsg = "An attempt was made to activate multiple environment activation profiles. profile={}".format(
            RuntimeVariables.AM_ACTIVATION_PROFILE
        )
        raise SemanticError(errmsg)

    this_app = ThisApplicationSingleton()

    this_app.override_variable_value(RuntimeVariablesDesc.AM_ACTIVATION_PROFILE, ActivationProfile.Command)

    _activate_profile_common(this_app)

    log_directory = RuntimeVariables.AM_LOG_DIRECTORY
    log_basename = RuntimeVariables.AM_LOG_BASENAME

    logging_location = LoggingLocation(log_directory, log_basename)

    console_level = RuntimeVariables.AM_LOG_LEVEL_CONSOLE
    file_level = RuntimeVariables.AM_LOG_LEVEL_FILE

    activate_standard_file_logging(logging_location, console_level, file_level)

    return
```

The pattern is consistent across the four activation functions —
guard against duplicate activation; set the profile; common
setup; profile-specific logging.  (One honest wrinkle:
`activate_profile_job` currently raises a plain `RuntimeError`
rather than `SemanticError` on the duplicate-activation guard,
where the other three raise `SemanticError`.)

The application's startup code calls *one* of the activate
functions:

```python
from ccai.thisapp.persona import establish_app_persona
from ccai.thisapp.activation import activate_profile_command

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

[`paths.py`](https://github.com/code-craft-ai-company/codecraftai-this-app/blob/main/source/packages/ccai/thisapp/paths.py)
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

The always-available helpers — these resolve from the persona /
output variables and do not require a job activation:

- **`get_presence_directory()`** — returns the application's
  well-known presence directory (`AM_PRESENCE_DIR`).
- **`get_directory_for_cached_files()`** — returns the
  per-application cache directory (`<presence_dir>/cache`).  Used
  for intermediate artifacts that persist across runs.
- **`get_path_for_output()`** — returns the output directory
  (`AM_OUTPUT_DIRECTORY`) where results and artifacts are
  deposited.
- **`get_path_for_shared_store()`** — returns the configured
  shared-store directory (raises `ConfigurationError` if
  `AM_SHARED_STORE_DIRECTORY` is unset).
- **`get_temporary_directory()` / `get_temporary_file(suffix, prefix)`** —
  a `temp/` directory under the output directory, and a temp
  file within it via `tempfile.mkstemp`.  Used by Volume 3's
  `ShellScript` task (chapter 19) to create the per-task temp
  script.
- **`collect_python_modules(search_dir, max_depth)`** — utility
  for module discovery; used by the chapter-7 factory machinery
  for namespace scanning.

The *activation-gated* helpers — these raise `SemanticError`
unless the process was activated as a job (i.e. `AM_JOB_TYPE` is
something other than `JobType.Unknown`).  They resolve paths that
only make sense inside a results-collecting job:

- **`get_path_for_artifacts(label)`** — `<output>/artifacts/<label>`.
- **`get_path_for_diagnostics(label)`** —
  `<output>/diagnostics/<label>`.  Used by the chapter-5
  final-attempt pattern when a wait fails and dumps state.
- **`get_path_for_testresults()`** — the test-results directory
  (`AM_RESULT_PATH_FOR_TESTS`, falling back to the output
  directory).
- **`get_path_for_testcase_by_products(test_id)`** — per-test-case
  scratch space under `<testresults>/tc-by-products/<test_id>`.
- **`get_summary_html_template_source()`**,
  **`get_summary_static_resource_src_dir()`**,
  **`get_summary_static_resource_dest_dir()`** — locate the HTML
  summary template and its static resources.

> **These activation-gated helpers are consumed by
> integration-runtime, not testbase.**  Their only caller is
> `RuntimeOutputContext` in
> [`codecraftai-integration-runtime`](https://github.com/code-craft-ai-company/codecraftai-integration-runtime/blob/main/source/packages/ccai/integrationruntime/runtimeoutputcontext.py),
> which `IntegratedTestJob` installs by overriding
> `TestJob._create_output_context()`.  Because the helpers raise
> `SemanticError` unless the process was activated as a job, this
> is precisely why *plain* `testbase` must not call them —
> standalone testbase runs without activation.

The directory layout is conventional:

```
<presence_dir>/
├── cache/
├── logs/
│   └── <log_basename>-<timestamp>.log
├── results/
│   └── testresults/
│       └── <starttime>/
│           ├── artifacts/<label>/
│           ├── diagnostics/<label>/
│           └── tc-by-products/<test_id>/
├── settings.config
└── ... (other artifacts)
```

A consumer that needs to write a file asks the helper for the
right directory; the directory is created if needed; the
consumer writes the file.  No path-construction code in
consumers; no per-application directory layout decisions.

## Option overrides

[`optionoverrides.py`](https://github.com/code-craft-ai-company/codecraftai-runtime/blob/main/source/packages/ccai/runtime/optionoverrides.py)
provides programmatic override helpers for runtime variables.
(Unlike the other runtime modules, this one has *not* yet been
folded into `ccai.thisapp` — it still lives in
`codecraftai-runtime`, importing `RuntimeVariablesDesc` from
`ccai.thisapp.runtimevariables`.)  The helpers are bundled as
static methods on an `AM_RUNTIME_OPTION_OVERRIDES` class; each
wraps an `override_variable_value` call for one knob.  The
pattern: a CLI tool's `--debug-debugger` flag sets
`AM_DEBUG_DEBUGGER`; a CLI tool's `--seed` flag sets
`AM_JOB_SEED`:

```python
class AM_RUNTIME_OPTION_OVERRIDES:

    @staticmethod
    def override_debug_debugger(debugger: str):
        this_app = ThisApplicationSingleton()
        this_app.override_variable_value(RuntimeVariablesDesc.AM_DEBUG_DEBUGGER, debugger)
        return

    @staticmethod
    def override_job_seed(job_seed: str):
        this_app = ThisApplicationSingleton()
        this_app.override_variable_value(RuntimeVariablesDesc.AM_JOB_SEED, job_seed)
        return
```

The CLI calls:

```python
@click.option("--debug-debugger", default=None, help="...")
@click.option("--seed", default=None, help="...")
def run_command(debug_debugger, seed):
    if debug_debugger is not None:
        AM_RUNTIME_OPTION_OVERRIDES.override_debug_debugger(debug_debugger)
    if seed is not None:
        AM_RUNTIME_OPTION_OVERRIDES.override_job_seed(seed)
    # ...
```

The CLI doesn't reach into the descriptor model directly — it
calls a *named override method* per knob.  When a knob is
renamed, the override method's signature stays stable; the
CLI doesn't change.  When a new knob is added, a new override
method joins the class; the CLI gets a new option that
calls it.

The pattern is the chapter-2 *named-parameter-passing*
discipline applied at the override layer.

## How the runtime layer is consumed

Three consumption patterns recur.

### Pattern 1: read a runtime variable

```python
from ccai.thisapp.runtimevariables import RuntimeVariables

run_id = RuntimeVariables.AM_RUN_ID
log_dir = RuntimeVariables.AM_LOG_DIRECTORY
job_seed = RuntimeVariables.AM_JOB_SEED
```

The accessor reads through the resolution pipeline (override →
environment → settings → default).  Cached after first read
within the resolution.

### Pattern 2: get a well-known path

```python
from ccai.thisapp.paths import (
    get_directory_for_cached_files,
    get_path_for_diagnostics,
    get_path_for_output,
)

cache_dir = get_directory_for_cached_files()
output_dir = get_path_for_output()
diagnostics_dir = get_path_for_diagnostics("connect-timeout")
```

The helper creates the directory if needed; subsequent calls
return the cached path.  (`get_path_for_diagnostics` is
activation-gated — it requires a job activation.)

### Pattern 3: activate the runtime profile

```python
from ccai.thisapp.activation import activate_profile_command

activate_profile_command()
```

One call.  The activation function does the rest — sets the
profile, configures logging, prepares the runtime environment.

## What the runtime layer is *not*

Three deliberate non-features.

### Not a job runner

`AM_JOB_*` variables identify the job, but the runtime layer
doesn't *run* jobs.  Job execution is Volume 1's territory
(`codecraftai-testbase`'s test job runner) and Volume 2's
(distributed test jobs).  The runtime layer provides identity
for whatever job is running; the running is elsewhere.

> **What testbase actually uses.**  A key consequence of the
> testbase decoupling is that *plain* `testbase` does **not**
> consume this runtime layer at all.  It does not import
> `RuntimeVariables`, the path helpers, or the activation
> functions.  Instead it reads the handful of core env knobs it
> needs (`AM_TESTROOT`, `AM_OUTPUT_DIRECTORY`, `AM_RUN_ID`,
> `AM_DEBUG_*`, `AM_DIAGNOSTICS_*`, `AM_TESTRUN_TIMEOUT`) directly
> from `os.environ` via a tiny, dependency-free module,
> [`ccai.testbase.environment`](https://github.com/code-craft-ai-company/codecraftai-testbase/blob/main/source/packages/ccai/testbase/environment.py).
> Standalone testbase runs *without* activation — no presence
> folder, no `RuntimeVariables`, no `codecraftai-this-app` — and
> records its run summary to stdout.  The full runtime variable
> set, the presence-folder results layout, and the rendered HTML
> summary are the responsibility of `IntegratedTestJob` in
> `codecraftai-integration-runtime`, which activates the process as a
> `JobType.TestRun` job and installs the `RuntimeOutputContext`
> shown above.

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
        AM_RUNTIME_OPTION_OVERRIDES.override_pipeline_id(os.environ["CI_PIPELINE_ID"])
        AM_RUNTIME_OPTION_OVERRIDES.override_pipeline_name(os.environ.get("CI_PIPELINE_NAME", "unnamed"))
    elif "GITHUB_RUN_ID" in os.environ:
        # GitHub Actions
        AM_RUNTIME_OPTION_OVERRIDES.override_pipeline_id(os.environ["GITHUB_RUN_ID"])
        AM_RUNTIME_OPTION_OVERRIDES.override_pipeline_name(os.environ.get("GITHUB_WORKFLOW", "unnamed"))
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
- **Volume 1 (Testing)** — the *integration* test runner
  (`IntegratedTestJob` in `codecraftai-integration-runtime`) reads
  `AM_JOB_*` for job identity, `AM_BUILD_*` / `AM_PIPELINE_*` for
  run metadata, and routes results through the presence-folder
  path helpers.  Plain `codecraftai-testbase` runs standalone and
  reads only its few core knobs from `os.environ` (see the
  testbase sidebar above) — it does not consume this layer.
- **Volume 3 (Orchestration)** — the chapter-21
  bootstrap-name flow propagates `AM_*_NAMES` from the runtime
  layer through to subprocess jobs.

The pattern is consistent across the portfolio: every running
process inherits the runtime layer; every higher tier extends
it with subsystem-specific descriptors; consumers read through
the typed accessors.

## Summary

- The runtime variable set, path helpers, activation profiles,
  and enumerations now live in **`ccai.thisapp`** (the old
  `codecraftai-runtime` package is being retired and holds only
  re-export shims).  They provide the *operational* runtime
  variable set plus well-known path helpers plus
  activation-profile commands.
- **`RuntimeVariablesDesc`** extends `PersonaVariablesDesc`
  (chapter 6) with ~40 descriptors organised into themes:
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
- **Four activation profiles** — Command, Console, Job, Service.
  Each is activated by a single call to `activate_profile_*`
  (`activate_profile_job` takes a `JobType`); duplicate
  activation is guarded.  Each configures logging appropriately.
- **Well-known path helpers** — always-available ones
  (`get_presence_directory`, `get_directory_for_cached_files`,
  `get_path_for_output`, `get_temporary_file`) and
  *activation-gated* ones (`get_path_for_testresults`,
  `get_path_for_diagnostics`, `get_path_for_testcase_by_products`,
  the summary-template helpers) that raise `SemanticError` unless
  the process was activated as a job.  Each creates the directory
  on first call and caches the path.  The gated helpers are
  consumed by integration-runtime's `RuntimeOutputContext`, not
  by plain testbase.
- **Option overrides** (`AM_RUNTIME_OPTION_OVERRIDES`, still in
  `codecraftai-runtime`) — static methods that wrap
  `override_variable_value` calls per runtime variable.
  The CLI doesn't reach into the descriptor model; it calls
  named override methods.
- The runtime layer is *not* a job runner, *not* a CI vendor
  adapter (the adapter lives in the project), *not* multi-tenant;
  and plain `testbase` does *not* consume it at all.

The next chapter introduces `codecraftai-configuration` — the
configuration-loading machinery that builds on the runtime
variable model and provides the multi-source, multi-format,
optionally-encrypted file loader the rest of the portfolio
relies on.
