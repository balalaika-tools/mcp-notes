# MCP Architecture: Hosts, Clients, Servers, and JSON-RPC

> **Who this is for**: Engineers comfortable with HTTP, JSON, and basic distributed systems
> who have read [01_what_is_mcp.md](01_what_is_mcp.md) and want the precise vocabulary —
> what a Host is, what a Client is, what shows up on the wire, and how a single tool call
> actually travels from an LLM application to a remote process and back.

This file defines the nouns (Host, Client, Server, Capability, Session) and verbs
(`initialize`, `tools/call`, `notifications/initialized`) used throughout the rest of the
notes. Everything downstream — primitives, lifecycle, transports — assumes you've internalized
the terms here.

> **Spec version referenced**: `2025-11-25` (current as of April 2026).

---

## 1. The Three Roles

MCP defines three roles. They are roles, not necessarily separate processes — a single
binary can play two roles at once — but the protocol always speaks in terms of these three.

| Role       | What it is                                                                 | Examples                                       |
|------------|-----------------------------------------------------------------------------|------------------------------------------------|
| **Host**   | The LLM application. Owns the model, the user, and the conversation.       | Claude Desktop, VS Code, Cursor, Zed, custom agents |
| **Client** | A protocol connector inside the Host. **One Client per connected Server.** | An in-process object managed by the Host       |
| **Server** | A process that exposes capabilities (tools, resources, prompts) to a Client. | `filesystem` server, `github` server, your own server |

A **Host** can contain **many Clients**, each with a **1:1 binding to one Server**. The Host
is the coordinator: it decides which servers to connect to, routes tool-call results back
into the model's context, applies user permissions, and aggregates results from multiple
servers into one conversation.

```
                    ┌──────────────────────────────────────────┐
                    │             HOST (Claude Desktop)        │
                    │   ┌────────────────────────────────────┐ │
                    │   │   LLM + conversation + UI          │ │
                    │   └──────────────┬─────────────────────┘ │
                    │                  │                       │
                    │   ┌──────────────┼─────────────────────┐ │
                    │   │              │  routing layer      │ │
                    │   └──┬────────┬──┴───┬─────────────────┘ │
                    │      │        │      │                   │
                    │   ┌──▼──┐  ┌──▼──┐  ┌▼────┐              │
                    │   │ Cli │  │ Cli │  │ Cli │   (Clients)  │
                    │   │ #1  │  │ #2  │  │ #3  │              │
                    │   └──┬──┘  └──┬──┘  └──┬──┘              │
                    └──────┼────────┼────────┼─────────────────┘
                           │        │        │   transport (stdio / HTTP)
                           ▼        ▼        ▼
                       ┌──────┐ ┌──────┐ ┌──────────┐
                       │ FS   │ │ GH   │ │ Postgres │      (Servers)
                       │Server│ │Server│ │  Server  │
                       └──────┘ └──────┘ └──────────┘
```

> **Key insight**: the Client is not the LLM. The Client is the wire-protocol speaker. The
> LLM lives in the Host and never touches MCP directly — it sees tool results that the Host
> formatted for it. This separation is why a single Host can mix servers from different
> vendors with no coordination between them.

---

## 2. The Wire Format: JSON-RPC 2.0

Every message exchanged between a Client and a Server is a **JSON-RPC 2.0** object. Always.
Whether the transport is stdio pipes or HTTP, the framing is JSON-RPC.

A JSON-RPC 2.0 object always carries `"jsonrpc": "2.0"`. There are exactly three message
shapes — request, response, and notification — distinguished by which fields are present.

### Request (Client → Server, expects a response)

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "/var/log/nginx/access.log",
      "max_bytes": 65536
    }
  }
}
```

- `id` is a unique correlator (integer or string). The peer must echo it on the response.
- `method` is the operation. MCP namespaces methods with `/` (e.g. `tools/call`, `resources/read`).
- `params` is method-specific.

### Response (Server → Client, correlated by `id`)

A response is **either** a `result` **or** an `error` — never both.

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "192.0.2.10 - - [23/Apr/2026:10:14:22 +0000] \"GET /api/health\" 200 17\n..."
      }
    ],
    "isError": false
  }
}
```

### Notification (either direction, no response expected)

A notification has **no `id`**. The peer must not respond to it.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

> **Rule**: presence of `id` is the only reliable way to tell a request from a notification.
> If you're parsing inbound traffic and see no `id`, do not allocate a response slot for it.

---

## 3. The Three Message Types Side by Side

| Type             | Has `id`? | Has `method`? | Has `result` / `error`? | Direction                     | Example                     |
|------------------|-----------|---------------|-------------------------|-------------------------------|-----------------------------|
| **Request**      | yes       | yes           | no                      | client→server **or** server→client | `tools/call`, `sampling/createMessage` |
| **Response**     | yes       | no            | exactly one             | reply to a request            | result of `tools/call`      |
| **Notification** | **no**    | yes           | no                      | either way, fire-and-forget   | `notifications/initialized`, `notifications/tools/list_changed` |

A few practical consequences:

- **Notifications cannot fail.** There is no error channel for them. If a notification can't
  be delivered, the sender finds out only via transport-level failure (broken pipe, HTTP 5xx).
- **Requests must be correlated.** Both sides need an outstanding-request map keyed by `id`,
  with timeout handling. The protocol does not specify a default timeout — pick one.
- **`id` collisions are the sender's responsibility.** Each side maintains its own `id`
  space. The Server's `id=1` and the Client's `id=1` are unrelated.

---

## 4. Bidirectional: Servers Can Send Requests Too

A common misconception is that Servers only respond. They don't. After the handshake, **the
Server can initiate requests back to the Client**. This is what enables some of MCP's most
interesting features:

| Server-initiated request   | What it asks the Client to do                           | Covered in           |
|----------------------------|---------------------------------------------------------|----------------------|
| `sampling/createMessage`   | "Run this prompt through your LLM and give me the text" | [03_primitives.md](03_primitives.md) |
| `elicitation/create`       | "Ask the user for this information"                     | [03_primitives.md](03_primitives.md) |
| `roots/list`               | "Which filesystem roots am I allowed to operate on?"    | [03_primitives.md](03_primitives.md) |

```
                Client                                       Server
                  │                                            │
                  │── tools/call (id=7) ──────────────────────>│
                  │                                            │  (server needs LLM help
                  │<── sampling/createMessage (id=2) ──────────│   to plan a sub-step)
                  │                                            │
                  │── result (id=2) ──────────────────────────>│
                  │                                            │
                  │<── result (id=7) ──────────────────────────│
                  │                                            │
```

> **Key insight**: MCP is peer-to-peer at the protocol layer. The Client/Server labels
> describe **who initiates the connection**, not **who can ask questions**. Once the session
> is up, both sides can act as a JSON-RPC requester. An MCP implementation that only handles
> the response direction is incomplete.

---

## 5. Capabilities: What Each Side Declares It Supports

MCP is intentionally extensible — newer features must not break older peers. The mechanism
is **capability negotiation** during the `initialize` handshake. Each side advertises what
it implements; neither side may use a feature the other didn't advertise.

### Client → Server: `initialize` request

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "roots": { "listChanged": true },
      "sampling": {},
      "elicitation": {}
    },
    "clientInfo": {
      "name": "claude-desktop",
      "version": "1.42.0"
    }
  }
}
```

### Server → Client: `initialize` response

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": { "listChanged": true, "subscribe": true },
      "prompts": { "listChanged": true },
      "logging": {}
    },
    "serverInfo": {
      "name": "filesystem",
      "version": "0.7.3"
    }
  }
}
```

### Client → Server: `notifications/initialized` (the handshake terminator)

Once the Client has read the response, it sends a **notification** (no `id`) to confirm the
session is live. Only after this notification may either side send normal traffic.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

| Capability key (server side) | Means the server can…                                                |
|------------------------------|----------------------------------------------------------------------|
| `tools`                      | Expose callable functions via `tools/list` and `tools/call`          |
| `resources`                  | Expose readable URIs via `resources/list` and `resources/read`       |
| `prompts`                    | Expose templated prompts via `prompts/list` and `prompts/get`        |
| `logging`                    | Emit `notifications/message` log entries to the client               |

| Capability key (client side) | Means the client can…                                                |
|------------------------------|----------------------------------------------------------------------|
| `roots`                      | Answer `roots/list` (filesystem boundaries the server may touch)     |
| `sampling`                   | Service `sampling/createMessage` (run LLM completions for the server)|
| `elicitation`                | Service `elicitation/create` (prompt the user for missing input)     |

The `listChanged` sub-capability is a promise that the side will emit a notification (e.g.
`notifications/tools/list_changed`) when its catalog mutates, so the peer can re-fetch.

> **Rule**: never call a method whose capability the peer didn't advertise. Servers should
> reject ungranted method calls with JSON-RPC error `-32601` (method not found). Clients
> should not even attempt them — gate the UI by capability flags.

The full handshake choreography lives in [04_lifecycle.md](04_lifecycle.md).

---

## 6. Sessions: Stateful vs Stateless

A **session** is the lifetime between a successful `initialize` and a `shutdown` (or
transport disconnect). Within a session, both peers can:

- Hold subscriptions (`resources/subscribe` keeps the server pushing updates for a URI).
- Track pending requests by `id`.
- Cache the negotiated capabilities and protocol version.

Sessions matter because they constrain the choice of transport.

| Property               | Stateful session                              | Stateless                                          |
|------------------------|-----------------------------------------------|----------------------------------------------------|
| Subscriptions          | Yes — server pushes updates                   | No — every call is independent                     |
| Server-→client requests| Yes (sampling, elicitation, roots query)      | Hard or impossible — no return channel             |
| Server identity        | Persistent across calls                       | Any worker may handle any call                     |
| Typical transport      | stdio (long-lived pipe), Streamable HTTP      | Plain request/response HTTP behind a load balancer |
| Failure mode           | Disconnect = lose subscriptions, cached state | None — but features above are unavailable          |

