# Automojo Books — Volumes and Topics

Foundation document for the planned book series drawn from the `automojo-*`
software portfolio.  The series is organised into eight numbered volumes plus
one appendix volume, each cleanly mapped to a product line in the portfolio
or to a cross-cutting discipline that the portfolio embodies.

This document is the working blueprint.  It covers, for every volume:

- A working title and subtitle.
- The intended audience.
- Why the volume earns a dedicated book (rather than a chapter elsewhere).
- A chapter-level table of contents, organised into Parts where the volume
  is large enough to need them.
- Estimated page count.
- Cross-references to other volumes.
- Notes on portfolio source material that makes the volume unusually
  book-shaped (commits, audit transcripts, tests already in place).

The series-level material (overview, voice, glossary, decision records, open
questions) follows the per-volume content.

---

## Series overview

### Vision

The `automojo-*` portfolio is an opinionated platform for building durable
automation software in Python.  Every package follows a uniform skeleton, a
shared coding standard, and a deliberate philosophy about debuggability,
explicitness, and cross-cutting contracts.  The book series exists to make
that philosophy reproducible — not as a marketing exercise for the platform,
but as a teaching artefact: a senior engineer can read the series and come
out with a discipline that works regardless of whether they continue to use
the automojo packages themselves.

### Audience profiles

| Profile | Volumes most relevant |
|---|---|
| Junior Python engineer entering automation work | 0, 1, 3 |
| SDET / test-infra engineer | 0, 1, 2, 3 |
| Platform / DevOps engineer building CI pipelines | 0, 3, 4, 6 |
| SRE running cross-OS automation at scale | 0, 3, 4, 6 |
| Integration engineer building protocol implementations | 0, 5 |
| AI / agent developer shipping production agents | 0, 5, 7 |
| Senior engineer / architect needing observability for long-lived services | 0, 8 |
| Internal-tool maintainer running a package distribution | Appendix |

Volume 0 is prerequisite for everyone.  Beyond that, volumes are
independent except where a forward reference is called out below.

### Reading order

The technical reading order is `0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8`.  The
marketing order may differ — Volume 8 (Time-Travel Debugging) is the
marquee book and could lead on the shelf even though it should be read
last.

A reader picking by interest can use the cross-reference table:

| If you start with | You'll need first |
|---|---|
| Vol 1 (Testing) | Vol 0 |
| Vol 2 (Distributed) | Vol 0, Vol 1 |
| Vol 3 (Integration & Orchestration) | Vol 0 |
| Vol 4 (Sysman) | Vol 0; cross-refs to Vol 3, Vol 5 |
| Vol 5 (Interop) | Vol 0 |
| Vol 6 (Topology / UI / Provisioning) | Vol 0; cross-refs to Vol 3, Vol 5 |
| Vol 7 (AI Agents) | Vol 0, Vol 1, Vol 5 |
| Vol 8 (Time-Travel) | Vol 0; case studies cross-ref to Vol 2, Vol 7 |
| Appendix (Distribution) | Vol 0 |

### Voice and style conventions

- **AutomationX as narrator.**  The series is written in the voice of the
  AutomationX Principal-SWE persona that already opens every repo's
  `AGENTS.md`.  This is consistent with the portfolio's style and gives the
  series a unifying first-person.
- **Code-as-anchor.**  Every claim is anchored on a concrete file:line
  reference into one of the repos.  This is how the series differentiates
  itself from generic "Python automation" books — readers can read the
  book and the source side-by-side.
- **Audit diaries as appendices.**  The recent audits (taskplus 17 bugs,
  uia 20 bugs, landscaping 19 bugs, foundation tier 334 tests added) are
  exactly the kind of "how a senior engineer reads a codebase" material
  that's hard to fake.  Each volume gets an appendix or sidebar that uses
  the relevant audit transcript.
- **Coding standards as a teaching tool.**  CLAUDE.md is referenced, but
  the *why* behind each rule is unpacked in Volume 0 and reinforced
  throughout.  Rules without rationale are not worth following; rules with
  clear rationale survive turnover.

---

## Volume 0 — Foundations of Automation Engineering with Python

**Subtitle:** *The AutomationX Discipline and the Foundation Tier*

**Audience:** Any Python engineer building a serious automation platform.
This is prerequisite reading for the rest of the series.

**Why it leads:** every later volume references the AutomationX persona,
the error vocabulary, the Aspects pattern, the explicit-`return`
convention, the configuration model, and the application-identity machinery.
Without it, every later book has to re-explain.

**Estimated pages:** 300–400.

**Portfolio source:** `automojo-interfaces`, `automojo-errors`,
`automojo-waiting`, `automojo-this-app`, `automojo-extension`,
`automojo-runtime`, `automojo-configuration`, `automojo-credentials`,
`automojo-dataprofiles`, `automojo-storage-database`, `automojo-xmodules`.

### Chapters

1. **The AutomationX philosophy** — debuggability over performance, explicit
   over implicit, the Aspects pattern, "errors should never pass silently."
2. **Coding standards as a tool, not a style guide** — `CLAUDE.md` decoded,
   with the *why* behind each rule (explicit `return` for breakpoint
   anchors, properties as data accessors only, no mutable defaults, no
   shortcut truthy checks, named-parameter passing for readability).
3. **The error vocabulary** — `SemanticError(BaseException)` must always
   bubble; `ConfigurationError` for environment problems;
   `AbstractMethodError` for unimplemented overrides; the multi-line
   error-message construction pattern.
4. **Interfaces and contracts** — `ICommandRunner`, `IFileOperator`,
   `ISystemContext`; abstract-base discipline; what belongs in
   `automojo-interfaces` vs in tier-specific shared packages.
5. **Waiting with rich diagnostics** — beyond `time.sleep`; timeout
   composition, attempt logging, observable retry.
6. **Application identity and presence** — `establish_app_persona`,
   `AppVarDesc`, `ContextPaths`, the persona-variable resolution pipeline.
7. **Extension and factory machinery** — `SuperFactory`,
   `FactoryProtocolBase`, the precedence model; how to register a new
   factory; how protocol resolution iterates and selects.
8. **Runtime variables and well-known paths** — the `automojo-runtime`
   package; activation; well-known directory layout.
9. **Configuration loading** — multi-source resolution
   (`dir://`, `http://`, `couchdb://`, `mongodb://`); the five
   configuration categories (Credentials, Landscapes, Topologies, Runtimes,
   Data Profiles); encryption with Fernet; bidirectional synchronisation
   to remote stores.
10. **Credentials, data profiles, storage scaffolding** — vocabulary
    layer; lookup-by-name patterns; what each foundation-tier package
    contributes and how they compose.

### Cross-references

None.  Volume 0 is prerequisite for everything.

### Source-material strengths

- The recent `automojo-configuration` requirements rework (17 module-scoped
  EARS requirements + Source Module Map) gives chapter 9 a ready-made
  reference structure.
