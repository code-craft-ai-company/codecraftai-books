# Chapter 16 — Reproducing CI Failures Locally

> *When a test passes locally but fails in CI, the question is
> always "what's different about CI?".  With typed profiles, the
> answer is one filename.*

The previous chapter cancelled the local-dev tax by moving every
piece of fresh-checkout state into typed configuration files.  The
result is a local-dev profile that mirrors a real environment with
high fidelity.  This chapter is the operational payoff: when a CI
run fails for a reason that doesn't reproduce locally, the
discipline lets a developer *capture* the exact resolved
configuration of the failing run, replay it on their laptop, and
debug the difference in their IDE.

The chapter is short on argument and long on machinery.  By this
point in the volume the case for typed configuration has been
made.  This chapter is what doing it well *enables*.

## Where the gap comes from

Every team eventually hits the same scenario.  A test passes when
the developer runs it locally.  The same test fails when CI runs
it.  The developer reads the CI log, sees the failure, finds it
inscrutable, and re-runs the test on their machine.  It passes.
They re-trigger the CI run; it fails again.  Three hours go by;
the developer is frustrated; the CI lane is blocked.

The gap is not magical.  The gap is *configuration drift*.
Concretely, eight common sources:

1. **Different log levels.**  A test that fails because of a
   log-level-dependent code path (an `if logger.isEnabledFor(...)`
   branch, a debug-only side-effect) behaves one way in CI's
   `INFO` and another way in local-dev's `DEBUG`.
2. **Different timeouts.**  A flaky network call completes in
   local-dev's 300 second timeout but not in CI's 30 second
   timeout.
3. **Different parallelism.**  A race condition that requires four
   concurrent workers shows up in CI's 4-worker pool but not in
   local-dev's 1-worker run.
4. **Different property-test sweeps.**  CI's `max_examples=10`
   covers a different region of input space than local-dev's
   `max_examples=100`.  The failing case happens to fall inside
   one window but not the other.
5. **Different random seeds.**  The same property test with the
   same `max_examples` produces a different sequence of cases when
   the seed differs.
6. **Different landscape entries.**  CI's staging-api lives at one
   host; local-dev's staging-api lives at another.  A test that
   relies on a host-specific quirk passes against one and fails
   against the other.
7. **Different credentials' permissions.**  CI's database
   credential is read-write; local-dev's is read-only (or vice
   versa).  A test that writes data passes in one and fails in
   the other.
8. **Different data fixtures.**  CI's database has been seeded
   with the integration-test dataset; local-dev's database has
   only the smoke-test dataset.  A test that queries data missing
   from one of the seeds fails on that side.

Every one of these gaps is a *configuration difference*.  Every
configuration difference is captured by typed configuration files
plus the active profile.  If the developer can faithfully
reconstruct CI's resolved configuration locally, the gap collapses.

## The capture primitive

The proposed primitive: a CLI command that emits a snapshot of
every resolved configuration value at a given moment in a run.

```bash
automojo profile capture --output ci-failure-snapshot.json
```

Output (the snapshot file):

```json
{
    "profile_name": "integration-ci",
    "captured_at": "2026-04-30T14:32:11Z",
    "captured_for": {
        "run_id": "ci-12345",
        "job": "integration-test",
        "commit_sha": "abc123def",
        "operator": null,
        "vendor": "gitlab-ci",
        "vendor_url": "https://gitlab.example.com/team/project/-/jobs/12345"
    },
    "resolved_variables": {
        "AM_PROFILE": "integration-ci",
        "AM_CONFIG_LANDSCAPE_NAMES": "staging-east",
        "AM_CONFIG_TOPOLOGY_NAMES": "staging-east-leader-follower",
        "AM_CONFIG_RUNTIME_NAMES": "integration-ci",
        "AM_CONFIG_CREDENTIAL_NAMES": "integration-ci",
        "AM_CONFIG_DATAPROFILE_NAMES": "integration-ci",
        "TESTBASE_MAX_EXAMPLES_CAP": "10",
        "TESTBASE_PROPERTY_SEED": "1714492331",
        "AM_LOG_LEVEL_CONSOLE": "INFO",
        "AM_LOG_LEVEL_LOGFILE": "DEBUG"
    },
    "configuration_snapshots": {
        "landscape": {
            "environment": {"label": "staging-east"},
            "apod": {"clients": [...]},
            "infrastructure": {"services": [...]}
        },
        "topology": {
            "environment": {"label": "staging-east-leader-follower"},
            "clusters": [...],
            "dependencies": [...],
            "roles": [...]
        },
        "runtime": {
            "logging": {"levels": {"console": "INFO", "logfile": "DEBUG"}},
            "parallelism": {"max_workers": 4},
            "timeouts": {"default_seconds": 30}
        },
        "credentials": [
            {
                "identifier": "ci-runner-ssh",
                "categories": ["ssh", "environment:integration-ci"],
                "username": "runner",
                "keyfile": "<redacted-path>",
                "keypasswd": "<redacted>",
                "allow_agent": false,
                "primitive": false
            }
        ],
        "dataprofiles": [...]
    }
}
```

