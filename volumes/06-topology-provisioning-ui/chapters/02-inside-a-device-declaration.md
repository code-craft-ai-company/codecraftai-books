# Chapter 2 — Inside a Device Declaration

> *A device entry in a Landscape file has three layers:
> primary access (the deviceType plus its required fields),
> Features (additional capabilities), and declarative metadata
> (platform, skip, group).  Features themselves come in two
> shapes — boolean flags and service references.  This chapter
> walks each layer with code anchors.*

The previous chapter walked the Landscape's two top-level
sections — `apod` and `infrastructure` — and showed how
`LandscapeDevice` instances are created from the declared
devices.  This chapter zooms in on the device declaration
itself.  By the end, every line of a real device YAML has a
named purpose.

## The reference declaration

A real-shape device declaration:

```yaml
- deviceType: network/ssh
  host: 192.168.1.30
  platform: linux
  credentials:
      - casey-node
  features:
      isolation: true
      serial: serial-concentrator: 1
      power: LPC934: 1
  skip: false
```

Six fields plus a Features section.  Three semantic layers:

1. **Primary access** — `deviceType`, `host`, `credentials`.
2. **Features** — `features:` map.
3. **Declarative metadata** — `platform`, `skip`.

(Plus optional `name`, `role`, `group` not shown here.  The
shape is open at the top level; the framework reads what it
recognises and ignores what it doesn't, leaving the rest for
user subclasses to interpret.)

## Layer 1 — Primary access

The `deviceType` field selects the **Coordinator** (chapter 3)
that manages this device.  Examples shipped:

- `network/ssh` → `SshCoordinator` (primary access via SSH).
- `network/client-linux` → `LinuxClientCoordinator`.
- `network/client-windows` → `WindowsClientCoordinator`.
- Project-specific deviceTypes register their own Coordinators
  through the factory protocol from Volume 0 chapter 7.

The Coordinator is what knows how to instantiate the device's
primary ProtocolExtension (chapter 3).  For `network/ssh`,
the Coordinator wires up an `SshAgent` and a `SshProtocolExtension`
that exposes `device.ssh`.

The fields that configure the primary access depend on the
deviceType:

- `host` — the address the Coordinator's primary protocol
  reaches.  IP, DNS name, hostname.
- `credentials:` — composable list of credential names.  The
  first name is the *primary* credential; alternates can be
  used by per-call overrides (Volume 2 chapter 11 walked this).

For an SSH device, `host` and `credentials` are sufficient;
the Coordinator opens an SSH session to `host` using the
first listed credential.  For other deviceTypes, the
Coordinator might require additional fields (a serial device
might want `port`; a Bluetooth device might want
`bdaddr`).  The framework's per-deviceType Coordinator is the
authority on what the primary-access fields are.

## Layer 2 — Features

The `features:` map declares *additional* capabilities beyond
the primary access.  Each key is a Feature name.  Each value
is one of two shapes.

### Boolean capability flags

```yaml
features:
    isolation: true
    has_gpu: false
```

A boolean flag declares that the device has (or doesn't have)
a property the test code or framework can query.  No Agent
gets attached; no ProtocolExtension materialises.  The
`LandscapeDevice` stores the flag in its `_features` dict and
project code can check it:

```python
if device._features.get("isolation"):
    # Apply isolation-specific test logic
    ...
```

The framework ships several boolean flags it recognises by
name (`isolation` is one); projects add their own freely.

A common pattern: a project-specific `LandscapeDevice` subclass
exposes the flag through a property:

```python
class MyProjectDevice(LandscapeDevice):

    @property
    def is_isolated(self) -> bool:
        rtnval = self._features.get("isolation", False)
        return rtnval
```

The property reads a property; no framework changes needed.

### Service-reference Features

```yaml
features:
    serial: serial-concentrator: 1
    power: LPC934: 1
```

A service-reference Feature names a service from the
Landscape's `infrastructure.services` section plus a port (or
other discriminator).  At runtime, the framework resolves the
service, instantiates the matching Agent against the
service's address, and attaches a ProtocolExtension on the
device.

The flow for `serial: serial-concentrator: 1`:

1. The `LandscapeIntegrationLayer` walks the device's
   `_features` dict.
2. It encounters the `serial` Feature with value
   `serial-concentrator: 1`.
3. It looks up `serial-concentrator` in
   `infrastructure.services`.  Finds it.
4. It uses the service's address and the port `1` to
   construct a `SerialAgent`.
5. It wraps the agent in a `SerialProtocolExtension` and
   attaches it to the device:
   `device.attach_extension("serial", serial_ext)`.

After integration, the test calls `device.serial.write(...)`
exactly as it would call `device.ssh.run_cmd(...)`.  Both go
through ProtocolExtensions.  The difference is that
`device.ssh` was wired by the *primary deviceType*'s
Coordinator; `device.serial` was wired by the Feature's
resolution flow.

### How the framework dispatches

Per the research findings on `landscapedevice.py:356–376`,
Feature-driven extension attachment looks like:

```python
# Roughly — the framework's Feature-attachment dispatcher.

def attach_features(self):
    if "power" in self._features:
        power_spec = self._features["power"]
        # ... resolve the service, instantiate the Agent, build the extension
        power_ext = build_power_extension(power_spec)
        self.attach_extension("power", power_ext)

    if "serial" in self._features:
        serial_spec = self._features["serial"]
        serial_ext = build_serial_extension(serial_spec)
        self.attach_extension("serial", serial_ext)

    # ... and so on for built-in Features ...
```

