# FastMCP Middleware: A Pipeline for Cross-Cutting Concerns

> **Who this is for**: Engineers who already have tools, resources, and prompts working
> (see [02_tools.md](02_tools.md) and [03_resources_prompts.md](03_resources_prompts.md))
> and have noticed they're copying the same logging, auth, and metrics boilerplate into
> every handler. Middleware is the fix.

Before reading this, understand request context and lifespan: **[04_context_lifespan.md](04_context_lifespan.md)**.

---

## 1. What Middleware Is in FastMCP

Middleware is a function — or, more commonly, a class — that wraps every JSON-RPC request
the server receives. It sees the call **before** the dispatcher routes it to a tool,
resource, or prompt handler, and it sees the response **after** that handler returns.

If you've written ASGI middleware (Starlette, FastAPI), Express middleware (Node), or a
Django middleware class, this is the same shape: an onion of layers, each one wrapping
`call_next`. The outermost layer sees the request first and the response last.

```
incoming JSON-RPC request
        │
        ▼
┌──────────────────────────┐
│  TracingMiddleware       │  ← outermost (registered first)
│  ┌────────────────────┐  │
│  │  LoggingMiddleware │  │
│  │  ┌──────────────┐  │  │
│  │  │   AuthMW     │  │  │
│  │  │  ┌────────┐  │  │  │
│  │  │  │handler │  │  │  │  ← actual @mcp.tool / @mcp.resource
│  │  │  └────────┘  │  │  │
│  │  └──────────────┘  │  │
│  └────────────────────┘  │
└──────────────────────────┘
        │
        ▼
outgoing JSON-RPC response
```

Why bother? Because the logic that should be uniform across every tool — request IDs,
timing, auth checks, rate limits, metrics — has no business being duplicated inside each
`@mcp.tool` body. Middleware extracts it once and applies it everywhere.

> **Key insight**: middleware runs in the JSON-RPC layer, **above** your handlers. It sees
> `tools/call`, `resources/read`, `prompts/get`, `tools/list`, and every other MCP method
> uniformly. Anything that should treat all of those the same way belongs here.

---

## 2. The Middleware Signature

A middleware is a class that subclasses `Middleware` and implements `__call__`:

```python
import logging
import time

from fastmcp.server.middleware import Middleware, MiddlewareContext, CallNext

logger = logging.getLogger(__name__)


class LoggingMiddleware(Middleware):
    async def __call__(self, ctx: MiddlewareContext, call_next: CallNext):
        method = ctx.method            # e.g. "tools/call", "resources/read"
        fastmcp_ctx = ctx.fastmcp_context
        request_id = (
            fastmcp_ctx.request_id
            if fastmcp_ctx and fastmcp_ctx.request_context
            else None
        )
        start = time.perf_counter()
        try:
            result = await call_next(ctx)
            elapsed_ms = (time.perf_counter() - start) * 1000
            logger.info(
                "ok",
                extra={
                    "method": method,
                    "request_id": request_id,
                    "ms": round(elapsed_ms, 1),
                },
            )
            return result
        except Exception:
            elapsed_ms = (time.perf_counter() - start) * 1000
            # logger.exception captures the traceback automatically
            logger.exception(
                "error",
                extra={
                    "method": method,
                    "request_id": request_id,
                    "ms": round(elapsed_ms, 1),
                },
            )
            raise  # re-raise so downstream layers and the client see the failure
```

### Two contexts, don't confuse them

There are **two** different context objects in play, and mixing them up is the most
common source of "why is this `None`?" confusion:

| Object                   | What it is                                                                 | Where it comes from        |
|--------------------------|----------------------------------------------------------------------------|----------------------------|
| `ctx: MiddlewareContext` | The JSON-RPC *envelope* for this hop — method, payload, timing.            | the middleware parameter   |
| `ctx.fastmcp_context`    | The per-request **`Context`** object — the exact one from [doc 04](04_context_lifespan.md), with the full logging / progress / state / sampling / lifespan API. | `MiddlewareContext.fastmcp_context` |

