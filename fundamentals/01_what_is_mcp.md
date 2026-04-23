# What Is MCP?

> **Who this is for**: Engineers who have heard of the Model Context Protocol but want
> to understand what problem it actually solves. You know what an LLM is, you have
> probably called a function-calling API, but you do not yet speak MCP's vocabulary.
> By the end of this file you will know why MCP exists, what it is (and is not), and
> who is shipping it as of April 2026.

---

## 1️⃣ The Problem: M × N Integrations

LLMs are useless in isolation. To do real work they need **context** (your files, your
database, your calendar, your codebase) and **actions** (send an email, open a PR,
deploy a service). For two years after ChatGPT shipped, every host application solved
this the same way: hand-written, bespoke integrations. One client, one tool, one glue
layer.

This does not scale.

```
                 ── BEFORE MCP ──

   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   Claude    │  │   ChatGPT   │  │   Cursor    │  │  Custom App │
   │   Desktop   │  │   Desktop   │  │             │  │             │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │                │
          │  ╔═════════════╪════════════════╪════════════════╪═════╗
          │  ║   custom    │   custom       │   custom       │     ║
          │  ║   glue      │   glue         │   glue         │     ║
          │  ║   per pair  │                │                │     ║
          │  ╚═════════════╪════════════════╪════════════════╪═════╝
          ▼                ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   GitHub    │  │   Postgres  │  │    Slack    │  │   Filesys   │
   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘

   M clients × N tools  =  M × N hand-written adapters
```

With **4 clients** and **4 tools**, that is **16 integrations**. With 10 clients and
100 tools — the realistic ecosystem — that is **1,000 integrations**, each owned by
some team that may or may not still be maintaining it. Every client reimplements the
same "talk to GitHub" plumbing. Every tool vendor writes the same OAuth flow against
every host that wants to support them.

> **Principle**: Whenever you see an `M × N` problem in systems design, the answer is
> almost always to introduce a shared abstraction layer that turns it into `M + N`.
> MCP is that layer for AI applications and the tools they need to reach.

---

## 2️⃣ The Solution: A Shared Protocol

MCP — the **Model Context Protocol** — is an open specification that defines exactly
how an AI application talks to an external tool, data source, or workflow. Once a tool
speaks MCP, every MCP-compatible client can use it. Once a client speaks MCP, it can
reach every MCP server in existence.

```
                 ── AFTER MCP ──

   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   Claude    │  │   ChatGPT   │  │   Cursor    │  │  Custom App │
   │   Desktop   │  │   Desktop   │  │             │  │             │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │                │
          └────────────────┴────────┬───────┴────────────────┘
                                    │
                          ┌─────────▼──────────┐
                          │    MCP protocol    │     ← one wire format,
                          │   (JSON-RPC 2.0)   │       one capability model,
                          └─────────┬──────────┘       one auth story
                                    │
          ┌────────────────┬────────┴───────┬────────────────┐
          ▼                ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │   GitHub    │  │   Postgres  │  │    Slack    │  │   Filesys   │
   │   server    │  │   server    │  │   server    │  │   server    │
   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘

   M clients + N servers  =  M + N integrations
```

Now the math is **M + N**. Add a new client and it inherits the entire server
ecosystem on day one. Add a new server and every client can use it without shipping
new code. This is the entire pitch — and it is the same pitch that USB, HTTP, LSP, and
SQL all made before it.

---

## 3️⃣ Mental Model: USB-C for AI

The cleanest analogy, and the one Anthropic leans on in the spec: **MCP is USB-C for
AI applications.**

| USB-C                                    | MCP                                                       |
|------------------------------------------|-----------------------------------------------------------|
| One physical port, many device classes   | One JSON-RPC protocol, many capabilities                  |
| Device declares what it is on connect    | Server declares its tools/resources/prompts on initialize |
| Host negotiates power, data, video       | Host negotiates protocol version, capabilities            |
| You don't care what's inside the cable   | You don't care what language the server is written in     |
| Works the same on a laptop or a phone    | Works the same in Claude Desktop or VS Code               |

A second analogy that resonates with engineers who lived through the IDE wars:
**MCP is LSP for tools.** Before the Language Server Protocol (Microsoft, 2016), every
editor wrote its own Python intellisense, its own Go formatter, its own Rust analyzer.
LSP collapsed that matrix — write one language server, get every editor for free.
MCP is making the same bet for the model-to-tool boundary.

