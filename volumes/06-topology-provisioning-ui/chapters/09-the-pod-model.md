# Chapter 9 — The Pod Model — Landscape + Topology Realised

> *Chapters 1 through 8 walked the schemas and the
> implementation.  This chapter walks what's at runtime when
> activation completes — the **Pod**, the unit of horizontal
> CI parallelism, the integrated state of a Landscape plus a
> Topology that test code runs against.*

The Pod is a concept that crosses Volumes 1, 2, 3, and 6.
Each volume carries part of the picture:

- Volume 1 chapter 9 introduced the vocabulary at the
  contract level.
- Volume 2 chapter 3 walked the Pod as the unit of horizontal
  parallelism.
- Volume 3 chapter 4 walked Specifications as the
  Pod-provisioning mechanism.
- Volume 6 (this volume) walks the *runtime instance* — what
  the framework holds in memory after activation.

This chapter closes Volume 6's Part II by assembling all the
pieces into one runtime picture.

## What a Pod is at runtime

A Pod is the runtime instance produced when a
`DistributedTestJob` activates against a Landscape and a
Topology.  Operationally, a Pod is:

- A `Landscape` singleton instance (or a project's subclass).
- Its `_layer_configuration._landscape_info` and
  `_topology_info` dicts.
- Its `_layer_integration._integrated_devices` and
  `_integrated_services` dicts holding `LandscapeDevice` and
  `LandscapeService` instances.
- Its `_layer_operational._operational_clusters` dict holding
  `LandscapeDeviceCluster` instances materialised from the
  Topology.
- The Coordinators managing each subset of devices.
- The ProtocolExtensions attached to each device.
- The `CredentialManager` holding resolved credentials.
- The `DataProfileManager` (Volume 2 chapter 12) holding
  resolved data profiles.

The Pod *is* the in-memory state.  Test code reaches into it
through `LandscapeSingleton()` and the device / service /
cluster accessors.

## The realisation flow

A reader should be able to walk Pod realisation step-by-step.
The full flow:

1. **CI tooling** reads a Specification (Volume 3 chapter 4),
   checks out or creates resources, emits a Landscape file
   and a Topology file.
2. **Orchestration** invokes a `DistributedTestJob` with the
   Landscape and Topology names as inputs.
3. **`DistributedTestJob.__enter__`** runs `begin()`, which
   constructs the recorder and the `DistributedTestSequencer`.
4. **`fire_attach_to_environment`** and
   **`fire_attach_to_framework`** propagate the landscape to
   integration couplings.
5. **`fire_activate_configuration`** calls
   `landscape.activate_configuration()`:
   - `LandscapeConfigurationLayer.load_landscape` reads the
     Landscape file via the
     `LANDSCAPE_CONFIGURATION_MAP`.
   - `load_topology` reads the Topology file.
   - `initialize_credentials` resolves the credentials.
   - `validate_topology` (the project's override) verifies
     the Topology's claim against the actual environment.
6. **`fire_activate_integration`** calls
   `landscape.activate_integration()`:
   - For each declared device: look up the right Coordinator
     by deviceType, call
     `coordinator.create_landscape_device(...)`, get back a
     `LandscapeDevice` (or project subclass).
   - For each device, walk the `_features` dict: built-in
     Features get their ProtocolExtensions wired
     automatically; project-specific Features are dispatched
     through the project's `LandscapeDevice` subclass's
     `attach_features`.
   - Same for services and the special-purpose subsystems
     (power, serial).
   - Result: `_integrated_devices` and `_integrated_services`
     are populated.
7. **`fire_activate_operations`** calls
   `landscape.activate_operations()`:
   - `LandscapeOperationalLayer.activate_coordinators` —
     each Coordinator probes its devices, brings them
     online, populates metadata.
   - `_create_clusters` consumes the `clusters` section of
     the Topology, materialises `LandscapeDeviceCluster`
     instances.
   - `establish_connectivity` verifies every device is
     reachable.
8. **The Pod is realised.**  Test code can now call
   `landscape.device(name)`, `landscape.devices_by_role(role)`,
   `landscape.cluster(name)`, etc.

After step 8 the Pod's runtime state is complete.  Tests run
against it sequentially (Volume 2 chapter 8).

## What test code sees

A test inside a realised Pod has the following accessors:

```python
landscape = LandscapeSingleton()

# Single device by name
casey_01 = landscape.device("casey-01")
casey_01.ssh.run_cmd("uptime")          # primary access
casey_01.serial.write("...")            # service-reference Feature
casey_01.power.cycle()                  # service-reference Feature

# Devices by role (Landscape's role: field)
workers = landscape.devices_by_role("worker")
for w in workers:
    w.ssh.run_cmd("...")

# Cluster by name
db_cluster = landscape.cluster("db-cluster")
for node in db_cluster.nodes.values():
    node.ssh.run_cmd("...")
spares = list(db_cluster.spares.values())

# Project-specific subclass accessors
zone_a_devices = landscape.devices_in_cooling_zone("zone-a")  # custom

# Service by name
vault = landscape.service("lab-vault")
vault.api.get("/secrets/...")           # service-specific extension

# Topology accessors (project-specific)
deps = landscape.topology.dependencies("primary")  # custom
```

The accessors are *uniform* — devices, services, and
clusters are queried the same way regardless of how they
were declared (apod / infrastructure / project-specific
sections).  The Pod's runtime exposes a clean test surface.

## The Pod's properties as a unit of parallelism

Volume 2 chapter 3 walked these properties; this chapter
restates them with the schema-level grounding from Volume 6:

### Internal interoperability

Within a Pod, the framework has *wired* every device's
ProtocolExtensions, every Cluster's membership, every
service's connection.  Test code calling `device.ssh.run_cmd`
hits a real SSH session; calling `cluster.nodes`
returns the actual node instances; calling
`service.db.execute` hits the actual database connection.
The framework's activation built all the wires.

### External isolation

Pods don't share state.  When CI runs ten Specifications
producing ten Pods, each Pod has its *own* Landscape
singleton, its *own* Coordinators, its *own* Credentials
manager.  Effects in Pod A — a cluster failover, a service
restart, a credential rotation — *cannot* propagate to Pod
B because Pod B has no shared object referencing Pod A's
resources.

The isolation property depends on Pod independence at the
*infrastructure* level (different physical resources, different
network namespaces, different shared services).  The framework
assumes this; the orchestration tier (Volume 3 chapter 4) is
what provides it through Specifications.

### Single-actor execution

Within a Pod, one test runs at a time (Volume 2 chapter 2's
commitment).  Tandem validators (Volume 2 chapter 5) attach to
the test for parallel observation; the test is the only
actor.

## When Pod realisation fails

Pod realisation can fail at each step:

- **Step 5 — Configuration.**  Files don't parse, credentials
  don't resolve, `validate_topology` rejects the topology.
  The framework's `ConfigurationError` or `RuntimeError`
  bubbles; activation aborts.
- **Step 6 — Integration.**  Unknown deviceTypes, misconfigured
  Features, Coordinator init errors.  Activation aborts.
- **Step 7 — Operational.**  `establish_connectivity` finds
  unreachable devices.  If `allow_missing_devices=False`,
  activation aborts; if `True`, activation continues with the
  missing devices flagged.

Each failure mode produces a *structured error report* the
orchestrator can read (Volume 3 chapter 21).  The
orchestrator decides whether to re-provision the Pod, mark
the run as infrastructure-failed, or escalate the campaign.

## The Pod as the unit of horizontal CI parallelism

The framework's commitment from Volume 2 chapter 3:

| Level | Parallel? | Why |
|---|---|---|
| Within a test | No | Single actor commitment |
| Across tests in one Pod | No | Shared environment |
| Across Pods | **Yes** | Pods are independent |

Horizontal scaling at the CI layer is *more Pods*.  Volume 3
chapter 4 walked Specifications as the Pod-provisioning
mechanism.  When a campaign needs to finish faster, the
orchestrator runs more Specifications concurrently → more
Pods → more `DistributedTestJob` processes → more parallel
test execution at the *job* level, with serial test
execution within each job.

## What the next chapter does

Chapter 10 walks the **provisioning** layer that produces
Landscapes and Topologies from Specifications.  The chapter
is honest about the current state: `codecraftai-provisioning` is
in scaffolding stage — the abstract `CloudProvisioner` is
empty, the AWS / Azure / GCP backends are stub subclasses,
the `ProvisioningEngine` parses YAML but doesn't yet
implement checkout-or-create.  Chapter 10 walks the *intended
shape* with anchors on the present scaffolding and forward
references to where the implementation will land.

This is the volume's first chapter on a topic outside
Landscape/Topology proper.  Chapters 11 and 12 cover UI
automation; chapters 13 and 14 cover the GitLab and Flask
worked examples; chapter 15 is the volume's case study.

End of Part II.
