# Sampling and Elicitation: When the Server Calls Back

> **Who this is for**: Engineers who can build FastMCP tools and resources but haven't
> realized servers can call *back into the host* — to borrow its LLM or to ask the user
> a follow-up question mid-tool. Assumes you've read
> [04_context_lifespan.md](04_context_lifespan.md) and understand the `Context` object.

Most MCP tutorials treat the server as a passive endpoint: client calls a tool, server
returns a result. That's the easy half. The other half — the half that confuses almost
everyone — is that an MCP server can pause mid-tool and ask the host for two things:

- **An LLM completion** (`ctx.sample()`) — "run this prompt through whatever model the
  user has and give me the answer."
- **A user response** (`ctx.elicit()`) — "I need one more piece of info; please show
  the user this form."

Both flip the request direction: server → client → host → back to server. Once you see
the pattern, a lot of designs that felt impossible (model-agnostic agents, interactive
confirmations, multi-step wizards) become routine.

---

## 1. Why These Exist

Both features solve real problems that *can't* be solved by tools alone.

**Sampling** exists so a server doesn't have to ship its own model credentials. A
"summarize this URL" tool that bundles an OpenAI key would (a) cost the developer money
every time someone installs it, (b) lock users into one vendor, and (c) leak that key
through every distribution channel. With sampling, the server hands a prompt to the
host, the host runs the inference using whatever model the user is *already paying for*,
and the result comes back over the same MCP connection. The server stays
model-agnostic and credential-free.

**Elicitation** exists for the moment a tool starts running and discovers the model
didn't pass enough information. Classic example: `wire_transfer(to_iban, amount)` —
the model picked an amount, but you'd really like a human to confirm before moving
money. Without elicitation your only options are erroring out (frustrating) or guessing
(dangerous). With elicitation you ask the user directly and proceed based on their
answer.

> **Key insight**: tools are *model → server*. Sampling and elicitation are
> *server → host*. They are the only ways for your server to initiate a request
> against the host instead of just responding to one.

---

## 2. Capability Gating

Both features require the **client** to advertise the capability during the MCP
`initialize` handshake. Not every host supports both — Claude Desktop and Cursor have
sampling and elicitation, but a minimal CLI client or a custom integration may not.

If the host didn't advertise the capability, calling `ctx.sample()` or `ctx.elicit()`
raises an error at runtime. You should check before you call:

```python
from fastmcp import Context, FastMCP
from fastmcp.exceptions import ToolError

mcp = FastMCP("capability-aware")

@mcp.tool
async def needs_sampling(prompt: str, ctx: Context) -> str:
    if "sampling" not in ctx.session.client_capabilities:
        raise ToolError("This server requires a host with sampling support.")
    # ... safe to call ctx.sample() below
    ...

@mcp.tool
async def needs_elicitation(ctx: Context) -> str:
    if "elicitation" not in ctx.session.client_capabilities:
        raise ToolError("This server requires a host with elicitation support.")
    ...
```

⚠️ Don't assume capability presence based on the host name — versions matter. Older
Claude Desktop builds had sampling but not elicitation; the SEP-1330 enum support
landed in late 2025. Always gate on `client_capabilities`, not on a hardcoded list.

> **Rule**: every tool that calls `ctx.sample()` or `ctx.elicit()` should either
> guard the capability up front or be documented as "host must support X" in its
> docstring. Surprise runtime errors are a terrible UX for a tool the user just
> asked their model to invoke.

---

## 3. Sampling — The Basics

The mental model: you're writing a function that needs an LLM, but you don't want to
pick the LLM. You hand the host a list of messages, optionally a system prompt, and
some preferences. The host decides which model to use, runs the call, and returns the
result.

```python
from fastmcp import Context, FastMCP
from fastmcp.types import SamplingMessage, ModelPreferences
import httpx

mcp = FastMCP("doc-summarizer")

async def _fetch_text(url: str) -> str:
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.text

@mcp.tool
async def summarize(url: str, ctx: Context) -> str:
    """Fetch a URL and ask the host's model to summarize it."""
    text = await _fetch_text(url)

    result = await ctx.sample(
        messages=[
            SamplingMessage(role="user", content=f"Summarize:\n\n{text[:8000]}"),
        ],
        system_prompt="You write tight three-sentence summaries.",
        model_preferences=ModelPreferences(
            # Hints are advisory — host may pick something else
            hints=[{"name": "claude-3-5-sonnet"}],
            intelligence_priority=0.7,
            speed_priority=0.3,
        ),
        max_tokens=200,
    )
    return result.content[0].text
```

