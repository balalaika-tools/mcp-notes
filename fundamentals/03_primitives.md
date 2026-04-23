# MCP Primitives — The Six Things the Protocol Standardizes

> **Who this is for**: Engineers who have read [01_what_is_mcp.md](01_what_is_mcp.md)
> and [02_architecture.md](02_architecture.md) and now want to know exactly what
> an MCP server can expose, what a client can do back, and how the two halves
> compose. This is the central conceptual file in this knowledge base — every
> later note (Python SDK chapters, transport notes, security notes) reuses the
> vocabulary defined here.

---

## 1. The Six Primitives at a Glance

MCP standardizes six distinct interaction patterns. Three flow from server to
client (the server *exposes* something the client can use). Three flow from
server to client as *requests back* — the server can ask the client to do
work on its behalf.

| Primitive       | Direction       | Who controls invocation         | What it's for                                                   |
|-----------------|-----------------|---------------------------------|-----------------------------------------------------------------|
| **Tools**       | server → client | **Model**-controlled            | Side-effecting actions the LLM may decide to call               |
| **Resources**   | server → client | **Application**-controlled      | Read-only content the host attaches to context                  |
| **Prompts**     | server → client | **User**-controlled             | Pre-built message templates the user picks from a menu          |
| **Sampling**    | client ← server | Server requests, client decides | Server asks the client to run an LLM completion                 |
| **Elicitation** | client ← server | Server requests, user decides   | Server asks the user for missing structured input mid-call      |
| **Roots**       | client ← server | Server requests, client decides | Server asks the client which filesystem/URI scopes are in scope |

> **Key insight**: The "who controls invocation" column is the most important
> design distinction in the entire spec. Tools, resources, and prompts all
> *look* similar (each is a named thing the server lists), but they exist as
> separate primitives precisely because the right *invoker* is different.
> Collapsing them into a single "endpoint" abstraction would force the host
> UI into bad defaults.

A useful mental picture:

```
        ┌──────────────────────── HOST (Claude Desktop, IDE) ─────────────────────────┐
        │                                                                             │
        │  ┌────────────────────────────── CLIENT ──────────────────────────────────┐ │
        │  │                                                                        │ │
        │  │  ──── server-exposed (server → client) ────                            │ │ 
        │  │                                                                        │ │
        │  │     tools/list        tools/call          ◀── model-controlled         │ │
        │  │     resources/list    resources/read      ◀── app-controlled           │ │
        │  │     prompts/list      prompts/get         ◀── user-controlled          │ │
        │  │                                                                        │ │
        │  │  ──── client-offered (server ← client) ────                            │ │
        │  │                                                                        │ │
        │  │     sampling/createMessage                ◀── server asks for an LLM   │ │
        │  │     elicitation/create                    ◀── server asks the user     │ │
        │  │     roots/list                            ◀── server asks for scope    │ │
        │  │                                                                        │ │
        │  └────────────────────────────────────────────────────────────────────────┘ │
        │                                  │                                          │
        └──────────────────────────────────┼──────────────────────────────────────────┘
                                           │  JSON-RPC 2.0 over stdio / HTTP+SSE
                                           ▼
                        ┌──────────────── MCP SERVER ───────────────┐
                        │  Owns: tool handlers, resource readers,   │
                        │  prompt templates, optional subscriptions │
                        └───────────────────────────────────────────┘
```

> **Rule**: A capability must be advertised in the `initialize` handshake
> before either side may use it. See [04_lifecycle.md](04_lifecycle.md) for the
> negotiation rules. Calling `tools/list` against a server that did not
> declare `tools` in its capabilities is a protocol error.

---

## 2. Tools — Model-Controlled Side Effects

Tools are the most-used primitive. They are the MCP equivalent of "function
calling" in a vendor LLM API, but cross-process and language-agnostic.

**Methods**

| Method        | Direction       | Purpose                                          |
|---------------|-----------------|--------------------------------------------------|
| `tools/list`  | client → server | Enumerate available tools                        |
| `tools/call`  | client → server | Invoke a tool with arguments                     |
| `notifications/tools/list_changed` | server → client | Server-pushed change signal |

**Anatomy of a tool**

A tool definition has a small set of well-known fields:

- `name` — unique within the server; the model uses this to dispatch.
- `description` — natural-language guidance for the model.
- `inputSchema` — **JSON Schema 2020-12** describing the arguments. Per
  **SEP-1613**, 2020-12 is the new default draft as of `2025-11-25`; older
  servers using draft-07 still work but new servers should target 2020-12.
