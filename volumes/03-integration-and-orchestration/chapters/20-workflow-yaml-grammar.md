# Chapter 20 — Workflow YAML Grammar

> *The workflow YAML is the file the workflow author writes.  The
> in-memory tree is what the engine runs.  This chapter is the
> bridge between the two.*

The previous two chapters introduced the task model and the five
shipped task types.  This chapter introduces the *YAML format*
that workflow authors write to compose those types into a
runnable orchestration, plus the
[`Workflow.parse`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/workflow.py)
classmethod that translates the YAML into the in-memory tree
chapter 18 described.

The grammar is small.  Six top-level fields, two node shapes
(`task` or `taskgroup`), four required-field validations, one
recursive walk that produces ordinals.  The smallness is the
point — a workflow YAML that fits in this grammar is one a
maintainer can read in five minutes.

## The top-level grammar

A workflow YAML is a single document with three required and
three optional top-level fields:

```yaml
# Required
name: <human-readable workflow name>
flow:
    - <flow node 1>
    - <flow node 2>
    - ...

# Optional
environment:
    <key>: <value>
    ...
parameters:
    <key>: <value>
    ...
```

The shipped
[`Workflow.parse`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/workflow.py)
implementation enforces this structure.  Two ConfigurationError
checks at the top:

```python
if "name" not in workflow_info:
    errmsg = "A workflow document must have a 'name' element."
    raise ConfigurationError(errmsg)

if "flow" not in workflow_info:
    errmsg = "A workflow document must have a 'flow' element which contains the tasks to be performed."
    raise ConfigurationError(errmsg)
```

Two optional fields fall back to empty dicts:

```python
environment = {}
if "environment" in workflow_info:
    environment = workflow_info["environment"]

parameters = {}
if "parameters" in workflow_info:
    parameters = workflow_info["parameters"]
```

Walk the four fields in order.

### `name`

The workflow's identity.  Required.  Carries no special
machinery — it does not select files, does not appear in
ordinals, does not affect execution.  It is the human-readable
identity the workflow shows up under in logs, in result reports,
in time-travel debugging captures.

A typical workflow names itself after its purpose:

```yaml
name: nightly-integration-suite
```

Workflow file naming is independent of the `name` field.  A file
called `nightly.yaml` containing `name: nightly-integration-suite`
is valid; consistency between the two is a maintenance preference,
not a framework requirement.

### `flow`

The list of top-level flow nodes — required.  Each item is either
a `task` node or a `taskgroup` node.  The list defines the
workflow's top-level sequence; nesting happens inside taskgroups.

The `flow` field is what the loader walks recursively to build
the in-memory tree.  Empty `flow: []` is permitted (a workflow
with no work) and runs successfully — useful for syntax-check
testing.

### `environment` (optional)

A dictionary of environment-variable overrides the workflow
applies before its flow runs:

```yaml
environment:
    BUILD_ID: "12345"
    DEPLOY_TARGET: "staging-east"
```

The shipped
[`execute_workflow`](../../../automojo-taskplus/source/packages/automojo/taskplus/execution.py)
function exports each entry into `os.environ` before the flow
executes.  This is the workflow's hook into the
plumbing-vs-content boundary discussed in chapter 7: the
environment block names the *fixed env-var interface* the
orchestration sets up for downstream processes.

The discipline from chapter 7 applies here.  The right
`environment:` block is small — typically the few `AM_*_NAMES`
selectors that name the configurations the workflow operates
under.  A workflow whose `environment:` block is fifteen lines
long is the chapter-2 env-var-accretion failure mode showing up
in workflow YAML.

### `parameters` (optional)

A dictionary of workflow-level parameters that flow to every
task's `perform_task` as the second argument:

```yaml
parameters:
    branch: main
    build: nightly-2026-04-30
    landscape: $HOME/automojo/config/default-landscape.yaml
```

Parameters are *workflow-time* values — known when the workflow
is invoked, used by tasks during execution.  They are distinct
from runtime configuration (chapter 7) which is *load-time*
content.  The boundary is fuzzy in small workflows; chapter 21
develops the contract in detail.

## The flow node shape

Each entry in the `flow:` list (and inside any `items:` list
inside a taskgroup) is a YAML mapping.  The mapping must have
*one* of two top-level keys: `task` or `taskgroup`.  The presence
of one key versus the other selects the node type.