Five sections matter:

- **`profile_name`** — the active profile when the snapshot was
  taken.  Identifies the starting point for replay.
- **`captured_for`** — provenance.  Which CI run, which commit,
  which vendor.  Useful for retrospective analysis weeks later.
- **`resolved_variables`** — every active `AppVarDesc` with its
  resolved value.  This is the answer to "what was the effective
  parameterisation of this run?"  Includes the property-test
  seed (`TESTBASE_PROPERTY_SEED`) so a property test failing on
  case 7 of 10 reproduces deterministically.
- **`configuration_snapshots`** — the actual content of each
  loaded category, post-resolution and post-overlay.  This is the
  defence against "the named configuration changed between then
  and now"; the snapshot has the *content* the run saw, not just
  the names.
- **`credentials` (under snapshots)** — the credential identifiers
  and metadata, with all secret fields replaced by `<redacted>`
  markers.  The snapshot is *not* a credential leak; the
  replay layer re-resolves the redacted fields against the local
  backend.

The capture command is roughly seventy lines of Python: walk
`AppVarSet.items()` (chapter 14), serialise each loaded
`CONFIGURATION_MAPS` entry (chapter 3), apply the chapter-11
redacting filter to credential fields, write the result as JSON.
The implementation reuses every primitive the volume has built so
far.

## CI integration: when to capture

Three patterns for wiring capture into a CI pipeline.

### Capture on failure

The most common pattern.  The CI step's failure path captures the
snapshot before exiting:

```yaml
# .gitlab-ci.yml
integration-test:
  variables:
    AM_PROFILE: "integration-ci"
  script:
    - automojo testbase test run tests/ || (
        automojo profile capture --output failure-snapshot.json
        && exit 1
      )
  artifacts:
    when: on_failure
    paths:
      - failure-snapshot.json
    expire_in: 1 month
```

Three properties of this wiring:

- **The capture only runs when the test fails.**  The vast
  majority of CI runs don't need the snapshot, and the I/O
  overhead is not paid in the common case.
- **The snapshot is uploaded as a job artifact.**  The developer
  downloads it from the failed CI job's artifact view.  Vendor-
  specific paths exist for this in every major CI vendor.
- **The artifact has a TTL.**  After a month, the snapshot is
  auto-deleted.  Long-term storage of CI snapshots is a separate
  concern (an audit-log feed for incident response, perhaps); the
  artifact path is for short-term debugging.

### Capture on every run

Some teams want the snapshot regardless of pass or fail, for
historical reasons (the `flaky-tests` dashboard, the
`config-drift` analyser).  The pattern:

```yaml
integration-test:
  script:
    - automojo testbase test run-with-capture tests/ \
        --capture-output ./snapshots/integration-ci-${CI_JOB_ID}.json
```

The `run-with-capture` wrapper captures *before* the test runs,
runs the test, and writes the snapshot regardless of test
outcome.  The snapshot includes the test outcome as a top-level
field.

### Capture on demand

For incident response, an on-call engineer can capture the active
profile of a *running* orchestration on a remote host:

```bash
automojo profile capture --remote ssh://runner-1.staging.east.internal --output now.json
```

