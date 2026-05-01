# Chapter 7 — Mocking and Stubbing Without the Framework Getting in the Way

> *Mocks exist for one reason: the system you'd have to summon
> to test the unit under test is too expensive, too slow, or
> too unavailable.  Once you've made the choice to mock, the
> framework should help — not impose its own opinions about
> what mocks should look like.  This chapter is about how
> testbase's mocking helpers stay out of the way.*

`automojo-testbase` ships its own mocking module rather than
wrapping `unittest.mock`.  The choice is deliberate: the
portfolio's mocking needs don't fit neatly under the
unittest.mock shape, and rather than fight it, testbase ships
primitives shaped for the actual cases.

This chapter walks the four primitives that make up the
mocking surface — `MockType`, `SpecViolationError`,
`SideEffect`, and the `Mock` class itself — and discusses when
to mock at all.  The chapter intentionally does *not* try to
re-derive the entire `unittest.mock` API; it walks the parts
testbase added because unittest.mock alone wasn't enough, and
the parts testbase made simpler because unittest.mock did
more.

## When to mock at all

Before talking about the API, the choice.  The portfolio has
three rules of thumb that drive whether a test mocks:

1. **Mock at the boundary, not in the middle.**  If the test
   verifies behaviour A → B → C, and only C touches the
   external system, mock only C.  Mocking B is a sign the test
   is testing too much.
2. **Don't mock what you don't own.**  Mocking the standard
   library, the `requests` HTTP client, or `datetime.now` is
   usually a code smell.  Wrap the dependency in a class you
   own and mock the class.
3. **An integration test that mocks the integration is not
   doing its job.**  If the test is for the integration with
   service X, the test must talk to a service-X-shaped
   thing — at minimum a fake-X-server, ideally service X
   itself.  A mock of X tests nothing X-specific.

These rules come from Volume 0 chapter 1's debuggability
principle: a mock that lies hides bugs from the test.  The
framework's job is to make the *legitimate* mocks easy and the
*illegitimate* mocks at least uncomfortable.  Spec violation
errors (below) are how testbase makes the second part work.

## `MockType`: knowing what kind of mock you have

```python
# automojo-testbase/source/packages/automojo/testbase/mocking.py:46–56

class MockType(IntEnum):
    Generic = 0
    Class = 1
    Instance = 2
    Property = 3
    Function = 4
    Method = 5
```

Every testbase `Mock` carries a `MockType`.  The type is what
the mock *represents*:

- `Generic` — a free-floating mock with no spec.  Use sparingly.
- `Class` — a mock that stands in for a class (calls to it
  return mocks shaped like instances).
- `Instance` — a mock that stands in for an instance of a
  class.
- `Property` — a mock that stands in for a property
  (`@property` / descriptor).
- `Function` — a mock that stands in for a free function.
- `Method` — a mock that stands in for a method (the first
  argument is `self`).

The type matters because testbase's spec validation behaves
differently for each kind.  A `Class` mock allows attribute
access for any class member; an `Instance` mock allows
attribute access only for instance members; a `Method` mock
expects to be called with `self` as the first argument.

A consumer who picks the wrong type gets a
`SpecViolationError` (below) at the first wrong access — not
a mysterious `AttributeError` later when the mock returns
something it shouldn't have.

## `SpecViolationError`: refusing to play along

```python
# automojo-testbase/source/packages/automojo/testbase/mocking.py:25–43

class SpecViolationError(BaseException):
    """
        Raised when a mock is used in a way that violates its spec.

        A spec violation is a programming error in the test, not a
        runtime condition the test can recover from.  The error
        inherits from BaseException so that a `try / except Exception`
        in the code under test cannot accidentally swallow it.
    """
    pass
```

`SpecViolationError` inherits from `BaseException` — the same
discipline as `SemanticError` and `ConfigurationError` from
Volume 0 chapter 3.  An ordinary `try / except Exception` does
not catch it.  This matters because a mock is, by definition,
inserted into code that is being tested; the code may have
broad exception handlers that would swallow a normal exception
and let the test pass spuriously.  By inheriting from
`BaseException`, a spec violation forces the test to fail
loudly.

A spec violation happens when:

- A `MockType.Instance` mock is accessed for a member the
  spec class does not declare.
