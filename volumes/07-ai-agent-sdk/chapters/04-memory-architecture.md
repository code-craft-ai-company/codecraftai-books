# Chapter 4 — Memory Architecture

> *An agent's memory is what makes it *more* than a stateless
> request/response.  The SDK's memory layer has three
> distinct concerns: *conversation history* (the dialogue
> the agent is having now), *persistent memory* (facts the
> agent should remember across runs), and *retrieval*
> (finding the relevant subset of memory for the current
> moment).  This chapter walks each.*

The memory layer has three loosely-coupled subsystems:

- **Conversation history** via `LLMConversation` —
  short-term, in-process, truncates as needed.
- **Persistent memory** via `MemoryStore` and its
  implementations — long-term, scope-aware, optionally
  embedded for semantic search.
- **Summarisation and retrieval** — `ContextSummarizer` for
  compressing conversation; `MemoryIndex` for finding
  relevant entries.

The chapter walks each.

## Conversation history — `LLMConversation`

**File:** `llmconversation.py:15–84`

```python
class LLMConversation:
    """
        An ordered, mutable sequence of LLMMessage objects
        representing a conversation history.
    """

    def truncate(self, max_messages: int) -> None:
        """
            Removes the oldest non-system messages to fit
            within the limit, preserving all system messages.
        """
        if len(self._messages) <= max_messages:
            return
        system_messages = [m for m in self._messages if m.role == "system"]
        non_system_messages = [m for m in self._messages if m.role != "system"]
        available_slots = max_messages - len(system_messages)
        kept_non_system = non_system_messages[-available_slots:] if available_slots > 0 else []
        self._messages = system_messages + kept_non_system
```

The class is an *ordered list of messages*, mutable, with
the framework's truncation discipline.  Three observations:

### System messages are pinned

A typical agent conversation has one or more `system`
messages at the start (the agent's instructions).  The
truncation rule preserves *all* system messages and
removes oldest *non-system* messages first.

This matters because system messages are the agent's
*shape* — without them, the LLM's responses drift away
from the agent's intended behaviour.  Truncation that
removed system messages would produce subtly broken
behaviour over long conversations.

### Truncation is *demand-driven*

The `LLMConversation` doesn't *automatically* truncate.
Application code calls `truncate(max_messages)` when it
decides the conversation is too long.  This separation lets
applications choose their truncation policy:

- Truncate before every LLM call (cap at N messages).
- Truncate on token-count threshold.
- Truncate after specific events (a "task completed"
  message clears the queue).
- Don't truncate (let the conversation grow indefinitely;
  the LLM provider may complain when context is exceeded).

Each agent's `start()` or `step()` method decides.

### Mutability is structured

The class is mutable (you can `append`, `truncate`,
`remove`) but the messages themselves are immutable.  An
`LLMMessage(role="user", content="hello")` constructed
once is constant for its lifetime; the conversation is the
ordered collection of such immutables.

This mirrors the framework's broader "data is immutable;
collections are mutable" pattern.

## Persistent memory — `MemoryStore`

**File:** `memorystore.py:25–127`

```python
class MemoryStore(abc.ABC):
    """
        Abstract base class for memory storage backends.
        Provides a common interface for storing, retrieving,
        searching, updating, and deleting MemoryEntry objects.
    """
    # ... store / retrieve / search / update / delete abstract methods ...
```

The store is *backend-agnostic*.  The SDK ships
`FileMemoryStore` (chapter 4 worked example below); other
backends — Redis, PostgreSQL, vector databases — are
implemented as subclasses.

Three concepts compose the persistent memory model:

### `MemoryEntry`

**File:** `memoryentry.py:17–48`

A single piece of remembered information:

- `identifier` — UUID or string key.
- `content` — the actual data.
- `tags` — list of strings for categorical filtering.
- `metadata` — dict for free-form attributes.
- `scope` — `MemoryScope` enum value (below).
- `created_at`, `updated_at` — timestamps.
- `embedding` — optional vector for semantic search.
- `expiration` — optional `ExpirationPolicy` for TTL.

Entries are *immutable* — modification means constructing a
new entry with the same identifier and `update()`-ing.

### `MemoryScope`

**File:** `memoryscope.py:13–24`

```python
class MemoryScope(Enum):
    """
        Visibility scope for memory entries.
    """
    AGENT = "agent"      # visible only to the owning agent
    SHARED = "shared"    # visible to agents using the same store
    GLOBAL = "global"    # visible to all agents in the runtime
```

Three scopes.  Each agent's memory queries default to
`AGENT` scope (only its own entries); a query with
`scope=MemoryScope.SHARED` returns entries from any agent
sharing the store; `GLOBAL` returns entries from all agents
in the runtime.