- The foundation-tier zero-test gaps closed in a prior session (334 tests
  across `automojo-errors`, `automojo-interfaces`, `automojo-storage-database`,
  `automojo-networking`) show *what good test coverage looks like* on a
  foundation package — useful as a sidebar pattern.

---

## Volume 1 — Building a Test Framework

**Subtitle:** *testbase, arcli, and Single-Host Test Infrastructure*

**Audience:** SDET and test-infrastructure engineers.

**Why it earns its own volume:** `testbase` is the flagship test framework
of the platform (~40k LOC, 146 test files, mature) and `arcli` is the
declarative CLI toolkit underneath every `automojo-*` CLI.  Together they
are the single-host substrate that distributed test execution (Vol 2)
extends.

**Estimated pages:** 400–500.

**Portfolio source:** `automojo-testbase`, `automojo-arcli`.

### Chapters

1. **Why `unittest` / `pytest` fall short for automation** — constraints,
   resource factories, landscape awareness; why automation tests need a
   different shape than application unit tests.
2. **`testbase` architecture** — plain function tests, the `verification`
   API, the testbase runner, the JSOS streaming results pipeline.
3. **Resource factories and dependency injection** — the testbase fixture
   model; how fixtures compose; how cleanup is sequenced.
4. **Property-based testing with `@given`** — the constraints model; the
   `TESTBASE_MAX_EXAMPLES_CAP` ceiling pattern (fast-CI vs full-sweep);
   reproducibility via seeds.
5. **Streaming JSOS results** — machine-readable, replayable test runs;
   per-event schema; consumer patterns (CI dashboards, replay tools).
6. **The `testbase agents` machine-readable CLI** — agent-driven test
   discovery, listing, and execution.
7. **Mocking and stubbing without the framework getting in the way** —
   the testbase mocking primitives; when to mock; when not to.
8. **`arcli` — the declarative CLI toolkit** — `Command`, `ArgField`,
   `OptionField`, parameter types; how `automojo-*` packages register
   their command trees.
9. **Building tests around landscape concepts (single-host edition)** —
   the contract that distributed jobs (Vol 2) will extend; what's
   landscape-aware vs landscape-agnostic on a single host.
10. **Case study: a 3,700-test suite that runs on one host** — the
    `automojo-agent-kit` test suite at `TESTBASE_MAX_EXAMPLES_CAP=10`;
    49-minute wall time; lessons from a real fast-CI configuration.

### Cross-references

- Forward to Vol 2 for the multi-host extension of `TestJob`.
- Forward to Vol 3 for orchestration-level invocation (taskplus invoking
  a testbase test run from a workflow).

### Source-material strengths

- `automojo-testbase` itself is the most-tested package in the portfolio
  (146 test files / 108 modules, better than 1:1).  The book can use the
  framework's own tests as worked examples.
- `arcli` recently went through validator-and-parser hardening with
  regression tests added in PR #13; the audit pattern from that work is
  good chapter-end material.

---

## Volume 2 — Distributed Automation

**Subtitle:** *The TestJob Inheritance Chain and Multi-Host Test Execution*

**Audience:** test engineers running multi-host integration suites;
platform engineers building automation that crosses machine boundaries.

**Why it earns its own volume:** distributed automation is the marquee
*capability* of the testing tier — it's the union of `automojo-testplus` +
`automojo-landscaping` + `automojo-interop`'s job machinery +
`automojo-protocol-ssh` + `automojo-credentials` + `automojo-dataprofiles`.
Seven repos collaborate.  The recent inheritance-chain refactor that
collapsed `DistributedTestJob` from ~615 lines to 90 by introducing
`InteropLibraryTestJob` as the landscape-aware hinge is a complete
evolution-of-a-design narrative.

**Estimated pages:** 400–500.

**Portfolio source:** `automojo-testplus`, `automojo-interop` (job
machinery half), `automojo-landscaping`.

### Chapters

1. **Why distributed automation is genuinely hard** — the failure modes
   single-host frameworks miss (one host of N down, partial credential
   failure, race conditions in setup/teardown, machine-clock drift,
   network partitions during tests, log aggregation gaps).
2. **Single actor, multiple observers — the determinism model** — the
   load-bearing design choice that distinguishes automation testing from
   application testing.  When the controller acts on a distributed
   environment, the test must wait for effects to propagate before
   checking; a second actor concurrently causing effects breaks
   ordering and breaks determinism.  Therefore: one actor per
   environment.  Tests that need concurrent observation use the
   tandem-testing mechanism (chapter 5), not parallelism.
3. **The Pod model — where parallelism actually lives** — at the *job*
   level, across independent **Pods** (a Pod = one Landscape + one
   Topology realised as a runtime instance).  Within a Pod, jobs run
   sequentially because the Pod is a shared environment.  Across Pods,
   jobs run in parallel because Pods are independent.  Why
   `pytest-xdist`-style test parallelism is the wrong model for
   distributed automation; why CI horizontal scaling means more Pods,
   not more tests-in-parallel.  Forward to Vol 3 for orchestration of
   Pod fan-out.
4. **The TestJob inheritance chain — the customisation cascade** —
   what each layer adds, why it lives where it lives, *and how the
   chain is the framework's customisation entry point* (Volume 0
   ch.1's meta-thesis made concrete).  Five classes in the hierarchy
   today:
   - `TestJob` (in `testbase`, `testjob.py:50`): single-host execution,
     fixtures, results, the five `fire_*` lifecycle hooks; template
     methods `_create_sequencer()` and `_create_recorder()`.
   - `DefaultTestJob` (in `testbase`, `testjob.py:621`): naming
     fallback; no overrides.
   - `InteropLibraryTestJob` (in `interop`, `testjob.py:22`): landscape
     activation overrides on the five `fire_*` hooks; the abstract
     `_get_landscape()` template method that breaks the would-be
     circular dependency with `landscaping`.
   - `DistributedTestJob` (in `testplus`, `distributedtestjob.py:24`):
     concrete `_get_landscape()` returning `LandscapeSingleton`;
     `_create_sequencer()` returning a `DistributedTestSequencer`.
   - `DefaultDistributedTestJob` (in `testplus`,
     `distributedtestjob.py:84`): naming fallback for distributed.

   Each layer adds *one* responsibility — single-host execution →
   landscape awareness → distributed scheduling.  The hierarchy is
   open: `generic_test_entrypoint(job_type=...)` and
   `distributed_test_entrypoint(job_type=...)` accept the user's
   subclass.  Real scaffolding examples live at
   `automojo-testbase/source/tests/automojo_tests/testbase/integration/test_custom_job_type_integration.py`
   (`CustomTestJob`, `FailingCustomTestJob`).

   *Note on parallel hierarchies:* `OrchestrationJob` (in
   `automojo-landscaping`, `orchestrationjob.py:34`) is **not** a
   `TestJob` subclass — it inherits from `ContextUser`.  Orchestration
   jobs and test jobs are separate hierarchies; Vol 3 covers
   orchestration's job model.
