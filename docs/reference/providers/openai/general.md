# OpenAI LLM Integration

This directory contains comprehensive documentation explaining the complete workflow of the
OpenAI provider classes, from initialization to execution across various modes (chat, completion,
streaming, tool calling, structured outputs, async).

## Overview

The OpenAI provider offers two main classes mapping to OpenAI's two API surfaces:

1. **`Completions`** -- Chat Completions API (GPT-4o, GPT-4-turbo, GPT-3.5, O1, legacy models)
2. **`Responses`** -- Responses API (O3, O4-mini, GPT-4o, modern reasoning models)

Both classes provide:

1. **Connection to OpenAI API** with automatic credential resolution
2. **Chat and completion APIs** with sync/async support
3. **Streaming responses** for real-time output
4. **Tool/function calling** for structured interactions
5. **Native structured outputs** via JSON-schema `response_format`
6. **Retry logic** with OpenAI-specific exception classification
7. **Integration with orchestrators** (TextCompletionLLM, ToolOrchestratingLLM)

## Example Usage

### Basic Chat

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    temperature=0.7,
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Say 'pong'.")])]
response = llm.chat(messages)
print(response.message.content)  # "Pong!"
```

### With Responses API

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Responses

llm = Responses(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Say 'pong'.")])]
response = llm.chat(messages)
print(response.message.content)  # "Pong!"
```

### With TextCompletionLLM

```python
import os
from pydantic import BaseModel
from serapeum.core.output_parsers import PydanticParser
from serapeum.core.llms import TextCompletionLLM
from serapeum.openai import Completions


class DummyModel(BaseModel):
    value: str


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

text_llm = TextCompletionLLM(
    output_parser=PydanticParser(output_cls=DummyModel),
    prompt="Generate any value: {value}",
    llm=llm,
)

result = text_llm(value="input")
# Returns: DummyModel(value="input")
```

### With ToolOrchestratingLLM

```python
import os
from pydantic import BaseModel
from serapeum.core.llms import ToolOrchestratingLLM
from serapeum.openai import Completions


class Album(BaseModel):
    title: str
    artist: str
    songs: list[str]


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

tools_llm = ToolOrchestratingLLM(
    schema=Album,
    prompt="Create an album about {topic} with two random songs",
    llm=llm,
)

result = tools_llm(topic="rock")
# Returns: Album(title="...", artist="...", songs=[...])
```

## Understanding the Workflow

### 1. [Execution Flow and Method Calls](./openai_sequence.md)

Shows the chronological flow of method calls and interactions across all usage patterns.

**Best for**:

- Understanding the order of operations
- Seeing how the provider communicates with OpenAI's API
- Debugging execution flow
- Understanding integration patterns

**Key Flows**:

- Initialization phase (lazy client creation)
- Direct chat/completion calls
- Streaming execution (sync and async)
- Tool calling with schema conversion
- Native structured outputs vs function-calling fallback
- Integration with TextCompletionLLM and ToolOrchestratingLLM
- Async operations

### 2. [Architecture and Class Relationships](./openai_class.md)

Illustrates the static structure, inheritance hierarchy, and relationships.

**Best for**:

- Understanding the architecture
- Seeing the dual-class design (Completions vs Responses)
- Seeing the mixin chain (Client, ModelMetadata, StructuredOutput)
- Identifying class responsibilities
- Understanding integration points

**Key Classes**:

- `Completions`: Chat Completions API implementation
- `Responses`: Responses API implementation
- `Client`: SDK client lifecycle mixin
- `ModelMetadata`: Tokenizer and model info mixin
- `StructuredOutput`: Native JSON-schema structured output mixin
- `Retry`: Retry logic with exception classification
- Response models: `ChatResponse`, `CompletionResponse`, `Message`

### 3. [Data Transformations and Validation](./openai_dataflow.md)

Tracks how data transforms through the system across different operation modes.

**Best for**:

- Understanding data transformations
- Identifying validation points
- Seeing error handling paths
- Understanding request/response formats

**Key Flows**:

- Initialization and credential resolution
- Chat request building and response parsing
- Completion via ChatToCompletion adapter
- Tool schema conversion (nested vs flat formats)
- Streaming chunk processing with ToolCallAccumulator
- Structured output via response_format or function-calling fallback

### 4. [Component Boundaries and Interactions](./openai_components.md)

Shows component boundaries, responsibilities, and interaction patterns.

**Best for**:

- Understanding system architecture
- Seeing component responsibilities
- Identifying interaction patterns
- Understanding integration layers

**Key Components**:

- User space (application code)
- OpenAI core (request building, response parsing, tool handling)
- Client layer (OpenAI SDK)
- OpenAI API (inference endpoints)
- Parser layer (ChatMessageParser, ToolCallAccumulator, ResponsesOutputParser)
- Orchestrator layer (TextCompletionLLM, ToolOrchestratingLLM)

### 5. [Lifecycle and State Management](./openai_state.md)

