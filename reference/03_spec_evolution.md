# MCP Spec Evolution: Versions, Deprecations, and the 2026 Roadmap

> **Who this is for**: Engineers maintaining or evaluating an MCP server, debugging
> version-mismatch bugs across heterogeneous fleets, or planning a migration from an
> older spec revision. Assumes you've read [02_python_vs_typescript.md](02_python_vs_typescript.md)
> and have a working mental model of the protocol from
> [../fundamentals/01_what_is_mcp.md](../fundamentals/01_what_is_mcp.md).

---

## 1. Versioning Model

MCP spec versions are **date strings** in `YYYY-MM-DD` form — not semver. The version
identifies the day a particular revision of the specification was published. Any single
deployed server or client implements exactly one version of the wire protocol at a time
(though some support multiple and pick one at handshake).

Negotiation happens during the `initialize` request:

```
client                                          server
  │                                                │
  │── initialize { protocolVersion: "2025-11-25" }─>│
  │                                                │   server checks: do I support that?
  │                                                │   if yes  → echo it back
  │                                                │   if no   → return its own preferred
  │                                                │             version (must be ≤ client's)
  │<── InitializeResult { protocolVersion: ... } ──│
  │                                                │
  │  if returned version is one I support  → continue
  │  else                                  → close the connection
```

> **Rule**: the agreed protocol version is **whatever the server returns** in
> `InitializeResult.protocolVersion`. The client decides whether it can speak that
> version; if not, it disconnects. The server is the authority because it knows what
> features it can actually serve.

Backwards compatibility is best-effort. The spec does not promise that a 2025-11-25
client can talk to a 2024-11-05 server — it only promises that both sides expose the
version they speak so the mismatch is visible at handshake instead of as a mysterious
500 four requests later.

> **Key insight**: every other version-mismatch debugging story you'll hit ultimately
> traces back to one party not echoing the version it actually implements. Log the
> negotiated version on every connection.

A few practical implications that fall out of the date-string model:

- **No semver implication**. `2025-06-18` is not "version 6.18 of MCP." It is
  literally "the spec as of June 18th, 2025." Two consecutive releases may
  contain breaking changes or may be additive only — you cannot tell from the
  version string alone. Read the changelog.
- **No prerelease channel in the version itself**. There is no `2025-11-25-rc1`.
  Drafts of a future version live in the SEP repository as merged but
  unpublished proposals. Once the spec maintainers cut a release, the date
  string is final.
- **Pin your version explicitly**. Both clients and servers should hardcode the
  versions they implement, not auto-detect at runtime. Auto-detection is what
  the `initialize` handshake is for; the application code itself should know
  what wire format it's emitting.
- **Multi-version support is rare but allowed**. A server can advertise support
  for multiple versions and pick the highest version the client also supports.
  Most servers just implement one. Hosts that bridge to many servers tend to
  support a wider range on the client side.

---

## 2. Timeline Overview

| Version | Released | Status (Apr 2026) | Headline change |
|---------|----------|-------------------|-----------------|
| `2024-11-05` | Nov 2024 | Legacy; some old servers still on this | Initial public release |
| `2025-03-26` | Mar 2025 | Deprecated; Streamable HTTP introduced (replaced HTTP+SSE) | First major iteration |
| `2025-06-18` | Jun 2025 | Older but supported | OAuth 2.1 + RFC 9728 mandated; Elicitation introduced |
| `2025-11-25` | Nov 2025 | **Current** | One-year anniversary release; Tasks, Icons, Extensions framework |

Four releases in twelve months. The cadence has been roughly quarterly with an extra
gap between June and November to absorb the Linux Foundation governance move. Expect
the 2026 cadence to slow modestly as the spec stabilizes.

---

## 3. `2024-11-05` — The Original Release

The first public MCP spec. Everything that followed is an evolution of these primitives.

**What shipped:**

