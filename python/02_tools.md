# Tools in FastMCP 3.x — The Deep Dive

> **Who this is for**: Engineers who finished [01_getting_started.md](01_getting_started.md)
> and want to ship MCP tools that survive contact with production traffic.
> Assumes Python 3.11+, async familiarity, and basic Pydantic.

Tools are the workhorses of MCP. Resources expose data, prompts shape conversations,
but tools are how a model takes action — query a database, hit an API, render a chart,
move money. Get them wrong and you've shipped a footgun. Get them right and the model
behaves like a careful operator.

This file covers the FastMCP 3.x tool surface end-to-end: decorator semantics, sync vs
async dispatch, Pydantic validation, structured output, error contracts, annotations,
versioning, and the failure modes that bite in production.

---

## 1. The `@mcp.tool` Decorator

The decorator inspects the function signature and turns it into an MCP tool definition.
Type hints become JSON Schema, the docstring becomes the tool description, and the
function name becomes the tool name.

```python
from fastmcp import FastMCP

mcp = FastMCP("example")

@mcp.tool
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b
```

What FastMCP generates from that:

| Source                   | MCP field                                    |
|--------------------------|----------------------------------------------|
| Function name `add`      | `tool.name = "add"`                          |
| Docstring                | `tool.description = "Add two numbers."`     |
| Parameter `a: int`       | `inputSchema.properties.a = {type: integer}` |
| Return type `-> int`     | `outputSchema.type = integer` (if declared)  |

Override the name when the Python identifier is awkward or you want a stable wire name
that survives refactoring:

```python
@mcp.tool(name="github_create_issue", description="Create a GitHub issue on a repo.")
def _create_issue_impl(repo: str, title: str) -> dict:
    ...
```

> **Rule**: the docstring **is** the contract the model sees. Treat it as user-facing
> copy. Vague docstrings produce vague tool calls.

---

## 2. Sync vs Async Tools

FastMCP accepts both `def` and `async def` tools. The runtime picks the right dispatch
path based on the function type:

```python
import httpx

@mcp.tool
def cpu_bound(n: int) -> int:
    """Sync — FastMCP runs this on a worker thread, doesn't block the event loop."""
    return sum(i * i for i in range(n))

@mcp.tool
async def io_bound(url: str) -> dict:
    """Async — FastMCP awaits it natively."""
    async with httpx.AsyncClient() as client:
        return (await client.get(url)).json()
```

> **Rule**: if your tool does blocking I/O (network, disk, subprocess), make it `async`
> and use an async client. CPU-bound work can stay sync — FastMCP dispatches to a
> threadpool automatically so the event loop keeps serving other requests.

The cost of getting this wrong:

```python
# ❌ Sync function doing blocking network I/O — stalls the event loop for every other tool
@mcp.tool
def fetch(url: str) -> dict:
    return requests.get(url).json()  # blocks the worker thread for the whole RTT

# ✅ Async with an async client — frees the loop while the request is in flight
@mcp.tool
async def fetch(url: str) -> dict:
    async with httpx.AsyncClient(timeout=10.0) as client:
        resp = await client.get(url)
        resp.raise_for_status()
        return resp.json()
```

⚠️ Mixing `requests` (sync) inside an `async def` is the worst of both worlds — you've
declared async dispatch but you're still blocking the event loop. The linter won't catch
it; code review must.

---

## 3. Pydantic Models for Complex Inputs

Type hints handle the simple cases. For anything nested, constrained, or with defaults,
reach for Pydantic. Validation runs **before** your function is called — bad input is
rejected with a structured error and your code never sees it.

```python
from pydantic import BaseModel, Field
from typing import Annotated

class CreateIssue(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    body: str = ""
    labels: list[str] = Field(default_factory=list, max_length=20)
    priority: Annotated[int, Field(ge=0, le=3)] = 1

@mcp.tool
async def create_issue(repo: str, issue: CreateIssue) -> dict:
    """Create a GitHub issue.

    Args:
        repo: Full repo slug, e.g. "octocat/hello-world".
        issue: The issue payload. Title is required; priority is 0 (low) to 3 (urgent).
    """
    ...
```

