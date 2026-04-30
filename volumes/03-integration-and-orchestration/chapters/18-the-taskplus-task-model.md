# Chapter 18 — The taskplus Task Model

> *Part V is the implementation.  The four parts before this one
> asked what the configurations should be and how they should
> compose; this part asks what runs them.*

Welcome to Part V.  The volume's discipline half — Parts I, II,
III, IV — is finished.  Six configuration categories named, a
credential discipline established, an environment-spanning profile
system proposed.  All of that has built up to one question: *what
is the thing that takes these typed inputs and runs an
orchestration?*

The answer in the portfolio is `automojo-taskplus`.  Six chapters
of Part V cover it: the task model (this chapter), the built-in
task types (chapter 19), the workflow YAML grammar (chapter 20),
how information flows from orchestration to job (chapter 21), how
flow control is expressed as data (chapter 22), and how to extend
the task model with new types (chapter 23).

By the end of Part V, the reader has the complete implementation
substrate for the discipline.  Part VI then wires it to real CI
vendors with worked examples.

## Why a task model at all

A workflow is a sequence of steps.  At the smallest scale, "a
workflow" can be a shell script — one command after another, with
exits between them.  At the next scale up, the shell script grows
conditionals, loops, and error handlers; eventually it has more
control flow than commands.

A task model is the answer to the question *"what's the smallest
thing the orchestration engine needs to know about a step in
order to run it?"*  Three properties are essential:

1. **A way to identify the step.**  Logs, traces, results all
   need to refer back to a specific step.  Without identity, when
   a step fails you can't say *which* step.
2. **A way to execute the step.**  The engine has to call into
   the step's implementation; the step has to return success or
   failure.
3. **A way to express control-flow context.**  Where does this
   step sit relative to other steps?  What happens if it fails?
   What happens if it succeeds?

The taskplus task model captures these three properties in three
classes — `FlowNode`, `Task`, `TaskGroup` — that together form a
small inheritance hierarchy in
[`automojo-taskplus/source/packages/automojo/taskplus/model/`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/).
The hierarchy is small enough to read in one sitting and rich
enough to express every workflow shape Part V's chapters will
build on.

## `FlowNode` — the abstract base

The root of the task model is
[`FlowNode`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/flownode.py),
an `abc.ABC` class with no abstract methods.  In its entirety:

```python
class FlowNode(abc.ABC):

    def __init__(self, ordinal: str, label: str, node_info: Dict[str, Any], logger: Logger):
        self._ordinal = ordinal
        self._label = label
        self._node_info = node_info
        self._logger = logger

        self._if_expr = node_info.get("if")
        return

    @property
    def if_expr(self) -> str:
        return self._if_expr

    @property
    def label(self) -> str:
        return self._label

    @property
    def node_info(self) -> dict:
        return self._node_info

    @property
    def ordinal(self) -> str:
        return self._ordinal
```

Five fields and four properties.  Every flow node — whether it's a
leaf task or a recursively-nested group — carries these.

### `ordinal` — the trace identity

The `ordinal` is a dotted string identifying the node's position
in the workflow.  Top-level nodes have ordinals `"1"`, `"2"`,
`"3"`; nested nodes have ordinals like `"2.3"` (the third item
inside the second top-level node) and `"2.3.1"` (the first item
inside that).

The ordinal is the trace key.  When a step fails, the log message
"step 2.3.1 failed" tells the engineer exactly where in the
workflow to look — even when the workflow has nested taskgroups
and parallel branches that defy a flat numbering.  The shipped
[`Workflow.parse`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/workflow.py)
generates ordinals as it walks the YAML; tasks and taskgroups
inherit their ordinals at construction time.

The ordinal is a *string* rather than a number.  This is deliberate
— `"2.10"` and `"2.2"` would compare incorrectly as floats, and
the dotted-path semantics is more important than the numeric one.

### `label` — the human identity

Where the ordinal is for machine routing, the `label` is for
human readability.  A task labelled `"Run smoke tests"` shows up
in logs and result reports under that name.  The label has no
uniqueness constraint within the workflow; two `"Cleanup"`-labelled
tasks at different ordinals are fine.

The label is the YAML's `task:` or `taskgroup:` field — when a
workflow author writes:

```yaml
- task: Run smoke tests
  tasktype: my.module@SmokeTest
```

the string `"Run smoke tests"` becomes the task's `label`.  This
is the chapter-20 grammar's view of the same field.

### `node_info` — the raw configuration

The `node_info` is the YAML dictionary that defined this node.
Every field the workflow author wrote — including the ones the
framework doesn't know about — is preserved here.  The
implementation detail matters: by storing the whole node info, the
task model lets concrete subclasses access task-specific fields
without the model needing to enumerate them.

