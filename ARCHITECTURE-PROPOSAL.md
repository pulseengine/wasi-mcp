# WASI MCP Architecture Proposal
## Redesign: From Protocol Operations to Runtime + Handlers

---

## Overview

This proposal redesigns WASI MCP to follow established WASI system interface patterns, specifically the **bidirectional pattern** exemplified by wasi-http. The new architecture separates concerns properly:

- **Host**: Provides MCP runtime (protocol, transport, middleware)
- **Component**: Imports runtime + exports handlers (domain logic only)

---

## Current vs Proposed Architecture

### Current (INCORRECT)

```wit
world mcp-server-world {
    export server;  // Component exports full MCP protocol
}

interface server {
    resource mcp-server {
        initialize: func(...) -> future<result<...>>;
        list-tools: func(...) -> future<result<...>>;
        call-tool: func(...) -> future<result<...>>;
        // ... 15+ protocol methods
    }
}
```

**Problems**:
- ❌ Export-only (no capability imports)
- ❌ Component implements protocol details
- ❌ 150+ lines for hello world
- ❌ No separation of concerns
- ❌ Violates WASI system interface patterns

### Proposed (CORRECT)

```wit
world mcp-backend {
    import runtime;   // Component uses MCP framework
    export handlers;  // Component provides callbacks
}

interface runtime {
    register-server: func(info: server-info) -> result<_, error>;
    register-tools: func(tools: list<tool-definition>) -> result<_, error>;
    serve: func() -> result<_, error>;
}

interface handlers {
    call-tool: func(name: string, args: list<u8>) -> result<tool-result, error>;
    read-resource: func(uri: string) -> result<resource-contents, error>;
}
```

**Benefits**:
- ✅ Bidirectional (import + export)
- ✅ Host handles protocol, component handles logic
- ✅ ~60 lines for hello world
- ✅ Clear separation of concerns
- ✅ Follows wasi-http pattern

---

## Complete WIT Specification

### Package Structure

```
wit/
├── types.wit              # Shared types (keep existing)
├── content.wit            # Content blocks (keep existing)
├── capabilities.wit       # Capability definitions (keep existing)
├── runtime.wit            # NEW: Runtime interface (component imports)
├── handlers.wit           # NEW: Handler interface (component exports)
├── client.wit             # Client interface (for MCP clients)
├── world.wit              # NEW: Worlds with runtime+handlers
├── deps.toml              # Dependencies
└── preview2/              # Preview2 versions (blocking)
    ├── types.wit
    ├── runtime.wit
    ├── handlers.wit
    └── world.wit
```

### runtime.wit (Complete)

