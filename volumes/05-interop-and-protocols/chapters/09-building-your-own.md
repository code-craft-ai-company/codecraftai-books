# Chapter 9 — Building Your Own Interop Extension

> *Eight chapters established the discipline, the patterns,
> and the worked examples.  This chapter is the *recipe*: the
> reproducible steps for adding a new protocol extension to
> the framework.  Following the recipe gives a project a
> clean, idiomatic, Aspects-driven extension that composes
> with everything else the framework offers.*

This chapter is the volume's payoff.  A reader who's followed
chapters 1–8 has the mental model; this chapter walks the
*action items* for turning that model into a working
extension.

The recipe is organised as a checklist.  Each step has:

- *What* to do.
- *Where* to put the code.
- *Why* — which principle from chapters 1–2 the step
  serves.

By the end, a reader writing a new MQTT, Modbus, or
custom-protocol extension will have a clear plan.

## Before you start — a few decisions

Three questions to answer before writing any code:

### 1. Is this a new protocol or an additional agent shape for an existing protocol?

If the protocol is *new* (no existing agent in the framework),
you're writing a new extension package.

If you're adding *another shape* of an existing protocol
(e.g., a session-pooled SSH variant, a connection-cached
REST variant), you're subclassing an existing agent —
follow the patterns from chapters 3, 4, or 6 directly.

### 2. Landscape-integrated or standalone?

The decision tree from chapter 8:

- Will my devices be declared in Landscape files?  → Landscape-
  integrated.
- Are my devices ad-hoc test-rig hardware?  → Standalone.

Most production protocols are landscape-integrated.
Standalone is fine for niche cases (test rigs, vehicle bus,
specialised hardware).

### 3. What's the protocol's natural call shape?

Some protocols are command-and-response (SSH, serial, CAN
message send).  Others are request / response (REST).
Others are event-driven (Bluetooth scanning, UPnP discovery).

The natural shape determines which Aspects subclass to
extend:

- **Command-shaped** → subclass `AspectsCmd`.  Add a
  `expected_status` analogue if your protocol has one.
- **Request / response** → subclass `AspectsRest` or
  `Aspects` directly.  Add HTTP-status-style allowed-codes.
- **Event-driven** → subclass `Aspects`.  Add timeout
  parameters specific to event-listening (e.g.,
  `discovery_timeout`, `event_buffer_size`).

The chosen shape determines the rest of the recipe.

## The recipe

Eleven steps.  Each maps to a concrete piece of work in the
new package.

### Step 1 — Create the package skeleton

```
automojo-interop-ext-mything/
├── pyproject.toml
└── source/
    └── packages/
        └── automojo/
            ├── interop/
            │   └── protocols/
            │       └── mything/
            │           ├── __init__.py
            │           ├── mythingaspects.py
            │           ├── mythingagent.py
            │           └── mythingsession.py
            └── landscaping/             # only if landscape-integrated
                └── extensions/
                    └── protocol/
                        └── mythingprotocolextension.py
```

The directory structure mirrors the shipped extensions
(chapter 8).  Both subtrees (`interop/...` and
`landscaping/...`) sit under the `automojo` namespace.

The `pyproject.toml`:

```toml
[tool.poetry]
name = "automojo-interop-ext-mything"
version = "0.1.0"
description = "MyThing protocol extension for the automojo framework"
packages = [{include = "automojo", from = "source/packages"}]

[tool.poetry.dependencies]
python = ">=3.10,<4.0"
automojo-interop = { git = "https://github.com/automationmojo/automojo-interop.git",
                     branch = "main" }
automojo-landscaping = { git = "https://github.com/automationmojo/automojo-landscaping.git",
                         branch = "main" }
# ... your protocol's third-party dependencies ...
```

The two `automojo-*` git dependencies are required for
landscape-integrated extensions.  Standalone extensions can
drop the `automojo-landscaping` dependency.

**Why this step:** the namespace structure is what makes the
framework discover your extension at startup (chapter 8's
"registration pattern").  No `pyproject.toml` magic
required — just put the code in the right namespace.

### Step 2 — Define the Aspects subclass

In `mythingaspects.py`:

