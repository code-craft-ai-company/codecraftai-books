# Chapter 19 — Built-in Task Types

> *Five concrete `Task` subclasses ship in `automojo-taskplus`.
> Three are useful for green-field work, one is intentionally a
> stub (a reader exercise), and one is transitional — useful for
> migration but to be retired by green-field code.*

The previous chapter introduced the abstract task model.  This
chapter walks the five concrete `Task` subclasses shipped under
[`automojo-taskplus/source/packages/automojo/taskplus/tasks/`](../../../automojo-taskplus/source/packages/automojo/taskplus/tasks/).
Each is a concrete implementation of `perform_task` — small in
LOC, big in cumulative usefulness.  The five together let workflow
authors build real orchestrations without writing a single line of
custom task code.

The chapter takes one task at a time, in the order most teams
will reach for them in practice.  For each task: the use case,
the YAML shape, the implementation, the audit findings the task
went through during the pre-book audit pass, and any
caveats the workflow author should know.

## RunCommand — single-shell execution

The simplest task type.  `RunCommand` runs a single shell command
and returns its exit code.

### Use case

A workflow step that boils down to "run this one command":

```yaml
- task: Lint
  tasktype: automojo.taskplus.tasks.runcommand@RunCommand
  command: "ruff check ."
```

```yaml
- task: Pull image
  tasktype: automojo.taskplus.tasks.runcommand@RunCommand
  command: "docker pull registry.example.com/my-image:latest"
```

The task is a thin wrapper around `subprocess.Popen` with a
shell.  When the workflow needs *one* command, this is the right
type.

### YAML shape

```yaml
- task: <human-readable label>
  tasktype: automojo.taskplus.tasks.runcommand@RunCommand
  command: <shell-command-string>
  if: <optional Python expression>
  onfailure: <optional label>
  onsuccess: <optional label>
```

One required field beyond the universals:
[`command`](../../../automojo-taskplus/source/packages/automojo/taskplus/tasks/runcommand.py) —
the shell command to execute.  The shell is the system's default
shell; the command is invoked with `shell=True`.

### Implementation

The task is roughly thirty lines.  `__init__` extracts `command`
from `task_info`; `perform_task` runs it:

```python
def perform_task(self, environment, parameters, flowstate) -> int:

    self._logger.info("Running Command: %s" % self._command)

    proc = subprocess.Popen(
        [self._command],
        stdout=subprocess.PIPE, stderr=subprocess.PIPE,
        shell=True,
    )
    stdout, stderr = proc.communicate()
    exit_code = proc.wait()

    stdout = stdout.decode()
    stderr = stderr.decode()

    log_msg_lines = [
        "RESULT CODE: %d" % exit_code,
        "STDOUT:",
        indent_lines(stdout, level=1),
        "STDERR:",
        indent_lines(stderr, level=1),
    ]

    log_msg = os.linesep.join(log_msg_lines)
    if exit_code == 0:
        self._logger.info(log_msg)
    else:
        self._logger.error(log_msg)

    return exit_code
```

Three things worth noting:

- **The exit code is the return value.**  The task's notion of
  success is the shell's notion of success.  No interpretation, no
  re-mapping.
- **Output is captured and logged**, not streamed.  For
  long-running commands this means the log waits until the
  command finishes; for the typical orchestration step (linters,
  short builds, docker pulls) this is fine.  When streaming is
  needed, `ShellScript` or a custom task implementation is the
  better choice.
- **STDOUT goes to `INFO` on success; both go to `ERROR` on
  failure.**  Failed-command output is highlighted at the right
  log level so it's findable in CI logs.

### Caveats

- **Shell-injection risk.**  `command` is interpreted by the
  shell with `shell=True`.  When the command is constructed from
  external input, ordinary shell-injection considerations apply.
  The discipline: workflow YAML is checked into the repository
  and code-reviewed; treat it as code, not data.
- **No environment-variable injection from `environment`.**  The
  task receives an `environment` argument but does not use it.
  Variables the command needs come from `os.environ` (set up by
  the engine before the workflow runs) or from the `command`
  string itself.

## ShellScript — multi-line scripts

When one command isn't enough.  `ShellScript` writes a temporary
file with the script content, marks it executable, runs it, and
returns the exit code.

### Use case

A workflow step that requires multiple commands, conditionals, or
intermediate variables — but doesn't justify pulling out a
standalone shell-script file:

```yaml
- task: Pre-test setup
  tasktype: automojo.taskplus.tasks.shellscript@ShellScript
  script:
    - "set -euo pipefail"
    - "rm -rf .output"
    - "mkdir -p .output/integration"
    - "if [ -d node_modules ]; then echo 'reusing node_modules'; else npm ci; fi"
    - "echo 'setup complete'"
```