A few things worth understanding about this call:

1. **The host picks the model.** Your hints are exactly that — hints. If the user
   only has GPT-4 configured, that's what runs.
2. **The user may be prompted to approve.** Most hosts implement a human-in-the-loop
   step where the user sees the prompt before it executes. Treat sampling as a
   privileged operation that may take seconds (because a human is reading).
3. **The result returns over the same MCP connection.** No webhooks, no polling — your
   `await` resolves with the assistant message.

> **Key insight**: sampling turns your server into a meta-agent. It can decompose
> a tool call into sub-prompts without ever touching a model API directly.

---

## 4. Model Preferences

`ModelPreferences` has three numeric knobs (each `0.0`–`1.0`) and a list of name hints.
The host weighs these against what the user has configured.

| Knob                     | Meaning                                                          |
|--------------------------|------------------------------------------------------------------|
| `intelligence_priority`  | How much you care about raw capability (reasoning, code, etc.)   |
| `speed_priority`         | How much you care about latency — prefer fast/small models       |
| `cost_priority`          | How much you care about being cheap (small or local models)      |
| `hints`                  | Name-substring hints like `"claude-3-5"` or `"gpt-4"`            |

The three priorities don't have to sum to 1; they're independent weights. A "summarize
this paragraph" tool might pass `speed_priority=0.9, intelligence_priority=0.2`. A
"write a unit test for this function" tool would invert that.

```python
# Latency-sensitive: classification, extraction, simple rewrites
ModelPreferences(speed_priority=0.9, cost_priority=0.7, intelligence_priority=0.2)

# Capability-sensitive: code generation, multi-step reasoning, planning
ModelPreferences(intelligence_priority=0.9, speed_priority=0.2, cost_priority=0.1)

# Cost-sensitive: bulk processing, background jobs
ModelPreferences(cost_priority=0.9, intelligence_priority=0.4, speed_priority=0.4)
```

💡 Hints can match by substring — `"claude-3-5"` matches both `claude-3-5-sonnet` and
`claude-3-5-haiku`. Use the broadest hint that still gets you what you want; over-
specifying causes the host to fall back to defaults when your exact match isn't there.

---

## 5. Multi-Turn Sampling

`messages` is a list. You can pass prior assistant turns to maintain context across
several inferences within one tool call:

```python
@mcp.tool
async def generate_test(function_source: str, ctx: Context) -> str:
    """Generate a pytest function in two stages: outline, then code."""
    outline_result = await ctx.sample(
        messages=[
            SamplingMessage(
                role="user",
                content=f"Outline a unit test for this function:\n\n{function_source}",
            ),
        ],
        max_tokens=300,
    )
    outline = outline_result.content[0].text

    # Second call sees the outline as prior context
    messages = [
        SamplingMessage(role="user", content="Outline a unit test for parse_csv"),
        SamplingMessage(role="assistant", content=outline),
        SamplingMessage(role="user", content="Now write the actual pytest function"),
    ]
    result = await ctx.sample(messages=messages, max_tokens=1000)
    return result.content[0].text
```

⚠️ Each `ctx.sample()` is a separate inference — there's no implicit memory between
calls. If you want context to carry, you have to assemble it into `messages` yourself.

---

## 6. Elicitation — The Basics

Elicitation is the *opposite* direction: instead of asking the host's LLM, you ask the
host's *user*. You hand it a Pydantic schema, it renders a form (or a TUI prompt, or
whatever the host supports), the user fills it out, and you get a typed result back.

```python
from pydantic import BaseModel, Field
from fastmcp import Context, FastMCP
from fastmcp.exceptions import ToolError

mcp = FastMCP("treasury")

class TransferConfirm(BaseModel):
    confirm: bool = Field(..., description="Are you sure you want to transfer this amount?")
    note: str = Field("", description="Optional reason for the transfer")

@mcp.tool
async def wire_transfer(to_iban: str, amount: float, ctx: Context) -> dict:
    """Initiate a wire transfer. Amounts over 10k require user confirmation."""
    if amount > 10_000:
        result = await ctx.elicit(
            message=f"Confirm transfer of {amount} to {to_iban}?",
            schema=TransferConfirm,
        )
        if result.action != "accept" or not result.data.confirm:
            return {"status": "cancelled"}
    return await _execute_transfer(to_iban, amount)
```

