# Chapter 3 — The Error Vocabulary

> *Errors that fail loudly are easier to debug than errors that
> fail silently.  An error vocabulary turns that principle into a
> menu of typed exceptions, each with a specific meaning and a
> specific must-bubble or must-handle status.*

The previous chapter established the coding standards.  This
chapter develops one of them in depth: the exception-handling
discipline.  The discipline rests on
[`ccai.errors`](../../../../codecraftai-foundations/source/packages/ccai/errors/exceptions.py)
— a small subpackage of `codecraftai-foundations` whose job is to
provide the **typed exception
vocabulary** every other ccai package uses.

The package's contents are simple in shape but consequential in
meaning.  Eighty exception classes; most are RuntimeError
subclasses; three are deliberately *not* — `SemanticError` and
`ConfigurationError` inherit from `BaseException` directly, and
`AbstractMethodError` from `RuntimeError`.  The inheritance
choices encode the must-bubble and must-handle decisions that
the rest of the portfolio relies on.

## The three special types

Three exception types in the vocabulary have semantics that
matter universally across the portfolio.  Every package
documents them; every package's exception handling respects
them.  This section is the canonical statement of what each
means.

### `SemanticError(BaseException)` — must always bubble

```python
class SemanticError(BaseException):
    """
        The base error object for errors that indicate that there is an issue with
        a piece of automation code and with the way the Automation Kit code is being
        utilized.
    """
```

`SemanticError` signals **API misuse**.  When code raises it, the
caller has invoked the API in a way the API can't make sense of —
passed contradictory parameters, called a method out of order,
constructed an object with mutually-exclusive options.  These are
*the caller's bugs*, and they need to surface to the caller.

The inheritance is deliberate: `BaseException`, not `Exception`.
This means `except Exception:` *does not catch*
`SemanticError`.  A bare `except Exception:` block — even a
broad recovery handler in well-meaning library code — does not
silently absorb a SemanticError.  The error bubbles up through
every Exception-targeted handler and reaches the top of the
stack, where the developer who made the API-misuse mistake sees
the failure with full context.

The discipline:

- **Code may not catch `SemanticError`.**  Ever.  The only
  legitimate handler is the *outermost* layer of the
  application, where the misuse is logged and the process exits.
- **Code may raise `SemanticError`** when the caller has
  violated the API's contract in a way the API can detect.

Concrete example.  Consider a credential class that requires
*either* a password or a keyfile but not both:

```python
class SshCredential(BaseCredential):

    def __init__(self, *, identifier: str, password: Optional[str] = None,
                 keyfile: Optional[str] = None):
        super().__init__(identifier=identifier)

        if password is not None and keyfile is not None:
            raise SemanticError(
                "SshCredential accepts password OR keyfile, not both."
            )

        if password is None and keyfile is None:
            raise SemanticError(
                "SshCredential requires one of: password, keyfile."
            )

        self._password = password
        self._keyfile = keyfile
        return
```

A caller who constructs `SshCredential(identifier="x",
password="pw", keyfile="/path")` has misused the API.  The
SemanticError fires; the constructor's caller sees the failure;
the caller fixes the misuse.  No layer of recovery code in
between gets the chance to swallow it.

### `ConfigurationError(BaseException)` — must bubble in most contexts

```python
class ConfigurationError(BaseException):
    """
        The base error object for errors that indicate that there is an issue related
        to improper configuration.
    """
```

`ConfigurationError` signals **configuration problems** — a
landscape file that's missing a required field, a credential
identifier that doesn't resolve, a runtime YAML with a
malformed entry.  Like `SemanticError`, it inherits from
`BaseException` so generic exception handlers don't silently
absorb it.

The semantics are slightly more nuanced than SemanticError's:

- **Library code may catch `ConfigurationError`** to *augment*
  the message and re-raise.  When a configuration problem
  surfaces three layers deep in a loader, the layer that knows
  about the source URI can catch the error, append its
  context, and re-raise:

  ```python
  try:
      payload = self._parse_yaml(content)
  except ConfigurationError as err:
      errmsg = "Error parsing %s: %s" % (self._source_uri, err)
      raise ConfigurationError(errmsg) from err
  ```

  The original exception is preserved as the cause (`from err`);
  the new message adds the source URI; the caller receives a
  more informative error.

