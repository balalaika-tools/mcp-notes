# Tools with the TypeScript SDK — Zod, Async, Structured Output

> **Who this is for**: TypeScript engineers who finished [01_getting_started.md](01_getting_started.md)
> and want to ship production-grade MCP tools. You should be comfortable with Zod, `async`/`await`,
> and basic JSON Schema. For the conceptual model of tools as a primitive, see
> [../fundamentals/03_primitives.md](../fundamentals/03_primitives.md). For the side-by-side with
> Python, see [../reference/02_python_vs_typescript.md](../reference/02_python_vs_typescript.md).

The official `@modelcontextprotocol/sdk` (1.29+) treats tools as the workhorse of your server:
typed inputs, typed outputs, async handlers, and rich metadata. This file covers everything you
need to build them well.

---

## 1. `registerTool` — The Canonical API

`registerTool` is the modern, metadata-rich way to declare a tool. The `inputSchema` is a Zod
*shape* (a plain object mapping keys to Zod types), not a `ZodObject`. The SDK builds both the
JSON Schema that ships to clients and the runtime validator from that shape.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "calc", version: "1.0.0" });

server.registerTool(
  "add",
  {
    title: "Add",
    description: "Add two numbers.",
    inputSchema: {
      a: z.number().describe("First operand"),
      b: z.number().describe("Second operand"),
    },
  },
  async ({ a, b }) => ({
    content: [{ type: "text", text: String(a + b) }],
  }),
);
```

A few details that matter:

- `title` is a **human-friendly label** for hosts (Claude Desktop shows it in the UI). `description`
  is what the model sees when deciding whether to call the tool — write it for an LLM, not a human.
- `.describe(...)` on each Zod field becomes the JSON Schema `description` for that property.
  Models read these. Don't skip them on non-obvious arguments.
- The handler return type is `{ content: ContentPart[]; isError?: boolean; structuredContent?: ... }`.
  `content` is **always an array**, even for a single text part.

> **Key insight**: the Zod shape is the single source of truth. The TS compiler infers the
> handler's input parameter type from it, the SDK validates incoming arguments against it at
> runtime, and clients receive a JSON Schema generated from it. One declaration, three jobs.

---

## 2. `tool()` — The Older, Shorter API

`tool()` predates `registerTool` and is still supported. It takes positional arguments instead of
a metadata object:

```typescript
server.tool(
  "add",
  "Add two numbers.",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => ({ content: [{ type: "text", text: String(a + b) }] }),
);
```

| API              | When to use                                            |
|------------------|--------------------------------------------------------|
| `registerTool`   | Production servers — supports `title`, `outputSchema`, `annotations` |
| `tool()`         | Quick prototypes, examples, throwaway scripts          |

Prefer `registerTool` everywhere new. `tool()` can't express output schemas or annotations, so
you'll outgrow it the first time you need structured output.

---

## 3. Async Handlers

Every handler is `async`. The SDK awaits whatever you return, so use `await` freely — for HTTP
calls, database queries, file I/O, anything. Just don't forget the `await`.

```typescript
import { fetch } from "undici";

server.registerTool(
  "fetch_status",
  {
    title: "Fetch URL Status",
    description: "Fetch a URL and return its HTTP status code.",
    inputSchema: { url: z.string().url() },
  },
  async ({ url }) => {
    // AbortSignal.timeout caps the request — without it a hung server would hang the tool call
    const r = await fetch(url, { signal: AbortSignal.timeout(5_000) });
    return { content: [{ type: "text", text: `HTTP ${r.status}` }] };
  },
);
```

Two things worth internalizing:

1. **Always set a timeout** on outbound network calls. MCP clients usually have their own
   tool-call timeouts, but a hung handler still wastes the slot and burns user patience.
2. **Don't block the event loop.** If you do CPU-heavy work (parsing huge files, image processing),
   move it to a worker thread. A blocked Node event loop blocks every other tool call too.

---

## 4. Complex Inputs with Zod

Zod's strength shows up the moment your tool takes more than two scalars. Defaults, constraints,
nested objects, enums — all expressed once and enforced everywhere.

```typescript
const CreateIssueShape = {
  title: z.string().min(1).max(200),
  body: z.string().default(""),
  labels: z.array(z.string()).default([]),
  priority: z.number().int().min(0).max(3).default(1),
  assignee: z.string().optional(),
};

server.registerTool(
  "create_issue",
  {
    title: "Create Issue",
    description: "Create a GitHub issue in the configured repository.",
    inputSchema: CreateIssueShape,
  },
  async (input) => {
    // input is fully typed:
    //   { title: string;
    //     body: string;
    //     labels: string[];
    //     priority: number;
    //     assignee?: string; }
    const issue = await github.createIssue(input);
    return { content: [{ type: "text", text: `Created #${issue.number}` }] };
  },
);
```

> **Critical**: TypeScript infers the handler's parameter type from the Zod shape. You get full
> intellisense, no `any`, and refactors that touch the schema break the handler at compile time.
> This is the single biggest reason to use the TS SDK over hand-rolling JSON-RPC.

A few Zod patterns that come up repeatedly:

```typescript
// Enums — generates a JSON Schema "enum" array
status: z.enum(["open", "closed", "draft"]),

