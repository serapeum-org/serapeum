# Component Boundaries and Interactions

This diagram shows how components interact during the complete lifecycle of the OpenAI LLM
provider.

```mermaid
graph TB
    subgraph User Space
        UC[User Code]
        PM[Pydantic Models: Person, Album]
    end

    subgraph OpenAI Core
        COMP[Completions Instance]
        RESP[Responses Instance]

        subgraph Configuration
            CFG[Configuration Fields]
            MD[Metadata]
            MODREG[Model Registry - models.yaml]
        end

        subgraph Client Management
            CL[Client Property]
            ACL[AsyncClient Property]
            LINIT[Lazy Initialization]
            CRED[Credential Resolution]
            FACTORY[Factory Methods]
        end

        subgraph Request Building
            BCRQ[Chat Request Builder]
            BCRQ_MSG[Message Formatter - to_openai_message_dicts]
            BCRQ_OPT[Model Kwargs Builder]
            BCRQ_TOOL[Tool Converter - to_openai_tool]
            BCRQ_FMT[Response Format Builder]
        end

        subgraph Response Parsing
            PRSP_CHAT[ChatMessageParser]
            PRSP_RESP[ResponsesOutputParser]
            PRSP_TOOL[Tool Call Extractor]
            PRSP_META[Token Count Extractor]
            PRSP_LOG[LogProbParser]
        end

        subgraph Stream Handling
            STRM_CHAT[Chat Stream Handler]
            STRM_RESP[Responses Stream Accumulator]
            ACC[ToolCallAccumulator]
        end

        subgraph Tool Handling
            PREP[_prepare_chat_with_tools]
            VAL[_validate_chat_with_tools_response]
            CHOICE[resolve_tool_choice]
        end

        subgraph Structured Output
            SO[StructuredOutput mixin]
            SO_CHECK[_should_use_structure_outputs]
            SO_SCHEMA[_prepare_schema]
            SO_PARSE[parse / aparse]
        end

        subgraph Retry Layer
            RETRY[retry decorator]
            RETRYABLE[is_retryable predicate]
        end
    end

    subgraph OpenAI SDK Layer
        CLI[SyncOpenAI Client]
        ACLI[AsyncOpenAI Client]

        subgraph SDK Operations
            CHAT_OP[chat.completions.create]
            COMP_OP[completions.create]
            RESP_OP[responses.create]
        end
    end

    subgraph OpenAI API
        SRV[OpenAI API Servers]

        subgraph API Endpoints
            EP_CHAT["/v1/chat/completions"]
            EP_COMP["/v1/completions"]
            EP_RESP["/v1/responses"]
        end

        subgraph Model Runtime
            MDL[Model: GPT-4o, O3, etc.]
        end
    end

    subgraph Orchestrator Layer
        TCL[TextCompletionLLM]
        TOL[ToolOrchestratingLLM]

        subgraph Orchestrator Components
            PRS[PydanticParser]
            PTMP[PromptTemplate]
            CTOOL[CallableTool]
        end
    end

    subgraph Response Models
        CRESP[ChatResponse]
        CORESP[CompletionResponse]
        MSG[Message]
    end

    %% Initialization Flow
    UC -->|1. Initialize| COMP
    UC -->|1. Initialize| RESP
    COMP -->|Store config| CFG
    RESP -->|Store config| CFG
    CFG -->|Lookup| MODREG
    MODREG -->|Context window, caps| MD

    %% Client lazy init
    COMP -->|On first use| LINIT
    RESP -->|On first use| LINIT
    LINIT -->|Resolve| CRED
    CRED -->|Build| FACTORY
    FACTORY -->|Create if None| CL
    FACTORY -->|Create if None| ACL
    CL -.->|Wraps| CLI
    ACL -.->|Wraps| ACLI

    %% Chat Completions Flow
    UC -->|2a. chat(messages)| COMP
    COMP -->|Build request| BCRQ
    BCRQ -->|Convert messages| BCRQ_MSG
    BCRQ -->|Build kwargs| BCRQ_OPT
    BCRQ_OPT -->|Final payload| RETRY
    RETRY -->|Delegate| CL
    CL -->|chat.completions.create| CLI
    CLI -->|HTTPS POST| EP_CHAT
    EP_CHAT -->|Route to| MDL
    MDL -->|Generate| EP_CHAT
    EP_CHAT -->|Response| CLI
    CLI -->|Return| CL
    CL -->|Raw response| PRSP_CHAT
    PRSP_CHAT -->|Extract message| MSG
    PRSP_CHAT -->|Extract tool_calls| PRSP_TOOL
    PRSP_META -->|Token counts| CRESP
    CRESP -->|Return| UC

    %% Responses Flow
    UC -->|2b. chat(messages)| RESP
    RESP -->|Build request| BCRQ
    BCRQ -->|Convert messages| BCRQ_MSG
    BCRQ_OPT -->|Final payload| RETRY
    CL -->|responses.create| CLI
    CLI -->|HTTPS POST| EP_RESP
    EP_RESP -->|Response| CLI
    CL -->|Raw response| PRSP_RESP
    PRSP_RESP -->|Parse output items| MSG
    CRESP -->|Return| UC

    %% Streaming Flow
    UC -->|2c. chat(messages, stream=True)| COMP
    COMP -->|stream=True| RETRY
    RETRY -->|Delegate| CLI
    CLI -->|Streaming POST| EP_CHAT
    EP_CHAT -.->|Chunk 1| CLI
    CLI -.->|Chunk 1| STRM_CHAT
    STRM_CHAT -.->|Accumulate tools| ACC
    ACC -.->|Yield| CRESP
    CRESP -.->|Yield| UC
    EP_CHAT -.->|Chunk N| CLI

    %% Tool Calling Flow
    UC -->|2d. generate_tool_calls(messages, tools)| COMP
    COMP -->|Prepare| PREP
    PREP -->|Convert to schema| BCRQ_TOOL
    PREP -->|Resolve choice| CHOICE
    COMP -->|Call chat| RETRY
    RETRY -->|API call| CLI
    CLI -->|Response with tool_calls| PRSP_CHAT
    PRSP_CHAT -->|Parse| PRSP_TOOL
    PRSP_TOOL -->|Return to| VAL
    VAL -->|Validated| CRESP
    CRESP -->|Return| UC

    %% Structured Output Flow
    UC -->|2e. parse(output_cls, prompt)| SO_PARSE
    SO_PARSE -->|Check| SO_CHECK
    SO_CHECK -->|Supported| SO_SCHEMA
    SO_SCHEMA -->|Add response_format| BCRQ_FMT
    BCRQ_FMT -->|Chat call| COMP
    SO_CHECK -->|Not supported| PREP
    PREP -->|Function-calling fallback| COMP

    %% Completion Flow (via adapter)
    UC -->|2f. complete(prompt)| COMP
    COMP -->|ChatToCompletion| BCRQ_MSG
    BCRQ_MSG -->|Wrap to Message| COMP
    COMP -->|chat()| CL
    CL -->|ChatResponse| COMP
    COMP -->|Extract text| CORESP
    CORESP -->|Return| UC

    %% TextCompletionLLM Integration
    UC -->|3a. TextCompletionLLM| TCL
    TCL -->|Use| PTMP
    TCL -->|Use| PRS
    UC -->|Call| TCL
    TCL -->|Format prompt| PTMP
    PTMP -->|Messages| COMP
    COMP -->|chat| CRESP
    CRESP -->|message.content| PRS
    PRS -->|Parse JSON| PM
    PM -->|Return| UC

    %% ToolOrchestratingLLM Integration
    UC -->|3b. ToolOrchestratingLLM| TOL
    TOL -->|Create tool from| PM
    PM -->|Schema| CTOOL
    TOL -->|Use| PTMP
    UC -->|Call| TOL
    TOL -->|generate_tool_calls| COMP
    COMP -->|tool_calls| TOL
    TOL -->|Execute tool| CTOOL
    CTOOL -->|Create| PM
    PM -->|Return| UC

    %% Retry Flow
    RETRY -->|On failure| RETRYABLE
    RETRYABLE -->|Retryable?| RETRY
    RETRYABLE -->|Not retryable| UC

    %% Styling
    classDef userClass fill:#e1f5ff,stroke:#01579b
    classDef openaiClass fill:#e0f2f1,stroke:#004d40
    classDef configClass fill:#fff9c4,stroke:#f57f17
    classDef clientClass fill:#f3e5f5,stroke:#4a148c
    classDef parserClass fill:#e8f5e9,stroke:#1b5e20
    classDef serverClass fill:#efebe9,stroke:#3e2723
    classDef orchestratorClass fill:#fce4ec,stroke:#880e4f
    classDef responseClass fill:#fff3e0,stroke:#e65100
    classDef retryClass fill:#ffecb3,stroke:#ff6f00

    class UC,PM userClass
    class COMP,RESP openaiClass
    class CFG,MD,MODREG configClass
    class CL,ACL,LINIT,CRED,FACTORY,CLI,ACLI,CHAT_OP,COMP_OP,RESP_OP clientClass
    class BCRQ,BCRQ_MSG,BCRQ_OPT,BCRQ_TOOL,BCRQ_FMT configClass
    class PRSP_CHAT,PRSP_RESP,PRSP_TOOL,PRSP_META,PRSP_LOG parserClass
    class SRV,EP_CHAT,EP_COMP,EP_RESP,MDL serverClass
    class TCL,TOL,PRS,PTMP,CTOOL orchestratorClass
    class CRESP,CORESP,MSG responseClass
    class RETRY,RETRYABLE retryClass
```