- **Library code may not catch `ConfigurationError` to absorb
  it.**  When the library can't recover from the configuration
  problem, the error bubbles.  The application's top-level
  handler reports the configuration issue and exits cleanly.

The two specific subclasses in the package make finer
distinctions:

- `MissingConfigurationError` — a required field or file is
  missing.
- `InvalidConfigurationError` — a field is present but holds an
  unusable value.

Both inherit from `ConfigurationError`, so the must-bubble
semantics apply to both.  Library code that needs to react to
"missing vs invalid" specifically can catch the subclass; library
code that just wants to know "configuration is wrong" catches
the base.

### `AssertionError` — built-in, but with discipline

`AssertionError` is Python's built-in.  The convention treats it
as the *condition or state failure in the target under test* —
the check that fires when the system isn't behaving as expected.

The discipline:

- **Test code raises `AssertionError`** through the
  `verification.assert_*` family or through bare `assert`
  statements.  The error propagates to the test runner, which
  reports the failure.
- **Library code does *not* raise `AssertionError`** for
  invariant violations.  Use `SemanticError` for "the caller
  used the API wrong" or `RuntimeError` (or a more specific
  subclass) for "something unexpected happened."  Reserving
  `AssertionError` for tests keeps the test framework's
  reporting clean.

## The other categories

The package's remaining ~80 exception classes fall into seven
operational categories.  Each is a `RuntimeError` subclass (or
inherits from a more specific built-in like `TimeoutError`),
which means generic `except Exception:` handlers *do* catch
them when appropriate.

### Command and process

- **`CommandError`** — a wrapped subprocess command returned a
  non-zero exit code.  The exception carries the `status`,
  `stdout`, and `stderr` of the failed command, so the handler
  has the operational context to diagnose.
- **`UnExpectedSessionClosureError`** — an SSH or similar
  session closed without explicit close.

### Resource lifecycle

- **`CheckinError`** — a resource checkin failed.
- **`CheckoutError`** — a resource checkout failed (chapter 6 of
  Volume 6 develops this in the landscape context).
- **`PublishError`** — publishing an artifact failed.

### Flow control

- **`CancelledError`** — an operation was cancelled.  Three
  subclasses with finer meaning: `TaskingCancelled` for
  task-level cancellation, `TaskingGroupCancelled` for
  group-level.
- **`HaltError`** — a child flow-control context asks the
  parent to halt.
- **`SkipError`** — a child flow-control context (typically a
  test case) asks to be skipped.

### Tasking and looping

- **`LooperError`** — generic looper-pool error.
- **`LooperQueueShutdownError`** — work queued on a shut-down
  queue, or a worker waiting on an empty shut-down queue.
- **`TaskingGroupRuntimeError`** — runtime error inside a
  task-group execution.
- **`TaskingGroupAssertionError`** — an assertion failed inside
  a task group.  Inherits from `AssertionError` (deliberate
  exception to the "library code doesn't raise AssertionError"
  rule because tasking groups *are* test-running code).

### Override / abstract method

- **`AbstractMethodError`** — an abstract method was called
  without being overridden.  Used by abstract base classes when
  the base provides a "you must override" stub:

  ```python
  class Task(FlowNode):

      @abstractmethod
      def perform_task(self, environment, parameters, flowstate) -> int:
          raise AbstractMethodError(
              "Task->perform_task must be overloaded by derived classes."
          ) from None
  ```

