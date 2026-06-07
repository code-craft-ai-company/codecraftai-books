# Chapter 7 — Subclassing `Landscape` and `Topology` — The Customisation Mechanism

> *Volume 0 chapter 1's meta-thesis: framework supplies the
> skeleton and patterns; the organisation customises and
> refines.  This chapter walks the *concrete mechanism* for the
> environment model: subclass `Landscape` (and possibly
> `Topology` and `LandscapeDevice`), register through the
> factory protocol, plug into the user's `TestJob` subclass.
> One mechanism, many extension points.*

The previous chapters set up everything a project needs to
know about *what* the framework provides.  This chapter walks
*how the project's customisations land in the framework's
machinery*.

The mechanism: subclassing.  Three subclass points the
project typically uses, and one factory protocol that
registers them so the framework picks them up.

## What gets subclassed

Three classes most projects subclass:

### `Landscape`

The project's Landscape subclass adds:

- Recognition of project-specific Feature names (the Feature
  dispatcher from chapter 2).
- A `validate_topology()` override (chapter 6).
- Project-specific properties that surface configuration
  data to test code (`landscape.region`, `landscape.tier`,
  etc.).
- Project-specific accessors that interpret user-defined
  Topology sections (`landscape.topology.dependencies`,
  `landscape.topology.regions`).

### `LandscapeDevice` (and per-deviceType subclasses)

The project's per-deviceType `LandscapeDevice` subclasses
add:

- Per-Feature attach steps for project-specific Features.
- Project-specific properties exposing Feature data
  (`device.gpu_count`, `device.cooling_loop`).
- Per-deviceType helper methods that wrap common operations
  (`device.cluster_membership_query()` from chapter 6's
  worked example).

### `Topology` (when the framework's defaults aren't enough)

If the framework ships a `Topology` base class with parsing
behaviour (and as of writing, the consumption is in
`LandscapeOperationalLayer._create_clusters`, not in a
`Topology` class per se — the architecture is evolving), a
project subclasses to add user-defined-section parsing.

In current source, the project's customisation entry for
Topology is the `validate_topology` override on the
`Landscape` subclass plus accessor methods on the
`Landscape` subclass that read `_topology_info` for
project-specific sections.  An emerging `Topology` class with
explicit parsing methods would be a natural extension point.

## How subclasses get registered

The mechanism is the **factory protocol** from Volume 0
chapter 7.  Briefly:

1. The framework defines a *factory protocol* — an interface
   (typically an `ccai.factories.protocols.<Something>FactoryProtocol`
   class) declaring the methods that produce framework
   objects.
2. The project provides an *implementation* — a class that
   inherits from the framework's `FactoryExtension` base and
   implements the protocol's methods to return the project's
   subclasses.
3. The project's implementation is registered through a
   *namespace package entry point* (the
   `ccai.factories` namespace in the project's
   `pyproject.toml` or equivalent).
4. At framework startup,
   `ConfiguredSuperFactorySingleton()` discovers the
   project's implementation by walking the namespace package.

For the Landscape, the relevant protocol is
`LandscapingFactoryProtocol`:

```python
# codecraftai-landscaping/source/packages/ccai/landscaping/landscapingfactoryprotocol.py
# (approximate — actual class name from the singleton lookup)

class LandscapingFactoryProtocol:

    @classmethod
    def get_landscape_type(cls) -> Type[Landscape]:
        """
            Return the Landscape class to instantiate for this project.
        """
        return Landscape  # default
```

A project's implementation:

```python
# my_project/factories.py

from ccai.factories.factoryextension import FactoryExtension
from ccai.landscaping.landscapingfactoryprotocol import LandscapingFactoryProtocol

from my_project.landscape import MyProjectLandscape


class MyProjectLandscapingFactory(FactoryExtension, LandscapingFactoryProtocol):

    PRECEDENCE = 30

    @classmethod
    def get_landscape_type(cls) -> Type[Landscape]:
        return MyProjectLandscape
```

The `PRECEDENCE = 30` is the priority; multiple registered
implementations are sorted by precedence.  Higher precedence
wins.

