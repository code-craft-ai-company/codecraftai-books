# Chapter 5 — Topology — Logical Organisation and Roles, with Verification

> *Where the Landscape is the physical inventory, the Topology
> is the logical organisation laid on top of it.  The framework
> imposes one canonical schema — `clusters` with
> `name`/`group`/`nodes`/`spares` — consumed by
> `LandscapeOperationalLayer._create_clusters`.  Everything
> else is left to the user to implement.  This chapter walks
> what the framework provides; chapter 6 walks the verification
> overload; chapter 7 walks how a project subclasses Topology
> for everything else.*

The Landscape answered *what physical resources exist?*  The
Topology answers *how are those resources logically organised?*

The two are *separate categories* in `automojo-configuration`
(separate `MergeMap`s — `LANDSCAPE_CONFIGURATION_MAP` and
`TOPOLOGY_CONFIGURATION_MAP`).  The same Landscape can support
many Topologies (one fleet organised differently for different
test campaigns).  The same Topology pattern can apply to many
Landscapes (the cluster-with-spare shape applied to different
fleets).

Volume 3 chapter 6 walked the file role at the orchestration
layer; this chapter walks the schema and its consumption.

## Why Topology is left to the user

A point Volume 3 chapter 6 made and this chapter reinforces:
**there is no `automojo-topology` package.**  Topology is a
*configuration category*, not a package.  The framework's
implementation lives across two repos:

- `automojo-configuration` provides the loader for the
  Topology configuration files (the
  `TOPOLOGY_CONFIGURATION_MAP`, the resolver function, the
  `AM_CONFIG_TOPOLOGY_*` variables).
- `automojo-landscaping` provides the runtime consumer
  (`LandscapeConfigurationLayer.load_topology` reads the
  loaded data into `_topology_info`;
  `LandscapeOperationalLayer._create_clusters` consumes one
  canonical section).

Beyond the canonical `clusters` shape (below), the framework
imposes no schema.  Per the userguide
(`automojo-docs/userguide/03-03-topology-file.rst:7–10`):

> *The Automation Kit currently does not have a fixed format
> for the topology file.  The topology file is intended for
> organizations to be able to configure landscape resources
> to the specific unique project specific states.*

This is the meta-thesis from Volume 0 chapter 1 in action:
the framework supplies the skeleton (the loader, the canonical
shape, the verification hooks) and the patterns (subclass
`Topology` to interpret your custom shape).  The organisation
supplies the content.

## The canonical `clusters` shape

The one schema the framework consumes directly is the
`clusters` section of a Topology file.  The shape:

```yaml
clusters:
    - name: db-cluster
      group: replication
      nodes:
          - host-a
          - host-b
          - host-c
      spares:
          - host-d
```

Each cluster has:

- **`name`** — the cluster's identifier.
- **`group`** — the device group the cluster's nodes belong
  to.  Devices in the Landscape carry a `group:` field; the
  cluster collects devices from that group.
- **`nodes:`** — the active members of the cluster, by
  Landscape device name.
- **`spares:`** — the spare members, by Landscape device name.

The framework's consumer:

**File:** `automojo-landscaping/source/packages/automojo/landscaping/layers/landscapeoperationallayer.py:601–636`

```python
def _create_clusters(self, integ_layer: LandscapeIntegrationLayer, topology_info: Dict[str, Any]):
    """
        Creates operational clusters from topology configuration.
    """
    if "clusters" in topology_info:
        clusters = topology_info["clusters"]
        for cinfo in clusters:
            cname = cinfo["name"]
            group_name = cinfo["group"]
            nodes = cinfo.get("nodes", [])
            spares = cinfo.get("spares", [])

            if group_name in table_of_device_groups:
                cgroup = table_of_device_groups[group_name]
                coordinator: NodeCoordinatorBase = cgroup.coordinator
                cluster = coordinator.create_cluster_for_devices(cname, cgroup, nodes, spares)
                cluster.enhance_metadata()
                self._operational_clusters[cname] = cluster
```

For each cluster declaration:

1. Look up the device group by the `group` name.
2. Use the group's `NodeCoordinatorBase` to create a
   `LandscapeDeviceCluster` instance from the nodes and
   spares.
3. Call `cluster.enhance_metadata()` to populate cluster-
   specific metadata (cluster type, internal identifiers).
4. Store the cluster in `_operational_clusters` keyed by name.

After the loop, the framework's `_operational_clusters` dict
holds one `LandscapeDeviceCluster` instance per declared
cluster.

## The `LandscapeDeviceCluster` class

**File:** `automojo-landscaping/source/packages/automojo/landscaping/landscapedevicecluster.py:23`

```python
class LandscapeDeviceCluster:
    """
        A :class:`LandscapeDeviceCluster` object is used to manage a group of
        nodes that are part of a computer cluster and a collection of spares
        that are eligible to participate in the cluster.
    """

    def __init__(self, name: str, nodes: Dict[str, NodeBase],
                 spares: Dict[str, NodeBase], group: LandscapeDeviceGroup) -> None:
        self._name = name
        self._nodes = nodes
        self._spares = spares
        self._group = group
```

