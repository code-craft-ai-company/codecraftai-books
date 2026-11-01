# Chapter 22 — Flow Control as Data

> *Sequencing a workflow is a property of the workflow file, not
> of the engine.  The engine reads gates and jumps from data; the
> data lives in the YAML the author wrote.*

The previous chapters introduced the model (chapter 18), the
shipped tasks (chapter 19), the YAML grammar (chapter 20), and
the four communication channels (chapter 21).  This chapter walks
the engine that *runs* the workflow — specifically, the small set
of mechanisms by which flow control becomes data in the YAML and
behaviour at runtime.

The
[`FlowEngine`](https://github.com/code-craft-ai-company/codecraftai-taskplus/blob/main/source/packages/ccai/taskplus/model/flowengine.py)
class shipped in `codecraftai-taskplus` is small — under a hundred
lines.  It implements three flow-control mechanisms:

1. **The `if_expr` gate** — conditional inclusion of a node.
2. **The `onfailure` / `onsuccess` jump** — label-based jumps
   driven by task return codes.
3. **The recursion into TaskGroups** — sequencing of nested
   structure.

Each mechanism is *data-driven*: the workflow author declares the
behaviour in the YAML, the engine reads the declaration and
acts on it.  No imperative orchestration code; the workflow file
is the source of truth for how the workflow behaves.

## The engine's outer loop

The shipped
[`FlowEngine.execute`](https://github.com/code-craft-ai-company/codecraftai-taskplus/blob/main/source/packages/ccai/taskplus/model/flowengine.py)
method is the outermost layer.  It runs once per workflow
invocation:

```python
def execute(self, workflow: Workflow) -> int:

    result_code = 0

    environment = workflow.environment
    parameters = workflow.parameters

    flowitems = workflow.flow

    flowstate = {}
    errors = []
    jump_context = JumpContext()

    self.process_flowitems(environment, parameters, flowstate, errors, flowitems, jump_context)

    if len(errors) > 0:
        result_code = 1

    return result_code
```

Six lines of visible behaviour:

1. **Pull `environment` and `parameters` from the workflow.**
   These are the chapter-21 channels 1 and 2.  The engine threads
   them through every task as the first two arguments to
   `perform_task`.
2. **Initialise an empty `flowstate` dict.**  This is channel 3 —
   the per-run mutable inter-task communication channel.
3. **Initialise an empty `errors` list.**  Tasks (or the engine
   itself) can append error records during the run; the final
   non-empty errors list translates to a non-zero workflow result.
4. **Initialise a `JumpContext`.**  The label-jump state for the
   whole workflow; the engine threads this through every level of
   recursion.
5. **Call `process_flowitems` on the top-level flow list.**  This
   is the recursive walk; every flow node visited or skipped flows
   through this method.
6. **Translate `errors` into a final result code.**  Empty errors
   list → success; non-empty → failure.

The engine is *single-pass*.  No retries at the engine level (the
chapter-18 design choice — retries belong inside tasks, not in
the engine), no parallelism, no backtracking.  One walk through
the flow tree, in order, with skip-and-jump primitives applied.

## `process_flowitems` — the recursive walk

The recursion is where every flow control decision is made.  In
its entirety:

```python
def process_flowitems(self, environment, parameters, flowstate, errors,
                     flowitems: List[FlowNode], jump_context: JumpContext):

    for fitem in flowitems:

        node_label = fitem.label

        # If we are seeking a specific node, skip any nodes that are not that node
        if jump_context.seeking_node is not None:
            if jump_context.seeking_node != node_label:
                continue
            else:
                jump_context.seeking_node = None

        include_node = True
        if fitem.if_expr is not None:
            include_node = self.evaluate_expression(fitem.if_expr, environment, parameters)

        if include_node:
            if isinstance(fitem, Task):

                task_result = fitem.execute(environment, parameters, flowstate)
                if task_result != 0:
                    if fitem.onfailure is not None:
                        jump_context.seeking_node = fitem.onfailure
                elif fitem.onsuccess is not None:
                    jump_context.seeking_node = fitem.onsuccess

            elif isinstance(fitem, TaskGroup):

                group_items = fitem.items

                self.process_flowitems(environment, parameters, flowstate, errors, group_items,
                                       jump_context)

    return
```

Twenty-five lines of method body.  Walk what each block does.

### The jump check (lines 67–72)

The first thing the loop does for each node is check whether a
jump is in progress.  When `jump_context.seeking_node` is set,
the engine is in *seek mode* — looking for a specific labelled
node, skipping everything until it finds it.

```python
if jump_context.seeking_node is not None:
    if jump_context.seeking_node != node_label:
        continue
    else:
        jump_context.seeking_node = None
```

Three behaviours:

- **Not seeking** (`seeking_node is None`): proceed normally.
- **Seeking and current node doesn't match** (`seeking_node !=
  node_label`): `continue` — skip this node entirely.  Do not
  evaluate its gate; do not execute it; do not recurse into it
  if it's a taskgroup.
- **Seeking and current node matches** (`seeking_node ==
  node_label`): clear the seeking state and fall through to
  process this node normally.

The seek-and-skip pattern is the engine's *only* flow-control
primitive beyond sequential traversal.  All cross-node jumps
reduce to setting `seeking_node` and letting the loop's next
iterations skip until they reach the named target.

### The gate evaluation (lines 74–76)

When the node is *not* being skipped by a jump, the engine checks
the node's `if_expr`:

```python
include_node = True
if fitem.if_expr is not None:
    include_node = self.evaluate_expression(fitem.if_expr, environment, parameters)
```

Two behaviours:

- **No `if_expr`** (`fitem.if_expr is None`): `include_node`
  stays True; the node will run.
- **`if_expr` set**: evaluate the expression with the workflow's
  environment and parameters as context.  When the expression
  returns truthy, `include_node` becomes True; when it returns
  falsy, `include_node` becomes False and the node is skipped.

The gate runs *after* the seek check.  This means: a node can be
skipped by either being on the wrong side of a jump or by having
its gate evaluate false.  Both produce the same outcome (the
node is bypassed) through different mechanisms.

### The execution branch (lines 78–93)

When the node is included, the engine dispatches by node type:

#### Task path

```python
if isinstance(fitem, Task):

    task_result = fitem.execute(environment, parameters, flowstate)
    if task_result != 0:
        if fitem.onfailure is not None:
            jump_context.seeking_node = fitem.onfailure
    elif fitem.onsuccess is not None:
        jump_context.seeking_node = fitem.onsuccess
```

The task's `execute` method runs (which delegates to
`perform_task` per chapter 18's Template Method pattern) and
returns an integer.  Then:

- **Task failed** (`task_result != 0`) and **`onfailure` is
  declared**: set `seeking_node` to the `onfailure` label.  The
  loop's next iteration will start skipping until the named
  task is found.
- **Task failed** and **`onfailure` is *not* declared**:
  do nothing — the task failure is silently absorbed and
  execution continues to the next node.  (Whether this is the
  right semantic is discussed below.)
- **Task succeeded** (`task_result == 0`) and **`onsuccess` is
  declared**: set `seeking_node` to the `onsuccess` label.
- **Task succeeded** and **`onsuccess` is *not* declared**:
  fall through to the next node normally.

#### TaskGroup path

```python
elif isinstance(fitem, TaskGroup):

    group_items = fitem.items

    self.process_flowitems(environment, parameters, flowstate, errors, group_items,
                           jump_context)
```

The engine recurses into the taskgroup's items, threading the
*same* `jump_context` into the recursion.  This is what allows
**cross-group jumps**: when a task inside group A sets
`seeking_node` to a label inside group B, the recursion bubbles
back up to the outer loop, which keeps skipping until the
recursion descends into group B and finds the labelled node.

The recursion is what makes the engine's flow control compose.
The same `process_flowitems` method handles both top-level
sequences and nested taskgroup items; every node is at *some*
depth, but the depth doesn't change how flow control works.

## `evaluate_expression` — the gate evaluator

The gate evaluator is two non-trivial lines:

```python
def evaluate_expression(self, if_expr: str, environment: Dict[str, str],
                        parameters: Dict[str, Any]) -> bool:

    flowstate = self._flowstate

    eval_result = eval(if_expr)

    return eval_result
```

The implementation uses Python's built-in `eval` against a string
expression, with three names available in the evaluation scope:
`environment`, `parameters`, and `flowstate`.

Three observations.

### The expression is Python

The `if_expr` field in the YAML is a *Python expression string*.
A workflow's gate looks like this:

```yaml
- task: Run benchmark
  tasktype: my.module@Benchmark
  if: "parameters['build_id'].startswith('nightly')"

- task: Tear down staging
  tasktype: my.module@Teardown
  if: "parameters['cleanup_after_run'] == True and not flowstate.get('teardown_skipped', False)"
```

The YAML author writes Python.  This is more powerful than (say)
a custom expression DSL — comparison operators, string methods,
boolean logic, dict subscript, and method calls are all
available.

It is also more dangerous than a custom DSL.  `eval` on
attacker-controlled input is well-known as a security
vulnerability; the discipline relies on workflow YAML being
*authored by trusted developers* and being checked into the
repository.  When a workflow's `if_expr` would consume external
input, the workflow author should prefer to compute the
condition in a task and write a boolean to flowstate, then gate
on the simpler `flowstate['xyz']` expression.

### `flowstate` is the engine's, not the workflow's

The evaluator pulls `flowstate` from `self._flowstate` —
specifically, the engine's instance variable.  The workflow's
flowstate dict (the per-run dict initialised in `execute` and
threaded through `process_flowitems`) is *different* from
`self._flowstate`.  This is a subtle implementation detail of
the shipped engine; gates that try to read flowstate may not see
what tasks have written.

This is a known limitation.  Workflow authors who want gates to
read task-produced state should use `parameters` (read-only,
visible everywhere) for stable values, or work around the
limitation by writing a custom task that combines computation
and conditional behaviour into one step.

### Errors propagate

When `eval` raises (a syntax error in the expression, a
`KeyError` from missing parameters, a `NameError` from referring
to an undefined name), the exception propagates through
`evaluate_expression` and through `process_flowitems` up to the
caller.  The engine does *not* catch the exception; the workflow
fails.

This is consistent with the volume's "errors should never pass
silently" discipline (chapter 0, foundation tier).  A malformed
gate is a workflow bug; failing loudly is better than treating
the gate as falsy and silently skipping the node.

## A worked example of jump propagation

To make the jump mechanics concrete, walk through what happens
when a task fails and triggers a cross-group jump.

The workflow:

```yaml
name: cross-group-jump-example
flow:
    - task: Setup
      tasktype: my.module@Setup

    - taskgroup: Test phase
      items:
          - task: Smoke
            tasktype: my.module@Smoke
          - task: Integration
            tasktype: my.module@Integration
            onfailure: Run cleanup
          - task: Benchmark
            tasktype: my.module@Benchmark

    - taskgroup: Cleanup phase
      items:
          - task: Stop services
            tasktype: my.module@StopServices
          - task: Run cleanup
            tasktype: my.module@RunCleanup
```

The `Integration` task at ordinal `"2.2"` has `onfailure: Run
cleanup`, naming the task at ordinal `"3.2"`.  Trace what the
engine does when `Integration` fails.

### Time T₀: Engine begins

```
seeking_node = None
flowitems = [Setup, TestPhase, CleanupPhase]
```

The outer loop starts walking the top-level flow.

### Time T₁: Setup runs

`Setup` is processed normally.  `seeking_node` was None;
`if_expr` was None; the engine executes the task.  Suppose it
succeeds; no `onsuccess` declared; fall through.

```
seeking_node = None
just-processed = Setup (ordinal 1)
```

### Time T₂: Engine recurses into Test phase

`TestPhase` is a `TaskGroup`; the engine calls
`process_flowitems` recursively with `flowitems =
[Smoke, Integration, Benchmark]`.

```
seeking_node = None
recursion-depth = 1
inner-flowitems = [Smoke, Integration, Benchmark]
```

### Time T₃: Smoke runs

The inner loop processes `Smoke`.  Succeeds; no `onsuccess`;
fall through.

```
seeking_node = None
just-processed = Smoke (ordinal 2.1)
```

### Time T₄: Integration runs and fails

The inner loop processes `Integration`.  The task's
`perform_task` returns non-zero.  The engine sees the failure
and the `onfailure` label:

```python
if task_result != 0:
    if fitem.onfailure is not None:
        jump_context.seeking_node = fitem.onfailure
```

`seeking_node` becomes `"Run cleanup"`.  The inner loop's next
iteration begins.

```
seeking_node = "Run cleanup"
just-processed = Integration (ordinal 2.2, FAILED)
```

### Time T₅: Benchmark is skipped

The inner loop reaches `Benchmark`.  `seeking_node` is set to
`"Run cleanup"`; `Benchmark`'s label is `"Benchmark"`; mismatch.
The engine `continue`s.  No gate evaluation, no execution.

```
seeking_node = "Run cleanup"
just-skipped = Benchmark (ordinal 2.3)
```

### Time T₆: Inner loop ends; recursion returns

The inner loop's iteration is complete.  `process_flowitems`
returns to its caller (the outer loop).  Critically, the
`seeking_node = "Run cleanup"` state survives the return — both
the outer and inner loops share the same `jump_context`
reference.

```
seeking_node = "Run cleanup"
recursion-depth = 0 (returned)
```

### Time T₇: Outer loop resumes; CleanupPhase reached

The outer loop's next iteration is `CleanupPhase`.  The engine
checks: `seeking_node = "Run cleanup"`; CleanupPhase's label is
`"Cleanup phase"`; mismatch.

But this is a TaskGroup, not a Task.  Does the seek-and-skip
treatment of TaskGroups skip the *whole group* or just the
group node itself?

Looking at the loop structure:

```python
for fitem in flowitems:
    node_label = fitem.label

    if jump_context.seeking_node is not None:
        if jump_context.seeking_node != node_label:
            continue       # <-- skips this iteration entirely
```

The `continue` skips the entire iteration — including the
recursion into the TaskGroup's items.  So when the outer loop
encounters `CleanupPhase` while seeking `"Run cleanup"` and the
group's *own* label doesn't match, the *whole group is
skipped without descending into its items*.

This is a subtle behaviour worth naming: **the engine's
seek-and-skip applies at every level of the tree, including
taskgroup boundaries**.  When a jump's target is *inside* a
taskgroup, the engine has to first enter the taskgroup —
which means matching the taskgroup's *own* label is the only
way for seek mode to descend into the items.

Look back at the workflow: `CleanupPhase` is labelled "Cleanup
phase".  The jump target is "Run cleanup".  Mismatch.  The
group is skipped.  *The cross-group jump doesn't reach its
target.*

This is a real subtlety in the shipped engine.  Workflow
authors who want cross-group jumps to work as intuited (jumping
from inside one group to a label inside another) should
declare the *outer* group's label to match the seek target — or
restructure so that the jump targets a top-level task rather
than one inside a different taskgroup.

The volume's foundation-document outline notes this kind of
subtlety as a place where the discipline of *expressing flow
control as data* exposes its limits: jumps are powerful in
theory, brittle in nested practice.  Chapter 26's case study
revisits the pattern and shows the workaround.

## What flow control as data buys

Despite the subtlety above, the discipline of flow control as
data has clear payoffs over imperative orchestration.

### Reviewability

A workflow's behaviour is visible in the YAML.  When a workflow
fails in production, the engineer reading the YAML can see
which gates and jumps applied to that run.  An imperative
orchestration with branches, loops, and exception handlers
distributed across a Python script does not have this property
— understanding what *would* have happened requires running the
code in your head.

### Captured-profile compatibility

Chapter 16's capture-and-replay primitive serialises the
*resolved configuration* but not the *engine's runtime
behaviour*.  This works because the engine's behaviour is
*derivable* from the workflow YAML plus the captured
configurations.  An imperative orchestration's behaviour would
also depend on the engine's internal state, the order of
exception handlers, and other runtime details that wouldn't
serialise cleanly.

### Audit story

A configuration change to the workflow's flow is a git diff,
visible at code review time.  An imperative-orchestration
change spread across multiple Python files is harder to review
as a unit.  The discipline keeps the changes small and
reviewable, the same way every other chapter of the volume has
argued for.

### Predictability

The engine's behaviour is bounded.  The shipped
`process_flowitems` is twenty-five lines; the worst-case
behaviour for a given input is computable.  The engine does
not retry, does not back off, does not introduce
non-determinism beyond what's in the workflow's `if_expr`
strings.  When a workflow misbehaves, the cause is in the YAML
or in a task's implementation, not in the engine's hidden
state.

## What flow control as data does *not* buy

Three honest scoping notes.

### Not arbitrary control flow

The engine has *gates* and *jumps*.  It does not have *loops*,
*parallelism*, *try/except*, or *coroutines*.  When the
orchestration needs richer control flow — running three tasks
concurrently, retrying with exponential backoff, branching on
runtime conditions richer than a single boolean — the right
answer is either:

- A custom `Task` implementation that does the richer behaviour
  internally (chapter 23 develops the extension recipe).
- A higher-level orchestration tool entirely (Airflow, Argo,
  Step Functions) that the workflow author chooses *because* it
  has the primitives this engine doesn't.

The chapter-27 ("where this approach is *not* the right fit")
honest-scoping discussion includes "heavy parallel fan-out" and
"dynamic workflow generation" precisely because the engine's
small set of primitives doesn't fit those use cases.

### Not transactional

The engine has no notion of *atomic* multi-task operations.
When task A succeeds and task B fails halfway through, there's
no built-in mechanism to "undo" task A.  Tasks that need
transactional semantics implement them themselves (most commonly
through chapter-19's `onfailure` jumps to a cleanup task).

The limitation is acceptable for most orchestration use cases
— pipeline steps are typically idempotent or have explicit
cleanup paths — but workflow authors should be aware that the
engine is not a transaction manager.

### Not parallelism

A `TaskGroup` does not run its items in parallel.  Items run
sequentially in declaration order.  When parallelism is
needed, the workflow author either:

- Spawns parallel work from inside a custom Task
  implementation.
- Uses the CI vendor's native parallelism (matrix builds,
  parallel jobs).
- Restructures the workflow into multiple workflows that the CI
  vendor runs in parallel.

The shipped engine is a sequential one-pass walker.  This is
deliberate (chapter 18's "what's not in the model" discussion);
parallelism at the engine level would require a scheduling
abstraction the simple model doesn't need.

## Audit findings on `flowengine.py`

The pre-book audit pass on this file found and fixed several
bugs.  All are now covered by the 115-test suite that came out
of the audit:

- **`evaluate_expression` had a `topology` parameter** in its
  signature that was never used and never passed.  Calls into
  the function from `process_flowitems` only passed
  `(if_expr, environment, parameters)`; the extra parameter was
  a vestigial leftover from an earlier API.  Fixed by removing
  the parameter.
- **`process_flowitems` had `topology` in its signature** with
  the same vestigial-parameter problem.  `execute` was passing
  it; the inner uses didn't reference it.  Fixed by removing.
- **`execute` had `result_code = 0`** before
  `process_flowitems` was called, then *never updated* unless
  errors were appended.  Tasks failing without `onfailure`
  jumps were silently absorbed; the workflow result code
  reflected only errors that someone had explicitly added to the
  errors list.  This is *still* the shipped behaviour — failing
  tasks without `onfailure` declarations don't increment the
  errors list — and the audit chose to preserve the behaviour as
  documented rather than change it; chapter-27's scoping
  discussion notes this as a known limitation.

The errors-list-as-result-channel pattern is the engine's
weakest design point.  A future iteration could replace it with
a strict-by-default mode where any unhandled task failure marks
the workflow as failed; the volume documents the current
behaviour rather than papering over it.

## Summary

- The
  [`FlowEngine`](https://github.com/code-craft-ai-company/codecraftai-taskplus/blob/main/source/packages/ccai/taskplus/model/flowengine.py)
  is a small (~100 LOC) sequential engine implementing three
  flow-control mechanisms: the `if_expr` gate, the
  `onfailure`/`onsuccess` label jump, and the recursive
  TaskGroup walk.
- The engine's outer loop (`execute`) sets up environment,
  parameters, flowstate, errors, and a `JumpContext`, then
  delegates to the recursive `process_flowitems`.
- Flow control is *seek-and-skip*: when a jump fires,
  `jump_context.seeking_node` is set; subsequent loop iterations
  skip every node whose label doesn't match, including taskgroup
  boundaries.
- The `if_expr` gate is a Python expression evaluated with
  `eval` against an environment containing `environment`,
  `parameters`, and `flowstate` (the engine's `_flowstate`, with
  the limitation that this is *not* the same as the workflow's
  per-run flowstate dict).
- Cross-group jumps work *if* the engine descends into the right
  taskgroup before encountering the target.  Jumps targeting
  labels inside a different taskgroup will skip the wrong group
  unless the workflow author structures the labels carefully —
  a real subtlety chapter 26 will revisit.
- The discipline of "flow control as data" buys reviewability,
  captured-profile compatibility, an audit story, and bounded
  predictability — at the cost of giving up arbitrary control
  flow, transactionality, and engine-level parallelism.
- The audit pass on `flowengine.py` removed two vestigial
  parameters (`topology`) and noted the errors-list-as-result-
  channel pattern as the engine's weakest design point.

The next chapter is the last in Part V: extending the task
model.  How to add a new task type when the five shipped types
don't cover the use case, with the chapter-19 `GitCheckout`
stub as the worked target.
