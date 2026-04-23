# Per-Request Context and Building Clients

> **Who this is for**: TypeScript engineers who already have tools, resources, and prompts
> running on an MCP server (see [03_resources_prompts.md](03_resources_prompts.md)) and now
> need to (a) reach back to the client mid-request — for logging, progress, sampling — and
> (b) build clients that talk to MCP servers.

---

## 1. Server-Side Context — the `extra` Parameter

Every tool, resource, and prompt handler in the TS SDK receives a second argument: a
per-request context object conventionally named `extra`. It carries the request id, the
session id, an `AbortSignal`, and the two methods you use to talk back to the client:
`sendNotification` and `sendRequest`.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

server.registerTool(
  "long_task",
  { title: "Long Task", description: "...", inputSchema: { items: z.array(z.string()) } },
  async ({ items }, extra) => {
    // extra is the per-request context — request id, session id, abort signal, server reference
    extra.sendNotification({
      method: "notifications/message",
      params: { level: "info", data: `processing ${items.length} items` },
    });
    for (let i = 0; i < items.length; i++) {
      if (extra.signal.aborted) throw new Error("cancelled");
      await processItem(items[i]);
      await extra.sendNotification({
        method: "notifications/progress",
        params: { progressToken: extra._meta?.progressToken, progress: i + 1, total: items.length },
      });
    }
    return { content: [{ type: "text", text: `done (${items.length})` }] };
  },
);
```

> **Key insight**: `extra` is scoped to a single in-flight request. Once your handler
> returns (or throws), the request lifecycle is over and any further `sendNotification`
> calls are silently dropped or rejected.

---

## 2. What's on `extra`

| Field | Use |
|-------|-----|
| `extra.requestId` | Correlate with the JSON-RPC request id |
| `extra.sessionId` | The session id (HTTP transports) |
| `extra.signal` | `AbortSignal` — fires on client cancellation |
| `extra.sendNotification(...)` | Push a notification to the client (logging, progress) |
| `extra.sendRequest(req, schema)` | Send a request to the client (sampling, elicitation, list_roots) |
| `extra.authInfo` | Auth info populated by the auth provider (production) |
| `extra._meta` | Request-level metadata (progressToken etc.) |

Two things to internalize:

- **`extra.signal` is the cancellation channel.** The client can send a `notifications/cancelled`
  at any time; the SDK aborts the signal. Long loops, `fetch` calls, and DB queries should
  all wire it through.
- **`extra.sendRequest` is bidirectional MCP.** The protocol is symmetric — the server can
  ask the client to do work (run an LLM call, prompt the user, list filesystem roots).
  This is how server-initiated sampling and elicitation work.

---

## 3. App-Scoped State (No Lifespan in the TS SDK)

Python's FastMCP has a first-class `lifespan` async context manager. The TS SDK does not.
The idiomatic replacement is **module-level singletons created at import time, plus a
`SIGTERM` cleanup hook**.

```typescript
import { Pool } from "pg";

const db = new Pool({ connectionString: process.env.DATABASE_URL });

server.registerTool(
  "get_user",
  { title: "Get User", description: "...", inputSchema: { id: z.string() } },
  async ({ id }) => {
    const { rows } = await db.query("SELECT * FROM users WHERE id = $1", [id]);
    return { content: [{ type: "text", text: JSON.stringify(rows[0] ?? null) }] };
  },
);

// Shutdown hook
process.on("SIGTERM", async () => { await db.end(); process.exit(0); });
```

TS doesn't have a Python-style lifespan context manager — module-level singletons + a
`SIGTERM` cleanup is idiomatic. If you want testability, wrap the singleton in a small
factory and inject it into a `createServer(deps)` function so tests can pass fakes.

> **Rule**: Don't construct the connection pool inside a tool handler. You'll create one
> per request, exhaust the database, and have nothing to clean up on shutdown.

⚠️ Stdio servers don't always receive `SIGTERM` — the parent process may just close stdin.
Listen for `process.stdin.on("end", ...)` too if clean shutdown matters.

---

## 4. Server-Initiated Sampling

Sampling lets the server ask the client to run an LLM call on its behalf. The server
doesn't need an API key — the client (the host application) owns the model relationship.

```typescript
import { CreateMessageResultSchema } from "@modelcontextprotocol/sdk/types.js";

