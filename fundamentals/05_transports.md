# Transports: How MCP Messages Move on the Wire

> **Who this is for**: Engineers who've read [02_architecture.md](02_architecture.md) and
> [04_lifecycle.md](04_lifecycle.md) and want to understand the actual byte-level transport
> options before implementing or operating a server. We assume you already know what
> JSON-RPC 2.0 looks like — this file is about the physical channel that carries it.

---

## 1. Why Transports Matter

Every MCP message is a JSON-RPC 2.0 envelope. That part is fixed. What changes between
deployments is **how those bytes travel** — over a Unix pipe, over HTTP, over a streaming
SSE channel. The choice has nothing to do with what your tools do and everything to do with
where they run and who can talk to them.

Transport choice affects four things you'll actually feel in production:

- **Deployment topology** — same host as the client, or behind a load balancer?
- **Scaling** — one process per user, or one process serving thousands?
- **Multi-tenancy** — can multiple clients share a server, and how do you isolate them?
- **Debugging** — can you `tcpdump` it, or are you tailing a subprocess's stderr?

The 2025-11-25 spec defines two transports, plus one legacy transport that's on its way
out:

| Transport          | When                  | Streaming                  | Multi-client | Status            |
|--------------------|-----------------------|----------------------------|--------------|-------------------|
| stdio              | Local subprocess      | Inherent (it's a pipe)     | No (1:1)     | Stable            |
| Streamable HTTP    | Remote / hosted       | Optional via SSE chunks    | Yes          | **Recommended**   |
| HTTP+SSE (legacy)  | Legacy remote         | Yes (SSE channel)          | Yes          | **Deprecated**    |

> **Key insight**: The JSON-RPC payload — `{"jsonrpc": "2.0", "method": "tools/call", ...}` —
> is byte-identical across all three transports. If you've written your handler logic
> against one transport, swapping to another is mostly a packaging concern, not a protocol
> rewrite.

---

## 2. stdio Transport

The simplest transport, and the one almost every desktop MCP integration uses today
(Claude Desktop, Cursor, Zed, the official Inspector). The client process **launches the
server as a subprocess** and talks to it over stdin/stdout.

### 2.1 Wire format

JSON-RPC messages are **line-delimited UTF-8 JSON**. One message per line, terminated by a
single `\n`. No framing headers, no length prefix — newlines are the delimiter, which means
your messages **must not contain unescaped newlines** (standard JSON serialization handles
this for you).

A `tools/call` request looks like this on the wire (one physical line, wrapped here for
readability):

```
{"jsonrpc":"2.0","id":42,"method":"tools/call","params":{"name":"search_docs","arguments":{"query":"context manager"}}}\n
```

The server's response on stdout, also one line:

```
{"jsonrpc":"2.0","id":42,"result":{"content":[{"type":"text","text":"Found 3 matches..."}]}}\n
```

### 2.2 stderr is for logging

This is the part everyone gets wrong on the first try. **stdout is sacred** — anything
written there must be a valid JSON-RPC message. If your server prints `"Server starting..."`
to stdout, you've corrupted the protocol stream and the client will hang or crash.

The 2025-11-25 spec explicitly broadened the stderr definition: it used to say "errors
only," but it now permits **any structured logging** — info, debug, warnings, anything you
want. The client typically captures stderr and surfaces it in a log panel for you.

```python
# ❌ Wrong — corrupts the JSON-RPC stream
print("Loading tools...")  # goes to stdout

# ✅ Correct — logging goes to stderr
import sys
print("Loading tools...", file=sys.stderr)

# ✅ Better — use logging configured to stderr
import logging
logging.basicConfig(stream=sys.stderr, level=logging.INFO)
log = logging.getLogger(__name__)
log.info("Loading tools...")
```

### 2.3 Pros and cons

**Pros:**

- Zero network setup. No ports, no TLS certificates, no firewall rules.
- Secure by default — there's no socket for an attacker to reach.
- Process lifetime is tied to the client. Client dies, subprocess dies (if you handle
  SIGTERM correctly).
- Trivially debuggable: pipe stdin/stdout to files and replay them.
- Perfect for CLI tools, IDE integrations, and anything that runs on the user's machine.

**Cons:**

- **One client per server process.** stdio is intrinsically 1:1.
- Doesn't scale beyond a single host — there's no way for a server in another datacenter
  to send you bytes over your stdin.
- Each client launch incurs subprocess startup cost (Python interpreter, Node runtime,
  etc.). For heavy servers this is noticeable.
- Hard to share state across clients without an external store, because each client gets
  its own process.

### 2.4 When to choose stdio

```
Are your users going to install your server locally?      → stdio
Is this an IDE or terminal integration?                   → stdio
Do you need zero-config security?                         → stdio
Are you exposing the server to untrusted callers?         → NOT stdio (use HTTP)
```

> **Rule**: If you can't articulate why your server needs to be remote, ship it as stdio.
> The vast majority of MCP servers in the wild are stdio-based, and that's not an accident.

---

## 3. Streamable HTTP Transport

The recommended transport for **remote / hosted** servers. Introduced to replace the
legacy two-endpoint HTTP+SSE design with a single, cleaner endpoint that supports both
unary requests and streamed responses.

### 3.1 The single endpoint model

One URL — for example `https://api.example.com/mcp` — handles every interaction. The HTTP
verb tells the server what kind of interaction it is:

| Method  | Purpose                                                      |
|---------|--------------------------------------------------------------|
| POST    | Client sends a JSON-RPC request (or notification, or batch)  |
| GET     | Client opens an SSE stream for server-initiated messages     |
| DELETE  | Client explicitly terminates a session                       |

This is much simpler to deploy than the legacy two-endpoint design — it's just one path
on your load balancer, one route in your reverse proxy, one rule in your auth middleware.

### 3.2 POST: client requests

The client POSTs a JSON-RPC envelope. The crucial header is `Accept`, which tells the
server **which response shapes the client can handle**:

```http
POST /mcp HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json, text/event-stream
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Mcp-Session-Id: 4f8a2b91-9c1e-4d7a-8b3f-2e6a1c8d5f90

{"jsonrpc":"2.0","id":17,"method":"tools/call","params":{"name":"summarize","arguments":{"url":"https://example.com/long-doc"}}}
```

The server then picks one of two response shapes:

**Shape A: `application/json`** — single response, best for simple stateless calls:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"jsonrpc":"2.0","id":17,"result":{"content":[{"type":"text","text":"Summary: ..."}]}}
```

**Shape B: `text/event-stream`** — server streams progress notifications and the final
response over a single SSE connection. Use this when the call is long-running and you want
to push `notifications/progress` updates:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"t1","progress":0.25}}

data: {"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"t1","progress":0.75}}

data: {"jsonrpc":"2.0","id":17,"result":{"content":[{"type":"text","text":"Summary: ..."}]}}

```

