# Chapter 8 — Extension Packages — UPnP, Tasker, Bluetooth, CAN

> *The portfolio ships four protocol-extension packages —
> UPnP for service discovery, Tasker for distributed task
> coordination, Bluetooth for BLE work, CAN for vehicle bus
> automation.  Each follows one of two integration patterns:
> landscape-integrated (UPnP, Tasker) or standalone
> (Bluetooth, CAN).  This chapter walks the four with code
> anchors.*

The shipped extension packages cover protocols
`codecraftai-interop` doesn't address directly.  They
demonstrate two distinct integration patterns:

- **Landscape-integrated** (UPnP, Tasker) — the protocol's
  agent inherits from `ProtocolExtension` and is
  instantiated by a Coordinator at landscape activation
  time.  Test code accesses the protocol through
  `device.<protocol>` (the typical Volume 6 pattern).
- **Standalone** (Bluetooth, CAN) — the protocol's
  coordinator is a normal class that test code uses
  directly.  No Landscape integration; the test owns the
  lifecycle.

The chapter is a tour.  Each section walks one package with
the package's class hierarchy, registration mechanism (if
any), and a representative method.

## UPnP — landscape-integrated

**Package:** `codecraftai-interop-ext-upnp`

UPnP (Universal Plug and Play) is a service-discovery
protocol used by media devices, IoT gear, network printers,
and consumer hardware.  The package provides device
discovery, service enumeration, and SOAP action invocation.

### Layout

```
codecraftai-interop-ext-upnp/source/packages/ccai/
├── interop/
│   ├── protocols/upnp/
│   │   ├── model/devices/             # device classification
│   │   ├── mapping/, content/, metadata/
│   │   └── upnpextensionprotocol.py
│   └── manufacturers/bose/upnp/       # vendor-specific helpers
└── landscaping/
    ├── coordinators/upnp/
    └── extensions/protocol/
        └── upnpprotocolextension.py
```

The package straddles two namespaces:

- `ccai.interop.protocols.upnp` — the UPnP protocol
  stack (no landscape dependency).
- `ccai.landscaping.coordinators.upnp` and
  `ccai.landscaping.extensions.protocol` — the
  landscape integration pieces.

### `UpnpProtocolExtension`

**File:**
`codecraftai-interop-ext-upnp/source/packages/ccai/landscaping/extensions/protocol/upnpprotocolextension.py:24–38`

```python
class UpnpProtocolExtension(UpnpRootDevice, ProtocolExtension):
    """
        UPnP agent integrated as a protocol extension for a
        landscape device.
    """

    def __init__(self, device_detail: "UpnpDeviceDetail",
                 presence: Optional["UpnpPresence"] = None):
        UpnpRootDevice.__init__(self, device_detail, presence)
        ProtocolExtension.__init__(self)
        return

    def initialize(self, coord_ref: weakref.ReferenceType,
                   basedevice_ref: weakref.ReferenceType,
                   extid: str, location: str,
                   configinfo: dict, deviceinfo: dict):
        UpnpRootDevice.initialize(self, coord_ref, basedevice_ref,
                                   extid, location, configinfo, deviceinfo)
        ProtocolExtension.initialize(self, coord_ref, basedevice_ref,
                                      extid, location, configinfo,
                                      deviceinfo=deviceinfo)
        return
```

The class uses **dual inheritance**:

- From `UpnpRootDevice` — the UPnP protocol-state root
  (services list, device metadata, action invocation
  endpoints).
- From `ProtocolExtension` — the framework's binding base
  (Volume 6 chapter 3) for attaching to a `LandscapeDevice`.

The `initialize` method calls *both* parents' initialisers.
After initialisation, the object is *both* a fully-formed
UPnP root device *and* a landscape protocol extension.

### Use in a test

Once a device's UPnP extension is attached, test code uses it
through the device:

```python
device = LandscapeSingleton().device("media-renderer-1")

# device.upnp is the UpnpProtocolExtension instance
service = device.upnp.find_service("AVTransport")
service.invoke_action("Play", InstanceID=0, Speed=1)
```

The `find_service` walks the UPnP root's service list; the
`invoke_action` makes a SOAP call via the underlying
protocol stack.  The SOAP call uses Aspects-driven
persistence under the hood — same pattern as the SSH /
REST agents.

