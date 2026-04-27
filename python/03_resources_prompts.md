# Resources and Prompts in FastMCP 3.x

> **Who this is for**: Engineers who already have `@mcp.tool` working and want to expose
> the other two server-side primitives. Assumes you've read [02_tools.md](02_tools.md)
> and understand the [primitives model](../fundamentals/03_primitives.md).

Tools are the loud part of MCP — the model calls them, results show up, everyone is
happy. **Resources** and **prompts** are the quieter primitives, but they're how you
turn a server from "RPC endpoint" into something the host can integrate properly:
data the host attaches to context, and reusable templates the user can pick from a menu.

---

## 1. When to Use What

The three primitives overlap enough that the boundary blurs in real servers. This
matrix is the one I keep coming back to:

| Need                                                   | Use      |
|--------------------------------------------------------|----------|
| Action the model decides to invoke                     | Tool     |
| Data the host should attach to the LLM context        | Resource |
| Reusable prompt template the user picks               | Prompt   |

> **Key insight**: who initiates matters. **Tools** are model-initiated. **Resources**
> are host-initiated (the application decides what to attach). **Prompts** are
> user-initiated (the human picks them from a menu or slash command).

If you find yourself writing a tool called `get_readme` whose only job is to expose a
known document for host-side context attachment, make it a resource. But if the model
needs to decide dynamically whether to fetch that README during task execution, make it
a tool — or a general document-reading tool. If you have a tool that returns a giant
block of boilerplate instructions to the model — that should be a prompt.

---

## 2. Resources — Basics

A resource is identified by a **URI** and returns content when read. For a static URI,
the decorator creates a `resources/list` entry and a `resources/read` handler; for a
templated URI, it creates a `resources/templates/list` entry and the read handler.

```python
from fastmcp import FastMCP

mcp = FastMCP("docs-server")

@mcp.resource("docs://readme")
def readme() -> str:
    """Project README."""
    return open("/path/to/README.md").read()
```

The URI scheme is yours to design — `docs://`, `github://`, `db://table/users` — pick
something self-describing and stable. The host may cache by URI, so changing the
scheme later means breaking every client that pinned to the old form.

Return values can be:

- `str` — treated as text content
- `bytes` — treated as a binary blob (FastMCP base64-encodes it)
- JSON-native values (`dict`, `list`, Pydantic models, primitives) — serialized to JSON text
- `ResourceResult` — for multiple content items, per-item MIME types, and response metadata

> **Rule**: set `mime_type` explicitly for anything other than plain text. FastMCP can
> serialize JSON-native values for you, but explicit `json.dumps(data)` plus
> `mime_type="application/json"` is still the least surprising pattern when the exact
> wire MIME type matters.

MIME type is specified with `mime_type="..."` on the decorator. Some concrete resource
classes, such as `FileResource`, can infer from files, but plain `@mcp.resource`
handlers should be explicit — hosts use MIME types to decide rendering, syntax
highlighting, and whether to inline or link.

| Content      | Good MIME type                         |
|--------------|----------------------------------------|
| Markdown     | `text/markdown`                        |
| JSON         | `application/json`                     |
| Plain text   | `text/plain`                           |
| PNG/JPEG     | `image/png`, `image/jpeg`              |
| Generic blob | `application/octet-stream`             |

---

## 3. Templated Resources (RFC 6570)

