# Chapter 27 — Where This Approach Is *Not* the Right Fit

> *Every opinionated discipline accumulates a list of cases where
> the opinion is wrong.  The chapters before this one made the
> case for the discipline; this one is honest about its limits.*

The volume has spent twenty-six chapters arguing for a specific
discipline: typed configurations as the contract between
orchestrations and jobs; the plumbing-vs-content boundary kept
small and fixed; a four-environment ladder where profile
selection replaces conditional branching.  The case study in
chapter 26 showed the discipline pay off on a real pipeline.

This chapter is the inverse.  Six categories of work where the
discipline does *not* fit, named honestly, with the alternative
tools that fit each better.  A short chapter — eight categories
of mismatch is enough — but an important one.  A reader who
finishes the volume and tries to apply the discipline to a
case it doesn't fit will produce something worse than what
they started with.  Knowing where to *not* apply the discipline
is part of knowing the discipline.

## The diagnostic question

Before any of the categories below, a single diagnostic question
catches most of them:

> *Is the orchestration's primary concern the **shape of the
> work** (which steps, in which order, with which configurations)?*

When the answer is yes, the discipline fits.  Pipelines, integration
suites, deployment workflows, scheduled batch jobs — all of
these are *shape-of-the-work* problems.  The chapter-26 case
study fits the discipline because the pipeline's value is in the
declarative description of what it does.

When the answer is no — when the orchestration's primary concern
is something *other* than shape (latency, parallelism, state,
event-reactivity, real-time response) — the discipline either
fits poorly or doesn't fit at all.  The categories below name
the most common no-cases.

## Category 1 — Heavy parallel fan-out

The taskplus engine sequences items in declaration order; a
TaskGroup with five children runs them one at a time.  When the
work *requires* parallelism — running a thousand subtests
concurrently against a partitioned dataset, fanning out a single
deploy across a hundred shards — the engine's sequential walk is
the wrong shape.

A workflow that wants 100-way parallelism could express it as
chapter-23 wrapper tasks that internally spawn parallel work,
but at that scale the wrapper task is doing the orchestration
job and the engine is just a launcher.  This is the
chapter-23 anti-pattern of "tasks that wrap arbitrary Python" —
the engine can't see the work being done.

### What fits better

For *static* heavy parallelism (the same task across N shards),
CI vendors' matrix-build features are usually the right answer.
GitLab's `parallel: matrix:`, GitHub Actions' `strategy.matrix:`,
Jenkins's `parallel { ... }`.  The chapter-24 vendor shim grows
to define the matrix; the per-shard taskplus workflow stays
small.

For *dynamic* heavy parallelism (variable shard count determined
at runtime, work-stealing schedulers, complex scheduling
constraints), bring in a real workflow engine: Argo Workflows,
Apache Airflow, Prefect, Temporal.  These have first-class
parallelism, dependency-aware scheduling, and dynamic-graph
support.  The discipline of typed configurations from Parts
II–IV still applies — the parallel engine consumes the same
landscape, runtime, credentials, and data profiles — but the
*orchestration* layer is the parallel engine, not taskplus.

## Category 2 — Dynamic workflow generation from streaming events

The taskplus YAML is parsed once at workflow start.  The flow
tree is fixed for the duration of the run.  When the workflow
needs to *change shape* based on events that arrive during
execution — adding tasks because new shards arrived, removing
tasks because conditions changed, restructuring the graph
mid-run — the static YAML is the wrong representation.

The discipline's `if_expr` gates can hide some dynamism (a
chapter-22 gate keyed on `flowstate` can include or exclude a
node based on what an earlier task wrote), but the *graph
structure* is fixed.  When the workflow needs new *nodes* that
didn't exist in the YAML, taskplus can't express it.

### What fits better

Event-driven orchestration tools: Apache NiFi, Apache
StreamPipes, AWS Step Functions with dynamic states, or a
custom Kubernetes Operator pattern.  Each models the workflow
as a *graph that mutates in response to events* — the engine's
core abstraction is exactly what the static-graph engine
doesn't have.