- `outputSchema` *(optional)* — schema for `structuredContent` in the result.
- `annotations` *(optional)* — UI/safety hints: `readOnlyHint`,
  `destructiveHint`, `idempotentHint`, `openWorldHint`, `title`.
- `icons` *(optional, new in 2025-11-25, SEP-973)* — list of icon URLs
  (`data:` or `https:`) for richer host UI.
- `_meta` *(optional, new in 2025-11-25)* — free-form server-namespaced
  metadata. Ignored by clients that don't recognize the keys.

**Concrete tool definition**

The following is a real `tools/list` response from a hypothetical GitHub
server. Every field is part of the wire format — nothing here is pseudocode.

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "tools": [
      {
        "name": "github_create_issue",
        "description": "Open a new issue on a GitHub repository. Returns the issue number and HTML URL.",
        "inputSchema": {
          "$schema": "https://json-schema.org/draft/2020-12/schema",
          "type": "object",
          "properties": {
            "owner":  { "type": "string", "description": "Repo owner login" },
            "repo":   { "type": "string", "description": "Repo name" },
            "title":  { "type": "string", "minLength": 1, "maxLength": 256 },
            "body":   { "type": "string" },
            "labels": {
              "type": "array",
              "items": { "type": "string" },
              "uniqueItems": true
            }
          },
          "required": ["owner", "repo", "title"],
          "additionalProperties": false
        },
        "outputSchema": {
          "type": "object",
          "properties": {
            "number":   { "type": "integer" },
            "html_url": { "type": "string", "format": "uri" }
          },
          "required": ["number", "html_url"]
        },
        "annotations": {
          "title": "Create GitHub Issue",
          "readOnlyHint": false,
          "destructiveHint": false,
          "idempotentHint": false,
          "openWorldHint": true
        },
        "icons": [
          { "src": "https://github.githubassets.com/favicons/favicon.svg",
            "mimeType": "image/svg+xml", "sizes": "any" }
        ],
        "_meta": {
          "github.com/rate-limit-cost": 1
        }
      }
    ]
  }
}
```

**Calling a tool**

```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "method": "tools/call",
  "params": {
    "name": "github_create_issue",
    "arguments": {
      "owner": "anthropics",
      "repo":  "mcp-spec",
      "title": "Clarify SEP-1303 error semantics",
      "labels": ["docs", "spec"]
    }
  }
}
```

**Tool result content**

A tool result is a list of typed content blocks plus an optional
`structuredContent` object (validated against `outputSchema`) and an
`isError` boolean.

```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "result": {
    "content": [
      { "type": "text", "text": "Opened issue #4217" }
    ],
    "structuredContent": {
      "number": 4217,
      "html_url": "https://github.com/anthropics/mcp-spec/issues/4217"
    },
    "isError": false
  }
}
```

The `content` array supports four block types:

| Type               | Fields                                          | Use case                          |
|--------------------|-------------------------------------------------|-----------------------------------|
| `text`             | `text`                                          | Plain string output               |
| `image`            | `data` (base64), `mimeType`                     | Screenshots, charts               |
| `audio`            | `data` (base64), `mimeType`                     | TTS output, recordings            |
| `resource`         | embedded `resource` object (text or blob)       | Returning a referenceable artifact|

**Two kinds of failure — and the SEP-1303 change**

This trips up everyone the first time, so it's worth being explicit:

| Failure kind                  | Wire shape                                               | Who recovers   |
|-------------------------------|----------------------------------------------------------|----------------|
| **Protocol Error**            | JSON-RPC `error` object (`code`, `message`)              | The client     |
| **Tool Execution Error**      | A normal `result` with `isError: true` and text content  | The model      |

The distinction matters because only Tool Execution Errors are surfaced back
to the LLM, which can then self-correct (e.g., retry with a fixed argument).
Protocol Errors short-circuit the call and are typically logged by the host.

> **New in 2025-11-25 (SEP-1303)**: Input validation failures — arguments
> that don't match `inputSchema` — now return as **Tool Execution Errors**,
> not Protocol Errors. Previously a missing required field produced a
> JSON-RPC error code and the model never saw it. Now the model gets a
> readable `isError: true` message and can retry. Servers built before
> `2025-11-25` may still emit the old shape; clients should tolerate both.

```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "result": {
    "content": [
      { "type": "text",
        "text": "Argument validation failed: 'title' is required but was not provided." }
    ],
    "isError": true
  }
}
```

⚠️ Don't conflate `isError: true` with "the tool exploded." It just means
"this call did not produce the result you wanted, and here's why in words."
A 404 from the underlying API, a validation failure, a permission denial —
all are normal Tool Execution Errors. Use a JSON-RPC error only when the
call itself is malformed (unknown tool name, bad JSON, transport failure).

---

## 3. Resources — Application-Controlled Read-Only Content

Resources are addressable, read-only data the server exposes by URI. Files,
database rows, API responses, generated reports — anything the host might
want to *attach to the conversation* without invoking a side effect.

The "application-controlled" framing is what separates a resource from a
tool. The model does not autonomously fetch resources. The host decides
what's relevant to attach (often by letting the user pick from a picker,
or by auto-attaching a project's README), and the resource content goes
into the model's context.

**Methods**

| Method                          | Direction       | Purpose                                                  |
|---------------------------------|-----------------|----------------------------------------------------------|
| `resources/list`                | client → server | Enumerate concrete resources                             |
| `resources/read`                | client → server | Fetch a resource's contents                              |
| `resources/templates/list`      | client → server | Enumerate parameterized resource URIs                    |
| `resources/subscribe`           | client → server | Watch a resource for changes                             |
| `resources/unsubscribe`         | client → server | Stop watching                                            |
| `notifications/resources/updated`        | server → client | "This URI changed; re-read it"                  |
| `notifications/resources/list_changed`   | server → client | "The set of resources changed; re-list"         |

**URIs**

A resource is identified by URI. The scheme tells the client what kind of
thing it is. Common schemes:

| Scheme     | Example                                                  |
|------------|----------------------------------------------------------|
| `file://`  | `file:///Users/alex/projects/api/README.md`              |
| `https://` | `https://api.example.com/reports/q3.pdf`                 |
| `git://`   | `git://repo/HEAD/src/main.rs`                            |
| custom     | `postgres://prod/orders/12345`, `slack://channel/C0123`  |

