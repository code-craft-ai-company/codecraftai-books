# Chapter 4 — Property-Based Testing with `@given`

> *Example-based tests verify the cases the test author thought
> of.  Property-based tests verify the property the test author
> stated, against an unbounded space of inputs.  The hard part
> is not writing the property — it is making the framework's
> generation budget *match* the time the suite has to run.  This
> chapter is about that.*

Property-based testing in `automojo-testbase` is conceptually a
sibling of Hypothesis — the testing library most Python engineers
know — but with two specific adaptations for automation suites:

1. **A runtime-overridable cap on `max_examples`.**  The
   environment variable `TESTBASE_MAX_EXAMPLES_CAP` constrains
   how many examples each property test may generate, regardless
   of what the test declared.  This lets a build engineer tune
   the *whole suite* without editing the *individual tests*.
2. **A deterministic per-test seed derived from the job seed.**
   The framework computes
   `SHA-256(job_seed + ":" + test_name)` and feeds it to the
   per-test generator.  Re-running a job with the same job seed
   reproduces the same per-test seeds — the basis for replayable
   property-based runs.

The rest of the chapter walks `@given`, the strategies it
accepts, the cap, and the per-test seed.  Where Hypothesis
behaviour differs, the chapter calls it out explicitly.

## The `@given` decorator

```python
# automojo-testbase/source/packages/automojo/testbase/properties/decorators.py:23–97

def given(*, constraints: Optional[Dict[str, Any]] = None, **strategies: Strategy) -> Callable:
    """
        Decorate a test function with strategies for property-based
        execution.

        :param constraints: optional dict with `max_examples` and/or
            `max_shrink_attempts` keys.
        :param strategies: keyword arguments mapping the test
            function's parameter names to Strategy instances.
    """
    def _decorator(func: Callable) -> Callable:
        func._property_ = {
            "strategies": strategies,
            "constraints": constraints if constraints is not None else {},
        }
        rtnval = func
        return rtnval
    return _decorator
```

The decorator stores metadata on the function under
`_property_`.  Per Volume 0 chapter 2's decorator discipline,
nothing about the function body changes.  The property runner
(below) is the thing that *uses* the metadata.

A typical use:

```python
import testbase
from testbase.strategies import integers, text

@testbase.given(
    a=integers(min_value=-1000, max_value=1000),
    b=integers(min_value=-1000, max_value=1000),
    constraints={"max_examples": 50},
)
def test_addition_is_commutative(a, b):
    """
        Verify that integer addition is commutative across the
        given range.
    """
    rtnval = (a + b) == (b + a)
    assert rtnval
    return
```

The strategy keys map onto the test function's parameter names.
`constraints={"max_examples": 50}` says "generate at most 50
examples per run."  The `max_shrink_attempts` key (also in the
constraints dict) caps how many shrinks the runner attempts after
finding a failure.

## Constraint vocabulary

The constraint dict supports two keys
(`automojo-testbase/source/packages/automojo/testbase/properties/decorators.py:42–44`):

- **`max_examples`** — integer, default 100.  How many distinct
  inputs to generate before declaring the property holds.
- **`max_shrink_attempts`** — integer, default 100.  After the
  runner finds an input that violates the property, how many
  attempts to make at finding a smaller input that still
  violates.

Per Volume 0 chapter 4 (the Aspects pattern), behaviour
parameters live on the test, not on a wrapping decorator.  There
is no `@retry`, no `@timeout`, no `@shrink_more` decorator.  The
test author writes `constraints={...}` and the runner respects
it — except where the runtime cap overrides.

## `TESTBASE_MAX_EXAMPLES_CAP`: the build engineer's knob

Here is where testbase parts company with Hypothesis.

```python
# automojo-testbase/source/packages/automojo/testbase/properties/propertyrunner.py:47–79

ENV_MAX_EXAMPLES_CAP = "TESTBASE_MAX_EXAMPLES_CAP"

def _resolve_max_examples_cap() -> Optional[int]:
    """
        Read the TESTBASE_MAX_EXAMPLES_CAP environment variable and
        return the integer value, or None if unset.

        :returns: integer cap or None.
        :raises: ValueError when the env var is set but unparseable.
    """
    raw = os.environ.get(ENV_MAX_EXAMPLES_CAP, None)
    if raw is None:
        rtnval = None
        return rtnval

    try:
        rtnval = int(raw)
    except ValueError as cause:
        err_msg_lines = [
            "TESTBASE_MAX_EXAMPLES_CAP must be a parseable integer.",
            "    VALUE: {!r}".format(raw),
        ]
        err_msg = os.linesep.join(err_msg_lines)
        raise ConfigurationError(err_msg) from cause

    return rtnval
```

