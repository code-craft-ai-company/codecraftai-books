# Chapter 10 — Cloud Provisioning Model

> *Volume 3 chapter 4 walked Specifications as the
> orchestration mechanism that produces Landscape and Topology
> files.  This chapter walks the package that's intended to do
> the *creation* side: `codecraftai-provisioning`.  An honest
> caveat first — the package is currently scaffolding.  The
> chapter walks the intended shape, anchors on the present
> scaffolding, and forward-references the implementation.*

A reader who reaches this chapter expecting a tour of working
AWS / Azure / GCP backends will be disappointed.
`codecraftai-provisioning` exists as a *scaffolding* in the
portfolio — the abstract `CloudProvisioner` is empty, the
vendor backends are stub subclasses, the `ProvisioningEngine`
parses YAML but doesn't yet perform checkouts or creations.

This chapter is honest about that.  The framework's *intent*
is clear (vendor-neutral cloud provisioning that produces
Landscape and Topology files for the next stage), but the
*implementation* is still under construction.  A reader
contributing to the package can use this chapter as the
entry point; a reader using the framework today should treat
the chapter as forward-reference and lean on Volume 3 chapter
4 for what's currently shipping.

## What the package looks like today

**File structure:**

```
codecraftai-provisioning/source/packages/ccai/provisioning/
├── model/cloud/
│   ├── cloudprovisioner.py     ← the abstract base
│   ├── cloudvenue.py           ← named cloud account / project
│   ├── cloudgroup.py           ← resource group within a venue
│   ├── cloudresourceprofile.py ← template for new resources
│   └── clouddevice.py          ← desired device entry
├── venues/
│   ├── aws/awscloudprovisioner.py    ← stub
│   ├── azure/azurecloudprovisioner.py ← stub
│   └── google/googlecloudprovisioner.py ← stub
└── provisioningengine.py       ← parses YAML; no execution yet
```

The model classes are the *vocabulary* the package will
eventually use: `CloudVenue` (a target cloud account),
`CloudGroup` (a resource group in that venue),
`CloudResourceProfile` (a template for new resources),
`CloudDevice` (a desired device declaration).  The
vocabulary is in place; the methods that act on it are not.

## The abstract base — currently empty

**File:**
`codecraftai-provisioning/source/packages/ccai/provisioning/model/cloud/cloudprovisioner.py:2-8`

```python
class CloudProvisioner:
    """
    Class to provision a cloud environment for the AutoMojo model.
    """

    def __init__():
        return
```

A literal placeholder.  The class is declared; the
constructor takes no arguments and returns; no methods exist
yet.

The shape that will eventually fill it: vendor-neutral
methods like `checkout_resources(spec)`,
`create_resources(spec)`, `release_resources(group)`,
`emit_landscape_file(group, output_path)`,
`emit_topology_file(group, output_path)`.  The vendor-specific
backends will override each method with cloud-API calls.

## The vendor backends — stubs

Each backend follows the same shape:

**File:**
`venues/aws/awscloudprovisioner.py:9-11`

```python
class AwsCloudProvisioner(CloudProvisioner):
    def __init__(self):
        return
```

**File:**
`venues/azure/azurecloudprovisioner.py:9-11`

```python
class AzureCloudProvisioner(CloudProvisioner):
    def __init__(self):
        return
```

**File:**
`venues/google/googlecloudprovisioner.py:9-11`

```python
class GoogleCloudProvisioner(CloudProvisioner):
    def __init__(self):
        return
```

Three vendors, three empty subclasses.  The intent is clear:
each vendor will eventually carry the cloud-API integration
for that vendor, exposing the same vendor-neutral interface
to callers above.

## The `ProvisioningEngine` — partial

**File:** `provisioningengine.py:11-52`

```python
class ProvisioningEngine:
    """
    The ProvisioningEngine class is responsible for managing the provisioning of resources.
    It handles the creation, deletion, and management of resources in a cloud environment.
    """

    def __init__(self, *, specinfo: dict):
        """
        Initializes the ProvisioningEngine instance.
        """
        self._specinfo = specinfo
        self._apodinfo = specinfo["apod"]
        self._envinfo = specinfo["environment"]

        self._cloud_venues = []
        if "venues" in self._envinfo:
            venues_list = self._envinfo["venues"]
            for venue in venues_list:
                self._cloud_venues.append(CloudVenue(**venue))
```

The engine *parses* the Specification (the same
Specification YAML Volume 3 chapter 4 walked) into Python
data structures: a list of `CloudVenue`s constructed from
the venue declarations, an `apodinfo` block, an `envinfo`
block.

