# Azure OpenAI Usage Examples

This guide provides comprehensive examples covering all possible ways to use the Azure OpenAI
provider classes based on real test cases from the codebase.

## Prerequisites: Azure OpenAI Setup

### 1. Create Azure OpenAI Resource

1. Go to [Azure Portal](https://portal.azure.com)
2. Create an **Azure OpenAI** resource
3. Deploy a model (e.g., `gpt-4o`) and note the **deployment name**
4. Copy the **endpoint URL** and **API key** from the resource

### 2. Set Environment Variables

```bash
export AZURE_OPENAI_API_KEY=your-azure-api-key
export AZURE_OPENAI_ENDPOINT=https://YOUR_RESOURCE.openai.azure.com/
export OPENAI_API_VERSION=2024-02-01
```

Or add to your `.env` file:

```
AZURE_OPENAI_API_KEY=your-azure-api-key
AZURE_OPENAI_ENDPOINT=https://YOUR_RESOURCE.openai.azure.com/
OPENAI_API_VERSION=2024-02-01
```

**Loading the `.env` file in Python:**

```python
from dotenv import load_dotenv

load_dotenv()
```

---

## Table of Contents

1. [Initialization Patterns](#initialization-patterns)
2. [Chat Operations](#chat-operations)
3. [Completion Operations](#completion-operations)
4. [Streaming Operations](#streaming-operations)
5. [Tool/Function Calling](#toolfunction-calling)
6. [Structured Outputs](#structured-outputs)
7. [Integration with Orchestrators](#integration-with-orchestrators)
8. [Async Operations](#async-operations)
9. [Azure Responses API](#azure-responses-api)
10. [Authentication Methods](#authentication-methods)

---

## Initialization Patterns

### 1. Basic with API Key

Minimal Azure configuration:

```python
import os
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",        # Azure deployment name
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

### 2. With Explicit Model

Specify model for capability detection:

```python
import os
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",                      # Used for capability checks
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
    temperature=0.7,
    max_tokens=1024,
)
```

### 3. Using Deployment Aliases

The `engine` parameter accepts several alias names:

```python
import os
from serapeum.azure_openai import Completions

# All of these are equivalent:
llm = Completions(
    engine="my-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

# Using deployment_name alias
llm = Completions(
    deployment_name="my-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

# Using azure_deployment alias
llm = Completions(
    azure_deployment="my-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

### 4. With Microsoft Entra ID (Azure AD)

Token-based authentication:

```python
import os
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from serapeum.azure_openai import Completions

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential,
    "https://cognitiveservices.azure.com/.default",
)

llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",
    azure_ad_token_provider=token_provider,
    use_azure_ad=True,
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

### 5. Auto Azure AD (No Custom Provider)

Automatic token refresh via DefaultAzureCredential:

```python
import os
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",
    use_azure_ad=True,                   # Auto-uses DefaultAzureCredential
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

### 6. Full Configuration

With all common parameters:

```python
import os
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
    temperature=0.7,
    max_tokens=2048,
    timeout=120.0,
    max_retries=5,
    strict=True,
    default_headers={"X-Custom-Header": "value"},
)
```

---

## Chat Operations

### 1. Single Turn Chat

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

messages = [
    Message(role=MessageRole.USER, chunks=[TextChunk(content="What is 2+2?")])
]

response = llm.chat(messages)
print(response.message.content)  # "4"
```

### 2. Multi-turn Conversation

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

messages = [
    Message(role=MessageRole.SYSTEM, chunks=[TextChunk(content="You are a helpful assistant.")]),
    Message(role=MessageRole.USER, chunks=[TextChunk(content="My name is Alice.")]),
    Message(role=MessageRole.ASSISTANT, chunks=[TextChunk(content="Hello Alice!")]),
    Message(role=MessageRole.USER, chunks=[TextChunk(content="What is my name?")]),
]

response = llm.chat(messages)
print(response.message.content)  # "Your name is Alice."
```

---

## Completion Operations

### 1. Basic Completion

```python
import os
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

response = llm.complete("The capital of France is")
print(response.text)  # "Paris"
```

---

## Streaming Operations

### 1. Stream Chat

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Count from 1 to 5.")])]

for chunk in llm.chat(messages, stream=True):
    print(chunk.delta, end="", flush=True)
```

### 2. Stream Completion

```python
import os
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

for chunk in llm.complete("Tell me a joke:", stream=True):
    print(chunk.delta, end="", flush=True)
```

**Note**: Azure may send empty filter result chunks during streaming. The provider handles
these automatically by skipping chunks with empty `choices`.

---

## Tool/Function Calling

### 1. Basic Tool Calling

```python
import os
from pydantic import BaseModel, Field
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.core.tools import CallableTool
from serapeum.azure_openai import Completions


class Album(BaseModel):
    """A music album."""
    title: str = Field(description="Album title")
    artist: str = Field(description="Artist name")
    songs: list[str] = Field(description="List of song titles")


llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

tool = CallableTool.from_model(Album)

message = Message(
    role=MessageRole.USER,
    chunks=[TextChunk(content="Create a rock album with two songs")],
)

response = llm.generate_tool_calls(tools=[tool], message=message)
tool_calls = llm.get_tool_calls_from_response(response)

for tc in tool_calls:
    result = tool.call(**tc.tool_kwargs)
    print(result)  # Album instance
```

### 2. Tool Calling from Function

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.core.tools import CallableTool
from serapeum.azure_openai import Completions


def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b


llm = Completions(
    engine="my-gpt4o-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

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

### 1. Native Structured Output

```python
import os
from pydantic import BaseModel, Field
from serapeum.azure_openai import Completions


class Person(BaseModel):
    name: str = Field(description="The person's name")
    age: int = Field(description="The person's age")


llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

result = llm.parse(
    output_cls=Person,
    prompt="Create a fictional person",
)
print(result)  # Person(name="Alice", age=30)
```

---

## Integration with Orchestrators

### 1. With TextCompletionLLM

```python
import os
from pydantic import BaseModel
from serapeum.core.output_parsers import PydanticParser
from serapeum.core.llms import TextCompletionLLM
from serapeum.azure_openai import Completions


class DummyModel(BaseModel):
    value: str


llm = Completions(
    engine="my-gpt4o-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

text_llm = TextCompletionLLM(
    output_parser=PydanticParser(output_cls=DummyModel),
    prompt="Value: {value}",
    llm=llm,
)

result = text_llm(value="input")
print(result.value)
```

### 2. With ToolOrchestratingLLM

```python
import os
from pydantic import BaseModel
from serapeum.core.llms import ToolOrchestratingLLM
from serapeum.azure_openai import Completions


class Album(BaseModel):
    title: str
    artist: str
    songs: list[str]


llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

tools_llm = ToolOrchestratingLLM(
    schema=Album,
    prompt="Create an album about {topic} with two random songs",
    llm=llm,
)

result = tools_llm(topic="rock")
print(result.title)
print(result.songs)
```

---

## Async Operations

### 1. Async Chat

```python
import asyncio
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Completions


async def main():
    llm = Completions(
        engine="my-gpt4o-deployment",
        api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
        azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
        api_version=os.environ.get("OPENAI_API_VERSION"),
    )

    messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello!")])]
    response = await llm.achat(messages)
    print(response.message.content)


asyncio.run(main())
```

### 2. Async Streaming

```python
import asyncio
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Completions


async def main():
    llm = Completions(
        engine="my-gpt4o-deployment",
        api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
        azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
        api_version=os.environ.get("OPENAI_API_VERSION"),
    )

    messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Count to 5")])]

    async for chunk in await llm.achat(messages, stream=True):
        print(chunk.delta, end="", flush=True)


asyncio.run(main())
```

### 3. Concurrent Async Requests

```python
import asyncio
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Completions


async def main():
    llm = Completions(
        engine="my-gpt4o-deployment",
        api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
        azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
        api_version=os.environ.get("OPENAI_API_VERSION"),
    )

    prompts = ["What is 2+2?", "What is 3+3?", "What is 4+4?"]
    tasks = [
        llm.achat([Message(role=MessageRole.USER, chunks=[TextChunk(content=p)])])
        for p in prompts
    ]

    responses = await asyncio.gather(*tasks)
    for prompt, response in zip(prompts, responses):
        print(f"{prompt} -> {response.message.content}")


asyncio.run(main())
```

---

## Azure Responses API

### 1. Basic Responses

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Responses

llm = Responses(
    engine="my-responses-deployment",
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
    instructions="You are a helpful assistant.",
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello!")])]
response = llm.chat(messages)
print(response.message.content)
```

### 2. Responses Streaming

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Responses

llm = Responses(
    engine="my-responses-deployment",
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Tell me a story.")])]

for chunk in llm.chat(messages, stream=True):
    print(chunk.delta, end="", flush=True)
```

---

## Authentication Methods

### Method 1: API Key (Recommended for Development)

```python
import os
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

### Method 2: Azure AD with Custom Token Provider

```python
import os
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from serapeum.azure_openai import Completions

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://cognitiveservices.azure.com/.default"
)

llm = Completions(
    engine="my-deployment",
    azure_ad_token_provider=token_provider,
    use_azure_ad=True,
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

### Method 3: Auto Azure AD (DefaultAzureCredential)

```python
import os
from serapeum.azure_openai import Completions

# Automatically uses DefaultAzureCredential for token refresh
llm = Completions(
    engine="my-deployment",
    use_azure_ad=True,
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

---

## Best Practices

### 1. Use Environment Variables for Credentials

```python notest
import os
from serapeum.azure_openai import Completions

# Good: credentials from environment
llm = Completions(
    engine="my-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

# Bad: hardcoded credentials
llm = Completions(
    engine="my-deployment",
    api_key="hardcoded-key",  # NEVER do this
    azure_endpoint="https://resource.openai.azure.com/",
    api_version="2024-02-01",
)
```

### 2. Reuse LLM Instances

```python
import os
from serapeum.azure_openai import Completions

# Good: create once, reuse
llm = Completions(
    engine="my-deployment",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
# Use llm for all requests
```

### 3. Use Azure AD for Production

```python
import os
from azure.identity import ManagedIdentityCredential, get_bearer_token_provider
from serapeum.azure_openai import Completions

# Production: use Managed Identity
credential = ManagedIdentityCredential()
token_provider = get_bearer_token_provider(
    credential, "https://cognitiveservices.azure.com/.default"
)

llm = Completions(
    engine="my-deployment",
    azure_ad_token_provider=token_provider,
    use_azure_ad=True,
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)
```

---

## See Also

- [OpenAI Provider](../openai/general.md) -- Base OpenAI provider documentation
- [OpenAI Examples](../openai/examples.md) -- Additional examples (all apply to Azure too)
- [Azure OpenAI Class Diagram](./azure_openai_class.md) -- Architecture overview
- [Azure OpenAI Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/) --
  Microsoft docs
