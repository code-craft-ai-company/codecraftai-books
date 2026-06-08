# Chapter 2 — The Aspects Pattern — The Delegation Contract

> *Chapter 1 carried the philosophy: persistence behaviours
> live in the interop layer, decisions stay in the application
> layer, the Aspects parameter is the delegation contract.
> This chapter walks the *code*.  `Aspects`, `AspectsCmd`,
> `ActionPattern`, `LoggingPattern`, `Freezable`,
> `MonitoredScope` — the classes and enums that make Measured
> Interoperability work.*

The previous chapter established the four principles of
Measured Interoperability.  This chapter walks the
implementation: the classes and the conventions that turn the
principles into working code.  By the end, a reader should be
able to construct an `AspectsCmd` for any specific call need
and know what each field does.

## The class — `Aspects`

**File:**
`codecraftai-interop/source/packages/ccai/interop/aspects.py:63–130`

```python
class Aspects(Freezable):
    """
        Aspects encapsulates the runtime behaviour parameters that
        govern interop API calls — timeouts, intervals, retry strategy,
        logging mode, and which transient errors to ignore.

        Aspects is immutable after construction (Freezable).
    """

    def __init__(self, *,
                 action_pattern: ActionPattern = ActionPattern.SINGLE_CALL,
                 completion_timeout: float = DEFAULT_COMPLETION_TIMEOUT,
                 completion_interval: float = DEFAULT_COMPLETION_INTERVAL,
                 connection_timeout: float = DEFAULT_CONNECTION_TIMEOUT,
                 connection_interval: float = DEFAULT_CONNECTION_INTERVAL,
                 inactivity_timeout: float = DEFAULT_INACTIVITY_TIMEOUT,
                 inactivity_interval: float = DEFAULT_INACTIVITY_INTERVAL,
                 logging_pattern: LoggingPattern = DEFAULT_LOGGING_PATTERN,
                 allowed_error_codes: Tuple[int, ...] = DEFAULT_ALLOWED_ERROR_CODES,
                 allowed_connection_exceptions: Tuple[BaseException, ...] = DEFAULT_ALLOWED_CONNECTION_EXCEPTIONS,
                 must_connect: bool = DEFAULT_MUST_CONNECT,
                 logger: Optional[logging.Logger] = None) -> None:
        super().__init__()

        self.action_pattern = action_pattern
        self.completion_timeout = completion_timeout
        self.completion_interval = completion_interval
        self.connection_timeout = connection_timeout
        self.connection_interval = connection_interval
        self.inactivity_timeout = inactivity_timeout
        self.inactivity_interval = inactivity_interval
        self.logging_pattern = logging_pattern
        self.allowed_error_codes = tuple(allowed_error_codes)
        self.allowed_connection_exceptions = tuple(allowed_connection_exceptions)
        self.must_connect = must_connect
        self.logger = logger

        self.freeze()
        return
```

Twelve fields.  Each is a behaviour knob the application layer
sets to control how the interop layer applies persistence.

### The fields

The fields divide into three groups by purpose:

**Persistence strategy:**

- `action_pattern` — the `ActionPattern` enum value naming the
  persistence behaviour (below).
- `must_connect` — when True, the call requires a connection;
  when False, connection failure is acceptable for a single-
  call attempt.

**Timing budget:**

- `completion_timeout` — total time before the call gives up
  (default 600 seconds = 10 minutes).
- `completion_interval` — how often to check for completion
  during the wait (default 10 seconds).
- `connection_timeout` — total time to establish the
  connection (default 16 seconds).
- `connection_interval` — how often to retry the connection
  (default 2 seconds).
- `inactivity_timeout` — time without progress before giving
  up (default 600 seconds).
- `inactivity_interval` — granularity of progress detection
  (default 0.5 seconds).

**Tolerance:**

- `allowed_error_codes` — tuple of status codes the call
  accepts as success-equivalent.  A status in this tuple
  doesn't trigger retry.
- `allowed_connection_exceptions` — tuple of exception types
  the call ignores during connection attempts (e.g., a
  `ConnectionRefusedError` from a service that's still
  starting up).

**Logging:**

- `logging_pattern` — the `LoggingPattern` enum value
  controlling when to log (below).
- `logger` — optional explicit logger; otherwise the
  call uses the framework default.