The class's *operation* methods — anything that would
actually call into a cloud — are not present.  A reader of
the source today sees the parsing path complete and the
acting path absent.

## Where the implementation will land

When the package matures, the expected shape:

```python
class ProvisioningEngine:

    def provision(self) -> ProvisioningResult:
        """
            Walk the spec, dispatch to the right vendor backend per venue,
            checkout-or-create resources, return a result containing
            the emitted Landscape and Topology files.
        """
        result = ProvisioningResult()

        for venue in self._cloud_venues:
            backend = self._select_backend(venue)
            for group in venue.groups:
                if group.mode == "checkout":
                    backend.checkout_group(group)
                elif group.mode == "create":
                    backend.create_group(group)
                else:
                    raise ConfigurationError(...)

        # Each venue's backend has now populated its in-memory
        # representation of provisioned resources.
        landscape_dict = self._build_landscape()
        topology_dict = self._build_topology()

        landscape_path = result.write_landscape(landscape_dict)
        topology_path = result.write_topology(topology_dict)

        return result
```

The shape of "checkout-or-create" Volume 3 chapter 4 promised
lands here: each `CloudGroup` declares a mode; the engine
dispatches to the backend's checkout or create method per
group.  The result is the emitted Landscape and Topology
files the next stage (the orchestrator's
`DistributedTestJob`) consumes.

This is *not the current implementation*.  This is the shape
the implementation will land at when the package is filled
in.  A contributor wanting to bring the package to life
follows this template.

## The vendor-neutral interface — eventual shape

The `CloudProvisioner` base class will eventually carry:

```python
class CloudProvisioner(ABC):

    def checkout_group(self, group: CloudGroup) -> None:
        """
            Reserve an existing pool of resources for this group.

            :param group: The CloudGroup declaring required resources.
            :raises: CloudCheckoutError when the pool can't supply the resources.
        """
        ...

    def create_group(self, group: CloudGroup) -> None:
        """
            Create new resources matching the group's CloudResourceProfile.
        """
        ...

    def release_group(self, group: CloudGroup) -> None:
        """
            Return resources to the pool (for checkout) or destroy them (for create).
        """
        ...

    def landscape_dict_for(self, group: CloudGroup) -> dict:
        """
            Return the Landscape-shape dict for this provisioned group.
        """
        ...

    def topology_dict_for(self, group: CloudGroup) -> dict:
        """
            Return the Topology-shape dict for this provisioned group.
        """
        ...
```

Six methods; vendor backends implement each.  AWS uses Boto3
calls; Azure uses the Azure SDK; GCP uses
`google-cloud-compute`.  The engine doesn't know or care
which vendor handles each call; it asks the right backend
for the venue.

## What the chapter does for today's reader

A reader reaching this chapter today should understand:

- **The vocabulary** — `CloudVenue`, `CloudGroup`,
  `CloudResourceProfile`, `CloudDevice`,
  `CloudProvisioner`, `ProvisioningEngine`.
- **The shape** — vendor-neutral interface, vendor backends,
  Specification → engine → emitted files.
- **The state** — current scaffolding, not production-ready.
  A real CI campaign relies on existing Landscape / Topology
  files (Volume 3 chapter 6's "static path" or the user
  tooling at job startup) rather than expecting
  `codecraftai-provisioning` to produce them.

A contributor who wants to advance the package has the
roadmap above.  A user who wants to *use* what's there
today should treat the package as a planning artefact.

## Why include this chapter at all

The chapter exists because the *vocabulary* and the
*intent* are both load-bearing in the rest of Volume 6 (and
in Volume 3).  When Volume 3 chapter 4 says "CI tooling
emits Landscape and Topology files," the reader should know
where that emission is intended to live (here) and that the
package is currently empty.  Hand-waving over the gap would
mislead.

The chapter also serves as a *contributor's entry point*.  A
reader who wants to advance the framework's provisioning
story sees the scaffolding, the intended shape, and the
methods to fill in.  The chapter is shorter than the
equivalent for a mature package would be, but its job is
*orientation*, not depth.

## What the next chapter does

Chapter 11 walks the **UI automation** package
(`codecraftai-uia`) — a contrast to the provisioning package in
that it *is* mature.  Cross-platform UI automation across
Linux, macOS, and Windows; the `UIElement` /
`ScreenElement` / `UIWindow` / `UIApplication` hierarchy;
the `MacApplication` / `LinuxApplication` wrappers; the
recent 20-bug audit.  Chapter 12 then walks the declarative
application mappings — the project's mechanism for
naming UI elements semantically.