The shipped parser enforces this with two ordered checks:

```python
if "task" in fnode_info:
    # ... task branch
elif "taskgroup" in fnode_info:
    # ... taskgroup branch
else:
    raise ConfigurationError("Unknown workflow node encountered. ...")
```

Three error cases are caught at parse time:

- **A `task` node that also has a `taskgroup` field** — caught
  with `raise ConfigurationError("A 'task' node should not have
  a 'taskgroup' field.")`.  A node should be one or the other;
  the YAML format does not let a single node be both.
- **A `taskgroup` node that lacks an `items` field** — caught
  with `raise ConfigurationError("A 'taskgroup' node must have
  an 'items' field.")`.
- **A node with neither `task` nor `taskgroup`** — caught with
  the unknown-node error, which dumps the offending node's
  content into the error message so the workflow author can find
  it.

These three checks are the entirety of structural validation in
the parser.  Beyond them, the grammar is permissive: a node may
carry any additional fields the framework doesn't recognise
without being rejected.

### Task node shape

```yaml
- task: <human-readable label>
  tasktype: <module>@<ClassName>
  if: <optional Python expression>
  onfailure: <optional label>
  onsuccess: <optional label>
  # ... task-specific fields ...
```

Two fields are required: `task` (the label) and `tasktype` (the
module-and-class reference).  The `if`, `onfailure`, and
`onsuccess` fields are optional and consumed by the
[`Task`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/task.py)
base class (chapter 18).  Task-specific fields — `command` for
`RunCommand`, `script` for `ShellScript`, `variables` for
`CreateEnvironment` — are read by the concrete task class
directly from `node_info` (chapter 19).

The `tasktype` field uses a `module@ClassName` syntax:

```yaml
tasktype: automojo.taskplus.tasks.runcommand@RunCommand
```

The parser splits on `@`, imports the module via
[`import_by_name`](../../../automojo-xmodules/source/packages/automojo/xmodules/ximport.py)
from `automojo-xmodules`, and looks up the class:

```python
task_type = fnode_info["tasktype"]
task_module_name, task_module_class = task_type.split("@")
task_module = import_by_name(task_module_name)

if hasattr(task_module, task_module_class):
    task_class = getattr(task_module, task_module_class)
    task_inst: Task = task_class(next_ordinal, task_label, fnode_info, logger)
    flow_items.append(task_inst)
else:
    error_msg = "The specified task module %r does not contain a class %r" % (
        task_module_name, task_module_class
    )
    raise ConfigurationError(error_msg) from None
```

Three observations on this dispatch:

- **Module imports happen at parse time.**  A workflow that
  references `my.module@MyTask` triggers an import of `my.module`
  during `Workflow.parse`.  Side effects of importing are paid
  during parsing, before any task runs.
- **The class lookup is `hasattr`-based.**  When the named class
  doesn't exist on the imported module, the error names *both*
  the module and the class — useful when the workflow author
  misspelled either one.
- **The reference is fully-qualified.**  There is no implicit
  module prefix, no shortform.  A `tasktype: RunCommand` would
  fail with a missing-`@` exception.  The fully-qualified
  requirement keeps the workflow's set of tasks explicit and
  greppable.

### Taskgroup node shape

```yaml
- taskgroup: <human-readable label>
  if: <optional Python expression>
  items:
      - <flow node>
      - <flow node>
      - ...
```

One required field beyond the universals: `items` (the list of
nested flow nodes).  Like the `flow` list at the top level, this
can contain `task` nodes, further `taskgroup` nodes, or a mix.

A taskgroup carries the `if` field but *not* the `onfailure` and
`onsuccess` fields — those are task-specific.  This is the
chapter-18 distinction: taskgroups are sequencing structures, not
units of work, so they don't have a notion of success or
failure of their own.  Failure handling for a taskgroup happens
through the failure of its constituent tasks.

## The recursive walk and ordinal generation

The parser's
[`_process_flow_info_items`](../../../automojo-taskplus/source/packages/automojo/taskplus/model/workflow.py)
method is the recursion that produces the in-memory tree.  It
takes a list of node info dictionaries and an `ordinal_prefix`
string, and returns a list of `FlowNode` instances:

```python
@classmethod
def _process_flow_info_items(cls, fnode_info_items: List[Dict[str, Any]],
                             ordinal_prefix: str = "") -> List[FlowNode]:

    flow_items: List[FlowNode] = []

    ordinal_position = 0

    for fnode_info in fnode_info_items:

        ordinal_position += 1

        next_ordinal = str(ordinal_position)
        if len(ordinal_prefix) > 0:
            next_ordinal = f"{ordinal_prefix}.{str(ordinal_position)}"

        # ... task or taskgroup branch ...

    return flow_items
```

Three things worth tracing.

### Position tracking

`ordinal_position` starts at zero and increments at the top of
each iteration.  After the increment, `next_ordinal` is `"1"` for
the first node, `"2"` for the second, and so on.  The list is
walked in declaration order, so ordinals match the YAML's
top-to-bottom order.

The audit pass on this file fixed a bug at this very line:
`ordinal_position` was previously *not* initialised before the
loop, raising `UnboundLocalError` on the first iteration.  The
shipped form initialises to zero (visible above).  The 115-test
suite added in the audit pass guards against regression.

### Ordinal prefixing

The `ordinal_prefix` parameter is empty for the top-level call
(producing ordinals `"1"`, `"2"`, `"3"`).  For recursive calls
into a taskgroup's items, the prefix is the taskgroup's own
ordinal (producing ordinals `"2.1"`, `"2.2"`, `"2.3"` inside the
taskgroup at ordinal `"2"`).

The recursive call:

```python
tg_titems = cls._process_flow_info_items(
    tg_items_info_list,
    ordinal_prefix=next_ordinal,
)
```

This is what produces the dotted-ordinal pattern from chapter 18.
Three-level nesting produces three-dot ordinals (`"2.3.1"`); the
recursion is unbounded, so any depth is valid.

### Taskgroup construction

When the loop encounters a taskgroup node, it recurses *first* to
build the children, then constructs the `TaskGroup`:

```python
tg_titems = cls._process_flow_info_items(
    tg_items_info_list,
    ordinal_prefix=next_ordinal,
)

tgroup_inst = TaskGroup(next_ordinal, tg_label, fnode_info, tg_titems, logger)
flow_items.append(tgroup_inst)
```

The post-audit form passes the `logger` argument explicitly.  The
audit found a missing-positional-arg bug here — pre-fix, the
`TaskGroup` constructor was called without the required `logger`
parameter, raising `TypeError` whenever a taskgroup appeared in a
workflow.  Fixed; covered by the test suite.

## Validation paths

The parser's validation is *structural* rather than *semantic*.
Three categories of error get caught:

### Missing required fields

- **`name`** missing from the document — `ConfigurationError`
  before any flow is processed.
- **`flow`** missing from the document — `ConfigurationError`
  before any flow is processed.
- **`tasktype`** missing from a task node — raises `KeyError`
  during the line `task_type = fnode_info["tasktype"]`.  The
  `KeyError` propagates; this is currently *not* a `ConfigurationError`
  but the symptom is the same (workflow fails to parse).
- **`items`** missing from a taskgroup node — `ConfigurationError`
  with the explicit message.

### Conflicting fields

- A `task` node also containing a `taskgroup` field — `ConfigurationError`
  rejecting the conflict.

### Unknown node shape

A node with neither `task` nor `taskgroup` — `ConfigurationError`
that includes the node's content in the message.  The post-audit
form uses `errmsg_lines.append(indent_lines(pformat(fnode_info,
indent=4, depth=2), level=1))` to format the offending node
nicely.  An audit finding fixed a previous bug where this used
`extend(...)` with a string (which iterates characters) instead
of `append(...)`.

### What's *not* validated

The parser does not validate:

- **Task-specific required fields.**  A `RunCommand` task without
  a `command` field will parse successfully and fail at first
  execution when `RunCommand.__init__` raises `KeyError("command")`.
  This is consistent with the chapter-18 design — the model
  doesn't know task-specific fields, and validation happens in
  the concrete task's constructor.
- **Cross-references.**  A task with `onfailure: cleanup` does
  not check whether a task labelled "cleanup" exists.  The check
  happens at execution time when the engine attempts the jump
  (chapter 22 develops this).