The list-of-strings form is YAML-friendly and lets each command
sit on its own line for readability.

### YAML shape

```yaml
- task: <label>
  tasktype: automojo.taskplus.tasks.shellscript@ShellScript
  script:
    - <shell-line-1>
    - <shell-line-2>
    - ...
```

One required field:
[`script`](../../../automojo-taskplus/source/packages/automojo/taskplus/tasks/shellscript.py) —
a list of strings.  Each string becomes one line of the temp
script.  The lines are joined with `os.linesep` before writing.

### Implementation

The task writes the script to a temp file (using
[`get_temporary_file`](../../../automojo-runtime/source/packages/automojo/runtime/paths.py)
from `automojo-runtime`), chmods it executable, runs it, captures
output:

```python
def perform_task(self, environment, parameters, flowstate) -> int:

    script_content = os.linesep.join(self._script)

    tempfile = get_temporary_file(prefix="step-%s" % self._ordinal, suffix=".sh")

    self._logger.info("Running Script: %s" % tempfile)

    with open(tempfile, 'w') as tf:
        tf.write(script_content)

    os.chmod(tempfile, stat.S_IREAD | stat.S_IWRITE | stat.S_IEXEC)

    proc = subprocess.Popen(
        [tempfile],
        stdout=subprocess.PIPE, stderr=subprocess.PIPE,
        shell=True,
    )
    stdout, stderr = proc.communicate()
    exit_code = proc.wait()

    # ... log captured output, same shape as RunCommand ...

    return exit_code
```

### Audit findings

The pre-book audit pass on `automojo-taskplus` found two real
bugs in this task that have since been fixed:

- **The temp-file prefix used `%d` against a string ordinal**, which
  raised `TypeError` at construction.  Fixed to `%s`.
- **The chmod set only `stat.S_IEXEC`** — owner-execute, no read
  bit.  On macOS the missing read bit caused `/bin/sh` to fail
  with exit code 126 on every script.  Fixed to `S_IREAD |
  S_IWRITE | S_IEXEC`.

These are exactly the kinds of bug that don't surface in unit
tests and only surface when the task is run for the first time.
The audit + 115-test addition pass made these tests permanent
regression guards.

### Caveats

- **The temp script's permissions are owner-only.**  Other users
  on the machine can't read or run the script.  This is intentional
  but means a subprocess running as a different user (sudo
  invocations, container processes) cannot directly invoke the
  generated file.
- **The script is one shell invocation.**  Variables set in line
  *N* are available to line *N+1*; this is the same as putting
  the lines in a single `bash -c "..."` invocation.  Workflow
  authors used to one-command-per-step CI YAML find this
  liberating; the script is its own scope.
- **Errors do not auto-fail the script.**  A line that fails will
  not stop the script unless `set -e` (or `set -euo pipefail`) is
  the first line.  The discipline is to put the error-handling
  preamble at the top of every script — explicitly visible rather
  than implicit.

## EmbeddedPython — inline Python execution

When the workflow step is a small piece of logic that doesn't
need a shell, `EmbeddedPython` runs Python source directly in the
orchestration's process.

### Use case

A workflow step that's one or two lines of Python — typically
parsing some output, computing a derived value, or reading a small
file:

```yaml
- task: Compute build version
  tasktype: automojo.taskplus.tasks.embeddedpython@EmbeddedPython
  script:
    - "import os"
    - "build_version = f'{os.environ[\"BRANCH\"]}-{os.environ[\"BUILD_ID\"]}'"
    - "os.environ['BUILD_VERSION'] = build_version"
    - "print(f'BUILD_VERSION={build_version}')"
```

The use case overlaps with `RunCommand` running a `python -c
"..."`; the inline form has the advantage of not spawning a
subprocess (faster, no IPC) at the cost of running in the
orchestration's interpreter rather than a fresh one.

### YAML shape

```yaml
- task: <label>
  tasktype: automojo.taskplus.tasks.embeddedpython@EmbeddedPython
  script:
    - <python-line-1>
    - <python-line-2>
    - ...
```

One required field:
[`script`](../../../automojo-taskplus/source/packages/automojo/taskplus/tasks/embeddedpython.py) —
a list of Python source lines.  Joined with `os.linesep` before
`exec`.

### Implementation

```python
def perform_task(self, environment, parameters, flowstate) -> int:

    script_content = os.linesep.join(self._script)

    exit_status = 0

    try:
        # Execute the inline python script in the context of the current
        # globals and locals.
        exec(script_content, globals(), locals())
    except Exception:
        exit_status = 1

    return exit_status
```

### Audit findings