```wit
package wasi:mcp@0.1.0;

/// MCP Runtime Interface
///
/// Components import this to register capabilities and start serving.
/// The host provides the MCP protocol implementation, transport layer,
/// authentication, middleware, and lifecycle management.
@since(version = 0.1.0)
interface runtime {
    use types.{
        error,
        tool-definition,
        resource-definition,
        resource-template,
        prompt-definition,
        notification,
        progress-token,
        log-level,
    };
    use capabilities.{server-capabilities};

    /// Server metadata and capabilities
    record server-info {
        name: string,
        version: string,
        capabilities: server-capabilities,
        instructions: option<string>,
    }

    /// Register this component's server information
    ///
    /// Must be called once before calling serve().
    /// Defines what capabilities this component provides.
    ///
    /// Example:
    /// ```rust
    /// runtime::register_server(ServerInfo {
    ///     name: "hello-world".to_string(),
    ///     version: "1.0.0".to_string(),
    ///     capabilities: ServerCapabilities {
    ///         tools: Some(ToolsCapability {}),
    ///         ..Default::default()
    ///     },
    ///     instructions: Some("A simple hello world server".to_string()),
    /// })?;
    /// ```
    @since(version = 0.1.0)
    register-server: func(info: server-info) -> result<_, error>;

    /// Register tools this component provides
    ///
    /// Can be called multiple times to register additional tools.
    /// Each tool must have a unique name.
    /// The input-schema must be valid JSON Schema.
    ///
    /// Example:
    /// ```rust
    /// runtime::register_tools(vec![
    ///     ToolDefinition {
    ///         name: "say_hello".to_string(),
    ///         description: "Say hello to someone".to_string(),
    ///         input_schema: json!({
    ///             "type": "object",
    ///             "properties": {
    ///                 "name": {"type": "string"}
    ///             }
    ///         }).to_vec(),
    ///         ..Default::default()
    ///     }
    /// ])?;
    /// ```
    @since(version = 0.1.0)
    register-tools: func(tools: list<tool-definition>) -> result<_, error>;

    /// Register resources this component provides
    ///
    /// Can be called multiple times to register additional resources.
    /// Each resource must have a unique URI.
    ///
    /// Example:
    /// ```rust
    /// runtime::register_resources(vec![
    ///     ResourceDefinition {
    ///         uri: "file:///README.md".to_string(),
    ///         name: "README".to_string(),
    ///         description: Some("Project README file".to_string()),
    ///         mime_type: Some("text/markdown".to_string()),
    ///         ..Default::default()
    ///     }
    /// ])?;
    /// ```
    @since(version = 0.1.0)
    register-resources: func(resources: list<resource-definition>) -> result<_, error>;

    /// Register resource templates (URI templates per RFC 6570)
    ///
    /// Can be called multiple times to register additional templates.
    /// Templates allow dynamic resource URIs like "file:///{path}".
    ///
    /// Example:
    /// ```rust
    /// runtime::register_resource_templates(vec![
    ///     ResourceTemplate {
    ///         uri_template: "file:///{path}".to_string(),
    ///         name: "file".to_string(),
    ///         description: Some("Read any file".to_string()),
    ///         mime_type: Some("application/octet-stream".to_string()),
    ///     }
    /// ])?;
    /// ```
    @since(version = 0.1.0)
    register-resource-templates: func(templates: list<resource-template>) -> result<_, error>;

    /// Register prompts this component provides
    ///
    /// Can be called multiple times to register additional prompts.
    /// Each prompt must have a unique name.
    ///
    /// Example:
    /// ```rust
    /// runtime::register_prompts(vec![
    ///     PromptDefinition {
    ///         name: "code_review".to_string(),
    ///         description: Some("Review code for issues".to_string()),
    ///         arguments: Some(vec![
    ///             PromptArgument {
    ///                 name: "language".to_string(),
    ///                 description: Some("Programming language".to_string()),
    ///                 required: Some(true),
    ///             }
    ///         ]),
    ///     }
    /// ])?;
    /// ```
    @since(version = 0.1.0)
    register-prompts: func(prompts: list<prompt-definition>) -> result<_, error>;

    /// Start serving MCP requests
    ///
    /// This function blocks and runs the MCP server event loop.
    /// The host handles:
    /// - MCP protocol (initialize, list-tools, call-tool, etc.)
    /// - Transport layer (stdio, HTTP, WebSocket)
    /// - Authentication and authorization
    /// - Middleware (rate limiting, logging, monitoring)
    /// - Request routing to component handlers
    ///
    /// Call this after all registration calls are complete.
    ///
    /// Example:
    /// ```rust
    /// runtime::serve()?;  // Blocks until server shuts down
    /// ```
    @since(version = 0.1.0)
    serve: func() -> result<_, error>;

    /// Send a notification to connected clients
    ///
    /// Used to notify clients about:
    /// - Resource updates (resources-updated)
    /// - Resource list changes (resources-list-changed)
    /// - Tool list changes (tools-list-changed)
    /// - Prompt list changes (prompts-list-changed)
    /// - Progress updates (progress)
    /// - Messages (message)
    ///
    /// Example:
    /// ```rust
    /// runtime::send_notification(Notification::ResourcesListChanged(
    ///     ResourcesListChangedNotification {}
    /// ))?;
    /// ```
    @since(version = 0.1.0)
    send-notification: func(notification: notification) -> result<_, error>;

    /// Log a message to MCP logging protocol
    ///
    /// If logging capability is enabled, this sends a log message
    /// to connected clients through the MCP logging protocol.
    ///
    /// Example:
    /// ```rust
    /// runtime::log(
    ///     LogLevel::Info,
    ///     "Tool execution completed".to_string(),
    ///     None
    /// )?;
    /// ```
    @since(version = 0.1.0)
    log: func(level: log-level, message: string, data: option<list<u8>>) -> result<_, error>;

    /// Report progress for long-running operations
    ///
    /// For operations that support progress tracking, report current progress.
    /// Clients can display progress bars or status updates.
    ///
    /// Example:
    /// ```rust
    /// runtime::report_progress(
    ///     "upload-progress".to_string(),
    ///     50,  // current progress
    ///     Some(100)  // total (optional)
    /// )?;
    /// ```
    @since(version = 0.1.0)
    report-progress: func(
        token: progress-token,
        progress: u64,
        total: option<u64>
    ) -> result<_, error>;
}
```

### handlers.wit (Complete)

```wit
package wasi:mcp@0.1.0;