- **Schema for `environment` and `parameters`.**  The dictionaries
  are accepted as-is.  Type coercion and field validation are
  the workflow author's responsibility (or the tasks' responsibility
  to validate before they consume).

Three layers of validation, in summary: the parser validates
structure; concrete tasks validate their own fields at
construction; the engine validates cross-references at execution.
Each layer catches its appropriate class of error close to the
source.

## A complete worked example

Putting all the pieces together — a workflow YAML that covers
each construct:

```yaml
# nightly-integration.yaml
name: nightly-integration-suite

environment:
    AM_PROFILE: "integration-ci"
    AM_LANDSCAPE_NAMES: "staging-east"

parameters:
    branch: main
    build_id: nightly-2026-04-30
    cleanup_after_run: true

flow:
    # Top-level setup task
    - task: Setup
      tasktype: automojo.taskplus.tasks.runcommand@RunCommand
      command: "automojo provisioning apply --spec config/specs/spec-staging-east.yaml"

    # Test phase as a taskgroup
    - taskgroup: Test phase
      items:
          - task: Smoke tests
            tasktype: automojo.taskplus.tasks.runcommand@RunCommand
            command: "automojo testbase test run tests/smoke"

          - task: Integration tests
            tasktype: automojo.taskplus.tasks.runcommand@RunCommand
            command: "automojo testbase test run tests/integration"
            onfailure: Run cleanup

          - task: Benchmark
            tasktype: automojo.taskplus.tasks.runcommand@RunCommand
            command: "automojo testbase test run tests/benchmark"
            if: "parameters['build_id'].startswith('nightly')"

    # Cleanup phase as a taskgroup, only if cleanup_after_run
    - taskgroup: Cleanup phase
      if: "parameters['cleanup_after_run'] == True"
      items:
          - task: Run cleanup
            tasktype: automojo.taskplus.tasks.shellscript@ShellScript
            script:
                - "set -euo pipefail"
                - "automojo provisioning teardown --spec config/specs/spec-staging-east.yaml"
                - "rm -rf .output/staging-east-canary"

          - task: Notify
            tasktype: automojo.taskplus.tasks.embeddedpython@EmbeddedPython
            script:
                - "import os"
                - "import requests"
                - "requests.post(os.environ['SLACK_WEBHOOK_URL'], json={'text': 'Nightly run complete'})"
```

The in-memory tree the loader produces:

```
Workflow(name='nightly-integration-suite')
├── environment: {AM_PROFILE: integration-ci, AM_LANDSCAPE_NAMES: staging-east}
├── parameters: {branch: main, build_id: nightly-2026-04-30, cleanup_after_run: true}
└── flow:
    ├── [0] Task("Setup", ordinal="1")
    │            command: "automojo provisioning apply --spec ..."
    ├── [1] TaskGroup("Test phase", ordinal="2")
    │            items:
    │            ├── [0] Task("Smoke tests", ordinal="2.1")
    │            ├── [1] Task("Integration tests", ordinal="2.2",
    │            │            onfailure="Run cleanup")
    │            └── [2] Task("Benchmark", ordinal="2.3",
    │                         if_expr="parameters['build_id'].startswith('nightly')")
    └── [2] TaskGroup("Cleanup phase", ordinal="3",
                      if_expr="parameters['cleanup_after_run'] == True")
                 items:
                 ├── [0] Task("Run cleanup", ordinal="3.1")
                 └── [1] Task("Notify", ordinal="3.2")
```

Three things visible in the tree.

### Cross-group jump

The `Integration tests` task at ordinal `"2.2"` declares
`onfailure: Run cleanup`.  The engine resolves the label by
walking the whole tree (chapter 22) and finds the `Run cleanup`
task at ordinal `"3.1"`.  When the integration test fails, the
engine skips ahead — past `"2.3"` (Benchmark) and the entire
cleanup-phase taskgroup's gate evaluation — directly to the
labelled task.

### Conditional taskgroup

The `Cleanup phase` taskgroup carries `if_expr:
"parameters['cleanup_after_run'] == True"`.  When the flag is
set false, the engine skips the entire group.  Three children of
the group go silent in one decision.

### Conditional task within a non-conditional group

The `Benchmark` task within the `Test phase` taskgroup carries
its own `if_expr`.  The taskgroup runs unconditionally; the
benchmark child runs only when the build is a nightly.  The two
gates compose: the parent gate is checked first; if it allows
the group to run, each child's gate is then checked.