> **Key insight**: MCP is not a new model, not a new framework, not a runtime. It is
> a **wire protocol** plus a small set of conventions about what messages mean. That
> is why it can be neutral across model vendors and host applications — it doesn't
> compete with any of them.

---

## 4️⃣ How It Differs from Raw Function Calling

If you have used OpenAI's function calling or Anthropic's tool use, you might be
asking: don't those already solve this? They solve part of it. Here is the boundary.

```
   ┌──────────────────────────────────────────────────────────────────┐
   │                                                                  │
   │   FUNCTION CALLING               MCP                             │
   │   ─────────────────              ───                             │
   │                                                                  │
   │   "Here's a JSON schema          "Here's a protocol for          │
   │    for a function the model       discovering, invoking, and     │
   │    can request to call."          observing tools, resources,    │
   │                                   and prompts at runtime."       │
   │                                                                  │
   │   In-process, in-prompt          Out-of-process, over a transport│
   │   Static — defined per call      Dynamic — discovered live       │
   │   Tools only                     Tools + resources + prompts +   │
   │                                  sampling + elicitation + roots  │
   │   No transport, no auth,         stdio + Streamable HTTP,        │
   │   no lifecycle                   OAuth 2.1, full lifecycle       │
   │   Vendor-specific schema         Vendor-neutral spec             │
   │                                                                  │
   └──────────────────────────────────────────────────────────────────┘
```

Function calling is the **mechanism** the model uses to express "I want to call this
thing." MCP is the **system** that defines where that thing lives, how it advertises
itself, how it is invoked, what it can return, how it is authenticated, and how its
lifecycle is managed. They compose: under the hood an MCP client typically still
uses the host model's function-calling primitive to surface MCP tools to the LLM.

> **Rule**: If your tool definition lives next to your prompt, you are doing function
> calling. If your tool runs as a separate process or service that any compliant
> client can connect to, you are doing MCP.

---

## 5️⃣ What MCP Is — and Is Not

A short list to anchor expectations before you go deeper.

**MCP is:**

- ✅ A JSON-RPC 2.0 message protocol for client-server communication
- ✅ A small vocabulary: `tools`, `resources`, `prompts`, `sampling`, `elicitation`, `roots`
- ✅ Two official transports: **stdio** (local) and **Streamable HTTP** (remote)
- ✅ A capability negotiation model — both sides declare what they support
- ✅ An open specification with reference SDKs in Python, TypeScript, Go, Rust, Java, C#, Kotlin, Swift, Ruby, PHP
- ✅ Model-agnostic by design — works with Claude, GPT, Gemini, Llama, anything

**MCP is not:**

- ❌ A model, a framework, or a runtime
- ❌ An agent loop or planner — it's the layer below those
- ❌ A replacement for REST or GraphQL APIs (it sits in front of them)
- ❌ Tied to Anthropic — see Section 6
- ❌ A way to make models smarter — it gives them more reach, not more reasoning
- ❌ A security boundary on its own — your server still has to authenticate and authorize

⚠️ Common confusion: people see "MCP server" and assume it means a Claude-specific
plugin. It doesn't. The same server binary works against Claude Desktop, ChatGPT
Desktop, VS Code, Cursor, Windsurf, and every other compliant host.

💡 If you find yourself asking "but how does the model decide when to call this?" —
that is a host concern, not an MCP concern. MCP delivers the menu; the host decides
how to put items in front of the model.

---

## 6️⃣ Who Is Using It (April 2026)

MCP shipped from Anthropic in **November 2024**. Eighteen months in, the adoption
curve looks less like "promising new standard" and more like "the way this is done
now."

| Surface                       | Status                                              |
|-------------------------------|-----------------------------------------------------|
| Claude Desktop / Claude Code  | First-party, native since launch                    |
| ChatGPT Desktop               | Shipped MCP support in early 2026                   |
| VS Code (1.99+)               | Built-in MCP client, GA                             |
| Cursor                        | Built-in, configured per workspace                  |
| Windsurf (Codeium)            | Built-in                                            |
| Cline                         | Built-in                                            |
| Zed, JetBrains, Neovim        | Plugins / first-party support landing through 2025–2026 |

The numbers, as of this writing:

- **~97M monthly SDK downloads** across the official language SDKs
- **10,000+ active MCP servers** in the wild
- **~6,400 servers** listed in the official **MCP Registry**
- Reference servers from AWS, Cloudflare, GitHub, Stripe, Linear, Notion, Sentry, and most major SaaS vendors

