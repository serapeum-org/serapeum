# Data Transformations and Validation

This diagram shows how data flows and transforms through the Azure OpenAI provider,
highlighting Azure-specific validation and credential resolution.

```mermaid
flowchart TD
    Start([User Code]) --> Init{Initialize Azure Completions}

    Init --> AzureVal[_validate_azure_env - mode before]

    subgraph Azure Validation Pipeline
        AzureVal --> ResolveEngine[Resolve engine from aliases]
        ResolveEngine --> CheckAliases{engine set?}
        CheckAliases -->|No| TryAliases[Try: deployment_name, deployment_id, deployment]
        CheckAliases -->|Yes| ValidateVersion
        TryAliases --> ValidateVersion[Validate api_version is set]
        ValidateVersion --> ResolveEndpoint[Resolve azure_endpoint from env]
        ResolveEndpoint --> CheckBase{api_base is default OpenAI?}
        CheckBase -->|Yes, no azure_endpoint| RaiseError([ValueError])
        CheckBase -->|OK| InheritedValidation
    end

    InheritedValidation --> OpenAIVal[_resolve_credentials - inherited]
    OpenAIVal --> ResetBase[_reset_api_base_for_azure]
    ResetBase --> ClearBase[Clear api_base if default or azure_endpoint set]
    ClearBase --> Ready([Azure Instance Ready])

    Ready --> CallType{Call Type?}

    CallType -->|chat| ChatPath[Chat Path]
    CallType -->|complete| CompletePath[Complete Path]
    CallType -->|generate_tool_calls| ToolsPath[Tools Path]
    CallType -->|stream chat| StreamPath[Stream Path]
    CallType -->|parse| StructuredPath[Structured Output Path]

    %% Client Creation
    ChatPath --> EnsureClient[Ensure client initialized]
    EnsureClient --> CheckClient{Client exists?}
    CheckClient -->|No| ResolveCreds[_get_credential_kwargs]

    subgraph Azure Credential Resolution
        ResolveCreds --> ResolveKey[_resolve_api_key]
        ResolveKey --> CheckAD{use_azure_ad?}
        CheckAD -->|Yes, provider set| CallProvider[azure_ad_token_provider]
        CheckAD -->|Yes, auto| RefreshToken[refresh_openai_azure_ad_token]
        CheckAD -->|No| UseKey[api_key or AZURE_OPENAI_API_KEY env]

        RefreshToken --> CheckExpiry{Token valid?}
        CheckExpiry -->|Yes, > 60s left| ReuseToken[Reuse existing token]
        CheckExpiry -->|No/None| AcquireNew[DefaultAzureCredential.get_token]
        AcquireNew --> StoreToken[Store AccessToken]

        CallProvider --> BuildKwargs
        ReuseToken --> BuildKwargs
        StoreToken --> BuildKwargs
        UseKey --> BuildKwargs

        BuildKwargs[Build credential kwargs]
        BuildKwargs --> AddAzureFields[Add: azure_endpoint, azure_deployment,<br/>api_version, azure_ad_token_provider]
    end

    AddAzureFields --> CreateClient[_build_sync_client]
    CreateClient --> AzureSDK[SyncAzureOpenAI - max_retries=0]
    CheckClient -->|Yes| BuildModelKwargs

    AzureSDK --> BuildModelKwargs[_get_model_kwargs]

    %% Model kwargs with engine swap
    subgraph Model Kwargs Building
        BuildModelKwargs --> InheritedKwargs[super._get_model_kwargs]
        InheritedKwargs --> MergeDefaults[Merge: defaults + additional_kwargs + overrides]
        MergeDefaults --> SwapModel[model = self.engine]
    end

    SwapModel --> ConvertMessages[to_openai_message_dicts]
    ConvertMessages --> RetryWrap[@retry decorator]
    RetryWrap --> APICall[client.chat.completions.create]
    APICall --> AzureAPI[Azure OpenAI API]
    AzureAPI --> Deployment[Model Deployment]
    Deployment --> RawResponse[ChatCompletion response]

    RawResponse --> ParseResponse[ChatMessageParser]
    ParseResponse --> ExtractTokens[_get_response_token_counts]
    ExtractTokens --> CreateResp[Create ChatResponse]
    CreateResp --> ReturnChat([Return ChatResponse])

    %% Complete Path
    CompletePath --> Adapter[ChatToCompletion adapter]
    Adapter --> WrapPrompt[Wrap prompt as Message]
    WrapPrompt --> ChatPath

    %% Tools Path
    ToolsPath --> PrepareTools[_prepare_chat_with_tools]
    PrepareTools --> ConvertTools[to_openai_tool for each tool]
    ConvertTools --> ResolveChoice[resolve_tool_choice]
    ResolveChoice --> ChatPathTools[Delegate to chat with tools]
    ChatPathTools --> ChatPath

    %% Stream Path
    StreamPath --> StreamKwargs[_get_model_kwargs with stream=True]
    StreamKwargs --> StreamCall[client.chat.completions.create stream=True]
    StreamCall --> StreamLoop{For each chunk}
    StreamLoop --> FilterEmpty{Empty choices?}
    FilterEmpty -->|Yes - Azure filter| SkipChunk[Skip chunk]
    FilterEmpty -->|No| ParseDelta[Parse delta]
    SkipChunk --> StreamLoop
    ParseDelta --> AccumTools[ToolCallAccumulator]
    AccumTools --> YieldResp[Yield ChatResponse]
    YieldResp --> CheckDone{finish_reason?}
    CheckDone -->|None| StreamLoop
    CheckDone -->|stop| EndStream([Stream Complete])

    %% Structured Output
    StructuredPath --> CheckSupport{JSON-schema supported?}
    CheckSupport -->|Yes| NativeSchema[response_format payload]
    NativeSchema --> ChatPath
    CheckSupport -->|No| FnCallFallback[Function-calling fallback]
    FnCallFallback --> ToolsPath

    %% Error paths
    RaiseError --> ErrorEnd([Raise Exception])
    APICall -->|Exception| RetryCheck{is_retryable?}
    RetryCheck -->|Yes| RetryWrap
    RetryCheck -->|No| ErrorEnd

    %% Styling
    style Start fill:#e1f5ff
    style Ready fill:#e1f5ff
    style ReturnChat fill:#c8e6c9
    style EndStream fill:#c8e6c9
    style RaiseError fill:#ffcdd2
    style ErrorEnd fill:#ffcdd2
    style AzureSDK fill:#f3e5f5
    style AzureAPI fill:#efebe9
    style RetryWrap fill:#ffecb3
```

