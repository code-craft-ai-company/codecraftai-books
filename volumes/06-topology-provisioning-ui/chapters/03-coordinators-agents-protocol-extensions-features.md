# Chapter 3 — Coordinators, Agents, ProtocolExtensions, Features

> *Four words.  Each has a precise meaning.  Casual usage of
> "extension" or "agent" is the source of most reader
> confusion in distributed automation.  This chapter pins
> down the vocabulary with code anchors and explains the full
> Coordinator → Agent → ProtocolExtension → Device chain.*

The framework's device-control machinery has four named
parts.  Each one does *one thing*; mixing them up makes the
code hard to read and the documentation harder to write.  This
chapter establishes the vocabulary the rest of the volume — and
indeed the rest of the series — uses uniformly.

## The four words

| Term | Role |
|---|---|
| **Coordinator** | Manages a class of devices and attaches Agents to them |
| **Agent** | Speaks a wire protocol |
| **ProtocolExtension** | Binds a specific Agent to a specific Device, exposing the protocol surface |
| **Feature** | A capability declared on a Device in the Landscape file |

The full layering for a service-reference Feature:

```
Landscape file
├── services section
│     └── named service (e.g. serial-concentrator at 10.0.0.50)
└── devices section
      └── device-1
            └── features tag: { serial: serial-concentrator: 1 }
                                ↓ at runtime
              device-1 instance
                ├── managed by Serial Coordinator
                └── .serial → ProtocolExtension
                                └── wraps → Serial Agent
                                              └── connects to serial-concentrator port 1
```

Each layer has a class.  Each class has a job.  Each job is
distinct from the others.  The rest of the chapter walks each
class.

## Coordinators

**File:** `automojo-landscaping/source/packages/automojo/landscaping/coordinatorbase.py:32`

```python
class CoordinatorBase:
    """
        Base class for all coordinators.
    """

    # Abstract methods (line 148–182)
    def activate(self, activation_params):
        raise AbstractMethodError(...)

    def configure_devices(self, landscape):
        raise AbstractMethodError(...)

    def establish_connectivity(self, activation_params):
        raise AbstractMethodError(...)
```

A **Coordinator** owns a *class of devices* and is responsible
for them through the activation lifecycle.  The base class is
abstract; subclasses implement:

- `activate(activation_params)` — bring the coordinator's
  devices online.  Probe each device, attach its
  ProtocolExtensions, populate metadata.
- `configure_devices(landscape)` — read the device
  configurations from the landscape and prepare per-device
  state.
- `establish_connectivity(activation_params)` — verify each
  device responds.  Returns
  `(host, ipaddr, status, stdout, stderr, exc)` tuples per
  device (Volume 2 chapter 6 walked this).

Three Coordinator hierarchies in the framework:

### `ClientCoordinatorBase`

**File:** `automojo-landscaping/source/packages/automojo/landscaping/clients/clientcoordinatorbase.py:55`

For *apod* clients — the runners and controllers.  The
Coordinator manages the framework's own machines, not the
system under test.

### `ServiceCoordinatorBase`

**File:** `automojo-landscaping/source/packages/automojo/landscaping/services/servicecoordinatorbase.py:57`

For *infrastructure services* — the named services from
the Landscape's `services` section.  Per-serviceType
subclasses handle different service types.

### `NodeCoordinatorBase`

**File:** `automojo-landscaping/source/packages/automojo/landscaping/clusters/nodecoordinatorbase.py:57`

For *cluster nodes* — devices that participate in clusters
declared by the Topology (chapter 5).  The Coordinator
materialises `LandscapeDeviceCluster` instances from the
Topology's `clusters` section (Volume 2 chapter 6 walked
the materialisation; chapter 9 of this volume walks the
result).

### Protocol-specific subclasses

Concrete Coordinator classes for specific deviceTypes:

- **`SshCoordinator`** at
  `coordinators/ssh/sshcoordinator.py:53`.  Handles
  `network/ssh` deviceTypes.  Its `establish_connectivity`
  (line 137–156) executes a remote echo command via
  `device.ssh.run_cmd()` and collects results.
- **`SerialCoordinatorBase`** at
  `coordinators/serial/serialcoordinatorbase.py:26`.  Handles
  serial-protocol devices.

Each protocol-specific Coordinator's `create_landscape_device`
method (e.g. `sshcoordinator.py:89–135`) instantiates the
`LandscapeDevice`, wires up the protocol's Agent, and
attaches the matching ProtocolExtension.

## Coordinator couplings — the registration mechanism

