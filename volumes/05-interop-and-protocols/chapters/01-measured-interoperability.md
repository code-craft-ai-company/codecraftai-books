# Chapter 1 — Measured Interoperability — The Philosophy

> *Networks flake.  Services take time to start.  A command that
> succeeds nine times out of ten will fail on the tenth run and
> leave a test suite in an unknown state.  The naive solution —
> `time.sleep(30)` before every interaction — wastes time on
> fast machines and still fails on slow ones.  Hand-written
> retry loops scatter timing decisions throughout the codebase.
> `automojo-interop` solves this with **Measured
> Interoperability**: a clean separation of *Mechanism* (what
> to do) from *Behaviour* (how persistently to do it).*

This chapter establishes the philosophy that shapes the entire
interop layer.  Every protocol package built on
`automojo-interop` — SSH, REST, UPnP, Tasker, Bluetooth, CAN,
serial, power — implements **Measured Interoperability**.  The
discipline is what makes the protocol stack predictable,
testable, and tunable across the wildly different timing
profiles real-world automation runs against.

The chapter is the volume's foundation.  Chapter 2 walks the
*mechanism* that implements the discipline — the Aspects
pattern.  The remaining seven chapters walk the protocol stack
itself.  But none of those make sense without this chapter's
framing.

## The premise — what the interop layer's workload actually is

Distributed automation and interop are dominated by
**persistence behaviours**: actions that must keep trying,
polling, waiting, or reconnecting until a condition is met or
a deadline passes.

Concrete examples from the portfolio:

- Run a command on a remote host; the command takes a
  variable amount of time; **retry** the read of stdout / stderr
  until the command exits.
- Reboot a device; **wait** for it to become reachable again
  via SSH.
- Apply a configuration change; **poll** until the change has
  propagated to all four nodes in the cluster.
- Check that a service is healthy; **retry** the health probe
  if it returns transient errors.
- Stream events from a device; **reconnect** when the wire
  drops.
- Send a power-cycle command; **wait** for the device to
  return to a known state.

Most interop API calls are not "do X once and return."  They
are "do X, then verify X took effect, retrying or polling
until success criteria are met."  The interop layer is built
around this premise; the framework's `ActionPattern` enum
names the canonical persistence behaviours:

```python
# automojo-interop/source/packages/automojo/interop/aspects.py:24–32

class ActionPattern(IntEnum):
    SINGLE_CALL = 0                    # Try once, return the result.
    SINGLE_CONNECTED_CALL = 1          # Wait for connection, then try once.
    DO_UNTIL_SUCCESS = 2               # Retry until success or deadline.
    DO_WHILE_SUCCESS = 3               # Poll while the condition holds.
    DO_UNTIL_CONNECTION_FAILURE = 4    # Retry until the wire drops.
```

Five named persistence behaviours.  Each is a *strategy* the
interop layer can apply to a call.  Each call selects one.

## What "measured" means

The userguide's framing:

> *`automojo-interop` solves this with Measured
> Interoperability: a clean separation of two concerns.*
>
> *Mechanism: The act of sending bytes to a target — invoking
> `subprocess.Popen`, opening a paramiko channel, writing to
> a serial port.  This is the agent's job.*
>
> *Behaviour: How tenaciously to retry, how long to wait
> before giving up, when to log results, which error codes to
> treat as transient.  This is the `Aspects` object's job.*
>
> — `automojo-docs/userguide/packages/automojo-interop.rst:25`

The word **measured** carries weight.  The interop layer is
not merely *interoperable* — it is *measured* in its
interoperability.  The framework's calls don't lock onto the
wire and refuse to give up.  They don't give up after the
first hiccup.  They apply *measured* persistence — exactly the
amount the application layer asked for, no more, no less.

The discipline is what distinguishes Measured Interoperability
from naive retry loops.  The naive approach has timing
decisions everywhere: `time.sleep(30)` here, `for i in
range(5)` there, `try / except` blocks scattered through test
bodies.  Measured Interoperability has a single mechanism —
the Aspects parameter — that carries timing decisions from
the application layer down to the interop layer where they're
applied.

## The four principles

Measured Interoperability rests on four load-bearing
principles.  These principles describe *who owns what
concerns* across the polling-dominated workload of the interop
layer.

### Principle 1 — Persistence behaviours live in the interop layer, not the application layer

Tests and helper code should *not* contain hand-written
persistence behaviours.  No `for _ in range(N): try: ...
except: time.sleep(...)` loops in test bodies.  Persistence
behaviours are the **interop layer's responsibility**, not the
test author's.

If they lived in the application layer:

- Every test would reinvent persistence, with subtly different
  semantics.
- Bugs would proliferate — off-by-one timeouts, wrong
  intervals, missing back-off, swallowed exceptions, retry-
  past-real-failure patterns that hide bugs.
- Behaviour would be inconsistent across the codebase.  One
  test polls every 0.5 seconds, another every 5 seconds,
  neither author thought about it.
- Code review couldn't catch persistence-bug patterns
  because they'd be scattered.