The function reads the environment variable at runtime.  When the
property runner constructs itself, it stores the cap.  When it
runs a test, it picks the *smaller* of (the test's declared
`max_examples`, the cap).  A test that says `max_examples=200`
running under `TESTBASE_MAX_EXAMPLES_CAP=10` runs ten examples.
A test that says `max_examples=5` running under the same cap
runs five.

The cap is *only an upper bound*.  It cannot make a test
generate more examples than its declaration; it can only generate
fewer.  This is intentional: a test author who deliberately
budgets 5 examples (because the strategy is expensive) does not
have their budget *raised* by an environment variable.

The cap is the build engineer's knob.  In CI, set it to 10 — the
suite finishes in a CI window short enough to run on every PR.
On a nightly full-sweep run, leave it unset — the suite runs at
its full declared budget.  In local development, set it
to 1 or 5 for a fast smoke pass.

This is the Aspects pattern from Volume 0 chapter 4 lifted to
*the suite level*.  The behaviour parameter (how many examples)
is set by the *caller* of the runner, not edited into hundreds
of test files.  Chapter 10 returns to this knob in detail —
the case study suite uses `TESTBASE_MAX_EXAMPLES_CAP=10` to
finish in 49 minutes wall time.

## Determinism: the per-test seed

Property-based tests are only useful in CI if they are
reproducible.  A test that fails non-deterministically because
each run generates different inputs is a flake.  Testbase
addresses this by deriving the per-test seed deterministically
from the job seed:

```python
# automojo-testbase/source/packages/automojo/testbase/properties/propertyrunner.py:82–97

def derive_test_seed(job_seed: bytes, test_name: str) -> int:
    """
        Derive a deterministic per-test seed from the job seed and the
        test name.

        :param job_seed: the run-level seed (bytes).
        :param test_name: the fully-qualified test function name.

        :returns: integer per-test seed suitable for feeding to a PRNG.
    """
    payload = job_seed + b":" + test_name.encode("utf-8")
    digest = hashlib.sha256(payload).digest()
    rtnval = int.from_bytes(digest[:8], byteorder="big")
    return rtnval
```

