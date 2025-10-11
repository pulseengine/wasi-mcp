# WASI System Interface Pattern Analysis
## Understanding What Makes a WASI System Interface

---

## Executive Summary

After analyzing all Phase 3 WASI proposals, I've identified **fundamental architectural patterns** that distinguish WASI system interfaces from application interfaces. Our current WASI MCP design follows the wrong pattern. This document proposes a corrected architecture that properly positions MCP as a WASI system interface.

**Key Finding**: WASI system interfaces are **capability providers imported by components**, not protocol operations exported by components. MCP must follow this pattern.

---

## Part 1: WASI System Interface Taxonomy

### Pattern 1: Pure Import (System Capability Provider)

**Examples**: wasi-io, wasi-filesystem, wasi-nn, wasi-clocks, wasi-random, wasi-sockets

**Characteristics**:
- Host owns the actual resource (filesystem, GPU, clock, network)
- Component imports capabilities from host
- Unidirectional: host → component
- Security model: host controls access, component requests

**WIT Pattern**:
```wit
// wasi-nn example
world ml {
    import inference;  // Component uses host's ML capabilities
    import graph;      // Host provides GPU/TPU access
    import tensor;     // Host manages hardware memory
}
```

**Why This Pattern**:
- Host has exclusive access to hardware/system resources
- Components are sandboxed and need permission
- Performance: host can optimize system-level operations
- Security: capability-based access control

**Examples**:
- **wasi-filesystem**: Host owns filesystem, component requests file operations
- **wasi-nn**: Host owns GPU/ML accelerators, component requests inference
- **wasi-io**: Host owns streams/sockets, component performs I/O
- **wasi-clocks**: Host owns system time, component queries current time

### Pattern 2: Bidirectional (Application Protocol)

**Example**: wasi-http

**Characteristics**:
- Component both consumes and provides services
- Bidirectional: host ⟷ component
- Component exports handlers, imports capabilities
- Used for proxies, middleware, services

**WIT Pattern**:
```wit
// wasi-http example
world proxy {
    // Component imports: make outgoing HTTP requests
    import wasi:http/outgoing-handler;
    import wasi:io/streams;
    import wasi:clocks/wall-clock;

    // Component exports: handle incoming HTTP requests
    export wasi:http/incoming-handler;
}

interface incoming-handler {
    /// Component implements this to handle incoming requests
    handle: func(request: incoming-request, response-out: response-outparam);
}

interface outgoing-handler {
    /// Component calls this to make outgoing requests
    handle: func(request: outgoing-request) -> result<incoming-response, error>;
}
```

**Why This Pattern**:
- HTTP services need to both receive requests (export handler) and make requests (import capabilities)
- Component acts as intermediary: receives from host, processes, calls out to services
- Typical for proxy, gateway, middleware patterns

**Key Insight**: Even here, the **handler is exported but simple**, while **complex capabilities are imported**

### Pattern 3: Pure Export (Component Provides Capability)

**Status**: **Not found in Phase 3 WASI proposals**

**Theoretical Pattern**:
```wit
world my-service {
    export my-capability;  // Component provides service to host
}
```

**Why Not Used**:
- WASI focuses on system capabilities (filesystem, network, time)
- Components don't typically provide system-level capabilities to hosts
- Security model doesn't support components controlling host resources
- If components provide services, they use **Pattern 2** (bidirectional)

---

## Part 2: System Interface vs Application Interface

### What Makes a WASI System Interface?

Based on analyzing all Phase 3 proposals, a WASI **system interface** has these characteristics:

#### 1. Capability-Based Security Model
- Host owns the actual resource
- Component requests access through capability handles
- Borrowed or owned resources with clear lifecycle
- Example: `descriptor` in wasi-filesystem is a borrowed capability

#### 2. Hardware/OS Abstraction
- Provides portable access to platform-specific resources
- Abstracts differences between Linux/Windows/macOS/WASI
- Example: wasi-clocks abstracts monotonic vs wall-clock time
- Example: wasi-nn abstracts TensorFlow/PyTorch/ONNX backends