So inside middleware you reach app state, logging, and the state store through
`ctx.fastmcp_context` — it's the same handle a tool gets as `ctx: Context`:

```python
async def __call__(self, ctx: MiddlewareContext, call_next):
    fctx = ctx.fastmcp_context          # the per-request Context from doc 04
    if fctx and fctx.request_context:   # may be None pre-session (e.g. on_initialize)
        await fctx.info(f"handling {ctx.method} as session {fctx.session_id}")
        db = fctx.lifespan_context.db   # reach lifespan resources, same as a tool
    return await call_next(ctx)
```

⚠️ `ctx.fastmcp_context` can be `None`, and even when present its `request_context` can
be `None` during the `initialize` handshake. Always guard
`if fctx and fctx.request_context:` before reading `request_id` / `session_id` — this is
why every example below has that check.

`MiddlewareContext` itself is deliberately small. In FastMCP 3.x it exposes `message`,
`fastmcp_context`, `source`, `type`, `method`, `timestamp`, and `copy()`. It does **not**
directly expose HTTP headers, request IDs, session IDs, or a mutable `state` dict.

Use the right layer for each kind of data:

| Need                      | Where to get it                                                                 |
|---------------------------|----------------------------------------------------------------------------------|
| MCP method                | `ctx.method`                                                                     |
| Operation payload         | `ctx.message`                                                                    |
| Request/session metadata  | `ctx.fastmcp_context.request_id` / `.session_id` when `request_context` exists   |
| HTTP headers              | `get_http_headers()` from `fastmcp.server.dependencies`                          |
| State for handlers        | async `await ctx.fastmcp_context.set_state(...)` / `await ctx.get_state(...)`     |

Use `get_http_headers(include_all=True)` when you also need headers FastMCP excludes by
default, such as `host` or `content-length`.

For tool-specific middleware, `ctx.message` is the operation params object, so a tool
name is `ctx.message.name`, not `ctx.request.params.name`.

`call_next` is an async callable — `await` it exactly once to invoke the rest of the
pipeline and get back the result.

> **Rule**: every code path through your middleware must either `await call_next(ctx)` or
> raise. Forgetting both is the single most common middleware bug, and it manifests as
> "the client just hangs."

---

## 3. Registering Middleware

Middleware is attached to the server with `add_middleware`. Order matters:

```python
from fastmcp import FastMCP

mcp = FastMCP("app")

mcp.add_middleware(LoggingMiddleware())                 # outermost
mcp.add_middleware(RateLimitMiddleware(per_minute=120))
mcp.add_middleware(MetricsMiddleware(registry=prom_registry))  # innermost
```

The first one you register is the **outermost** layer. On the request side it runs first;
on the response side it runs last. So in the example above, `LoggingMiddleware` is the
first to see an incoming request and the last to see the outgoing response — which means
its timing measurement includes everything underneath, including rate limiting and
metrics overhead.

| Position             | Sees request | Sees response |
|----------------------|--------------|---------------|
| Registered first     | First        | Last          |
| Registered last      | Last         | First         |

If you want timings to reflect the full work the server did on behalf of a request — and
you almost always do — register your tracing/logging middleware first.

---

## 4. Hook-Style Middleware

The default `__call__` is a catch-all. For finer-grained interception, FastMCP exposes
named hooks for each MCP method, so you can react only to (say) `tools/call` without
wrapping every other request:

```python
from fastmcp.server.middleware import Middleware, MiddlewareContext
from fastmcp.exceptions import ToolError

DEPRECATED_TOOLS = {"old_search", "legacy_geocode"}
SUCCESSORS = {
    "old_search": "search_v2",
    "legacy_geocode": "geocode",
}


class ToolFilterMiddleware(Middleware):
    async def on_call_tool(self, ctx: MiddlewareContext, call_next):
        tool_name = ctx.message.name
        if tool_name in DEPRECATED_TOOLS:
            # Loud failure — tells the agent exactly what to switch to
            raise ToolError(
                f"{tool_name} has been removed; use {SUCCESSORS[tool_name]} instead."
            )
        return await call_next(ctx)

    async def on_list_tools(self, ctx: MiddlewareContext, call_next):
        tools = await call_next(ctx)
        # Hide deprecated tools from discovery, but still allow direct calls
        # to fail loudly above (in case an old client cached the name).
        return [t for t in tools if t.name not in DEPRECATED_TOOLS]
```