## What `Workflow.parse` does and doesn't do

The parser's job is structural.  Three things it *does*:

1. **Validate top-level shape** — name, flow, optionally
   environment and parameters.
2. **Recurse the flow tree** — produce ordinals, dispatch by
   `task`/`taskgroup`, instantiate the appropriate class.
3. **Resolve `tasktype` references** — import the named module,
   look up the named class, instantiate.

Three things it *doesn't*:

1. **Resolve cross-references.**  `onfailure: cleanup` is not
   validated against the rest of the tree; an unresolvable label
   surfaces only at execution time.
2. **Validate task-specific fields.**  `RunCommand` without a
   `command` field passes the parser; the missing field is caught
   when the task is instantiated.  Different tasks have different
   field requirements; the parser doesn't know them.
3. **Type-check `parameters` and `environment`.**  Both are
   accepted as plain dicts.  When tasks consume them, type
   coercion and validation happen at the consuming task's
   discretion.

The split is deliberate.  The parser is small; the concrete tasks
own their own field validation; the engine owns cross-reference
resolution.  Each layer's responsibility fits in a small piece
of code.

## Audit findings on `Workflow.parse`

The pre-book audit pass on this file found and fixed several
bugs:

- **`yaml.safe_load(workflow_stream, Loader=yaml.SafeLoader)`** —
  `safe_load` does not accept a `Loader` kwarg.  Fixed to plain
  `yaml.safe_load(workflow_stream)`.
- **`ordinal_position` used before initialisation** — caused
  `UnboundLocalError`.  Fixed to initialise at zero before the
  loop.
- **`tg_label = fnode_info["task"]`** — copy-paste typo where the
  taskgroup branch was reading the wrong key.  Fixed to
  `fnode_info["taskgroup"]`.
- **`TaskGroup(next_ordinal, tg_label, fnode_info, tg_titems)`** —
  missing the required `logger` positional arg.  Fixed.
- **`return` (no value)** at the end of `_process_flow_info_items`
  — the function should return `flow_items` but was returning
  `None`.  Fixed.
- **`errmsg_lines.extend(indent_lines(pformat(...)))`** — `extend`
  with a string iterates characters; should have been `append`.
  Fixed, with the call signature also corrected to include
  `level=1`.

Six bugs in one file.  The parser is now reliable; the 115-test
suite locks every fix in place.  This is the kind of "code that
clearly hadn't been runtime-tested" pattern the volume's audit
chapter (and the foundation document's audit-diary appendix
table) anchors on.

## Summary

- A workflow YAML is a single document with up to four
  top-level fields: **`name`** (required), **`flow`** (required),
  **`environment`** (optional dict of env-var exports),
  **`parameters`** (optional dict of workflow-level values).
- Each entry in `flow` (and inside any taskgroup's `items`) is
  either a **`task`** node or a **`taskgroup`** node.  A task
  node has a `task` (label) and a `tasktype` (module@Class
  reference); a taskgroup node has a `taskgroup` (label) and
  an `items` list.
- The `tasktype` reference uses `<module>@<ClassName>` syntax;
  the parser imports the module via `import_by_name` and
  instantiates the named class with the standard FlowNode
  constructor signature.
- The recursive walk produces ordinals automatically:
  top-level nodes get `"1"`, `"2"`, `"3"`; nested nodes inside
  a taskgroup at ordinal `"2"` get `"2.1"`, `"2.2"`, `"2.3"`;
  the recursion is unbounded.
- The parser validates *structural* shape (required fields,
  conflicting fields, unknown node shapes).  It does not
  validate cross-references (chapter 22), task-specific fields
  (concrete tasks do), or content schemas of `environment` and
  `parameters`.
- A complete worked example (a nightly-integration workflow with
  setup, a test phase, a conditional benchmark child, an
  `onfailure` jump, and a conditionally-skipped cleanup phase)
  shows every construct in one document.
- Six bugs in `Workflow.parse` were fixed during the pre-book
  audit pass.  The 115-test suite locks every fix in place; the
  parser is now reliable.

The next chapter develops the contract between an orchestration
and a job — how the workflow's `environment` and `parameters`
fields, plus the per-run `flowstate` dict, become the
information-flow channels between layers.