#### 3. Performance-Critical Path
- Operations need to be fast (often zero-copy)
- Direct access to system resources without serialization overhead
- Example: wasi-io streams use buffer sharing, not copying
- Example: wasi-nn tensors reference shared memory

#### 4. Lifecycle Management
- Resources have clear creation, usage, disposal semantics
- Host manages resource lifecycle (files, sockets, ML models)
- Component borrows capabilities temporarily
- Example: `descriptor.drop()` in wasi-filesystem

#### 5. Foundational Dependency
- Other interfaces build on top of system interfaces
- Example: wasi-http depends on wasi-io/streams
- Example: Many interfaces depend on wasi-clocks for timestamps

### What is an Application Interface?

In contrast, an **application interface** (our current MCP design):

#### 1. Protocol Operations
- Defines request/response message flow
- Example: `list-tools()`, `call-tool()`, `read-resource()`
- Protocol-faithful: maps 1:1 to JSON-RPC methods

#### 2. Domain-Specific Logic
- Not hardware abstraction
- Not OS-level operations
- Business/application logic (tools, AI agents, data access)

#### 3. Composable Services
- Components provide functionality to be orchestrated
- Not system-level capabilities
- Example: MCP server provides tools to AI agents

#### 4. Higher-Level Abstraction
- Built on top of system interfaces
- Example: MCP uses wasi-io for transport, wasi-http for HTTP transport

---

## Part 3: Why Our Current MCP Design is Wrong

### Our Current Design (INCORRECT)

```wit
// Current wasi-mcp design
world mcp-server-world {
    export server;  // ❌ Component exports full MCP protocol
}

interface server {
    resource mcp-server {
        initialize: func(...) -> future<result<...>>;
        list-tools: func(...) -> future<result<...>>;
        call-tool: func(...) -> future<result<...>>;
        list-resources: func(...) -> future<result<...>>;
        read-resource: func(...) -> future<result<...>>;
        list-prompts: func(...) -> future<result<...>>;
        get-prompt: func(...) -> future<result<...>>;
        subscribe: func(...) -> future<result<...>>;
        complete: func(...) -> future<result<...>>;
        elicit: func(...) -> future<result<...>>;
        set-level: func(...) -> future<result<...>>;
        // ... 15+ protocol methods
    }
}
```

### Problems

#### 1. **Wrong Direction**
- We have components **exporting** MCP protocol
- Should follow wasi-nn pattern: components **import** MCP runtime
- Host should provide MCP framework, components use it

#### 2. **Wrong Abstraction Level**
- Exposing full protocol operations (initialize, pagination, cursors)
- Should expose only: "here are my tools, here's how to call them"
- Protocol handling should be in host, not component

#### 3. **Not a System Capability**
- MCP is not a hardware abstraction (like filesystem or GPU)
- MCP is a protocol/framework for AI-to-tool communication
- Components don't provide "MCP hardware", they provide domain tools

#### 4. **Mismatched Lifecycle**
- Our `resource mcp-server` suggests stateful protocol handler
- Reality: components just need to register tools and provide callbacks
- Lifecycle should be: register → serve → call handlers

#### 5. **Transport Confusion**
- Our design includes protocol operations (initialize, list, call)
- Transport (stdio, HTTP, WebSocket) should be host responsibility
- Component shouldn't care about JSON-RPC, stdio, or HTTP

#### 6. **Missing the Point**
- Developers want: "I have these functions, make them available to AI"
- Our design forces: "Implement full MCP protocol with pagination, cursors, progress tokens"
- Compare:
  - Production (with macros): **25 lines** `#[mcp_tools] impl { async fn my_tool() }`
  - Our WIT: **150+ lines** implementing all protocol methods

---

## Part 4: The Correct WASI MCP Architecture

### Architectural Principle

**MCP should follow the wasi-http pattern**:
- Component **imports** MCP runtime (framework/registration API)
- Component **exports** handlers (simple callbacks for tool execution)
- Host provides MCP protocol, transport, auth, middleware

### Proposed Architecture

