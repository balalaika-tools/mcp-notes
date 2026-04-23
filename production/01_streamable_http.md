# Streamable HTTP in Production

> **Who this is for**: Operators and SREs deploying an MCP server behind a real load
> balancer (nginx, ALB, Cloudflare). Assumes you've read
> [../fundamentals/05_transports.md](../fundamentals/05_transports.md) and
> [../typescript/05_advanced.md](../typescript/05_advanced.md) and understand the JSON-RPC
> 2.0 wire format MCP rides on.

---

## 1. Why Streamable HTTP

Until protocol revision `2025-03-26`, MCP-over-HTTP required **two** endpoints: a `POST`
target for client→server requests and a separate long-lived `GET` SSE endpoint for
server→client messages. Operators hated it. It demanded coordinated routing, two sets of
CORS rules, and a fragile session-handoff dance because the SSE endpoint had to know
which POST endpoint a session belonged to.

**Streamable HTTP collapses this into a single URL** that multiplexes everything by HTTP
method:

```
            ┌──────────────────────────────────────────┐
            │   https://api.example.com/mcp/           │
            ├──────────────────────────────────────────┤
   POST  ──▶│  client → server JSON-RPC                │
   GET   ──▶│  open server-push SSE stream             │
   DELETE──▶│  terminate session                       │
   OPTIONS─▶│  CORS preflight (browser hosts only)     │
            └──────────────────────────────────────────┘
```

A single POST may return either a plain JSON response **or** an SSE stream — the server
chooses, based on whether it needs to emit progress notifications before the final
result. This means quick, stateless calls cost nothing extra (one request, one JSON
response, connection closed), while long-running or interactive calls upgrade transparently.

> **Key insight**: Streamable HTTP is not "SSE for MCP." It's HTTP with an *optional*
> SSE upgrade per response. Most of your traffic will be plain JSON — the SSE machinery
> only kicks in when the server has something to stream.

---

## 2. The Endpoint Contract

A Streamable HTTP server exposes **one URL**. Convention is a trailing slash
(`/mcp/`), but the spec doesn't require it — pick one and stick with it; redirects
across the SSE upgrade boundary are a footgun.

| Method    | Required? | Purpose                                                                    |
|-----------|-----------|----------------------------------------------------------------------------|
| `POST`    | MUST      | Client sends one JSON-RPC request, response, or notification               |
| `GET`     | MAY       | Client opens long-lived SSE stream for server-initiated traffic            |
| `DELETE`  | MAY       | Client asks the server to terminate a session explicitly                   |
| `OPTIONS` | MAY       | CORS preflight; required if browser-based hosts will connect               |

Every request from a client at protocol version `2025-06-18` or later **must** carry an
`Mcp-Protocol-Version` header set to the version negotiated during `initialize`. Older
clients omit it; servers should fall back to `2025-03-26` semantics in that case.

```
Mcp-Protocol-Version: 2025-06-18
```

⚠️ A surprising number of bugs trace back to clients that send the header on the first
POST but drop it on the GET stream. Tighten your server to require it on every request
once it has been seen — otherwise you'll silently downgrade and lose features like
elicitation.

---

## 3. Request — POST

This is the workhorse. Client sends a JSON-RPC payload; server responds with one of
three shapes.

**Required headers from the client**:

```
Content-Type: application/json
Accept: application/json, text/event-stream
```

The dual `Accept` is non-negotiable — even if the client doesn't expect streaming, it
**must** advertise both, because the server picks the response media type. A client
that sends only `application/json` is buggy and well-behaved servers will reject it
with `406 Not Acceptable`.

**Body**: a single JSON-RPC request, response, or notification.

### 3.1 Server response: plain JSON

For quick, synchronous tool calls with no progress reporting:

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 142

{"jsonrpc":"2.0","id":7,"result":{"content":[{"type":"text","text":"42"}]}}
```

This is the cheapest path. No SSE framing, no chunked encoding necessary, no long-lived
connection. Use it whenever you can — it's friendlier to every intermediary in the
network path.

### 3.2 Server response: SSE stream

When the server wants to emit progress notifications before the final result:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

id: 1
data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"t-1","progress":1,"total":3}}

id: 2
data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"t-1","progress":2,"total":3}}

id: 3
data: {"jsonrpc":"2.0","id":7,"result":{"content":[{"type":"text","text":"done"}]}}

```

The stream ends when the server sends the final response (matching the request `id`)
and closes the connection. The client knows it's done because the response correlates
with its outstanding request.

### 3.3 Server response: 202 Accepted