The default `__call__` implementation in the base `Middleware` class dispatches to the
appropriate `on_*` method based on `ctx.method`. So you can mix and match: implement
`__call__` when you want to wrap **every** request uniformly (logging, tracing), and
implement `on_*` hooks when you only care about one or two methods.

| Hook                  | Triggered by              |
|-----------------------|---------------------------|
| `on_call_tool`        | `tools/call`              |
| `on_list_tools`       | `tools/list`              |
| `on_read_resource`    | `resources/read`          |
| `on_list_resources`   | `resources/list`          |
| `on_get_prompt`       | `prompts/get`             |
| `on_list_prompts`     | `prompts/list`            |

> **Key insight**: hooks let you write middleware that's surgical. A tool-only authz
> check shouldn't fire on `resources/list` — use `on_call_tool`, not `__call__`.

---

## 5. A Real-World Rate Limiter

A token-bucket-style limiter, per session, in-memory:

```python
import time
from collections import defaultdict, deque

from fastmcp.server.middleware import Middleware, MiddlewareContext
from fastmcp.exceptions import ToolError


class RateLimitMiddleware(Middleware):
    def __init__(self, per_minute: int = 60):
        self.per_minute = per_minute
        # One sliding-window deque per client
        self._buckets: dict[str, deque[float]] = defaultdict(deque)

    async def __call__(self, ctx: MiddlewareContext, call_next):
        fastmcp_ctx = ctx.fastmcp_context
        client_id = (
            fastmcp_ctx.session_id
            if fastmcp_ctx and fastmcp_ctx.request_context
            else "anonymous"
        )
        now = time.monotonic()
        bucket = self._buckets[client_id]

        # Drop timestamps older than the 60s window
        while bucket and now - bucket[0] > 60:
            bucket.popleft()

        if len(bucket) >= self.per_minute:
            raise ToolError("rate limit exceeded — try again in a minute")

        bucket.append(now)
        return await call_next(ctx)
```

A few things to notice:

- **`time.monotonic`**, not `time.time`. Wall-clock can jump backwards; monotonic can't.
- **Sliding window via deque**, not a fixed bucket. A fixed bucket lets a client send
  `2 * per_minute` requests across a window boundary; a sliding window doesn't.
- **`fastmcp_context.session_id` when available** — during initialization and other
  phases without an MCP request context, this example falls back to `"anonymous"`.
  That's usually too coarse for production; in practice you'd key off an authenticated
  user ID extracted by an upstream auth middleware.

⚠️ This is **per-process**. If you run more than one replica of the server, each replica
keeps its own counters and a client effectively gets `N * per_minute` requests across
`N` replicas. For multi-replica deployments, back this with Redis (a single
`INCR` + `EXPIRE` per request, or a Lua script for sliding window). See
[../production/03_scaling.md](../production/03_scaling.md).

---

## 6. Auth Middleware

Extract a user from the request, verify the token, stash the user on the context so
downstream handlers can read it without re-parsing JWTs:

```python
from fastmcp.server.middleware import Middleware, MiddlewareContext
from fastmcp.server.dependencies import get_http_headers
from fastmcp.exceptions import AuthorizationError

from myapp.auth import verify_jwt, InvalidToken


class AuthMiddleware(Middleware):
    async def on_request(self, ctx: MiddlewareContext, call_next):
        headers = get_http_headers()
        token = headers.get("authorization", "").removeprefix("Bearer ")
        try:
            user = await verify_jwt(token)
        except InvalidToken:
            # AuthorizationError surfaces as an auth failure to the client.
            raise AuthorizationError("authentication required")

        # Tools/resources can read this via await ctx.get_state("user")
        # in their handler bodies. Don't put auth-sensitive material in logs.
        if ctx.fastmcp_context:
            await ctx.fastmcp_context.set_state(
                "user",
                user,
                serializable=False,
            )
        return await call_next(ctx)
```

