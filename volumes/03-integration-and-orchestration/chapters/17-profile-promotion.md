# Chapter 17 — Profile Promotion

> *Your local-dev profile is not a draft of your staging profile.
> Your local-dev profile **is** your staging profile with two
> overlay files added.*

This is the last chapter of Part IV.  Chapters 13 through 16 built
the profile discipline: the four-environment ladder (chapter 13),
the profile mechanism (chapter 14), the local-dev migration
(chapter 15), the capture-and-replay primitive (chapter 16).
Each of those chapters treated the profiles as static — fixed
configurations that already exist, applied to a run.

This chapter is about the *dynamics*: how profiles change over
time, how additions propagate from one environment to another, and
how the discipline keeps four environments in sync without
coordinated rewrites.

## The promotion direction

A common misreading of the four-environment ladder is that
configurations *travel from local-dev up to production*.  That
reading is correct in spirit and slightly wrong in mechanics.
What actually travels is *changes*; the profile *files themselves*
sit in a fixed inheritance hierarchy.

The hierarchy from chapter 14, drawn explicitly:

```
                base
                 │
       ┌─────────┼──────────────────┬──────────────────┐
       ▼         ▼                  ▼                  ▼
   local-dev  integration-ci    staging-east       prod
                                       │
                                       ▼
                              staging-east-canary
```

`base` is the root.  Each environment profile extends `base` and
picks its own selections.  Some environments (`staging-east-canary`)
extend other environments rather than `base` directly.  The
hierarchy is acyclic, finite, and stable; profile files don't
move around.

What *moves* is changes:

- A new knob added to `base` becomes available in every profile
  that extends it.  This is **upward promotion** through the
  hierarchy.
- A new knob added to `local-dev` only appears in `local-dev`.
  Until and unless it is promoted to `base` (or to whichever
  shared ancestor the change should reach), it stays scoped.
  This is **non-promotion** — the change has been added to one
  profile but not migrated to its peers.
- An overlay added to `staging-east-canary` only appears in that
  one descendant profile.  The change is intentionally scoped
  narrow.  This is **scoped addition** — the change was promoted
  *to* a leaf profile, not *through* the hierarchy.

The discipline of this chapter is the conscious, reviewable
choice of which of these three moves to make every time a profile
changes.

## The promotion model

Three moves, each with a different level of impact and a different
review story.

### Promotion to `base`

The most impactful move.  A change to `base` reaches every
descendant profile — local-dev, integration-CI, staging-east,
staging-east-canary, prod, and any future profile.

Promotions to `base` are appropriate when the change represents a
*genuinely shared* concern that should apply everywhere:

- A new universal field on the runtime schema (e.g., adding a
  `tracing.sample_rate` field with a sane default).
- A new credential identifier the application code newly
  requires (e.g., adding `metrics-exporter` as a credential
  reference all environments must resolve).
- A new security-critical default (e.g., changing the `base`
  runtime to require TLS verification on by default).

When a change isn't *genuinely shared*, it doesn't belong in
`base`.  A new local-dev convenience knob is not a `base`
addition; it is a `local-dev` addition.

The review story for a `base` promotion is a single PR that
modifies one file (`profiles/base.yaml`) and that touches every
environment by reference.  The PR's reviewers know they are
making a multi-environment change; the diff is small and
focused; the impact is broad.

### Scoped addition

A change to one specific profile.  The most common move.

A new field on the local-dev runtime — say, lowering the timeout
for a particular slow test — affects only local-dev.  The PR
modifies one file (`config/runtime/runtime-local-dev.yaml`); no
other environment is affected.

Scoped additions are how the *day-to-day* of profile maintenance
works.  Most changes belong here.  When in doubt, a change is a
scoped addition; promote it later if it turns out to be more
broadly useful.

### Override

A change that *contradicts* an inherited value.  Inheritance via
`extends:` cascades values from ancestors down; a child profile
can shadow an inherited value with a more specific one.  Override
is how the descendant says *"this thing in my parent does not
apply here."*

Common overrides:

- `prod` overrides `base.runtime.logging.levels.console` from
  `INFO` to `WARN` to reduce log volume.
- `staging-east-canary` overrides `staging-east.runtime.parallelism.max_workers`
  from 8 to 2 because canary runs are deliberately slower.
