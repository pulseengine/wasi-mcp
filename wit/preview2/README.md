# WASI MCP Preview2 Interfaces

**Status**: Blocking operations for immediate use with stable toolchains

Preview2-compatible WIT interfaces for immediate prototyping with current stable WebAssembly toolchains. These interfaces use blocking (synchronous) operations instead of async futures.

## Architecture

Preview2 follows the same **bidirectional pattern** as Preview3:

```
┌─────────────────────────────────────────┐
│            Host Runtime                  │
│  - MCP Protocol Implementation           │
│  - Transport Layer (stdio/HTTP/WS)      │
│  - Middleware (auth, logging, etc.)     │
└─────────────────────────────────────────┘
                   ↕
         imports runtime (blocking)
         exports handlers (blocking)
                   ↕
┌─────────────────────────────────────────┐
│      Component (Your Code)               │
│  - Registration: register-server(), etc. │
│  - Handlers: call-tool(), read-resource()│
└─────────────────────────────────────────┘
```

**Key Principle**: Component imports runtime capabilities and exports handlers, just like Preview3, but all operations are blocking.

## Key Differences from Preview3

| Feature | Preview3 | Preview2 |
|---------|----------|----------|
| **Async support** | `future<result<T, error>>` | `result<T, error>` (blocking) |
| **WASI version** | 0.2.3 | 0.2.0 |
| **I/O support** | wasi:io/streams, wasi:io/poll | Not needed (blocking) |
| **Toolchain** | Requires async support | Stable toolchains |
| **Status** | Aspirational (future) | Available now |
| **Architecture** | Bidirectional (import+export) | Bidirectional (import+export) ✅ |

**Important**: Both Preview2 and Preview3 use the same bidirectional architecture. The only difference is async vs blocking operations.

## Worlds

### mcp-backend-preview2
For components that provide MCP tools/resources/prompts:
```wit
world mcp-backend-preview2 {
    import runtime;   // Register and serve
    export handlers;  // Execute tools, read resources
    import wasi:clocks/wall-clock@0.2.0;
}
```

### mcp-client-preview2
For components that consume MCP servers:
```wit
world mcp-client-preview2 {
    import client;    // Connect and make requests
    import wasi:clocks/wall-clock@0.2.0;
}
```

### mcp-proxy-preview2
For components that aggregate/transform MCP servers:
```wit
world mcp-proxy-preview2 {
    import runtime;   // Serve downstream
    export handlers;  // Handle downstream requests
    import client;    // Call upstream servers
    import wasi:clocks/wall-clock@0.2.0;
}
```

## Building

To validate Preview2 interfaces with Bazel:

```bash
bazel build //:mcp_preview2
```

To generate bindings:

```bash
# Rust
wit-bindgen rust wit/preview2/ --out-dir src/bindings/

# Go
wit-bindgen-go wit/preview2/ --out-dir bindings/
```

## Status

✅ **Ready for prototyping** - Architecture redesigned, validated with Bazel