A reader studying the field list might notice that two
distinct timing concepts coexist: *completion* (the overall
"is the work done?" budget) and *inactivity* (the "are we
making progress?" budget).  A long-running command can
exceed the completion timeout while still showing progress;
the inactivity timeout catches the case where the command
hangs without producing any output.  Both serve as escape
hatches for different failure modes.

## `AspectsCmd` — command-shaped subclass

**File:**
`codecraftai-interop/source/packages/ccai/interop/aspects.py:133–197`

```python
class AspectsCmd(Aspects):
    """
        Aspects subclass for command-shaped calls: shell commands,
        SSH commands, serial commands.

        Adds:
        - expected_status: the exit code(s) treated as success.
        - user: the user identity to run the command as.
        - pty_params: pseudo-terminal parameters.
    """

    def __init__(self, *,
                 expected_status: Union[int, Tuple[int, ...]] = 0,
                 user: Optional[str] = None,
                 pty_params: Optional[dict] = None,
                 # ... inherited Aspects fields ...
                 **kwargs) -> None:
        super().__init__(**kwargs)
        # Note: the freeze() in super().__init__ has already happened.
        # AspectsCmd-specific fields must be set before freeze.
        # The actual implementation handles this carefully.
        self.expected_status = expected_status
        self.user = user
        self.pty_params = pty_params
        return
```

Three command-specific additions:

- **`expected_status`** — the status code (or tuple of codes)
  the call treats as success.  Default `0`.  A test that
  expects a non-zero status sets `expected_status=1` and the
  call returns normally when the command exits 1.
- **`user`** — the user identity to run the command as.  For
  SSH calls, this picks one of the credentials registered in
  the `SshAgent`'s `users` dict.  For local commands, this
  is the user the subprocess runs as (when the framework
  has the privilege to switch).
- **`pty_params`** — optional dict of pseudo-terminal
  parameters when the command needs an interactive PTY.

Other protocol-specific Aspects subclasses follow the same
pattern.  `AspectsRest` (chapter 6) adds REST-specific fields
like `retry_logging_interval` and `session`.  Future
protocols can subclass to add their own protocol-specific
behaviour parameters without touching the base class.

## `ActionPattern` — the persistence behaviours

**File:** `aspects.py:24–32`

```python
class ActionPattern(IntEnum):
    SINGLE_CALL = 0
    SINGLE_CONNECTED_CALL = 1
    DO_UNTIL_SUCCESS = 2
    DO_WHILE_SUCCESS = 3
    DO_UNTIL_CONNECTION_FAILURE = 4
```

Five named behaviours.  Each is a complete persistence
strategy.  The `ActionPattern` field on the Aspects object is
how the application layer selects which strategy applies.

### `SINGLE_CALL`

> *Try once.  Return whatever happens.*

The default.  No persistence — the simplest behaviour.
Appropriate for:

- Calls whose success is *expected* and whose failure is
  meaningful (e.g., "is the file present?").
- Calls where retrying would corrupt state (e.g., "submit
  this transaction").
- Quick health checks where a single failure is the
  diagnostic signal.

A `SINGLE_CALL` honours the timing budgets only for the
single attempt.  If the call exceeds `completion_timeout`,
the interop layer raises `TimeoutError` per principle 4.

### `SINGLE_CONNECTED_CALL`

> *Wait for a connection.  Then try once.*

A two-phase strategy: persist the *connection* (using
`connection_timeout` / `connection_interval`), then issue
the call once.  Appropriate when:

- The remote service might not be reachable yet (just
  started, mid-deployment).
- The actual call is one-shot (not a polling operation), but
  reaching the service is the unreliable part.

### `DO_UNTIL_SUCCESS`

> *Retry until the expected status is observed, or until the
> deadline.*

The most common persistence behaviour.  Repeatedly issues
the call; treats each non-`expected_status` result as a
retry-trigger; gives up only when:

- The call returns `expected_status` (success → return), or
- `completion_timeout` elapses (failure → raise
  `TimeoutError`), or
- The deadline is hit while a retry is in progress (failure
  → raise).

Useful for:

- "Wait for the cluster to converge."  Poll a status command
  until it reports converged.
- "Wait for the device to come back after reboot."  Poll an
  SSH connection until it succeeds.
- "Wait for the service to register itself in DNS."  Resolve
  the hostname until it returns the expected address.

### `DO_WHILE_SUCCESS`

> *Keep running while the expected status holds.  Stop on
> mismatch.*

The mirror image.  Useful when *observing* a steady-state
condition that's expected to be true *until something
changes*:

- "Hold the connection open and watch logs; stop when the
  service reports a fault."
- "Monitor the cluster's health while a test runs; stop if
  health degrades."
- "Stream events from the device and capture them."

A tandem validator (Volume 2 chapter 5) is a higher-level
construct that uses `DO_WHILE_SUCCESS` semantics under the
hood.

### `DO_UNTIL_CONNECTION_FAILURE`

> *Retry until the connection drops.*

The least common pattern.  Useful for *observing-until-end*
scenarios where the call should keep going as long as the
connection holds.  When the connection drops, the call
returns whatever was accumulated.

## `LoggingPattern` — what to log

**File:** `aspects.py:34–40`

```python
class LoggingPattern(IntEnum):
    ALL_RESULTS = 0
    FAILURE_ONLY = 1
    SUCCESS_ONLY = 2
```

Three modes for emit-while-running logging:

- **`ALL_RESULTS`** (default) — log every attempt's outcome.
  Useful for diagnosing flake.
- **`FAILURE_ONLY`** — log only failures.  Useful for
  production runs where success is expected and shouldn't
  fill the logs.
- **`SUCCESS_ONLY`** — log only successes.  Useful for
  capturing what *did* happen during long-running observers.

The `LoggingPattern` field is independent of the framework's
overall log-level setting.  Even with the framework's logger
at INFO, an Aspects with `LoggingPattern.FAILURE_ONLY` keeps
successful retries quiet — the application layer chose to
suppress the noise.

## `Freezable` — immutability after construction

**File:**
`codecraftai-xmodules/source/packages/ccai/xmodules/freezable.py:13`

```python
class Freezable:
    """
        Mixin that enforces post-construction immutability.

        A subclass calls self.freeze() at the end of __init__.  After
        freeze, any attempt to set an attribute raises AttributeError.
    """

    def __init__(self) -> None:
        object.__setattr__(self, "__frozen__", False)
        return

    def freeze(self) -> None:
        object.__setattr__(self, "__frozen__", True)
        return

    def __setattr__(self, name: str, value: object) -> None:
        if getattr(self, "__frozen__", False):
            err_msg = "Cannot set attribute '{}' on frozen {} instance.".format(
                name, type(self).__name__,
            )
            raise AttributeError(err_msg)
        object.__setattr__(self, name, value)
        return
```

The mechanism is small and efficient:

- `__init__` sets `__frozen__ = False` (via direct
  `object.__setattr__` to bypass the override).
- The subclass's `__init__` does its setup.
- `self.freeze()` flips `__frozen__ = True`.
- Subsequent `obj.x = ...` calls hit `__setattr__`, see
  `__frozen__` is True, and raise.

The result: an `Aspects` instance is *immutable* after its
constructor returns.  This is what makes Aspects safe to
share across concurrent calls — no thread can mutate an
aspects instance another thread is using.

The cost is that creating a *modified* aspects requires
constructing a new instance.  Idiomatic code:

```python
fast_aspects = AspectsCmd(
    completion_timeout=10,
    action_pattern=ActionPattern.SINGLE_CALL,
)

# Later, want a slightly different version — construct anew.
patient_aspects = AspectsCmd(
    completion_timeout=300,
    completion_interval=5,
    action_pattern=ActionPattern.DO_UNTIL_SUCCESS,
)

# Use both freely; they don't interfere.
```

The Aspects-construction overhead is negligible compared to
the cost of the protocol calls themselves.

## `MonitoredScope` — the deadlock detector

**File:**
`codecraftai-xmodules/source/packages/ccai/xmodules/xlogging/monitoredscope.py:34`

```python
class MonitoredScope:
    """
        Context manager that registers thread-scoped monitoring with
        a global ScopeMonitor daemon.  If the thread does not exit
        the scope before timeout + notify_delay, the monitor logs an
        error.

        Used by interop calls to detect when a thread has gotten
        stuck inside a blocking I/O call past the agreed timeout.
    """

    def __init__(self, label: str, message: str, timeout_context: TimeoutContext,
                 notify_delay: float = 30):
        # ...
        return
```

`MonitoredScope` is the *deadlock detector* the interop layer
wraps every persistent call in.  When a call enters a
`MonitoredScope`, the global `ScopeMonitor` daemon thread is
told "expect this thread to leave the scope within
`timeout + notify_delay` seconds."  If the thread doesn't
leave (because it's stuck in a syscall, or the underlying
library is hanging), the monitor logs an error reporting
the thread is past its budget.

Use sites:

- `LocalCommandAgent.run_cmd` at line 104 — wraps subprocess
  execution.
- `SshBase.run_cmd` at line 545 — wraps SSH command
  execution.

The monitor doesn't *kill* the thread (Python doesn't make
that safe).  But the log entry is the diagnostic signal that
something has gone wrong at the wire level — typically the
network stack hanging on a syscall or a paramiko bug.

A reader debugging "why did this hang past the timeout" works
backwards from the monitor's log entry to the thread's stuck
syscall.

## The default constants

**File:** `aspects.py:43–62`

```python
DEFAULT_COMPLETION_TIMEOUT = 600
DEFAULT_COMPLETION_INTERVAL = 10
DEFAULT_CONNECTION_TIMEOUT = 16
DEFAULT_CONNECTION_INTERVAL = 2
DEFAULT_ALLOWED_ERROR_CODES: Tuple[int] = []
DEFAULT_ALLOWED_CONNECTION_EXCEPTIONS: Tuple[BaseException] = []
DEFAULT_MUST_CONNECT = False
```

And the default instances at line 202:

```python
DEFAULT_ASPECTS = Aspects()
DEFAULT_CMD_ASPECTS = AspectsCmd()
```

The default values are deliberate but conservative:

- 10-minute completion timeout — long enough for slow
  operations on real lab gear.
- 10-second completion interval — frequent enough for
  responsive feedback, infrequent enough to not hammer the
  system.
- 16-second connection timeout — enough to handle the
  occasional slow-handshake case.

Most calls use the defaults.  Calls that need different
budgets construct their own AspectsCmd.  The defaults are
*starting points*, not rules.

## The aspect-passing flow through a call

A worked example.  A test calls
`device.ssh.run_cmd("uptime", aspects=fast_aspects)`.  The
flow:

1. **Test code** — declares the call with explicit aspects.
2. **`device.ssh`** — the `SshProtocolExtension` (Volume 6 chapter 3),
   wrapping the SSH agent.
3. **`SshAgent.run_cmd`** — receives `aspects=fast_aspects`.
   If `aspects is None` (caller passed nothing), falls back to
   `self._aspects` (set at agent construction).
4. **`SshBase._run_cmd`** — the template method.  Resolves
   `completion_timeout`, `inactivity_timeout`, etc. from the
   aspects.  Constructs a `TimeoutContext`.
5. **Action-pattern dispatch** — examines
   `aspects.action_pattern`.  For `SINGLE_CALL`, executes
   once.  For `DO_UNTIL_SUCCESS`, enters a retry loop using
   the timeout context.
6. **`MonitoredScope` wrap** — wraps each attempt in a scope
   with the relevant timeout.
7. **`_ssh_execute_command`** — runs the actual paramiko
   call.
8. **Return path** — status / stdout / stderr returned to the
   test, OR a `TimeoutError` / `ConnectionError` raised when
   the contract is exceeded.

Steps 2 through 7 are inside the interop layer.  The test
sees only steps 1 and 8.  The aspect parameter is the only
thing the test specified about *how* the call should behave;
everything else is the framework following the contract.

## The relationship to the framework's wider patterns

A few connections the chapter has gestured at and the rest
of Volume 5 makes explicit:

- **Volume 0 chapter 4 (Aspects pattern philosophy)** —
  introduced Aspects as a Volume-0-level discipline.  This
  chapter is the implementation of that discipline.
- **Volume 0 chapter 5 (waiting primitives)** — the
  `wait_for_it` function that Volume 5 chapter 4's SSH agent
  uses internally is the same primitive validators (Volume 2
  chapter 5) use.
- **Volume 0 chapter 3 (error vocabulary)** — the typed
  exceptions raised by the interop layer follow Volume 0's
  bubble-the-failure discipline.
- **Volume 6 chapter 3 (Coordinator/Agent/ProtocolExtension)** —
  the Agents whose calls take Aspects are the Agents Volume 6
  named.

These cross-references are what make the volume hold together
with the rest of the series.  Measured Interoperability is
not a Volume 5 invention; it's the implementation of
foundation-tier disciplines from Volume 0, applied to the
specific concerns of cross-machine wire calls.

## What's next

Chapter 3 walks `codecraftai-interop` itself — the package
structure, the Agent base classes, the shipped protocols
(local commands, SSH paramiko-wrapped, serial-over-TCP, DNS,
power), and the `am-interop` CLI scaffolding.  Where this
chapter walked the *delegation contract*, the next chapter
walks the *agents that fulfil it*.

Chapter 4 then deep-dives the from-scratch SSH stack in
`codecraftai-protocol-ssh` — packets up to agents — as an
in-depth worked example of Measured Interoperability applied
to a complex protocol.
