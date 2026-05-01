# Chapter 7 — Extension and Factory Machinery

> *Some packages need to plug new behaviour into existing extension
> points without modifying the consumers of those extensions.
> The factory machinery in `automojo-extension` is the formal
> mechanism that makes this possible — declaratively, with
> precedence ordering, with substitutability.*

This chapter introduces
[`automojo-extension`](../../../../automojo-extension/source/packages/automojo/extension/),
the foundation-tier package whose job is to provide the
plug-in machinery the rest of the portfolio uses.  The package
defines three kinds of artifact:

- **`FactoryProtocolBase`** — the base of every factory protocol;
  declares what kind of factory it is.
- **`FactoryExtension`** — the base of every concrete factory;
  registers itself by precedence.
- **`SuperFactory`** — the engine that walks discovered factories
  and produces instances by protocol.

The pattern is *not* the GoF Factory pattern in textbook form.
It's a *registration-based, precedence-ordered, multi-source*
factory model that deals with the realities of an
`automojo-*`-style portfolio: many packages, each contributing
factories that override or supplement the base.

## The problem the machinery solves

Consider a sysman feature that needs to *create a network
adapter representation* on a per-OS basis.  The base sysman
package defines a generic `NetworkAdapter` class.  RHEL needs a
`RhelNetworkAdapter` with NetworkManager integration.  Ubuntu
needs an `UbuntuNetworkAdapter` with netplan handling.  macOS
needs a `MacOSNetworkAdapter` with `networksetup` shelling.
Windows needs PowerShell-driven implementations.  UEFI doesn't
need network adapters at all.

The naive solution: a big switch statement keyed on the host
OS.  A `if os == "rhel": return RhelNetworkAdapter()` chain.
This works for one extension point and grows untenable across a
portfolio.  Every new OS adds branches; every new extension
point duplicates the switch.

The `automojo-extension` solution: each per-OS package
*registers* a factory that produces its OS-specific
implementation.  The `SuperFactory` discovers all registered
factories at startup, sorts them by precedence, and dispatches
calls.  Adding a new OS is a *new package* with a registered
factory; the consumer code doesn't change.

## The three artifact types

### `FactoryProtocolBase` — the contract for protocols

[`factoryprotocolbase.py`](../../../../automojo-extension/source/packages/automojo/extension/factory/factoryprotocolbase.py)
defines the marker base for any factory protocol declaration.  In
its entirety:

```python
class FactoryProtocolBase(Protocol):

    factory_protocol_name = None

    def __init_subclass__(cls, *args, **kwargs):
        if cls.factory_protocol_name is None:
            errmsg = (
                "The 'factory_protocol_name' class level field must be set on derived protocol types."
            )
            raise RuntimeError(errmsg)
        return super().__init_subclass__(*args, **kwargs)
```

Two things to note:

- The base is a `typing.Protocol`, *not* an abstract base class.
  Implementers don't inherit from `FactoryProtocolBase`
  directly; they inherit from a *derived protocol* that
  declares the methods of the protocol.
- `__init_subclass__` enforces that every derived protocol sets
  `factory_protocol_name` to a unique string.  A protocol that
  forgets this raises `RuntimeError` at class definition time —
  *before* any instances are created.

A concrete protocol declaration in a higher tier looks like:

```python
class NetworkAdapterFactoryProtocol(FactoryProtocolBase):

    factory_protocol_name = "network_adapter"

    def get_network_adapter_types(self) -> List[Type[NetworkAdapter]]:
        ...
```

The protocol declares *one method*: how to enumerate the
network-adapter types this factory provides.  Different
protocols can have different methods; the only universal
constraint is `factory_protocol_name`.

### `FactoryExtension` — the base for concrete factories

A *concrete factory* is a class that implements one or more
factory protocols.  It inherits from `FactoryExtension`:

```python
class RhelFactoryExtension(FactoryExtension, NetworkAdapterFactoryProtocol):

    PRECEDENCE = 100   # higher = later = wins overrides

    @classmethod
    def get_network_adapter_types(cls) -> List[Type[NetworkAdapter]]:
        return [RhelNetworkAdapter]
```

