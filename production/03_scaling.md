# Scaling MCP Servers

> **Who this is for**: Engineers running an MCP server beyond a single replica. You've shipped
> something that works on one box and now traffic, tenants, or HA requirements are forcing
> the conversation about horizontal scale. Assumes you've read
> [01_streamable_http.md](01_streamable_http.md) and [02_authentication.md](02_authentication.md).

---

## 1. The Stateful-vs-Stateless Decision

This is the most important production choice you make. Get it wrong and you spend the next
six months either retrofitting a session store onto a stateless deployment or fighting
sticky-session bugs on a deployment that never needed sessions in the first place.

The Streamable HTTP transport supports both modes. They are not interchangeable — they
unlock different feature sets and impose very different operational constraints.

| Mode | What | Pros | Cons |
|------|------|------|------|
| **Stateful** | Server returns `Mcp-Session-Id` on initialize; client echoes on every request; session-scoped state lives in process memory | Subscriptions work, server-push (sampling/elicitation) is simple, fewer auth checks per request | Requires sticky sessions, harder to scale horizontally, replica restart drops sessions |
| **Stateless** | No session id; every request re-initializes implicitly; nothing kept between calls | Trivially horizontal-scalable, replica failure is invisible, works on serverless | No subscriptions, no live notifications, no server-push request flows |

> **Key insight**: Stateful is not "better" — it's "more capable but more expensive to
> operate." If your tools are pure RPC (input → output, no subscriptions, no mid-call
> prompts), stateless is dramatically simpler and the right default. Reach for stateful
> only when a feature you actually need forces it.

The decision is largely irreversible without a client-visible breaking change, so make it
deliberately. The next two sections give you a checklist.

---

## 2. What Forces Stateful

If any of these apply to your server, you must run stateful and accept the operational
overhead:

- **Resource subscriptions** — clients call `resources/subscribe` and you fire
  `notifications/resources/updated` when something changes. The notification has to find
  the subscriber, which means the subscriber's connection has to be reachable from the
  replica that knows about the change. In practice that means the same replica.
- **Live capability changes** — `notifications/tools/list_changed`,
  `notifications/resources/list_changed`, `notifications/prompts/list_changed`. Same
  reasoning: the server is initiating the message.
- **Server-initiated requests** — `sampling/createMessage` (server asks the client's LLM
  to complete something) and `elicitation/create` (server asks the user a question
  mid-tool-call). These require the GET stream to be open so the server can push the
  request and correlate the response.
- **Long-lived logical conversations** where you genuinely need server-side state to
  persist between calls (e.g., a multi-step workflow tool that holds intermediate
  results in memory).

> **Note**: Progress notifications mid-call (`notifications/progress`) do *not* force
> stateful mode. The SSE response on a single POST already carries them — the server can
> emit progress frames before the final response on the same HTTP connection. This case
> alone is fine on stateless.

If none of the above apply, default to stateless. You can always add a stateful endpoint
later for the specific tool that needs it; the reverse migration is brutal.

---

## 3. What Allows Stateless

Stateless mode is appropriate when:

- **Tool calls complete in a single request/response** — the client POSTs, the server
  computes, returns, done. No follow-up messages from the server.
- **No mid-call user prompts** — the server doesn't need to ask the user for
  clarification or approval during a tool call.
- **Tools are pure RPC** — input goes in, output comes out, any side effects are
  committed before the response is sent.
- **Auth is per-request** — you re-validate the token on each call (which you should be
  doing anyway; see [02_authentication.md](02_authentication.md)).

The vast majority of practical MCP servers — wrappers around REST APIs, database query
tools, file fetchers, search backends — fit cleanly into stateless. The default in most
new SDK examples is shifting toward stateless for exactly this reason.

---

## 4. Stateless Config — Python (FastMCP)

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("api")

# ... tools/resources/prompts ...