The framework's `LandscapeSingleton()` function (chapter 1):

```python
# codecraftai-landscaping/source/packages/ccai/landscaping/wellknown.py:28

def LandscapeSingleton() -> "Landscape":
    global LANDSCAPE_SINGLETON

    if LANDSCAPE_SINGLETON is None:
        super_factory = ConfiguredSuperFactorySingleton()

        SINGLETON_LOCK.acquire()
        try:
            from ccai.landscaping.landscapingfactoryprotocol import LandscapingFactoryProtocol

            if LANDSCAPE_SINGLETON is None:
                LandscapeType = super_factory.get_override_types_by_order(
                    LandscapingFactoryProtocol.get_landscape_type)

                if LandscapeType is None:
                    from ccai.landscaping.landscape import Landscape
                    LandscapeType = Landscape

                LANDSCAPE_SINGLETON = LandscapeType()
        finally:
            SINGLETON_LOCK.release()

    return LANDSCAPE_SINGLETON
```

The factory lookup walks registered implementations of
`get_landscape_type`, sorted by precedence.  The first one
that returns a non-None class wins.  If no implementation is
registered, falls back to the base `Landscape`.

## The integration with `TestJob` (Volume 2 chapter 4)

Volume 2 chapter 4 walked the `TestJob` inheritance chain:

```
TestJob → InteropLibraryTestJob → DistributedTestJob
```

`DistributedTestJob._get_landscape()` returns
`LandscapeSingleton()`.  The singleton returns the project's
Landscape subclass (if registered) or the framework's default.

So the customisation cascade is:

1. Project subclasses `Landscape` → `MyProjectLandscape`.
2. Project subclasses `LandscapeDevice` → `MyProjectDevice`
   (per deviceType, possibly multiple).
3. Project registers a `LandscapingFactoryProtocol`
   implementation that returns `MyProjectLandscape`.
4. Project may also subclass `DistributedTestJob` if
   additional job-level customisation is needed (Volume 2
   chapter 4 covered this).
5. The CI / orchestration runs
   `distributed_test_entrypoint(job_type=YourJob)`.
6. The job's `_get_landscape()` calls `LandscapeSingleton()`,
   which returns `MyProjectLandscape`.
7. The framework's activation walks `MyProjectLandscape`'s
   `validate_topology`, instantiates `MyProjectDevice`s for
   each device, dispatches Features through the project's
   subclass methods.

The chain is uniform.  Every `codecraftai-*` extension point
goes through some version of this pattern: framework declares
a protocol, project implements it, namespace package
registers it, factory walks the registrations at startup.

## A worked subclass — start to finish

Suppose a project's automation domain involves devices with a
custom `cooling_loop` Feature that references a glycol-loop
service.

### 1. The custom Landscape

```python
# my_project/landscape.py

from ccai.landscaping.landscape import Landscape
from typing import List, Tuple


class MyProjectLandscape(Landscape):

    def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
        errors = []
        warnings = []

        # Inherit framework checks.
        framework_errors, framework_warnings = super().validate_topology(topology_info)
        errors.extend(framework_errors)
        warnings.extend(framework_warnings)

        # Verify custom 'cooling_zones' section.
        cooling_zones = topology_info.get("cooling_zones", [])
        for zone in cooling_zones:
            zone_name = zone["name"]
            zone_devices = zone.get("devices", [])
            for dev_name in zone_devices:
                dev = self.devices.get(dev_name)
                if dev is None:
                    errors.append(
                        "Cooling zone '{}' references device '{}' not in landscape.".format(
                            zone_name, dev_name
                        )
                    )

        return errors, warnings
```

### 2. The custom Device