In a tool handler, you'd then read it as:

```python
@mcp.tool
async def my_tool(arg: str, ctx: Context) -> str:
    user = await ctx.get_state("user")
    if not user.has_scope("tools:write"):
        raise ToolError("insufficient scope")
    ...
```

`Context` state methods are async in FastMCP 3.x. Passing `serializable=False` keeps the
parsed user object (which a JWT decodes to, not JSON) in **request scope** — alive for
this one call and never written to the serializable store. For the full session-vs-request
state model, see [04_context_lifespan.md §7](04_context_lifespan.md#7-session-state--set_state--get_state).

If you have synchronous helper functions that need the current user or tenant, bind that
value to a `ContextVar` in middleware and reset it in a `finally` block instead of trying
to read FastMCP state synchronously.

Two notes on production-grade auth:

- ✅ Validate **issuer**, **audience**, **expiration**, and **signature**. A cached JWKS
  client (with refresh on `kid` miss) keeps the verification cost in the microseconds.
- ❌ Don't roll your own JWT parser. Use `PyJWT` or `python-jose` and let them handle
  the spec edge cases.

For a real OAuth Resource Server pattern with proper scope checks and token introspection,
see [../production/02_authentication.md](../production/02_authentication.md).

---

## 7. Tool-Level Metrics with Prometheus

Counters and a latency histogram, scoped per tool name:

```python
from prometheus_client import Counter, Histogram

from fastmcp.server.middleware import Middleware, MiddlewareContext
from fastmcp.exceptions import ToolError

CALLS = Counter(
    "mcp_tool_calls_total",
    "MCP tool calls",
    ["tool", "status"],
)
LATENCY = Histogram(
    "mcp_tool_latency_seconds",
    "MCP tool latency",
    ["tool"],
)


class MetricsMiddleware(Middleware):
    async def on_call_tool(self, ctx: MiddlewareContext, call_next):
        tool = ctx.message.name
        with LATENCY.labels(tool=tool).time():
            try:
                result = await call_next(ctx)
                CALLS.labels(tool=tool, status="ok").inc()
                return result
            except ToolError:
                # Expected tool-level failures: validation, authz, domain errors.
                CALLS.labels(tool=tool, status="tool_error").inc()
                raise
            except Exception:
                # Hard exceptions: handler crashed, middleware raised, etc.
                CALLS.labels(tool=tool, status="exception").inc()
                raise
```

The three statuses — `ok`, `tool_error`, `exception` — let you distinguish:

| Status      | Meaning                                                    |
|-------------|------------------------------------------------------------|
| `ok`        | Handler returned a successful result                       |
| `tool_error` | Handler raised `ToolError` (validation, authz, domain)    |
| `exception` | Handler or middleware raised — bug, timeout, dependency    |

A jump in `tool_error` rate usually means a client started sending bad inputs; a jump in
`exception` rate usually means **you** broke something. Alert on them differently.

---

## 8. OpenTelemetry Tracing Middleware

FastMCP 3.x ships built-in OTel support, but you'll often want a custom span on top — for
extra attributes, baggage propagation, or to integrate with an existing tracer config:

```python
from opentelemetry import trace

from fastmcp.server.middleware import Middleware, MiddlewareContext

tracer = trace.get_tracer("mcp.server")


class TracingMiddleware(Middleware):
    async def __call__(self, ctx: MiddlewareContext, call_next):
        fastmcp_ctx = ctx.fastmcp_context
        request_id = None
        session_id = None
        if fastmcp_ctx and fastmcp_ctx.request_context:
            request_id = fastmcp_ctx.request_id
            session_id = fastmcp_ctx.session_id

        with tracer.start_as_current_span(ctx.method or "mcp.request") as span:
            span.set_attribute("mcp.request_id", request_id or "")
            span.set_attribute("mcp.session_id", session_id or "")
            try:
                return await call_next(ctx)
            except Exception as exc:
                span.record_exception(exc)
                span.set_status(trace.StatusCode.ERROR)
                raise  # re-raise unchanged so the caller sees the original error
```

Pair this with a logging formatter that includes the current trace ID, and your logs and
traces correlate automatically. See [../production/04_observability.md](../production/04_observability.md)
for the full picture, including span events for `call_next`, sampling strategies, and
exporter configuration.

---

## 9. Composition Order

The order in which you register middleware matters more than people expect, because it
determines which layers' costs get measured and which errors get logged. The
recommendation:

```python
mcp.add_middleware(TracingMiddleware())          # 1. outermost
mcp.add_middleware(LoggingMiddleware())          # 2.
mcp.add_middleware(MetricsMiddleware())          # 3.
mcp.add_middleware(AuthMiddleware())             # 4.
mcp.add_middleware(RateLimitMiddleware(per_minute=120))  # 5.
mcp.add_middleware(ToolFilterMiddleware())       # 6. innermost
```

Why this order:

1. **Tracing** outermost so every span captures the full lifetime of the request,
   including time spent in every other middleware.
2. **Logging** next so logs include timings for everything below — auth checks, rate
   limit lookups, the handler itself.
3. **Metrics** next so the latency histogram reflects user-perceived latency (minus
   tracing/logging overhead, which is negligible).
4. **Auth** before rate limiting so unauthenticated requests get rejected before they
   consume rate limit budget. Otherwise a malicious client can deny service to a real
   one by exhausting their bucket with bogus tokens.
5. **Rate limiting** before business middleware so abusive clients can't trigger
   expensive work.
6. **Tool filtering / business middleware** innermost — these run on requests that have
   already passed every other check.

> **Principle**: outer middleware should be **cheap and observability-focused**; inner
> middleware should be **expensive and decision-making**. Put the things that always run
> on the outside and the things that might short-circuit on the inside.

---

## 10. Failure Modes

**❌ Forgetting to `await call_next(ctx)`**

```python
# ❌ The pipeline never advances; the client hangs until its timeout
async def __call__(self, ctx, call_next):
    logger.info("got %s", ctx.method)
    # forgot to await call_next(ctx) — silent drop
```

The fix is always the same: every code path must either `await call_next(ctx)` and
return its result, or `raise`.

**❌ Re-raising as a different exception type**

```python
# ❌ Loses the original traceback
try:
    return await call_next(ctx)
except Exception as exc:
    raise RuntimeError(f"call failed: {exc}")  # original traceback gone
```

```python
# ✅ Bare `raise` preserves the original exception and traceback
try:
    return await call_next(ctx)
except Exception:
    logger.exception("call failed")
    raise
```

If you genuinely need to wrap, use `raise NewError(...) from exc` so the chain is
preserved.

**❌ Mutating `ctx.message` after `call_next` returns**

```python
# ❌ The dispatcher already read params; mutating them now does nothing
async def __call__(self, ctx, call_next):
    result = await call_next(ctx)
    ctx.message.name = "rewritten"  # too late
    return result
```

If you need to rewrite a request, do it **before** `call_next`. If you need to rewrite a
response, mutate `result` (or build a new one) — not the request.

**⚠️ Middleware that opens HTTP clients per-request**

```python
# ❌ New connection pool every request — massive overhead
async def __call__(self, ctx, call_next):
    async with httpx.AsyncClient() as http:
        await http.post(audit_url, json=...)
    return await call_next(ctx)
```

```python
# ✅ Share a client created in lifespan; reuse the connection pool
class AuditMiddleware(Middleware):
    def __init__(self, http: httpx.AsyncClient):
        self._http = http

    async def __call__(self, ctx, call_next):
        await self._http.post(audit_url, json=...)
        return await call_next(ctx)
```

See [04_context_lifespan.md](04_context_lifespan.md) for how to wire long-lived clients
through the lifespan and pass them into your middleware constructor.

---

**Next**: [07_clients_advanced.md — Clients, Composition, and Advanced Patterns](07_clients_advanced.md)
