# OpenAI Usage Examples

This guide provides comprehensive examples covering all possible ways to use the OpenAI
provider classes based on real test cases from the codebase.

## Prerequisites: OpenAI API Key

The examples in this guide use the [OpenAI API](https://platform.openai.com/), which requires
an API key.

**Steps to create your API key:**

1. Create an account at [platform.openai.com](https://platform.openai.com) (or sign in)
2. Navigate to [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
3. Click **Create new secret key**
4. Copy the key immediately -- it will not be shown again

**Set the environment variable:**

```bash
export OPENAI_API_KEY=sk-your-api-key-here
```

Or add it to your `.env` file:

```
OPENAI_API_KEY=sk-your-api-key-here
```

**Loading the `.env` file in Python:**

Install [`python-dotenv`](https://pypi.org/project/python-dotenv/):

```bash
pip install python-dotenv
```

Then load it at the top of your script:

```python notest
from dotenv import load_dotenv

load_dotenv()  # loads variables from .env into os.environ
```

All examples below read the key via `os.environ.get("OPENAI_API_KEY")`.

---

## Table of Contents

1. [Basic Usage](#basic-usage)
2. [Initialization Patterns](#initialization-patterns)
3. [Chat Operations](#chat-operations)
4. [Completion Operations](#completion-operations)
5. [Streaming Operations](#streaming-operations)
6. [Tool/Function Calling](#toolfunction-calling)
7. [Structured Outputs](#structured-outputs)
8. [Integration with Orchestrators](#integration-with-orchestrators)
9. [Async Operations](#async-operations)
10. [Responses API](#responses-api)

---

## Basic Usage

### Simple Chat (Completions API)

The most straightforward way to use the OpenAI provider:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Say 'pong'.")])]
response = llm.chat(messages)
print(response.message.content)  # "Pong!"
```

### Simple Completion

Using the completion API (delegates to chat internally):

```python
import os
from serapeum.openai import Completions

llm = Completions(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
)

response = llm.complete("Say 'pong'.")
print(response.text)  # "Pong!"
```

---

## Initialization Patterns

### 1. Basic Initialization

Minimal configuration:

```python
import os
from serapeum.openai import Completions

llm = Completions(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
)
```

### 2. Full Configuration (Completions)

With all common parameters:

```python
import os
from serapeum.openai import Completions

llm = Completions(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    temperature=0.8,
    max_tokens=1024,
    timeout=120.0,
    max_retries=5,
    strict=True,                    # Strict JSON-schema for tools
    additional_kwargs={"top_p": 0.9},
)
```

### 3. O1 Reasoning Model

With reasoning effort configuration:

```python
import os
from serapeum.openai import Completions

llm = Completions(
    model="o1-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    reasoning_effort="medium",      # "none", "minimal", "low", "medium", "high", "xhigh"
)
# Note: temperature is forced to 1.0 for O1 models
```

### 4. Responses API Initialization

With Responses-specific features:

```python
import os
from serapeum.openai import Responses

llm = Responses(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    instructions="You are a helpful assistant.",  # System-level instructions
    store=True,                                   # Server-side persistence
)
```

### 5. Responses API with Built-in Tools

```python
import os
from serapeum.openai import Responses

llm = Responses(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    built_in_tools=[
        {"type": "web_search_preview"},
    ],
)
```

### 6. Custom HTTP Headers

```python
import os
from serapeum.openai import Completions

llm = Completions(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    default_headers={"X-Custom-Header": "value"},
)
```

---

## Chat Operations

### 1. Single Turn Chat

Basic conversation:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

messages = [
    Message(role=MessageRole.USER, chunks=[TextChunk(content="What is 2+2?")])
]

response = llm.chat(messages)
print(response.message.content)  # "4"
```

### 2. Multi-turn Conversation

With conversation history:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

messages = [
    Message(role=MessageRole.SYSTEM, chunks=[TextChunk(content="You are a helpful math tutor.")]),
    Message(role=MessageRole.USER, chunks=[TextChunk(content="What is 2+2?")]),
    Message(role=MessageRole.ASSISTANT, chunks=[TextChunk(content="2+2 equals 4.")]),
    Message(role=MessageRole.USER, chunks=[TextChunk(content="What about 3+3?")]),
]

response = llm.chat(messages)
print(response.message.content)  # "3+3 equals 6."
```

### 3. Chat with Parameters

Passing custom parameters:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Write a creative story.")])]

response = llm.chat(
    messages,
    temperature=0.9,
    max_tokens=500,
)
```

---

## Completion Operations

### 1. Basic Completion

Simple text completion (uses chat under the hood via ChatToCompletion adapter):

```python
import os
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

prompt = "The capital of France is"
response = llm.complete(prompt)
print(response.text)  # "Paris"
```

### 2. Completion with Parameters

Custom generation settings:

```python
import os
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

response = llm.complete(
    "Once upon a time",
    temperature=0.8,
    max_tokens=200,
)
print(response.text)
```

---

## Streaming Operations

### 1. Stream Chat

Real-time streaming chat:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Count from 1 to 5.")])]

for chunk in llm.chat(messages, stream=True):
    print(chunk.delta, end="", flush=True)
# Outputs: "1" ", 2" ", 3" ", 4" ", 5"
```

### 2. Stream Completion

Real-time streaming completion:

```python
import os
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

for chunk in llm.complete("Write a haiku about coding:", stream=True):
    print(chunk.delta, end="", flush=True)
```

### 3. Processing Stream with Delta

Access incremental content:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Tell me a joke.")])]

full_response = ""
for chunk in llm.chat(messages, stream=True):
    delta = chunk.delta
    if delta:
        full_response += delta
        print(delta, end="", flush=True)

print(f"\n\nFull response: {full_response}")
```

---

## Tool/Function Calling

### 1. Basic Tool Calling

Using tools with Completions:

```python
import os
from pydantic import BaseModel, Field
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.core.tools import CallableTool
from serapeum.openai import Completions


class Album(BaseModel):
    """A music album."""
    title: str = Field(description="Album title")
    artist: str = Field(description="Artist name")
    songs: list[str] = Field(description="List of song titles")


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

tool = CallableTool.from_model(Album)

message = Message(
    role=MessageRole.USER,
    chunks=[TextChunk(content="Create a rock album with two songs")],
)

response = llm.generate_tool_calls(tools=[tool], message=message)
tool_calls = llm.get_tool_calls_from_response(response)

for tool_call in tool_calls:
    result = tool.call(**tool_call.tool_kwargs)
    print(result)  # Album instance
```

### 2. Tool with Required Mode

Force the model to always use a tool:

```python
import os
from pydantic import BaseModel
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.core.tools import CallableTool
from serapeum.openai import Completions


class Album(BaseModel):
    title: str
    artist: str
    songs: list[str]


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))
tool = CallableTool.from_model(Album)

message = Message(
    role=MessageRole.USER,
    chunks=[TextChunk(content="Create an album")],
)

response = llm.generate_tool_calls(
    tools=[tool],
    message=message,
    tool_required=True,  # tool_choice="required"
)

tool_calls = llm.get_tool_calls_from_response(response)
print(len(tool_calls))  # >= 1 (guaranteed)
```

### 3. Strict Mode Tool Calling

Enable strict JSON-schema validation for tools:

```python
import os
from pydantic import BaseModel
from serapeum.core.tools import CallableTool
from serapeum.openai import Completions


class MathResult(BaseModel):
    operation: str
    result: float


llm = Completions(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    strict=True,  # Enables strict mode for all tools
)

tool = CallableTool.from_model(MathResult)
# strict=True adds additionalProperties=false to tool schema
```

### 4. Parallel Tool Calls

Allow multiple tool calls in a single response:

```python
import os
from pydantic import BaseModel
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.core.tools import CallableTool
from serapeum.openai import Completions


class Album(BaseModel):
    title: str
    artist: str
    songs: list[str]


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))
tool = CallableTool.from_model(Album)

message = Message(
    role=MessageRole.USER,
    chunks=[TextChunk(content="Create two albums: one rock and one jazz")],
)

response = llm.generate_tool_calls(
    tools=[tool],
    message=message,
    allow_parallel_tool_calls=True,
)

tool_calls = llm.get_tool_calls_from_response(response)
print(len(tool_calls))  # 2 (if model returns parallel calls)
```

### 5. Tool Calling from Function

Create tools from Python functions:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.core.tools import CallableTool
from serapeum.openai import Completions


def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))
tool = CallableTool.from_function(add)

message = Message(
    role=MessageRole.USER,
    chunks=[TextChunk(content="What is 2 + 3?")],
)

response = llm.generate_tool_calls(tools=[tool], message=message)
tool_calls = llm.get_tool_calls_from_response(response)

for tc in tool_calls:
    result = tool.call(**tc.tool_kwargs)
    print(result)  # 5
```

---

## Structured Outputs

### 1. Native Structured Output (parse)

Use `parse()` for native JSON-schema structured outputs on supported models:

```python
import os
from pydantic import BaseModel, Field
from serapeum.openai import Completions


class Person(BaseModel):
    name: str = Field(description="The person's name")
    age: int = Field(description="The person's age")


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

result = llm.parse(
    output_cls=Person,
    prompt="Create a fictional person",
)
print(result)  # Person(name="Alice", age=30)
```

### 2. Async Structured Output

```python
import asyncio
import os
from pydantic import BaseModel
from serapeum.openai import Completions


class Person(BaseModel):
    name: str
    age: int


async def main():
    llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))
    result = await llm.aparse(
        output_cls=Person,
        prompt="Create a fictional person",
    )
    print(result)


asyncio.run(main())
```

---

## Integration with Orchestrators

### 1. With TextCompletionLLM

Use Completions with `TextCompletionLLM` for structured outputs:

```python
import os
from pydantic import BaseModel
from serapeum.core.output_parsers import PydanticParser
from serapeum.core.llms import TextCompletionLLM
from serapeum.openai import Completions


class DummyModel(BaseModel):
    value: str


llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

parser = PydanticParser(output_cls=DummyModel)

text_llm = TextCompletionLLM(
    output_parser=parser,
    prompt="Value: {value}",
    llm=llm,
)

result = text_llm(value="input")
print(result.value)  # "input"
```

### 2. With ToolOrchestratingLLM

Use Completions with `ToolOrchestratingLLM` for tool-based workflows:

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
print(result.title)
print(result.artist)
print(result.songs)
```

### 3. Streaming with ToolOrchestratingLLM

Stream tool execution results:

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
    prompt="Create albums about {topic}",
    llm=llm,
    allow_parallel_tool_calls=False,
)

for album in tools_llm(topic="rock", stream=True):
    print(f"Received: {album.title}")
```

---

## Async Operations

### 1. Async Chat

Non-blocking chat:

```python
import asyncio
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions


async def async_chat_example():
    llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

    messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello!")])]
    response = await llm.achat(messages)
    print(response.message.content)


asyncio.run(async_chat_example())
```

### 2. Async Completion

Non-blocking completion:

```python
import asyncio
import os
from serapeum.openai import Completions


async def async_complete_example():
    llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

    response = await llm.acomplete("Say hello")
    print(response.text)


asyncio.run(async_complete_example())
```

### 3. Async Streaming Chat

Non-blocking streaming:

```python
import asyncio
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions


async def async_stream_example():
    llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

    messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Count to 5")])]

    async for chunk in await llm.achat(messages, stream=True):
        print(chunk.delta, end="", flush=True)


asyncio.run(async_stream_example())
```

### 4. Concurrent Async Requests

Process multiple requests concurrently:

```python
import asyncio
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions


async def process_multiple():
    llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

    prompts = ["What is 2+2?", "What is 3+3?", "What is 4+4?"]

    tasks = [
        llm.achat([Message(role=MessageRole.USER, chunks=[TextChunk(content=prompt)])])
        for prompt in prompts
    ]

    responses = await asyncio.gather(*tasks)

    for prompt, response in zip(prompts, responses):
        print(f"{prompt} -> {response.message.content}")


asyncio.run(process_multiple())
```

### 5. Async with ToolOrchestratingLLM

Async tool orchestration:

```python
import asyncio
import os
from pydantic import BaseModel
from serapeum.core.llms import ToolOrchestratingLLM
from serapeum.openai import Completions


class Album(BaseModel):
    title: str
    artist: str
    songs: list[str]


async def async_tool_example():
    llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

    tools_llm = ToolOrchestratingLLM(
        schema=Album,
        prompt="Create an album about {topic}",
        llm=llm,
    )

    result = await tools_llm.acall(topic="pop")
    print(result.title)


asyncio.run(async_tool_example())
```

---

## Responses API

### 1. Basic Responses Chat

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Responses

llm = Responses(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    instructions="You are a helpful assistant.",
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello!")])]
response = llm.chat(messages)
print(response.message.content)
```

### 2. Responses with Built-in Tools

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Responses

llm = Responses(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    built_in_tools=[{"type": "web_search_preview"}],
)

messages = [
    Message(
        role=MessageRole.USER,
        chunks=[TextChunk(content="What are the latest news about AI?")],
    )
]
response = llm.chat(messages)
print(response.message.content)
```

### 3. Responses with Stateful Conversation

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Responses

llm = Responses(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    track_previous_responses=True,  # Enables server-side conversation state
    store=True,
)

# First turn
messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="My name is Alice")])]
response = llm.chat(messages)

# Second turn (server remembers context)
messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="What is my name?")])]
response = llm.chat(messages)
print(response.message.content)  # Should reference "Alice"
```

### 4. Responses Streaming

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Responses

llm = Responses(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Tell me a story.")])]

for chunk in llm.chat(messages, stream=True):
    print(chunk.delta, end="", flush=True)
```

---

## Best Practices

### 1. Reuse LLM Instances

Create once, use many times:

```python notest
import os
from serapeum.openai import Completions
from serapeum.core.llms import Message, MessageRole, TextChunk

# Good: Create once
llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

# Reuse for multiple calls
response1 = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="hi")])])
response2 = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="bye")])])
```

### 2. Use Appropriate Model

Choose the right model for your use case:

```python
import os
from serapeum.openai import Completions, Responses