A common hybrid pattern: use taskplus for the *static*
backbone of an orchestration (the shape that's known at
design time), and have a taskplus task invoke an event-driven
sub-orchestrator for the dynamic portion.  The sub-orchestrator
runs to completion, returns a result; the taskplus workflow
sequences around it.  The discipline keeps the static parts
typed; the dynamic parts live in their proper engine.

## Category 3 — Sub-second event-driven flows

The taskplus engine has measurable startup latency.  Loading
the workflow YAML, resolving configurations through
`codecraftai-configuration`, instantiating credential and data-profile
managers, walking the recursive flow tree — all takes hundreds
of milliseconds at minimum, often a second or two on
configuration-heavy workflows.

For orchestration steps that take seconds-to-minutes (the typical
CI / integration / deployment use case), the startup cost is
amortised away.  For orchestration steps that take *milliseconds*
— a per-request fan-out in a serving pipeline, a real-time
feature-flag-based experiment routing, an event-handler chain —
the startup latency dominates.

### What fits better

In-process event handlers (the application's own code, dispatched
through whatever event-routing primitive fits — `asyncio` event
loops, `rxpy`, message-queue consumers).  For inter-service
event flows, dedicated stream processors: Apache Kafka Streams,
Apache Flink, AWS Kinesis Data Analytics.

The discipline of typed configurations *does* still apply at the
service-startup layer: the service that handles events should
read its own configuration through the same chapter-3 typed
mechanisms.  But the per-event orchestration is in the service's
hot path, and that hot path needs a different engine.

## Category 4 — Multi-day stateful workflows with checkpoint/resume

The taskplus engine has no persistent state.  When the
orchestration's process exits — for whatever reason: scheduled
job ends, machine reboots, container is preempted — the run is
gone.  The next run starts from scratch.

This is fine for runs that complete in hours.  For runs that
span days or weeks — a long-running data migration, a multi-stage
ML training pipeline with human-in-the-loop checkpoints, a
phased deployment with manual approval gates — the lack of
persistent state is disqualifying.  The orchestration cannot be
*paused* and *resumed*; it cannot survive the underlying
machine's lifecycle.

### What fits better

Workflow engines with built-in durable state: Temporal, Cadence,
AWS Step Functions, Azure Durable Functions, Apache Airflow's
operator model.  These persist orchestration state to a
backing store (a database, a queue, a state machine) and resume
from where they left off after restart.

For the simpler "long-running but not multi-day" case — a
six-hour nightly run — taskplus is fine; the chapter-16
capture-and-replay primitive provides a different kind of
resumability (recover by replaying captured configuration, not
by resuming mid-run state).

## Category 5 — ML training pipelines

ML training has its own orchestration vocabulary: experiment
tracking, hyperparameter sweeps, model versioning, dataset
versioning, GPU scheduling, checkpoint management.  These are
specialised concerns that map poorly onto the chapter-3
configuration categories.  A landscape doesn't capture "the
GPU model you need"; a runtime doesn't capture "the experiment
tracking server"; the typed-credential vocabulary doesn't
capture "an ML platform's experiment session token."

The discipline could be *extended* to cover these — chapter-23's
extension recipe applies — but the cost-benefit changes.  The
extensions would replicate functionality that exists in
purpose-built ML tooling.  When a team's ML stack already
includes MLflow, Weights & Biases, or Kubeflow, the
orchestration layer should be those tools, not a custom
taskplus extension.

### What fits better

Purpose-built ML orchestration tools: Kubeflow Pipelines,
Metaflow, ZenML, Argo Workflows with the ML extensions, AWS
SageMaker Pipelines.  Each models ML-specific concerns as
first-class citizens.

