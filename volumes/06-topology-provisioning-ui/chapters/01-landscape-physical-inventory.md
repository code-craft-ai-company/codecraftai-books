# Chapter 1 — Landscape — Physical Inventory and Wiring

> *Volumes 1, 2, and 3 referenced "the Landscape" as input the
> framework had loaded.  This chapter walks what's *in* a
> Landscape file and how `automojo-landscaping` represents
> it.  Two top-level sections: `apod` (the automation
> participants — runners, drivers, controllers) and
> `infrastructure` (services and devices the automation
> operates against).  The shape is intentional, the
> implementation is small, and the user's customisation
> point is clear.*

A Landscape is the answer to the question *what physical
resources exist and how are they connected?*  The Landscape
file is a YAML or JSON document; the loader is in
`automojo-configuration`; the consumer is `automojo-landscaping`.
Volume 3 chapter 5 walked the file's role at the orchestration
layer; this chapter walks what's inside the file and how the
framework's classes represent it once loaded.

This is the first of nine chapters that walk the environment
model end to end.  By chapter 9 a reader will have walked
Landscape, devices, services, the Coordinator/Agent/
ProtocolExtension chain, FriendlyIdentifier, the canonical
Topology shape, the verification overload, the subclassing
mechanism, the BaseExt extension model, and the Pod model.

## The two top-level sections

A real Landscape (the example shipped at
`automojo-automation-kit/userguide/examples/landscape.yaml`)
has the shape:

```yaml
environment:
    label: production

apod:
    clients:
        - deviceType: network/client-linux
          name: casey-01
          host: 192.168.1.21
          role: taskerserver
          credentials:
              - pi-cluster
          features:
              isolation: false
          skip: false
        # ... more clients ...

infrastructure:
    services:
        - serviceType: network/service
          name: someservice
          credentials:
              - adminuser

    devices:
        - deviceType: network/ssh
          name: dut-rhel-1
          host: 10.0.1.10
          platform: rhel
          credentials:
              - dut-root
          features:
              serial: serial-concentrator: 1
              power: lpc934: 1
              isolation: false
          skip: false
        # ... more devices ...
```

Three top-level keys:

- **`environment:`** — a small metadata block with the
  environment's label.  Used in result records to identify
  which environment a run was against.
- **`apod:`** — the **automation pod**, naming the hosts that
  *participate in the automation*.  Runners, controllers,
  drivers.  These are the machines that *do the work*.
- **`infrastructure:`** — the systems the automation
  *operates against*.  Services (databases, vaults, message
  queues) and devices (the systems under test).

## Why `apod` and `infrastructure` are different

