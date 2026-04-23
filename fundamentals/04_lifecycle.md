# Connection Lifecycle: Handshake, Operation, Shutdown

> **Who this is for**: Engineers who have read [02_architecture.md](02_architecture.md) and
> [03_primitives.md](03_primitives.md) and want to know what bytes actually fly across the
> wire when a client connects to an MCP server — from the first `initialize` request through
> ping/heartbeat to a clean shutdown.

---

## 1. The Three Lifecycle Phases

Every MCP connection moves through three strictly-ordered phases. You cannot skip a phase,
and you cannot run operational requests before the handshake completes.

```
   ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
   │ 1. INITIALIZE    │───>│ 2. OPERATION     │───>│ 3. SHUTDOWN      │
   │                  │    │                  │    │                  │
   │ - initialize     │    │ - tools/call     │    │ - close stdin    │
   │ - capabilities   │    │ - resources/read │    │ - close socket   │
   │ - version        │    │ - notifications  │    │ - DELETE session │
   │ - initialized    │    │ - ping           │    │                  │
   └──────────────────┘    └──────────────────┘    └──────────────────┘
        synchronous               long-lived              transport
        request/response          bidirectional           dependent
```

| Phase          | Who initiates | What's allowed                                      |
|----------------|---------------|-----------------------------------------------------|
| Initialize     | Client        | Only `initialize` request and `initialized` notif    |
| Operation      | Either side   | Anything advertised in capability negotiation       |
| Shutdown       | Either side   | Transport-level disconnect; no special RPC          |

> **Rule**: An operational request sent before `notifications/initialized` is a protocol
> error. The server should reject it; the client should never send one.

---

## 2. Initialize Handshake

Three messages, in this order:

1. Client → Server: `initialize` **request** (has an `id`, expects a response)
2. Server → Client: `initialize` **result** (matches the `id`)
3. Client → Server: `notifications/initialized` (no `id`, no response)

After step 3, the connection is operational.

### 2.1. Client `initialize` request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "roots": {
        "listChanged": true
      },
      "sampling": {},
      "elicitation": {}
    },
    "clientInfo": {
      "name": "claude-code",
      "title": "Claude Code",
      "version": "1.4.2"
    }
  }
}
```

A few things to notice:

- `protocolVersion` is the **highest** version the client supports — not a list.
- `capabilities` is an object, not an array. Each capability name maps to a sub-object that
  may carry feature flags (here, `roots.listChanged: true` advertises that the client will
  send `notifications/roots/list_changed` when the user opens or closes a workspace).
- `clientInfo.name` is a stable machine identifier; `title` is the optional human-readable
  display name (added in `2025-11-25`); `version` is the client's own semver.

### 2.2. Server `initialize` result

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "tools": {
        "listChanged": true
      },
      "resources": {
        "subscribe": true,
        "listChanged": true
      },
      "prompts": {
        "listChanged": false
      },
      "logging": {},
      "completions": {}
    },
    "serverInfo": {
      "name": "github-mcp-server",
      "title": "GitHub MCP Server",
      "version": "0.9.1"
    },
    "instructions": "Use list_repos before calling get_repo to discover valid owner/name pairs. All write operations require the user to confirm via elicitation."
  }
}
```

- The server **may** downgrade `protocolVersion` (see §3).
- `instructions` is free-form text the host SHOULD inject into the model's system prompt.
  Treat it as an authoring hint from the server — short, declarative, no markdown.
- Server capabilities mirror the client's: each one is an object whose sub-flags advertise
  optional behavior (e.g., `resources.subscribe` means the server supports
  `resources/subscribe` for change notifications).

### 2.3. Client `notifications/initialized`

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

No `id`, no `params`, no response. This is the client's "I've processed your capabilities,
let's begin" signal. Until the server receives this, it MUST NOT send any requests
(notifications are also discouraged but not strictly forbidden).

### 2.4. Sequence diagram

```
Client                                Server
  │                                     │
  │── initialize (id=1) ───────────────>│
  │   protocolVersion: "2025-11-25"     │
  │   capabilities: {...}               │
  │   clientInfo: {...}                 │
  │                                     │
  │                                     │  [server picks version,
  │                                     │   builds capability set]
  │                                     │
  │<── result (id=1) ───────────────────│
  │    protocolVersion: "2025-11-25"    │
  │    capabilities: {...}              │
  │    serverInfo: {...}                │
  │    instructions: "..."              │
  │                                     │
  │  [client validates version,         │
  │   stores agreed capabilities]       │
  │                                     │
  │── notifications/initialized ───────>│
  │                                     │
  │═══════ OPERATION PHASE BEGINS ══════│
  │                                     │
  │── tools/list (id=2) ───────────────>│
  │<── result (id=2) ───────────────────│
  │              ...                    │
```

