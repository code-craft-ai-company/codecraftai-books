# Chapter 10 — Remote Objects over a Wire: `codecraftai-python-rpc`

> *Every protocol in this volume moves bytes and hands the
> caller a result.  `codecraftai-python-rpc` moves something
> stranger: a reference.  A client holds an object that lives
> in another process, calls its methods, passes it callbacks,
> and lets it go — and the framework keeps the two processes
> agreeing about which objects still exist.  This chapter is
> the newest package on the volume's floor and the one that
> depends most heavily on everything the earlier chapters
> established.*

Chapters 3 through 8 walked protocols that expose *operations*:
run a command over SSH, resolve a name, issue a REST call, drive
a UPnP device.  The caller assembles arguments, the protocol
carries them, a result comes back.  The object model on each
side is private; only the operation crosses the wire.

`codecraftai-python-rpc` crosses the line those chapters stayed
behind.  It lets a process publish a Python object and lets
another process use that object *as an object* — attribute
access, method calls, callbacks travelling in both directions,
futures for long operations — with the framework responsible
for identity, lifetime and failure semantics in between.  That
is a considerably harder contract than "carry this request",
and the package's design is mostly a catalogue of the ways
the naive version goes wrong.

The chapter reads the package from the wire upward: the frame,
the codec, the session, the proxy, the lease, and finally the
two entry points that a reader actually types.  Throughout, the
question is the same one the volume has been asking since
Chapter 1: *what does the caller measure, and what does the
framework guarantee?*

## What's in the package

```
codecraftai-python-rpc/source/packages/ccai/rpc/
├── __init__.py                 # the public surface
├── aspects.py                  # AspectsRpc, rpc_aspects_scope, remaining_time
├── endpoints.py                # endpoint descriptors and resolution
├── errors.py                   # the RPC exception vocabulary
├── rpcsocketserver.py          # RpcSocketServer
├── rpcsocketclient.py          # RpcSocketClient
├── core/
│   ├── looperqueue.py          # bounded queue with shutdown
│   ├── looper.py               # one worker thread over a queue
│   ├── looperpool.py           # elastic pool of loopers
│   ├── flattenable.py          # by-value contract for user types
│   ├── futures.py              # RemoteFuture, Progress, wait_all, as_completed
│   ├── leases.py               # ILeaseAware
│   ├── protocol/               # constants, 24-byte header, message classes
│   ├── serialization/          # tags, registry, placeholder, codec
│   ├── transport/              # duplex socket I/O, in-process pairs, fault injection
│   ├── session/                # DuplexSessionBase, dispatch, heartbeat, client, server
│   ├── proxy/                  # RemoteProxy, reference table, cache, wrappers
│   ├── dispatch/               # CallContext and ordering lanes
│   └── observability/          # metrics, live-state dump
└── security/                   # authenticators and TLS upgraders
```

About 4,600 lines of package code and 174 tests, including a
cross-process test that starts a real server in a child
process.  The only internal dependency is
`codecraftai-foundations` (Volume 0): the exception vocabulary,
the base `Aspects` class, and `Freezable`.

Two things the tree announces before a line of it is read.
First, there is no `pickle` anywhere.  Second, the transport,
session and proxy layers are separate directories with
separate tests, which is what lets the package test session
semantics over an in-process socket pair rather than a real
network.

## The frame

Everything on the wire is a *frame*: an eight-byte length
prefix, a fixed twenty-four-byte header, and a payload.  The
header is a `struct` with five fields
(`core/protocol/header.py:23–35`):

```python
_HEADER_FORMAT = "!HBB16sI"

@dataclass(frozen=True)
class MessageHeader:
    version: int
    mtype: MessageType
    mcode: int
    correlation_id: uuid.UUID
    payload_length: int
```

