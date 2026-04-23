# The MCP Ecosystem in April 2026

> **Who this is for**: An engineer or platform team trying to figure out where
> their server fits and which infrastructure to adopt. Assumes you've already
> read the production track ([deployment](../production/06_deployment.md),
> [auth](../production/02_authentication.md), [security](../production/05_security.md))
> and now need a map of the surrounding ecosystem — hosts, registries, gateways,
> and governance — before committing to a stack.

---

## 1. The Ecosystem at a Glance

Two things changed the shape of MCP between late 2025 and early 2026: the
spec moved under foundation governance, and the host count crossed the
threshold where "MCP-compatible" became a checkbox feature rather than a
differentiator. As of April 2026:

| Dimension | State (April 2026) |
|-----------|--------------------|
| **Current spec** | `2025-11-25` |
| **Steward** | Linux Foundation / Agentic AI Foundation (AAIF), donated December 2025 |
| **Monthly SDK downloads** | ~97M across all language SDKs |
| **Active servers** | 10,000+ in production use |
| **Official Registry** | ~6,400 listed servers |
| **Community registries** | Tens of thousands of additional servers |
| **Cross-vendor parity** | The same server runs against Anthropic, OpenAI, and IDE hosts without modification |

The cross-vendor point matters more than the raw download numbers. Eighteen
months ago, "MCP" effectively meant "Claude Desktop." Today a server you
publish runs against Claude, ChatGPT, and every major IDE without any
host-specific code paths. That is the single biggest architectural shift since
the protocol launched, and it's why the gateway and registry layers below
exist as standalone product categories.

```
                       ┌──────────────────────┐
                       │   MCP Spec 2025-11-25 │
                       │  (AAIF / LF stewarded)│
                       └──────────┬───────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
   ┌────▼─────┐             ┌─────▼─────┐             ┌─────▼─────┐
   │  Hosts   │             │ Registries│             │  Gateways │
   │ (Claude, │             │ (Official,│             │  (Kong,   │
   │  ChatGPT,│             │  MCP.so,  │             │  MCPX,    │
   │  Cursor) │             │  Smithery)│             │  IBM)     │
   └────┬─────┘             └─────┬─────┘             └─────┬─────┘
        │                         │                         │
        └─────────────────────────┼─────────────────────────┘
                                  │
                          ┌───────▼────────┐
                          │  Your Server   │
                          │  (one codebase)│
                          └────────────────┘
```

> **Key insight**: Treat MCP in 2026 like HTTP in 2005 — the protocol is
> settled, the value has migrated to the layer above (frameworks, gateways,
> registries) and the layer below (hosting platforms). You will spend more
> time choosing infrastructure than wrestling with the wire format.

---

## 2. Host Applications

A "host" is anything that opens an MCP connection on behalf of a model.
Some are GUI desktop apps, some are IDEs, some are browser-based, and one is
a terminal CLI. They differ in transports supported, auth flows, and how
much of the spec they actually implement.

| Host                | Transports             | Auth      | OS                       | Notes                                                            |
|---------------------|------------------------|-----------|--------------------------|------------------------------------------------------------------|
| **Claude Desktop**  | stdio, Streamable HTTP | OAuth 2.1 | macOS, Windows           | Originator; gold-standard reference impl                         |
| **Claude Code** (CLI) | stdio, Streamable HTTP | OAuth 2.1 | macOS, Linux, Windows    | Anthropic's terminal coding agent                                |
| **ChatGPT Desktop** | stdio, Streamable HTTP | OAuth 2.1 | macOS, Windows           | Adopted MCP early 2026; Settings → Advanced → Edit MCP Config     |
| **VS Code** (1.99+) | stdio, Streamable HTTP | OAuth 2.1 | Cross-platform           | Native via GitHub Copilot Chat                                   |
| **Cursor**          | stdio, Streamable HTTP | OAuth 2.1 | Cross-platform           | Strong MCP UX                                                    |
| **Windsurf**        | stdio, Streamable HTTP | OAuth 2.1 | Cross-platform           |                                                                  |
| **Cline** (VS Code ext) | stdio, Streamable HTTP | OAuth 2.1 | Cross-platform       |                                                                  |
| **Web ChatGPT**     | Streamable HTTP        | OAuth 2.1 | Browser                  | Browser-only — needs CORS                                        |
| **Web Claude** (claude.ai) | Streamable HTTP | OAuth 2.1 | Browser                  | Browser-only — needs CORS                                        |

A few things this table doesn't say out loud:

- **Browser hosts cannot launch stdio servers.** If you want your server
  reachable from claude.ai or chatgpt.com, it must be a remote HTTP server
  with proper CORS headers. A `command: "uvx ..."` entry is invisible to the
  browser.
- **Desktop hosts can do both.** stdio is fine for local-only servers (file
  systems, local databases, dev tools). HTTP is fine for hosted services.
  Most production servers ship both modes from the same codebase.
- **"OAuth 2.1" is shorthand for the MCP auth flow.** All listed hosts
  implement the dynamic client registration + PKCE pattern from the spec; you
  don't need to special-case any of them on the server side.
- **Spec lag is real.** Claude Desktop and Claude Code track the spec within
  a release. Other hosts may be one or two revisions behind on newer features
  like elicitation or tasks.

### Picking a transport based on the host matrix

If you only care about a subset of hosts, the matrix simplifies:

| If you target...                    | Transport you need                  |
|-------------------------------------|-------------------------------------|
| Only Claude Desktop / Claude Code   | stdio is enough                     |
| All desktop hosts                   | stdio (HTTP optional)               |
| Any browser host                    | Streamable HTTP + CORS              |
| The full matrix                     | Both — and they should share code   |

In practice "the full matrix" is almost always the right answer because the
incremental work to add HTTP once you have stdio is small (the SDKs handle
both with the same handler functions), and the addressable user base roughly
doubles.

> **Rule**: Build for stdio + Streamable HTTP from day one. Anything narrower
> locks you out of half the host matrix.

---

## 3. Configuration Formats

Despite the variety of hosts, they nearly all read a Claude-Desktop-compatible
JSON file. This is one of the quiet wins of the ecosystem — you can copy a
config snippet from a server's README into any host without translation.

```json
{
  "mcpServers": {
    "weather": {
      "command": "uvx",
      "args": ["mcp-weather"],
      "env": { "OPENWEATHER_KEY": "..." }
    },
    "github": {
      "url": "https://mcp.github.com/",
      "auth": { "type": "oauth" }
    }
  }
}
```

The two entry shapes — `command` for local stdio servers and `url` for remote
HTTP servers — cover essentially every deployment. Where the hosts disagree
is the *path* to the file:

| Host                | Path                                                                                                       |
|---------------------|------------------------------------------------------------------------------------------------------------|
| **Claude Desktop**  | `~/Library/Application Support/Claude/claude_desktop_config.json` (mac), `%APPDATA%\Claude\claude_desktop_config.json` (Windows) |
| **Claude Code**     | `.mcp.json` in project root, or `~/.claude/mcp.json`                                                       |
| **ChatGPT Desktop** | `~/Library/Application Support/OpenAI/ChatGPT/mcp.json` (mac)                                              |
| **VS Code**         | workspace `.vscode/mcp.json` or user settings                                                              |
| **Cursor**          | `~/.cursor/mcp.json`                                                                                       |

💡 If you ship a server, document install as "add this block to your host's
MCP config" and link to all five paths. Users will copy-paste; don't make
them think.

⚠️ **Do not put secrets in `env` blocks shared across teammates.** Workspace
configs (`.mcp.json`, `.vscode/mcp.json`) get committed to git by accident
constantly. Use OAuth or a secret-broker gateway (see §5) for anything you
wouldn't paste in Slack.

---

## 4. Registries

Registries are how users discover servers. Think of them like package
registries (npm, PyPI) — one canonical source, several community catalogs of
varying curation, plus niche curated lists.

| Registry                  | Operator         | Servers (Apr 2026)   | Listing/discovery                    | Notes                                            |
|---------------------------|------------------|----------------------|--------------------------------------|--------------------------------------------------|
| **Official MCP Registry** | AAIF / Anthropic | ~6,400               | API + web + JSON catalog             | The canonical source; hosts pull from here       |
| **MCP.so**                | Community        | Tens of thousands    | Web catalog + API                    | Largest but lower curation                       |
| **Smithery**              | Smithery         | ~Thousands           | Hosted serverless MCP + registry     | Also hosts servers for you                       |
| **MCPMarket / Glama / Pulse** | Various      | Smaller              | Curated lists                        | Niche/quality-focused                            |

### Publishing to the official registry

Include a `server.json` next to your code describing the transport, install
command, and config schema. Submit via the registry's CLI or web flow. A
minimal example:

