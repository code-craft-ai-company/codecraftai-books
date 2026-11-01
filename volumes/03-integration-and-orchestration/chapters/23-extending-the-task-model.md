# Chapter 23 — Extending the Task Model

> *Five shipped task types are enough for a starter set, never
> enough for a real codebase.  This chapter is the recipe for
> adding the sixth, seventh, and eighth.*

The previous five chapters of Part V established the task model
(chapter 18), the shipped task types (chapter 19), the YAML
grammar (chapter 20), the orchestration-to-job contract (chapter
21), and the engine that runs the result (chapter 22).  This
chapter closes Part V with the *extension recipe* — how to add a
new task type when the shipped five don't cover the use case.

The chapter uses a worked target: turning the shipped
[`GitCheckout`](https://github.com/code-craft-ai-company/codecraftai-taskplus/blob/main/source/packages/ccai/taskplus/tasks/gitcheckout.py)
stub from chapter 19 into a real Git checkout implementation.
The same recipe applies to any other extension; `GitCheckout` is
chosen because it's the smallest visible gap in the shipped set
and because it's the chapter-19 reader exercise.

The recipe is short.  Six steps; roughly thirty lines of code per
step; one new entry in the YAML's `tasktype:` namespace.

## When to write a new task type

The shipped five — `RunCommand`, `ShellScript`,
`EmbeddedPython`, `GitCheckout` (stub), `CreateEnvironment` —
cover most common needs.  Before reaching for a new task class,
check whether the existing types fit:

| Use case | Reach for |
|---|---|
| Run a single shell command | `RunCommand` |
| Run a multi-line shell script | `ShellScript` |
| Run a small piece of Python | `EmbeddedPython` |
| Set environment variables (legacy) | `CreateEnvironment` (with chapter-7 caveat) |

The five cover the vast majority of orchestration needs.  When
you find yourself writing a `RunCommand` that wraps a Python
script that wraps a custom CLI to do *one specific thing*, you
have a candidate for a new task type.

Three signals that a new task type is the right answer:

### Signal 1: the same wrapper appears in multiple workflows

When three workflows all have a step like this:

```yaml
- task: Pull image
  tasktype: ccai.taskplus.tasks.runcommand@RunCommand
  command: "docker pull registry.example.com/${IMAGE_NAME}:${IMAGE_TAG}"
```

with `IMAGE_NAME` and `IMAGE_TAG` varying — the workflows are
duplicating wrapper logic.  A `DockerPull` task with `image:`
and `tag:` fields would replace the duplication with a typed
construct.

### Signal 2: the shell command is structurally complex

When the workflow needs to run a command with structured arguments
(a JSON config blob, a multi-step retry loop, an output parser),
expressing it as `command:` strings becomes brittle.  A custom
task that takes typed YAML fields and constructs the
underlying call internally is more maintainable.

### Signal 3: the task needs to interact with the orchestration

When the task wants to write to flowstate, log structured events,
read from a runtime configuration, or spawn related child
processes — all things the shipped tasks do *minimally* — a
custom task with explicit access to those mechanisms is the right
answer.

## The recipe in six steps

Worked target: turn `GitCheckout` from a stub into a real
implementation.

### Step 1 — Decide the YAML shape

What fields will the workflow author write?  For Git checkout:

```yaml
- task: Check out app
  tasktype: ccai.taskplus.tasks.gitcheckout@GitCheckout
  repository_url: "git@github.com:example/app.git"
  branch: main                              # optional; default 'main'
  destination: "${WORKSPACE}/app"
  credential: ci-bot-github                  # credential identifier
  shallow: true                              # optional; default false
```

Five fields beyond the universals: `repository_url`, `branch`,
`destination`, `credential`, `shallow`.  Two are required
(`repository_url`, `destination`); three are optional with sane
defaults.

The `credential:` field references a credential identifier
(chapter 9).  The actual credential object will be looked up
through `CredentialManagerSingleton` at task execution.  No
secret material lives in the workflow YAML.

### Step 2 — Write the constructor

The task class extends `Task` and reads its required fields from
`task_info` (the chapter-18 universal):

```python
"""
.. module:: gitcheckout
    :platform: Darwin, Linux, Unix, Windows
    :synopsis: Real GitCheckout task — replaces the chapter-19 stub.

.. moduleauthor:: CodeCraftX <automationx@code-craft.ai>
"""

from typing import Any, Dict, Optional

import os
import subprocess

from ccai.errors.exceptions import ConfigurationError

from ccai.taskplus.model.task import Task


class GitCheckout(Task):
    """
        Checks out a Git repository at a specific branch into a
        destination directory.  Authenticates using a credential
        identifier resolved through CredentialManagerSingleton.
    """

    def __init__(self, ordinal: str, label: str, task_info: dict, logger):
        super(GitCheckout, self).__init__(ordinal, label, task_info, logger)

        if "repository_url" not in task_info:
            errmsg = "GitCheckout requires a 'repository_url' field."
            raise ConfigurationError(errmsg)
        if "destination" not in task_info:
            errmsg = "GitCheckout requires a 'destination' field."
            raise ConfigurationError(errmsg)

        self._repository_url = task_info["repository_url"]
        self._destination = task_info["destination"]
        self._branch = task_info.get("branch", "main")
        self._credential_id = task_info.get("credential", None)
        self._shallow = task_info.get("shallow", False)

        return
```

Two design decisions visible above:

- **Required fields are validated at construction time.**  A
  workflow that lacks `repository_url` fails to *load*, not to
  execute.  This matches the chapter-19 pattern — `RunCommand`
  fails fast on missing `command`, `ShellScript` on missing
  `script`.  The workflow author finds out about the mistake
  before the orchestration starts.
- **Optional fields have explicit defaults.**  The constructor
  uses `task_info.get(field, default)` with the default visible
  in code.  When the YAML omits `branch:`, the task uses
  `"main"`; when it omits `shallow:`, the task uses `False`.
  Defaults are part of the task's contract and live near the
  field reads.

### Step 3 — Write `perform_task`

The implementation does the actual work.  Three concerns:
resolve the credential, build the Git command, run it.

```python
def perform_task(self, environment, parameters, flowstate) -> int:

    self._logger.info(
        "GitCheckout: %s @ %s -> %s" %
        (self._repository_url, self._branch, self._destination)
    )

    # Step 3a: resolve credential
    git_env = os.environ.copy()
    if self._credential_id is not None:
        git_env = self._apply_credential_to_env(git_env)

    # Step 3b: build the Git command
    cmd = ["git", "clone", "--branch", self._branch]
    if self._shallow:
        cmd.extend(["--depth", "1"])
    cmd.append(self._repository_url)
    cmd.append(self._destination)

    # Step 3c: run it
    proc = subprocess.Popen(
        cmd,
        env=git_env,
        stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    )
    stdout, stderr = proc.communicate()
    exit_code = proc.wait()

    stdout_text = stdout.decode()
    stderr_text = stderr.decode()

    if exit_code == 0:
        self._logger.info("Git clone succeeded.")
        flowstate["gitcheckout.last_destination"] = self._destination
    else:
        self._logger.error(
            "Git clone failed (exit %d): %s" %
            (exit_code, stderr_text.rstrip())
        )

    return exit_code
```

Four implementation notes:

#### Credential resolution

The credential is fetched through the
[`CredentialManagerSingleton`](https://github.com/code-craft-ai-company/codecraftai-credentials/blob/main/source/packages/ccai/credentials/wellknown.py)
(chapter 9) via a helper:

```python
def _apply_credential_to_env(self, env: Dict[str, str]) -> Dict[str, str]:
    """
        Resolve the credential identifier and apply Git auth to env.
    """
    from ccai.credentials.wellknown import CredentialManagerSingleton

    cred = CredentialManagerSingleton().lookup_credential(self._credential_id)

    # Different credential types apply differently.  Two common cases:
    if hasattr(cred, "token"):
        # GithubTokenCredential or PersonalApiTokenCredential
        env["GIT_ASKPASS"] = "/bin/echo"   # never prompt
        env["GIT_USERNAME"] = getattr(cred, "username", "git")
        env["GIT_PASSWORD"] = cred.token
        env["GIT_HTTP_EXTRAHEADER"] = f"Authorization: Bearer {cred.token}"
    elif hasattr(cred, "keyfile"):
        # SshCredential
        env["GIT_SSH_COMMAND"] = (
            f"ssh -i {cred.keyfile} -o StrictHostKeyChecking=no "
            f"-o IdentitiesOnly=yes"
        )

    return env
```

The credential's *type* dispatches the auth strategy.  An SSH
credential becomes `GIT_SSH_COMMAND`; a token credential becomes
HTTP basic-auth headers.  The dispatch is *the task's*
responsibility, not the credential's.  This is consistent with
chapter-9's vocabulary-versus-resolution boundary: the credential
is data; the consumer decides how to apply it.

The credential lookup is a programmatic call — chapter 11's
"never log a credential" rule applies.  Note the absence of any
`logger.info(cred)` or `f"{cred}"` patterns; the task reads
fields and applies them directly without intermediate
serialisation.

#### Subprocess invocation

The task uses `subprocess.Popen` with `env=git_env` rather than
`shell=True`.  Two reasons:

- The credential-bearing env vars need to reach the Git
  subprocess deterministically.  `env=git_env` is explicit;
  `shell=True` introduces a shell that could mangle the env vars
  through expansion or quoting.
- Avoiding shell injection.  When the task's fields come from
  workflow YAML (which is checked in and reviewed) the risk is
  modest, but the explicit `cmd` list-of-args form is uniformly
  safer.

#### Flowstate write

On success, the task writes `flowstate["gitcheckout.last_destination"]`
so subsequent tasks can read where the checkout landed:

```yaml
- task: Check out app
  tasktype: ccai.taskplus.tasks.gitcheckout@GitCheckout
  repository_url: "git@github.com:example/app.git"
  destination: "${WORKSPACE}/app"

- task: Build app
  tasktype: ccai.taskplus.tasks.runcommand@RunCommand
  command: "cd $(ccai flowstate get gitcheckout.last_destination) && make build"
```

The `gitcheckout.` prefix is the chapter-21 namespacing
convention.  Multiple `GitCheckout` tasks in one workflow could
trip over each other on this key — when the workflow has
multiple checkouts, the task should write a per-task key
(`gitcheckout.<label>.last_destination`).  The discipline is the
workflow author's; the task gives a sensible default.

#### Error reporting

On failure, the task logs the *stderr* output at error level.
This is the chapter-19 pattern carried forward — failed-command
output goes to the right log level so it's findable in CI logs.
The task does *not* log stdout on failure; Git's stdout on
failure is typically empty.

### Step 4 — Register the tasktype reference

The task is now usable in workflows by its module-and-class
reference:

```yaml
- task: Check out app
  tasktype: ccai.taskplus.tasks.gitcheckout@GitCheckout
  ...
```

The
[`Workflow.parse`](https://github.com/code-craft-ai-company/codecraftai-taskplus/blob/main/source/packages/ccai/taskplus/model/workflow.py)
loader (chapter 20) resolves the reference at parse time:
imports the module, looks up the class, instantiates with the
chapter-18 constructor signature.

Two observations on registration:

- **No central registry.**  The task class doesn't need to be
  registered anywhere; the YAML's `module@Class` reference is
  the registration.  This is why custom tasks can live anywhere
  in the importable namespace — the workflow author writes the
  fully-qualified reference and `import_by_name` resolves it.
- **The module must be importable from the orchestration's
  context.**  When the task lives in a separate package, the
  package must be installed (or on `PYTHONPATH`) before the
  orchestration runs.  This is straightforward when the task
  ships in `codecraftai-taskplus` itself; for tasks shipped in a
  consumer package (e.g., a project's own `myproject.taskplus`
  module), the package needs to be installed alongside.

### Step 5 — Write tests

Every shipped task in `codecraftai-taskplus` has a corresponding
test file in
[`source/testroots/ccai/tests/singlehost/taskplus/`](https://github.com/code-craft-ai-company/codecraftai-taskplus/tree/main/source/testroots/ccai/tests/singlehost/taskplus).
A new task type adds a new test file with the same shape.

A minimal test suite for `GitCheckout`:

```python
"""Tests for the :class:`GitCheckout` task."""

import logging
import os
import shutil

from ccai.testbase import verification

from ccai.errors.exceptions import ConfigurationError
from ccai.taskplus.tasks.gitcheckout import GitCheckout


def _make_logger() -> logging.Logger:
    return logging.getLogger("test.gitcheckout")


def test_construct_requires_repository_url() -> None:
    """Construction without 'repository_url' raises ConfigurationError."""

    raised = False
    try:
        GitCheckout(
            ordinal="1",
            label="checkout",
            task_info={"destination": "/tmp/x"},
            logger=_make_logger(),
        )
    except ConfigurationError:
        raised = True

    verification.assert_equal(raised, True)
    return


def test_construct_requires_destination() -> None:
    """Construction without 'destination' raises ConfigurationError."""

    raised = False
    try:
        GitCheckout(
            ordinal="1",
            label="checkout",
            task_info={"repository_url": "https://example/r.git"},
            logger=_make_logger(),
        )
    except ConfigurationError:
        raised = True

    verification.assert_equal(raised, True)
    return


def test_construct_stores_required_fields() -> None:
    """Required fields are stored as private attributes."""

    task = GitCheckout(
        ordinal="1",
        label="checkout",
        task_info={
            "repository_url": "https://example/r.git",
            "destination": "/tmp/x",
        },
        logger=_make_logger(),
    )

    verification.assert_equal(task._repository_url, "https://example/r.git")
    verification.assert_equal(task._destination, "/tmp/x")
    verification.assert_equal(task._branch, "main")    # default
    verification.assert_equal(task._shallow, False)    # default
    verification.assert_equal(task._credential_id, None)  # default

    return


def test_construct_uses_explicit_branch_and_shallow() -> None:
    """Optional fields are honoured when present."""

    task = GitCheckout(
        ordinal="1",
        label="checkout",
        task_info={
            "repository_url": "https://example/r.git",
            "destination": "/tmp/x",
            "branch": "develop",
            "shallow": True,
        },
        logger=_make_logger(),
    )

    verification.assert_equal(task._branch, "develop")
    verification.assert_equal(task._shallow, True)
    return


def test_perform_task_with_invalid_url_returns_nonzero() -> None:
    """A clone against an unreachable URL returns non-zero."""

    target = "/tmp/codecraftai-gitcheckout-test"
    if os.path.exists(target):
        shutil.rmtree(target)

    task = GitCheckout(
        ordinal="1",
        label="checkout",
        task_info={
            "repository_url": "/dev/null/this-does-not-exist.git",
            "destination": target,
        },
        logger=_make_logger(),
    )

    result = task.perform_task(environment={}, parameters={}, flowstate={})

    verification.assert_true(result != 0)
    return
```

Five tests covering: two constructor validation cases, the
required-fields-stored case, the optional-fields-honoured case,
and the perform-task-fails-loudly case.  Real Git operations
against real repositories are typically left out of unit tests
(integration tests run them); the suite focuses on the parts of
the task that don't need a live network.

### Step 6 — Document the task type

The task class's docstring is the workflow author's API
reference.  At minimum, the docstring names the YAML fields and
their semantics:

```python
class GitCheckout(Task):
    """
        Checks out a Git repository.

        Required YAML fields:

        :repository_url: Git repository URL (HTTPS or SSH).
        :destination: Local filesystem path the repository is cloned to.

        Optional YAML fields:

        :branch: Branch to check out.  Default 'main'.
        :credential: Credential identifier resolved at execution time
                      via CredentialManagerSingleton.  Default None
                      (use system Git auth — SSH agent, gitconfig, etc.).
        :shallow: When True, performs --depth=1 clone.  Default False.

        Flowstate writes on success:

        :gitcheckout.last_destination: The destination path of the
                                        successful clone.

        Failure modes:

        * Returns the Git CLI's exit code on failure.
        * Logs stderr at ERROR level on non-zero exit.
    """
```

The docstring is what the workflow author reads to know how to
use the task.  Per the volume's chapter-0-anchored coding-standards
discipline, the docstring uses Sphinx field syntax (`:param:`,
`:returns:`, etc.) and is parseable by the documentation
generator.

## A more substantial extension: a `Retry` wrapper

The recipe above produces a *leaf* task — a class that inherits
directly from `Task` and does work.  A more substantial extension
is a *wrapper* task that *modifies* another task's behaviour.
Worked target: a task that retries a wrapped task on failure.

```yaml
- task: Flaky API call (with retry)
  tasktype: ccai.taskplus.tasks.retry@Retry
  attempts: 3
  delay_seconds: 5
  inner:
      tasktype: ccai.taskplus.tasks.runcommand@RunCommand
      command: "curl -f https://flaky-api.example.com/healthcheck"
```

The `Retry` task takes two retry-control fields plus an `inner:`
field whose value is itself a task description (a sub-YAML).  The
implementation:

```python
import time

from ccai.errors.exceptions import ConfigurationError
from ccai.xmodules.ximport import import_by_name

from ccai.taskplus.model.task import Task


class Retry(Task):
    """
        Wraps another task and retries it on failure.

        :attempts: Maximum number of attempts (including the first).
        :delay_seconds: Seconds to sleep between attempts.
        :inner: A sub-task description with 'tasktype:' and any other fields
                 the inner task type requires.
    """

    def __init__(self, ordinal: str, label: str, task_info: dict, logger):
        super().__init__(ordinal, label, task_info, logger)

        if "inner" not in task_info:
            errmsg = "Retry requires an 'inner' field."
            raise ConfigurationError(errmsg)

        self._attempts = task_info.get("attempts", 3)
        self._delay_seconds = task_info.get("delay_seconds", 1)

        # Construct the inner task.
        inner_info = task_info["inner"]
        inner_tasktype = inner_info["tasktype"]
        inner_module_name, inner_class_name = inner_tasktype.split("@")
        inner_module = import_by_name(inner_module_name)
        inner_class = getattr(inner_module, inner_class_name)

        # Inner task gets a derived ordinal so its log lines are
        # distinguishable from the outer wrapper's.
        inner_ordinal = f"{ordinal}.inner"
        inner_label = f"{label} (inner)"
        self._inner = inner_class(inner_ordinal, inner_label, inner_info, logger)

        return

    def perform_task(self, environment, parameters, flowstate) -> int:
        last_result = 1

        for attempt in range(1, self._attempts + 1):
            self._logger.info(
                "Retry attempt %d of %d for %s" %
                (attempt, self._attempts, self._label)
            )
            last_result = self._inner.perform_task(
                environment, parameters, flowstate
            )
            if last_result == 0:
                return 0

            if attempt < self._attempts:
                self._logger.info(
                    "Inner task failed; sleeping %ds before retry" %
                    self._delay_seconds
                )
                time.sleep(self._delay_seconds)

        return last_result
```

Three things worth naming:

### The wrapper instantiates its inner task

The constructor uses the same `import_by_name` machinery the
workflow loader uses (chapter 20).  This is what lets the
`Retry` task wrap *any* other `Task` — `RunCommand`, `ShellScript`,
a custom task, or even another `Retry`.  The model's
`Task → perform_task` contract is the only thing the wrapper
needs to know about its inner.

### The wrapper preserves the model's contract

The wrapper's `perform_task` returns an int (0 = success,
non-zero = last failure).  The `FlowEngine` (chapter 22) sees
the wrapped task as a single task — the retry loop is internal.
This is the **Aspects pattern** from Volume 5: behaviour
parameters (retry policy) are packaged into a wrapping
construct rather than being added to the engine.  The chapter-18
"what's not in the model" discussion specifically called out
that retries belong inside tasks; this is the canonical example.

### The pattern composes

A workflow author can stack wrappers — a `Retry` wrapping a
`Timeout` wrapping a `RunCommand` is a perfectly sensible
composition:

```yaml
- task: Bounded retry
  tasktype: my.module@Retry
  attempts: 3
  delay_seconds: 5
  inner:
      tasktype: my.module@Timeout
      timeout_seconds: 30
      inner:
          tasktype: ccai.taskplus.tasks.runcommand@RunCommand
          command: "./run-flaky.sh"
```

Each wrapper handles one concern (retry, timeout, the actual
work).  The model's small-and-uniform interface is what makes
the composition clean.

## Anti-patterns to avoid

Three patterns that *look* like extensions but aren't.

### Don't subclass an existing concrete task

A `RunCommandWithRetry` that subclasses `RunCommand` to add retry
logic is the wrong shape.  Inheritance couples retry to the
specific command-running implementation; the same retry logic
would have to be duplicated for `ShellScriptWithRetry` and
`EmbeddedPythonWithRetry`.  The wrapper-task pattern from the
previous section is the right answer.

### Don't put framework concerns in tasks

A task that mutates the engine's `JumpContext`, that re-orders
the workflow's flow list, or that conditionally re-defines the
`if_expr` semantics is reaching past the model's interface into
the engine.  When a use case requires this, the right response
is *not* a custom task — it's either a custom engine (which the
volume's chapter-27 honest-scoping discussion would direct
elsewhere) or a workflow restructuring that expresses the
intent through declarative gates and jumps.

### Don't create tasks that wrap arbitrary Python code

A `RunPythonFunction` task that takes a `module:` and `function:`
field and calls the function is nominally extensible, but it's
also a *sub-engine* — the function it calls becomes a
mini-orchestration the framework can't see into.  The recipe is
to *make the work a task itself*, not to write a task that runs
arbitrary work.  The chapter-18 task model is small precisely so
that every step is *visible to the engine*.

## Sharing custom tasks

A team's custom tasks typically live in a separate package —
something like `myproject-taskplus`.  Three patterns for
sharing:

### Per-project `taskplus` package

The project's `pyproject.toml` declares a `myproject.taskplus`
namespace; tasks live there:

```
myproject-taskplus/
└── source/packages/myproject/taskplus/tasks/
    ├── gitcheckout.py            # the recipe target above
    ├── docker_pull.py
    ├── kubectl_apply.py
    └── retry.py                  # the wrapper above
```

Workflows reference them by full module path:

```yaml
- task: Pull image
  tasktype: myproject.taskplus.tasks.docker_pull@DockerPull
  ...
```

The package is installed alongside `codecraftai-taskplus` in the
orchestration's environment.  No registration step required.

### Cross-team task library

When multiple teams in an organisation share custom tasks, the
shared package becomes its own thing —
`codecraftai-taskplus-extras` or similar.  The pattern is the
same; the maintenance overhead grows with the number of teams,
which the foundation document's "cross-team profile sharing"
note also covers (chapter 14).

### Contributing back

Tasks that prove broadly useful can be contributed back to
`codecraftai-taskplus` itself.  The criterion: the task is generic
(not project-specific), well-tested, and fits the
"environment-neutral" discipline of the shipped set.  The
`GitCheckout` recipe target is itself a good candidate —
filling in the stub is exactly the contribution path, and the
recipe above is the implementation.

## Summary

- **The five shipped task types** cover most orchestration
  needs.  When the same wrapper appears in multiple workflows,
  when the shell command is structurally complex, or when the
  task needs to interact with the orchestration, a custom task
  is the right answer.
- **The recipe is six steps**: decide the YAML shape; write the
  constructor with required-field validation; write `perform_task`
  with credential resolution + work + error reporting + flowstate
  writes; reference by `module@Class` in the workflow YAML;
  write tests; document.  The total surface is roughly
  150 lines for a non-trivial task.
- **Wrapper tasks** like `Retry` and `Timeout` are the pattern
  for adding behaviour parameters (the chapter-5 Aspects
  pattern in Volume 5).  Wrappers preserve the model's small
  interface; concerns compose by stacking wrappers.
- **Three anti-patterns to avoid**: inheriting from concrete
  tasks (use wrappers instead), reaching into engine internals
  (the engine is bounded), wrapping arbitrary Python (make the
  work a task itself).
- **Custom tasks live in a separate package** (or directly in
  the project's namespace).  No central registration; the
  YAML's `module@Class` reference is the registration.  Tasks
  that prove broadly useful can be contributed back to
  `codecraftai-taskplus`.

This is the last chapter of Part V.  The volume now has the
complete implementation surface: the task model (ch 18), the
shipped types (ch 19), the YAML grammar (ch 20), the
orchestration-to-job contract (ch 21), the engine (ch 22), and
the extension recipe (ch 23).  Every piece anchored on
audited code; every code citation backed by 115 tests in the
test suite the audit pass produced.

Part VI is what remains: wiring the implementation to real CI
vendors, bridging to the other volumes (distributed jobs from
Volume 2, sysman from Volume 4, provisioning from Volume 6),
walking the case-study port end-to-end, and naming honestly
where this discipline is *not* the right fit.
