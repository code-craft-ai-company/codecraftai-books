# Chapter 3 — Tool Design

> *Tools are how an agent affects the world beyond text.
> The `codecraftai-agent-kit` ships a small, opinionated tool
> system: declarative `INPUT_SCHEMA` and `OUTPUT_SCHEMA`,
> `ToolExecutor` that validates inputs, runs the tool, and
> validates outputs, two typed exceptions for the failure
> modes, structured `ToolResult` return.*

The SDK's tool system is small enough to walk in one
chapter.  Three classes (`Tool`, `ToolSchema`, `ToolExecutor`)
plus two exceptions (`ToolValidationError`,
`ToolExecutionError`) plus one result type (`ToolResult`).

By the end, a reader should be able to write a custom tool,
register it with an agent, and dispatch it correctly through
the executor.

## The `Tool` base class

**File:** `tool.py:23–83`

```python
class Tool(abc.ABC):
    """
        Abstract base class for all tools.  Subclasses define
        tool behavior by implementing the execute method and
        declaring schemas via class-level constants.
    """

    TOOL_NAME: str = None
    TOOL_DESCRIPTION: str = ""
    TOOL_VERSION: str = "1.0.0"
    INPUT_SCHEMA: dict = None
    OUTPUT_SCHEMA: dict = None

    def __init__(self, tags: dict = None,
                 credential_names: list = None) -> None:
        self._name = self.TOOL_NAME
        self._description = self.TOOL_DESCRIPTION
        self._version = self.TOOL_VERSION
        self._input_schema = None
        self._output_schema = None
        # ... tags + credential_names initialisation ...

        if self._name is None:
            errmsg = "A Tool name is required.  Set TOOL_NAME on the class."
            raise SemanticError(errmsg)

        if self.INPUT_SCHEMA is not None:
            try:
                self._input_schema = ToolSchema.from_dict(schema_data=self.INPUT_SCHEMA)
            except Exception as schema_err:
                errmsg_lines = [
                    "Invalid INPUT_SCHEMA on tool '{}'.".format(self._name),
                    "Schema validation detail: {}".format(str(schema_err)),
                ]
                errmsg = os.linesep.join(errmsg_lines)
                raise SemanticError(errmsg) from schema_err
        # ... similar OUTPUT_SCHEMA validation ...

        return

    @abc.abstractmethod
    def execute(self, inputs: dict, credentials: dict = None) -> dict:
        """
            Executes the tool with the given validated inputs.
        """
        ...
```

Five class-level constants every tool sets:

- **`TOOL_NAME`** — the tool's unique identifier.  The
  `ToolRegistry` looks up tools by name.  Required (raises
  `SemanticError` if absent).
- **`TOOL_DESCRIPTION`** — natural-language description.
  Sent to the LLM when the agent declares its tools to the
  provider.
- **`TOOL_VERSION`** — the tool's version.  Useful for
  evolution — version bumps signal to consumers that the
  tool's interface changed.
- **`INPUT_SCHEMA`** — declarative dict describing the
  tool's parameters.  Validated at execution time.
- **`OUTPUT_SCHEMA`** — declarative dict describing the
  tool's return shape.  Also validated.

And one abstract method:

- **`execute(inputs, credentials)`** — the tool's actual
  logic.  Receives a *validated* inputs dict and (optionally)
  a credentials dict resolved from `credential_names`.
  Returns a dict that will be *validated* against the
  `OUTPUT_SCHEMA`.

A worked tool subclass:

```python
class FetchWeatherTool(Tool):

    TOOL_NAME = "fetch_weather"
    TOOL_DESCRIPTION = "Look up the current weather for a city."
    TOOL_VERSION = "1.0.0"

    INPUT_SCHEMA = {
        "parameters": [
            {"name": "city", "type": "string", "required": True,
             "description": "City name (e.g., 'San Francisco')"},
            {"name": "country", "type": "string", "required": False,
             "default": "US",
             "description": "ISO country code"},
            {"name": "units", "type": "string", "required": False,
             "default": "imperial",
             "description": "'imperial' (Fahrenheit) or 'metric' (Celsius)"},
        ]
    }

    OUTPUT_SCHEMA = {
        "parameters": [
            {"name": "city", "type": "string", "required": True},
            {"name": "country", "type": "string", "required": True},
            {"name": "temperature", "type": "float", "required": True},
            {"name": "units", "type": "string", "required": True},
            {"name": "conditions", "type": "string", "required": True},
        ]
    }

    def __init__(self):
        super().__init__(credential_names=["weather-api-key"])
        return

    def execute(self, inputs: dict, credentials: dict = None) -> dict:
        api_key = credentials["weather-api-key"]
        # ... call the weather API with the resolved key ...
        # ... parse the response into the OUTPUT_SCHEMA shape ...
        return {
            "city": inputs["city"],
            "country": inputs["country"],
            "temperature": 72.0,
            "units": inputs["units"],
            "conditions": "sunny",
        }
```

