# `Context` and `lifespan` — Dependency Injection in FastMCP

> **Who this is for**: Engineers who already have FastMCP tools working and are now
> asking the obvious next question: "where do I put my database connection pool?"
> Assumes you've read [03_resources_prompts.md](03_resources_prompts.md).

---

## 1. Two Scopes — Quick Mental Model

FastMCP gives you exactly two scopes for shared state. Internalize the distinction
before you write any code; almost every "where does this go?" question collapses
once you have the right scope.

| Scope        | Lifetime                          | Created via                          | Typical contents                                  |
|--------------|-----------------------------------|--------------------------------------|---------------------------------------------------|
| **Lifespan** | Process — startup to shutdown     | `lifespan=` arg on `FastMCP(...)`    | DB pools, HTTP clients, ML models, caches, locks  |
| **Context**  | One JSON-RPC request (one tool call) | Auto-injected `Context` parameter | Per-call logging, progress, sampling, elicitation |

```
┌──────────────────────── process ─────────────────────────┐
│                                                          │
│   lifespan startup ──► AppState { db_pool, http, ... }   │
│                              │                           │
│         ┌────────────────────┼────────────────────┐      │
│         ▼                    ▼                    ▼      │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐ │
│   │ tool call│         │ tool call│         │ tool call│ │
│   │  ctx #1  │         │  ctx #2  │         │  ctx #3  │ │
│   └──────────┘         └──────────┘         └──────────┘ │
│         │                    │                    │      │
│   lifespan teardown ◄────────┴────────────────────┘      │
└──────────────────────────────────────────────────────────┘
```

> **Key insight**: lifespan is *what you have*; `Context` is *how the current request
> reaches it*. They are not alternatives — they collaborate. You almost always need
> both.

---

## 2. Lifespan with `@asynccontextmanager`

The lifespan is an async context manager. Everything before `yield` runs once at
startup; everything after runs once at shutdown. The value you `yield` becomes the
lifespan context that every tool can reach through `Context`.

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator
from dataclasses import dataclass
import asyncpg
import httpx
from fastmcp import FastMCP


@dataclass
class AppState:
    """Container for all app-scoped resources. One instance per process."""
    db: asyncpg.Pool
    http: httpx.AsyncClient


@asynccontextmanager
async def lifespan(app: FastMCP) -> AsyncIterator[AppState]:
    """Acquire app-wide resources on startup; release on shutdown."""
    db = await asyncpg.create_pool(
        "postgresql://localhost/app",
        min_size=2,
        max_size=20,
    )
    http = httpx.AsyncClient(timeout=10.0)
    try:
        yield AppState(db=db, http=http)
    finally:
        # try/finally guarantees cleanup runs even if a tool crashes the loop
        # or the process receives SIGTERM mid-flight.
        await http.aclose()
        await db.close()


mcp = FastMCP("app", lifespan=lifespan)
```

A few things worth highlighting:

- The yielded value (`AppState(...)`) is what tools later see as
  `ctx.request_context.lifespan_context`. Use a dataclass or `TypedDict` for
  IDE-friendly attribute access; a bare `dict` works but loses type checking.
- Always wrap the `yield` in `try/finally`. If startup partially succeeds and a
  later step raises, you still want the resources you already acquired to release.
- `lifespan` runs inside the event loop FastMCP owns — there is no second loop and
  no thread pool magic. Anything you `await` here blocks startup.

> **Rule**: anything that holds a network socket, a file handle, or a process — DB
> pools, HTTP clients, gRPC channels, model weights, Redis pools — belongs in
> lifespan. Creating one per tool call is the most common production bug in FastMCP
> servers.

---

## 3. Adding a `Context` Parameter to a Tool

To get at lifespan state (or any of the per-request capabilities), declare a
parameter typed as `Context`. FastMCP detects it by type annotation and injects it
automatically — the model never sees this parameter, because FastMCP strips it from
the JSON Schema it advertises.

```python
from fastmcp import Context


