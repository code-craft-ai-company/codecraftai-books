# Chapter 1 — What Integration-Level Automation Is

> *Most CI failures are not test failures.  Most CI failures are
> environment failures, configuration failures, or coordination
> failures.  The discipline that separates the three is integration
> automation.*

A pipeline run is conventionally drawn as a stack of layers:

```
┌─────────────────────────────────────────────────────┐
│  CI provider (GitLab CI / GitHub Actions / Jenkins) │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  Orchestration                                      │  ← integration
│   "what high-level steps run, in what order,        │     automation
│    with what fallbacks?"                            │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  Jobs                                               │
│   "given step N is 'run tests', execute the         │
│    test suite (possibly across N hosts)."           │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  Tasks                                              │
│   individual tests, sysman feature calls,           │
│   interop ops, the things the jobs are made of      │
└─────────────────────────────────────────────────────┘
```

This chapter is about the second layer.  Most teams conflate it with
the first or the third — they put orchestration logic into vendor YAML,
or they put orchestration logic into job code as conditional branches
keyed off environment variables.  Both choices have predictable failure
modes.  Both choices are how the fifty-line YAML grows to six hundred
lines.

## What "integration-level" means

Three jobs to be done sit at the orchestration layer:

1. **Sequencing.**  Which steps run, in what order, with what
   parallelism, with what fallback when one fails.
2. **Parameterisation.**  Which configuration is in scope for this run
   — which landscape, which runtime profile, which credential set,
   which data profiles.  This is not "set forty environment variables";
   this is "select named typed configurations."
3. **Coordination.**  How information flows between steps — the output
   of step *N* informing the input of step *N+1*; the artifacts of
   build informing the inputs of test; the results of test informing
   the deploy gate.

A well-built orchestration layer has clean expression of all three.  A
typical CI YAML expresses (1) clearly, (2) badly, and (3) by accident
— the accidental expression being string-substitution into shell
commands of values that came from environment variables that were
exported by an earlier step that nobody remembers anymore.

## What "integration-level" doesn't mean

Three things that are *not* integration-level automation, despite often
being conflated with it:

- **Build automation.**  Compiling code, linking artifacts, packaging
  containers — these are tasks an orchestration may *invoke*, but they
  are not orchestration.  A build script is a single step in a larger
  flow.  When the build script tries to also be the orchestration, the
  Makefile is fifteen hundred lines and contains a function called
  `is_running_in_ci`.
- **Job execution.**  Running a test suite, running a deployment,
  running a sysman feature against a fleet — these are jobs.  Jobs
  may run on multiple hosts; jobs may have internal sequencing of
  their own (Volume 2 on distributed automation is about exactly this).
  But a job is invoked *by* an orchestration; it is not an orchestration.
- **Provisioning.**  Creating cloud resources, allocating containers,
  setting up databases — these are *resources* an orchestration
  consumes, expressed declaratively as Specifications (chapter 4).
  When a CI YAML contains forty lines of `aws ec2 create-instance`
  shell, those lines are misplaced; the orchestration should consume a
  resource by name, not by recipe.

The discipline of integration automation is to keep these three out
of the orchestration layer.  The orchestration says "this Specification
must be live" or "run this Job" or "publish this artifact."  It does
not contain the recipe for any of the three.

## Why this matters

Three concrete consequences flow from getting the layer separation
right:

### Pipelines become readable

A workflow that invokes a Job by name (rather than inlining seventy
lines of shell to do the Job's work) is short.  Short pipelines fit in
code review; long pipelines do not.  Pipelines that fit in code review
get reviewed; pipelines that do not, do not.  The first place
discipline pays off is in the simple act of reading what the pipeline
does on a Tuesday afternoon when something has gone wrong.

### Pipelines become portable

A workflow expressed in `automojo-taskplus`'s YAML is identical
whether it runs on GitLab CI, GitHub Actions, Jenkins, or a developer's
laptop.  The CI vendor's YAML shrinks to a trigger and a runner image;
everything else moves into the portable workflow.  The portable
workflow can be checked into the same repo as the code; it can be
versioned with the code; it can be diffed; it can be tested in
isolation.

### Pipelines become debuggable

When a pipeline fails, the question "what configuration was in scope
when it failed?" has an answer.  The answer is the names of the typed
configurations the orchestration selected: landscape `staging-east`,
runtime `verbose`, credentials `ci-runner`.  Those are typed objects
with schemas; they are not the union of forty environment variables
exported in unknowable order.  An engineer reproducing the failure
locally selects the same names.  This is the subject of Part IV.

## The shape of this volume

The volume has six parts.  This chapter — and the next two — present
the discipline.  Part II names the six configuration categories and
introduces each.  Part III is the credentials chapter (which is a
thicker subject than the other five categories combined and earns its
own part).  Part IV is the local-dev → CI → staging → prod story —
how the discipline pays off in real workflows.  Part V is the
implementation in `automojo-taskplus`.  Part VI is the case study and
the wiring to real CI vendors.

A reader who learns nothing else from this volume should leave with
this: the orchestration layer is not the place for environment
variables.  Environment variables are the symptom of a missing
configuration category.  Every time you find yourself reaching for an
`export FOO=bar` in a pipeline, ask which of the six categories the
value belongs in — and then put it there instead.

## Summary

- Integration-level automation is the orchestration layer between the
  CI provider and the jobs that do work.
- It has three jobs: sequencing, parameterisation, coordination.
- Most teams conflate it with build automation, job execution, or
  provisioning.  All three conflations are how pipelines grow
  unmanageably.
- The discipline of this volume is to keep the orchestration layer
  small and to express its parameterisation as named typed
  configurations rather than as environment variables.
- The six configuration categories — Specifications, Landscapes,
  Topologies, Runtimes, Credentials, Data Profiles — are introduced in
  Part II.