Depicts the lifecycle states, transitions, and state variables.

**Best for**:

- Understanding instance lifecycle
- Seeing state transitions
- Identifying error states
- Understanding concurrency and event-loop safety

**Key States**:

- Uninitialized -> Configured (initialization with validators)
- Configured -> ClientInitialized (lazy client creation)
- Idle <-> Processing* (request handling with retry)
- Processing -> Error -> Idle (error handling with retryable classification)

### 6. [Usage Examples](./examples.md)

Comprehensive examples from real test cases.

**Best for**:

- Learning by example
- Understanding practical usage
- Seeing all API variants
- Integration patterns

**Key Examples**:

- Basic chat and completion
- Streaming operations (sync and async)
- Tool/function calling (single, parallel, strict mode)
- Native structured outputs via `parse()`
- Integration with orchestrators
- Responses API usage (built-in tools, reasoning, stateful conversations)
- Error handling

## Core Capabilities

### 1. Chat API (Completions)

```
Direct conversation via Chat Completions API:
- Single and multi-turn conversations
- System messages for context
- Custom parameters (temperature, top_p, max_tokens, etc.)
- O1 model support (reasoning_effort, forced temperature)
- Audio modalities
- Log probabilities
```

### 2. Chat API (Responses)

```
Conversation via Responses API:
- System-level instructions
- Built-in tools (web_search, file_search, code_interpreter)
- Stateful multi-turn (track_previous_responses)
- Reasoning options for O3/O4 models
- Server-side response storage
```

### 3. Completion API

```
Text completion via ChatToCompletion adapter:
- Converts prompt to chat message
- Delegates to chat API
- Extracts text from response
- Also supports legacy Completions endpoint (text-davinci, etc.)
```

### 4. Streaming

```
Real-time response generation:
- Stream chat responses (sync and async)
- Stream completion responses
- Chunk-by-chunk processing with ToolCallAccumulator
- Delta content access
- Token usage in final chunk (stream_options)
```

### 5. Tool/Function Calling

```
Structured interactions with tools:
- Automatic schema conversion to OpenAI format
- Strict mode (additionalProperties=false)
- Single or parallel tool calls
- Tool choice control (auto/required/none/specific)
- Streaming tool calls
```

### 6. Native Structured Outputs

```
JSON-schema response_format (modern models):
- parse() / aparse() methods
- Streaming structured outputs
- Automatic fallback to function-calling for unsupported models
- Strict JSON-schema validation
```

### 7. Async Operations

```
Non-blocking execution:
- Async chat and completion
- Async streaming
- Event-loop safety (recreates client on closed loops)
- Concurrent request handling
```

## Key Design Patterns

### 1. Lazy Initialization

Clients are created on first use, not during `__init__`:

```python notest
@property
def client(self) -> SyncOpenAI:
    if self._client is None:
        self._client = self._build_sync_client(**self._get_credential_kwargs())
    return self._client
```

### 2. Template Method Pattern

Client base class defines factory methods, subclasses override:

```python notest
# Client base class
def _build_sync_client(self, **kwargs: Any) -> SyncOpenAI:
    return SyncOpenAI(**kwargs)

# AzureOpenAI overrides
def _build_sync_client(self, **kwargs: Any) -> SyncAzureOpenAI:
    return SyncAzureOpenAI(**kwargs)
```

### 3. Mixin Composition

Completions composes multiple mixins for separation of concerns:

```python notest
class Completions(StructuredOutput, ModelMetadata, Client, ChatToCompletion, FunctionCallingLLM):
    # StructuredOutput: native JSON-schema structured outputs
    # ModelMetadata: tokenizer and model info
    # Client: SDK client lifecycle and credentials
    # ChatToCompletion: completion API via chat adapter
    # FunctionCallingLLM: tool-calling interface
    ...
```

### 4. Retry with Exception Classification

Framework-level retry with OpenAI-specific retryable detection:

```python notest
@retry(is_retryable, logger)
def _chat(self, messages, **kwargs) -> ChatResponse:
    # SDK max_retries=0, framework handles retries
    ...
```

### 5. Adapter Pattern

OpenAI adapts between internal types and OpenAI API format:

- `Message` -> OpenAI message dicts (via `to_openai_message_dicts`)
- `BaseTool` -> OpenAI tool schema (nested or flat)
- OpenAI response -> `ChatResponse`/`CompletionResponse` (via ChatMessageParser)

## Integration Architecture

```
User Application
    |
ToolOrchestratingLLM / TextCompletionLLM
    |
Completions / Responses
    |
Client mixin (SyncOpenAI / AsyncOpenAI)
    |
OpenAI API (HTTPS)
    |
Model Runtime (GPT-4o, O3, etc.)
```

### TextCompletionLLM Integration

```
1. Formats prompt with variables
2. Checks is_chat_model -> True
3. Calls Completions.chat()
4. Parses response with PydanticParser
5. Returns validated model instance
```