// Discriminated unions — for "type" fields with different shapes
event: z.discriminatedUnion("type", [
  z.object({ type: z.literal("push"), sha: z.string() }),
  z.object({ type: z.literal("pr"),   number: z.number() }),
]),

// Coercion — accept "5" as a number (use sparingly; LLMs usually send the right type)
limit: z.coerce.number().int().positive().max(100).default(10),
```

---

## 5. Structured Output with `outputSchema`

If your tool returns data the model (or a downstream tool) will operate on, declare an
`outputSchema` and populate `structuredContent`. The model still gets a human-readable text part,
but it also receives validated JSON it can pass to the next tool call without re-parsing.

```typescript
const IssueResultShape = {
  number: z.number(),
  url: z.string().url(),
  createdAt: z.string(),
};

server.registerTool(
  "create_issue",
  {
    title: "Create Issue",
    description: "Create a GitHub issue and return its identifiers.",
    inputSchema: CreateIssueShape,
    outputSchema: IssueResultShape,
  },
  async (input) => {
    const issue = await github.createIssue(input);
    const out = {
      number: issue.number,
      url: issue.htmlUrl,
      createdAt: issue.createdAt,
    };
    return {
      content: [{ type: "text", text: `Created #${issue.number}` }],
      structuredContent: out, // validated against IssueResultShape before being sent
    };
  },
);
```

> **Rule**: If `outputSchema` is set, you **must** return `structuredContent`, and it must match.
> The SDK validates it server-side and will surface a clear error if it doesn't. Don't lie about
> your output shape.

When to bother with `outputSchema`:

- ✅ Tool returns IDs, URLs, or fields that the next tool call will need
- ✅ Tool is part of a chain (search → fetch → analyze)
- ❌ Tool just renders text for the user ("here's the weather forecast")

---

## 6. Errors — The Right Way

The `2025-11-25` spec revision (SEP-1303) clarified the distinction that already existed in the
SDK: **validation errors are tool errors, not protocol errors**. They flow back to the model as
`isError: true` so the model can self-correct, instead of aborting the JSON-RPC call.

There are two ways to signal a tool error from your handler:

**1. Return `isError: true` explicitly** (preferred — you control the message):

```typescript
server.registerTool(
  "transfer",
  {
    title: "Transfer Funds",
    description: "Transfer funds between two accounts.",
    inputSchema: {
      from: z.string(),
      to: z.string(),
      amount: z.number().positive(),
    },
  },
  async ({ from, to, amount }) => {
    const acct = await db.getAccount(from);
    if (!acct) {
      return {
        content: [{ type: "text", text: `unknown account: ${from}` }],
        isError: true,
      };
    }
    if (acct.balance < amount) {
      return {
        content: [{
          type: "text",
          text: `insufficient funds (${acct.balance} < ${amount})`,
        }],
        isError: true,
      };
    }
    await db.transfer(from, to, amount);
    return { content: [{ type: "text", text: "transferred" }] };
  },
);
```

**2. Throw an `Error`** — the SDK catches it, wraps it in a tool error result, and includes the
message:

```typescript
if (!acct) throw new Error(`unknown account: ${from}`);
```

| Approach            | When to use                                                  |
|---------------------|--------------------------------------------------------------|
| `return isError`    | Expected business errors (validation, missing record, 404)   |
| `throw new Error`   | Unexpected failures (network down, DB unreachable, bug)      |

> **Key insight**: an error message is a prompt. The model sees it and decides what to do next.
> "insufficient funds (50 < 100)" lets the model adjust and retry; "ERR_BAL_LOW" makes it guess.
> Write error messages like you're writing instructions for a slightly distracted intern.

---

## 7. Tool Annotations — Hints to Hosts

Annotations are non-binding hints that tell the host **what kind of tool this is**. Hosts use
them to decide when to prompt the user for confirmation, when to run a tool automatically, and
how to display it in the UI.

```typescript
server.registerTool(
  "get_user",
  {
    title: "Get User",
    description: "Read a user record by ID.",
    inputSchema: { id: z.string() },
    annotations: {
      readOnlyHint: true,      // doesn't mutate state
      destructiveHint: false,  // safe to retry / undo unnecessary
      idempotentHint: true,    // calling twice == calling once
      openWorldHint: false,    // operates on a closed world (our DB only)
    },
  },
  async ({ id }) => ({
    content: [{ type: "text", text: JSON.stringify(await db.user(id)) }],
  }),
);
```

| Annotation         | Meaning                                                                |
|--------------------|------------------------------------------------------------------------|
| `readOnlyHint`     | Tool only reads — no state change                                      |
| `destructiveHint`  | Tool can delete or overwrite data (host should warn before calling)    |
| `idempotentHint`   | Calling N times has the same effect as calling once                    |
| `openWorldHint`    | Tool touches the open internet (vs. your closed system)                |

⚠️ **Always set `destructiveHint: true`** on tools that delete, overwrite, or mutate external
state. Hosts default to "safe unless told otherwise"; without the hint, the user won't get a
confirmation prompt before your tool drops their production database.

---

## 8. Multiple Content Parts

A tool can return any mix of text, images, and embedded resources. The model receives them all.

```typescript
return {
  content: [
    { type: "text", text: "Here's the chart." },
    { type: "image", data: pngBase64, mimeType: "image/png" },
    {
      type: "resource",
      resource: {
        uri: "report://q1",
        mimeType: "text/markdown",
        text: reportMd,
      },
    },
  ],
};
```

When to reach for each:

- **`text`** — the default; what the model reads.
- **`image`** — base64-encoded; the model sees it if it's vision-capable. Always set `mimeType`.
- **`resource`** — embed a resource the model (or user) can reference later by URI. Useful when
  your tool generates a document the user might want to revisit. See
  [03_resources_prompts.md](03_resources_prompts.md) for more on resources.

---

## 9. Updating Tools at Runtime

`registerTool` returns a handle you can use to disable, re-enable, or update the tool while the
server is running. The SDK emits `notifications/tools/list_changed` automatically so connected
clients refresh their tool list.

```typescript
const handle = server.registerTool(
  "experimental_search",
  {
    title: "Experimental Search",
    description: "Behind a feature flag.",
    inputSchema: { query: z.string() },
  },
  async ({ query }) => ({ content: [{ type: "text", text: await search(query) }] }),
);