- A `MockType.Method` mock is called without `self`.
- A `MockType.Function` mock is called with the wrong number
  of arguments and signature checking is enabled.
- A `MockType.Class` mock is used as if it were an instance
  (or vice versa).

The principle: if the test is wrong about *what shape of
thing* it is mocking, the framework refuses to play along.

## `SideEffect`: the simplest mock behaviour

For mocks whose only job is to return a value or run a
callable, testbase ships `SideEffect`:

```python
# automojo-testbase/source/packages/automojo/testbase/mocking.py:59–120

class SideEffect:
    def __init__(self,
                 callable: Callable,
                 default_args: Optional[Union[List[Any], Tuple[Any, ...]]] = None,
                 default_kwargs: Optional[Dict[str, Any]] = None) -> None:
        self._callable = callable
        self._default_args = default_args if default_args is not None else ()
        self._default_kwargs = default_kwargs if default_kwargs is not None else {}
        return

    def __call__(self, *args, **kwargs) -> Any:
        merged_args = self._default_args + tuple(args)
        merged_kwargs = {**self._default_kwargs, **kwargs}
        rtnval = self._callable(*merged_args, **merged_kwargs)
        return rtnval
```

Three things to notice:

1. The default args and kwargs are stored as instance
   variables in `__init__` — Volume 0 chapter 2's discipline
   on instance variables.
2. The mutable defaults trap (Volume 0 chapter 2) is avoided
   by using `None` as the default and assigning the empty
   tuple / dict inside.
3. The merge order matters: caller arguments *override* the
   defaults.  A test sets up a `SideEffect(lambda x: x*2,
   default_args=(5,))` and the call `effect()` returns 10;
   `effect(7)` returns 14.

`SideEffect` is the building block for "this mock returns X"
or "this mock runs Y when called."  Larger mock behaviours
compose `SideEffect` instances.

## `Mock` and `MockContext`

The two top-level mocking constructs:

- **`Mock`** (`automojo-testbase/source/packages/automojo/testbase/mocking.py:1603`)
  — the spec-aware mock class itself.  Constructed with a
  spec (a class, an instance, or a function), it produces
  attribute access and call behaviour shaped by the spec.
- **`MockContext`** (line 304) — a context manager that
  sets up a group of mocks, optionally patches names in a
  module, and tears the patches down on exit.

A representative usage:

```python
import testbase
from testbase.mocking import Mock, MockContext, MockType
from automojo.billingkit.invoice import InvoiceService

@testbase.attach.parameter(some_db_session, identifier="db")
def test_invoice_service_uses_billing_runtime(db):
    """
        Verify InvoiceService consults the billing runtime.
    """
    runtime = Mock(spec=BillingRuntime, mock_type=MockType.Instance)
    runtime.tax_rate = SideEffect(lambda: Decimal("0.0875"))

    with MockContext() as ctx:
        ctx.patch("automojo.billingkit.invoice.get_runtime",
                  Mock(spec_class=BillingRuntime,
                       mock_type=MockType.Function,
                       side_effect=SideEffect(lambda: runtime)))

        service = InvoiceService(db=db)
        invoice = service.create_invoice(...)
        rtnval = invoice.tax_amount > Decimal("0")

    assert rtnval
    return
```

The `Mock(spec=BillingRuntime, ...)` constructor produces a
mock whose attribute access is restricted to members of
`BillingRuntime`.  Setting `runtime.tax_rate = SideEffect(...)`
binds a callable behaviour to the attribute.  The
`MockContext` patches `get_runtime` in the module under test
for the duration of the `with` block; on exit, the patch is
unwound automatically.

The pattern reads similarly to `unittest.mock.patch` but with
testbase-specific specifics: spec validation is on by default,
spec-violation errors bubble unstoppably, and the `MockType`
disambiguates what kind of thing each mock is.

## What testbase mocking does *not* do

A short list of unittest.mock conveniences testbase deliberately
omits:

- **`MagicMock`'s automatic dunder support.**  Testbase's
  `Mock` does *not* auto-magic `__len__`, `__iter__`,
  `__getitem__`, etc.  If you need a mock that supports
  iteration, you set up the dunder explicitly with a
  `SideEffect`.  The reasoning: an auto-magic iter that
  silently returns an empty iterator hides bugs more often
  than it helps.