/// MCP Handler Interface
///
/// Components export this to handle incoming MCP requests.
/// The host calls these functions when MCP protocol operations arrive.
/// All JSON parsing, validation, and protocol handling is done by the host.
@since(version = 0.1.0)
interface handlers {
    use types.{error};
    use content.{content-block, prompt-message};

    /// Result of tool execution
    record tool-result {
        /// Content blocks returned by the tool
        content: list<content-block>,
        /// Whether this is an error result
        is-error: option<bool>,
    }

    /// Contents of a resource
    record resource-contents {
        /// Content blocks for the resource
        contents: list<content-block>,
    }

    /// Contents of a prompt
    record prompt-contents {
        /// Prompt messages to display
        messages: list<prompt-message>,
    }

    /// Completion suggestion
    record completion {
        /// Completion value
        value: string,
        /// Optional label for display
        label: option<string>,
    }

    /// Reference type for completion
    enum completion-ref-type {
        /// Complete tool name
        tool-name,
        /// Complete resource URI
        resource-uri,
        /// Complete prompt name
        prompt-name,
        /// Complete argument name
        argument-name,
    }

    /// Elicitation field
    record elicitation-field {
        /// Field name
        name: string,
        /// Field type (e.g., "string", "number", "boolean")
        field-type: string,
        /// Field description
        description: option<string>,
        /// Whether field is required
        required: option<bool>,
    }

    /// Elicitation response
    record elicitation-response {
        /// Collected field values (JSON)
        values: list<u8>,
    }

    /// Called when a client requests tool execution
    ///
    /// The host has already:
    /// - Validated the tool name exists
    /// - Validated arguments against the tool's input_schema
    /// - Parsed the JSON into bytes
    ///
    /// The component should:
    /// - Parse arguments (already validated)
    /// - Execute the tool logic
    /// - Return result content blocks
    ///
    /// Arguments are JSON bytes. For Rust:
    /// ```rust
    /// let params: MyParams = serde_json::from_slice(&arguments)?;
    /// ```
    ///
    /// Example:
    /// ```rust
    /// fn call_tool(name: String, arguments: Vec<u8>) -> Result<ToolResult, Error> {
    ///     match name.as_str() {
    ///         "say_hello" => {
    ///             let params: SayHelloParams = serde_json::from_slice(&arguments)?;
    ///             Ok(ToolResult {
    ///                 content: vec![ContentBlock::Text(TextContent {
    ///                     text: format!("Hello, {}!", params.name.unwrap_or("World".into())),
    ///                     ..Default::default()
    ///                 })],
    ///                 is_error: Some(false),
    ///             })
    ///         }
    ///         _ => Err(Error::tool_not_found(name))
    ///     }
    /// }
    /// ```
    @since(version = 0.1.0)
    call-tool: func(
        name: string,
        arguments: list<u8>
    ) -> result<tool-result, error>;

