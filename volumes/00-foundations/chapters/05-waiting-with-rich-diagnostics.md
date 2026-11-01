# Chapter 5 — Waiting with Rich Diagnostics

> *Every codebase that interacts with anything outside its
> process has a `wait_for(...)` function in it somewhere.  Most
> of them are buggy.  This chapter is about the small
> investment that makes the function not buggy.*

The previous chapter introduced the interfaces.  This chapter
introduces the smallest substantial subpackage in the foundation
tier:
[`ccai.waiting`](https://github.com/code-craft-ai-company/codecraftai-foundations/tree/main/source/packages/ccai/waiting),
shipped by `codecraftai-foundations`.
Three files, two classes, one function.  The subpackage's job is
to give the rest of the portfolio a *correct* answer for "wait
for a condition with a timeout, and tell me what happened if
it didn't work."

The package's interface is small enough to fit in this paragraph:
the `wait_for_it(looper, *args, timeout=N, interval=I, ...)`
function calls `looper(wait_context)` repeatedly until either
the looper returns `True` (condition met, done) or the timeout
elapses (raise `TimeoutError` with rich context).  The function
is half a page; the *correctness* of the function is the rest
of this chapter.

## Why a package for this

A naive `wait_for` is half a dozen lines:

```python
def wait_for(condition, timeout=60, interval=5):
    deadline = time.time() + timeout
    while time.time() < deadline:
        if condition():
            return True
        time.sleep(interval)
    return False
```

Every Python codebase has some version of this function.  Most
of them ship subtle bugs:

- The `True/False` return loses every bit of *why* it didn't
  succeed.  The caller has to wrap the call with their own
  diagnostic.
- The final iteration is the *last* one before the deadline; the
  caller wants to know whether the deadline cut off a
  *near-miss* or a *not-even-close* — there's no way to tell.
- The first iteration runs *immediately*; sometimes the
  condition needs a brief warm-up, and the first check is wasted.
- The interval is constant; sometimes it should be variable
  (back off on long-running waits).
- The condition is called on every loop; logging that the wait
  is still in progress requires an out-of-band hack.
- The timeout's *what for* — what was the wait actually waiting
  for? — never makes it into the resulting error.

Each bug is small.  Cumulatively, they make every team's
`wait_for` slightly different and slightly worse than it could
be.  The
[`ccai.waiting`](https://github.com/code-craft-ai-company/codecraftai-foundations/tree/main/source/packages/ccai/waiting)
subpackage's job is to fix all of them in one place so that every
caller benefits.

## The four shipped types

The package defines exactly four types in
[`waitmodel.py`](https://github.com/code-craft-ai-company/codecraftai-foundations/blob/main/source/packages/ccai/waiting/waitmodel.py):

- **`TimeoutContext`** — the state object that tracks a single
  wait operation.
- **`WaitCallback`** — a `Protocol` defining the looper-function
  signature.
- **`WaitContext`** — currently a thin alias for `TimeoutContext`,
  with the name reserved for future divergence.
- **`WaitGate`** — a wrapper around `threading.Event` that adds
  message and timeout metadata.

Plus one context manager:

- **`WaitingScope`** — clears a list of gates on entry, waits
  for them on exit.

And one function in
[`waitfunction.py`](https://github.com/code-craft-ai-company/codecraftai-foundations/blob/main/source/packages/ccai/waiting/waitfunction.py):

- **`wait_for_it(looper, *args, ..., wctx=None)`** — the
  public entry point.

This is the entire shipped API.  Five concepts, one function.
The smallness is intentional; chapter 1's *if-it's-hard-to-
explain-try-another-way* principle would have rejected a
larger interface.

## `TimeoutContext` — the state object

The
[`TimeoutContext`](https://github.com/code-craft-ai-company/codecraftai-foundations/blob/main/source/packages/ccai/waiting/waitmodel.py)
is the chapter's substantive contribution.  It tracks a single
wait operation and exposes the operation's progress to both the
caller and the looper function.

The constructor takes four parameters:

```python
def __init__(self, timeout: float, interval: float=0, delay: float=0,
             doevery: int=-1, what_for: Optional[str]=None):
    self._timeout = timeout
    self._interval = interval
    self._delay = delay
    self._doevery = doevery
    self._what_for = what_for

    self._loop_counter = 0
    self._now_time = None
    self._start_time = None
    self._end_time = None
    self._final_attempt = False
    self._wait_state = TimeoutState.NotStarted
    return
```

Five timing fields plus internal state.  Walk each one.

### `timeout` — the deadline budget

The maximum time in seconds the operation may run.  After the
context's `mark_begin()` call, the deadline is fixed at
`start_time + timedelta(seconds=timeout)`.

### `interval` — the gap between checks

The seconds to sleep between consecutive looper invocations.
Default zero (back-to-back checks).  Typical value 1–5 seconds
for most operations.

### `delay` — the warm-up

The initial seconds to sleep *before* the first looper invocation.
Default zero (start immediately).  Useful when the system being
waited on needs a moment to settle (a service that just started,
a database that's still doing initial loads).

### `doevery` — the heartbeat interval

The loop-counter interval at which the caller wants a "still
waiting" event.  Default `-1` (no heartbeat).  When set to a
positive integer N, the property `is_do_every_interval` returns
`True` on every N-th loop iteration; the looper can use this to
log a heartbeat without spamming the log on every check:

```python
def looper(wctx: TimeoutContext) -> bool:
    if wctx.is_do_every_interval:
        logger.info("Still waiting for service to be healthy...")

    return service.is_healthy()
```

This addresses the "logging on every loop is too noisy" failure
mode of the naive `wait_for`.

### `what_for` — the diagnostic string

The string description of *what is being waited for*.  Carried
into any timeout error the context produces, so the failure
report names the wait's purpose without the caller having to
add it.

```python
wctx = TimeoutContext(
    timeout=60,
    interval=2,
    what_for="staging-db to accept connections",
)
```

A timeout from this context produces:

```
Timeout waiting for staging-db to accept connections:
    timeout=60 start_time=2026-04-30 14:32:11 end_time=2026-04-30 14:33:11 now_time=2026-04-30 14:33:13 time_diff=0:01:02
```

The error message is *self-documenting*.  No log archaeology
required.

### State tracking

Internal fields track the operation's state through five
states (`TimeoutState` enum from
[`constants.py`](https://github.com/code-craft-ai-company/codecraftai-foundations/blob/main/source/packages/ccai/waiting/constants.py)):

- `NotStarted` — the context exists but `mark_begin()` hasn't been
  called.
- `Tracking` — reserved for future use.
- `Running` — the wait is in progress.
- `Completed` — the looper returned `True`.
- `TimedOut` — the deadline was reached.

The state machine is internal but observable through the
`completed`, `has_timed_out`, and `wait_state` properties.

## `wait_for_it` — the function

The public entry point in
[`waitfunction.py`](https://github.com/code-craft-ai-company/codecraftai-foundations/blob/main/source/packages/ccai/waiting/waitfunction.py)
is *one function*.  Annotated with the chapter's discipline:

```python
def wait_for_it(looper: WaitCallback, *largs, what_for: Optional[str]=None,
                delay: float=DEFAULT_WAIT_DELAY,
                interval: float=DEFAULT_WAIT_INTERVAL,
                timeout: float=DEFAULT_WAIT_TIMEOUT,
                lkwargs: Optional[Dict[Any, Any]]=None,
                wctx: Optional[WaitContext]=None):
    if lkwargs is None:
        lkwargs = {}

    if what_for is None:
        what_for = "'{}'".format(looper.__name__)

    if wctx is None:
        wctx = WaitContext(timeout, interval=interval, delay=delay, what_for=what_for)

    if wctx.delay > 0:
        time.sleep(DEFAULT_WAIT_DELAY)

    wctx.mark_begin()

    condition_met = False

    while True:
        condition_met = looper(wctx, *largs, **lkwargs)
        if condition_met:
            wctx.mark_complete()
            break

        if not wctx.should_continue():
            break

        if wctx.interval > 0:
            time.sleep(wctx.interval)

    if not condition_met:
        # Mark the time we are performing the final attempt
        wctx.mark_time()
        wctx.mark_final_attempt()
        condition_met = looper(wctx, *largs, **lkwargs)

    if not condition_met:
        toerr = wctx.create_timeout(what_for)
        raise toerr

    wctx.mark_complete()

    return
```

Five things worth naming.

### Defaults are package-level constants

`DEFAULT_WAIT_DELAY = 0`, `DEFAULT_WAIT_INTERVAL = 5`, and
`DEFAULT_WAIT_TIMEOUT = 60` are defined in
[`constants.py`](https://github.com/code-craft-ai-company/codecraftai-foundations/blob/main/source/packages/ccai/waiting/constants.py).
Keeping the defaults as named constants makes them
greppable across the codebase and changeable in one place.

### `what_for` defaults to the looper's `__name__`

When the caller doesn't provide a `what_for`, the function uses
the looper function's name.  This is the *graceful fallback*
discipline — when context isn't provided, infer something
reasonable rather than leave the failure report empty.

### The looper's first invocation precedes `should_continue`

The function's loop runs the looper *first*, then checks whether
to continue.  This means: the first iteration always runs, even
if `timeout=0` was passed.  A caller who genuinely wants
"check once and timeout immediately" gets the check.

### The "final attempt" pattern

This is the chapter's most subtle correctness contribution.  When
the loop's main pass exhausts the timeout, the function does
*one more invocation* of the looper with `wctx.final_attempt =
True`.  The looper can detect the final attempt and produce a
*final diagnostic snapshot* — log the system state, dump
relevant data, capture a screenshot — even though the wait is
about to fail.

Without this pattern, the timeout error fires at the end of the
last sleep, with no information about *what state the system was
actually in* at the moment of failure.  With the pattern, the
final invocation is a deliberate "last chance to capture the
failure context" call.

```python
def wait_for_service_healthy(wctx: TimeoutContext, service) -> bool:
    is_healthy = service.is_healthy()
    if not is_healthy and wctx.final_attempt:
        logger.error("Final state: %s", service.dump_diagnostic_state())
    return is_healthy
```

The looper logs the diagnostic state *only* on the final
attempt — every intermediate failure is silent (correct for
"still waiting"), and the final failure is loud (correct for
"timed out, here's why").

### `create_timeout` produces a rich `TimeoutError`

When the function gives up, it asks the context to construct
the error:

```python
toerr = wctx.create_timeout(what_for)
raise toerr
```

The `create_timeout` method (in `TimeoutContext`) builds the
diagnostic message from the timing fields plus the `what_for`
description.  The chapter-3 timeout-error vocabulary
(`CompletionTimeout`, `ConnectionTimeout`, `InActivityTimeout`)
applies — callers who want a *specific* timeout type wrap the
call:

```python
try:
    wait_for_it(check_db_ready, timeout=60, what_for="staging-db ready")
except TimeoutError as err:
    raise ConnectionTimeout(str(err)) from err
```

The `TimeoutError` from the package becomes a `ConnectionTimeout`
at the wrapping layer.  The original message is preserved; the
type changes for the caller's discrimination.

## `WaitGate` and `WaitingScope` — the threading primitives

Two additional types address the *cross-thread waiting* case.

### `WaitGate` — a threading.Event with metadata

[`WaitGate`](https://github.com/code-craft-ai-company/codecraftai-foundations/blob/main/source/packages/ccai/waiting/waitmodel.py)
wraps a `threading.Event` and adds three pieces of metadata:

```python
class WaitGate:
    def __init__(self, gate: threading.Event,
                 message: Optional[str]=None,
                 timeout: Optional[float]=None,
                 timeout_args: Optional[list]=None):
        self._gate = gate
        self._message = message
        self._timeout = timeout
        self._timeout_args = timeout_args
```

The `message` is the timeout diagnostic the gate carries; the
`timeout` is the default wait duration; `timeout_args` are
formatting arguments for the message.

The gate exposes the underlying event's `set()`, `clear()`,
`is_set()`, and `wait()` methods, plus the `raise_timeout` flag
that converts a wait failure into a `TimeoutError` directly.

### `WaitingScope` — coordinated multi-gate waits

```python
class WaitingScope:
    def __init__(self, gates: List[WaitGate]):
        self._gates = gates

    def __enter__(self):
        for gate in self._gates:
            gate.clear()

    def __exit__(self, ex_type, ex_inst, ex_tb):
        return

    def wait(self):
        for gate in self._gates:
            gate.wait()
```

The pattern: enter the scope to *clear* all gates, do work
that's expected to set them, then call `wait()` to *wait for
all of them*.  This is the cross-thread version of `wait_for_it`
— the looper polls a single condition; the scope waits on
multiple events that are set by other threads.

A typical usage:

```python
service_ready = WaitGate(threading.Event(), timeout=30, message="service ready")
config_loaded = WaitGate(threading.Event(), timeout=10, message="config loaded")

with WaitingScope([service_ready, config_loaded]):
    start_service()      # this thread sets service_ready
    load_config()        # this thread sets config_loaded

# Outside the scope, both gates are set
```

The `WaitingScope` is a small abstraction.  Most callers don't
need it.  When the orchestration is genuinely multi-threaded
and the synchronisation is *gate-based*, the scope is the
right primitive.

## Three patterns the package enables

Three usage patterns recur across the portfolio.

### Pattern 1: simple wait for a condition

```python
from ccai.waiting.waitfunction import wait_for_it

def is_db_accepting_connections(wctx, host, port):
    try:
        with socket.create_connection((host, port), timeout=2):
            return True
    except (ConnectionRefusedError, socket.timeout):
        return False

wait_for_it(
    is_db_accepting_connections,
    "staging-db.internal", 5432,
    what_for="staging-db accepting connections",
    timeout=60,
    interval=2,
)
```

Six lines.  The looper takes the wait context plus any
positional args and returns True/False.  The function handles
the loop, the timeout, the diagnostic.

### Pattern 2: wait with heartbeat logging

```python
def is_long_job_done(wctx, job):
    if wctx.is_do_every_interval:
        logger.info("Job %s still running (%d/%d sec elapsed)",
                    job.id, wctx.timeout - (wctx.end_time - datetime.now()).seconds,
                    wctx.timeout)
    return job.is_done()

wait_for_it(
    is_long_job_done,
    job=long_running_job,
    timeout=3600,
    interval=10,
)
```

The looper checks `is_do_every_interval` to throttle log output.
With `interval=10`, the heartbeat fires every 60 seconds (every
sixth check); without this pattern, every check would log,
producing noise.

To set up `doevery=6`, the caller passes a custom
`TimeoutContext`:

```python
wctx = TimeoutContext(timeout=3600, interval=10, doevery=6, what_for="long job")
wait_for_it(is_long_job_done, wctx=wctx, lkwargs={"job": long_running_job})
```

### Pattern 3: wait with final-attempt diagnostics

```python
def is_test_environment_healthy(wctx, env):
    healthy = env.health_check()
    if not healthy and wctx.final_attempt:
        # Last chance — capture full diagnostic for the failure
        env.dump_state(path="./failure-diag.txt")
        env.capture_logs(path="./failure-logs/")
    return healthy

wait_for_it(
    is_test_environment_healthy,
    env=staging_environment,
    what_for="staging environment healthy",
    timeout=300,
)
```

The diagnostic dump runs *only* on the final attempt — once,
right at the moment of failure, with the full system state
available.  Without this pattern, the engineer debugging the
timeout has to *re-run* the test to capture the state, which
often produces a successful run because the flake doesn't
reproduce.

## What the package does *not* do

Three deliberate non-features.

### Not async/await

The shipped `wait_for_it` is a synchronous function.  It uses
`time.sleep` between checks and blocks the calling thread.  When
the calling code is `async`, a different primitive is needed —
typically `asyncio.wait_for` or a custom looper — and the
shipped package doesn't provide one.

This is a deliberate scoping choice.  An async equivalent could
be added; the discipline is to wait until two callers actually
need it before adding the parallel implementation.  Adding it
preemptively would double the surface area for hypothetical
benefit.

### Not retries with backoff

The `interval` parameter is constant.  When the caller wants
exponential backoff (interval doubles after each failed check),
the looper itself implements it — typically by tracking attempts
in a closure or in `wctx`'s loop counter.

The shipped package doesn't bake in a backoff curve because no
single curve fits every use case.  Linear, exponential, capped-
exponential, jittered — each is a few lines of looper code.  The
discipline keeps the chapter-2 *no-clever-defaults* rule.

### Not progress estimation

The `wait_for_it` function knows when the deadline is, but it
doesn't know the *progress* of the underlying operation.  When
the caller wants "60% done" rather than "44 of 60 seconds
elapsed," that's a property of the operation, not of the wait.

The package leaves progress estimation to the operation — the
looper can call into the operation and compute progress, then
include it in the heartbeat log.  The wait function stays
neutral about progress; it only knows about time.

## Cross-references

The waiting primitives are used across every higher tier of the
portfolio:

- **Volume 0 chapter 9 (Configuration loading)** uses
  `wait_for_it` for the HTTP source's connection retries.
- **Volume 3** uses the package implicitly through tasks that
  block on subprocess completion; the chapter-19 `RunCommand`
  task wraps `subprocess.communicate()` which is its own wait
  primitive.
- **Volume 5 (Interop)** uses `wait_for_it` extensively for SSH
  session establishment, DNS lookups, and similar
  network-bound waits.
- **Volume 6 (Landscape)** uses `wait_for_it` for "wait for the
  device to come online" patterns during landscape activation.

The `Aspects` pattern from Volume 5 chapter 1 is the affirmative
form of *behaviour parameterisation* the chapter-2 decorator
discipline pointed at.  Aspects' `timeout`, `interval`, and
`retry_count` fields are exactly the fields the wait function
expects — the two layers compose cleanly.

## Summary

- **`ccai.waiting`** is a small subpackage of `codecraftai-foundations`
  with one job: give
  the rest of the portfolio a correct, observable, timeout-
  aware *wait for a condition* primitive.
- The shipped API is **five concepts and one function**:
  `TimeoutContext`, `WaitCallback`, `WaitContext`, `WaitGate`,
  `WaitingScope`, plus `wait_for_it`.
- `TimeoutContext` carries five timing fields (`timeout`,
  `interval`, `delay`, `doevery`, `what_for`) plus internal
  state.  The `what_for` field is what makes timeout errors
  self-documenting.
- The **final-attempt pattern** is the chapter's most subtle
  correctness contribution.  After the timeout exhausts, one
  more looper invocation with `wctx.final_attempt = True` lets
  the looper capture diagnostic state at the moment of failure.
- `wait_for_it` is the public entry point.  Defaults to
  `DEFAULT_WAIT_TIMEOUT=60`, `DEFAULT_WAIT_INTERVAL=5`,
  `DEFAULT_WAIT_DELAY=0`.  Returns nothing on success; raises
  `TimeoutError` with rich context on failure.
- `WaitGate` and `WaitingScope` provide the cross-thread
  variant — gates are `threading.Event` wrappers with timeout
  metadata; the scope clears gates on entry and waits for them
  on exit.
- Three patterns recur: **simple condition wait**, **wait with
  heartbeat logging** (via `doevery` + `is_do_every_interval`),
  **wait with final-attempt diagnostics** (via
  `wctx.final_attempt`).
- The package is *not* async/await aware, *not* a retry-with-
  backoff library, and *not* a progress estimator.  Each
  scoping choice is deliberate.
- Higher tiers (orchestration, interop, landscape, sysman) all
  use the wait function for time-bounded operations.

The next chapter introduces application identity and presence —
the `ThisApplication` machinery that gives every running process
in the portfolio a structured identity, with the persona-variable
descriptors that make configuration overrides traceable.
