# Component Boundaries and Interactions

This diagram shows how components interact during the complete lifecycle of the Azure OpenAI
LLM provider, highlighting the Azure extension points.

```mermaid
graph TB
    subgraph User Space
        UC[User Code]
        PM[Pydantic Models]
    end

    subgraph Azure OpenAI Provider
        ACOMP[Azure Completions]
        ARESP[Azure Responses]

        subgraph AzureClient Mixin
            AMIXIN[AzureClient]
            AVAL[Azure Validators]
            ACRED[Azure Credential Resolution]
            AFACTORY[Azure Factory Methods]
            AAD[Azure AD Token Refresh]
        end

        subgraph Inherited from OpenAI
            OCOMP[OpenAI Completions Logic]
            ORESP[OpenAI Responses Logic]
            CLIENT[Client Mixin]
            SO[StructuredOutput]
            MM[ModelMetadata]
            RETRY[Retry Decorator]
        end

        subgraph Request Building
            BCRQ[Chat Request Builder]
            BCRQ_MSG[Message Formatter]
            BCRQ_TOOL[Tool Converter]
            MODEL_KW[Model Kwargs - engine swap]
        end

        subgraph Response Parsing
            PRSP[ChatMessageParser]
            PRSP_RESP[ResponsesOutputParser]
            ACC[ToolCallAccumulator]
        end
    end

    subgraph Azure SDK Layer
        ACLI[SyncAzureOpenAI]
        AACLI[AsyncAzureOpenAI]

        subgraph SDK Operations
            CHAT_OP[chat.completions.create]
            RESP_OP[responses.create]
        end
    end

    subgraph Azure OpenAI Service
        SRV[Azure OpenAI Resource]

        subgraph API Endpoints
            EP_CHAT[Chat Completions Endpoint]
            EP_RESP[Responses Endpoint]
        end

        subgraph Deployments
            DEP[Model Deployment]
        end
    end

    subgraph Authentication
        APIKEY[API Key]
        AZUREAD[Azure AD / Entra ID]
        DEFCRED[DefaultAzureCredential]
        TOKPROV[Token Provider Callback]
    end

    subgraph Orchestrator Layer
        TCL[TextCompletionLLM]
        TOL[ToolOrchestratingLLM]
    end

    %% Initialization
    UC -->|1. Initialize| ACOMP
    UC -->|1. Initialize| ARESP
    ACOMP --> AMIXIN
    ARESP --> AMIXIN

    %% Validation pipeline
    AMIXIN --> AVAL
    AVAL -->|1. _validate_azure_env| AVAL
    AVAL -->|2. _resolve_credentials| CLIENT
    AVAL -->|3. _reset_api_base| AMIXIN

    %% Client creation
    ACOMP -->|First use| AFACTORY
    AFACTORY -->|_build_sync_client| ACLI
    AFACTORY -->|_build_async_client| AACLI

    %% Credential resolution
    AFACTORY -->|_get_credential_kwargs| ACRED
    ACRED -->|API Key auth| APIKEY
    ACRED -->|Azure AD auth| AAD
    AAD -->|Custom provider| TOKPROV
    AAD -->|Auto refresh| DEFCRED
    DEFCRED --> AZUREAD

    %% Chat flow
    UC -->|2a. chat(messages)| ACOMP
    ACOMP --> MODEL_KW
    MODEL_KW -->|model -> engine| OCOMP
    OCOMP --> BCRQ
    BCRQ --> BCRQ_MSG
    BCRQ --> RETRY
    RETRY --> ACLI
    ACLI -->|HTTPS| EP_CHAT
    EP_CHAT --> DEP
    DEP -->|Response| ACLI
    ACLI --> PRSP
    PRSP --> UC

    %% Responses flow
    UC -->|2b. chat(messages)| ARESP
    ARESP --> MODEL_KW
    MODEL_KW --> ORESP
    ORESP --> BCRQ
    RETRY --> ACLI
    ACLI -->|HTTPS| EP_RESP
    EP_RESP --> DEP
    ACLI --> PRSP_RESP
    PRSP_RESP --> UC

    %% Tool calling
    UC -->|2c. generate_tool_calls| ACOMP
    ACOMP --> BCRQ_TOOL
    BCRQ_TOOL --> OCOMP

    %% Streaming
    UC -->|2d. chat(stream=True)| ACOMP
    ACOMP --> OCOMP
    OCOMP -->|stream| ACLI
    ACLI -.->|chunks| ACC
    ACC -.->|yield| UC

    %% Orchestrators
    UC -->|3a. TextCompletionLLM| TCL
    TCL --> ACOMP
    UC -->|3b. ToolOrchestratingLLM| TOL
    TOL --> ACOMP

    %% Styling
    classDef userClass fill:#e1f5ff,stroke:#01579b
    classDef azureClass fill:#e0f2f1,stroke:#004d40
    classDef openaiClass fill:#fff9c4,stroke:#f57f17
    classDef sdkClass fill:#f3e5f5,stroke:#4a148c
    classDef serverClass fill:#efebe9,stroke:#3e2723
    classDef authClass fill:#e8f5e9,stroke:#1b5e20
    classDef orchClass fill:#fce4ec,stroke:#880e4f

    class UC,PM userClass
    class ACOMP,ARESP,AMIXIN,AVAL,ACRED,AFACTORY,AAD azureClass
    class OCOMP,ORESP,CLIENT,SO,MM,RETRY,BCRQ,BCRQ_MSG,BCRQ_TOOL,MODEL_KW,PRSP,PRSP_RESP,ACC openaiClass
    class ACLI,AACLI,CHAT_OP,RESP_OP sdkClass
    class SRV,EP_CHAT,EP_RESP,DEP serverClass
    class APIKEY,AZUREAD,DEFCRED,TOKPROV authClass
    class TCL,TOL orchClass
```