Five things to notice:

1. **Class-level constants for the tool's metadata.**
   Subclasses set them; the base validates at construction.
2. **The `INPUT_SCHEMA` and `OUTPUT_SCHEMA` are
   dictionaries**.  They get parsed into `ToolSchema`
   instances inside `__init__`.
3. **Defaults handle missing optional parameters.**  When
   the LLM calls `fetch_weather` with only `city`, the
   schema's defaults populate `country="US"` and
   `units="imperial"`.
4. **Credentials are *named*, not embedded.**  The tool
   declares `credential_names=["weather-api-key"]`; the
   `ToolExecutor` resolves the name through the
   credentials manager (Volume 2 chapter 11) and passes
   the resolved value to `execute()`.
5. **The tool's logic returns a plain dict** — no special
   return type required.  The dict is *validated* against
   `OUTPUT_SCHEMA` before reaching the agent.

## The custom schema system

**File:** `toolschema.py:14–56`

```python
class ToolSchema:
    """
        Declarative schema definition for tool input or output
        parameters.  Supports type constraints, required/optional
        marking, defaults, and validation.
    """

    SUPPORTED_TYPES = {"string", "integer", "float", "boolean", "list", "dictionary"}

    TYPE_MAP = {
        "string": str,
        "integer": int,
        "float": float,
        "boolean": bool,
        "list": list,
        "dictionary": dict,
    }

    def __init__(self, parameters: list = None) -> None:
        if parameters is not None:
            self._parameters = list(parameters)
        else:
            self._parameters = []
        return
```

Six supported types — `string`, `integer`, `float`,
`boolean`, `list`, `dictionary`.  The schema is
*declarative* — a parameter is described by:

- `name` (required) — the parameter's key.
- `type` (required) — one of the six supported types.
- `required` (optional, default True) — whether the
  parameter must be present.
- `default` (optional) — used when the parameter is
  optional and absent.
- `description` (optional) — natural-language help.

### Why custom and not Pydantic

A reader from a Python web background would expect Pydantic.
The SDK doesn't use it.  Reasons:

- **Pydantic v1 vs v2** — large API differences, ongoing
  community migration.  Pinning Pydantic v2 means
  applications stuck on v1 can't use the SDK; pinning v1
  means the SDK becomes a bottleneck for v2 migration.
- **Schema-as-data, not schema-as-class.**  A tool's schema
  is a `dict` literal — easy to introspect, easy to
  serialise to JSON for sending to the LLM (which expects
  JSON Schema), easy to compare across tool versions.
- **Smaller surface.**  Six types covers what tools need.
  A custom validator that handles those six is a few
  hundred lines.  Pydantic brings hundreds of features the
  agent-kit doesn't need (validators, computed fields,
  serialisers, etc.).