For payloads that don't require a response — pure notifications and pure responses:

```
HTTP/1.1 202 Accepted
Content-Length: 0
```

The classic example is `notifications/initialized`, which the client sends after
processing `InitializeResult`. There's nothing to stream, nothing to return.

> **Rule**: A `202` response **must** have an empty body. If you send a payload, clients
> that expect strict adherence will treat it as a protocol error.

---

## 4. Server-Push Stream — GET

The GET stream is how the server reaches *into* the client unprompted: sampling
requests, elicitation prompts, `roots/list` queries, and `notifications/*` events
(`tools/list_changed`, `resources/updated`, `prompts/list_changed`, log messages).

```
GET /mcp/ HTTP/1.1
Host: api.example.com
Accept: text/event-stream
Mcp-Session-Id: 9f2a-4c8e-...
Mcp-Protocol-Version: 2025-06-18
```

The server responds with `200 OK`, `Content-Type: text/event-stream`, and **holds the
connection open indefinitely**. SSE events flow whenever the server has something to
say:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

id: evt-001
data: {"jsonrpc":"2.0","method":"notifications/tools/list_changed"}

id: evt-002
data: {"jsonrpc":"2.0","id":"srv-1","method":"sampling/createMessage","params":{...}}

```

Each SSE event has three useful fields:

| Field   | Purpose                                                              |
|---------|----------------------------------------------------------------------|
| `id:`   | Resumption token. Client persists the most recent value.             |
| `event:`| Optional event name. MCP rarely uses this; defaults to `message`.    |
| `data:` | The JSON-RPC payload. Multi-line `data:` is concatenated with `\n`.  |

If the GET stream is the channel for a server→client request (like `sampling/createMessage`),
the client's response goes back via a **POST** with the JSON-RPC response in the body.
The GET stream is unidirectional — POSTs carry everything client→server.

⚠️ A client may open at most **one** active GET stream per session. Some servers tolerate
multiple; treat it as an anti-pattern. If you need fan-out, do it inside the client.

---

## 5. Sessions — `Mcp-Session-Id`

Sessions are *opt-in* and live entirely in one HTTP header.

**Stateful mode** (most servers):

1. Client sends `initialize` POST.
2. Server returns `InitializeResult` plus an `Mcp-Session-Id: <uuid-or-jwt>` response
   header.
3. Client echoes that header on **every** subsequent request (POST, GET, DELETE).
4. If the server can't find the session (expired, restarted, wrong replica), it returns
   `404 Not Found`. The client must re-initialize from scratch.

```
← HTTP/1.1 200 OK
  Mcp-Session-Id: 9f2a4c8e-7b1d-4e6a-90c2-1a3b5f8d2e1c
  Content-Type: application/json
  ...

→ POST /mcp/
  Mcp-Session-Id: 9f2a4c8e-7b1d-4e6a-90c2-1a3b5f8d2e1c
  ...
```

**Stateless mode**:

The server does not return `Mcp-Session-Id` on initialize. Every request stands alone —
no session-bound state, no GET stream is meaningful, and any request can be load-balanced
to any replica without sticky routing. This is the right choice for pure tool-execution
servers that don't need server-initiated traffic. See
[03_scaling.md](03_scaling.md) for when to pick which mode.

> **Key insight**: The session ID is opaque to the client. Servers commonly use a UUID,
> but a signed JWT works too — and lets stateless replicas validate sessions without a
> shared store. The client doesn't care.

💡 The session ID **must** be ASCII-safe and reasonably short. Clients and proxies
sometimes have header-size limits around 8 KB total; a 7 KB JWT is asking for trouble.

---

## 6. Resumability with `Last-Event-ID`

SSE has a built-in resumption protocol. The MCP spec layers on top of it.

```
Client                          Server
  │                                │
  │ GET /mcp/  Accept: SSE         │
  │ Mcp-Session-Id: 9f2a-...       │
  │───────────────────────────────▶│
  │                                │
  │  id: 100  data: {...}          │
  │◀────────────────────────────── │
  │  id: 101  data: {...}          │
  │◀───────────────────────────────│
  │                                │
  │     ✗ network drops            │
  │                                │
  │ GET /mcp/  Accept: SSE         │
  │ Mcp-Session-Id: 9f2a-...       │
  │ Last-Event-ID: 101             │
  │──────────────────────────────▶ │
  │                                │
  │ id: 102  data: {...}← replayed │
  │◀───────────────────────────────│
  │ id: 103  data: {...}           │
  │◀───────────────────────────────│
