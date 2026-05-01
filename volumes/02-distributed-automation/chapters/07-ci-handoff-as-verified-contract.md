# Chapter 7 — The CI → Job Hand-off as a Verified Contract

> *The CI system promises the test job two things and the test
> job verifies both before the first test runs.  Landscape
> says "here are your resources"; the job verifies reachability.
> Topology says "here is the configured state"; the job verifies
> the configured state matches.  Verification is a precondition,
> not a feature.*

The previous chapter walked the three landscape activation
layers: load files (Configuration), instantiate objects
(Integration), bring online and verify (Operational).  This
chapter zooms in on the *verification* part and frames it for
what it really is: the **load-bearing precondition** that
makes the determinism model from chapter 2 actually
deterministic.

The framing matters because verification is easy to under-value.
A reader skimming the activation layers might treat
`establish_connectivity` as an implementation detail — "of
course you check the devices are reachable."  But verification
is more than a check: it is a *contract* between the CI system
and the test job.  Treating it as a contract changes how
failures are diagnosed and how the orchestrator behaves when
verification fails.

## The contract has two halves

The CI system provisions a Pod by reading a Specification
(Volume 3 chapter 4), checking out or creating resources, and
emitting a Landscape file plus a Topology file.  The two files
are then handed off to a `DistributedTestJob`, along with a
chosen Runtime (Volume 3 chapter 7).  The hand-off carries two
*promises*:

- **Landscape promise.**  "Here are the resources allotted to
  you.  Each one is at the declared address, reachable with
  the declared protocol, using the declared credentials."
- **Topology promise.**  "Here is the initial configuration of
  those resources.  The clusters are formed as the Topology
  declares.  The roles are assigned as the Topology declares.
  The relationships are wired as the Topology declares."

The test job's job, before running any tests, is to *verify
both promises*.  If either fails, the test job refuses to
proceed; the Pod did not match what was handed off, and
running tests against it would produce results that depend on
the mismatch — non-deterministic by chapter 2's argument.

## Landscape verification — the reachability check

Landscape verification confirms that *every resource declared
in the Landscape is actually reachable at its declared
address, with its declared protocol, using its declared
credentials*.

The mechanism is `LandscapeOperationalLayer.establish_connectivity`
(chapter 6 walked it in detail).  Specifically, the layer
walks every Coordinator (services, power, serial, devices)
and asks each Coordinator's
`establish_connectivity(activation_params)` method to probe
its set.  Each probe returns a tuple
`(host, ipaddr, status, stdout, stderr, exc)`.  Probes that
return a non-zero status, a `False` boolean, or an exception
are collected as failures.

When all coordinators have reported, the operational layer
inspects the failure list.  If `allow_missing_devices` is
False (the default), a non-empty failure list raises
`RuntimeError` and the test job refuses to proceed.  The
exception's message names every failed host with its IP,
status, stderr, and exception.

### What this catches

- **Wrong addresses.**  The Landscape says
  `host: 192.168.1.30`, but the device actually lives at
  `192.168.1.31`.  Probe fails; verification reports the
  mismatch.
- **Down hosts.**  The Landscape says the host exists, but
  the host is currently down.  Probe times out; verification
  reports the timeout.
- **Bad credentials.**  The credentials in the Landscape
  resolve, but the resolved credentials aren't valid for
  authentication.  Probe surfaces `SshAuthError`; verification
  reports the authentication failure.
- **Wrong protocol.**  The Landscape says
  `deviceType: network/ssh`, but the device only speaks Telnet.
  The Coordinator's probe fails; verification reports the
  protocol mismatch.
- **Service unavailability.**  An infrastructure service
  declared in the Landscape's `services` section is unreachable.
  The Coordinator-for-services probe fails; verification
  reports it.

### What this does NOT catch

Landscape verification only checks *reachability*.  It does
not check:

- Whether the device's *configured state* matches what the
  test expects.  That's Topology verification.
- Whether the device's software versions are right.  That's
  application-specific test logic, not framework verification.
- Whether the device's data is in a known state.  That's
  test-fixture territory.

The separation is deliberate.  The framework owns reachability
and configured-state (Topology); the test owns everything
else.

## Topology verification — the configured-state check

Topology verification confirms that *the resources are
configured as the Topology declares*.