```wit
package wasi:mcp@0.1.0;

/// ========================================
/// PART 1: RUNTIME (Component Imports)
/// ========================================
/// The MCP runtime provides framework services to components.
/// This is what components USE to register capabilities and serve requests.

@since(version = 0.1.0)
interface runtime {
    use types.{error, tool-definition, resource-definition, prompt-definition};
    use capabilities.{server-capabilities};

    /// Server metadata and capabilities
    record server-info {
        name: string,
        version: string,
        capabilities: server-capabilities,
        instructions: option<string>,
    }

    /// Register this component's server information
    /// Called once during initialization
    register-server: func(info: server-info) -> result<_, error>;

    /// Register tools this component provides
    /// Can be called multiple times to add tools
    register-tools: func(tools: list<tool-definition>) -> result<_, error>;

    /// Register resources this component provides
    register-resources: func(resources: list<resource-definition>) -> result<_, error>;

    /// Register resource templates (URI templates per RFC 6570)
    register-resource-templates: func(templates: list<resource-template>) -> result<_, error>;

    /// Register prompts this component provides
    register-prompts: func(prompts: list<prompt-definition>) -> result<_, error>;

    /// Start serving requests
    /// This blocks and runs the MCP server event loop
    /// Host handles: protocol, transport, auth, middleware
    serve: func() -> result<_, error>;

    /// Send a notification to connected clients
    /// Used for: resources-updated, tools-changed, progress, etc.
    send-notification: func(notification: notification) -> result<_, error>;

    /// Log a message (sent to MCP logging protocol if enabled)
    log: func(level: log-level, message: string, data: option<list<u8>>) -> result<_, error>;

    /// Report progress for long-running operations
    report-progress: func(
        token: progress-token,
        progress: u64,
        total: option<u64>
    ) -> result<_, error>;
}

/// ========================================
/// PART 2: HANDLERS (Component Exports)
/// ========================================
/// Components export these simple handler interfaces.
/// Host calls these when MCP protocol operations arrive.

@since(version = 0.1.0)
interface handlers {
    use types.{error, content-block};

    /// Called when a client requests tool execution
    /// Arguments are pre-validated JSON (per tool's input_schema)
    /// Return value is serialized back to MCP protocol
    call-tool: func(
        name: string,
        arguments: list<u8>  // JSON bytes
    ) -> result<tool-result, error>;

    /// Called when a client reads a resource
    /// URI is pre-validated against registered resources
    read-resource: func(uri: string) -> result<resource-contents, error>;

    /// Called when a client requests a prompt
    /// Name is pre-validated against registered prompts
    get-prompt: func(
        name: string,
        arguments: option<list<u8>>  // JSON bytes
    ) -> result<prompt-contents, error>;

    /// Called when a client subscribes to resource updates (optional)
    handle-subscribe: func(uri: string) -> result<_, error>;

    /// Called when a client unsubscribes (optional)
    handle-unsubscribe: func(uri: string) -> result<_, error>;

    /// Called for auto-completion requests (optional)
    handle-complete: func(
        ref-type: completion-ref-type,
        ref-value: string
    ) -> result<list<completion>, error>;

    /// Called for elicitation requests (optional)
    handle-elicit: func(
        message: string,
        fields: list<elicitation-field>
    ) -> result<elicitation-response, error>;
}

/// Result types for handlers
record tool-result {
    content: list<content-block>,
    is-error: option<bool>,
}

record resource-contents {
    contents: list<content-block>,
}

record prompt-contents {
    messages: list<prompt-message>,
}

/// ========================================
/// PART 3: WORLDS
/// ========================================

/// MCP backend component world
/// Components that provide MCP tools/resources/prompts
@since(version = 0.1.0)
world mcp-backend {
    /// Component imports runtime to register capabilities
    import runtime;

    /// Component exports handlers to execute tools
    export handlers;

    /// Component can use standard WASI capabilities
    import wasi:io/streams@0.2.3;
    import wasi:clocks/wall-clock@0.2.3;
    import wasi:clocks/monotonic-clock@0.2.3;
}

/// MCP client component world (for building MCP clients in WASM)
@since(version = 0.1.0)
world mcp-client {
    /// Component imports MCP client operations
    import client;

    /// Component can use standard WASI capabilities
    import wasi:io/streams@0.2.3;
    import wasi:clocks/wall-clock@0.2.3;
}
```