---

## 3. Protocol Version Negotiation

MCP versions are **date strings**, not semver. The wire format is `YYYY-MM-DD`.

| Version       | Notable changes                                                  |
|---------------|------------------------------------------------------------------|
| `2024-11-05`  | First public release                                             |
| `2025-03-26`  | Streamable HTTP transport; sessions; OAuth flows                 |
| `2025-06-18`  | Elicitation primitive; structured tool output                    |
| `2025-11-25`  | `clientInfo.title` / `serverInfo.title`; tightened auth rules    |

### 3.1. Negotiation rules

- The client sends the **highest** version it supports.
- The server responds with the version it agrees to use. This MUST be a version both sides
  understand. The server will typically pick the client's version if it supports it,
  otherwise the highest version it knows.
- The client MUST then either (a) accept the server's chosen version and proceed, or
  (b) disconnect.

```json
// Client supports up to 2025-11-25
"params": { "protocolVersion": "2025-11-25", ... }

// Server only knows up to 2025-06-18 — it downgrades
"result": { "protocolVersion": "2025-06-18", ... }

// Client must now decide:
//   ✅ continue using 2025-06-18 features only
//   ❌ refuse and close the connection
```

> **Key insight**: Backwards compatibility is **best-effort**, not guaranteed. A client
> targeting `2025-11-25` cannot assume any new fields (like `clientInfo.title`) will be
> respected by a server that downgraded to `2025-06-18`. Inspect the agreed version and
> branch on it.

⚠️ A client that ignores the server's chosen version and keeps sending newer-version
features is **undefined behavior**. The server may parse it, may error, or may silently
drop fields. Always honor the agreed version.

---

## 4. Capability Negotiation

Capabilities are how each side tells the other "here's what I can do." A capability that
isn't advertised is effectively non-existent: calling its methods is a protocol error.

### 4.1. Client capabilities

| Capability      | Meaning                                                          |
|-----------------|------------------------------------------------------------------|
| `roots`         | Client can serve `roots/list` (the workspace folders)            |
| `sampling`      | Client can fulfill `sampling/createMessage` (LLM completions)    |
| `elicitation`   | Client can prompt the user via `elicitation/create`              |
| `experimental`  | Vendor-prefixed extensions (free-form object)                    |

### 4.2. Server capabilities

| Capability     | Meaning                                                           |
|----------------|-------------------------------------------------------------------|
| `tools`        | Server exposes `tools/list` and `tools/call`                      |
| `resources`    | Server exposes `resources/list` and `resources/read`              |
| `prompts`      | Server exposes `prompts/list` and `prompts/get`                   |
| `logging`      | Server can emit `notifications/message` log entries               |
| `completions`  | Server supports `completion/complete` (autocomplete for prompts)  |
| `experimental` | Vendor-prefixed extensions                                        |

### 4.3. Sub-flags

Each capability is an **object**, not a boolean. Sub-flags refine behavior:

```json
{
  "tools": {
    "listChanged": true
  },
  "resources": {
    "subscribe": true,
    "listChanged": true
  }
}
```

| Sub-flag       | Where it appears                       | Meaning                                              |
|----------------|----------------------------------------|------------------------------------------------------|
| `listChanged`  | `tools`, `resources`, `prompts`, `roots` | Sender will emit `*/list_changed` notifications      |
| `subscribe`    | `resources`                            | Server supports `resources/subscribe` for live diffs |

> **Rule**: If the server advertises `tools` but **not** `tools.listChanged: true`, the
> client must NOT expect `notifications/tools/list_changed` and should poll instead (or
> just trust the initial list).

### 4.4. Enforcement

```json
// ❌ Server didn't advertise "prompts" — this returns -32601 (Method not found)
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "prompts/list"
}

// ✅ Server advertised "prompts: {}" — this works
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "prompts/list"
}
```

> **Key insight**: Capability negotiation is the **only** source of truth for what's
> available on the connection. Don't infer from `serverInfo.name` or hardcode behavior per
> server — always check the agreed capability set.

---

## 5. Operation Phase

Once both sides have exchanged the handshake, the connection is fully bidirectional.
What's allowed:

- **Requests in either direction**, scoped to advertised capabilities.
  - Server-originated requests typically use client capabilities (`sampling/createMessage`,
    `elicitation/create`, `roots/list`).
  - Client-originated requests use server capabilities (`tools/call`, `resources/read`,
    etc.).
- **Notifications** in either direction (no `id`, no response).
- **Pings** from either side at any time.

