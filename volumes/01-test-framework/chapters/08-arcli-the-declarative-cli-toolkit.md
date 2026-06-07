# Chapter 8 — `arcli`: The Declarative CLI Toolkit

> *Every package in the portfolio that ships a CLI ships
> the same CLI: same decorator surface, same flag parsing,
> same `--help` rendering, same error reporting.  The package
> that makes that uniformity possible is `codecraftai-arcli`.
> This chapter is about how it works and why each design
> choice was the right one.*

`codecraftai-arcli` is the declarative CLI toolkit underneath
every `codecraftai-*` command-line surface: `testbase`,
`taskplus`, `configuration`, `arcli` itself, and a dozen
others.  Reading `arcli` is the prerequisite for reading any
of those CLIs; once you can read `arcli`, every CLI in the
portfolio reads the same way.

This chapter walks the four declarative pieces — `Command`,
`GroupingCommand`, `ArgField`, `OptionField` — plus the
parameter-type system, the help-text generator, and the
validator-and-parser hardening from PR #13.

## What `arcli` is for

A CLI in the modern Python world has plenty of options:
`argparse`, `click`, `typer`, `cleo`, hand-rolled parsers
on top of `sys.argv`.  Each has a position.  `arcli`
exists because the portfolio needed three properties from
its CLI toolkit that no off-the-shelf option offered all of:

1. **Decorator-driven declaration.**  A command is a
   function with decorators that declare its arguments,
   options, and group membership.  No subclassing.
2. **First-class command trees.**  A CLI is a tree of
   groups and commands, not a flat list.  The toolkit must
   make tree assembly idiomatic — not a bolt-on after the
   fact.
3. **Validator composition that works for list-mode
   options.**  An option that can be repeated (`--filter
   foo --filter bar --filter baz`) must validate each
   element.  Click and typer have nuances here that the
   portfolio's validator-and-parser hardening (PR #13)
   addressed at the framework level.

The chapter assumes the reader has used `argparse` or
`click` enough to recognise the patterns; everything that
follows is `arcli`'s take.

## A minimal `arcli` command

```python
import arcli

@arcli.option("--name", required=True, help="Greeting target.")
@arcli.command("greet")
def cmd_greet(name: str):
    """
        Print a greeting to the named target.
    """
    print(f"Hello, {name}!")
    return
```

The decorators apply bottom-up:

1. `@arcli.command("greet")` turns `cmd_greet` into a
   `Command` object whose name is `"greet"` and whose
   callable is the original function.
2. `@arcli.option("--name", ...)` attaches an `OptionField`
   to the command.

The command can be registered into a tree:

```python
my_root.add_command(cmd_greet)
```

And invoked from the shell:

```bash
my_root greet --name Alice
# → Hello, Alice!
```

That is the whole shape.  Everything else in this chapter
is *what those decorators construct* and *how the runtime
parser walks them*.

## The `Command` class

```python
# codecraftai-arcli/source/packages/ccai/arcli/model/command.py:22–61

class Command:
    def __init__(self, name: str, callable: Callable) -> None:
        self._name = name
        self._callable = callable
        self._flags_short = {}
        self._flags_long = {}
        self._traps = []
        self._options = collections.OrderedDict()
        self._arguments = []
        return
```

Seven instance variables, all initialised in `__init__` per
Volume 0 chapter 2.  Notable choices:

- `_options` is an `OrderedDict`.  Argument order in the
  declaration is preserved — important for `--help` output
  and for the generated usage line.
- `_flags_short` and `_flags_long` are separate dicts.  An
  option can have a short flag (`-v`), a long flag
  (`--verbose`), or both; the parser looks up the
  corresponding `OptionField` in O(1).
- `_traps` is a list of error-trap callables.  When an
  argument or option fails to parse / validate, the trap
  callables run before the framework reports.  Custom
  traps let a command attach diagnostic context.

The class has `add_argument(argfield)`, `add_option(optfield)`,
`parse(argv)`, and `execute()` methods.  The decorator
machinery calls `add_argument` / `add_option` at decoration
time; the runtime parser calls `parse` / `execute`.

