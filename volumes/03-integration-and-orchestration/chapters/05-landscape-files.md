# Chapter 5 — Landscape Files

> *A test that runs against staging-db doesn't need to know that
> staging-db lives at db-staging-east-1.internal.  The test needs to
> know that staging-db exists.  Where it lives is a property of the
> Landscape, not of the test.*

The previous chapter ended with a Specification that produced three
EC2 instances tagged `runner` in a group called `ci-runner-pool`.
Three instances exist; the orchestration is about to start; the
orchestration needs to know which instances they are and how to
reach them.

That is the job of the Landscape.

A Landscape is the **inventory of what's in scope for a pipeline
run**.  It is the second of the six configuration categories, the
first that lives inside `codecraftai-configuration` (chapter 4's
Specification was the exception).  Every orchestration consumes
exactly one resolved Landscape — the resolution is the result of
loading one or more landscape files by name and merging them through
the standard
[`MergeMap`](https://github.com/code-craft-ai-company/codecraftai-xmodules/blob/main/source/packages/ccai/xmodules/xcollections/mergemap.py)
mechanism introduced in chapter 3.

## The structure of a Landscape file

A Landscape file has two top-level concerns: **who runs the
automation** and **what the automation runs against**.  These are the
`apod` and `infrastructure` sections respectively.  A third
top-level key — `environment` — names and labels the landscape
itself.

The simplest possible landscape:

```yaml
# landscape-minimal.yaml
environment:
    label: minimal

apod:
    clients: []

infrastructure:
    services: []
```

A label, no clients, no services.  This is a valid file — the
orchestration will load it, the resolved landscape will have zero
participants, and the orchestration's first job-resolution step will
report that the landscape is empty.  Useful as a starting template
or as a no-op landscape for syntax-only test runs; not useful for
actual work.

A real landscape — the one shipped at
[`codecraftai-automation-kit/userguide/examples/landscape.yaml`](https://github.com/code-craft-ai-company/codecraftai-automation-kit/blob/main/userguide/examples/landscape.yaml)
— populates both sections:

```yaml
# landscape.yaml — shipped example
environment:
    label: production

apod:
    clients:
    -   deviceType: network/client-linux
        name: casey-01
        host: 192.167.1.21
        role: taskerserver
        credentials:
            - pi-cluster
        features:
            isolation: false
        skip: false

    -   deviceType: network/client-linux
        name: casey-02
        host: 192.167.1.22
        role: taskerserver
        credentials:
            - pi-cluster
        features:
            isolation: false
        skip: false

    # ... casey-03, casey-04 ...

infrastructure:
    services:
        - serviceType: network/service
          name: someservice
          credentials:
              - adminuser
```

Four Linux clients participating in the automation pod, all sharing
the `pi-cluster` credentials, all serving the `taskerserver` role;
one infrastructure service named `someservice` reachable through the
`adminuser` credentials.  That is the landscape under which the
orchestration will run.

The two-section split is deliberate.  **`apod`** (the
**automation pod** — yes, this is where the word *Pod* in the
"Pod model" comes from; Volume 6 chapter 9 walks the runtime
realisation in detail) is the hosts that *participate in* the
automation: runners, drivers, controllers.  **`infrastructure`**
is the systems that the automation *operates against*: services
under test, supporting databases, fixtures.  The distinction
answers a real operational question — *which hosts can the
orchestration reboot, reset, or re-provision without disrupting
itself?*  Answer: only `apod` hosts.  An orchestration that
tried to reboot an `infrastructure` host without coordination
would interrupt the very service its tests are checking; an
orchestration that tried to leave an `apod` host alone forever
could never recover from a runner crash.  The two answers
require two sections.

This Landscape, paired with a Topology that says how these
resources are organised (chapter 6), realises into a **Pod** at
runtime — the actual collection of interoperating apod clients
and infrastructure services a job runs against.  Volume 6
chapter 9 walks the realisation; this chapter ends at the
declaration.

## The `apod` — who runs the automation

The **apod** (automation pod) names the hosts that *participate in
the automation*.  These are the runners, the drivers, the controllers
— the machines doing the work.  Each apod client has a small set of
fields:

| Field | Purpose | Example values |
|---|---|---|
| `deviceType` | Classification path identifying what kind of device this is | `network/client-linux`, `network/client-windows`, `serial/device-arm` |
| `name` | Friendly identifier the orchestration uses to refer to this client | `casey-01`, `runner-east-1`, `dut-mac-mini-3` |
| `host` | The reachable address (IP or DNS name) | `192.167.1.21`, `runner-east-1.staging.internal` |
| `role` | What this client *does* in this landscape | `taskerserver`, `runner`, `coordinator`, `dut` |
| `credentials` | List of credential identifiers used to reach the client | `[pi-cluster]`, `[ci-runner-ssh, ci-runner-aws]` |
| `features` | Per-host feature flags | `{isolation: false}`, `{hardware_acceleration: true}` |
| `skip` | Whether to exclude this client from the current run | `false`, `true` |

Three of these — `deviceType`, `role`, `credentials` — are the
points where the landscape composes with other categories:

- `deviceType` selects which device-handler implementation will be
  instantiated to drive this client.  The handler is registered
  through `codecraftai-extension`'s factory protocol; the landscape
  says "make me a `network/client-linux`," and the factory machinery
  resolves to the right concrete class.
- `role` is application-defined.  It exists for the orchestration
  to filter clients by purpose ("give me all `taskerserver` clients
  in the pod").  The role vocabulary is yours; the framework does
  not constrain it.
- `credentials` references credential identifiers (chapters 9–12).
  The credential material lives in a credentials configuration; the
  landscape names what to look up, not what to use.

The `skip: false` field is the on-the-fly inclusion control.  A
landscape with one client marked `skip: true` is a shorter pod for
this run without the client being deleted from the file.  Useful
during incidents — exclude a flaky runner from one CI run without
permanently removing it.

## The `infrastructure` — what the automation runs against

The **infrastructure** names the *systems under test*: the
applications, services, databases, devices, anything the apod
*operates on* during the run.  Each infrastructure entry has a
similar but smaller field set:

| Field | Purpose | Example values |
|---|---|---|
| `serviceType` | Classification path identifying what kind of service this is | `network/service`, `database/postgres`, `web/api` |
| `name` | Friendly identifier the orchestration uses to refer to this service | `someservice`, `staging-api`, `metrics-collector` |
| `credentials` | List of credential identifiers used to access the service | `[adminuser]`, `[staging-api-token]` |

Service entries can carry additional fields the
`serviceType`-specific handler knows how to read — a
`database/postgres` service might carry `hostname`, `port`,
`database`, `schema` fields; a `web/api` service might carry
`endpoint`, `version`, `auth_scheme` fields.  The shipped example
keeps the fields minimal because the example is illustrative.  The
schema for each `serviceType` is owned by the handler; the landscape
file is a thin envelope.

The deliberate separation between `apod` and `infrastructure` is the
chapter's central architectural claim.  A test runner is not a
system under test, even when both run on the same kind of hardware.
The runner's job is to *operate*; the service's job is to be
*operated upon*.  When a single configuration tries to describe
both without distinguishing them, the orchestration cannot answer
the question "which hosts can I freely reboot, and which hosts must
I never touch?"  With the separation, the answer is structural:
apod members are reboot-safe, infrastructure members are not.

## The `environment` block

The third top-level section, `environment`, names the landscape
itself:

```yaml
environment:
    label: staging-east
```

The `label` is the human-readable identity of the landscape: it
appears in logs, in result reports, in time-travel debugging
sessions.  When an orchestration runs against three different
landscapes through the year, the labels distinguish the runs in
historical analysis.

The `environment` block can carry additional metadata fields
(region, datacenter, cost-center) that aren't fixed by the framework
but travel with the landscape into observability systems.  Like the
service-type extension fields, these are owned by the consuming
code, not by the landscape format.

## The `codecraftai-landscaping` runtime model

The Landscape file is the configuration; the
[`Landscape`](https://github.com/code-craft-ai-company/codecraftai-landscaping/blob/main/source/packages/ccai/landscaping/landscape.py)
class in `codecraftai-landscaping` is the runtime model the
orchestration interacts with.  Volume 6 of this series covers the
runtime model in depth; Chapter 5 of this volume cares only about
the file-to-runtime boundary.

In the runtime model, each apod `client` becomes a
[`LandscapeDevice`](https://github.com/code-craft-ai-company/codecraftai-landscaping/blob/main/source/packages/ccai/landscaping/landscapedevice.py),
each infrastructure `service` becomes a
[`LandscapeService`](https://github.com/code-craft-ai-company/codecraftai-landscaping/blob/main/source/packages/ccai/landscaping/landscapeservice.py),
and the whole landscape is wrapped in a
[`Landscape`](https://github.com/code-craft-ai-company/codecraftai-landscaping/blob/main/source/packages/ccai/landscaping/landscape.py)
singleton with checkin / checkout primitives for safe coordinated
access.  The mapping is one-to-one: every entry in the YAML produces
exactly one runtime object.  No magic, no inference.

## A complete worked example

Building on the Specification from chapter 4 — three EC2 instances
tagged `runner` in `ci-runner-pool` — here is the corresponding
Landscape that consumes the provisioned resources:

```yaml
# landscape-staging-east.yaml
environment:
    label: staging-east
    region: us-east-1
    cost_center: engineering-test

apod:
    clients:
        - deviceType: network/client-linux
          name: runner-1
          host: ec2-runner-1.staging.us-east-1.internal
          role: runner
          credentials:
              - ci-runner-ssh
          features:
              parallelism_max: 4
          skip: false

        - deviceType: network/client-linux
          name: runner-2
          host: ec2-runner-2.staging.us-east-1.internal
          role: runner
          credentials:
              - ci-runner-ssh
          features:
              parallelism_max: 4
          skip: false

        - deviceType: network/client-linux
          name: runner-3
          host: ec2-runner-3.staging.us-east-1.internal
          role: runner
          credentials:
              - ci-runner-ssh
          features:
              parallelism_max: 4
          skip: false

infrastructure:
    services:
        - serviceType: database/postgres
          name: staging-db
          host: staging-db.us-east-1.internal
          port: 5432
          database: app_staging
          credentials:
              - staging-db-readwrite

        - serviceType: web/api
          name: staging-api
          endpoint: https://api-staging.us-east-1.internal
          version: v2
          credentials:
              - staging-api-token
```

Three runners in the apod, two services in the infrastructure.  The
hostnames and IPs are pinned to specific machines; the credential
identifiers are pinned to specific credential records the resolution
layer (chapter 10) will look up.  Nothing in this file embeds a
hostname in code, a credential in plaintext, or a behaviour knob
(parallelism, log level, retry count) where it doesn't belong.

The orchestration that loads this landscape sees a populated
`Landscape` object with three devices (each a `LandscapeDevice`) and
two services (each a `LandscapeService`).  The orchestration's
test-job resolution decides which devices to use for which steps;
the orchestration's data-access layer routes through the services.
The orchestration code itself is unchanged whether it runs against
staging-east, staging-west, or local-dev — only the landscape file
selected by `AM_LANDSCAPE_NAMES` differs.

## Multiple landscapes per project

The Landscape category is plural by design.  A real automation
project will have several landscape files, each named for a
deployment target:

```
config/landscapes/
├── landscape-local-dev.yaml          # one runner, mocks for services
├── landscape-staging-east.yaml       # three runners, real staging services
├── landscape-staging-west.yaml       # mirror of east, different region
├── landscape-staging-canary.yaml     # smaller pod for canary deploys
├── landscape-prod-readonly.yaml      # read-only access for prod debugging
```

The orchestration selects one by name through the `AM_LANDSCAPE_NAMES`
environment variable (the fixed-interface principle from chapter 7).
The plumbing — code, Dockerfiles, CI YAML — references the variable;
the variable's value names the file; the file describes the
landscape.  Adding a new landscape is a single-file change.

A landscape can also be *generated* rather than hand-authored.  Per
the Specification → Landscape seam from chapter 4, a provisioner can
emit the `apod.clients` section directly from the resources it
created.  Hand-authored landscapes describe pre-existing
environments; generated landscapes describe environments the
provisioner brought into existence for this run.  The orchestration
does not distinguish the two.

## Composition with the rest of the categories

The Landscape sits at the inventory level.  Other categories layer
on top of it:

```
                  Landscape
            (who and what; structural)
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   Topology    Data Profiles    Credentials
   (chapter 6)  (chapter 8)     (chapters 9-12)

       │              │              │
       │ references   │ references   │ resolves
       │ landscape    │ landscape    │ credential
       │ members      │ members      │ identifiers
       │              │              │
       └──────────────┴──────────────┘
                       │
                       ▼
              Runtime profile
              (chapter 7)
              parameterises how
              orchestration uses
              the layered model
```

- **Topology (chapter 6)** says how landscape members relate.  The
  topology references members by name (`runner-1`, `staging-db`).
  Without the landscape, the topology has nothing to refer to.
- **Data Profiles (chapter 8)** carry the connection details for
  data stores reachable in the landscape.  A profile references a
  landscape service by name; the service's `host` and `port` plus
  the profile's connection options yield a connection.
- **Credentials (chapters 9–12)** are referenced by every layer.
  The landscape names credential identifiers; the resolution layer
  produces materialised credentials at runtime; the orchestration
  uses them.
- **Runtime profile (chapter 7)** parameterises the orchestration's
  *use* of the layered model — timeouts, retry, parallelism — but
  does not alter the model itself.

The composition is one-directional.  The landscape is the structural
truth; everything else references it.  Changing a landscape ripples
into the orchestration only at the seams where the orchestration
queries the landscape (which client to run on, which service to call
into).  Topologies, profiles, and credentials referencing the
landscape need their own updates; this is the cost of explicit
references and it is paid up-front rather than as a runtime
mystery.

## What a Landscape is *not*

The Landscape file is restricted by design.  Three things to keep
out:

### Not a topology

Landscape says *what's there*; topology says *how it's connected*.
A landscape lists three runners; a topology over that landscape says
"runner-1 is the cluster coordinator; runner-2 and runner-3 join the
cluster runner-1 forms."  The cluster relationship is not in the
landscape file.

A common mistake: adding a `cluster_members` field to a landscape
client.  Resist this.  When the relationship between two clients
varies by test scenario — runner-1 coordinates in scenario A but is
a worker in scenario B — embedding the relationship in the
landscape makes the same physical landscape need two different
files.  Topology files (chapter 6) are the place for varying
relationships.

### Not a runtime configuration

Landscape says *what's there*; runtime says *how processes run*.  A
landscape lists three runners; a runtime profile says "give the
integration job a 30-second default timeout."  The timeout is not in
the landscape file even though it controls how the landscape is
exercised.

The test of "does this belong in the landscape?" is: *would two
different orchestration runs against the same physical environment
need different values of this field?*  If yes, it's runtime; if no,
it's structural and belongs in the landscape.

### Not a credential store

Landscape says *what credentials to look up*, by identifier.  It
does not store credential material.  A common mistake is to embed a
plaintext password as a `password:` field on a client.  Resist this
forever.  The chapters of Part III explain the credential resolution
discipline; the landscape's role is to name credentials, not to
contain them.

## Migration: from hardcoded hostnames to Landscape references

A pipeline that has accumulated hostnames in shell scripts, in test
fixtures, and in CI YAML migrates to a Landscape in three steps:

1. **Extract the inventory.**  Walk every place in the codebase that
   references a hostname.  Group them: test runners on one side,
   systems under test on the other.  Each group becomes one or more
   landscape entries.

2. **Replace direct references with name lookups.**  Code that did
   `requests.get("https://api-staging.us-east-1.internal/v2/...")`
   now does
   `requests.get(f"{landscape.get_service('staging-api').endpoint}/...")`.
   The orchestration code asks the resolved landscape for the
   service it wants by name; the landscape returns the current host
   and port.

3. **Add per-environment landscape files.**  Once one
   landscape exists, add a `landscape-local-dev.yaml` for the
   developer running the same code against local mocks; add a
   `landscape-staging-west.yaml` for the failover region.  The
   pipeline now runs against any of them by setting
   `AM_LANDSCAPE_NAMES`.

After migration, hostnames in code are gone.  Hostnames in CI YAML
are gone.  When a service moves — staging-api migrates from
us-east-1 to us-east-2 — the change is one line in one landscape
file.

## Where Landscapes don't fit

Honest scoping notes:

- **Trivial single-host scripts.**  A one-off script that just runs
  on the developer's machine doesn't need a landscape.  The
  discipline applies when the same code needs to run against
  multiple environments or with multiple participants.
- **Highly dynamic environments.**  Landscapes that change shape
  every few seconds — autoscaling pools, ephemeral build farms,
  event-driven worker spawning — strain the file model.  The
  portfolio's pattern for this is to *generate* the landscape from
  a service-discovery query at orchestration start, then treat the
  generated file as the inventory for the rest of the run.  This is
  out of scope for this chapter but is consistent with the
  discipline.
- **Configuration that's truly global.**  A "fleet manager"
  configuring thousands of devices across a global deployment is
  beyond what a single landscape file expresses well.  The pattern
  there is to compose multiple landscape files (regional fleets,
  composed at the `MergeMap` layer) — see chapter 14 on profile
  composition.

The discipline scales from one-host local-dev to dozens of
hosts-across-regions cleanly.  Above that scale, the *file* gives
way to file-of-files via composition; the underlying model
continues to apply.

## Summary

- A **Landscape** is the inventory of what's in scope for an
  orchestration run.  It has two participant categories — the
  **apod** (who runs the automation) and the **infrastructure**
  (what the automation runs against) — separated deliberately.
- Apod clients carry `deviceType`, `name`, `host`, `role`,
  `credentials`, `features`, and `skip` fields.  Infrastructure
  services carry `serviceType`, `name`, and `credentials` plus
  service-type-specific extension fields.
- Credential identifiers in landscape entries are *references*; the
  credential material lives in the credential category and is
  resolved at runtime.
- Landscapes are plural by design — one file per environment.  The
  orchestration selects by name through `AM_LANDSCAPE_NAMES`.
- A Landscape does not contain topology (how things relate), runtime
  configuration (how processes run), or credential material (the
  secrets themselves).  Each has its own category.
- Migration from hardcoded hostnames is mechanical: extract the
  inventory, replace direct references with name lookups, add
  per-environment landscape files.
- The runtime model in `codecraftai-landscaping` produces one
  `LandscapeDevice` per apod client and one `LandscapeService` per
  infrastructure service.  The mapping is one-to-one; the file is
  the structural truth.

The next chapter introduces Topology — the relationships between
landscape members, expressed as a separate file so that a single
physical landscape can host multiple orchestration scenarios.
