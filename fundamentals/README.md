# Fundamentals

> Concepts, architecture, and protocol mechanics — read this section before touching code.

[![MCP Spec](https://img.shields.io/badge/MCP_Spec-2025--11--25-6E3FE0.svg)](https://modelcontextprotocol.io/specification/2025-11-25)
[![JSON-RPC](https://img.shields.io/badge/JSON--RPC-2.0-F7DF1E.svg)](https://www.jsonrpc.org/specification)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_what_is_mcp.md](01_what_is_mcp.md) | Concept | The "USB-C for AI" model and why it exists |
| [02_architecture.md](02_architecture.md) | Architecture | Hosts, clients, servers; JSON-RPC message flow |
| [03_primitives.md](03_primitives.md) | Primitives | Tools, resources, prompts, sampling, elicitation, roots |
| [04_lifecycle.md](04_lifecycle.md) | Protocol | Initialize handshake, capability negotiation, sessions |
| [05_transports.md](05_transports.md) | Transports | stdio vs Streamable HTTP (and deprecated HTTP+SSE) |

---

## Reading Order

1. **What Is MCP** — set the mental model first
2. **Architecture** — names for the moving parts
3. **Primitives** — the actual building blocks
4. **Lifecycle** — what happens when a client connects
5. **Transports** — how bytes move on the wire

---

## Prerequisites

- Comfortable with JSON
- Familiar with HTTP and request/response basics
- A passing acquaintance with LLM tool use (function calling) helps but is not required