server.registerTool(
  "summarize",
  { title: "Summarize", description: "...", inputSchema: { url: z.string().url() } },
  async ({ url }, extra) => {
    const text = await (await fetch(url)).text();
    const result = await extra.sendRequest(
      {
        method: "sampling/createMessage",
        params: {
          messages: [{ role: "user", content: { type: "text", text: `Summarize:\n\n${text.slice(0, 8000)}` } }],
          maxTokens: 200,
          systemPrompt: "You write tight three-sentence summaries.",
          modelPreferences: { hints: [{ name: "claude-3-5-sonnet" }], intelligencePriority: 0.7 },
        },
      },
      CreateMessageResultSchema,
    );
    const summary = (result.content as any).text;
    return { content: [{ type: "text", text: summary }] };
  },
);
```

A few non-obvious points:

- **The schema is required.** `sendRequest` takes a Zod schema as its second argument and
  validates the response. If you forget it the call won't even compile.
- **`modelPreferences` is a hint, not a contract.** The client picks the actual model.
  Use `intelligencePriority` / `speedPriority` / `costPriority` to express trade-offs.
- **Sampling requires the client to advertise `capabilities.sampling`** during initialize
  (see section 8). If it didn't, your `sendRequest` rejects.

For elicitation (asking the user a structured question), the shape is the same — swap
the method to `elicitation/create` and the schema to `ElicitResultSchema`. See
[../python/05_sampling_elicitation.md](../python/05_sampling_elicitation.md) for the
mental model; it transfers verbatim.

---

## 5. Client Basics — Recap

A client is the other half of MCP. Whether you're writing a host application (an IDE, a
chat UI) or a script that drives a server for testing, the API is the same.

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const transport = new StdioClientTransport({ command: "npx", args: ["tsx", "server.ts"] });
const client = new Client({ name: "demo", version: "1.0.0" });

await client.connect(transport);

const { tools } = await client.listTools();
const { resources } = await client.listResources();
const { prompts } = await client.listPrompts();

const result = await client.callTool({ name: "add", arguments: { a: 2, b: 3 } });
const resource = await client.readResource({ uri: "docs://readme" });
const prompt = await client.getPrompt({ name: "review_pr", arguments: { diff: "..." } });

await client.close();
```

The transport is the only thing that changes between local subprocess servers
(`StdioClientTransport`) and remote services (`StreamableHTTPClientTransport`). Once
connected, the `Client` API is transport-agnostic.

---

## 6. HTTP Client with Auth

For remote servers, use `StreamableHTTPClientTransport`. Auth headers go in
`requestInit.headers` and are sent with every request, including the SSE upgrade.

```typescript
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const transport = new StreamableHTTPClientTransport(
  new URL("https://api.example.com/mcp"),
  {
    requestInit: {
      headers: { Authorization: `Bearer ${process.env.MCP_TOKEN}` },
    },
  },
);
const client = new Client({ name: "demo", version: "1.0.0" });
await client.connect(transport);
```

⚠️ `requestInit.headers` applies to outgoing HTTP requests but doesn't help if the server
expects a token refresh mid-session. For OAuth-style flows, you want an `authProvider`
(covered in [05_advanced.md](05_advanced.md)).

---

## 7. Implementing Client-Side Handlers — Sampling and Elicitation

If your client supports sampling or elicitation, you must register handlers for the
incoming requests. This is what host applications do.

```typescript
client.setRequestHandler(CreateMessageRequestSchema, async (req) => {
  // Forward to your LLM provider
  const reply = await callAnthropic(req.params.messages);
  return {
    role: "assistant",
    content: { type: "text", text: reply },
    model: "claude-3-5-sonnet",
    stopReason: "endTurn",
  };
});

client.setRequestHandler(ElicitRequestSchema, async (req) => {
  const answer = await promptUserInTerminal(req.params.message, req.params.requestedSchema);
  return { action: "accept", content: answer };
});
```

