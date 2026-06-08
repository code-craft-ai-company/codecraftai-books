# Chapter 4 — Interfaces and Contracts

> *An interface is a promise about behaviour without a commitment
> to implementation.  The promise is the contract; the contract
> is what makes packages substitutable.*

The previous chapter named the typed exception vocabulary that
makes packages talk about errors in the same language.  This
chapter names the typed *interface* vocabulary that makes
packages substitute for each other without their callers
caring which implementation is on the other end.

The package is
[`codecraftai-interfaces`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/).
Eight files; eight protocols; one purpose: **shared contract
definitions that downstream packages implement and other
downstream packages consume.**  The package itself contains no
behaviour — only the typed shape of the behaviour.

## Why protocols, not abstract base classes

Python has two main mechanisms for declaring interfaces:

- **Abstract base classes** (`abc.ABC` with `@abstractmethod`) —
  the implementer *inherits* from the ABC.  Type checkers verify
  the inheritance.  Runtime instance checks (`isinstance(x,
  MyABC)`) are accurate.
- **Protocols** (`typing.Protocol`) — structural typing.  The
  implementer matches the protocol if it provides the required
  methods, regardless of inheritance.  Type checkers verify the
  shape; runtime instance checks require `runtime_checkable`.

The portfolio's interfaces use **`Protocol`** uniformly.  Two
reasons:

1. **Decoupling.**  An implementer that already inherits from
   another base (or that lives in a package that doesn't depend
   on `codecraftai-interfaces` at the inheritance level) can still
   *be* an `ICommandRunner` if it has the right method shape.
   No forced inheritance; no diamond-inheritance problems.
2. **Substitutability.**  Test mocks and fakes don't need to
   inherit from the interface to type-check as it.  A test
   builds a small class with the right methods and passes it
   wherever `ICommandRunner` is expected.

The cost of protocols: less rich runtime introspection.  When
the application needs *runtime* enforcement of an interface
(rare in the portfolio), abstract base classes are the right
choice.  When the application needs *structural* checks at the
type-checker level, protocols are.  The portfolio uses protocols
because the structural-check use case dominates.

## The eight protocols

The shipped package has eight protocol files.  This section
walks each, anchored on the actual code.

### `ICommandRunner` — running commands across system boundaries

[`icommandrunner.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/icommandrunner.py)
defines the most-used protocol in the portfolio.  Its job: provide
a uniform shape for *running a command*, regardless of whether
the command runs locally, over SSH, in a container, or via some
other transport.

```python
@dataclass
class RunResult:
    exit_code: Optional[int]
    stdout: str
    stderr: str
    timed_out: bool = False


class ICommandRunner(Protocol):
    def run(
        self,
        args: List[str],
        *,
        elevated: bool = False,
        cwd: Optional[str] = None,
        env: Optional[Dict[str, str]] = None,
        stdin: Optional[str] = None,
        timeout: Optional[float] = None,
        on_output: Optional[Callable[[str], None]] = None,
        on_error: Optional[Callable[[str], None]] = None,
        capture_output: bool = True,
    ) -> RunResult: ...
```

Eight keyword-only parameters.  The *positional* parameter is the
command (`args` as a list, not a string — explicit-is-better and
shell-injection-safer).  The keyword-only parameters are
*aspects* of execution: elevated privilege, working directory,
environment, stdin, timeout, output streaming callbacks, capture
flag.

The shape is consistent across implementations:

- A *local* `LocalCommandRunner` calls `subprocess.run` with
  the args.
- A *remote* `SshCommandRunner` packages the args into an SSH
  command and runs them on the remote host.
- A *containerised* `DockerCommandRunner` calls `docker exec`.
- A *test mock* `FakeCommandRunner` returns canned `RunResult`
  values without running anything.

Code that needs to "run a command" takes an `ICommandRunner`
parameter and gets to work without knowing which runner it
received.

### `IFileOperator` — filesystem operations across boundaries

[`ifileoperator.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/ifileoperator.py)
is `ICommandRunner`'s sibling for filesystem work.  Five methods:
`write_text`, `delete`, `read_text`, `list_dir`, `exists`.  Each
takes an `elevated: bool` parameter, mirroring the privilege
discipline from the runner.