@mcp.tool
async def get_user(user_id: str, ctx: Context) -> dict:
    """Look up a user by id. The `ctx` parameter is injected by FastMCP."""
    state: AppState = ctx.request_context.lifespan_context
    async with state.db.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT id, email, created_at FROM users WHERE id = $1",
            user_id,
        )
    return dict(row) if row else {}
```

Notes that bite people:

- The parameter must be **type-annotated** as `Context` (or `fastmcp.Context`).
  Naming it `ctx` is convention; the type annotation is what FastMCP looks at.
- The model calling your tool sees a schema with only `user_id` — `ctx` is invisible
  to the LLM. You don't need to (and shouldn't) document it in the docstring's
  argument list.
- You can put `ctx` anywhere in the parameter list. Convention is "last", but
  FastMCP doesn't care about position — it cares about type.

⚠️ If you forget the type annotation (`def get_user(user_id, ctx):`), FastMCP will
treat `ctx` as a normal parameter. The model will see it, hallucinate a value for
it, and your tool will get a string where it expected a `Context`.

> **Under the hood**: `ctx: Context` is sugar. At registration time FastMCP rewrites
> it to `ctx: Context = CurrentContext()` and resolves it through a dependency-injection
> system. You can inject narrower things — just the access token, a single token claim,
> the raw HTTP request — the same way. See [§8](#8-other-ways-to-inject--the-current-family).

---

## 4. What `Context` Gives You — Full Reference

`Context` is the entire surface area for a tool's interaction with the runtime. Its
members fall into five jobs: **reach app state**, **identify the request**, **talk back
to the client**, **reach other components**, and **persist session state**. You won't
use all of these in one tool — but knowing the shape stops you from reinventing things
FastMCP already hands you.

**Reach app-scoped state and the server**

| Member                  | Returns                          | Notes                                                                                          |
|-------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| `ctx.lifespan_context`  | whatever your `lifespan` yielded | The shortcut for app state. Same value as `ctx.request_context.lifespan_context`, but also works inside background tasks where there is no request context. |
| `ctx.fastmcp`           | the `FastMCP` server instance    | Reach server config and registered components. Held by weakref — don't stash it past the call. |

So the `get_user` tool above is more idiomatically written `state: AppState = ctx.lifespan_context`.

**Identify the request**

| Member                | Returns                                          | Notes                                                                                                   |
|-----------------------|--------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `ctx.request_id`      | `str`                                            | JSON-RPC request id; correlate logs with a single call. **Raises** if the MCP session isn't established yet. |
| `ctx.session_id`      | `str`                                            | Stable id for the session, on **every** transport (HTTP uses the `mcp-session-id` header; stdio/SSE get a generated UUID). Use it as a key into Redis/session storage. |
| `ctx.client_id`       | `str \| None`                                    | Id the client supplied at init, when it supplied one.                                                   |
| `ctx.transport`       | `"stdio" \| "sse" \| "streamable-http" \| None`  | Which transport is serving this request.                                                                |
| `ctx.request_context` | `RequestContext \| None`                         | The raw SDK request context. **`None` until the MCP session is established** (e.g. inside `on_initialize` middleware) — guard before reading `request_id`/`session_id`. |

**Talk back to the client** (all `async`)

| Method                                                | Use                                                                                              |
|-------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| `await ctx.debug(msg)` / `info` / `warning` / `error` | Send a `notifications/message` log entry to the client (see [§5](#5-logging-from-tools)).         |
| `await ctx.report_progress(progress, total, message)` | Send a `notifications/progress` (see [§6](#6-progress-reporting-for-long-running-tools)). Works in foreground and background tasks. |
| `await ctx.sample(messages, ...)`                     | Server-initiated LLM call — see [05_sampling_elicitation.md](05_sampling_elicitation.md).         |
| `await ctx.elicit(message, response_type)`            | Ask the user to fill in structured input. Pass an explicit `response_type` (a Pydantic model, dataclass, or primitive) — omitting it is deprecated. |
| `await ctx.list_roots()`                              | Ask the client which filesystem / URI roots it has granted access to.                             |
| `await ctx.send_notification(notif)`                  | Send an arbitrary server notification (e.g. `ToolListChangedNotification()`).                     |
| `await ctx.close_sse_stream()`                        | Gracefully close the HTTP stream so the client reconnects — used to dodge load-balancer timeouts on very long tools (StreamableHTTP + `EventStore` only; no-op otherwise). |

**Reach other components on this server** (all `async`)

| Method                                          | Use                                                          |
|-------------------------------------------------|--------------------------------------------------------------|
| `await ctx.read_resource(uri)`                  | Read a resource exposed by *this* server.                    |
| `await ctx.list_resources()` / `list_prompts()` | Enumerate this server's resources/prompts (auto-paginated).  |
| `await ctx.get_prompt(name, arguments)`         | Render one of this server's prompts.                         |

**Persist session state** — `set_state` / `get_state` / `delete_state`, covered in [§7](#7-session-state--set_state--get_state).

**Background tasks** (only relevant if you use `task=True`): `ctx.is_background_task` (bool) and `ctx.task_id` tell a handler it's running in a Docket worker rather than a live request. `elicit`/`sample`/`report_progress` transparently switch to task-aware implementations there.

### What `Context` does *not* carry

Inside a tool, `ctx` is effectively your one-stop object — but "everything" has exactly
two deliberate gaps, and both are "wrong layer" rather than "missing feature":

| Not on `Context`                          | Where it lives instead                                                                 | Why                                                                                       |
|-------------------------------------------|----------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| The MCP **method** + raw, undispatched **params** (`tools/call`, the unparsed payload) | `MiddlewareContext.method` / `.message` (see [06_middleware.md](06_middleware.md))      | By the time you're in a handler, the method is already routed and the params are already your function arguments. Only middleware needs the envelope. |
| The raw **HTTP request / headers**         | `get_http_request()` / `get_http_headers()` from `fastmcp.server.dependencies`, or `CurrentRequest()` / `CurrentHeaders()` injected in the signature ([§8](#8-other-ways-to-inject--the-current-family)) | `Context` gives you `session_id` and `transport`, but the HTTP layer is below MCP. It's reached through the DI helpers, not a `ctx` attribute. |

> **Key insight**: `Context` is a *bidirectional* handle. It's not just "the request
> object" — it lets your server call back into the client (sample, elicit, log, progress).
> That capability is what separates MCP from a plain RPC framework. The only things it
> deliberately omits are the middleware **envelope** and the raw **HTTP layer** — both
> because they belong to a different layer than "I'm running a request and need to act on it."

---

## 5. Logging from Tools

Tool-side logging through `ctx` is delivered to the client as
`notifications/message` events. Different clients render them differently — Claude
Desktop weaves them into the tool-call UI; the MCP Inspector shows them in a side
panel; raw clients can filter them by level.

```python
@mcp.tool
async def import_csv(path: str, ctx: Context) -> dict:
    """Import a CSV into the database. Streams progress logs to the client."""
    await ctx.info(f"opening {path}")
    rows = _read(path)
    await ctx.info(f"parsed {len(rows)} rows")

    if not rows:
        await ctx.warning("no rows found — nothing to import")
        return {"imported": 0}

    state: AppState = ctx.lifespan_context
    async with state.db.acquire() as conn:
        await conn.executemany(
            "INSERT INTO records (key, value) VALUES ($1, $2)",
            [(r["key"], r["value"]) for r in rows],
        )

    await ctx.info(f"imported {len(rows)} rows")
    return {"imported": len(rows)}