## `GroupingCommand`: the tree node

```python
# codecraftai-arcli/source/packages/ccai/arcli/model/groupingcommand.py:17–52

class GroupingCommand(Command):
    def __init__(self, name: str, callable: Callable) -> None:
        super().__init__(name, callable)
        self._command_nodes = {}
        return

    @property
    def has_children(self) -> bool:
        rtnval = len(self._command_nodes) > 0
        return rtnval

    def add_command(self, command: Command) -> None:
        self._command_nodes[command.name] = command
        return
```

`GroupingCommand` is the tree node: a `Command` that owns
children.  The decorator is `@arcli.group("name")`; the
function it wraps becomes the group's "entry function" —
typically a no-op that just describes the group, since the
real work lives in the child commands.

The `_command_nodes` dict keys by name; the parser dispatches
to a child by looking up `argv[0]` in `_command_nodes`.  An
unknown subcommand raises a usage error.

There is *no* separate `CommandSet` or `CommandRegistry`
class.  The tree *is* the model — a `GroupingCommand` with
nested `GroupingCommand`s and leaf `Command`s, all the way
down.

## `ArgField`: positional arguments

```python
# codecraftai-arcli/source/packages/ccai/arcli/model/fields/argfield.py:16–62

class ArgField:
    def __init__(self,
                 name: str,
                 alias: Optional[str] = None,
                 ptype: Type[ParameterBase] = None,
                 required: bool = True,
                 default: Any = None,
                 help: Optional[str] = None) -> None:
        self._name = name
        self._alias = alias if alias is not None else name
        self._ptype = ptype if ptype is not None else StringParameter
        self._required = required
        self._default = default
        self._help = help
        return
```

Six instance variables, again initialised in `__init__`.  The
declaration shape:

```python
@arcli.argument("path", ptype=Path(exists=True), help="Input file path.")
@arcli.command("ingest")
def cmd_ingest(path: str):
    ...
```

The argument's `name` is the kwarg name on the function; the
parser binds the parsed value to that kwarg when invoking.
`Path(exists=True)` here is a *parameter type* — see below.

## `OptionField`: flags

```python
# codecraftai-arcli/source/packages/ccai/arcli/model/fields/optionfield.py:40–122

class OptionField:
    def __init__(self,
                 *flags: str,
                 alias: Optional[str] = None,
                 ptype: Type[ParameterBase] = None,
                 default: Optional[Any] = None,
                 required: bool = False,
                 mode: OptionMode = OptionMode.Simple,
                 help: Optional[str] = None) -> None:
        self._flags = flags
        # Sort flags into short / long for parser dispatch.
        self._flag_short = None
        self._flag_long = None
        for flag in flags:
            if flag.startswith("--"):
                self._flag_long = flag
            elif flag.startswith("-"):
                self._flag_short = flag
        self._ptype = ptype if ptype is not None else StringParameter
        self._mode = mode
        self._default = default
        self._required = required
        self._help = help
        return
```

`OptionMode` is the interesting addition over `ArgField`:

- `OptionMode.Simple` — the option appears at most once on
  the command line.  `--name foo` sets the value to `"foo"`.
- `OptionMode.Multiple` — the option may appear multiple
  times.  `--filter foo --filter bar` produces a list
  `["foo", "bar"]`.
- `OptionMode.Counter` — the option counts occurrences.
  `-vvv` produces 3.

The mode drives both parsing and validation.  PR #13's
hardening (below) was specifically about getting validation
right for `Multiple` mode.

## The parameter type system

```python
# codecraftai-arcli/source/packages/ccai/arcli/model/parameters/parameterbase.py:13–34

class ParameterBase(Generic[PT]):
    def __init__(self, alias: str, tvalue: Union[str, List[str]]) -> None:
        self._alias = alias
        self._tvalue = tvalue
        return

    def get_value(self) -> Union[PT, None]:
        err_msg_lines = [
            "ParameterBase.get_value must be overridden by subclasses.",
            "    ALIAS: {}".format(self._alias),
        ]
        err_msg = os.linesep.join(err_msg_lines)
        raise AbstractMethodError(err_msg)
```

