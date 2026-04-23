# Getting Started with the TypeScript MCP SDK

> **Who this is for**: Node/TypeScript developers comfortable with ESM and Zod who have never
> built an MCP server. Assumes Node 20+, an editor with TS support, and basic familiarity with
> `npx`. If you've already worked through the Python track ([../python/01_getting_started.md](../python/01_getting_started.md)),
> this is the TS-shaped equivalent — same shape, different ergonomics.

The official SDK is `@modelcontextprotocol/sdk`. It's the same project Anthropic ships
the Python SDK from, and the public surface mirrors it closely: high-level `McpServer`
and `Client` classes, transports for stdio and Streamable HTTP, and Zod-driven schemas
that get serialized to JSON Schema for the wire.

---

## 1. Install

```bash
npm install @modelcontextprotocol/sdk zod
# or
pnpm add @modelcontextprotocol/sdk zod
```

Latest at the time of writing is **1.29.0**. The 1.x line is the recommended target for
production through Q1 2026 — a stable 2.0 is expected around then with breaking changes
to the low-level transport contract. New projects should pin a `^1.29` range and plan a
migration window when 2.0 lands.

Runtime and project requirements:

- **Node 20 or later.** The SDK uses `AbortSignal.timeout`, native `fetch`, and
  `node:crypto.randomUUID()` without polyfills. Node 18 is EOL and will not work cleanly.
- **ESM only.** The SDK is published as ESM. Your `package.json` must contain
  `"type": "module"`. CommonJS consumers need a dynamic `import()` and lose top-level
  `await`.
- **Zod is a peer dependency.** It's listed separately so the SDK doesn't pin a Zod
  version that conflicts with yours. Use Zod v3.

A minimum `package.json` looks like this:

```json
{
  "name": "weather-demo",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.29.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "tsx": "^4.19.0",
    "typescript": "^5.6.0",
    "@types/node": "^20.14.0"
  }
}
```

---

## 2. Hello-World Server

Create `server.ts`. This is a complete, runnable MCP server that exposes one tool:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "weather-demo",
  version: "1.0.0",
});