Two things this declares:

- The class implements `NetworkAdapterFactoryProtocol` (declared
  by the higher tier).
- The class has a `PRECEDENCE` class attribute — a number that
  controls dispatch order.  Higher numbers win when multiple
  factories implement the same protocol.

The `PRECEDENCE` value is the chapter's substantive design
choice.  When two factories implement the same protocol, the
`SuperFactory`'s default behaviour is to walk them by descending
precedence and use the first one that produces a non-None
result.  This means: a project that wants to override the
default RHEL implementation ships its own factory with a higher
`PRECEDENCE`; the override automatically wins.

### `SuperFactory` — the dispatch engine

[`superfactory.py`](../../../../automojo-extension/source/packages/automojo/extension/factory/superfactory.py)
is the engine.  Its job: discover all registered factory
classes at startup, sort them by precedence, and provide
dispatch methods for protocol-specific lookups.

The discovery happens through a known package namespace:
`automojo.factories`.  Every package that ships factory classes
puts them under that namespace; the `SuperFactory` scans
`sys.path` for the namespace, walks every package found,
imports every module, and collects every `FactoryExtension`
subclass:

```python
def scan_automojo_factories_namespace() -> List[str]:
    search_paths = set([p for p in sys.path])
    rel_amr_factories = os.path.dirname(os.path.dirname(os.path.dirname(__file__)))
    search_paths.add(rel_amr_factories)

    modules_found = []

    for dir_path in search_paths:
        check_dir = os.path.join(dir_path, "automojo", "factories")
        if os.path.exists(check_dir) and os.path.isdir(check_dir):
            check_dir_modules = scan_for_descendant_modules(check_dir)
            modules_found.extend(check_dir_modules)

    return modules_found
```

The pattern is *namespace-package-driven*.  Python's namespace
package mechanism lets multiple installed packages contribute
to the same dotted namespace — `automojo-sysman-linux-redhat`
contributes `automojo.factories.sysman.linux.redhat`,
`automojo-sysman-linux-ubuntu` contributes
`automojo.factories.sysman.linux.ubuntu`, and so on.  The
`SuperFactory` doesn't care which package contributed which
factory; it sees them all in the unified namespace.

After discovery, the `SuperFactory` sorts by precedence (using
`operator.attrgetter('PRECEDENCE')`) and exposes dispatch
methods.

## The five dispatch modes

The `SuperFactory` provides five dispatch methods, each suited
to a different consumption pattern:

### `create_instance_by_order(get_type_method)`