- **Tools, resources, prompts** — the three server-side primitives. Tool calling is
  RPC-style; resources are URI-addressable read-only blobs; prompts are templated
  message scaffolds the host can offer to the user.
- **Sampling** (server → client) — the server can ask the client to run an LLM
  completion on its behalf. This is the inversion that makes MCP interesting:
  servers can be agentic without owning a model.
- **Roots** (server → client) — the server can ask the client what filesystem (or
  conceptual) roots it should operate within.
- **HTTP+SSE transport** — two endpoints: a POST endpoint for client→server messages
  and a separate GET-based SSE stream for server→client messages. Pairing them
  required a session ID that flowed in headers. The split caused real operational
  pain: load balancers would route the POST and GET to different backends, sticky
  sessions had to be configured by hand, and every reconnect required correlating
  the new SSE stream back to the original session.
- **stdio transport** — JSON-RPC framed over stdin/stdout, with stderr reserved for
  human-readable logs. Still the dominant local-process transport.
- **No standardized auth** — every server rolled its own. Some used bearer tokens in
  headers, some used query strings, some assumed network-level isolation. This was
  the largest single source of pain in the first six months.
- **JSON Schema draft 7** — never explicitly mandated, but the de-facto dialect
  used in tool input schemas and validators across all early SDKs.

⚠️ If you find a server still on `2024-11-05` in 2026, treat it as an artifact. It
will not have any of the auth, transport, or correctness improvements that followed.

---

## 4. `2025-03-26` — Streamable HTTP Arrives

Four months after the initial release, the first major correction. The two-endpoint
HTTP+SSE transport had proved painful: cross-endpoint session correlation was
fragile, load balancers mangled it, and serverless platforms could not host it.

**What changed:**

- **Streamable HTTP transport** introduced. One endpoint, multiple verbs:
  - `POST` — client sends a request; server responds with either JSON or an SSE
    upgrade for streaming responses
  - `GET` — client opens a long-lived SSE stream for unsolicited server-push
    messages (notifications, sampling requests)
  - `DELETE` — client signals session termination so the server can free state
- **HTTP+SSE** marked for deprecation but still allowed in this version for
  transitional compatibility.
- **Tool annotations** added: `readOnlyHint`, `destructiveHint`, `idempotentHint`,
  `openWorldHint`. These are advisory hints to the host — they do not change
  protocol behavior, but a well-behaved host can use them to gate confirmations
  ("This tool says it's destructive — confirm before running?").
- **First steps toward standardized auth** — advisory text on bearer tokens and
  OAuth, not yet mandatory. Implementations were encouraged but not required to
  adopt it.
- **Resource subscriptions** formalized — the existing pattern of subscribing to
  resource updates was given proper notification message shapes.

> **Rule**: if you're writing new code in 2026 and you see HTTP+SSE in any
> dependency or example, stop. The deprecation has now stuck for a year; nothing
> new should target it.

---

## 5. `2025-06-18` — Auth Becomes Mandatory

The release that turned MCP from "neat protocol" into "thing enterprise security can
sign off on." Three months after Streamable HTTP, the auth story was finally pinned
down.

**What changed:**

- **OAuth 2.1 + PKCE mandatory** for HTTP transports. Implementations MUST support
  the OAuth 2.1 authorization code flow with PKCE. No more home-rolled bearer
  schemes for compliant servers.
- **RFC 9728 Protected Resource Metadata** required. Every server must expose
  `/.well-known/oauth-protected-resource` so clients can discover the authorization
  server, scopes, and resource identifiers without out-of-band configuration.
- **401 + `WWW-Authenticate` flow standardized**. Unauthenticated requests get a
  `401 Unauthorized` with a `WWW-Authenticate` header pointing at the resource
  metadata URL. This is the entrypoint for the entire discovery dance.
- **Elicitation primitive** introduced (`elicitation/create`). The server can now
  ask the user — through the client — for structured input mid-tool-call. This
  enabled flows like "the tool needs a confirmation" or "tell me which of these
  three records you meant" without inventing a side channel.