Four instance variables, four properties:

- `name` — the cluster's name.
- `nodes` — dict of `node_name → node_object`.
- `spares` — dict of `spare_name → spare_object`.
- `group` — the device group.

The class is small because the *runtime* concept of a cluster
is small.  A test that wants to query the cluster's
membership uses `cluster.nodes`; a test that wants to swap a
node for a spare uses the cluster's swap method (added by
project-specific subclasses); the framework's job is to
*hold* the cluster shape, not to *operate* the cluster.

Operating the cluster — running cluster-management commands,
verifying replication state, performing failovers — is the
project's responsibility, written into `LandscapeDeviceCluster`
subclasses or into test bodies.

## Why the canonical shape is just `clusters`

The framework picks `clusters` as the one canonical shape
because *every distributed test the portfolio has shipped*
needed cluster management.  A cluster is a recurring concept
across automation domains: distributed databases, distributed
queues, replicated services, redundant device fleets.  The
shape is uniform enough that the framework can offer
materialisation without imposing semantics.

What a cluster's *members do* — the protocol they speak, the
state machine they implement, the failover discipline — is
project-specific.  The framework provides the membership
materialisation; the project provides the operations.

## Beyond `clusters` — open schema

A typical Topology file goes well beyond `clusters`.
A representative shape with project-defined sections:

```yaml
environment:
    label: cross-region-failover

clusters:
    - name: primary
      group: web-tier
      nodes: [web-1, web-2, web-3]
      spares: [web-4]
    - name: secondary
      group: web-tier
      nodes: [web-5, web-6]
      spares: []

dependencies:
    - upstream: primary
      downstream: db-cluster
      direction: write
    - upstream: secondary
      downstream: db-cluster
      direction: read-only

roles:
    primary_lb: lb-1
    secondary_lb: lb-2
    monitor: monitor-1

regions:
    east: [primary, db-cluster]
    west: [secondary]
```

The `clusters` section is consumed by the framework.  The
`dependencies`, `roles`, and `regions` sections are
*user-defined*.  The framework loads them into the
`TOPOLOGY_CONFIGURATION_MAP` as raw dict data, but doesn't
*do* anything with them — the project's `Topology` subclass
interprets them.

## Composability

Per Volume 3 chapter 3's universal composability, Topology
files compose like every other category.  An automation
campaign typically composes:

```
Topology names: ["base-topology", "cross-region-overrides"]
→ Loader reads base-topology.yaml + cross-region-overrides.yaml
→ MergeMap overlay (later names win)
→ Single topology_info dict
→ LandscapeOperationalLayer._create_clusters consumes the
  merged clusters section
→ Project's Topology subclass consumes the rest of the merged dict
```

The composition is at the file level.  An override file
might add new clusters, replace a cluster's spares list,
add new project-specific sections, or override existing
ones.  The merged result is what activation sees.

This is what makes the four-environment ladder from Volume 3
chapter 13 work for Topologies.  Local-dev uses
`base-topology + dev-overrides`; CI uses
`base-topology + ci-overrides`; staging uses
`base-topology + staging-overrides`.  Same suite of tests,
four Topologies, four Pods.

## The role of `roles`

The `roles:` section in a Topology declares named roles for
specific devices.  Volume 6 chapter 1's Landscape declarations
already had `role:` per device; the Topology's `roles`
section adds a *Topology-specific* role overlay.

A device's `role:` in the Landscape is its *intrinsic* role
("this is a worker," "this is a controller").  A Topology's
`roles:` is a *configuration-specific* role assignment ("in
this Topology, the primary load balancer is `lb-1`").  The
two coexist; tests can query either:

```python
# Intrinsic role — Landscape level
workers = LandscapeSingleton().devices_by_role("worker")

# Topology role — Topology level
primary_lb = LandscapeSingleton().topology.role("primary_lb")
```

The `topology` accessor returns the framework's `Topology`
object (the subclass the project provided); `role(name)` is a
method the project's subclass implements to interpret the
`roles` section.

## When the canonical shape is enough

For projects whose distributed automation maps onto cluster
membership and not much else, the canonical `clusters` shape
is sufficient.  No subclass needed; the framework's
`LandscapeDeviceCluster` does the job.  Test code uses
`cluster.nodes` and `cluster.spares` directly.

For projects with richer Topology semantics, chapter 7 walks
the subclassing mechanism.

## What the next chapter does

Chapter 6 walks the verification overload — Topology as an
*executable assertion*, not just a description.  The framework
ships a `validate_topology()` stub; the project's subclass
overrides it to verify the configured state matches the
Topology's claim.  Chapter 6 walks the override pattern
in detail with worked examples for the canonical clusters
shape and a project-specific section.