Volume 3 chapter 5 walked the operational distinction.  The
short version: only `apod` hosts may be rebooted, reset, or
re-provisioned by the orchestration without disrupting the
automation itself.  Infrastructure hosts are the system under
test — touching them is the test's business, and only when
the test is the actor (chapter 2 of Volume 2's commitment).

The two-section split also explains the etymology of "Pod"
in the framework's vocabulary.  The `apod` is the
*automation pod*; "Pod" (the unit of horizontal CI
parallelism — Volume 2 chapter 3, Volume 3 chapter 27) is
the runtime instance of *one* automation pod plus its
infrastructure plus its topology realised together.  The word
comes from the configuration; the concept gets richer as it
moves through the framework.

## How the Landscape gets loaded

The chain of loading is straightforward:

1. `automojo-configuration`'s `resolve_landscape_configuration`
   reads the named Landscape files (per the
   `AM_CONFIG_LANDSCAPE_NAMES` variable from Volume 0 chapter
   9), merges them via `MergeMap` overlay, and stores the
   result in `LANDSCAPE_CONFIGURATION_MAP`.
2. `LandscapeConfigurationLayer.load_landscape` in
   `automojo-landscaping` reads the merged dict and stores it
   as `landscape_info` (Volume 2 chapter 6 walked the
   activation phase).
3. `LandscapeIntegrationLayer.initialize_landscape` walks the
   `landscape_info` and instantiates `LandscapeDevice` and
   `LandscapeService` objects for each declared device and
   service.

After step 3, the `Landscape` singleton's `devices` and
`services` properties return live objects representing the
declared resources.

## The `Landscape` class

**File:** `automojo-landscaping/source/packages/automojo/landscaping/landscape.py:50`

```python
class Landscape:
    """
        The base class for all derived :class:`Landscape` objects.
        ... transitions through activation stages:

        * Configuration
        * Integration
        * Operational
    """

    this_app = ThisApplicationSingleton()
    context = this_app.context
    logger = getEnhancedLogger()
    landscape_lock = threading.RLock()

    landscape_type = None
    interactive_mode: bool = False
    instance = None
    instance_initialized = False
    configuring_thread_id = None

    _configured_gate: threading.Event = threading.Event()
    _integration_gate: threading.Event = None
    _operational_gate: threading.Event = None
```

Three threading gates (`_configured_gate`,
`_integration_gate`, `_operational_gate`) coordinate
multi-thread activation.  In a distributed test job, the
controller's main thread drives activation, but other threads
(diagnostic captures, validators) might query the landscape
mid-activation.  The gates ensure they wait for the relevant
phase to complete before reading.

The instance's four-layer attribute references:

```python
def __init__(self):
    # ...
    self._layer_install: LandscapeInstallationLayer = None
    self._layer_configuration: LandscapeConfigurationLayer = None
    self._layer_integration: LandscapeIntegrationLayer = None
    self._layer_operational: LandscapeOperationalLayer = None
    self._topology_description: TopologyIntegrationLayer = None
```

Five layer references.  Volume 2 chapter 6 covered three of
them (Configuration / Integration / Operational); the
Installation layer is a fifth that handles installed-coupling
discovery (couplings registered through the factory protocol
from Volume 0 chapter 7), and `_topology_description` is the
topology integration layer that holds the loaded topology.

## The `LandscapeDevice` class

**File:** `automojo-landscaping/source/packages/automojo/landscaping/landscapedevice.py:41`

```python
class LandscapeDevice(FeatureAttachedObject):
    """
        The base class for all landscape devices.  ... provides
        attachment points and mechanisms for adding DeviceExtensions
        to the :class:`LandscapeDevice`.
    """
    FEATURE_TAGS = []

    def __init__(self, lscape: "Landscape", coordinator: "CoordinatorBase",
                 friendly_id: FriendlyIdentifier, device_type: str, device_config: dict):
        super().__init__()
        self._lscape_ref = weakref.ref(lscape)
        self._coord_ref = weakref.ref(coordinator)
        self._friendly_id = friendly_id
        self._device_type = device_type
        self._device_config = device_config
        self._group = ""
        if "group" in device_config:
            self._group = device_config["group"]
        self._device_lock = threading.RLock()
        self._extensions: Dict[str, ProtocolExtension] = {}
        self._features = {}
        if "features" in device_config:
            self._features = device_config["features"]
        self._has_ssh_credential = False
        self._ssh_credential = None
```

Several pieces matter:

- **`FeatureAttachedObject` base.**  This base class (in
  `automojo-testbase`) provides the dispatch mechanism for
  Feature handlers.  A subclass-specific Feature is a method
  that checks `self._features` and acts accordingly.
- **Weak references.**  `_lscape_ref` and `_coord_ref` are
  weak refs to avoid circular-reference garbage-collection
  issues.  The device exists for the duration of the
  landscape's run; the references through `lscape` and
  `coordinator` properties are guaranteed to dereference
  successfully during that window.
- **`_extensions: Dict[str, ProtocolExtension]`.**  The
  per-protocol attachment table.  When a Coordinator attaches
  an SSH ProtocolExtension to the device, it lands in this
  dict at key `"network/ssh"`.  The test's
  `device.ssh.run_cmd` call walks through the dict.
- **`_features = device_config.get("features", {})`.**  The
  features tag from the YAML is stored verbatim.  Feature
  handlers in subclasses query `_features` directly.
- **The credentials story.**  `_has_ssh_credential` and
  `_ssh_credential` are pre-extracted convenience fields for
  SSH-shaped devices; the `_credentials` dict (set later)
  holds all the resolved credentials.  Volume 2 chapter 11
  covered the credentials side of activation.

## The `LandscapeService` class

**File:** `landscapeservice.py:41`

```python
class LandscapeService(FeatureAttachedObject):
    """
        The base class for all landscape services.
    """
    FEATURE_TAGS = []

    def __init__(self, lscape: "Landscape", coordinator: "CoordinatorBase",
                 friendly_id: FriendlyIdentifier, service_type: str, service_config: dict):
        super().__init__()
        self._lscape_ref = weakref.ref(lscape)
        self._coord_ref = weakref.ref(coordinator)
        self._friendly_id = friendly_id
        self._service_type = service_type
        self._service_config = service_config
        self._service_lock = threading.RLock()
```

Smaller than `LandscapeDevice`.  Services are simpler:
they don't have Features the way devices do (most services
just have a connection profile and credentials), and they
don't have the device-specific things (deviceType, host,
platform).  A serviceType identifies what *kind* of service —
`network/service`, `database/postgres`, `messaging/kafka` —
and the framework's per-serviceType `ServiceCoordinatorBase`
subclass knows how to manage it.