```json
{
  "name": "io.github.your-org/mcp-weather",
  "version": "1.4.0",
  "description": "Current weather and 7-day forecast via OpenWeatherMap.",
  "transports": ["stdio", "streamable-http"],
  "install": {
    "stdio": { "command": "uvx", "args": ["mcp-weather"] },
    "http":  { "url":  "https://weather.example.com/mcp" }
  },
  "config": {
    "OPENWEATHER_KEY": {
      "type": "string",
      "required": true,
      "secret": true,
      "description": "API key from openweathermap.org"
    }
  }
}
```

Hosts that integrate the registry (Claude Desktop's marketplace, Cursor's
"Browse MCP servers" panel, etc.) read this file to render install UIs. Get
the `config` block right and your server becomes a one-click install
everywhere; get it wrong and users have to hand-edit JSON.

### Discovery patterns hosts actually use

Different hosts surface registry data differently:

- **Claude Desktop** ships a built-in marketplace UI that pulls from the
  official registry; users browse by category and install with one click.
- **Cursor** has a "Browse MCP servers" panel that mixes official + curated
  community sources.
- **VS Code** integrates discovery into the GitHub Copilot Chat config flow.
- **Claude Code** keeps it CLI-driven — `claude mcp add <name>` resolves
  against the official registry by default.

What this means in practice: a single well-formed `server.json` in the
official registry gets you native install UX in every Tier 1 host without
any per-host work.

> **Key insight**: Publish to the official registry first, mirror to MCP.so
> for reach. Skip the curated lists until you have real adoption — they pull
> from the canonical sources anyway.

---

## 5. Gateways

By 2026 the gateway has become the dominant production architecture for any
team running more than a handful of servers. The pattern is familiar from
API gateways: clients connect to one well-known endpoint that fans out to
many backend servers, terminating auth, enforcing policy, and centralizing
observability.

```
   ┌──────────┐       ┌──────────────┐       ┌──────────────┐
   │  Client  │──────>│  MCP Gateway │──────>│  Server A    │
   │  (host)  │       │              │──────>│  Server B    │
   └──────────┘       │ • auth       │──────>│  Server C    │
                      │ • rate limit │       └──────────────┘
                      │ • allow-list │
                      │ • audit log  │
                      │ • secrets    │
                      └──────────────┘
```

| Gateway                     | Type                          | Strengths                                          | Notes                                  |
|-----------------------------|-------------------------------|----------------------------------------------------|----------------------------------------|
| **Kong MCP**                | API gateway w/ MCP module     | Mature gateway features (rate limit, auth, plugins) | Best fit if you already run Kong       |
| **Lunar MCPX**              | MCP-native                    | Built for MCP from day one                         | Strong observability + governance      |
| **Docker MCP Gateway**      | Container-aware               | Tight Docker Desktop integration                   | Local dev → prod path                  |
| **IBM ContextForge**        | Enterprise governance         | Audit, compliance, RBAC, secret brokering          | Aimed at large enterprises             |
| **Microsoft MCP Gateway**   | Azure-native                  | OAuth via Entra ID                                 | Best on Azure                          |
| **MCPJungle**               | OSS                           | Simpler, community-driven                          | Good starting point                    |

### Common responsibilities (regardless of vendor)

Every production-grade gateway in this list handles the same core jobs.
If you're evaluating one, check that all of these are covered:

- **Auth termination + scope check** — verify the client's OAuth token,
  resolve scopes, drop the request before it touches a backend
- **Rate limiting per client / per tool** — protect both the gateway and
  expensive backends (LLM-backed tools, paid APIs)
- **Tool allow/deny lists** — restrict which clients can call which tools,
  often per scope or tenant
- **Audit logging** — every tool call, every argument (with PII scrubbing),
  every result; compliance requires it
- **Secrets brokering** — the gateway holds upstream API keys; backends
  receive short-lived tokens it mints. Users never see the real credentials.
- **Optional: prompt-injection scanning** — run tool arguments and results
  through a scanner before they reach the model

### Choosing among the six

The list above looks similar on paper; in practice the choice usually comes
down to which infrastructure you already operate:

- **Already running Kong, Envoy, or another API gateway?** Use Kong MCP. The
  operational story (deploys, dashboards, on-call runbooks) is identical to
  what you already have.
- **No existing gateway, want best-in-class MCP-specific features?** Lunar
  MCPX. The observability and policy primitives are built around MCP
  semantics, not bolted on.
- **Heavy Docker Desktop usage in dev?** Docker MCP Gateway. The dev-to-prod
  story is the cleanest because the same gateway runs locally and remotely.
