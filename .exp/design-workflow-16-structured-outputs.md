# High-Level Design: Structured Outputs Workflow (#16)

## Overview

The Structured Outputs workflow in the xAI Python SDK allows developers to generate model responses that adhere to a predefined JSON structure, defined using Pydantic `BaseModel` subclasses. By providing a JSON schema derived from the Pydantic model to the chat API via the `response_format` parameter (set to `FORMAT_TYPE_JSON_SCHEMA`), the model is constrained to output valid JSON matching the schema. The SDK then parses this JSON into a type-safe Pydantic instance, enabling reliable extraction of structured data from prompts, including multimodal inputs like images.

This feature is integrated into the chat API, supporting both synchronous (`sync/chat.py`) and asynchronous (`aio/chat.py`) modes. It is demonstrated in examples such as `examples/sync/structured_outputs.py`, where it extracts receipt details (date, items, total) from an image using vision models like `grok-2-vision`.

Key aspects:
- **Schema Enforcement**: Model generates compliant JSON; non-compliance may result in invalid output requiring manual handling.
- **Use Cases**: Data extraction (e.g., forms, receipts), agentic workflows needing typed outputs, API integrations.
- **Limitations**: Relies on model adherence to schema; streaming requires manual content accumulation before parsing; validation is client-side via Pydantic.

## Components

- **Pydantic Models**: User-defined `BaseModel` subclasses (e.g., `class Receipt(BaseModel): date: datetime; items: list[Item]; ...`). Provides schema via `model_json_schema()` for API constraint and `model_validate_json()` for parsing.
- **Chat Creation (`BaseClient.create`)**: Accepts `response_format: Union[ResponseFormat, type[BaseModel]]`. If `BaseModel`, converts to `chat_pb2.ResponseFormat(format_type=FORMAT_TYPE_JSON_SCHEMA, schema=json.dumps(schema))`.
- **Parse Method (`Chat.parse` in sync/aio)**: Convenience wrapper—sets schema on `proto.response_format`, invokes `sample()` (gRPC `GetCompletion`), creates `Response`, parses content, returns `(Response, T)`.
- **Response Class**: Wraps `chat_pb2.GetChatCompletionResponse`; exposes `content` (JSON string), `usage` (tokens), `tool_calls`, etc. Buffers chunks for streaming if used.
- **Proto Definitions (`proto/v*/chat_pb2.py`)**: `GetCompletionsRequest` includes `response_format`; supports versioning (v5/v6).
- **Utility Layers**: Telemetry (OpenTelemetry spans with GenAI attrs like `gen_ai.request.model`, prompts/responses if enabled); interceptors (auth, timeouts); poll_timer for deferred (not directly used here).
- **Types (`types/chat.py`)**: `ResponseFormat` excludes "json_schema" to encourage `parse`; other types like `Content` for messages (text, image, file).
- **Examples & Tests**: `examples/*/structured_outputs.py` shows parse vs. manual; tests in `tests/*/chat_test.py` likely cover schema setting/parsing.

## Sequence Diagram: Core Flow (Parse Method)

```mermaid
sequenceDiagram
    participant U as User/App
    participant C as xAI Client
    participant Ch as Chat Instance
    participant Grpc as gRPC Layer
    participant Api as xAI API
    participant Py as Pydantic Validator

    U->>+C: client = Client()
    C->>+Grpc: Initialize channel, stubs, interceptors
    U->>+Ch: chat = client.chat.create(model, messages=[system(...)])
    Ch->>Ch: Set up GetCompletionsRequest proto
    U->>+Ch: chat.append(user(prompt, image(url)))
    Ch->>Ch: Append Message to proto.messages
    par Parse Flow
        U->>+Ch: response, parsed = chat.parse(MyModel)
        Ch->>Ch: Set response_format: FORMAT_TYPE_JSON_SCHEMA, schema=model_json_schema()
        Note right of Ch: Create OTEL span: "chat.parse {model}"
        Ch->>+Grpc: _stub.GetCompletion(request)
        Grpc->>+Api: HTTPS/gRPC: Send request w/ schema constraint
        Api->>+Api: Model inference constrained to schema
        Api->>-Grpc: Response proto w/ JSON content in outputs[].message.content
        Grpc->>-Ch: proto response
        Ch->>Ch: Create Response(proto, index)
        Ch->>+Py: MyModel.model_validate_json(response.content)
        Py->>-Ch: Validated instance or ValidationError
        Ch->>-U: (response, parsed_model)
    and Manual Flow (alternative)
        U->>+Ch: chat.create(..., response_format=MyModel)
        Ch->>Ch: Set schema in create
        U->>+Ch: response = chat.sample()
        Ch->>+Grpc: GetCompletion
        Note right of Ch: Similar gRPC flow
        Grpc->>-Ch: response
        Ch->>-U: response
        U->>+Py: MyModel.model_validate_json(response.content)
        Py->>-U: parsed_model
    end
    Note over C,Api: Auth via Bearer token in metadata<br/>Retries (5x exp backoff on UNAVAILABLE)<br/>Timeouts (default 15min)
```

## Additional Diagrams

### gRPC Proto Interaction

```mermaid
sequenceDiagram
    participant SDK as SDK (chat.py)
    participant Proto as Proto Msg
    participant Stub as ChatStub
    participant Channel as gRPC Channel

    SDK->>Proto: Build GetCompletionsRequest (model, messages, response_format={json_schema, schema})
    Proto->>Stub: Pass request
    Stub->>Channel: Invoke GetCompletion unary RPC
    Channel->>Channel: Apply interceptors (Auth, Timeout), serialize proto
    Note over Channel: Send over TLS to api.x.ai:443
    Channel->>Stub: Deserialize response proto
    Stub->>SDK: Return GetChatCompletionResponse
    SDK->>Proto: Extract outputs[0].message.content (JSON str)
```

## Other High-Level Design Aspects

- **Integration with Other Features**:
  - **Multimodal**: Supports image/file contents in messages for vision-based extraction (e.g., `image(url, detail="high")` adds ~tokens based on resolution).
  - **Tools/Functions**: Schema applies to assistant message; can combine with `tools` for hybrid workflows.
  - **Streaming**: `chat.stream()` yields chunks; accumulate `chunk.content` then parse (not in `parse`, which is non-streaming).
  - **Deferred/Stored**: Can use `defer()` for long tasks; `store_messages=True` for persistence, but schema unchanged.

- **Error & Resilience**:
  - **Validation**: Pydantic raises `ValidationError` on mismatch; handle via try/except or `model_validate_json` strict mode.
  - **API Errors**: gRPC statuses (e.g., INVALID_ARGUMENT if schema invalid, RESOURCE_EXHAUSTED for quotas).
  - **Fallbacks**: If schema fails, fallback to "json_object" or "text"; monitor via telemetry.

- **Performance Considerations**:
  - Schema size adds to prompt tokens; complex nested models increase latency/tokens.
  - Client-side parsing overhead minimal; model enforcement reduces post-processing.

- **Versioning & Compatibility**:
  - Proto v5/v6 support; newer versions may enhance schema features.
  - Pydantic v2+ assumed (via deps); schema evolution via model updates.

- **Observability**:
  - OTEL spans capture request attrs (model, temp, schema type), response (tokens, content if not disabled via env).
  - Custom attrs for structured outputs traceable via `gen_ai.output.type: "json_schema"`.

This design leverages gRPC efficiency, Pydantic ergonomics, and model capabilities for robust structured generation in Python apps.