- `prod-readonly` overrides `staging.runtime.timeouts.default_seconds`
  from 30 to 5 because production calls must fail fast.

Overrides are *additive* in the file structure (a new field in
the descendant) but *subtractive* in semantics (they negate an
inherited behaviour).  The review story is the same as a scoped
addition: one file changes, the diff is local, the impact is
local.

## The "automatically promoted" property

The discipline has an important compositional property: **a new
knob added to `base` is automatically available in every
descendant**, with the descendant's runtime configuration loader
picking up the new field on next process start.  No coordinated
edit of every environment profile is needed.

Two examples make this concrete.

### Example: adding a tracing field

The team decides to enable distributed tracing across the
platform.  They add a `tracing` section to the runtime schema.
The change goes into `base`:

```yaml
# config/runtime/runtime-base.yaml — modified
breakpoints: []
debugger: none
logging:
    levels:
        console: INFO
        logfile: DEBUG
parallelism:
    max_workers: 4
timeouts:
    default_seconds: 30

# NEW
tracing:
    enabled: true
    sample_rate: 0.1
    exporter: otlp
    exporter_endpoint: ${OTLP_ENDPOINT:-http://localhost:4318}
version: 1.0.0
```

After the merge, the next CI run loads `runtime-integration-ci`
which (via the chapter-14 `MergeMap` composition) inherits from
`runtime-base`.  The `tracing` section appears in the resolved
runtime configuration.  Tracing is on for CI.

The next staging run inherits the same way.  Tracing is on for
staging.

The next local-dev run inherits the same way.  Tracing is on
locally (with the local fallback OTLP endpoint).

No environment-specific edits.  No "did anyone update the
staging config?" question.  The promotion was a single file
change to `base`, and the cascade propagated automatically.

### Example: adding a per-environment override

Three months later, the team realises that local-dev should not
have tracing on by default — it adds noise to the logs and slows
down breakpoint debugging.  Local-dev's runtime profile gains a
single line:

```yaml
# config/runtime/runtime-local-dev.yaml — modified
# (inherits everything from base)
tracing:
    enabled: false      # override base; local-dev runs without
                        # tracing for inner-loop speed
```

The override is *additive in the file* (one new line); *negating
in semantics* (it disables the inherited tracing).  Other
environments are unchanged.  Local-dev now runs without tracing;
CI / staging / prod run with tracing.

The team did not have to:

- Remove the `tracing` field from `base`.
- Edit every environment's runtime to set `enabled: true`
  individually.
- Branch on environment in application code.

The two changes — turn tracing on globally, turn it off in
local-dev specifically — are reviewable PRs with focused diffs.
The cumulative effect is the desired behaviour.

## The audit story

A profile change has the same audit characteristics as any other
configuration change in the discipline:

- **Every profile file is in version control.**  Profile changes
  are git diffs.  The diff is reviewable in the same way every
  other code change is.
- **Every profile change is small.**  A `base` promotion changes
  one file.  A scoped addition changes one file.  An override
  changes one file.  None of the moves involve coordinated
  cross-file edits — which is the structural reason changes are
  reviewable in the first place.
- **Every active profile is captured at runtime.**  The chapter-16
  capture primitive emits a snapshot of the resolved profile,
  including which file each value came from.  When a question
  arises *months later* — "what was the staging-east timeout six
  months ago?" — the answer is in a captured snapshot, not in
  someone's memory.

The discipline turns cross-environment configuration into a
managed artifact.  Configuration drift becomes detectable
(captures from different environments diff; changes between
captures are visible).  Configuration regressions become
attributable (each value's origin file is traceable through the
inheritance chain).  Cross-environment surprises become rare
(the cascade is mechanical; surprises require explicit
overrides, which themselves are diffs in version control).

## Worked example: the promotion of a new field

Trace one change through the four environments.  The change is
small: add a per-orchestration `tracing.sample_rate` knob that
local-dev wants at `1.0` (sample everything for debugging) and
production wants at `0.01` (sample sparsely to control cost).

### Day 1 — propose

A senior engineer adds the `tracing` section to the runtime
schema, with a default for `sample_rate`:

```yaml
# config/runtime/runtime-base.yaml — PR #1
tracing:
    enabled: true
    sample_rate: 0.1            # default for non-prod
    exporter: otlp
    exporter_endpoint: ${OTLP_ENDPOINT:-http://localhost:4318}
```

PR #1 is reviewed by the SRE team (they own the tracing
infrastructure).  The PR description names which environments
inherit the default (every environment by default).  Reviewers
confirm the default is sane and the merge happens.

### Day 2 — local-dev preference

A developer wants tracing at `1.0` locally for an active
debugging session.  They add an override:

```yaml
# config/runtime/runtime-local-dev.yaml — PR #2 (developer's branch)
tracing:
    sample_rate: 1.0    # full sample for local debugging
```

PR #2 is reviewed by the developer's team.  Diff is one line.
Impact is local-dev only.  Merged.

### Day 3 — production preference

The SRE team learns that `0.1` sample rate is producing too much
trace volume in production.  They want production at `0.01`.
PR:

```yaml
# config/runtime/runtime-prod.yaml — PR #3
tracing:
    sample_rate: 0.01   # cost control in prod
```

PR #3 is reviewed by the SRE team.  Diff is one line.  Impact is
production only.  Merged.

### Day 4 — staging keeps the default

Staging inherits the `base` default of `0.1`.  No PR is required
for staging; the default applies automatically.  The staging
environment now runs at `sample_rate: 0.1` — *because that is
what the inheritance produced*, not because anyone explicitly set
it for staging.

### Day 100 — auditing the cascade

Three months later, an SRE wants to understand why staging
produces ten times the trace volume of production.  They run:

```bash
ccai profile capture --output staging-now.json
ccai profile capture --output prod-now.json
diff <(jq '.configuration_snapshots.runtime.tracing' staging-now.json) \
     <(jq '.configuration_snapshots.runtime.tracing' prod-now.json)
```

The diff shows `sample_rate: 0.1` in staging vs `sample_rate: 0.01`
in prod.  The SRE walks the inheritance: staging inherits from
`base`; `base` declares `0.1`; staging does not override.  Prod
overrides at `0.01`.  The cascade is documented in the capture
plus the original PRs.

The investigation took five minutes.  Without the discipline, the
same investigation would have involved finding which engineer
last touched the tracing config (none of them), reading three
different configuration sources (CI YAML, environment variables,
hardcoded defaults), and reconstructing the answer from
incomplete information.

## Anti-patterns: how the discipline gets broken

Three patterns produce the same kinds of drift the discipline is
supposed to prevent.  Each is recognisable; each is the result of
a small short-cut that compounds.

### Profile fork

A change is supposed to apply to multiple environments, but the
engineer copies the value into each profile rather than promoting
it to a shared ancestor.

Symptom: `runtime-staging-east.yaml`, `runtime-staging-west.yaml`,
and `runtime-prod.yaml` all have the same value for some field;
the value is different from `base`.

Why it happens: the engineer is making a quick fix and promoting
to `base` would require additional review.  Copying into three
files is "faster."

How to detect: a periodic script that walks the runtime files
across environments, finds fields with identical values across
multiple environments, and flags them as candidates for `base`
promotion.

How to fix: when the script flags a duplicate, *promote it to
`base`* (or to the nearest shared ancestor), and remove the
duplicates.  The drift is undone; the discipline is restored.

### Divergent rewrite

A profile is supposed to inherit from `base` but was authored as
a complete copy with `extends:` pointing at `base` for *form's
sake* and no actual inherited fields.

Symptom: a profile YAML lists every category selection
explicitly even when several of the selections match `base`'s
defaults.

Why it happens: the engineer writing the new profile used
another profile (`staging-east`) as a starting point and didn't
prune the inherited values when they happened to match.

How to detect: a `ccai profile diff <child> base` that shows
zero differences for a particular category means the child is
explicitly setting what it would inherit anyway.

How to fix: drop the redundant explicit values from the child.
The behaviour doesn't change; the file gets shorter; the
inheritance becomes truthful.

### Copy-paste promotion

A change is "promoted" by copying the modified file from one
environment to another, rather than by editing the source-of-
truth profile and letting inheritance cascade.

