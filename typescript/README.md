# TypeScript

> Build MCP servers and clients in TypeScript using the official `@modelcontextprotocol/sdk`.

[![TypeScript](https://img.shields.io/badge/TypeScript-5.4+-3178C6.svg?logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![Node.js](https://img.shields.io/badge/Node-20+-339933.svg?logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![SDK](https://img.shields.io/badge/SDK-1.29+-2088FF.svg)](https://www.npmjs.com/package/@modelcontextprotocol/sdk)
[![Zod](https://img.shields.io/badge/Zod-3+-3068B7.svg)](https://zod.dev)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_getting_started.md](01_getting_started.md) | Setup | Install the SDK, write a hello-world server and client |
| [02_tools_zod.md](02_tools_zod.md) | Tools | Tool definitions with Zod schemas, async, errors, structured output |
| [03_resources_prompts.md](03_resources_prompts.md) | Resources & Prompts | Static + templated resources, prompt templates |
| [04_context_clients.md](04_context_clients.md) | Context & Clients | The `extra` request context, building TS clients |
| [05_advanced.md](05_advanced.md) | Advanced | Streamable HTTP, OAuth, server composition, low-level Server API |

---

## Reading Order

1. **Getting Started** — the minimal server you can stand up
2. **Tools with Zod** — schema-first tool definitions
3. **Resources & Prompts** — the other server primitives
4. **Context & Clients** — how to read request metadata, then write a client
5. **Advanced** — production transport, auth, composition

---

## Prerequisites

- Node 20+ and TypeScript 5.4+
- Comfort with async/await, ESM imports, and Zod (or willing to learn)
- Read [Fundamentals](../fundamentals/README.md) first if you're new to MCP

> If you've already read the [Python](../python/README.md) section, the [Python vs TypeScript comparison](../reference/02_python_vs_typescript.md) shows the mapping side-by-side.
