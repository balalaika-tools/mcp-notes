# Python (FastMCP) vs TypeScript SDK: Side-by-Side

> **Who this is for**: An engineer who already knows one of the two MCP SDKs and is
> evaluating — or actively porting to — the other. Assumes you've read
> [01_ecosystem.md](01_ecosystem.md) and have at least skimmed
> [../python/01_getting_started.md](../python/01_getting_started.md) or
> [../typescript/01_getting_started.md](../typescript/01_getting_started.md).

This is a concept-by-concept mapping, not an introduction. Every section shows the same
job done in both languages so you can translate idioms directly. Where the two diverge
meaningfully (lifespan, middleware, transports), the divergence is called out as the
headline rather than buried in prose.

---

## 1. Versions and Ecosystems

The two SDKs target the same protocol but live in very different ecosystems.

| Aspect           | Python (FastMCP)                        | TypeScript SDK                          |
|------------------|------------------------------------------|------------------------------------------|
| Package          | `fastmcp` (PyPI)                         | `@modelcontextprotocol/sdk` (npm)        |
| Latest (Apr 2026)| 3.2.4                                    | 1.29.0 (v2 stable Q1 2026)               |
| Maintainer       | PrefectHQ                                | Anthropic / community                    |
| Schema lib       | Pydantic v2                              | Zod 3                                    |
| Min runtime      | Python 3.11                              | Node 20                                  |

A few things worth understanding before you look at code:

- **Python's FastMCP is a third-party wrapper** over the `mcp` reference package.
  PrefectHQ owns it; Anthropic does not. It moves faster and is more opinionated.
- **The TypeScript SDK is the reference implementation.** Anthropic ships it in lock-step
  with spec changes — usually a release lands within a week of the spec PR.
- **Pydantic vs Zod** is the single biggest day-to-day difference. Pydantic models become
  JSON Schema for free; Zod requires you to spell out shapes. We'll see this everywhere.

> **Key insight**: Python optimizes for ergonomics and convention. TypeScript optimizes
> for spec fidelity and explicit control. Neither is "better" — they reward different
> teams.

---

## 2. Hello-World Server

Same tool — `add(a, b)` — registered and started on stdio.

**Python**:

```python
from fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

if __name__ == "__main__":
    mcp.run()
```

**TypeScript**:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "demo", version: "1.0.0" });