When `RunCommand` (chapter 19) wants its `command` field, it
reads `node_info["command"]`.  When `EmbeddedPython` wants its
`script`, it reads `node_info["script"]`.  The base class doesn't
know about these fields; the subclasses pull what they need.

### `logger` — the observability hook

Every node carries a Python `Logger` reference.  This is the
mechanism for log emission during execution; concrete tasks
typically emit `logger.info("STEP: %s - %s" % (label, ordinal))`
at the start of execution and `logger.info` or `logger.error` for
results.

The logger is *passed in* rather than fetched.  This keeps the
task model decoupled from the specific logging configuration —
the workflow loader (chapter 20) chooses which logger every node
gets, which is how runtime configurations (chapter 7's logging
levels) reach the tasks without the tasks knowing about the
runtime configuration system.

### `if_expr` — the conditional gate

The base class also surfaces an optional `if` expression from the
node info.  Workflow authors declare it like this:

```yaml
- task: Run staging cleanup
  tasktype: my.module@StagingCleanup
  if: "parameters['cleanup_after_run'] == True"
```

The expression is a Python string evaluated by the
[`FlowEngine`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/flowengine.py)
at execution time.  When the expression is `None` (the field was
absent), the node always runs.  When the expression evaluates to
falsy, the node is skipped.

The `if_expr` is the chapter-22 flow control's gate; this chapter
just notes that it lives on every flow node and that the base
class exposes it.  Chapter 22 develops the evaluation semantics.

## `Task` — the leaf

A `Task` is a `FlowNode` that *does work*.  It has two
properties beyond the base — `onfailure` and `onsuccess` — and
two methods — `execute` and the abstract `perform_task`.

```python
class Task(FlowNode):

    def __init__(self, ordinal: str, label: str, node_info: Dict[str, Any], logger: Logger):
        super().__init__(ordinal, label, node_info, logger)

        self._onfailure = node_info.get("onfailure", None)
        self._onsuccess = node_info.get("onsuccess", None)

        return

    @property
    def onfailure(self) -> Optional[str]:
        return self._onfailure

    @property
    def onsuccess(self) -> Optional[str]:
        return self._onsuccess

    def execute(self, environment, parameters, flowstate) -> int:
        self._logger.info("STEP: %s - %s" % (self._label, self._ordinal))
        result = self.perform_task(environment, parameters, flowstate)
        return result

    @abc.abstractmethod
    def perform_task(self, environment, parameters, flowstate) -> int:
        raise AbstractMethodError("TaskBase->execute must be overloaded by derived classes.") from None
```

### `onfailure` and `onsuccess` — declarative jumps

Both fields are *optional*.  Each names another node's `label`
that the engine should jump to when this task ends in the
relevant outcome.

A task with `onfailure: cleanup` and `onsuccess: notify` means:
*"if I fail, jump to the task labelled cleanup; if I succeed,
jump to the task labelled notify."*  Tasks in between are
skipped.

This is the chapter-22 flow-control mechanism.  The chapter
develops the `JumpContext` machinery that drives the actual
jumps; the model simply records the pair of fields on every task.
Tasks with neither field set fall through to the next node in
sequence, which is the default behaviour.

### `execute` and `perform_task` — the two-method pattern

The `execute` method is concrete and not meant to be overridden.
It logs the step and delegates to `perform_task`.  The
`perform_task` method is abstract; concrete tasks (chapter 19)
override it with their actual work.

The split is deliberate.  The base class wants every task to
log its start and to be drivable through the same `execute(...)`
interface.  But the *contents* of the work — running a shell
command, executing a Python script, doing a Git checkout — are
task-specific.  The Template Method pattern keeps the
common-context concerns (logging, signature) in the base while
letting the concrete classes specify the work.

Three signature notes:

- The method takes three positional arguments: `environment`
  (typically the resolved runtime environment), `parameters` (the
  workflow-level parameters), and `flowstate` (a per-run mutable
  dictionary that tasks can read and write).  These are the
  three communication channels orchestration uses to talk to
  tasks; chapter 21 develops the contract.
- The method returns an `int`.  Zero is success; any non-zero
  value is failure.  The integer can be a process exit code (for
  shell-driven tasks), an enum (for application-defined error
  classifications), or just `0`/`1` for the simplest cases.
- The method is *not* allowed to raise to indicate failure.  Tasks
  that experience exceptions catch them and translate to a
  non-zero return.  This is partly a design choice (failure
  semantics are a single channel) and partly a pragmatic one (the
  `FlowEngine`'s sequencing relies on non-exception returns to
  drive `onfailure` jumps; an exception would skip the engine's
  flow logic).

