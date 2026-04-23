# Advanced TypeScript SDK — Streamable HTTP, OAuth, and Server Composition

> **Who this is for**: TypeScript engineers who have shipped a basic MCP server
> with `McpServer` and now need to put it behind a load balancer, gate it with
> OAuth, host multiple servers in one process, or break out of the high-level
> API. Assumes you've worked through
> [04_context_clients.md](04_context_clients.md) and are comfortable with
> Express, Promises, and Node streams.

By the end of this file you should know which transport mode to choose, how
the SDK's auth helpers compose with Express middleware, and when to drop the
ergonomic `McpServer` wrapper for the lower-level `Server` class.

---

## 1. Streamable HTTP — Stateful with Sticky Sessions

Stateful Streamable HTTP keeps one transport per client session and routes
every subsequent request from that client back to the same in-memory
transport. This is the mode you want when the server needs to push
notifications (`notifications/resources/updated`,
`notifications/tools/list_changed`) or hold per-session scratch state.

The shape is: **one HTTP endpoint, three verbs, one map of session → transport**.
POST creates or feeds a session, GET opens the SSE channel for server-initiated
messages, DELETE tears the session down explicitly.

```typescript
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { isInitializeRequest } from "@modelcontextprotocol/sdk/types.js";
import { randomUUID } from "node:crypto";

const app = express();
app.use(express.json());

const transports = new Map<string, StreamableHTTPServerTransport>();

app.post("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  let transport = sessionId ? transports.get(sessionId) : undefined;

  if (!transport && isInitializeRequest(req.body)) {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (id) => transports.set(id, transport!),
      onsessionclosed: (id) => transports.delete(id),
    });
    await makeServer().connect(transport);
  } else if (!transport) {
    res.status(400).json({
      jsonrpc: "2.0",
      error: { code: -32600, message: "no session — initialize first" },
      id: null,
    });
    return;
  }

  await transport.handleRequest(req, res, req.body);
});

// GET = open the SSE channel for server→client messages
app.get("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  const transport = sessionId ? transports.get(sessionId) : undefined;
  if (!transport) { res.status(404).end(); return; }
  await transport.handleRequest(req, res);
});

// DELETE = explicit session termination
app.delete("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  const transport = sessionId ? transports.get(sessionId) : undefined;
  if (!transport) { res.status(404).end(); return; }
  await transport.handleRequest(req, res);
});

function makeServer() {
  const s = new McpServer({ name: "demo", version: "1.0.0" });
  // ... registerTool, registerResource, registerPrompt ...
  return s;
}

app.listen(8080);
```

Two things worth zooming in on:

- `isInitializeRequest` is the SDK's published guard. **Only** the initialize
  call may create a session. If a caller POSTs anything else without a session
  id, you reject with JSON-RPC error `-32600`. Without that guard you'd let
  a stray `tools/call` create a fresh, empty session — and you'd never notice
  until customers complained that their resources had vanished.
- `onsessioninitialized` and `onsessionclosed` are the only safe place to
  mutate the `transports` map. The transport assigns the id internally; trying
  to predict it before connect is a race.

> **Rule**: stateful mode requires sticky-session load balancing. If two
> sequential POSTs from the same client land on different replicas, the second
> one finds an empty `Map` and 404s. See
> [Production: Scaling MCP](../production/03_scaling.md) for the LB
> configuration that makes this work.

---

## 2. Streamable HTTP — Stateless Mode

Stateless mode trades server-push capability for trivial horizontal scaling.
Each request gets a fresh `McpServer` and a fresh transport; nothing in the
process outlives the response. Any replica can handle any request, so a
round-robin load balancer is fine.

```typescript
app.post("/mcp", async (req, res) => {
  // No session id, fresh server per request
  const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined });
  const server = makeServer();
  await server.connect(transport);

  res.on("close", () => { transport.close(); server.close(); });

  await transport.handleRequest(req, res, req.body);
});
```