mcp.run(
    transport="http",
    host="0.0.0.0",
    port=8080,
    stateless_http=True,    # no Mcp-Session-Id; every request stands alone
    json_response=True,     # always return application/json (no SSE framing)
)
```

The `stateless_http=True` flag tells FastMCP not to issue session ids and not to retain
per-session state. The `json_response=True` flag forces the response to be a single JSON
object instead of an SSE stream — useful when you have no progress notifications to emit
and want maximum compatibility with HTTP infrastructure that doesn't grok long-lived
streams (CDNs, some WAFs, simple reverse proxies).

Behind any L4 or L7 load balancer; deploy on AWS Lambda via an adapter, Cloud Run,
Fargate, or a plain Kubernetes Deployment. Horizontal scale needs no special routing —
any replica can serve any request.

> **Rule**: If you set `stateless_http=True`, do not store anything in module-level
> mutable state expecting it to be visible to the next request. The next request might
> hit a different replica, a different container, or a freshly-cold-started Lambda.

---

## 5. Stateless Config — TypeScript

```typescript
import express from "express";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { makeServer } from "./server.js";

const app = express();
app.use(express.json());

app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,   // explicitly disables sessions
  });
  const server = makeServer();
  await server.connect(transport);

  // Tear down per-request resources when the response closes — prevents leaks
  // when the client disconnects mid-stream or the request errors.
  res.on("close", () => {
    transport.close();
    server.close();
  });

  await transport.handleRequest(req, res, req.body);
});

app.listen(8080);
```

The server and transport are constructed per request and torn down after — this is the
defining characteristic of stateless mode in the TS SDK. It's cheap because `makeServer()`
just wires handlers; the expensive things (DB pools, HTTP clients, auth caches) live
outside.

> **Rule**: Lifespan resources — database connection pools, outbound HTTP clients, JWKS
> caches, anything you don't want to recreate per-request — MUST be module-level
> singletons created at startup, not request-scoped. If you create a Postgres pool inside
> `makeServer()`, you'll exhaust your database in the first minute of traffic.

```typescript
// ✅ Module-level — created once, shared across all requests
import { Pool } from "pg";
const pgPool = new Pool({ connectionString: process.env.DATABASE_URL, max: 10 });

function makeServer() {
  const server = new Server({ name: "api", version: "1.0.0" }, { capabilities: { tools: {} } });
  server.setRequestHandler(CallToolRequestSchema, async (req) => {
    const result = await pgPool.query("SELECT ...");  // reuses pool across replicas
    return { content: [{ type: "text", text: JSON.stringify(result.rows) }] };
  });
  return server;
}
```

---

## 6. Stateful + Sticky Sessions — Three Patterns

If you've decided you need stateful, the next problem is making sure the client's second
request lands on the same replica as the first. Three patterns, in order of
preference:

### 6a. L7 sticky by header

Most modern L7 load balancers can hash on a custom header. Use `Mcp-Session-Id` as the
affinity key — it's already on every request after initialize.

**nginx**:
```nginx
upstream mcp {
    hash $http_mcp_session_id consistent;
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
    server 10.0.1.12:8080;
}

