# Chapter 2 — Why Most CI YAML Grows Monstrously

> *Every CI YAML file that reaches six hundred lines was once a
> fifty-line file.  No one ever set out to write the six-hundred-line
> version.  The growth is the symptom of a category mistake at the
> first commit.*

This chapter is about how a CI configuration file gets out of hand.
The argument is not that vendor YAML is bad — GitLab CI, GitHub
Actions, and Jenkins have all done genuinely good engineering on the
expressiveness of their configuration formats.  The argument is that
vendor YAML is asked to do six things at once, and the absence of
typed structure for any of them is what makes the file unreadable
after a year.

If you understand the six failure modes in this chapter, you understand
why the rest of the volume insists on naming and typing the things
those failure modes blur together.

## Failure mode 1: the environment-variable accretion

The first symptom of trouble is `export`.

A team adds `BUILD_VERSION` to set the version stamp on a build.  The
next quarter, someone adds `STAGING_HOST` for a deployment step.  Six
months later there is `DB_CONN_STRING`, then `RETRY_LIMIT`, then
`USE_FAST_PATH`, then `LANDSCAPE_OVERRIDE_FILE`.  Each export was
added by a different engineer; each was added because at the time it
was the path of least resistance.

The accreted environment-variable set has four problems:

1. **No schema.**  No one can list the full set of variables the
   pipeline reads without grepping the entire codebase.  When a
   variable is removed, no one notices for months.
2. **No types.**  Every value is a string.  `RETRY_LIMIT=3` and
   `RETRY_LIMIT=three` and `RETRY_LIMIT=` and `RETRY_LIMIT=`
   (trailing whitespace) all behave differently and none of them fail
   loudly.
3. **No namespacing.**  `HOST` could mean the build host or the
   target host or the database host.  When two scripts export `HOST`
   for different meanings, the second one wins; whichever script ran
   second gets to define what `HOST` means for the rest of the
   pipeline.
4. **No provenance.**  When a pipeline behaves differently than
   expected and the engineer asks "why did `RETRY_LIMIT` end up at
   1?", the answer requires reading every step that ran before the
   failing one to find which one re-exported it.  There is no audit
   trail.

The deeper problem is that **environment variables are the wrong
data structure for parameterising orchestrations.**  They are a
process-isolation mechanism repurposed as a configuration system, and
they fail at configuration in all the ways above.  Chapter 7 makes
the affirmative case for runtime configurations as the right data
structure.

## Failure mode 2: the conditional thicket

The second symptom is `if`.

A pipeline that needs to behave one way in CI and another way locally
grows a conditional.  Then the staging environment gets its own
conditional.  Then the nightly long-run gets one.  After a year:

```yaml
script:
  - if [ "$CI_PIPELINE_SOURCE" = "merge_request_event" ]; then
      ./scripts/run-fast.sh;
    elif [ "$CI_PIPELINE_SOURCE" = "schedule" ]; then
      if [ "$NIGHTLY_FULL_SWEEP" = "1" ]; then
        ./scripts/run-everything.sh;
      else
        ./scripts/run-medium.sh;
      fi;
    elif [ "$CI_COMMIT_REF_NAME" = "main" ]; then
      ./scripts/run-staging.sh;
    else
      ./scripts/run-fast.sh;
    fi
```

This is a composite of four orthogonal concerns smeared into one
shell expression:

- *Which test scope?* (fast / medium / full)
- *Which target environment?* (local / CI / staging)
- *Which trigger?* (MR / push / schedule)
- *Which special-mode override?* (the `NIGHTLY_FULL_SWEEP` flag)

Each concern wants its own typed configuration.  The conditional
thicket exists because the team had no place to put four named
profiles, so they hid the profile selection inside a shell
conditional.  Chapter 14 names this shape and proposes the discipline.

## Failure mode 3: the inlined recipe

The third symptom is forty lines of `aws ec2 ...` in a deploy step.

This is a pipeline that has absorbed provisioning logic.  The
pipeline does not consume a *cloud VM* by name; it contains the
recipe for creating one.  The recipe is hand-rolled, idempotent only
on best-effort, and copied between three pipelines that all want a
similar VM.  When the cloud SDK ships a new minor release, three
pipelines break.

The inlined recipe is the orchestration layer absorbing work that
belongs in a Specification (chapter 4).  A well-built orchestration
*declares* the resources it consumes — "I need an EC2 instance
matching specification `runner-medium`" — and a separate provisioning
layer turns that declaration into an actual instance.  When the
cloud SDK changes, the provisioning layer changes; the pipelines do
not.