Custom schemes are encouraged. The host doesn't need to know the scheme
semantics — it just passes the URI back to the server in `resources/read`.

**Listing concrete resources**

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "result": {
    "resources": [
      {
        "uri":  "file:///workspace/api/README.md",
        "name": "API README",
        "description": "Top-level project documentation",
        "mimeType": "text/markdown",
        "annotations": { "audience": ["user", "assistant"], "priority": 0.9 }
      },
      {
        "uri":  "postgres://prod/schema",
        "name": "Production database schema",
        "mimeType": "application/sql"
      }
    ]
  }
}
```

**Reading a resource**

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "method": "resources/read",
  "params": { "uri": "file:///workspace/api/README.md" }
}
```

The result returns one or more `contents` entries — text or binary:

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "result": {
    "contents": [
      {
        "uri":  "file:///workspace/api/README.md",
        "mimeType": "text/markdown",
        "text": "# Orders API\n\nThis service exposes ..."
      }
    ]
  }
}
```

For binary content, `text` is replaced with `blob` (base64-encoded bytes).

**Resource templates (RFC 6570)**

For unbounded URI spaces (every issue in a repo, every row in a table),
servers expose **templates** instead of enumerating every concrete URI.
Templates use **RFC 6570 URI Template** syntax.

```json
{
  "resourceTemplates": [
    {
      "uriTemplate": "github://repos/{owner}/{repo}/issues/{number}",
      "name": "GitHub issue",
      "mimeType": "application/json"
    },
    {
      "uriTemplate": "postgres://prod/orders/{order_id}",
      "name": "Order record",
      "mimeType": "application/json"
    }
  ]
}
```

The host expands the template (often via UI: "pick an issue") and then
issues a normal `resources/read` against the expanded URI.

**Subscriptions**

For data that changes — a log file, a Jira ticket the user is watching, a
build status — the client can subscribe:

```json
{ "jsonrpc": "2.0", "id": 13, "method": "resources/subscribe",
  "params": { "uri": "file:///workspace/build.log" } }
```

When the resource changes, the server pushes a notification (no `id`, no
response expected):

```json
{ "jsonrpc": "2.0", "method": "notifications/resources/updated",
  "params": { "uri": "file:///workspace/build.log" } }