The framework's stance from Volume 6 chapter 5: Topology is
*overloaded* as both a description and an executable
assertion.  The same file that says "Nodes A, B, C are
cluster members and Node D is a spare" is also asserting
that the cluster *is* configured that way.  Topology
verification is what closes the loop.

The mechanism is `LandscapeConfigurationLayer.validate_topology(topology_info)`
(line 536).  The shipped implementation is a stub:

```python
def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
    """
        Validates the topology configuration.
    """
    errors = []
    warnings = []
    return errors, warnings
```

Empty errors, empty warnings.  *The stub passes everything.*
This is intentional, per Volume 0 chapter 1's meta-thesis:
the framework supplies the skeleton and patterns; the
organisation customises and refines.  Topology verification is
*one of the explicit places* where the framework expects the
project to specialise.

A project's `Landscape` (or `Topology`) subclass overrides
`validate_topology` with the project-specific verification
logic.  The override walks the loaded `topology_info` dict,
inspects each section the project cares about, and returns
errors / warnings for any mismatch with the actual Pod state.

### A worked override

Suppose a project's Topology declares clusters using the
canonical `clusters` shape (Volume 6 chapter 5):

```yaml
clusters:
    - name: db-cluster
      group: replication
      nodes: [host-a, host-b, host-c]
      spares: [host-d]
```

The project's `validate_topology` override:

```python
class MyProjectLandscape(Landscape):

    def validate_topology(self, topology_info: dict) -> Tuple[List[str], List[str]]:
        errors = []
        warnings = []

        clusters = topology_info.get("clusters", [])
        for cluster_decl in clusters:
            cluster_name = cluster_decl["name"]
            declared_nodes = cluster_decl["nodes"]
            declared_spares = cluster_decl["spares"]

            # Probe each node and ask the cluster manager for membership.
            for node_name in declared_nodes:
                node_dev = self.devices.get(node_name)
                if node_dev is None:
                    errors.append(
                        "Cluster '{}' declares node '{}' but no such device "
                        "is in the integrated landscape.".format(cluster_name, node_name)
                    )
                    continue

                actual_membership = node_dev.cluster_membership_query()
                if actual_membership.cluster_name != cluster_name:
                    errors.append(
                        "Node '{}' should be member of cluster '{}' per topology, "
                        "but reports membership in '{}'.".format(
                            node_name, cluster_name, actual_membership.cluster_name
                        )
                    )

                if actual_membership.role == "spare":
                    errors.append(
                        "Node '{}' should be active member of cluster '{}' per topology, "
                        "but reports spare role.".format(node_name, cluster_name)
                    )

            # Verify spares.
            for spare_name in declared_spares:
                spare_dev = self.devices.get(spare_name)
                if spare_dev is None:
                    errors.append(
                        "Cluster '{}' declares spare '{}' but no such device "
                        "is in the integrated landscape.".format(cluster_name, spare_name)
                    )
                    continue

                actual_membership = spare_dev.cluster_membership_query()
                if actual_membership.role != "spare":
                    errors.append(
                        "Node '{}' should be spare of cluster '{}' per topology, "
                        "but reports role '{}'.".format(
                            spare_name, cluster_name, actual_membership.role
                        )
                    )

        return errors, warnings
```

The override:

- Walks each cluster declared in the topology.
- For each declared node, queries the live device for its
  cluster membership.
- Reports errors when the live state doesn't match the
  declared state.
- For each declared spare, verifies the role is actually spare.

When the framework calls `validate_topology(topology_info)`,
this override runs.  If the override returns a non-empty
errors list, the test job's activation fails.  The Pod is not
realised.  Tests do not run.

### What this catches

- **Cluster membership mismatch.**  Node A is supposed to be
  in cluster X but is actually in cluster Y.
- **Role mismatch.**  Node D is supposed to be a spare but is
  actually an active member (or vice versa).
- **Missing devices.**  A node referenced by the topology is
  not in the integrated landscape (couldn't be resolved
  during integration layer activation).
- **Project-specific state assertions.**  Anything the project
  cares about — load-balancer health, replication lag,
  active configuration version — checked against the
  Topology's claim.

## Why this matters for determinism

A test against an unverified Pod cannot be deterministic.