### How This Works

#### Component Side (Minimal Boilerplate)

```rust
// Rust component using WASI MCP

use wasi::mcp::{runtime, handlers};

#[derive(serde::Serialize, serde::Deserialize)]
struct SayHelloParams {
    name: Option<String>,
}

// Component exports handlers interface
impl handlers::Guest for MyComponent {
    fn call_tool(name: String, arguments: Vec<u8>) -> Result<ToolResult, Error> {
        match name.as_str() {
            "say_hello" => {
                let params: SayHelloParams = serde_json::from_slice(&arguments)?;
                let msg = format!("Hello, {}!", params.name.unwrap_or("World".to_string()));
                Ok(ToolResult {
                    content: vec![ContentBlock::Text(TextContent {
                        text: msg,
                        ..Default::default()
                    })],
                    is_error: Some(false),
                })
            }
            _ => Err(Error::tool_not_found(name))
        }
    }

    fn read_resource(uri: String) -> Result<ResourceContents, Error> {
        Err(Error::resource_not_found(uri))
    }

    fn get_prompt(name: String, args: Option<Vec<u8>>) -> Result<PromptContents, Error> {
        Err(Error::prompt_not_found(name))
    }
}

fn main() {
    // Register server info
    runtime::register_server(ServerInfo {
        name: "hello-world".to_string(),
        version: "1.0.0".to_string(),
        capabilities: ServerCapabilities {
            tools: Some(ToolsCapability {}),
            ..Default::default()
        },
        instructions: Some("A simple hello world server".to_string()),
    }).unwrap();

    // Register tools
    runtime::register_tools(vec![
        ToolDefinition {
            name: "say_hello".to_string(),
            description: "Say hello to someone".to_string(),
            input_schema: serde_json::to_vec(&json!({
                "type": "object",
                "properties": {
                    "name": {"type": "string", "description": "Name to greet"}
                }
            })).unwrap(),
            ..Default::default()
        }
    ]).unwrap();

    // Start serving (blocks, host handles protocol)
    runtime::serve().unwrap();
}
```

**Lines of code**: ~60 lines (vs 150+ with current design)

#### Host Side (MCP Runtime Implementation)

The host (Wasmtime, wasm-edge, browser, etc.) provides:

1. **MCP Protocol Handler**
   - Implements JSON-RPC 2.0 over MCP protocol
   - Handles: initialize, list-tools, call-tool, notifications
   - Manages pagination cursors, progress tokens, request IDs

2. **Transport Layer**
   - stdio transport (stdin/stdout JSON-RPC)
   - HTTP transport (HTTP Server-Sent Events)
   - WebSocket transport
   - Component doesn't care which transport is used

3. **Middleware Stack**
   - Authentication (API keys, OAuth, mTLS)
   - Authorization (role-based access control)
   - Rate limiting
   - Monitoring/metrics
   - Logging (structured logging, sanitization)
   - Security (input validation, XSS prevention)

4. **Component Lifecycle**
   - Load component WASM
   - Call component's register functions
   - Call component's serve()
   - Route incoming MCP requests to component handlers
   - Serialize/deserialize between MCP protocol and handlers

### Benefits of This Architecture

#### ✅ Correct WASI Pattern
- Follows wasi-http bidirectional model
- Component imports runtime (framework), exports handlers (callbacks)
- Clear separation: host = protocol/transport, component = domain logic

#### ✅ Minimal Component Boilerplate
- ~60 lines for hello world (vs 150+ with protocol-faithful design)
- Just: register + implement handlers
- No protocol concerns (pagination, cursors, progress tokens)

#### ✅ System Interface Characteristics
- **Capability-based**: Component imports MCP runtime capability from host
- **Hardware/OS abstraction**: Host abstracts transport (stdio/HTTP/WebSocket)
- **Lifecycle management**: Host manages MCP session, component handles tool calls
- **Foundational**: MCP runtime can be used by many component backends

