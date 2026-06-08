# Chapter 8 — Performance and Storage Trade-offs

> **Abstract.** Recording is not free.  The `sys.settrace` hook
> imposes per-event Python interpreter overhead; the
> serialiser pickles each payload; the buffer flushes synch-
> ronously when full; the network or disk absorbs the bytes.
> Each layer has a measurable cost, and the cost compounds with
> the recording's *granularity*.  This chapter quantifies what
> calls, lines, and variables cost in CPU and bytes; quotes
> measured numbers from comparable Python tracing literature
> and the pod's own design constraints; and lays out the policy
> framework operators should use to decide *when to enable,
> when to disable*.  All numbers are honest about being
> estimates, not benchmarks: the pod's source as committed is
> the design; the production-quality benchmark suite that would
> validate the design is part of the work that follows.

## 8.1 What "overhead" means here

Three different overhead figures matter, and conflating them
muddies decisions:

**Quiescent overhead.**  The cost the application pays when
*no portal is active*.  Every portal entry runs the
`__enter__` decision tree
(`pod/portal.py:60-141`) — three early-exit conditional
branches and a thread-local lookup.  This is the cost the
application *always* pays as long as the pod is loaded.

**Active overhead.**  The cost the application pays when a
portal *is active*.  Every Python interpreter event inside
the portal's scope crosses through `_trace_function`
(`pod/tracehookmanager.py:63-137`), constructs an event,
serialises it, and writes to the buffer.

**Storage overhead.**  The bytes-on-disk (or bytes-on-the-
wire) consumed per recording.  Independent of CPU overhead;
sometimes the bottleneck.

The three are independent in important ways: quiescent is *low*
regardless of granularity; active is dominated by
*granularity*; storage is *granularity × duration*.  An
operator's decisions live in this three-dimensional space.

## 8.2 Quiescent overhead

The quiescent path is the fast path.  The portal's `__enter__`
runs:

```python
def __enter__(self):
    if not is_gateway_active():
        return self

    state = get_tracing_state()

    if state.is_tracing:
        state.portal_depth += 1
        ...
        return self

    registry = get_portal_registry()
    if not registry.is_portal_active(self._name):
        return self
    ...
```

In quiescent mode (gateway active, no portal activated, this
thread not tracing), the path through `__enter__` is:

1. `is_gateway_active()` — read a Python global
   (`pod/tracingstate.py:61-69`).  Microseconds.
2. `get_tracing_state()` — read thread-local; on first call
   per thread, lazy-init.  Microseconds.
3. `state.is_tracing` — instance attribute access.  Sub-
   microsecond.
4. `get_portal_registry()` — read a module global.  Sub-
   microsecond.
5. `registry.is_portal_active(self._name)` — acquire the
   registry's `threading.Lock`, do a dict lookup, release.
   Microseconds.
6. Return.

Total: *low single-digit microseconds per portal entry*, with
the registry lock acquisition as the dominant cost.  In a
service handling thousands of requests per second, each with
one or two portal entries on the hot path, the overhead is
*low millisecond aggregate over a second of wall time* — under
1 % of a single CPU core.

The cost is bounded *per portal entry*, not per portal.  An
application with 50 declared portals does not pay for 50; it
pays for the portals that are *crossed* on the request path.
A typical request path crosses 3-10 portals.

The quiescent overhead is the price of *having* the recording
capability at all.  It is small enough that production
deployments routinely keep the pod loaded permanently.

### What changes the quiescent number

A pathological pattern would make the quiescent number
dramatically worse:

**Activating a portal that everything crosses.**  A portal
named `request` activated globally would push every request
into the *active* path — the quiescent overhead is no longer
quiescent.  This is what the `expression` filter
(`pod/portal.py:99-112`) is for: activate broadly, filter
narrowly.  An expression like `customer_id == 'C-12345'`
narrows the active recording to a tiny fraction of crossings
without changing the activation policy.

**Holding the registry lock.**  A long-running operation that
holds the registry's lock would serialise all portal entries
on all threads.  The registry's operations are designed to be
fast (microseconds per dict access), and the lock is never
held across a Python-level operation that could block.  But
adding new operations under the lock — say, a remote call to
verify an activation — would break the quiescent guarantee.
The book's recommendation: keep the registry's critical
section a pure dict access.