Three things to internalize:

1. **`result.action`** is one of `"accept"`, `"decline"`, or `"cancel"`. Always
   check it *before* touching `result.data`. A declined or cancelled elicitation
   has no data.
2. **The schema is a real Pydantic model.** `result.data` is a validated instance,
   not a dict. Type checkers know about it.
3. **The host renders the form.** You don't get to control the UI — only the schema,
   field descriptions, and the prompt message. Write descriptions assuming a human
   will read them.

> **Rule**: elicitation always gives the user an out. If your tool can't proceed
> without an answer, design the cancellation path explicitly — return a sensible
> status, raise a clean error, or undo any partial work. Never just hang.

---

## 7. Elicitation — Single/Multi-Select Enums (SEP-1330, 2025-11-25)

The 2025-11-25 spec revision (SEP-1330) added first-class enum rendering. A field
typed as `str` with an `enum` constraint renders as a single-select dropdown; a field
typed as `list[str]` with an `enum` constraint renders as a multi-select.

```python
from pydantic import BaseModel, Field

class ChooseDB(BaseModel):
    database: str = Field(
        ...,
        json_schema_extra={"enum": ["prod", "staging", "dev"]},
    )
    schemas: list[str] = Field(
        default_factory=list,
        json_schema_extra={"enum": ["public", "audit", "internal"]},
    )
```

For human-readable labels, use the `oneOf`/`const`/`title` form — hosts that
implement SEP-1330 will display the title and submit the const:

```python
class ChooseEnv(BaseModel):
    env: str = Field(
        ...,
        json_schema_extra={
            "oneOf": [
                {"const": "prod", "title": "Production"},
                {"const": "stg", "title": "Staging"},
                {"const": "dev", "title": "Development"},
            ],
        },
    )
```

⚠️ Older hosts that haven't shipped SEP-1330 will fall back to a free-text input.
If you target both, accept the broadest set of valid strings server-side and
re-validate after the elicitation returns.

---

## 8. When NOT to Use Elicitation

Elicitation is a sharp tool. The temptation, once you have it, is to use it for
everything — and that's how you build a tool that nobody wants to invoke twice.

❌ **For required arguments.** If your tool needs a `repo_url`, declare it in the
tool schema. The model will pass it. Elicitation should never be the model's first
encounter with a parameter you've always known about.

❌ **For sensitive data the user shouldn't have to retype.** API keys, tokens,
passwords — these belong in the host's secrets store or your server's
configuration. Asking the user to paste their AWS access key into an elicitation
prompt is both a UX failure and (on hosts that log inputs) a security one.

❌ **As a chat replacement.** One missing piece per call. If you find yourself
chaining three elicitations, the right answer is usually a better tool schema or
a single elicitation with a richer Pydantic model.

✅ **For confirmations on destructive actions.** Wire transfers, deletes, pushes
to production. The model can't reliably decide whether the human really meant it.

✅ **For genuinely ambiguous user intent.** "Did you mean the prod or staging
database?" when both are plausible from context.

✅ **For auth or scope decisions** that the host hasn't already negotiated.

> **Key insight**: every elicitation call is a synchronous wait on a human.
> Treat them like modal dialogs in a desktop app — sparingly, only when you
> truly cannot proceed without input.

---

## 9. Combining Sampling and Elicitation

Once you have both, you can build tools that interleave human and model decisions
within a single call. This is where MCP starts to feel like an actual agent
framework rather than a function-calling protocol.

```python
from pydantic import BaseModel, Field
from fastmcp import Context, FastMCP
from fastmcp.types import SamplingMessage
from fastmcp.exceptions import ToolError

mcp = FastMCP("blog-writer")

class AngleChoice(BaseModel):
    description: str = Field(..., description="One-sentence description of the angle")

@mcp.tool
async def write_blog_post(topic: str, ctx: Context) -> str:
    """Ask the user for an angle, then have the host's LLM draft the post."""
    angle = await ctx.elicit(
        message=f"What angle for the post on '{topic}'?",
        schema=AngleChoice,
    )
    if angle.action != "accept":
        raise ToolError("user cancelled")

    draft = await ctx.sample(
        messages=[
            SamplingMessage(
                role="user",
                content=(
                    f"Write a 500-word post on {topic} from this angle: "
                    f"{angle.data.description}"
                ),
            ),
        ],
        max_tokens=800,
    )
    return draft.content[0].text
```