`version` is the protocol version, checked on every header
against a supported range (`core/protocol/constants.py:15–16`).
`mtype` is one of *request*, *reply* or *exception*
(`constants.py:22`), and `mcode` says which request or reply
(`constants.py:28–58`).  The sixteen-byte `correlation_id` ties
a reply to the request that caused it.  `payload_length` lets a
reader allocate before it reads.

The header is decoded *before* the payload, and the payload
never has to be parsed to route a message.  That ordering is
the whole point of a fixed header: the session's reader thread
can match a reply to its waiter, or hand a request to the
dispatcher, without touching the codec.  The reader is never
blocked on a slow decode, so a heartbeat reply is seen
promptly even while a large reply is queued behind it.

The message classes in `core/protocol/messages.py` are thin
dataclasses registered by `(type, code)` through a decorator
(`messages.py:90`).  There are twelve requests, twelve
replies and one exception message.  The names are the
vocabulary of the rest of the chapter: `Hello`, `Ping`,
`Close`, `GetService`, `GetAttr`, `SetAttr`, `DelAttr`, `Call`,
`Release`, `Renew`, `Cancel`, `Progress`.

## The codec: a closed set of types, no `pickle`

The temptation in any Python RPC design is `pickle`.  It
handles every type, it round-trips faithfully, and it is a
remote-code-execution primitive by construction — a peer that
can send you a pickle can run code in your process.  The
package refuses it outright and instead ships a tagged binary
codec (`core/serialization/codec.py`) over a *closed* set of
types.

The tags are an `IntEnum` (`core/serialization/tags.py:19`).
The set they cover is deliberately short: `None`, `bool`,
`int` (varint, zigzag for negatives — `codec.py:39–72`),
`float`, `str`, `bytes`, `list`, `tuple`, `dict` with string
keys, `set`, `frozenset`, timezone-aware `datetime`, `date`,
`timedelta`, `UUID`, exceptions, and three extension points:

- **Registered enums** (`registry.py:28`): an enum class maps to
  a string tag, and the value's *name* travels.
- **Registered `Flattenable` types** (`registry.py:37`) and
  **registered dataclasses** (`registry.py:46`): the type maps
  to a tag and its `flatten()` dictionary travels.  A dataclass
  is registered with one call; the codec never introspects an
  unregistered type.
- **References** (`placeholder.py:19`): an object that travels
  *by reference* is replaced by a `ReferencePlaceholder` naming
  the owning session and the reference id.  The proxy layer
  turns placeholders back into proxies on the way in.

Anything outside that set fails *before* a byte is sent, with
`SerializationError` naming the offending type and the path to
it inside the argument tree.  Chapter 2's framing applies
directly: the codec is a *mechanism*, and the mechanism's job
is to fail loudly when asked to do something outside its
contract, never to guess.

Depth is bounded (`tags.py:16`, `MAX_DEPTH = 64`) so a
self-referential structure cannot hang an encoder, and the
maximum message size is a session setting
(`constants.py:19`, sixty-four megabytes by default) enforced
when the length prefix is read, before any allocation.

Exceptions deserve a note.  A remote exception arrives as the
*same type* when the type lives in a preloaded module —
`builtins` and the Volume 0 vocabulary (`codec.py:36`) — with
the remote traceback attached as `remote_traceback`.  Any other
type arrives as `RemoteError` carrying the original type name.
The codec never imports a module on the say-so of a peer.
That single rule is the difference between "an exception
crossed the wire" and "the peer chose which module to import
into my process".

## Work dispatch: queues, loopers and pools

Before the session can be understood, the threading substrate
under it needs a paragraph.  Three small classes in `core/`
carry every unit of work in the package:

- `LooperQueue` (`core/looperqueue.py:30`) — a bounded queue
  with an explicit shutdown sentinel (`looperqueue.py:21`), so a
  worker blocked on `get()` wakes when the queue is closed
  instead of when its timeout happens to expire.
- `Looper` (`core/looper.py:24`) — one named thread draining
  one queue.