This is what host applications do. A few practical notes:

- **`action` for elicitation has three values**: `"accept"`, `"decline"`, `"cancel"`.
  Decline means "I asked the user and they refused"; cancel means "the user dismissed
  without answering."
- **Sampling handlers should respect `req.params.modelPreferences`** when choosing a
  model, but you're not required to.
- **Always handle errors inside the handler.** A thrown error is converted to a JSON-RPC
  error response, but the message gets surfaced to the server-side caller — keep it
  user-safe.

---

## 8. Client Capabilities

Capabilities tell the server what optional features your client supports. They're
exchanged once during `initialize` and never re-negotiated.

```typescript
const client = new Client(
  { name: "demo", version: "1.0.0" },
  {
    capabilities: {
      sampling: {},          // I can answer sampling/createMessage
      elicitation: {},       // I can answer elicitation/create
      roots: { listChanged: true },  // I expose roots and notify on changes
    },
  },
);
```

| Capability | What it promises |
|------------|------------------|
| `sampling: {}` | You'll handle `sampling/createMessage` requests |
| `elicitation: {}` | You'll handle `elicitation/create` requests |
| `roots: {}` | You'll respond to `roots/list` |
| `roots: { listChanged: true }` | Plus you'll send `notifications/roots/list_changed` when the set changes |

> **Rule**: Declare only what you've actually wired up. A capability without a handler
> means servers will call into a void and time out.

---

## 9. Notifications from the Server

Servers push notifications for logging, progress, and list-changed events. Register
handlers with `setNotificationHandler`.

```typescript
client.setNotificationHandler(LoggingMessageNotificationSchema, (n) => {
  console.log(`[${n.params.level}]`, n.params.data);
});
client.setNotificationHandler(ProgressNotificationSchema, (n) => {
  console.log(`progress: ${n.params.progress}/${n.params.total ?? "?"}`);
});
client.setNotificationHandler(ToolListChangedNotificationSchema, async () => {
  const { tools } = await client.listTools();
  console.log("tool list changed, now:", tools.map(t => t.name));
});
```

The `*ListChanged` notifications are how dynamic tool sets work — a server might add or
remove tools as the user authenticates with different services. Re-list and refresh your
UI when you see one.

💡 Progress notifications only arrive if the client included a `_meta.progressToken` in
the original request. Generate a unique token per call (a counter or `crypto.randomUUID()`
substring) and you'll get matching progress events streamed back.

---

## 10. Failure Modes

- ❌ Forgetting to `await client.close()` — leaks the subprocess (stdio) or HTTP session
- ❌ Throwing synchronously from a handler — TS SDK wraps it but the stack trace can be misleading; use `throw new Error("...")`
- ❌ Calling `extra.sendNotification` after the handler returned → the request lifecycle is over
- ⚠️ Not setting an `AbortSignal` timeout on `fetch` calls inside tool handlers — easy to hang on slow upstreams
- ❌ Not declaring `capabilities.sampling` on the client but expecting servers to call `sampling/createMessage` — silently fails

A concrete pattern for the timeout point:

```typescript
async ({ url }, extra) => {
  const ac = new AbortController();
  const t = setTimeout(() => ac.abort(), 10_000);
  // Forward client cancellation into our own controller
  extra.signal.addEventListener("abort", () => ac.abort());
  try {
    const res = await fetch(url, { signal: ac.signal });
    return { content: [{ type: "text", text: await res.text() }] };
  } finally {
    clearTimeout(t);
  }
}
```

> **Key insight**: Composing `extra.signal` with your own `AbortController` is the
> idiomatic way to honour both client cancellation and a local timeout. Don't pass
> `extra.signal` directly to `fetch` if you also want a timeout — wire them through a
> single controller.

---

**Next**: [05_advanced.md — Streamable HTTP, Auth, Composition](05_advanced.md)
