# Chapter 6 — Landscape Activation

> *Volume 1 chapter 9 carried the landscape boundary at the
> contract level: `TestJob`'s five `fire_*` hooks are no-ops on
> single-host; subclasses give them bodies.  This chapter walks
> the bodies.  When the framework "activates" a landscape, what
> actually happens?  Three layers, each with a job, each with
> failure modes the next layer can't recover.*

`Landscape.activate_configuration()`,
`Landscape.activate_integration()`, and
`Landscape.activate_operations()` are the three method calls
`InteropLibraryTestJob` makes during a distributed test run.
Each delegates to a corresponding *layer* on the `Landscape`
class:

- **`activate_configuration`** delegates to
  `LandscapeConfigurationLayer`.
- **`activate_integration`** delegates to
  `LandscapeIntegrationLayer`.
- **`activate_operations`** delegates to
  `LandscapeOperationalLayer`.

The three layers are the framework's separation of concerns
for landscape activation: load files → instantiate objects →
bring online and verify.  Each layer has a clear responsibility
and a clear failure mode.

## The three layers at a glance

| Layer | Responsibility | Output | Failure mode |
|---|---|---|---|
| Configuration | Load Landscape and Topology files; initialise credentials; let integration couplings inspect config | `landscape_info`, `topology_info`, `CredentialManager` populated | Missing files; schema errors; unresolved credentials |
| Integration | Instantiate device, service, power, serial objects from the loaded config | `_integrated_devices`, `_integrated_services`, etc. populated | Unknown deviceTypes; misconfigured Features; coordinator-init errors |
| Operational | Activate coordinators; attach extensions; verify connectivity | Devices online and reachable | Unreachable devices; failed authentication; service-side errors |

The order matters.  Each layer requires the previous layer's
output.  An attempted operational activation against a
landscape whose integration layer didn't produce devices
fails immediately.  An integration activation against a
landscape whose configuration layer didn't load fails
immediately.  The framework enforces the order through the
`Landscape` class's three methods, each of which checks its
predecessor's state.

## Layer 1 — `LandscapeConfigurationLayer`

**File:** `automojo-landscaping/source/packages/automojo/landscaping/layers/landscapeconfigurationlayer.py:51`

The configuration layer is the "load and validate" phase.
It owns the on-disk representation of the landscape and
topology, plus the credentials and runtime references they
contain.

### Key methods