Centralising persistence at the interop layer means: one
implementation, well-tested, consistently applied.  The test
author calls `device.power.cycle()` — under the hood, the
function applies the chosen `ActionPattern` to the call.  The
test author doesn't write the loop.

### Principle 2 — Decisions about flow control stay in the application layer

The library / low-level protocol code does NOT make
application-level decisions about what to do when things go
wrong.  The interop layer doesn't decide "wait 30 seconds and
try again, but only twice" on its own — that's a *behaviour*
the application layer chose.

When the interop layer *does* execute such decisions (which
it must, to perform persistence behaviours), it does so under
a **delegation contract**.  The application layer passes the
*details of the behaviour* to the interop layer; the interop
layer follows the delegation faithfully.

**The Aspects pattern is the delegation contract.**

The `Aspects` parameter on every interop call (chapter 2 walks
it in detail) is how the application layer hands the interop
layer its policy:

- Which `ActionPattern` to apply.
- How long to wait (`completion_timeout`,
  `connection_timeout`, `inactivity_timeout`).
- How often to poll (`completion_interval`,
  `connection_interval`, `inactivity_interval`).
- What counts as success (`expected_status`,
  `allowed_error_codes`).
- What `LoggingPattern` to emit while running.

Different callers use different policies on the same library
function; the library code doesn't change.

This is why decorators are wrong for this purpose (Volume 0
chapter 2's discipline lands here).  A `@retry(times=3,
delay=10)` decorator hard-codes the policy at the *library*
level — every call site gets the same policy whether or not
it's the right one.  An `aspects=AspectsCmd(...)` parameter
passes the policy *from the caller per call* — every caller
chooses.

### Principle 3 — Neither too brief nor too persistent

The interop layer cannot:

- **Immediately give up.**  A single failed packet, a
  momentary connection blip, a transient timeout — these
  would manifest as test failures despite the underlying
  system being fine.  False negatives.
- **Be too persistent.**  Retrying forever, retrying through
  real failures, polling past actual breakage — these *hide
  bugs*.  A test that should have failed (because the system
  is genuinely broken) appears to pass because the interop
  layer kept retrying until coincidentally succeeding.
  False positives.

All interop code adheres to **flow control contracts** that
specify **persistence criteria**.  The criteria define *when
to stop trying*:

- Total time elapsed (`completion_timeout`).
- Time without progress (`inactivity_timeout`).
- Number of attempts.
- A specific error code that means "permanent failure, don't
  retry."
- A specific success condition that means "yes, done."

The criteria are encoded in the Aspects object.  The interop
layer follows them — *exactly* as long as they say to try,
*exactly* when they say to stop.

The framework's defaults (from
`automojo-interop/source/packages/automojo/interop/aspects.py:43–62`):

```python
DEFAULT_COMPLETION_TIMEOUT = 600          # 10 minutes total
DEFAULT_COMPLETION_INTERVAL = 10          # check every 10 seconds
DEFAULT_CONNECTION_TIMEOUT = 16           # 16 seconds to establish
DEFAULT_CONNECTION_INTERVAL = 2           # retry connection every 2s
DEFAULT_ALLOWED_ERROR_CODES: Tuple[int] = []
DEFAULT_ALLOWED_CONNECTION_EXCEPTIONS: Tuple[BaseException] = []
DEFAULT_MUST_CONNECT = False
```

These defaults are sensible starting points.  A specific call
that needs different criteria builds its own
`AspectsCmd(completion_timeout=...)` and passes it.

### Principle 4 — When the contract's bounds are exceeded, control reverts via exception

The interop layer never makes the decision "we tried hard
enough, let's give up silently and report success."  It also
never makes the decision "we'll keep trying past the agreed
budget because we *think* it'll work eventually."  When the
persistence criteria from the application layer are
*exhausted* — the timeout elapsed, the attempt count reached,
the deadline passed — the interop layer **raises an
exception** and returns control to the application layer.

The exception is the *exit mechanism* of the delegation
contract.  The application layer:

- *Delegated* persistence responsibility (principle 1).
- *Specified* the criteria via Aspects (principle 2).
- The interop layer *honoured* the criteria (principle 3).
- The interop layer *reverts control* when the criteria are
  exhausted (principle 4).

The application layer then decides what to do.  Catch the
exception and retry with different aspects.  Fail the test.
Escalate to the orchestrator.  Try a fallback approach.  The
interop layer doesn't choose; it surfaces the situation and
lets the application choose.

This is why the interop layer's exception types are *typed*
(`SshAuthError`, `SshSFTPError`, `TimeoutError`,
`ConnectionError`) and not generic `RuntimeError`.  The
exception's type tells the application layer *which kind* of
contract bound was hit, so the application can dispatch on
the type:

```python
try:
    device.ssh.run_cmd(cmd, aspects=fast_aspects)
except TimeoutError:
    # The completion_timeout elapsed.  Try with longer aspects.
    device.ssh.run_cmd(cmd, aspects=patient_aspects)
