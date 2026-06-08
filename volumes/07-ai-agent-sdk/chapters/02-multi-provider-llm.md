# Chapter 2 — Multi-Provider LLM Abstraction

> *Production agents don't pin themselves to one LLM
> provider.  Cost varies.  Capabilities differ.  Outages
> happen.  The `codecraftai-agent-kit` ships a small, abstract
> `LLMProvider` interface that any provider implementation
> must honour, plus an `AlternateProviderStrategy` for
> failover when the primary is unavailable.*

This chapter walks the SDK's LLM-provider layer.  By the
end, a reader should be able to:

- Implement a new provider class for any LLM API.
- Register multiple providers on a single agent.
- Wire failover between providers via
  `AlternateProviderStrategy`.

## The premise — providers vary

A production agent's relationship with LLM providers has
three properties that the SDK accommodates:

### Cost varies

GPT-4-class models cost more than smaller models.  Claude
Sonnet costs less than Claude Opus.  An agent that handles
a wide range of work routes *simple* requests to the cheap
model and *hard* requests to the capable one.  Cost-aware
routing requires the agent to *know* which provider to call;
the SDK exposes the registry directly.

### Capabilities differ

Tool calling syntax differs subtly between providers.
Anthropic's tool-use protocol differs from OpenAI's
function-calling.  Streaming response formats differ.
Token-counting algorithms differ.  An SDK that abstracts
these away would lose information; the agent-kit's
abstraction is *thin* — it gives a uniform shape but
preserves provider-specific options through `**kwargs`.

### Outages happen

Every cloud provider has outages.  An agent that *only*
talks to one provider fails when that provider has a bad
day.  An agent with `AlternateProviderStrategy` configured
fails over to a backup transparently to the test code.
Chapter 5 walks the broader resilience story; this chapter
walks the specific provider-failover mechanism.

## The `LLMProvider` base class

**File:** `llmprovider.py:16–60`

```python
class LLMProvider(abc.ABC):
    """
        Abstract base class for Large Language Model providers.

        Subclasses implement the complete and complete_stream
        methods for a specific LLM service.  The base class
        validates that a provider name and model name are
        supplied at construction time.
    """

    def __init__(self, provider_name: str, model_name: str,
                 config: dict = None,
                 credential_names: list = None,
                 tags: dict = None) -> None:
        if provider_name is None or len(str(provider_name).strip()) == 0:
            errmsg = "A provider name is required when constructing an LLMProvider."
            raise SemanticError(errmsg)
        if model_name is None or len(str(model_name).strip()) == 0:
            errmsg = "A model name is required when constructing an LLMProvider."
            raise SemanticError(errmsg)
        self._provider_name = provider_name
        self._model_name = model_name
        self._config = dict(config) if config is not None else {}
        self._credential_names = list(credential_names) if credential_names is not None else []
        self._tags = dict(tags) if tags is not None else {}
        return
```

Five constructor parameters:

- **`provider_name`** — the *vendor* (e.g., `"openai"`,
  `"anthropic"`, `"azure-openai"`).  Required.
- **`model_name`** — the *model* (e.g., `"gpt-4o"`,
  `"claude-opus-4-7"`).  Required.
- **`config`** — provider-specific configuration (base URL,
  API version, organisation ID, etc.).  Optional dict.
- **`credential_names`** — names referencing the credential
  manager (Volume 2 chapter 11).  The actual API keys are
  looked up at runtime.
- **`tags`** — free-form metadata for routing decisions
  (`{"tier": "premium", "task_type": "code"}`).

The constructor *defends against bad inputs* with
`SemanticError` raises (Volume 0 chapter 3 discipline).
Empty strings, None values, missing required fields fail
fast at construction.

## The two abstract methods

**File:** `llmprovider.py:100–125`

```python
@abc.abstractmethod
def complete(self, conversation, **kwargs):
    """
        Executes a synchronous completion request.

        :param conversation: An LLMConversation with the
                             messages to send.
        :returns: An LLMResponse with the completion result.
        :raises: LLMRequestError on provider-side failure.
    """
    ...


@abc.abstractmethod
def complete_stream(self, conversation, handler, **kwargs):
    """
        Executes a streaming completion request.

        :param conversation: An LLMConversation with the
                             messages to send.
        :param handler: The LLMStreamHandler to receive chunks.
        :returns: The final assembled LLMResponse.
        :raises: LLMRequestError on provider-side failure.
    """
    ...
```

Two methods.  Provider implementations override both.

### `complete()` — synchronous request/response

Used when the agent wants the full response before
proceeding.  Takes an `LLMConversation` (chapter 4 walks the
conversation model) and returns an `LLMResponse`.  The
provider blocks until the full response is available.

### `complete_stream()` — streaming request

Used when the agent wants to *stream* the response (display
incremental output, abort early, transform on the fly).
Takes a `handler` callback that receives chunks; returns
the *final* assembled response after streaming completes.