The `sessionIdGenerator: undefined` is the explicit opt-out — the transport
won't emit an `Mcp-Session-Id` header and will reject GET/DELETE on the
endpoint. Each call is a unary request/response.

| Aspect                           | Stateful                              | Stateless                            |
|----------------------------------|---------------------------------------|--------------------------------------|
| Load balancing                   | Sticky (session affinity required)    | Round-robin / any                    |
| `notifications/*/list_changed`   | ✅ Works                               | ❌ Silently dropped                   |
| Server-side scratch state        | ✅ Per-session in memory               | ❌ Push to a database / Redis         |
| Cold-start cost per request      | One per session                       | One per request                      |
| Memory footprint                 | O(active sessions)                    | O(in-flight requests)                |
| Restart semantics                 | All clients must re-initialize        | Transparent — next request just works|
| Observability                     | Trace by session id                   | Trace by request id                  |

> **Key insight**: choose stateless first. You can almost always shape a
> stateful design into a stateless one by externalizing the state (Redis,
> Postgres, your auth provider). You cannot retrofit sticky sessions onto a
> load balancer that doesn't support them. The full trade-off is documented
> in [../production/03_scaling.md](../production/03_scaling.md).

A subtle one: `res.on("close", ...)` is not optional. If the client hangs up
mid-call (timeout, browser tab closed, network blip), Express won't tell the
SDK transport — and the transport will keep the in-memory `Server` alive
waiting for a response it can never send. Over a few hours of misbehaving
clients you'll leak enough transports to OOM the process. The two-line
cleanup in the example above is the thing that prevents this.

---

## 3. OAuth — Using `mcpAuthRouter` (the SDK's helper)

MCP standardized on OAuth 2.1 with two RFCs that matter for the server:
**RFC 9728** (Protected Resource Metadata) and **SEP-991** (Dynamic Client
Registration via Client ID Metadata Documents). Hand-rolling those endpoints
is a footgun; the SDK ships `mcpAuthRouter` to wire them up correctly.

The most common deployment is **MCP server as OAuth resource server** — your
MCP server validates tokens but delegates issuance to an upstream identity
provider (Auth0, WorkOS, your own Hydra, etc.). `ProxyOAuthServerProvider`
covers exactly this case.

```typescript
import { mcpAuthRouter, ProxyOAuthServerProvider } from "@modelcontextprotocol/sdk/server/auth/router.js";

const oauth = new ProxyOAuthServerProvider({
  endpoints: {
    authorizationUrl: "https://auth.example.com/oauth/authorize",
    tokenUrl: "https://auth.example.com/oauth/token",
    revocationUrl: "https://auth.example.com/oauth/revoke",
  },
  verifyAccessToken: async (token) => {
    const claims = await verifyJwt(token);
    return { token, clientId: claims.client_id, scopes: claims.scope.split(" ") };
  },
  getClient: async (clientId) => /* lookup registered client */,
});

app.use(mcpAuthRouter({
  provider: oauth,
  issuerUrl: new URL("https://auth.example.com"),
  baseUrl: new URL("https://api.example.com"),
  serviceDocumentationUrl: new URL("https://docs.example.com/mcp"),
}));
```

Mounting this single router gives you, for free:

- `/.well-known/oauth-authorization-server` — RFC 8414 server metadata
- `/.well-known/oauth-protected-resource` — RFC 9728 resource metadata that
  tells clients *which* authorization servers to trust
- `/register` — Dynamic Client Registration (SEP-991 Client ID Metadata
  Documents flow)
- The standardized `401 Unauthorized` + `WWW-Authenticate: Bearer
  resource_metadata="..."` response that bootstraps the whole discovery
  dance for a fresh client

`verifyAccessToken` is the only piece you must implement against your IdP.
If your IdP supports introspection, call it; otherwise verify a signed JWT
locally with the IdP's JWKS. See
[../production/02_authentication.md](../production/02_authentication.md) for
the end-to-end picture, including how Claude Desktop and Cursor walk this
flow.