## Data Transformation Examples

### 1. Azure Initialization

```python notest
Input:
  Completions(
      engine="my-gpt4o-deployment",
      model="gpt-4o",
      api_key="azure-key",
      azure_endpoint="https://myres.openai.azure.com/",
      api_version="2024-02-01",
  )

Transformations:
  1. _validate_azure_env (mode="before"):
     - engine = "my-gpt4o-deployment" (resolved from aliases)
     - api_version = "2024-02-01" (validated present)
     - azure_endpoint = "https://myres.openai.azure.com/"

  2. Inherited validation:
     - model = "gpt-4o" (capability checks)
     - api_key = "azure-key"

  3. _reset_api_base_for_azure:
     - api_base = None (cleared, azure_endpoint takes over)

  4. Model registry:
     - context_window = 128000
     - is_chat_model = True
     - is_function_calling_model = True

Output:
  Azure Completions instance with engine="my-gpt4o-deployment"
```

### 2. Azure Client Creation

```python notest
Trigger: First chat/complete call

Transformations:
  1. _resolve_api_key():
     - use_azure_ad=False -> api_key = "azure-key"

  2. _get_credential_kwargs():
     {
       "api_key": "azure-key",
       "max_retries": 0,
       "timeout": httpx.Timeout(60.0),
       "default_headers": None,
       "azure_endpoint": "https://myres.openai.azure.com/",
       "azure_deployment": "my-gpt4o-deployment",
       "api_version": "2024-02-01",
       "azure_ad_token_provider": None,
       "http_client": None
     }

  3. _build_sync_client(**kwargs):
     SyncAzureOpenAI(
       api_key="azure-key",
       azure_endpoint="https://myres.openai.azure.com/",
       azure_deployment="my-gpt4o-deployment",
       api_version="2024-02-01",
       max_retries=0,
       ...
     )

Output:
  SyncAzureOpenAI client stored in _client
```

### 3. Azure Chat Request