```

The client decides whether to re-read immediately, debounce, or surface a
"refresh" affordance to the user. The notification carries no diff — it's
"something changed, ask again if you care."

> **Key insight**: Resources are *not* a remote filesystem. They're an
> opinionated read-only attachment surface. If you find yourself needing
> mutation, pagination, search-by-content, or anything resembling REST,
> you almost certainly want a **tool**, not a resource.

---

## 4. Prompts — User-Controlled Templates

Prompts are pre-baked message sequences the user can invoke from the host
UI — typically as slash commands ("/`summarize-pr`") or quick-action menus.
The server defines the template; the user fills in the arguments; the
client receives a list of messages ready to feed to the model.

**Methods**

| Method                              | Direction       | Purpose                  |
|-------------------------------------|-----------------|--------------------------|
| `prompts/list`                      | client → server | Enumerate prompts        |
| `prompts/get`                       | client → server | Render a prompt          |
| `notifications/prompts/list_changed`| server → client | List changed             |

**Listing prompts**

```json
{
  "jsonrpc": "2.0",
  "id": 21,
  "result": {
    "prompts": [
      {
        "name": "summarize_pr",
        "description": "Summarize a GitHub pull request for a release notes entry.",
        "arguments": [
          { "name": "pr_url", "description": "Full PR URL", "required": true },
          { "name": "audience", "description": "engineers | users | execs",
            "required": false }
        ]
      }
    ]
  }
}
```

**Getting a prompt**

```json
{
  "jsonrpc": "2.0",
  "id": 22,
  "method": "prompts/get",
  "params": {
    "name": "summarize_pr",
    "arguments": {
      "pr_url":   "https://github.com/anthropics/mcp/pull/812",
      "audience": "users"
    }
  }
}
```

The server returns ready-to-send messages (the client may concatenate them
with the rest of the conversation):

```json
{
  "jsonrpc": "2.0",
  "id": 22,
  "result": {
    "description": "Release-notes summary",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Summarize PR https://github.com/anthropics/mcp/pull/812 for a 'users' audience. Keep it under 80 words."
        }
      }
    ]
  }
}
```

> **Rule**: prompts are user-invoked. Don't use them as a covert way to
> inject system messages — that's what your server's *implementation*
> (which constructs the messages) is for. Prompts exist so users get a
> discoverable menu of useful starting points.

---

## 5. Sampling — Server Asks Client to Run an LLM

Sampling inverts the usual flow: the **server** asks the **client** to
run an LLM completion. This is what lets a server be model-agnostic — it
doesn't carry an API key, doesn't pick a model, doesn't pay the bill. It
just describes what it wants generated, and the client (which already has
the user's chosen model and credentials) executes.

**Method**: `sampling/createMessage` (server → client)

**Server-side request**

```json
{
  "jsonrpc": "2.0",
  "id": 31,
  "method": "sampling/createMessage",
  "params": {
    "messages": [
      { "role": "user",
        "content": { "type": "text",
                     "text": "Classify this support ticket: 'My invoice is wrong.'" } }
    ],
    "modelPreferences": {
      "hints":              [ { "name": "claude-sonnet" } ],
      "intelligencePriority": 0.7,
      "speedPriority":        0.5,
      "costPriority":         0.3
    },
    "systemPrompt":   "You are a triage assistant. Reply with one of: BILLING, TECHNICAL, ACCOUNT.",
    "includeContext": "thisServer",
    "maxTokens":      32
  }
}
```

Note that the server expresses *preferences* — a hint at a model family
plus relative priority weights — not a hard model name. The client maps
those preferences onto whatever models it has access to. A server that
runs against Claude Desktop, Cursor, Zed, and Continue will get four
different physical models, all satisfying the same request.

**Client-side response**

```json
{
  "jsonrpc": "2.0",
  "id": 31,
  "result": {
    "role":       "assistant",
    "content":    { "type": "text", "text": "BILLING" },
    "model":      "claude-sonnet-4-5",
    "stopReason": "end_turn"
  }
}
```

**Human-in-the-loop**

The spec encourages — and most clients implement — a confirmation step:
the user sees the prompt the server is about to run, the model it would
target, and approves or modifies it. This is by design. A server saying
"please run this LLM call with my system prompt against the user's
account" is privilege-equivalent to running arbitrary code on the user's
inference budget; the user should consent.

> **Key insight**: sampling is what makes "agentic" MCP servers possible
> without coupling them to any one model vendor. A code-review server
> can be written once and run against whatever LLM the host happens to
> have configured.

---

## 6. Elicitation — Server Asks the User for Missing Input

A tool call sometimes can't proceed without input the model didn't
provide — a confirmation, a choice from a list, a missing parameter.
Elicitation lets the server *pause mid-tool-call* and ask the user
directly via the host UI.

**Method**: `elicitation/create` (server → client)

Added in the `2025-06-18` revision; refined in `2025-11-25` (SEP-1330)
with proper enum support — single- and multi-select, with display titles
distinct from underlying values.

**Schema-driven request**

```json
{
  "jsonrpc": "2.0",
  "id": 41,
  "method": "elicitation/create",
  "params": {
    "message": "Which environment should I deploy to?",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "environment": {
          "type": "string",
          "enum":      ["staging", "production"],
          "enumNames": ["Staging (us-east-1)", "Production (global)"]
        },
        "confirm": {
          "type": "boolean",
          "description": "Yes, I want to deploy"
        }
      },
      "required": ["environment", "confirm"]
    }
  }
}
```

**Three possible responses**

The user has three outcomes; the response shape encodes which one happened:

| `action`    | Meaning                              | Includes `content`? |
|-------------|--------------------------------------|---------------------|
| `accept`    | User filled the form and submitted   | Yes                 |
| `decline`   | User explicitly refused              | No                  |
| `cancel`    | User dismissed (closed the dialog)   | No                  |

```json
{
  "jsonrpc": "2.0",
  "id": 41,
  "result": {
    "action": "accept",
    "content": {
      "environment": "staging",
      "confirm":     true
    }
  }
}
```

> **Rule**: servers must handle all three responses. Treating `cancel`
> as `decline` (or vice versa) is a UX bug — `decline` is a deliberate
> "no", `cancel` is "I changed my mind / closed the window". The right
> follow-up tool message differs.

⚠️ Don't use elicitation for things the model could plausibly know. If a
required argument is something the conversation contained, it's better
for the model to retry the tool call than to interrupt the user.

---

## 7. Roots — Server Asks Client for Filesystem/URI Scope

Roots tell the server "you may operate within these locations." A code
analysis server, for example, needs to know which directories on the
user's machine it's allowed to read. Rather than the user re-typing the
path, the host advertises roots and the server queries them.

**Methods**

| Method                          | Direction       | Purpose                            |
|---------------------------------|-----------------|------------------------------------|
| `roots/list`                    | server → client | Enumerate accessible roots         |
| `notifications/roots/list_changed`| client → server | "Roots changed; re-list"         |

**Request and response**

```json
{ "jsonrpc": "2.0", "id": 51, "method": "roots/list" }
```

```json
{
  "jsonrpc": "2.0",
  "id": 51,
  "result": {
    "roots": [
      { "uri": "file:///Users/alex/projects/api",      "name": "api" },
      { "uri": "file:///Users/alex/projects/frontend", "name": "frontend" }
    ]
  }
}
```

Roots can use any URI scheme, not just `file://`. A host that has
authenticated to GitHub might advertise `github://repos/myorg` as a
root, telling servers "anything under this path is fair game."