The discipline of typed configurations still applies *around*
the ML pipeline — the data profiles for the training data, the
credentials for the model registry, the runtime knobs for the
training-cluster — but the orchestration layer is the ML tool.

## Category 6 — Real-time / hard-deadline systems

Real-time orchestration — a workflow that *must* respond within
a guaranteed time bound, where missing the deadline is a system
failure — is a different engineering problem.  The taskplus
engine is not real-time; the OS schedulers it runs on are not
real-time; the configuration-loading machinery has no timing
guarantees.

For orchestration in *soft* real-time (best-effort latency
targets, occasional misses tolerated), other tools are
better-fit but the gap is smaller.  For *hard* real-time
(industrial control systems, automotive ECUs, medical devices),
the entire toolchain is different — RTOS, time-triggered
schedulers, formal verification.

### What fits better

For hard real-time, RTOS-based architectures: VxWorks, QNX,
RT-Linux's PREEMPT_RT, or specialised industrial control
frameworks like CODESYS.  The volume's discipline does not
extend to these environments and the volume's audience is not
typically working in them.

## Category 7 — One-off scripts and small projects

The discipline has a fixed cost.  Authoring six configuration
files, a profile set, and a workflow YAML for a project that
runs *one* CI job and is going to be archived in three months
is overkill.  The cost of the discipline exceeds the benefit
when the surface area is small enough that the chapter-2
failure modes can't really accumulate.

A useful rule of thumb: when the project's CI YAML is *under
fifty lines* and is *unlikely to grow*, the discipline is more
ceremony than payoff.  Use the vendor's YAML directly; keep
the secrets in the vendor's secret store; defer the discipline
until the project's CI grows past a hundred lines or branches
across multiple environments.

### What fits better

Vendor-native YAML, kept simple.  When the file gets big
enough to start showing chapter-2 failure modes, the discipline
is worth adopting.  Until then, the cost is too high.

## Category 8 — Highly opinionated platforms with their own orchestration

Some platforms come with their own deeply integrated
orchestration: Salesforce's Apex/Flow, ServiceNow workflows,
Workday Studio, Pega.  These platforms expect orchestration
to be expressed in their native vocabulary; integrating a
taskplus workflow with them is fighting the platform.

When a team is building *on top of* such a platform, the
discipline of this volume applies *to the parts of the work
outside the platform* (the build pipeline, the integration
testing of the platform-side code, the deployment of supporting
services), not to the platform's own orchestration.

### What fits better

Use the platform's native orchestration for platform-side work;
apply the discipline of this volume to whatever is *not* the
platform.  The boundary is wherever the platform's
orchestration capabilities end.

## The build-vs-buy question

Behind every category above is the same question: when should
a team *adopt* the discipline of this volume versus *bring in
a different tool*?

A small framework for the decision:

### Adopt the discipline when

- The orchestration is *primarily* about the shape of the
  work — sequencing of build, test, deploy, cleanup phases
  with dependencies and conditional inclusion.
- The orchestration runs on a manageable cadence — minutes to
  hours — and complete-restart-on-failure is acceptable.
- Multiple environments (local-dev / CI / staging / prod) are
  in scope and the team values configuration-driven differences
  over code-driven ones.
- The team has the capacity to maintain typed configuration
  files and the discipline that comes with them.
- The tool ecosystem the team is already using
  (`codecraftai-*`, or any portable typed-config toolkit) is
  available and stable.

### Bring in a different tool when

- The orchestration is primarily about something other than
  shape — parallelism, state, event-reactivity, real-time
  response.
- A purpose-built tool already exists for the specific problem
  (ML pipelines, durable workflows, event-stream processing).
- The team is small enough that adopting the discipline's
  ceremony costs more than it saves.
- The orchestration's lifecycle requirements exceed what
  taskplus provides (multi-day, hard-deadline,
  sub-second-latency).

The volume has nothing against any of the alternative tools.
Each fits its niche.  The discipline is *opinionated*, not
*universal*; opinionated is the only way to be useful at all.