```python
class IFileOperator(Protocol):
    def write_text(self, path: str, content: str, elevated: bool = False) -> None: ...
    def delete(self, path: str, elevated: bool = False) -> None: ...
    def read_text(self, path: str, elevated: bool = False) -> str: ...
    def list_dir(self, path: str, elevated: bool = False) -> List[str]: ...
    def exists(self, path: str, elevated: bool = False) -> bool: ...
```

Same substitutability: a `LocalFileOperator` works against the
local filesystem; a `SshFileOperator` works against a remote
filesystem over SSH; a `S3FileOperator` works against S3-compatible
object storage.  The caller doesn't care.

The minimalism is deliberate.  Files have many possible
operations (set permissions, change owner, create symlinks, copy
ranges, append, truncate); the interface picks the **five most
universally needed** and stops there.  Implementers can offer
*more* on their concrete classes; the interface only promises
these five.

### `ISerializable` — dict round-trip

[`iserializable.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/iserializable.py)
is the smallest protocol in the package.  Two methods:

```python
class ISerializable(Protocol):

    @classmethod
    def from_dict(cls, data: Dict[str, Any]): ...

    def as_dict(self) -> Dict[str, Any]: ...
```

`as_dict` produces a dictionary representation; `from_dict`
constructs an instance from one.  The contract is round-trip:
`obj == obj.from_dict(obj.as_dict())` should hold for any
serialisable object.

Code that needs to serialise heterogeneous objects (write to
JSON, write to YAML, send over the wire to another process)
takes `ISerializable` instances and gets to work.  Code that
needs to *consume* a configuration document constructs the
right concrete class through `from_dict`.

The chapter-9 configuration loader uses this pattern: every
typed configuration class implements `ISerializable`; the
loader calls `from_dict` to construct instances from parsed
YAML/JSON.

### `IBasicCredential` — username + password

[`ibasiccredential.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/ibasiccredential.py)
is the smallest credential interface.  Two read-only properties:

```python
class IBasicCredential(Protocol):
    @property
    def username(self) -> str: ...

    @property
    def password(self) -> str: ...
```

Used by code that consumes basic credentials without caring
*which* credential class produced them.  A
[`CouchDbProfile.connection_string(credobj: IBasicCredential)`](../../../../codecraftai-dataprofiles/source/packages/ccai/dataprofiles/couchdbprofile.py)
takes any object with `.username` and `.password` properties —
a real `BasicCredential` from `codecraftai-credentials`, a test
`FakeBasicCredential`, an inherited `SshCredential` that has
both fields, etc.

The credential package's `BasicCredential` *implements* this
protocol; `CouchDbProfile` *consumes* it.  The implementing
package doesn't depend on the consuming package; the consuming
package doesn't depend on the implementing package; both depend
on the interfaces package.  This is the dependency-inversion
discipline in action.

### `IIncludeFilter` and `IExcludeFilter` — list filtering

[`iincludefilter.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/iincludefilter.py)
and
[`iexcludefilter.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/iexcludefilter.py)
provide a uniform interface for filtering lists of items.  They
are siblings:

```python
class IIncludeFilter(Protocol):
    def should_include(self, check_object: Any) -> bool: ...
    def filter(self, input_list: List[Any]) -> List[Any]: ...

class IExcludeFilter(Protocol):
    def should_exclude(self, check_object: Any) -> bool: ...
    def filter(self, input_list: List[Any]) -> List[Any]: ...
```

The pair separates *include* and *exclude* semantics
deliberately.  An include filter says "keep these"; an exclude
filter says "drop these."  Composing the two — running both
against the same list — gives a uniform "include unless
excluded" pattern that works without conditional flags.

Both protocols ship a `filter(input_list)` *default* implementation
that walks the list and applies `should_include` /
`should_exclude` to each item.  Implementers only need to
override `should_include` (or `should_exclude`); the list walk
is provided.

This is one of the few places where the protocol carries a
default implementation.  It works because Python's `Protocol` is
permissive about whether members are defined; the structural
check passes if the implementer either overrides or inherits the
default.

### `IHint` — friendly identification