```python
from typing import Optional, Tuple
from automojo.interop.aspects import Aspects, AspectsCmd, ActionPattern, LoggingPattern
from automojo.interop.aspects import (
    DEFAULT_COMPLETION_TIMEOUT, DEFAULT_COMPLETION_INTERVAL,
    DEFAULT_CONNECTION_TIMEOUT, DEFAULT_CONNECTION_INTERVAL,
    # ... etc
)


# MyThing-specific defaults — tune to your protocol's typical
# behaviour.
DEFAULT_MYTHING_RESPONSE_TIMEOUT = 30
DEFAULT_MYTHING_HEARTBEAT_INTERVAL = 5


class MyThingAspects(Aspects):
    """
        Aspects subclass for MyThing protocol calls.

        Adds protocol-specific fields:
        - response_timeout: time to wait for a response after sending
        - heartbeat_interval: how often to send heartbeats during long calls
    """

    def __init__(self, *,
                 response_timeout: float = DEFAULT_MYTHING_RESPONSE_TIMEOUT,
                 heartbeat_interval: float = DEFAULT_MYTHING_HEARTBEAT_INTERVAL,
                 # ... pass through the standard Aspects parameters ...
                 **kwargs):
        super().__init__(**kwargs)
        self.response_timeout = response_timeout
        self.heartbeat_interval = heartbeat_interval
        # ... self.freeze() if subclassing Aspects directly ...
```

Three things to get right:

- **Inherit from the right base.**  Command-shaped → extend
  `AspectsCmd`.  Other → extend `Aspects` directly.
- **Add only protocol-specific fields.**  Don't redeclare
  fields the base already has.  Don't override defaults
  unless your protocol genuinely needs different ones.
- **Pass through `**kwargs` to the base class's `__init__`.**
  Callers should be able to set the standard Aspects fields
  (`completion_timeout`, `action_pattern`, etc.) without
  fighting the subclass's signature.

A `DEFAULT_MYTHING_ASPECTS = MyThingAspects()` constant at
module level is conventional.

**Why this step:** principle 2 of Measured Interoperability
(chapter 1) — the application layer specifies behaviour.
The Aspects subclass is *the* delegation contract for your
protocol.

### Step 3 — Implement the agent base

In `mythingagent.py`:

```python
from typing import Optional, Tuple
from automojo.interop.aspects import ActionPattern
from automojo.xmodules.xlogging import getEnhancedLogger
from .mythingaspects import MyThingAspects, DEFAULT_MYTHING_ASPECTS


class MyThingAgent:
    """
        Agent for the MyThing protocol.

        Vends sessions; supports direct one-shot calls; honours the
        chosen ActionPattern for retry / persistence.
    """

    def __init__(self,
                 host: str,
                 credential,
                 port: int = 12345,
                 aspects: MyThingAspects = DEFAULT_MYTHING_ASPECTS):
        self._host = host
        self._credential = credential
        self._port = port
        self._aspects = aspects
        self._logger = getEnhancedLogger()
        return

    def send_message(self, message: bytes,
                     aspects: Optional[MyThingAspects] = None
                     ) -> bytes:
        """
            Send a message; receive the response.

            Honours the action_pattern from aspects.
        """
        if aspects is None:
            aspects = self._aspects

        # Dispatch on action pattern.
        if aspects.action_pattern == ActionPattern.SINGLE_CALL:
            return self._do_single_call(message, aspects)
        elif aspects.action_pattern == ActionPattern.DO_UNTIL_SUCCESS:
            return self._do_until_success(message, aspects)
        else:
            err_msg = f"Unsupported action pattern: {aspects.action_pattern}"
            raise ConfigurationError(err_msg)
```

The agent's job (per chapter 3's discussion of agents):

- Take an `Aspects` parameter on every public method.
- Dispatch on `action_pattern`.
- Honour the timing / retry / logging parameters.
- Wrap calls in `MonitoredScope` for deadlock detection.
- Raise typed exceptions when the contract is exceeded.

The implementation of `_do_single_call`, `_do_until_success`,
etc. follows the template from `LocalCommandAgent` (chapter
3) or `SshBase` (chapter 4).  Copy the structure; replace the
underlying mechanism.

**Why this step:** principle 1 of Measured Interoperability
— persistence behaviours live in the interop layer.  The
agent is *where* the persistence behaviours live.

### Step 4 — Define typed exceptions

```python
# In mythingexceptions.py (or alongside the agent)

class MyThingError(Exception):
    """
        Base class for MyThing-protocol-specific errors.
    """
    pass


class MyThingAuthError(MyThingError):
    """
        Authentication or authorisation failed.
    """
    pass


class MyThingProtocolError(MyThingError):
    """
        Protocol violation (malformed message, unexpected
        sequence, etc.).
    """
    pass
```

The agent's persistence loops raise these on contract
exceeded.  The application layer catches them per the
typed-dispatch pattern from chapter 1.