Why this is better than raw `dict`:

| Approach                   | Validation         | Schema in MCP        | Errors                     |
|----------------------------|--------------------|----------------------|----------------------------|
| `issue: dict`              | None — you do it   | `{type: object}`     | You raise; model guesses   |
| `issue: CreateIssue`       | Pydantic, automatic| Full nested schema   | Field-level, model-correctable |

The `Annotated[int, Field(ge=0, le=3)]` form lets you attach validators without
sacrificing the readable type. Use it for any bounded primitive — page sizes, ports,
percentages, retry counts.

💡 Use `Field(..., description="...")` on every non-obvious field. Those descriptions
land in the JSON Schema and the model uses them when constructing arguments.

---

## 4. Structured Output

By default, a tool's return value is serialized to text and shown to the model. That's
fine for a string, but lossy for anything structured — the model has to re-parse JSON
out of a text blob.

Declare a Pydantic return type and FastMCP emits both `structuredContent` (the parsed
JSON) and a text fallback. The model gets the best of both worlds: human-readable text
plus a machine-parseable object it can index into directly.

```python
from datetime import datetime
from pydantic import BaseModel

class IssueResult(BaseModel):
    number: int
    url: str
    created_at: datetime
    labels: list[str]

@mcp.tool
async def create_issue(repo: str, issue: CreateIssue) -> IssueResult:
    """Create a GitHub issue and return its identifiers."""
    response = await _gh_post(f"/repos/{repo}/issues", issue.model_dump())
    return IssueResult(
        number=response["number"],
        url=response["html_url"],
        created_at=response["created_at"],
        labels=[lbl["name"] for lbl in response["labels"]],
    )
```

What ends up on the wire:

```json
{
  "content": [{"type": "text", "text": "{\"number\": 42, \"url\": \"...\"}"}],
  "structuredContent": {
    "number": 42,
    "url": "https://github.com/octocat/hello-world/issues/42",
    "created_at": "2026-04-23T10:15:00Z",
    "labels": ["bug"]
  }
}
```

> **Key insight**: an output schema is not just documentation — it's a contract.
> The model can plan multi-step workflows like "create the issue, then comment on
> issue.number" because `number` is guaranteed to be an integer.

You can also return a `dict` and declare a separate output schema, but a Pydantic
return type is simpler and keeps the schema next to the data.

---

## 5. Errors — The Right Way

Tool errors come in two flavors in MCP, and the distinction matters a lot:

| Flavor                | Behavior                                      | When to use                    |
|-----------------------|-----------------------------------------------|--------------------------------|
| **Protocol Error**    | Terminates the call; transport-level failure  | Auth missing, malformed request|
| **Tool Execution Error** | `isError: true` in the result; model sees it| Bad input, business rule failure|

> **New in 2025-11-25 (SEP-1303)**: validation errors now return as Tool Execution
> Errors (`isError: true` with text content) so the model can self-correct, rather
> than as Protocol Errors which terminate the call. This is a meaningful behavior
> change — older clients used to lose the conversation on a bad arg.

In FastMCP, raising `ToolError` returns `isError: true` with your message visible to
the model. That's exactly what you want for any condition the model could fix by
trying again with different arguments:

```python
from fastmcp.exceptions import ToolError

@mcp.tool
async def transfer(from_acct: str, to_acct: str, amount: float) -> dict:
    """Transfer funds between accounts."""
    if amount <= 0:
        raise ToolError(f"amount must be positive, got {amount}")
    if from_acct == to_acct:
        raise ToolError("from_acct and to_acct must differ")
    if not await _account_exists(from_acct):
        raise ToolError(f"unknown account: {from_acct}")
    if not await _account_exists(to_acct):
        raise ToolError(f"unknown account: {to_acct}")
    if (balance := await _balance(from_acct)) < amount:
        raise ToolError(f"insufficient funds: balance={balance}, requested={amount}")
    return await _execute_transfer(from_acct, to_acct, amount)
```