    /// Called when a client reads a resource
    ///
    /// The host has already:
    /// - Validated the resource URI exists or matches a template
    ///
    /// The component should:
    /// - Read the resource data
    /// - Return content blocks (text, image, etc.)
    ///
    /// Example:
    /// ```rust
    /// fn read_resource(uri: String) -> Result<ResourceContents, Error> {
    ///     if uri == "file:///README.md" {
    ///         Ok(ResourceContents {
    ///             contents: vec![ContentBlock::Text(TextContent {
    ///                 text: std::fs::read_to_string("README.md")?,
    ///                 content_type: "text/markdown".to_string(),
    ///                 ..Default::default()
    ///             })]
    ///         })
    ///     } else {
    ///         Err(Error::resource_not_found(uri))
    ///     }
    /// }
    /// ```
    @since(version = 0.1.0)
    read-resource: func(uri: string) -> result<resource-contents, error>;

    /// Called when a client requests a prompt
    ///
    /// The host has already:
    /// - Validated the prompt name exists
    /// - Parsed arguments (if provided)
    ///
    /// The component should:
    /// - Generate prompt messages based on arguments
    /// - Return prompt messages (system, user, assistant)
    ///
    /// Example:
    /// ```rust
    /// fn get_prompt(name: String, arguments: Option<Vec<u8>>) -> Result<PromptContents, Error> {
    ///     match name.as_str() {
    ///         "code_review" => {
    ///             let args: CodeReviewArgs = serde_json::from_slice(&arguments.unwrap())?;
    ///             Ok(PromptContents {
    ///                 messages: vec![
    ///                     PromptMessage {
    ///                         role: MessageRole::System,
    ///                         content: "You are a code reviewer".to_string(),
    ///                     },
    ///                     PromptMessage {
    ///                         role: MessageRole::User,
    ///                         content: format!("Review this {} code", args.language),
    ///                     }
    ///                 ]
    ///             })
    ///         }
    ///         _ => Err(Error::prompt_not_found(name))
    ///     }
    /// }
    /// ```
    @since(version = 0.1.0)
    get-prompt: func(
        name: string,
        arguments: option<list<u8>>
    ) -> result<prompt-contents, error>;

    /// Called when a client subscribes to resource updates (optional)
    ///
    /// If subscriptions capability is enabled, this is called when
    /// a client wants to receive notifications about resource changes.
    ///
    /// The component should:
    /// - Track the subscription
    /// - Later call runtime::send_notification() when resource changes
    ///
    /// Return error with code "unsupported-operation" if subscriptions
    /// are not supported.
    ///
    /// Example:
    /// ```rust
    /// fn handle_subscribe(uri: String) -> Result<(), Error> {
    ///     SUBSCRIPTIONS.lock().unwrap().insert(uri);
    ///     Ok(())
    /// }
    /// ```
    @since(version = 0.1.0)
    handle-subscribe: func(uri: string) -> result<_, error>;

    /// Called when a client unsubscribes from resource updates (optional)
    ///
    /// The component should stop tracking this subscription.
    ///
    /// Example:
    /// ```rust
    /// fn handle_unsubscribe(uri: String) -> Result<(), Error> {
    ///     SUBSCRIPTIONS.lock().unwrap().remove(&uri);
    ///     Ok(())
    /// }
    /// ```
    @since(version = 0.1.0)
    handle-unsubscribe: func(uri: string) -> result<_, error>;

