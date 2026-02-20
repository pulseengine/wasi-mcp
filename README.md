<div align="center">

# wasi-mcp

<sup>Proposed WASI API for Model Context Protocol</sup>

&nbsp;

![WIT](https://img.shields.io/badge/WIT-654FF0?style=flat-square&labelColor=1a1b27)
![WebAssembly](https://img.shields.io/badge/WebAssembly-654FF0?style=flat-square&logo=webassembly&logoColor=white&labelColor=1a1b27)
![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-blue?style=flat-square&labelColor=1a1b27)

</div>

&nbsp;

A proposed [WebAssembly System Interface](https://github.com/WebAssembly/WASI) API for the Model Context Protocol, providing typed operations for MCP 2025-06-18 with WASI Preview3 async support.

### Current Phase

**Phase 0 — Pre-Proposal** (targeting Phase 1)

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
- [References](#references)

## Introduction

The Model Context Protocol (MCP) is an open standard that enables AI applications to connect to external data sources, tools, and workflows. This WASI proposal enables WebAssembly components to implement MCP servers and clients with full protocol support.

**Why WASI for MCP?**
- **Portability**: Run MCP servers across any WASM runtime
- **Security**: Leverage WebAssembly's capability-based security model
- **Performance**: Native async operations with WASI Preview3
- **Composability**: Component Model enables MCP server aggregation
- **Type Safety**: Strongly-typed WIT interfaces prevent protocol errors

**Protocol Coverage**: This proposal implements the complete MCP 2025-06-18 specification including protocol initialization, resources with templates and subscriptions, tools with annotations, prompts with arguments, pagination, progress tokens, notifications, logging, and streaming support.

## Two Versions: Preview2 + Preview3

| Version | Status | Use Case | Location |
|---------|--------|----------|----------|
| **Preview3** | Aspirational | Final standardization with async | `wit/*.wit` |
| **Preview2** | Available now | Immediate prototyping (blocking ops) | `wit/preview2/*.wit` |

## Goals

1. Complete MCP Protocol Support in WIT
2. Protocol-Faithful Design with direct mapping of MCP methods
3. WASI Preview3 Async with `future<result<T, error>>`
4. Follow WASI Patterns from successful Phase 3 proposals
5. Clear Path to Phase 1

## Non-goals

- Transport Implementation (handled by hosts)
- JSON-RPC Layer (implementation detail)
- AI Model Integration (out of scope)
- Backward Compatibility (only targeting latest MCP spec)

## API Overview

**Bidirectional Interface Pattern** — Components import runtime capabilities and export handlers:

```wit
// Component IMPORTS runtime to register and serve
interface runtime {
    register-server: func(info: server-info) -> future<result<_, error>>;
    register-tools: func(tools: list<tool-definition>) -> future<result<_, error>>;
    serve: func() -> future<result<_, error>>;
}

// Component EXPORTS handlers to execute operations
interface handlers {
    call-tool: func(name: string, arguments: list<u8>) -> future<result<tool-result, error>>;
    read-resource: func(uri: string) -> future<result<resource-contents, error>>;
}
```

**Interface Structure:**
```
wit/
├── types.wit          # Core MCP types
├── capabilities.wit   # Server/Client capabilities
├── content.wit        # Content blocks
├── runtime.wit        # Runtime API (component imports)
├── handlers.wit       # Handler interface (component exports)
├── client.wit         # Client operations
├── world.wit          # Component worlds
└── preview2/          # Preview2 for immediate prototyping
```

## Examples

See the full [Examples section](https://github.com/pulseengine/wasi-mcp#examples) for MCP backend, client, and streaming usage patterns in Rust.

## Implementation Status

### Completed

- Core types with all MCP protocol types
- Complete capabilities system
- Content type system with variants
- Typed server and client operations
- Notification system and streaming interface
- World definitions with deps.toml
- Preview2 interfaces for immediate prototyping

### Next Steps for Phase 1

1. Create reference implementations (Rust, Go, C++)
2. Gather stakeholder feedback (Anthropic, WASI Subgroup)
3. Submit Phase 1 proposal

## References

- [MCP 2025-06-18 Specification](https://github.com/modelcontextprotocol/specification)
- [wasi-http](https://github.com/WebAssembly/wasi-http) — Protocol-faithful HTTP mapping
- [Component Model](https://github.com/WebAssembly/component-model)
- [WASI Preview 3](https://github.com/WebAssembly/WASI)

## License

Apache-2.0

---

<div align="center">

<sub>Part of <a href="https://github.com/pulseengine">PulseEngine</a> &mdash; formally verified WebAssembly toolchain for safety-critical systems</sub>

</div>
