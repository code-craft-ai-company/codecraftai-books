# Volume 3 — Detailed Chapter Outline

This document goes one level deeper than the volume entry in
[`automojo-volumes.md`](../../automojo-volumes.md).  For each chapter
it lists: the opening hook, the key arguments, the code anchors, the
worked examples, the reader exercises, the cross-references, and an
estimated page count.

The outline is updated as drafting proceeds.  Chapters with status
"drafted" have a corresponding file under `chapters/`; chapters with
status "outlined" have detail here but no draft yet.

---

## Part I — The Discipline

### Chapter 1 — What Integration-Level Automation Is

- **Status:** drafted (`chapters/01-what-integration-automation-is.md`)
- **Hook:** Most CI failures are not test failures.  Most CI failures
  are environment failures, configuration failures, or coordination
  failures.  The discipline that separates the three is integration
  automation.
- **Key arguments:**
  1. The pipeline stack has four layers (CI provider / orchestration /
     jobs / tasks); this volume is about the orchestration layer.
  2. Orchestration has three jobs: sequencing, parameterisation,
     coordination.
  3. Most teams conflate orchestration with build automation, job
     execution, or provisioning — predictable failure modes follow.
  4. Done well: pipelines become readable, portable, debuggable.
- **Code anchors:** none in this chapter (conceptual).
- **Worked examples:** layered diagram of pipeline stack.
- **Exercises:** none.
- **Cross-references:** forward to Part II (categories), Part V
  (taskplus implementation), Part VI (case study).
- **Pages:** 12–15.

### Chapter 2 — Why Most CI YAML Grows Monstrously

- **Status:** drafted (`chapters/02-why-ci-yaml-grows.md`)
- **Hook:** Every six-hundred-line CI YAML file was once a fifty-line
  file.  No one set out to write the six-hundred-line version.  The
  growth is the symptom of a category mistake at the first commit.
- **Key arguments:** six failure modes, each the absence of a typed
  configuration category:
  1. Environment-variable accretion (all six categories missing)
  2. Conditional thicket (Runtimes missing)
  3. Inlined recipe (Specifications missing)
  4. Credential leak through env (Credentials missing)
  5. Host name in YAML (Landscapes + Data Profiles missing)
  6. Topology buried in step ordering (Topologies missing)
- **Worked examples:** an env-var accretion timeline; a conditional
  thicket smearing four orthogonal concerns into one shell expression.
- **Exercises:** none in this chapter (the prescription comes in Parts
  II–IV).
- **Cross-references:** forward to chapters 4 (specifications), 5
  (landscapes), 6 (topologies), 7 (runtimes), 8 (data profiles), 9–12
  (credentials).
- **Pages:** 15–20.

### Chapter 3 — The Contract: Six Typed Configuration Categories

- **Status:** drafted (`chapters/03-the-six-categories.md`)
- **Hook:** Every parameter to every pipeline belongs in exactly one
  of six categories.  When you find yourself reaching for an
  environment variable, ask which of the six the value belongs in —
  and then put it there instead.
- **Key arguments:**
  1. Six categories — Specifications, Landscapes, Topologies,
     Runtimes, Credentials, Data Profiles.
  2. Five live in `automojo-configuration`; Specifications lives in
     `automojo-provisioning`.
  3. All six share five properties: typed at rest, looked up by name,
     optionally encrypted, composed via `MergeMap`, resolved into
     application context.
  4. The categories form a layered model — Specifications create
     the inventory, Landscapes describe it, Topologies describe its
     relationships, Runtimes/Credentials parameterise the processes,
     Data Profiles name the data.
- **Code anchors:**
  - `automojo-configuration/source/packages/automojo/configuration/configurationmaps.py`
    — five `CONFIGURATION_MAPS` instances.
  - `automojo-configuration/source/packages/automojo/configuration/configurationvariables.py`
    — five sets of variable descriptors.
  - `automojo-configuration/source/packages/automojo/configuration/sources/directorysource.py`
    — `.yaml`/`.yml`/`.json` loader fallback.
  - `automojo-xmodules/.../xcollections/mergemap.py` — composition
    machinery.
  - `automojo-this-app/.../contextpaths.py` — context insertion
    points.