- **Regulated industry — finance, healthcare, government?** IBM ContextForge.
  The compliance features (SOC 2, HIPAA, audit retention) are productized.
- **All-in on Azure?** Microsoft MCP Gateway with Entra ID. OAuth flows
  integrate with existing tenant policies for free.
- **Just want to start?** MCPJungle. OSS, simple, easy to outgrow gracefully.

⚠️ The single biggest mistake teams make is exposing servers directly when
they have more than two of them. Without a gateway you end up rebuilding
auth, rate limiting, and audit in every server — badly, and inconsistently.

---

## 6. Hosting Platforms

Recap from [`../production/06_deployment.md`](../production/06_deployment.md) —
where the server actually runs:

| Platform                       | Model                          | Best for                                             |
|--------------------------------|--------------------------------|------------------------------------------------------|
| **Cloud Run / App Runner**     | Stateless, per-request billing | Most HTTP servers; scales to zero                    |
| **Cloudflare Workers**         | Global edge, 30s CPU (TS only) | Latency-sensitive read-only tools                    |
| **Cloudflare Containers**      | Stateful with Durable Objects  | Long-running sessions with edge benefits             |
| **Fly.io**                     | Stateful, per-region VMs       | Sticky sessions, regional pinning                    |
| **AWS Lambda**                 | Pure RPC tools, no SSE         | Burst workloads where streaming isn't needed         |
| **Kubernetes**                 | Full control                   | Existing k8s shop; complex multi-service deployments |
| **Smithery / Mintlify**        | Managed MCP hosting            | Don't want to run infra; trade control for speed     |

The platform choice mostly falls out of two questions: do you need state
between requests (sticky sessions, Durable Objects, or Fly), and do you need
streaming/SSE (rules out plain Lambda).

---

## 7. Governance

The December 2025 donation to the Linux Foundation — under the new **Agentic
AI Foundation (AAIF)** — was the biggest structural change in the protocol's
short history. It moved the spec from "Anthropic publishes, others adopt" to
a foundation-led process with public proposals and multi-vendor steering.

### The SEP process

**SEP** (Spec Enhancement Proposal) is the mechanism for changing the spec.
Proposals are public, numbered, and discussed in the open. Recent ones worth
knowing:

| SEP        | Topic                          |
|------------|--------------------------------|
| **SEP-1303** | Validation errors            |
| **SEP-1330** | Enum elicitation             |
| **SEP-1613** | JSON Schema 2020-12          |
| **SEP-991**  | Client ID Metadata           |
| **SEP-973**  | Icons                        |
| **SEP-1686** | Tasks                        |
| **SEP-932**  | Governance                   |
| **SEP-1730** | SDK tiering                  |

If you're trying to understand why a feature is shaped the way it is, the SEP
is almost always the best starting point — far more useful than the spec
text itself, which is the *outcome* of the discussion, not the rationale.

### SDK tiering (SEP-1730)

The foundation now formally tiers SDKs by maintenance level:

- **Tier 1**: **Python (FastMCP)** and **TypeScript** — maintained,
  feature-complete, in lockstep with the spec. New spec features land here
  first, often before the spec is officially published.
- **Tier 2**: **Go, C#, Java, Kotlin, Rust, Swift** — official but lag the
  spec by one or more revisions. Suitable for production if you don't need
  the very latest features.
- **Tier 3**: **Community SDKs** — varying quality. Read the source before
  depending on one for anything important.

> **Rule**: If you're picking a language for a new server today and have a
> free choice, pick Python or TypeScript. Tier 2 is fine, but you will
> occasionally wait for features. Tier 3 is a research project disguised as
> a dependency.

---

## 8. Notable Real-World Deployments

From talks at the **MCP Dev Summit NA 2026** — useful as a sanity check that
the architecture patterns above actually scale:

- **Uber**: ~1,500 monthly active agents through their internal gateway +
  registry. The gateway pattern from §5, applied at scale, with a private
  registry mirror feeding the gateway's allow-list.
- **Anthropic**: Claude Desktop and Claude Code; powers tool use across the
  product line. Eats their own dog food at every layer.
- **Cursor**: Tens of thousands of users connecting to community MCP servers.
  Their config UI is essentially a registry browser.
