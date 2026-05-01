# Chapter 2 — Pod Instrumentation

> **Abstract.** The pod's contract with the application is a
> pair of context managers — `TimeTravelGateway` at the top and
> `TimeTravelPortal("name")` at every interesting scope — and a
> process-wide registry that decides which portal names are
> *active* at any moment.  This chapter walks the entry points
> the application sees, the state machine the pod runs through
> when a portal opens, the configuration object that controls
> recording fidelity, and the static-vs-dynamic instrumentation
> trade-off the design forces on the operator.  All claims are
> anchored on `automojo-timetravel-pod`.

## 2.1 The two context managers

The application surface of the pod is two classes.  Everything
else — the trace hook, the recorder chain, the serialiser, the
backend — is reached through them.

`TimeTravelGateway`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/gateway.py:22`)
is the application-scope context manager.  It is entered exactly
once, near the top of the application's lifetime; it owns the
monotonic clock that all timestamps are measured against, holds
the registry of currently-active recorders, and broadcasts the
two boundary events that bracket every recording — a
`GatewayStartEvent` on entry, a `GatewayStopEvent` on exit.

`TimeTravelPortal`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/portal.py:28`)
is the per-scope context manager.  It is entered every time
control flows into a portal scope, regardless of whether that
portal is currently *activated*.  The cost of entering an
inactive portal is three early-exit conditional branches and
one thread-local lookup; the cost of entering an active portal
is the installation of `sys.settrace`, the construction of a
recorder chain, and an event broadcast.

The asymmetry is deliberate.  Portal entry is on the hot path
of the application — every request, every scheduled task, every
agent step might cross a portal boundary.  Gateway entry is
once per process.  The framework optimises the common, hot,
cheap case (portal entry while inactive) and accepts higher
cost on the rare, cold, expensive case (gateway start, portal
activation).

A minimal usage looks like this:

```python
from automojo.timetravel.pod import TimeTravelGateway, TimeTravelPortal

def main() -> None:
    with TimeTravelGateway(backend_uri="file:///var/log/mysvc/trace.bin"):
        run_service()

def handle_request(req):
    with TimeTravelPortal("http.request"):
        return process(req)
```

The gateway is not a daemon; it is a `with` block that *brackets
the recording window*.  In a long-lived service the gateway is
typically entered in `main()` and never exited until the process
terminates.  In a CLI command, the gateway brackets the
command's body.  In a unit test, the gateway brackets the test
body.

## 2.2 The startup configuration

`TimeTravelConfig`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/configuration.py:19`)
collects every knob the pod exposes into a single object.  Its
13 `__slots__` fields cover four concerns:

- **What to record.**  `active_portals` (a list of names to
  activate at startup), `trace_calls`, `trace_lines`,
  `trace_variables`, `record_inner_portals` (the four trace-
  granularity booleans).
- **Where to record it.**  `backend_uri` (a `file://`, `tcp://`,
  or `unix://` URI; resolved by the backend registry).
- **How to be remote-controlled.**  `remote_enabled`,
  `remote_host`, `remote_port`, `remote_credential` (for the
  JSON-lines control server).
- **How to be agent-controlled.**  `mcp_enabled`,
  `mcp_transport`, `mcp_host`, `mcp_port`, `mcp_pipe` (for the
  Model Context Protocol bridge — see ch. 5).

The bootstrap function `configure_timetravel`
(`configuration.py:259-304`) reads a `TimeTravelConfig` and
produces the runtime side-effects:

```python
def configure_timetravel(config: TimeTravelConfig) -> dict:
    servers = {}

    # Activate portals
    registry = get_portal_registry()
    if config.active_portals:
        registry.activate_portals(config.active_portals)

    # Create backend (if URI specified)
    if config.backend_uri:
        backend_registry = create_default_registry()
        backend_registry.create_backend(config.backend_uri)

    # Start remote control server (if enabled)
    if config.remote_enabled:
        remote_server = RemoteControlServer(
            host=config.remote_host,
            port=config.remote_port,
            credential=config.remote_credential
        )
        remote_server.start()
        servers["remote"] = remote_server

    # Start MCP server (if enabled)
    if config.mcp_enabled:
        mcp_server = McpServer()
        if config.mcp_transport == "tcp":
            mcp_server.start_background_tcp(config.mcp_host, config.mcp_port)
        elif config.mcp_transport == "pipe" and config.mcp_pipe:
            mcp_server.start_background_pipe(config.mcp_pipe)
        else:
            mcp_server.start_background_stdio()
        servers["mcp"] = mcp_server

    return servers
```