    /// Called for auto-completion requests (optional)
    ///
    /// If completions capability is enabled, this is called to provide
    /// auto-completion suggestions for tool names, resource URIs, etc.
    ///
    /// Example:
    /// ```rust
    /// fn handle_complete(ref_type: CompletionRefType, ref_value: String) -> Result<Vec<Completion>, Error> {
    ///     match ref_type {
    ///         CompletionRefType::ResourceUri => {
    ///             // Complete file paths
    ///             Ok(vec![
    ///                 Completion { value: "file:///src/".into(), label: Some("src/".into()) },
    ///                 Completion { value: "file:///tests/".into(), label: Some("tests/".into()) },
    ///             ])
    ///         }
    ///         _ => Ok(vec![])
    ///     }
    /// }
    /// ```
    @since(version = 0.1.0)
    handle-complete: func(
        ref-type: completion-ref-type,
        ref-value: string
    ) -> result<list<completion>, error>;

    /// Called for elicitation requests (optional)
    ///
    /// If elicitation capability is enabled, this is called to request
    /// structured input from the user through the client UI.
    ///
    /// The component should return the collected field values as JSON.
    ///
    /// Example:
    /// ```rust
    /// fn handle_elicit(message: String, fields: Vec<ElicitationField>) -> Result<ElicitationResponse, Error> {
    ///     // This would typically be implemented by the host/client, not component
    ///     Err(Error::not_supported("Elicitation not supported"))
    /// }
    /// ```
    @since(version = 0.1.0)
    handle-elicit: func(
        message: string,
        fields: list<elicitation-field>
    ) -> result<elicitation-response, error>;
}
```

### world.wit (Complete)

```wit
package wasi:mcp@0.1.0;

/// MCP Backend Component World
///
/// Use this world for components that provide MCP tools, resources, or prompts.
///
/// The component:
/// - Imports `runtime` to register capabilities and start serving
/// - Exports `handlers` to execute tools/resources/prompts
/// - Imports standard WASI capabilities for I/O, time, etc.
///
/// Example:
/// ```rust
/// // Component exports handlers
/// impl handlers::Guest for MyComponent {
///     fn call_tool(name: String, args: Vec<u8>) -> Result<ToolResult, Error> {
///         // Implementation
///     }
///     // ... other handlers
/// }
///
/// fn main() {
///     runtime::register_server(...)?;
///     runtime::register_tools(...)?;
///     runtime::serve()?;
/// }
/// ```
@since(version = 0.1.0)
world mcp-backend {
    /// Component imports MCP runtime to register and serve
    import runtime;

    /// Component exports handlers to execute tools
    export handlers;

    /// Standard WASI capabilities for I/O
    import wasi:io/poll@0.2.3;
    import wasi:io/streams@0.2.3;

    /// Standard WASI capabilities for time
    import wasi:clocks/wall-clock@0.2.3;
    import wasi:clocks/monotonic-clock@0.2.3;
}

/// MCP Client Component World
///
/// Use this world for components that act as MCP clients (call MCP servers).
///
/// The component:
/// - Imports `client` to make MCP requests
/// - Imports standard WASI capabilities
///
/// Example:
/// ```rust
/// fn main() {
///     let client = client::connect("stdio")?;
///     client.initialize(...)?;
///     let tools = client.list_tools(...)?;
///     let result = client.call_tool("say_hello", args)?;
/// }
/// ```
@since(version = 0.1.0)
world mcp-client {
    /// Component imports MCP client operations
    import client;

    /// Standard WASI capabilities
    import wasi:io/poll@0.2.3;
    import wasi:io/streams@0.2.3;
    import wasi:clocks/wall-clock@0.2.3;
    import wasi:clocks/monotonic-clock@0.2.3;
}
```

---

## Example: Hello World Component

### Rust Implementation (~60 lines)

```rust
// hello-world component using WASI MCP

use wasi::mcp::{runtime, handlers, types, content};
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct SayHelloParams {
    name: Option<String>,
}

// Component exports handlers interface
struct MyHandlers;