- `LooperPool` (`core/looperpool.py:22`) — an elastic set of
  loopers over a shared queue, growing between a minimum and a
  maximum worker count when the queue's depth passes a
  high-water mark.

The pool's bounds come from `AspectsRpc`
(`dispatch_min_workers`, `dispatch_max_workers`,
`dispatch_highwater`).  The choice to make them *Aspects* rather
than constructor arguments is Chapter 2 again: they are
behaviour knobs, and behaviour knobs travel in the Aspects
bundle so a caller who needs different concurrency for one
session sets it in one place.

Every thread the package starts is named
`rpc-<role>-<purpose>-<session>`.  When a process is dumped
with `faulthandler` or inspected in a debugger, the names say
which session and which purpose each thread serves.  Volume 0's
first principle — *debuggability over performance* — is cheap
to honour here and expensive to retrofit.

## The session

`DuplexSessionBase` (`core/session/base.py:103`) is the centre
of the package, and at 660 lines it is the largest file.  A
session owns one transport, one reader thread, one writer
queue, a dispatch pool, a heartbeat monitor and a reference
table.  Client and server sessions
(`core/session/client.py`, `core/session/server.py`) differ
only in who speaks first.

### Request correlation

`send_request` (`base.py:306`) allocates a correlation id,
registers a `Waiter` (`base.py:69`), writes the frame and blocks
on the waiter with a deadline.  The reader thread matches
replies to waiters by correlation id and never blocks on
anything but the socket.  Because requests are correlated
rather than sequenced, a session can have many calls in flight
from many threads, and — this matters for callbacks — a
*request arriving from the peer* can be dispatched while one of
our own requests is still waiting.  That is what lets a server
call back into a client listener during the client's own call.

### Heartbeat and inactivity

`HeartbeatMonitor` (`core/session/heartbeat.py:20`) sends a
`Ping` every `heartbeat_interval` seconds of silence and
declares the peer gone after `inactivity_timeout` seconds
without *any* inbound frame.  Both are Aspects.  The detection
is measured, in the Chapter 1 sense: the session does not
"assume" a peer is alive because a socket is open; it observes
frames and acts on their absence.  When the monitor gives up,
every waiter is failed with `ConnectionLostError` and every
reference the peer held is expired — which brings us to leases.

### Dispatch, deadlines and lanes

Inbound requests go to a `DispatchLooper`
(`core/session/dispatch.py:27`).  Each request is wrapped in a
`CallContext` (`core/dispatch/context.py:25`) carrying the
caller's session, the *remaining time* the caller sent with the
request, the caller's context pairs, and a cancellation flag.
Service code reads these through module functions —
`remaining_time()`, `check_cancelled()`, `report_progress()`
(`context.py:87–107`) — so a service method needs no framework
type in its signature to cooperate with deadlines and
cancellation.

Ordering is the subtle part.  The package has *lanes*
(`core/dispatch/lanes.py:19`): a per-target queue that
serialises deliveries to one object.  Lanes apply to *one-way
notifications only* (`dispatch.py:53–62`).  The comment at that
site records the reason, and it is worth quoting because it is
the kind of decision that only a failing test teaches:

> Ordering lanes apply to one-way callbacks only.  An
> acknowledged call already serialises its caller, and a lane
> on an acknowledged call would deadlock a re-entrant chain
> that targets the same object.

A calls B, B calls back into A's listener, the listener calls B
again: if every delivery to one target waited its turn in a
lane, the chain deadlocks on itself.  Acknowledged calls are
already ordered by the fact that the caller is blocked waiting
for them.  Only fire-and-forget notifications need the lane,
and only for them is it safe.

## Proxies and references