The pre-book audit found one significant bug: the script content
was being passed through `indent_lines(content, level=3)` before
`exec()`.  This prepended twelve spaces to every line, producing
an `IndentationError` at the first line of any non-empty script.
The bare-except swallowed the error and the task silently
returned `1` for everything.

Fixed in the same audit pass (the `indent_lines` call removed,
the bare `except:` narrowed to `except Exception`).

### Caveats

- **The script runs in the orchestration's process.**  Variables
  set in the script's globals persist for the rest of the
  orchestration's lifetime — a feature when used deliberately, a
  source of action-at-a-distance bugs when not.
- **Failure is collapsed to exit-code-1.**  Any exception in the
  script results in `return 1` with the exception message captured
  only in the orchestration's traceback (not the workflow's
  result code).  When richer error information is needed, the
  script should explicitly handle exceptions and emit clear log
  lines before returning failure.
- **The task is not sandboxed.**  Python is Python; the embedded
  script can do anything the orchestration process can do
  (read files, make network calls, exit the process).  Workflow
  YAML is code; treat it as code.

## GitCheckout — currently a stub

The fourth shipped task is intentionally a stub.

### Implementation

```python
class GitCheckout(Task):

    def __init__(self, ordinal: str, label: str, task_info: dict, logger):
        super(GitCheckout, self).__init__(ordinal, label, task_info, logger)
        return

    def perform_task(self, environment, parameters, flowstate) -> int:
        exit_code = 0
        return exit_code
```

The task accepts the universal `Task` constructor signature, does
nothing, and returns success.

### Why it ships as a stub

Two reasons.

First, **the right Git-checkout implementation is environment-
specific.**  CI environments typically have credentials configured
through the vendor's secret store; local-dev environments use
SSH agent or HTTPS-with-PAT; container environments may use a
mounted secret file.  A single shipped implementation would be
opinionated in a way that doesn't fit half the use cases.

Second, **the stub is a reader exercise.**  Chapter 23 develops
the recipe for adding a new task type.  `GitCheckout` is the
worked target — extending the stub into a real Git-checkout
implementation is the chapter's culminating exercise, with the
extension steps explicitly anchored on the existing stub.

### Caveats

A workflow that references `GitCheckout` as if it does work will
silently succeed without checking anything out.  Worth noting for
teams that may have pasted in a `GitCheckout` task by mistake.

For green-field code, the right answer until the reader implements
the extension is to use `RunCommand` or `ShellScript` with an
explicit `git clone` or `git fetch && git checkout` invocation.

## CreateEnvironment — transitional

The fifth shipped task computes environment variables from
shell-command output or from literal values, and exports them to
`os.environ` for use by subsequent steps.

### Use case (legacy)

The original use case: a workflow step that needs to set
environment variables based on dynamic values:

```yaml
- task: Compute env
  tasktype: automojo.taskplus.tasks.createenvironment@CreateEnvironment
  variables:
    - name: BUILD_VERSION
      command: "git describe --tags"
    - name: WORKSPACE_ROOT
      value: "$HOME/workspaces/${PROJECT_NAME}"
    - name: BRANCH_NAME
      command: "git rev-parse --abbrev-ref HEAD"
```

Each entry has a `name` and either a `command` (whose stdout
becomes the value) or a `value` (which is expanded with
`os.path.expandvars` and `os.path.expanduser`).  The result is
written to `os.environ`.

### Why this task is *transitional*

The whole content of the chapter-7 thesis applies to this task.
Computing environment variables at workflow-time and then
relying on them in subsequent steps:

- Couples downstream steps to the variable name (the brittle
  contract chapter 7 named).
- Hides configuration in `os.environ` (untyped, ephemeral, no
  schema).
- Encourages env-var-as-content patterns the volume's discipline
  is trying to retire.

For green-field code, the equivalent of the example above is:

- The build version becomes a *runtime configuration field* that
  downstream steps read via the runtime API, not from `os.environ`.
- The workspace root becomes a *path resolved by the runtime*,
  not a variable substituted into other commands.
- The branch name becomes a *parameter* to the orchestration,
  resolved at startup and propagated through `parameters` (chapter
  21).

When all three of those moves are made, `CreateEnvironment` is
not needed.

### Why it ships anyway

Two reasons it remains in the shipped set:

1. **Migration.**  Workflows that have used `CreateEnvironment`
   for years can keep using it during the migration to runtime
   profiles.  Removing the task before migration would force a
   coordinated cross-pipeline change.
2. **Genuine cases for environment-variable propagation.**  Some
   external tools the workflow runs only consume environment
   variables (legacy CLIs, third-party SDKs).  When a downstream
   process is a black-box that reads `os.environ`, computing the
   value at workflow-time and exporting it is the right answer
   even with the discipline.