For timeouts, the convention is to raise the standard
Python `TimeoutError` rather than a custom subclass —
consistent with `SshBase` and `RestClientBase`.  For
connection failures, raise `ConnectionError`.

**Why this step:** principle 4 of Measured Interoperability
— typed exceptions revert control to the application layer
when the contract is exceeded.  Your typed exceptions are
the vocabulary for that reversion.

### Step 5 — Implement the session (if stateful)

If your protocol has a *connected* state worth preserving
across calls (like SSH session reuse), implement a session
class as a context manager:

```python
class MyThingSession:
    """
        Stateful MyThing session.  Holds the connection open
        across calls.  Used as a context manager.
    """

    def __init__(self, agent: "MyThingAgent",
                 aspects: MyThingAspects = None):
        self._agent = agent
        self._aspects = aspects or agent._aspects
        self._connection = None

    def __enter__(self):
        self._connection = self._open_connection(self._agent._host,
                                                 self._agent._port,
                                                 self._aspects)
        return self

    def __exit__(self, ex_type, ex_val, ex_tb) -> bool:
        if self._connection is not None:
            self._connection.close()
            self._connection = None
        return False

    def send_message(self, message: bytes,
                     aspects: Optional[MyThingAspects] = None
                     ) -> bytes:
        """
            Send via the open connection.  Cheaper than agent.send_message.
        """
        # ... same dispatch as agent.send_message, but reusing self._connection ...
```

If your protocol is stateless (every call opens / closes its
own connection — like REST), skip this step.  The agent's
methods are sufficient.

**Why this step:** matches the pattern from chapter 4
(`SshAgent` stateless + `SshSession` stateful).  Some
protocols benefit from the duality; others don't need it.

### Step 6 — Decide on integration: landscape-integrated or standalone

If standalone, skip to step 9 (write the test scaffolding).

If landscape-integrated, continue with steps 7 and 8.

### Step 7 — Implement the ProtocolExtension subclass

In `landscaping/extensions/protocol/mythingprotocolextension.py`:

```python
import weakref
from typing import Optional
from automojo.landscaping.extensions.protocol.protocolextension import ProtocolExtension


class MyThingProtocolExtension(ProtocolExtension):
    """
        MyThing protocol exposed as a landscape device extension.
    """

    def __init__(self, agent: "MyThingAgent"):
        super().__init__()
        self._agent = agent
        return

    def initialize(self, coord_ref: weakref.ReferenceType,
                   extends_ref: weakref.ReferenceType,
                   extid: str, location: str,
                   configinfo: dict, **kwargs) -> None:
        super().initialize(coord_ref, extends_ref, extid, location, configinfo, **kwargs)
        return

    def send_message(self, message: bytes,
                     aspects: Optional["MyThingAspects"] = None) -> bytes:
        """
            Forward to the underlying agent.
        """
        return self._agent.send_message(message, aspects=aspects)
```

Two patterns to choose between:

- **Composition** (above) — the extension *holds* an agent
  and forwards calls.  Simpler.
- **Dual inheritance** (chapter 8's UPnP example) — the
  extension *inherits from* both the protocol's root
  class and `ProtocolExtension`.  More integrated but more
  complex.

Composition is the right starting point for most cases.
Dual inheritance is for protocols where the extension's
interface needs to be the protocol's full root surface (UPnP
needs to expose all the UPnP root device methods, not just a
handful of forwards).

**Why this step:** Volume 6 chapter 3's vocabulary —
ProtocolExtension binds a protocol Agent to a Device.  Your
extension is what makes `device.mything` work in test code.

### Step 8 — Implement the Coordinator

In `landscaping/coordinators/mything/mythingcoordinator.py`:

```python
from automojo.landscaping.coordinatorbase import CoordinatorBase


class MyThingCoordinator(CoordinatorBase):
    """
        Coordinator for MyThing devices.
    """

    def create_landscape_device(self, device_config: dict,
                                friendly_id) -> "LandscapeDevice":
        """
            Instantiate a LandscapeDevice with a MyThingProtocolExtension
            attached.
        """
        # ... construct the device ...
        # ... construct the agent ...
        # ... wrap in MyThingProtocolExtension ...
        # ... attach ...
        return device

    def establish_connectivity(self, activation_params) -> List[Tuple]:
        """
            Verify reachability to all managed devices.

            Returns list of (host, ipaddr, status, stdout, stderr, exc) tuples.
        """
        results = []
        for device in self.children.values():
            try:
                # Test the device.
                self._probe_device(device)
                results.append((device.host, device.ipaddr, 0, "", "", None))
            except Exception as ex:
                results.append((device.host, device.ipaddr, None, "", "", ex))
        return results
```