## Failure mode 4: the credential leak through env

The fourth symptom is `DB_PASSWORD: $CI_DB_PASSWORD` in the YAML.

Vendor CI systems all provide secrets management — masked variables
that don't print to logs.  But the moment a secret is exposed as an
environment variable, three things go wrong at once:

1. The secret is now in the process environment of every step that
   runs after it is exported, including steps that should not have it.
2. The secret can be accidentally logged when a child process echoes
   `env` for debugging — the masking applies to the CI system's log
   capture but not to the inner subprocess.
3. The secret has no scope, no rotation, no audit.  It is a string
   in the bag of strings.

A typed credential (chapter 9) has an identity, a type, a scope, and
a lifetime.  It is fetched from a credential store at the moment of
use and not earlier.  It is never an environment variable; it is a
typed object.  The vendor's masked-variable feature is the *first*
line of defence, not the *only* line of defence.

## Failure mode 5: the host name in the YAML

The fifth symptom is a hardcoded hostname or IP address.

A pipeline that is going to run a test against a staging database
contains `STAGING_DB_HOST: db-staging-east-1.internal`.  Six months
later there is also `STAGING_DB_HOST_FAILOVER`,
`STAGING_DB_HOST_AS_OF_2026Q1`, and a comment explaining which is
current.  Eventually someone copies the pipeline to test against
prod-readonly and forgets to update the host; the test runs against
staging by accident, no one notices, and the test was meaningless for
three months.

The problem is the absence of a Landscape (chapter 5) and a Data
Profile (chapter 8).  A landscape names hosts; a data profile names
*the database* without saying which host it lives on; a runtime
configuration says which landscape and which profile are in scope for
this run.  The pipeline references `staging-db` (a profile name).
The profile is resolved at runtime to the current host.  When the
host changes, the profile changes; the pipeline does not.

## Failure mode 6: the topology buried in step ordering

The sixth and most subtle symptom is correct step order with no
explanation.

A pipeline runs `prepare-db` before `start-app` because the app
expects the schema; it runs `start-app` before `run-smoke-tests`
because the smoke tests hit the app; it runs `run-smoke-tests` before
`tear-down` because tear-down needs the smoke results.  The
*topology* of the system — the dependencies between components — is
encoded as the order of steps in a flat list.  When someone wants to
add a step, they must reason about the implicit graph from a flat
list.  When the graph has fifteen edges, the list reordering becomes
error-prone.

A Topology (chapter 6) makes the graph explicit.  Steps declare which
components they need to be live; the engine sequences them.  The flat
list reappears as an *output* of topology resolution rather than as
the *input* it had to be.

## The pattern across all six

Each failure mode is the absence of a typed configuration category:

| Failure mode | Missing category |
|---|---|
| 1 — env-var accretion | All six (the env vars are the symptom) |
| 2 — conditional thicket | Runtimes (chapter 7) |
| 3 — inlined recipe | Specifications (chapter 4) |
| 4 — credential leak | Credentials (chapters 9–12) |
| 5 — host name in YAML | Landscapes (chapter 5) + Data Profiles (chapter 8) |
| 6 — topology in ordering | Topologies (chapter 6) |

The six categories are not arbitrary.  Each one is the answer to a
recurring failure mode of CI YAML.  The next part of the volume
introduces them in turn.

## Why this matters even if you keep using vendor YAML

A reader who finishes Volume 3 may not adopt `codecraftai-taskplus` — and
that's fine.  The discipline does not depend on the toolkit.  The
discipline says:

- Express each of the six categories as a typed file.
- Reference those files by name in your pipeline.
- Resolve the references at runtime, not at YAML-write time.
- Compose configurations through overlay, not through string
  substitution.

A team that does this in GitLab YAML has a shorter, clearer YAML.  A
team that does this with a tool-neutral workflow runner like
`taskplus` has a portable workflow.  Either is better than the
six-hundred-line file.  The book makes the case for the tool-neutral
path because portability has compound benefits — local-dev parity,
CI-vendor migration, debug reproducibility — but the discipline
itself is the larger lesson.

## Summary

- Six common failure modes turn fifty-line CI YAML into six-hundred-line
  CI YAML.  Each failure mode is the absence of a specific typed
  configuration category.
- Environment variables are the wrong data structure for orchestration
  parameterisation.  They lack schema, types, namespacing, and
  provenance.
- Conditional thickets, inlined recipes, leaked credentials,
  hardcoded hostnames, and implicit topologies are all instances of
  the same problem: a missing typed configuration.
- The next part of the volume names the six categories and shows how
  each one closes a failure mode.
