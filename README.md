# WASI Model Context Protocol (MCP)

A proposed [WebAssembly System Interface](https://github.com/WebAssembly/WASI) API for the Model Context Protocol, providing typed operations for MCP 2025-06-18 with WASI Preview3 async support.

### Current Phase

**Phase 0 - Pre-Proposal** (targeting Phase 1)

### Champions

- TBD

### Approach

**Protocol-Faithful Design**: This proposal provides typed WIT interfaces that map directly to MCP protocol operations, similar to how wasi-http maps HTTP semantics. Each MCP method has a corresponding typed function with proper async patterns.

## Table of Contents

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [API Overview](#api-overview)
- [Examples](#examples)
- [Design Discussion](#design-discussion)
- [Implementation Status](#implementation-status)
- [Stakeholder Interest](#stakeholder-interest)
- [References](#references)

## Introduction

The Model Context Protocol (MCP) is an open standard that enables AI applications to connect to external data sources, tools, and workflows. This WASI proposal enables WebAssembly components to implement MCP servers and clients with full protocol support.

**Why WASI for MCP?**
- **Portability**: Run MCP servers across any WASM runtime
- **Security**: Leverage WebAssembly's capability-based security model
- **Performance**: Native async operations with WASI Preview3
- **Composability**: Component Model enables MCP server aggregation
- **Type Safety**: Strongly-typed WIT interfaces prevent protocol errors

**Protocol Coverage**: This proposal implements the complete MCP 2025-06-18 specification including:
- ✅ Protocol initialization and version negotiation
- ✅ Resources with templates and subscriptions
- ✅ Tools with annotations
- ✅ Prompts with arguments
- ✅ Pagination with cursors
- ✅ Progress tokens for long-running operations
- ✅ Notifications for list changes and updates
- ✅ Logging protocol
- ✅ Content types (text, image, embedded resources, links)
- ✅ Streaming support for large resources

## Two Versions: Preview2 + Preview3

This proposal provides **both** Preview2 and Preview3 interfaces:

| Version | Status | Use Case | Location |
|---------|--------|----------|----------|
| **Preview3** | Aspirational | Final standardization with async | `wit/*.wit` |
| **Preview2** | Available now | Immediate prototyping (blocking ops) | `wit/preview2/*.wit` |

**Preview2** allows you to build working prototypes **TODAY** with stable toolchains:
```wit
// Preview2: Synchronous/blocking (available now)
initialize: func(params: initialize-params) -> result<initialize-result, error>;
```

**Preview3** is the target design with proper async support:
```wit
// Preview3: Async with futures (when Preview3 is ready)
initialize: func(params: initialize-params) -> future<result<initialize-result, error>>;
```

See [`wit/preview2/README.md`](wit/preview2/README.md) for details on prototyping with Preview2.

## Goals

1. **Complete MCP Protocol Support**: Implement all MCP 2025-06-18 features in WIT
2. **Protocol-Faithful Design**: Direct mapping of MCP methods to typed WIT functions
3. **WASI Preview3 Async**: Native async with `future<result<T, error>>`
4. **Follow WASI Patterns**: Learn from successful Phase 3 proposals (HTTP, Filesystem, Sockets)
5. **Easy Validation**: Straightforward to verify against MCP specification
6. **Clear Path to Phase 1**: Well-defined proposal structure and requirements

## Non-goals

- **Transport Implementation**: Transport (stdio, HTTP) is handled by hosts
- **JSON-RPC Layer**: Protocol serialization is implementation detail
- **AI Model Integration**: Direct LLM integration is out of scope
- **Backward Compatibility**: Only targeting latest MCP spec

## API Overview

**Interface Structure** (following WASI multi-interface pattern):

```
wit/
├── deps.toml                  # wasi:io, wasi:clocks dependencies
├── types.wit                  # Core MCP types (request-id, cursor, resources, tools, prompts)
├── capabilities.wit           # ServerCapabilities, ClientCapabilities
├── content.wit                # Content blocks (text, image, embedded-resource)
├── server.wit                 # Typed server operations (initialize, list-resources, call-tool, etc.)
├── client.wit                 # Typed client operations
├── notifications.wit          # Notification types (list-changed, updated, progress)
├── streaming.wit              # Streaming for large resources
├── world.wit                  # Component worlds (mcp-server, mcp-client, mcp-proxy)
└── preview2/                  # Preview2 version for immediate prototyping
    ├── README.md              # Preview2 guide
    ├── types.wit              # Simplified types (blocking)
    ├── server.wit             # Synchronous server operations
    └── world.wit              # Preview2 component worlds
```

**Key Design**: Each MCP protocol method has a typed WIT function:

```wit
interface server {
    resource mcp-server {
        // Corresponds to MCP method: initialize
        initialize: func(params: initialize-params)
            -> future<result<initialize-result, error>>;

        // Corresponds to MCP method: resources/list
        list-resources: func(params: paginated-request, meta: option<request-meta>)
            -> future<result<list-resources-result, error>>;

        // Corresponds to MCP method: tools/call
        call-tool: func(name: string, arguments: option<list<u8>>, meta: option<request-meta>)
            -> future<result<call-tool-result, error>>;

        // ... all other MCP methods
    }
}
```

## Examples

### MCP Server with Typed Operations

```rust
use wasi::mcp::{types::*, server::*, capabilities::*};

#[export]
fn create_database_server() -> Result<mcp-server, error> {
    let implementation = Implementation {
        name: "database-context".to_string(),
        version: "1.0.0".to_string(),
        website_url: None,
    };

    let capabilities = ServerCapabilities {
        resources: Some(ResourcesCapability {
            subscribe: Some(true),
            list_changed: Some(true),
        }),
        tools: Some(ToolsCapability {
            list_changed: Some(false),
        }),
        prompts: None,
        logging: None,
        sampling: None,
        elicitation: None,
        completions: None,
        experimental: None,
    };

    create_server(implementation, capabilities)
}

// Implement typed server operations
impl mcp_server {
    async fn initialize(&self, params: InitializeParams)
        -> Result<InitializeResult, Error>
    {
        // Perform version negotiation
        let protocol_version = if params.protocol_version == "2025-06-18" {
            "2025-06-18".to_string()
        } else {
            "2025-03-26".to_string() // Fallback
        };

        Ok(InitializeResult {
            protocol_version,
            capabilities: self.get_capabilities(),
            server_info: self.get_implementation(),
            instructions: Some("Database context provider for SQL queries".to_string()),
        })
    }

    async fn list_resources(&self, params: PaginatedRequest, meta: Option<RequestMeta>)
        -> Result<ListResourcesResult, Error>
    {
        let resources = vec![
            Resource {
                uri: "db://tables".to_string(),
                name: "database-tables".to_string(),
                title: Some("Database Tables".to_string()),
                description: Some("List of all database tables".to_string()),
                mime_type: Some("application/json".to_string()),
                size: None,
                annotations: None,
            },
        ];

        Ok(ListResourcesResult {
            resources,
            next_cursor: None,
        })
    }

    async fn call_tool(&self, name: String, arguments: Option<Vec<u8>>, meta: Option<RequestMeta>)
        -> Result<CallToolResult, Error>
    {
        match name.as_str() {
            "execute-query" => {
                let args: QueryArgs = serde_json::from_slice(&arguments.unwrap())?;
                let result = execute_database_query(&args.query).await?;

                Ok(CallToolResult {
                    content: vec![ContentBlock::Text(TextContent {
                        content_type: "text".to_string(),
                        text: serde_json::to_string(&result)?,
                        annotations: None,
                    })],
                    structured_content: Some(serde_json::to_vec(&result)?),
                    is_error: Some(false),
                })
            }
            _ => Err(Error::tool_not_found(format!("Unknown tool: {}", name)))
        }
    }
}
```

### MCP Client with Typed Operations

```rust
use wasi::mcp::{client::*, types::*};

#[export]
async fn query_mcp_server() -> Result<Vec<Resource>, Error> {
    let client_impl = Implementation {
        name: "ai-agent".to_string(),
        version: "1.0.0".to_string(),
        website_url: None,
    };

    let capabilities = ClientCapabilities {
        roots: None,
        sampling: None,
        elicitation: None,
        experimental: None,
    };

    let client = create_client(client_impl, capabilities)?;

    // Initialize connection
    let init_params = InitializeParams {
        protocol_version: "2025-06-18".to_string(),
        capabilities,
        client_info: client_impl,
    };

    let init_result = client.initialize(init_params).await?;
    client.send_initialized()?;

    // List available resources
    let resources_result = client.list_resources(
        PaginatedRequest { cursor: None },
        None
    ).await?;

    Ok(resources_result.resources)
}
```

### Streaming Large Resources

```rust
use wasi::mcp::streaming::*;

#[export]
async fn read_large_file(uri: String) -> Result<Vec<u8>, Error> {
    // Create streaming resource
    let streaming_resource = create_streaming_resource(uri).await?;

    // Create chunked reader (64KB chunks)
    let reader = create_chunked_reader(streaming_resource, 64 * 1024)?;

    let mut content = Vec::new();

    // Read progressively
    while !reader.is_end() {
        if let Some(chunk) = reader.read_chunk().await? {
            content.extend_from_slice(&chunk);

            // Progress tracking
            if let Some(progress) = reader.progress_percent() {
                println!("Progress: {:.1}%", progress);
            }
        }
    }

    Ok(content)
}
```

## Design Discussion

### Protocol-Faithful Approach

Following successful WASI proposals like wasi-http, we map MCP semantics directly to WIT:

**wasi-http precedent**: Maps HTTP methods, status codes, headers as explicit types
**wasi-mcp approach**: Maps MCP resources, tools, prompts, capabilities as explicit types

This provides:
- ✅ Type safety for all MCP operations
- ✅ Clear correspondence to MCP specification
- ✅ Easy validation against MCP conformance tests
- ✅ Familiar to MCP SDK developers

### Typed Methods vs Generic Message Handling

**Previous approach** (generic):
```wit
handle-message: func(message: list<u8>) -> future<result<list<u8>, error>>;
```

**Current approach** (typed):
```wit
initialize: func(params: initialize-params) -> future<result<initialize-result, error>>;
list-resources: func(params: paginated-request) -> future<result<list-resources-result, error>>;
call-tool: func(name: string, arguments: option<list<u8>>) -> future<result<call-tool-result, error>>;
```

Benefits:
- Compiler catches protocol errors at build time
- No need for components to parse JSON-RPC
- Clear async boundaries for each operation
- Better documentation and tooling support

### WASI Pattern Adherence

Following Phase 3 proposal patterns:

| Pattern | Source | Implementation |
|---------|--------|----------------|
| deps.toml dependency management | wasi-http, wasi-sockets | ✅ wit/deps.toml |
| @since versioning | wasi-http | ✅ All interfaces |
| Multi-interface structure | wasi-http, wasi-filesystem | ✅ 8 focused interfaces |
| Resource-based design | wasi-filesystem | ✅ mcp-server resource |
| Streaming integration | wasi-sockets | ✅ streaming.wit |
| Pollable for events | wasi-io | ✅ subscribe-events |
| Comprehensive error types | wasi-http | ✅ Detailed error-code enum |

### Complete MCP Feature Coverage

| MCP Feature | WIT Location | Status |
|-------------|--------------|--------|
| Protocol initialization | types.wit:initialize-params | ✅ |
| Version negotiation | types.wit:protocol-version | ✅ |
| Server capabilities | capabilities.wit:server-capabilities | ✅ |
| Client capabilities | capabilities.wit:client-capabilities | ✅ |
| Resources | types.wit:resource | ✅ |
| Resource templates (RFC 6570) | types.wit:resource-template | ✅ |
| Resource subscriptions | server.wit:subscribe-resource | ✅ |
| Tools | types.wit:tool | ✅ |
| Tool annotations | content.wit:tool-annotations | ✅ |
| Prompts | types.wit:prompt | ✅ |
| Prompt arguments | types.wit:prompt-argument | ✅ |
| Content blocks | content.wit:content-block | ✅ |
| Embedded resources | content.wit:embedded-resource | ✅ |
| Pagination (cursor) | types.wit:cursor | ✅ |
| Progress tokens | types.wit:progress-token | ✅ |
| Notifications | notifications.wit | ✅ |
| Logging | capabilities.wit:logging-capability | ✅ |
| Sampling | capabilities.wit:sampling-capability | ✅ |
| Elicitation | capabilities.wit:elicitation-capability | ✅ |

## Implementation Status

### Completed (Workstreams 1-4)

- [x] Core types with all MCP protocol types
- [x] Complete capabilities system
- [x] Content type system with variants
- [x] Typed server operations for all MCP methods
- [x] Typed client operations
- [x] Notification system
- [x] Streaming interface
- [x] World definitions
- [x] deps.toml with WASI dependencies
- [x] @since annotations throughout

### In Progress (Workstream 5)

- [ ] README documentation (this file)
- [ ] Usage examples
- [ ] Migration guide
- [ ] API reference documentation

### Next Steps for Phase 1

1. ✅ Complete WIT interface definitions (Preview3 + Preview2)
2. ✅ Validate WIT files with wasm-tools (via rules_wasm_component)
3. Create reference implementations:
   - [ ] Rust MCP server (Preview2 for immediate demo)
   - [ ] Go MCP client (Preview2 for immediate demo)
   - [ ] C++ MCP proxy (Preview2 for immediate demo)
   - [ ] Preview3 examples (when toolchain support is ready)
4. Gather stakeholder feedback (Anthropic, WASI Subgroup)
5. Submit Phase 1 proposal

## Stakeholder Interest

**Target Stakeholders:**
- **Anthropic** (MCP creators) - Protocol alignment validation
- **WASI Subgroup** - Review of WASI patterns and conventions
- **WebAssembly Component Model WG** - Resource management patterns
- **MCP SDK maintainers** - Rust, TypeScript implementations

**Key Questions for Stakeholders:**
1. Does the protocol-faithful approach correctly map MCP semantics?
2. Are there missing MCP features in the WIT definitions?
3. Do the async patterns align with WASI Preview3 direction?
4. Is the interface granularity appropriate?

## References

### MCP Specification
- [MCP 2025-06-18 Specification](https://github.com/modelcontextprotocol/specification)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Documentation](https://modelcontextprotocol.io)

### WASI Proposals (Phase 3 Patterns)
- [wasi-http](https://github.com/WebAssembly/wasi-http) - Protocol-faithful HTTP mapping
- [wasi-filesystem](https://github.com/WebAssembly/wasi-filesystem) - Resource management patterns
- [wasi-sockets](https://github.com/WebAssembly/wasi-sockets) - Async and streaming patterns

### WebAssembly
- [Component Model](https://github.com/WebAssembly/component-model)
- [WIT Language](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md)
- [WASI Preview 3](https://github.com/WebAssembly/WASI)

## License

This proposal is licensed under the Apache License 2.0.
