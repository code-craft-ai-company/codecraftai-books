# Chapter 8 — Service-Specific Framework Extensions

> *`BaseExt` is a different extension model from
> `ProtocolExtension`.  Where ProtocolExtensions wrap protocol
> agents and attach to devices, `BaseExt` is the
> service-specific framework-extension class for adding
> infrastructure-API integrations (the VMware vSphere model
> shipped in source).  This chapter walks `BaseExt` and
> distinguishes it from the other extension points the volume
> has covered.*

The framework has *two* extension models for the environment
layer.  Chapter 3 walked the first: `ProtocolExtension`
attached to a `LandscapeDevice`, providing the device's
protocol surface.  This chapter walks the second: `BaseExt`,
which extends a *service*'s machinery with infrastructure-API
helpers.

The two models look superficially similar — both are
"extensions" in name — but they serve different purposes.
The disambiguation matters because mixing them up makes the
code hard to read and the documentation confusing.

## The two models compared

| | `ProtocolExtension` | `BaseExt` |
|---|---|---|
| Wraps | A protocol Agent | A service-specific helper class |
| Attaches to | A `LandscapeDevice` | A service implementation (e.g. `VSphereAgent`) |
| Purpose | Expose protocol on device (`device.ssh`) | Add API capability to service (`agent.Topology`) |
| User-extension | Subclass for new protocols | Subclass for new infrastructure APIs |
| Scope | Per-device | Per-service-type |
| Reference | Chapter 3 of this volume | This chapter |

Both are "left to the user" extension points but they extend
*different parts of the framework*.

## The `BaseExt` shape

**File:** `codecraftai-landscaping/source/packages/ccai/landscaping/services/vmware/vsphere/ext/baseext.py:22`

```python
class BaseExt:
    """
        Base class for all vSphere extensions used by a :class:`VSphereAgent`.
    """

    def __init__(self, agent: "VSphereAgent"):
        """
            Creates a base extension bound to a vSphere agent.

            :param agent: Owning :class:`VSphereAgent` instance.
        """
        self._agent_ref = weakref.ref(agent)
```

A `BaseExt` is small.  It holds a weak reference to its
*owning agent* — in the shipped example, a `VSphereAgent` that
talks to vCenter.  The base class provides no methods beyond
the constructor; subclasses implement the actual API helpers.

## A worked subclass — `TopologyExt`

**File:** `codecraftai-landscaping/source/packages/ccai/landscaping/services/vmware/vsphere/ext/topologyext.py:23`

```python
class TopologyExt(BaseExt):
    """
        Extension that retrieves topology information from vCenter.
    """

    def __init__(self, agent: "VSphereAgent"):
        """
            Initializes the topology extension.

            :param agent: Owning :class:`VSphereAgent` instance.
        """
        super().__init__(agent)
        return
```

`TopologyExt` extends `BaseExt` to add vSphere-specific
topology queries.  In the full implementation, the class adds
methods like:

- `list_datacenters()` — returns the vCenter's known
  datacenters.
- `list_clusters(datacenter)` — returns clusters in a
  datacenter.
- `list_hosts(cluster)` — returns the hosts in a cluster.
- `query_vm_topology(vm)` — returns a VM's host /
  cluster / datacenter context.

The methods all reach through `self._agent_ref()` to the
vSphere agent, which holds the connection state and credentials.

The agent then exposes the extension as a property:

```python
# Approximate shape — the agent's __init__ wires up its extensions.

class VSphereAgent:

    def __init__(self):
        # ...
        self._topology_ext = TopologyExt(self)
        self._datacenter_ext = DatacenterExt(self)
        self._vm_ext = VmExt(self)
        # ...

    @property
    def Topology(self) -> TopologyExt:
        return self._topology_ext
```

Test code that wants vSphere topology data:

```python
agent = ... # vSphere agent for the lab
datacenters = agent.Topology.list_datacenters()
for dc in datacenters:
    clusters = agent.Topology.list_clusters(dc)
    # ...
```

The pattern is *namespacing*: `agent.Topology.<method>`
groups topology-related methods under a clear prefix.
`agent.Datacenter.<method>` would be datacenter-related;
`agent.VM.<method>` VM-related.  Without `BaseExt`, all the
agent's methods would be flat on the agent class.  With
`BaseExt`, the agent's surface is hierarchical.

## When to use `BaseExt`

The framework's discipline: when a *service*'s API surface
is large enough that flat exposure becomes unwieldy, group
related methods through `BaseExt` subclasses.

For the vSphere example, the agent has dozens of methods —
datacenters, clusters, hosts, VMs, networks, datastores,
snapshots, tags, alarms.  A flat agent class would be
hundreds of methods.  `BaseExt` partitions the surface into
intuitive namespaces.

For smaller services, `BaseExt` is overkill.  A service with
five methods just exposes them flat on the service class.
The decision is the project's; the framework provides the
pattern, the project uses it when warranted.

## Distinction from project-specific Topology subclassing

A reader might ask: *if `TopologyExt` exists for vSphere
topology queries, why is there also Topology subclassing
(chapter 7)?*  The two are different.

- **`TopologyExt`** queries vCenter for *vCenter's notion of
  topology* (datacenters, clusters of physical hosts).  This
  is the *infrastructure provider's* topology.
- **Project's `Topology` subclass** parses the project's
  *Topology configuration file* (the YAML the project
  authors).  This is the *automation's* topology — the
  cluster-with-spare shape, the project-specific sections.

The two coexist because a project running automation against
vSphere-provisioned infrastructure cares about both: the
*provider's* topology (which physical host is the VM running
on?) and the *automation's* topology (which logical cluster
is the VM a member of?).  Two questions, two answers, two
extension mechanisms.

## The factory protocol for `BaseExt`-based services

When a project wants to add a new service-specific extension
beyond vSphere — say, an Active Directory service or a
Kubernetes service — the pattern is:

1. **Subclass `ServiceCoordinatorBase`** (chapter 3) for the
   new service type.
2. **Implement the agent class** that talks to the service's
   API (e.g., `AdAgent` for Active Directory).
3. **Subclass `BaseExt`** for each logical group of API
   methods.
4. **Wire the extensions on the agent** in `__init__`.
5. **Register through the factory protocol** so the framework
   discovers the new service type.

The pattern reuses everything established in earlier
chapters.  The framework's `BaseExt` is the *organising
pattern* for the service's API surface; everything else is the
project's API.

## The relationship to `ControlExtension`

A point of confusion the chapter should clear up:
`ControlExtension` (chapter 3) is yet another extension
class.  Where:

- `ProtocolExtension` extends a `LandscapeDevice` with a
  protocol surface (`device.ssh`).
- `ControlExtension` extends a `LandscapeDevice` with a
  service-driven *control point* (e.g., a power-control
  service exposing `device.power`).
- `BaseExt` extends a *service's agent* with namespaced API
  helpers (`agent.Topology`).

Three different extension mechanisms, three different
purposes.  The Volume 6 reader keeps them straight by
remembering *what each one extends*:

- `ProtocolExtension` extends a Device.
- `ControlExtension` extends a Device (from a Service).
- `BaseExt` extends a Service-agent.

When in doubt, the chapter that introduces the relevant class
will disambiguate.

## What the next chapter does

Chapter 9 walks the **Pod model** in concrete detail — the
runtime instance produced by combining a Landscape and a
Topology, the materialisation of clusters from the canonical
shape, the unit of horizontal CI parallelism.  This chapter
established the two extension models; chapter 9 closes Part
II with the high-level picture of what the activated
environment looks like at runtime.

Part III then walks provisioning, UI automation, GitLab, and
Flask scaffolding — additional packages in the volume's
scope.