```

Severity choice mirrors stdlib logging:

- `debug` — diagnostic noise; usually filtered out by default.
- `info` — normal milestones the user might care about ("opened file", "wrote 200 rows").
- `warning` — something unusual but recoverable.
- `error` — the operation is going to fail or has failed.

⚠️ Every `ctx.info(...)` is shipped over the wire. Don't dump entire payloads —
truncate large strings, summarize lists. A chatty tool can saturate the transport
on a slow connection.

---

## 6. Progress Reporting for Long-Running Tools

For anything that takes more than a couple of seconds, emit progress. The client
can render a progress bar; without progress, the user just sees a spinner and
starts wondering whether it hung.

```python
@mcp.tool
async def reindex(ctx: Context) -> dict:
    """Rebuild the search index. Reports per-item progress."""
    items = await _all_items()
    total = len(items)

    for i, item in enumerate(items):
        await _index(item)
        await ctx.report_progress(
            progress=i + 1,
            total=total,
            message=f"indexed {item.id}",
        )

    return {"reindexed": total}
```

Two operational details:

- `progress` and `total` are floats; you can use a fractional `progress` (0.0–1.0)
  with `total=1.0` if you don't have a clean integer count.
- If the user cancels the request from the client, FastMCP raises
  `asyncio.CancelledError` inside your tool at the next `await`. Let it propagate.
  Do *not* catch and swallow it — that defeats cancellation and leaves the client
  thinking the tool is still running.

```python
import asyncio