[`ihint.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/ihint.py)
is a small protocol for objects that can produce a "hint" string
matching them against another object:

```python
class IHint(Protocol):
    def get_hint(self) -> str: ...
    def matches(self, obj: Any) -> bool: ...
```

Used by `codecraftai-landscaping`'s
[`FriendlyIdentifier`](../../../../codecraftai-landscaping/source/packages/ccai/landscaping/friendlyidentifier.py)
class.  A landscape's apod entries carry friendly names ("dev01",
"runner-east-1") that match the actual host name through hint
strings.  The protocol formalises the relationship.

### `ITestJob` — the test-job contract

[`itestjob.py`](../../../../codecraftai-interfaces/source/packages/ccai/interfaces/itestjob.py)
is the largest of the eight protocols.  It defines the contract
for a test job — the unit of work `codecraftai-testbase` and its
extensions execute:

```python
class ITestJob(Protocol):
    @property
    def importerrors(self): ...
    def begin(self) -> None: ...
    def execute(self) -> int: ...
    def finalize(self) -> None: ...
    def query(self, query: str, format: int=0) -> List[Any]: ...
```

Five methods plus one property.  The lifecycle is `begin` →
`execute` → `finalize`; the property surfaces import errors
that occurred during the job's load; the `query` method exposes
introspection for tooling.

The protocol is the contract Volume 2's distributed test jobs
implement (`TestJob` → `InteropLibraryTestJob` →
`DistributedTestJob`).  Each layer of the inheritance chain
satisfies the protocol; consumers (test runners, reporters,
schedulers) talk to whichever layer of the chain they have.

## How the interfaces compose

The eight protocols are not independent — they compose into the
larger contracts the higher tiers of the portfolio rely on.

### Composition example: a sysman feature

A Volume 4 sysman feature operates on a per-host basis.  Each
host is represented by a `HostContext` that carries (among
other things):

- An `ICommandRunner` — to run commands on the host.
- An `IFileOperator` — to read and write files on the host.
- An `IBasicCredential` — to authenticate to the host.

The sysman feature's code takes a `HostContext` and uses the
three interfaces.  The same feature works against:

- A *local* host (LocalCommandRunner + LocalFileOperator + a
  null credential).
- A *remote* host over SSH (SshCommandRunner + SshFileOperator +
  SshCredential).
- A *containerised* host (DockerCommandRunner +
  DockerFileOperator + a basic credential).
- A *mocked* host in unit tests (FakeCommandRunner +
  InMemoryFileOperator + FakeCredential).

The feature's source code is identical in every case.  The
substitutability is what makes the cross-OS contract layer in
Volume 4 work — the same feature implementation runs against
every host because the per-host *machinery* is interface-shaped.

### Composition example: a test runner

A test runner consumes:

- An `ITestJob` — the job to run.
- An `IIncludeFilter` and `IExcludeFilter` — to scope which
  tests within the job to execute.
- An `ISerializable` — to serialise results.

The runner's code takes the four interfaces and runs.  The same
runner works against single-host jobs (`TestJob`),
landscape-aware jobs (`InteropLibraryTestJob`), and distributed
jobs (`DistributedTestJob`) — because all three implement
`ITestJob`.

## What the interfaces do *not* do

Three deliberate non-features.

### Not runtime-enforced (typically)

The protocols are *type-checker enforced*, not runtime-enforced.
A Python program can pass a non-implementer where an
`ICommandRunner` is expected; the type checker complains; the
runtime doesn't.  When *runtime* enforcement is needed, the
consuming code does an explicit `isinstance` check (with the
protocol marked `@runtime_checkable`) or — more often — does
duck-typing checks (`hasattr(runner, 'run')`).

This is consistent with Python's structural-typing tradition
and with the philosophy's *explicit-is-better-than-implicit*
principle.  Runtime enforcement of an interface is *additional*
rigour that costs runtime cycles; the type checker catches the
common case at zero runtime cost.

### Not behaviour, just shape

The protocols describe *what methods exist with what
signatures*; they do not describe *what those methods do*.  A
malicious `ICommandRunner` implementation that returns random
data instead of running the command satisfies the type
contract; the runtime check would still pass; the bug is in the
implementer's behaviour.

The portfolio mitigates this through testing — each concrete
implementation has its own test suite that verifies the
*behaviour* matches the documented contract.  The interface
package doesn't do that work; it shouldn't.

### Not capability discovery

The interfaces don't declare what *capabilities* an
implementation has beyond the methods it implements.  A
filesystem implementation that supports symlinks doesn't have a
"supports_symlinks" flag on the interface; a runner that
streams output doesn't have a "supports_streaming" flag.

When a consumer needs capability discovery, the answer is *a
separate capability protocol*, not feature flags on a base
protocol.  This keeps the base protocols small and the
discovery logic explicit.

## How interfaces drive the dependency graph

The `codecraftai-interfaces` package sits at the bottom of the
portfolio's dependency graph.  Specifically:

```
                    codecraftai-interfaces  (this package)
                            ▲
                            │ imported by everyone
                            │
           ┌────────────────┼─────────────────┐
           │                │                 │
   codecraftai-credentials   codecraftai-       codecraftai-landscaping
           │              dataprofiles            │
           │                │                     │
           └────────────────┼─────────────────────┘
                            │
                            ▼
                codecraftai-interop, sysman, ...
```

Two important properties of this graph:

- **`codecraftai-interfaces` depends on nothing.**  Not even on
  `codecraftai-errors`.  It's the leaf; it can be installed
  standalone.
- **Every package that wants to *interoperate* depends on
  `codecraftai-interfaces`.**  The dependency is small (eight
  small files); the value is large (every package speaks the
  shared contract language).

When a package introduces a new interface, the right place is
*here*, not in the introducing package's own surface.  This
keeps the interfaces *centralised*; consumers only need to
import from one place.

## When to add a new interface

Three signals that a new interface is the right move:

### Signal 1: two implementations, one consumer

When a consuming package has *one shape it wants to consume* and
two or more implementing packages produce that shape, the shape
is an interface.  The interface goes in `codecraftai-interfaces`;
both implementers depend on the interfaces package; the consumer
depends only on the interfaces package.

### Signal 2: testing requires a substitute

When unit testing a package would benefit from a fake of some
external dependency, the dependency's shape is a candidate
interface.  Testing the sysman feature without spawning real SSH
sessions requires a fake `ICommandRunner`; making the fake easy
to write requires `ICommandRunner` to be a protocol.

### Signal 3: cross-tier substitutability

When a higher-tier package wants to plug a different
implementation in for a lower-tier dependency (chapter-7's
extension and factory machinery is the substantive form of this
pattern), the seam is an interface.

The package's `codecraftai-interfaces` deliberately stays *small*.
Eight files today; growth is deliberate.  Each new interface
goes through the question: *do at least two existing or planned
implementations want to satisfy this shape?*  If yes, add it; if
no, don't add the interface yet.

## Summary

- **`codecraftai-interfaces`** is the leaf of the portfolio's
  dependency graph.  Eight protocol files; one purpose: shared
  contract definitions for substitutable behaviour.
- The protocols are **`typing.Protocol`** (structural typing),
  not `abc.ABC` (nominal typing).  Implementers don't inherit;
  they match by shape.  The cost is less runtime introspection;
  the benefit is decoupling and easy mocking.
- The eight protocols:
  - **`ICommandRunner`** — running commands across local,
    remote, containerised boundaries.
  - **`IFileOperator`** — filesystem operations with the same
    cross-boundary uniformity.
  - **`ISerializable`** — dict round-trip via `from_dict` and
    `as_dict`.
  - **`IBasicCredential`** — username + password read-only
    properties.
  - **`IIncludeFilter`** / **`IExcludeFilter`** — uniform list
    filtering with default implementations of `filter()`.
  - **`IHint`** — friendly identification.
  - **`ITestJob`** — the test-job lifecycle contract used by
    Volume 1 and Volume 2.
- The interfaces **compose** into larger contracts.  A sysman
  feature consumes `ICommandRunner` + `IFileOperator` +
  `IBasicCredential` to operate on a host — local, remote,
  containerised, or mocked, all with the same code.
- The protocols describe **shape, not behaviour**.  Behaviour
  conformance is verified through each concrete implementation's
  test suite; the interface package itself contains no tests of
  behaviour.
- New interfaces earn their place by satisfying **at least two
  existing or planned implementations**.  The package stays
  small deliberately; growth is gated.

The next chapter introduces the waiting primitives — small
helpers for the recurring problem of "wait for a condition" that
turn the trivial loop into a debugged, observable, timeout-aware
operation.