#### ✅ Ergonomic for Developers
- Natural flow: register tools → implement handlers → serve
- Easy to add language bindings with macros:
  ```rust
  #[mcp_tool]
  async fn say_hello(name: Option<String>) -> String {
      format!("Hello, {}!", name.unwrap_or("World".to_string()))
  }
  ```
  Macro generates: registration + handler dispatch

#### ✅ Host-Side Protocol Evolution
- Component doesn't implement MCP protocol
- Host can upgrade MCP version without recompiling components
- New MCP features (sampling, elicitation) added as optional handlers

#### ✅ Security and Middleware
- Host controls authentication, authorization, rate limiting
- Component can't bypass security policies
- Monitoring/logging happens at host level

#### ✅ Transport Agnostic
- Component code same for stdio, HTTP, WebSocket
- Host chooses transport based on deployment
- Component just implements: call-tool, read-resource

---

## Part 5: Comparison with Current Design

### Current Design (Protocol-Faithful)

```wit
world mcp-server-world {
    export server;  // ❌ Wrong direction
}

interface server {
    resource mcp-server {
        initialize: func(...)          // ❌ Protocol details in component
        list-tools: func(...)          // ❌ Protocol details
        call-tool: func(...)           // ✅ This is the actual work
        list-resources: func(...)      // ❌ Protocol details
        read-resource: func(...)       // ✅ This is the actual work
        list-prompts: func(...)        // ❌ Protocol details
        get-prompt: func(...)          // ✅ This is the actual work
        subscribe: func(...)           // ❌ Protocol details
        complete: func(...)            // ✅ Optional work
        elicit: func(...)              // ✅ Optional work
        set-level: func(...)           // ❌ Protocol details
        // ...
    }
}
```

**Analysis**:
- 11 methods: 6 are protocol plumbing, 5 are actual work
- Component must implement protocol details (pagination, initialization)
- No separation between framework and domain logic
- Export-only (no capability imports)

### Proposed Design (Runtime + Handlers)

```wit
world mcp-backend {
    import runtime;   // ✅ Component uses framework
    export handlers;  // ✅ Component provides callbacks
}

interface runtime {
    register-server: func(...)      // ✅ Simple registration
    register-tools: func(...)       // ✅ Simple registration
    register-resources: func(...)   // ✅ Simple registration
    serve: func(...)                // ✅ Host handles protocol
    send-notification: func(...)    // ✅ Optional helper
    log: func(...)                  // ✅ Optional helper
}

interface handlers {
    call-tool: func(...)       // ✅ Pure domain logic
    read-resource: func(...)   // ✅ Pure domain logic
    get-prompt: func(...)      // ✅ Pure domain logic
    handle-subscribe: func(...)  // ✅ Optional callback
    handle-complete: func(...)   // ✅ Optional callback
}
```

**Analysis**:
- Runtime: 6 methods, all simple registration/helpers
- Handlers: 5 methods, all pure domain logic
- Clear separation: register → serve → handle
- Bidirectional (import + export)
- Protocol details hidden in host

### Side-by-Side Comparison

| Aspect | Current (Protocol) | Proposed (Runtime+Handlers) |
|--------|-------------------|----------------------------|
| **Direction** | Export-only | Import + Export (bidirectional) |
| **Pattern** | Component exports protocol | Component uses runtime + provides handlers |
| **WASI Precedent** | None (unique) | wasi-http (proxy pattern) |
| **Lines of Code** | ~150 for hello world | ~60 for hello world |
| **Protocol Handling** | Component implements | Host implements |
| **Transport** | Component concern | Host concern |
| **Middleware** | Component implements | Host implements |
| **Auth/Security** | Component implements | Host implements |
| **Macro-Friendly** | No (too much boilerplate) | Yes (register + handlers) |
| **Lifecycle** | Resource semantics (complex) | Register → serve (simple) |
| **Evolution** | Recompile for MCP updates | Host updates, components unchanged |

---

## Part 6: Implementation Strategy

### Phase 1: Redesign WIT Interfaces

**Goal**: Replace protocol-faithful design with runtime+handlers pattern