server.registerTool(
  "add",
  {
    title: "Add",
    description: "Add two numbers.",
    inputSchema: { a: z.number(), b: z.number() },
  },
  async ({ a, b }) => ({ content: [{ type: "text", text: String(a + b) }] }),
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

The two snippets do the same thing, but the TypeScript version is roughly twice as long.
That ratio is representative of the rest of this guide — Python convention does work that
TypeScript asks you to do explicitly.

**Diff at a glance**:

- Python derives the schema from type hints. TypeScript builds it from a Zod shape.
- Python uses the docstring as the tool description. TypeScript wants an explicit
  `description` field.
- Python auto-wraps the return value in MCP text content. TypeScript expects you to
  return the full `{ content: [...] }` envelope.
- Python's `mcp.run()` picks a transport based on env or args. TypeScript wants you to
  instantiate a transport object and call `server.connect(transport)` yourself.

> **Rule**: When porting Python → TypeScript, plan for ~2x the code. Most of the bulk is
> envelope construction and Zod boilerplate, not real logic.

---

## 3. Tools

This is where you'll spend most of your time. Concept mapping table:

| Concept                   | Python                                        | TypeScript                                                     |
|---------------------------|-----------------------------------------------|----------------------------------------------------------------|
| Define                    | `@mcp.tool`                                   | `server.registerTool(name, def, handler)`                      |
| Async                     | `async def` (or sync, threadpool-dispatched)  | `async (input, extra) => ...`                                  |
| Validation                | Pydantic models, `Field(...)`                 | Zod shapes, `.refine`, `.min`, `.max`                          |
| Structured output         | Return Pydantic model with declared return type | `outputSchema` + `structuredContent` in result               |
| Error (model-correctable) | `raise ToolError("msg")`                      | `return { isError: true, content: [...] }`                     |
| Annotations               | `@mcp.tool(annotations={...})`                | `annotations: { ... }` in def                                  |
| Versioning                | `@mcp.tool(version="2.0")`                    | Manual: register two tools with different names                |
| Icons (2025-11-25)        | `@mcp.tool(icon="...")`                       | `icons: [...]` in def                                          |

A few pairs are worth seeing in code.

**Validation with constraints** — Python uses Pydantic `Field`, TypeScript uses Zod
chains:

```python
from pydantic import Field
from typing import Annotated

@mcp.tool
def search(
    query: Annotated[str, Field(min_length=1, max_length=200)],
    limit: Annotated[int, Field(ge=1, le=100)] = 10,
) -> list[dict]:
    """Search the index."""
    return run_search(query, limit)
```

```typescript
server.registerTool(
  "search",
  {
    title: "Search",
    description: "Search the index.",
    inputSchema: {
      query: z.string().min(1).max(200),
      limit: z.number().int().min(1).max(100).default(10),
    },
  },
  async ({ query, limit }) => ({
    content: [{ type: "text", text: JSON.stringify(await runSearch(query, limit)) }],
  }),
);
```

**Errors the model should retry** — Python raises a typed exception; TypeScript returns
an error result:

```python
from fastmcp.exceptions import ToolError

@mcp.tool
def fetch_user(user_id: str) -> dict:
    user = db.get_user(user_id)
    if not user:
        raise ToolError(f"User {user_id} not found. Try listing users first.")
    return user.model_dump()
```

```typescript
server.registerTool(
  "fetch_user",
  { title: "Fetch user", inputSchema: { user_id: z.string() } },
  async ({ user_id }) => {
    const user = await db.getUser(user_id);
    if (!user) {
      return {
        isError: true,
        content: [{ type: "text", text: `User ${user_id} not found. Try listing users first.` }],
      };
    }
    return { content: [{ type: "text", text: JSON.stringify(user) }] };
  },
);
```

> **Key insight**: A thrown error in TypeScript becomes a transport-level failure the
> client cannot recover from. Always return `{ isError: true }` for problems the model
> should see and retry.

---

## 4. Resources

Resources are the spec's read-only data side. Both SDKs support URI templates from RFC
6570.

**Python** — function parameters become template variables automatically:

```python
@mcp.resource("github://repos/{owner}/{repo}/issues/{number}")
async def issue(owner: str, repo: str, number: int) -> dict:
    return await fetch_issue(owner, repo, number)
```

**TypeScript** — `ResourceTemplate` is a separate type, and the handler returns the full
`contents` envelope:

```typescript
import { ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";

server.registerResource(
  "issue",
  new ResourceTemplate("github://repos/{owner}/{repo}/issues/{number}", { list: undefined }),
  { title: "Issue", mimeType: "application/json" },
  async (uri, { owner, repo, number }) => ({
    contents: [
      {
        uri: uri.href,
        mimeType: "application/json",
        text: JSON.stringify(await fetchIssue(owner, repo, number as string)),
      },
    ],
  }),
);
```

Things to notice:

- Python infers URI template variables from the function signature. TypeScript wants
  `ResourceTemplate` constructed explicitly so the SDK knows whether `list` is supported.
- Python returns the resource value; FastMCP wraps it in a content envelope. TypeScript
  builds the `contents` array yourself, including `uri` and `mimeType`.
- Path parameters in TypeScript come in as `string` (or `string[]` if the template
  expands to a list). Cast or coerce as needed — there's no per-parameter type schema.

⚠️ A common porting bug: forgetting `{ list: undefined }` in TypeScript. If you set it
to a function, the SDK treats the template as enumerable and clients may call
`resources/list` against it.

---

## 5. Prompts

Prompts are reusable message templates the user invokes (typically as a slash command).

**Python**:

```python
from fastmcp import Message

@mcp.prompt
def review_pr(diff: str, focus: str = "") -> list[Message]:
    return [Message(role="user", content=f"Review:\n{diff}\nFocus: {focus or 'general'}")]
```

**TypeScript**:

```typescript
server.registerPrompt(
  "review_pr",
  {
    title: "Review PR",
    description: "Generate a thorough code review",
    argsSchema: { diff: z.string(), focus: z.string().optional() },
  },
  ({ diff, focus }) => ({
    messages: [
      {
        role: "user",
        content: { type: "text", text: `Review:\n${diff}\nFocus: ${focus ?? "general"}` },
      },
    ],
  }),
);
```

Same shape as tools and resources: Python infers, TypeScript declares. The TypeScript
content object is also one level deeper — `{ type: "text", text: ... }` rather than the
Python-side bare string.

---

## 6. Per-Request Context

Every server-side handler can interact with the calling client mid-request: log, report
progress, sample, elicit input, read another resource, observe cancellation, inspect
auth. Both SDKs support all of it; the API shape differs sharply.

| Capability             | Python (`Context`)                                      | TypeScript (`extra`)                                                                  |
|------------------------|----------------------------------------------------------|---------------------------------------------------------------------------------------|
| Logging                | `await ctx.info("...")`                                  | `extra.sendNotification({ method: "notifications/message", params: {...} })`          |
| Progress               | `await ctx.report_progress(p, total, msg)`               | `extra.sendNotification({ method: "notifications/progress", params: {...} })`         |
| Sampling               | `await ctx.sample(messages, ...)`                        | `await extra.sendRequest({ method: "sampling/createMessage", params }, schema)`       |
| Elicitation            | `await ctx.elicit(message, schema)`                      | `await extra.sendRequest({ method: "elicitation/create", params }, schema)`           |
| Read another resource  | `await ctx.read_resource(uri)`                           | `await extra.sendRequest({ method: "resources/read", params: { uri } }, schema)`      |
| Cancellation           | `asyncio.CancelledError` raised in the handler           | `extra.signal: AbortSignal`                                                           |
| Auth info              | `ctx.auth_info`                                          | `extra.authInfo`                                                                      |

**Python is more ergonomic.** High-level methods, declared schemas, automatic envelope
parsing. **TypeScript is closer to the wire.** You call `sendRequest` with the right
JSON-RPC method name and a Zod schema for the response — the SDK validates the response
against your schema and gives you back a typed value. Both produce the same JSON-RPC
traffic.

A worked example — a long-running tool that reports progress and logs:

```python
@mcp.tool
async def process_dataset(path: str, ctx: Context) -> dict:
    rows = load(path)
    await ctx.info(f"Loaded {len(rows)} rows")
    for i, row in enumerate(rows):
        await process(row)
        await ctx.report_progress(i + 1, len(rows), f"row {i + 1}/{len(rows)}")
    return {"processed": len(rows)}
```

```typescript
server.registerTool(
  "process_dataset",
  { title: "Process dataset", inputSchema: { path: z.string() } },
  async ({ path }, extra) => {
    const rows = await load(path);
    await extra.sendNotification({
      method: "notifications/message",
      params: { level: "info", data: `Loaded ${rows.length} rows` },
    });
    for (let i = 0; i < rows.length; i++) {
      if (extra.signal.aborted) throw new Error("cancelled");
      await process(rows[i]);
      await extra.sendNotification({
        method: "notifications/progress",
        params: { progress: i + 1, total: rows.length, message: `row ${i + 1}/${rows.length}` },
      });
    }
    return { content: [{ type: "text", text: JSON.stringify({ processed: rows.length }) }] };
  },
);
```

Notice the cancellation difference: Python uses `asyncio.CancelledError` thrown into the
coroutine when the client cancels; TypeScript exposes an `AbortSignal` you check
explicitly. Both are correct — they reflect the runtime's native cancellation model.

> **Principle**: When a Python helper says `ctx.foo(...)`, the TypeScript equivalent is
> almost always `extra.sendRequest` or `extra.sendNotification` with the matching
> JSON-RPC method name. Once you've internalized the spec method names, the TypeScript
> SDK becomes much less verbose.

---

## 7. App-Scoped Resources

This is one of the larger philosophical gaps between the two SDKs.

**Python (FastMCP)** — first-class lifespan, just like Starlette / FastAPI:

```python
from contextlib import asynccontextmanager
from fastmcp import FastMCP

@asynccontextmanager
async def lifespan(app):
    db = await create_pool(dsn=os.environ["DATABASE_URL"])
    try:
        yield AppState(db=db)
    finally:
        await db.close()

mcp = FastMCP("app", lifespan=lifespan)

@mcp.tool
async def get_user(user_id: str, ctx: Context) -> dict:
    db = ctx.lifespan_state.db
    return await db.fetch_one("SELECT * FROM users WHERE id = $1", user_id)
```

**TypeScript** — no first-class lifespan; idiomatic Node uses module-level singletons
plus manual signal handlers:

```typescript
import { Pool } from "pg";

const db = new Pool({ connectionString: process.env.DATABASE_URL });

process.on("SIGTERM", async () => {
  await db.end();
  process.exit(0);
});

server.registerTool(
  "get_user",
  { title: "Get user", inputSchema: { user_id: z.string() } },
  async ({ user_id }) => {
    const row = await db.query("SELECT * FROM users WHERE id = $1", [user_id]);
    return { content: [{ type: "text", text: JSON.stringify(row.rows[0]) }] };
  },
);
```

The TS approach works but doesn't help you with:

- testability (the singleton lives at module scope)
- multi-tenant configuration (you'd build dependency injection yourself)
- ordered shutdown across multiple resources

If you want lifespan-style behavior in TS, the conventional answer is to drop down to the
low-level `Server` class and own the `connect` / `close` lifecycle yourself.

---

## 8. Middleware

Another significant divergence.

- **Python**: first-class `Middleware` base class with hooks like `on_call_tool`,
  `on_read_resource`, `on_get_prompt`. Registered with `mcp.add_middleware(...)`. Stack
  composes cleanly. See [../python/06_middleware.md](../python/06_middleware.md).
- **TypeScript**: nothing built-in for **protocol-level** middleware. You have three
  options, none of which are quite as clean:
  1. Wrap individual handlers (write a `withLogging(handler)` helper).
  2. Use Express middleware for HTTP-level concerns (auth headers, rate limiting,
     request IDs). Doesn't help with stdio.
  3. Drop to the low-level `Server` class and intercept `setRequestHandler` calls
     yourself.

If your design depends on cross-cutting concerns — audit logs, tenant scoping,
request-scoped tracing — Python is materially easier. TypeScript will get you there but
you'll write the framework yourself.

---

## 9. Streamable HTTP Serving

**Python** — one line:

```python
mcp.run(transport="http", host="0.0.0.0", port=8080, stateless_http=True, json_response=True)
```

FastMCP picks Starlette internally, wires up `/mcp`, manages sessions, handles SSE
upgrades, and respects `stateless_http` for serverless deployments.

**TypeScript** — full Express integration with a manually-managed session map:

```typescript
import express from "express";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { randomUUID } from "node:crypto";

const app = express();
app.use(express.json());

const transports = new Map<string, StreamableHTTPServerTransport>();

app.post("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  let transport = sessionId ? transports.get(sessionId) : undefined;

  if (!transport) {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (id) => transports.set(id, transport!),
    });
    await server.connect(transport);
  }

  await transport.handleRequest(req, res, req.body);
});

app.listen(8080);
```

See [../typescript/05_advanced.md](../typescript/05_advanced.md) for the full pattern
including DELETE for session teardown and the `Mcp-Session-Id` response header. More
boilerplate, more control — exactly the trade you'd expect from a framework-agnostic SDK.

---

## 10. Auth

Both SDKs implement the OAuth 2.1 + RFC 9728 protected-resource pattern from the
2025-11-25 spec, with very different glue.

**Python**:

```python
from fastmcp.auth import OAuthResourceServerAuth

mcp = FastMCP(
    "secure",
    auth=OAuthResourceServerAuth(
        issuer="https://auth.example.com",
        audience="https://mcp.example.com",
        required_scopes=["mcp:read"],
    ),
)

@mcp.tool(scopes=["mcp:write"])
def update_record(id: str, data: dict) -> dict:
    return db.update(id, data)
```

One object, declarative scopes per tool, RFC 9728 metadata served at
`/.well-known/oauth-protected-resource` automatically.

**TypeScript**:

```typescript
import { mcpAuthRouter, requireBearerAuth } from "@modelcontextprotocol/sdk/server/auth/router.js";

app.use(mcpAuthRouter({ provider: myProvider }));

app.post(
  "/mcp",
  requireBearerAuth({ verifier: tokenVerifier, requiredScopes: ["mcp:read"] }),
  async (req, res) => {
    /* ... */
  },
);

server.registerTool(
  "update_record",
  { title: "Update", inputSchema: { id: z.string(), data: z.object({}).passthrough() } },
  async ({ id, data }, extra) => {
    if (!extra.authInfo?.scopes?.includes("mcp:write")) {
      return { isError: true, content: [{ type: "text", text: "missing scope mcp:write" }] };
    }
    return { content: [{ type: "text", text: JSON.stringify(await db.update(id, data)) }] };
  },
);
```

More glue (Express middleware, manual scope check), same outcome on the wire.

---

## 11. Testing

Both SDKs ship in-memory transports so you don't have to spin up a process for unit
tests.

**Python** — `Client` accepts a server instance directly:

```python
import pytest
from fastmcp import Client

@pytest.mark.asyncio
async def test_add():
    async with Client(mcp) as client:
        result = await client.call_tool("add", {"a": 2, "b": 3})
        assert result.data == 5
```

**TypeScript** — paired in-memory transports:

```typescript
import { InMemoryTransport } from "@modelcontextprotocol/sdk/inMemory.js";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";

const [clientT, serverT] = InMemoryTransport.createLinkedPair();
const client = new Client({ name: "test", version: "1.0.0" });
await Promise.all([client.connect(clientT), server.connect(serverT)]);

const result = await client.callTool({ name: "add", arguments: { a: 2, b: 3 } });
expect(result.content[0]).toEqual({ type: "text", text: "5" });
```

Same idea, slightly more wiring. Both run in the same process — no subprocess, no
network, no flake.

---

## 12. Which to Pick

A blunt decision matrix. None of these are deal-breakers; they're tilts.

| If you...                                                | Pick                              |
|----------------------------------------------------------|-----------------------------------|
| Need maximum developer ergonomics                        | **Python (FastMCP)**              |
| Already run a Python data/ML stack                       | **Python**                        |
| Need first-class lifespan / middleware / composition     | **Python**                        |
| Need an OpenAPI / FastAPI bridge                         | **Python**                        |
| Already run a Node / TS web stack                        | **TypeScript**                    |
| Need browser execution (Cloudflare Workers, Deno Deploy) | **TypeScript**                    |
| Need the most control over the wire                      | **TypeScript** (closer to spec)   |
| Want lock-step with the spec authors                     | **TypeScript** (Anthropic-maintained) |

Two pragmatic notes:

- **Don't pick the language your team doesn't use.** A "better" SDK in a foreign
  ecosystem will lose to a slightly clunkier one your team can ship in. Both SDKs are
  good enough that the surrounding ecosystem (auth library, observability stack,
  deployment story) matters more than the SDK itself.
- **Both can interop with the same clients.** Claude Desktop, Cursor, the inspector — all
  speak the protocol, not the SDK. You can run a Python server and a TypeScript server
  side-by-side under the same client config.

---

## 13. Idiom Heads-Up

Five things that bite engineers porting in either direction:

- **Schema source**. Python infers schemas from type hints; TypeScript requires Zod for
  every input. Don't try to "share schemas" across the two — write each natively.
- **Async everywhere in TS**. Python tools can be sync (FastMCP threadpools them).
  TypeScript handlers should always be `async` even if the body is synchronous —
  middleware and transport assume a promise return.
- **Return shape**. Python returns just the data; FastMCP wraps it. TypeScript returns
  the full protocol envelope (`{ content: [...] }`). Forgetting this is the #1 porting
  bug.
- **Registration style**. Python decorators auto-wire on import. TypeScript is explicit:
  every tool, resource, and prompt is its own `register*` call. Some teams put each
  registration in its own file and import them from a central `setup.ts`.
- **Errors vs error results**. In Python, `raise ToolError(...)` is correct. In
  TypeScript, `throw` becomes a transport failure — for model-correctable errors you
  must `return { isError: true, content: [...] }`.

> **Key insight**: Most porting work is mechanical — the protocol is the same, the
> capabilities map cleanly. The real friction is at the seams: lifespan, middleware,
> HTTP serving, auth glue. Plan extra time for those four when porting in either
> direction.

---

**Next**: [03_spec_evolution.md — Spec Evolution Timeline](03_spec_evolution.md)