## Component Interaction Patterns

### 1. Initialization Pattern

```
User Code
  +-- Completions.__init__ / Responses.__init__
      +-- Store: model, temperature, max_tokens, strict, ...
      +-- Model Registry lookup: context_window, capabilities
      +-- Create Metadata: is_chat_model, is_function_calling_model
      +-- Validators: _validate_model (O1 temp forcing), _resolve_credentials
      +-- Set _client=None, _async_client=None (lazy init)
```

### 2. Client Lazy Initialization Pattern

```
User -> Completions.chat
  +-- Access self.client property
      +-- Check if self._client is None
          +-- If None: _get_credential_kwargs()
          |   +-- Resolve api_key from env or parameter
          |   +-- Build kwargs: api_key, base_url, timeout, max_retries=0, headers
          +-- _build_sync_client(**kwargs)
          |   +-- Return SyncOpenAI(**kwargs)
          +-- Store in self._client
      +-- Return self._client
```

### 3. Chat Request Pattern (Completions)

```
User -> Completions.chat(messages, **kwargs)
  +-- _get_model_kwargs(**kwargs)
  |   +-- Merge: model defaults + additional_kwargs + per-call overrides
  |   +-- Handle O1: rename max_tokens -> max_completion_tokens
  |   +-- Add: modalities, audio config if set
  +-- _chat(messages, **all_kwargs)  [decorated with @retry]
  |   +-- to_openai_message_dicts(messages)
  |   |   +-- For each Message: convert role, content, tool_calls
  |   +-- self.client.chat.completions.create(**payload)
  |   +-- ChatMessageParser(response.choices[0].message).build()
  |   +-- _get_response_token_counts(response)
  |   +-- Create ChatResponse
  +-- Return ChatResponse
```