The trade-off: tools that need *complex nested
validation* (e.g., "this parameter is a list of dicts each
with these specific keys") have to either compose multiple
parameters or do the validation in `execute()`.  Most tools
don't need this; the few that do work around it.

### The `validate` method

**File:** `toolschema.py:68–111`

```python
def validate(self, values: dict) -> list:
    """
        Validates a dictionary of values against this schema.

        Checks for missing required parameters, applies default
        values, and verifies type constraints.

        :returns: A list of validation error strings.  Empty list
                  if all values are valid.
    """
    errors = []
    for param_def in self._parameters:
        param_name = param_def["name"]
        param_type = param_def["type"]
        is_required = param_def.get("required", True)

        has_value = param_name in values
        if has_value is False:
            if is_required is True:
                errmsg = "Missing required parameter: '{}'".format(param_name)
                errors.append(errmsg)
            else:
                has_default = "default" in param_def
                if has_default is True:
                    default_value = param_def["default"]
                    values[param_name] = default_value
        else:
            provided_value = values[param_name]
            expected_python_type = self.TYPE_MAP[param_type]
            is_correct_type = isinstance(provided_value, expected_python_type)
            if is_correct_type is False:
                actual_type_name = type(provided_value).__name__
                errmsg = "Parameter '{}': expected type '{}', got '{}'".format(
                    param_name, param_type, actual_type_name
                )
                errors.append(errmsg)

    return errors
```

The validator returns a *list of error strings*, not a
boolean.  A schema with three missing required parameters
returns three error strings.  This matches the framework's
list-of-errors discipline (Volume 0 chapter 2's
multi-line-error-message pattern).

The validator *mutates the input dict* — defaults are
applied in place.  After validation, `values` has every
required parameter (those that the caller passed plus
defaults filled in for optionals).

## The `ToolRegistry`

**File:** `toolregistry.py:16–49`

```python
class ToolRegistry:
    """
        Registry tracking available Tool instances for an Agent.
        Provides lookup by name, tags, and enumeration.
    """

    def register(self, tool: Tool) -> None:
        tool_name = tool.name
        is_duplicate = tool_name in self._tools
        if is_duplicate is True:
            errmsg_lines = [
                "Duplicate tool registration attempted.",
                "  Tool name: '{}'".format(tool_name),
            ]
            errmsg = os.linesep.join(errmsg_lines)
            raise SemanticError(errmsg)
        self._tools[tool_name] = tool
        return

    def lookup(self, tool_name: str) -> Tool:
        rtnval = self._tools.get(tool_name, None)
        return rtnval

    def lookup_by_tags(self, **tags: str) -> list:
        # ... tag-matching logic ...
```

Each agent owns a `ToolRegistry`.  Registration:

- Adds the tool by name.
- *Refuses duplicates* — raises `SemanticError` if a tool
  with the same name is already registered.

Lookup:

- `lookup(name)` — returns the tool or `None`.
- `lookup_by_tags(**tags)` — returns tools matching the
  given tag dict (intersection — every tag must match).

The registry is *agent-scoped* — same pattern as the LLM
provider registry (chapter 2).  Different agents in the
same process have independent tool sets.

## The `ToolExecutor`

**File:** `toolexecutor.py:21–162`

The executor is the *orchestrator* that wraps a tool call:

1. Validate the inputs against the tool's `INPUT_SCHEMA`.
   If validation fails, raise `ToolValidationError`.
2. Resolve the credentials referenced by the tool's
   `credential_names`.
3. Invoke the tool's `execute(inputs, credentials)`.
4. Validate the return value against the tool's
   `OUTPUT_SCHEMA`.  If validation fails, raise
   `ToolValidationError`.
5. Return a `ToolResult` with status / output / duration.

The executor is what the agent's tool-call dispatch path
uses.  When the LLM responds with `"call fetch_weather with
{city: 'San Francisco'}"`, the agent's tool dispatcher:

1. Looks up `fetch_weather` in the registry.
2. Hands the tool to the executor.
3. The executor does the dance above.
4. The agent receives a `ToolResult` and decides what to
   do (pass it back to the LLM as a tool result; log it;
   etc.).

The agent doesn't talk to the tool *directly*; it goes
through the executor.  This separation lets the executor
add cross-cutting concerns (validation, credential
resolution, result wrapping) without each tool having to
implement them.

## The two typed exceptions

The SDK distinguishes two failure modes through typed
exceptions.

### `ToolValidationError`

**File:** `toolerrors.py:15–83`

```python
class ToolValidationError(AgentKitError):
    """
        Raised when input or output validation against a
        ToolSchema fails.

        :param tool_name: The name of the tool that failed validation.
        :param direction: The validation direction, "input" or "output".
        :param errors: A list of validation error strings.
    """

    def __init__(self, tool_name: str, direction: str, errors: List[str]) -> None:
        self._tool_name = tool_name
        self._direction = direction
        self._errors = list(errors)
        errmsg = self._format_message()
        super().__init__(errmsg)
        return
```

Raised when:

- The agent calls a tool with arguments that violate
  `INPUT_SCHEMA` (`direction="input"`).
- The tool returns a value that violates `OUTPUT_SCHEMA`
  (`direction="output"`).

The `direction` field tells the agent *who* was wrong:
input failures suggest the LLM produced bad arguments;
output failures suggest the tool's own logic is wrong.

### `ToolExecutionError`

**File:** `toolerrors.py:99–148`

```python
class ToolExecutionError(AgentKitError):
    """
        Raised when a Tool's execute method encounters a
        runtime error.

        :param tool_name: The name of the tool that failed execution.
        :param cause: The original exception that caused the failure.
    """

    def __init__(self, tool_name: str, cause: Exception) -> None:
        self._tool_name = tool_name
        self._cause = cause
        errmsg = self._format_message()
        super().__init__(errmsg, cause=cause)
        return
```

Raised when the tool's `execute()` method raises any
exception (caught by the executor).  The original cause is
preserved via `cause=`.

The two-exception split mirrors the *contract violation* vs
*implementation failure* distinction:

- **Validation error** — the contract (schema) was
  violated.  Either the LLM constructed bad arguments or
  the tool returned bad output.
- **Execution error** — the contract was honoured, but
  the tool's logic blew up (network failure, timeout,
  resource exhaustion).

