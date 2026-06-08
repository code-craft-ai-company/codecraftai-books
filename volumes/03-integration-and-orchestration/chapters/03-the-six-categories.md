# Chapter 3 — The Contract: Six Typed Configuration Categories

> *Every parameter to every pipeline belongs in exactly one of six
> categories.  When you find yourself reaching for an environment
> variable, ask which of the six the value belongs in — and then put
> it there instead.*

The previous chapter named six failure modes of CI YAML and pointed
out that each one is the absence of a typed configuration category.
This chapter names the six categories.  The chapters in Part II
introduce them one at a time with file-format examples and worked
patterns.  The chapters in Part III dig into Credentials, the most
substantial of the six, in dedicated detail.  This chapter is the
table of contents for everything that follows in Parts II–III.

## The six categories at a glance

| Category | What it describes | What it answers | Implementation in the portfolio |
|---|---|---|---|
| **Specifications** | Resources to create or check out before the orchestration runs | What needs to *exist* — VMs, containers, accounts, repos, databases | `codecraftai-provisioning` (`ProvisioningEngine`) |
| **Landscapes** | The environment of hosts, devices, networks, services | What's *available* — the inventory under automation | `codecraftai-configuration` (Landscape category) + `codecraftai-landscaping` |
| **Topologies** | The relationships between landscape entities | How the inventory is *connected* — clusters, dependencies, ordering | `codecraftai-configuration` (Topology category) |
| **Runtimes** | How processes execute — timeouts, retry, parallelism, log levels, sentinels | How everything *runs* — the knobs | `codecraftai-configuration` (Runtime category) |
| **Credentials** | Authentication material, scoped to hosts and services | How things *authenticate* | `codecraftai-credentials` (vocabulary) |
| **Data Profiles** | Connection profiles for data stores | Where data *lives* | `codecraftai-configuration` (DataProfile category) + `codecraftai-dataprofiles` |

Five of the six categories are loaded by `codecraftai-configuration`.
Specifications is the exception — it is consumed by the provisioning
engine before the orchestration starts and is therefore handled by a
different package.  The discipline is unified at the orchestration
layer; the implementation has a seam at the provisioning boundary.

Each category is a separate concern.  They compose, but they do not
overlap.  A landscape file does not contain credentials; a credentials
file does not contain runtimes; a topology does not contain
infrastructure listings.  The separation is what makes them
independently versionable, swappable, and reviewable.

A note on a *seventh* file the reader will encounter in
`codecraftai-docs/userguide/03-07-user-file.rst`: **`user.yaml` is
not a seventh category.**  It is the **default Runtime overlay
for local-user runs** — the bottom layer of the runtime
composition stack on a developer laptop, with explicitly-
selected Runtime files merging on top.  CI runs do not include
`user.yaml`.  Functionally the User file is a Runtime; it is
documented separately because its *role* is operator-personal
(my preferences on my laptop), not orchestration-shared.

**A property worth naming up front: all six categories are
uniformly composable.**  Every category resolver in
`codecraftai-configuration` takes a *list* of names and a *list*
of files and merges them via `MergeMap` overlay; later names
override earlier names where keys collide.  Composability is
what enables profile composition (Part IV) — the same machinery
that overlays `UPNP Diagnostics + Bluetooth Diagnostics` for a
Runtime overlays `Checkout Resources + Environment Resources`
for a Landscape and `lab-wide + experiment-overrides` for
Credentials.  One pattern, six categories.

## What every category has in common

Five rules apply to all six categories:

1. **Typed file at rest.**  Each category is one or more YAML or JSON
   files.  The loader tries `.yaml`, `.yml`, `.json` in that order.
   This is implemented in `codecraftai-configuration`'s
   [`DirectorySource`](../../../codecraftai-configuration/source/packages/ccai/configuration/sources/directorysource.py)
   for filesystem sources; the same fallback applies to HTTP, CouchDB,
   and MongoDB sources.

2. **Lookup by name, not by path.**  Pipeline code says "use landscape
   `staging-east`," not "open `/etc/ccai/landscapes/staging-east.yaml`."
   The mapping from name to file (or to database document) is the
   loader's job.  This is what makes the same pipeline runnable in
   local-dev (where `staging-east` resolves to a file in `~/.ccai/`),
   in CI (where it resolves to an HTTP URL on a config service), and
   in a debug replay (where it resolves to a snapshot inside a tarball).