The stream closes after the response with the matching `id` is delivered. Each `data:`
line is one JSON-RPC message; blank lines separate SSE events per the SSE spec.

> **Key insight**: SSE here is just a chunked HTTP response with a specific `Content-Type`.
> It works through any HTTP/1.1 or HTTP/2 proxy that supports streaming. No WebSocket
> upgrade dance, no special infrastructure.

### 3.3 GET: server-initiated messages

Some MCP features need the **server to push messages to the client without being asked** —
sampling requests, elicitation prompts, `notifications/tools/list_changed`, etc. For that,
the client opens a long-lived SSE stream:

```http
GET /mcp HTTP/1.1
Host: api.example.com
Accept: text/event-stream
Mcp-Session-Id: 4f8a2b91-9c1e-4d7a-8b3f-2e6a1c8d5f90
```

The server keeps this connection open and writes events as they arise:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"jsonrpc":"2.0","id":"s-9","method":"sampling/createMessage","params":{...}}

data: {"jsonrpc":"2.0","method":"notifications/tools/list_changed"}

```

When the client wants to respond to a server-initiated request, it issues a **separate
POST** with the matching `id` — the GET stream is one-way (server → client), and the
client uses normal POST for its replies.

### 3.4 DELETE: terminating a session

When the client is done, it can release server-side resources cleanly:

```http
DELETE /mcp HTTP/1.1
Host: api.example.com
Mcp-Session-Id: 4f8a2b91-9c1e-4d7a-8b3f-2e6a1c8d5f90
```

The server should drop any per-session state. This is optional — sessions can also expire
on idle — but it's the polite thing to do when shutting down a client.

### 3.5 Sessions and the `Mcp-Session-Id` header

Sessions are **opt-in**. The flow:

1. Client sends `initialize` request **without** a session header.
2. Server may respond with `Mcp-Session-Id: <uuid>` in the response headers.
3. If the server included a session id, the client **must echo it** on every subsequent
   request. If the server didn't, the client doesn't send one — the server is stateless.

```
┌─────────┐                               ┌─────────┐
│ Client  │                               │ Server  │
└────┬────┘                               └────┬────┘
     │                                         │
     │── POST initialize ────────────────────> │
     │   (no session header)                   │
     │                                         │
     │<── 200 OK ──────────────────────────────│
     │   Mcp-Session-Id: 4f8a...               │
     │   {result: {...}}                       │
     │                                         │
     │── POST tools/call ────────────────────> │
     │   Mcp-Session-Id: 4f8a...  ← echoed     │
     │                                         │
     │<── 200 OK + JSON or SSE ────────────────│
     │                                         │
     │── DELETE /mcp ────────────────────────> │
     │   Mcp-Session-Id: 4f8a...               │
     │                                         │
     │<── 204 No Content ──────────────────────│
     ▼                                         ▼
