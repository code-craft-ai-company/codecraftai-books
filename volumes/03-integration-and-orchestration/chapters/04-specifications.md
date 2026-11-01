# Chapter 4 — Specifications

> *A pipeline that contains forty lines of `aws ec2 create-instance` is
> two pipelines pretending to be one.  The first is a provisioner; the
> second is the actual orchestration.  The Specification separates
> them.*

The previous chapter named six configuration categories.  This chapter
introduces the first one: **Specifications** — the declarative
description of the resources that must exist before an orchestration
runs.

Specifications are the only one of the six categories that does not
live inside `codecraftai-configuration`.  They live in
[`codecraftai-provisioning`](https://github.com/code-craft-ai-company/codecraftai-provisioning/tree/main/source/packages/ccai/provisioning).
The reason is structural: the other five categories are loaded *by*
the orchestration as it starts; Specifications are consumed *before*
the orchestration starts.  They describe a state of the world that the
orchestration assumes is true.

## The provisioner-orchestration confusion

Most CI pipelines that automate cloud-backed environments contain two
concerns smeared into one file: how to produce the environment, and
what to do with it once produced.  When the two are mixed, three
predictable failures follow:

1. **Recipes drift.**  The thirty lines of `aws ec2 create-instance`
   in pipeline A are subtly different from the thirty lines in
   pipeline B.  The differences are accidental, accumulated over
   months of small fixes nobody propagated across all three
   pipelines that need similar VMs.

2. **Re-runs are not idempotent.**  Imperative provisioning code is
   hard to make idempotent, because imperative code does not have a
   clean way to say "if the resource already exists in the right
   shape, leave it alone."  Re-running an imperative provisioning
   step on a partially-provisioned environment leaves a mess.

3. **The orchestration cannot run against a pre-existing environment.**
   When the imperative provisioning is part of the orchestration,
   you cannot easily say "skip provisioning today; the environment
   is already up."  The orchestration is welded to its provisioner.

The discipline of this chapter is to factor the two concerns apart.
The orchestration declares the resources it needs.  A separate
provisioning step makes the declaration true.  The orchestration
itself is unchanged whether the provisioning step ran or whether a
matching environment was already up.

## The Specification format

A Specification is a YAML file with two top-level sections:

```yaml
environment:
  # how to provision: where, in which account, with which credentials,
  # under which logical groupings
  venues: [...]
  groups: [...]

apod:
  # what to provision: the concrete devices that make up the
  # automation pod, each referencing a profile that describes the
  # resource shape

  devices: [...]

profiles:
  # the parameterised resource templates referenced by devices

  - {...}
  - {...}
```

The structure is parsed by
[`ProvisioningEngine.create_provisioning_engine(specification=...)`](https://github.com/code-craft-ai-company/codecraftai-provisioning/blob/main/source/packages/ccai/provisioning/provisioningengine.py)
which loads the YAML and instantiates four kinds of model objects.
The four are layered abstractions, not a flat list.  Understanding
the layering is the key to writing maintainable specs.

### Layer 1 — CloudVenue: where we provision

A
[`CloudVenue`](https://github.com/code-craft-ai-company/codecraftai-provisioning/blob/main/source/packages/ccai/provisioning/model/cloud/cloudvenue.py)
describes a target account in a cloud provider:

```yaml
environment:
  venues:
    - name: aws-staging
      provider: aws
      management_url: https://ec2.us-east-1.amazonaws.com
      subscription: "123456789012"   # AWS account ID
      region: us-east-1
      credential: ci-runner-aws
```

A venue says: *this is an AWS account in us-east-1, accessed with the
`ci-runner-aws` credential.*  The `credential` field is a *reference*
into the credentials category (chapters 9–12) — the venue does not
embed secret material; it names a credential the resolution layer
will look up at runtime.

Multiple venues can coexist in one Specification — you can provision
across AWS and Azure in a single spec — but the typical case is one
venue per spec, with multiple specs for multi-cloud setups.  Per
the per-provider provisioners shipped under
[`venues/{aws,azure,google}/`](https://github.com/code-craft-ai-company/codecraftai-provisioning/tree/main/source/packages/ccai/provisioning/venues),
the supported providers are AWS, Azure, and Google Cloud.

### Layer 2 — CloudGroup: how resources are organised

A
[`CloudGroup`](https://github.com/code-craft-ai-company/codecraftai-provisioning/blob/main/source/packages/ccai/provisioning/model/cloud/cloudgroup.py)
describes a logical grouping with associated tags:

```yaml
environment:
  groups:
    - name: ci-runner-pool
      description: "Worker pool for the integration test suite"
      tags:
        - key: cost-center
          value: engineering-test
        - key: lifecycle
          value: ephemeral
```

Groups are the unit of cleanup.  When the orchestration is finished
and the provisioning step is asked to tear down, it tears down by
group.  Devices that share a group share a fate.  Tags travel with
the devices into the cloud provider's billing and management
systems, so the cost-center attribution is automatic.

### Layer 3 — CloudResourceProfile: what kind of resource

A
[`CloudResourceProfile`](https://github.com/code-craft-ai-company/codecraftai-provisioning/blob/main/source/packages/ccai/provisioning/model/cloud/cloudresourceprofile.py)
is a parameterised template for resources of a particular shape:

```yaml
profiles:
  - name: runner-medium
    description: "Standard medium-sized integration runner"
    credential: ci-runner-aws
    parameters:
      instance_type: t3.medium
      image: ami-0abcd1234
      subnet: subnet-0123456789abcdef0
      security_groups:
        - sg-runner-default
      iam_profile: runner-instance-profile
      ebs_volumes:
        - device: /dev/sda1
          size_gb: 30
          type: gp3
    tags:
      - key: role
        value: runner
```

A profile is the *recipe*, but it is the recipe expressed as data,
not as imperative code.  The provisioner for the venue's provider
knows how to turn the parameters into the right calls — the AWS
provisioner reads `instance_type` and calls `RunInstances`; the
Azure provisioner reads the same parameter and calls
`compute.virtual_machines.begin_create_or_update`.  The same
profile, with provider-specific parameter blocks, can target either
cloud.

### Layer 4 — CloudDevice: which specific resources

A
[`CloudDevice`](https://github.com/code-craft-ai-company/codecraftai-provisioning/blob/main/source/packages/ccai/provisioning/model/cloud/clouddevice.py)
is a single instance to be provisioned, referencing a group, a
profile, and a role:

```yaml
apod:
  devices:
    - identifier: runner-1
      category: ec2-instance
      group: ci-runner-pool
      profile: runner-medium
      role: runner
      tags:
        - key: index
          value: "1"
    - identifier: runner-2
      category: ec2-instance
      group: ci-runner-pool
      profile: runner-medium
      role: runner
      tags:
        - key: index
          value: "2"
    - identifier: runner-3
      category: ec2-instance
      group: ci-runner-pool
      profile: runner-medium
      role: runner
      tags:
        - key: index
          value: "3"
```

Three devices, each pointing at the `runner-medium` profile, all in
the `ci-runner-pool` group.  The provisioner instantiates three
EC2 instances of the same shape, tags them per-device, and
collectively tears them down when the group is dismissed.

## A complete worked example

Here is the assembled Specification for a three-runner integration
pool, end to end:

```yaml
# spec-ci-runner-pool.yaml
environment:
  venues:
    - name: aws-staging
      provider: aws
      management_url: https://ec2.us-east-1.amazonaws.com
      subscription: "123456789012"
      region: us-east-1
      credential: ci-runner-aws
  groups:
    - name: ci-runner-pool
      description: "Worker pool for the integration test suite"
      tags:
        - key: cost-center
          value: engineering-test
        - key: lifecycle
          value: ephemeral

profiles:
  - name: runner-medium
    description: "Standard medium-sized integration runner"
    credential: ci-runner-aws
    parameters:
      instance_type: t3.medium
      image: ami-0abcd1234
      subnet: subnet-0123456789abcdef0
      security_groups:
        - sg-runner-default
      iam_profile: runner-instance-profile
      ebs_volumes:
        - device: /dev/sda1
          size_gb: 30
          type: gp3
    tags:
      - key: role
        value: runner

apod:
  devices:
    - identifier: runner-1
      category: ec2-instance
      group: ci-runner-pool
      profile: runner-medium
      role: runner
      tags: [{key: index, value: "1"}]
    - identifier: runner-2
      category: ec2-instance
      group: ci-runner-pool
      profile: runner-medium
      role: runner
      tags: [{key: index, value: "2"}]
    - identifier: runner-3
      category: ec2-instance
      group: ci-runner-pool
      profile: runner-medium
      role: runner
      tags: [{key: index, value: "3"}]
```

The file is consumed by:

```python
from ccai.provisioning.provisioningengine import ProvisioningEngine

engine = ProvisioningEngine.create_provisioning_engine(
    specification="config/specs/spec-ci-runner-pool.yaml"
)

# The engine has parsed the spec and instantiated:
#   1 CloudVenue   (aws-staging)
#   1 CloudGroup   (ci-runner-pool)
#   1 CloudResourceProfile (runner-medium)
#   3 CloudDevice  (runner-1, runner-2, runner-3)
```

The provisioning step, run before the orchestration, ensures three
EC2 instances exist in the `aws-staging` venue, in the
`ci-runner-pool` group, matching the `runner-medium` profile.  If
they already exist with matching identifiers and tags, the step is a
no-op.  If they exist with the wrong shape, the step replaces them.
If they don't exist, the step creates them.

The orchestration that runs *after* provisioning sees a Landscape
(chapter 5) populated with three runners.  The orchestration does
not know — and does not need to know — whether they were created
fresh or pre-existed.

## Idempotence is the design contract

The single most important property of a Specification is that
*applying it is idempotent*.  Apply the same Specification twice,
and the second apply is a no-op.

The reason this matters is operational.  In a healthy CI system,
many things go wrong: a network blip mid-provision, a partial
rollout, a cancelled job that left half a runner pool created.  The
recovery story for a non-idempotent provisioner is bespoke debugging
every time.  The recovery story for an idempotent provisioner is
*re-apply*.

Idempotence is achievable because the Specification is *declarative*.
The spec says "three EC2 instances should exist with these
properties."  The provisioner inspects the current state, computes
the diff, and converges.  Imperative provisioning code does not
admit this analysis.

A consequence of declarative-and-idempotent: a Specification can be
the source of truth for an environment's existence.  Re-running the
provisioner on a schedule keeps the environment in the desired
state, repairing drift.  This is the same operating model
Terraform, Pulumi, and other infrastructure-as-code systems use; the
`codecraftai-provisioning` package brings it inside the portfolio so
the same toolchain that runs the orchestration can also run the
provisioner.

## Composition with the rest of the categories

Specifications sit at the top of the configuration stack and feed
into the others:

```
   Specification
        │
        │ ProvisioningEngine.create_provisioning_engine
        │ (applied)
        ▼
   ┌────────────────────────────┐
   │  Live cloud resources      │
   │  (3 EC2 instances tagged   │
   │   ci-runner-pool / runner) │
   └──────────────┬─────────────┘
                  │
                  │ landscape generation
                  │ (or hand-authored landscape that
                  │  references the provisioned resources)
                  ▼
   ┌────────────────────────────┐
   │  Landscape                 │
   │  (apod members =           │
   │   runner-1, runner-2,      │
   │   runner-3)                │
   └──────────────┬─────────────┘
                  │
                  │ Topology references
                  │ landscape members
                  ▼
   ┌────────────────────────────┐
   │  Topology                  │
   │  (runner-cluster:          │
   │    members = runner-1,     │
   │      runner-2, runner-3)   │
   └────────────────────────────┘
```

The `apod` section in the Specification mirrors the `apod` section
in the Landscape (chapter 5).  This is intentional: the
provisioner's output is the landscape's input.  The shared `apod`
shape is the seam between the two stages of the pipeline.

The credentials referenced by the venue (`ci-runner-aws`) and by the
profile travel through the resolution layer (chapters 9–12) at
provisioning time and again at orchestration time.  The same
credential set is used by both — provisioning to create the
resources, orchestration to access them.

The runtime profile (chapter 7) parameterises *how* the provisioner
runs (timeouts, retry, log levels for the provisioning step itself)
without touching the Specification.  Same provisioner, same spec,
different runtime: useful for distinguishing production-grade
provisioning from local-dev rapid iteration.

## What Specifications is *not*

Three things to keep out of a Specification, every one of which a
team will be tempted to put in:

### Not the orchestration

A Specification is consumed *before* the orchestration runs.  It
describes the static shape of the environment.  It does not
describe what to do with the environment.  Things like "run smoke
tests after provisioning" or "deploy the application image" are
orchestration concerns; they belong in `taskplus` workflows
(chapter 20), not in the spec.

The boundary is sharp: the Specification's output is *resources
exist*.  The orchestration's input is *resources exist*.  The two
do not overlap.

### Not a build script

Building a container image, compiling code, packaging artifacts —
these are not provisioning.  Provisioning creates the substrate;
build steps run on top.  When a Specification grows a `build_image`
section, the section is misplaced; either the image is built once
out-of-band and referenced by the spec's `image: ami-0abcd1234`
field, or the build is its own orchestration step that produces an
artifact the spec consumes.

### Not a runtime config

The Specification carries *structural* data — what hardware shape,
what region, what security group.  It does not carry *behavioural*
data — log levels, retry counts, parallelism.  When a Specification
sprouts a `log_level` field, the field belongs in the runtime
configuration (chapter 7) for the orchestration that runs against
the provisioned environment.

The test of "does this belong in the spec?" is: *would the answer
to this question change if the orchestration ran twice against the
same environment?*  If the answer changes, it's runtime
configuration.  If the answer doesn't change, it's structural and
belongs in the spec.

## Migration: from inlined recipes to Specifications

A pipeline that has accumulated forty lines of `aws ec2 ...` migrates
to a Specification in three steps:

1. **Lift the recipe.**  Take the imperative provisioning shell and
   rewrite it as a Specification YAML.  Each `aws ec2 run-instances`
   call becomes a CloudDevice referencing a CloudResourceProfile.
   Hostnames and IPs that were hardcoded become outputs the
   provisioner returns.

2. **Replace the inline shell with a provisioner invocation.**  The
   pipeline's shell call becomes:
   ```bash
   codecraftai-provisioning apply --spec config/specs/spec-ci-runner-pool.yaml
   ```
   The migration is mechanical; the spec carries the same intent the
   shell carried.

3. **Generate the landscape from the provisioner output.**  The
   provisioner emits the apod section of a landscape file as part of
   its run.  The orchestration consumes the landscape, not the
   provisioner output directly.

After migration, the pipeline's provisioning shell is gone.  The
forty lines have moved to the Specification, which is now versioned,
diffable, idempotent, and shareable across pipelines.

## Where Specifications doesn't fit

Honest scoping notes:

- **Bare-metal labs.**  The portfolio's provisioning model assumes
  cloud APIs.  A lab full of physical hardware needs a different
  provisioning approach (PXE, Ansible against a static inventory, or
  the kind of orchestrated lab-power package
  [`codecraftai-interop`](https://github.com/code-craft-ai-company/codecraftai-interop) ships).  Treat the
  static inventory as the Landscape (chapter 5) and skip the
  Specification.
- **SaaS-only environments.**  If the environment is "an account in
  a SaaS product," there is no provisioner — the provisioning is
  manual, one-time, and out of band.  The spec doesn't apply; the
  orchestration consumes a Landscape that names the SaaS endpoints.
- **Local-dev environments.**  A developer running the pipeline
  against `docker-compose up` is not provisioning cloud resources.
  The Specification is empty (or omitted); the Landscape names the
  local containers.  The orchestration runs unchanged.

The discipline is robust to these cases: when there is no
Specification, the orchestration sees the Landscape directly.  The
spec layer is *additive* — it earns its keep when there's cloud
infrastructure to provision.

## Summary

- A **Specification** is a declarative manifest of the resources that
  must exist before an orchestration runs.
- It lives in `codecraftai-provisioning`, separate from the five other
  configuration categories that live in `codecraftai-configuration`,
  because it is consumed *before* the orchestration rather than *by*
  the orchestration.
- The format has four layered abstractions: **CloudVenue** (where),
  **CloudGroup** (how organised), **CloudResourceProfile** (what kind),
  **CloudDevice** (which specific instances).
- Applying a Specification is **idempotent**: apply twice, no-op the
  second time.  This is the design contract that makes Specs
  recoverable from partial failure.
- The Specification's output (live cloud resources) is the
  Landscape's input.  The shared `apod` shape is the seam.
- Specifications does *not* contain orchestration steps, build
  recipes, or runtime configuration.  Each has its own category.
- Migration from inlined CI shell to Specification is mechanical:
  lift the recipe, swap the inline call for `codecraftai-provisioning
  apply --spec ...`, generate the landscape from the provisioner.
- Where there's no cloud to provision (bare-metal, SaaS-only,
  local-dev), the discipline degrades gracefully — the
  Specification is empty or omitted, and the orchestration runs
  against a static Landscape directly.

The next chapter introduces the Landscape — the inventory of what's
in scope for a pipeline run, whether produced by a Specification or
hand-authored.
