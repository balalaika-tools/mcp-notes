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

If you find yourself writing a tool called `get_readme` whose only job is to return a
file — that should be a resource. If you have a tool that returns a giant block of
boilerplate instructions to the model — that should be a prompt.

---

## 2. Resources — Basics

A resource is identified by a **URI** and returns content when read. The decorator
turns a function into both a `resources/list` entry and a `resources/read` handler.

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

Return value can be:

- `str` — treated as text content
- `bytes` — treated as a binary blob (FastMCP base64-encodes it)
- `list[ContentPart]` — for multi-part responses (text + image, etc.)

MIME type is inferred from the URI suffix (`.md` → `text/markdown`) or specified
explicitly with `mime_type="..."` on the decorator. When in doubt, set it explicitly —
hosts use it to decide rendering, syntax highlighting, and whether to inline or link.

| URI suffix | Inferred MIME           |
|------------|-------------------------|
| `.md`      | `text/markdown`         |
| `.json`    | `application/json`      |
| `.txt`     | `text/plain`            |
| `.png`     | `image/png`             |
| (none)     | `application/octet-stream` for bytes, `text/plain` for str |

---

## 3. Templated Resources (RFC 6570)

Hard-coded URIs are fine for singletons (the README, today's status page) but not for
collections. **Templated resources** use [RFC 6570](https://www.rfc-editor.org/rfc/rfc6570)
URI templates so a single handler covers a whole family of URIs.

```python
import httpx

@mcp.resource("github://repos/{owner}/{repo}/issues/{number}")
async def github_issue(owner: str, repo: str, number: int) -> dict:
    """A specific GitHub issue."""
    async with httpx.AsyncClient() as client:
        r = await client.get(
            f"https://api.github.com/repos/{owner}/{repo}/issues/{number}"
        )
        r.raise_for_status()
        return r.json()
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

Returning `bytes` is the simple path; for richer responses use `BlobResourceContents`
or return a list of content parts.

```python
from fastmcp.types import BlobResourceContents

@mcp.resource("photos://{photo_id}", mime_type="image/jpeg")
async def photo(photo_id: str) -> bytes:
    return await _fetch_photo_bytes(photo_id)
```

FastMCP handles base64 encoding for binary returns automatically — you do not assemble
the wire format yourself. The MIME type on the decorator flows through to the response
so the host knows it's looking at a JPEG and not random bytes.

⚠️ Don't return multi-megabyte blobs without thinking. The host has to ship the
content into the model's context, and base64 inflates payloads by ~33%. For anything
large, return a reference (URL, ID) and let the model decide whether to fetch it.

---

## 5. Listing Dynamically

The default `resources/list` enumerates every `@mcp.resource(...)`-decorated function
with a static URI. For dynamic catalogs — pages from a CMS, tables from a discovered
schema — override the listing:

```python
@mcp.list_resources
async def my_resources() -> list[dict]:
    """Override the default static list."""
    return [
        {
            "uri": f"doc://page/{p.id}",
            "name": p.title,
            "mimeType": "text/markdown",
        }
        for p in await _fetch_all_pages()
    ]
```

This is most useful when:

- The resource set is computed at runtime (queried from a DB, fetched from an API)
- You want to surface a curated subset and not every possible templated URI
- Listing is expensive and you want to cache or paginate

The reads themselves still go through your normal `@mcp.resource` handlers — the
override only changes what shows up in the catalog.

---

## 6. Subscriptions — Live-Updating Resources

For resources that change while the host is connected (live metrics, document edits,
build status), MCP supports **subscriptions**. The client calls `resources/subscribe`
on a URI; the server pushes `notifications/resources/updated` whenever the underlying
data changes; the client re-reads.

```python
from fastmcp import Context

@mcp.resource("metrics://current/{metric}")
async def current_metric(metric: str, ctx: Context) -> dict:
    return await _read_metric(metric)

# When the underlying data changes, notify subscribers:
async def on_metric_change(metric: str):
    await mcp.notify_resource_updated(f"metrics://current/{metric}")
```

You wire `on_metric_change` into wherever your application detects the change — a
pub/sub callback, a file watcher, a polling loop. FastMCP tracks which sessions
subscribed to which URIs and only pushes to relevant clients.

The flow looks like:

```
client                              server
  │                                   │
  │── resources/subscribe(uri) ──────>│
  │<─────────────────────── ack ──────│
  │                                   │
  │           [time passes]           │
  │                                   │
  │                                   │  data changes externally
  │                                   │  → notify_resource_updated(uri)
  │<── notifications/resources/updated│
  │                                   │
  │── resources/read(uri) ───────────>│
  │<─────────── new content ──────────│
```

Note the server doesn't push the *content* — only a notification that the URI changed.
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
from fastmcp.types import Message

mcp = FastMCP("review")

@mcp.prompt
def review_pr(diff: str, focus_areas: list[str] = []) -> list[Message]:
    """Generate a thorough code review."""
    sys = "You are a senior engineer doing a thorough code review."
    user = (
        f"Review this diff:\n\n{diff}\n\n"
        f"Focus on: {', '.join(focus_areas) or 'general quality'}"
    )
    return [
        Message(role="user", content=sys),
        Message(role="user", content=user),
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

## 8. Prompt Arguments and Completion

Argument schemas are derived directly from the function signature:

- Each parameter becomes a prompt argument
- Type hints map to JSON Schema types
- Default values mark the argument as optional
- The docstring becomes the prompt's description

For arguments with a fixed set of valid values (environments, services, languages),
register a **completion provider** so the host can offer suggestions while the user
types:

```python
@mcp.prompt
def deploy(env: str, service: str) -> list[Message]: ...

@deploy.completion("env")
def env_completions() -> list[str]:
    return ["dev", "staging", "prod"]
```

Clients call `completion/complete` with a partial value and the argument name; the
server returns matching suggestions. This is what powers the dropdown when a user
starts typing into a slash command.

💡 Completion handlers can be async and can take the partial input as a parameter if
you want to filter against a remote source (autocompleting a service name from a
service registry, for example).

---

## 9. Embedded Resources in Tool Results

A pattern that ties everything together: a tool can return **embedded resources** in
its response. The host attaches them to the model's context the same way it would for
a regular resource read, but the trigger was the model calling a tool.

```python
from fastmcp.types import EmbeddedResource

@mcp.tool
async def fetch_docs(query: str) -> list[EmbeddedResource]:
    """Search docs and return matching pages as embedded resources."""
    matches = await _search(query)
    return [
        EmbeddedResource(
            type="resource",
            resource={
                "uri": m.uri,
                "mimeType": "text/markdown",
                "text": m.text,
            },
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

**Subscribing on a stateless HTTP server**

The subscribe call succeeds, but `notify_resource_updated` has no session to push to.
The client waits forever for an update that will never arrive. Either run a stateful
transport or document that your subscriptions are no-ops in stateless mode.

**Prompts that include the entire conversation history**

```python
# ⚠️ Brittle and expensive
@mcp.prompt
def summarize_chat(history: list[dict]) -> list[Message]:
    return [Message(role="user", content=f"Summarize: {history}")]
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
