# Chapter 6 — Topology Files

> *Step ordering is a flat list.  Component dependency is a graph.
> When a flat list pretends to be a graph, inserting a step in the
> middle becomes a logic puzzle.*

The previous chapter introduced Landscape — the inventory of who
and what is in scope for an orchestration run.  A Landscape lists
participants.  It does not say *how those participants relate to
each other*.

Two test runners might both be in the apod, but in one orchestration
they form a redundant cluster (the loss of one is tolerable) and in
another they are a leader-follower pair (the loss of the leader is
not).  Two infrastructure services might both be in the landscape,
but one orchestration has a hard ordering between them ("staging-db
must be live before staging-api can start") while another has no
such constraint.

Relationships of this kind are the third configuration category:
**Topology**.

## Topology is the relationships, not the participants

The boundary between Landscape and Topology is sharp:

| Question | Category |
|---|---|
| Is this host part of the run? | Landscape |
| Does this host depend on that host? | Topology |
| What hostname does this host have? | Landscape |
| What role does this host play in this orchestration? | Topology (when role varies by orchestration) |
| What is this host? | Landscape (`deviceType`) |
| How does this host relate to its peers? | Topology |

The same physical Landscape can host multiple Topologies.  The four
runners in a staging-east apod can be:

- **Topology A — flat:** four independent workers, no coordinator,
  loss-tolerant.
- **Topology B — leader-follower:** `runner-1` is leader; `runner-2`,
  `runner-3`, `runner-4` are followers.
- **Topology C — two clusters:** `runner-1` and `runner-2` form a
  Tier-1 high-availability pair; `runner-3` and `runner-4` form a
  Tier-2 batch pool.

Same hosts.  Same Landscape file.  Three different Topology files.
A pipeline switches between them by setting `AM_TOPOLOGY_NAMES`
(per the fixed-interface principle from chapter 7).  Without the
separation, every relationship change would require duplicating the
Landscape — the four-runner Landscape would become three near-copies,
and any change to the underlying hosts would have to be propagated
across all of them.

This is the chapter's central architectural claim: *Topology is its
own file because relationships vary independently from inventory.*

## A note on the schema

A Landscape file has a shipped schema and a shipped example
([`codecraftai-automation-kit/userguide/examples/landscape.yaml`](https://github.com/code-craft-ai-company/codecraftai-automation-kit/blob/main/userguide/examples/landscape.yaml)).
A Topology file at the time of writing has the *configuration
plumbing* in place — `codecraftai-configuration` loads topology files
through the standard `MergeMap` mechanism and exposes them through
[`AM_CONFIG_TOPOLOGY_NAMES`](https://github.com/code-craft-ai-company/codecraftai-configuration/blob/main/source/packages/ccai/configuration/configurationvariables.py)
and the `TOPOLOGY_CONFIGURATION_MAP`, and the
[`TopologyIntegrationLayer`](https://github.com/code-craft-ai-company/codecraftai-landscaping/blob/main/source/packages/ccai/landscaping/layers/topologyintegrationlayer.py)
class is shipped as the runtime hook — but the *schema content* is
consumer-defined.  Different orchestration teams have different
notions of what relationships matter to them.

There is **one exception** to the consumer-defined schema rule.
The framework consumes a `clusters` section with a fixed shape:
each cluster has a `name`, a `group`, a list of `nodes`, and a
list of `spares`.  The shape is consumed by
[`LandscapeOperationalLayer._create_clusters`](https://github.com/code-craft-ai-company/codecraftai-landscaping/blob/main/source/packages/ccai/landscaping/layers/landscapeoperationallayer.py)
which materialises `LandscapeDeviceCluster` objects from the
data.  This is the framework's only structural requirement on
the Topology file's content — every other top-level section
the user defines is parsed as raw `MergeMap` data and passed
through to the user's `Topology` subclass for interpretation.

That single canonical `clusters` shape is also what enables the
verification contract from chapter 21: the framework knows how
to verify a `clusters` section ("are these nodes really cluster
members?  Is the spare really standing by?") because it knows
the shape.  User-defined sections require user-supplied
verification logic in a `Topology` subclass.

This chapter therefore proposes a canonical schema with three
concerns: **clusters** (the framework's shape, extended
slightly), **dependencies**, and **topology-specific roles**.
These three cover the relationship questions that recur across
every orchestration the author has built.  A reader who needs
more (routing tables, replication topologies, sharding
strategies) extends the schema additively; the
`TOPOLOGY_CONFIGURATION_MAP` is a `MergeMap` of arbitrary YAML,
so new top-level keys are free to add — but the user is then
responsible for subclassing `Topology` to interpret them
(Volume 6 chapter 7 walks the subclassing mechanism).

## The proposed Topology schema

A Topology file has four top-level sections: `environment` (which
landscape this Topology applies to), `clusters` (groups that act
together), `dependencies` (directional ordering), and `roles`
(member roles within this Topology).

```yaml
# topology-staging-east-leader-follower.yaml
environment:
    label: staging-east-leader-follower
    landscape: staging-east

clusters:
    - name: runner-cluster
      members:
          - runner-1
          - runner-2
          - runner-3
      coordinator: runner-1
      cardinality:
          min: 2
          max: 4

dependencies:
    - source: smoke-tests
      requires:
          - staging-api
          - staging-db
    - source: staging-api
      requires:
          - staging-db

roles:
    - member: runner-1
      role: cluster-leader
    - member: runner-2
      role: cluster-worker
    - member: runner-3
      role: cluster-worker
```

Walk each section.

### `environment` — naming and binding

```yaml
environment:
    label: staging-east-leader-follower
    landscape: staging-east
```

Two fields:

- `label` — the human-readable identity of this Topology.  Appears
  in logs, in result reports, in time-travel debugging sessions.
  Distinct from the Landscape's label so that "ran against
  Landscape `staging-east` with Topology
  `staging-east-leader-follower`" is unambiguous in retrospective
  analysis.
- `landscape` — which Landscape this Topology applies to.  The
  loader does not enforce the binding (a single Topology file can
  describe relationships that apply across multiple Landscapes); the
  field is documentary, telling a reader which Landscape the
  Topology was *designed* against.  Practically, the
  cross-validation step at orchestration startup checks that every
  member name referenced in `clusters`, `dependencies`, and `roles`
  exists in the resolved Landscape.

### `clusters` — members that act together

A cluster is a named group of Landscape members that the
orchestration treats as a logical unit.  The
[`LandscapeDeviceCluster`](https://github.com/code-craft-ai-company/codecraftai-landscaping/blob/main/source/packages/ccai/landscaping/landscapedevicecluster.py)
class in `codecraftai-landscaping` is the runtime model the
orchestration interacts with; the Topology file is the
configuration that produces the runtime cluster.

```yaml
clusters:
    - name: runner-cluster
      members:
          - runner-1
          - runner-2
          - runner-3
      coordinator: runner-1
      cardinality:
          min: 2
          max: 4
```

Four fields per cluster:

- `name` — the cluster's identifier; orchestrations refer to
  clusters by this name.
- `members` — a list of Landscape member names.  These must resolve
  in the bound Landscape.
- `coordinator` (optional) — the member that plays the coordinator
  role for this cluster.  If unset, the cluster is leaderless.  When
  set, the orchestration's coordinator-aware steps use this field
  rather than electing one at runtime.
- `cardinality` (optional) — soft and hard bounds on the cluster
  size.  `min` is the threshold below which the cluster is
  considered failed; `max` is informational and lets growth-aware
  steps know when to stop adding members.  Either may be omitted.

A Landscape member can belong to multiple clusters in the same
Topology — `runner-1` could be in both a `runner-cluster` and a
`monitoring-cluster` if the orchestration treats it both as a worker
and as a metrics collector.  The discipline does not prohibit this;
it merely requires that each cluster name resolves to a clear
membership.

### `dependencies` — directional ordering

A dependency is a one-directional "X requires Y" relationship
between Topology entities.  Members, services, *and clusters* can
all appear on either side:

```yaml
dependencies:
    # smoke-tests can't run until staging-api and staging-db are live
    - source: smoke-tests
      requires:
          - staging-api
          - staging-db

    # staging-api can't start until staging-db is live
    - source: staging-api
      requires:
          - staging-db

    # the runner cluster (the cluster as a whole) can't operate
    # until staging-api is reachable
    - source: runner-cluster
      requires:
          - staging-api
```

The orchestration uses dependencies for two things:

1. **Sequencing.**  Steps that operate on `staging-api` are not
   started until steps that bring `staging-db` to a ready state have
   completed.  The orchestration's flow engine resolves the
   dependency graph and emits a topologically-sorted execution
   order.  This is what "the flat list of steps becomes an *output*
   of topology resolution" from the chapter's opening means.

2. **Tear-down.**  When the orchestration ends, dependencies run in
   reverse: tear-down of `staging-db` is gated on tear-down of
   `staging-api` (so the API isn't left dangling without its
   database).  Implicit tear-down ordering through hand-maintained
   shutdown sequences is one of the most error-prone patterns in
   pipeline engineering; making the graph explicit here puts the
   graph through code review.

The graph must be **acyclic**.  A cyclic dependency
(A requires B; B requires A) is a configuration error caught at
load time.  The error message names the cycle so the engineer can
break it.

### `roles` — topology-specific assignments

A Landscape member already carries a `role` field — `taskerserver`,
`runner`, `dut`.  That role is the member's *Landscape-time*
identity, the thing that's true regardless of which orchestration
runs.  A Topology overlay can introduce a *Topology-time* role that
varies by orchestration:

```yaml
roles:
    - member: runner-1
      role: cluster-leader
    - member: runner-2
      role: cluster-worker
    - member: runner-3
      role: cluster-worker
```

The same `runner-1` is `runner` in the Landscape and `cluster-leader`
in this Topology.  In a different Topology
(`topology-staging-east-flat.yaml`) the same `runner-1` is just
`worker`, with no leadership status.

The role vocabulary is application-defined.  The framework makes no
distinction between `cluster-leader` and `quorum-witness` and
`canary-runner`; those are names the orchestration code recognises
and acts on.  The Topology file is where the names are bound to
members.

A common usage pattern: orchestration code queries the resolved
Topology for "all members with role X" and dispatches work
accordingly.  When `cluster-leader` work needs to happen, the code
asks the Topology, gets `runner-1`, and routes the call.  The
Landscape is unchanged.

## A complete worked example

Here is a Topology that pairs with the staging-east Landscape from
chapter 5.  The Landscape had three runners and two services
(`staging-db`, `staging-api`).  This Topology arranges them as a
leader-follower runner cluster with a clear database-then-API
ordering:

```yaml
# topology-staging-east-leader-follower.yaml
environment:
    label: staging-east-leader-follower
    landscape: staging-east

clusters:
    - name: runner-cluster
      members:
          - runner-1
          - runner-2
          - runner-3
      coordinator: runner-1
      cardinality:
          min: 2
          max: 3

dependencies:
    # services come up in dependency order:
    # staging-db must be live before staging-api starts
    - source: staging-api
      requires:
          - staging-db
    # the cluster needs the api before any work happens
    - source: runner-cluster
      requires:
          - staging-api
    # smoke tests run last, after everything else is live
    - source: smoke-tests
      requires:
          - runner-cluster
          - staging-api
          - staging-db

roles:
    - member: runner-1
      role: cluster-leader
    - member: runner-2
      role: cluster-worker
    - member: runner-3
      role: cluster-worker
```

The orchestration that loads this Topology obtains a graph it can
walk:

```
staging-db ─────► staging-api ─────► runner-cluster
       │                  │                  │
       └──────────────────┴──────────────────┴──────► smoke-tests
```

The flow engine's startup sequence is `[staging-db, staging-api,
runner-cluster, smoke-tests]`.  The tear-down sequence is the
reverse.  Neither was hand-maintained.

Now consider the alternate Topology — same Landscape, flat
arrangement:

```yaml
# topology-staging-east-flat.yaml
environment:
    label: staging-east-flat
    landscape: staging-east

clusters:
    - name: runner-cluster
      members:
          - runner-1
          - runner-2
          - runner-3
      cardinality:
          min: 1   # any one runner is enough

dependencies:
    - source: smoke-tests
      requires:
          - runner-cluster
          - staging-api
          - staging-db

roles:
    - member: runner-1
      role: cluster-worker
    - member: runner-2
      role: cluster-worker
    - member: runner-3
      role: cluster-worker
```

No coordinator.  Lower cardinality threshold.  All members play the
same role.  The Landscape file is identical; only the Topology file
differs.  The pipeline runs against either by setting
`AM_TOPOLOGY_NAMES=staging-east-flat` or
`AM_TOPOLOGY_NAMES=staging-east-leader-follower` — no other change.

## Composition with the rest of the categories

Topology references Landscape members by name.  It is *referenced
by* nothing else; the resolved Topology is consumed by the
orchestration's flow engine and by orchestration-aware code
querying for roles and clusters.

```
Landscape  ◄──── references ──── Topology
   │                                 │
   │                                 │
   │ also referenced by              │ consumed by
   │                                 │
   ▼                                 ▼
Data Profiles, Credentials      Orchestration flow engine
                                Cluster-aware orchestration code
                                Role-based dispatch
```

The references go one way: Topology to Landscape.  This is what
makes Landscape the structural truth and Topology a layered overlay.
Changing a Topology never invalidates a Landscape; changing a
Landscape may invalidate the Topologies built on top of it (the
cross-validation step at startup catches dangling member references).

The Runtime profile (chapter 7) parameterises *how* the
orchestration walks the Topology — timeouts on dependency-readiness
checks, parallelism within cluster operations — without altering the
Topology.  Same Topology, different Runtimes give different
operational behaviour over the same relationship graph.

Credentials (chapters 9–12) are referenced by the Landscape, not
the Topology.  Topology is the relationship layer; access control
remains a Landscape-and-credential concern.

## Multiple Topologies per Landscape: the architectural payoff

The volume's earlier chapters argued for separation of concerns
across the six categories.  Topology is the chapter where the payoff
is most concrete.

Without Topology files, an orchestration that needs leader-follower
behaviour today and flat behaviour tomorrow has two options:

1. **Two Landscape files** that duplicate the inventory but differ
   in some structural field (`role` per client).  This forces every
   change to the underlying inventory — a host's IP changes, a
   credential rotates — to be made twice.  Eventually the two files
   drift, and one orchestration is run against a stale copy.

2. **A single Landscape with a code-side switch** (an environment
   variable, a CLI flag, a hardcoded conditional) that selects
   between leader-follower and flat behaviour.  This is the Chapter
   2 "conditional thicket" failure mode in fresh costume.

Topology files make this a non-problem.  One Landscape, multiple
Topologies, the relationship choice expressed as data and selected
by name.  An orchestration that needs to support both modes selects
between them by changing one line in the CI workflow:

```yaml
# .gitlab-ci.yml — leader-follower run
integration:
  variables:
    AM_LANDSCAPE_NAMES: "staging-east"
    AM_TOPOLOGY_NAMES: "staging-east-leader-follower"
    AM_RUNTIME_NAMES: "fast-ci"

# .gitlab-ci.yml — flat run
integration:
  variables:
    AM_LANDSCAPE_NAMES: "staging-east"
    AM_TOPOLOGY_NAMES: "staging-east-flat"
    AM_RUNTIME_NAMES: "fast-ci"
```

Three categories each named by their `AM_*_NAMES` variable, each
selectable independently.  This is the configurations-as-content
discipline from chapter 7 applied to the relationship layer.

## What Topology is *not*

Three things to keep out of a Topology file.  Each will be
tempting; resist all three.

### Not a Landscape

Topology references Landscape members; it does not declare them.  A
Topology that contains a `host: 192.167.1.21` field for a member is
a Topology that has absorbed Landscape data.  Resist.  The reason
is the same as in the previous chapter — when one file is the
source of truth for inventory, every other file references it,
without duplication.

If a member appears in a Topology but not in the bound Landscape,
the cross-validation step at startup raises a `ConfigurationError`
naming the missing member.  This is the contract that lets readers
trust a Topology in isolation: every reference resolves, or load
fails loudly.

### Not a workflow

Topology says *what depends on what*; the workflow (chapter 20)
says *what to do, in what order, with what fallbacks*.  These are
related but distinct: the dependency graph from a Topology is an
*input* to workflow planning, not the workflow itself.

A Topology that contains imperative steps (`- run: deploy.sh`) has
crossed into workflow territory.  Resist.  The workflow YAML is
where steps live; Topology is purely declarative relationship data.

### Not a runtime configuration

Topology says *what the relationships are*; runtime says *how the
orchestration treats them*.  A `timeout_seconds` field on a cluster
is misplaced; the timeout applies to *how the orchestration verifies
cluster readiness*, which is runtime.  The Topology should say
"cluster `runner-cluster` exists with these members"; the runtime
profile should say "give each cluster-readiness probe 30 seconds."

The test of "does this belong in the Topology?" is the same as for
Landscape: *would two different orchestration runs against the same
relationship graph need different values of this field?*  If yes,
it's runtime; if no, it's structural and belongs in the Topology.

## Migration: from implicit step ordering to explicit Topology

A pipeline that has accumulated step ordering through hand-maintained
sequences migrates to a Topology in three steps:

1. **Extract the dependency graph.**  Walk the existing pipeline's
   step list and identify the implicit dependencies.  When step *N*
   is "wait for the database to be ready," that's a dependency from
   step *N+1* to a `staging-db` Landscape entry.  When step *M* is
   "deploy the API after the database," that's a dependency from
   `staging-api` to `staging-db`.

2. **Identify the clusters and roles.**  Wherever the existing
   pipeline assumes that some hosts coordinate or that one host is
   the leader, those assumptions become cluster definitions and
   role assignments in the Topology.

3. **Replace the step list with workflow steps that consume the
   Topology.**  The `taskplus` workflow (chapter 20) goes from a
   list of explicit ordering to a set of steps each of which
   declares its required Topology entities; the FlowEngine sequences
   them automatically.

After migration, the pipeline's ordering is no longer imperative.
Adding a step is a Topology change ("smoke-tests requires
runner-cluster") plus a workflow change ("here's the smoke-tests
step"); the position of the step in the execution order is computed
from the graph, not specified by hand.

## Where Topology doesn't fit

Honest scoping notes:

- **Trivial single-step pipelines.**  A pipeline with three
  sequential steps and no branching does not need an explicit
  Topology.  The discipline applies when relationships are
  non-trivial — multiple participants, multiple dependencies, role
  variation across runs.
- **Fully ad-hoc workflows.**  A one-time investigation or a
  developer's debug session does not need a Topology file.  The
  loader's `default-topology` fall-through (which produces an empty
  graph) handles this case — the orchestration runs without Topology
  constraints, and the workflow's explicit step ordering carries
  the day.
- **Streaming graph generation.**  Topologies that change shape
  every few seconds — autoscaling clusters where membership shifts
  during a run — strain the file model.  The pattern there is to
  *generate* a snapshot Topology at orchestration start (from the
  service-discovery output), then treat it as the Topology for the
  run.  Out of scope for this chapter; consistent with the
  discipline.

The discipline scales from "no relationships, single host" to "many
participants with varied relationships across multiple Topologies."
Above that scale, Topologies compose through the standard `MergeMap`
overlay (chapter 14), the same way Landscapes do.

## Summary

- A **Topology** is the relationships between Landscape members,
  expressed as a separate file because the same physical inventory
  can host multiple relationship arrangements.
- The proposed schema has three concerns: **clusters** (members that
  act together, optionally with a coordinator and cardinality
  bounds); **dependencies** (one-directional "X requires Y"
  relationships used for sequencing and tear-down); **roles**
  (Topology-time role assignments distinct from Landscape-time
  roles).
- The configuration plumbing for Topology — `MergeMap` loading,
  `AM_TOPOLOGY_NAMES`, the `TopologyIntegrationLayer` — is shipped;
  the schema content is consumer-defined, and this chapter
  proposes the canonical shape.
- Topology references Landscape members by name; the cross-
  validation step at startup catches dangling references.  No data
  flows from Topology back into Landscape.
- The architectural payoff: one Landscape can host many Topologies.
  Switching between leader-follower and flat behaviour is a
  one-line change to `AM_TOPOLOGY_NAMES`, not a duplicated Landscape
  file.
- A Topology is not a Landscape (no inventory), not a workflow (no
  imperative steps), and not a runtime configuration (no behaviour
  knobs).
- Migration from implicit step ordering to explicit Topology is
  mechanical: extract the dependency graph, identify clusters and
  roles, replace the imperative step list with Topology-aware
  workflow steps.

The next chapter — already drafted — is the thesis chapter on
Runtime files vs Environment Variables.  After that, Part II
finishes with chapter 8 on Data Profiles.