`RemoteProxy` (`core/proxy/remoteproxy.py:56`) is what a caller
holds.  Attribute reads become `GetAttr` requests, writes become
`SetAttr`, method calls become `Call`, and a small list of
dunder methods (`remoteproxy.py:22`) is forwarded so a remote
sequence supports `len()` and indexing.  Access to
underscore-prefixed names is refused at the *owner* unless the
type is decorated `@expose_private`, and writes to the root
service object are refused unless it is `@mutable_service`
(`core/proxy/wrappers.py:114–127`).  Both are opt-in: the
default is the conservative one.

Method access returns a `_RemoteMethod` (`remoteproxy.py:25`)
that carries three verbs.  Calling it is the ordinary
acknowledged call.  `rpc_notify(...)` sends the call one-way
and returns immediately.  `rpc_call_async(...)` returns a
`RemoteFuture`.  A method decorated `@oneway` on the owner's
side (`wrappers.py:106`) is always delivered as a notification
regardless of how the caller invokes it, which is how a service
declares "this method's result is never interesting" once
instead of at every call site.

### By value or by reference — never by guess

An argument crosses the wire in exactly one of two ways, and the
caller must say which.  Types in the codec's closed set travel
by value.  A class decorated `@by_ref` (`wrappers.py:86`)
always travels as a reference: the owner registers the object
in its reference table and sends a placeholder; the holder
receives a proxy.  A caller can also decide per call with the
`ByRef` and `ByValue` wrappers (`wrappers.py:48–84`).  An
unwrapped, unregistered object raises `SerializationError`
before the request is sent.

The design rejects the friendlier alternative — "proxy anything
we can't serialise" — because the friendlier alternative hides
the most expensive decision in the system.  A by-value argument
costs one copy.  A by-reference argument costs a lease, a table
entry in the owner, a proxy in the holder, and one round trip
*per attribute access* for the rest of its life.  Making the
choice explicit at the type or the call site keeps that cost
visible where it is paid.

### The reference table and the cache

The owner's side is a `ReferenceTable` (`core/proxy/table.py`):
reference id to object, plus the set of holders and each
holder's lease expiry.  The holder's side is a proxy cache
(`core/proxy/cache.py`) so two placeholders for one remote
object resolve to one proxy, and so garbage collection of the
proxy can release the reference.

That last sentence hides a race worth recording.  A proxy's
method objects were originally created on each attribute
access and held only a weak path back to the proxy; a caller
that stored `listener.on_event` and dropped `listener` could
see the proxy collected, the reference released, and the stored
method fail on its next call.  The fix is
`RemoteMethodDescriptor` (`wrappers.py:21`) plus a rule that
every derived view — a method, an Aspects view — holds a strong
reference to its base proxy.  A reader who inherits this
package should treat that invariant as load-bearing.

## Leases: agreeing about what exists

Two processes holding references to each other's objects need a
rule for when an object may be freed.  Reference counting
across a network is a known bad idea — a lost `Release` message
leaks the object forever, and a crashed holder never sends one.
The package uses *leases* instead (`specs/leases`), and the
mechanism is small enough to state completely.

- When an owner sends a reference, it records the holder with a
  lease of `lease_duration` seconds (`base.py:424–432`).
- The holder renews every live reference it still holds, in one
  `Renew` request carrying all of the ids, well inside the
  duration.  The owner replies with the ids it no longer knows
  (`base.py:520–522`), so a holder learns promptly that a
  reference is gone rather than at its next call.
- A holder that drops a proxy sends `Release` as a notification
  (`base.py:409–413`).
- The owner sweeps expired leases (`base.py:488`), and an
  expired or released object with an `on_lease_lapsed` method
  (`core/leases.py:22`, `ILeaseAware`) is told so
  (`base.py:461–476`).  A session-level callback,
  `on_lease_lapse` (`base.py:455`), sees every lapse.
- When a peer is declared gone by the heartbeat monitor, every
  lease it held lapses at once.

Two refinements exist because tests found them.  A released
reference leaves a *tombstone* so a late `Renew` for it is
answered "unknown" rather than resurrecting it, and the owner
remembers *recently released* objects briefly so that a
reference released and re-sent within one round trip does not
race its own release.