- **`load_landscape(activation_params)`** — reads the
  `LANDSCAPE_CONFIGURATION_MAP` (populated earlier by
  `automojo-configuration`'s loader) and stores it as
  `self._landscape_info`.
- **`load_topology(activation_params)`** — same for
  `TOPOLOGY_CONFIGURATION_MAP`, stored as
  `self._topology_info`.  Line 267.
- **`initialize_credentials()`** — instantiates
  `CredentialManager` for the loaded credentials.  Anything
  the test job will need to authenticate against must resolve
  here.
- **`attach_to_environment()`** (lines 201–219) — gives every
  installed integration coupling a chance to *inspect* the
  configuration *before* device creation.  Quoted:

  ```python
  def attach_to_environment(self):
      lscape = self.landscape

      # During the landscape initialization and before we start creating devices, give
      # all of the installed coordinator couplings and opportunity to peek at the configuration
      # and perform any special validation or internal initialization, this also provides the
      # integrations with a first glance at the test landscape after the configuration files
      # have been loaded.
      integ_coupling_table = lscape.installed_integration_couplings
      for integ_key in integ_coupling_table:
          integ_coupling = integ_coupling_table[integ_key]
          integ_coupling.attach_to_environment(lscape)

      return
  ```

- **`validate_landscape()`** and **`validate_topology()`** —
  structural validation hooks.  The shipped
  `validate_topology()` (line 536) is a stub returning empty
  errors / warnings; the user overrides it in a `Landscape`
  subclass to enforce project-specific schema.

### What can go wrong here

- **Missing files.**  If
  `AM_CONFIG_LANDSCAPE_NAMES` references a name that
  `ConfigurationLoader` cannot find, load fails before any
  device is created.  This is the cleanest failure to
  diagnose — the path is explicit, the resolver explicitly
  reports what it tried.
- **Schema errors.**  YAML or JSON parse errors land here.
- **Credential resolution failures.**  A credential name in the
  Landscape that doesn't resolve through any configured backend
  fails here.  The test job sees a `ConfigurationError` and
  refuses to proceed.

The configuration layer is *fast*.  Loading is filesystem +
parse + dict-walk.  No network calls.  No device probing.
Failure here means the CI system handed the job
configuration that doesn't even parse — a clear "your
inputs are wrong" signal.

## Layer 2 — `LandscapeIntegrationLayer`

**File:** `automojo-landscaping/source/packages/automojo/landscaping/layers/landscapeintegrationlayer.py:43`

The integration layer is the "instantiate objects" phase.
It walks the loaded `landscape_info` and constructs in-memory
representations of every device, service, power controller,
and serial coordinator the landscape declares.

### Key method

`initialize_landscape(activation_params)` at lines 295–341 is
the layer's entry point.  Its body:

```python
def initialize_landscape(self, activation_params: LandscapeActivationParams):
    lscape = self.landscape

    devices = None
    with lscape.begin_locked_landscape_scope() as locked:

        layer_config = lscape.layer_configuration

        if layer_config.landscape_info is not None:

            if not activation_params.disable_device_activation:
                # Initialize the devices so we know what they are...
                devices = self._initialize_landscape_devices(layer_config)

                if self._power_request_count > 0:
                    self._initialize_landscape_power(layer_config)

                if self._serial_request_count > 0:
                    self._initialize_landscape_serial(layer_config)
            else:
                self.logger.info("LandscapeIntegrationLayer: 'Device Activation' was disabled.")
                devices = {}

            if not activation_params.disable_service_activation:
                services = self._initialize_landscape_services(layer_config)
            else:
                self.logger.info("LandscapeIntegrationLayer: 'Service Activation' was disabled.")
                services = {}

            self._integrated_devices = devices.copy()
            self._integrated_services = services.copy()
```

Three things to notice:

1. **The lock.**  `begin_locked_landscape_scope()` is held
   during initialisation to prevent concurrent activation from
   multiple threads (a real concern with the
   `LandscapeSingleton` pattern).
2. **The activation flags.**  `disable_device_activation` and
   `disable_service_activation` are
   `LandscapeActivationParams` fields letting the test job
   skip device/service initialisation when only the
   configuration layer is needed (e.g., for inspection or
   audit).  The default is to activate both.
3. **The on-demand power and serial init.**  The integration
   layer only initialises power and serial subsystems if at
   least one device requested them (tracked via
   `_power_request_count` and `_serial_request_count`).
   Pods without power-control devices skip the power init
   entirely.

### What gets created

After `initialize_landscape` returns:

- **`integrated_devices`** — a thread-safe dict of `device_name
  → LandscapeDevice` instances.  Each device knows its
  deviceType, host, credentials, Features, and the
  ProtocolExtensions it should expose.
- **`integrated_services`** — same for services.
- **`integrated_power`** — power-controller objects.
- **`integrated_serial`** — serial-coordinator objects.

The objects exist but are not yet *online*.  No SSH session has
been opened.  No serial port has been probed.  The integration
layer is a graph-construction phase; the operational layer is
the first-contact phase.

### What can go wrong here

- **Unknown deviceTypes.**  A Landscape declaring
  `deviceType: network/specialdevice` against a framework that
  has no Coordinator registered for `network/specialdevice`
  fails here.  The Volume 0 chapter 7 factory model is what
  determines whether a deviceType is known; a project adding a
  new deviceType registers a Coordinator subclass through the
  factory protocol.
- **Misconfigured Features.**  A Feature that references a
  service-name that doesn't exist in the `services` section
  fails here.  The integration layer's
  `_initialize_landscape_devices` walks every device's
  Features and looks up service references; a missing service
  is a `ConfigurationError`.
- **Coordinator-init errors.**  A Coordinator subclass's own
  `__init__` failing surfaces here.

## Layer 3 — `LandscapeOperationalLayer`

**File:** `automojo-landscaping/source/packages/automojo/landscaping/layers/landscapeoperationallayer.py:41`

The operational layer is where landscape activation *contacts
the real world*.  It activates each Coordinator (which brings
its devices online), attaches ProtocolExtensions to devices,
and verifies connectivity.

### Key method

`establish_connectivity(activation_params)` at lines 391–480 is
the layer's verification entry point.  Excerpted:

```python
def establish_connectivity(self, activation_params: LandscapeActivationParams):
    lscape = self.landscape
    layer_integ = lscape.layer_integration

    failures = []

    coordinators_for_services = layer_integ.coordinators_for_services
    for coord in coordinators_for_services.values():
        results = coord.establish_connectivity(activation_params)
        for item in results:
            host, ipaddr, status, stdout, stderr, exc = item
            if exc is not None:
                failures.append(item)
            elif status is not None:
                if isinstance(status, bool):
                    if not status:
                        failures.append(item)
                elif status != 0:
                    failures.append(item)

    # ... similar loops for power, serial, devices ...

    if len(failures) > 0:
        errmsg_lines = [
            "Errors encountered while establishing connectivity with the landscape.",
            "FAILURES:"
        ]

        for item in failures:
            host, ipaddr, status, stdout, stderr, exc = item
            if exc is not None:
                errmsg_lines.append(f"    Host: {host} IP: {ipaddr} Exception: {exc}")
            else:
                errmsg_lines.append(f"    Host: {host} IP: {ipaddr} Status: {status} Stderr: {stderr}")

        errmsg = os.linesep.join(errmsg_lines)

        if not activation_params.allow_missing_devices:
            raise RuntimeError(errmsg)
        else:
            self.logger.error(errmsg)

    return
```

Three properties of the verification:

1. **The result tuple shape.**  Each coordinator's
   `establish_connectivity` returns a list of
   `(host, ipaddr, status, stdout, stderr, exc)` tuples.  The
   tuple captures: which host, what IP it resolved to, what
   status code the probe returned, stdout / stderr from the
   probe, and any exception caught.  This is the primary
   forensic record for "why is host X unreachable."
2. **The `allow_missing_devices` flag.**  When False (the
   default), any failure raises `RuntimeError` and the test
   job refuses to proceed.  When True (typical for smoke
   runs), failures are logged and the run continues with
   missing devices.
3. **The error message construction.**  The framework follows
   the Volume 0 chapter 2 multi-line-error-message discipline:
   build a list of lines, format with `os.linesep.join`,
   raise.  An SDET reading the error gets a structured failure
   report with the host, IP, status, and stderr — enough to
   triage without re-running.

### What gets activated

`activate_coordinators(activation_params)` runs *before*
`establish_connectivity`.  For each Coordinator:

- The Coordinator's `activate()` method runs.  This typically:
  - Walks each device the Coordinator manages.
  - Probes the device for additional metadata (model number,
    OS version, hardware info).
  - Attaches the appropriate ProtocolExtensions to the device
    (`device.ssh`, `device.serial`, `device.power`).
- Per-device features that need the Coordinator's bridging
  (e.g., a serial console reachable through a serial concentrator)
  get their ProtocolExtensions wired through the Coordinator's
  registry.

After `activate_coordinators` plus `establish_connectivity`
succeed, the Pod is *operationally live*.  Devices respond to
calls; ProtocolExtensions are attached; service references
resolve; credentials work.

### What can go wrong here

- **Unreachable devices.**  Network down, host crashed,
  firewall blocking, wrong port — all surface as failures in
  the result tuples.
- **Failed authentication.**  Credentials resolved (layer 1),
  ProtocolExtensions attached (above), but the actual SSH
  authentication failed.  Surface as `SshAuthError` in the
  exc field.
- **Timeout exceeded.**  The Coordinator's probe took too long
  — either a long-running connection attempt hung, or the
  device is up but the probe is slow.  Surface as `TimeoutError`.

## The order: why each phase requires the previous

Volume 0 chapter 4's protocol discipline shows up here.  The
three layers are *strictly ordered*:

1. **Configuration** must succeed before Integration (the
   integration layer reads `landscape_info` from the
   configuration layer).
2. **Integration** must succeed before Operational (the
   operational layer iterates the integrated coordinators
   from the integration layer).
3. **Operational** must succeed before tests run (a test
   against a Pod whose devices aren't online and reachable is
   not deterministic — chapter 2's commitment).

The framework enforces the order by checking state at each
layer's entry.  An out-of-order activation surfaces a
`SemanticError` (Volume 0 chapter 3) — the framework refuses
to run the operational layer if integration didn't produce
devices, refuses to run integration if configuration didn't
load, etc.

## Activation parameters

`LandscapeActivationParams`
(`automojo-landscaping/source/packages/automojo/landscaping/landscapeparameters.py:19`)
is the dataclass that controls activation:

```python
class LandscapeActivationParams(SimpleNamespace):
    disable_device_activation: bool=False
    disable_service_activation: bool=False
    allow_missing_devices: bool=False
    allow_unknown_devices: bool=False
    allow_missing_services: bool=False
    allow_unknown_services: bool=False
    validate_features: bool=True
    validate_topology: bool=True
```

Each flag answers a real question:

- `disable_device_activation` — skip device activation
  entirely.  Useful for test jobs that only inspect
  configuration.
- `allow_missing_devices` — proceed even when some devices
  fail to come online.  Useful for smoke runs.
- `validate_topology` — run the topology validator before
  proceeding.  Default `True`; chapter 7 walks the
  verification contract that hangs on this flag.

The default values are *strict*: activation requires every
device and service to come up, every Feature to validate,
every Topology assertion to verify.  A project running a
deliberate smoke / partial run flips the flags explicitly;
the strict defaults ensure no production run accidentally
runs against a half-broken Pod.

## What this means for the verification contract

Chapter 7 walks the verification contract as it appears at
the orchestration / job seam — what the CI system has
promised the job, and how the job verifies the promise.  The
*mechanism* of verification is what this chapter just walked.
Specifically:

- **Landscape verification = `establish_connectivity` returning
  no failures.**  Every device declared in the Landscape can
  be reached at its declared address with its declared
  protocol using its declared credentials.
- **Topology verification = the user's `validate_topology()`
  override returning no errors.**  The shipped stub returns
  empty; a project's `Topology` subclass overrides it to check
  the project-specific Topology shape.

If either fails (and the activation params are strict), the
test job refuses to run.  The Pod is not realised.  Tests do
not start.

## What the next chapter does

Chapter 7 walks the verification contract at the
orchestration-to-job seam.  The CI system says "here are your
resources" (the Landscape) and "here is the configured state"
(the Topology); the test job verifies both before the first
test runs.  This chapter walked the *layers* that do the work;
chapter 7 walks the *contract* the layers honour.