- **Microsoft**: VS Code adopted as a first-class MCP host. Drove a lot of
  the Tier 2 SDK quality improvements (C#, in particular).

The pattern across all four: client (host) talks to a gateway (or a managed
registry that acts gateway-like), which fans out to many small servers. None
of them connect clients directly to backend servers in production.

A second pattern worth noting from the same talks: the successful internal
deployments treat MCP servers as **disposable adapters**, not as
applications. Each server wraps one upstream system (a database, an internal
API, a SaaS tool) and stays small enough that rewriting it from scratch is
cheaper than refactoring it. Uber's ~1,500 active agents are powered by
hundreds of small servers, not a handful of large ones. The gateway is what
makes that fan-out tractable.

---

## 9. Picking Your Stack — Decision Framework

Use the table below as a starting point. The "right" stack scales with the
size of the audience and the compliance requirements, not the technical
ambition of the server.

| Situation                                       | Transport         | Auth              | Gateway            | Registry                          |
|-------------------------------------------------|-------------------|-------------------|--------------------|-----------------------------------|
| **Solo dev / small team, single server**        | stdio             | None / API key    | None               | Official + PyPI/npm               |
| **Single team behind one product**              | HTTP              | Simple OAuth      | Basic LB only      | Official                          |
| **Platform team for an organization**           | HTTP              | OAuth + scopes    | Kong / Lunar / MS  | Official + private mirror         |
| **Enterprise compliance**                       | HTTP              | Federated SSO     | ContextForge       | Private registry + audit pipeline |

A few notes on how to read this:

- **Don't pre-build for a tier you're not in.** A solo project with a
  ContextForge gateway in front of it is cosplay. Add the gateway when you
  have at least two servers and at least one team that isn't yours
  consuming them.
- **Pick a gateway that matches your existing infra.** Already on Kong?
  Kong MCP. On Azure? Microsoft. On Docker Desktop with no other infra?
  Docker MCP Gateway. The MCP-specific features matter less than the
  operational fit.
- **The official registry is non-negotiable for public servers.** Even if you
  also publish to MCP.so, the official entry is what hosts trust by default.

### A walk through one path

To make this concrete, here's what the platform-team row typically looks
like in production:

```
       internet
          │
          ▼
  ┌───────────────┐    ┌────────────────┐
  │  Auth (OIDC)  │───>│  MCP Gateway   │
  │  IdP / Entra  │    │  (Kong / MCPX) │
  └───────────────┘    └───────┬────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌─────────┐      ┌─────────┐      ┌─────────┐
        │ Server1 │      │ Server2 │      │ Server3 │
        │ (FastMCP)│     │ (TS SDK)│      │ (Go SDK)│
        └────┬────┘      └────┬────┘      └────┬────┘
             │                │                │
             └────────────────┴────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Private registry │
                    │ (mirror of       │
                    │  official)       │
                    └──────────────────┘
```

Auth is centralized at the IdP, the gateway handles policy and audit, the
servers themselves are minimal handlers, and the registry is private but
mirrors the official catalog so internal users get the same discovery
experience as the public one.

---

## 10. The Honest Current State

The ecosystem looks settled from a thousand feet, but a few rough edges are
worth flagging before you commit to anything:

- **The spec moves quickly.** Some hosts lag a release behind. Don't assume
  a spec feature works on every host until you've tested it. The Tier 1 SDKs
  give you the spec on day one; the host catches up later.
- **Community servers vary wildly in quality.** Treat any third-party server
  as hostile until you've reviewed it. A malicious server can return crafted
  text to hijack the model, exfiltrate tool outputs, or chain into other
  servers. See [`../production/05_security.md`](../production/05_security.md)
  for the threat model.
- **"MCP-compatible" doesn't mean "supports every primitive."** Confirm
  transport, sampling, elicitation, and auth before depending on them.
  Plenty of hosts say "MCP-compatible" but only implement tools — not
  prompts, not resources, not sampling.
- **Streamable HTTP support is universal among Tier 1 SDKs but spotty in
  Tier 3.** If your server depends on a Tier 3 SDK, test the HTTP path
  explicitly; don't assume parity with stdio.
- **Gateways are still differentiating.** The category is two years old.
  Expect consolidation, acquisitions, and at least one of the names in §5 to
  pivot or shut down within the next eighteen months. Pick one, but don't
  build deep custom integrations — keep the policy layer portable.

> **Key insight**: The protocol is settled enough to bet on. The
> infrastructure around it is not. Choose your server architecture so that
> swapping the gateway, registry, or hosting platform underneath it is a
> weekend's work, not a quarter's.

---

**Next**: [02_python_vs_typescript.md — Python vs TypeScript Comparison](02_python_vs_typescript.md)