Suppose the Topology says "Node A is the cluster leader; Node
B is a follower."  The test says "kill the leader, verify the
cluster elects a new one."  If the Pod was actually
configured with Node B as the leader (the Topology was wrong
or the CI system mis-provisioned), the test kills Node B
instead — and the cluster behaviour is different.  The test
might pass (a leader was elected) but the *property the test
was checking* (election behaviour after killing the configured
leader) was never actually verified.

Verification eliminates this class of failure.  When
verification passes, the test knows the Pod's state matches
its expectations; the test's actor-driven effects produce
deterministic outcomes against a known-state environment.

When verification fails, the framework refuses to run.  The
test job exits with a structured error report (Volume 0
chapter 3's bubble-the-failure discipline applied at the
hand-off boundary).  The orchestrator sees the failure and
can decide:

- **Re-provision the Pod.**  Often the right answer for
  transient infrastructure issues.
- **Mark the run as infrastructure-failed.**  Distinct from
  test-failed; the report should reflect "we couldn't run the
  tests," not "the tests failed."
- **Halt the campaign.**  If multiple Pods are failing
  verification, something systemic is wrong and continuing
  wastes resources.

Volume 3 chapter 21 walks how the orchestrator integrates
verification into its Step 2.5 flow; this chapter walks the
job-side mechanism the orchestrator depends on.

## When to relax verification

The strict defaults
(`allow_missing_devices=False`, `validate_topology=True`)
are correct for production runs.  There are a few cases
where a project legitimately relaxes them:

### Smoke runs

A "smoke" run wants to exercise the framework's plumbing
without requiring the whole Pod.  The `allow_missing_devices=True`
flag lets the run proceed with whatever devices come up;
unreachable devices are logged but don't fail activation.

A smoke run's scope is "did the framework start up cleanly
against this Pod?"  Not "did all the tests pass?"  Treating
the two as the same scope erases useful information.

### Inspection / audit runs

A run that only inspects the loaded configurations (without
running tests) can use
`disable_device_activation=True` and
`disable_service_activation=True` to skip the integration and
operational layers entirely.  Useful for "what does this
configuration look like?" workflows that don't touch the real
environment.

### Topology development

When a project is *developing* a `validate_topology` override,
it might want to run with `validate_topology=False` to skip
the override and observe what raw Topology data is loaded.
This is a development affordance, not a production setting.

In every case the relaxation is *explicit*.  The defaults
ensure that a production run accidentally relaxing
verification has to *deliberately* edit code or flags to do
so.  The framework refuses to silently downgrade verification.

## What the verification contract is *not*

Three things readers commonly conflate with verification that
are not part of it.

### It is not test discovery

Verification runs after test discovery.  The test discovery
machinery (Volume 1 chapter 2) walks the testroot and builds
a list of tests; verification runs after that list is
populated and decides whether the Pod is ready to run them.
A verification failure does not invalidate the discovery
results; the orchestrator can re-run discovery against the
same testroot if it wants.

### It is not test fixture setup

Test fixtures (Volume 1 chapter 3) run *per-test*; they
acquire and release resources for individual tests.
Verification runs *per-job*, before any test fixture has been
exercised.  A test fixture that relies on a service the
Landscape declares can assume the service is reachable
because verification already confirmed reachability.

### It is not orchestration's responsibility

Per Volume 3 chapter 21, the orchestration layer hands off
*names* to the job; the job resolves the names and verifies
them.  Orchestration does not run verification — it
*receives* the verification result (success or
structured-error) from the job and decides what to do next.

The separation matters because orchestration runs in a
different process from the job.  Putting verification in
orchestration would require the orchestrator to have all the
landscape activation machinery in its process — which would
duplicate code and break the layered design.  Putting
verification in the job lets the job own its own
preconditions; orchestration just receives the report.

## What the next chapter does

Chapter 8 walks the `DistributedTestSequencer` in detail.  The
sequencer is what *runs* the suite once verification passes —
the loop that walks each test in declaration order, calls the
test, captures the result, advances.  Chapter 8 specifically
addresses the question "why doesn't the distributed sequencer
parallelise tests?" and walks the differences between the
distributed sequencer and the base `TestSequencer`.

The full Part II of the volume — chapters 4 through 7 — has
walked the *customisation cascade* (chapter 4), the
*observation mechanism* (chapter 5), the *activation layers*
(chapter 6), and the *verification contract* (chapter 7).
Part III walks the *runtime machinery* — sequencer, job_seed,
the wire, credentials, data profiles.