The flow:

```
model ──invokes write_blog_post──> server
                                     │
                                     │── ctx.elicit() ──> host ──> user
                                     │<── angle ─────────────────────│
                                     │
                                     │── ctx.sample() ──> host ──> user-approved LLM
                                     │<── draft ─────────────────────────────────────│
                                     │
model <── returns draft ───────────  │
```

Three round-trips for one tool call. All of it riding on the same MCP session.

---

## 10. Failure Modes

❌ **Calling `ctx.sample()` from a stateless HTTP server with no GET stream.** The
server-initiated request needs a way back to the client. Plain request/response HTTP
gives you no channel for the server to push the sampling request through. If you're
running over HTTP, use the streamable transport (with sessions) or fall back to
stdio. Stateless `POST`-only servers will raise on the first sampling call.

❌ **Looping `ctx.sample()` in a tight loop without a guard.** Each call burns the
host's tokens. A well-meaning "summarize each section" loop on a 200-section document
will quietly spend a lot of money. Always cap the iteration count and consider
returning partial results when you hit it:

```python
MAX_SAMPLES = 20

results = []
for i, section in enumerate(sections):
    if i >= MAX_SAMPLES:
        results.append(f"... ({len(sections) - i} sections truncated)")
        break
    out = await ctx.sample(
        messages=[SamplingMessage(role="user", content=f"Summarize: {section}")],
        max_tokens=100,
    )
    results.append(out.content[0].text)
```

⚠️ **Elicitation prompts that include sensitive data from prior tool returns.**
Some hosts log elicitation messages for audit purposes. If your prompt embeds a
just-fetched secret ("Confirm rotation of token `sk_live_…`?"), that secret may
end up in logs you don't control. Be conservative — say *what* is being changed,
not the secret itself.

❌ **Treating elicitation as "I'll always get an answer."** Users cancel. They
close dialogs. They walk away. Always handle `result.action != "accept"`
*before* you touch `result.data`. Accessing `.data` on a cancelled result is
how you ship a tool that crashes the moment a user clicks Cancel.

```python
# ❌ Wrong — assumes the user accepted
result = await ctx.elicit(message="Pick one", schema=Choice)
chosen = result.data.option   # AttributeError on cancel

# ✅ Right — branch on action first
result = await ctx.elicit(message="Pick one", schema=Choice)
if result.action != "accept":
    return {"status": "cancelled"}
chosen = result.data.option
```

⚠️ **Sampling latency surprises.** A `ctx.sample()` call can take tens of seconds
(model inference + human approval). If your tool is also doing other I/O, set
explicit timeouts on everything else so a slow sample doesn't get blamed on your
HTTP fetch. Logs that say "tool took 47s" are useless when you don't know which
component owned the time.

---

## 11. Mental Model Recap

| Direction              | Mechanism            | Who responds        | Use it for                          |
|------------------------|----------------------|---------------------|-------------------------------------|
| Client → Server        | Tools, resources     | Server code         | Normal tool calls                   |
| Server → Host (LLM)    | `ctx.sample()`       | Host's model        | Model-agnostic inference            |
| Server → Host (User)   | `ctx.elicit()`       | Human user          | Confirmations, missing input        |
| Server → Host (Logs)   | `ctx.info/warn/...`  | Host's log surface  | Progress, diagnostics               |

If you remember nothing else: **anything that needs the host's resources — its
model, its user, its logs — flows through `ctx`. Tools that try to do these things
themselves (bundled API keys, blocking `input()` calls, `print` to stderr) are
fighting the protocol.**

> **Related reading**: [`fundamentals/03_primitives.md`](../fundamentals/03_primitives.md)
> for the protocol-level view of sampling and elicitation, and
> [`typescript/05_advanced.md`](../typescript/05_advanced.md) for the same patterns
> in the TypeScript SDK.

---

**Next**: [06_middleware.md — Middleware](06_middleware.md)
