# Clients, Composition, Proxying, and API Bridges

> **Who this is for**: Engineers who have built a FastMCP server and now need to (a) call
> one programmatically, (b) compose several servers into a single process, (c) bridge
> transports, or (d) wrap an existing REST API as MCP. Assumes you've worked through
> [06_middleware.md](06_middleware.md) and are comfortable with `async`/`await`.

A server is only half the system. The other half is the code that talks to it — test
harnesses, orchestration layers, desktop hosts, and bridges to legacy APIs. FastMCP's
`Client` is the single abstraction for all of those, and the same library provides
composition and bridging primitives so you don't have to hand-roll transport plumbing.

---

## 1. The `Client` Class

`Client` is transport-agnostic. You point it at something — a subprocess command, an
HTTP URL, or an in-memory `FastMCP` instance — and it figures out the right transport.
Every client is an async context manager; the `async with` block owns the connection
lifetime.

```python
import asyncio
from fastmcp import Client, FastMCP

async def main():
    # 1. Stdio subprocess — FastMCP spawns `python server.py` and talks over pipes.
    #    This is the default for local dev and desktop hosts like Claude Desktop.
    async with Client("python server.py") as client:
        tools = await client.list_tools()
        print([t.name for t in tools])

    # 2. HTTP server — streamable HTTP transport with SSE for server->client messages.
    async with Client("https://api.example.com/mcp/") as client:
        result = await client.call_tool("search", {"query": "mcp"})

    # 3. In-memory — no transport at all. The client calls the server's handlers
    #    directly in the same process. Used almost exclusively for tests.
    server = FastMCP("test")

    @server.tool
    def add(a: int, b: int) -> int:
        return a + b

    async with Client(server) as client:
        result = await client.call_tool("add", {"a": 2, "b": 3})
        assert result.data == 5

asyncio.run(main())
```

> **Key insight**: the in-memory transport is the killer feature for tests. No
> subprocesses, no ports, no JSON-RPC round-trips. Your test runs in one event loop
> and still exercises the full tool-dispatch, validation, and middleware stack.

The transport is selected by sniffing the target: a string starting with `http://` or
`https://` uses HTTP; a string that looks like a command uses stdio; a `FastMCP`
instance uses in-memory. You can also pass an explicit `transport=...` if the heuristic
is wrong (e.g. an HTTPS URL that should be SSE instead of streamable HTTP).

---

## 2. All Client Operations

Every MCP primitive has a corresponding client method. Listing calls are cheap — the
server caches capability metadata — so call them freely.

```python
async with Client(target) as client:
    # Discovery
    tools = await client.list_tools()                 # list[Tool]
    resources = await client.list_resources()         # list[Resource]
    templates = await client.list_resource_templates() # list[ResourceTemplate]
    prompts = await client.list_prompts()             # list[Prompt]

    # Invocation
    result = await client.call_tool("name", {"arg": 1})
    content = await client.read_resource("uri://...")
    messages = await client.get_prompt("name", {"arg": "x"})

    # Subscribe to a resource — the server pushes change notifications over
    # the transport's back-channel (SSE for HTTP, stderr-multiplexed for stdio).
    await client.subscribe_resource("metrics://current/cpu")
```

`call_tool` returns a `CallToolResult` with both the raw content blocks and a
`.data` property that gives you the parsed structured output (if the tool declared an
output schema). Prefer `.data` in tests — you get real Python objects instead of
hand-parsing text blocks.

| Method                      | Returns                      | Typical use |
|-----------------------------|------------------------------|-------------|
| `list_tools()`              | `list[Tool]`                 | Capability discovery, UIs |
| `call_tool(name, args)`     | `CallToolResult`             | Running a tool |
| `read_resource(uri)`        | `list[ResourceContent]`      | Fetching data |
| `subscribe_resource(uri)`   | `None` (pushes via handler)  | Live dashboards |
| `get_prompt(name, args)`    | `list[PromptMessage]`        | Fetching a reusable prompt |

---

## 3. Authentication for HTTP Clients

HTTP MCP servers are usually behind auth. FastMCP ships two auth strategies that plug
into the `Client`.

```python
import os
from fastmcp import Client
from fastmcp.client.auth import BearerAuth

auth = BearerAuth(token=os.environ["MCP_TOKEN"])

async with Client("https://api.example.com/mcp/", auth=auth) as client:
    result = await client.call_tool("search", {"q": "mcp"})
```