## How `apod` becomes `LandscapeDevice` instances

The integration layer's flow:

1. `LandscapeIntegrationLayer.initialize_landscape` reads
   `landscape_info["apod"]["clients"]`.
2. For each client entry, the layer looks up the right
   *Coordinator* by `deviceType` (e.g.,
   `network/client-linux` → `LinuxClientCoordinator`).
3. The Coordinator's `create_landscape_device` instantiates
   a `LandscapeDevice` (or a subclass for the deviceType)
   with the parsed `device_config` dict.
4. The Coordinator attaches initial ProtocolExtensions
   based on the deviceType.
5. The new device lands in
   `_layer_integration._integrated_devices[name]`.

After the loop, `_integrated_devices` holds every apod
client as a live `LandscapeDevice` instance.

## How `infrastructure.devices` becomes `LandscapeDevice`

Same flow as apod clients, but the source is
`landscape_info["infrastructure"]["devices"]` and the
`installed_integration_couplings` (configured through the
factory protocol) determine which Coordinator handles each
deviceType.

The framework supports both `apod` and `infrastructure` devices
in the same activation pass; the difference is *which list
the device appears in*, not *what class of object the device
becomes*.  An apod client and an infrastructure device are
both `LandscapeDevice` instances; what distinguishes them is
operational discipline (the orchestration treats them
differently — see Volume 3 chapter 5).

## How `infrastructure.services` becomes `LandscapeService`

Walks `landscape_info["infrastructure"]["services"]`.  Same
shape: per-serviceType Coordinator instantiates
`LandscapeService` (or subclass) instances.

After the integration phase completes, the `Landscape`
singleton's `services` property returns a dict of these
instances keyed by name.

## What's in the `device_config` dict

For the device YAML earlier in this chapter:

```yaml
- deviceType: network/ssh
  host: 10.0.1.10
  platform: rhel
  credentials:
      - dut-root
  features:
      serial: serial-concentrator: 1
      power: lpc934: 1
      isolation: false
  skip: false
```

The `device_config` dict the framework passes to
`LandscapeDevice.__init__` is essentially this YAML decoded:

```python
device_config = {
    "deviceType": "network/ssh",
    "host": "10.0.1.10",
    "platform": "rhel",
    "credentials": ["dut-root"],
    "features": {
        "serial": "serial-concentrator: 1",
        "power": "lpc934: 1",
        "isolation": False,
    },
    "skip": False,
}
```

The framework parses the dict, extracts `device_config["host"]`,
`device_config["features"]`, etc., and stores them on the
`LandscapeDevice` instance.  Subsequent chapters of this
volume walk what each part means in detail (chapter 2 walks
the device declaration shape; chapter 3 walks the
Coordinator/Agent/ProtocolExtension/Feature vocabulary).

## Composability

Per the Volume 3 chapter 3 universal-composability
property, a Landscape is the merged result of one or more
named files.  An automation campaign typically composes:

```
Landscape names: ["lab-base", "experiment-overrides"]
→ Loader reads lab-base.yaml + experiment-overrides.yaml
→ MergeMap overlay (later names win)
→ Single landscape_info dict
→ LandscapeIntegrationLayer instantiates devices/services
```

The composition is *file-level*, not device-level.  An
override file might add new devices, replace properties on
existing devices (matching by name), or change the services
section.  The merged result is what the integration layer
sees.

This is what makes the four-environment ladder from Volume 3
chapter 13 work for landscapes the same way it works for
runtimes.  Local-dev composes `lab-base + dev-overrides`;
CI composes `lab-base + ci-overrides`; staging composes
`lab-base + staging-overrides`.

## What the next chapter does

Chapter 2 walks inside a single device declaration in detail
— the three layers (primary access, Features, declarative
metadata) and the Feature dual-shape (boolean capability flags
vs service-reference Features that materialise into
ProtocolExtensions at runtime).  The chapter is short and
concrete; it answers *exactly* what the YAML shape means.

Chapter 3 then walks the framework's vocabulary for the
device-control machinery: Coordinators, Agents,
ProtocolExtensions, Features.  The vocabulary is precise; the
chapter's job is to make casual usage of "extension" or
"agent" disambiguate uniformly.