- **Worked examples:** a YAML fragment for each of the six
  categories.
- **Exercises:** none yet (one per category in Part II).
- **Cross-references:** forward to all of Parts II, III, IV.
- **Pages:** 18–22.

---

## Part II — The Configuration Categories

### Chapter 4 — Specifications

- **Status:** outlined.
- **Hook:** *(draft) "A pipeline that contains forty lines of `aws ec2
  create-instance` is two pipelines pretending to be one.  The first
  is a provisioner; the second is the actual orchestration.  The
  Specification separates them."*
- **Key arguments:**
  1. Specifications declare resources that must exist *before* the
     orchestration runs.  They are checkout-or-create manifests.
  2. The portfolio implements specifications in
     `automojo-provisioning`, distinct from the configuration loader.
  3. Specification YAML has top-level `apod` and `environment` keys.
  4. Idempotent on re-run is the design contract: a spec is
     applied; the desired state is reached; subsequent applies are
     no-ops.
  5. The orchestration consumes the *output* of specification
     application — a live landscape — without containing the recipe.
- **Code anchors:**
  - `automojo-provisioning/source/packages/automojo/provisioning/provisioningengine.py:55-73`
    — `create_provisioning_engine(specification=...)`.
- **Worked examples:**
  - A full `spec-runner-pool.yaml` example creating a 3-node EC2 runner pool.
  - The orchestration that consumes the resulting landscape — note
    that the orchestration does not know whether the landscape was
    created from the spec or pre-existed.
- **Exercises:**
  - Write a Specification for a 2-node Postgres cluster (primary +
    replica).  Show the corresponding landscape that would result.
- **Cross-references:** forward to chapter 5 (landscapes — the output
  of spec application); forward to chapter 25 (orchestration consuming
  specs as a step).
- **Pages:** 18–22.

### Chapter 5 — Landscape Files

- **Status:** outlined.
- **Hook:** *(draft) "A test that runs against staging-db doesn't need
  to know that staging-db lives at db-staging-east-1.internal.  The
  test needs to know that staging-db exists.  Where it lives is a
  property of the landscape, not of the test."*
- **Key arguments:**
  1. A landscape is the inventory of what's in scope for a pipeline
     run.
  2. Top-level keys (per the existing example): `environment` (label,
     region, etc.), `apod` (the automation pod — clients/runners),
     `infrastructure` (services under test).
  3. Landscapes are versioned files.  Multiple landscapes coexist
     (staging-east, staging-west, prod-readonly); the pipeline
     selects one by name.
  4. The Landscape category lives in `automojo-configuration`'s
     `LANDSCAPE_CONFIGURATION_MAP`; the in-memory model lives in
     `automojo-landscaping` (treated in Volume 6).
- **Code anchors:**
  - `automojo-automation-kit/userguide/examples/landscape.yaml`
    — the canonical example file.
  - `automojo-configuration/source/packages/automojo/configuration/configurationmaps.py`
    `resolve_landscape_configuration` function.
- **Worked examples:**
  - The staging-east landscape file from the chapter 3 preview,
    expanded to a complete file with apod members + infrastructure
    services.
  - A second landscape (local-dev) showing how the same pipeline
    runs against a single-host local landscape.
- **Exercises:**
  - Write a landscape file for a 5-host, 2-tier (web/db) test
    environment.  Distinguish the apod from the infrastructure.
- **Cross-references:** forward to chapter 6 (topologies layered on
  top); Volume 6 chapter 1 (the in-memory `Landscape` class).
- **Pages:** 22–26.

### Chapter 6 — Topology Files

- **Status:** outlined (no real example file in portfolio yet; book
  introduces canonical example).
- **Hook:** *(draft) "Step ordering is a flat list.  Component
  dependency is a graph.  When a flat list pretends to be a graph,
  inserting a step in the middle becomes a logic puzzle."*
- **Key arguments:**
  1. A topology is the explicit graph of relationships between
     landscape entities.
  2. A landscape can host many topologies (the same VMs serve
     different roles in different test scenarios).  A topology can
     apply to many landscapes (the same dependency graph holds across
     staging-east and staging-west).
  3. Topology gives the engine the information it needs to sequence
     steps automatically — the flat list of steps becomes an *output*
     of topology resolution rather than the input.
  4. Cluster membership, dependency edges, coordinator selection are
     all topology-level concerns.
- **Code anchors:**
  - `automojo-configuration/.../configurationvariables.py` — the
    `TOPOLOGY_*` descriptors.
  - `automojo-configuration/.../configurationmaps.py`
    `resolve_topology_configuration`.
  - *(file format example to be authored as part of book)*
- **Worked examples:**
  - The book introduces a canonical topology schema (clusters,
    dependencies, ordering).  Schema is contributed back to the
    portfolio's `userguide/examples/` directory.
  - A topology layered on the staging-east landscape from chapter 5,
    showing how the same hosts can serve different topologies.
- **Exercises:**
  - Given the 5-host landscape from chapter 5, write a topology that
    expresses the dependency "smoke-tests need web-tier; web-tier
    needs db-tier."
- **Cross-references:** backward to chapter 5; Volume 6 for the
  in-memory model.
- **Pages:** 18–22.

### Chapter 7 — Runtime Files vs Environment Variables

- **Status:** outlined — the *thesis chapter* of the volume.
- **Hook:** *(draft) "The cleanest single example of the discipline is
  a single line.  `TESTBASE_MAX_EXAMPLES_CAP=10`.  That line, in a
  fast-CI runtime profile, switches every property test in the suite
  from a 100-case sweep to a 10-case sweep — without touching a line
  of test code.  The same line, omitted from a nightly profile, lets
  the suite run at full strength.  Nothing in the test source asks
  whether it's CI; the runtime profile carries the answer."*
- **Key arguments:**
  1. Why env vars proliferate (every team adds one, no one removes one).
  2. What goes wrong (forgotten exports, namespace collision, type
     confusion, no schema, no provenance).
  3. What runtime configs fix (typed schemas, profile selection at
     startup, namespaced overrides, traceable provenance).
  4. The migration pattern: env-var-soup → runtime profile + 1-line
     `--profile prod` selector.
  5. A shipped runtime YAML has these top-level keys: `breakpoints`,
     `debugger`, `logging`, `paths`, `paths-templates`,
     `results-configuration`, `testroot`, `version`.  Adding a new
     knob means adding a key here, not exporting a new env var.
- **Code anchors:**
  - `automojo-testbase/source/testroots/report-tests/result-tests/v2/example2/runtime-declared.yaml`
    — real shipped runtime profile.
  - `automojo-testbase/userguide/10-04-03-testing-frameworks-testbase.rst`
    — the `TESTBASE_MAX_EXAMPLES_CAP` documentation.
  - `automojo-this-app/source/packages/automojo/thisapp/thisapplication.py:222-256`
    — `override_variable_value` (the mechanism by which a runtime
    config overrides a default).
- **Worked examples:**
  - Before: a CI YAML with 12 environment-variable exports for log
    levels, retry counts, parallelism, and timeouts.
  - After: a single `runtime-fast-ci.yaml` referenced by name; the CI
    YAML drops to 0 exports.
  - Side-by-side: `runtime-fast-ci.yaml` vs `runtime-full-sweep.yaml`
    showing what changes between the two profiles.
- **Exercises:**
  - Identify three environment variables in a CI pipeline you
    maintain and migrate them into a runtime profile.
- **Cross-references:** Forward to chapter 14 (profiles as a discipline);
  forward to chapter 16 (reproducing CI failures locally).
- **Pages:** 25–30.  The thickest chapter in Part II.

### Chapter 8 — Data Profiles

- **Status:** outlined (no real example file in portfolio yet; book
  introduces canonical example).
- **Hook:** *(draft) "A connection string in environment-variable form
  is a hostname, a port, a username, a password, a database name, and
  three optional flags concatenated into a string nobody can parse
  reliably.  A data profile is the same six items as named, typed
  fields."*
- **Key arguments:**
  1. A data profile names a data store: hostname, port, database
     identifier, schema, transport flags, and a credential
     *reference*.
  2. The pipeline references the profile by name; the profile resolves
     at runtime.  When the host changes, the profile changes; the
     pipeline does not.
  3. The portfolio defines several profile classes
     (`BaseDataProfile`, `DatabaseBasicSocketProfile`,
     `DatabaseBasicTcpProfile`, `CouchDBProfile`,
     `MongoDBAtlasProfile`, `SnowflakeProfile`).
  4. Profiles compose with credentials: the profile says *where*; the
     credential says *who*.  The two are deliberately separate.
- **Code anchors:**
  - `automojo-dataprofiles/source/packages/automojo/dataprofiles/`
    — the profile class hierarchy.
  - `automojo-configuration/.../configurationvariables.py` — the
    `DATAPROFILE_*` descriptors.
- **Worked examples:**
  - A `dataprofile-staging-db.yaml` with `SnowflakeProfile` +
    `DatabaseBasicTcpProfile` for staging Redis (introduced in chapter
    3, expanded here).
  - The Python code that reads the profile by name and opens a
    connection.
- **Exercises:**
  - Define a `dataprofile-local-dev.yaml` that points the same
    profile names at local Docker containers.
- **Cross-references:** Forward to Part III for the credential
  reference resolution; forward to chapter 14 (profiles as a
  discipline).
- **Pages:** 18–22.

---

## Part III — Credential Management

Substantial chapters anchored on the finding that
`automojo-credentials` is a pure vocabulary layer with no shipped
backends.  Part III therefore documents the vocabulary, then proposes
the backend discipline the reader implements.

### Chapter 9 — Credential Vocabularies

- **Status:** outlined.
- **Hook:** *(draft) "There are only eleven kinds of credential a
  pipeline ever needs.  This chapter names all eleven and explains
  why the right answer to 'what about credential type X?' is almost
  always 'use one of the eleven and add a `categories` tag.'"*
- **Key arguments:**
  1. The eleven shipped credential classes in `automojo-credentials`,
     their fields, and their typical use cases.
  2. Every credential has three universal properties: `identifier`,
     `categories`, `role`.  Why each matters.
  3. The `categories` tag is the extension point for application-
     specific scoping (e.g., `[runner-fleet]`, `[aws-staging]`).
  4. Why this is a *vocabulary* and not a *backend* — the deliberate
     separation between the credential type and the secret-resolution
     mechanism.
- **Code anchors:**
  - `automojo-credentials/source/packages/automojo/credentials/basecredential.py`
    — `BaseCredential`.
  - All 11 concrete credential classes, with file paths from the
    research findings.
- **Worked examples:**
  - A `credentials-ci-runner.yaml` file containing one of each
    common type.
- **Exercises:**
  - Map the credentials your team uses today (likely scattered
    across env vars and shell scripts) into the eleven types.
- **Cross-references:** forward to chapters 10–12 for resolution.
- **Pages:** 18–22.

### Chapter 10 — Credential Resolution

- **Status:** outlined.
- **Hook:** *(draft) "The `automojo-credentials` package gives you the
  vocabulary.  The package does not give you a way to *fetch* the
  secret value.  This chapter is about that gap and how to close it."*
- **Key arguments:**
  1. The portfolio ships zero resolution backends — the design choice
     is that resolution is consumer-supplied.
  2. The reader's resolution layer takes a `Credential` instance and
     returns the same instance with the secret materialised.  It does
     not change the credential's *identity*; it changes only the
     secret material.
  3. Standard backends a reader will likely build: file-on-disk,
     environment-variable-fallback, vault (Vault / AWS SM / Azure KV /
     GCP SM), keychain, prompt, derived.
  4. The resolution-order recipe: prefer specific over generic; prefer
     vault over file; prefer file over env; prefer env over prompt.
  5. Host scoping (also consumer-supplied): the reader's resolver
     consults the active landscape to scope a credential to a host.
- **Code anchors:**
  - `automojo-credentials/source/packages/automojo/credentials/credentialmanager.py`
    — registry pattern; consumer hands it a dict.
  - The 11 credential classes from chapter 9.
- **Worked examples:**
  - A 60-line `CredentialResolver` that reads a credentials YAML
    file, expands `${VAULT:/path}` references, and host-scopes via
    the active landscape.
  - The same resolver wired into an orchestration's startup.
- **Exercises:**
  - Write a `FileBackedResolver` that loads secret values from a
    Fernet-encrypted file (chapter 11 covers the encryption side).
- **Cross-references:** Forward to chapter 11 (encryption at rest);
  forward to chapter 12 (per-environment resolution).
- **Pages:** 25–30.

### Chapter 11 — Credentials in Flight

- **Status:** outlined.
- **Hook:** *(draft) "A credential at rest is a file you encrypt.  A
  credential in flight is a process attribute that lives only in the
  process's memory and never touches disk again.  Most credential
  bugs happen at the boundary between the two."*
- **Key arguments:**
  1. Encryption at rest with Fernet (already shipped in
     `automojo-configuration`'s `cryptography.py`).  Re-use, don't
     re-implement.
  2. Rotation during long runs — what happens when a credential
     expires mid-pipeline.  The discipline: credentials are refreshed
     by reference, not by re-export.
  3. The audit trail: what was used, when, by which step.
  4. The "never log" rule: credentials in `__repr__`, in tracebacks,
     in error messages.  The discipline: all credential classes
     override `__repr__` to elide the secret.
- **Code anchors:**
  - `automojo-configuration/source/packages/automojo/configuration/cryptography.py`
    — Fernet helpers.
  - The 11 credential classes — chapter notes which currently elide
    secrets in `__repr__` and which don't (likely an audit finding).
- **Worked examples:**
  - A `RotatingCredentialResolver` that re-fetches a credential when
    `expires_at` is close.
  - Log redaction patterns.
- **Exercises:**
  - Audit your codebase for `print(credential)` /
    `f"{credential}"` / `logger.info(credential)` patterns.
- **Pages:** 18–22.

### Chapter 12 — Credentials Across Environments

- **Status:** outlined.
- **Hook:** *(draft) "The same credential vocabulary works in local
  dev, in CI, in staging, in prod.  What changes is the backend that
  resolves it.  The pipeline does not know which backend resolved its
  credentials."*
- **Key arguments:**
  1. Local-dev backend: a developer-owned encrypted file in
     `~/.automojo/credentials/`.
  2. CI backend: vault integration; secrets pulled at job start;
     never written to disk.
  3. Staging backend: the same vault, different scope.
  4. Prod-readonly backend: vault with audit logging on.
  5. The pipeline references credentials by *identifier*; the active
     profile (chapter 14) selects the backend.
- **Code anchors:** none new — composes chapters 10 and 11.
- **Worked examples:**
  - A `BackendByProfile` resolver that switches backends based on the
    active profile name.
- **Exercises:**
  - Sketch the migration path from "shared admin credentials in
    1Password" to "per-pipeline credentials in vault" using the
    discipline.
- **Cross-references:** forward to chapter 14 (profiles).
- **Pages:** 15–18.

---

## Part IV — Workflow Automation Across Environments

The selling-point part of the volume.  Per the research findings,
chapters 13–17 *propose* the discipline using portfolio building
blocks rather than documenting shipped tooling.

### Chapter 13 — The Development-Environment Ladder

- **Status:** outlined.
- **Hook:** *(draft) "There are four environments a piece of code
  visits in its lifetime: a developer's laptop, the CI integration
  pipeline, a staging deployment, and prod-readonly debugging.  Most
  teams treat these as four codebases that happen to overlap.  The
  discipline of this part is to treat them as four profiles of the
  same codebase."*
- **Key arguments:**
  1. The four environments — local-dev, integration-CI, staging,
     prod-readonly — are *profiles*, not codebases.
  2. The lifecycle of a single test: written on a laptop, runs in
     CI, debugged from a staging artifact, replayed against prod.
  3. The cost of treating them as four codebases: divergent
     `if 'CI' in os.environ` branches, divergent setup scripts,
     divergent docker-composes.
  4. The cost of treating them as four profiles: a single typed
     ladder where promoting a change means changing the profile, not
     the code.
- **Worked examples:** a single test traced through the four
  environments, showing what changes (the profile name) and what
  doesn't (everything else).
- **Cross-references:** forward to chapters 14–17.
- **Pages:** 12–15.

### Chapter 14 — Configuration Profiles as a Discipline

- **Status:** outlined.
- **Hook:** *(draft) "A profile is a name plus a `MergeMap`.  This
  chapter proposes the convention that turns the existing
  `automojo-configuration` machinery into a user-facing profile
  system."*
- **Key arguments:**
  1. The mechanism exists in code (`MergeMap` composition,
     `override_variable_value`) but is not exposed at the
     configuration-file level.
  2. Proposed file-level convention:
     ```yaml
     extends: base.yaml
     overlay:
       runtime:
         logging.levels.console: DEBUG
     ```
  3. Profile selection at process start: `--profile staging-east`.
  4. The book proposes a `ContextPaths.ACTIVE_PROFILE` slot to hold
     the active profile name across the application context.
  5. A proposed `automojo profile show` CLI command that iterates
     `AppVarSet.items()` and dumps resolved values.
- **Code anchors:**
  - `automojo-xmodules/.../mergemap.py` — the composition primitive.
  - `automojo-this-app/.../thisapplication.py` `override_variable_value`.
  - `automojo-this-app/.../contextpaths.py` — the proposed insertion
    point for `ACTIVE_PROFILE`.
- **Worked examples:**
  - A 4-file profile set: `base.yaml`, `local.yaml`, `ci.yaml`,
    `staging.yaml` showing additive overlay.
- **Exercises:** implement the proposed `automojo profile show`
  command on top of `AppVarSet.items()`.
- **Cross-references:** forward to chapters 15–17.
- **Pages:** 22–26.

### Chapter 15 — The Local-Dev Tax and How Typed Configs Cancel It

- **Status:** outlined.
- **Hook:** *(draft) "The `.env.local` file is a confession.  Every
  team that has one is admitting that their pipeline depends on
  configuration the codebase doesn't name."*
- **Key arguments:**
  1. The local-dev tax: `.env.local`, `direnv`, `.tool-versions`,
     `make setup`, the README's "you might also need to..." section.
  2. Each piece of local-dev tax is a missing typed configuration.
  3. With profiles, the only setup is `--profile local`.
  4. What the chapter walks through: a fresh checkout, every piece
     of "tribal knowledge" required to run the test suite, and how
     each piece migrates into a typed configuration.
- **Worked examples:** a fresh-checkout checklist for a typical
  Python project, before and after typed-config adoption.
- **Cross-references:** chapter 14 for profile composition; chapter 16
  for the reproducibility payoff.
- **Pages:** 15–18.

### Chapter 16 — Reproducing CI Failures Locally

- **Status:** outlined.
- **Hook:** *(draft) "When a test passes locally but fails in CI, the
  question is always 'what's different about CI?'.  With typed
  profiles, the answer is one filename."*
- **Key arguments:**
  1. The replay primitive: capture the resolved profile from the
     failing CI run; reconstruct it locally.
  2. The book proposes a `--replay-config=<file>` flag and a
     corresponding capture step in CI.
  3. The captured profile is a JSON snapshot of all resolved
     variables (with secrets *redacted* — they're re-resolved
     locally from the local backend).
  4. `TESTBASE_MAX_EXAMPLES_CAP` as the worked-out small example —
     the smallest possible runtime knob.
- **Code anchors:**
  - `automojo-testbase` `propertyrunner.py` `ENV_MAX_EXAMPLES_CAP` —
    the canonical small example.
- **Worked examples:** a CI-failure reproduction walked end-to-end —
  failing job, captured profile, local replay, isolated bug.
- **Exercises:** implement the `--replay-config` flag against
  `automojo-this-app`'s `override_variable_value`.
- **Pages:** 18–22.

### Chapter 17 — Profile Promotion

- **Status:** outlined.
- **Hook:** *(draft) "Your local-dev profile is not a draft of your
  staging profile.  Your local-dev profile *is* your staging profile
  with two overlay files added."*
- **Key arguments:**
  1. Profiles compose additively: staging = local + staging-overlay;
     prod = local + staging-overlay + prod-overlay.
  2. No rewrites, no parallel codebases.
  3. The promotion model: when a new knob is added to local, it
     automatically appears in staging unless explicitly overridden.
     This is the opposite of the typical "did anyone update the
     staging config too?" failure.
  4. The audit story: every overlay file is a diff; every promotion
     is a reviewable change.
- **Worked examples:** the chapter-14 4-file profile set traced
  through a knob addition.
- **Pages:** 12–15.

---

## Part V — taskplus and the FlowEngine

The implementation half.  Now-well-known terrain after the recent
audit + 115-test addition.

### Chapter 18 — The taskplus task model

- **Status:** outlined.
- **Code anchors:** `FlowNode`, `Task`, `TaskGroup` in
  `automojo-taskplus/source/packages/automojo/taskplus/model/`.
- **Pages:** 15–18.

### Chapter 19 — Built-in task types

- **Status:** outlined.
- **Code anchors:** `RunCommand`, `ShellScript`, `EmbeddedPython`,
  `GitCheckout` (stub — reader exercise), `CreateEnvironment`
  (transitional tool — runtime configs replace most uses).
- **Pages:** 18–22.

### Chapter 20 — Workflow YAML grammar

- **Status:** outlined.
- **Code anchors:** `Workflow.parse`, `_process_flow_info_items`.
- **Pages:** 18–22.

### Chapter 21 — Passing information to jobs

- **Status:** outlined.  *Anchor of Part V.*
- **Key argument:** the bootstrap-name → resolve-config flow.
  Orchestration injects *names* into job env (e.g.,
  `AM_LANDSCAPE_NAMES=staging-east`); job startup uses
  `automojo-configuration` to resolve names into structured configs;
  job code reads typed configurations, never raw env vars.
- **Pages:** 18–22.

### Chapter 22 — Flow control as data

- **Status:** outlined.
- **Code anchors:** `JumpContext`, `FlowEngine.execute`,
  `process_flowitems`, `evaluate_expression`.
- **Pages:** 15–18.

### Chapter 23 — Extending the task model

- **Status:** outlined.
- **Code anchors:** `import_by_name` / `tasktype: module@ClassName`
  resolution.
- **Pages:** 12–15.

---

## Part VI — Real-World Wiring

### Chapter 24 — Wiring to a real CI vendor

- **Status:** outlined.
- **Worked examples:** GitLab CI shim, GitHub Actions shim, Jenkins
  shim — each ~10 lines.
- **Pages:** 18–22.

### Chapter 25 — Bridges to other layers

- **Status:** outlined.
- **Key arguments:** orchestration → distributed test job (Vol 2);
  orchestration → sysman feature (Vol 4); orchestration → provisioning
  (Vol 6).
- **Pages:** 15–18.

### Chapter 26 — Case study

- **Status:** outlined.  *Capstone of the volume.*
- **Worked example:** porting a 200-line GitLab YAML with 47 env vars
  to a 30-line taskplus workflow + 5 named configs + 4 profiles.
- **Pages:** 30–35.  Longest chapter in the volume.

### Chapter 27 — Where this approach is *not* the right fit

- **Status:** outlined.
- **Key arguments:** heavy parallel fan-out, dynamic workflow
  generation from streaming events, sub-second event-driven flows.
  Honest about scope.
- **Pages:** 8–10.

---

## Volume totals (rough)

- Part I: 45–57 pages (3 chapters)
- Part II: 101–122 pages (5 chapters)
- Part III: 76–92 pages (4 chapters)
- Part IV: 79–96 pages (5 chapters)
- Part V: 96–117 pages (6 chapters)
- Part VI: 71–85 pages (4 chapters)
- Front matter, glossary, index: ~30 pages
- **Total: 498–599 pages.**  Tracks the original 500–600 estimate.
