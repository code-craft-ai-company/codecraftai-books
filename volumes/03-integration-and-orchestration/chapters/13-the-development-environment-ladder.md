# Chapter 13 — The Development-Environment Ladder

> *There are four environments a piece of code visits in its
> lifetime: a developer's laptop, the CI integration pipeline, a
> staging deployment, and prod-readonly debugging.  Most teams
> treat these as four codebases that happen to overlap.  The
> discipline of this part is to treat them as four profiles of the
> same codebase.*

Welcome to Part IV.  Parts I, II, and III named six configuration
categories and showed how each one expresses a slice of an
orchestration's parameterisation as typed data.  Part III ended
with the credential-specific application of an environment-spanning
pattern: same file, same identifiers, different backends.  This
part generalises that pattern to *every* configuration category and
proposes the discipline that lets a single codebase scale from a
developer's laptop to production debugging without rewrites at any
boundary.

The first chapter of the part — this one — establishes the ladder
itself.  The four chapters that follow propose the machinery
(chapter 14 on profile composition), the local-dev story (chapter
15), the CI-failure-reproduction story (chapter 16), and the
profile-promotion story (chapter 17).  Together the five chapters
make the case that "works on my machine" and "works in CI but not
locally" are *symptoms of a missing typed configuration*, and that
the cure is the profile discipline.

## A note on shipped tooling

An earlier research pass that grounded this volume found that the
portfolio has solid building blocks for the profile discipline —
`override_variable_value`, `MergeMap`, `AppVarSet`, `ContextPaths`,
`get_variable_value`, the existing five `AM_CONFIG_*_NAMES`
descriptors — but does not yet ship the *user-facing tooling* a
profile system needs (no `automojo profile show` CLI, no
`profile diff` command, no file-level `extends:`/`overlay:` syntax,
no captured-profile-replay mechanism).

Part IV therefore presents the profile discipline as a *recipe the
reader implements* using the shipped primitives as the
substrate.  Each chapter calls out which pieces are already in
place and which the reader will need to build.  By the end of Part
IV the reader has a complete profile system in their portfolio,
all of it composable with the rest of the platform.

## The four environments

A piece of automation visits four environments in its lifetime.
The discipline is built around treating each as a *profile* — a
named bundle of category selections — rather than as a separate
codebase.

### local-dev

The developer's workstation.  A single host running a small set
of mock services.  Fast feedback, low cardinality, generous
timeouts (because the developer wants to set breakpoints), and
verbose logging.

What's distinctive:

- **One participant in the apod** (the developer's own machine).
- **Mocks or local containers** for the infrastructure services.
- **Filesystem-backed credentials** or interactive prompts.
- **Verbose logging**, often `DEBUG`-level on every component.
- **Small property-test sweeps** (`TESTBASE_MAX_EXAMPLES_CAP=10`).

The local-dev profile exists to support the inner loop: write
code, run a single test, see the output in seconds, fix the bug.
Anything that introduces friction at this layer slows the team
down on every change.

### integration-CI

The CI pipeline running on every merge request.  Multiple
participants in the apod (a runner pool); real or
realistic-stand-in services; medium cardinality; tight timeouts;
structured logging routed to a central system.

What's distinctive:

- **Several participants in the apod** (the runner cluster).
- **Real staging services** (or carefully isolated CI-only
  instances of them).
- **Vault or cloud-secrets-manager** for credentials.
- **Structured logging** routed to the team's log aggregation.
- **Medium property-test sweeps** (`max_examples=10` for fast
  feedback on every MR; full sweep on nightly runs).

The integration-CI profile is the *gate*: code that reaches
staging has passed integration-CI.  The profile's job is to be
faithful enough to staging that passing integration-CI is
predictive of staging working, while remaining fast enough to fit
in a per-MR feedback budget.

### staging

A pre-production environment that runs the same orchestration
against real services.  Larger apods, full cardinality, the same
tight timeouts as production, full structured logging.

What's distinctive:

- **Production-shaped apods**, often the same number of runners
  as production.
- **Real services**, often the same infrastructure as production
  but separated by namespace, account, or region.
- **The same Vault** as production (or a tightly-controlled
  staging Vault), with audit logging always on.
- **Full property-test sweeps** (no `TESTBASE_MAX_EXAMPLES_CAP`,
  multi-hour runs).

The staging profile catches the issues integration-CI's medium
cardinality misses — the `max_examples=100` property test that
finds the once-per-thousand-cases edge case the CI run misses; the
soak test that exposes a memory leak the per-MR feedback can't.

### prod-readonly

An on-call engineer reading production state for incident
debugging.  No write access.  Audit-mandatory.  Time-bound access.

What's distinctive:

- **No apod participants in the conventional sense**.  The
  on-call engineer's workstation is the only "runner."
- **Production services**, accessed read-only.
- **Short-lived Vault tokens** issued by the on-call workflow with
  the operator's identity and the incident ticket as part of the
  fetch context.
- **Mandatory audit** — every credential lookup emits an event
  with operator and ticket attribution.
- **Aggressive timeouts** so a misbehaving production query never
  blocks beyond a small bound.

The prod-readonly profile exists for *one* purpose: to let an
engineer debug a production incident with the same tools they use
in every other environment, without giving them the permissions
to make the incident worse.

## The codebase-treatment approach: where it goes wrong

The natural response to four environments is to introduce *four
codebases* that happen to overlap.  Concretely, this looks like:

- A `tests/local/` directory and a `tests/integration/` directory
  with parallel test files.
- Setup scripts named `setup-local.sh`, `setup-ci.sh`, `setup-staging.sh`.
- Three or four `docker-compose.*.yaml` files with mostly the same
  services.
- A `conftest.py` (if pytest) full of `if os.environ.get("CI")`
  branches that change fixture behaviour.

The branches start small.  An early one looks like this:

```python
def get_database_host():
    if os.environ.get("CI"):
        return "db-ci.internal"
    return "localhost"
```

After a year, the function looks like this:

```python
def get_database_host():
    if os.environ.get("CI"):
        if os.environ.get("CI_PIPELINE_SOURCE") == "schedule":
            return os.environ.get("NIGHTLY_DB_HOST", "db-nightly.internal")
        return os.environ.get("CI_DB_HOST", "db-ci.internal")
    if os.environ.get("STAGING_RUN"):
        return os.environ.get("STAGING_DB_HOST", "db-staging.internal")
    if os.environ.get("PROD_DEBUG"):
        # readonly only; refuse if not in incident mode
        if os.environ.get("INCIDENT_TICKET") is None:
            raise RuntimeError("PROD_DEBUG without INCIDENT_TICKET")
        return os.environ.get("PROD_RO_DB_HOST", "db-prod-ro.internal")
    return os.environ.get("LOCAL_DB_HOST", "localhost")
```

This is the chapter 2 conditional-thicket failure mode in fresh
costume.  Four orthogonal concerns — *which environment am I in*,
*which sub-mode of that environment*, *which database does this
environment have*, *what guardrails apply* — smeared into one
shell expression.

The cost compounds.  Three concrete failure modes:

### Test code becomes environment-aware

Tests that should describe behaviour now describe behaviour *and*
environment routing.  A test that wants to assert "the API
responds with HTTP 200 to a `GET /healthz`" needs to first decide
which API to talk to, which depends on which environment it's in,
which depends on the env-var soup.  The test grows a setup section
that's longer than the test itself.

### Setup scripts diverge

`setup-local.sh` and `setup-ci.sh` start as twins.  After six
months, `setup-ci.sh` has acquired three extra steps for things
that work-around CI-specific quirks; `setup-local.sh` has acquired
two different extra steps for developer-laptop quirks.  The
intersection of the two is shrinking, but neither team owns the
job of keeping them in sync.

### `if 'CI' in os.environ` becomes uncatchable

Every legitimate use of the env-var-as-flag pattern paves the way
for the *next* less-legitimate use.  Eventually, business logic
branches on `CI`.  A test discovers that production has a quirk CI
doesn't and adds an `if not os.environ.get("CI"):` branch to
"work around" it.  The branch survives for three years; the
production quirk gets fixed; the branch never gets removed; the
codebase now has dead code that still subtly diverges between
environments.

The deeper issue is that **environment differences are
unbounded under the codebase-treatment approach**.  Every
divergence is paid for once with a new branch and forever with the
maintenance.  The team's velocity on cross-environment changes
slows, then stops.

## The profile-treatment approach: what it asks of the code

The discipline of Part IV is the inverse: treat the four
environments as *profiles*, where a profile is a named bundle of
configuration selections.  The code does *not* know which profile
is active.  The code reads typed configurations through the
runtime API; the runtime configurations are different between
profiles; the code is identical.

The same `get_database_host` from above becomes:

```python
def get_database_host():
    profile = ThisApplicationSingleton().get_data_profile("primary-db")
    return profile.host
```

Two layers in this rewrite:

1. **The application code asks the runtime** for the data profile
   named `primary-db`.  It does not ask "which environment am I
   in?"
2. **The data profile** named `primary-db` exists in every
   environment; its content (the host, the port, the credential
   reference) differs.  The selection of which `primary-db`
   profile applies to this run was made when the profile was
   loaded — long before this code ran.

Three properties of the rewrite worth naming:

- **The code has no environment branches.**  The
  conditional-thicket failure mode cannot reach this function;
  there is no place in the function where an environment check
  could be added that the discipline wouldn't immediately reject.
- **Adding a fifth environment is a profile addition.**  When the
  team stands up `staging-east-canary` as a fifth deployment, the
  data profile gains a new entry; the code is unchanged.
- **Removing an environment is a profile deletion.**  When `staging`
  is decommissioned, the data profile entries with that scope are
  removed; the code is unchanged.

The discipline asks the code to be *passive* with respect to
environment.  The code reads typed configurations.  The
configurations carry the environment-specific differences.  The
plumbing — chapter 7's `AM_*_NAMES` family — selects which
configurations are loaded.

## The lifecycle of a single test

Trace a single test through the four environments.  The test is
modest: *create a user via the API, fetch the user back, assert
that the response matches the input*.

```python
# tests/test_user_lifecycle.py
def test_user_create_and_fetch():
    """
        Create a user; fetch it back; assert the response matches
        the input.
    """
    api = ThisApplicationSingleton().get_data_profile("staging-api")
    cred = CredentialManagerSingleton().lookup_credential(api.credential)

    payload = {"name": "alice", "email": "alice@example.com"}

    create_response = http_post(
        url=f"{api.endpoint}/users",
        headers={"Authorization": f"Bearer {cred.token}"},
        json=payload,
    )
    verification.assert_equal(create_response.status_code, 201)

    user_id = create_response.json()["id"]
    fetch_response = http_get(
        url=f"{api.endpoint}/users/{user_id}",
        headers={"Authorization": f"Bearer {cred.token}"},
    )
    verification.assert_equal(fetch_response.status_code, 200)
    verification.assert_equal(fetch_response.json()["name"], "alice")
    verification.assert_equal(fetch_response.json()["email"], "alice@example.com")

    return
```

The test's body never names an environment.  It asks the runtime
for `staging-api` (a Data Profile name) and the credential the
profile references; it makes the calls; it asserts the response.
That's it.

### Day 1 — local-dev

The developer is writing the test.  They run it on their laptop.
The relevant configurations:

```yaml
# config/dataprofiles/dataprofiles-local-dev.yaml
dataprofiles:
    - identifier: staging-api      # same identifier in every env
      category: web/api
      endpoint: http://localhost:8080
      credential: local-api-token

# config/credentials/credentials.yaml (already shown in chapter 12)
- identifier: local-api-token
  categories: [api-token, environment:local-dev]
  token: ${PROMPT:Local API token}

# config/runtime/runtime-local-dev.yaml
logging:
    levels:
        console: DEBUG
timeouts:
    default_seconds: 300        # generous; the developer wants to set breakpoints
property_test_caps:
    max_examples: 5             # fast feedback
```

The developer runs:

```bash
AM_PROFILE=local-dev pytest tests/test_user_lifecycle.py -v
```

The orchestration's startup reads `AM_PROFILE=local-dev`, builds
the resolver with the local-dev backend roster, loads
`dataprofiles-local-dev.yaml`, `credentials.yaml`, and
`runtime-local-dev.yaml`, and runs the test.  The test calls
`http://localhost:8080/users`, gets the prompted local API token,
and passes.

### Day 7 — integration-CI

The developer pushes the branch.  The CI pipeline picks it up.
The pipeline's vendor YAML sets `AM_PROFILE=integration-ci`:

```yaml
# .gitlab-ci.yml
test:
  variables:
    AM_PROFILE: "integration-ci"
  script:
    - automojo testbase test run tests/test_user_lifecycle.py
```

The relevant configurations:

```yaml
# config/dataprofiles/dataprofiles-integration-ci.yaml
dataprofiles:
    - identifier: staging-api
      category: web/api
      endpoint: https://api-ci.staging.internal
      credential: ci-api-token

# config/credentials/credentials.yaml — already includes ci-api-token

# config/runtime/runtime-integration-ci.yaml
logging:
    levels:
        console: INFO
timeouts:
    default_seconds: 30
property_test_caps:
    max_examples: 10
```

The same test runs.  Same code.  Different `staging-api` —
pointing at the CI staging API.  Different `runtime` — tighter
timeouts, INFO-level logging.  Different credential — the CI Vault
provides the API token.

### Day 14 — staging

Once the change merges, the nightly staging run picks it up.
`AM_PROFILE=staging-east`:

```yaml
# config/dataprofiles/dataprofiles-staging-east.yaml
dataprofiles:
    - identifier: staging-api
      category: web/api
      endpoint: https://api-staging-east.internal
      credential: staging-api-token

# config/runtime/runtime-staging-east.yaml
logging:
    levels:
        console: INFO
timeouts:
    default_seconds: 30
property_test_caps:
    max_examples: null    # unset; full sweep
```

The same test runs.  Different endpoint, different token, no
property-test cap (the nightly run sweeps every property at full
strength).  The test that passed in CI may now uncover an issue
the CI run's `max_examples=10` missed.

### Day 30 — prod-readonly

A production incident has occurred.  The on-call engineer wants
to confirm that the affected user record is in fact corrupted in
production.  They want to fetch the user — *not* create one, since
prod is read-only — using their on-call ticket as the authentication
context.

They cannot run the full `test_user_create_and_fetch` test
(production is read-only); they can run a *fetch-only* variant.
The discipline accommodates this with profile-keyed test selection,
which chapter 17 develops further.  For now, the relevant point:
the engineer types

```bash
AM_PROFILE=prod-readonly INCIDENT_TICKET=INC-12345 \
    automojo testbase test run tests/test_user_fetch_only.py
```

and the same machinery loads the prod-readonly profile, fetches a
short-lived audited Vault token tied to their identity and the
ticket, and runs the fetch-only test against production.  The
audit trail records every credential lookup with the operator and
the ticket.

### What changed and what didn't

In four environments the test went through, *the test's code was
identical*.  The configurations differed:

| | local-dev | integration-CI | staging-east | prod-readonly |
|---|---|---|---|---|
| `staging-api` endpoint | `http://localhost:8080` | `https://api-ci.staging.internal` | `https://api-staging-east.internal` | (production endpoint) |
| Credential identifier | `local-api-token` | `ci-api-token` | `staging-api-token` | `prod-api-token` |
| Credential backend | Prompt | Vault (CI) | Vault (Staging) | Vault (Prod, audited) |
| Logging level | DEBUG | INFO | INFO | INFO |
| Timeout default | 300s | 30s | 30s | 5s |
| Property-test cap | 5 | 10 | unset | (test is read-only) |

Six dimensions of difference; zero lines of conditional code.  The
environment differences are entirely expressed as content; the
code is plumbing-level passive.

This is the discipline.

## What the discipline asks of the codebase

Three concrete commitments make the discipline possible:

### Read configurations through the runtime

Every configuration value the code consumes — every host, every
port, every timeout, every log level, every credential, every
landscape entry — comes through the runtime API.  No
`os.environ.get(...)` in application code.  No hardcoded URLs.
No conditional branches keyed off environment markers.

The acceptable plumbing-level uses of `os.environ` from chapter 7
remain (configuration selectors, process identity,
external-system bootstrapping).  The application-level uses of
environment variables for *content* are gone.

### Reference configurations by name

Code says "give me the data profile named `staging-api`," not
"give me the data profile for the current environment."  The
runtime decides which `staging-api` is in scope based on the
active profile; the code never branches.

Stable identifiers across environments are the most important
property.  When `staging-api` means one thing in CI and a different
thing in staging-east, the discipline has been broken; the code
will start to grow conditionals to "fix" the inconsistency, and
the team is back to the codebase-treatment failure mode.

### Treat profile selection as plumbing

The selection of which profile is active is a plumbing concern —
one environment variable (`AM_PROFILE` or the
chapter-7 `AM_*_NAMES` family).  It is set by the CI pipeline, by
the developer's shell, by the on-call workflow.  Application code
*never* sets the profile.

The line between "plumbing reads configuration selectors from the
environment" and "application reads configuration content from the
environment" is the line the discipline draws.  A function that
reads `AM_PROFILE` to decide what to do is plumbing; application
code never has business doing that.

## The shipped substrate

The portfolio's existing machinery is the foundation the next four
chapters build on.  Each chapter cites the building blocks it
extends:

- **`override_variable_value`**
  ([`automojo-this-app/.../thisapplication.py`](../../../automojo-this-app/source/packages/automojo/thisapp/thisapplication.py))
  — the primitive for runtime variable overrides.  Chapter 14 uses
  it to implement profile composition.
- **`MergeMap`**
  ([`automojo-xmodules/.../mergemap.py`](../../../automojo-xmodules/source/packages/automojo/xmodules/xcollections/mergemap.py))
  — the primitive for layered configuration with overlay
  semantics.  Chapter 14 uses it to implement file-level profile
  composition.
- **`AppVarSet`**
  ([`automojo-this-app/.../appvarset.py`](../../../automojo-this-app/source/packages/automojo/thisapp/appvarset.py))
  — iterating the active descriptor set.  Chapter 14 uses it as
  the basis for the proposed `automojo profile show` CLI command.
- **`ContextPaths`**
  ([`automojo-this-app/.../contextpaths.py`](../../../automojo-this-app/source/packages/automojo/thisapp/contextpaths.py))
  — the well-known-path enumeration where active configurations
  live.  Chapter 14 proposes adding `ACTIVE_PROFILE` here.
- **`AM_*_NAMES`**
  ([`automojo-configuration/.../configurationvariables.py`](../../../automojo-configuration/source/packages/automojo/configuration/configurationvariables.py))
  — the existing five environment-variable selectors for the
  configuration categories.  Chapter 14 extends the family with
  the proposed `AM_PROFILE`.

Each chapter of Part IV stands up a piece of the discipline on
top of these primitives.  The reader can implement them
incrementally; nothing in chapter 14 depends on chapter 16, and
nothing in chapter 17 depends on chapter 15.

## The chapters that follow

The four chapters that complete Part IV develop specific aspects
of the discipline.

- **Chapter 14 — Configuration profiles as a discipline.**  The
  mechanics: a profile is a named bundle of category selections;
  profiles compose via overlay (`base.yaml` plus `staging.yaml`);
  selection happens at process start with `--profile staging`.
  Proposes the `ACTIVE_PROFILE` context path and the `automojo
  profile show` CLI.
- **Chapter 15 — The local-dev tax and how typed configs cancel it.**
  Walks through everything that makes a fresh checkout
  unusable today (`.env.local`, `direnv`, `make setup`, README's
  "you might also need to..." section) and shows each piece's
  migration into a typed configuration.
- **Chapter 16 — Reproducing CI failures locally.**  The replay
  primitive: capture the resolved profile from the failing CI run
  (with secrets redacted), re-run the same job locally with the
  captured profile, debug in the IDE.  Builds on the audit trail
  from chapter 11 and the resolver from chapter 10.
- **Chapter 17 — Profile promotion.**  How `local-dev` becomes
  `staging-east-canary` becomes `staging-east` becomes `prod`,
  through additive overlay rather than rewrite.  The promotion
  model and the audit story for cross-environment changes.

## Summary

- A piece of automation visits four environments — **local-dev,
  integration-CI, staging, prod-readonly** — in its lifetime.
  Each is a *profile*, not a separate codebase.
- The codebase-treatment approach to environment differences
  produces unbounded growth: env-var-keyed conditionals smear
  orthogonal concerns into one expression, setup scripts diverge,
  business logic acquires environment branches.  The cost
  compounds; the team's velocity on cross-environment changes
  eventually stops.
- The profile-treatment approach asks the code to be *passive*
  with respect to environment.  Code reads typed configurations
  through the runtime API; the configurations differ between
  profiles; the code is identical across all four environments.
- Three commitments make the discipline possible: read
  configurations through the runtime, reference them by stable
  identifiers, and treat profile selection as plumbing rather than
  as application logic.
- A worked test trace through the four environments shows six
  dimensions of difference (endpoints, credentials, backends,
  log levels, timeouts, property-test caps) carried entirely as
  *content*, with zero lines of conditional code in the test
  itself.
- The shipped portfolio provides the substrate
  (`override_variable_value`, `MergeMap`, `AppVarSet`,
  `ContextPaths`, the existing `AM_*_NAMES` family) that the next
  four chapters build the user-facing profile system on top of.
- Chapters 14, 15, 16, 17 develop the mechanics, the local-dev
  story, the CI-replay story, and the promotion story.

The next chapter proposes the profile mechanism itself: what a
profile is, how profiles compose, how selection happens at process
start, and what the `automojo profile show` CLI command would look
like as an addition to the portfolio.