**Change notifications**

If the user opens a new project or revokes access mid-session, the
client pushes:

```json
{ "jsonrpc": "2.0", "method": "notifications/roots/list_changed" }
```

The server should re-issue `roots/list` and adjust its behavior — e.g.,
invalidate cached file listings, refuse new operations on dropped roots.

> **Key insight**: roots are *advisory*. The client still enforces the
> real boundary (sandboxing, file permissions, OS-level checks). A
> server that ignores roots and tries to read `~/.ssh/id_rsa` won't
> succeed — but it also won't be invited back. Respect the root list as
> if it were a hard wall.

---

## 8. New in `2025-11-25`

Three additions are worth knowing about. The first two are listed here
for vocabulary; the third is a meta-mechanism that affects how future
features get added.

### 8.1 Tasks (experimental, SEP-1686)

Tools that take a long time to run — a code refactor across a large
repo, a deep search, a long-running agent — don't fit the synchronous
"call now, wait for the result" pattern. **Tasks** add a "call now,
fetch later" handle: the server returns a task ID immediately, the
client polls or subscribes for state transitions, and the result is
retrieved when ready.

**State machine**

```
   ┌──────────┐   needs input    ┌──────────────────┐
   │ working  │ ────────────────>│ input_required   │
   └────┬─────┘                  └────────┬─────────┘
        │                                  │ user provides
        │                                  ▼
        │                            ┌──────────┐
        │ ───────────────────────────│ working  │
        │                            └────┬─────┘
        │ done                            │
        ▼                                 │ error
   ┌──────────┐                           ▼
   │completed │                    ┌──────────┐
   └──────────┘                    │  failed  │
                                   └──────────┘

                   user/client cancels at any time
                                  ▼
                            ┌──────────┐
                            │cancelled │
                            └──────────┘
```