```python notest
Input:
  messages = [Message(role=USER, chunks=[TextChunk(content="Hello")])]

Transformations:
  1. _get_model_kwargs():
     - super()._get_model_kwargs() -> {"model": "gpt-4o", "temperature": 0.7}
     - Swap: all_kwargs["model"] = "my-gpt4o-deployment"
     - Result: {"model": "my-gpt4o-deployment", "temperature": 0.7}

  2. to_openai_message_dicts():
     [{"role": "user", "content": "Hello"}]

  3. SyncAzureOpenAI.chat.completions.create(
       model="my-gpt4o-deployment",
       messages=[{"role": "user", "content": "Hello"}],
       temperature=0.7
     )
     -> Sent to: https://myres.openai.azure.com/openai/deployments/
                 my-gpt4o-deployment/chat/completions?api-version=2024-02-01

  4. ChatMessageParser -> Message -> ChatResponse

Output:
  ChatResponse with assistant message
```

### 4. Azure AD Token Refresh

```python notest
Input:
  existing_token = AccessToken(token="old-token", expires_on=time.time() + 30)
  # Expires in 30 seconds (< 60s buffer)

Transformations:
  1. refresh_openai_azure_ad_token(existing_token):
     - Check: expires_on - time.time() = 30 < 60 -> needs refresh
     - credential = DefaultAzureCredential()
     - new_token = credential.get_token(
         "https://cognitiveservices.azure.com/.default"
       )
     - Return: AccessToken(token="new-token", expires_on=time.time() + 3600)

  2. _resolve_api_key():
     - self._azure_ad_token = new_token
     - api_key = "new-token"

Output:
  Fresh bearer token used for SDK client
```

### 5. Azure Streaming with Filter Chunks

```python notest
Input:
  messages = [Message(role=USER, chunks=[TextChunk(content="Count to 3")])]
  stream = True

Transformations:
  Azure may send empty filter result chunks:

  Chunk 1: ChatCompletionChunk(choices=[])  # Azure content filter
    -> Skip (empty choices)

  Chunk 2: ChatCompletionChunk(choices=[{delta: {content: "1"}}])
    -> ChatResponse(delta="1")
    -> Yield

  Chunk 3: ChatCompletionChunk(choices=[{delta: {content: ", 2, 3"}, finish_reason: "stop"}])
    -> ChatResponse(delta=", 2, 3")
    -> Yield

Output:
  Generator yielding ChatResponse objects (filter chunks skipped)
```

## Validation Points

### 1. Azure-Specific Validation (_validate_azure_env)

- **engine**: Must be resolvable from aliases; at least one must be set
- **api_version**: Must be non-None
- **azure_endpoint**: Resolved from `AZURE_OPENAI_ENDPOINT` env if not set
- **api_base**: Must not be default OpenAI URL without azure_endpoint

### 2. Inherited Validation

- All OpenAI validation points apply (model, temperature, api_key, etc.)

### 3. Credential Validation

- **API Key mode**: api_key or `AZURE_OPENAI_API_KEY` env must be set
- **Azure AD mode**: Either token_provider callback or DefaultAzureCredential must work
- **Token expiry**: Tokens refreshed when < 60 seconds until expiration

## Error Handling

### Azure-Specific Errors

```
ValueError: Missing engine/deployment name (no alias resolved)
ValueError: Missing api_version
ValueError: Default OpenAI URL without azure_endpoint
ValueError: Azure AD ClientAuthenticationError (wrapped)
```

### Inherited Error Handling

All retry logic and exception classification from OpenAI provider applies identically.

## Data Flow Summary

```
User Input
  |
Azure Validation Pipeline (engine, api_version, endpoint)
  |
Inherited Validation (model, temperature, credentials)
  |
Azure Credential Resolution (API key or Azure AD token)
  |
Azure Client Creation (SyncAzureOpenAI / AsyncAzureOpenAI)
  |
Model Kwargs Building (model -> engine swap)
  |
Message Conversion (to_openai_message_dicts)
  |
Retry Wrapper (is_retryable classification)
  |
Azure SDK API Call (HTTPS to Azure endpoint)
  |
Raw Response (ChatCompletion, with possible filter chunks)
  |
Response Parsing (ChatMessageParser / ResponsesOutputParser)
  |
Typed Response (ChatResponse / CompletionResponse)
  |
User Output
```