if (!featureFlags.experimentalSearch) {
  handle.disable();          // stops advertising; calls return an error
}

// Later — flag flipped on
handle.enable();

// Or update metadata in place
handle.update({ description: "Now generally available." });
```

Use cases:

- **Feature flags** — gate tools per deployment without restarting the server
- **Per-user permissions** — disable destructive tools for read-only sessions (rebuild the server
  per session is usually cleaner, but `disable()` works for shared instances)
- **Hot config reloads** — update `description` when the underlying behavior changes

---

## 10. Failure Modes

The mistakes that show up in code review again and again:

❌ **Forgetting `await` on a promise inside the handler.** The handler returns before the work
finishes; the response is wrong or empty; downstream calls fail mysteriously.

```typescript
// ❌ Returns before the DB write completes
async ({ id }) => {
  db.delete(id);  // missing await — fire-and-forget
  return { content: [{ type: "text", text: "deleted" }] };
}

// ✅
async ({ id }) => {
  await db.delete(id);
  return { content: [{ type: "text", text: "deleted" }] };
}
```

❌ **Returning `{ content: ... }` without the array brackets.** `content` is always an array.
The SDK rejects a bare object with a validation error.

```typescript
// ❌
return { content: { type: "text", text: "hi" } };
// ✅
return { content: [{ type: "text", text: "hi" }] };
```

❌ **Using `z.object({...})` for `inputSchema`.** Pass the *shape* (the plain object of keys to
Zod types), not the wrapped `ZodObject`. The SDK wraps it for you.

```typescript
// ❌
inputSchema: z.object({ a: z.number(), b: z.number() }),
// ✅
inputSchema: { a: z.number(), b: z.number() },
```

⚠️ **Throwing strings or non-`Error` values.** `throw "bad"` works in JavaScript but the SDK
can't extract a `.message`, so the model receives a useless empty error. Always throw `Error`
instances.

```typescript
// ❌
throw "user not found";
// ✅
throw new Error("user not found");
```

⚠️ **Skipping `destructiveHint: true` on mutating tools.** Hosts default to treating tools as
safe. Without the hint, users won't see confirmation prompts before destructive actions —
exactly the kind of thing that ends up in a postmortem.

⚠️ **Letting a slow handler hang the server.** The MCP client typically has a tool-call timeout
(60s default in many hosts). If your handler doesn't have its own timeout, you'll burn the slot
and frustrate the user. Wrap external calls in `AbortSignal.timeout(...)`.

💡 **Tip**: when debugging, log the JSON-RPC frames. Set `MCP_DEBUG=1` (or whatever your
transport supports) and watch what's actually flowing over stdio. Most "tool not being called"
issues turn out to be schema validation errors the LLM silently routed around.

---

**Next**: [03_resources_prompts.md — Resources and Prompts](03_resources_prompts.md)
