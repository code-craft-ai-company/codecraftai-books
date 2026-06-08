# Chapter 2 — Coding Standards as a Tool, Not a Style Guide

> *A style guide tells you how to format your code.  A coding
> standard tells you how to engineer it.  This chapter is about
> the second.*

The previous chapter named ten engineering principles.  This
chapter takes those principles and turns them into the specific
*rules* every package in the foundation tier follows.  Each rule
has a *reason*.  The reason is the chapter's central concern;
the rules without the reasons are arbitrary, and arbitrary rules
get worked around.

The canonical source of these rules is the
[`CLAUDE.md`](../../../../CLAUDE.md) file at the root of
`/Users/myron/repos/`.  The file's job is to tell automated coding
assistants what the project's conventions are; the same file
serves equally well as the human-readable reference for the
team.  This chapter unpacks each rule with the rationale and a
worked example.

## Formatting rules with engineering reasons

The first family of rules looks like style preferences but isn't.
Each one buys something specific.

### No semicolons to terminate or join lines

Python permits semicolons; the convention forbids them.  The
reason is debuggability: a line that uses semicolons to chain
statements (`x = 1; y = 2; z = x + y`) is one source-line that
hits multiple bytecode instructions.  When the debugger breaks at
that line, it stops at the first instruction; stepping doesn't
take you to the next statement on the same line — it takes you to
*the next line*.

Splitting `x = 1; y = 2; z = x + y` into three separate lines
gives the debugger three breakpoint anchors.  This is a small
example of debuggability-paramount; the cost is one newline per
statement, the benefit is one breakpoint per statement.

### `snake_case` for functions and methods

The convention is consistent with PEP 8.  The reason is *machine
readability* — when the codebase is grepped for symbol names
across packages, the convention tells the reader whether a name is
a function (`run_command`), a class (`RunCommand`), or a module
(`run_command.py`).  No string can be ambiguous about its kind.

### Type hints on every parameter and return

```python
# Required
def parse_workflow(workflow_stream: TextIOWrapper) -> Workflow:
    ...

# Forbidden
def parse_workflow(workflow_stream):
    ...
```

The discipline is universal: every parameter and every return,
including for private helpers.  The reason is twofold:

- **Static analysis** catches type mismatches before they reach
  runtime.  The IDE flags the call site that passes the wrong
  type; the linter flags the function that returns the wrong
  shape.
- **Documentation** comes for free.  A reader of an unfamiliar
  function knows what it expects and what it produces from the
  signature alone, without having to read the body.

### `self` and `cls` discipline

Instance methods take `self` as the first argument; class methods
take `cls`.  The convention is consistent with Python's standard
naming.  The reason is the *constraint satisfier* role from
chapter 1 — when the codebase is uniform on this point, a method's
binding is unambiguous from the first parameter.

## Return statement rules

The `return` discipline is one of the most distinctive parts of
the convention.  It has four specific rules.

### Every function and method ends with `return`

```python
def configure_logger(logger: Logger) -> None:
    logger.setLevel(logging.INFO)
    logger.addHandler(StreamHandler())
    return
```

The `return` is required even when the function returns `None`.
Three reasons (the chapter-1 preview develops these in detail):

1. **Debugger breakpoint anchor.**  The Python interpreter
   generates a return-bytecode whether or not the source has the
   keyword.  An explicit `return` lets the debugger attach a
   breakpoint at a *named line*, which inspects every local
   variable in the function before exit.  Without the keyword,
   the breakpoint lands on the closing brace's invisible
   bytecode location, which most debuggers handle awkwardly.
2. **Refactor safety.**  When a function is deleted (or a
   `def` line is removed during refactoring), its trailing
   `return` becomes *unreachable code* in the next function down.
   Linters flag unreachable code; the bug is caught.  Without the
   trailing `return`, the orphaned function body silently merges
   with the next function's body, and the bug ships.
3. **Function-end visibility.**  `return` at column zero
   (after dedent) is a clear delimiter — the reader knows the
   function ends here.

### Assign return values to a local variable before returning

```python
# Discouraged
def get_user_count(query: Query) -> int:
    return query.execute().scalar() + 1

# Preferred
def get_user_count(query: Query) -> int:
    raw_count = query.execute().scalar()
    user_count = raw_count + 1
    return user_count
```

The local variable serves two purposes.  At debug time, the
debugger can break on the `return` and inspect `user_count` to
see what's about to be returned.  At maintenance time, the
intermediate name documents the calculation — `raw_count + 1` is
opaque without the variable name; `user_count = raw_count + 1`
explains the offset's meaning.

