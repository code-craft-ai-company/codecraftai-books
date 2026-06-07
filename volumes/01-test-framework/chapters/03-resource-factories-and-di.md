# Chapter 3 — Resource Factories and Dependency Injection

> *A test that imports its dependencies cannot run on a machine
> that doesn't have them.  A test that declares its dependencies
> can.  This chapter is about the difference, and about the small
> set of decorators testbase uses to express it.*

The previous chapter walked the architecture from CLI invocation
to result file.  This chapter zooms in on one phase — the
*Collection* and *Graph* phases — where resource factories are
discovered, dependency graphs are built, and the test
parameters are bound to the resources they need.

The vocabulary here matters.  Pytest calls them *fixtures*.
Spring (in another world) calls them *beans*.  Dagger / Guice
call them *bindings*.  Testbase calls them *originators*, and
the way they get attached to a test parameter is the
`@testbase.attach.parameter(...)` decorator.  The vocabulary is
not arbitrary: an *originator* is the function that *originates*
the resource; the *attachment* binds the originator's output to
a parameter on a test.

By the end of this chapter you will know how to register an
originator, how to declare a parameter on a test, and how the
sequencer's Graph phase lines them up.

## The principle: declare, don't import

A pytest test that needs a database session imports a fixture:

```python
# pytest
from myproject.testfixtures import db_session

def test_some_query(db_session):
    rtnval = db_session.query(...).all()
    assert len(rtnval) > 0
```

The import works only if `myproject.testfixtures` is on the
Python path *at the time the test module is imported*.  On a
laptop without the dev database, the import fails — and pytest
reports a collection error rather than a clean "this test isn't
runnable here."

A testbase test attaches a parameter via decoration:

```python
# testbase
import testbase

@testbase.attach.parameter(db_session, identifier="db")
def test_some_query(db):
    """
        Verify that the canonical query returns at least one row.
    """
    rtnval = db.query(...).all()
    assert len(rtnval) > 0
    return
```

The `db_session` reference here is the *originator function* —
a function registered with the framework via
`@configuration_factory()` or `@integration_factory()` (more on
the difference below).  The originator is *not imported* at the
test file's top level.  It is referenced; the framework
resolves the reference during the Collection phase, and only
then does it know whether the resource is available on this
machine.

The consequence is that a test author can *declare* a dependency
that won't resolve on their laptop, and the framework's response
is "this test is unrunnable here" — not an import error.

## Originators: where resources come from

An originator is a Python function that yields a resource.  It
lives in a *factory module* — typically
`<repo>/source/packages/<vendor>/<package>/factories/` — and is
decorated with one of three registrations:

```python
# codecraftai-testbase/source/packages/ccai/testbase/declaration/registrations.py:45–93

def integration_factory(scope: ResourceLifespan = ResourceLifespan.Session):
    """
        Decorator that registers a generator function as an integration
        factory yielding IntegrationCoupling subclasses.
    """
    def _decorator(func: Callable) -> Callable:
        InjectionRegistry.register_integration_factory(func, scope=scope)
        rtnval = func
        return rtnval
    return _decorator
```

The three families:

- **`@integration_factory()`** — yields integration couplings.
  Used to register, for example, a coupling to a credentials
  store, a coupling to a device pool.  These are typically
  *Session*-scoped: one instance for the whole run.
- **`@configuration_factory()`** — yields configuration
  couplings.  Used to register a runtime file source, a
  landscape source.  Typically Session-scoped.
- **`@resource_factory()`** — yields plain resources (database
  sessions, HTTP clients, mock objects).  Lifespan varies —
  Session, Package, Test.

The decorator does not *modify* the function; it registers the
function with `InjectionRegistry`
(`codecraftai-testbase/source/packages/ccai/testbase/injection/injectionregistry.py`).
This is the Volume 0 chapter 2 discipline: decorators that
register, not decorators that wrap.  When `func` is called,
nothing about its body has been altered.  Per Volume 0 the
decorator family is metadata + registration only — never
behaviour modification.

## Lifespan: when the resource is created and destroyed

`ResourceLifespan` is an enum
(`codecraftai-testbase/source/packages/ccai/testbase/resourcelifespan.py`)
with three values:

- **Session** — created once at the start of the run, destroyed
  once at the end.  Use for expensive resources that are
  read-only or thread-safe across tests: a credentials store, a
  device pool registry, a configuration object.
- **Package** — created once per test package, destroyed when
  the package's tests finish.  Use for moderately expensive
  resources that you don't want to share across packages:
  per-package mock servers, per-package temporary directories
  with cross-test continuity.
- **Test** — created and destroyed for each test invocation.
  Use for resources that need a clean slate per test: a fresh
  database transaction, a per-test subdirectory.