- **`Mcp-Protocol-Version` header** required on all HTTP requests after handshake.
  Lets gateways and intermediaries route by version without parsing JSON-RPC bodies.
- **Improved progress notifications** — token-based correlation. A long-running
  call returns a progress token; the server emits notifications carrying that
  token; the client correlates them to the originating request.
- **Audio content type** added to tool results. Tools could now return audio blobs
  alongside text and image content.

See [../production/02_authentication.md](../production/02_authentication.md) for the
full auth story — this version is where most of it was nailed down.

---

## 6. `2025-11-25` — The Anniversary Release (Current)

One year after the original spec. The release where MCP grew up: long-running tools
got first-class handles, the schema dialect was modernized, governance moved to a
foundation, and the SDK ecosystem was formally tiered.

**What changed:**

- **JSON Schema 2020-12** is now the default dialect (SEP-1613). Brings recursive
  `$dynamicRef`, `prefixItems` for tuple-style validation, `unevaluatedProperties`,
  better conditional schemas, and a saner draft path forward. Old draft-07 schemas
  generally still validate, but new servers should target 2020-12.
- **Tasks (experimental, SEP-1686)** — "call now, fetch later" handles for
  long-running tools. A tool can return a task handle immediately and then move
  through states:
  ```
  working ──> input_required ──> working ──> completed
                  │
                  └──> failed / cancelled
  ```
  The client polls or subscribes to the task. This is the answer to "what about
  tools that take ten minutes?" — previously, you either blocked the connection
  or invented an out-of-band polling protocol.
- **Icons metadata (SEP-973)** — tools, resources, and prompts can carry an
  `icons: [...]` array with `data:` URIs or `https:` URLs. Hosts use these to
  render richer UIs (a database tool with the Postgres elephant, a Slack tool
  with the Slack mark, etc.).
- **Extensions framework** — formal way to declare, discover, and configure
  optional features beyond the core spec. Replaces the previous ad-hoc
  "experimental capability" pattern. Lets vendors ship features without forking
  the spec.
- **Tool execution errors vs. protocol errors (SEP-1303)** — input validation
  errors now return as **Tool Execution Errors** (`isError: true` with text
  content describing what was wrong), not as protocol-level JSON-RPC errors.
  The model sees the validation failure inline and can self-correct on the next
  turn, instead of the whole tool call aborting.
- **Improved enum elicitation (SEP-1330)** — single-select and multi-select enums
  are formalized via `oneOf` with `const` + `title` pairs. Hosts can render proper
  pickers; titles can differ from the underlying constant value.
- **Client ID Metadata Documents (SEP-991)** — recommended dynamic client
  registration mechanism. The client publishes a metadata URL; the authorization
  server fetches and trusts that document. Solves the "every new client needs to
  pre-register with every AS" scaling problem.
- **OpenID Connect Discovery 1.0** added as a recommended discovery mechanism
  alongside RFC 9728 metadata.
- **stdio stderr broadened** — explicitly allowed for any structured logging,
  not just errors. Previously the spec was ambiguous; now it's clear stderr is
  the universal sideband.
- **Governance formalized (SEP-932)** — MCP moved under the Linux Foundation /
  Agentic AI Foundation in December 2025. SEPs are the formal change-proposal
  mechanism.
- **SDK tiering (SEP-1730)** — official tier system:
  - **Tier 1** — Python, TypeScript (Anthropic-maintained, ship same day as spec)
  - **Tier 2** — Go, C#, Java, Kotlin, Rust, Swift (foundation-maintained)
  - **Tier 3** — community SDKs (everything else)

A worked example of the SEP-1303 error-shape change, since it's the one behavioral
break most existing tools will need to address:

```python
# ❌ pre-2025-11-25 pattern — validation error becomes a JSON-RPC error,
#    aborts the call, model never sees the message inline
@server.tool()
async def lookup_user(user_id: str) -> dict:
    if not user_id.startswith("usr_"):
        raise ValueError("user_id must start with 'usr_'")  # propagates as protocol error
    return await db.fetch_user(user_id)

# ✅ 2025-11-25 pattern — return isError content so the model can self-correct
@server.tool()
async def lookup_user(user_id: str) -> ToolResult:
    if not user_id.startswith("usr_"):
        return ToolResult(
            isError=True,
            content=[TextContent(
                type="text",
                text="user_id must start with 'usr_' (e.g., 'usr_abc123')"
            )],
        )
    user = await db.fetch_user(user_id)
    return ToolResult(content=[TextContent(type="text", text=user.json())])
```

Reserve real protocol errors for cases where the call genuinely cannot be made
(server is shutting down, the tool was deleted, etc.) — anything that's just
"the input you gave me was wrong" should now be a Tool Execution Error.

---

## 7. Migration Guide — Common Version Jumps

### From `2024-11-05` → `2025-11-25` (rare but possible)

If you're sitting on a server that hasn't been touched in over a year:

- **Replace HTTP+SSE with Streamable HTTP** — single endpoint, see
  [../production/01_streamable_http.md](../production/01_streamable_http.md) for
  the wire details.
- **Add OAuth 2.1 + RFC 9728 metadata** — this is the largest change. Stand up
  an authorization server (or wire to your existing IdP), publish
  `/.well-known/oauth-protected-resource`, implement the 401 + `WWW-Authenticate`
  flow.
- **Migrate JSON Schema** if you used draft-07-only features. Most schemas
  validate fine under 2020-12 with no edits, but anything that relied on
  removed/renamed keywords needs adjustment.
- **Update tool error handling** — return `isError: true` with text content for
  validation errors instead of throwing protocol errors. Audit every place where
  you raised an exception out of a tool handler.
- **Add tool annotations** where appropriate (`readOnlyHint`, `destructiveHint`).
  Cheap to add, materially better host UX.

### From `2025-03-26` → `2025-11-25`

The middle ground. You have Streamable HTTP and tool annotations already.

- **Make auth mandatory** — you may have skipped it when it was advisory. Now
  it's required for HTTP transports. Same RFC 9728 + OAuth 2.1 work as above.
- **Add `Mcp-Protocol-Version` header** on all HTTP requests after handshake.
  One-line change in your client; servers should validate it.
- **Adopt the new JSON Schema 2020-12 dialect** for new tool schemas.
- **Switch validation errors to `isError: true`** — same SEP-1303 audit as above.

### From `2025-06-18` → `2025-11-25`

The easy jump — you already have auth, the modern transport, elicitation, and
the protocol-version header.

- Mostly compatible. Adopt new features (icons, tasks, extensions) on your own
  schedule.
- **Switch validation errors to the new `isError: true` pattern** (SEP-1303) —
  this is the one behavioral change that affects existing tools.
- **Adopt JSON Schema 2020-12 features** in new tool schemas where they help.

> **Tip** 💡: regardless of starting point, run both old and new versions side-by-side
> behind a version-aware router during migration. The protocol-version header makes
> this trivial — route to the new pod for `2025-11-25`, the old pod for everything
> else, until your client fleet is fully upgraded.

---

## 8. What's Deprecated and When It Goes Away

| Feature | Deprecated in | Removed in |
|---------|---------------|------------|
| HTTP+SSE transport (two-endpoint) | 2025-03-26 | Servers should drop support; some clients still call it through 2026 |
| OAuth 2.0 implicit flow | 2025-06-18 | Already removed — never supported in OAuth 2.1 |
| OAuth password grant | 2025-06-18 | Already removed |
| PKCE `plain` method | 2025-06-18 | `S256` only |
| Old DCR (RFC 7591) for new ecosystems | 2025-11-25 (recommended) | Not removed; SEP-991 (Client ID Metadata) preferred going forward |