The scope distinction matters for *multi-agent systems*.
Two agents collaborating on a task can share information
through `SHARED` entries; private agent state stays
`AGENT`-scoped.

### `EmbeddingProvider`

**File:** `embeddingprovider.py:15–41`

```python
class EmbeddingProvider(abc.ABC):
    """
        Abstract base for embedding providers — services that
        turn text into fixed-length vectors for semantic search.
    """

    @abc.abstractmethod
    def embed(self, text: str) -> list[float]:
        ...

    @abc.abstractmethod
    def batch_embed(self, texts: list[str]) -> list[list[float]]:
        ...
```

When an `EmbeddingProvider` is configured on a store,
new entries get embeddings computed at storage time.
Queries with `semantic_text` parameters use cosine
similarity through `MemoryIndex` (below).

The provider is optional.  Stores without an embedding
provider support exact-match and tag-based queries; semantic
queries return empty.

## `FileMemoryStore` — the worked implementation

**File:** `filememorystore.py:32–195`

```python
class FileMemoryStore(MemoryStore):
    """
        Built-in MemoryStore implementation that persists
        MemoryEntry objects to JSON files on the local file
        system.  Each entry is stored as a separate JSON file
        using the entry identifier as the file name.
    """
```

The simplest persistent backend.  Each entry becomes a
JSON file at `<store_root>/<identifier>.json`.  Reads,
writes, deletes are filesystem operations.  No database;
no schema migrations; no indexing beyond what
`MemoryIndex` (in-memory) computes.

Limitations:

- **Scaling.**  Tens of thousands of entries become slow.
  Production agents with large memories use a different
  backend (Redis, PostgreSQL, etc.).
- **Concurrency.**  Multiple agents writing to the same
  store risk file-level race conditions.  The SDK's
  file-locking is best-effort.

For development and small-scale production agents,
`FileMemoryStore` is enough.  For larger agents, the
abstract `MemoryStore` is the integration point — write a
custom backend, register it.

## Querying — `MemoryQuery`

**File:** `memoryquery.py:15–45`

```python
class MemoryQuery:
    def __init__(self,
                 tags: list = None,
                 metadata: dict = None,
                 scope: MemoryScope = None,
                 start_time: datetime = None,
                 end_time: datetime = None,
                 semantic_text: str = None,
                 max_results: int = None,
                 min_relevance: float = None) -> None:
```

A query is a *bundle of filters*:

- **`tags`** — entries must have all listed tags.
- **`metadata`** — entries must have matching metadata
  key/value pairs.
- **`scope`** — entries must be in the given scope.
- **`start_time` / `end_time`** — temporal bounds on
  `created_at`.
- **`semantic_text`** — for similarity search; the store's
  `EmbeddingProvider` embeds this and `MemoryIndex` ranks
  entries by cosine similarity.
- **`max_results`** — limit the result set.
- **`min_relevance`** — for semantic queries, only return
  entries above a relevance threshold.

The store applies each filter; the result is the
intersection of entries that match all filters.

### `MemoryIndex` — semantic similarity

**File:** `memoryindex.py:17–52`

```python
class MemoryIndex:
    """
        In-memory cosine-similarity index over embedded entries.
    """

    def search(self, query_vector: list[float],
               max_results: int = 10,
               min_relevance: float = 0.0) -> list[ScoredEntry]:
        """
            Returns entries ranked by cosine similarity to
            query_vector.
        """
        # ... cosine similarity ranking ...
```

The index is *in-memory*.  For each query:

1. Compute the query's embedding (via the
   `EmbeddingProvider`).
2. Rank stored entries by cosine similarity.
3. Filter to top-K above threshold.

The implementation is straightforward — dot-product
divided by L2 norms.  Stores can override with their own
indexing (a vector-database-backed store would push
similarity search into the database).

## Summarisation — `ContextSummarizer`

**File:** `contextsummarizer.py:22–108`

```python
def summarize_messages(self, messages: list) -> LLMMessage:
    """
        Summarizes a list of LLMMessage objects into a single
        summary message.
    """
    if messages is None or len(messages) == 0:
        errmsg = "Cannot summarize an empty list of messages."
        raise SemanticError(errmsg)

    content_parts = []
    for msg in messages:
        content_parts.append("[{}] {}".format(msg.role, msg.content))
    combined_content = "\n".join(content_parts)

    prompt_content = ("Summarize the following conversation "
                      "concisely, preserving key information:\n\n"
                      "{}").format(combined_content)
    # ... call LLM provider with this prompt; return the response as
    # a system-role LLMMessage ...
```