```

**Server obligations**:

- Assign a monotonic `id:` to every event.
- On reconnect, parse `Last-Event-ID` and replay events with `id` greater than that value.
- Persist enough event history that resumption is meaningful. Common windows: 5 minutes
  or 1000 events, whichever is smaller.

**Operational reality**: only some implementations support this. The TypeScript SDK
ships an `InMemoryEventStore`; the Python SDK ships an `EventStore` interface but no
default. **Single-replica deployments** can use in-memory storage. **Multi-replica
deployments** need a shared store — Redis Streams is the typical choice because it
gives you the right primitive (ordered, ID-addressable log) for free.

❌ Don't try to use a relational table for the event store on a hot path. The write
amplification will eat you alive — every notification becomes an `INSERT`. Use
something log-shaped.

---

## 7. Reverse-Proxy Gotchas

This is where most production deployments break. Reverse proxies and load balancers
were designed for short-lived request/response cycles; SSE violates every assumption.

### 7.1 Disable response buffering for SSE

If the proxy buffers the response body, your "real-time" stream arrives as one batch
when the response closes. Useless.

**nginx**:

```nginx
location /mcp/ {
    proxy_pass http://mcp_upstream;
    proxy_http_version 1.1;
    proxy_set_header Connection "";

    # Critical for SSE
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

It's also good practice to set the response header from the application:

```
X-Accel-Buffering: no
```

Nginx honors this per-response, so you don't have to disable buffering globally if you
have a mixed workload.

**Cloudflare**: Cloudflare buffers by default and caps free-tier response time around
100 seconds. Streaming responses are supported on paid plans, but **long-lived GET
streams are still an awkward fit**. For real push semantics through Cloudflare, route
the GET stream through a Cloudflare Tunnel rather than the standard proxy — the tunnel
treats the connection as opaque TCP and won't impose HTTP-level timeouts.

**AWS ALB**: ALB supports HTTP/1.1 streaming responses but enforces an idle timeout
(default 60 seconds). Bump it to the maximum (4000 seconds) for the listener handling
MCP traffic, or use a dedicated listener for `/mcp/` so unrelated traffic isn't affected.

### 7.2 Idle timeouts

The GET stream is, by design, idle most of the time. If your LB closes idle connections
at 60 seconds, every stream will get killed every minute and your client will reconnect
in a hot loop.

| Proxy / LB         | Setting to extend                                    | Recommended value |
|--------------------|------------------------------------------------------|-------------------|
| nginx              | `proxy_read_timeout`                                 | `3600s` or higher |
| AWS ALB            | Idle timeout (per LB or per listener attribute)      | `4000s` (max)     |
| Cloudflare         | Plan-dependent; use Tunnels for unbounded            | n/a               |
| GCP Load Balancer  | Backend service `timeoutSec`                         | `3600` or higher  |
| Envoy / Istio      | `route.timeout` and `idle_timeout`                   | `0s` (disabled)   |

Pair this with a **server-side keepalive**: have the server emit an SSE comment line
(`: keepalive\n\n`) every 15–30 seconds. It's invisible to the client but resets every
intermediary's idle counter.

### 7.3 HTTP/2 and HTTP/3

Both work fine for Streamable HTTP. The pitfalls are:

- ⚠️ HTTP/2 servers sometimes coalesce small responses into a single frame; ensure
  your runtime flushes on every SSE event boundary.
- ⚠️ Some intermediaries downgrade to HTTP/1.1 when they see `text/event-stream` and
  buffer the result. Test end-to-end, not just at the origin.
- ✅ HTTP/2 multiplexing means a client can run the GET stream and POST requests on
  the same TCP connection, which is a real efficiency win.

### 7.4 Sticky sessions

If you're running stateful mode behind multiple replicas, you **must** route every
request for a given session to the same replica. The session state lives in memory
(or in a per-replica cache); a request that lands on the wrong replica gets a `404`.

Strategies, in order of preference:

1. **External session store + stateless routing**: store session state in Redis, look
   it up on every request. Best for elasticity.
2. **Sticky sessions by header**: route based on `Mcp-Session-Id`. Most LBs only sticky
   on cookies, so this often requires a custom hash policy (Envoy, HAProxy can do it).
3. **Cookie-based stickiness**: the LB issues a cookie on the initialize response and
   the client echoes it. Browsers do this for free; non-browser clients usually don't.

See [03_scaling.md](03_scaling.md) for the full treatment.

---

## 8. CORS for Browser-Based Hosts

If your server will be reached from a browser-resident MCP host (claude.ai, the
ChatGPT web app, embeddable browser MCP UIs), CORS configuration is mandatory.

```
Access-Control-Allow-Origin: https://claude.ai
Access-Control-Allow-Methods: POST, GET, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, Mcp-Session-Id, Mcp-Protocol-Version
Access-Control-Expose-Headers: Mcp-Session-Id
Access-Control-Max-Age: 86400
```

Things to get right:

- **`Allow-Origin`**: list specific origins, not `*`. Browser hosts send credentials
  (cookies, `Authorization`), and credentialed CORS requests forbid wildcards.
- **`Allow-Headers`**: must include `Mcp-Session-Id` and `Mcp-Protocol-Version` —
  these are non-standard headers, so they're not in the CORS-safelisted set.
- **`Expose-Headers: Mcp-Session-Id` is the one nobody remembers.** Browsers hide
  every response header from JavaScript except a tiny safelisted set. The client
  needs to read `Mcp-Session-Id` to echo it on subsequent requests, and it can't
  unless you explicitly expose it.
- **`Max-Age`**: 24 hours of preflight caching is fine and saves a lot of `OPTIONS`
  round-trips.

❌ Setting `Access-Control-Allow-Origin: *` *and* sending session cookies will silently
fail in the browser with no obvious error. Always pin to specific origins.

---

## 9. TLS, HTTP Versions, and the `Origin` Header

The MCP spec requires servers to **validate the `Origin` header on POST requests** when
running locally. This guards against DNS-rebinding attacks: a malicious page sets DNS
to resolve `evil.example` to `127.0.0.1`, then makes requests to your localhost MCP
server hoping to extract data.

For local development:

```python
ALLOWED_ORIGINS = {"http://localhost:5173", "https://claude.ai"}

def validate_origin(request):
    origin = request.headers.get("Origin")
    if origin is not None and origin not in ALLOWED_ORIGINS:
        raise HTTPException(403, "Origin not allowed")
```

Other local-only hardening:

- ✅ Bind to `127.0.0.1`, never `0.0.0.0`. Even with `Origin` checks, an open bind
  is a public attack surface.
- ✅ For TCP-only local servers, prefer Unix domain sockets when possible.

For production:

- ✅ TLS everywhere. Reject plain HTTP at the edge — `301` to HTTPS.
- ✅ Modern cipher suites. Disable TLS 1.0/1.1.
- ⚠️ If you terminate TLS at the LB and run plain HTTP between LB and origin, ensure
  the network between them is private. Set `X-Forwarded-Proto: https` so the app
  knows the original scheme for `Origin` validation and OAuth redirect URLs.

---

## 10. End-to-End Wire Trace

A complete annotated session — initialize, notification, GET stream open, tool call
with progress, session termination.

```
# 1. Initialize — first POST establishes the session
→ POST /mcp/ HTTP/1.1
  Host: api.example.com
  Content-Type: application/json
  Accept: application/json, text/event-stream

  {"jsonrpc":"2.0","id":1,"method":"initialize","params":{
    "protocolVersion":"2025-06-18",
    "capabilities":{"sampling":{}},
    "clientInfo":{"name":"my-host","version":"1.0.0"}
  }}

← HTTP/1.1 200 OK
  Mcp-Session-Id: 9f2a4c8e-7b1d-4e6a-90c2-1a3b5f8d2e1c
  Content-Type: application/json

  {"jsonrpc":"2.0","id":1,"result":{
    "protocolVersion":"2025-06-18",
    "capabilities":{"tools":{"listChanged":true},"resources":{}},
    "serverInfo":{"name":"example-mcp","version":"2.3.0"}
  }}

# 2. Initialized notification — no body in response
→ POST /mcp/ HTTP/1.1
  Mcp-Session-Id: 9f2a4c8e-7b1d-4e6a-90c2-1a3b5f8d2e1c
  Mcp-Protocol-Version: 2025-06-18
  Content-Type: application/json
  Accept: application/json, text/event-stream

  {"jsonrpc":"2.0","method":"notifications/initialized"}

← HTTP/1.1 202 Accepted
  Content-Length: 0

# 3. Open the server-push stream — held open indefinitely
→ GET /mcp/ HTTP/1.1
  Mcp-Session-Id: 9f2a4c8e-7b1d-4e6a-90c2-1a3b5f8d2e1c
  Mcp-Protocol-Version: 2025-06-18
  Accept: text/event-stream

← HTTP/1.1 200 OK
  Content-Type: text/event-stream
  Cache-Control: no-cache

  : keepalive

  (server pushes notifications and server-initiated requests on this stream)

# 4. Tool call that streams progress — server picks SSE response
→ POST /mcp/ HTTP/1.1
  Mcp-Session-Id: 9f2a4c8e-7b1d-4e6a-90c2-1a3b5f8d2e1c
  Mcp-Protocol-Version: 2025-06-18
  Content-Type: application/json
  Accept: application/json, text/event-stream

  {"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
    "name":"index_documents",
    "arguments":{"path":"/data/2026"},
    "_meta":{"progressToken":"abc"}
  }}

← HTTP/1.1 200 OK
  Content-Type: text/event-stream

  id: 1
  data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"abc","progress":1,"total":3}}

  id: 2
  data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"abc","progress":2,"total":3}}

  id: 3
  data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"abc","progress":3,"total":3}}

  id: 4
  data: {"jsonrpc":"2.0","id":2,"result":{"content":[{"type":"text","text":"Indexed 412 documents."}]}}

# 5. Explicit session termination
→ DELETE /mcp/ HTTP/1.1
  Mcp-Session-Id: 9f2a4c8e-7b1d-4e6a-90c2-1a3b5f8d2e1c
  Mcp-Protocol-Version: 2025-06-18

← HTTP/1.1 200 OK
```

Note that step 3 (the GET) and step 4 (the SSE-bearing POST) are **two separate
streams**. Both are `text/event-stream`, but the GET is the persistent server-push
channel while the POST stream is request-scoped and ends when the response is delivered.

---

## 11. Failure Modes

The catalog of things that have personally cost me on-call hours.

❌ **Buffered reverse proxy**

Symptom: progress notifications all arrive at once, right when the final response
lands. Cause: proxy buffering the SSE body. Fix: `proxy_buffering off;` (nginx),
`X-Accel-Buffering: no` from the app, equivalent on your LB. Test with `curl -N`
straight through the proxy.

❌ **LB idle timeout shorter than stream lifetime**

Symptom: the GET stream silently closes every N seconds; clients reconnect in a tight
loop; high reconnection rate but no errors logged at the origin. Cause: LB idle
timeout. Fix: bump idle timeout to >300s (ideally 3600s+) **and** emit server-side
keepalives every 15–30 seconds.

❌ **Missing `Expose-Headers: Mcp-Session-Id`**

Symptom: browser-based client can't establish a session; every request after
`initialize` returns `404`. Cause: the browser receives the `Mcp-Session-Id` header but
hides it from JS. Fix: add `Access-Control-Expose-Headers: Mcp-Session-Id`.

❌ **Server returns plain JSON when client sent only `Accept: text/event-stream`**

Symptom: client throws "unexpected content type" and disconnects. Cause: usually a
buggy client (it should advertise both), but also a server that ignores `Accept`. Fix:
client should always send `Accept: application/json, text/event-stream`; server should
honor whatever the client offered, or `406` if it can't.

❌ **Server returns SSE when client only accepts `application/json`**

Same root cause, opposite direction. The server picked SSE for streaming progress, but
the client never advertised SSE support. Fix: the client *must* advertise both. Reject
clients that don't with `406 Not Acceptable` so the bug surfaces immediately rather than
festering.

⚠️ **Stateful mode + no sticky sessions → 404 storm**

Symptom: clients work fine for the first few requests, then start getting `404`s
randomly as their session-bound requests hit replicas without that session. Fix: either
sticky sessions on `Mcp-Session-Id`, or move session state to a shared store. See
[03_scaling.md](03_scaling.md).

❌ **Forgetting `Mcp-Protocol-Version` on subsequent requests**

Symptom: works against permissive servers, fails with `400` against strict ones. Fix:
once `initialize` returns a negotiated version, send it on every request thereafter.
Most SDKs do this automatically; the bug usually appears in hand-rolled clients.

⚠️ **Returning `200 OK` on `DELETE` after the session is already gone**

Not strictly wrong, but `204 No Content` or `404` is more honest. A client that called
`DELETE` twice (e.g., on retry) shouldn't infer that the second one closed something
that wasn't there.

⚠️ **Not flushing SSE writes**

Subtle Python and Node bug: writing to the response stream doesn't always flush. In
Starlette, use `StreamingResponse` with an async generator. In Node, call
`response.flush()` if available, or `response.write()` returns false (apply
backpressure). Without flushing, events sit in the framework's output buffer.

---

**Next**: [02_authentication.md — OAuth 2.1, PKCE, RFC 9728](02_authentication.md)
