# Resources and Prompts in `@modelcontextprotocol/sdk`

> **Who this is for**: TypeScript engineers who have finished
> [02_tools_zod.md](02_tools_zod.md) and can register tools with Zod schemas.
> You know `McpServer`, `registerTool`, and how a stdio or HTTP transport hangs
> off the server. Now you need the other two primitives — **resources** and
> **prompts** — and the patterns that go with them.

This file mirrors [`../python/03_resources_prompts.md`](../python/03_resources_prompts.md)
for the official `@modelcontextprotocol/sdk` (1.29+). The mental model is the
same across SDKs; only the API surface changes.

---

## 1. Quick Decision Matrix

The three MCP primitives look similar on the wire but answer very different
questions. Pick the wrong one and you'll fight the protocol forever.

| Need                                                | Use          |
|-----------------------------------------------------|--------------|
| Action the model decides to invoke                  | **Tool**     |
| Data the host should attach to the LLM context      | **Resource** |
| Reusable prompt template the user picks             | **Prompt**   |

> **Rule of thumb**: if the *model* triggers it, it's a tool. If the *host*
> attaches it, it's a resource. If the *user* picks it from a menu, it's a
> prompt.

A search API is a **tool** (the model decides when to search). A README is a
**resource** (the host can pin it into context). "Review this PR" is a
**prompt** (the user picks it from a slash-command menu and the template
expands into messages).

---

## 2. Static Resource

The simplest resource: one URI, one handler, returns one chunk of text. Use
this for things like a project README, a config file, or any document with a
fixed identity.

```typescript
import { McpServer, ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";
import { readFile } from "node:fs/promises";

server.registerResource(
  "readme",
  "docs://readme",
  {
    title: "Project README",
    description: "Top-level README",
    mimeType: "text/markdown",
  },
  async (uri) => ({
    contents: [{
      uri: uri.href,
      mimeType: "text/markdown",
      text: await readFile("/path/to/README.md", "utf8"),
    }],
  }),
);
```

The handler returns `{ contents: [...] }` — note the plural; one resource read
can return multiple parts. That's deliberate: a "resource" is a logical asset,
not a single file. A PDF resource might return a text extraction *and* the
binary blob; a directory resource might return a listing across several files.

> **Key insight**: `uri` inside the handler is a `URL` object, not a string.
> Use `uri.href` when you echo it back in `contents[].uri`.

---

## 3. Templated Resources with `ResourceTemplate`

Static URIs only work for assets you can enumerate at registration time. For
parameterized resources — issues, users, file paths — use `ResourceTemplate`
with [RFC 6570](https://www.rfc-editor.org/rfc/rfc6570) URI templates:

```typescript
import { ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";

server.registerResource(
  "github_issue",
  new ResourceTemplate("github://repos/{owner}/{repo}/issues/{number}", { list: undefined }),
  {
    title: "GitHub Issue",
    description: "A specific GitHub issue",
    mimeType: "application/json",
  },
  async (uri, vars) => {
    const { owner, repo, number } = vars as { owner: string; repo: string; number: string };
    const r = await fetch(`https://api.github.com/repos/${owner}/${repo}/issues/${number}`);
    const data = await r.json();
    return {
      contents: [{
        uri: uri.href,
        mimeType: "application/json",
        text: JSON.stringify(data, null, 2),
      }],
    };
  },
);
```

A few things worth calling out:

- **`vars` is loosely typed.** The SDK gives you `Record<string, string | string[]>`;
  cast to your concrete shape (or validate with Zod) before use. Each
  `{name}` in the template becomes a key.
- **`list: undefined`** opts out of enumeration. If your URI space is finite
  and small, pass a function that returns concrete instances — the host can
  then show them in a picker. For a GitHub issues template, enumeration is
  unbounded, so leave it `undefined`.
- **The URI scheme is yours.** `github://`, `docs://`, `db://` — pick
  something distinctive so URIs in tool results are unambiguous.

> **Principle**: templates expand a single registration into an entire URI
> space. Don't register one resource per row in a database — register one
> template that takes the row id.

---

## 4. Binary Content — base64-encoded `blob`

Resources aren't text-only. For images, PDFs, archives, and anything else
that isn't UTF-8, use the `blob` field instead of `text`:

```typescript
server.registerResource(
  "photo",
  new ResourceTemplate("photos://{id}", { list: undefined }),
  { title: "Photo", mimeType: "image/jpeg" },
  async (uri, { id }) => {
    const bytes = await fetchPhoto(id as string);
    return {
      contents: [{
        uri: uri.href,
        mimeType: "image/jpeg",
        blob: Buffer.from(bytes).toString("base64"),
      }],
    };
  },
);
```

> **Rule**: use `text` for UTF-8 content, `blob` for everything else. Setting
> both is undefined behavior; the host will pick one and the other will be
> silently ignored.

The `blob` field must be **base64-encoded**. The protocol is JSON over the
wire, so raw bytes aren't an option. Node's `Buffer.from(bytes).toString("base64")`
is the canonical conversion.

---

## 5. Prompts — Basics

Prompts are reusable message templates that the *user* invokes — typically
from a slash-command menu in the host UI. Unlike tools, the model never sees
prompts as callable; the host expands them into messages and feeds those
messages to the model.

```typescript
import { z } from "zod";

server.registerPrompt(
  "review_pr",
  {
    title: "Review Pull Request",
    description: "Generate a thorough code review",
    argsSchema: {
      diff: z.string().describe("Unified diff to review"),
      focus: z.string().optional().describe("Optional focus areas, comma-separated"),
    },
  },
  ({ diff, focus }) => ({
    messages: [
      {
        role: "user",
        content: {
          type: "text",
          text: `You are a senior engineer doing a thorough code review.\n\nReview this diff:\n\n${diff}\n\nFocus on: ${focus || "general quality"}`,
        },
      },
    ],
  }),
);
```

Prompts return `{ messages: [...] }` — typically a list of user/assistant turns
ready to feed to a chat completion. The host passes these straight to the
model; the prompt handler is just a string-templating layer with typed args.

> **Key insight**: a prompt is not the system prompt. It's a starter for the
> *conversation*. If you want to pin instructions for the whole session, that's
> the host's job, not a prompt's.

---

## 6. Multi-Message Prompts

Real prompts often need multiple turns — prior context, then a new question;
or a few-shot example before the actual ask. The `messages` array is ordered
exactly as it'll be sent to the model:

```typescript
server.registerPrompt(
  "debug_with_context",
  {
    title: "Debug With Context",
    description: "Debug an error with prior session context",
    argsSchema: { error: z.string(), priorWork: z.string().optional() },
  },
  ({ error, priorWork }) => ({
    messages: [
      ...(priorWork ? [{ role: "user" as const, content: { type: "text" as const, text: `Earlier context:\n${priorWork}` } }] : []),
      { role: "user" as const, content: { type: "text" as const, text: `I'm seeing this error:\n\n${error}\n\nWhat's the most likely cause?` } },
    ],
  }),
);
```

Note the `as const` casts — TS is strict about literal types here. Without
them, TypeScript widens `"user"` to `string` and `"text"` to `string`, which
fails the SDK's discriminated-union schema validation. The error message is
not always obvious, so it's worth the muscle memory.

> **Principle**: keep prompt templates *focused*. A prompt that pulls in an
> entire conversation history defeats the point — the host can already do
> that. Prompts should be small, composable starters.

---

## 7. Argument Completion (for Slash-Command UX)

When the user is typing arguments into a prompt's UI, the host can ask the
server for completion suggestions. This is what powers slash-command
autocomplete in clients like Claude Desktop and Cursor.

```typescript
import { Completable } from "@modelcontextprotocol/sdk/server/completable.js";

server.registerPrompt(
  "deploy",
  {
    title: "Deploy",
    description: "Deploy a service to an environment",
    argsSchema: {
      env: new Completable(z.string(), () => ["dev", "staging", "prod"]),
      service: z.string(),
    },
  },
  ({ env, service }) => ({
    messages: [{ role: "user", content: { type: "text", text: `Deploy ${service} to ${env}.` } }],
  }),
);
```

Hosts call `completion/complete` while the user types; the completion function
returns suggestions. The function gets the current partial value and the
already-bound arguments, so you can scope completions dynamically — for
example, suggest service names that exist in the *currently selected* env.

> **Tip**: completion functions can be async and hit a backend. Just keep
> them fast — the host typically calls them on every keystroke.

---

## 8. Resource Subscriptions — Live Updates

Some resources change over time: a metric stream, a build's status, the
current contents of a file being edited. Clients can subscribe to a resource
URI and the server pushes updates as they happen.