### 4. Chat Request Pattern (Responses)

```
User -> Responses.chat(messages, **kwargs)
  +-- _get_model_kwargs(**kwargs)
  |   +-- Build: model, input (messages), instructions
  |   +-- Handle reasoning_options (exclude temp/top_p when set)
  |   +-- Merge built_in_tools with per-call tools
  +-- _chat(messages, **all_kwargs)  [decorated with @retry]
  |   +-- to_openai_message_dicts(messages, api="responses")
  |   +-- self.client.responses.create(**payload)
  |   +-- ResponsesOutputParser(response.output).build()
  |   +-- Create ChatResponse
  +-- Return ChatResponse
```

### 5. Streaming Pattern

```
User -> Completions.chat(messages, stream=True)
  +-- _stream_chat(messages, **kwargs)  [decorated with @retry(stream=True)]
      +-- Build request with stream=True, stream_options
      +-- self.client.chat.completions.create(**payload)
      +-- ToolCallAccumulator() initialized
      +-- For each chunk:
          +-- Extract delta.content
          +-- accumulator.add_chunk(chunk) if tool_calls
          +-- Create ChatResponse with delta
          +-- Yield ChatResponse
```

### 6. Tool Calling Pattern

```
User -> Completions.generate_tool_calls(messages, tools, **kwargs)
  +-- _prepare_chat_with_tools(tools, message, **kwargs)
  |   +-- For each tool in tools:
  |       +-- to_openai_tool(tool.metadata)
  |       +-- Build: {"type": "function", "function": {name, params, strict}}
  |   +-- resolve_tool_choice(tool_choice, tool_required)
  |   +-- Build messages list from message + chat_history
  +-- Call self.chat(messages, tools=..., tool_choice=...)
  +-- _validate_chat_with_tools_response(response, tools)
  +-- Return ChatResponse
```