@mcp.tool
async def reindex(ctx: Context) -> dict:
    items = await _all_items()
    try:
        for i, item in enumerate(items):
            await _index(item)
            await ctx.report_progress(i + 1, len(items))
    except asyncio.CancelledError:
        await ctx.warning("reindex cancelled by client; partial state retained")
        raise  # re-raise — don't silently turn cancellation into a normal return
    return {"reindexed": len(items)}
```

---

## 7. Session State — `set_state` / `get_state`

Beyond reaching lifespan resources, `Context` carries a small key-value store. This is
the supported way to pass data *between* a request's middleware and its handler, or
across several tool calls in the same session. Two scopes, picked by the `serializable`
flag:

| Scope                          | Set with                          | Lifetime                                  | Value constraint                          |
|--------------------------------|-----------------------------------|-------------------------------------------|-------------------------------------------|
| **Session** (default)          | `set_state(k, v)`                 | Persists across every request in the session (1-day TTL) | Must be JSON-serializable     |
| **Request** (`serializable=False`) | `set_state(k, v, serializable=False)` | Only the current request (one tool call)  | Anything — clients, connections, objects  |

Keys are automatically namespaced by `session_id`, so one client never sees another's
state. `get_state` returns `None` for a missing key.

```python
@mcp.tool
async def remember(value: str, ctx: Context) -> str:
    await ctx.set_state("last_value", value)          # survives to the next call
    return "stored"

@mcp.tool
async def recall(ctx: Context) -> str:
    return await ctx.get_state("last_value") or "nothing yet"
```

The canonical pattern is **middleware produces, handler consumes**. Auth middleware
verifies a token once and stashes the parsed user object (non-serializable) so every
handler reads it without re-parsing a JWT:

```python
# in middleware (see 06_middleware.md §6)
await ctx.fastmcp_context.set_state("user", user, serializable=False)

# in any handler
user = await ctx.get_state("user")
```

⚠️ Request-scoped (`serializable=False`) state does **not** survive to the next request,
and in distributed/serverless HTTP deployments it isn't shared across machines. For
cross-request or cross-replica state, store something JSON-serializable (an id, not the
live object) and rehydrate from it.

💡 Storing a non-serializable value with the default `serializable=True` raises a
`TypeError` whose message tells you to pass `serializable=False`. If you see that error,
you wanted request scope.

---

## 8. Other Ways to Inject — the `Current*` Family

`ctx: Context` is the workhorse, but it's one member of a dependency-injection system.
When a handler only needs *one slice* of the request, you can inject that slice directly
by giving a parameter a `Current*()` default. FastMCP resolves it per call and hides it
from the tool's JSON Schema, exactly like `ctx`.

| Dependency             | Injects                                        | Reach for it when                                              |
|------------------------|------------------------------------------------|----------------------------------------------------------------|
| `CurrentContext()`     | the `Context`                                  | identical to `ctx: Context` (the annotation rewrites to this)  |
| `CurrentFastMCP()`     | the `FastMCP` server                           | you need server introspection but not the full context        |
| `CurrentRequest()`     | the Starlette `Request`                        | HTTP transports only — raw request access                     |
| `CurrentHeaders()`     | `dict[str, str]` of HTTP headers (incl. `authorization`) | reading headers without calling `get_http_headers()` |
| `CurrentAccessToken()` | the verified `AccessToken`                     | auth is configured and you want the whole token; raises if unauthenticated |
| `TokenClaim("sub")`    | a single claim, as `str`                       | you only need one field — user id, tenant, email              |
| `Progress()`           | a progress handle that also works in background tasks | progress that must work under a Docket worker          |

```python
from fastmcp.server.dependencies import TokenClaim, CurrentHeaders