---

## 4. Reading Auth in Tool Handlers

`mcpAuthRouter` only sets up the discovery endpoints. To actually gate the
MCP endpoint you bolt on `requireBearerAuth` as Express middleware. The
verifier surfaces parsed claims on `req.auth` and on the `extra.authInfo`
argument inside every tool, resource, and prompt handler.

```typescript
import { requireBearerAuth } from "@modelcontextprotocol/sdk/server/auth/middleware/bearerAuth.js";

app.post("/mcp", requireBearerAuth({ verifier: oauth }), async (req, res) => {
  // ...
});

server.registerTool(
  "delete_account",
  { /* ... */ },
  async ({ id }, extra) => {
    if (!extra.authInfo?.scopes.includes("admin")) {
      return { isError: true, content: [{ type: "text", text: "admin scope required" }] };
    }
    // ...
  },
);
```

A few patterns that pay off in production:

- Treat `extra.authInfo` as the single source of truth. Don't sniff the raw
  Authorization header inside the handler — `requireBearerAuth` has already
  validated, parsed, and normalized it.
- Push fine-grained checks (per-resource ACLs, tenancy filters) into the
  handler, but keep coarse-grained checks (is this token valid at all?) in
  the middleware. That keeps the handler focused on business logic.
- Return a soft `isError: true` content block instead of throwing for
  authorization failures the model could reasonably retry against (e.g.
  "you need to re-elicit a confirmation"). Throw for hard "this token has
  no business here" denials.
- Cache `verifyAccessToken` results for the token's remaining lifetime
  (clip to a sane max like 5 minutes). JWT verification is cheap; an
  introspection round-trip on every tool call is not.

> **Principle**: authentication is a transport concern; authorization is a
> handler concern. Don't mix them.

The `extra` object also carries `requestId`, `sessionId`, and `signal` (an
`AbortSignal` that fires if the client disconnects). Threading the signal
into your downstream `fetch` calls means a hung client cancellation
actually cancels the underlying API call instead of letting it run to
completion and waste your rate budget.

---

## 5. Composing Multiple `McpServer` Instances

Once you have more than one MCP surface — say a GitHub server and a Jira
server — you have two options. You can build a single "super-server" with
prefixed tools (`github_create_issue`, `jira_create_issue`), or you can keep
each server clean and route at the HTTP edge.

The second approach scales much better. Each `McpServer` stays single-purpose
and independently testable; clients see two distinct MCP servers at two
distinct URLs.

```typescript
const githubServer = makeGithubServer();
const jiraServer = makeJiraServer();

const githubTransports = new Map<string, StreamableHTTPServerTransport>();
const jiraTransports = new Map<string, StreamableHTTPServerTransport>();

app.post("/mcp/:tenant", async (req, res) => {
  const map = req.params.tenant === "github" ? githubTransports : jiraTransports;
  const make = req.params.tenant === "github" ? makeGithubServer : makeJiraServer;
  // ... same dispatch as above using `map` and `make()`
});
```

| Approach            | Pros                                               | Cons                                              |
|---------------------|----------------------------------------------------|---------------------------------------------------|
| Super-server        | One URL to publish; one OAuth registration         | Tool name collisions; one bad tool taints all     |
| Per-tenant routing  | Independent versioning; clean blast radius          | Multiple URLs; client must register each          |

> **Rule**: prefix-based super-servers are a code smell. If you find yourself
> writing `tenant_action` tool names, lift the tenant into the URL path.

The same composition pattern works for sub-products of one logical service.
A "code review" MCP that wraps GitHub, GitLab, and Bitbucket is far cleaner
as `/mcp/github`, `/mcp/gitlab`, `/mcp/bitbucket` — each backed by its own
`McpServer` with its own credentials and rate limits — than as one server
exposing `github_review_pr`, `gitlab_review_mr`, `bitbucket_review_pr`. The
client UX is identical (the user picks which integration to enable); the
server-side blast radius is dramatically smaller.