```typescript
// Notify subscribers when underlying data changes:
await server.server.notification({
  method: "notifications/resources/updated",
  params: { uri: "metrics://current/cpu" },
});
```

The flow is:

```
client                        server
  │                              │
  │── resources/subscribe ──────>│
  │<─ ack ───────────────────────│
  │                              │
  │              [data changes]  │
  │<─ notifications/resources/updated ─│
  │── resources/read ───────────>│   (client re-fetches)
  │<─ contents ──────────────────│
  │                              │
```

> **Warning**: subscriptions only work over **stateful sessions**. On a
> stateless HTTP server (every request a new session), `subscribe` is a silent
> no-op — the server has nowhere to remember the subscription. See
> [`../production/03_scaling.md`](../production/03_scaling.md) for the
> stateful-vs-stateless tradeoff.

The notification *doesn't* carry the new value. It's a hint that the client
should re-fetch via `resources/read`. This keeps the protocol simple and lets
the client decide whether it cares enough to spend the round trip.

---

## 9. Embedded Resources in Tool Results — Common Pattern

Tools and resources compose. A search tool that returns matching documents
shouldn't dump 50 KB of markdown into a `text` content block — it should
return *resource references* the host can attach properly:

```typescript
server.registerTool(
  "search_docs",
  { title: "Search Docs", description: "Search docs and return matches", inputSchema: { q: z.string() } },
  async ({ q }) => {
    const matches = await searchDocs(q);
    return {
      content: matches.map((m) => ({
        type: "resource" as const,
        resource: { uri: m.uri, mimeType: "text/markdown", text: m.text },
      })),
    };
  },
);
```

Now the host treats each match as a first-class resource: it can show them in
a sidebar, let the user pin one, or embed them with proper provenance. The
model still sees the text, but the *host* sees structured artifacts.

> **Key insight**: tools that return structured artifacts always beat tools
> that return blobs of text. The host can do more with the former — and the
> user can see *what* the model used, not just *what* the model said.

---

## 10. Failure Modes

Things that look right but blow up in production:

- ❌ **Returning huge resources inline (>1MB).** Context budgets explode and
  you'll burn through tokens before the model sees anything useful. Paginate
  or summarize. If a resource is genuinely huge, return a summary plus a
  templated URI for the full thing.

- ❌ **Templated resource without sanitizing path components.** A template
  like `file://{path}` invites SSRF and path traversal — `{path}` could be
  `../../etc/passwd` or `http://internal-service/admin`. Validate every
  component against an allowlist before using it.

- ❌ **Subscriptions on a stateless HTTP server.** `subscribe` returns success
  but no notifications ever fire, because there's no session to hold the
  subscription. Use stateful sessions or pick a different pattern (e.g.
  polling).

- ⚠️ **Prompts that pull in entire conversation history.** Expensive and
  brittle — every invocation re-loads the same context, and your template
  becomes a memory store rather than a starter. Keep templates focused on the
  arguments the user just provided.

- ❌ **Forgetting `as const` on the discriminator fields** (`type: "text"`,
  `role: "user"`, `type: "resource"`). TS widens to `string`, the SDK's Zod
  schema rejects it, and the error message points somewhere unhelpful. When
  in doubt, add `as const` to every literal that participates in a
  discriminated union.

---

## 11. Putting It Together

A reasonable mental model for which primitive to reach for:

```
                      ┌──────────────────────────┐
                      │   Who initiates this?    │
                      └────────────┬─────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
        ┌───────────┐        ┌───────────┐        ┌───────────┐
        │   Model   │        │   Host    │        │   User    │
        │ (decides) │        │ (attaches)│        │  (picks)  │
        └─────┬─────┘        └─────┬─────┘        └─────┬─────┘
              │                    │                    │
              ▼                    ▼                    ▼
            Tool              Resource              Prompt
```

Tools, resources, and prompts are the three sides of the MCP triangle. They
overlap on the wire — all three return structured content — but their *roles*
are distinct. Get the role right and the rest of the design tends to fall
into place.

For a deeper conceptual treatment of why MCP carves things this way, see
[`../fundamentals/03_primitives.md`](../fundamentals/03_primitives.md). For
the Python-flavored equivalent of this file, see
[`../python/03_resources_prompts.md`](../python/03_resources_prompts.md).

---

**Next**: [04_context_clients.md — Context and Clients](04_context_clients.md)