### 5.1. The notification catalog

| Notification                              | Direction      | When it fires                                |
|-------------------------------------------|----------------|----------------------------------------------|
| `notifications/cancelled`                 | Either         | Cancelling a previously-sent request         |
| `notifications/progress`                  | Either         | Progress update for a long-running request   |
| `notifications/message`                   | Server→Client  | Log entry (debug/info/warning/error)         |
| `notifications/tools/list_changed`        | Server→Client  | Tool catalog changed                         |
| `notifications/resources/list_changed`    | Server→Client  | Resource catalog changed                     |
| `notifications/prompts/list_changed`      | Server→Client  | Prompt catalog changed                       |
| `notifications/resources/updated`         | Server→Client  | A subscribed resource's content changed      |
| `notifications/roots/list_changed`        | Client→Server  | Workspace roots changed (e.g., user opened folder) |

### 5.2. Ping (heartbeat)

Either side may send a `ping` request at any time. Empty params, empty result. It exists
solely to detect dead TCP/stdio connections.

```json
// Request (either direction)
{
  "jsonrpc": "2.0",
  "id": 99,
  "method": "ping"
}

// Response
{
  "jsonrpc": "2.0",
  "id": 99,
  "result": {}
}
```

💡 No specific cadence is mandated. A common policy: ping every 30s if the connection has
been idle that long, and treat 3 missed pings (or a 90s round-trip timeout) as dead.

---

## 6. Cancellation

Long-running requests can be cancelled by sending a notification, **not** a request. The
notification carries the `id` of the original request.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/cancelled",
  "params": {
    "requestId": 42,
    "reason": "User pressed Esc"
  }
}
```

Semantics:

- The receiver SHOULD stop processing as soon as practical, but is **not required to** —
  cancellation is best-effort. A tool call that's already committed a write may legitimately
  ignore the cancellation.
- The receiver SHOULD still respond to the original request (typically with an error result
  whose code/message indicates cancellation). This avoids leaking pending requests on the
  sender side.
- It is safe to cancel a request whose response is already in flight; the sender just
  ignores the late response.

⚠️ Don't cancel `initialize`. Behavior is undefined and you'll likely end up with a
zombie connection.

---

## 7. Progress Reporting

For long operations (a multi-step refactor tool, a large file scan), the originating side
opts in to progress by attaching a `progressToken` in the request's `_meta`:

```json
{
  "jsonrpc": "2.0",
  "id": 17,
  "method": "tools/call",
  "params": {
    "name": "rebuild_search_index",
    "arguments": {"path": "/repo"},
    "_meta": {
      "progressToken": "rebuild-2026-04-23-a"
    }
  }
}
```

The receiver then emits one or more progress notifications, **referencing the same token**:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "rebuild-2026-04-23-a",
    "progress": 340,
    "total": 1200,
    "message": "Indexing src/auth/..."
  }
}
```

| Field           | Type             | Notes                                          |
|-----------------|------------------|------------------------------------------------|
| `progressToken` | string \| number | Echo the value the sender chose                |
| `progress`      | number           | Current units processed (monotonically increasing) |
| `total`         | number           | Optional. If omitted, treat as indeterminate   |
| `message`       | string           | Optional human-readable status                 |

> **Rule**: A receiver MUST NOT emit progress notifications for a request that didn't
> include a `progressToken`. Doing so is harmless but wasteful — the sender has no way to
> correlate them.

---

## 8. Shutdown

There is no `shutdown` RPC. The signal is **transport-level disconnect**.

### 8.1. Stdio transport

The client owns the server process and tears it down explicitly:

```
1. Client closes the server's stdin.
   → Server's read loop sees EOF, exits cleanly.

2. If the server hasn't exited within a reasonable timeout (~5s):
   → Client sends SIGTERM. Well-behaved servers handle this and exit.

3. If still alive after another timeout (~2s):
   → Client sends SIGKILL. Unrecoverable; the OS terminates the process.
```

> **Rule**: Always close stdin first. Sending SIGTERM to a server that's mid-write to
> stdout can produce truncated JSON-RPC frames in any logs the parent process is capturing.

### 8.2. HTTP transport

For Streamable HTTP, shutdown is even simpler:

- For one-shot requests: just close the HTTP connection.
- For sessions (when the server issued an `Mcp-Session-Id` — see §9): send an explicit
  `DELETE` to the MCP endpoint with the `Mcp-Session-Id` header. This frees server-side
  state immediately.

```http
DELETE /mcp HTTP/1.1
Host: server.example.com
Mcp-Session-Id: 7f3a2c10-94e8-4a21-9b3e-1c5e1f8b4d20

HTTP/1.1 204 No Content
```