---

## 6. Dropping to the Low-Level `Server` Class

`McpServer` is an opinionated builder over the lower-level `Server`. It
assumes your tool list is known at construction time, that you want to use
Zod for schemas, and that one `register*` call binds one handler. When any
of those assumptions break, drop down.

The escape hatches you most often need:

- A tool list that depends on the caller's permissions or tenancy and must
  be computed per request
- Hand-routing of `tools/call` through your own dispatch table (e.g. you
  load handlers from a plugin directory at startup)
- Implementing experimental or non-standard methods that `McpServer`
  doesn't expose

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "low-level", version: "1.0.0" },
  { capabilities: { tools: {}, logging: {} } },
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: await dynamicallyComputeTools(),  // e.g. fetched from a database
}));

server.setRequestHandler(CallToolRequestSchema, async (req) => {
  // Hand-route to your handler registry
  const handler = handlers[req.params.name];
  if (!handler) throw new Error(`unknown tool: ${req.params.name}`);
  return await handler(req.params.arguments);
});
```

Notice that you must declare capabilities up front — there's no
`registerTool` to do it for you. If you forget `tools: {}` in the
capabilities object, the client thinks you don't expose tools at all and
won't call `tools/list`.

> **Key insight**: the low-level `Server` is what `McpServer` is built on,
> not a separate product. You can mix them: hand the same transport to an
> `McpServer` for the easy stuff and intercept specific request schemas
> with `setRequestHandler` for the dynamic ones.

A practical heuristic for when to drop down:

| Situation                                                    | Use            |
|--------------------------------------------------------------|----------------|
| Static tool list, fixed schemas, one handler per tool        | `McpServer`    |
| Tool list depends on caller / tenant / time-of-day           | `Server`       |
| You want Zod validation and `outputSchema` enforcement       | `McpServer`    |
| You need to plug your own JSON-Schema validator              | `Server`       |
| You're loading handlers from a plugin directory at startup   | `Server`       |
| You want `notifications/tools/list_changed` to "just work"   | `McpServer`    |

---

## 7. CORS for Browser-Based Hosts

Claude.ai, ChatGPT's MCP integration, and Cursor's web client all run in a
browser. That means your MCP server needs CORS set up correctly — and the
single most common mistake is forgetting to expose the session header.

```typescript
import cors from "cors";

app.use(cors({
  origin: ["https://claude.ai", "https://chatgpt.com", /\.cursor\.com$/],
  exposedHeaders: ["Mcp-Session-Id"],   // critical — clients need to read this
  allowedHeaders: ["Authorization", "Content-Type", "Mcp-Session-Id", "Mcp-Protocol-Version"],
}));
```

The browser will let your server *send* the `Mcp-Session-Id` header, but
unless you list it in `exposedHeaders` the JavaScript client cannot
*read* it from the response. The result: the client never learns its own
session id, every subsequent POST is treated as a new initialize attempt,
and you get a stream of 400 errors that look nothing like a CORS problem.

⚠️ Be conservative with `origin`. Wide-open CORS on an MCP server lets any
website mint requests against your tools using the user's cookies or
in-browser tokens. List the hosts you actually need.

---

## 8. Graceful Shutdown

Streamable HTTP holds long-lived SSE connections. If you let `process.exit()`
fire while requests are in flight, clients see truncated responses and
sessions leak server-side resources (open files, DB transactions). Wire up
SIGTERM and SIGINT handlers that drain in order: sessions first, then HTTP,
then the rest of your I/O.

```typescript
const httpServer = app.listen(8080);

