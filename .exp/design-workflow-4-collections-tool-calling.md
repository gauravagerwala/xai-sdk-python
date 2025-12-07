# High-Level Design: Collections Tool Calling Workflow

## Overview

The Collections Tool Calling workflow enables retrieval-augmented generation (RAG) by integrating semantic search capabilities over user-created vector collections into agentic chat conversations. This allows the model to dynamically retrieve and cite relevant information from uploaded documents to inform its responses.

From the project workflows, it uses collections as tools in agentic chat for semantic search over uploaded documents. Input includes document URLs or content for upload, collection creation parameters, and chat prompts. Output includes tool calls, search results, citations, and reasoned model responses.

The workflow demonstrates server-side tool execution, where the xAI API handles the search internally upon tool invocation by the model, simplifying client-side implementation while leveraging the collections infrastructure for private data retrieval.

Key features:
- Document upload and indexing into collections with embedding.
- Server-side `collections_search` tool configurable with collection IDs, limits, instructions, and retrieval modes (hybrid, semantic, keyword).
- Streaming chat responses with visibility into tool calls, reasoning, citations, and usage.
- Supports models like `grok-4-fast` capable of tool use.

## Components

- **Collections Module** (`src/xai_sdk/collections.py`): Provides APIs for creating collections, uploading and indexing documents. Relies on gRPC stubs from `proto/v*/collections_pb2_grpc.py` and `documents_pb2.py` for embedding and retrieval.
- **Tools Module** (`src/xai_sdk/tools.py`): Defines `collections_search` function to construct `chat_pb2.Tool` with `CollectionsSearch` proto message, specifying searchable collections and search parameters.
- **Chat Module** (`src/xai_sdk/chat.py`): Manages multi-turn conversations with tool support, including streaming, tool call handling, and response parsing (citations, server-side tool usage). Integrates with `types/chat.py` for Pydantic models.
- **Proto Layer**: `chat_pb2.py` defines `Tool` and `CollectionsSearch` for tool configuration; `documents_pb2.py` for retrieval modes like `HybridRetrieval`.
- **Client Layer**: `BaseClient` handles gRPC channels, auth, retries; service clients abstract proto interactions.
- **Examples**: `examples/sync/collections_tool.py` (and aio equivalent) showcases end-to-end: collection creation, parallel document upload from URLs, tool-enabled chat query, streaming output with citations.
- **Server Infrastructure** (opaque to SDK): xAI API services for chat (processes tool calls), collections/documents (vector search, indexing).

## Sequence Diagram: Collection Setup and Document Ingestion

```mermaid
sequenceDiagram
    participant User as User/Application
    participant SDK as xAI SDK Client
    participant API as xAI API (Collections Service)
    participant DB as Vector DB/Internal Storage

    User->>SDK: client.collections.create(name="tesla-filings", ...)
    SDK->>API: gRPC Collections.CreateCollection
    API->>DB: Create vector collection
    DB-->>API: collection_id
    API-->>SDK: CreateCollectionResponse
    SDK-->>User: collection_id

    Note over User,DB: Repeat for document uploads (parallel possible via ThreadPoolExecutor):
    User->>SDK: fetch content (e.g., PDF via requests.get)
    SDK->>API: Collections.UploadDocument(collection_id, name, data=bytes, wait_for_indexing=True)
    API->>DB: Process: chunk, embed (using specified model), index into vectors
    DB-->>API: Indexing status/metadata
    API-->>SDK: UploadDocumentResponse (doc_id, status)
```

## Sequence Diagram: Agentic Chat with Tool Calling

```mermaid
sequenceDiagram
    participant User as User/Application
    participant SDK as xAI SDK Chat Client
    participant API as xAI Chat Service
    participant Model as Grok Model
    participant Coll as Collections Service/DB

    User->>SDK: chat = client.chat.create(model="grok-4-fast", tools=[collections_search(collection_ids=[id])])
    SDK->>API: gRPC Chat.GetCompletionsRequest(messages=[user("Query requiring search...")], tools=[Tool(collections_search=CollectionsSearch(collection_ids=[id], limit=10, ...))], stream=True)
    API->>Model: Provide context (messages, tool schemas inferred from protos), generate tokens
    Model->>API: Detects need for search, outputs ToolCall {name="collections_search", arguments={query: extracted from context, ...}}
    API->>Coll: Server-side execution: Search collections (ids from tool config) with query, apply instructions/mode/limit
    Coll-->>API: Retrieval: top matches (chunks/text, scores, metadata, citation info)
    API->>Model: Inject ToolOutput message (formatted results) into context
    Model->>API: Reason over results, generate final answer incorporating data with citations
    API->>SDK: Stream chunks: tool_calls, reasoning_tokens (thinking indicator), content deltas, final response with citations/server_side_tool_usage
    SDK-->>User: Parse stream: print tool invocations, progress, response text, citations, usage stats
```

## Additional Design Aspects

- **Tool Configuration**: `collections_search` allows up to 10 collection_ids; optional custom `instructions` for ranking/relevance; retrieval modes via strings or protos (default hybrid).
- **Server-Side Benefits**: Automatic handling of tool calls (no client loop needed), parallel execution possible, secure (no client access to execution logic), integrated citations.
- **Integration with Chat Features**: Combines with streaming, stored chats, structured outputs, telemetry. Tool calls visible in responses for logging/monitoring.
- **Data Flow**: Documents uploaded as bytes/file; auto-chunked/embedded; search returns context windows suitable for model input.
- **Edge Cases**: Empty collections return no results; rate limits/quotas enforced server-side; errors propagated via gRPC status (e.g., INVALID_ARGUMENT for bad config).
- **Async Support**: Full aio equivalents in `aio/` submodule for non-blocking ops.
- **Observability**: OpenTelemetry spans for chat invocations, tool executions (with query/results attrs if enabled).
- **Dependencies**: Leverages `requests` for URL fetches in examples; protobuf/grpcio for comms; Pydantic for response validation.

This workflow exemplifies modular SDK design: separate services for data management (collections) and inference (chat), bridged via server-side tools for powerful agentic capabilities.