**Tasks**:
1. Create `runtime.wit` with registration and framework APIs
2. Create `handlers.wit` with simple callback interfaces
3. Update `world.wit` with `mcp-backend` world (import + export)
4. Keep `types.wit` and `content.wit` (shared types)
5. Archive old `server.wit` to `archive/server-protocol-faithful.wit`

**Validation**:
- Bazel validation with rules_wasm_component
- Both Preview2 and Preview3 versions

### Phase 2: Reference Implementations

**Goal**: Prove the architecture works with real components

**Implementations**:
1. **Rust**:
   - wit-bindgen generates bindings
   - Provide `wasi-mcp-rust` crate with macro helpers
   - Example: hello-world in ~60 lines

2. **Go**:
   - wit-bindgen-go generates bindings
   - Example: file-server providing filesystem tools

3. **C++**:
   - wit-bindgen-cpp generates bindings
   - Example: system-tools providing process management

### Phase 3: Host Runtime

**Goal**: Implement MCP runtime in popular WASM hosts

**Hosts**:
1. **Wasmtime**: Rust host with full WASI support
2. **wasm-edge**: Edge computing runtime
3. **Browser**: In-browser MCP components

**Runtime Provides**:
- MCP protocol implementation (initialize, list, call, subscribe)
- Transport layer (stdio, HTTP, WebSocket)
- Middleware (auth, security, monitoring)
- Component lifecycle management

### Phase 4: Macro Ergonomics

**Goal**: Make it as easy as production Rust MCP macros

**Rust Macros**:
```rust
#[wasi_mcp::server(name = "hello-world")]
struct HelloWorld;

#[wasi_mcp::tools]
impl HelloWorld {
    /// Say hello to someone
    async fn say_hello(name: Option<String>) -> Result<String> {
        Ok(format!("Hello, {}!", name.unwrap_or("World".to_string())))
    }
}
```

**Generated Code**:
- Calls `runtime::register_server()` with server info
- Generates tool definition with JSON schema from function signature
- Calls `runtime::register_tools()` with tool list
- Implements `handlers::Guest::call_tool()` with dispatch logic
- Calls `runtime::serve()` in main

### Phase 5: Documentation and Examples

**Goal**: Comprehensive documentation for component authors

**Content**:
1. **Tutorial**: "Building Your First MCP Component"
2. **Examples**: hello-world, file-server, api-gateway, database
3. **API Reference**: Generated from WIT with explanations
4. **Migration Guide**: From protocol-faithful to runtime+handlers
5. **Best Practices**: Error handling, streaming, subscriptions

---

## Part 7: Open Questions

### Q1: Should we keep the protocol-faithful design as well?

**Option A**: Replace entirely with runtime+handlers
- Pros: Clear direction, no confusion, focused effort
- Cons: Loses protocol-faithful design work

**Option B**: Keep both as Layer 1 (protocol) and Layer 2 (runtime)
- Pros: Flexibility for advanced use cases
- Cons: Complexity, maintenance burden, confusion about which to use

**Recommendation**: **Option A** - Replace entirely. The protocol-faithful design serves no clear use case for components.

### Q2: How to handle optional capabilities?

Some components provide only tools, others provide tools + resources + prompts.

**Option A**: All handlers required, return "not supported" errors
```rust
fn read_resource(uri: String) -> Result<ResourceContents, Error> {
    Err(Error::not_supported("Resources not provided"))
}
```

**Option B**: Optional handler interfaces
```wit
world mcp-backend-tools-only {
    import runtime;
    export tool-handlers;  // Only tools
}

world mcp-backend-full {
    import runtime;
    export tool-handlers;
    export resource-handlers;
    export prompt-handlers;
}
```

**Recommendation**: **Option A** - Simpler world definition, capabilities declared in `register_server()`, errors for unsupported operations.

### Q3: How to handle streaming for large resources?

**Option A**: Use wasi-io/streams for large content
```wit
read-resource: func(uri: string) -> result<resource-stream, error>;

resource resource-stream {
    read-chunk: func() -> result<option<content-block>, error>;
}
```

**Option B**: Just use content-block list, host buffers if needed
```wit
read-resource: func(uri: string) -> result<list<content-block>, error>;
```