async function shutdown() {
  console.log("shutting down");
  for (const t of transports.values()) await t.close();
  httpServer.close();
  await db.end();
  process.exit(0);
}
process.on("SIGTERM", shutdown);
process.on("SIGINT", shutdown);
```

The order matters: `transport.close()` flushes any pending SSE events to
the client and lets the in-flight `handleRequest` calls return cleanly.
`httpServer.close()` then stops accepting new connections and waits for
existing ones to drain. Closing the database last avoids cancelling a
query that an in-flight tool call still needed.

> **Rule**: tear down sessions before closing the HTTP server. The reverse
> order (`httpServer.close()` first) cuts the socket out from under
> `transport.close()` and you'll see "write after end" errors in your logs.

---

## 9. Testing Pattern — In-Memory Transport

The SDK ships an in-memory transport pair specifically for tests. You wire
a `Client` and a `Server` to opposite ends and exercise the full MCP
protocol without ever touching a socket or spawning a subprocess. Tests run
in milliseconds, integrate cleanly with vitest/jest, and exercise exactly
the same code paths that production uses.

```typescript
import { InMemoryTransport } from "@modelcontextprotocol/sdk/inMemory.js";

const [client, server] = InMemoryTransport.createLinkedPair();
await mcpServer.connect(server);
await mcpClient.connect(client);
const result = await mcpClient.callTool({ name: "add", arguments: { a: 2, b: 3 } });
```

This is also the right place to test sampling, elicitation, and
notifications — anything that's awkward over real HTTP because it
involves the server initiating a request to the client. The Python
equivalent is documented in
[../python/07_clients_advanced.md](../python/07_clients_advanced.md).

> **Tip**: write at least one integration test per tool that goes through
> the full client/server round trip. Unit tests of the handler in isolation
> miss schema-validation bugs, capability-negotiation mistakes, and
> serialization edge cases.

A useful test fixture is to spin up the in-memory pair in a `beforeEach`,
register your tools, and yield a fully-initialized `Client` to each test.
Tests stay focused on the protocol-visible behavior — what arguments the
client sends, what content blocks come back — rather than poking at
internal handler implementations that can refactor freely.

---

## 10. Failure Modes

A grab bag of bugs that have actually shipped to production. None of them
surface as obvious errors — they all look like "it works on my machine."

- ❌ **Reusing a single `StreamableHTTPServerTransport` for multiple
  sessions** — sessions cross-contaminate. Each session needs its own
  transport instance, created inside the POST handler when
  `isInitializeRequest` returns true.
- ❌ **Forgetting to handle GET (`/mcp`)** — the server can't push
  notifications, so `notifications/tools/list_changed` and
  `notifications/resources/updated` silently disappear. The client never
  knows your tool list changed and keeps calling stale tools.
- ❌ **Missing `exposedHeaders: ["Mcp-Session-Id"]` in CORS** — browser
  clients can't read the session id, so every request looks like a fresh
  initialize. You'll see a flood of 400s from claude.ai or chatgpt.com but
  curl from your laptop will work fine.
- ⚠️ **Stateful mode without sticky sessions** — every other request lands
  on a different replica with no entry in its `Map`, returning 404. Looks
  like a flaky network until you correlate with replica id.
- ❌ **Verifying tokens manually instead of using `requireBearerAuth`** —
  easy to miss the RFC 9728 metadata flow. Clients then can't discover
  your authorization server and you fall back to ad-hoc browser pop-ups
  or hand-distributed tokens.
- ⚠️ **Not setting `res.on("close", ...)` in stateless mode** — long-running
  tool calls leak transports when the client hangs up before the response
  finishes. Memory creeps up over hours, then the process OOMs at 3 a.m.
- ❌ **Mounting `mcpAuthRouter` *after* `requireBearerAuth`** — the
  discovery endpoints get gated behind auth they were meant to bootstrap.
  Order of `app.use` calls matters; mount the router unauthenticated, then
  apply `requireBearerAuth` only to `/mcp`.

---

**Next**: [Production: Streamable HTTP](../production/01_streamable_http.md)