The job seed is a fixed-length byte string set once at job start
(typically by `TestJob`'s constructor).  The test name is the
fully-qualified module path plus function name.  SHA-256 over
`job_seed + b":" + test_name` produces a 32-byte digest; the
first eight bytes become a 64-bit integer the runner feeds to
the strategy.

The properties this gives:

- **Re-running the same job seed reproduces every per-test
  seed.**  A failing run can be replayed end-to-end.
- **Adding a test does not change the seeds of existing tests.**
  The seed is a function of *that test's* name and the job seed,
  so the inputs other tests see are unchanged.
- **Renaming a test changes its seed.**  This is unavoidable —
  but the build engineer who renames a test should expect to
  re-baseline that test's recent runs anyway.

A specific question this raises: can the test author override
the per-test seed?  No.  Per the property runner declaration
(`automojo-testbase/source/packages/automojo/testbase/properties/decorators.py:49`)
there is no `seed` constraint key.  The `job_seed` is the only
seed input; the framework is the only seed deriver.  This is
deliberate: a test that gets to override its own seed cannot be
re-baselined to a known-good run by an external operator.

## The PropertyRunner

The runner that ties the strategies, constraints, cap, and seed
together is `PropertyRunner`
(`automojo-testbase/source/packages/automojo/testbase/properties/propertyrunner.py:100–120`):

```python
class PropertyRunner:
    def __init__(self,
                 constraints: Dict[str, Any],
                 job_seed: bytes,
                 test_name: str,
                 recorder: ResultRecorder,
                 parent_inst: TestInstance) -> None:
        self._constraints = constraints
        self._job_seed = job_seed
        self._test_name = test_name
        self._recorder = recorder
        self._parent_inst = parent_inst
        self._cap = _resolve_max_examples_cap()
        return
```

Six instance variables, all set in `__init__` per the Volume 0
chapter 2 discipline (no late-introduced instance variables).
The runner exposes a `run()` method that:

1. Resolves the effective `max_examples` (`min(declared, cap)`).
2. Resolves the `max_shrink_attempts` (declared only — the cap
   does not apply).
3. Derives the per-test seed.
4. Drives the strategy generators for `max_examples` iterations.
5. For each iteration: emits a *property-iteration* event to
   the recorder, runs the test body with the generated inputs,
   emits a result event.
6. On failure, runs up to `max_shrink_attempts` shrinks and
   reports the smallest failing input.

The result events get recorded to the JSOS stream same as
example-based tests; Chapter 5 covers the format.

## Composition with `@testbase.attach.parameter`

A `@given` test can also have attached parameters from Chapter
3.  The decorators compose:

```python
@testbase.attach.parameter(db_session, identifier="db")
@testbase.given(
    invoice_number=integers(min_value=1, max_value=10**9),
    constraints={"max_examples": 25},
)
def test_invoice_lookup_round_trips(db, invoice_number):
    """
        Verify that an invoice round-trips through the lookup API.
    """
    inserted = db.insert_invoice(invoice_number)
    looked_up = db.lookup_invoice(invoice_number)
    rtnval = (inserted == looked_up)
    assert rtnval
    return
```

The strategy keys (`invoice_number`) and the attachment keys
(`db`) are disjoint — they bind to different parameters.  The
attached resources are acquired *once* at the resource lifespan
boundary (Session, Package, or Test); the strategy values are
generated *per iteration* of the property test.

A subtle point: when `db` is a Test-scoped resource, the runner
re-acquires `db` on *every iteration*, not just every test
invocation.  If `db` is expensive, prefer Package or Session
scope — or generate the per-iteration data inside the test body
rather than asking the originator to do it.

## Strategy library

Testbase ships its own strategy library (under
`automojo-testbase/source/packages/automojo/testbase/strategies/`).
The vocabulary is similar to Hypothesis:

- `integers(min_value=..., max_value=...)`
- `floats(min_value=..., max_value=..., allow_nan=...)`
- `text(min_size=..., max_size=..., alphabet=...)`
- `booleans()`
- `lists(elements=..., min_size=..., max_size=...)`
- `tuples(*element_strategies)`
- `dictionaries(keys=..., values=..., min_size=..., max_size=...)`
- `one_of(*strategies)`
- `none()` and `just(value)`

Composition is via `lists(elements=integers())` and `one_of(...)`.
A user-defined strategy is a class that subclasses
`Strategy` and implements `draw(self, rng)` — the per-iteration
hook the runner calls.

The strategy implementations are testbase's own, not Hypothesis's.
This is a deliberate choice: it keeps the framework's dependency
graph free of Hypothesis (a sizable external dependency) and
gives the runner control over the generation budget at a
per-test level.

## Shrinking

When the runner finds an input that violates the property, it
attempts to *shrink* it — find a smaller, structurally-simpler
input that also violates the property.  The shrink algorithm is
strategy-specific: integers shrink toward zero, lists shrink by
removing elements, strings shrink toward empty.

The `max_shrink_attempts` constraint caps the number of shrink
attempts.  When the cap is reached the runner reports the
smallest-found-so-far input.

The reported failure carries:

- The input that failed.
- The seed used.
- The number of shrinks attempted.
- The exception (or the failed assertion's left/right operands).

A test author re-running locally with the reported input can
reproduce the failure deterministically — without re-running
the entire generation budget.

## What property-based testing is good for

A working list, with examples from the portfolio:

- **Round-trip invariants**: encode/decode, serialise/parse,
  insert/lookup.  Property: `decode(encode(x)) == x`.
- **Algebraic identities**: commutativity, associativity,
  identity, idempotency.
- **Boundary discovery**: for any `i in [min, max]`, function
  `f(i)` does not raise.
- **Ordering invariants**: a sort produces a non-decreasing
  sequence; a deduplicator never emits the same key twice.
- **Configuration loaders**: for any valid config dict shaped
  like `S`, the loader produces an object whose properties match
  `S`'s field semantics.

What property-based testing is *not* good for: end-to-end
integration tests where each iteration is expensive (each one
takes a database round-trip).  Those should remain
example-based, with a small handful of carefully chosen inputs.

## What's next

- Chapter 5 — Streaming JSOS results.  Each iteration of a
  property-based test becomes a JSOS record; the format and
  the consumers.
- Chapter 6 — The `agents` CLI.  How `agents testresult list`
  and `agents testresult failures` consume the JSOS stream.

Chapter 10's case study uses `TESTBASE_MAX_EXAMPLES_CAP=10` on
a 3,700-test single-host suite.  By the time the reader reaches
that chapter the cap will be familiar — the case study just
shows what the suite-level wall time is when the cap is set.