### 7. Structured Output Pattern

```
User -> Completions.parse(output_cls=Person, prompt="...")
  +-- _should_use_structure_outputs()
  |   +-- Check: structured_output_mode == DEFAULT AND model in JSON_SCHEMA_MODELS
  +-- If native JSON-schema:
  |   +-- _prepare_schema(kwargs, schema)
  |   +-- Call chat(messages, response_format={type: json_schema, ...})
  |   +-- Parse response content as JSON -> validate with Pydantic
  +-- If function-calling fallback:
  |   +-- _ensure_tool_choice(kwargs) -> tool_choice="required"
  |   +-- Call via base class (LLM.parse) -> function calling
  +-- Return Model instance
```

### 8. Retry Pattern

```
@retry(is_retryable, logger) wraps _chat, _stream_chat, _achat, etc.
  +-- Try API call
  +-- On exception:
      +-- is_retryable(exc)?
      |   +-- APIConnectionError -> True (retry)
      |   +-- APITimeoutError -> True (retry)
      |   +-- Status 429/500/502/503/504/408 -> True (retry)
      |   +-- Status 401/400/403/404 -> False (raise immediately)
      +-- If retryable and attempts < max_retries:
      |   +-- Exponential backoff with jitter
      |   +-- Retry
      +-- Else: raise
```

## Component State Management

### Completions/Responses Instance State

```
Initialization:
  - model: str (immutable after init)
  - temperature: float (immutable, forced to 1.0 for O1)
  - max_tokens: int | None (immutable after init)
  - api_key: str (resolved during init)
  - api_base: str | None (resolved during init)
  - timeout: float (immutable after init)
  - _client: SyncOpenAI | None (mutable, lazy-initialized)
  - _async_client: AsyncOpenAI | None (mutable, lazy-initialized)
  - _async_client_loop: AbstractEventLoop | None (mutable, tracks event loop)

Runtime:
  - _client: None -> SyncOpenAI instance (on first sync use)
  - _async_client: None -> AsyncOpenAI instance (on first async use)
  - _async_client_loop: tracks current event loop for async client
```

## Error Boundaries

### 1. Configuration Errors (Initialization)

```
Validators:
  +-- _validate_model: O1 model checks, Responses-only model rejection
  +-- _resolve_credentials: API key presence
  +-- Pydantic field validation: temperature range, timeout >= 0
```

### 2. Retry Classification (During Call)

```
is_retryable(exception):
  +-- Retryable: APIConnectionError, 429, 500, 502, 503, 504, 408
  +-- Non-retryable: 401, 400, 403, 404 -> raise immediately
```

### 3. Response Parsing Errors

```
ChatMessageParser / ResponsesOutputParser:
  +-- Missing fields -> KeyError
  +-- Invalid tool_calls JSON -> logger.warning, empty dict fallback
  +-- Unexpected response format -> ValueError
```

## Component Dependencies

### Completions/Responses Depend On

- `openai` package (SyncOpenAI, AsyncOpenAI, SDK types)
- `tiktoken` (tokenizer for token counting)
- `httpx` (HTTP client)
- `serapeum.core` (BaseLLM, FunctionCallingLLM, Message, ChatResponse, etc.)
- `pydantic` (configuration and structured outputs)

### Completions/Responses Are Used By

- `TextCompletionLLM` (as the LLM engine)
- `ToolOrchestratingLLM` (as the function-calling LLM)
- `AzureOpenAI` (as the base for Azure-specific subclasses)
- Direct user code (standalone usage)