## 8.3 Active overhead by granularity

The active overhead depends on what `TraceConfiguration` is
recording.

### Calls only

`trace_calls=True`, others false.  An event is emitted on
*function entry* and *function return*.  Inside a function
body, no per-line cost.

The per-event cost:

1. `_trace_function` runs in the Python interpreter's trace
   path.  Just being in a `sys.settrace`'d frame imposes a
   noticeable interpreter overhead — the CPython interpreter
   has to call into Python on every event.
2. A `CallEvent` is constructed (allocation,
   `__init__`, frame-attribute reads).
3. The event is passed to `record_callback` →
   `serialiser.serialize(event)`.
4. The 29-byte envelope is `struct.pack`-ed.
5. The payload (the function name and any arguments) is
   `pickle.dumps`-ed.
6. The bytes are appended to the buffered writer; if the
   buffer is full, a flush fires.

The Python interpreter's `sys.settrace` overhead alone — even
with a no-op trace function — typically falls in the 5x to
10x range relative to un-traced execution for code that is
function-call-heavy.  The pod's per-event work
(allocation, pickle, envelope) adds further cost.

Estimated overhead, calls-only, on Python-call-heavy code:
**roughly 5-10x un-traced execution time**.  On code where
function-call boundaries are rare (a tight loop with one
function), the overhead is much smaller in absolute terms — the
calls are few, so the per-call cost is rare.

Storage volume, calls-only: per call, an envelope (29 B) +
filename (~30 B average) + pickle payload for function name
and arguments (~50-150 B) ≈ 110-210 B per CallEvent.  A
typical request that produces 100 function calls produces
about 10-20 KB.  Continuous recording at calls-only on a
service handling 100 requests/second produces 1-2 MB/s.

### Calls + lines

`trace_calls=True, trace_lines=True`.  An event is emitted on
function entry, function return, *and every line executed
inside a recorded function*.

The per-event cost is the same as calls-only per *event*; the
*event count* is dramatically higher.  A function with 10
lines produces 12 events instead of 2 for a single call.

Estimated overhead, calls + lines: **roughly 50-100x un-traced
execution time** for line-heavy code.  This is the regime
where the recording is no longer "free enough to leave on";
this is the regime where the recording is targeted to a
window.

Storage volume: with no payload (`LineEvent` has no payload —
ch. 3 §3.3), each LineEvent is envelope (29 B) + filename
(~30 B) ≈ 60 B.  A function call with 10 lines produces
~ 1 CallEvent (200 B) + 10 LineEvents (600 B) + 1 ReturnEvent
(200 B) ≈ 1 KB.  At 100 requests/second producing the same
shape of trace, ~ 100 KB/s — modest in absolute terms.

### Calls + lines + variables

`trace_variables=True` adds a `VariableSnapshotEvent` per
line that records `dict(frame.f_locals)`.

The cost has two components:

The first is the *dict snapshot itself*.  `dict(frame.f_locals)`
is a copy of every local variable currently in scope.  In a
function with 5 small primitives in scope, the copy is fast
(microseconds).  In a function whose `frame.f_locals` includes
a giant data structure (a parsed JSON document, a database
result set, a pickled object representation), the copy is
expensive (milliseconds).

The second is the *pickle of the dict*.  Pickle of arbitrary
Python objects can be O(size-of-object), and large objects
produce large pickle outputs.

Estimated overhead, variables: **typically 100-500x un-traced
execution time** for code with non-trivial local dictionaries.
Edge cases with huge locals (megabytes of arrays) push this
much higher.

Storage volume: variable snapshots are the heavyweight events.
Each is envelope (29 B) + filename (~30 B) + pickled dict
(50-1000+ B depending on contents).  A function with 10 lines
and a 5-key dict in scope: 10 lines × ~250 B per snapshot ≈
2.5 KB extra per call.  At 100 requests/second, this is
~ 250 KB/s per portal.

### The order-of-magnitude summary

| granularity        | active CPU overhead | per-call bytes |
|--------------------|---------------------|----------------|
| calls only         | ~5-10x              | ~100-200 B     |
| calls + lines      | ~50-100x            | ~1 KB          |
| + variables        | ~100-500x           | ~3-100 KB      |