Implementation: the remote `automojo profile capture` runs in the
target process's context (which requires the orchestration to
expose a small introspection RPC, similar to Volume 8's
time-travel debugging hooks).  This pattern is more complex and
out of scope for the chapter's main thread; mentioned for
completeness.

## The replay primitive

The matching command on the developer side:

```bash
automojo testbase test run --replay-profile failure-snapshot.json tests/
```

The `--replay-profile` flag tells the orchestration's startup:

1. **Read** the snapshot file.
2. **Skip** normal profile resolution.  The snapshot *is* the
   resolved configuration; no `AM_PROFILE` lookup, no `extends:`
   chain walk.
3. **Apply** the snapshot's `resolved_variables` via
   `override_variable_value` calls.  Now every active variable has
   the value it had in CI.
4. **Register** the snapshot's `configuration_snapshots` as
   highest-priority `MergeMap` overlays for each category.  Now
   the loaded landscape, topology, runtime, and data profiles are
   the same content the CI run saw.
5. **Re-resolve** redacted credentials against the local backend
   roster.  For each credential whose `<redacted>` markers
   indicate missing secret material, the resolver looks up the
   credential identifier in the *local-dev* credential file and
   uses the local-dev value.

After step 5, the test runs with CI's exact non-secret
configuration — same landscape, same topology, same runtime, same
property-test seed, same `TESTBASE_MAX_EXAMPLES_CAP` — and
locally-resolvable credentials.

The credentials substitution is the security boundary.  The
developer cannot replay against production credentials they don't
have access to; the replay uses *their* local-dev credentials,
even when the snapshot was captured from CI.

## What the replay reproduces and what it doesn't

The replay reproduces every gap from the list at the top of the
chapter:

| Gap | Reproduced by replay? |
|---|---|
| Different log levels | ✓ — runtime configuration is in the snapshot |
| Different timeouts | ✓ — runtime configuration |
| Different parallelism | ✓ — runtime configuration |
| Different property-test sweeps | ✓ — `TESTBASE_MAX_EXAMPLES_CAP` is captured |
| Different random seeds | ✓ — `TESTBASE_PROPERTY_SEED` is captured |
| Different landscape entries | ✓ — landscape snapshot is in the file |
| Different credentials' *behaviour* | ✗ — local credentials replace CI credentials |
| Different data fixtures | ✗ — the data is in the database, not in the configuration |

The two gaps the replay does *not* reproduce are the two where
the configuration is not the source of truth:

- **Credential behaviour.**  A test that fails because the CI
  credential has different permissions than the local credential
  will pass under replay because replay uses local credentials.
  This is intentional (the security boundary); the discipline for
  this case is to ensure the local-dev credential has the same
  permissions as the CI credential — same role, same scope tags,
  same backend-side ACL.  When this is true, the gap doesn't
  exist.
- **Data fixtures.**  A test that depends on rows existing in a
  database depends on the database's *content*, which lives in
  the data store, not in the data-profile configuration.  Replay
  can't reach into the data store and reproduce its state.  The
  discipline for this case is to ensure data fixtures are
  themselves typed (managed by a Specification or by a test-time
  fixture loader); the replay then re-runs the fixture loader to
  produce the same state locally.

For the eight common gaps, six are reproduced cleanly and two
require complementary discipline.  The discipline closes the
others.

## A worked CI-failure walk-through

A flaky property test:

```python
@given(integers=st.lists(st.integers(), min_size=1, max_size=100))
def test_sum_is_at_least_max(integers):
    """ The sum of a list of integers is at least the max. """
    verification.assert_true(sum(integers) >= max(integers))
    return
```

The test is incorrect — for lists of negatives, the sum can be
*less* than the max.  The bug is silent for most randomly-
generated input (lists with at least one positive element rarely
violate the property) but surfaces when the random sequence
produces an all-negative list.

### The CI failure

CI runs the test with `TESTBASE_MAX_EXAMPLES_CAP=10` and a
property-test seed of `1714492331`.  At case 7 of 10 the seed
produces `[-3, -1, -2]`; sum is `-6`, max is `-1`; `-6 >= -1` is
false; the test fails.

The CI logs:

```
FAILED tests/test_property.py::test_sum_is_at_least_max
Falsifying example: integers=[-3, -1, -2]
```