**Recommendation**: **Option B** for MVP, add Option A in future if needed. Most resources are small (<1MB).

### Q4: How to handle authentication from component side?

Components may need to call external APIs (databases, cloud services) that require auth.

**Option A**: Component handles auth internally
- Pros: Simple, component is self-contained
- Cons: Secrets management in component code

**Option B**: Host provides capability to access secrets
```wit
import wasi:keyvalue/secrets;  // Host-provided secrets

fn call_tool() {
    let api_key = secrets::get("OPENAI_API_KEY")?;
    // Use api_key
}
```

**Recommendation**: **Option B** - Use existing WASI proposals (wasi-keyvalue) for secrets. Host manages secrets, component imports capability.

### Q5: Preview2 vs Preview3 for initial release?

**Option A**: Preview2 only (blocking)
- Pros: Works today with stable toolchains
- Cons: Blocking I/O less efficient

**Option B**: Preview3 only (async)
- Pros: Better performance, future-proof
- Cons: Requires async toolchain support (not stable yet)

**Option C**: Both (dual release)
- Pros: Immediate prototyping (P2) + future readiness (P3)
- Cons: Maintain two versions

**Recommendation**: **Option C** - Dual release. Preview2 for immediate adoption, Preview3 for when toolchains ready. Clear migration path.

---

## Conclusion

### What We Learned

1. **WASI system interfaces are capability providers imported by components**
   - wasi-io, wasi-filesystem, wasi-nn all follow this pattern
   - Host owns resources, components request access

2. **Application protocol interfaces use bidirectional pattern**
   - wasi-http: component imports capabilities + exports handlers
   - Clear separation: host handles protocol, component handles logic

3. **Export-only pattern is wrong for MCP**
   - MCP is not a hardware capability (like GPU or filesystem)
   - Components don't "provide MCP", they provide tools using MCP runtime
   - Protocol handling belongs in host, not component

4. **Our current design violates WASI patterns**
   - We have export-only (should be bidirectional)
   - We expose protocol operations (should expose registration + handlers)
   - We target protocol implementers (should target tool developers)

### Recommended Actions

1. **Redesign WIT interfaces** to follow runtime+handlers pattern
2. **Validate with stakeholders** (Anthropic, WASI community)
3. **Build reference implementations** in Rust/Go/C++
4. **Implement host runtime** in Wasmtime
5. **Create macro helpers** for ergonomic developer experience

### Success Criteria

A successful WASI MCP system interface will:
- ✅ Follow WASI patterns (bidirectional import+export)
- ✅ Match developer expectations (~60 line hello world)
- ✅ Enable macro ergonomics similar to production Rust MCP
- ✅ Separate concerns (host = protocol, component = logic)
- ✅ Be extensible (new MCP features don't break components)
- ✅ Be secure (host controls auth, middleware, rate limiting)
- ✅ Be performant (zero-copy where possible, async support)

---

## Appendix A: Full Proposed WIT (Summary)

```wit
// runtime.wit - Component imports this
interface runtime {
    register-server: func(info: server-info) -> result;
    register-tools: func(tools: list<tool-definition>) -> result;
    register-resources: func(resources: list<resource-definition>) -> result;
    register-prompts: func(prompts: list<prompt-definition>) -> result;
    serve: func() -> result;
    send-notification: func(notification) -> result;
    log: func(level, message, data) -> result;
    report-progress: func(token, progress, total) -> result;
}

// handlers.wit - Component exports this
interface handlers {
    call-tool: func(name: string, arguments: list<u8>) -> result<tool-result>;
    read-resource: func(uri: string) -> result<resource-contents>;
    get-prompt: func(name: string, arguments: option<list<u8>>) -> result<prompt-contents>;
    handle-subscribe: func(uri: string) -> result;
    handle-unsubscribe: func(uri: string) -> result;
    handle-complete: func(ref-type, ref-value) -> result<list<completion>>;
}

// world.wit
world mcp-backend {
    import runtime;
    export handlers;
    import wasi:io/streams;
    import wasi:clocks/wall-clock;
}
```

This architecture positions WASI MCP as a true **system interface** following established WASI patterns, rather than an application protocol interface.