impl handlers::Guest for MyHandlers {
    fn call_tool(name: String, arguments: Vec<u8>) -> Result<handlers::ToolResult, types::Error> {
        match name.as_str() {
            "say_hello" => {
                let params: SayHelloParams = serde_json::from_slice(&arguments)
                    .map_err(|e| types::Error::invalid_params(e.to_string()))?;

                let greeting = format!(
                    "Hello, {}!",
                    params.name.unwrap_or_else(|| "World".to_string())
                );

                Ok(handlers::ToolResult {
                    content: vec![content::ContentBlock::Text(content::TextContent {
                        text: greeting,
                        content_type: "text/plain".to_string(),
                        annotations: None,
                    })],
                    is_error: Some(false),
                })
            }
            _ => Err(types::Error::tool_not_found(name))
        }
    }

    fn read_resource(_uri: String) -> Result<handlers::ResourceContents, types::Error> {
        Err(types::Error::not_supported("Resources not provided".to_string()))
    }

    fn get_prompt(_name: String, _args: Option<Vec<u8>>) -> Result<handlers::PromptContents, types::Error> {
        Err(types::Error::not_supported("Prompts not provided".to_string()))
    }

    fn handle_subscribe(_uri: String) -> Result<(), types::Error> {
        Err(types::Error::not_supported("Subscriptions not supported".to_string()))
    }

    fn handle_unsubscribe(_uri: String) -> Result<(), types::Error> {
        Err(types::Error::not_supported("Subscriptions not supported".to_string()))
    }

    fn handle_complete(_ref_type: handlers::CompletionRefType, _ref_value: String) -> Result<Vec<handlers::Completion>, types::Error> {
        Ok(vec![])
    }