States: `working`, `input_required`, `completed`, `failed`, `cancelled`.

Tasks are **experimental** in `2025-11-25` — the methods, message
shapes, and even the name may change. Don't bake tasks into a server
you intend to ship to production today; do plan for them in your
mental model of where MCP is going.

### 8.2 Icons metadata (SEP-973)

Tools, resources, and prompts can now carry an `icons` array of icon
descriptors — `data:` URIs for inline graphics, `https:` URIs for
hosted ones, with optional `mimeType` and `sizes`. Hosts use these to
render richer pickers and tool-call surfaces.

```json
"icons": [
  { "src": "data:image/svg+xml;base64,PHN2ZyB...", "mimeType": "image/svg+xml" },
  { "src": "https://example.com/icon.png", "mimeType": "image/png", "sizes": "64x64" }
]
```

⚠️ Hosts may strip or sandbox `data:` URIs; never assume a specific icon
will render. Always include a useful `name` and `description` — the
icon is a UI nicety, not a substitute for legible metadata.

### 8.3 Extensions framework

The spec now formally defines how optional, non-core features can be
declared, discovered, and configured. Previously, vendors who wanted to
add capabilities had to either pollute the core spec or ship
out-of-band. The extensions framework gives them a sanctioned path:
declare an extension namespace in `initialize`, expose its methods
under a clear prefix, and let clients negotiate support cleanly.

This matters because it lowers the cost of innovation. Instead of every
new idea forcing a spec-wide debate, a vendor can ship an extension,
prove it in production, and *then* propose it for core inclusion.
Expect more rapid evolution at the edges as a result.

---

## 9. Putting It Together — A Minimal End-to-End Trace

Here is a single conversation turn that exercises four primitives at
once. Understanding this trace is the payoff for everything above.

```
1. User opens host, picks slash command "/triage_ticket"
   ─────────────────────────────────────────────────────────────────
   Host → Server:   prompts/get { name: "triage_ticket", args: {...} }
   Server → Host:   { messages: [ ... ] }                              ← (3) PROMPT

2. Host adds the user's currently-selected ticket as context
   ─────────────────────────────────────────────────────────────────
   Host → Server:   resources/read { uri: "zendesk://tickets/8821" }
   Server → Host:   { contents: [ ... ] }                              ← (3) RESOURCE

3. Model decides it needs to look up the customer's plan tier
   ─────────────────────────────────────────────────────────────────
   Host → Server:   tools/call { name: "lookup_account",
                                 arguments: { ticket_id: 8821 } }      ← (2) TOOL
   Server can't find the customer ID in the ticket. It asks:
   Server → Host:   elicitation/create { message: "Customer email?",
                                         requestedSchema: {...} }      ← (6) ELICITATION
   User types email; host returns:
   Host → Server:   { action: "accept", content: { email: "..." } }

4. Server calls Zendesk, returns the result
   ─────────────────────────────────────────────────────────────────
   Server → Host:   { content: [ { type: "text",
                                   text: "Plan: Enterprise" } ],
                      isError: false }
```

Four primitives — prompt, resource, tool, elicitation — woven into one
user turn, each invoked by the *correct* party (user, application,
model, server). That separation of control is the whole point of the
six-primitive design.

---

## 10. Where to Go Next

The next file walks through the connection lifecycle: how `initialize`
works, how capabilities are negotiated (so the rules in §1 about
"declared first, used second" become concrete), and how shutdowns and
re-connects are handled.

For the Python implementation of these primitives, jump to:
- [../python/02_tools.md](../python/02_tools.md) — defining tools with `@server.tool`
- [../python/03_resources_prompts.md](../python/03_resources_prompts.md) — resources and prompt templates
- [../python/05_sampling_elicitation.md](../python/05_sampling_elicitation.md) — server → client requests

For the conceptual prerequisite, see [02_architecture.md](02_architecture.md).

---

**Next**: [04_lifecycle.md — Connection Lifecycle and Capability Negotiation](04_lifecycle.md)