Every `ToolError` message above is something the model can **act on** — fix the sign,
pick a different account, ask the user to top up. That's the whole point.

What about other exceptions? FastMCP catches them too, but by default the message is
**hidden** from the model and replaced with a generic error string. This is a security
default — exception messages frequently leak stack frames, SQL fragments, internal
hostnames, or PII.

```python
# ❌ ValueError message ("connection to db-prod-replica-3.internal failed") leaks
#    infrastructure details to the model and, transitively, to the chat UI
@mcp.tool
async def lookup(user_id: str) -> dict:
    return await db.fetch_one("SELECT * FROM users WHERE id = $1", user_id)
```

If you've audited your exceptions and want the messages to surface, opt in:

```python
mcp = FastMCP("example", mask_error_details=False)  # surface raw exception messages
```

> **Rule**: use `ToolError` for messages you've intentionally written for the model.
> Let everything else stay masked. Never `mask_error_details=False` on a server that
> talks to a production database without an exception-sanitization layer in front.

---

## 6. Tool Annotations

Annotations are **hints** for the host (Claude Desktop, an IDE, a custom client). They
don't change tool behavior — they let well-behaved hosts skip confirmation prompts for
safe calls and surface warnings for dangerous ones.

```python
@mcp.tool(
    annotations={
        "readOnlyHint": True,        # safe to call without confirmation
        "destructiveHint": False,    # doesn't delete or modify
        "idempotentHint": True,      # repeated calls are equivalent
        "openWorldHint": False,      # doesn't reach the open internet
    }
)
def get_user(id: str) -> dict:
    """Look up a user by ID."""
    ...
```

The four standard hints:

| Hint               | Means                                              | Host behavior                  |
|--------------------|----------------------------------------------------|--------------------------------|
| `readOnlyHint`     | Tool only reads state                              | Skip confirmation prompts      |
| `destructiveHint`  | Tool deletes or overwrites data                    | Show a "destructive" warning   |
| `idempotentHint`   | N calls produce the same effect as 1 call          | Safe to retry on transport error|
| `openWorldHint`    | Tool reaches arbitrary external systems            | Mark as network-dependent      |

A pair of contrasting examples:

```python
@mcp.tool(annotations={"readOnlyHint": True, "idempotentHint": True})
def list_files(path: str) -> list[str]:
    """List files in a directory."""
    return os.listdir(path)

@mcp.tool(annotations={"destructiveHint": True, "idempotentHint": False})
def delete_branch(repo: str, branch: str) -> dict:
    """Delete a branch from a Git repository. Irreversible."""
    ...
```

⚠️ Annotations are **advisory**. A malicious server can lie and claim a destructive tool
is read-only. Hosts should treat hints as one input among many — not gospel.

---

## 7. Tool Versioning

FastMCP 3.x lets you publish multiple versions of the same tool side-by-side. This is
the migration story you wanted in 2.x: change a default, add a required field, restrict
a return shape — without breaking clients pinned to the old behavior.

Because the decorator's `description=` **overrides** the function docstring as the
model-facing description, keep public-facing copy in `description=` and use the
docstring for internal migration notes that the model never sees:

```python
from typing import Annotated
from pydantic import Field

async def _search_impl(query: str, limit: int) -> list[dict]:
    # Shared implementation — both versions call this.
    ...

@mcp.tool(
    name="search",
    version="2.0",
    description=(
        "Search the index. Use this version for normal search requests. "
        "Supports a lower default result limit for faster, cheaper calls."
    ),
)
async def search_v2(
    query: Annotated[str, Field(description="The search query to run against the index.")],
    limit: Annotated[
        int,
        Field(ge=1, le=50, description="Maximum number of results to return."),
    ] = 10,
) -> list[dict]:
    """Internal migration notes.

    v2 is the preferred implementation.
    Changed default limit from 50 → 10 to reduce default cost and latency.
    """
    return await _search_impl(query=query, limit=limit)

@mcp.tool(
    name="search",
    version="1.0",
    description=(
        "Search the index using legacy result-limit behavior. "
        "Use only when a client explicitly requires version 1.0."
    ),
)
async def search_v1(
    query: Annotated[str, Field(description="The search query to run against the index.")],
    limit: Annotated[
        int,
        Field(ge=1, le=100, description="Maximum number of results to return."),
    ] = 50,
) -> list[dict]:
    """Internal migration notes.

    v1 kept for backward compatibility.
    Removal target: 2026 Q3, after telemetry shows no remaining v1 traffic.
    Difference from v2: default limit is 50 instead of 10.
    """
    return await _search_impl(query=query, limit=limit)
```

What each piece does:

| Element                | Purpose                                                        |
|------------------------|----------------------------------------------------------------|
| `description=`         | Model/client-facing copy — set this and the docstring is ignored |
| Docstring              | Internal migration notes — developer-only, never exposed       |
| `name="search"`        | Stable public tool name, survives function renames             |
| `version="1.0"/"2.0"` | Explicit version pinning for client negotiation                |
| Function name          | Implementation detail — not exposed when `name=` is set        |
| `Annotated + Field`    | Per-parameter descriptions and validation                      |
| `_search_impl`         | Shared logic — avoids duplicating business code               |

Keep the model-facing `description` focused on tool selection, not operational metadata:

```python
# ✅ Guides the model toward correct version selection
description="Use only when a client explicitly requires version 1.0."

# ❌ Migration bookkeeping in the wrong place
description="v1 kept for old clients. Will be removed in 2026 Q3."
```

Clients see both versions in the tool list and can pin to the version they tested
against. Once your telemetry shows v1 traffic at zero, delete it.

> **Pattern**: deprecate, don't delete. Keep v1 around for at least one release cycle
> after v2 ships, with the removal date in the docstring — not the model-facing
> description.

---

## 8. Icons

Also new in 2025-11-25: tools can declare an icon URL or data URI. Hosts use these to
render tool palettes, autocomplete menus, and audit trails.

```python
@mcp.tool(icon="https://example.com/icons/search.svg")
def search(query: str) -> list[dict]:
    """Search the catalog."""
    ...

# Inline SVG via data URI — no network fetch, works offline
@mcp.tool(icon="data:image/svg+xml;base64,PHN2ZyB...")
def lookup(id: str) -> dict:
    ...
```

💡 Prefer data URIs for first-party tools. External icon URLs make your tool listing a
network dependency and a fingerprinting vector.

---

## 9. Returning Rich Content

Text is the default return shape, but tools can return images, audio, or embedded
resources. FastMCP exposes these as content classes you return directly:

```python
from fastmcp.types import ImageContent, AudioContent, EmbeddedResource

@mcp.tool
def render_chart(data: list[float]) -> ImageContent:
    """Render a line chart of the given series as PNG."""
    png_bytes = _render(data)
    return ImageContent(data=png_bytes, mimeType="image/png")

@mcp.tool
async def transcribe(audio_url: str) -> AudioContent:
    """Fetch an audio file and return its bytes for the model to ingest."""
    async with httpx.AsyncClient() as client:
        bytes_ = (await client.get(audio_url)).content
    return AudioContent(data=bytes_, mimeType="audio/mp3")
```

A tool can also return a list mixing types — say, a chart and a caption:

```python
@mcp.tool
def chart_with_summary(data: list[float]) -> list:
    return [
        ImageContent(data=_render(data), mimeType="image/png"),
        f"Mean: {sum(data) / len(data):.2f}, n={len(data)}",
    ]
```