A Coordinator subclass doesn't just *exist* — it has to be
*discoverable* by the framework.  The discovery mechanism is
the **CoordinatorCoupling**.

**Example:** `coordinators/ssh/sshcoordinatorcoupling.py:44`

```python
class SshCoordinatorCoupling(CoordinatorCoupling):
    """
        The SshCoordinatorCoupling handle the requirement registration for the SSH coordinator.
    """
    pathbase = "/ssh"
    integration_root: str = "apod"
    integration_section: str = "devices"
    integration_leaf: str = "deviceType"
    integration_class: str = INTEGRATION_CLASS_NETWORK_SSH

    @classmethod
    def attach_to_environment(cls, landscape: "Landscape"):
        """
            This API is called so that the IntegrationCoupling can
            process configuration information.
        """
```

A coupling is **declarative metadata**:

- `integration_root` — `"apod"` or
  `"infrastructure"` (which Landscape section).
- `integration_section` — `"devices"` or `"services"`.
- `integration_leaf` — `"deviceType"` or `"serviceType"`.
- `integration_class` — the string the deviceType / serviceType
  is matched against (`"network/ssh"` for SSH).

When the framework activates the integration layer, it walks
the installed couplings, matches each device's deviceType
against the couplings' `integration_class`, and uses the
matched coupling's `COORDINATOR_TYPE` to instantiate the
right Coordinator.

The coupling is stateless metadata; the Coordinator is
stateful runtime.  One coupling type → one Coordinator type.
A project adding a new deviceType writes a Coupling subclass
and a Coordinator subclass; both are registered through the
factory protocol from Volume 0 chapter 7.

## Agents

The framework's `Agent` term is loose in the source code —
different protocols call them different things.  The
canonical example, `SshAgent` from `automojo-protocol-ssh`
(Volume 5), wraps a per-host SSH connection state and vends
`SshSession` instances tailored to specific tasks (Volume 2
chapter 10 walked the SSH side).

For the purpose of Volume 6's vocabulary:

> *An Agent is the protocol-speaking object.  Given a host
> address and credentials, it knows how to talk the wire
> protocol.*

Agents are *not* directly attached to `LandscapeDevice`
instances.  A ProtocolExtension wraps an Agent; the device
exposes the ProtocolExtension.  This separation lets the
ProtocolExtension expose a *device-level* surface
(`device.ssh.run_cmd`) while the Agent does the actual
protocol work (open SSH session, exchange packets, return
results).

## ProtocolExtensions

**File:** `automojo-landscaping/source/packages/automojo/landscaping/extensions/protocol/protocolextension.py:28`

```python
class ProtocolExtension:
    """
        ... allows for the extension of functionality for a
        :class:`LandscapeDevice` or :class:`LandscapeService`.
        The :class:`ProtocolExtension` is used by the UPnP, SSH
        and Other coordinators to extend landscape devices with
        metadata and device connectivity for the UPnP, SSH and
        Other protocols.
    """

    def __init__(self):
        self._coord_ref = None
        self._extends_ref = None
        self._configinfo = None
        self._extid = None
        self._location = None

    def initialize(self, coord_ref, extends_ref, extid, location, configinfo, **kwargs):
        ...
```

A **ProtocolExtension**:

- Holds a weak reference to its owning Coordinator
  (`_coord_ref`).
- Holds a weak reference to the Device or Service it extends
  (`_extends_ref`).
- Stores the protocol-specific configuration (`_configinfo`),
  the extension's unique identifier (`_extid`), and the
  network location (`_location`).
- Is attached to a Device via `device.attach_extension(name,
  ext)` — the device's `_extensions` dict at line 113 of
  `landscapedevice.py` carries the attachment.

A ProtocolExtension subclass implements the *device-level
methods* the test calls — `run_cmd` on an SSH extension,
`write` and `read` on a serial extension, `cycle` on a
power extension.  Each method is a thin wrapper around the
underlying Agent's protocol calls.

The Coordinator is responsible for *creating* the
ProtocolExtension, *initializing* it with the right Agent and
configuration, and *attaching* it to the Device.  The Device
is responsible for *exposing* the ProtocolExtension as a
device-level attribute (`device.ssh`).

### A `ProtocolExtension` is not a `BaseExt`

A point of vocabulary worth pinning down: `ProtocolExtension`
is **not** the same as `BaseExt` (chapter 8).  They serve
different purposes:

- `ProtocolExtension` extends a `LandscapeDevice` with
  protocol connectivity.  The most common kind of extension.
- `BaseExt` is a service-specific framework-extension class
  (e.g., the vSphere extension model).  It extends a
  service's behaviour, not a device's protocol surface.