server {
    listen 443 ssl;
    location /mcp/ {
        proxy_pass http://mcp;
        proxy_buffering off;          # SSE requires unbuffered streaming
        proxy_read_timeout 600s;      # long-lived GET stream for server-push
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

The `consistent` keyword uses ketama-style consistent hashing so adding or removing a
backend doesn't reshuffle every existing session. The `proxy_buffering off` line is
critical — without it nginx will buffer the SSE response and your clients will hang
waiting for events that are sitting in nginx's buffer.

The initialize request has no `Mcp-Session-Id` header yet, so it hashes to an empty
string and lands on a single replica. This is fine for a small fleet but becomes a hot
spot at scale; consider hashing the empty case on a different field (source IP, request
id) to spread initialize load.

### 6b. L7 cookie-based stickiness

ALB, HAProxy, and nginx-plus can issue an opaque cookie on the first response; the
client sends it back on subsequent requests and the LB pins them to the same backend.

Works well for browser-based MCP hosts that handle cookies natively. Doesn't work for
clients that don't (most CLI MCP clients ignore Set-Cookie). Check your client matrix
before relying on this.

### 6c. Service mesh

Istio's `DestinationRule` supports header-based consistent hashing as a first-class
feature:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: mcp-server
spec:
  host: mcp-server
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpHeaderName: Mcp-Session-Id
```

This is the cleanest Kubernetes-native option if you're already running a mesh. Linkerd
has equivalent functionality via `ServiceProfile`. The win over nginx is that you don't
manage a separate LB layer — the mesh does it inside your cluster, and you get mTLS,
metrics, and retries for free.

> **Anti-pattern**: Source-IP affinity (L4 sticky). It works until clients NAT through a
> shared egress (corporate proxies, Cloud NAT, mobile carrier CGNAT) and suddenly half
> your sessions stack on one replica. Header-based hashing avoids this.

---

## 7. Externalizing State — When Sticky Isn't Enough

Sticky sessions break in three scenarios: replica restarts (deploy, OOM kill, node
drain), zone failures, and any case where you need a session to survive longer than the
process holding it. If your users genuinely need cross-replica session portability,
you have to externalize the session state.

- **Session store**: Redis with `Mcp-Session-Id` as the key. The value holds session
  metadata — negotiated capabilities, lifespan-derived state, subscription set, last
  activity timestamp. On every request, the replica reads the session from Redis,
  processes the call, writes back any updates. TTL the keys so abandoned sessions don't
  accumulate.

- **Event store for resumability**: Redis Streams or Kafka, partitioned by session id.
  Replicas write outgoing notifications to the stream; whichever replica currently holds
  the client's GET stream reads from the stream and pushes events down. Combined with
  the spec's `Last-Event-ID` resumability mechanism, this lets a client reconnect to a
  different replica after a network blip and pick up where it left off.

- **Pending request map**: when the server initiates a request to the client (sampling,
  elicitation), it allocates a request id and waits for the response. If the response
  might come back to a different replica than the one that sent the request, the request
  id → response future map must live in shared storage too. Redis with pub/sub works:
  the issuing replica subscribes to a channel keyed by request id; the receiving replica
  publishes the response there.

```python
# Sketch — externalized session read on every request
import redis.asyncio as redis
import json

r = redis.from_url("redis://session-store:6379")

async def load_session(session_id: str) -> dict | None:
    raw = await r.get(f"mcp:session:{session_id}")
    return json.loads(raw) if raw else None

async def save_session(session_id: str, state: dict, ttl: int = 3600) -> None:
    await r.setex(f"mcp:session:{session_id}", ttl, json.dumps(state))
```

> **Warning**: This is operationally heavy. You're now running a stateful service that
> depends on Redis availability, you have a serialization boundary on every request, and
> you've added a network hop to every call. Only invest in this if your users genuinely
> need cross-replica session portability — most don't. Sticky sessions plus a graceful
> drain on deploy covers 95% of real-world cases.

---

## 8. Autoscaling Considerations

### Stateless

Autoscale on CPU or RPS like any other HTTP service. HPA on Kubernetes, target tracking
on ECS, concurrency on Cloud Run. There's nothing special — every request stands alone,
new replicas pick up traffic immediately, dying replicas drop in-flight requests but
clients retry.

### Stateful

Scaling **out** is fine: new sessions land on new pods via the consistent-hash LB, no
existing sessions move. The rebalance is gradual as old sessions expire.

Scaling **in** is the problem. A pod going away takes its sessions with it. You need to
drain: stop accepting new sessions, let existing ones finish or time out, then exit.

```python
import signal
from mcp.server.fastmcp import FastMCP
from mcp.server.fastmcp.exceptions import ToolError

mcp = FastMCP("api")
draining = False

def on_sigterm(*_):
    global draining
    draining = True

signal.signal(signal.SIGTERM, on_sigterm)

@mcp.middleware
async def reject_new_sessions(ctx, call_next):
    # Refuse new sessions during drain so the LB sends them elsewhere.
    # Existing sessions continue working until they idle out or the grace period ends.
    if draining and ctx.method == "initialize":
        raise ToolError("server draining; try another replica")
    return await call_next(ctx)
```

Pair with a Kubernetes `terminationGracePeriodSeconds` long enough to cover your
expected session length. If your sessions live 10 minutes and your grace period is 30
seconds, you'll kill in-flight tool calls every deploy.

```yaml
spec:
  terminationGracePeriodSeconds: 900   # 15 minutes; tune to your session lifetime
  containers:
    - name: mcp
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]   # let LB notice the pod is gone
```

> **Tip**: If your sessions can run for hours, draining will block every deploy for the
> full duration. At that point either (a) cap session TTL at something deploy-friendly,
> or (b) externalize state per Section 7 and let the pod die without ceremony.

---

## 9. Multi-Tenant Scaling

Two patterns, very different operational shapes:

- **Tenant per session**: a single replica serves any tenant. Auth carries the tenant id
  (claim in the JWT, header on the request), tools scope their data access by that id.
  Trivially scalable — same as any multi-tenant SaaS API. Stateless mode is a natural
  fit because there's no session-scoped tenant state to manage.

- **Tenant per process**: each tenant gets its own deployment. A gateway routes by
  hostname (`acme.mcp.example.com` → acme deployment) or path prefix
  (`/tenants/acme/mcp` → acme deployment). Useful when you need hard isolation —
  regulated industries, dedicated DB credentials per tenant, per-tenant resource
  limits. Operationally heavy: you're managing N deployments instead of one.

> **Recommendation**: For SaaS MCP, start with tenant-per-session + stateless + a
> per-request tenant context derived from the auth token. Move tenants onto dedicated
> deployments only when you have a concrete reason (compliance, noisy-neighbor, custom
> resource limits). Don't preemptively split — you'll spend forever managing
> deployments instead of building features.

```python
@mcp.tool()
async def query_orders(ctx: Context, since: str) -> list[dict]:
    # Tenant id pulled from the validated JWT, not from any session state.
    # Works identically on every replica; no sticky routing needed.
    tenant_id = ctx.request.auth.claims["tenant_id"]
    async with pgPool.acquire() as conn:
        return await conn.fetch(
            "SELECT * FROM orders WHERE tenant_id = $1 AND created_at >= $2",
            tenant_id, since,
        )
```

---

## 10. Capacity Planning Rules of Thumb

- **File descriptors**: Each open SSE stream is a long-lived TCP connection. On Linux
  the default `ulimit -n` is often 1024 — you'll run out by your hundredth concurrent
  client. Bump to 65536 or higher in the systemd unit / pod spec, and watch
  `/proc/<pid>/limits` to confirm.

- **I/O-bound tools**: Most MCP tools wrap upstream APIs and spend their time waiting on
  network. Use async everywhere (`asyncio` in Python, native promises in TS) and a
  generous worker count — uvicorn `--workers 4` per CPU core is a fine starting point,
  with each worker handling hundreds of concurrent async requests.

- **CPU-bound tools**: If a tool does real computation (parsing, vectorizing,
  cryptographic ops), it'll block the event loop. Push these to a worker pool
  (`asyncio.to_thread`, `ProcessPoolExecutor`, or a separate worker service called via
  RPC).

- **No spec-defined concurrency limits**: The MCP spec doesn't dictate per-connection
  concurrency, request rate, or payload size limits. Profile your top-5 tools under
  realistic load and set per-tool semaphores if you hit upstream rate limits — better
  to queue inside your server than to get throttled by a third-party API.

```python
# Per-tool concurrency limit — protects an upstream that allows 10 concurrent calls
import asyncio

stripe_semaphore = asyncio.Semaphore(10)

@mcp.tool()
async def lookup_customer(customer_id: str) -> dict:
    async with stripe_semaphore:
        return await stripe_client.customers.retrieve(customer_id)
```

- **Memory per session**: For stateful mode, multiply your average per-session memory
  footprint by your target concurrent session count. If each session holds 5 MB of
  conversational state and you want to support 10k concurrent sessions, you need 50 GB
  across the fleet. This usually drives the decision to externalize state.

---

## 11. Failure Modes

- ❌ **Stateful + non-sticky LB** → 404 storm. Round-robin sends the second request to a
  different replica that has never heard of the session id. The fix is sticky sessions
  per Section 6, not retries on the client.

- ❌ **Stateless + tools that subscribe** → silent no-op. The server accepts the
  `resources/subscribe` call and returns success, but the notification it would fire
  later has nowhere to go because there's no GET stream. Clients watch nothing happen
  and assume the resource never changes.

- ❌ **Module-level mutable state in stateless mode** → cross-tenant leakage. A
  classic: caching the last-seen user id at module scope "for performance" and then
  serving that user's data to a different tenant on the next request. Module-level
  state must be immutable (config) or per-key (a dict keyed by tenant id, not a
  bare variable).

- ❌ **Counting active sessions for autoscaling** → wrong signal. Sessions can idle for
  hours while the underlying process does no work. Scale on RPS, active-call gauges, or
  CPU — not on session count, which lags real load by ages and doesn't shrink when
  traffic dies.

- ⚠️ **Termination grace period < session timeout** → kills in-flight tool calls every
  deploy. Set `terminationGracePeriodSeconds: 600+` on Kubernetes, longer if your
  sessions can run for tens of minutes. If you can't afford long drains, cap session
  TTL on the server.

- ❌ **Sharing one DB pool across thousands of concurrent stateless requests** →
  exhaust the pool. The instinct to make `max_connections=1000` per replica is wrong
  — you'll exhaust the database, not your pool. Lower per-replica pool size (10–20 is
  often right), scale replicas instead, and use a connection proxy (PgBouncer, RDS
  Proxy) in front of the database to amortize connections across the fleet.

- ⚠️ **Buffered reverse proxies eating SSE** — nginx with default `proxy_buffering on`
  will hold the SSE response until the buffer fills or the request ends. Clients see
  no events for minutes, then a flood. Always `proxy_buffering off` on the MCP location
  block, and verify with `curl -N` before declaring victory.

- ❌ **Trusting the client to send `Mcp-Session-Id` correctly** — some clients
  (especially older ones, especially when retried by middleware) drop the header on
  retry. Defensive servers log session-id misses with enough context to diagnose
  client-side bugs, rather than silently 404'ing.

---

## 12. Decision Flowchart

```
              ┌─────────────────────────────────────────┐
              │ Do your tools use subscriptions,        │
              │ live capability changes, or             │
              │ server-initiated sampling/elicitation?  │
              └────────────────────┬────────────────────┘
                       no          │          yes
                  ┌────────────────┴────────────────┐
                  ▼                                 ▼
         ┌────────────────┐              ┌────────────────────┐
         │   STATELESS    │              │     STATEFUL       │
         │ stateless_http │              │  Mcp-Session-Id    │
         │ json_response  │              │  sticky LB needed  │
         └────────┬───────┘              └─────────┬──────────┘
                  │                                │
                  ▼                                ▼
         ┌────────────────┐              ┌─────────────────────┐
         │ Any L4/L7 LB.  │              │ Single replica OK?  │
         │ Autoscale on   │              └────────┬────────────┘
         │ CPU or RPS.    │                yes    │    no
         │ Lambda / Run   │              ┌────────┴────────┐
         │ work fine.     │              ▼                 ▼
         └────────────────┘     ┌────────────────┐  ┌──────────────┐
                                │ Single pod +   │  │ Sticky LB    │
                                │ deploy with    │  │ (header hash │
                                │ rolling restart│  │  on session  │
                                └────────────────┘  │  id) +       │
                                                    │ long drain   │
                                                    └──────┬───────┘
                                                           │
                                                  cross-replica
                                                  portability needed?
                                                           │
                                                  ┌────────┴────────┐
                                                  ▼                 ▼
                                           ┌──────────┐     ┌──────────────┐
                                           │   no     │     │     yes      │
                                           │  done    │     │ Externalize  │
                                           └──────────┘     │ state: Redis │
                                                            │ + event store│
                                                            └──────────────┘
```

---

## 13. Cross-References

- Previous: [02_authentication.md — OAuth, Tokens, Auth Middleware](02_authentication.md)
- Next: [04_observability.md — OpenTelemetry, Logs, Metrics](04_observability.md)
- Related transport mechanics: [01_streamable_http.md](01_streamable_http.md)
- Transport fundamentals: [../fundamentals/05_transports.md](../fundamentals/05_transports.md)
- Lifespan resources in Python: [../python/04_context_lifespan.md](../python/04_context_lifespan.md)
- Advanced TypeScript patterns: [../typescript/05_advanced.md](../typescript/05_advanced.md)

---

**Next**: [04_observability.md — OpenTelemetry, Logs, Metrics](04_observability.md)