This is **config-driven dispatch**, not decorator-based.
The dispatcher checks `if "power" in self._features` and
acts.  A project-specific Feature is added by subclassing
`LandscapeDevice` and adding a per-Feature attach step:

```python
class MyProjectDevice(LandscapeDevice):

    def attach_features(self):
        super().attach_features()  # built-ins

        if "cooling" in self._features:
            cooling_spec = self._features["cooling"]
            cooling_ext = build_cooling_extension(cooling_spec)
            self.attach_extension("cooling", cooling_ext)
        return
```

The subclass is registered through the factory protocol so
the Coordinator instantiates `MyProjectDevice` instead of the
base `LandscapeDevice`.  Chapter 7 walks the subclassing
mechanism in detail.

### Why two shapes?

Because the Features-tag has to express two different ideas
without bifurcating into two different sections:

- **Capability flags** are *properties of the device that have
  no off-device counterpart*.  "Is the device isolated?" is
  about the device's network position; there's no service to
  point at.
- **Service-reference Features** are *capabilities that come
  from another resource the device is wired to*.  "How do I
  reach this device's serial console?" is answered by a
  service, not by the device itself.

A unified `features:` section that allows both shapes keeps
the YAML compact.  The cost is the dual-form parsing logic in
the dispatcher; the benefit is the readability of declarations
like the reference at the top of this chapter.

## Layer 3 — Declarative metadata

The remaining fields are *neither primary access nor
Features*.  They're metadata the framework or test code can
query.

- `name:` — the device's friendly name (also wrapped in a
  `FriendlyIdentifier` — chapter 4).  If absent, the framework
  derives one from the deviceType + host.
- `role:` — a string identifying the device's role in the
  test.  Used by Topology declarations (chapter 5) and by
  test code (`landscape.devices_by_role("worker")`).
- `platform:` — for OS-aware tests.  RHEL / Ubuntu / macOS /
  Windows / etc.  Useful when a single device's behaviour
  differs across OSes and the test wants to skip on the
  wrong OS.
- `group:` — used by the Topology's `clusters` section
  (chapter 5) to group devices.
- `skip:` — a declarative test-applicability flag at the
  device level.  When true, tests that target this device
  are skipped without running.

The framework's parsing of these fields is mostly literal —
copy from the YAML to the `LandscapeDevice` instance.  Test
code reads them through properties (`device.name`,
`device.platform`, `device.role`).

## What goes where

A summary table:

| Field | Layer | Purpose |
|---|---|---|
| `deviceType` | Primary access | Selects the Coordinator |
| `host`, `port`, `credentials` | Primary access | Configure the primary Coordinator |
| `features:` (boolean) | Features | Capability flags queryable as data |
| `features:` (service-ref) | Features | Materialise into ProtocolExtensions at runtime |
| `name`, `role`, `platform`, `group`, `skip` | Metadata | Read by framework / test code |

The clean separation lets the framework parse without
ambiguity.  The user adding a new field for a project-specific
purpose puts it under `features:` (if it's a capability or a
service-reference) or as top-level metadata (if it's a query
target).  The framework ignores fields it doesn't recognise;
the project's `LandscapeDevice` subclass knows to read them.

## A worked declaration

Bringing it together with annotations:

```yaml
- deviceType: network/ssh           # Primary access type → SshCoordinator
  name: dut-rhel-1                  # Friendly name
  host: 10.0.1.10                   # Primary access address
  role: worker                      # Test-code-queryable role
  platform: rhel                    # OS-aware test discriminator
  credentials:                      # Composable credential list
      - dut-root
      - dut-emergency
  features:
      isolation: false              # Boolean flag — device is NOT isolated
      serial: serial-concentrator: 1   # Service-ref → device.serial
      power: LPC934: 1                 # Service-ref → device.power
      gpu_count: 4                  # Project-specific flag (subclass-handled)
  skip: false                       # Test-applicability flag
```

After the integration layer processes this:

- The framework's `LandscapeDevice` instance carries `name`,
  `host`, `role`, `platform`, `credentials`, `_features`,
  `skip`.
- The `SshCoordinator` has wired `device.ssh` (an
  `SshProtocolExtension`).
- The framework's Feature dispatcher has wired
  `device.serial` and `device.power` (both
  `ProtocolExtension`s).
- A project's `MyProjectDevice` subclass has read
  `_features["gpu_count"]` and exposed it as
  `device.gpu_count`.
- `skip=false` lets tests against this device proceed
  normally.

A test asks the framework for the device by name (or by role),
calls into the ProtocolExtensions, and gets work done.  None
of the test's code knows the difference between the framework's
built-in Features and the project's custom ones; the
ProtocolExtensions look the same.

## What the next chapter does

Chapter 3 walks the four-word vocabulary for the device-control
machinery: Coordinators, Agents, ProtocolExtensions, Features.
Each word has a precise meaning; casual usage is the source of
most reader confusion in the rest of the volume.  By the end
of chapter 3, the layered chain
`device → Coordinator → ProtocolExtension → Agent → service`
is established as the framework's mental model, and chapter
4's discussion of `FriendlyIdentifier` for device discovery
becomes natural.