The function returns a dict of running servers keyed by
`"remote"` and `"mcp"`, so the application can keep references
for shutdown.  Calling `configure_timetravel` does *not* enter
the gateway; the gateway is still entered by the application
at the appropriate scope.  The function is purely about
bootstrapping the side-channels: the portal registry's initial
state, the backend, the control plane.

The choice to keep the gateway out of `configure_timetravel` is
deliberate and worth understanding.  The gateway controls the
recording-window timestamp origin
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/traceclock.py`).
A long-running service that wants the gateway alive for the
process lifetime enters it as a `with` block in `main()`.  A
short-lived task that wants the gateway alive for the task's
extent enters it inside the task's body.  The application
decides; `configure_timetravel` does not enforce.

## 2.3 The gateway's invariants

`TimeTravelGateway.__enter__` does three things, in order
(`gateway.py`):

1. Records the current wall-clock time and starts the monotonic
   clock — so every event timestamp is an *offset from gateway
   start*.
2. Sets the process-wide `_gateway_active` flag to `True`
   (`tracingstate.py:71-79`); this is the flag every portal
   reads first to decide whether to do anything.
3. Emits a `GatewayStartEvent` to all registered recorders.

`TimeTravelGateway.__exit__` (lines 73-101) is more intricate
because it has to land safely even if a recording in flight has
errored:

```python
def __exit__(self, exc_type, exc_val, exc_tb):
    try:
        # Emit gateway stop event
        stop_event = GatewayStopEvent(
            timestamp_ns=self._clock.timestamp(),
            thread_id=threading.get_ident(),
            wall_clock_time=time.time(),
        )
        self._broadcast_event(stop_event)

        # Flush and close all active recorders
        for recorder in self._active_recorders:
            recorder.flush()
            recorder.close()
        self._active_recorders.clear()
    finally:
        self._clock.stop()
        set_gateway_active(False)
        set_active_gateway(None)
    return False
```

Three observations.

First, the `try` body emits the stop event *before* flushing —
so the stop boundary appears in the recorded stream even if a
flush fails afterwards.  This is the same boundary-marker
discipline JSOS uses (Volume 1 ch. 5): the marker is broadcast,
then the buffered data is committed.

Second, the `finally` block always runs, regardless of any
exception in the `try`.  The clock is stopped, the gateway flag
is cleared, the active-gateway singleton is unset.  Even if a
recorder's `flush()` or `close()` raises, the next gateway
attempt on the same process will not see stale state.

Third, the `__exit__` returns `False` — meaning any exception
that propagated *into* `__exit__` (an error in the `with` body,
or in the recorder cleanup) is *not* suppressed.  The pod is not
in the business of swallowing the application's exceptions.
This is the same convention every interop layer in the
portfolio honours (Volume 5: control reverts via typed
exceptions).

## 2.4 The portal's decision tree

`TimeTravelPortal.__enter__`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/portal.py:60-141`)
is the heart of the pod.  It is run on every entry into every
portal scope; its cost dominates the application's *quiescent*
overhead.  The decision tree it implements is, in order:

```
1. Is the gateway active?            → no  → return self
2. Is this thread already tracing?   → yes → bump depth, return self
3. Is this portal's name activated?  → no  → return self
4. Does the activation have an
   expression?                       → if so, eval it; if false,
                                       return self
5. Construct recorder chain.
6. Install sys.settrace hook.
7. Mark thread as tracing,
   depth = 1.
```

Steps 1-4 are guards; steps 5-7 are activations.  In the typical
production case — most portals not currently active — only steps
1 and 3 actually evaluate, and both fall through to `return
self`.