except SshAuthError:
    # Auth failed.  Credentials might have rotated.  Re-resolve.
    device.refresh_credentials()
    device.ssh.run_cmd(cmd, aspects=fast_aspects)
except ConnectionError:
    # The wire dropped.  Different recovery — escalate.
    raise InfrastructureError("Pod is unhealthy")
```

The typed exceptions are the *vocabulary* through which the
contract reverts control.

## The status-vs-exception split

A subtlety worth pinning down because it surprises readers:
**a non-zero exit status from a remote command does NOT raise
in the interop layer.**

The reason: a non-zero status is *within* the contract.  The
contract was "run this command, capture status / stdout /
stderr."  The contract was honoured even when the command
exited 1.  The application layer reads the status and
decides whether the non-zero is meaningful.

A test that *expects* a non-zero status (say, "this command
should fail because the file doesn't exist") wants
`status == 1`, not an exception.  Treating the non-zero as
an exception would force every caller to wrap in `try /
except` even for expected failures.

By contrast, a *transport failure* (network drop, SSH auth
failure, connection timeout) **exceeds the contract** —
"capture status / stdout / stderr" cannot be honoured if the
session never opened.  The interop layer raises a typed
exception per principle 4.

The split is:

- **Status as data** — a non-zero exit code is data the
  application reads.
- **Exception as control reversion** — a contract violation
  reverts control.

Together, they cover the full surface.  Application code
reads status when it cares about the command's exit; catches
typed exceptions when the contract was exceeded.

## How the principles interlock

The four principles compose into one design:

1. The interop layer *does the persistence work* (principle 1) →
2. *under a delegation contract* (principle 2) →
3. *with persistence criteria that bound behaviour* (principle 3) →
4. *and reverts control via typed exception when the criteria are exhausted* (principle 4).

The Aspects parameter is the artifact carrying all four.  The
application layer *delegates* persistence (principle 1),
passes its *decisions about flow control* via the Aspects
parameter (principle 2), the parameter encodes the
*persistence criteria* (principle 3), and the interop layer
follows the criteria — including raising on exhaustion
(principle 4).

## Where this philosophy applies

**The whole interop layer for all protocols uses Measured
Interoperability.**  Every protocol package in the portfolio
implements it:

- `automojo-interop` — the core.  Local commands, SSH (paramiko-
  wrapped), serial-over-TCP, power, DNS / mDNS.
- `automojo-protocol-ssh` — from-scratch SSH.
- `automojo-restclient-base` — REST foundation; `AspectsRest`
  is the REST-shaped Aspects subclass.
- `automojo-restclient-github-public` — auto-generated GitHub
  client; every method takes `aspects=AspectsRest(...)`.
- `automojo-interop-ext-upnp`, `-tasker`, `-bluetooth`, `-can`
  — protocol extensions.

The discipline is uniform.  An engineer who understands this
chapter can read any protocol agent in the portfolio and know
what the Aspects parameter is for.

## Why this design holds up at scale

Three properties make Measured Interoperability work in
production:

### Configuration consistency

Because every interop call accepts `Aspects`, a project that
needs longer timeouts in CI (slower lab) and shorter timeouts
locally (faster laptop) sets the *Aspects* differently per
environment.  No code changes; the same library calls work
both places.

### Concurrent-safe sharing

`Aspects` and its subclasses inherit from `Freezable`.  Once
constructed, an `Aspects` instance is immutable.  This makes
it safe to share across concurrent calls — no risk of one
thread mutating an aspects instance another thread is using.
Volume 2 chapter 11's cross-host fan-out relies on this
property.

### Composability with the framework

The framework's testjob lifecycle, validators, configuration
profiles, and orchestration all compose with Measured
Interoperability cleanly.  A tandem validator (Volume 2
chapter 5) is *itself* an application of the four principles
at a higher framework layer: validators centralise polling
(principle 1), take their criteria as constructor parameters
(principle 2), bound persistence (principle 3), raise
`AssertionError` on contract exhaustion (principle 4).  The
discipline scales.

## What this chapter set up

By the end of this chapter a reader should:

- Know the formal name **Measured Interoperability** and the
  Mechanism / Behaviour separation.
- Recognise **persistence behaviours** as the umbrella term
  for retry / poll / wait / reconnect patterns.
- Be able to name the five `ActionPattern` modes:
  `SINGLE_CALL`, `SINGLE_CONNECTED_CALL`,
  `DO_UNTIL_SUCCESS`, `DO_WHILE_SUCCESS`,
  `DO_UNTIL_CONNECTION_FAILURE`.
- Understand the four principles: persistence in interop,
  decisions in application, criteria bound persistence,
  control reverts via exception.
- Distinguish status-as-data from exception-as-control-
  reversion.

The next chapter walks the **Aspects pattern** in code
detail: the `Aspects` and `AspectsCmd` classes, their fields,
their immutability, the `LoggingPattern` enum, the `MonitoredScope`
deadlock-detection mechanism, and the typical aspect-passing
flow through an interop call.