Each parameter type is a subclass of `ParameterBase` whose
`get_value()` does the parsing.  The built-ins:

- `StringParameter` — strips quotes; returns `str`.
- `IntegerParameter` — `int(tvalue)`; returns `int`.
- `BooleanParameter` — recognises true/false/yes/no/0/1.
- `ListOfStringParameter` — for `OptionMode.Multiple`;
  returns `List[str]`.
- `ListOfIntegerParameter` — `Multiple`-mode integer list.

The base class is `Generic[PT]`, where `PT` is the produced
type.  Type checkers that introspect arcli get the right
inference.

### Validators as type factories

```python
# codecraftai-arcli/source/packages/ccai/arcli/model/parameters/validators.py:14–227
```

The validator system is *factory functions* that produce
`ParameterBase` subclasses configured with the validator's
constraints:

- `Choice(choices, case_sensitive=True, base_type=StringParameter)`
  — restricts the value to a fixed set.
- `Range(min_val, max_val)` — numeric-range validation.
- `Path(exists=True, file_okay=True, dir_okay=True, writable=False)`
  — filesystem path validation.
- `Regex(pattern, flags=0)` — regex-match validation.
- `Custom(converter)` — apply an arbitrary callable.

Each factory returns a class.  Using a validator looks like:

```python
@arcli.option("--mode",
              ptype=Choice(("dev", "stage", "prod")),
              required=True)
```

The `ptype` *is the produced class*, not an instance — the
parser instantiates it with the raw `tvalue` per parsed
occurrence.

## `OptionMode.Multiple` and the PR #13 hardening

A subtle bug existed in the validator chain before PR #13.
The validators (above) were written assuming `tvalue` was a
single string.  In `OptionMode.Multiple` the parser passes a
*list* of strings — and the original validators applied
their logic to the list-as-a-whole, which made `Choice`
either reject all list values (if the list wasn't itself a
choice) or pass none (if it accidentally was).

The fix in PR #13 (commit `64ced05`, "Harden validator and
parser behavior with regression tests") refactored each
validator's `get_value()` to branch on
`isinstance(self._tvalue, list)`:

```python
# codecraftai-arcli/source/packages/ccai/arcli/model/parameters/validators.py:22–46

def get_value(self) -> Any:
    if isinstance(self._tvalue, list):
        validated_values = []
        for titem in self._tvalue:
            citem = self._base_type(self._alias, titem).get_value()
            vitem = self._validate_choice_value(citem)
            validated_values.append(vitem)
        rtnval = validated_values
    else:
        val = self._base_type(self._alias, self._tvalue).get_value()
        val = self._validate_choice_value(val)
        rtnval = val
    return rtnval
```

Two paths: one for list-mode, one for scalar-mode.  Each
path applies the same `_validate_*_value()` helper to the
canonical-typed value.  The base type runs first (parses
the string), then the validator runs over the parsed value.

A 162-line regression suite at
`codecraftai-arcli/source/tests/test_validators.py` exercises
each validator in both modes.  The suite is the pattern an
SDET should mirror when adding a new validator: write the
single-value tests, write the multi-value tests, run them
both.

The hardening is a Volume 0 chapter 1 success story —
errors that were passing silently (a `--filter` value that
didn't match any choice was silently dropped instead of
rejected) now surface loudly.

## Decorators: how the trees assemble

The four decorators that assemble a tree:

- **`@arcli.command("name")`** — wraps a function in a
  `Command`.
- **`@arcli.group("name")`** — wraps a function in a
  `GroupingCommand`.
- **`@arcli.entry("name")`** — wraps a function in the
  *root* `GroupingCommand` of a CLI binary (the entry
  point).
- **`@arcli.argument(...)`**, **`@arcli.option(...)`** —
  attach `ArgField` / `OptionField` to the wrapped command.

A composite example from `codecraftai-testbase`:

```python
# codecraftai-testbase/source/packages/ccai/testbase/cli/testbase_command.py:32–64

@arcli.option("-v", "--verbose",
              default=0,
              mode=OptionMode.Counter,
              help="Set the verbosity level.")
@arcli.entry("testbase")
def testbase_root_command(verbose: int):
    initialize_testbase_runtime()
    return

testbase_root_command.add_command(group_testbase_agents)
testbase_root_command.add_command(group_testbase_services)
```

The `@arcli.entry("testbase")` wraps the function in the
root.  The `-v / --verbose` option is in `Counter` mode —
`-vv` becomes `2`.  Two child groups are added: the
`agents` group from Chapter 6 and a `services` group.

## Help-text generation

```python
# codecraftai-arcli/source/packages/ccai/arcli/helpgenerator.py:15–72

class HelpGenerator:
    @staticmethod
    def generate_help(command: Command) -> str:
        parts = []
        parts.append(HelpGenerator._generate_usage(command))
        if command.callable.__doc__:
            doc = textwrap.dedent(command.callable.__doc__).strip()
            parts.append(f"  {doc}")
        options_output = HelpGenerator._generate_options(command)
        if options_output:
            parts.append("Options:")
            parts.append(options_output)
        if isinstance(command, GroupingCommand):
            commands_output = HelpGenerator._generate_commands(command)
            if commands_output:
                parts.append("Commands:")
                parts.append(commands_output)
        rtnval = os.linesep.join(parts)
        return rtnval
```

Help is auto-generated from:

- The command's name and arguments → the usage line.
- The function's docstring → the description.
- The command's options → the Options section.
- The grouping command's children → the Commands section.

A test author writes a docstring on the function (per the
Volume 0 chapter 2 docstring style) and arcli's `--help`
output is correct.  No second source of truth.

## Examples in the wild

The agents group from Chapter 6:

```python
# codecraftai-testbase/source/packages/ccai/testbase/cli/cmdtree/agents/__init__.py:18–26

@arcli.group("agents")
def group_testbase_agents():
    """
        Commands for AI agents to extract and analyze test result data.
    """
    return

group_testbase_agents.add_command(group_testbase_agents_summary)
group_testbase_agents.add_command(group_testbase_agents_testresult)
group_testbase_agents.add_command(group_testbase_agents_config)
```

A leaf command in the agents tree:

```python
# codecraftai-testbase/source/packages/ccai/testbase/cli/cmdtree/agents/config/extract.py:32–34

@arcli.option("--config-file", required=True, help=HELP_CONFIG_FILE)
@arcli.command("extract")
def command_agents_config_extract(config_file: str):
    """
        Extracts key configuration information from a test run
        startup-configuration.json file.
    """
    ...
```

The pattern is identical at every level: decorate the
function, attach the result to the parent.  The tree
self-documents through `--help`.

## What `arcli` doesn't do

A short list, in the spirit of Chapter 7's "what testbase
mocking doesn't do":

- **No automatic generation of completions.**  Bash / zsh
  completions are out of scope.  The portfolio's CLIs that
  ship completions ship them as separate files.
- **No async commands.**  A command's callable is sync.
  An async command should call `asyncio.run(...)` itself.
- **No automatic prompting for missing required values.**
  A required option that's missing is a usage error, not a
  prompt.  This is a deliberate choice: the portfolio's
  CLIs run unattended in CI, where prompting is wrong.
- **No subprocess shelling.**  `arcli` parses; the
  command's callable does whatever it does.  The toolkit
  has no opinion about subprocess execution (which lives in
  Volume 0's `codecraftai-this-app` and Volume 5's protocol
  layer).

## What's next

- Chapter 9 — Single-host landscape concepts.  How tests
  describe their environment without dragging in the
  distributed landscape.
- Chapter 10 — Case study: a 3,700-test suite that runs on
  one host with `TESTBASE_MAX_EXAMPLES_CAP=10`.

The `arcli` decorators in this chapter are the *same*
decorators every CLI in the portfolio uses.  Volume 3's
taskplus CLI uses them; Volume 5's protocol-ssh uses them;
Volume 7's agent SDK uses them; Volume 8's time-travel
debugging tools use them.  Once you can read this chapter,
you can read every CLI.