Symptom: a `git log` on `runtime-base.yaml` shows no recent
activity, but `git log` on `runtime-staging-east.yaml`,
`runtime-staging-west.yaml`, and `runtime-prod.yaml` shows the
same commit landing on each in quick succession.

Why it happens: the engineer doesn't trust inheritance to do the
right thing, so they apply the change manually to each file.

How to detect: a CI check that flags PRs touching multiple
environment files with the same diff and suggests promoting to
`base` instead.

How to fix: revert the per-environment changes; apply the change
to `base`; let inheritance carry it.  This is the discipline at
its most useful — the inheritance is *exactly* the mechanism the
copy-paste avoided.

## Where the promotion model doesn't apply

Three boundaries to keep clear.

### Categorical breaks

Some changes are not promotable.  When prod uses Snowflake for
its data warehouse but staging uses BigQuery, no amount of
inheritance can pretend they are the same.  The data-profile
files for each environment specify different categories
(`snowflake` in one, `bigquery` in the other), and the
application code has to handle both.

The discipline accommodates categorical breaks by keeping
*irreconcilable differences out of inheritance*.  Each
environment selects its own data profile; no shared ancestor
declares a category that doesn't apply universally.

### Fork-then-merge changes

Some changes start scoped and become broad.  A knob added to
local-dev as a debug helper turns out to be useful in CI.  The
promotion path:

1. Begin life in `local-dev` (scoped addition).
2. Confirm the change is useful in CI by setting it in
   `integration-ci` (scoped addition, second profile).
3. *Now* the change has two homes; promote to `base`; remove
   from both leaf profiles.

Step 3 is the key one.  A change that lives in two profiles is
a candidate for `base` promotion.  Skipping the promotion leaves
the team in the profile-fork anti-pattern.

### Permission-gated changes

Some changes affect production and require a different review
board.  The discipline accommodates this by *file ownership*: a
GitHub `CODEOWNERS` file (or its equivalent) restricts who can
approve changes to `runtime-prod.yaml` even when the change
landed in `base` and merely cascaded.

The reviewers of a `base` change are responsible for understanding
the cascade.  When the cascade reaches production with consequences
that need stricter review, the discipline asks the reviewer to
also enlist the production-config owners.  This is human process
on top of mechanical inheritance; the inheritance does not enforce
the process by itself.

## Summary — and end of Part IV

- A profile inheritance hierarchy is a stable structure where
  changes propagate predictably.  Three moves available: **promote
  to `base`** (cascade everywhere), **scoped addition** (one
  profile only), **override** (negate an inherited value in a
  descendant).
- A `base` promotion is the most impactful move and the rarest;
  scoped additions and overrides are the daily currency of profile
  maintenance.
- The "automatically promoted" property — a new field in `base`
  is available in every descendant on next run — eliminates
  coordinated cross-environment edits.  Adding a knob is one
  PR; the cascade is mechanical.
- Every profile change is in version control; every change is
  small; every active profile is captured at runtime.  Drift is
  detectable; regressions are attributable; surprises are rare.
- A worked example traced one tracing-config knob through four
  environments: PR to `base` adds the field, PR to local-dev
  overrides for full-sample debugging, PR to prod overrides for
  cost control, staging inherits the `base` default with no PR.
  Three months later the cascade is auditable from captured
  snapshots.
- Three anti-patterns — **profile fork** (duplicating across
  environments instead of promoting to base), **divergent rewrite**
  (copying instead of inheriting), **copy-paste promotion**
  (manual cascade instead of letting inheritance do it) — are the
  recurring ways the discipline gets broken.  Each has a
  detection signature and a mechanical fix.
- Three categories of change *don't* fit promotion: **categorical
  breaks** (different vendors per environment), **fork-then-merge
  changes** (knobs that start scoped and grow into broad), and
  **permission-gated changes** (where human review process layers
  on top of mechanical inheritance).

This is the last chapter of Part IV.  The volume has now built
the discipline (Parts I–III) and developed the environment-
spanning machinery (Part IV).  The next part — Part V — is the
implementation: `codecraftai-taskplus` as the workflow engine that
references the configurations the discipline has produced, by
name from a portable workflow YAML.  Where Parts II–IV asked
*what* the configurations should be and *how* they should
compose, Part V is *what runs them*.