5. **Tandem testing in depth** — the actor / observer model.
   `ValidatorCoupling`, the `attach_to_test(testscope, suffix)`
   mechanism, the per-validator `TestResult` recorded alongside the
   test's own.  How a test with N validators becomes one test with
   N+1 outcomes.  When to add a validator (steady-state checks during
   a test) vs. when to put the check in the test body.  Reference
   implementation:
   `automojo-testbase/source/packages/automojo/testbase/tandem/validators.py`.
6. **Landscape activation** — what each `fire_*` event does at landscape
   level (device wake, credential pre-fetch, topology realisation, etc.).
7. **The CI → job hand-off as a verified contract** — Landscape says
   "here are your resources" and the job verifies reachability;
   Topology says "here is the configured state" and the job verifies
   the configured state matches.  The
   `LandscapeConfigurationLayer.validate_topology()` override hook;
   the `validate_topology` flag on `LandscapeActivationParams`;
   verification as a precondition for the determinism model in
   chapter 2.
8. **The `DistributedTestSequencer`** — coordination, dependency
   resolution, ordering primitives.  *(Note: this chapter does **not**
   cover "parallelism control" — that framing is wrong for the
   single-actor model.  The sequencer enforces serial execution within
   a Pod; cross-Pod parallelism is an orchestration concern, Vol 3.)*
9. **`job_seed` and reproducibility** — how a single seed survives the
   trip across machines.
10. **The remote-execution wire** — SSH transport (forward to Vol 5),
    session reuse, command framing, error propagation.
11. **Credentials at scale** — multi-host specifics; how the framework
    from Vol 3 Part III applies to live distributed runs.
12. **Data profiles in remote workers** — why you don't ship config files;
    why you do ship profile names plus a resolver.
13. **Result aggregation** — streaming JSOS from N workers back to one
    coordinator without losing ordering; the failure-mode where one
    worker's stream stops.
14. **Partial-run semantics** — what "the suite passed" means when one of
    five hosts crashed mid-run.
15. **Debugging a distributed failure** — forward reference to Vol 8
    (time-travel) for the production version.
16. **Case study: a 5-host integration test for a sysman feature** —
    forward reference to Vol 4 (sysman); landscape configuration,
    topology verification, dependency wiring, partial-failure handling,
    tandem-validator coverage of cross-host steady state.

### Cross-references

- Backward to Vol 1 (testbase) for the single-host substrate.
- Forward to Vol 3 (orchestration) for "who invokes a distributed job."
- Forward to Vol 4 (sysman) for the canonical use case.
- Forward to Vol 5 (interop / SSH) for the wire.
- Forward to Vol 8 (time-travel) for the debugging story.

### Source-material strengths

- The `automojo-testplus` `5ae46f5` inheritance-chain refactor (615 → 90
  lines) plus the `automojo-interop` `c785e9b` introduction of
  `InteropLibraryTestJob` are recent and well-documented.  Chapter 2
  writes itself.
- The two-pass testplus audit (`72431d9` fixed 9 bugs, `c60d7c7` fixed 5
  more) plus the landscaping six-pass audit (19 bugs across 14 files,
  +101 testbase tests across 11 test files) are pre-built audit-diary
  appendix material.

---

## Volume 3 — Integration Automation and CI Orchestration

**Subtitle:** *Typed Configurations as the Contract Between Orchestrations
and Jobs*

**Audience:** platform engineers, DevOps, QA infrastructure engineers
running multi-step pipelines that need to provision, run, observe, and
tear down across multiple machines and configurations.  Also: any Python
engineer who wants a development workflow that scales from local laptop
to nightly full-sweep without rewriting code at any boundary.

**Why it earns its own volume — and is the headline volume of the
series:** the orchestration layer is structurally distinct from the job
layer, and the "typed configurations as contract" thesis is what
unifies the entire portfolio's integration story.  Volumes 1, 2, 4, 5,
and 6 all reference Part IV ("Workflow Automation Across Environments")
because profiles are how you eliminate `os.environ.get('CI')` branches
across a codebase.

**Estimated pages:** 500–600.  The largest non-AI volume.

**Portfolio source:** `automojo-taskplus`, `automojo-configuration`,
`automojo-credentials`, plus parts of `automojo-this-app`,
`automojo-landscaping`, `automojo-provisioning`.

### Part I — *The Discipline*

1. **What integration-level automation is** — the layer above unit /
   integration testing in the traditional sense; the layer that ties
   together "create a fleet, configure it, run a multi-machine test,
   aggregate results, tear down or hand off."
2. **Why most CI YAML grows monstrously** — vendor lock-in, untyped
   strings, environment-variable proliferation, no rollback semantics,
   no provenance, no separation of orchestration from job from task.
3. **The contract: six typed configuration categories** — Specifications,
   Landscapes, Topologies, Runtimes, Credentials, Data Profiles.

### Part II — *The Configuration Categories*

The six categories are *uniformly composable* via `MergeMap` overlay
— a property the chapters in this part name and demonstrate
consistently.  Each category resolver takes a list of names and a
list of files and merges them; later names override earlier ones.

4. **Specifications** — declarative resource-creation manifests;
   checkout-or-create semantics; idempotent provisioning; the separation
   of "what should exist" from "how to make it exist."  *The dynamic
   path:* a Specification declares the resources a Pod needs; CI
   tooling consumes the Specification, performs the checkout or
   creation, and **emits the Landscape file and Topology file** that
   the test job will then verify and run against.  *The static path:*
   pre-existing Landscape + Topology files; no Specification needed.
   *The hybrid path:* user tooling generates the Topology file at job
   startup from the Landscape it was given.  Composition: multiple
   Specification files compose to assemble a larger Pod.
5. **Landscape files — physical inventory and wiring** — modelling
   *what resources exist and how they are physically connected* as
   data.  Landscape composition (`Checkout Resources + Environment
   Resources` is a canonical pairing — pool-checkout merged with
   permanent infrastructure).  The verification contract: the test job
   verifies every resource in the Landscape is reachable at the
   declared address with the declared protocol using the declared
   credentials.  Forward to Vol 6 for what's *inside* a Landscape file
   schema-wise.
6. **Topology files — logical organisation and roles** — modelling
   *how those resources are organised and what roles they play
   together* as data.  Why Topology is *separate* from Landscape (a
   Landscape hosts many Topologies; a Topology applies to many
   Landscapes; they live in separate `MergeMap`s in
   `automojo-configuration` for a reason).  Examples: "Players A, B, C
   form a Home Sound System"; "Nodes A, B, C are cluster members;
   Node D is a spare."  Topology composition (combine cluster
   topology + gateway topology).  The verification contract: the test
   job verifies the resources are *configured as the Topology
   indicates* — Topology is overloaded as both a description *and* an
   executable assertion.  The canonical `clusters` shape consumed by
   `LandscapeOperationalLayer._create_clusters` is the only schema
   the framework imposes; everything else is user-defined.  Forward
   to Vol 6.
