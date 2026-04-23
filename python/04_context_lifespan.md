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

---

## 4. What `Context` Gives You

The `Context` object is the entire surface area for a tool's interaction with the
runtime. Memorize this table — it's the API you'll reach for over and over.

| Attribute / Method                                  | Use                                                                    |
|-----------------------------------------------------|------------------------------------------------------------------------|
| `ctx.request_context.lifespan_context`              | App-scoped state (whatever your `lifespan` yielded)                    |
| `ctx.request_id`                                    | Correlate logs with the JSON-RPC request id                            |
| `ctx.client_id`                                     | Stable id for the connected client (when sessions are enabled)         |
| `await ctx.debug(msg)` / `info` / `warning` / `error` | Send a `notifications/message` log entry to the client                 |
| `await ctx.report_progress(progress, total, message)` | Send a `notifications/progress` to the client                          |
| `await ctx.read_resource(uri)`                      | Read another resource exposed by *this* server                         |
| `await ctx.sample(messages, ...)`                   | Server-initiated LLM call — see [05_sampling_elicitation.md](05_sampling_elicitation.md) |
| `await ctx.elicit(message, schema)`                 | Ask the user to fill in missing structured input                       |
| `await ctx.list_roots()`                            | Ask the client which filesystem / URI roots it has granted access to   |

> **Key insight**: `Context` is a *bidirectional* handle. It's not just "the request
> object" — it lets your server call back into the client. That capability is what
> separates MCP from a plain RPC framework.

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

    state: AppState = ctx.request_context.lifespan_context
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

## 7. Lifespan for Non-Async Resources

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

## 8. Per-Request Scratch Space

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

---

## 9. Using FastAPI Alongside FastMCP

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

## 10. Failure Modes

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

⚠️ **Capturing `Context` in a closure that outlives the request**. The context is
tied to the JSON-RPC request; once the tool returns, the context is invalid. Don't
stash it in module-level state, don't pass it into a `create_task` that survives
the call.

> **Key insight**: lifespan owns the *resources*; `Context` is the *handle* a
> single request uses to reach them and to communicate back to the client. Get
> those two responsibilities right and FastMCP's DI story is — genuinely — almost
> all you need.

---

**Next**: [05_sampling_elicitation.md — Sampling and Elicitation](05_sampling_elicitation.md)