## A note on hybrid systems

Real-world automation rarely lives in one orchestration tool.
A team might have:

- **Argo Workflows** for ML training (durable, parallel).
- **Temporal** for multi-day customer-onboarding workflows
  (durable, stateful).
- **Apache Kafka Streams** for real-time event processing
  (low-latency, event-driven).
- **`codecraftai-taskplus`** for CI / integration / deployment
  pipelines (typed, portable, environment-spanning).

The discipline of this volume applies to the *taskplus
portion* of the team's automation portfolio.  The other tools
have their own disciplines; the four can coexist without
conflicting.

What *is* shared across all four is the **typed configuration
discipline** of Parts II–IV.  Every tool above benefits from
having its credentials in a typed credential vocabulary, its
landscapes in typed inventory files, its runtime knobs in
typed runtime profiles.  The configuration discipline is more
universal than the orchestration discipline.  A team that
adopts the configuration discipline gets the inter-tool
coherence even when the orchestration tools differ.

This is the volume's final claim: **the configuration
discipline survives independently of the orchestration tool.**
A team that walks away from this volume preferring Argo or
Temporal over taskplus has not failed to adopt the discipline
— they have adopted the *transferable* part.  The taskplus
implementation is the volume's worked example, not the
discipline itself.

## Closing the volume

The volume's central thesis, restated for the last time:

> *Configuration files allow arbitrary parameterisation of
> runtime behaviour without modifying the automation plumbing
> code.*

Twenty-seven chapters made the case.  The six categories of
typed configuration (Specifications, Landscapes, Topologies,
Runtimes, Credentials, Data Profiles).  The credential
discipline that earns its own four chapters.  The four-environment
ladder where profile selection replaces codebase divergence.
The taskplus implementation that ties the configurations to
real workflow execution.  The vendor wiring that confines the
CI YAML to ten lines of plumbing.  The case study that ports a
200-line monstrosity to 30 lines of vendor shim plus 17 typed
configuration files in nine commits over thirty days.

The discipline is opinionated.  It is not universal.  This
final chapter has been honest about where it doesn't fit.

What the discipline *does* offer — across every place it does
fit — is the same payoff in different shapes.  Pipelines that
are readable.  Pipelines that are portable.  Pipelines that are
debuggable.  Pipelines that survive vendor migrations as
thirty-day projects rather than multi-quarter rewrites.
Configuration drift that is detectable; regressions that are
attributable; cross-environment surprises that are rare.

The discipline is, ultimately, a small set of decisions:

- Treat configuration content as data, not as plumbing.
- Reference configurations by stable identifiers; resolve them
  through machinery, not through application code.
- Compose environments through profiles, not through codebases.
- Express flow as data; let the engine read it.
- Keep the plumbing layer small enough to fit on a screen.

Five decisions.  Twenty-seven chapters of consequences.  The
team that makes the decisions ships pipelines that scale; the
team that doesn't ships the chapter-26 200-line YAML and grows
into the chapter-2 failure modes.

The choice is the reader's.  This volume has done what books can
do — name the discipline, show the implementation, walk the
worked example, scope the limits.  The application is the
reader's project, with the reader's team, against the reader's
constraints.

The next volume in the series — Volume 4 — develops the
cross-OS system-management discipline introduced briefly through
the chapter-25 sysman bridge.  Where this volume has been about
*how the work is orchestrated*, the next volume is about *what
the work is*: the 52 cross-OS contracts that let a single
sysman feature run on RHEL, Ubuntu, macOS, Windows, and UEFI
without per-OS branches in the calling code.  The discipline
is the same; the surface area is different.

For now: the volume is finished.  The tools it has named are
real; the audits it has cited produced real bugs and real test
suites; the configurations it has shown have been read,
audited, and run.  Apply the discipline where it fits.  Skip it
where it doesn't.  The next pipeline you build will be easier
than the last one if you do.