3. **Encryption is a property of the file, not the category.**  Any
   category can be Fernet-encrypted by setting an `"encrypted_content"`
   key with the ciphertext and a `"format"` field for the inner
   payload's format.  The loader transparently decrypts when given a
   key or keyphrase.  This is done at
   [`ConfigurationLoader`](../../../codecraftai-configuration/source/packages/ccai/configuration/configurationloader.py).

4. **Composition through MergeMap, not through string substitution.**
   Each category's resolved configuration is a
   [`MergeMap`](../../../codecraftai-xmodules/source/packages/ccai/xmodules/xcollections/mergemap.py),
   which holds an ordered list of source maps and composes them with
   overlay semantics — later maps shadow earlier maps key by key, but
   the lower-priority entries are preserved and inspectable.  This is
   what makes profile composition (chapter 14) work.

5. **Resolved into the application context.**  After loading, each
   category (except Credentials) is inserted into the shared
   serializable application context at a known
   [`ContextPath`](../../../codecraftai-this-app/source/packages/ccai/thisapp/contextpaths.py).
   This is how the configuration travels with the process and reaches
   distributed workers.  Credentials are deliberately *not* inserted
   into the context — that's a security boundary, covered in
   chapter 11.

## How the categories relate to each other

The six categories form a layered model.  Specifications create the
inventory.  Landscapes describe the inventory.  Topologies describe
the relationships within the inventory.  Runtimes parameterise the
processes that operate on the inventory.  Credentials authenticate
the processes to the inventory.  Data Profiles tell the processes
where to find data within the inventory.

```
                ┌──────────────────────────────────────────┐
                │  Specifications                          │
                │  "what should exist"                     │
                │  consumed by codecraftai-provisioning       │
                │  before the orchestration starts         │
                └──────────────────────┬───────────────────┘
                                       │ creates
                                       ▼
                ┌──────────────────────────────────────────┐
                │  Landscapes                              │
                │  "what's there"                          │
                │  hosts, devices, networks, services      │
                └──────────────────────┬───────────────────┘
                                       │
                ┌──────────────────────┴───────────────────┐
                │                                          │
                ▼                                          ▼
   ┌─────────────────────────┐         ┌──────────────────────────┐
   │  Topologies             │         │  Data Profiles           │
   │  "how it's connected"   │         │  "where data lives"      │
   │  clusters, dependencies │         │  per-store connection    │
   └─────────────────────────┘         │  profiles                │
                                        └──────────────────────────┘
                                       
                ┌──────────────────────────────────────────┐
                │  Runtimes                                │
                │  "how processes run"                     │
                │  timeouts, retry, log levels, parallelism│
                └──────────────────────────────────────────┘
                
                ┌──────────────────────────────────────────┐
                │  Credentials                             │
                │  "how to authenticate"                   │
                │  per-type, per-scope, per-lifetime       │
                └──────────────────────────────────────────┘
```

The two boxes at the bottom — Runtimes and Credentials — are
orthogonal to the layered model.  Runtimes apply to processes
regardless of which landscape they're in; credentials apply to access
attempts regardless of which topology is in scope.  They are
parameterisations *of the things that act on* the inventory, not
descriptions *of* the inventory.

## A first taste of each category

The chapters in Part II will introduce each category in detail.  As a
preview — just to fix mental images before the full treatment — here
is what each looks like as a YAML fragment:

### Specifications (chapter 4)

```yaml
# spec-runner-medium.yaml — declarative resource manifest
environment:
  region: us-east-1
  zone: us-east-1a
apod:
  name: ci-runner-pool
  members:
    - role: runner
      type: ec2
      size: t3.medium
      count: 3
      image: ami-0abcd...
```

A specification is consumed by `ProvisioningEngine.create_provisioning_engine(specification=...)`
before the orchestration runs.  The orchestration assumes the
specification's resources exist; the provisioning step makes that
assumption true.

### Landscapes (chapter 5)

```yaml
# landscape-staging-east.yaml — inventory of what's there
environment:
  label: staging-east
apod:
  clients:
    - name: ci-runner-1
      role: runner
      hostname: runner-1.staging.internal
infrastructure:
  services:
    - name: api
      hostname: api.staging.internal
      port: 8443
```