Summarisation is *another LLM call*.  The summariser:

1. Formats the messages with role prefixes.
2. Constructs a system prompt asking for a concise summary.
3. Calls the configured LLM provider.
4. Returns a single system-role `LLMMessage` with the
   summary.

The cost is *one LLM call's worth of tokens*.  For long
conversations, this is far cheaper than carrying the full
history in every subsequent call.

### When to summarise

The SDK doesn't impose a summarisation policy.  Common
patterns:

- **Threshold-based.**  When the conversation exceeds N
  messages, summarise the oldest M messages and replace
  them with the summary.
- **Token-based.**  When the token count approaches the
  context window, summarise to fit.
- **Phase-based.**  After a logical phase (a multi-step
  task completes), summarise the phase and start fresh.

Application code decides.  A typical agent run has
explicit summarisation calls between work units.

## Checkpointing — `CheckpointManager`

**File:** `checkpointmanager.py:22–75`

```python
class CheckpointManager:
    """
        Persists and restores WorkflowExecution state at
        configurable checkpoint boundaries using a MemoryStore
        backend.
    """

    def save_checkpoint(self, execution_id: str,
                        context: WorkflowContext,
                        completed_step_ids: set,
                        current_step_id: str) -> None:
        """
            Persists a WorkflowCheckpoint to the MemoryStore.
        """
        try:
            context_state = context.as_dict()
            workflow_name = context.get_metadata("workflow_name", "unknown")
            checkpoint = WorkflowCheckpoint(
                workflow_name=workflow_name,
                execution_id=execution_id,
                completed_step_ids=completed_step_ids,
                context_state=context_state,
                current_step_id=current_step_id,
            )
        # ... store as MemoryEntry ...
```

Checkpoints are how agents *survive crashes*.  When a
long-running workflow's process dies, a fresh process can:

1. Load the most recent checkpoint via
   `manager.load_checkpoint(execution_id)`.
2. Reconstruct the workflow context from the checkpoint's
   `context_state`.
3. Resume from the `current_step_id` rather than starting
   over.

The checkpoint is itself a `MemoryEntry`, stored through
the same `MemoryStore` that persists other agent memory.
This means checkpoints inherit the store's properties —
on disk for `FileMemoryStore`, durable for a
database-backed store, distributed for a Redis-backed
store.

Checkpoints are tagged with the workflow name for efficient
lookup; `manager.list_checkpoints(workflow_name)` returns
all checkpoints for a given workflow.

Chapter 5 walks the workflow layer that *uses* checkpoints.

## Memory's interaction with cost tracking

Every memory operation that reads or writes goes through
the store.  The store *can* (but doesn't have to) record
costs:

- A `FileMemoryStore` has zero per-operation cost — just
  filesystem I/O.
- A vector-database-backed store has per-query cost
  (database CPU, network I/O).
- A store that *embeds at write time* via an
  `EmbeddingProvider` has per-write cost (the embedding
  call's tokens).

When the embedding step uses an LLM-style provider, those
calls *do* hit the agent's cost tracker.  An agent that
writes 10,000 entries to memory and embeds each one pays
for 10,000 embedding requests.  The cost tracker shows the
breakdown by provider; an SDET reviewing costs can see
"embedding-provider-1" charged separately from
"main-llm-provider."

## What memory deliberately *doesn't* do

A few patterns the SDK leaves to the application:

### No automatic summarisation

The summariser is a *tool* the agent calls when it
decides.  The SDK doesn't watch the conversation length
and trigger summarisation automatically.

This is for *predictability*.  An automated summarisation
that fires unexpectedly mid-conversation can produce
surprising behaviour (the LLM "forgets" things it said two
turns ago).  Explicit calls let the application decide
*when* the cost / behaviour-change is acceptable.

### No automatic memory expiration

Entries can carry an `ExpirationPolicy`, but expiration is
*evaluated lazily* — entries are removed on the next query
or write that touches them.  No background sweeper.

This avoids the operational complexity of a sweeper thread
that needs to be scheduled, monitored, and tuned.

### No cross-agent message passing

The SDK's memory is for *state*, not *messaging*.  Agents
that need to communicate use a separate message-routing
layer (the SDK ships a `MessageRouter` — chapter 5
covers).

## What's next

Chapter 5 walks **workflow orchestration for agents**.
The SDK's `WorkflowExecutor` traverses a DAG of steps with
conditional branches, parallel execution, checkpointing
(via the `CheckpointManager` from this chapter), and
human-in-the-loop approval gates.  Chapter 5 also covers
`CircuitBreaker` for handling repeated failures
gracefully.
