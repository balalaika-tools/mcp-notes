# Python (FastMCP)

> Build MCP servers and clients in Python using FastMCP 3.x — the Pythonic, batteries-included framework.

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB.svg?logo=python&logoColor=white)](https://www.python.org)
[![FastMCP](https://img.shields.io/badge/FastMCP-3.2+-009688.svg)](https://gofastmcp.com)
[![Pydantic](https://img.shields.io/badge/Pydantic-2+-E92063.svg?logo=pydantic&logoColor=white)](https://docs.pydantic.dev)

---

## Contents

| File | Topic | Description |
|------|-------|-------------|
| [01_getting_started.md](01_getting_started.md) | Setup | Install FastMCP, write a hello-world server, connect a client |
| [02_tools.md](02_tools.md) | Tools | Sync/async tools, validation, errors, structured output, versioning |
| [03_resources_prompts.md](03_resources_prompts.md) | Resources & Prompts | URI templates, content types, prompt templates |
| [04_context_lifespan.md](04_context_lifespan.md) | Context | Per-request `Context`, app-scoped lifespan, dependency injection |
| [05_sampling_elicitation.md](05_sampling_elicitation.md) | Server → Client | Server-initiated LLM calls and user input requests |
| [06_middleware.md](06_middleware.md) | Middleware | Pipeline for logging, auth, rate-limiting cross-cutting concerns |
| [07_clients_advanced.md](07_clients_advanced.md) | Advanced | Building clients, server composition, proxying, OpenAPI bridge |

---

## Reading Order

1. **Getting Started** — minimal server + client running locally
2. **Tools** — the most-used primitive, in depth
3. **Resources & Prompts** — the other two server-side primitives
4. **Context & Lifespan** — how to share resources (DBs, HTTP clients) across calls
5. **Sampling & Elicitation** — the inverse direction: server requesting from the client
6. **Middleware** — cross-cutting concerns
7. **Clients & Advanced** — composition, proxying, OpenAPI conversion

---

## Prerequisites

- Python 3.11+
- Familiarity with `async`/`await` and type hints
- Read [Fundamentals](../fundamentals/README.md) first if you don't know what tools/resources/prompts are