7. **Runtime files vs environment variables** — the core thesis chapter:
   - Why env vars proliferate (every team adds one, no one removes one).
   - What goes wrong (forgotten exports, namespace collision, type
     confusion, no schema).
   - What runtime configs fix (typed schemas, profile selection at
     startup, namespaced overrides, traceable provenance).
   - The migration pattern: env-var-soup → runtime profile + 1-line
     `--profile prod` selector.
   - *Runtimes are composable:* a Runtime selection of `UPNP
     Diagnostics + Bluetooth Diagnostics` merges both diagnostic
     profiles into one effective runtime.  The diagnostic-stacking
     example is the cleanest demonstration of why composition matters.
   - *The User file (`user.yaml`, documented separately in
     `userguide/03-07-user-file.rst`) is **the default Runtime for
     local-user runs** — not a seventh configuration category.* The
     User file sits at the bottom of the local-user runtime overlay
     stack; explicitly-selected Runtime files compose on top.  CI
     runs do not include the User file.  This explains why the user
     file is documented but not in the six-category taxonomy:
     functionally it is a Runtime.
   - *Note on the userguide's "not extensible" claim:* per
     `03-04-runtime-file.rst:36–43`, runtime configuration is "not
     designed to be extensible like the other configuration objects."
     This refers to extensibility (registering new runtime *types*),
     not composability (overlaying multiple runtime *files*).
     Runtimes remain composable.
8. **Data profiles** — connection profiles for data stores; lookup-by-name
   vs hardcoded URIs scattered through code.  Composable like the
   other categories: lab-wide profiles merged with experiment-specific
   overrides.

### Part III — *Credential Management*

9. **Credential vocabularies** — basic auth, SSH keypairs, API tokens,
   OAuth, certificates, derived credentials; types vs scopes vs lifetimes.
10. **Credential resolution** — vault, file, environment-as-fallback (the
    only legitimate env-var case), prompt, derived from another
    credential; host-scoped vs role-scoped lookup; the resolution order.