# Fast and cheap
quick_llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

# High quality
quality_llm = Completions(model="gpt-4o", api_key=os.environ.get("OPENAI_API_KEY"))

# Reasoning
reasoning_llm = Completions(
    model="o1-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    reasoning_effort="medium",
)

# Responses API for built-in tools
search_llm = Responses(
    model="gpt-4o-mini",
    api_key=os.environ.get("OPENAI_API_KEY"),
    built_in_tools=[{"type": "web_search_preview"}],
)
```

### 3. Handle Errors Gracefully

Always handle potential errors:

```python
import os
import openai
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

try:
    response = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello")])])
except openai.AuthenticationError:
    print("Invalid API key")
except openai.RateLimitError:
    print("Rate limited -- retry with framework handles this automatically")
except openai.APIConnectionError:
    print("Could not connect to OpenAI")
except Exception as e:
    print(f"Unexpected error: {e}")
```

### 4. Monitor Response Metadata

Use metadata for monitoring:

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.openai import Completions

llm = Completions(model="gpt-4o-mini", api_key=os.environ.get("OPENAI_API_KEY"))

response = llm.chat([Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello")])])

print(f"Prompt tokens: {response.additional_kwargs.get('prompt_tokens')}")
print(f"Completion tokens: {response.additional_kwargs.get('completion_tokens')}")
print(f"Total tokens: {response.additional_kwargs.get('total_tokens')}")
```

---

## See Also

- [Execution Flow and Method Calls](./openai_sequence.md) -- Detailed sequence diagrams
- [Architecture and Class Relationships](./openai_class.md) -- Class structure
- [Data Transformations and Validation](./openai_dataflow.md) -- Data flow details
- [Component Boundaries and Interactions](./openai_components.md) -- System components
- [Lifecycle and State Management](./openai_state.md) -- State management
- [OpenAI vs Responses API](./openai-vs-responses.md) -- API comparison