    fn handle_elicit(_message: String, _fields: Vec<handlers::ElicitationField>) -> Result<handlers::ElicitationResponse, types::Error> {
        Err(types::Error::not_supported("Elicitation not supported".to_string()))
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Register server info
    runtime::register_server(runtime::ServerInfo {
        name: "hello-world".to_string(),
        version: "1.0.0".to_string(),
        capabilities: capabilities::ServerCapabilities {
            tools: Some(capabilities::ToolsCapability {}),
            ..Default::default()
        },
        instructions: Some("A simple hello world MCP server".to_string()),
    })?;

    // Register tools
    runtime::register_tools(vec![
        types::ToolDefinition {
            name: "say_hello".to_string(),
            title: Some("Say Hello".to_string()),
            description: "Say hello to someone".to_string(),
            input_schema: serde_json::to_vec(&serde_json::json!({
                "type": "object",
                "properties": {
                    "name": {
                        "type": "string",
                        "description": "Name to greet (optional)"
                    }
                }
            }))?,
            output_schema: None,
            annotations: None,
        }
    ])?;

    // Start serving (blocks)
    runtime::serve()?;

    Ok(())
}
```

**Lines of Code**: ~110 lines (vs 150+ with protocol-faithful design)

With macros, this reduces to ~25 lines (same as production Rust MCP).

---

## Migration Plan

### Phase 1: Update WIT Interfaces (Week 1)

**Tasks**:
1. Create new `runtime.wit` interface
2. Create new `handlers.wit` interface
3. Update `world.wit` with `mcp-backend` world
4. Keep `types.wit`, `content.wit`, `capabilities.wit` (unchanged)
5. Archive old `server.wit` to `archive/`
6. Create Preview2 versions (blocking operations)
7. Update `README.md` with new architecture

**Validation**:
- Bazel validation with `rules_wasm_component`
- Both Preview2 and Preview3 validate successfully
- Generate bindings for Rust/Go/C++

### Phase 2: Reference Implementations (Week 2-3)

**Rust**:
1. Generate bindings: `wit-bindgen rust wit/ --out-dir src/bindings/`
2. Implement hello-world example (~60 lines)
3. Implement file-server example (tools + resources)
4. Test with MCP Inspector

**Go**:
1. Generate bindings: `wit-bindgen-go wit/ --out-dir bindings/`
2. Implement hello-world example
3. Test with MCP Inspector

### Phase 3: Host Runtime (Week 4-6)

**Wasmtime Host**:
1. Implement `runtime` interface in Rust
   - Registration functions
   - MCP protocol handler
   - Stdio transport
2. Implement component lifecycle
   - Load WASM component
   - Call registration functions
   - Route MCP requests to handlers
3. Add HTTP transport
4. Add authentication middleware
5. Add monitoring/metrics

### Phase 4: Macro Helpers (Week 7-8)

**Rust Macros**:
```rust
#[wasi_mcp::server(name = "hello-world", version = "1.0.0")]
struct HelloWorld;

#[wasi_mcp::tools]
impl HelloWorld {
    /// Say hello to someone
    async fn say_hello(name: Option<String>) -> Result<String> {
        Ok(format!("Hello, {}!", name.unwrap_or("World".to_string())))
    }
}
```

Generates:
- Server registration
- Tool definitions with JSON schema
- Handler implementations with dispatch
- Serve call

### Phase 5: Documentation (Week 9)

**Content**:
1. Tutorial: "Building Your First MCP Component"
2. API Reference (generated from WIT)
3. Examples: hello-world, file-server, api-gateway
4. Migration guide from protocol-faithful design
5. Best practices

---

## Open Questions & Decisions Needed

### Q1: Keep protocol-faithful design?

**Options**:
- A: Remove entirely, replace with runtime+handlers
- B: Keep both (Layer 1: protocol, Layer 2: runtime)

**Recommendation**: **Option A** - Replace entirely. No clear use case for protocol-faithful in components.

### Q2: Optional capabilities handling?

**Options**:
- A: All handlers required, return "not supported" errors
- B: Multiple worlds (mcp-backend-tools-only, mcp-backend-full)

**Recommendation**: **Option A** - Simpler. Capabilities declared in `register_server()`, errors for unsupported operations.

### Q3: Streaming for large resources?

**Options**:
- A: Add streaming resource type with wasi:io/streams
- B: Use content-block list, host buffers

**Recommendation**: **Option B** for MVP, add **Option A** later if needed.

### Q4: Authentication/secrets?

**Options**:
- A: Component handles auth internally
- B: Component imports wasi:keyvalue/secrets from host

**Recommendation**: **Option B** - Host manages secrets, component imports capability.

### Q5: Preview2 vs Preview3?

**Options**:
- A: Preview2 only (blocking, works today)
- B: Preview3 only (async, future-proof)
- C: Both (dual release)

**Recommendation**: **Option C** - Both. Preview2 for immediate adoption, Preview3 when toolchains ready.

---

## Success Criteria

A successful redesign will achieve:

1. ✅ **Follows WASI patterns**: Bidirectional (import+export) like wasi-http
2. ✅ **Minimal boilerplate**: ~60 lines for hello world (vs 150+)
3. ✅ **Clear separation**: Host = protocol, component = domain logic
4. ✅ **Macro-friendly**: Can add ergonomic macros like production Rust MCP
5. ✅ **Extensible**: New MCP features don't break existing components
6. ✅ **Secure**: Host controls auth, middleware, rate limiting
7. ✅ **Performant**: Async support (Preview3), zero-copy where possible
8. ✅ **Production-ready**: Matches or exceeds production MCP ergonomics

---

## Timeline

| Week | Phase | Deliverable |
|------|-------|-------------|
| 1 | WIT Redesign | New runtime.wit, handlers.wit, world.wit |
| 2-3 | Reference Impls | Rust + Go hello-world examples |
| 4-6 | Host Runtime | Wasmtime host with stdio + HTTP transport |
| 7-8 | Macro Helpers | Rust #[wasi_mcp::server] macro |
| 9 | Documentation | Tutorial, API reference, examples |

**Total**: ~9 weeks to production-ready WASI MCP system interface

---

## Conclusion

This redesign transforms WASI MCP from a protocol-operations interface (wrong pattern) to a proper WASI system interface following the bidirectional pattern established by wasi-http.

**Key Changes**:
- Component **imports** runtime (capability provider)
- Component **exports** handlers (simple callbacks)
- Host provides protocol, transport, middleware
- Component provides domain logic only

This architecture positions WASI MCP as a true system interface that developers will want to use.