⚠️ Skipping the `DELETE` won't break correctness — sessions expire on the server's TTL —
but it leaks server memory and may keep streaming subscriptions alive longer than needed.

---

## 9. Sessions (HTTP Transports)

Stdio is implicitly session-scoped (one process = one session). HTTP needs an explicit
session identifier so multiple requests from the same client land on the same server-side
state.

### 9.1. Session establishment

When the client sends the `initialize` POST over HTTP, the server may set a response header:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Mcp-Session-Id: 7f3a2c10-94e8-4a21-9b3e-1c5e1f8b4d20

{"jsonrpc":"2.0","id":1,"result":{...}}
```

The session ID can be a UUID (opaque), a JWT (carrying signed claims about the client), or
any other server-chosen string.

### 9.2. Session usage

Every subsequent request MUST include the header:

```http
POST /mcp HTTP/1.1
Host: server.example.com
Content-Type: application/json
Mcp-Session-Id: 7f3a2c10-94e8-4a21-9b3e-1c5e1f8b4d20

{"jsonrpc":"2.0","id":2,"method":"tools/list"}
```

### 9.3. Session expiration

If the server has GC'd the session (TTL elapsed, server restarted, sticky-routing miss),
it returns:

```http
HTTP/1.1 404 Not Found

{"error": "Session not found"}
```

The client's only valid recovery is to **re-initialize from scratch** — there is no resume
protocol. Buffer in-flight requests, run a fresh `initialize`, replay capability checks,
then reissue.

### 9.4. Stateful vs stateless servers

| Server style | Issues `Mcp-Session-Id`? | Behavior                                   |
|--------------|--------------------------|--------------------------------------------|
| Stateful     | Yes                      | Per-session memory: subscriptions, auth, etc. |
| Stateless    | No (omits header)        | Each request is independent; no session     |

Stateless servers scale horizontally without sticky routing. Stateful servers either need
sticky routing or a shared store (Redis, DB) keyed by session ID — see
[../production/03_scaling.md](../production/03_scaling.md).

> **Foreshadow**: The full mechanics of Streamable HTTP — POST vs SSE responses, resumable
> streams, the `Last-Event-ID` header — are covered in
> [05_transports.md](05_transports.md). Multi-instance session strategies are in
> [../production/03_scaling.md](../production/03_scaling.md).

---

## 10. Common Failure Modes

**❌ Sending operational requests before `notifications/initialized`**

```
Client                     Server
  │                          │
  │── initialize ───────────>│
  │<── result ───────────────│
  │── tools/list ───────────>│   ← protocol error: handshake not complete
```

The client must send `notifications/initialized` first. Servers should reject anything else
with a `-32002` (or implementation-defined) error.

**❌ Calling a method the server didn't advertise**

```json
// Server's capabilities did NOT include "prompts"
// Client sends:
{"jsonrpc":"2.0","id":3,"method":"prompts/list"}

// Server replies:
{"jsonrpc":"2.0","id":3,"error":{"code":-32601,"message":"Method not found"}}
```

Always inspect the negotiated capability set and gate code paths on it.

**❌ Ignoring the agreed protocol version**

The server downgraded to `2025-06-18` but the client keeps sending `clientInfo.title`
(added in `2025-11-25`). Behavior is undefined: the server may parse it, may reject it,
may silently drop the field. Branch on the agreed version.

**⚠️ Forgetting `notifications/initialized` entirely**

This is the **most common bug** in new MCP client implementations. Symptoms: the server
appears to hang indefinitely after responding to `initialize`. The client thinks it's
ready; the server is still waiting for the third message. Always check that your client
fires the notification immediately after processing the `initialize` result.

**⚠️ Cancelling without expecting a late response**

Cancellation is a notification, not a synchronous abort. The receiver may already have the
response in flight when your cancellation arrives. Your client must tolerate (and discard)
a response for an `id` it has already cancelled — don't crash, don't double-handle.

**⚠️ Treating session expiration as a transport error**

A 404 with `Mcp-Session-Id` mid-conversation is **not** a network failure — it's the
server saying "your session is gone, start over." Distinguish this from connection-reset
errors and wire it to your re-initialize path, not your reconnect-with-backoff path.

---

> **Key insight**: The lifecycle is the contract. Once you understand the three phases,
> what's negotiated in each, and how shutdown is just transport disconnect, every other
> piece of MCP — transports, primitives, auth — is implementation detail layered on top.

---

**Next**: [05_transports.md — Transports: stdio and Streamable HTTP](05_transports.md)