The Coordinator is the framework's hook for activating your
protocol's devices during landscape activation (Volume 2
chapter 6).  Your Coordinator:

- Subclasses the appropriate `CoordinatorBase` variant
  (`ClientCoordinatorBase` for apod clients;
  `ServiceCoordinatorBase` for infrastructure services).
- Creates `LandscapeDevice` instances when called.
- Implements `establish_connectivity` returning the standard
  result-tuple format.

**Why this step:** the Coordinator is what makes your
extension *visible* to the framework's activation flow.
Without it, the framework wouldn't know how to bring your
devices online.

### Step 9 — Register a CoordinatorCoupling

The CoordinatorCoupling is the declarative metadata that maps
deviceTypes to Coordinators.  In
`landscaping/coordinators/mything/mythingcoordinatorcoupling.py`:

```python
from automojo.landscaping.coupling.coordinatorcoupling import CoordinatorCoupling
from .mythingcoordinator import MyThingCoordinator


INTEGRATION_CLASS_MYTHING = "network/mything"


class MyThingCoordinatorCoupling(CoordinatorCoupling):
    """
        Registers MyThingCoordinator as the handler for
        deviceType 'network/mything'.
    """

    pathbase = "/mything"
    integration_root: str = "infrastructure"   # or "apod"
    integration_section: str = "devices"
    integration_leaf: str = "deviceType"
    integration_class: str = INTEGRATION_CLASS_MYTHING
    COORDINATOR_TYPE = MyThingCoordinator
```

When the framework activates a Landscape, it walks every
device's `deviceType`, looks up the matching Coupling by
`integration_class`, and instantiates the
`COORDINATOR_TYPE`.  A device declared as
`deviceType: network/mything` automatically routes to your
Coordinator.

**Why this step:** Volume 6 chapter 3's coupling-vs-
coordinator distinction.  The Coupling is the registration
metadata; the Coordinator is the runtime manager.

### Step 10 — Tests

Three test layers:

- **Unit tests** for the protocol logic.  Mock the wire;
  verify the message construction, parsing, error handling.
- **Aspects tests** verifying that each `ActionPattern` mode
  works.  Mock the wire to fail-then-succeed; verify
  `DO_UNTIL_SUCCESS` retries.  Verify timeouts raise.
- **Integration tests** with a real or fake server.  Run the
  full agent against a mock service; verify protocol-level
  correctness.

Place tests under `source/tests/automojo_tests/`:

```
source/tests/automojo_tests/
└── interop/
    └── protocols/
        └── mything/
            ├── test_aspects.py
            ├── test_agent.py
            └── test_session.py
```

