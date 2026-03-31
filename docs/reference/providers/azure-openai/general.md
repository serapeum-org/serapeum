# Azure OpenAI LLM Integration

This directory contains comprehensive documentation explaining the complete workflow of the
Azure OpenAI provider classes, from initialization to execution across various modes (chat,
completion, streaming, tool calling, structured outputs, async).

## Overview

The Azure OpenAI provider extends the base OpenAI provider with Azure-specific features:

1. **`Completions`** -- Azure Chat Completions API (deployment-based model access)
2. **`Responses`** -- Azure Responses API (deployment-based, modern reasoning models)

Both classes inherit all capabilities from their OpenAI parent classes and add:

1. **Azure deployment management** (engine/deployment name mapping)
2. **Dual authentication** (API key or Microsoft Entra ID / Azure AD)
3. **Azure-specific credential resolution** (environment variables, token refresh)
4. **Template Method pattern** for SDK client creation (SyncAzureOpenAI / AsyncAzureOpenAI)
5. **Full feature parity** with OpenAI provider (chat, streaming, tools, structured outputs)

## Example Usage

### Basic Chat with API Key

```python
import os
from serapeum.core.llms import Message, MessageRole, TextChunk
from serapeum.azure_openai import Completions

llm = Completions(
    engine="my-gpt4o-deployment",
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Say 'pong'.")])]
response = llm.chat(messages)
print(response.message.content)  # "Pong!"
```

### With Microsoft Entra ID (Azure AD)

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

messages = [Message(role=MessageRole.USER, chunks=[TextChunk(content="Hello!")])]
response = llm.chat(messages)
print(response.message.content)
```

### With TextCompletionLLM

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
    model="gpt-4o",
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
    api_version=os.environ.get("OPENAI_API_VERSION"),
)

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
# Returns: Album(title="...", artist="...", songs=[...])
```

## Understanding the Workflow

### 1. [Execution Flow and Method Calls](./azure_openai_sequence.md)

Shows the chronological flow of method calls, focusing on Azure-specific credential resolution
and client creation.

**Best for**:

- Understanding Azure-specific initialization
- Seeing credential resolution flow (API key vs Azure AD)
- Debugging deployment configuration
- Understanding how Azure classes delegate to OpenAI parent

### 2. [Architecture and Class Relationships](./azure_openai_class.md)

Illustrates the static structure, showing how AzureClient extends the base OpenAI classes.

**Best for**:

- Understanding the Azure extension pattern
- Seeing factory method overrides
- Identifying Azure-specific configuration
- Understanding the AzureClient mixin

### 3. [Data Transformations and Validation](./azure_openai_dataflow.md)

Tracks how data transforms through the system, highlighting Azure-specific validation.

**Best for**:

- Understanding Azure credential resolution
- Seeing the multi-stage validation pipeline
- Understanding deployment name -> model mapping
- Token refresh flow for Azure AD

### 4. [Component Boundaries and Interactions](./azure_openai_components.md)

Shows component boundaries with Azure-specific components highlighted.

**Best for**:

- Understanding Azure extension points
- Seeing the AzureClient mixin role
- Understanding credential providers
- Comparing with base OpenAI architecture

### 5. [Lifecycle and State Management](./azure_openai_state.md)

Depicts lifecycle states with Azure-specific initialization stages.

**Best for**:

- Understanding Azure credential lifecycle
- Seeing token refresh timing
- Understanding API version requirements
- Event-loop safety with Azure clients

### 6. [Usage Examples](./examples.md)

Comprehensive examples covering Azure-specific configuration patterns.

**Best for**:

- Learning Azure-specific setup
- Understanding deployment configuration
- Seeing authentication methods
- Integration patterns

## Core Capabilities

All capabilities from the OpenAI provider are inherited, plus:

### 1. Deployment Management

```
Azure-specific model access via deployments:
- engine parameter maps to Azure deployment name
- model parameter used for capability checks
- Aliases: deployment_name, deployment_id, deployment, azure_deployment
```

### 2. Dual Authentication

```
API Key:
- Via api_key parameter or AZURE_OPENAI_API_KEY env var
- Fallback to OPENAI_API_KEY env var

Microsoft Entra ID (Azure AD):
- Via azure_ad_token_provider callback
- Auto-refresh via DefaultAzureCredential
- Token validity checking (60s buffer)
```

### 3. Azure-Specific Credential Resolution

```
Multi-stage validation pipeline:
1. _validate_azure_env (mode="before"): engine aliases, api_version, endpoint
2. _resolve_credentials (mode="after"): inherited API key resolution
3. _reset_api_base_for_azure (mode="after"): reset default OpenAI URL
```

## Key Design Patterns

### 1. Template Method Pattern (Client Creation)

Azure overrides factory methods from the Client mixin:

```python notest
# AzureClient overrides
def _build_sync_client(self, **kwargs: Any) -> SyncAzureOpenAI:
    return SyncAzureOpenAI(**kwargs)

def _build_async_client(self, **kwargs: Any) -> AsyncAzureOpenAI:
    return AsyncAzureOpenAI(**kwargs)
```

### 2. Extended Credential Resolution