- **String-based attribute lookup at construction time.**
  unittest.mock lets you write `mock.foo.bar.baz()` and
  silently invents the chain.  Testbase's spec-aware `Mock`
  refuses unknown attributes; the chain has to be set up
  with explicit attributes that have explicit specs.
- **Implicit return values.**  A testbase `Mock` of
  `MockType.Function` raises `SpecViolationError` if called
  before a `SideEffect` is bound.  The framework refuses to
  invent a return value the test author did not declare.

These are not small differences.  A test author migrating from
unittest.mock will hit each of them.  The framework's
philosophy from Volume 0 chapter 1 — that explicit beats
implicit, even at the cost of more typing — drives every one.

## Mocking and the resource factory model

Mocks fit into the resource factory model from Chapter 3 in
two ways:

### Mock-yielding originators

A factory can yield a mock instead of a real resource:

```python
@testbase.factories.resource_factory(scope=ResourceLifespan.Test)
def billing_runtime_mock():
    """
        Yields a Test-scoped mock of BillingRuntime for unit tests
        that don't need the real runtime.
    """
    runtime = Mock(spec=BillingRuntime, mock_type=MockType.Instance)
    runtime.tax_rate = SideEffect(lambda: Decimal("0.0875"))
    yield runtime
    return
```

A test attaches the mock-yielding factory exactly as it would
attach a real one:

```python
@testbase.attach.parameter(billing_runtime_mock, identifier="runtime")
def test_invoice_total(runtime):
    ...
```

The advantage: the test does not know it has a mock.  Swapping
to a real `BillingRuntime` later means changing the factory,
not the test.

### `MockContext` for short-lived patches

For patches that should not outlive a single test,
`MockContext` is the right shape.  The factory model is for
resources whose lifespan matches one of the standard
(Session / Package / Test) scopes; the `MockContext` is for
patches that need to be torn down at a *finer* scope —
mid-test, around a specific code path.

The two compose: a Test-scoped factory provides the standing
mock; a `MockContext` inside the test patches one specific
name for one specific code path.

## Mocks in JSOS records

Mocks do not produce special JSOS records.  A test that runs
against a mock produces the same JSOS shape (Chapter 5) as a
test that runs against a real resource.  This is intentional:
the agents CLI (Chapter 6) treats mock-running tests
identically to real-resource tests.  The fact that a test was
mocked is information for the test author and the test's
maintainer; the framework's record of the run does not need
to know.

If a build engineer wants to track which tests use mocks, the
convention is to attach a moniker — `@testbase.moniker("unit")`
on tests that mock everything, `@testbase.moniker("integration")`
on tests that talk to real resources.  The JSOS record
carries the monikers (Chapter 5), and the agents CLI can
filter on them.

## A worked anti-example

A mock that hides a bug:

```python
# Bad: this test passes because the mock pretends success.

def test_invoice_creation_emits_audit_log():
    audit_log = Mock(spec=AuditLog, mock_type=MockType.Instance)
    audit_log.emit = SideEffect(lambda *args, **kwargs: None)

    service = InvoiceService(audit_log=audit_log)
    service.create_invoice(...)

    rtnval = True  # The test asserts nothing useful.
    assert rtnval
    return
```

The test "passes" but it does not verify the property the
test name claims.  The audit log was mocked; the call to
`emit` was made; the test does not check that the call
happened.  A reviewer should reject this test.

The fix:

```python
def test_invoice_creation_emits_audit_log():
    """
        Verify that creating an invoice records an entry on the
        audit log.
    """
    audit_log = Mock(spec=AuditLog, mock_type=MockType.Instance)
    audit_log.emit = SideEffect(lambda *args, **kwargs: None)

    service = InvoiceService(audit_log=audit_log)
    service.create_invoice(...)

    rtnval = audit_log.emit.call_count > 0
    assert rtnval
    return
```

Now the test verifies the property the name claims.  The
`call_count` attribute is one of the spec-aware behaviours
testbase mocks expose for verification — the framework's
counterpart to unittest.mock's call-tracking facilities.

## What's next

- Chapter 8 — `arcli`.  The CLI toolkit underneath every
  testbase subcommand, including the `agents` group.
- Chapter 9 — Single-host landscape concepts.  How to write a
  test that uses a *real* resource selectively, with the
  right hooks for Volume 2's distributed runner to extend.