The activation block (5-7) is more substantive.  Two flavours
exist depending on whether a backend URI is configured:

```python
def _create_record_callback(self, gateway, capture_id):
    backend_uri = capture_id or gateway.backend_uri

    if backend_uri:
        # Real recorder chain
        backend_registry = create_default_registry()
        backend = backend_registry.create_backend(backend_uri)
        serializer = TraceSerializer()
        writer = BufferedTraceWriter(backend, max_buffer_size=65536)

        class PortalRecorder(TraceRecorder):
            def record(self, event):
                data = self.serializer.serialize(event)
                self.writer.write(data)
            def flush(self):
                self.writer.flush()
            def close(self):
                self.writer.close()

        recorder = PortalRecorder(writer, serializer)
        gateway.register_recorder(recorder)
        ...
```

The chain is **backend → serializer → buffered writer →
recorder**.  Each layer has a single responsibility; each is
swappable.  The 65,536-byte buffer (auto-flush at full) is the
default chosen at this site; a different recording profile could
override it without changing the recorder ABC.

The fall-through case — no backend URI configured — is a list
collector:

```python
else:
    # Fallback: in-memory collector for testing
    collected_events = []
    def record_callback(event):
        collected_events.append(event)
    state.recorder = _ListRecorder(collected_events)
```

This second branch is what unit tests use.  A test enters a
portal, runs code, exits the portal, and inspects the in-memory
event list.  Production deployments configure a backend URI.

## 2.5 The portal registry

`PortalRegistry`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/portalregistry.py:12`)
is the structure that holds *which named portals are active
right now*.  It is a singleton (`get_portal_registry()` at line
133) keyed by portal name, with each value a dict of `active`,
`expression`, `once`, and `capture_id`.

Every public method takes a `threading.Lock` before reading or
writing the dictionary
(`portalregistry.py:25-41`):

```python
def activate_portal(self, name: str, expression: str = None,
                    once: bool = False, capture_id: str = None) -> None:
    with self._lock:
        self._portals[name] = {
            "active": True,
            "expression": expression,
            "once": once,
            "capture_id": capture_id,
        }
    return
```

The `get_portal_info` method
(`portalregistry.py:71-83`) explicitly returns a *copy* of the
internal dict so a caller cannot mutate registry state by
mutating the returned object:

```python
def get_portal_info(self, name: str) -> dict:
    with self._lock:
        rtnval = self._portals.get(name, {})
        # Make a copy to avoid exposing the internal structure
        rtnval = dict(rtnval)
    return rtnval
```

This is the kind of small discipline that prevents action-at-a-
distance bugs three years later — the operator-tool client
queries portal info, sets `active=False` on the returned dict
by accident, and the registry is silently corrupted.  The copy
defends against that.

The lock is uncontended in the common case: it is taken on
every portal-entry's `is_portal_active` check, but the check is
microseconds long.  The lock is contended when remote-control
commands flood the registry — a debug session that bulk-
activates 50 portals at once will serialise through this lock —
but those events are infrequent compared to portal entries.

## 2.6 The thread-local state

The decision tree in §2.4 does not actually take the registry's
lock at step 2 ("is this thread already tracing?").  Instead it
reads thread-local state.  The
`TracingState`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/tracingstate.py:15`)
holds four fields per thread:

- `is_tracing` — bool, true while a portal is open and the
  trace hook is installed.
- `recorder` — the active recorder, used by the hook callback to
  publish events.
- `portal_depth` — int, the current portal nesting level on this
  thread.
- `active_portal_name` — str, the name of the outermost
  currently-active portal.

The lookup helper (`tracingstate.py:44-55`) lazy-initialises:

```python
_thread_local = threading.local()

def get_tracing_state() -> TracingState:
    state = getattr(_thread_local, "tracing_state", None)
    if state is None:
        state = TracingState()
        _thread_local.tracing_state = state
    return state
```

There is no global lock here — `threading.local()` is the
standard library's per-thread storage primitive, and the lazy
initialisation pattern is idempotent on a per-thread basis.