⚠️ Image and audio bytes are base64-encoded over the wire. A 10 MB image becomes
~13 MB of payload and counts against the model's context window. Resize before
returning.

---

## 10. Real-World Failure Modes

**Catching exceptions and returning an error string**

```python
# ❌ The model sees a successful tool call with text "error: db down".
#    It will happily proceed as if everything worked.
@mcp.tool
async def lookup(id: str) -> str:
    try:
        return await db.fetch(id)
    except Exception as e:
        return f"error: {e}"

# ✅ ToolError sets isError: true so the model knows to retry or apologize.
@mcp.tool
async def lookup(id: str) -> dict:
    try:
        return await db.fetch(id)
    except DatabaseError as e:
        raise ToolError(f"database lookup failed for id={id}: {e}")
```

**Returning a dict without a schema**

```python
# ❌ Only the text representation reaches the model. It has to parse JSON out of text.
@mcp.tool
def stats() -> dict:
    return {"users": 42, "errors": 3}

# ✅ Pydantic return type → structuredContent + text fallback.
class Stats(BaseModel):
    users: int
    errors: int

@mcp.tool
def stats() -> Stats:
    return Stats(users=42, errors=3)
```

**Naming a destructive tool without the hint**

```python
# ⚠️ The host can't warn the user before this fires.
@mcp.tool
def delete_database(name: str) -> dict: ...

# ✅ Both name and annotation make the danger explicit.
@mcp.tool(annotations={"destructiveHint": True, "idempotentHint": False})
def delete_database(name: str) -> dict:
    """PERMANENTLY delete a database. This cannot be undone."""
    ...
```

**Unbounded inputs**

```python
# ❌ Model can pass a 10,000-element list and DoS your service.
@mcp.tool
async def fetch_many(urls: list[str]) -> list[dict]:
    return [await _fetch(u) for u in urls]

# ✅ Bound the list and each element.
class FetchRequest(BaseModel):
    urls: list[Annotated[str, Field(max_length=2000)]] = Field(..., max_length=20)

@mcp.tool
async def fetch_many(req: FetchRequest) -> list[dict]:
    return await asyncio.gather(*[_fetch(u) for u in req.urls])
```

**Sync function doing blocking I/O**

```python
# ❌ Stalls the event loop. Other tools queue behind every HTTP RTT.
@mcp.tool
def fetch(url: str) -> dict:
    return requests.get(url).json()

# ✅ Async function with an async client.
@mcp.tool
async def fetch(url: str) -> dict:
    async with httpx.AsyncClient(timeout=10.0) as client:
        return (await client.get(url)).json()
```

**Leaking exception details to the model**

```python
# ❌ With mask_error_details=False, the model sees:
#    "asyncpg.exceptions.UndefinedColumnError: column users.ssn does not exist"
mcp = FastMCP("example", mask_error_details=False)

# ✅ Default masking + explicit ToolError for the cases the model should know about.
mcp = FastMCP("example")  # mask_error_details=True is the default

@mcp.tool
async def get_user(id: str) -> User:
    if not _is_valid_id(id):
        raise ToolError(f"id must be a UUID, got {id!r}")
    return await db.fetch_one(...)
```

---

## Cross-References

- **Previous**: [01_getting_started.md](01_getting_started.md)
- **Next**: [03_resources_prompts.md](03_resources_prompts.md)
- **Related**: [../fundamentals/03_primitives.md](../fundamentals/03_primitives.md)
  — the protocol-level definition of tools, resources, and prompts
- **Related**: [../typescript/02_tools_zod.md](../typescript/02_tools_zod.md)
  — the same patterns in the TypeScript SDK with Zod schemas

---

**Next**: [03_resources_prompts.md — Resources and Prompts](03_resources_prompts.md)