Chapter 8 walks `BaseExt` in detail.  When the volume says
"extension" without qualification, the chapter context will
disambiguate, but readers should keep the distinction in mind.

## ControlExtension — the service control point

**File:** `automojo-landscaping/source/packages/automojo/landscaping/extensions/control/controlextension.py:27`

```python
class ControlExtension:
    """
        ... allows for addition of controls to extend the
        functionality of a :class:`LandscapeDevice`.  The
        :class:`ControlExtension` is used by the various services
        to link controls back to services.
    """

    def __init__(self):
        self._service_ref = None
        self._extends_ref = None
        self._control_id = None
        self._control_info = None
        self._extid = None
        self._location = None
```

A **ControlExtension** is the *service-side* of a
service-reference Feature.  Where a ProtocolExtension extends
a Device with protocol connectivity, a ControlExtension links
a Service's control point to the Device that uses it.

Example: a power-control service has a `ControlExtension` per
device-port pair.  The `LPC934` power controller has 16
ports; 16 ControlExtensions, each linking to the device
plugged into that port.  The Device's
`PowerProtocolExtension` holds a back-reference to the
service's ControlExtension; calls to `device.power.cycle()`
go through the protocol extension to the control extension to
the service.

## Features

Per chapter 2, **Features** are *declarations on a device in
the Landscape*.  A Feature is configuration data, not a class.

- A boolean Feature (`isolation: true`) is just a flag in the
  device's `_features` dict.
- A service-reference Feature (`serial: serial-concentrator: 1`)
  triggers the framework's Feature-attachment dispatcher to
  resolve the service, build an Agent, build a
  ProtocolExtension, and attach it to the device.

The dispatch is **config-driven**, not decorator-based.  The
framework's `LandscapeDevice` (or its subclass) checks
`if "serial" in self._features` and acts.  Adding a new
Feature name means subclassing `LandscapeDevice` and adding
a per-Feature attach step (chapter 2 worked an example).

## The full chain — a worked traversal

A test calls `device.serial.write("hello")`.  The chain:

1. **Test code** holds a reference to a `LandscapeDevice`.
2. **`device.serial`** is the `SerialProtocolExtension`
   attached at activation time by the framework's Feature
   dispatcher.
3. **The ProtocolExtension's `write(data)`** method delegates
   to its wrapped Agent.
4. **The Agent** is a `SerialAgent` instance bound to the
   serial concentrator service at the device's declared port.
5. **The Agent's `write` method** issues the protocol-level
   command — opening a TCP connection to the concentrator,
   selecting port 1, writing the bytes.

Going the other direction (the activation flow that built the
chain):

1. **Coordinator activation** — the integration layer's
   per-deviceType Coordinator (e.g., `SshCoordinator` for the
   primary access) starts up.
2. **`create_landscape_device`** — Coordinator instantiates
   `LandscapeDevice` from the YAML config.
3. **Primary ProtocolExtension** — Coordinator wires the
   device's primary protocol extension (`device.ssh`).
4. **Feature dispatcher** — framework walks the device's
   `_features` dict.  For `serial`, it resolves the
   `serial-concentrator` service, builds a `SerialAgent`,
   wraps in a `SerialProtocolExtension`, attaches via
   `device.attach_extension("serial", serial_ext)`.
5. **Operational layer** — `establish_connectivity` verifies
   each ProtocolExtension actually connects.

Two sequences, two directions.  The activation flow builds
the chain at startup; the test-call flow walks the chain at
runtime.  Both use the same chain; the chain is the
contract.

## Disambiguating "extension"

The word "extension" appears in three places in the
portfolio.  Always disambiguate:

- `automojo-extension` is the namespace-package factory
  discovery package (Volume 0 chapter 7).  Has nothing to do
  with Landscape.
- `ProtocolExtension` is the device-level protocol binding
  (this chapter).
- `ControlExtension` is the service-side control-point linkage
  (this chapter).
- `BaseExt` (chapter 8) is the service-specific
  framework-extension model (vSphere example).

When the book says "the extension," it's usually either
ProtocolExtension or BaseExt.  Context makes it clear; when
in doubt, the chapter will disambiguate.

## What the next chapter does

Chapter 4 walks `FriendlyIdentifier` — the framework's
mechanism for naming and resolving devices and services.  The
class supports both regex-based identity matching and
substring-based equality with a non-trivial history (a
pass-4 audit fixed an `__eq__` bug that always returned
True).  The chapter walks the class's design and the
identity-vs-hint distinction with code anchors.