### ToolOrchestratingLLM Integration

```
1. Converts output_cls to CallableTool
2. Formats prompt with variables
3. Calls Completions.generate_tool_calls()
4. Completions converts tool to OpenAI schema
5. API returns tool_calls
6. Executes tool to create instance
7. Returns model instance(s)
```

## Performance Considerations

1. **Client Reuse**: Client created once and reused for all requests
2. **Async Support**: Separate async client with event-loop safety
3. **Streaming**: Reduces latency for long responses
4. **Connection Pooling**: httpx client handles connection reuse
5. **Lazy Initialization**: Only create clients when needed
6. **SDK Retry Disabled**: `max_retries=0` avoids double-retry with framework `@retry`

## Configuration Options

### Essential (Completions)

- `model`: Model name (e.g., "gpt-4o-mini", "gpt-4-turbo", "o1-mini")
- `api_key`: OpenAI API key (default: from `OPENAI_API_KEY` env var)
- `temperature`: Sampling temperature (0.0-2.0, default: 0.1)

### Essential (Responses)

- `model`: Model name (e.g., "gpt-4o-mini", "o3", "o4-mini")
- `api_key`: OpenAI API key (default: from `OPENAI_API_KEY` env var)
- `instructions`: System-level instructions (default: None)

### Generation

- `max_tokens`: Maximum generation tokens (default: None)
- `top_p`: Top-p sampling (default: 1.0, Responses only)
- `logprobs`: Return log-probabilities (default: None, Completions only)
- `reasoning_effort`: O1 model reasoning budget (Completions only)
- `reasoning_options`: O1 reasoning config dict (Responses only)

### Connection

- `api_base`: Base URL (default: `https://api.openai.com/v1`)
- `api_version`: API version string (mainly for Azure)
- `timeout`: HTTP timeout in seconds (default: 60.0)
- `default_headers`: Custom HTTP headers
- `max_retries`: Retry attempts (default: 3)

### Advanced

- `additional_kwargs`: Extra API parameters
- `strict`: Strict JSON-schema mode for tools (default: False)
- `modalities`: Output modalities, e.g. `["text", "audio"]` (Completions only)
- `audio_config`: Audio output configuration (Completions only)
- `track_previous_responses`: Stateful multi-turn (Responses only)
- `built_in_tools`: Built-in tool configs (Responses only)

## Error Handling

### Retryable Errors

```
APIConnectionError: Network connectivity issue
APITimeoutError: Request timeout exceeded
RateLimitError: 429 Too Many Requests
InternalServerError: 500 Internal Server Error
502, 503, 504: Gateway/service unavailable
408: Request Timeout
```

### Non-Retryable Errors

```
AuthenticationError: 401 Invalid API key
BadRequestError: 400 Malformed request
PermissionDeniedError: 403 Insufficient permissions
NotFoundError: 404 Model or endpoint not found
```

### Configuration Errors

```
ValueError: Missing required field (api_key, model)
ValidationError: Invalid parameter value (temperature out of range)
```

## Prerequisites

### API Key

```bash
# Set environment variable
export OPENAI_API_KEY=sk-your-api-key-here
```

### Python Requirements

```bash
# Install serapeum-openai
pip install serapeum-openai

# Or install from source
uv pip install -e libs/providers/openai
```

## Troubleshooting

### Issue: AuthenticationError

```
Solution: Verify your API key
  export OPENAI_API_KEY=sk-your-api-key
```

### Issue: Model Not Found

```
Solution: Check model availability and spelling
  llm = Completions(model="gpt-4o-mini", ...)  # Use a valid model name
```

### Issue: Rate Limiting

```
Solution: Reduce request frequency or increase max_retries
  llm = Completions(model="gpt-4o-mini", max_retries=5, ...)
```

### Issue: Timeout

```
Solution: Increase timeout
  llm = Completions(model="gpt-4o-mini", timeout=120.0, ...)
```

## Next Steps

1. **Start with [Examples](./examples.md)** for practical usage patterns
2. **Review [Sequence Diagrams](./openai_sequence.md)** to understand execution flow
3. **Study [Class Diagram](./openai_class.md)** to understand architecture
4. **Explore [Data Flow](./openai_dataflow.md)** to understand transformations
5. **Check [State Management](./openai_state.md)** for lifecycle details
6. **Read [OpenAI vs Responses API](./openai-vs-responses.md)** for API comparison

## See Also

- [Azure OpenAI Provider](../azure-openai/general.md) -- Azure-hosted OpenAI models
- [TextCompletionLLM](../../core/llms/orchestrators/text_completion_llm/general.md) -- Structured
  completion orchestrator
- [ToolOrchestratingLLM](../../core/llms/orchestrators/tool_orchestrating_llm/general.md) --
  Tool-based orchestrator
- [OpenAI Official Documentation](https://platform.openai.com/docs) -- OpenAI API documentation