```python
# my_project/devices.py

from ccai.landscaping.landscapedevice import LandscapeDevice
from my_project.coolingextension import CoolingProtocolExtension


class MyProjectDevice(LandscapeDevice):

    def attach_features(self):
        super().attach_features()  # built-in Features

        if "cooling_loop" in self._features:
            cooling_spec = self._features["cooling_loop"]
            cooling_ext = build_cooling_extension(self, cooling_spec)
            self.attach_extension("cooling_loop", cooling_ext)
        return

    @property
    def is_isolated(self) -> bool:
        rtnval = self._features.get("isolation", False)
        return rtnval

    @property
    def cooling_zone(self) -> str:
        rtnval = self._features.get("cooling_loop", {}).get("zone", "default")
        return rtnval
```

### 3. The factory implementation

```python
# my_project/factories.py

from typing import Type

from ccai.factories.factoryextension import FactoryExtension
from ccai.landscaping.landscape import Landscape
from ccai.landscaping.landscapingfactoryprotocol import LandscapingFactoryProtocol

from my_project.landscape import MyProjectLandscape


class MyProjectLandscapingFactory(FactoryExtension, LandscapingFactoryProtocol):

    PRECEDENCE = 30

    @classmethod
    def get_landscape_type(cls) -> Type[Landscape]:
        rtnval = MyProjectLandscape
        return rtnval
```

### 4. The namespace registration

In the project's `pyproject.toml`:

```toml
[project]
name = "my-project"

[tool.setuptools.packages.find]
namespaces = true

# Namespace package: my_project lives under ccai.extensions
[tool.setuptools]
package-dir = {"" = "src"}
```

And the project structure includes a `src/ccai/extensions/my_project/__init__.py`
that re-exports `MyProjectLandscapingFactory`.  Volume 0
chapter 7 walked the namespace-package pattern in detail.

### 5. Use

When a `DistributedTestJob` runs, `LandscapeSingleton()`
returns `MyProjectLandscape`.  The Landscape's
`validate_topology` runs, including the cooling-zone
verification.  Devices instantiate as `MyProjectDevice`
(through additional per-deviceType factory machinery), and
their `attach_features` recognises the `cooling_loop`
Feature, attaching a `CoolingProtocolExtension` so test code
can call `device.cooling_loop.set_target_temp(...)`.

The framework code never changed.  The project's code is
self-contained in its own package.  The integration is
through the factory + subclass pattern.

## What this gives the project

By subclassing, the project has:

- **Full control over what the Topology means.**  Custom
  sections, custom verification, custom accessors.
- **Full control over what Features mean.**  Built-in plus
  any number of project-specific.
- **No fork of the framework.**  Project lives in its own
  package; framework lives in its own package; integration
  through the factory protocol.
- **Composability with other projects.**  Multiple registered
  factories with different precedences let
  framework-supplied + tier-supplied + project-supplied
  customisations coexist.

This is the Volume 0 chapter 1 meta-thesis at its sharpest:
the framework opinionates the *patterns* (subclass +
register + plug into TestJob); the organisation supplies
the *bodies* (what the subclasses do).

## What's not customisable

A few things the framework deliberately doesn't expose for
customisation:

- **The TestJob lifecycle order.**  The five `fire_*` hooks
  fire in a specific order; subclasses override individual
  hooks but cannot reorder them.  The order is the contract
  Volume 1 chapter 2 established.
- **The single-actor + tandem-observer model.**  No subclass
  hook lets a project run two tests concurrently in a Pod.
  The determinism commitment is structural (Volume 2 chapter
  2).
- **The JSOS streaming format.**  Subclasses produce records
  with the same shape as the framework's; the format is fixed
  to keep the agents CLI (Volume 1 chapter 6) consumer
  contract stable.

These non-customisations are *features*: they prevent
subclasses from breaking framework-wide invariants the rest
of the series assumes.  A subclass that respected the
non-customisable bits while customising everything else is
the framework's intended shape.

## What the next chapter does

Chapter 8 walks `BaseExt` — a *different* extension model
the framework uses for service-specific code (the VMware
vSphere worked example).  It is *not* the same extension
point as `ProtocolExtension` or `Landscape` subclassing; it
exists for adding service-specific Coordinator helpers
(e.g., a vSphere agent that knows how to query topology from
vCenter).  The chapter distinguishes the two extension models
clearly.
