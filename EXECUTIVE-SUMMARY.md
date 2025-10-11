# WASI MCP: Executive Summary
## Critical Architectural Redesign Required

---

## The Problem

Our current WASI MCP design violates fundamental WASI system interface patterns:

### Current Design (WRONG)
```wit
world mcp-server-world {
    export server;  // ❌ Component exports full MCP protocol
}
```

**Critical Issues**:
- ❌ **Wrong direction**: Export-only (no capability imports)
- ❌ **Wrong abstraction**: Protocol operations instead of domain logic
- ❌ **Wrong pattern**: No WASI precedent for this approach
- ❌ **Wrong complexity**: 150+ lines for hello world

---

## The Solution

Follow the **wasi-http bidirectional pattern**: Component imports runtime + exports handlers

### Proposed Design (CORRECT)
```wit
world mcp-backend {
    import runtime;   // ✅ Component uses MCP framework
    export handlers;  // ✅ Component provides callbacks
}
```

**Benefits**:
- ✅ **Correct direction**: Bidirectional (import + export)
- ✅ **Correct abstraction**: Domain logic only (tools, resources)
- ✅ **Correct pattern**: Follows wasi-http precedent
- ✅ **Correct complexity**: ~60 lines for hello world

---

## What We Learned

### Analysis of Phase 3 WASI Proposals

| Proposal | Pattern | Component Role |
|----------|---------|----------------|
| wasi-io | Pure Import | Uses streams from host |
| wasi-filesystem | Pure Import | Requests file operations |
| wasi-nn | Pure Import | Requests ML inference |
| wasi-http | **Bidirectional** | **Exports handlers + imports capabilities** |
| wasi-mcp (current) | Pure Export | ❌ Exports protocol operations |
| wasi-mcp (proposed) | **Bidirectional** | ✅ **Exports handlers + imports runtime** |

### Key Insight

**WASI system interfaces are capability providers that components import.**

- wasi-io: Host provides streams
- wasi-filesystem: Host provides file access
- wasi-nn: Host provides ML inference
- wasi-http: Host provides HTTP capabilities + calls component handlers
- **wasi-mcp should**: Host provides MCP runtime + calls component handlers

---

## Architecture Comparison

### Current (Protocol-Faithful)

**Component implements**:
```rust
impl McpServer {
    fn initialize(...) -> future<result<...>>;
    fn list_tools(...) -> future<result<...>>;
    fn call_tool(...) -> future<result<...>>;
    fn list_resources(...) -> future<result<...>>;
    fn read_resource(...) -> future<result<...>>;
    fn list_prompts(...) -> future<result<...>>;
    fn get_prompt(...) -> future<result<...>>;
    fn subscribe(...) -> future<result<...>>;
    fn complete(...) -> future<result<...>>;
    fn elicit(...) -> future<result<...>>;
    fn set_level(...) -> future<result<...>>;
    // ... 15+ protocol methods
}
```

**Problems**:
- Component implements MCP protocol (pagination, cursors, initialization)
- Component handles transport concerns
- 150+ lines for hello world
- No macro ergonomics possible
- Protocol evolution breaks components

### Proposed (Runtime + Handlers)

**Component implements**:
```rust
fn main() {
    // Register what you provide
    runtime::register_server(ServerInfo { ... });
    runtime::register_tools(vec![
        ToolDefinition { name: "say_hello", ... }
    ]);

    // Start serving (host handles protocol)
    runtime::serve();
}

impl Handlers {
    fn call_tool(name: String, args: Vec<u8>) -> Result<ToolResult> {
        // Pure domain logic
        match name.as_str() {
            "say_hello" => { /* implementation */ }
        }
    }

    fn read_resource(uri: String) -> Result<ResourceContents> {
        // Pure domain logic
    }
}
```

**Benefits**:
- Component implements domain logic only
- Host handles protocol, transport, middleware
- ~60 lines for hello world
- Macro ergonomics possible: `#[wasi_mcp::tool] fn say_hello()`
- Protocol evolution doesn't break components

---

## Detailed Analysis Documents

I've created three comprehensive documents:

### 1. WASI-SYSTEM-INTERFACE-ANALYSIS.md (8,000 words)

**Comprehensive pattern analysis**:
- **Part 1**: WASI system interface taxonomy
  - Pattern 1: Pure Import (system capability provider)
  - Pattern 2: Bidirectional (application protocol)
  - Pattern 3: Pure Export (not found in Phase 3!)
