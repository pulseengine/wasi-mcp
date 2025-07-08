# WASI Model Context Protocol (MCP)

A proposed [WebAssembly System Interface](https://github.com/WebAssembly/WASI) API for the Model Context Protocol, targeting WASI Preview3 with full async support.

### Current Phase

Phase 1 - Feature Proposal

### Champions

- [Your Name/Organization]

### Portability Criteria

TODO before entering Phase 2.

**Preview3 Async Requirements:**
- Components MUST support `future<T>` and `stream<T>` types
- All blocking operations MUST be async with proper cancellation
- Implementations MUST integrate with `wasi:io/poll` for event coordination
- Resource lifecycle MUST follow WASI resource management patterns

## Table of Contents [if the explainer is longer than one printed page]

- [Introduction](#introduction)
- [Goals [or Motivating Use Cases, or Scenarios]](#goals-or-motivating-use-cases-or-scenarios)
- [Non-goals](#non-goals)
- [API walk-through](#api-walk-through)
  - [Use case 1](#use-case-1)
  - [Use case 2](#use-case-2)
- [Detailed design discussion](#detailed-design-discussion)
  - [[Tricky design choice 1]](#tricky-design-choice-1)
  - [[Tricky design choice 2]](#tricky-design-choice-2)
- [Considered alternatives](#considered-alternatives)
  - [[Alternative 1]](#alternative-1)
  - [[Alternative 2]](#alternative-2)
- [Stakeholder Interest & Feedback](#stakeholder-interest--feedback)
- [References & acknowledgements](#references--acknowledgements)

### Introduction

This proposal introduces a WebAssembly System Interface (WASI) API for the Model Context Protocol (MCP), enabling WebAssembly components to act as MCP servers and clients with full async support. The MCP is an open standard that standardizes how applications provide context to Large Language Models (LLMs), facilitating secure and standardized connections between AI systems and data sources.

**WASI Preview3 Integration:** This proposal targets WASI Preview3, leveraging native async functions with `future<T>` and `stream<T>` types for optimal performance and composability. The design follows established WASI patterns from late-stage proposals (HTTP, Filesystem, Sockets) with proper resource management and error handling.

By integrating MCP with WASI Preview3, this proposal enables WebAssembly components to serve as portable, secure, and interoperable context providers for AI applications. This allows AI tools to access structured data, execute functions, and retrieve resources through WebAssembly components that can run across different platforms and environments while maintaining strong security boundaries and async performance characteristics.

### Goals [or Motivating Use Cases, or Scenarios]

- **Async-First MCP Servers**: Enable WebAssembly components to act as MCP servers with native async support for optimal performance
- **Secure Context Provision**: Leverage WebAssembly's security model and WASI resource management for safe context access
- **Streaming Operations**: Support large resource access and long-running tool executions through Preview3 streaming
- **Interoperable AI Integration**: Allow AI tools to interact with WebAssembly-based data sources through standardized async MCP interfaces
- **Component Composition**: Enable modular MCP architectures where different components handle different aspects (resources, tools, prompts)
- **Cross-Platform Compatibility**: Ensure MCP servers built with this API run consistently across different operating systems and architectures
- **Future-Proof Design**: Align with WASI Preview3 direction and async evolution for long-term viability

### Non-goals

- **Full MCP Implementation**: This proposal focuses on the WASI interface layer, not a complete MCP implementation
- **Transport Layer Details**: Transport mechanisms (HTTP, stdio) are handled by the host environment, not the WASI interface
- **AI Model Integration**: Direct integration with specific AI models or providers is outside the scope
- **Legacy Protocol Support**: Only targeting MCP specification (no backwards compatibility with proprietary protocols)
- **Synchronous APIs**: No support for blocking operations; all operations are async-first following Preview3 patterns

### API walk-through

The full API documentation can be found in the [WIT files](./wit/).

**Interface Structure:**
- `wasi:mcp/types@0.1.0` - Core types, resources, and error handling
- `wasi:mcp/server@0.1.0` - Server implementation with async operations
- `wasi:mcp/client@0.1.0` - Client operations for consuming MCP services
- `wasi:mcp/streaming@0.1.0` - Streaming operations for large data and long-running tools

This API enables WebAssembly components to implement async MCP servers with:

1. **Async Server Management**: Initialize and manage MCP server instances with `future<T>` operations
2. **Streaming Resource Handling**: Register and serve resources with streaming support for large data
3. **Long-Running Tool Execution**: Execute tools asynchronously with progress monitoring and cancellation
4. **Real-Time Prompt Rendering**: Provide reusable prompt templates with async rendering
5. **Protocol Message Processing**: Handle MCP protocol messages with native async support

#### Async Database Context Provider

A WebAssembly component that provides async database query capabilities:

```rust
// WebAssembly component implementing async MCP server
use wasi::{mcp::types::*, mcp::server::*};

#[export]
async fn init_mcp_server() -> Result<McpServer, Error> {
    let config = ServerConfig {
        name: "database-context".to_string(),
        version: "1.0.0".to_string(),
        description: Some("Async database context provider".to_string()),
        protocol_version: "2024-11-05".to_string(),
        capabilities: ServerCapabilities {
            resources: Some(ResourceCapabilities { subscribe: true, list_changed: true }),
            tools: Some(ToolCapabilities { list_changed: true }),
            ..Default::default()
        },
        ..Default::default()
    };
    
    let server = create_server(config).await?;
    
    // Register database resource with async operation
    let resource_config = McpResourceConfig {
        uri: "db://tables".to_string(),
        name: "Database Tables".to_string(),
        description: "Available database tables and schemas".to_string(),
        mime_type: "application/json".to_string(),
        metadata: None,
    };
    
    register_resource(&server, resource_config).await?;
    
    // Register async query tool
    let tool_config = McpToolConfig {
        name: "execute-query".to_string(),
        description: "Execute SQL queries on the database".to_string(),
        input_schema: query_schema(),
        output_schema: Some(result_schema()),
        metadata: None,
    };
    
    register_tool(&server, tool_config).await?;
    
    Ok(server)
}

#[export]
async fn handle_tool_execution(tool_id: &str, args: &[u8]) -> Result<Vec<u8>, Error> {
    match tool_id {
        "execute-query" => {
            let request: QueryRequest = serde_json::from_slice(args)?;
            
            // Async database query with streaming results
            let result = execute_database_query(&request.query).await?;
            
            Ok(serde_json::to_vec(&result)?)
        }
        _ => Err(Error::new(ErrorCode::ToolNotFound, "Unknown tool"))
    }
}
```

#### Streaming File System Provider

A component that provides streaming file system access:

```rust
use wasi::{mcp::streaming::*, mcp::server::*};

#[export]
async fn create_streaming_resource(resource_id: &str) -> Result<StreamingResource, Error> {
    let resource = get_resource(&server, resource_id).await?;
    let streaming_resource = StreamingResource::create(resource).await?;
    
    Ok(streaming_resource)
}

#[export]
async fn handle_large_file_read(resource_id: &str) -> Result<ChunkedReader, Error> {
    let streaming_resource = create_streaming_resource(resource_id).await?;
    let reader = ChunkedReader::create(streaming_resource, 64 * 1024).await?; // 64KB chunks
    
    Ok(reader)
}

#[export]
async fn stream_file_content(reader: ChunkedReader) -> Result<Vec<u8>, Error> {
    let mut content = Vec::new();
    
    while !reader.is_end_of_resource() {
        if let Some(chunk) = reader.read_chunk().await? {
            content.extend_from_slice(&chunk);
        }
    }
    
    Ok(content)
}
```

### Detailed design discussion

The API design follows both the MCP specification and WASI Preview3 conventions, adapting MCP for the WebAssembly component model with full async support. Key design choices are detailed in the [WIT files](./wit/).

#### Preview3 Async Architecture

The API is built around WASI Preview3 async primitives:

- **Future-based Operations**: All potentially blocking operations return `future<T>` for async execution
- **Streaming Support**: Large data operations use `stream<T>` for efficient memory usage
- **Polling Integration**: Resources provide `pollable` handles for event-driven programming
- **Resource Management**: Proper WASI resource lifecycle with borrowing and ownership

#### Multi-Interface Design

Following WASI conventions, the API is split into focused interfaces:

- **`wasi:mcp/types@0.1.0`**: Core types, resources, and error handling
- **`wasi:mcp/server@0.1.0`**: Server implementation with async operations
- **`wasi:mcp/client@0.1.0`**: Client operations for consuming MCP services  
- **`wasi:mcp/streaming@0.1.0`**: Streaming operations for large data and long-running tools

This allows for modular MCP implementations where different components can handle different aspects.

#### Resource-Based Security Model

The API leverages WebAssembly's capability-based security with WASI resources:

- **Resource Types**: All MCP entities (servers, resources, tools) are proper WASI resources
- **Capability Handles**: Access requires explicit resource handles (unforgeable capabilities)
- **Sandboxed Execution**: WebAssembly components run in isolated environments
- **Explicit Permissions**: All operations require explicit resource access

#### Async Message Handling

The `handle-message` function provides async MCP protocol processing:

```wit
handle-message: func(
    server: borrow<mcp-server>,
    message: list<u8>
) -> future<result<list<u8>, error>>;
```

This enables:
- **Non-blocking Protocol Processing**: Long-running operations don't block the event loop
- **Concurrent Request Handling**: Multiple requests can be processed simultaneously
- **Transport Abstraction**: Host environment handles transport (HTTP, stdio) while component handles protocol

#### Standard Error Handling

The API uses WASI-standard error patterns with base error resources:

```wit
resource error {
    code: func() -> error-code;
    message: func() -> string;
    to-debug-string: func() -> string;
}

enum error-code {
    invalid-request,
    method-not-found,
    invalid-params,
    internal-error,
    resource-not-found,
    tool-not-found,
    prompt-not-found,
    unauthorized,
    forbidden,
    timeout,
    rate-limited,
    unavailable,
}
```

This ensures proper error propagation and debugging capabilities following WASI conventions.

#### Streaming and Performance

The streaming interface enables efficient handling of large data:

- **Chunked Reading**: Large resources can be read in configurable chunks
- **Backpressure Handling**: Streaming operations support flow control
- **Progress Monitoring**: Long-running operations provide progress updates
- **Cancellation Support**: Operations can be cancelled when no longer needed

### Considered alternatives

#### Synchronous-First API Design

We considered maintaining synchronous APIs with optional async variants, but chose async-first because:

- **Preview3 Alignment**: Full alignment with WASI Preview3 async direction
- **Performance**: Better handling of I/O-bound operations without blocking
- **Composability**: Native async composition enables complex MCP workflows
- **Future-Proofing**: Easier to optimize async operations than retrofit sync APIs

#### Direct JSON-RPC Integration

We considered exposing the JSON-RPC 2.0 protocol directly through WASI, but decided against it because:

- **Complexity**: Components would need to implement JSON-RPC protocol details
- **Coupling**: Tight coupling between components and transport layer
- **Maintenance**: Protocol changes would require component updates
- **Async Mismatch**: JSON-RPC doesn't naturally align with async resource patterns

The current abstraction provides cleaner separation of concerns and better async integration.

#### Single Monolithic Interface

An alternative would be a single large interface containing all MCP functionality, but this was rejected because:

- **Modularity**: Less modular and harder to compose
- **Reusability**: Harder to reuse individual components
- **Testing**: More difficult to test individual aspects
- **WASI Patterns**: Doesn't follow established WASI multi-interface conventions

The current four-interface design (`types`, `server`, `client`, `streaming`) provides better modularity and follows WASI patterns.

#### Blocking Operations with Timeouts

We considered providing blocking operations with configurable timeouts, but chose the async approach because:

- **Thread Safety**: Async operations are inherently thread-safe
- **Resource Usage**: No thread-per-operation overhead
- **Cancellation**: Native cancellation support through futures
- **Integration**: Better integration with async host environments

### Stakeholder Interest & Feedback

TODO before entering Phase 3.

**Potential Implementers:**
- AI development platforms using WebAssembly
- MCP server library authors  
- WebAssembly runtime providers (Wasmtime, WAMR, etc.)
- AI application developers building on MCP
- Cloud providers offering AI services

**Areas of Interest:**
- **Anthropic (MCP creators)** - feedback on protocol alignment and async patterns
- **WebAssembly runtime providers** - Preview3 async implementation feasibility
- **AI tool developers** - API usability and performance characteristics
- **WASI Subgroup** - alignment with Preview3 direction and conventions
- **Component Model WG** - resource management and async patterns

**Preview3 Specific Feedback Needed:**
- Async operation patterns and future/stream type usage
- Resource lifecycle management with async operations
- Integration with existing WASI interfaces (io, clocks, etc.)
- Performance implications of async MCP operations

### References & acknowledgements

Many thanks for valuable feedback and advice from:

- The Model Context Protocol team at Anthropic
- The WebAssembly Component Model working group
- The WASI Subgroup for guidance on proposal structure and Preview3 patterns
- The broader WebAssembly community for security model insights
- Late-stage WASI proposal authors for async patterns and conventions

**Key References:**
- [Model Context Protocol Specification](https://modelcontextprotocol.io/introduction)
- [WebAssembly Component Model](https://github.com/WebAssembly/component-model)
- [WASI Preview 3 Async Specification](https://github.com/WebAssembly/WASI/blob/main/Contributing.md)
- [WIT Language Specification](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md)
- [WASI HTTP Proposal](https://github.com/WebAssembly/wasi-http) - async patterns reference
- [WASI Filesystem Proposal](https://github.com/WebAssembly/wasi-filesystem) - resource management reference
- [WASI Sockets Proposal](https://github.com/WebAssembly/wasi-sockets) - streaming patterns reference