Both methods accept `**kwargs` for provider-specific
options.  This is *the* place where the abstraction lets
provider differences leak through:

```python
# OpenAI provider — temperature and top_p
response = openai_provider.complete(
    conversation,
    temperature=0.7, top_p=0.95,
)

# Anthropic provider — max_tokens (required) and
# system parameter (separate from messages)
response = anthropic_provider.complete(
    conversation,
    max_tokens=4096, system="You are a helpful assistant",
)
```

A reader writing portable agent code constructs `kwargs`
based on the active provider:

```python
common_kwargs = {"max_tokens": 4096}
if isinstance(provider, OpenAIProvider):
    kwargs = {**common_kwargs, "temperature": 0.7}
elif isinstance(provider, AnthropicProvider):
    kwargs = {**common_kwargs, "system": "You are helpful"}

response = provider.complete(conversation, **kwargs)
```

The pattern is *honest* — the application sees the provider
differences instead of having them hidden behind a
lossy abstraction.

## A worked provider implementation

A skeleton for a custom provider implementation (a real
OpenAI provider would be more elaborate — error handling,
retries, token counting, streaming):

```python
import openai
from ccai.agentkit.llmprovider import LLMProvider
from ccai.agentkit.llmconversation import LLMConversation
from ccai.agentkit.llmresponse import LLMResponse
from ccai.agentkit.llmmessage import LLMMessage
from ccai.agentkit.tokenusage import TokenUsage


class OpenAIProvider(LLMProvider):
    """
        OpenAI Chat Completions API provider.
    """

    def __init__(self, model_name: str, api_key: str,
                 base_url: str = None, organization: str = None,
                 credential_names: list = None,
                 tags: dict = None) -> None:
        super().__init__(
            provider_name="openai",
            model_name=model_name,
            config={"base_url": base_url, "organization": organization},
            credential_names=credential_names or [],
            tags=tags,
        )
        self._client = openai.OpenAI(
            api_key=api_key,
            base_url=base_url,
            organization=organization,
        )
        return

    def complete(self, conversation: LLMConversation, **kwargs) -> LLMResponse:
        # Convert LLMConversation to OpenAI's format.
        openai_messages = [
            {"role": msg.role, "content": msg.content}
            for msg in conversation.messages
        ]

        # Make the API call.
        response = self._client.chat.completions.create(
            model=self._model_name,
            messages=openai_messages,
            **kwargs,
        )

        # Translate back to LLMResponse.
        choice = response.choices[0]
        message = LLMMessage(
            role=choice.message.role,
            content=choice.message.content,
        )
        usage = TokenUsage(
            prompt_tokens=response.usage.prompt_tokens,
            completion_tokens=response.usage.completion_tokens,
            total_tokens=response.usage.total_tokens,
        )

        rtnval = LLMResponse(
            message=message,
            token_usage=usage,
            provider_name=self._provider_name,
            model_name=self._model_name,
            raw=response,
        )
        return rtnval

    def complete_stream(self, conversation, handler, **kwargs) -> LLMResponse:
        # ... streaming variant; calls handler.on_chunk(...) for each
        # incremental chunk and returns the final assembled response ...
        ...
```

The provider:

- Translates `LLMConversation` into the format OpenAI's
  Python SDK expects.
- Makes the API call.
- Translates the response back into `LLMResponse` plus
  `TokenUsage`.
- Preserves the raw response for debugging.

Implementations for Anthropic, Azure, local models follow
the same shape — different translation, same skeleton.

## The per-agent provider registry

Each Agent owns an `LLMProviderRegistry` that maps
`(provider_name, model_name)` tuples to provider instances:

```python
class MyAgent(Agent):

    def initialize(self) -> None:
        super().initialize()

        # Register two providers.
        self._llm_provider_registry.register(
            OpenAIProvider(
                model_name="gpt-4o",
                api_key=os.environ["OPENAI_API_KEY"],
            )
        )
        self._llm_provider_registry.register(
            AnthropicProvider(
                model_name="claude-opus-4-7",
                api_key=os.environ["ANTHROPIC_API_KEY"],
            )
        )
```

Lookup at use time:

```python
# By (provider, model) tuple
gpt = self._llm_provider_registry.lookup("openai", "gpt-4o")
claude = self._llm_provider_registry.lookup("anthropic", "claude-opus-4-7")

# By name only (returns first match)
gpt_default = self._llm_provider_registry.lookup_by_provider("openai")

# Default provider (the first registered, or explicitly set)
default = self._llm_provider_registry.lookup_default()
```

The registry is *agent-scoped*, not global.  Each agent
instance has its own.  This matters because:

- Different agents in the same process can use different
  providers without collision.
- Test agents can register mock providers
  (`Mock(spec=LLMProvider)` from Volume 1 chapter 7's
  testbase mocking).