The `AbstractMethodError` import on line 23 of `task.py`
([`automojo-errors`](../../../automojo-errors/source/packages/automojo/errors/exceptions.py))
is the canonical error for "this base method must be overridden."
A concrete task that forgets to override `perform_task` raises
this at first call.

## `TaskGroup` — the recursive grouping

A `TaskGroup` is a `FlowNode` that *contains other flow nodes*.
It does no work of its own; instead, it sequences and groups
its children.

```python
class TaskGroup(FlowNode):

    def __init__(self, ordinal: str, label: str, node_info: Dict[str, Any],
                 items: List[FlowNode], logger: Logger):
        super().__init__(ordinal, label, node_info, logger)

        self._items = items
        return

    @property
    def items(self) -> List[FlowNode]:
        return self._items
```

One additional field beyond the base: `items`, a list of child
`FlowNode` instances.  Children may themselves be `Task` instances,
or further `TaskGroup` instances.  The recursion is unbounded —
deeply nested groupings are valid.

The taskgroup is the only structure in the model that produces
the dotted-ordinal pattern — a `TaskGroup` at ordinal `"2"` with
three children gives them ordinals `"2.1"`, `"2.2"`, `"2.3"`.
Without the grouping, every node is at the top level and every
ordinal is a single number.

### Why grouping matters

Three reasons to group rather than just listing.

#### Logical separation

A workflow that has a "build" phase, a "test" phase, and a
"deploy" phase reads better when each phase is a `TaskGroup`
than when they're flat siblings.  The phase becomes one node
in the workflow's top-level list; what's inside the phase is a
nested concern.

#### Conditional skipping

Because `TaskGroup` is itself a `FlowNode`, it carries the
`if_expr` field.  A whole phase can be conditionally skipped:

```yaml
- taskgroup: Deploy phase
  if: "parameters['deploy_after_test'] == True"
  items:
      - task: Build artifact
        tasktype: my.module@BuildArtifact
      - task: Push to registry
        tasktype: my.module@RegistryPush
      - task: Deploy
        tasktype: my.module@Deploy
```

When `deploy_after_test` is False, the entire group is skipped
— three child tasks at once, with one `if` decision.

#### Failure jumps that span groups

A task inside group A can declare `onfailure:` jumping to a
label inside group B.  The engine resolves the jump by walking
through groups; the cross-group jump is allowed.  This means a
deploy task can jump to a cleanup taskgroup on failure, without
the deploy task having to reach inside the cleanup group's
implementation.

The semantics here are nuanced — chapter 22 develops them in
detail — but the model's role is to make them *expressible*.  The
recursive `TaskGroup` is the structure the engine walks.

## A worked example of the hierarchy

The four-class structure is best understood through a concrete
workflow.  Here is a small workflow YAML and the in-memory tree
the loader produces:

```yaml
# Workflow YAML (preview of chapter 20's grammar)
name: nightly-integration
flow:
    - task: Setup
      tasktype: my.module@SetupTask

    - taskgroup: Test phase
      items:
          - task: Smoke
            tasktype: my.module@SmokeTest
          - task: Integration
            tasktype: my.module@IntegrationTest
            onfailure: Skip teardown

    - taskgroup: Teardown phase
      items:
          - task: Stop services
            tasktype: my.module@StopServices
          - task: Skip teardown
            tasktype: my.module@NoOp
```

The in-memory representation:

```
Workflow
├── flow[0]: Task("Setup", ordinal="1")
├── flow[1]: TaskGroup("Test phase", ordinal="2")
│              items[0]: Task("Smoke", ordinal="2.1")
│              items[1]: Task("Integration", ordinal="2.2",
│                             onfailure="Skip teardown")
└── flow[2]: TaskGroup("Teardown phase", ordinal="3")
               items[0]: Task("Stop services", ordinal="3.1")
               items[1]: Task("Skip teardown", ordinal="3.2")
```

Five `Task` nodes and two `TaskGroup` nodes, every node a
`FlowNode` carrying `(ordinal, label, node_info, logger)`.  The
`Integration` task at ordinal `"2.2"` has `onfailure="Skip
teardown"`; when it fails the engine looks up the label "Skip
teardown" across the whole tree, finds it at ordinal `"3.2"`, and
jumps the execution there.

The same shape recurses arbitrarily.  A workflow with a `Test
phase` taskgroup containing a `Per-host setup` taskgroup
containing five `Run hostprep` tasks would be a three-deep tree;
the model handles it with no special-case code.

## Universal field invariants

A few invariants worth naming, since the chapter has implicitly
relied on them.