```python notest
def _get_credential_kwargs(self, is_async: bool = False) -> dict[str, Any]:
    # Resolve API key (supports Azure AD token refresh)
    self._resolve_api_key()

    # Build kwargs with Azure-specific fields
    kwargs = {
        **parent_kwargs,
        "azure_endpoint": self.azure_endpoint,
        "azure_deployment": self.azure_deployment,
        "azure_ad_token_provider": self.azure_ad_token_provider,
        "api_version": self.api_version,
    }
    return kwargs
```

### 3. Deployment Name Mapping

```python notest
def _get_model_kwargs(self, **kwargs: Any) -> dict[str, Any]:
    all_kwargs = super()._get_model_kwargs(**kwargs)
    # Swap model -> engine for Azure deployments
    all_kwargs["model"] = self.engine
    return all_kwargs
```

## Integration Architecture

```
User Application
    |
ToolOrchestratingLLM / TextCompletionLLM
    |
Azure Completions / Azure Responses
    |                |
    |     AzureClient mixin (factory overrides)
    |                |
    +-- OpenAI Completions / OpenAI Responses (inherited behavior)
    |
Client mixin (SyncAzureOpenAI / AsyncAzureOpenAI)
    |
Azure OpenAI API (HTTPS)
    |
Azure Model Deployment (GPT-4o, O3, etc.)
```

## Configuration Options

### Azure-Specific

- `engine`: Azure deployment name (required; aliases: `deployment_name`, `deployment_id`,
  `deployment`, `azure_deployment`)
- `azure_endpoint`: Azure OpenAI resource URL (e.g., `https://YOUR_RESOURCE.openai.azure.com/`)
- `api_version`: Azure API version string (e.g., `"2024-02-01"`, required)
- `use_azure_ad`: Authenticate via Microsoft Entra ID (default: False)
- `azure_ad_token_provider`: Callback returning bearer token for Azure AD

### Inherited from OpenAI

- `model`: Model name for capability checks (default: `"gpt-35-turbo"`)
- `temperature`: Sampling temperature (0.0-2.0)
- `max_tokens`: Maximum generation tokens
- `timeout`: HTTP timeout in seconds (default: 60.0)
- `max_retries`: Retry attempts (default: 3)
- `strict`: Strict JSON-schema mode for tools
- All other OpenAI Completions/Responses parameters

## Environment Variables

### Required

- `OPENAI_API_VERSION`: API version (e.g., `"2024-02-01"`)
- `AZURE_OPENAI_ENDPOINT`: Azure resource URL

### Authentication (one of)

- `AZURE_OPENAI_API_KEY`: API key (or `OPENAI_API_KEY` as fallback)
- Azure AD credentials (via `DefaultAzureCredential`)

### Optional

- `AZURE_OPENAI_COMPLETIONS_ENGINE`: Default Completions deployment name
- `AZURE_OPENAI_RESPONSES_ENGINE`: Default Responses deployment name

## Error Handling

All error handling from the OpenAI provider is inherited, plus:

### Azure-Specific Errors

```
ValueError: Missing engine/deployment name
ValueError: Missing api_version
ValueError: Default OpenAI base URL without azure_endpoint
ClientAuthenticationError: Azure AD token acquisition failure (wrapped in ValueError)
```

## Prerequisites

### Azure Setup

1. Create an Azure OpenAI resource in the Azure portal
2. Deploy a model (e.g., `gpt-4o`) and note the deployment name
3. Get the resource endpoint URL and API key

### Environment Variables

```bash
export AZURE_OPENAI_API_KEY=your-azure-api-key
export AZURE_OPENAI_ENDPOINT=https://YOUR_RESOURCE.openai.azure.com/
export OPENAI_API_VERSION=2024-02-01
```

### Python Requirements

```bash
# Install serapeum-azure-openai
pip install serapeum-azure-openai

# Dependencies (automatically installed):
#   serapeum-core, serapeum-openai, azure-identity, httpx
```

## Troubleshooting

### Issue: Missing api_version

```
Solution: Set the API version
  export OPENAI_API_VERSION=2024-02-01
```

### Issue: Default OpenAI URL with Azure

```
Solution: Set azure_endpoint
  llm = Completions(
      engine="my-deployment",
      azure_endpoint="https://YOUR_RESOURCE.openai.azure.com/",
      ...
  )
```

### Issue: Azure AD Token Failure

```
Solution: Ensure Azure credentials are configured
  az login  # or set AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID
```

### Issue: Deployment Not Found

```
Solution: Check deployment name matches Azure portal
  llm = Completions(engine="exact-deployment-name", ...)
```

## Next Steps

1. **Start with [Examples](./examples.md)** for practical usage patterns
2. **Review [Sequence Diagrams](./azure_openai_sequence.md)** for execution flow
3. **Study [Class Diagram](./azure_openai_class.md)** for architecture
4. **Explore [Data Flow](./azure_openai_dataflow.md)** for transformations
5. **Check [State Management](./azure_openai_state.md)** for lifecycle details

## See Also

- [OpenAI Provider](../openai/general.md) -- Base OpenAI provider (parent classes)
- [TextCompletionLLM](../../core/llms/orchestrators/text_completion_llm/general.md) -- Structured
  completion orchestrator
- [ToolOrchestratingLLM](../../core/llms/orchestrators/tool_orchestrating_llm/general.md) --
  Tool-based orchestrator
- [Azure OpenAI Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/) --
  Microsoft Azure OpenAI docs