The reason the design *splits* into a thread-local state and a
process-wide registry is the classic *what changes vs what is
read* axis:

- The registry is *read* from many threads on every portal
  entry, but *written* rarely (only when an activation command
  arrives).  A `Lock` is the right tool: it protects the
  occasional write against the frequent reads.
- The state is *read* and *written* on every portal entry on a
  particular thread, but *only that thread*.  A `threading.local`
  is the right tool: no lock needed, because no other thread is
  reading or writing the same fields.

## 2.7 Nested portals

When a thread already inside an active portal enters a second
portal, the second portal does *not* install a second
`sys.settrace`.  It increments `state.portal_depth` and (if
`record_inner_portals` is set in the trace configuration) emits
a `PortalEnterEvent` so the trace contains the boundary marker.

```python
if state.is_tracing:
    state.portal_depth += 1
    if config.record_inner_portals:
        record(PortalEnterEvent(timestamp=clock.timestamp(),
                                portal_name=self._name))
    return self
```

The exit path mirrors the entry: `state.portal_depth -= 1`, and
if the `record_inner_portals` flag is set, a `PortalExitEvent`.
The hook is uninstalled only when `state.portal_depth` drops to
zero (i.e. the outermost portal exits).

This *single-hook-many-portals* design has two consequences.

The first is correctness.  `sys.settrace` is a per-thread
setting — Python only allows one trace function at a time per
thread.  If portals stacked their hooks, the inner hook would
*replace* the outer hook, the outer hook's events would stop
being recorded for the inner scope, and reinstalling the outer
hook on inner-portal exit would either work (only because
Python's `sys.settrace` semantics happen to support this) or
produce gaps.  The single-hook approach sidesteps the issue
entirely.

The second is annotation.  When the off-line tooling
reconstructs the timeline, the `PortalEnterEvent` /
`PortalExitEvent` markers are the boundaries it uses to *filter
the recording to a portal scope*.  Volume 8 ch. 4 walks
`tracedata/portal_filter.py` (server-side) and `pod/portalfilter.py`
(pod-side; same algorithm) — both are tiny state machines that
turn the markers into scope membership.

## 2.8 Conditional activation

Activation can carry an `expression` string:

```python
registry.activate_portal(
    name="invoice.charge.attempt",
    expression="invoice.amount > 10000",
    once=True,
    capture_id="big-invoice-anomaly",
)
```

When a portal entry encounters this activation, it evaluates
the expression in the *caller's frame*
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/portal.py:99-112`):

```python
expression = portal_info.get("expression")

if expression is not None:
    try:
        frame = sys._getframe(1)
        result = eval(expression, frame.f_globals, frame.f_locals)
        if not result:
            return self
    except Exception:
        return self
```

If the expression evaluates falsy, the portal returns without
activating tracing — the recording is reserved for the calls
that *match the predicate*.  If evaluating the expression
raises (a `NameError` because `invoice` was not in scope, a
`TypeError` because `invoice.amount` was None), the exception
is swallowed and the portal does not activate.  This last
behaviour is a quiet override of the portfolio's "errors should
never pass silently" rule, justified by the use case: *a
malformed predicate must not crash the production application
that is using the recording as a debugging tool.*  The cost is
that operators submitting bad predicates see no activation and
no error feedback — the feedback path is logging.

This is a tractable design point worth pushing on.  Two options
were available:

- **Swallow.**  As implemented.  The portal silently does not
  activate.  Pros: no production risk.  Cons: silent failure;
  the operator does not learn that their predicate is wrong.
- **Surface.**  Raise to the operator-tool client that submitted
  the activation.  Pros: visible error.  Cons: requires a
  channel for asynchronous notification of failed activations,
  because the predicate is evaluated *later*, not at activation
  time.

The implemented choice is the conservative one.  Volume 8 notes
the trade-off; the source comment is honest about the swallow
(`portal.py:108-112`).

## 2.9 The trace configuration

`TraceConfiguration`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/traceconfiguration.py:10-63`)
is an immutable, four-boolean object that controls *what* the
trace hook records on every Python event:

```python
class TraceConfiguration:
    __slots__ = ("_trace_calls", "_trace_lines", "_trace_variables", "_record_inner_portals")

    def __init__(
        self,
        trace_calls: bool = True,
        trace_lines: bool = False,
        trace_variables: bool = False,
        record_inner_portals: bool = False,
    ) -> None:
        self._trace_calls = trace_calls
        self._trace_lines = trace_lines
        self._trace_variables = trace_variables
        self._record_inner_portals = record_inner_portals
```

The `__slots__` plus read-only properties enforce immutability
— there is no setter for `_trace_calls` etc.  The configuration
is a *value* in the strict sense: two configurations with the
same booleans are interchangeable.

The four knobs cover four growing recording densities:

- **calls only** (`trace_calls=True`, others false): a
  `CallEvent` for each function entered, a `ReturnEvent` for
  each function exited.  A function-graph trace.  Cheap.
- **calls + lines** (`trace_lines=True` added): on every line
  executed inside a recorded function, a `LineEvent`.  About 3-
  10x the volume of calls-only, depending on the code's loop
  density.
- **calls + lines + variables** (`trace_variables=True` added):
  on every line executed, a `VariableSnapshotEvent` carrying the
  current `frame.f_locals` as a `dict`.  An order of magnitude
  more volume than calls + lines.
- **+ inner portals** (`record_inner_portals=True` added):
  `PortalEnterEvent` and `PortalExitEvent` markers for nested
  portals.  Negligible additional volume, but enables off-line
  filtering by portal scope.

Volume 8 ch. 8 measures the cost.  The numbers, briefly:
calls-only is a few-percent overhead on a typical Python
service.  Lines is 5-10x.  Variables is 50-100x and a flood of
data.  The design intent is that `calls` is the production
default, `lines` is the targeted-window default, and `variables`
is the *last-resort, narrow-scope, this-bug-now* default.

## 2.10 The event model

`TraceEvent`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/traceevents.py:29`)
is the base class of nine concrete event types
(`traceevents.py:12-20`):

```python
class EventType:
    CALL = 1
    RETURN = 2
    LINE = 3
    VARIABLE_SNAPSHOT = 4
    PORTAL_ENTER = 5
    PORTAL_EXIT = 6
    GATEWAY_START = 7
    GATEWAY_STOP = 8
    EXCEPTION = 9
```

Each event uses `__slots__` for memory efficiency, has a
nanosecond-precision `timestamp_ns` field (an integer offset
from gateway start), a `thread_id`, and a per-type payload.

A representative event — the variable snapshot — looks like:

```python
class VariableSnapshotEvent(TraceEvent):
    __slots__ = ("filename", "lineno", "local_variables")

    def __init__(self, timestamp_ns: int, thread_id: int,
                 filename: str, lineno: int, local_variables: dict) -> None:
        super().__init__(EventType.VARIABLE_SNAPSHOT, timestamp_ns, thread_id)
        self.filename = filename
        self.lineno = lineno
        self.local_variables = local_variables
```

The `local_variables` field is a literal `dict` snapshot of
`frame.f_locals` at the point of the event.  This is the most
expensive event in the model — it serialises an arbitrary dict,
including whatever the application happened to have on the
stack.  It is also the most informative, because it is the
*answer* to the on-call engineer's "what was in scope when this
fired?" question.

Boundary events — `GatewayStartEvent`, `GatewayStopEvent` — also
carry a `wall_clock_time` (float, seconds since epoch) so the
recording can be correlated with external systems
(application logs, APM traces, alerting timestamps).  All other
events are nanosecond offsets only.

## 2.11 The trace hook

The actual `sys.settrace` callback lives in
`TraceHookManager._trace_function`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/tracehookmanager.py:63-137`).
It dispatches on the Python event type (the second argument
Python passes to a trace function), constructs the appropriate
`TraceEvent` subclass, and pushes it through the `record_callback`:

```python
def _trace_function(self, frame, event, arg):
    timestamp_ns = self._clock.timestamp()
    record = self._record_callback

    if event == "call":
        if self._trace_calls:
            call_event = CallEvent(
                timestamp_ns=timestamp_ns,
                thread_id=self._thread_id,
                filename=frame.f_code.co_filename,
                lineno=frame.f_lineno,
                function_name=frame.f_code.co_name,
            )
            record(call_event)
        return self._trace_function

    elif event == "line":
        if self._trace_lines:
            line_event = LineEvent(...)
            record(line_event)
        if self._trace_variables:
            var_event = VariableSnapshotEvent(
                timestamp_ns=timestamp_ns,
                thread_id=self._thread_id,
                filename=frame.f_code.co_filename,
                lineno=frame.f_lineno,
                local_variables=dict(frame.f_locals),
            )
            record(var_event)

    return self._trace_function
```

Two performance details are worth the attention.

First, `self._trace_calls`, `self._trace_lines`,
`self._trace_variables` are cached as instance attributes
(`tracehookmanager.py:42-44`) instead of being read out of the
`TraceConfiguration` on every event.  Attribute lookup is faster
than property access; on a hot trace, the difference matters.

Second, the trace function returns *itself* on every event.
Python's `sys.settrace` semantics require this — the returned
function is what gets installed as the *line* trace function for
the frame that just received a *call* event.  Returning `None`
turns line tracing off for that frame; returning `self._trace_function`
keeps the hook installed.  The pod always returns
`self._trace_function`, ensuring the recording continues across
nested calls.

The hook does *not* catch exceptions raised in `record(...)`.
If the recorder fails — a backend disconnect, a serialiser bug,
an OOM — the exception propagates back up through the trace
hook and out of the recorded code.  This is consistent with
the gateway's `__exit__` returning `False` (§2.3): the pod does
not pretend to handle errors it cannot resolve.

## 2.12 The decorator

For functions whose entire body is the portal scope,
`@trace_portal("name")`
(`automojo-timetravel-pod/source/packages/automojo/timetravel/pod/decorators.py:19-46`)
is the convenience form:

```python
def trace_portal(name: str):
    def decorator(func):
        if inspect.iscoroutinefunction(func):
            @functools.wraps(func)
            async def async_wrapper(*args, **kwargs):
                with TimeTravelPortal(name):
                    rtnval = await func(*args, **kwargs)
                return rtnval
            return async_wrapper
        else:
            @functools.wraps(func)
            def sync_wrapper(*args, **kwargs):
                with TimeTravelPortal(name):
                    rtnval = func(*args, **kwargs)
                return rtnval
            return sync_wrapper

    return decorator
```

The decorator does nothing the explicit `with` form does not do
— it just wraps the function body in a `with TimeTravelPortal(name)`.
The branch on `inspect.iscoroutinefunction` is what allows the
same decorator to work on both sync and async functions; the
async wrapper awaits the coroutine *inside* the portal scope,
so the recording covers the entire async lifetime.

Decorators in the portfolio are constrained by the project's
coding standards.  The rule (in `automojo-*` `CLAUDE.md`):
*"Decorators are only allowed for adding metadata or for
registering a function with a mapping system.  Do NOT use
behavior-modifying decorators."*  `@trace_portal` is on the
edge — it modifies behaviour by wrapping the call in a context
manager.  The justification, taken on its own terms, is that
the wrapped behaviour is *purely additive* and *purely
declarative*: it does not change the function's semantics; it
adds a recording boundary.  Even so, the explicit `with` form
remains the dominant pattern in the portfolio's own code; the
decorator is offered for ergonomics on functions whose body
*is* the portal.

## 2.13 Static vs dynamic instrumentation

The pod's portals are *static*, not dynamic.  They are written
into the application source as `with` blocks or `@trace_portal`
decorators, before the code is built and deployed.  The
operator's runtime control is *which static portals to
activate*, not *where to add new portals*.