- **Part 2**: System vs application interface characteristics
- **Part 3**: Why our current MCP design is wrong
- **Part 4**: The correct WASI MCP architecture
- **Part 5**: Comparison with current design
- **Part 6**: Implementation strategy
- **Part 7**: Open questions

### 2. ARCHITECTURE-PROPOSAL.md (6,000 words)

**Complete implementation specification**:
- Current vs proposed architecture comparison
- **Complete WIT specification**:
  - `runtime.wit` (~200 lines, fully documented)
  - `handlers.wit` (~150 lines, fully documented)
  - `world.wit` (complete worlds)
- **Example**: Hello world component (~60 lines)
- **Migration plan**: 9-week timeline
- **Open questions**: 5 key decisions needed
- **Success criteria**: 8 measurable outcomes

### 3. /tmp/wasi-mcp-critical-review.md (existing)

**Production implementation analysis**:
- Examines pulseengine/mcp Rust implementation
- Identifies mismatch between WIT and production patterns
- Documents macro-driven ergonomics
- Compares 25-line production hello world vs 150+ line WIT approach

---

## The "System Interface" Distinction

You asked: **"I want this to be a system interface, not just a WIT interface"**

### What Makes a WASI System Interface

Based on analyzing all Phase 3 WASI proposals, a **system interface**:

1. **Capability-Based Security**
   - Host owns actual resources (filesystem, GPU, network)
   - Component requests access through capabilities
   - Example: wasi-filesystem's `descriptor` is a borrowed capability

2. **Hardware/OS Abstraction**
   - Abstracts platform-specific resources
   - Example: wasi-nn abstracts TensorFlow/PyTorch/ONNX
   - Example: wasi-clocks abstracts monotonic vs wall-clock

3. **Performance-Critical**
   - Zero-copy where possible
   - Direct access to system resources
   - Example: wasi-io streams use buffer sharing

4. **Lifecycle Management**
   - Clear resource creation, usage, disposal
   - Host manages lifecycle
   - Component borrows capabilities temporarily

5. **Foundational**
   - Other interfaces build on top
   - Example: wasi-http depends on wasi-io

### MCP as a System Interface

**Current design**: NOT a system interface
- Exposes protocol operations, not capabilities
- Component exports, doesn't import
- No hardware abstraction
- No lifecycle management

**Proposed design**: IS a system interface
- ✅ **Capability-based**: Component imports `runtime` capability
- ✅ **Abstraction**: Host abstracts transport (stdio/HTTP/WebSocket)
- ✅ **Lifecycle**: Host manages MCP sessions, component handles tools
- ✅ **Foundational**: MCP runtime can be used by many backends

---

## The Import vs Export Question

You asked about **wasi-nn's direction** and whether MCP should be imported.

### wasi-nn Pattern (CORRECT for ML)
```wit
world ml {
    import inference;  // Component imports ML from host
    import graph;
    import tensor;
}
```

**Why?** Host owns GPU/TPU hardware, component requests inference.

### Our Current MCP Pattern (WRONG)
```wit
world mcp-server-world {
    export server;  // Component exports MCP protocol
}
```

**Why wrong?** Component doesn't own "MCP hardware". MCP is a framework/protocol.

### Correct MCP Pattern (wasi-http style)
```wit
world mcp-backend {
    import runtime;   // Component imports MCP framework
    export handlers;  // Component exports callbacks
}
```

**Why correct?**
- Host provides MCP runtime (protocol, transport, auth)
- Component provides domain logic (tools, resources)
- Bidirectional: component uses framework + provides handlers

---

## Macro Ergonomics Comparison

### Production Rust MCP (25 lines)
```rust
#[mcp_server(name = "Hello World")]
#[derive(Default, Clone)]
pub struct HelloWorld;

#[mcp_tools]
impl HelloWorld {
    pub async fn say_hello(&self, params: SayHelloParams) -> Result<String> {
        Ok(format!("Hello, {}!", params.name.unwrap_or("World".to_string())))
    }
}

HelloWorld::with_defaults().serve_stdio().await?.run().await?;
```

### Current WASI MCP Design (150+ lines)
```rust
// Must implement full McpBackend trait
// Must handle all protocol methods (initialize, list, pagination, cursors)
// Must create JSON schemas manually
// Must manage server resource lifecycle
// Must deal with future<> wrapping
// No macro shortcuts
```