A landscape is loaded by name — `landscape-staging-east` — and
resolved into the
[`LANDSCAPE_CONFIGURATION_MAP`](../../../codecraftai-configuration/source/packages/ccai/configuration/configurationvariables.py).
Anywhere in the pipeline that needs to know "what hosts are
participating," the answer comes from the resolved map, not from a
hardcoded list.

### Topologies (chapter 6)

```yaml
# topology-staging-east.yaml — relationships
environment:
  label: staging-east
clusters:
  - name: api-cluster
    members: [api-1, api-2, api-3]
    coordinator: api-1
dependencies:
  - source: smoke-tests
    targets: [api-cluster]
  - source: api-cluster
    targets: [database]
```

A topology is loaded alongside a landscape; the landscape says what
exists, the topology says how things relate.  A pipeline that wants
to "tear things down in dependency order" reads the topology, not a
hand-maintained shutdown sequence.

### Runtimes (chapter 7)

```yaml
# runtime-fast-ci.yaml — the knobs
breakpoints: []
debugger: none
logging:
  levels:
    console: WARNING
    logfile: INFO
parallelism:
  max_workers: 8
timeouts:
  default_seconds: 30
property_test_caps:
  max_examples: 10
version: 1.0.0
```

A runtime profile names how processes run.  A different profile —
`runtime-full-sweep.yaml` — sets `max_examples` to unset and
`max_workers` to 1 for a thorough overnight run.  Same code, different
profile, different behaviour.  No environment variables.

### Credentials (chapter 9–12)

```yaml
# credentials-ci-runner.yaml — typed credentials
- identifier: ci-runner-ssh
  type: SshCredential
  username: runner
  keyfile: /run/secrets/ssh/ci-runner
  categories: [runner-fleet]
- identifier: ci-runner-aws
  type: AwsAccessKeyCredential
  keyid: AKIA...
  secret: ${VAULT:/secret/aws/ci-runner}
  categories: [aws-staging]
```

Each entry maps to one of the eleven shipped credential types in
`codecraftai-credentials`.  Resolving the secret value (the `${VAULT:...}`
expansion above) is the reader's responsibility — Part III explains
how to build the resolution layer the portfolio leaves to the
consumer.

### Data Profiles (chapter 8)

```yaml
# dataprofile-staging-db.yaml — connection profiles
- name: staging-db-readwrite
  type: SnowflakeProfile
  account: my-org
  warehouse: staging-wh
  database: app
  schema: public
  credential: staging-snowflake-rw
- name: staging-redis
  type: DatabaseBasicTcpProfile
  hostname: redis.staging.internal
  port: 6379
  credential: staging-redis-default
```

A data profile is the named connection.  The pipeline references
`staging-db-readwrite` by name; at runtime the profile is resolved,
the credential reference is resolved, and the connection opens.  No
URLs in code.  No hostnames in environment variables.

## What the rest of the volume does with this

Part II walks through Specifications, Landscapes, Topologies,
Runtimes, and Data Profiles in five chapters (chapters 4 through 8).
Part III gives Credentials four chapters of its own (chapters 9
through 12) because the credential discipline has more moving parts
than the other five combined.  Part IV (chapters 13 through 17) shows
how to use these categories together as profiles that span the
local-dev → CI → staging → prod ladder.  Part V (chapters 18 through
23) introduces `codecraftai-taskplus` as the workflow engine that
references these configurations by name from a portable workflow YAML.
Part VI (chapters 24 through 27) wires the workflow to a real CI
vendor and walks a worked port end-to-end.

The reader who has finished Part I has the discipline.  The rest of
the book is the implementation.

## Summary

- Six configuration categories — Specifications, Landscapes,
  Topologies, Runtimes, Credentials, Data Profiles — are the
  orchestration layer's typed inputs.
- Five live in `codecraftai-configuration`; Specifications lives in
  `codecraftai-provisioning` because it runs before the orchestration
  rather than alongside it.
- Each category has the same five properties: typed at rest, looked
  up by name, optionally encrypted, composed via `MergeMap`, resolved
  into the shared application context (Credentials excepted).
- The categories form a layered model: Specifications create the
  inventory; Landscapes describe it; Topologies describe its
  relationships; Runtimes and Credentials parameterise the processes
  that act on it; Data Profiles name the data inside it.
- Each subsequent chapter of Parts II and III takes one category and
  shows it in detail.