@mcp.tool
async def add_expense(
    amount: float,
    user_id: str = TokenClaim("sub"),     # extracted from the access token's claims
    headers: dict = CurrentHeaders(),     # raw HTTP headers, authorization included
) -> dict:
    # user_id and headers are injected — the model only sees `amount`
    return {"user": user_id, "amount": amount}
```

The same values are reachable **imperatively** (e.g. from a plain helper function that
doesn't take `ctx`) via `fastmcp.server.dependencies`: `get_context()`, `get_server()`,
`get_http_request()`, `get_http_headers()`, and `get_access_token()`. The last returns
`None` instead of raising when there's no authenticated user — use it for "auth optional"
code paths.

> **Key insight**: prefer `ctx: Context` for general tools. Reach for
> `TokenClaim` / `CurrentAccessToken` / `CurrentHeaders` when a handler needs exactly one
> piece of the request — making that dependency explicit in the signature reads better
> than fishing it out of `ctx` and is trivially unit-testable (just pass the value).

---

## 9. Lifespan for Non-Async Resources

Plenty of useful things have purely sync setup — loading a JSON config file,
deserializing a model with `joblib.load`, opening a sqlite file. Use the same
`@asynccontextmanager` pattern; just call the sync code synchronously inside it.

```python
import json
from pathlib import Path

@dataclass
class AppState:
    config: dict
    model: object  # whatever your ML lib returns

@asynccontextmanager
async def lifespan(app: FastMCP) -> AsyncIterator[AppState]:
    config = json.loads(Path("config.json").read_text())  # sync — fine at startup
    model = _load_model(config["model_path"])             # sync — also fine
    try:
        yield AppState(config=config, model=model)
    finally:
        # No cleanup needed — Python GC handles it. Still keep try/finally for
        # symmetry and so adding async resources later doesn't introduce a leak.
        pass
```

💡 If a sync setup step is genuinely slow (>1s) and you have other async setup to
do, run it in a thread with `asyncio.to_thread(...)` so the loop isn't blocked
while you wait. For most servers this is overkill.

---

## 10. Per-Request Scratch Space

People sometimes ask: "what's the FastMCP way to share state between helper
functions called from one tool?" Answer: **local variables**. There's no
per-request DI container to register things into — you already *are* in the
per-request scope. Just pass values down the call stack.

```python
@mcp.tool
async def synthesize(query: str, ctx: Context) -> str:
    state: AppState = ctx.request_context.lifespan_context
    async with state.db.acquire() as conn:        # local — scoped to this call
        rows = await _fetch(conn, query)
        ranked = _rank(rows)                       # plain function, plain args
        return _format(ranked)
