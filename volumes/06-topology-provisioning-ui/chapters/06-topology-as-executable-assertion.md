# Chapter 6 — Topology as an Executable Assertion

> *A Topology file declares the configured state of resources.
> The framework treats that declaration as an executable
> assertion: at job startup, the framework's `validate_topology`
> hook runs; if the configured state doesn't match the
> Topology's claim, the job refuses to run.  The framework
> ships a stub validator; the project supplies the override.*

The previous chapter walked the canonical `clusters` shape and
established that beyond that shape, Topology is open to the
user.  This chapter walks the *verification* overload — the
mechanism by which a Topology file becomes more than a
description.

The framing matters.  A Topology that's *just a description*
is useful documentation but doesn't catch CI mis-provisioning.
A Topology that's an *executable assertion* fails loudly when
the CI system has handed off resources that don't match the
declared configuration.  Volume 2 chapter 7 walked the
verification contract at the contract level; this chapter
walks the implementation in detail.

## Where verification happens

The activation flow from Volume 2 chapter 6:

1. `fire_activate_configuration()` — loads files, initialises
   credentials, calls `validate_topology()`.
2. `fire_activate_integration()` — instantiates devices and
   services from the loaded config.
3. `fire_activate_operations()` — brings devices online via
   Coordinators; runs `establish_connectivity` for Landscape
   verification (chapter 7 of Volume 2 walked the
   reachability check).

`validate_topology()` runs in phase 1, before devices are
even instantiated.  This is intentional: a Topology that
doesn't parse or doesn't match the Landscape's resource list
should fail *before* the framework spends time bringing
devices online.

## The shipped stub

**File:** `codecraftai-landscaping/source/packages/ccai/landscaping/layers/landscapeconfigurationlayer.py:536`

```python
def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
    """
        Validates the topology configuration.
    """
    errors = []
    warnings = []
    return errors, warnings
```

The shipped implementation passes everything.  Empty errors,
empty warnings.  Per the meta-thesis, this is intentional:
the framework supplies the *hook*; the organisation supplies
the *body*.

A reader's natural question: *if the stub passes everything,
how does verification ever fail?*

The answer: when the project subclasses `Landscape` (or
`Topology`) and overrides `validate_topology`.  Without the
override, no verification happens.  With the override, the
override decides what counts.

## The override — pattern

A representative override that verifies the canonical
`clusters` shape:

```python
class MyProjectLandscape(Landscape):

    def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
        errors = []
        warnings = []

        # Inherit the framework's checks (currently a no-op).
        framework_errors, framework_warnings = super().validate_topology(topology_info)
        errors.extend(framework_errors)
        warnings.extend(framework_warnings)

        # Verify each cluster's nodes are actually members.
        clusters = topology_info.get("clusters", [])
        for cdecl in clusters:
            cname = cdecl["name"]
            declared_nodes = cdecl.get("nodes", [])
            declared_spares = cdecl.get("spares", [])

            for node_name in declared_nodes:
                node_dev = self.devices.get(node_name)
                if node_dev is None:
                    errors.append(
                        "Cluster '{}' declares node '{}' but no such device "
                        "is in the integrated landscape.".format(cname, node_name)
                    )
                    continue

                # Project-specific membership query.
                actual_membership = node_dev.cluster_membership_query()
                if actual_membership.cluster_name != cname:
                    errors.append(
                        "Node '{}' should be member of cluster '{}' per topology, "
                        "but reports membership in '{}'.".format(
                            node_name, cname, actual_membership.cluster_name
                        )
                    )

                if actual_membership.role == "spare":
                    errors.append(
                        "Node '{}' should be active member of cluster '{}' per topology, "
                        "but reports spare role.".format(node_name, cname)
                    )

            for spare_name in declared_spares:
                spare_dev = self.devices.get(spare_name)
                if spare_dev is None:
                    errors.append(
                        "Cluster '{}' declares spare '{}' but no such device "
                        "is in the integrated landscape.".format(cname, spare_name)
                    )
                    continue

                actual_membership = spare_dev.cluster_membership_query()
                if actual_membership.role != "spare":
                    errors.append(
                        "Node '{}' should be spare of cluster '{}' per topology, "
                        "but reports role '{}'.".format(
                            spare_name, cname, actual_membership.role
                        )
                    )

        return errors, warnings
```

The override:

- Calls `super().validate_topology(topology_info)` first (good
  citizen — preserves framework checks if any get added).
- Walks each cluster declaration.
- For each declared node, queries the live device for its
  cluster membership (a project-specific method
  `cluster_membership_query()` on the project's
  `LandscapeDevice` subclass).
- Reports errors when the live state doesn't match the
  declaration.
- Returns `(errors, warnings)`.

When the framework calls `validate_topology(topology_info)`
and the return list of errors is non-empty, activation fails.
The Pod is not realised.  Tests do not run.

## The override — verifying user-defined sections

For a Topology with project-specific sections beyond
`clusters`, the override extends to verify them too.  Suppose
the Topology has:

```yaml
clusters:
    - { name: primary, group: web-tier, nodes: [w1, w2, w3], spares: [w4] }

dependencies:
    - upstream: primary
      downstream: db-cluster
      direction: write
```

The override adds dependency verification:

```python
def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
    errors = []
    warnings = []

    # ... cluster verification as before ...

    # Verify each dependency's upstream and downstream are real clusters.
    dependencies = topology_info.get("dependencies", [])
    for dep in dependencies:
        upstream_name = dep["upstream"]
        downstream_name = dep["downstream"]

        if upstream_name not in self._operational_clusters:
            errors.append(
                "Dependency declares upstream '{}' that is not an operational cluster.".format(
                    upstream_name
                )
            )

        if downstream_name not in self._operational_clusters:
            errors.append(
                "Dependency declares downstream '{}' that is not an operational cluster.".format(
                    downstream_name
                )
            )

        # Project-specific: verify the actual wiring.
        if upstream_name in self._operational_clusters and downstream_name in self._operational_clusters:
            upstream = self._operational_clusters[upstream_name]
            downstream = self._operational_clusters[downstream_name]

            wiring = check_wiring(upstream, downstream)
            if wiring.direction != dep["direction"]:
                errors.append(
                    "Dependency '{}' → '{}' should be '{}' per topology, "
                    "but wiring shows '{}'.".format(
                        upstream_name, downstream_name,
                        dep["direction"], wiring.direction
                    )
                )

    return errors, warnings
```

The pattern repeats.  Each project-specific section in the
Topology gets a verification block in the override.  The
framework holds the dispatch; the project writes the checks.

## What verification catches

The classes of failure verification catches:

### CI provisioning errors

The most common case.  CI provisioned a 3-node cluster but
the Topology declared 4.  Verification catches the mismatch
and refuses to run; the orchestrator (Volume 3 chapter 21)
treats it as an infrastructure failure, re-provisions, and
retries.

### Configuration drift

A previously-correct Pod was modified between CI runs (a
manual reconfiguration, an automated tool, a partial
deployment).  Verification catches the drift; the
orchestrator can decide whether to repair the Pod or fail.

### Topology / Landscape mismatch

The Topology declares a cluster with `nodes: [w1, w2, w3]`
but the Landscape only declares `w1` and `w2` (typo,
incomplete merge, missing override file).  Verification's
"no such device in the integrated landscape" error catches
the mismatch.

### Stale Topology files

A Topology file from yesterday's experiment still in the
config repo, accidentally selected by today's CI.
Verification's structural checks fail because today's Pod
shape doesn't match yesterday's Topology.  Better than
running tests against a wrong-shape Pod and getting confusing
results.

## What verification deliberately does NOT do

Three things verification deliberately doesn't catch:

### Application-level state

"The database has 1,000 rows of seed data."  This is fixture
territory (Volume 1 chapter 3), not verification territory.
Verification is about *infrastructure shape*, not data.

### Software versions

"Every node runs Postgres 16."  Useful, but not a Topology
concern — closer to a separate "version-pinning" verification
the project might add as its own validator.  The framework's
verification is for declared topology-level state.

### Behavioural correctness

"The cluster fails over within 30 seconds."  This is *test*
territory — a tandem validator (Volume 2 chapter 5) is the
right shape for behavioural observation during a test, not a
pre-test verification.

## When verification fails — what the orchestrator sees

The framework propagates verification failures as errors that
abort `activate_configuration`.  The error includes the list
of `errors` returned by the override, joined for human
reading.  A representative orchestrator log:

```
[2026-05-01 15:34:22] Activation failed in fire_activate_configuration:
    Topology validation failed.
    ERRORS:
        Cluster 'primary' declares node 'w3' but no such device is in the integrated landscape.
        Cluster 'primary' declares spare 'w4' but no such device is in the integrated landscape.
        Dependency 'primary' → 'db-cluster' should be 'write' per topology, but wiring shows 'none'.
    Job refused to proceed.
```

The orchestrator reads the structured failure (Volume 3
chapter 21's verified-contract step), decides what to do, and
either re-provisions or escalates.  The framework's job ends
at "refused to proceed"; the orchestrator's job picks up.

## The `validate_topology` flag

`LandscapeActivationParams.validate_topology` (default
`True`) is the on/off switch for verification.  When `False`,
the override is skipped and any topology mismatch goes
uncaught.

When to set `False`:

- **Topology development.**  When a project is *writing* its
  override and wants to observe what raw Topology data is
  loaded without the override running.
- **Single-host smoke runs.**  Where the Topology is
  irrelevant because no clusters exist.
- **Inspection-only runs.**  Where the test job is just
  examining configuration without exercising it.

When to keep `True` (the default):

- **Every production CI run.**

The strict default is intentional.  A project that
accidentally relaxes verification has to *deliberately* set
the flag to `False`; the framework refuses to silently
downgrade.

## The relationship to Vol 2 ch.5's validators

Topology verification is *pre-test* — runs once at activation,
before any test fires.  Tandem validators (Volume 2 chapter
5) are *during-test* — run alongside the test, observing
mid-test state.  The two are complementary:

- **Verification** catches "the Pod is wrong" failures
  before they confuse tests.
- **Tandem validators** catch "the Pod's behaviour is wrong"
  failures during tests.

A test-with-validators against a verified-Topology Pod is
the framework's full guarantee: the starting state is right
(verified), the actor's effects are observed (validators),
the test's outcome is reproducible (job_seed plus verified
state).

## What the next chapter does

Chapter 7 walks the subclassing mechanism in full — how a
project subclasses `Landscape` and `Topology` to plug in its
custom Feature handling, its custom Topology shapes, and its
custom verification.  The chapter walks the
`codecraftai-landscaping/wellknown.py:LandscapeSingleton`
factory-overridable singleton, the registration pattern, and
the integration with the `TestJob` customisation cascade
from Volume 2 chapter 4.

The skeleton + patterns meta-thesis from Volume 0 chapter 1
lands here at its sharpest: subclassing is the concrete
mechanism that ties every "left to the user" note in the
volumes to a real engineering action.