## Tasker — distributed task coordination

**Package:** `codecraftai-interop-ext-tasker`

The Tasker protocol is a custom-shaped distributed-task
RPC.  A *tasker controller* manages a network of *tasker
nodes*; each node executes tasks the controller submits.
Used in the portfolio for tests that need to coordinate
work across many hosts in parallel (e.g., applying a
configuration to twelve hosts simultaneously, with
per-host result tracking).

### Layout

```
codecraftai-interop-ext-tasker/source/packages/ccai/
├── interop/protocols/tasker/
│   ├── taskercontroller.py
│   ├── taskerservice.py
│   ├── taskernode.py
│   ├── tasking.py
│   ├── taskingevent.py
│   ├── taskinggroupscope.py
│   └── taskeraspects.py
└── landscaping/clients/taskers/
    └── clientroles.py
```

Three layers of the Tasker stack:

- **Controller** — the orchestrating node.
- **Nodes** — workers that execute tasks.
- **Tasking** — the task descriptor / payload model.

### `TaskerController`

**File:**
`codecraftai-interop-ext-tasker/source/packages/ccai/interop/protocols/tasker/taskercontroller.py:50–75`

```python
class TaskerController:
    """
        The TaskerController object lets you startup and control
        task processing across a collection of clients.
    """

    def __init__(
        self,
        logging_directory: Optional[str] = None,
        summary_progress: Optional[SummaryProgressDelivery] = None,
        aspects: Optional[TaskerAspects] = DEFAULT_TASKER_ASPECTS,
    ):
        self._logging_directory = logging_directory
        self._summary_progress = summary_progress
        self._aspects = aspects

        self._tasker_nodes: List[TaskerNode] = []
        self._network_started = False
        return
```

The controller takes a `TaskerAspects` (a Tasker-specific
Aspects subclass) at construction.  Its job:

- Manage the lifecycle of a network of tasker nodes.
- Distribute `Tasking` objects to the nodes.
- Aggregate results.

A typical flow:

```python
controller = TaskerController(
    logging_directory="results/tasker-logs/",
    aspects=DEFAULT_TASKER_ASPECTS,
)

# Add nodes (real implementations register with the landscape).
for host in fleet:
    controller.add_node(TaskerNode(host=host, ...))

controller.startup()

# Submit a tasking to all nodes.
tasking = Tasking(payload="apply-config", config_data=config)
results = controller.execute_tasking_on_all_nodes(tasking)

# Aggregate.
for node, result in results.items():
    ...

controller.shutdown()
```

### Why Tasker exists

Tasker is the framework's answer to "I need to drive twenty
hosts in parallel from the controller's process" without
spawning twenty SSH sessions.  The trade-off:

- **SSH fan-out** (Volume 2 chapter 11's pattern) is general
  and works against any host.  But it has per-call SSH
  handshake overhead.
- **Tasker** requires a tasker-aware service running on
  each node.  But once running, task submission is cheap and
  results can be aggregated centrally.

Pick Tasker when:

- The fleet is large enough that SSH overhead matters.
- The hosts are under your control (you can install the
  tasker service).
- You need rich result aggregation (per-task status, custom
  payload types).

### Landscape integration

Tasker integrates with landscape similarly to UPnP — a
`TaskerCoordinator` discovers tasker nodes, attaches a
`TaskerProtocolExtension` to each device, and exposes
`device.tasker` for test code to use.

Tasker's protocol-extension class is at
`landscaping/clients/taskers/clientroles.py` and follows the
same dual-inheritance pattern as `UpnpProtocolExtension`.

## Bluetooth — standalone

**Package:** `codecraftai-interop-ext-bluetooth`

Bluetooth Low Energy via the Bumble library.  Not
landscape-integrated; the coordinator manages its own
lifecycle.

### Layout

```
codecraftai-interop-ext-bluetooth/source/packages/ccai/interop/bluetooth/
├── bluetoothcoordinator.py
├── bluetoothdevice.py
├── bluetoothpresence.py
├── bluetoothsearchwindow.py
└── deviceevidence.py
```

### `BluetoothCoordinator`

**File:**
`codecraftai-interop-ext-bluetooth/source/packages/ccai/interop/bluetooth/bluetoothcoordinator.py:10–72`

```python
class BluetoothCoordinator:
    """
        Coordinator for Bluetooth operations using Bumble.

        Manages its own background thread and asyncio loop to
        provide a synchronous API.
    """

    def __init__(self, transport_address="usb:0", device_config=None):
        self.transport_address = transport_address
        self.device_config = device_config or {
            "name": "MojoBluetooth",
            "address": "F0:F1:F2:F3:F4:F5",
        }
        self.device = None
        self.transport = None

        self._loop = None
        self._loop_thread = None
        self._running = False

    def startup(self):
        """Start the coordinator and its background loop."""
        if self._running:
            return

        self._running = True
        self._loop = asyncio.new_event_loop()
        self._loop_thread = threading.Thread(
            target=self._run_loop, args=(self._loop,),
            daemon=True, name="am-bt-coordinator-loop",
        )
        self._loop_thread.start()

        try:
            self.submit_coroutine(self._async_startup())
        except Exception as e:
            self.shutdown()
            raise

    def shutdown(self):
        """Stop the coordinator and its background loop."""
        if self._loop and self._loop.is_running():
            try:
                self.submit_coroutine(self._async_shutdown())
            except Exception as e:
                logger.warning(f"Error during async shutdown: {e}")

            self._loop.call_soon_threadsafe(self._loop.stop)

        if self._loop_thread and self._loop_thread.is_alive():
            self._loop_thread.join(timeout=5.0)
```

The coordinator has its own asyncio loop running in a
background thread.  Tests using Bluetooth call synchronous
methods on the coordinator; the coordinator submits
coroutines to the loop and waits for results.

A few observations worth flagging:

- **No Aspects.**  The coordinator does not consume an
  `Aspects` parameter.  Its persistence behaviours are
  ad-hoc (each method's own retry / timeout choices),
  not framework-uniform.
- **Self-managed lifecycle.**  Tests `startup()` and
  `shutdown()` the coordinator explicitly.
- **No landscape integration.**  Bluetooth devices are
  discovered through `start_scanning()` rather than
  declared in a Landscape file.

### Honest assessment

`BluetoothCoordinator` is the *least integrated* of the four
extension packages.  It works as a standalone helper but
doesn't compose with the rest of the framework:

- It doesn't take Aspects.
- It doesn't surface as `device.bluetooth` on a
  LandscapeDevice (no protocol extension).
- It doesn't participate in landscape verification (chapter
  7 of Volume 2).

A reader using it should treat it as an *external dependency
wrapped for convenience*, not as a fully-integrated
framework component.  Future work might bring it into the
Aspects discipline.

## CAN — standalone

**Package:** `codecraftai-interop-ext-can`

Controller Area Network (CAN) bus messaging via the
`python-can` library.  Like Bluetooth, standalone and
unintegrated.

### Layout

```
codecraftai-interop-ext-can/source/packages/ccai/interop/ext/can/
├── cancoordinator.py
├── candevice.py
├── canpresence.py
├── cansearchwindow.py
└── candeviceevidence.py
```

### `CanCoordinator`

**File:**
`codecraftai-interop-ext-can/source/packages/ccai/interop/ext/can/cancoordinator.py:9–49`

```python
class CanCoordinator:
    """
        Coordinator for CAN bus interactions.
    """

    def __init__(self, channel: str, bustype: str = 'virtual',
                 bitrate: int = 500000):
        """
            :param channel: CAN interface name (e.g. 'can0', 'vcan0').
            :param bustype: Bus type (e.g. 'socketcan', 'virtual').
            :param bitrate: Bitrate of the CAN bus.
        """
        self._channel = channel
        self._bustype = bustype
        self._bitrate = bitrate
        self._bus: Optional[can.BusABC] = None

    def connect(self):
        """Connect to the CAN bus."""
        try:
            self._bus = can.Bus(channel=self._channel,
                                bustype=self._bustype,
                                bitrate=self._bitrate)
        except Exception as e:
            raise

    def disconnect(self):
        """Disconnect from the CAN bus."""
        if self._bus:
            self._bus.shutdown()
            self._bus = None

    def send_message(self, arbitration_id: int,
                     data: Union[List[int], bytes],
                     is_extended_id: bool = False):
        """
            Send a message on the CAN bus.
        """
        if not self._bus:
            raise RuntimeError("Not connected to CAN bus")

        msg = can.Message(arbitration_id=arbitration_id, data=data,
                          is_extended_id=is_extended_id)
        try:
            self._bus.send(msg)
        except can.CanError as e:
            raise
```

A direct wrapper around `python-can`.  Same observations as
Bluetooth:

- No Aspects.
- Self-managed lifecycle.
- No landscape integration.

CAN automation in the portfolio is for vehicle-bus testing
and ECU integration — niche use cases that don't typically
overlap with the rest of the test framework's concerns.  The
package gives those tests a working CAN client without
trying to integrate it deeply with Landscape / Topology.

## The two integration patterns

The four packages divide cleanly into two patterns:

| Package | Pattern | Aspects? | Landscape integration? |
|---|---|---|---|
| UPnP | Landscape-integrated | Yes (via Aspects subclass) | Yes (`UpnpProtocolExtension`) |
| Tasker | Landscape-integrated | Yes (`TaskerAspects`) | Yes |
| Bluetooth | Standalone | No | No |
| CAN | Standalone | No | No |

The pattern choice is pragmatic.  UPnP and Tasker target
*lab gear* that the landscape models (media renderers,
worker hosts).  Bluetooth and CAN target *test rigs* that
sit alongside the landscape (a Bluetooth dongle plugged into
the controller; a CAN bus connected to a test vehicle).

A reader writing a new extension package should ask:

- Will my devices be declared in a Landscape file?  →
  Landscape-integrated.
- Are my devices ad-hoc test-rig hardware that the test
  controls directly?  → Standalone.

## The registration pattern (UPnP example)

For the *landscape-integrated* extensions, registration uses
Python's namespace-package discovery.

### `pyproject.toml`

```toml
[tool.poetry]
name = "codecraftai-interop-ext-upnp"
packages = [{include="ccai", from="source/packages"}]

[tool.poetry.dependencies]
python = ">=3.10,<4.0"
codecraftai-interop = ">=1.0.0,<2.0.0"
codecraftai-landscaping = ">=1.0.0,<2.0.0"
```

The package declares itself under the `ccai` namespace
package.  When installed (via poetry / pip), Python's
implicit namespace-package mechanism makes
`ccai/interop/protocols/upnp/` and
`ccai/landscaping/extensions/protocol/upnpprotocolextension.py`
discoverable as if they were part of the framework.

### Discovery at framework startup

At framework startup, the landscape coordinator scans
installed packages for submodules under
`ccai.landscaping.extensions.protocol`.  When it finds
`UpnpProtocolExtension`, it knows to dispatch UPnP-shaped
device declarations to that extension.

The mechanism is described in Volume 0 chapter 7's factory
discovery walkthrough.  The pattern repeats here: *no
explicit entry-points configuration*; the namespace
package's structure is the registration.

A consequence: installing the UPnP package by adding it to
`pyproject.toml`'s dependency list is *enough* to enable
UPnP support in the framework.  The framework discovers the
extension automatically.

### Standalone packages don't need registration

Bluetooth and CAN bypass the namespace mechanism for
landscape integration (since they have none).  They're just
ordinary Python packages that test code imports directly:

```python
from ccai.interop.bluetooth.bluetoothcoordinator import BluetoothCoordinator

bt = BluetoothCoordinator(transport_address="usb:0")
bt.startup()
# ... use bt ...
bt.shutdown()
```

Registration is moot when there's nothing to register *into*.

## What this chapter set up

By the end of the chapter:

- A reader knows the four shipped extension packages and
  what each one does.
- The two integration patterns (landscape-integrated vs
  standalone) are clear.
- The dual-inheritance pattern (`UpnpProtocolExtension(UpnpRootDevice,
  ProtocolExtension)`) is recognised as the way landscape
  integration works.
- The honest limitations of Bluetooth and CAN (no Aspects,
  no landscape integration) are flagged.

The next chapter — the volume's last — gives the *recipe*
for adding a new protocol extension.  Step-by-step: define
the protocol's agent, define its Aspects subclass,
implement the protocol stack, decide on integration pattern,
register via namespace package.  The recipe synthesises
everything the volume has covered.