- **`ordinal` is unique within a workflow.**  The
  `Workflow.parse` machinery enforces this by construction (the
  loader generates ordinals in walk order).
- **`label` is *not* required to be unique.**  Two tasks both
  labelled "Cleanup" at different ordinals can coexist.  The
  jump machinery in chapter 22 handles label collisions by
  matching the *first* task with the named label in the engine's
  walk order; consult that chapter for edge cases.
- **`node_info` is the workflow author's source of truth.**  Every
  field the author wrote is preserved; the model adds nothing.
  Concrete tasks pull what they need; unknown fields are ignored
  silently.  This is permissive on purpose — workflow YAML can
  carry application-specific metadata the framework doesn't
  understand.
- **`logger` is constructed once and shared.**  Every node in a
  given workflow carries the same logger instance.  Per-task
  logging differentiation is handled by the logger's filtering
  and routing, not by per-task logger creation.
- **The recursion is finite.**  A taskgroup whose items list
  contains itself would be a cycle; the loader's
  `_process_flow_info_items` recurses but does not detect cycles
  because the YAML format makes cycles impossible to express.

## What's *not* in the task model

Three things the task model deliberately omits.

### Concurrency

The model has no notion of *parallel* execution.  Tasks within a
group run sequentially in declaration order.  A task group does
*not* dispatch its items concurrently.

When parallel execution is needed, the workflow author either:

- Uses a different task implementation (a `RunCommand` task that
  itself spawns parallel subprocesses).
- Handles it at a higher layer (multiple workflows running in
  parallel from a CI vendor's matrix-build feature).
- Extends the task model with a new node type (chapter 23 covers
  the extension recipe).

The omission is deliberate — Part V's discipline is that the task
model is *small and predictable*.  Adding concurrency at the
model layer would require a scheduling abstraction the simple
sequential model doesn't need.

### Retries

The model has no notion of *retries*.  A task that returns
non-zero is a failed task; the engine's flow control (`onfailure`
jumps) takes over.  The model does not retry the task with
backoff or with reset state.

When retries are needed, they live *inside* a task's
`perform_task`.  A concrete `RetryingHttpTask` could implement its
own retry loop; the model would see one task that took five
attempts to return zero.  Chapter 23's extension recipe shows the
pattern.

This is consistent with the shipped portfolio's **Aspects pattern**
from chapter 5 of Volume 5: behaviour parameters belong on the
task, not on the engine.

### State

The model has no first-class state beyond what's in `flowstate`
(the per-run mutable dictionary passed through `perform_task`).
Tasks read from it and write to it freely; there's no schema, no
type checking, no scoping.

When richer state management is needed — typed state, scoped
visibility, persistence — the workflow author uses an external
state store (a database, a key-value cache, a configuration map)
and references it through the runtime configuration.  The
discipline is to keep `flowstate` small and ad-hoc; persistent
state belongs in proper data stores.

## Summary

- The taskplus task model is a small inheritance hierarchy in
  [`automojo-taskplus/source/packages/automojo/taskplus/model/`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/).
  Three classes: **`FlowNode`** (abstract base), **`Task`**
  (leaf with work), **`TaskGroup`** (recursive grouping).
- Every flow node carries five universal fields:
  - `ordinal` — dotted-path machine identity (e.g., `"2.3.1"`).
  - `label` — human-readable identity from the YAML's
    `task:`/`taskgroup:` field.
  - `node_info` — the raw YAML dictionary, preserved verbatim.
  - `logger` — observability hook injected by the loader.
  - `if_expr` — optional gate evaluated by the FlowEngine.
- `Task` adds `onfailure` and `onsuccess` (label-based jumps)
  plus the abstract `perform_task(environment, parameters,
  flowstate) -> int` method.  Zero is success; non-zero is
  failure; tasks do not raise to indicate failure.
- `TaskGroup` adds `items: List[FlowNode]` and contains other
  flow nodes recursively.  The recursion produces the
  dotted-ordinal pattern.
- The model is *deliberately small*: no concurrency primitives,
  no retry primitives, no first-class state.  Extensions live in
  task implementations (chapter 19, chapter 23), not in the
  model.
- The same model handles flat workflows, nested phases,
  cross-group failure jumps, and conditionally-skipped phases
  uniformly — without special-case code.

The next chapter introduces the five built-in task types
shipped in `automojo-taskplus/tasks/`: `RunCommand`,
`ShellScript`, `EmbeddedPython`, `GitCheckout`, and
`CreateEnvironment`.  Each is a concrete `Task` subclass; each
illustrates the model's `perform_task` pattern; the chapter
notes which are well-shipped and which (per the audit findings)
are transitional or stub.