- An agent's providers can be reconfigured per agent
  instance — useful for A/B testing two configurations.

## `AlternateProviderStrategy` — failover

**File:** `alternateproviderstrategy.py:12–36`

```python
class AlternateProviderStrategy(DegradationStrategy):
    """
        Routes the operation to an alternative provider or
        backend when the primary dependency is unavailable.

        :param alternate_callable: The callable to invoke as
                                   the alternate provider.
    """

    def __init__(self, alternate_callable: Callable) -> None:
        super().__init__("alternate_provider")
        self._alternate_callable = alternate_callable
        return

    def degrade(self, dependency_name: str, exception: Exception,
                context: Optional[dict] = None):
        """
            Invokes the alternate callable.

            :param dependency_name: Name of the unavailable dependency.
            :param exception: The exception that triggered degradation.
            :param context: Optional context dictionary.
            :returns: The result from the alternate callable.
        """
        return self._alternate_callable()
```

`AlternateProviderStrategy` is one *strategy* in the SDK's
broader resilience layer (chapter 5 walks the rest).
Specifically for providers, it's used when the primary
fails and the agent wants to *automatically* fall back
to an alternate.

A typical configuration:

```python
class MyAgent(Agent):

    def initialize(self) -> None:
        super().initialize()

        # Primary provider — Anthropic.
        primary = AnthropicProvider(model_name="claude-opus-4-7", ...)
        self._llm_provider_registry.register(primary)

        # Alternate provider — OpenAI as fallback.
        alternate = OpenAIProvider(model_name="gpt-4o", ...)
        self._llm_provider_registry.register(alternate)

        # Failover strategy — when Anthropic fails, use OpenAI.
        self._failover_strategy = AlternateProviderStrategy(
            alternate_callable=lambda: alternate.complete(self._current_conversation)
        )
```

When the primary's `complete()` raises a transient error
(rate limit, 503, network blip), the resilience pipeline
dispatches to the failover strategy, which calls the
alternate provider.

The trade-off the strategy makes: **the test code doesn't
know which provider responded**.  A request that's served
by Anthropic vs OpenAI may behave differently — different
phrasings, different lengths, different tool-call shapes.
The application has to be resilient to this *response*-
level variation if it uses cross-provider failover.

For deterministic tests, the strategy is typically
*disabled* (test code always wants to know which provider
is responding).  For production resilience, it's *enabled*.

## Cost tracking integration

Every successful `complete()` call records its token usage
on the agent's `CostTracker`:

```python
def complete(self, conversation: LLMConversation, **kwargs) -> LLMResponse:
    # ... API call as above ...

    # Record cost.
    cost_estimate = self._compute_cost(usage)
    self._cost_tracker.record(
        provider_name=self._provider_name,
        token_usage=usage,
        cost_estimate=cost_estimate,
    )

    return rtnval
```

The provider knows its own *prices* (per-million-token cost
for prompt / completion) and computes the cost from the
returned `TokenUsage`.  The `CostTracker` accumulates across
all calls.

Chapter 6 walks the cost tracker in detail.  The
integration point worth noting here: *the provider does the
recording*.  Application code that calls
`provider.complete(...)` doesn't have to remember to log
costs — the provider handles it.

## What the abstraction *deliberately* doesn't do

A short list of things the `LLMProvider` interface
intentionally leaves to implementations:

### It doesn't standardise tool-call shapes

OpenAI's function-calling and Anthropic's tool-use protocol
differ in JSON shape, message threading, and error
semantics.  The SDK doesn't try to unify them; instead,
*tool calls* (chapter 3) live above the provider layer
and the `LLMResponse` includes a structured
`tool_calls` list that the agent processes.  Provider
implementations translate to/from the tool-call list at
their boundary.

### It doesn't standardise streaming chunk shapes

`LLMStreamHandler` receives chunks; what's *in* a chunk is
provider-specific.  An agent that wants chunk-level
behaviour writes provider-specific handlers; an agent that
just wants the assembled response uses `complete()`.

### It doesn't standardise error types

OpenAI raises `openai.RateLimitError`; Anthropic raises
`anthropic.RateLimitError`.  Different classes, same
concept.  The SDK ships an `LLMRequestError` that providers
*can* wrap their native errors in, but the wrapping is the
provider implementation's choice.

The discipline is honest: *some provider differences don't
collapse cleanly*, and forcing them to collapse loses
information.

## What's next

Chapter 3 walks the **tool architecture**.  The `Tool` base
class with class-level `TOOL_NAME`, `INPUT_SCHEMA`,
`OUTPUT_SCHEMA`.  The custom `ToolSchema` validation system
(not Pydantic — the SDK rolls its own).  The
`ToolExecutor` that validates inputs, resolves credentials,
invokes the tool, validates outputs.  The two typed
exceptions (`ToolValidationError`, `ToolExecutionError`)
that surface failures structurally.
