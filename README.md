# Model Context Protocol (MCP) Notes

> A practical, opinionated guide to MCP — concepts first, then production-grade implementations in Python (FastMCP) and TypeScript.

[![MCP Spec](https://img.shields.io/badge/MCP_Spec-2025--11--25-6E3FE0.svg)](https://modelcontextprotocol.io/specification/2025-11-25)
[![FastMCP](https://img.shields.io/badge/FastMCP-3.2+-009688.svg?logo=python&logoColor=white)](https://gofastmcp.com)
[![TypeScript SDK](https://img.shields.io/badge/TS_SDK-1.29+-3178C6.svg?logo=typescript&logoColor=white)](https://www.npmjs.com/package/@modelcontextprotocol/sdk)
[![Python](https://img.shields.io/badge/Python-3.11+-3776AB.svg?logo=python&logoColor=white)](https://www.python.org)
[![Node.js](https://img.shields.io/badge/Node-20+-339933.svg?logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## Structure

```
mcp-notes/
│
│ ── FUNDAMENTALS ─────────────────────────────────────────
├── fundamentals/           Concepts, architecture, primitives, lifecycle, transports
│
│ ── PYTHON (FastMCP) ─────────────────────────────────────
├── python/                 Server + client implementations using FastMCP 3.x
│
│ ── TYPESCRIPT ───────────────────────────────────────────
├── typescript/             Equivalent walkthrough using @modelcontextprotocol/sdk
│
│ ── PRODUCTION ───────────────────────────────────────────
├── production/             Streamable HTTP, OAuth 2.1, scaling, OTel, security
│
│ ── REFERENCE ────────────────────────────────────────────
└── reference/              Ecosystem map, language comparison, spec timeline
```

---

## Contents

### Fundamentals — [full index](fundamentals/README.md)

[![Spec](https://img.shields.io/badge/MCP_Spec-2025--11--25-6E3FE0.svg)](https://modelcontextprotocol.io/specification/2025-11-25)

| Guide | Description |
|-------|-------------|
| [What Is MCP](fundamentals/01_what_is_mcp.md) | The "USB-C for AI" mental model and why MCP exists |
| [Architecture](fundamentals/02_architecture.md) | Hosts, clients, servers; JSON-RPC 2.0 message flow |
| [Primitives](fundamentals/03_primitives.md) | Tools, resources, prompts, sampling, elicitation, roots |
| [Lifecycle](fundamentals/04_lifecycle.md) | Initialize handshake, capability negotiation, sessions |
| [Transports](fundamentals/05_transports.md) | stdio vs Streamable HTTP (and the deprecated HTTP+SSE) |

### Python (FastMCP) — [full index](python/README.md)

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB.svg?logo=python&logoColor=white)](https://www.python.org)
[![FastMCP](https://img.shields.io/badge/FastMCP-3.2+-009688.svg)](https://gofastmcp.com)
[![Pydantic](https://img.shields.io/badge/Pydantic-2+-E92063.svg?logo=pydantic&logoColor=white)](https://docs.pydantic.dev)

| Guide | Description |
|-------|-------------|
| [Getting Started](python/01_getting_started.md) | Install FastMCP 3.x, hello-world server and client |
| [Tools](python/02_tools.md) | Sync/async tools, schema validation, errors, versioning |
| [Resources & Prompts](python/03_resources_prompts.md) | URI templates, content types, prompt definitions |
| [Context & Lifespan](python/04_context_lifespan.md) | Per-request `Context`, app-scoped lifespan, DI |
| [Sampling & Elicitation](python/05_sampling_elicitation.md) | Server-initiated LLM calls and user input requests |
| [Middleware](python/06_middleware.md) | Pipeline for logging, auth, rate-limiting cross-cutting concerns |
| [Clients & Advanced](python/07_clients_advanced.md) | Building clients, server composition, proxying, OpenAPI bridge |

### TypeScript — [full index](typescript/README.md)

[![TypeScript](https://img.shields.io/badge/TypeScript-5.4+-3178C6.svg?logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![Node.js](https://img.shields.io/badge/Node-20+-339933.svg?logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![SDK](https://img.shields.io/badge/SDK-1.29+-2088FF.svg)](https://www.npmjs.com/package/@modelcontextprotocol/sdk)

| Guide | Description |
|-------|-------------|
| [Getting Started](typescript/01_getting_started.md) | Install `@modelcontextprotocol/sdk`, hello-world server |
| [Tools with Zod](typescript/02_tools_zod.md) | Tool definitions with Zod schemas, async, errors |
| [Resources & Prompts](typescript/03_resources_prompts.md) | Static + templated resources and prompt templates |
| [Context & Clients](typescript/04_context_clients.md) | `extra` context object, building TS clients |
| [Advanced](typescript/05_advanced.md) | Streamable HTTP server, auth provider, server composition |

### Production — [full index](production/README.md)

[![OAuth](https://img.shields.io/badge/OAuth-2.1-EB5424.svg)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-1.30+-425CC7.svg)](https://opentelemetry.io)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED.svg?logo=docker&logoColor=white)](https://www.docker.com)

| Guide | Description |
|-------|-------------|
| [Streamable HTTP](production/01_streamable_http.md) | The current production transport, in depth |
| [Authentication](production/02_authentication.md) | OAuth 2.1 + PKCE, RFC 9728 Resource Server pattern |
| [Scaling](production/03_scaling.md) | Stateful vs stateless, sticky sessions, horizontal scale |
| [Observability](production/04_observability.md) | OpenTelemetry tracing, structured logs, Prometheus metrics |
| [Security](production/05_security.md) | Threat model, gateways, prompt injection, sandboxing |
| [Deployment](production/06_deployment.md) | Containers, the official registry, runtime concerns |

### Reference — [full index](reference/README.md)

| Guide | Description |
|-------|-------------|
| [Ecosystem](reference/01_ecosystem.md) | Registries, gateways, host applications, hosting platforms |
| [Python vs TypeScript](reference/02_python_vs_typescript.md) | Side-by-side feature comparison |
| [Spec Evolution](reference/03_spec_evolution.md) | Spec timeline 2024-11 → 2025-11-25 and what changed |

---

## Reading Order

> [!TIP]
> Not sure where to start? Pick the path that matches your goal.

### The Newcomer Path — "What is this thing?"

1. [What Is MCP](fundamentals/01_what_is_mcp.md) — the mental model
2. [Architecture](fundamentals/02_architecture.md) — how the pieces fit
3. [Primitives](fundamentals/03_primitives.md) — what you can actually build
4. [Python: Getting Started](python/01_getting_started.md) — type code, see it work

### The Builder Path — "I'm shipping a server next week"

1. [Architecture](fundamentals/02_architecture.md) and [Transports](fundamentals/05_transports.md) — the wire-level mental model
2. [Python: Tools](python/02_tools.md) **or** [TypeScript: Tools with Zod](typescript/02_tools_zod.md)
3. [Context & Lifespan](python/04_context_lifespan.md) — how to share resources across requests
4. [Streamable HTTP](production/01_streamable_http.md) and [Scaling](production/03_scaling.md)
5. [Authentication](production/02_authentication.md) and [Security](production/05_security.md)

### The Operator Path — "I'm running someone else's server"

1. [What Is MCP](fundamentals/01_what_is_mcp.md) — vocabulary
2. [Lifecycle](fundamentals/04_lifecycle.md) — what handshakes look like
3. [Streamable HTTP](production/01_streamable_http.md) — the transport you'll terminate
4. [Authentication](production/02_authentication.md) and [Security](production/05_security.md)
5. [Observability](production/04_observability.md) and [Deployment](production/06_deployment.md)
6. [Ecosystem](reference/01_ecosystem.md) — registries, gateways, available servers

### The Polyglot Path — "I know one language, I want both"

1. Pick your starting language ([Python](python/01_getting_started.md) or [TypeScript](typescript/01_getting_started.md))
2. Read your "Tools" and "Resources" files
3. Jump to [Python vs TypeScript](reference/02_python_vs_typescript.md) for the side-by-side
4. Read the equivalent files in the other language

---

## How to Use These Notes

- **Numbered files = reading order within a directory.** `01_*` first.
- **Cross-references are relative paths.** Click them; they resolve.
- **Code is production-grade.** Real imports, real error handling. Copy-paste, then adapt.
- **The spec is the source of truth.** When this guide and the spec disagree, the spec wins. Open an issue or PR.