Hard-coded URIs are fine for singletons (the README, today's status page) but not for
collections. **Templated resources** use [RFC 6570](https://www.rfc-editor.org/rfc/rfc6570)
URI templates so a single handler covers a whole family of URIs.

```python
import httpx
import json

@mcp.resource("github://repos/{owner}/{repo}/issues/{number}", mime_type="application/json")
async def github_issue(owner: str, repo: str, number: int) -> str:
    """A specific GitHub issue."""
    async with httpx.AsyncClient() as client:
        r = await client.get(
            f"https://api.github.com/repos/{owner}/{repo}/issues/{number}"
        )
        r.raise_for_status()
        return json.dumps(r.json())
```

Two list endpoints come into play:

| Endpoint                      | Returns                                                    |
|-------------------------------|------------------------------------------------------------|
| `resources/list`              | Concrete URIs (the static `@mcp.resource("docs://readme")`) |
| `resources/templates/list`    | URI templates (`github://repos/{owner}/{repo}/issues/{number}`) |

Clients can `resources/read` either form — for templates they substitute parameters
into the URI before the call. FastMCP routes the read back to the matching handler and
parses the path components into the function's typed arguments.

> **Rule**: don't enumerate templated URIs in `resources/list`. If the set is huge
> (every issue, every row in a table), templates are the right answer — the host asks
> for what it needs.

---

## 4. Binary Content

Returning `bytes` is the simple path. FastMCP turns the bytes into MCP
`BlobResourceContents` and base64-encodes them for the protocol response.

```python
@mcp.resource("photos://{photo_id}", mime_type="image/jpeg")
async def photo(photo_id: str) -> bytes:
    return await _fetch_photo_bytes(photo_id)
```

For multiple pieces or per-item MIME metadata, return `ResourceResult` with
`ResourceContent` items. Do **not** hand-assemble `BlobResourceContents` unless you are
working at the lower-level MCP SDK layer.

```python
from fastmcp.resources import ResourceContent, ResourceResult

@mcp.resource("reports://{report_id}")
async def report_bundle(report_id: str) -> ResourceResult:
    markdown, chart_png = await _build_report(report_id)
    return ResourceResult(
        contents=[
            ResourceContent(markdown, mime_type="text/markdown"),
            ResourceContent(chart_png, mime_type="image/png"),
        ],
        meta={"report_id": report_id},
    )
```

FastMCP handles base64 encoding for binary returns automatically. The MIME type on the
decorator or `ResourceContent` flows through to the response so the host knows whether
it is looking at a JPEG, Markdown, JSON, or random bytes.

⚠️ Don't return multi-megabyte blobs without thinking. The host has to ship the
content into the model's context, and base64 inflates payloads by ~33%. For anything
large, return a reference (URL, ID) and let the model decide whether to fetch it.

---

## 5. Listing Dynamically

The default `resources/list` enumerates concrete resources registered with
`@mcp.resource(...)`, `mcp.add_resource(...)`, or a provider. FastMCP 3 does **not**
document an `@mcp.list_resources` decorator; that pattern belongs to the low-level
`mcp.server.Server` API.

For dynamic catalogs — pages from a CMS, resources from an API, tenant-specific
objects — add a provider that returns real `Resource` objects:

```python
from collections.abc import Sequence

from fastmcp import FastMCP
from fastmcp.resources import Resource
from fastmcp.server.providers import Provider


class CmsResourceProvider(Provider):
    async def _list_resources(self) -> Sequence[Resource]:
        pages = await _fetch_all_pages()
        return [self._page_resource(p) for p in pages]

    def _page_resource(self, page) -> Resource:
        page_id = page.id

        async def read_page() -> str:
            return await _fetch_page_markdown(page_id)

        return Resource.from_function(
            read_page,
            uri=f"docs://page/{page_id}",
            name=page.title,
            mime_type="text/markdown",
        )


mcp = FastMCP("cms", providers=[CmsResourceProvider()])
```

This is most useful when:

- The resource set is computed at runtime (queried from a DB, fetched from an API)
- You want to surface a curated subset and not every possible templated URI
- Listing is expensive and you want to cache or paginate

For large parameterized spaces, prefer a resource template instead of enumerating every
possible resource. Providers are for when the catalog itself is a dynamic data source.

---

## 6. Resource List Notifications

FastMCP automatically sends `notifications/resources/list_changed` when resources or
templates are added, enabled, or disabled during an active MCP request context. That is
about the catalog changing, not the content of a specific URI changing.

```python
@mcp.resource("data://example")
def example_resource() -> str:
    return "Hello!"

# Inside a tool or other MCP request handler, these trigger list_changed notifications.
mcp.disable(keys={"resource:data://example"})
mcp.enable(keys={"resource:data://example"})
```

The MCP protocol also defines `resources/subscribe` and
`notifications/resources/updated` for content changes. Current FastMCP 3 docs do not
show a high-level `mcp.notify_resource_updated(...)` helper, so do not document that
as a FastMCP API unless your project provides its own wrapper or you are using the
low-level SDK directly.

The protocol-level content-update flow looks like:

```
client                              server
  │                                   │
  │── resources/subscribe(uri) ──────>│
  │<─────────────────────── ack ──────│
  │                                   │
  │           [time passes]           │
  │                                   │
  │                                   │  data changes externally
  │                                   │  → server sends updated notification
  │<── notifications/resources/updated│
  │                                   │
  │── resources/read(uri) ───────────>│
  │<─────────── new content ──────────│
```

Note that the server doesn't push the *content* — only a notification that the URI changed.
The client decides whether to re-read. This keeps the protocol cheap when an update
happens but no one is actively looking.

> **Key insight**: subscriptions need a **stateful session**. Stateless HTTP transport
> (each request is independent) has nowhere to send the push notification. If you
> need live updates, you need stdio, SSE, or session-pinned HTTP — see
> [scaling notes](../production/03_scaling.md).

---

## 7. Prompts — Basics

A **prompt** is a parameterized template that produces a list of messages ready to
seed a conversation. The host typically surfaces prompts as slash commands, quick
actions, or a "Templates" menu — the **user** picks one and fills in the arguments.

```python
from fastmcp import FastMCP
from fastmcp.prompts import Message

mcp = FastMCP("review")

@mcp.prompt
def review_pr(diff: str, focus_areas: list[str] | None = None) -> list[Message]:
    """Generate a thorough code review."""
    focus_areas = focus_areas or []
    intro = "You are a senior engineer doing a thorough code review."
    user = (
        f"Review this diff:\n\n{diff}\n\n"
        f"Focus on: {', '.join(focus_areas) or 'general quality'}"
    )
    return [
        Message(intro),
        Message(user),
    ]
```

The function body builds the messages; the decorator exposes the function over MCP
with its parameters as the prompt's argument schema. The host calls `prompts/get`
with the arguments, gets back the message list, and decides what to do with them —
usually inject into the active conversation.

> **Rule**: prompts return *messages*, not *tools to call*. They're a way to share
> well-crafted templates with the user, not a way to control the model's behavior
> behind the scenes.

---

## 8. Prompt Arguments

Argument schemas are derived directly from the function signature:

- Each parameter becomes a prompt argument
- Type hints document and validate the expected value
- Default values mark the argument as optional
- The docstring becomes the prompt's description

MCP prompt arguments are strings on the wire. FastMCP lets you write typed Python
parameters and handles useful conversions, including JSON strings for simple lists and
dicts. Keep prompt inputs simple; deeply nested custom models make a lousy menu form.

```python
from typing import Literal

@mcp.prompt
def deploy(env: Literal["dev", "staging", "prod"], service: str) -> str:
    """Prepare a deployment checklist."""
    return f"Prepare a deployment checklist for {service} in {env}."
```

No current FastMCP 3 decorator API for prompt argument completions is documented, so
avoid examples like `@deploy.completion("env")` unless you have confirmed the exact
version you are teaching supports it.

---

## 9. Embedded Resources in Tool Results

A pattern that ties everything together: a tool can return **embedded resources** in
its response. The host attaches them to the model's context the same way it would for
a regular resource read, but the trigger was the model calling a tool.

```python
from mcp.types import EmbeddedResource, TextResourceContents

@mcp.tool
async def fetch_docs(query: str) -> list[EmbeddedResource]:
    """Search docs and return matching pages as embedded resources."""
    matches = await _search(query)
    return [
        EmbeddedResource(
            type="resource",
            resource=TextResourceContents(
                uri=m.uri,
                mimeType="text/markdown",
                text=m.text,
            ),
        )
        for m in matches
    ]
```

Why this matters:

- The model sees both the URI **and** the content — it can cite the source by URI
- The host can de-duplicate against resources it already has in context
- The user can re-open the resource later from a citation

This is the right pattern for search-style tools, RAG retrievers, and anything where
"I looked something up, here is the page" is the answer. Don't return raw strings
that strip away the provenance.

---

## 10. Real-World Failure Modes

**Returning huge resources inline**

```python
# ❌ Reads a 50MB log file straight into the response
@mcp.resource("logs://today")
def today_log() -> str:
    return open("/var/log/app.log").read()

# ✅ Paginate or summarize
@mcp.resource("logs://today/tail/{lines}")
def today_log_tail(lines: int = 200) -> str:
    return _tail_file("/var/log/app.log", lines=min(lines, 5000))
```

Anything over ~1MB will blow up the context budget on the host side and slow every
subsequent model call. Paginate, summarize, or return a download URL instead.

**Templated resource without sanitizing inputs**

```python
# ❌ SSRF — the URI fragment becomes part of an outbound request
@mcp.resource("fetch://{url}")
async def fetch(url: str) -> str:
    return (await httpx.AsyncClient().get(url)).text

# ❌ Path traversal — ../../etc/passwd lands in your file read
@mcp.resource("file://{name}")
def read_file(name: str) -> str:
    return open(f"/data/{name}").read()
```

Templates feel like routing, but they're untrusted input. Validate against an
allowlist, normalize paths, and never pass raw URI components into network or
filesystem calls.

**Trying to push resource updates without a stateful delivery path**

The subscribe call may succeed at the protocol layer, but an update notification has
no session to push to. The client waits forever for an update that will never arrive.
Either run a stateful transport or document that subscriptions are not supported.

**Prompts that include the entire conversation history**

```python
# ⚠️ Brittle and expensive
@mcp.prompt
def summarize_chat(history: list[dict]) -> list[Message]:
    return [Message(f"Summarize: {history}")]
```

Prompts are templates, not full conversations. Keep them focused — the host already
has the live context. If you need historical data, expose it as a resource the host
can attach selectively.

---

## 11. Putting It Together

A well-designed server uses all three primitives, each for what it's best at:

```
┌──────────────┐         ┌────────────────────────────────┐
│ user         │ picks   │ Prompts                        │
│              │────────>│   review_pr(diff, focus)       │
│              │         │   write_release_notes(version) │
└──────┬───────┘         └────────────────────────────────┘
       │
       ▼
┌──────────────┐ attaches┌────────────────────────────────┐
│ host app     │────────>│ Resources                      │
│              │         │   docs://readme                │
│              │         │   github://repos/{o}/{r}/...   │
└──────┬───────┘         └────────────────────────────────┘
       │
       ▼
┌──────────────┐ invokes ┌────────────────────────────────┐
│ model        │────────>│ Tools                          │
│              │         │   create_pr(title, body)       │
│              │         │   merge_pr(number)             │
└──────────────┘         └────────────────────────────────┘
```

Same server, three audiences. The user gets templates; the host gets data; the model
gets actions. When you catch yourself stuffing one primitive into the role of another,
back up and ask which audience you're really serving.

---

**Next**: [04_context_lifespan.md — Context and Lifespan](04_context_lifespan.md)