The effect, for the reader who only ever uses the entry points,
is that a callback object passed to a server simply *works* for
as long as the server keeps it, is freed when the server drops
it or dies, and can ask to be told when that happens.  The
whole apparatus exists so that the user-visible rule can be
that short.

## Aspects, applied to a wire

`AspectsRpc` (`aspects.py:33`) extends the Volume 0 `Aspects`
base with the knobs a wire needs:

| Knob | Default | Governs |
|---|---|---|
| `completion_timeout` | 600 s | how long a call may take |
| `connection_timeout` | 30 s | one connect attempt |
| `max_attempts`, `completion_interval` | 1, 2 s | connect retries and their spacing |
| `heartbeat_interval`, `inactivity_timeout` | 10 s, 60 s | liveness |
| `lease_duration` | 90 s | reference lifetime without renewal |
| `dispatch_min_workers`, `dispatch_max_workers`, `dispatch_highwater` | 2, 8, 4 | the dispatch pool |
| `logging_pattern`, `retry_logging_interval` | failure-only, 5 s | what gets logged, how often |

The bundle is frozen after validation (`aspects.py:38`), and
`replace(**changes)` (`aspects.py:59`) is the way to derive a
variant.  Three scopes apply it:

1. **Per client or server**, in the constructor.
2. **Per proxy**, with `proxy.rpc_with_aspects(aspects)`, which
   returns a *view* sharing the same reference.
3. **Per block**, with `rpc_aspects_scope(aspects)`
   (`aspects.py:124`), a context manager that overrides for
   every call made on the current thread inside it.

The caller's `completion_timeout` does not stay on the caller's
side.  The remaining time travels in the request, the service
reads it with `remaining_time()` (`aspects.py:137` on the
caller, `context.py:87` on the service), and a service that
forwards to a second service can pass on what is left.  This is
the *delegation contract* of Chapter 2 realised end to end: the
caller states the budget once and every hop below it inherits
the budget rather than inventing its own.

## Long operations

`RemoteFuture` (`core/futures.py:40`) is the async form of a
call.  `result(timeout)` blocks; `cancel()` sends a `Cancel`
request that the service observes through `check_cancelled()`;
`add_progress_callback` receives `Progress` records
(`futures.py:24`) that the service emits with
`report_progress(fraction, message)`.  `wait_all` and
`as_completed` (`futures.py:146–155`) fan a set of futures back
in.

Progress and cancellation are ordinary one-way messages in the
protocol (`Progress`, `Cancel`), which is why they work while
the call they concern is still in flight: they share the wire
with it, correlated by the same id, and the dispatcher treats
them as notifications rather than queueing them behind the
call they are trying to influence (`dispatch.py:72`).

## Transport and the fault injector

`Transport` (`core/transport/base.py`) is a small duplex
interface; `DuplexSocketIO` (`core/transport/socketio.py`) is
the real one.  Two details are worth the reader's attention.
First, `close()` *always* releases the file descriptor, even
when the peer closed first — the package once leaked one
descriptor per peer-initiated close, and the fix is a single
`_released` flag.  Second, an in-process socket pair
(`core/transport/inprocess.py`) lets the whole session stack be
tested without a listening port, and `FaultInjectingTransport`
(`core/transport/faults.py:21`) lets tests drop, delay or
truncate frames on demand.

Testing a protocol by injecting faults at the transport seam,
rather than by mocking the session, is the single most
important testing decision in the package.  It is why the
lease, heartbeat and correlation logic have real coverage for
the failure paths that only happen on a bad network.

## The entry points

The two classes a reader types are the smallest part of the
package.  `RpcSocketServer` (`rpcsocketserver.py:33`) accepts
connections, upgrades them if asked, authenticates them if
asked, and gives each one a server session over the same
service object.  `RpcSocketClient`
(`rpcsocketclient.py:32`) connects with the Aspects' retry
budget, performs the `Hello` exchange, and hands back a proxy
typed with the caller's protocol so an IDE sees the methods:

```python
from ccai.rpc import AspectsRpc, RpcSocketClient, RpcSocketServer

server = RpcSocketServer(Greeter(), ("0.0.0.0", 9040))
server.start()

client = RpcSocketClient(GreeterProtocol, ("server-host", 9040),
                         aspects=AspectsRpc(max_attempts=30, completion_interval=2.0))
client.connect()
greeter = client.get_service()
greeter.greet("world", loud=True)
```

Security lives beside the entry points rather than inside the
protocol.  An *authenticator* (`security/auth.py:20`) checks a
credential presented in `Hello`; an *upgrader*
(`security/tls.py:21–37`) wraps the accepted socket in TLS or
mutual TLS before the first frame.  Both are optional callables
handed to the constructors.  The protocol itself never learns
whether it is running over TLS, which is the right amount of
ignorance: the wire discipline is the same either way, and the
security decision is made once at the edge.

Observability follows the same edge principle.
`server.describe()` and `client.session.describe()` return plain
dictionaries; `dump_state()` (`core/observability/state.py:30`)
renders every live server and client in the process; and
`install_dump_on_signal` (`state.py:44`) wires that dump to a
signal so a stuck process can be asked what it is doing without
a debugger.  The `ccai.rpc.wire` logger, at debug level, traces
every frame header.

## Where it sits in the portfolio

The package is the newest arrival on this floor, and its
dependencies say where it belongs: only Volume 0's foundations
below it, and nothing in the volume above it depends on it
yet.  The intended consumers are Volume 2's distributed
sequencer — which today moves results over SSH and JSOS
(Volume 2 ch.10, ch.13) and which a reference-carrying wire
would let hold *live* worker objects rather than streams — and
Volume 7's agent runtime, where a tool that runs in another
process is exactly a remote object with a lease.

For a reader deciding whether to reach for it, the test is the
volume's usual one.  If the interaction is *operations* — run
this, fetch that — one of the protocols in Chapters 3 to 8 is
the smaller tool and should win.  If the interaction is a
*conversation between objects* — callbacks, long-lived
handles, progress, cancellation — the leases and correlation
machinery here are what you would otherwise be writing badly by
hand.

## What this chapter set up

- **The frame**: an eight-byte length, a twenty-four-byte header
  with a correlation id, routing decided before decoding.
- **The codec**: a closed type set, explicit registration for
  user types, references as placeholders, exceptions rebuilt
  only from preloaded modules — and no `pickle`.
- **The session**: correlated requests, a reader that never
  decodes, heartbeat-measured liveness, deadlines and
  cancellation carried in a `CallContext`, lanes for
  notifications only.
- **Proxies**: by-value or by-reference, never by guess;
  conservative defaults for private and mutable access; views
  that keep their base alive.
- **Leases**: the rule a holder sees is one sentence; the
  mechanism is renewals, tombstones and lapse callbacks.
- **Aspects on a wire**: one budget, stated once, inherited by
  every hop.

## End of Volume 5

The chapters that come next are in other volumes.  Volume 4
(*Cross-Platform System Management*) walks the sysman tier
that uses these protocols heavily.  Volume 7 (*Building an AI
Agent SDK*) builds on Volume 5's REST foundations for the
agent's tool-calling layer, and is the first candidate for
the remote-object model this chapter closed on.  Volume 8
(*Time-Travel Debugging in Production*) instruments wire calls
for replay.

Volume 5's contribution to the series is the *protocol
discipline*.  Every other volume that involves cross-machine
communication relies on what was established here:
**Measured Interoperability**, applied to whatever protocol
the situation needs, with the Aspects pattern as the
delegation contract that makes the discipline scale — and,
after this chapter, with a way to let objects themselves cross
the wire when operations are not enough.