`BearerAuth` sets `Authorization: Bearer <token>` on every request. Use this for
service-to-service calls with a long-lived token from a secrets manager.

For user-facing flows use `OAuthAuth`, which performs the RFC 9728 protected-resource
metadata dance, discovers the authorization server, and runs a PKCE flow in the
browser:

```python
from fastmcp.client.auth import OAuthAuth

auth = OAuthAuth(
    client_id=os.environ["MCP_CLIENT_ID"],
    redirect_uri="http://localhost:8765/callback",
    scope="mcp:read mcp:tools",
)
async with Client("https://api.example.com/mcp/", auth=auth) as client:
    ...
```

Tokens are cached on disk (in the user's config dir) so subsequent runs skip the
browser flow until the refresh token expires. For the server-side counterpart — how to
accept and verify these tokens — see [Production: Authentication](../production/02_authentication.md).

> ⚠️ Never hard-code tokens. `BearerAuth` takes a plain string so it's easy to paste
> one in during debugging; resist the urge. Use env vars, `op read`, or a real
> secrets manager.

---

## 4. Client-Side Handlers: Sampling and Elicitation

An MCP server can ask the client for things: "sample this LLM for me" (sampling) or
"ask the user this question" (elicitation). Host applications like Claude Desktop
handle these automatically. When you're writing your own client — say, a CI bot or a
custom TUI — you provide the handlers yourself.

```python
from anthropic import AsyncAnthropic
from fastmcp import Client
from fastmcp.client.types import SamplingResult, ElicitResult
from mcp.types import TextContent

anthropic = AsyncAnthropic()

async def my_sampler(messages, params):
    """Server asked us to run an LLM. We pick the provider."""
    response = await anthropic.messages.create(
        model="claude-3-5-sonnet-latest",
        messages=[{"role": m.role, "content": m.content.text} for m in messages],
        max_tokens=params.max_tokens or 1024,
    )
    return SamplingResult(
        role="assistant",
        content=[TextContent(type="text", text=response.content[0].text)],
        model="claude-3-5-sonnet-latest",
    )

async def my_elicitor(message, schema):
    """Server wants input from the user. We render a prompt however we like."""
    # In a TUI this would be an input dialog; in CI you'd reject.
    answer = ask_user_tui(message, schema)
    return ElicitResult(action="accept", data=answer)

async with Client(
    "python server.py",
    sampler=my_sampler,
    elicitor=my_elicitor,
) as client:
    await client.call_tool("summarize_inbox", {})
```

This is exactly what a host like Claude Desktop does internally — the only difference
is scale and UI polish. If the server asks for sampling and you haven't registered a
sampler, the request fails with a protocol error; don't omit the handler for tools
that need it.

---

## 5. Server Composition: `mount`

Large projects accumulate tools across unrelated subsystems — GitHub, Jira, Slack,
internal services. You could run four servers, or you could compose them into one.

```python
from fastmcp import FastMCP
from .github_server import gh_mcp     # FastMCP("github") with create_issue, etc.
from .jira_server import jira_mcp     # FastMCP("jira")  with create_ticket, etc.
from .slack_server import slack_mcp

root = FastMCP("dev-tools")
root.mount("github", gh_mcp)   # tools become github_create_issue, etc.
root.mount("jira", jira_mcp)
root.mount("slack", slack_mcp)

if __name__ == "__main__":
    root.run(transport="http", port=8080)
```

What `mount` does:

| Primitive          | Namespacing                                          |
|--------------------|------------------------------------------------------|
| Tools              | `<prefix>_<tool_name>` (e.g. `github_create_issue`)  |
| Resources          | URI prefix: `github+<original-uri>`                  |
| Resource templates | Same prefixing as resources                          |
| Prompts            | `<prefix>_<prompt_name>`                             |
| Lifespans          | All mounted servers' lifespans run on startup        |
| Middleware         | Mounted server's middleware still applies to its own tools |

```
          ┌──────────────────────── root (dev-tools) ────────────────────┐
          │                                                              │
client ──►│  tool: github_create_issue   tool: jira_create_ticket   ...  │
          │         │                              │                     │
          │         ▼                              ▼                     │
          │   ┌──────────┐                   ┌──────────┐                │
          │   │ gh_mcp   │                   │ jira_mcp │                │
          │   │ lifespan │                   │ lifespan │                │
          │   └──────────┘                   └──────────┘                │
          └──────────────────────────────────────────────────────────────┘
```

> **Rule**: always pass an explicit prefix. Mounting without one leaves you with
> name collisions and undefined resolution order.

Composition is static — mounts happen at process start. For dynamic routing
(tenant-per-request, per-user tool visibility), use middleware or a proxy instead.

---

## 6. Server Proxying: Transport Translation

A proxy is a FastMCP server whose implementation is another FastMCP client. It lets
you translate between transports, add auth, inject middleware, or expose a remote
server locally.

```python
import asyncio
import os
from fastmcp import Client, FastMCPProxy
from fastmcp.client.auth import BearerAuth

async def main():
    # The upstream is a remote HTTP server. We expose it locally over stdio
    # so Claude Desktop (which only speaks stdio) can use it.
    upstream = Client(
        "https://api.example.com/mcp/",
        auth=BearerAuth(token=os.environ["MCP_TOKEN"]),
    )
    async with upstream:
        proxy = FastMCPProxy("local-proxy", upstream=upstream)
        await proxy.run_async(transport="stdio")

asyncio.run(main())
```

Typical uses:

- **Desktop host bridging**: your host speaks stdio but the service is HTTP-only.
  Run the proxy as a local subprocess.
- **Adding auth in front of someone else's server**: the upstream has no auth; you
  put a proxy with middleware in front and deploy that publicly.
- **Per-tenant fan-out**: route upstream based on the authenticated identity by
  combining a proxy with custom middleware.
- **Debugging**: log every request and response by proxying a server to itself.

> ⚠️ A bare proxy inherits the upstream's security posture. If the upstream has no
> auth, the proxy has no auth. Always layer authentication middleware in front of a
> public proxy — see [Production: Deployment](../production/06_deployment.md).

---

## 7. OpenAPI Bridge

If you already have a documented REST API, you don't need to rewrite it as MCP.
`from_openapi` walks a spec and produces one tool per operation.

```python
import os
from fastmcp import FastMCP
from fastmcp.openapi import from_openapi
from fastmcp.client.auth import BearerAuth

mcp = from_openapi(
    spec_url="https://api.example.com/openapi.json",
    base_url="https://api.example.com",
    auth=BearerAuth(token=os.environ["API_TOKEN"]),
)

if __name__ == "__main__":
    mcp.run(transport="http", port=8080)
```

Mapping rules:

| OpenAPI concept         | MCP concept                                  |
|-------------------------|----------------------------------------------|
| `operationId`           | Tool name (falls back to `METHOD_/path`)     |
| Path params             | Required tool inputs                         |
| Query params            | Tool inputs (optional unless `required`)     |
| Request body (JSON)     | Nested tool input object                     |
| `summary` / `description` | Tool description                           |
| `200` response schema   | Tool output schema                           |

You almost always want per-operation overrides. Spec-generated names are ugly, and you
need to mark destructive operations so clients warn users before running them.

```python
mcp = from_openapi(
    spec_url="https://api.example.com/openapi.json",
    base_url="https://api.example.com",
    auth=BearerAuth(token=os.environ["API_TOKEN"]),
    overrides={
        "POST_/users": {
            "tool_name": "create_user",
            "annotations": {"destructiveHint": False, "idempotentHint": False},
        },
        "DELETE_/users/{id}": {
            "hidden": True,  # don't expose at all — too dangerous to ship
        },
        "GET_/users/{id}": {
            "tool_name": "get_user",
            "annotations": {"readOnlyHint": True},
        },
    },
)
```

> 💡 The bridge is a great way to get 80% of the way there quickly, then selectively
> replace tools with hand-written FastMCP ones where you need richer behavior (e.g.
> multi-step workflows, server-side sampling, resource subscriptions).

---

## 8. FastAPI Bridge

If your API is a live FastAPI app in the same process, skip OpenAPI entirely. FastMCP
can introspect the `FastAPI` object directly — no spec file, no HTTP round-trip.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from fastmcp import FastMCP

api = FastAPI()

class User(BaseModel):
    id: str
    name: str
    email: str

@api.get("/users/{user_id}")
async def get_user(user_id: str) -> User:
    return await db.fetch_user(user_id)

@api.post("/users")
async def create_user(payload: User) -> User:
    return await db.insert_user(payload)

# Produce an MCP server from the FastAPI app
mcp = FastMCP.from_fastapi(api, name="users-mcp")

if __name__ == "__main__":
    mcp.run(transport="http", port=8080)
```

FastMCP reads the route table, pulls the pydantic types from function signatures, and
produces tools with correct input and output schemas. Since the conversion is
in-process, tool calls short-circuit the HTTP layer — they call the FastAPI handlers
directly, which is faster and avoids issues with internal-only routes.

| Route type                     | Becomes                                |
|--------------------------------|----------------------------------------|
| `GET /users/{id}`              | Tool `get_user(id: str)`               |
| `POST /users`                  | Tool `create_user(payload: User)`      |
| `GET /users` (returns list)    | Tool `list_users()` with array output  |
| WebSocket / streaming routes   | Skipped (use a hand-written tool)      |

> **Rule**: the FastAPI bridge is for the common case. If a route needs side effects
> that only make sense over HTTP (cookies, request-scoped middleware, custom response
> headers), write a dedicated FastMCP tool instead.

---

## 9. Testing Pattern: In-Memory Client

The in-memory transport plus pytest's `asyncio` plugin gives you end-to-end server
tests that run in milliseconds.

```python
# tests/test_user_server.py
import pytest
from fastmcp import Client

from app.server import mcp  # the actual FastMCP instance from your server module


@pytest.mark.asyncio
async def test_get_user_returns_structured_data():
    async with Client(mcp) as client:
        result = await client.call_tool("get_user", {"user_id": "abc"})
        assert result.data["id"] == "abc"
        assert result.data["email"].endswith("@example.com")


@pytest.mark.asyncio
async def test_list_tools_includes_admin_tools():
    async with Client(mcp) as client:
        tools = await client.list_tools()
        names = {t.name for t in tools}
        assert "create_user" in names
        assert "delete_user" in names


@pytest.mark.asyncio
async def test_unknown_tool_raises():
    async with Client(mcp) as client:
        with pytest.raises(Exception, match="Unknown tool"):
            await client.call_tool("not_a_real_tool", {})
```

Pair this with a lifespan fake so you don't hit a real database:

```python
# conftest.py
from contextlib import asynccontextmanager
from app.server import mcp

@asynccontextmanager
async def fake_lifespan(server):
    server.state.db = InMemoryDB()  # swap in a stub
    yield
    await server.state.db.close()

mcp.lifespan = fake_lifespan  # override in tests only
```

> **Key insight**: in-memory tests exercise every layer of the server except the
> transport — validation, middleware, tool dispatch, resource resolution, structured
> output. If they pass, the only thing that can still break is the wire protocol,
> which you test once in an integration suite.

---

## 10. Failure Modes

**❌ Forgetting `async with`**

```python
# Leaks the subprocess — the stdio pipe stays open until GC eventually runs
client = Client("python server.py")
await client.list_tools()  # AttributeError anyway: connection not opened
```

Always use the context manager. There's no manual `connect()` / `close()` API on
purpose.

**❌ Mounting servers with overlapping tool names and no prefixes**

```python
root.mount("", server_a)  # adds create_user
root.mount("", server_b)  # also adds create_user — silent collision
```

The resolution order is undefined. Always give `mount` an explicit, unique prefix.

**❌ Proxying without auth in front**

```python
# The upstream requires no auth; now you've made it public
proxy = FastMCPProxy("public-proxy", upstream=internal_client)
await proxy.run_async(transport="http", port=8080)
```

Put `BearerAuthMiddleware` or the equivalent in front, or bind the proxy to
`127.0.0.1` only. Never expose an unauthenticated upstream to the open internet.

**⚠️ The OpenAPI bridge exposes whatever's in the spec**

The spec was written for humans writing HTTP calls; it may include admin endpoints,
bulk deletes, and internal-only operations. Review every generated tool and either
hide dangerous ones (`hidden: True`) or mark them with `destructiveHint=True` so
clients surface a confirmation dialog.

**⚠️ FastAPI bridge and dependency injection**

If your FastAPI routes use `Depends(...)` with request-scoped dependencies (auth
headers, cookies), those don't run through the MCP tool path — there's no HTTP
request to extract from. Either refactor the dependency to be callable without a
request, or write a dedicated FastMCP tool for those endpoints.

**⚠️ In-memory tests hide transport bugs**

They're fast and thorough but they don't serialize anything. If your tool returns a
type that can't be JSON-encoded, in-memory tests pass and HTTP tests fail. Run at
least a smoke suite against an HTTP-backed client in CI.

---

**Next**: [TypeScript: Getting Started](../typescript/01_getting_started.md)