### The local "passes"

The developer pulls the branch and runs:

```bash
automojo testbase test run --profile local-dev tests/test_property.py
```

The local-dev runtime profile sets `TESTBASE_MAX_EXAMPLES_CAP=5`
(local-dev preference for fast feedback) and the property-test
seed is fresh (let's say `1714600000`, the current time in
seconds when the developer ran the test).  The five generated
cases all happen to have at least one positive integer; the
property holds; the test passes.

The developer is confused.  The CI log shows the failing
example explicitly: `[-3, -1, -2]`.  Why doesn't reproducing the
example locally fail too?

Without replay, the developer would manually paste the failing
example into a unit test, confirm it fails, and proceed to fix
the bug.  This is acceptable for one failure but expensive for
many — and many tests don't print their falsifying example as
clearly as Hypothesis does.

### The replay

The developer downloads the failure snapshot from the CI job's
artifacts and runs:

```bash
automojo testbase test run --replay-profile failure-snapshot.json tests/test_property.py
```

The replay:

- Sets `TESTBASE_MAX_EXAMPLES_CAP=10` (from the snapshot).
- Sets `TESTBASE_PROPERTY_SEED=1714492331` (from the snapshot).
- Loads the same staging-east landscape, topology, etc. (irrelevant
  for this test, but uniformly applied).
- Re-resolves credentials against local-dev.

The test runs.  Because the seed is identical and the
`max_examples` is identical, the property-test framework
generates the same sequence of cases — case 7 is `[-3, -1, -2]`,
the assertion fails, the test fails locally.

The developer can now set a breakpoint on the failing assertion,
run the test under the debugger, step through the implementation,
and find the bug.  The whole loop — capture, download, replay,
debug — takes ten minutes; without replay, the equivalent loop
might involve three CI re-runs and an hour of guessing.

## `TESTBASE_MAX_EXAMPLES_CAP` as the canonical small case

The `TESTBASE_MAX_EXAMPLES_CAP` knob, introduced in chapter 7 as
the smallest example of a typed runtime configuration that
switches behaviour, is the smallest worked example of replay.
A property test that fails at case 7 of CI's 10 will:

- **Pass locally without replay.**  Different cap (or no cap)
  plus different seed produces a different case sequence.
- **Fail locally with replay.**  Same cap, same seed reproduces
  the same case sequence and the same failure.

The chapter's wider point: the discipline of treating *every*
behaviour-controlling parameter as runtime configuration —
chapter 7's argument — is what makes replay work.  If
`TESTBASE_MAX_EXAMPLES_CAP` were a flag on the test runner
instead of a runtime configuration field, the snapshot wouldn't
capture it, and the replay wouldn't reproduce the behaviour.
Each piece of "this is just an env var I sometimes set" is a
piece that resists capture.  The discipline of typed runtime
configuration is what *closes* the gap; replay is the operational
mechanism that *exploits* the closure.

## Captured profile as audit artifact

Once captures exist, they are useful for more than debugging the
specific failure that produced them.  Three secondary uses worth
naming:

### Cross-environment diff

Capture the profile of a successful staging run; capture the
profile of a successful production run; diff them.  The diff
shows every configuration difference between staging and
production.  Differences that *should* exist (different
landscape labels, different credentials) appear; differences
that *shouldn't* (a runtime field that's set in one and unset in
the other) are the candidates for the next round of staging-vs-
production drift fixes.

### Historical replay

A test that passed six months ago and now fails — but on the
same code — is almost certainly a case of environment drift.  A
captured profile from the six-months-ago passing run lets the
developer replay against the historical configuration and
confirm whether the failure is configuration-driven or genuinely
a regression.  When the historical replay passes, the issue is
configuration drift; when it fails, the issue is in the code or
in something the configuration didn't capture (data fixture
drift, dependency upgrades).

### Compliance and audit

Some regulated environments require the team to be able to
demonstrate, after the fact, the exact configuration of a
production-affecting run.  Captured profiles satisfy this with
no additional machinery: every run captures, every capture is
archived, every archive is searchable by `captured_for` fields.

## What the replay primitive does *not* fix

Three boundaries to keep clear:

### Not a fix for code that depends on undeclared state

If a test reads `os.environ["FOO"]` directly (in violation of
the chapter-7 plumbing-vs-content principle), the snapshot will
not capture `FOO` because `FOO` is not a typed runtime variable.
The replay will not set `FOO`; the local replay will read
whatever `FOO` happens to be in the developer's environment, and
the gap remains.

The fix is the chapter-7 discipline: every behaviour-controlling
value becomes a typed configuration field.  Once it's typed, it's
captured; once it's captured, it's reproducible.

### Not a fix for non-deterministic tests

A test whose outcome depends on wall-clock time, random numbers
generated outside the property-test framework's seed, network
latency, or the moon phase will not reproduce under replay even
when the captured profile is identical.  Non-deterministic tests
are a separate engineering problem; the discipline can't fix them
without first making them deterministic.

The pattern that does work is to capture *the random seeds the
test uses internally* in the snapshot.  Test code that does
`random.seed(int(time.time()))` would migrate to
`random.seed(int(os.environ.get("TEST_SEED", time.time())))` and
have the runtime configuration include `test_seed`.  Once the
seed is captured, the test is deterministic for replay purposes.

### Not a fix for data drift

Replay doesn't reach into the database or the queue or the
filesystem.  When the test depends on a row existing or a file
being present, replay reproduces the *configuration* of the run
but not the *state of the world*.  The discipline:

- Use **data fixtures** (managed by a Specification or by a
  test-time fixture loader) so the data state is itself typed
  and reproducible.
- Where the data is genuinely external and beyond the
  orchestration's control (a third-party API's user database, a
  shared Kafka topic), document that the test depends on
  externalities and quarantine it from the deterministic test
  suite.

## Implementation cost

The capture and replay primitives together are roughly two
hundred lines of Python:

- `automojo profile capture`: ~70 lines (walk `AppVarSet.items()`,
  serialise the configuration maps, apply redaction, write JSON).
- `automojo testbase test run --replay-profile`: ~80 lines (parse
  the snapshot, override variables, register overlays, swap in
  local credentials, hand off to the existing test runner).
- The CI-side capture-on-failure wrapper: ~10 lines of bash per
  vendor.
- The redacting filter: already shipped (chapter 11).
- The variable iteration: already shipped (`AppVarSet`).
- The configuration snapshot: already shipped
  (`CONFIGURATION_MAPS` is the source of truth).

The discipline is built on top of what the portfolio already
provides.  The new tooling is small.

## Summary

- The "passes locally, fails in CI" gap has eight common sources
  — log levels, timeouts, parallelism, property-test sweeps,
  random seeds, landscape entries, credentials, data fixtures.
  Six of the eight are *configuration differences*; the
  discipline reproduces them faithfully via replay.
- The **capture primitive** (`automojo profile capture`) emits a
  JSON snapshot of every resolved configuration value at a moment
  in a run.  Includes resolved variables, configuration content
  per category, credential metadata with secrets redacted, and
  capture provenance (CI run, commit, vendor).
- The **replay primitive**
  (`automojo testbase test run --replay-profile <file>`) reads a
  snapshot, applies variable overrides, registers
  configuration overlays, re-resolves redacted credentials
  against the local backend roster, and runs the orchestration
  with the captured configuration.
- The `TESTBASE_MAX_EXAMPLES_CAP` + `TESTBASE_PROPERTY_SEED`
  pair makes property-test failures deterministically
  reproducible.  Same cap + same seed = same case sequence.
- The replay is *not* a fix for code that reads undeclared
  state, *not* a fix for genuinely non-deterministic tests, and
  *not* a fix for data drift.  Each is a separate engineering
  problem with its own discipline.
- Captured profiles double as audit artifacts: cross-environment
  drift detection, historical replay for "it used to work," and
  compliance evidence for regulated environments.
- The capture and replay primitives together are roughly two
  hundred lines on top of the shipped portfolio.  Every piece
  the discipline depends on already exists.

The next chapter is the last in Part IV: profile promotion.  How
a local-dev profile additively becomes staging via overlay; how
staging additively becomes prod via overlay; how additions in
one profile flow naturally into descendants without coordinated
edits across files.