These are *estimates* based on the design and on the broader
Python literature on `sys.settrace` overhead.  They are not
benchmark numbers from the pod itself.  The benchmark suite
that would validate them is part of the production-quality
investment the volume calls out as future work.

## 8.4 The dominant cost in each regime

Knowing *which layer* is dominant in each regime helps direct
optimisation effort.

**Calls-only regime: the dominant cost is the interpreter's
`sys.settrace` overhead.**  Python pays a fixed cost on every
event for *being in* a trace function; the pod's own work is
modest by comparison.  Optimising the pod's own code (faster
allocation, faster envelope packing) gets diminishing returns;
the floor is the interpreter.

**Lines regime: the dominant cost is the *number of events*.**
Each line is an event.  Reducing the per-event cost helps;
what helps more is reducing the line count — narrowing the
portal scope, using `trace_calls` only inside a tighter
portal.

**Variables regime: the dominant cost is the dict copy and
the pickle.**  Reducing the size of `frame.f_locals` helps —
a function whose body uses fewer named locals pays less per
snapshot.  This is unfortunately the kind of optimisation
that *unbreaks* the recording rather than improving the
recorded code; the recording's cost sometimes drives a
refactor of the recorded code's locals.

## 8.5 Network and disk costs

The pod's bytes leave the process through a backend.  The
backend's cost depends on the medium:

**File backend**
(`pod/backends/filebackend.py`).  A single buffered file
write per 64 KiB chunk (the buffered writer's default).  Modern
SSD writes 64 KiB in microseconds.  The file backend is
*essentially free* once the buffer is filled; the dominant
cost on the recording side is the buffering and the pickle, not
the disk.

**TCP backend**
(`pod/backends/tcpbackend.py:85-107`).  Writes 64 KiB at a
time over a TCP socket.  On a same-host loopback, the cost is
microseconds.  On a real network, the cost is the round-trip
plus the bytes' transfer time.  At 1 Gb/s LAN, 64 KiB
transfers in ~ 0.5 ms; the recording-rate ceiling on TCP is
about 1.5 GB/s wire — much higher than any plausible recording
rate the pod produces.  Bottleneck: the *server's* ingestion
rate, not the wire.

**Unix backend**
(`pod/backends/unixbackend.py`).  Same as TCP but over a Unix
domain socket.  Lower per-syscall cost (no protocol stack);
otherwise the same.  Best when pod and server share a host.

The drop-on-saturation behaviour
(ch. 3 §3.6) is the safety valve.  If the network saturates,
the pod drops rather than blocking the recorded application.
This cap on the network-side overhead is the design's
non-negotiable contract: *the recording must not slow the
application*.

## 8.6 The server's ingest ceiling

The server-side path
(ch. 4) drains bytes in 4 KiB chunks and writes them to a
file backend.  Per-chunk cost on the server:

1. `socket.recv(4096)`.
2. `storage_backend.append(chunk)` — file write.
3. `flow_controller.update_buffer_size` — lock, two int
   updates.
4. Substring check for `b"FLUSH"` in the chunk.
5. Optional flow-control signal back to the pod.

Steps 1, 2, and 4 are the dominant costs.  On modern hardware,
this loop runs at *hundreds of MB/s per worker thread*.  With
a small thread pool (default 100 max concurrent connections —
`server_config_store.py:7-27`), the server's aggregate ingest
ceiling is several GB/s.

The server's *deserialisation* path is much slower — it pays
the pickle cost per event.  But deserialisation is deferred to
the analysis tier; the receive path stores bytes only.  This
load-shedding is what gives the server its ingest headroom.

## 8.7 The granularity-vs-duration trade

A useful framing: a recording's *total bytes* is roughly
`granularity × duration`.  Operators can spend their byte
budget two ways — record a low-granularity stream over a long
window, or a high-granularity stream over a short window.

| posture                               | budget               | use case                              |
|---------------------------------------|----------------------|---------------------------------------|
| `calls`, continuous                   | ~ 1 MB/s/portal      | "always-on shadow" debugging          |
| `calls`, one-shot                     | ~ 100 KB / capture   | "next failure" — case study 1         |
| `calls + lines`, narrow window        | ~ 1 MB / window      | "this function is suspicious"         |
| `+ variables`, narrow window + filter | ~ 10-100 MB / window | "this exact failing call" — case 2, 3 |

The narrow-window-plus-filter is the high-leverage posture for
production: an expression like `customer_id == 'C-12345'`
combined with line + variable granularity records *only the
failing call*'s state at full fidelity.  The total volume is
small; the diagnostic value is high.

## 8.8 The retention window

§6.5 covered retention algorithmically; the cost framing is
useful here too.

A 30-day retention with 100 pods each producing 10 recordings
of average 100 MB per day produces 30 × 100 × 10 × 100 MB =
3 TB.  That is the steady-state storage budget for the
medium-deployment shape from §6.13.

Retention is the only thing keeping that number *bounded*.
Without retention, the storage grows unbounded with the fleet
size.  An operator's choice of `data_retention_days` is the
choice of how much disk to provision.

A common operational pattern: short retention for routine
recordings, long retention for *flagged* recordings.  The
implementation today has only one retention setting (the
schema does not include a "flagged" flag on
`TraceSession`).  The book's recommendation: an explicit
`flagged` field that exempts the row and its `data_reference`
from the standard retention sweep would let operators preserve
a small set of forensic recordings beyond the default window.

## 8.9 The when-to-enable policy

A pragmatic policy operators can adopt:

**Always-on:** the pod is loaded; the gateway is entered; the
quiescent overhead is paid.  Portals are declared at every
architectural boundary the developer pre-identified.

**On the canary:** during a deploy or feature rollout, a
canary subset of pods runs with `calls=True` on a subset of
portals, recording continuously.  Storage volume is
manageable; the recording is the *baseline* against which
production behaviour is compared.

**On incident:** when an alert fires, the on-call operator
activates the relevant portals on the affected pods at the
*right granularity for the question*.  Calls-only for "did
the path execute?", lines for "where did it diverge?",
variables for "what was in scope?".  Filter aggressively with
expressions when the question is about a specific request or
customer.

**Never on every pod, every portal, full granularity.**  This
is the regime where the recording's overhead competes with
the application's work.  No production deployment should run
in this regime.

## 8.10 The failure-mode budget

The book has called out three places the recording can fail
under load:

**Pod-side drop on saturation (ch. 3 §3.6).**  TCP backend
writes that fail fall back to reconnect; reconnects that fail
drop bytes.  Recording becomes lossy.  Counter-policy: budget
the recording rate so peak saturation does not happen.

**Pod-side flush loss on crash.**  The 64 KiB buffer's
contents are lost on `SIGKILL` or a Python segfault.  Counter-
policy: smaller buffer (more frequent flushes, more syscall
cost) or a tee to file (the file backend flushes synchronously
and is immune to crash loss within the buffer).

**Server-side flow-control mismatch.**  The server signals
PAUSE/RESUME via out-of-band markers in the stream; the pod's
TCP backend (today) does not honour them.  The pod relies on
drop-on-saturation if it cannot keep up.  Counter-policy:
either teach the pod's backend to honour flow control, or run
with sufficient bandwidth headroom that flow control rarely
fires.

The three failure modes together imply a recording is *best-
effort, not guaranteed*.  An operator who needs a guaranteed
recording — every event, no drops, on every failure path —
needs a different tier (a system-level recorder, an OS-syscall
trace, a deterministic-replay debugger).  The pod's mandate is
*production-friendly post-mortem*; that is a different
contract.

## 8.11 Cost framing for the architect

A team adopting time-travel debugging at scale has three
budget questions:

**CPU budget.**  How much production CPU am I willing to
spend on recording?  Quiescent: <1 %.  Calls on a few
portals: a few percent.  Lines or variables: an order of
magnitude more, only on opt-in scopes.  The architectural
question is: *which portals are always-on, which are
on-demand*?

**Storage budget.**  What is the long-tail per-day rate of
recording, and what retention can the operations team afford?
For a medium deployment with 30-day retention, plan for low
single-digit TB.  The architectural question is: *what is the
default retention, and what is the flagged-recording
escape hatch*?

**Operator budget.**  How much operator attention does the
recording consume?  A team that recordings every incident and
files them away gets nothing back; a team that *reads* their
recordings as part of every postmortem gets debugging insight
that compounds.  The architectural question is: *who reads
the recordings, and what is the workflow*?

The three budgets are not separable.  A team that under-spends
on CPU but over-spends on operator attention ends up
forensically fast but operationally noisy.  A team that
over-spends on CPU but under-spends on operator attention
ends up with hard-drives full of unanalysed traces.  The
right balance is service-by-service.

## 8.12 What the production-grade benchmark suite would do

Honest framing: the numbers in this chapter are *design
estimates* informed by the broader Python literature on
`sys.settrace`, the pod's own architecture, and the volume
math from §8.3.  They are not benchmark numbers from the pod
itself.

The work that would harden these numbers:

- **A microbenchmark** of `_trace_function` on synthetic Python
  code at each granularity, controlling for function-call
  density, line density, and variable-dict size.  Output:
  per-event cost in nanoseconds.
- **An end-to-end benchmark** of a representative service
  (a Flask app, an asyncio worker, an agent) under load, with
  and without each granularity active.  Output: latency
  percentiles and throughput at each granularity.
- **A storage-rate benchmark** measuring bytes-per-second
  produced by each granularity on the same representative
  service.  Output: storage budget for the policy tables.
- **A recovery benchmark** measuring how the deserialiser
  handles partial / corrupted streams (today, §3.7 noted the
  reader breaks at corruption).  Output: durability claims.

These four benchmark suites would replace the estimates in
this chapter with measured numbers.  Until they exist, this
chapter's claims are *design intent supported by analogous
Python tracing literature*, not *measured on the pod*.

The book is honest about the gap because the alternative —
quoting unfounded numbers as facts — would lose the reader's
trust.  Volume 8 ch. 8's contribution is the *framework* for
thinking about cost; the *numbers* are placeholders for the
benchmarks that follow.

## 8.13 The closing recommendation

A team adopting the time-travel pod / server tier should:

- **Declare portals everywhere they expect to need to debug.**
  Names, scopes, expression-friendly variables in scope.
- **Enable the gateway in every long-running service.**
  Quiescent overhead is small; the cost of *not* having the
  capability when the incident fires is large.
- **Default to calls-only, opt-in for lines and variables.**
  On-the-canary continuous calls; on-incident targeted higher
  granularity.
- **Use expressions to filter aggressively.**  An activation
  with `expression="customer_id == 'C-12345'"` is the
  difference between recording every request and recording
  the one that matters.
- **Set a retention policy and stick to it.**  The default 30
  days is reasonable for most teams; longer for compliance,
  shorter for cost-sensitive deployments.
- **Read the recordings.**  The recording is only as
  valuable as the operator's discipline of *reading* it.  A
  postmortem template that includes "did anyone walk the
  recording?" makes the discipline visible.

The pod / server tier is the production-quality answer to *we
couldn't reproduce it*.  The cost is real but bounded; the
discipline is learnable; the payoff is the kind of
debuggability that turns intermittent production failures into
*known-and-fixed* problems.

## 8.14 Volume close

Volume 8 has walked the time-travel debugging tier from its
motivating constraints (ch. 1) through the pod's
instrumentation surface (ch. 2), the wire protocol (ch. 3),
the server's ingestion path (ch. 4), the remote-control plane
(ch. 5), the multi-tenant data and analysis story (ch. 6), the
case-study walkthroughs (ch. 7), and the cost framework
(ch. 8).

The volume's recurring posture: *honest about scaffolding,
honest about gaps, anchored on file:line citations into real
source*.  The pod and the server are the marquee artefact of
the portfolio.  Their architecture is locked; their wire
protocol is in flux (§3.8); their REST API is shape-only
(§6.9); their granularity-update path is incomplete (§5.3);
their analysis tier is real and unit-tested.

The book is what it claims to be: an operational discipline
encoded in a pair of Python packages, documented at the level
of detail an engineer who must extend or operate them needs.

← Back to [`07-case-studies.md`](07-case-studies.md).

← Volume 8 README: [`../README.md`](../README.md).

← Volume index: [`../../../codecraftai-volumes.md`](../../../codecraftai-volumes.md).