- **`NotSupportedError`** — a method that exists is not
  supported in the current context (different from "must be
  overridden").
- **`UnSupportedPlatformError`** — the current platform doesn't
  support the operation (e.g., a macOS-specific call invoked on
  Linux).
- **`UnknownParameterError`** — an unknown keyword argument was
  passed.

### Timeout

Three subclasses of Python's built-in `TimeoutError`, each
naming a specific timeout shape:

- **`CompletionTimeout`** — an operation didn't complete in the
  allotted time.
- **`ConnectionTimeout`** — a connection attempt didn't succeed
  in the allotted time.
- **`InActivityTimeout`** — an operation produced no progress
  for too long (different from "didn't complete" — the operation
  may still be making progress, but in a different dimension).

### Command-line errors

A family of `CommandlineError` subclasses for argparse-style
parsing problems: `BadOptionUsageError`, `BadParameterError`,
`MissingArgumentError`, `MissingOptionError`, `NoSuchOptionError`,
`TooManyArgumentsError`.

### HTTP errors

The largest sub-family — roughly 50 classes — mirroring HTTP
status codes.  Three top-level categories
(`HttpResponse3xxError`, `HttpResponse4xxError`,
`HttpResponse5xxError`) each have specific subclasses for the
common status codes (`HttpNotFoundError`,
`HttpUnAuthorizedError`, `HttpInternalServerError`, etc.).

The discipline:

- **HTTP-aware code may catch the specific subclass** when it
  has a recovery path (retry on a 503, return cached result on
  a 404, refresh token on a 401).
- **HTTP-aware code may catch the family base** (e.g.,
  `HttpResponse5xxError`) when it wants to handle all
  server-side errors uniformly.
- **HTTP-unaware code lets them bubble.**  They are
  `HTTPException` subclasses; generic exception handlers see
  them.

The fine-grained vocabulary is what makes per-status-code
recovery practical without a regex over the message text.

## The six exception-handling rules

Chapter 1 previewed the six rules; chapter 2 mentioned them; this
chapter is the canonical statement.

### Rule 1: Don't handle exceptions you can't recover from

```python
# Forbidden
try:
    result = fetch_remote_value()
except Exception:
    pass    # silently absorbed — the result is now undefined
```

The pattern silently produces an undefined value.  When the
caller eventually uses it, the failure is far from the cause,
and the engineer debugging has no breadcrumb.  The discipline:
*don't catch what you can't handle*.  Let the exception bubble;
the caller gets to decide.

### Rule 2: Catch and reclassify to add specific contextual information

```python
# Required, when the catching layer has context the inner layer doesn't
try:
    payload = self._parse_yaml(content)
except yaml.YAMLError as err:
    errmsg = "Failed to parse landscape file %s." % self._source_path
    raise InvalidConfigurationError(errmsg) from err
```

The catching layer knows the *source path*; the inner layer
doesn't.  The catch reclassifies the generic `yaml.YAMLError`
into a specific `InvalidConfigurationError` with the path
embedded.  The original exception is preserved as `__cause__`
through `from err`.

### Rule 3: Catch and reclassify to improve code-context in failure reports

```python
# Required, when the catching layer can name *what it was trying to do*
try:
    cluster = self._build_cluster(name=name, members=members)
except KeyError as err:
    errmsg = "Building cluster %r failed: missing landscape member %s." % (name, err)
    raise ConfigurationError(errmsg) from err
```

A `KeyError` from deep inside the dispatch tree is hard to
attribute.  Reclassifying it at the layer that *named the
operation* turns the failure into a clear "I was building
cluster X and member Y was missing."

### Rule 4: Catch and add extra contextual information

The variant where the original exception type is preserved but
the *message* is enriched:

```python
try:
    self._engine.connect()
except ConnectionTimeout as err:
    err.add_note("Last successful connection: %s" % self._last_success)
    raise
```

(`add_note` is Python 3.11+; for earlier Pythons, the same
pattern uses `from` and a wrapping exception.)  The original
exception type is preserved, so callers' type-based handlers
still work; the message gains operational context.

### Rule 5: SemanticError must NEVER be caught

The discipline applies to library code, application code, and
test code uniformly.  A `try: ... except SemanticError:` block
*anywhere* is a violation.  The rationale (above): SemanticError
signals API misuse, and absorbing it lets the misuse continue
silently.

The same rule applies to `ConfigurationError` in most contexts —
the exception should reach the layer that *can act on* a
configuration problem (typically the application's top-level
handler that emits a clear error message and exits).  Library
layers may *augment* the error and re-raise (rule 2/3) but may
not absorb it.

### Rule 6: In all other cases, let exceptions bubble

The default is *no catching*.  When a function doesn't have a
specific recovery path, it doesn't have a `try`/`except`.  The
exception travels up the stack; some higher layer either has a
recovery or surfaces the failure to the user.

This is the *errors-never-silent* principle in action.  A
codebase whose default is "catch and recover" accumulates
recovery points that don't actually recover; the failures get
absorbed; the bugs ship.  A codebase whose default is "let it
bubble" produces clear failures that the engineer can diagnose.

## Why `BaseException` for SemanticError and ConfigurationError

Python's exception hierarchy:

```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── ArithmeticError, LookupError, OSError, RuntimeError, ValueError, ...
    └── (most user-defined exceptions)
```

The convention is that `Exception` is for application errors
that *can* be caught; `BaseException` is for system-level events
(`SystemExit`, `KeyboardInterrupt`) that *should not* be caught
by generic `except Exception:` handlers.

By inheriting `SemanticError` and `ConfigurationError` from
`BaseException` directly, the package opts them into the same
"don't catch generically" treatment.  A library wrapper with
`except Exception:` for legitimate recovery does *not* swallow a
SemanticError — the caller's API misuse still surfaces.

This is one of the portfolio's most consequential design
choices.  Most third-party libraries put their custom exceptions
in the Exception subtree; the ccai portfolio deliberately
lifts the must-bubble ones out.  It's a small choice in the
class definition, with a large consequence at every catch site
across every package.

## The vocabulary's role across the portfolio

Every other package in the foundation tier uses
`ccai.errors` for its exception vocabulary.  Specific uses:

- **Volume 0 chapter 4 (Interfaces)** uses
  `AbstractMethodError` to signal that an abstract method's stub
  was hit because no subclass overrode it.
- **Volume 0 chapter 5 (Waiting)** uses `CompletionTimeout`,
  `ConnectionTimeout`, `InActivityTimeout` for the three
  timeout shapes.
- **Volume 0 chapter 9 (Configuration loading)** uses
  `ConfigurationError`, `MissingConfigurationError`,
  `InvalidConfigurationError` for the load-time validation.
- **Volume 3 (every chapter)** uses the vocabulary uniformly for
  both library-side and application-side error reporting.

The vocabulary is *what makes the portfolio interoperate*.  When
two packages from different tiers want to talk about errors, they
talk in the shared language.  A landscape error in Volume 6 that
surfaces during a Volume 3 orchestration run is the same
exception type both packages understand.

## Summary

- **`ccai.errors`** is a small subpackage of `codecraftai-foundations` with ~80
  exception classes organised into seven operational categories
  plus three *special* types whose semantics matter universally.
- **`SemanticError(BaseException)`** signals API misuse.  Must
  always bubble.  Library code may not catch.  Inherits from
  `BaseException` so generic `except Exception:` does not
  swallow it.
- **`ConfigurationError(BaseException)`** signals configuration
  problems.  Library code may catch only to *augment* the
  message and re-raise.  Same `BaseException` inheritance for
  the same reason.
- **`AssertionError`** is Python's built-in.  The convention
  treats it as the test-framework's signal; library code does
  not raise it.
- The other 80 exceptions are organised by operational category:
  command/process, resource lifecycle, flow control, tasking,
  override/abstract, timeout, command-line, HTTP.  Each is a
  `RuntimeError` (or specific-built-in) subclass that *can* be
  caught when appropriate.
- The **six exception-handling rules** govern when to catch and
  when to bubble:
  1. Don't catch what you can't handle.
  2. Catch to add contextual information.
  3. Catch to improve code-context in reports.
  4. Catch to add operational context to existing errors.
  5. Never catch `SemanticError`.
  6. Default to letting exceptions bubble.
- The **`BaseException` choice** for `SemanticError` and
  `ConfigurationError` is the portfolio's most consequential
  exception-handling decision.  It's small in the class
  definition; large at every catch site.
- The vocabulary is what makes packages **interoperate**.  Two
  packages from different tiers talk about errors in the shared
  language.

The next chapter introduces interfaces and contracts — the
abstract-base-class machinery that lets packages declare what
implementers must provide without specifying how.