The test discipline mirrors the rest of the framework
(Volume 1 chapter 2's testbase tests).

**Why this step:** consistency with the rest of the
portfolio's test discipline (Volumes 0–4).  An extension
that ships without these tests doesn't meet the framework's
bar.

### Step 11 — Documentation

Add a `userguide/packages/automojo-interop-ext-mything.rst`
file in the docs repo (or alongside the package's README)
that walks:

- What the protocol is.
- The class hierarchy.
- A representative call.
- The Aspects defaults.
- Limitations and caveats.

The shipped extensions' userguide pages (e.g.,
`automojo-docs/userguide/packages/automojo-interop.rst`) are
the reference shape.

**Why this step:** the framework's "Measured
Interoperability" name only works if the *Behaviour* side is
documented for the protocol's specific concerns.  A reader
can't understand your extension's persistence model from the
code alone.

## A worked tour of the recipe

A concrete example.  Suppose you're adding MQTT support to
the framework.

### Step-by-step

1. **Package skeleton** — `automojo-interop-ext-mqtt/` with
   the standard structure.
2. **Aspects** — `MqttAspects(Aspects)` with
   `qos_level` (0/1/2), `keep_alive_interval`, the standard
   timeouts.
3. **Agent** — `MqttAgent` wrapping `paho-mqtt`'s client.
   `publish(topic, payload, aspects)` and
   `subscribe(topic, callback, aspects)` are the primary
   methods.
4. **Exceptions** — `MqttError`, `MqttAuthError`,
   `MqttProtocolError`.
5. **Session** — `MqttSession` for tests that publish to
   many topics in sequence (saves connection setup).
6. **Decision** — landscape-integrated.  MQTT brokers are
   declared as services in Landscape files.
7. **ProtocolExtension** — composition pattern; forward
   to the agent.
8. **Coordinator** — `MqttCoordinator(ServiceCoordinatorBase)`;
   establish_connectivity opens a test connection to verify
   the broker.
9. **Coupling** — `MqttCoordinatorCoupling` with
   `integration_class = "messaging/mqtt"`.
10. **Tests** — happy path, retry on disconnect, timeout
    handling, QoS-level semantics.
11. **Documentation** — `automojo-interop-ext-mqtt.rst` with
    the protocol overview and API examples.

A test using the new extension:

```python
broker = LandscapeSingleton().service("mqtt-broker-1")

# Publish a message with QoS 1, retry until acknowledged.
patient_aspects = MqttAspects(
    action_pattern=ActionPattern.DO_UNTIL_SUCCESS,
    completion_timeout=30,
    qos_level=1,
)
broker.mqtt.publish("sensors/temperature", b"42.0", aspects=patient_aspects)
```

The test reads naturally because the recipe was followed.
`broker.mqtt` is the ProtocolExtension; `publish` is the
agent's method; `aspects` is the delegation contract.

## Common pitfalls

A short list of mistakes that surface when writing a new
extension:

### Decorating with retry rather than passing aspects

Wrong:

```python
@retry(times=3, delay=10)
def send_message(self, message: bytes) -> bytes:
    # ...
```

Right:

```python
def send_message(self, message: bytes,
                 aspects: Optional[MyThingAspects] = None) -> bytes:
    if aspects is None:
        aspects = self._aspects
    # Dispatch on aspects.action_pattern
    # ...
```

The decorator hard-codes the retry policy.  The Aspects
parameter delegates the policy to the caller.  Volume 0
chapter 2's discipline.

### Mutating Aspects after construction

Wrong:

```python
aspects = MyThingAspects()
aspects.completion_timeout = 30  # AttributeError if Freezable
```

Right:

```python
aspects = MyThingAspects(completion_timeout=30)
```

If your Aspects subclass inherits from `Freezable` (it
should), the mutation raises.  Construct a new instance.

### Catching exceptions and silently retrying

Wrong:

```python
try:
    response = self._send(message)
except Exception:
    # Log and pretend success.
    pass
```

Right:

```python
try:
    response = self._send(message)
except (TimeoutError, ConnectionError):
    # If the action_pattern says retry, retry.  Otherwise re-raise.
    if aspects.action_pattern == ActionPattern.DO_UNTIL_SUCCESS:
        # ... retry loop ...
    else:
        raise
```

Principle 4 — control reverts via exception when the
contract's bounds are exceeded.  Silently swallowing
exceptions hides bugs.

### Skipping the typed exceptions

Generic `RuntimeError` is a code smell in interop code.
Define typed exceptions for each meaningful failure mode so
application code can dispatch on the type.

### Skipping the integration tests

Unit tests that mock the wire don't catch *protocol*-level
bugs.  An integration test against a real or fake server is
the only way to verify the protocol implementation is correct
end-to-end.

## What this volume covered

Looking back across the volume:

- **Chapter 1** — Measured Interoperability and the four
  principles.
- **Chapter 2** — the Aspects pattern as the delegation
  contract.
- **Chapter 3** — `automojo-interop` as the foundation.
- **Chapter 4** — SSH from scratch as a worked deep-dive.
- **Chapter 5** — networking primitives below the protocol
  layer.
- **Chapter 6** — REST foundations.
- **Chapter 7** — auto-generation at scale (the GitHub
  client).
- **Chapter 8** — four shipped extension packages and two
  integration patterns.
- **Chapter 9** — the recipe for adding your own.

A reader who's followed the volume now has:

- The vocabulary (Mechanism vs Behaviour, persistence
  behaviours, ActionPattern, LoggingPattern).
- The discipline (the four principles).
- The shipped agents (`LocalCommandAgent`, `SshAgent`,
  `RestClientBase`, the four extensions).
- The pattern for adding more.

The protocol stack is no longer a black box.

## End of Volume 5

The chapters that come next are in other volumes.  Volume 4
(*Cross-Platform System Management*) walks the sysman tier
that uses these protocols heavily.  Volume 7 (*Building an AI
Agent SDK*) builds on Volume 5's REST foundations for the
agent's tool-calling layer.  Volume 8 (*Time-Travel Debugging
in Production*) instruments wire calls for replay.

Volume 5's contribution to the series is the *protocol
discipline*.  Every other volume that involves cross-machine
communication relies on what was established here:
**Measured Interoperability**, applied to whatever protocol
the situation needs, with the Aspects pattern as the
delegation contract that makes the discipline scale.