### No compound `return` statements

```python
# Forbidden
def find_or_default(key: str, registry: Dict[str, Any]) -> Any:
    return registry.get(key, default_for_key(key))

# Required
def find_or_default(key: str, registry: Dict[str, Any]) -> Any:
    if key in registry:
        return registry[key]

    default_value = default_for_key(key)
    return default_value
```

The forbidden form has *two* function calls and a conditional all
on one line.  Setting a breakpoint on it gives no fine-grained
control.  The required form gives the engineer three breakpoint
anchors — the `if`, the first `return`, and the second `return` —
and makes the conditional logic explicit.

### Standalone `return` for void functions

```python
def install_logger(logger: Logger) -> None:
    logger.addHandler(StreamHandler())
    return     # not `return None`; just `return`
```

When the function returns nothing meaningful, the `return`
statement is bare.  This is a convention rather than a
correctness issue (the two forms are equivalent), but the bare
form distinguishes void functions visually from functions that
happen to return `None` as a sentinel.

## No mutable defaults

```python
# Forbidden — every call shares the same list
def add_item(item: str, items: List[str] = []) -> List[str]:
    items.append(item)
    return items

# Required
def add_item(item: str, items: Optional[List[str]] = None) -> List[str]:
    if items is None:
        items = []
    items.append(item)
    return items
```

The forbidden form has a default argument that is *evaluated
once at function definition time*.  Every call that doesn't
override `items` shares the same list — an action-at-a-distance
bug that's easy to introduce and hard to catch.

The required form takes `None` as the default and creates the
mutable inside the body.  Every call gets a fresh list when no
list is passed.  The verbosity is two extra lines; the safety is
permanent.

## Class and file naming

The portfolio's class-and-file-naming convention is opinionated:

```
# File: someclass.py
class SomeClass:
    pass
```

The class name is `PascalCase` with no underscores.  The file
name is the class name lowercased, also without underscores.  One
class per file, where the class is the file's reason for existing.

The reason: navigation.  When an engineer reads `from
ccai.taskplus.tasks.runcommand import RunCommand`, the path
`ccai/taskplus/tasks/runcommand.py` is the file's location
mechanically.  No mental translation, no searching, no
case-folding rules to remember.

The convention has consequences.  When a module needs five small
classes that belong together, they don't all live in one file —
they live in five files in the same package, with `__init__.py`
re-exporting them as needed.  This is *more* files than a less
disciplined codebase, by design; the navigation gain is worth the
file count.

## Class-variable initialisation in `__init__`

```python
class Workflow:

    def __init__(self, environment: Dict[str, str], parameters: Dict[str, Any], flow: List[FlowNode]):
        self._environment = environment
        self._parameters = parameters
        self._flow = flow
        return
```

Every instance variable is initialised in `__init__`, and *only*
in `__init__`.  Adding a `self.foo = bar` in any other method
violates the rule.

The reason: when the constructor lists every instance variable in
one place, a reader can see the class's full state in one view.
When variables are introduced in random methods, the reader has
to walk the whole class to know what state exists.  The first
pattern is *predictable*; the second is *archaeology*.

## Decision expressions

Two specific rules govern conditional expressions.

### No shortcut truthy/falsy checks

```python
# Forbidden
if some_list:
    # ...

# Required
if some_list is not None and len(some_list) > 0:
    # ...
```