11. **Credentials in flight** — rotation during long runs (matters for
    hour-plus integration suites); encryption at rest (Fernet from
    `automojo-configuration`'s cryptography module); the audit trail.
12. **Credentials across environments** — same credential *vocabulary*
    across local-dev / CI / staging / prod, different *backends*; how a
    credential reference resolves differently per profile without product
    code knowing.

### Part IV — *Workflow Automation Across Environments*

The selling point of the discipline.  Most CI books skip this and most
automation tools force the choice between "works in CI but not locally"
and "works locally but doesn't scale to CI."

13. **The development-environment ladder** — local-dev / integration-CI /
    staging / prod-readonly as *profiles*, not codebases.  The lifecycle
    of a single test from "I'm writing it on my laptop" to "it just ran
    on the nightly sweep across 5 OSes."
14. **Configuration profiles as a discipline** — base profile plus overlay
    composition; how profiles compose vs override; profile selection at
    process start (`--profile local`); profile diff as a first-class
    debugging artefact.
15. **The local-dev tax and how typed configs cancel it** — no
    `.env.local` files, no `direnv`, no "did you `make setup` first."
    Profile selection is the only setup.  The chapter walks through what's
    missing from a fresh checkout and shows how typed configs make every
    missing piece named, typed, and located.
16. **Reproducing CI failures locally** — pull the runtime profile (and
    credential references, and landscape selection) from the failing CI
    run; run the failing job locally with the same configuration; debug
    in your IDE.  The `TESTBASE_MAX_EXAMPLES_CAP` pattern is a tiny
    example — runtime config that switches behaviour between fast-CI and
    full-sweep without changing code.
17. **Profile promotion** — your local-dev profile additively becomes
    staging via overlay; staging additively becomes prod via overlay.  No
    rewrites; the discipline keeps the four environments siblings, not
    strangers.

### Part V — *taskplus and the FlowEngine*

18. **The taskplus task model** — `FlowNode`, `Task`, `TaskGroup`,
    ordinals, the `if_expr` gate.
19. **Built-in task types** — `RunCommand`, `ShellScript`,
    `EmbeddedPython`, `GitCheckout`, `CreateEnvironment` (with the
    "you probably don't need `CreateEnvironment` if you use Runtimes
    properly" caveat for green-field code).
20. **Workflow YAML grammar** — `name` / `environment` / `parameters` /
    `flow` / nesting / `onfailure` / `onsuccess`.
21. **Passing information to jobs** — the contract between an orchestration
    and a job is *named configurations + verification*, not just config:
    - Orchestration parameters say "use landscape `staging-east + prod-
      shared`, topology `cluster-of-4 + gateway-pair`, runtime
      `verbose + upnp-diagnostics`, credentials `ci-runner +
      experiment-overrides`" — names per category, composable lists.
    - The orchestration injects those *names* into the job's environment
      (one variable per category, just enough to bootstrap).
    - The job's startup uses `automojo-configuration` to resolve each
      name list into the actual structured config via `MergeMap`
      overlay (every category, same pattern).
    - **The job verifies the realised configuration before the first
      test runs:** Landscape verification confirms every resource is
      reachable; Topology verification confirms every resource is
      configured as the topology declares.  A failed verification
      means the CI system handed off something it shouldn't have, and
      the job refuses to run.  This is what makes the determinism
      contract from Volume 2 ch.2 actually deterministic.
    - Inside the job, code reads typed configurations — never env vars.
22. **Flow control as data** — `onfailure` / `onsuccess` jumps,
    `JumpContext`, ordinal numbering for trace ("step 2.3.1 failed"),
    taskgroup recursion.
23. **Extending the task model** — your own `Task` subclass; registering
    via `tasktype: module@ClassName`; contributing it back as a portfolio
    extension.

### Part VI — *Real-World Wiring*

24. **Wiring to a real CI vendor** — GitLab CI / GitHub Actions / Jenkins
    shims; the runner is dumb, the workflow is portable; what stays in
    vendor YAML (the trigger and the runner image) and what moves into
    taskplus YAML (everything else).
25. **Bridges to other layers** — orchestration → distributed test job
    (Vol 2); orchestration → sysman feature execution (Vol 4);
    orchestration → resource provisioning (Vol 6).
26. **Case study: porting a 200-line GitLab YAML with 47 env vars to a
    30-line taskplus workflow + 5 named configs + 4 profiles.**  Before /
    after, with the working diffs.
27. **Where this approach is *not* the right fit** — heavy parallel
    fan-out *within* a Pod (the framework forbids it by design — Vol 2
    ch.2's determinism model), dynamic workflow generation from
    streaming events, sub-second event-driven flows.  Honesty about
    scope; build-vs-buy chapter.  *The right shape of horizontal CI
    scaling for this discipline:* provision more Specifications →
    more Pods → more parallel jobs.  Each Pod is independent; jobs
    across Pods run concurrently; tests within a Pod stay sequential.
    A "20-fold parallel CI" means twenty Specifications, not twenty
    parallel tests inside one Pod.

### Cross-references

- Backward to Vol 0 (configuration loading, application identity).
- Forward to Vol 2 (the job layer that orchestration invokes).
- Forward to Vol 4 (sysman feature execution as orchestration target).
- Forward to Vol 6 (provisioning as orchestration target).

### Source-material strengths

- `automojo-taskplus` is now the most-thoroughly-audited orchestration
  package in the tier (115 tests, 17 source bugs fixed across two
  passes).  The FlowEngine internals can use real test cases as worked
  examples in chapter 18.
- The recent `automojo-configuration/specs/requirements.md` rework
  formalises the five configuration categories with module-scoped EARS
  requirements — chapters 4–8 anchor on it directly.
- The `automojo-this-app` `override_variable_value` work that updates
  live persona-variable instance fields is the runtime-config-vs-env-var
  migration story in microcosm — perfect mid-Part IV worked example.
- The `TESTBASE_MAX_EXAMPLES_CAP` pattern documented in
  `automojo-testbase/userguide/10-04-03-testing-frameworks-testbase.rst`
  is the smallest possible working example of "runtime config switches
  behaviour without code changes" — opens chapter 16.
- The empty CLI command bodies at `taskplus/cli/cmdtree/testing/jobs.py`
  lines 49 and 62 are still open — perfect chapter-end exercise material.
- The `automojo-taskplus/source/packages/automojo/taskplus/execution.py`
  module is confirmed dead code with a signature mismatch against modern
  `Task.execute` — gift for a "what to keep, what to delete" chapter.

### Open questions to resolve before drafting

1. Does "Specifications" refer to a sixth managed type within
   `automojo-configuration` (alongside Credentials / Landscapes /
   Topologies / Runtimes / Data Profiles), or is it a separate concept
   that lives in `automojo-provisioning` or in the per-repo `specs/`
   directory pattern?
2. What credential-resolution backends are shipped in
   `automojo-credentials` today (vault integrations, AWS Secrets Manager,
   1Password, etc.) versus reader-supplied?
3. Are there established profile-introspection commands in the portfolio
   (e.g., `automojo profile show`, programmatic `current_profile()`
   accessor) that chapters 14–17 should anchor on?

---

## Volume 4 — Cross-Platform System Management

**Subtitle:** *50 Cross-OS Contracts and 5 Reference Implementations*

**Audience:** platform / SRE engineers who need cross-OS automation that
actually works.

**Why it earns its own volume:** the sysman tier is the most disciplined
product line in the portfolio.  50 cross-OS contracts in
`automojo-sysman-common/specs/contracts/` plus five reference
implementations (RHEL, Ubuntu, macOS, Windows, UEFI) totalling
3,656 / 3,656 testbase tests passing.  The "do it this way" book in the
series.

**Estimated pages:** 400–500.

**Portfolio source:** `automojo-sysman-common`,
`automojo-sysman-linux-redhat`, `automojo-sysman-linux-ubuntu`,
`automojo-sysman-macos`, `automojo-sysman-windows`, `automojo-sysman-uefi`.

### Chapters

1. **Why a contract layer** — the trap of OS-by-OS feature flags; the
   semantic-spec discipline that prevents drift.
2. **The 50 cross-OS contracts** — anatomy of a contract; the EARS
   structure; how "applies to my tier" is decided.
3. **`featurebase` and the per-OS implementation pattern** — base classes;
   how a feature is added; how a feature is tested.
4. **RHEL** — `dnf`, `systemctl`, `firewalld`, `selinux`,
   `subscription-manager`, `cockpit`, `insights`, `tuned`; the layered
   spec architecture; 46 features in the registry.
5. **Ubuntu / Debian** — `apt`, `systemctl`, `netplan`, `ufw`; the
   parallel feature surface to RHEL; 50 modules / 814 tests.
6. **macOS** — `brew`, `launchctl`, `networksetup`, `pf`, `defaults`;
   50 modules / 809 tests.
7. **Windows** — PowerShell `winget`, `Get-Service`, `NetSecurity`,
   `bcdedit`; 50 modules / 917 tests.
8. **UEFI firmware** — boot-management, secure-boot, EFI variables,
   capsule-update, ESP management; the firmware-tier scope decision
   (5 features intentional, not under-scope).
9. **Spec-driven development for system administration** — per-OS specs
   paired with implementation, never authored speculatively ahead of
   code; what changes when this rule is broken.
10. **Building cross-OS test parity** — 3,656 tests across 5 packages,
    all green; how to set up a cross-OS comparison harness; what should
    differ vs what shouldn't.

### Cross-references

- Backward to Vol 0 (foundation tier).
- Forward to Vol 3 (orchestration → sysman feature as orchestration step).
- Backward to Vol 5 (interop) for the wire.

### Source-material strengths

- The sysman tier is fully green and stable.  Every chapter has a
  reference implementation already shipped.
- The 50 contracts at `automojo-sysman-common/specs/contracts/` are the
  cleanest spec set in the portfolio.

---

## Volume 5 — Interop and Protocols

**Subtitle:** *The Aspects Pattern, the Wire, and the Extension Model*

**Audience:** integration engineers, protocol implementors.

**Why it earns its own volume:** `automojo-interop` provides the
foundational pattern (Aspects) used across the portfolio for behaviour
parameterisation, plus protocol implementations that the rest of the
portfolio depends on.  `automojo-protocol-ssh` is substantial code
(transport, session, SFTP, tunnels, keys/auth) with its own audience.

**Estimated pages:** 400–500.

**Portfolio source:** `automojo-interop` (protocol half),
`automojo-protocol-ssh`, `automojo-networking`, `automojo-restclient-base`,
`automojo-restclient-github-public`, `automojo-interop-ext-{upnp, tasker,
bluetooth, can}`.

### Chapters

1. **The Aspects pattern** — packaging behaviour knobs (timeout, retry,
   sync, logging) into a single scalable parameter; why this is *not* a
   `@retry` decorator.
2. **Read-only vs state-modifying APIs** — when Aspects is required vs
   optional; the design heuristic.
3. **`automojo-interop`** — agent bases, protocol implementations
   (DNS / mDNS, SSH, serial, power), the `am-interop` CLI.
4. **SSH from scratch** — transport, session, SFTP, tunnels,
   key/password auth (`automojo-protocol-ssh`).
5. **Networking toolkit primitives.**
6. **REST client foundations** — `automojo-restclient-base`.
7. **The auto-generated GitHub public-API client** —
   `automojo-restclient-github-public`; what's worth testing in a
   568-module generated package; smoke-vs-comprehensive testing
   strategy.
8. **Extension packages** — UPnP / Tasker / Bluetooth / CAN; the
   extension registration pattern.
9. **Building your own interop extension** — the pattern as a
   reproducible recipe.

### Cross-references

- Backward to Vol 0 (foundation tier).
- Forward to Vol 2 (distributed automation uses SSH for the wire).
- Forward to Vol 6 (UI automation extends similar patterns).

---

## Volume 6 — Topology, Provisioning, and UI Automation

**Subtitle:** *Modelling the Environment Under Automation*

**Audience:** test / automation engineers wiring up complex environments;
platform engineers building automation infrastructure.

**Why it earns its own volume:** these packages model the *environment*
the rest of the platform operates on — landscape (what's there),
topology (how it's organised), provisioning (how to make it appear),
and UI automation (how to drive applications inside it).  The volume
is bound by the same "environment as data" thesis that runs through
Vol 3 — but where Vol 3 covers the *category-level* concerns (what's
in a Landscape file, what's in a Topology file, how they compose),
Vol 6 covers the *implementation* concerns (how `automojo-landscaping`
loads them, what scaffolding the framework supplies, what the user
implements, the canonical `clusters` shape, the verification hooks).

**Important framing note — there is no `automojo-topology` package.**
Topology is a *configuration category* loaded by `automojo-configuration`
into `TOPOLOGY_CONFIGURATION_MAP`, with consumption code distributed
across `automojo-landscaping`'s layers (`TopologyIntegrationLayer`,
`LandscapeConfigurationLayer.load_topology()`,
`LandscapeOperationalLayer._create_clusters()`).  The volume's
Topology coverage walks the framework's *minimal scaffolding* (the
loader, the canonical `clusters` schema, the `BaseExt`-based extension
model with the VMware vSphere `TopologyExt` as a worked example) and
explicitly leaves the rest of the schema to the user.  *Topologies are
left up to the user to implement.*

**Estimated pages:** 400–500.

**Portfolio source:** `automojo-landscaping`, `automojo-provisioning`,
`automojo-uia`, `automojo-gitlab`, `automojo-webservice-flask`.
*Topology* is not a package — it lives across `automojo-configuration`
(loader) and `automojo-landscaping` (consumer).

### Chapters

1. **Landscape — physical inventory and wiring** — what a Landscape
   *is* (the resources that exist + how they're physically connected);
   `Landscape`, `LandscapeIntegrationLayer`,
   `LandscapeOperationalLayer`, the layer-installation discipline.
   The `MergeMap`-based composition model (`Checkout Resources +
   Environment Resources` as the canonical pairing).  The Landscape's
   *two top-level sections*: `services` (infrastructure services like
   serial concentrators, power controllers, vaults — each with a
   name, type, and address) and `devices` (each with a deviceType +
   primary access fields + Features).
2. **Inside a device declaration — deviceType, Features, metadata** —
   the three layers of a device entry in a Landscape file:
   - *Primary access* — `deviceType: network/ssh` plus `host`,
     `credentials`, `platform`.  The deviceType selects the
     Coordinator that attaches the primary ProtocolExtension.
   - *Features* — additional capabilities beyond primary access.  Two
     shapes: **boolean capability flags** (e.g. `isolation: true` —
     pure data) and **service-reference Features** (e.g.
     `serial: serial-concentrator: 1` — names a service from the
     services section + a port; materialises into a ProtocolExtension
     at runtime).
   - *Declarative metadata* — `platform`, `skip`, the composable
     `credentials` list.
3. **Coordinators, Agents, ProtocolExtensions, Features** — the
   four-word vocabulary for the device-control machinery, used
   precisely throughout the rest of the volume:
   - **Agents** *speak protocols* (SSH Agent, UPnP Agent, Serial
     Agent, Power Agent).
   - **Coordinators** *manage devices and attach Agents to devices.*
   - **ProtocolExtensions** *connect a specific Agent to a Device.*
   - **Features** *are declared on devices and (for service-reference
     Features) reference services.*
   The full layering walked end-to-end with the serial-concentrator
   example.
4. **Device discovery and friendly identifiers** — `FriendlyIdentifier`
   patterns; identity match vs hint; the equality and matching
   semantics.
5. **Topology — logical organisation and roles, with verification** —
   what a Topology *is* (logical grouping + role assignment, examples
   from sound systems and clustered storage); why Topology and
   Landscape are *different* MergeMaps; the canonical `clusters` shape
   (`name` / `group` / `nodes` / `spares`) consumed by
   `LandscapeOperationalLayer._create_clusters`; the verification
   contract (`validate_topology` as both flag and override hook).
6. **Topology as an executable assertion** — the verification overload
   walked from the source: `LandscapeConfigurationLayer.validate_topology()`
   stub, the user-supplied override pattern, the `validate_topology`
   flag on `LandscapeActivationParams`.  Why a Topology fails
   verification means the CI system mis-configured the resources, not
   the test, and what the orchestrator should do about it.
7. **Subclassing `Landscape` and `Topology` — the customisation
   mechanism** — the *concrete* realisation of Volume 0 ch.1's
   meta-thesis for the environment model.  The framework supplies
   the base classes that handle the canonical shapes; the
   organisation subclasses them to recognise project-specific
   Features and Topology sections.  The user's subclasses get
   plugged into a run by being returned from the user's `TestJob`
   subclass's template methods (Volume 1 ch.2's customisation
   cascade; Volume 2 ch.4's inheritance chain).  Worked example: a
   `MyProjectLandscape(Landscape)` recognising a `cooling` Feature
   alongside the built-in serial / power / isolation Features.
8. **Service-specific framework extensions** — the `BaseExt` /
   `TopologyExt` extension model; the VMware vSphere
   `services/vmware/vsphere/ext/topologyext.py` worked example.
   *This is a different extension point than `Landscape` /
   `Topology` subclassing* (chapter 7) — `BaseExt` is for
   service-specific code that fetches state from infrastructure
   APIs (vSphere, Active Directory, Kubernetes), while subclassing
   is for project-specific schema interpretation.  The chapter
   distinguishes the two.
9. **The Pod model — landscape + topology realised** — a Pod is the
   runtime instance produced by combining a Landscape with a
   Topology.  How a Pod is the unit of horizontal CI parallelism
   (Volume 2 ch.3, Volume 3 ch.27); how Pod independence is what makes
   parallel jobs safe; why test-level parallelism within a Pod
   remains forbidden.
10. **Cloud provisioning model** — `automojo-provisioning`;
    vendor-neutral interface; AWS / Azure / GCP backends; the
    checkout-vs-create story (cross-ref to Vol 3 Specifications); how
    provisioning emits Landscape + Topology files for the next stage.
11. **Cross-platform UI automation** — Linux (AT-SPI / X11 / Wayland),
    macOS (Accessibility / scripting bridges), Windows (UIA); the
    `Mac*` / `Linux*` wrapper hierarchy; the `UIElement` /
    `ScreenElement` / `UIWindow` / `UIApplication` model.
12. **Declarative UI mappings** — `DeclarativeApplication`,
    `ElementHint`, the per-app mapping classes
    (`Calculator`, `LibreOfficeCalc`, `MicrosoftExcel`); separation of
    "what's in the app" from "how to drive it."
13. **GitLab integration** — anatomy of a real-world REST integration;
    what's in the auto-generated client vs the hand-written code;
    testing strategy.
14. **Web-service scaffolding with Flask** —
    `automojo-webservice-flask`; the patterns the website front-end
    uses.
15. **Case study: a landscape + topology that drives a sysman feature
    test across four OSes** — full integration walk-through, including
    Landscape composition (services + devices + Features), Topology
    verification, custom Landscape subclass with a project-specific
    Feature, Pod realisation, and cross-references to Vol 2 / Vol 3 /
    Vol 4.

### Cross-references

- Backward to Vol 0 (foundation tier).
- Forward to Vol 3 (landscape and topology files; provisioning
  Specifications).
- Backward to Vol 4 (sysman as the action layer the landscape drives).

### Source-material strengths

- The `automojo-landscaping` six-pass audit (19 bugs, +101 tests across
  11 test files) is the largest audit-diary appendix in the portfolio.
- The recent `automojo-uia` audit (20 bugs, 31-file docstring fix, two
  passes) is similar — both make the chapters concrete.
- The `MacApplication` class and the `DeclarativeApplication` model
  are well-shaped for chapter 6's worked examples.

---

## Volume 7 — Building an AI Agent SDK

**Subtitle:** *Lifecycle, Tools, Memory, and Observability for
Production Agents*

**Audience:** developers building production AI agents.

**Why it earns its own volume:** `automojo-agent-kit` is the largest
codebase in the portfolio (~46k LOC, 217 modules, 292 testbase test
files, 3,744 tests at fast-CI cap).  The book leans heavily on the
patterns established in Vols 1, 3, and 5, and applies them to the
specific problem of long-running, tool-using, observable agents.

**Estimated pages:** 500–700.  The fattest volume.

**Portfolio source:** `automojo-agent-kit`.

### Chapters

1. **Why an SDK and not a framework** — the lifecycle / tools / memory
   split; the cost of opinionated frameworks for production agents.
2. **Multi-provider LLM abstraction** — OpenAI, Anthropic, others;
   model-agnostic prompt and tool dispatch.
3. **Tool design** — schema, dispatch, error semantics; how a tool fails
   gracefully back into the agent loop.
4. **Memory architecture** — short-term, long-term, summarisation,
   retrieval; what's stored where and why.
5. **Workflow orchestration for agents** — agent-driven workflows vs
   workflow-driven agents; when to use which.
6. **Observability for agents** — tracing, metrics, replay; the
   debug-the-LLM problem.
7. **Testing AI agents** — what's testable (3,744 testbase tests at
   cap=10, 49-minute wall time), what's not, what property tests buy
   you when LLM behaviour drifts.
8. **Property tests at scale** — running 415+ property tests over
   hours; full-sweep ~21,000 individual checks; what these tests
   actually catch.
9. **Building an agent that uses the rest of the automojo portfolio** —
   an agent that runs sysman features against a landscape, observed
   through time-travel debugging.

### Cross-references

- Backward to Vol 0, Vol 1 (testbase fixtures), Vol 5 (Aspects
  pattern for tools).
- Forward to Vol 8 (observability, time-travel debugging an agent run).

### Source-material strengths

- The 3,744-test suite at `TESTBASE_MAX_EXAMPLES_CAP=10` is unusual in
  scale for an AI SDK and provides the worked examples for chapter 7.
- The full-sweep run (13,054 passing checks before completion across
  415 property tests, 11+ hours, 0 failures) is a unique data point.

---

## Volume 8 — Time-Travel Debugging in Production — **drafted**

**Subtitle:** *Pod Instrumentation, Server Ingestion, and the
Replay-Driven Debugging Workflow*

**Audience:** senior engineers, platform / infrastructure teams running
long-lived Python services.

**Why it earns its own volume:** `automojo-timetravel-pod` and
`automojo-timetravel-server` are the marquee product of the portfolio.
Could be the gateway book in marketing order ("here's the thing that
makes the rest worth it") even though it reads last in technical order.

**Estimated pages:** 300–400.

**Portfolio source:** `automojo-timetravel-pod`,
`automojo-timetravel-server`.

**Drafting status:** fully drafted, 8 chapters, 5,399 lines, ~52,000
words, in
[`volumes/08-timetravel-debugging/`](volumes/08-timetravel-debugging/).

### Chapters

1. **Why time-travel debugging** — the gap left by traditional logging
   and tracing; the production-vs-staging asymmetry; the "we couldn't
   reproduce it" trap.
2. **Pod instrumentation** — named "portal" scopes; instrumentation
   density vs cost; the static-vs-dynamic instrumentation choice.
3. **The trace-event protocol** — wire format; ordering guarantees;
   loss semantics under partition.
4. **Server-side ingestion** — `automojo-timetravel-server`; storage
   model; query model.
5. **Remote debug control** — protocol; session lifecycle; security
   model.
6. **Session management at scale** — multi-tenant; retention; replay
   bandwidth.
7. **Case studies** — debugging a flaky distributed test (cross-ref
   Vol 2); debugging an agent execution (cross-ref Vol 7); debugging a
   sysman feature on a remote host (cross-ref Vol 4).
8. **Performance and storage trade-offs** — the runtime overhead
   budget; the storage budget; when to enable, when to disable.

### Cross-references

- Backward to Vol 0.
- Backward case-study refs to Vol 2, Vol 4, Vol 7.

---

## Appendix Volume — Operating an Automojo Subscription Platform

**Subtitle:** *From Git-SHA Pinning to a Published-Package Ecosystem*

**Audience:** maintainers of internal-tool ecosystems / private package
distributions.

**Why it earns its own (slim) volume:** `RELEASE-PIPELINE.md` scopes a
full subscription distribution model that is orthogonal to the
technical content of Vols 0–8 — it's the operational story behind making
the portfolio installable as packages rather than as Git references.
Currently more aspirational than written-up, but the design is locked.

**Estimated pages:** 200.

**Portfolio source:** `RELEASE-PIPELINE.md`,
`website-automationmojo`, the per-repo Poetry / pyproject scaffolding.

### Chapters

1. **Versioning at scale** — SemVer adoption; killing universal `0.0.1`;
   `versions.toml` manifest; `__version__` constants; the migration plan.
2. **Per-repo `release.yml` GitHub Action** — triggered on tag push;
   build, attest, upload.
3. **Website-side `/publish/` POST endpoint** — server-side ingestion;
   subscriber-tier visibility rules.
4. **Subscriber install flow** — auth, package selection, install
   instructions.
5. **Public-tier vs private-tier package selection** — what ships
   publicly, what stays gated, and why.
6. **Build provenance and supply chain** — attestation, SBOM,
   reproducible builds.

### Cross-references

- Backward to Vol 0 only.

---

## Cross-cutting material

### Series glossary (seed terms)

- **Aspects** — the behaviour-parameter object pattern from
  `automojo-interop`.  See Vol 5 ch. 1.
- **AutomationX** — the Principal-SWE persona that narrates the series
  and opens every repo's `AGENTS.md`.
- **ContextPaths** — the well-known path enumeration in
  `automojo-this-app` used to insert configuration into the application
  context.
- **DeclarativeApplication** — the UI-mapping factory/proxy class in
  `automojo-uia`.  Returns a concrete `UIApplication` when valid.
- **Device** — a landscape entity representing a host, instrument, or
  service.
- **FactoryProtocolBase / SuperFactory** — the precedence-ordered
  factory machinery from `automojo-extension`.
- **FlowEngine / FlowNode / Task / TaskGroup** — the taskplus task
  model.  See Vol 3 ch. 18.
- **InteropLibraryTestJob** — the landscape-aware hinge class in
  `automojo-interop` that breaks the would-be circular dependency
  between `testbase` and `landscaping`.  Parent of `DistributedTestJob`.
  See Vol 2 ch. 2.
- **Landscape** — the model of hosts / devices / services under
  automation.  See Vol 6 ch. 1.
- **MergeMap** — the automojo-configuration map type used for layered
  configuration with priority-ordered insertion.
- **Persona** — the application-identity model from
  `automojo-this-app`.  Established by `establish_app_persona`.
- **SemanticError** — the exception type that **must always bubble**;
  inherits from `BaseException`, not `Exception`, specifically so
  generic catches don't silence it.  See Vol 0 ch. 3.
- **Topology** — the relationships between landscape entities.
  See Vol 6 ch. 3.
- **TESTBASE_MAX_EXAMPLES_CAP** — the runtime-config-as-ceiling pattern
  that switches `@given` properties between fast-CI (`=10`) and
  full-sweep (unset).  Tiny example of Vol 3 Part IV's thesis.

### Decision records (proposed series-level appendix)

These are the "why we did it this way" essays that should sit
end-of-series so any volume can reference them without rehashing.

1. Why no `@retry` decorator — Aspects pattern instead.
2. Why `SemanticError` must always bubble.
3. Why properties are simple data accessors only.
4. Why typed configurations replace environment variables at the
   orchestration / job boundary.
5. Why every function ends in an explicit `return`.
6. Why parameters are passed by name, not by position.
7. Why per-OS sysman specs are paired with implementation, never
   authored speculatively ahead of code.

### Audit diary appendices

The recent audits are pre-built case-study material — capturing the
audit transcript alongside the fix commits gives readers a "code review
in motion" view that's hard to fake.

| Audit | Volume | Source |
|---|---|---|
| `automojo-landscaping` six-pass audit | Vol 6 | 19 bugs across 14 files; +101 tests across 11 files |
| `automojo-taskplus` audit + tests | Vol 3 | 17 bugs across 8 modules; +115 tests across 13 files |
| `automojo-uia` audit | Vol 6 | 20 bugs (Crit/High/Med); 31-file docstring fix |
| `automojo-testplus` two-pass audit | Vol 2 | 14 bugs across 9 files; led to the inheritance-chain refactor |
| Foundation-tier zero-test gap closure | Vol 0 / Vol 1 | 334 tests added across `errors`, `interfaces`, `storage-database`, `networking` |

---

## Series production notes

### What's *missing* from the portfolio as book-source

The portfolio's strength is uniform structure and disciplined contracts.
The portfolio's weakness as book-source is that **case-study material is
thin** — most repos have specs and code but few `case-studies/` or
`walkthroughs/` directories.  To turn the portfolio into a book series
the missing pieces are:

1. A **"why we did it this way"** essay per volume — currently scattered
   in commit messages and `AGENTS.md`.
2. **Real-world walkthroughs** — debugging-session transcripts, refactor
   narratives, audit pass write-ups (the audits listed above are the
   raw material).
3. **Reader exercises** — most repos have `examples/` directories but
   the examples are run-once demos, not graduated learning paths.

### Drafting recommendations

- **Draft Volume 3 first.**  It is the headline volume; the recent work
  on taskplus and configuration makes it the most concretely
  book-shaped; success here sets the voice for the rest.  Volume 0 can
  be drafted alongside since Vol 3's chapter 1 needs Vol 0's framing.
- **Capture audits as you go.**  The next time a major audit happens,
  preserve the transcript (the user-facing dialog plus the file:line
  findings plus the fix commits) into `case-studies/<package>/<date>/`
  in this repo.  These become appendix material.
- **Keep code anchors.**  Every chapter outline above can be backed by
  concrete file:line references; the drafting process should preserve
  those references rather than paraphrase them.
- **Use the recent inheritance-chain work as the model chapter.**  The
  `automojo-testplus` `5ae46f5` + `automojo-interop` `c785e9b` pair
  shows what a clean evolution-of-a-design narrative looks like; aim
  for that quality in every volume's flagship chapter.

### Open questions to answer before each volume drafts

A consolidated list of the open questions called out in the per-volume
sections, plus series-level questions:

- **Vol 3:**
  - Is "Specifications" a sixth managed type within
    `automojo-configuration` or a separate concept living in
    `automojo-provisioning` / per-repo `specs/`?
  - What credential-resolution backends are shipped vs reader-supplied
    in `automojo-credentials`?
  - Are there profile-introspection commands or programmatic
    accessors for chapters 14–17 to anchor on?
- **Vol 5:**
  - Testing strategy decision for `automojo-restclient-github-public`
    (568 modules, 0 tests) — smoke-only? hand-test the framework?
- **Vol 7:**
  - How much of the book is "build an agent SDK" (recipe) vs "use
    `automojo-agent-kit`" (reference)?
- **Series-level:**
  - Publishing model — single publisher, per-volume or boxed set,
    print + ebook + open-source companion?
  - Voice — first-person AutomationX persona throughout, or
    third-person for objectivity in case studies?
  - Code samples — do they live inline in the book, or in a separate
    `automojo-books-examples/` repo with chapter-tagged branches?

---

*This document is the foundation for series planning.  As volumes are
drafted, the per-volume sections will fork into individual planning
files under a `volumes/` directory in this repo.  Until then, edits
should land in this single document so the whole picture stays
coherent.*
