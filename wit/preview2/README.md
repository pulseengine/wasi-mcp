# WASI MCP Preview2 Interfaces

This directory contains **Preview2-compatible** WIT interfaces for immediate prototyping with stable WASI Preview2 toolchains.

## Why Preview2?

While the main WIT interfaces in `../` use **Preview3 async patterns** (the aspirational design), Preview2 allows you to:

- ✅ **Build working prototypes TODAY** with stable Rust/Go/C++/JS toolchains
- ✅ **Validate the interface design** with real implementations
- ✅ **Demonstrate to stakeholders** with actual working code
- ✅ **Test the ergonomics** before Preview3 is ready

## Key Differences from Preview3

| Feature | Preview3 (../) | Preview2 (here) |
|---------|---------------|-----------------|
| **Async support** | `future<result<T, error>>` | `result<T, error>` (blocking) |
| **WASI version** | 0.2.3 | 0.2.0 |
| **Complexity** | Full MCP 2025-06-18 | Simplified for prototyping |
| **Status** | Aspirational target | Available now |
| **Use case** | Final standardization | Immediate prototyping |

## Example: Blocking vs Async

**Preview3 (aspirational):**
```wit
initialize: func(params: initialize-params)
    -> future<result<initialize-result, error>>;
```

**Preview2 (available now):**
```wit
initialize: func(params: initialize-params)
    -> result<initialize-result, error>;
```

## Building with Preview2

```bash
# Build Preview2 version
bazel build //:mcp_preview2

# Use in your component
wit_library(
    name = "my_mcp_server",
    srcs = ["server.wit"],
    deps = ["@wasi_mcp//:mcp_preview2"],
)
```

## Component Worlds

### mcp-server-preview2
Implement an MCP server with synchronous operations:
```wit
world mcp-server-preview2 {
    import wasi:clocks/wall-clock@0.2.0;
    export server;  // Synchronous MCP operations
}
```

### mcp-client-preview2
Build an MCP client with blocking calls:
```wit
world mcp-client-preview2 {
    import wasi:clocks/wall-clock@0.2.0;
    import server;  // Make blocking MCP calls
}
```

## What's Included

- **types.wit** - Core MCP types (simplified)
- **server.wit** - Synchronous server operations
- **world.wit** - Preview2 component worlds

**Not included** (use Preview3 version for full spec):
- Streaming interfaces
- Complete notification system
- Full capabilities model
- Content block variants

## Migration Path

1. **Now**: Prototype with Preview2 (this directory)
2. **Later**: When Preview3 is stable, migrate to `../` interfaces
3. **Eventually**: Preview3 becomes the standard

The core API shapes are the same, just add `future<>` wrappers when migrating.

## Examples

See the examples directory for:
- Rust MCP server (Preview2)
- Go MCP client (Preview2)
- C++ MCP proxy (Preview2)

## Status

✅ **Ready for prototyping** - All interfaces validated with Bazel