Walks all factories in precedence order, calls
`get_type_method` on each, and returns an instance of the
*last* type returned (the lowest-precedence factory's type).
This is the "fallback to default" mode — the highest-precedence
factory's contribution can be `None` (returning nothing) and
the dispatch falls through to lower-precedence factories.

### `create_instance_for_each(get_type_method)`

Walks all factories and returns a list of instances, one per
factory that returned a non-None type.  Use when the consumer
wants *all* contributed implementations.

### `get_override_types_by_order(get_type_method)`

Like `create_instance_by_order` but returns the *type*, not an
instance.  The caller instantiates the type itself.  Use when
the consumer needs control over construction arguments.

### `iterate_override_types_for_each(get_type_method, iterate_type_lists=True)`

A generator that yields each contributed type one by one.  When
a factory's `get_type_method` returns a *list* of types, the
generator yields each type in the list separately (controlled by
the `iterate_type_lists` parameter — note: per the recent
hygiene-sweep typo fix, the parameter name was corrected from
the previous `interate_type_lists`).

This is the workhorse dispatch — most consumer code uses this to
iterate every contribution.

### `iterate_override_types_for_each` use-case

A typical consumer:

```python
super_factory = ConfiguredSuperFactorySingleton()

for adapter_type in super_factory.iterate_override_types_for_each(
        NetworkAdapterFactoryProtocol.get_network_adapter_types
):
    register_adapter_type(adapter_type)
```

The consumer doesn't know which packages contributed which
adapter types.  It walks the contributions in precedence order
and registers each one.  The same loop works whether the host
is RHEL, Ubuntu, macOS, or has multiple OS-specific factories
installed.

## How factories register

A package that wants to contribute a factory does three things:

### 1. Create the namespace-package directory

The package's source layout includes
`automojo/factories/<scope>/`:

```
automojo-sysman-linux-redhat/
└── source/packages/
    └── automojo/
        └── factories/
            └── sysman/
                └── linux/
                    └── redhat/
                        ├── __init__.py
                        └── networkadapterfactory.py
```

The directories are *namespace packages* (no `__init__.py` until
the leaf, or empty `__init__.py`).  This is what lets multiple
installed packages contribute under the same `automojo.factories.*`
prefix without conflicting.

### 2. Define the factory class

`networkadapterfactory.py` contains:

```python
from automojo.extension.factory.factoryextension import FactoryExtension
from automojo.sysman.common.networkadapterfactoryprotocol import NetworkAdapterFactoryProtocol
from automojo.sysman.linux.redhat.networkadapter import RhelNetworkAdapter


class RhelNetworkAdapterFactoryExtension(FactoryExtension, NetworkAdapterFactoryProtocol):

    PRECEDENCE = 100

    @classmethod
    def get_network_adapter_types(cls):
        return [RhelNetworkAdapter]
```

The class is a `FactoryExtension` and implements the protocol.
Its `PRECEDENCE` is set; its method returns the contributed
type(s).

### 3. Install the package

`pip install automojo-sysman-linux-redhat`.  Python's namespace
package mechanism merges the package's `automojo.factories.*`
contribution into the `automojo.factories` namespace.  The
`SuperFactory`'s next discovery pass finds the new factory.

That's the full registration.  No central registry, no manual
imports, no init-time configuration.  The package is installed;
the factory is discovered.

## The precedence ordering

`PRECEDENCE` is a class attribute — an integer.  Higher numbers
*win* when the dispatch reaches a "use the first one" mode.

The convention across the portfolio:

- `PRECEDENCE = 100` — the *base* contribution from a foundation
  package.
- `PRECEDENCE = 200` — a *standard override* from a per-OS or
  per-environment package.
- `PRECEDENCE = 300+` — a *project-specific override* from a
  consuming application.

The numbers are conventions, not enforced.  When two factories
have the same `PRECEDENCE`, the dispatch order is *insertion
order* (which is determined by namespace-walking order — not
predictable across reinstalls).  The discipline is: don't share
`PRECEDENCE` values between factories that implement the same
protocol.

## What the factory machinery is *not*

Three deliberate non-features.

### Not a service locator

The factories return *types*, not *instances*.  The consumer
instantiates the type and manages the lifetime of the instance.
This is consistent with the philosophy's
*explicit-is-better-than-implicit* principle — service-locator
patterns hide instance lifetime, which complicates resource
management.

The five dispatch methods include both type-returning forms
(`get_override_types_by_order`,
`iterate_override_types_for_each`) and instance-returning forms
(`create_instance_by_order`, `create_instance_for_each`).
Consumers that want types use the first; consumers that want
instances use the second.  Both are explicit about which mode
they're using.

### Not runtime-mutable

Factories are discovered at startup and sorted once.  After
that, the factory list is fixed.  Dynamically registering a
factory at runtime — say, after a plugin is installed — requires
re-running discovery.  The shipped `SuperFactory` doesn't
expose an API for this; consumers that need it call the
discovery functions directly.

This is consistent with chapter-1's *stability-over-performance*
principle.  A runtime-mutable factory list would buy
hot-reload support at the cost of locking and synchronisation;
the shipped form is simpler and covers the typical use case.

### Not a dependency-injection container

The factories produce types that satisfy a protocol.  They do
*not* declare dependencies between types, do *not* automatically
construct dependency graphs, do *not* manage transitive
lifetimes.  Each factory is independent; each factory call is
independent; composition is the consumer's responsibility.

When the consumer's needs grow into "I want a type *and* its
dependencies wired up," the right answer is either to extend
the factory protocol (each factory now returns a wired
instance) or to bring in an actual DI container (a separate
tool entirely).  The chapter-2 *if-it's-hard-to-explain-try-
another-way* principle gates the decision.

## How `automojo-this-app` configures the SuperFactory

The `automojo-this-app` package's `wellknown.py` exposes a
helper:

```python
def ConfiguredSuperFactorySingleton():
    global CONFIGURED_SUPER_FACTORY

    if CONFIGURED_SUPER_FACTORY is None:
        # ... discovery + configuration ...
        CONFIGURED_SUPER_FACTORY = build_super_factory()

    return CONFIGURED_SUPER_FACTORY
```

The pattern: *one* `SuperFactory` instance per process, lazily
constructed on first use.  Subsystems that need factory
dispatch call `ConfiguredSuperFactorySingleton()` and use the
returned instance; the discovery happens once.

The `AppPersona.AM_EXTENSIONS` field (chapter 6) carries a
semicolon-separated list of *additional* extension modules the
`SuperFactory` should scan beyond the default
`automojo.factories` namespace.  This is the project-specific
extension hook — a project ships its own factories under a
different namespace and points the persona at it.

## Cross-references

The factory machinery is used by:

- **Volume 4 (Sysman)** for the per-OS implementation dispatch
  catalogued in chapter 2 of that volume.
- **Volume 5 (Interop)** for the protocol implementation
  registry — DNS, SSH, serial, power, etc.  Each protocol
  implementation registers as a factory under
  `automojo.factories.interop.*`.
- **Volume 6 (Landscape)** for the device-detail-mapper
  registry — UPnP devices, networking devices, etc.

The pattern is consistent: each package contributes factories;
the consumer dispatches by protocol; the discovery is automatic.

The chapter-23 of Volume 3 (Extending the Task Model) is the
*opposite* pattern — there, the workflow YAML's `tasktype:
module@Class` reference is *direct*, not factory-mediated.  The
choice is deliberate: workflow tasks are *named explicitly* by
the workflow author, who wants visibility into which task is
selected; cross-OS implementations are *dispatched* by the
runtime, which abstracts the choice from the consumer.  Both
patterns coexist; the choice between them is the chapter-1
*explicit-vs-implicit* question applied to extension dispatch.

## Summary

- **`automojo-extension`** ships the registration-based,
  precedence-ordered factory machinery that lets packages plug
  new behaviour into existing extension points without
  modifying the consumers.
- Three artifact types: **`FactoryProtocolBase`** (the marker
  for a factory protocol; enforces unique
  `factory_protocol_name`), **`FactoryExtension`** (the base
  for concrete factories; declares `PRECEDENCE`),
  **`SuperFactory`** (the dispatch engine).
- Discovery is **namespace-package-driven**.  Packages ship
  factories under the `automojo.factories.*` namespace; the
  `SuperFactory` scans `sys.path` for the namespace and
  imports every contributed module.  Adding a new factory is a
  new package install; no central registry.
- The **five dispatch modes** —
  `create_instance_by_order`, `create_instance_for_each`,
  `get_override_types_by_order`,
  `iterate_override_types_for_each`, plus the per-protocol
  custom dispatchers — give consumers a choice between
  type-returning and instance-returning forms.
- **Precedence convention**: 100 (base), 200 (per-OS / per-env
  override), 300+ (project-specific override).  Higher wins.
- The machinery is *not* a service locator (returns types, not
  managed instances), *not* runtime-mutable (discovery happens
  at startup), *not* a DI container (no transitive dependency
  graph).
- `ConfiguredSuperFactorySingleton()` from
  `automojo-this-app/wellknown.py` is the per-process singleton.
  Discovery is lazy on first call; the
  `AppPersona.AM_EXTENSIONS` field configures additional
  namespaces beyond `automojo.factories`.
- Used by Volume 4 (sysman per-OS dispatch), Volume 5
  (interop protocol registry), Volume 6 (device-detail
  mappers).  The pattern is universal across the portfolio.

The next chapter introduces the runtime variables and well-known
paths — the `automojo-runtime` package that builds on chapter 6's
identity machinery to give every running process a structured
view of its runtime environment.