server.registerTool(
  "get_temperature",
  {
    title: "Get Temperature",
    description: "Return the current temperature in a city.",
    inputSchema: {
      city: z.string().describe("City name"),
    },
  },
  async ({ city }) => ({
    content: [{ type: "text", text: `It is 22°C in ${city}.` }],
  }),
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

A few things worth pointing out:

- **`McpServer` is the high-level API.** It owns the registry of tools, resources, and
  prompts and wires them to the transport for you. There's also a low-level `Server`
  class in `@modelcontextprotocol/sdk/server/index.js` if you need to handle raw MCP
  request types yourself — it's covered in `05_advanced.md`. For 95% of servers, stick
  with `McpServer`.
- **`registerTool` takes `(name, definition, handler)`.** The definition object holds
  `title`, `description`, `inputSchema`, and optional `annotations`. The handler is an
  async function that receives the parsed, type-checked arguments.
- **`inputSchema` is a Zod *shape*, not a full Zod object.** You pass the inner record
  (`{ city: z.string() }`), not `z.object({...})`. The SDK wraps it and converts it to
  JSON Schema for the `tools/list` response automatically. The handler's parameter type
  is inferred from this shape — `city` is `string`, no manual typing needed.
- **The handler returns a `CallToolResult`.** `{ content: [...] }` with one or more
  content blocks. `type: "text"` is the common case; image and resource blocks are also
  valid. Throwing from the handler returns a tool error to the client; returning
  `{ isError: true, content: [...] }` returns a tool error with content.
- **`.js` extensions in imports.** This is ESM under TypeScript with `"module": "Node16"`.
  You write `.js` even though the source is `.ts`. The SDK does this internally too.

> **Key insight**: `McpServer` is mostly a thin wrapper that turns ergonomic registration
> calls into the JSON-RPC handlers that `Server` would otherwise require you to write by
> hand. When something is mysterious, drop one level down and look at the `Server` source
> — it's small.

---

## 3. Run It

Three ways, in order of how often you'll use each:

```bash
# Dev: tsx runs TypeScript directly with no build step
npx tsx server.ts

# Production: compile, then run plain Node
npx tsc
node dist/server.js

# Inspect: the official MCP Inspector UI for poking at tools/resources/prompts
npx @modelcontextprotocol/inspector npx tsx server.ts
```

The Inspector spawns your server, connects over stdio, and gives you a web UI at
`http://localhost:6274` (or whatever it prints). You can list tools, call them with
arbitrary arguments, watch the JSON-RPC traffic, and replay requests. It is the single
most useful debugging tool for MCP — use it before wiring into Claude Desktop, every time.

⚠️ Anything your server writes to **stdout** that isn't JSON-RPC will corrupt the protocol
and the client will disconnect. Keep `console.log` out of stdio servers — use
`console.error` (stderr) or a real logger that writes to a file. This is the #1 source of
"my server appears in Claude Desktop but immediately disappears" reports.

---

## 4. Streamable HTTP Server

Stdio is great for local tools and Claude Desktop. For network-accessible servers, use
**Streamable HTTP** — the MCP transport that replaced the older SSE transport in spec
revision 2025-03-26. It's a single endpoint that handles both POST (request) and
GET (server-sent events for streaming), with optional session affinity.

The pattern looks like this with Express:

```typescript
import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { randomUUID } from "node:crypto";
import { z } from "zod";

const app = express();
app.use(express.json());

// One transport per session. Stateful pattern: the client holds a session ID
// across requests, and we route to the same in-memory transport+server pair.
const transports: Record<string, StreamableHTTPServerTransport> = {};

app.all("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  let transport = sessionId ? transports[sessionId] : undefined;

  if (!transport) {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (id) => {
        transports[id] = transport!;
      },
    });

    // Each session gets its own McpServer instance — keeps per-session state
    // (auth context, subscriptions, in-flight requests) isolated.
    const server = makeServer();
    await server.connect(transport);
  }

  await transport.handleRequest(req, res, req.body);
});

function makeServer() {
  const s = new McpServer({ name: "weather-demo", version: "1.0.0" });
  s.registerTool(
    "get_temperature",
    {
      title: "Get Temperature",
      description: "Return the current temperature in a city.",
      inputSchema: { city: z.string() },
    },
    async ({ city }) => ({
      content: [{ type: "text", text: `It is 22°C in ${city}.` }],
    }),
  );
  return s;
}

app.listen(8080, () => {
  console.error("MCP server listening on http://localhost:8080/mcp");
});
```

A few production considerations even at this stage:

- **`app.all("/mcp")`** because the transport handles GET, POST, and DELETE on the same
  path. Don't split them across separate route handlers — `transport.handleRequest`
  dispatches internally based on method.
- **The `transports` map leaks if you never clean up.** Wire `transport.onclose` to
  `delete transports[id]` in production (covered in `05_advanced.md`).
- **Stateless mode** is the alternative: pass `sessionIdGenerator: undefined` and
  construct a fresh transport+server per request. Simpler, no session storage, but you
  lose server-initiated streaming and any per-session state. Good for serverless.
  Detailed in `05_advanced.md`.

> **Rule**: in stateful mode, the client *must* echo the `Mcp-Session-Id` header it
> received during initialization on every subsequent request. If it doesn't, you'll see
> a new session per request and silently waste memory.

---

## 5. Hello-World Client

Most engineers start by writing servers, but writing a client first is the fastest way
to understand the protocol — you see exactly what your server returns.

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

// Stdio client *spawns* the server as a subprocess and talks to it over the
// child's stdin/stdout. The client owns the process lifecycle.
const transport = new StdioClientTransport({
  command: "npx",
  args: ["tsx", "server.ts"],
});

const client = new Client({ name: "demo-client", version: "1.0.0" });
await client.connect(transport);

const tools = await client.listTools();
console.log(tools);
// → { tools: [ { name: "get_temperature", description: "...", inputSchema: {...} } ] }

const result = await client.callTool({
  name: "get_temperature",
  arguments: { city: "Lisbon" },
});
console.log(result.content[0]);
// → { type: "text", text: "It is 22°C in Lisbon." }

await client.close();
```

Notes:

- **`client.connect(transport)` performs the MCP `initialize` handshake** — protocol
  version negotiation, capability exchange — before resolving. After it resolves,
  `client.serverCapabilities` and `client.serverVersion` are populated.
- **`callTool` arguments are not type-checked at compile time.** The client doesn't
  know your server's schemas at TypeScript build time. If you control both ends,
  generate types from the server schema (or share them via a workspace package).
- **Always `await client.close()`** — for stdio, this kills the child process. Skip it
  and you'll leak processes during dev.

---

## 6. HTTP Client

Same `Client` class, different transport:

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const transport = new StreamableHTTPClientTransport(
  new URL("http://localhost:8080/mcp"),
);

const client = new Client({ name: "demo", version: "1.0.0" });
await client.connect(transport);

const tools = await client.listTools();
console.log(tools);

await client.close();
```

The transport handles session-ID tracking automatically — it stores the
`Mcp-Session-Id` returned by the server during init and echoes it on every subsequent
request. You don't manage that header yourself.

For authenticated servers, pass headers via the second constructor argument:

```typescript
const transport = new StreamableHTTPClientTransport(
  new URL("https://api.example.com/mcp"),
  {
    requestInit: {
      headers: { Authorization: `Bearer ${token}` },
    },
  },
);
```

OAuth flows (the spec defines a complete auth model on top of HTTP) are covered in
`05_advanced.md`.

---

## 7. Async Handlers

Every handler in the TypeScript SDK is `async`. There's no sync/async split like the
Python SDK has, because Node is single-threaded and async-first by design — synchronous
work blocks the event loop, period.

```typescript
// ✅ Idiomatic — async handler, await I/O
server.registerTool("fetch_user", { /* ... */ }, async ({ id }) => {
  const user = await db.users.findById(id);
  return { content: [{ type: "text", text: JSON.stringify(user) }] };
});

// ❌ Synchronous heavy work blocks every other in-flight request
server.registerTool("hash_file", { /* ... */ }, ({ path }) => {
  const data = fs.readFileSync(path);          // blocks
  const hash = crypto.createHash("sha256")     // blocks
    .update(data)
    .digest("hex");
  return { content: [{ type: "text", text: hash }] };
});
```

For genuinely CPU-bound work, use `node:worker_threads` or shell out to a child process.
The handler itself stays async; the work happens off the main thread.

> **Key insight**: Python's "do I make this `async def` or `def`?" question doesn't exist
> here. Always `async`. Always `await`. The SDK assumes it.

---

## 8. Wire It to Claude Desktop

To use your server from the Claude Desktop app, edit
`~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or
`%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "weather-demo": {
      "command": "npx",
      "args": ["tsx", "/abs/path/to/server.ts"]
    }
  }
}
```

Then **fully quit and restart** Claude Desktop — `Cmd-Q`, not just closing the window.
Your server should appear in the tools picker (the small hammer icon in the input area).

Production deployments typically compile to JS first and point `command` at `node`:

```json
{
  "mcpServers": {
    "weather-demo": {
      "command": "node",
      "args": ["/abs/path/to/dist/server.js"],
      "env": {
        "NODE_ENV": "production",
        "WEATHER_API_KEY": "..."
      }
    }
  }
}
```

⚠️ Paths must be **absolute**. Claude Desktop launches the command from its own working
directory, not yours. Relative paths silently fail to start.

If the server doesn't appear, check the Claude Desktop logs at
`~/Library/Logs/Claude/mcp-server-weather-demo.log` — every stderr line your server
writes ends up there, which is why you should be writing logs to stderr in the first
place.

---

## 9. Project Setup Checklist

Before you ship anything, verify:

| Setting                  | Where                | Value                                    |
|--------------------------|----------------------|------------------------------------------|
| Node version             | `engines` / CI       | `>=20.0.0`                               |
| Module type              | `package.json`       | `"type": "module"`                       |
| TS target                | `tsconfig.json`      | `"target": "ES2022"`                     |
| TS module                | `tsconfig.json`      | `"module": "Node16"` or `"NodeNext"`     |
| Module resolution        | `tsconfig.json`      | `"moduleResolution": "Node16"`           |
| Strict mode              | `tsconfig.json`      | `"strict": true`                         |
| Import extensions        | source files         | `.js` (yes, even from `.ts`)             |

A minimal `tsconfig.json` that just works:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "declaration": true,
    "sourceMap": true
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

The SDK's `.d.ts` files are excellent — hover any export in your editor and you'll get
the full signature and JSDoc. Lean on this. There's almost no need to read the source
to figure out what a function takes.

💡 If you see `Cannot find module '@modelcontextprotocol/sdk/server/mcp.js' or its
corresponding type declarations`, you've got `"moduleResolution": "node"` (the legacy
algorithm). Switch to `Node16` or `NodeNext` — the SDK uses package `exports` and the
old resolver doesn't understand them.

---

## 10. Next Steps

You now have a working server, a working client, both transports, and a wired-up Claude
Desktop. The next file goes deep on the tools API — Zod schemas, output schemas, tool
annotations, error handling, and the structured-content patterns that production tools
need.

If you came here from the Python track, the closest reference points are:

- [../python/01_getting_started.md](../python/01_getting_started.md) — same content, Python shape
- [../python/07_clients_advanced.md](../python/07_clients_advanced.md) — sampling, roots, elicitation (TS equivalents in `04_clients.md`)
- [../reference/02_python_vs_typescript.md](../reference/02_python_vs_typescript.md) — side-by-side API comparison

---

**Next**: [02_tools_zod.md — Tools with Zod](02_tools_zod.md)