A few notes on the deprecation table:

- **HTTP+SSE** is the only transport-level removal. If you're shipping a server
  in 2026 you can drop the SSE-pair endpoints entirely. If you're shipping a
  client, you might still want a fallback for one more cycle to handle stragglers
  in the wild.
- **OAuth 2.0 implicit, password grant, and PKCE `plain`** are not really MCP
  removals — they were never permitted under OAuth 2.1, which MCP adopted
  wholesale in `2025-06-18`. If you had a custom auth flow inherited from an
  OAuth 2.0 implementation, it doesn't carry forward.
- **RFC 7591 DCR** is not gone. It's still a perfectly valid way to register a
  client with an AS. SEP-991's Client ID Metadata Documents are the
  forward-recommended approach for new ecosystems where you don't want every
  client pre-registered, but legacy DCR will keep working.

---

## 9. 2026 Roadmap Themes

Drawn from the 2026 MCP Roadmap blog post and the December 2025 "Future of
Transports" piece. None of these are in the spec yet; treat them as direction,
not commitment.

- **Transport scalability** — better stateless support, possibly **WebSockets**
  as an additional transport. The motivation is bidirectional streaming that
  doesn't require the SSE-upgrade dance and that survives load balancers and
  proxies more cleanly than long-poll patterns.
- **Agent-to-agent communication** — patterns for one MCP server to talk to
  another, or for multi-agent orchestration where servers compose into pipelines.
  Today this is possible but ad hoc; expect formal patterns and possibly new
  primitives.
- **Governance maturation** — broader SEP process participation now that the
  spec lives under the Linux Foundation. Expect more formal compatibility
  commitments and clearer deprecation timelines.
- **Enterprise readiness** — gateways, registries, RBAC, and audit features more
  deeply integrated into the spec rather than being left to vendors. The current
  state — every enterprise vendor ships its own gateway with overlapping
  features — is not sustainable.
- **Tasks moving from experimental to stable** — the 2025-11-25 task model is
  marked experimental. Expect ratification (and likely a few breaking shape
  tweaks) in a 2026 release.
- **Streaming output semantics** for tool returns — today, streaming tool output
  is text/SSE-shaped. The roadmap signals typed streaming responses (think:
  partial structured objects, deltas, typed event streams) as a first-class
  return shape.

A reasonable bet for what 2026 looks like in practice: one or two minor
releases that promote experimental features (Tasks chief among them) to
stable, plus a larger release that introduces a new transport option
(WebSockets is the favorite) and the agent-to-agent patterns. Don't expect
2024-style upheaval — the foundation move and the SDK tiering both signal a
shift toward stability.

---

## 10. How to Read the Spec Yourself

The spec is the source of truth. SDK docs, blog posts, and notes like this one
are aids — when in doubt, the spec wins.

- **Spec home**: <https://modelcontextprotocol.io/specification/2025-11-25>
- **Changelog**: <https://modelcontextprotocol.io/specification/2025-11-25/changelog>
- **SEP repo** (change proposals): <https://github.com/modelcontextprotocol/specification/tree/main/sep>
- **Blog**: <https://blog.modelcontextprotocol.io>

> **Rule**: the spec is normative; SDK documentation is not. If a Python SDK
> behaves differently from a TypeScript SDK on the same input, at most one of
> them matches the spec. Reading the relevant spec section directly is faster
> than triangulating from two SDK READMEs.

When reading a SEP, skim the "Motivation" and "Specification" sections first —
the "Rationale" and "Alternatives" sections are valuable but long. SEP numbers
referenced in this file (SEP-973, SEP-991, SEP-1303, SEP-1330, SEP-1613,
SEP-1686, SEP-1730, SEP-932) will all resolve in the SEP repo if you want the
full history.

---

## 11. Debugging Version Mismatch

