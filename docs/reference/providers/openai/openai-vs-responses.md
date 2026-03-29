# Completions vs Responses: Choosing the Right Class

The `serapeum-openai` package provides two LLM classes that wrap different OpenAI API
endpoints. This guide explains when to use each and the detailed differences between them.

## Quick Decision Guide

Use **`Completions`** when you need:

- Legacy text completion models (e.g., `text-davinci-003`)
- Native JSON-schema structured output (via `response_format`)
- Audio/multimodal output (`modalities`, `audio_config`)

Use **`Responses`** when you need:

- Built-in tools (web search, file search, code interpreter, computer use, MCP)
- Stateful multi-turn conversations (server tracks context)
- Image generation in responses
- Response storage on OpenAI's platform

---

## API Endpoint Mapping

| | `Completions` | `Responses` |
|---|---|---|
| **Underlying SDK call** | `client.chat.completions.create()` / `client.completions.create()` | `client.responses.create()` |
| **OpenAI API docs** | [Chat Completions](https://platform.openai.com/docs/api-reference/chat) | [Responses](https://platform.openai.com/docs/api-reference/responses) |
| **API maturity** | Stable, widely adopted | Newer API (2025+) |

---

## Feature Comparison

### Core Capabilities

| Feature | `Completions` | `Responses` |
|---|---|---|
| Chat (sync/async) | Yes | Yes |
| Streaming (sync/async) | Yes | Yes |
| Text completion endpoint | Yes (`client.completions.create`) | No (bridged via `ChatToCompletionMixin`) |
| Tool / function calling | Yes | Yes |
| `@overload` typed `stream` param | Yes | Yes |

### Structured Output

| Feature | `Completions` | `Responses` |
|---|---|---|
| **Mechanism** | `StructuredOutput` mixin | Function calling (`tool_choice="required"`) |
| Native JSON-schema (`response_format`) | Yes (for supported models) | No |
| Function-calling fallback | Yes (when model doesn't support JSON schema) | Always uses function calling |
| Streaming structured output | Yes | Yes (via base class) |

The `Completions` class uses a two-tier strategy: if the model supports native JSON-schema
structured output (e.g., `gpt-4o`, `gpt-4o-mini`), it uses the `response_format` parameter
directly. Otherwise, it falls back to function calling, the same approach `Responses` always
uses.

### Built-in Tools & Platform Features

| Feature | `Completions` | `Responses` |
|---|---|---|
| Web search | No | Yes |
| File search | No | Yes |
| Code interpreter | No | Yes |
| Computer use | No | Yes |
| MCP tool calls | No | Yes |
| Image generation | No | Yes |
| Response storage (`store`) | No | Yes |
| Stateful conversations (`track_previous_responses`) | No | Yes |
| Input truncation | No | Yes (`truncation` field) |

### Model Configuration

| Parameter | `Completions` | `Responses` |
|---|---|---|
| `model` (required) | Yes | Yes |
| `temperature` | Yes (0.0-2.0) | Yes (0.0-2.0) |
| `max_tokens` / `max_output_tokens` | `max_tokens` | `max_output_tokens` |
| `top_p` | No (use `additional_kwargs`) | Yes (field) |
| `logprobs` / `top_logprobs` | Yes (fields) | No |
| `reasoning_effort` | Yes (string field) | No (use `reasoning_options` dict) |
| `reasoning_options` | No | Yes (dict, more flexible) |
| `modalities` | Yes (e.g., `["text", "audio"]`) | No |
| `audio_config` | Yes | No |
| `instructions` | No | Yes (system-level instructions) |
| `strict` | Yes | Yes |
| `additional_kwargs` | Yes | Yes |

### Connection & Retry

Both classes inherit from the shared `Client` mixin and have identical connection parameters:

| Parameter | Default | Description |
|---|---|---|
| `api_key` | `None` (reads `OPENAI_API_KEY`) | OpenAI API key |
| `api_base` | `None` (uses `https://api.openai.com/v1`) | Base URL (for Azure or proxies) |
| `api_version` | `None` | API version (for Azure) |
| `max_retries` | `3` | Retries on transient errors (handled by OpenAI SDK) |
| `timeout` | `60.0` | Request timeout in seconds |
| `default_headers` | `None` | Custom HTTP headers |

### Model Validation

| Behavior | `Completions` | `Responses` |
|---|---|---|
| O1 model temperature forcing | Yes (forces `1.0`) | Yes (forces `1.0`) |
| API compatibility check | Yes (rejects Responses-only models) | No (accepts all models) |
| Completion model support | Yes (`is_chat_model` check routes to correct endpoint) | No (always uses Responses API) |

---

## Architecture

### Class Hierarchy

```
BaseLLM
  -> LLM
    -> FunctionCallingLLM
      -> ChatToCompletion + Client + ModelMetadata + StructuredOutput -> Completions
      -> ChatToCompletion + Client + ModelMetadata                   -> Responses
```

### Shared Mixins

| Mixin | Purpose | Used by |
|---|---|---|
| `Client` | Credentials, lazy SDK client creation, event-loop safety | Both |
| `ModelMetadata` | `_get_model_name()`, `_tokenizer` (tiktoken) | Both |
| `StructuredOutput` | Native JSON-schema structured output with fallback | `Completions` only |
| `ChatToCompletion` | Bridges `complete()` via `chat()` for chat-only models | Both |

---

## Usage Examples

### Basic Chat

Both classes have the same chat interface:

```python
from serapeum.openai import Completions, Responses
from serapeum.core.llms import Message, MessageRole, TextChunk

# Chat Completions API
llm = Completions(model="gpt-4o-mini", api_key="sk-...")
response = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello!")])])
print(response.message.content)

# Responses API
llm = Responses(model="gpt-4o-mini", api_key="sk-...")
response = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello!")])])
print(response.message.content)
```

### Streaming

```python
from serapeum.core.llms import Message, MessageRole, TextChunk

# Both support the same streaming interface
for chunk in llm.chat(
    [Message(role=MessageRole.USER, chunks=[TextChunk(content="Tell me a story")])],
    stream=True,
):
    print(chunk.delta, end="", flush=True)
```

### Structured Output

```python
from pydantic import BaseModel
from serapeum.openai import Completions, Responses


class City(BaseModel):
    name: str
    country: str
    population: int

# Completions: uses native JSON schema (response_format) when supported
llm = Completions(model="gpt-4o-mini", api_key="sk-...")
result = llm.parse(City, "What is the capital of France?")

# Responses: always uses function calling
llm = Responses(model="gpt-4o-mini", api_key="sk-...")
result = llm.parse(City, "What is the capital of France?")
```

### Built-in Tools (Responses only)

```python
from serapeum.openai import Responses
from serapeum.core.llms import Message, MessageRole, TextChunk

llm = Responses(
    model="gpt-4o-mini",
    api_key="sk-...",
    built_in_tools=[{"type": "web_search_preview"}],
)

response = llm.chat(
    [Message(role=MessageRole.USER, chunks=[TextChunk(content="What happened today?")])]
)
# Response may include web search results in additional_kwargs
```

### Stateful Conversations (Responses only)

```python
from serapeum.openai import Responses
from serapeum.core.llms import Message, MessageRole, TextChunk

llm = Responses(
    model="gpt-4o-mini",
    api_key="sk-...",
    track_previous_responses=True,  # automatically sets store=True
)

# First message
r1 = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="My name is Alice")])])

# Second message -- server remembers the first via previous_response_id
r2 = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="What's my name?")])])
print(r2.message.content)  # "Your name is Alice"
```

### Async

```python
import asyncio
from serapeum.openai import Completions
from serapeum.core.llms import Message, MessageRole, TextChunk


async def main():
    llm = Completions(model="gpt-4o-mini", api_key="sk-...")

    # Non-streaming
    response = await llm.achat(
        [Message(role=MessageRole.USER, chunks=[TextChunk(content="Hi!")])]
    )

    # Streaming
    async for chunk in await llm.achat(
        [Message(role=MessageRole.USER, chunks=[TextChunk(content="Tell me a story")])],
        stream=True,
    ):
        print(chunk.delta, end="")


asyncio.run(main())
```

---

## Response Differences

### ChatResponse Structure

Both classes return `ChatResponse`, but the `additional_kwargs` content differs:

| Field | `Completions` | `Responses` |
|---|---|---|
| `response.message.content` | Text content | Text content |
| `response.message.chunks` | `TextChunk`, `ToolCallBlock` | `TextChunk`, `ToolCallBlock`, `Image`, `ThinkingBlock` |
| `response.raw` | `ChatCompletion` (OpenAI SDK type) | `Response` (OpenAI SDK type) |
| `additional_kwargs["usage"]` | Token usage dict | Token usage object |
| `additional_kwargs["built_in_tool_calls"]` | N/A | List of built-in tool call results |
| `additional_kwargs["annotations"]` | N/A | Text annotations (citations, etc.) |
| `response.likelihood_score` | Log probabilities (if requested) | N/A |

---

## When to Migrate Between Classes

### From `Completions` to `Responses`

Consider migrating if you need built-in tools, stateful conversations, or image
generation. Key changes:

- `max_tokens` -> `max_output_tokens`
- `reasoning_effort` -> `reasoning_options={"effort": "low"}`
- `logprobs`/`top_logprobs` are not available
- Structured output will use function calling instead of native JSON schema
- Text completion models are not supported

### From `Responses` to `Completions`

Consider migrating if you need native JSON-schema structured output, log probabilities,
or audio output. Key changes:

- `max_output_tokens` -> `max_tokens`
- `reasoning_options` -> `reasoning_effort`
- `built_in_tools`, `track_previous_responses`, `store`, `instructions`, `truncation` are not available
- `top_p` must be passed via `additional_kwargs`