The forbidden form mashes two conditions ("not None" and "not
empty") into one.  When the variable is `None`, the test is
False; when the variable is `[]`, the test is also False; when
the variable is `[0]`, the test is True.  These three cases have
different semantics and probably want different handling — but
the shortcut hides them.

The required form makes the two checks explicit.  When the
variable is `None`, the first clause short-circuits.  When the
variable is empty, the second clause does.  When the variable has
content, both pass.  Each branch can have its own error message,
its own logging, its own recovery path.

### Separate error conditions, separate handling

```python
# Forbidden
if name is None or len(name) == 0 or name in BLACKLIST:
    raise ConfigurationError("Name is invalid.")

# Required
if name is None:
    raise ConfigurationError("The 'name' parameter is required.")
if len(name) == 0:
    raise ConfigurationError("The 'name' parameter cannot be empty.")
if name in BLACKLIST:
    raise ConfigurationError("The name '%s' is blacklisted." % name)
```

The forbidden form gives the same error message regardless of
which condition triggered.  When the user receives "Name is
invalid," they don't know whether to fix a missing parameter, an
empty string, or a blacklist violation.

The required form gives a *specific* error per condition.  The
caller's mental model snaps to the right fix immediately.  This
is the *errors-never-silent* principle from chapter 1 made
operational at the validation layer.

## Decorator usage

Decorators are *only* allowed for two purposes:

1. **Adding metadata.**  `@priority(1)`, `@categorize("integration")`,
   `@deprecated(since="2.0")`.  The decorator attaches a value or
   tag to the function; the function's behaviour is unchanged.
2. **Registering with a mapping.**  `@app.route("/")`,
   `@click.command()`, `@register_handler("user_login")`.  The
   decorator adds the function to a registry; the function's body
   is otherwise untouched.

Decorators are *not* allowed for:

- **Behaviour modification.**  No `@retry`, no `@timeout`, no
  `@cache`, no `@log_calls`.  Behaviour modification belongs in
  parameters or in wrapping objects (see Volume 5's Aspects
  pattern), not in decorators.
- **Cross-cutting concerns.**  No `@requires_auth`, no
  `@validate_input`, no `@audit_log`.  These belong in explicit
  middleware or in the function's body.

The reason: behaviour-modifying decorators *hide* control flow.
A function decorated with `@retry(3)` doesn't run once when the
caller invokes it; it runs up to three times, with delays in
between, with exception swallowing built in.  The caller reading
the call site has no visual cue that any of this is happening.
The same behaviour expressed as an explicit `aspects=Aspects(retry=3)`
parameter is *visible* at the call site; the reader knows what
they're getting.

This is the *explicit-is-better-than-implicit* principle from
chapter 1 made operational at the decorator boundary.  Volume 5
develops the Aspects pattern as the affirmative answer for
behaviour-knob parameterisation.

## Documentation comments

The convention is Sphinx-style triple-quoted docstrings:

```python
def some_function(a: int, b: str) -> str:
    """
        This is a test function.

        :param a: The value for 'a'.
        :param b: The value for 'b'.

        :returns: The word 'blah' in a string.
        :raises: ValueError
    """
    if a <= 0:
        raise ValueError("'a' must be greater than 0")

    rtnval = "blah"
    return rtnval
```

The shape:

- Multi-line triple-quote, even for single-line descriptions.
- Description starts on the *second* line.  The first line is
  whitespace.
- Body indented four spaces inside the opening triple quote.
- `:param name:` for parameters.  No type in the docstring (the
  type hint carries it).
- `:returns:` for the return.  No type in the docstring.
- `:raises:` for the exception types the function can raise.

The shape produces parseable docstrings — Sphinx's `autodoc` and
the IDE's hover-help both render these correctly.  The
type-information-in-hints discipline avoids duplicating the type
in the docstring (where it would drift from the signature).

## Exception handling

The exception-handling discipline has six rules covered briefly
in chapter 1; chapter 3 develops them in full.  As a preview:

1. Don't handle exceptions you can't recover from.
2. Catch and reclassify to add specific contextual information.
3. Catch and reclassify to improve code-context in failure
   reports.
4. Catch and add extra contextual information about unique
   circumstances.
5. `SemanticError` must NEVER be caught.  Same for
   `ConfigurationError` in most contexts.
6. In all other cases, let exceptions bubble.

The discipline is *not* "exceptions are good" or "exceptions are
bad."  It is *errors that fail loudly are easier to debug than
errors that fail silently*, applied at the catch boundary.

## Multi-line error messages

Long error messages are constructed as a list of lines, then
joined:

```python
err_msg_lines = ["Failed to find expected devices."]
err_msg_lines.append("EXPECTED: ({})".format(len(expected)))
for key in expected:
    err_msg_lines.append("    {}:".format(key))

err_msg = os.linesep.join(err_msg_lines)
raise TimeoutError(err_msg) from None
```

Three reasons:

- **Cross-platform line endings.**  `os.linesep` is `\r\n` on
  Windows and `\n` on Unix.  Joining with the constant
  produces correct line endings on every platform without the
  author having to think about it.
- **Composability.**  Building the list with conditionals —
  appending different sections based on what context is
  available — is straightforward.  Building the same string with
  `+=` and embedded newlines is brittle.
- **Readability.**  The list-of-lines form makes the error
  message's *structure* visible in code: a header line, a
  count, an indented per-item block, a footer.  The string-
  concatenation form hides the structure.

## InterOp API design — the Aspects pattern

For APIs that interact with remote systems (SSH, REST,
filesystem operations), the discipline forbids behaviour-modifying
decorators (rule above) and forbids hardcoded retry/timeout
behaviour.  Instead, the API takes an `Aspects` object as a
parameter:

```python
def run_command(command: str, *, aspects: Aspects = None) -> CommandResult:
    if aspects is None:
        aspects = DEFAULT_ASPECTS
    # ... use aspects.timeout, aspects.retry_count, aspects.retry_backoff_ms ...
```

The `Aspects` object packages every behaviour parameter the API
might want — timeout, retry count, retry backoff, logging level,
sync vs async — into one scalable parameter.  Adding a new
behaviour knob is one field in the `Aspects` class; existing
callers don't need to know about it.

Volume 5 chapter 1 develops the Aspects pattern in full,
including the read-only-vs-state-modifying API design heuristic
(read-only APIs typically don't need Aspects; state-modifying
APIs typically do).  Chapter 1 of this volume previewed the
pattern; chapter 23 of Volume 3 used it for the Retry wrapper
task.  The discipline is the same across volumes.

## Properties as simple data accessors

```python
# Required
@property
def host(self) -> str:
    return self._host

# Forbidden
@property
def host(self) -> str:
    if self._host is None:
        self._host = self._connection.lookup_host()
    return self._host
```

A property is a typed read of a stored value.  It is *not* a
hook for lazy initialisation, side-effecting computation, or any
operation that might block.

The reason: properties look like attribute access at the call
site.  When `obj.host` triggers a 30-second SSH connection, the
caller has no visual cue that any work is happening.  Worse,
when `obj.host` is read inside an `__init__` of a related class,
the lazy initialisation can produce circular dependencies that
are nearly impossible to debug.

When lazy initialisation is genuinely needed, the right shape is
an explicit method (`get_host()`) with the work named.  When
caching is needed, a method called `lookup_host()` that caches
internally is fine — the call site sees a method invocation and
knows that work might be happening.

## Coding standards across volumes

The standards in this chapter apply to every volume in the
series.  Specific applications:

- **Volume 1 (Testing)** uses the `assert_*` family of
  verification functions instead of bare Python `assert`
  statements; the explicit-is-better discipline shows up in test
  assertions.
- **Volume 3 (Orchestration)** uses the no-mutable-defaults rule
  in workflow YAML loading; the `parse` classmethod doesn't have
  `Workflow=Workflow()` defaults.
- **Volume 5 (Interop)** is the affirmative-Aspects volume; the
  decorator discipline of this chapter justifies the Aspects
  pattern's existence.

When you read code in any volume of the series and find yourself
asking "why does it look like this?" — chapter 1 is the
philosophy; this chapter is the rules; the next chapter is the
error vocabulary that makes the *errors-never-silent* principle
operational.

## Summary

- The standards in this chapter are **rules with reasons**.  Each
  rule has a debuggability, refactor-safety, or
  explicit-is-better justification.  The reasons matter; rules
  without reasons get worked around.
- **Formatting rules**: no semicolons, snake_case functions, type
  hints everywhere, `self`/`cls` discipline.  Each one buys
  something specific.
- **Return statement rules**: every function ends with `return`,
  return values assigned to local variables before returning, no
  compound returns, bare `return` for void functions.  The
  rationale is debuggability + refactor safety + function-end
  visibility.
- **No mutable defaults**: prevents action-at-a-distance bugs
  from shared mutable state across calls.  Two extra lines per
  function with a default; permanent safety.
- **Class-and-file naming**: PascalCase classes; lowercase-no-
  underscores file names matching the class.  One class per file.
  More files than a less-disciplined codebase, by design.
- **No shortcut truthy/falsy checks**.  `if some_list is not None
  and len(some_list) > 0` makes both conditions explicit; each
  can have its own error message and recovery path.
- **Decorators only for metadata or registration**.  Behaviour-
  modifying decorators are forbidden; behaviour parameters live
  in `Aspects` objects.
- **Sphinx-style docstrings** on every public function, with
  `:param:` / `:returns:` / `:raises:` and no type duplication.
- **Properties as simple data accessors only**.  Lazy init,
  side-effecting computation, and blocking operations belong in
  explicit methods, not properties.
- **Multi-line error messages** built from a list joined with
  `os.linesep`.  Cross-platform, composable, readable.

The next chapter develops the error vocabulary that makes the
*errors-never-silent* principle operational.  Three special
exception types — `SemanticError`, `ConfigurationError`,
`AssertionError` — have specific must-bubble or must-catch
semantics that the rest of the portfolio depends on.