```

### 3.6 Stateless mode

If the server doesn't issue a session id, every POST is independent. This is the easy mode
for horizontal scaling — any pod behind your load balancer can handle any request. No
sticky sessions, no shared cache, no Redis-backed session store. Just stateless JSON-RPC
over HTTP.

The tradeoff: features that genuinely need server-side state (long-lived sampling
conversations, ongoing subscriptions) won't work cleanly in stateless mode. Pick stateless
when you can; reach for sessions only when you need them.

> **Foreshadow**: Production-grade Streamable HTTP — TLS, auth, request limits, retries,
> session storage, scale-out — gets a full file at
> [`../production/01_streamable_http.md`](../production/01_streamable_http.md). This
> section is the protocol-level summary; that file is the operations playbook.

---

## 4. Legacy HTTP+SSE Transport (Deprecated)

The original remote transport, defined in earlier MCP drafts. **Don't build on it.** It
still exists in some servers and clients, so you should know what it looks like to
recognize it in the wild.

### 4.1 The two-endpoint design

HTTP+SSE used **two separate endpoints**:

```
┌─────────┐                               ┌─────────┐
│ Client  │                               │ Server  │
└────┬────┘                               └────┬────┘
     │                                         │
     │── GET /sse ───────────────────────────> │  open SSE stream
     │<── data: endpoint info ─────────────────│
     │                                         │
     │── POST /messages ─────────────────────> │  send JSON-RPC requests
     │<── 202 Accepted ────────────────────────│
     │                                         │
     │<── data: {jsonrpc response} ────────────│  responses come back over SSE
     ▼                                         ▼