A test catching `ToolValidationError` learns its prompt
or tool definition is broken.  A test catching
`ToolExecutionError` learns the tool's environment failed.

## The `ToolResult` structured return

**File:** `toolresult.py:10–96`

```python
class ToolResult:
    """
        Immutable result of a single tool execution.
    """

    STATUS_SUCCESS = "success"
    STATUS_FAILURE = "failure"

    def __init__(self, tool_name: str, status: str,
                 output: dict = None,
                 duration: float = 0.0,
                 error: Exception = None) -> None:
        self._tool_name = tool_name
        self._status = status
        self._output = output
        self._duration = duration
        self._error = error
        return
```

When the executor catches an exception, instead of
re-raising, it can construct a `ToolResult` with
`status=STATUS_FAILURE` and the exception object.  The
agent receives a *result*, not an exception.

Two patterns coexist:

### Strict mode — exception bubbles

```python
try:
    result = executor.execute(tool, inputs)
except ToolValidationError as ex:
    # Bad input — log, retry, escalate.
    ...
except ToolExecutionError as ex:
    # Tool broke — log, retry, escalate.
    ...
```

The executor raises; the agent catches.  Used when failures
are *meaningful* — the agent wants to see the typed
exception and dispatch on the type.

### Result mode — failure surfaces in the result

```python
result = executor.execute(tool, inputs, raise_on_failure=False)

if result.status == ToolResult.STATUS_FAILURE:
    # The agent can pass the failure back to the LLM as
    # a "tool failed" message and let the LLM decide.
    self._llm_send_tool_failure(tool.name, str(result.error))
elif result.status == ToolResult.STATUS_SUCCESS:
    self._llm_send_tool_result(tool.name, result.output)
```

The executor catches and wraps; the agent reads the result.
Used when the agent wants to *let the LLM cope* — passing
the failure back into the conversation lets the LLM choose
to try a different tool, ask the user, or abandon the
task.

Both patterns are supported; the choice is the agent's.

## Cross-cutting concerns the executor handles

Beyond input / output validation, the executor handles:

### Credential resolution

The tool declares `credential_names = ["api-key"]`.  The
executor resolves the names via the agent's credentials
manager and passes the resolved dict to `execute()`.  The
tool never sees the credentials lookup logic.

### Duration tracking

The executor times the call.  The `ToolResult.duration`
field captures the wall-clock time.  This feeds the
`agent.tool.execution.duration` histogram metric (chapter
6) automatically.

### Observability events

The executor emits structured events:

- `tool.execution.started` — before the call.
- `tool.execution.completed` — after success.
- `tool.execution.failed` — after failure.

The agent's `ExecutionRecorder` (chapter 6) captures these
events for replay.  The agent's `Tracer` opens spans
covering the call.

### Error wrapping

Generic exceptions caught from `execute()` are wrapped in
`ToolExecutionError` so the agent sees a typed exception
regardless of what the tool's underlying code did.

## What the tool system *doesn't* do

A short list of patterns the SDK deliberately doesn't
support:

### No automatic tool generation

The SDK doesn't introspect functions and turn them into
tools.  Tools are *explicit* — a class with explicit
schemas.  Auto-generation would let the application
accidentally expose arbitrary functions to the LLM (a
security concern); the explicit shape is *intentional*.

### No nested tool calls

A tool's `execute()` receives validated inputs and returns
a dict.  It cannot call back into the agent's
`tool_dispatcher`.  Tool composition happens at the
*agent* level — the agent decides which tools to call in
which order based on LLM responses.

This is for clarity.  Nested tool calls could create cycles
and make debugging harder.  The agent has the full picture
of what's been called and when; tools focus on their own
work.

### No automatic LLM tool-call format conversion

The agent's *adapter* (provider-specific code) converts the
agent's tool list into the LLM's tool-call format.
OpenAI's function-calling JSON differs from Anthropic's
tool-use JSON.  The provider implementation handles the
translation; the SDK's tool layer is provider-agnostic.

## What's next

Chapter 4 walks the **memory architecture**.  `MemoryStore`,
`MemoryScope` (AGENT / SHARED / GLOBAL),
`LLMConversation` truncation that preserves system
messages, summarisation via `ContextSummarizer`, semantic
search via `MemoryIndex`, checkpointing for resume.

The memory layer is *structurally similar* to the tool
layer — base classes, registries, validation — but
conceptually different.  Where tools are *outputs* (the
agent affecting the world), memory is *state* (the agent
remembering what happened).