A stateless deployment **can** still serve `tools/call` and `tools/list` perfectly well,
because those are pure request/response. It just gives up the bidirectional and subscription
features. The Streamable HTTP transport (introduced 2025) was designed precisely so a server
can be deployed behind a normal load balancer **and** still maintain logical sessions via
session IDs in HTTP headers — covered in [05_transports.md](05_transports.md).

> **Key insight**: "Is this server stateful?" determines what features it can offer.
> Choose the transport based on which features you need, not the other way around.

---

## 7. End-to-End: A Tool Call from Keystroke to Response

Here is what actually happens when a user in Claude Desktop says "summarize the latest
nginx access log" and the host has a `filesystem` server connected.

```
 user                Host (Claude Desktop)               Client #1            Server (fs)
   │                       │                                │                     │
   │── "summarize log" ───>│                                │                     │
   │                       │                                │                     │
   │                       │  LLM emits tool_use:           │                     │
   │                       │  read_file(path="/var/log/...")│                     │
   │                       │                                │                     │
   │                       │── dispatch ───────────────────>│                     │
   │                       │                                │── JSON-RPC request ─>│
   │                       │                                │   id=42             │
   │                       │                                │   tools/call        │
   │                       │                                │                     │
   │                       │                                │             ┌───────┴────────┐
   │                       │                                │             │ open(), read() │
   │                       │                                │             │ stat checks    │
   │                       │                                │             │ build content[]│
   │                       │                                │             └───────┬────────┘
   │                       │                                │                     │
   │                       │                                │<── JSON-RPC reply ──│
   │                       │                                │   id=42             │
   │                       │                                │   result.content[]  │
   │                       │                                │                     │
   │                       │<── tool_result ────────────────│                     │
   │                       │                                │                     │
   │                       │  feed result back to LLM,      │                     │
   │                       │  LLM emits final answer        │                     │
   │                       │                                │                     │
   │<── "10,432 requests, ─│                                │                     │
   │     mostly 200s..."   │                                │                     │
```

Each arrow between Client and Server is a JSON-RPC 2.0 frame on the negotiated transport.
Each arrow between Host and Client is an in-process function call (no serialization).

---

## 8. Errors: Two Distinct Layers

MCP uses two error mechanisms, and conflating them is a common bug.

### 8.1 JSON-RPC Protocol Errors

These signal **the protocol itself failed**: bad JSON, unknown method, invalid params,
internal exception in the dispatcher. Standard JSON-RPC 2.0 codes apply.

| Code     | Meaning                | When you'd see it                                                 |
|----------|------------------------|-------------------------------------------------------------------|
| `-32700` | Parse error            | Server got malformed JSON                                         |
| `-32600` | Invalid request        | Missing `jsonrpc`, missing `method`, etc.                         |
| `-32601` | Method not found       | Client called `tools/call` on a server with no `tools` capability |
| `-32602` | Invalid params         | `tools/call` with a missing required argument                     |
| `-32603` | Internal error         | Unhandled exception in the server's request handler               |

Wire format:

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "error": {
    "code": -32602,
    "message": "Invalid params: 'path' is required",
    "data": { "missing": ["path"] }
  }
}
```

### 8.2 Application Errors (Tool Execution Failures)

If the **tool itself ran but failed** — file not found, API quota exceeded, validation
rejected — that's **not** a JSON-RPC error. It's a successful response whose `result`
carries `isError: true`. The LLM is supposed to see this and decide what to do.

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Error: file not found: /var/log/nginx/access.log"
      }
    ],
    "isError": true
  }
}
```

> **Key insight**: a JSON-RPC error means "the call could not be made." `isError: true`
> means "the call was made and the tool reports it failed." LLMs handle the latter
> gracefully — they read the message and try something else. JSON-RPC errors usually
> indicate a bug in the Client or Server, not a recoverable runtime condition.

⚠️ A common server-side mistake is to raise an exception inside a tool handler and let it
become a `-32603` internal error. Catch it, format the failure as text, and return
`isError: true`. The LLM can recover from the latter; it cannot from the former.

---

## 9. Vocabulary Recap

Before moving on, you should be able to answer all of these in one sentence:

| Term                         | Quick definition                                                                  |
|------------------------------|-----------------------------------------------------------------------------------|
| **Host**                     | The LLM application — owns the model, conversation, and user.                     |
| **Client**                   | An in-Host connector, 1:1 with a Server, speaks JSON-RPC.                         |
| **Server**                   | A process exposing tools, resources, and prompts.                                 |
| **JSON-RPC 2.0**             | The wire framing — every MCP message has `"jsonrpc": "2.0"`.                      |
| **Request**                  | Has `id` and `method`, expects a response.                                        |
| **Response**                 | Has `id` and exactly one of `result` or `error`.                                  |
| **Notification**             | Has `method` but no `id`, fire-and-forget.                                        |
| **Capability**               | A flag declared in `initialize` saying "I support feature X."                     |
| **Session**                  | The lifetime between successful `initialize` and disconnect.                      |
| **`isError: true`**          | A successful response whose tool reports application-level failure.               |

---

**Next**: [03_primitives.md — Primitives: Tools, Resources, Prompts](03_primitives.md)