In **December 2025**, Anthropic donated MCP to the **Linux Foundation** under the
newly formed **Agentic AI Foundation (AAIF)**. The current spec version is
**2025-11-25** — released on the one-year anniversary of MCP's launch. Governance now
sits with a multi-vendor steering committee; Anthropic is one voice among many. This
is the same playbook OpenTelemetry, Kubernetes, and OpenAPI followed, and it is the
single biggest signal that MCP is not going to be re-platformed out from under you.

> **Key insight**: A protocol only matters if both sides adopt it. With Anthropic and
> OpenAI both shipping MCP in their flagship clients, and Microsoft shipping it in
> VS Code, the two-sided market problem is solved. Building an MCP server in 2026 is
> the equivalent of building a website with HTML — you don't have to defend the choice.

---

## 7️⃣ Where This Is Heading

The 2026 roadmap, based on AAIF working group output and the public spec milestones,
is dominated by four themes. These will be covered in depth later in this knowledge
base; for now, just know they exist.

```
   ┌──────────────────────────────────────────────────────────┐
   │                                                          │
   │   1. Multi-server orchestration                          │
   │      Hosts coordinating dozens of servers per session,   │
   │      with conflict resolution and per-tool permissions.  │
   │                                                          │
   │   2. Stronger remote story                               │
   │      OAuth 2.1 hardening, mTLS, scoped tokens,           │
   │      enterprise SSO patterns for hosted servers.         │
   │                                                          │
   │   3. Agent-to-agent (A2A) bridges                        │
   │      MCP as the substrate for one agent to expose its    │
   │      capabilities to another.                            │
   │                                                          │
   │   4. Streaming + structured output                       │
   │      Tool results that stream progress, return rich      │
   │      typed payloads, and support partial cancellation.   │
   │                                                          │
   └──────────────────────────────────────────────────────────┘
```

The direction of travel is clear: MCP is moving from "expose a tool to a chatbot" to
"the connective tissue for agentic systems." If you are building an LLM-powered
product in 2026, the question is no longer *whether* to support MCP, but *which side*
of the protocol you are on — host, server, or both.

---

## 8️⃣ Why This Matters for You

The argument cuts differently depending on which side you sit on. Both sides win, but
they win for different reasons.

**If you are a tool / data / SaaS vendor (server author):**

- Write the integration **once**. Every current and future MCP host can use it.
- Stop maintaining a Claude plugin, an OpenAI plugin, a LangChain integration, a Cursor extension, etc.
- Your existing REST API stays exactly as it is — the MCP server is a thin facade.
- You inherit a permission and discovery story you did not have to design.

**If you are a host / IDE / agent platform author (client author):**

- You ship a runtime that is immediately useful with **thousands** of existing servers.
- You don't have to convince every SaaS vendor to write a custom plugin for you.
- You get to focus on what actually differentiates your product — UX, model routing,
  agent loops — instead of writing the 47th GitHub integration.
- The capability negotiation model means servers can evolve without breaking you.

**If you are an application engineer (consumer of both):**

- You can stand up a private MCP server for your team's internal tools in an afternoon,
  and every developer's IDE picks it up.
- You stop writing prompt-glue to describe APIs to the model. The server does it.
- Your team's tooling is no longer locked to whichever LLM vendor you started with.

> **Principle**: The right time to adopt a connector standard is when both ends of
> the connection have committed to it. For MCP, that moment was Q1 2026.

---

## 9️⃣ What's Next in This Guide

You now have the *why*. The rest of `fundamentals/` builds the *how*:

- **[02_architecture.md](02_architecture.md)** — Hosts, clients, servers, the trust boundary, and the full message flow
- **[03_primitives.md](03_primitives.md)** — Tools, resources, prompts, sampling, elicitation, roots
- **[04_lifecycle.md](04_lifecycle.md)** — Initialize, capability negotiation, operation, shutdown
- **[05_transports.md](05_transports.md)** — stdio vs Streamable HTTP, and when to use which

When you are ready to write code, jump to a language entry point:

- Python: **[../python/01_getting_started.md](../python/01_getting_started.md)**
- TypeScript: **[../typescript/01_getting_started.md](../typescript/01_getting_started.md)**

---

**Next**: [02_architecture.md — Hosts, Clients, Servers](02_architecture.md)