The most common production headaches with MCP are version mismatches dressed up
as something else. Symptom-driven cheat sheet:

**Symptom**: client errors with "method not found"
- **Cause**: server is on an older spec version that doesn't have that method
  (e.g., calling `elicitation/create` against a `2025-03-26` server).
- **Fix**: log the negotiated `protocolVersion`. If it's older than the version
  that introduced the method, you have your answer.

**Symptom**: 401 with no `WWW-Authenticate` header
- **Cause**: server is pre-`2025-06-18`. It can't do RFC 9728 discovery because
  the RFC wasn't required yet.
- **Fix**: either upgrade the server or fall back to out-of-band auth
  configuration for that server. Don't try to make the client guess — the whole
  point of standardized discovery is that you stop guessing.

**Symptom**: HTTP+SSE two endpoints expected on a `2025-11-25` client; client
tried Streamable HTTP and 404'd
- **Cause**: client thinks it's talking to a Streamable HTTP server; server is
  actually still on the two-endpoint HTTP+SSE design.
- **Fix**: either upgrade the server, or use a client that probes both transports.
  Long-term, retire the server.

**Symptom**: tool calls fail with cryptic JSON-RPC errors when the input is
slightly malformed
- **Cause**: server is pre-`2025-11-25` and treats validation failures as
  protocol errors, not as Tool Execution Errors. The model never sees the
  validation message and can't self-correct.
- **Fix**: upgrade the server, or wrap the tool to catch validation errors
  internally and return them as `isError: true` content.

**General fix #1**: log the negotiated `protocolVersion` from `InitializeResult`
on every connection. Surface it in your dashboards and connection-quality
metrics. Most version-mismatch debugging is "I assumed the server was on X but
it's actually on Y."

**General fix #2**: when interoperating across versions, downgrade to the lower
of the two parties' versions. The spec mandates this on the server side — the
server must return a version it supports that is no higher than the client's
requested version. If your server isn't doing this, fix it; it's a spec
violation, not a feature.

> **Key insight**: every MCP version-mismatch bug is debuggable in under five
> minutes if you logged the negotiated protocol version on connect. Every
> version-mismatch bug is a multi-hour adventure if you didn't.

A minimal observability sketch — log this on every successful handshake and
forget about it until a bug report makes it useful:

```python
# After initialize completes
logger.info(
    "mcp_session_established",
    extra={
        "server_name": init_result.server_info.name,
        "server_version": init_result.server_info.version,
        "protocol_version": init_result.protocol_version,  # the negotiated date string
        "client_protocol_version": client.protocol_version,  # what we asked for
        "transport": transport_type,                          # stdio | streamable_http
        "capabilities": list(init_result.capabilities.keys()),
    },
)
```

When a user reports "the foo tool stopped working," the first query you run is
"what protocol version did they negotiate?" If it's older than the version that
introduced the feature, you don't even need to read the stack trace.

For version-aware routing on the server side (when you genuinely support
multiple versions), key your dispatch table on the negotiated version rather
than branching inside every handler:

```python
DISPATCH = {
    "2025-06-18": handlers_v2025_06_18,
    "2025-11-25": handlers_v2025_11_25,
}

def get_handlers(session: McpSession):
    return DISPATCH.get(session.protocol_version, handlers_v2025_11_25)
```

This keeps the version-specific behavior localized and auditable instead of
sprinkling `if version >= ...` checks throughout the codebase.

See [../fundamentals/04_lifecycle.md](../fundamentals/04_lifecycle.md) for the
full handshake state machine, including where in the lifecycle each version
field becomes available.

---

**You've reached the end of the notes.** From here, the [official spec](https://modelcontextprotocol.io/specification/2025-11-25) and the [SEP repo](https://github.com/modelcontextprotocol/specification/tree/main/sep) are the source of truth. Re-read [Fundamentals: What Is MCP](../fundamentals/01_what_is_mcp.md) if you want a tour from the top.