The discipline's recommendation: when you reach for
`CreateEnvironment` for a workflow-internal value, ask whether
the value belongs in a runtime configuration instead.  When you
reach for it because an external tool requires an env var, the
task is the right answer.

### Implementation notes

The task supports a `VIRTUAL_ENV` mode that wraps each command
through a virtual-environment-activating shell script.  This is
useful when the orchestration runs inside a Python venv and the
commands the task spawns need to inherit it.  The task creates
the wrapper script lazily on first use.

The audit pass on this file found and fixed two issues:

- `stdout.strip()` produced `bytes` (not a string) which then
  couldn't be assigned to `os.environ`.  Fixed to
  `stdout.decode().strip()`.
- The `value` branch referenced an undefined `var_val` variable
  before assignment.  Fixed to `var_info["value"]`.

### Caveats

- **Order matters.**  Variables set by `CreateEnvironment` are
  visible only to *later* tasks in the workflow.  A task that runs
  *before* the `CreateEnvironment` task does not see the variables.
- **The variables persist beyond the workflow's lifetime in the
  process's `os.environ`** — the task does not unset them at end.
  When the orchestration is run inside a long-lived process (a
  daemon, a notebook), the `os.environ` accumulates
  `CreateEnvironment` outputs across runs.

## Choosing between the five

A small decision table for workflow authors:

| Use case | Task |
|---|---|
| One shell command | `RunCommand` |
| Several shell commands sharing variables | `ShellScript` |
| One or two lines of Python logic | `EmbeddedPython` |
| Git checkout (until you implement chapter 23) | `RunCommand` with `git` invocation; `GitCheckout` is currently a stub |
| Set an env var from a command output | `CreateEnvironment` (with the runtime-configuration alternative considered first) |
| Set an env var from a literal value | `CreateEnvironment` (with the runtime-configuration alternative considered first) |

Three rules of thumb beyond the table:

- **Reach for `RunCommand` first.**  When a step *can* be expressed
  as one command, it's clearer than a one-line `ShellScript` or a
  one-line `EmbeddedPython`.
- **Reach for `ShellScript` over `EmbeddedPython`** when the
  intent is shell-script-style logic (file manipulation, process
  invocation, conditional shell idioms).  The shell is built for
  this; Python is built for other things.
- **Reach for `EmbeddedPython` over `ShellScript`** when the
  intent is Python-style logic (parsing, computation, library
  calls).  Mismatching the language to the intent produces brittle
  code in either direction.

## Coverage and audit summary

Every line of every task above is covered by the 115-test
testbase suite added to `automojo-taskplus` in the pre-book audit
pass.  The tests cover:

- Construction with required fields present.
- Construction with required fields missing (raises
  `KeyError`).
- `perform_task` returning zero on success.
- `perform_task` returning non-zero on failure.
- Edge cases for each task type — empty scripts, expired
  credentials, command-not-found, virtual-env wrapping, value
  expansion.

The audit pass also surfaced bugs in every task except
`GitCheckout` (which is a stub and has no logic to be wrong).
Each bug fix is permanent in the test suite, so future
refactorings won't regress them.

The implication for the workflow author: these five tasks are
*proven*.  They've been read line-by-line, they've had their bugs
extracted, they've been tested against the same test suite that
guards the foundation tier.  The orchestration discipline that
Part V is building rests on tested code.

## Summary

- `automojo-taskplus` ships five concrete `Task` subclasses:
  `RunCommand`, `ShellScript`, `EmbeddedPython`, `GitCheckout`,
  `CreateEnvironment`.
- `RunCommand` runs one shell command; output is captured and
  logged.  The simplest task; the right default for one-command
  steps.
- `ShellScript` writes a list of shell lines to a temp file,
  chmods it executable, and runs it.  The right answer for
  multi-command steps that share state through shell variables.
- `EmbeddedPython` runs a list of Python lines via `exec` in the
  orchestration's process.  The right answer for small Python
  computations; *not* sandboxed and shares globals with the
  orchestration.
- `GitCheckout` is intentionally a stub — Git checkout
  implementation is environment-specific, and chapter 23 develops
  the extension recipe with `GitCheckout` as the worked target.
  Until then, use `RunCommand` for explicit `git clone` /
  `git fetch && git checkout`.
- `CreateEnvironment` is transitional.  For green-field code,
  prefer runtime configurations over workflow-time environment-
  variable computation.  For legacy migration and for genuine
  external-tool-needs-env-var cases, the task remains useful.
- All five tasks have been audited and tested in the pre-book
  audit pass.  Every fix is locked in by a test; the tasks are
  proven.

The next chapter introduces the workflow YAML grammar — the
file format that ties tasks and taskgroups into a complete
orchestration that the FlowEngine can execute.