### Proposed WASI MCP Design (~60 lines, reducible to ~25 with macros)
```rust
fn main() {
    runtime::register_server(ServerInfo { ... });
    runtime::register_tools(vec![ToolDefinition { ... }]);
    runtime::serve();
}

impl Handlers {
    fn call_tool(name: String, args: Vec<u8>) -> Result<ToolResult> {
        match name.as_str() {
            "say_hello" => { /* implementation */ }
        }
    }
}

// With macros (future):
#[wasi_mcp::server(name = "Hello World")]
struct HelloWorld;

#[wasi_mcp::tools]
impl HelloWorld {
    async fn say_hello(name: Option<String>) -> Result<String> {
        Ok(format!("Hello, {}!", name.unwrap_or("World".to_string())))
    }
}
```

---

## Recommendation

### Immediate Actions Required

1. **Validate with stakeholders** (Anthropic, WASI community)
   - Present both approaches (current vs proposed)
   - Get feedback on bidirectional pattern
   - Confirm this is the right abstraction

2. **Make redesign decision**
   - If stakeholders agree: proceed with redesign
   - If not: understand what pattern they want

3. **Implement redesign** (if approved)
   - Week 1: New WIT interfaces
   - Week 2-3: Reference implementations
   - Week 4-6: Host runtime
   - Week 7-8: Macro helpers
   - Week 9: Documentation

### Decision Point

**The fundamental question**:

Should WASI MCP be:
- **A) Protocol operations** (current design)
  - Component exports full MCP protocol
  - Use case: Building MCP protocol implementations
  - Users: MCP SDK developers

- **B) Backend interface** (proposed design)
  - Component imports runtime + exports handlers
  - Use case: Building MCP servers with tools/resources
  - Users: Application developers

**Recommendation**: **Option B** - Backend interface

**Reasoning**:
1. Matches production usage patterns
2. Follows WASI system interface principles
3. Enables macro ergonomics
4. Separates protocol from domain logic
5. More users will build backends than implement protocol

---

## Success Metrics

A successful WASI MCP system interface will achieve:

| Metric | Current | Proposed | Target |
|--------|---------|----------|--------|
| Lines for hello world | 150+ | 60 | < 100 |
| WASI pattern match | 0% | 100% | 100% |
| Macro support | No | Yes | Yes |
| Protocol in component | Yes | No | No |
| Host-side protocol | No | Yes | Yes |
| Matches production | No | Yes | Yes |
| Stakeholder approval | ? | ? | Yes |

---

## Timeline

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Stakeholder validation | 1 week | Approval to proceed |
| WIT redesign | 1 week | New runtime.wit, handlers.wit |
| Reference implementations | 2 weeks | Rust + Go examples |
| Host runtime | 3 weeks | Wasmtime host |
| Macro helpers | 2 weeks | Rust macros |
| Documentation | 1 week | Tutorial, API docs |
| **Total** | **10 weeks** | **Production-ready system interface** |

---

## Conclusion

Our current WASI MCP design is **technically correct but architecturally wrong**:
- ✅ Complete protocol coverage
- ✅ Valid WIT syntax
- ✅ Validates with tools
- ❌ **Wrong WASI pattern** (export-only instead of bidirectional)
- ❌ **Wrong abstraction** (protocol operations instead of domain logic)
- ❌ **Wrong complexity** (150+ lines instead of ~60)

The proposed redesign **transforms WASI MCP into a true system interface**:
- ✅ Follows wasi-http bidirectional pattern
- ✅ Component imports runtime (capability provider)
- ✅ Component exports handlers (callbacks)
- ✅ Host handles protocol, transport, middleware
- ✅ ~60 lines for hello world (reducible to ~25 with macros)
- ✅ Matches production usage patterns

**Next step**: Validate with stakeholders before proceeding with redesign.

---

## Documents for Review

1. **WASI-SYSTEM-INTERFACE-ANALYSIS.md** - Comprehensive pattern analysis
2. **ARCHITECTURE-PROPOSAL.md** - Complete implementation specification
3. **/tmp/wasi-mcp-critical-review.md** - Production implementation analysis

All three documents support the same conclusion: **bidirectional runtime+handlers pattern is correct for WASI MCP**.