## Component Interaction Patterns

### 1. Azure Initialization Pattern

```
User Code
  +-- AzureCompletions.__init__(engine, api_key, azure_endpoint, api_version, ...)
      +-- _validate_azure_env (mode="before"):
      |   +-- Resolve engine from aliases (deployment_name, deployment_id, etc.)
      |   +-- Validate api_version is set
      |   +-- Resolve azure_endpoint from env
      +-- OpenAI.__init__ (inherited):
      |   +-- _validate_model: O1 checks
      |   +-- _resolve_credentials: API key
      +-- _reset_api_base_for_azure (mode="after"):
      |   +-- Clear api_base if default OpenAI URL
      +-- Set _client=None, _async_client=None
```

### 2. Azure Client Creation Pattern

```
User -> AzureCompletions.chat
  +-- Access self.client property (inherited from Client)
      +-- Check if self._client is None
      +-- If None:
          +-- AzureClient._get_credential_kwargs()
          |   +-- _resolve_api_key()
          |   |   +-- If use_azure_ad + token_provider: call provider()
          |   |   +-- If use_azure_ad + no provider: refresh_openai_azure_ad_token()
          |   |   +-- Else: api_key or AZURE_OPENAI_API_KEY env
          |   +-- Build kwargs:
          |       api_key, max_retries=0, timeout, default_headers,
          |       azure_endpoint, azure_deployment, api_version,
          |       azure_ad_token_provider, http_client
          +-- AzureClient._build_sync_client(**kwargs)
          |   +-- Return SyncAzureOpenAI(**kwargs)
          +-- Store in self._client
```

### 3. Azure Request Pattern

```
User -> AzureCompletions.chat(messages, **kwargs)
  +-- AzureClient._get_model_kwargs(**kwargs)
  |   +-- OpenAI._get_model_kwargs(**kwargs)  # inherited
  |   +-- all_kwargs["model"] = self.engine  # swap model for deployment
  +-- OpenAI._chat(messages, **all_kwargs)  # inherited, @retry decorated
  |   +-- to_openai_message_dicts(messages)
  |   +-- self.client.chat.completions.create(**payload)
  |   |   (client is SyncAzureOpenAI, sends to Azure endpoint)
  |   +-- ChatMessageParser(response)
  +-- Return ChatResponse
```

### 4. Azure AD Token Refresh Pattern

```
AzureClient._resolve_api_key()
  +-- If use_azure_ad:
  |   +-- If azure_ad_token_provider set:
  |   |   +-- api_key = azure_ad_token_provider()
  |   +-- Else (auto refresh):
  |       +-- refresh_openai_azure_ad_token(self._azure_ad_token)
  |       |   +-- If token is None or expires in < 60s:
  |       |   |   +-- credential = DefaultAzureCredential()
  |       |   |   +-- token = credential.get_token(
  |       |   |   |       "https://cognitiveservices.azure.com/.default"
  |       |   |   |   )
  |       |   |   +-- Return new AccessToken
  |       |   +-- Else: return existing token
  |       +-- self._azure_ad_token = refreshed_token
  |       +-- api_key = token.token
  +-- Else:
      +-- api_key = self.api_key or os.getenv("AZURE_OPENAI_API_KEY")
```

## Component Dependencies

### Azure Provider Depends On

- `serapeum-openai` (OpenAI Completions, Responses, Client, all parsers)
- `serapeum-core` (BaseLLM, FunctionCallingLLM, Message, etc.)
- `openai` SDK (`AzureOpenAI`, `AsyncAzureOpenAI`)
- `azure-identity` (`DefaultAzureCredential`, `AccessToken`)
- `httpx` (HTTP client)
- `pydantic` (configuration)

### Azure Provider Is Used By

- `TextCompletionLLM` (as the LLM engine)
- `ToolOrchestratingLLM` (as the function-calling LLM)
- Direct user code

### What Azure Adds Over OpenAI

- `AzureClient` mixin: 4 factory method overrides + validation
- `utils.py`: `refresh_openai_azure_ad_token()`, `resolve_from_aliases()`
- Azure-specific environment variable resolution
- Deployment name -> model mapping in `_get_model_kwargs`