The alternative — dynamic instrumentation — would let the
operator say, at runtime, "trace `mymodule.somefunc` as if it
were wrapped in a portal."  Tools in the broader Python
ecosystem do this: `sys.settrace` itself is the underlying
mechanism, and a debugger-like tool that takes a fully-qualified
function path could install a hook at runtime.

The pod chose static instrumentation for three reasons.

First, **scope discipline.**  A static portal expresses an
*intent*: "this scope is one I expect to need to debug."  The
intent is encoded in the source review — a code reviewer can see
that the developer thought `http.request` was a debugging-worthy
boundary.  A dynamic tool with no source-side annotation can
trace anywhere, which makes the activation policy harder to
reason about and harder to audit.

Second, **performance predictability.**  A static portal's cost
is bounded — the inactive cost is three thread-local conditional
branches (§2.4); the active cost is `sys.settrace`'d Python
inside a known scope.  A dynamic tool that activates by qualified
name has to install hooks at function entry, return, or line
boundaries it does not know in advance; the cost surface is
larger.

Third, **operator vocabulary.**  "Activate the
`invoice.charge.attempt` portal" is a sentence the on-call
engineer can type without consulting the source.  "Install a
trace hook on `automojo.invoicing.service.InvoiceService.charge`
at line 412" is not.  The named-portal abstraction trades some
flexibility for a vocabulary the production operator already
shares with the developer.

The trade-off is real and worth naming.  Static instrumentation
*forecloses* the bug whose right scope nobody anticipated.  If
the failing call is in code that has no portal, the operator
cannot record it without a code change, a build, and a deploy —
which can be a long road in production.  The pod accepts that
foreclosure as the price of the three benefits above.

A practical mitigation: the portfolio's own coding-standards
document encourages portals at every architecturally meaningful
boundary — request entry, request exit, RPC call, RPC reply, DB
transaction, agent step, scheduled-task body, message-queue
consumer.  In a fully-instrumented service the *coverage* of
static portals approaches the operator's likely debug surface,
even if the exact line is not directly inside one.

## 2.14 Where the portals belong

Using the portal pattern well is mostly a question of *where to
place portals*.  The portfolio's own code provides several
patterns:

- **At every external entry point.**  Every HTTP handler, every
  message-queue consumer, every CLI subcommand opens a portal
  named for the entry point (`http.request`,
  `kafka.consumer.invoice-events`, `cli.invoke.deploy`).
- **At every external call.**  Every database query, every RPC,
  every third-party API call opens a portal named for the
  destination (`db.query`, `rpc.payment-service.charge`,
  `http.client.stripe`).
- **At every long-running task body.**  Every scheduled task,
  every background worker, every agent step opens a portal
  named for the task type (`task.daily-rollup`,
  `worker.invoice-batch`, `agent.step`).
- **At every "I expect this to be hard to debug" scope.**  The
  scope the senior developer imagined would be the locus of
  pain — the cache-invalidation block, the retry loop, the
  branch that handles the rare schema variant — opens a portal
  named for that intent (`cache.invalidate.user`,
  `retry.flaky-third-party`, `legacy.migration.v1-to-v2`).

A typical service ends up with on the order of 10-50 named
portals.  The number is small because the *named* unit is a
*scope of debug interest*, not a function or a line.  Many
functions live inside a single portal.

The cost of a static portal in inactive state — three
conditional branches and a thread-local lookup — is small
enough that this density is supportable.  Volume 8 ch. 8 quotes
the measured numbers.

## 2.15 Cross-references and the next chapter

What the application surface looks like is now in place.  The
next layer down is *what the recording looks like on the wire*:
the envelope format the serialiser emits, the layout of the
buffered writer's output, and the contract the server is built
to parse on the other end.

The next chapter walks the trace-event protocol — including a
documented mismatch between the pod-side serialiser and the
server-side deserialiser as the source stands today.  That
mismatch is honest engineering signal worth surfacing: the
protocol has evolved, the two ends have not yet been re-
synchronised, and the book does not pretend otherwise.

→ Continue to [`03-trace-event-protocol.md`](03-trace-event-protocol.md).

← Back to [`01-why-time-travel.md`](01-why-time-travel.md).