```

If you find yourself wanting "per-request global state", you almost certainly want
to either (a) pass `ctx` and the dataclass down explicitly, or (b) restructure so
the state isn't ambient. Ambient request-scoped state is the kind of pattern that
looks clean for two weeks and then turns into a debugging nightmare.

The one legitimate exception is when the *producer* isn't the tool body but
middleware running before it — there's no call stack to thread a value through. That's
exactly what request-scoped `ctx.set_state(..., serializable=False)` is for
([§7](#7-session-state--set_state--get_state)). Inside a single tool, prefer locals.

---

## 11. Using FastAPI Alongside FastMCP

A common production setup: you have an existing FastAPI app and you want to add an
MCP endpoint to it. Mount FastMCP's HTTP transport into FastAPI and share one
lifespan between the two.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastmcp import FastMCP


@dataclass
class AppState:
    db: asyncpg.Pool


@asynccontextmanager
async def lifespan(app):
    db = await asyncpg.create_pool("postgresql://localhost/app")
    try:
        yield AppState(db=db)
    finally:
        await db.close()


mcp = FastMCP("app", lifespan=lifespan)
api = FastAPI(lifespan=lifespan)

# Mount the MCP HTTP transport under /mcp on the FastAPI app:
api.mount("/mcp", mcp.streamable_http_app())
```

Now `uvicorn myapp:api --port 8000` exposes both:

- Your REST endpoints (`/users`, `/health`, etc.) handled by FastAPI.
- The MCP transport at `/mcp`, served by FastMCP, sharing the same DB pool.

⚠️ Note that FastAPI and FastMCP each invoke `lifespan` themselves. With the
arrangement above, `lifespan` runs **twice** — once for each app — so you'll get
two pools. For real shared state, write a single lifespan that creates the
resources and threads them into both apps via module-level state, or use FastMCP's
built-in `streamable_http_app(lifespan=...)` integration to take over fully. Read
the FastMCP docs for the current recommended wiring; the API around mounted
sub-apps shifted in 2024–2025.

---

## 12. Failure Modes

Real things that go wrong in production:

❌ **Creating a new DB pool inside every tool**. The single most common FastMCP bug.
You'll exhaust Postgres' connection limit within minutes of any moderate traffic,
and tools will start timing out on `acquire()`. Always lifespan.

```python
# ❌ Wrong — new pool per call
@mcp.tool
async def query(sql: str) -> list:
    pool = await asyncpg.create_pool(DSN)   # creates a new pool every request
    async with pool.acquire() as c:
        return await c.fetch(sql)
```

❌ **Forgetting `try/finally` in lifespan**. If startup raises after you've already
opened a pool, the pool never closes, and on container restart you may briefly hold
double the connections.

❌ **`Context` parameter without type hint**. FastMCP can't detect it, the model
sees a parameter it shouldn't, and you'll get a string where you expected an
object. The traceback is confusing because everything looks right at the call
site.

⚠️ **Logging large payloads through `ctx.info`**. Notifications travel over the
same transport as responses. Dumping a 5 MB SQL result row-by-row through
`ctx.info` will tank latency for that session and may exceed client buffers.
Truncate.

⚠️ **Using lifespan resources from background tasks started before lifespan
finishes**. If you spawn `asyncio.create_task(...)` during lifespan startup, that
task can race ahead and try to use a half-initialized `AppState`. Either
`await` everything in startup serially, or guard the background task with an
`asyncio.Event` you set after `yield`.

⚠️ **Reading `ctx.request_id` / `ctx.session_id` before the session exists**. Both
raise (or, for `session_id`, need a fallback) when `ctx.request_context is None` — which
is the case during `on_initialize` middleware and other pre-session phases. Guard with
`if ctx.request_context is not None:` before touching them. This is why the middleware
examples in [06_middleware.md](06_middleware.md) check `fastmcp_ctx.request_context`
before reading ids.

⚠️ **Capturing `Context` in a closure that outlives the request**. The context is
tied to the JSON-RPC request; once the tool returns, the context is invalid. Don't
stash it in module-level state, don't pass it into a `create_task` that survives
the call. (Background tasks are the supported exception — use `task=True`, which gives
the worker its own task-aware `Context`, rather than smuggling the request's context out.)

> **Key insight**: lifespan owns the *resources*; `Context` is the *handle* a
> single request uses to reach them and to communicate back to the client. Get
> those two responsibilities right and FastMCP's DI story is — genuinely — almost
> all you need.

---

**Next**: [05_sampling_elicitation.md — Sampling and Elicitation](05_sampling_elicitation.md)