```

Requests went to one URL (POST), responses came back asynchronously over a different URL
(GET SSE). The server gave the client a "messages endpoint" URL via the SSE channel during
setup.

### 4.2 Why it was replaced

- **Two endpoints, two routes, two sets of auth rules.** Awkward to deploy, easy to
  misconfigure.
- **Bidirectional flow was clunky.** Correlating a POST request with its SSE response
  required matching `id`s across two physical channels.
- **No clean stateless mode.** The SSE channel was inherently long-lived, which made
  load-balancing painful.
- **Operational opacity.** Monitoring a request meant correlating logs across two
  endpoints.

The MCP spec's "Future of Transports" post (December 2025) was unambiguous: Streamable
HTTP is the path forward. HTTP+SSE is **deprecated**, present only for backward
compatibility.

### 4.3 Migration

If you have an existing HTTP+SSE server:

- Plan a migration to Streamable HTTP. Most server SDKs (Python `mcp`, TypeScript
  `@modelcontextprotocol/sdk`) have implementations of both, so the change is often
  swapping a constructor.
- During the transition, you can serve both transports from the same process if your
  framework supports it — old clients hit the legacy endpoints, new clients hit
  `/mcp`.
- New servers in 2026 should not implement HTTP+SSE at all.

---

## 5. Selection Matrix

When in doubt, walk this list top to bottom and stop at the first match:

```
Are you building a CLI/IDE plugin meant to run locally?  → stdio
Are you exposing a multi-tenant remote API?              → Streamable HTTP (stateless if possible)
Do you need server push (sampling, live notifications)?  → Streamable HTTP (with GET stream)
Do you have an existing HTTP+SSE server?                 → Plan migration to Streamable HTTP
Are you considering WebSockets?                          → See section 6 — probably not
```

The decision is rarely close. stdio for local, Streamable HTTP for remote — that covers
99% of what you'll build.

---

## 6. What About WebSockets?

WebSockets are **not currently a standard MCP transport.** The 2025-11-25 spec defines
stdio and Streamable HTTP, full stop.

The spec authors' "Future of Transports" post (December 2025) discussed WebSockets as a
possible future addition — they're attractive for symmetric, low-overhead duplex
communication, especially for browser-based clients. But the current consensus is that
Streamable HTTP's POST + GET SSE pattern is good enough:

- Bidirectional? Covered (POST for client→server, GET SSE for server→client).
- Streaming? Covered (SSE response shape on POST).
- Standards-compliant? Yes — works through any HTTP-aware proxy.
- Browser support? Native (fetch + EventSource).

Adding WebSockets would mean a third transport with its own framing, its own auth model,
its own infrastructure requirements (some proxies don't speak the upgrade dance). The
spec authors decided the marginal benefit didn't justify the protocol surface.

If you're building a browser-based MCP client today, use Streamable HTTP. If WebSockets
land in a future spec revision, the JSON-RPC payload won't change.

---

## 7. Common Mistakes

❌ **Writing log lines to stdout instead of stderr in a stdio server.** This corrupts the
JSON-RPC framing. The client either disconnects or, worse, silently misroutes responses.
Always write logs to stderr. If you use `print()` or `console.log()` anywhere in a stdio
server's hot path, fix it before shipping.

❌ **Forgetting `Accept: application/json, text/event-stream` on Streamable HTTP
requests.** A spec-compliant server that wanted to stream a long response will refuse to do
so if you only sent `Accept: application/json`. Always advertise both — the server picks.

❌ **Building new servers on HTTP+SSE in 2026.** It's deprecated. Every minute you spend
on it is a minute you'll spend migrating later. Use Streamable HTTP from day one.

❌ **Holding open a stdio subprocess across user requests in a "server" sense.** stdio is
fundamentally **one client, one subprocess, one session.** If you're trying to multiplex
multiple clients over one stdio process, you've reinvented HTTP poorly. Use Streamable
HTTP.

⚠️ **Sharing an `Mcp-Session-Id` across two HTTP clients.** Sessions are per-client. The
server's session state — initialization params, capability negotiation, subscriptions —
belongs to one client. If two clients share the id, you'll see racy initialization, lost
notifications, and very confusing bug reports. Each client must initialize its own session.

⚠️ **Forgetting that GET /mcp is a long-lived SSE stream.** Some load balancers idle-time
out connections after 30–60 seconds. If your clients are mysteriously losing
server-initiated notifications, check your LB's idle timeout — you'll need to extend it
or implement a keepalive (SSE comments — `: ping\n\n` — work fine).

---

## 8. Putting It Together

Here's the same `tools/call` request expressed in all three transports, side by side.
Notice that **only the framing changes** — the JSON-RPC envelope is identical:

**stdio:**

```
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ping","arguments":{}}}\n
```

**Streamable HTTP:**

```http
POST /mcp HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ping","arguments":{}}}
```

**HTTP+SSE (legacy):**

```http
POST /messages HTTP/1.1
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ping","arguments":{}}}
```

(With the response coming back later over a separate `GET /sse` channel.)

The protocol is the protocol. Pick the transport that matches your deployment and move on.

---

## 9. Cross-References

- **Previous**: [04_lifecycle.md — Connection Lifecycle](04_lifecycle.md)
- **Next in fundamentals**: this is the last fundamentals file
- **Build something**: [`../python/01_getting_started.md`](../python/01_getting_started.md)
- **Production depth on remote transports**: [`../production/01_streamable_http.md`](../production/01_streamable_http.md)
- **Related**:
  - [`../production/02_authentication.md`](../production/02_authentication.md) — HTTP auth, OAuth, bearer tokens
  - [`../production/03_scaling.md`](../production/03_scaling.md) — session stickiness, horizontal scale-out

---

**Next**: [Python: Getting Started](../python/01_getting_started.md) — or jump to [Production: Streamable HTTP](../production/01_streamable_http.md) for the deep dive on remote transports.