The choice of lifespan is the *test author's* choice but the
*framework's* mechanism: the sequencer's Graph phase resolves
each parameter's lifespan and inserts the right number of
acquisition / release boundaries into the traversal plan.

## The originator as a generator

The originator is a generator, not a plain function.  The
generator pattern lets the originator clean up after itself:

```python
@testbase.factories.integration_factory(scope=ResourceLifespan.Session)
def db_session():
    """
        Yields a session-scoped database connection.
    """
    conn = connect_to_db(...)
    try:
        yield conn
    finally:
        conn.close()
    return
```

Everything before the `yield` is acquisition; the value yielded
is what the test receives; everything after the `yield` is
cleanup.  The framework drives `next()` to acquire and the
generator's `__exit__` (via a `try`/`finally`) to release.

The trailing `return` at function end is the explicit-return
discipline from Volume 0 chapter 1.  In a generator the `return`
is also the way to signal `StopIteration` cleanly.

## `@testbase.attach.parameter`: binding the originator to a test

A test asks for a resource by attaching a parameter:

```python
# codecraftai-testbase/source/packages/ccai/testbase/declaration/attachments.py

@testbase.attach.parameter(db_session, identifier="db")
def test_some_query(db):
    """
        Verify the canonical query returns at least one row.
    """
    rtnval = db.query(...).all()
    assert len(rtnval) > 0
    return
```

The decorator takes:

- A reference to the originator (`db_session`).
- An `identifier` — the parameter name on the test function.
- Optionally `constraints` — see below.

When the sequencer builds the dependency graph, it sees the
attachment, resolves the originator from the registry, computes
the lifespan-aware acquisition plan, and inserts a binding into
the traversal plan that ensures the originator's yielded value
is bound to the parameter at the moment the test runs.

The originator function is *not called at import time*.  The
attachment decoration only stores metadata on the test
function; resolution happens during Collection.  This is the
property that lets a test author declare a dependency on a
resource their machine cannot provide.

## `parameter()` vs `@testbase.attach.parameter`

There are two surfaces for declaring a parameter, and the
distinction trips people up at first:

- **`parameter()`** — module-level call (not a decorator)
  that creates a *global* parameter from an originator.  Lives
  in
  `codecraftai-testbase/source/packages/ccai/testbase/declaration/originations.py:31–92`.
  Useful when many tests in a package will need the same
  parameter, and you don't want to repeat the `@attach`
  decoration on each.
- **`@testbase.attach.parameter(...)`** — decorator on a single
  test (or on a `TestGroup` to apply to all tests under it).
  Lives in
  `codecraftai-testbase/source/packages/ccai/testbase/declaration/attachments.py`.
  Useful for per-test attachments.

The trap: `parameter()` is *not* a decorator.  Calling
`@parameter(...)` on a test function will fail.  The originations
module includes an explicit error trap (lines 101–103) that
catches the misuse and points the user at
`@testbase.attach.parameter()`.  The error message is the kind
of guard from Volume 0 chapter 3 — the framework noticing it's
about to behave wrongly, and refusing with a `SemanticError`-
shaped message rather than failing silently.

## Constraints on attachments

The `constraints` argument on
`@testbase.attach.parameter(..., constraints=...)` is where
fixture-level Aspects live.  The constraint dict can specify:

- `life_span` — override the originator's default lifespan for
  this attachment.
- `assigned_scope` — pin the resource to a specific package or
  test scope.
- Originator-specific keys.

This is the Aspects pattern from Volume 0 chapter 4: rather
than have a `@retry_on_failure` decorator wrap the originator,
the originator accepts an `Aspects`-shaped parameter that the
attachment can fill in.  The behaviour is *parameterised*, not
*decorated*.  A test that needs the resource with retries
declares retries; a test that wants the unwrapped resource
declares no retries.  Same originator; different aspects.

## The dependency graph

During the *Graph* phase
(`codecraftai-testbase/source/packages/ccai/testbase/sequencing/testsequencer.py:106–150`),
the sequencer walks the discovered tests and their attached
parameters, looks up each originator in the injection registry,
and constructs a directed acyclic graph where:

- Each originator is a node.
- Each test is a node.
- Each `@testbase.attach.parameter` reference is an edge.
- Each cross-originator dependency (one originator that needs
  another resource as input) is an edge.

The sequencer topologically sorts the graph to derive an
acquisition order.  Cycles are detected here — a cycle in the
resource graph is a `SemanticError` and bubbles immediately,
because by Volume 0 chapter 3 a circular dependency is an API
misuse the framework cannot reasonably recover from.

The graph also drives the *release* order: resources are
released in reverse acquisition order, so a session-scoped
credentials store is closed *after* the last test that depended
on it has finished.

## Composition: an originator that depends on another originator

Originators can depend on other originators by declaring their
own attachments:

```python
@testbase.factories.resource_factory(scope=ResourceLifespan.Session)
@testbase.attach.parameter(credentials_store, identifier="creds")
def db_session(creds):
    """
        Yields a database session authenticated against the credentials
        store.
    """
    user, password = creds.lookup("db_main")
    conn = connect_to_db(user=user, password=password)
    try:
        yield conn
    finally:
        conn.close()
    return
```

Now `db_session` depends on `credentials_store`.  The Graph
phase sees the dependency and ensures the credentials store is
acquired *before* the session, and released *after*.

This composition is how testbase handles the realistic case
where a "fixture" is not one resource but a chain of them: a
session that needs credentials that need a profile that needs
a runtime configuration.  Volume 3 chapters 18–23 walk a
similar chain at orchestration scale.

## Cleanup ordering

The framework guarantees that cleanup runs in *reverse*
acquisition order.  If `db_session` was acquired after
`credentials_store` (because `db_session` depends on
`credentials_store`), then `db_session` will be released
*before* `credentials_store` is.  This is critical for
resources that must be released while their underlying
dependencies are still alive — a database session must be
closed while the credentials are still valid, not after the
credentials store has been torn down and the auth tokens
revoked.

The framework runs each generator's post-yield cleanup as a
separate try/finally block, so a failure in one cleanup does
not abort the others.  The sequencer logs failed cleanups as
errors but does not raise.  This is one of the few places where
testbase deliberately *does not* let an exception bubble — per
Volume 0 chapter 3, "errors should never pass silently unless
purposefully handled and silenced," and a partial-cleanup error
is the canonical "we're already on the way out, log and
continue" case.

## A worked example

Walking the path concretely.  Suppose a test author writes:

```python
# tests/billing/test_invoice.py
import testbase
from ccai.billingkit.factories import db_session, billing_runtime

@testbase.attach.parameter(db_session, identifier="db")
@testbase.attach.parameter(billing_runtime, identifier="runtime")
def test_invoice_total_includes_tax(db, runtime):
    """
        Verify that an invoice total includes tax for runtime profile X.
    """
    invoice = create_test_invoice(db, runtime)
    expected = invoice.subtotal * Decimal("1.0875")
    actual = invoice.total
    rtnval = (actual == expected)
    assert rtnval
    return
```

What happens:

1. **Discovery**: the file is imported.  The two `@attach`
   decorators stash `(originator_ref, identifier)` pairs on
   `test_invoice_total_includes_tax`.  No originator is called
   yet.
2. **Collection**: the sequencer scans the registered
   originators.  `db_session` is a session-scoped resource_factory
   that depends on `credentials_store`.  `billing_runtime` is a
   session-scoped configuration_factory that depends on
   `runtime_profile`.
3. **Graph**: the sequencer builds a DAG with five nodes
   (`credentials_store`, `db_session`, `runtime_profile`,
   `billing_runtime`, `test_invoice_total_includes_tax`) and
   four edges.  Topological sort gives acquisition order:
   `credentials_store`, `db_session`, `runtime_profile`,
   `billing_runtime`, then test execution.  Release runs
   reverse.
4. **Traversal**: the sequencer drives the generators in order.
   `credentials_store` yields a credentials object.
   `db_session` consumes it, yields a session.  Same for the
   runtime path.  Then the test runs with `db` bound to the
   yielded session and `runtime` bound to the yielded runtime
   object.  The recorder gets a "started" event before the
   test body runs and a "finished" event after.
5. **Cleanup**: in reverse order — `billing_runtime`'s
   post-yield, `runtime_profile`'s, `db_session`'s,
   `credentials_store`'s.

This is the worked example you can carry into Chapter 5 (where
the recorder's events become JSOS records) and into Chapter 9
(where the same dependency graph extends to landscape-aware
resources).

## What this enables

The originator + attach + lifespan model is the foundation for
three things later in the volume and the series:

- **Property-based tests with shared resources** (Chapter 4).
  A `@given` test still uses `@attach.parameter` for non-strategy
  parameters; the property runner cycles through generated
  values for the strategy parameters while reusing the
  attached resources across iterations.
- **The `agents` CLI's config extraction** (Chapter 6).  The
  config extract reads the same originator registry to tell an
  agent what resources the suite needs.
- **The distributed runner's landscape activation** (Volume 2).
  `InteropLibraryTestJob`'s `fire_activate_*` hooks fire at
  *exactly* the same phase boundaries described here; the
  hooks just have non-trivial bodies.

## What's next

- Chapter 4 — Property-based testing.  How `@given` strategies
  compose with attached parameters; the per-test seed; the CI
  cap.
- Chapter 5 — Streaming JSOS.  Each "started" / "finished"
  event becomes a JSOS record; the format and the three output
  files.
