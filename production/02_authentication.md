# MCP Authentication: OAuth 2.1 + PKCE + RFC 9728

> **Who this is for**: Engineers about to expose an MCP server beyond `localhost`. Assumes
> familiarity with OAuth 2.0 fundamentals (authorization code flow, JWTs, JWKS) and that
> you've read [01_streamable_http.md](01_streamable_http.md) — auth sits on top of the
> Streamable HTTP transport.

---

## 1. The Mental Model

The MCP spec settled on a clean separation of concerns: an **MCP server is an OAuth 2.1
Resource Server**, full stop. It validates incoming bearer tokens. It does not issue them,
it does not run a login UI, it does not store passwords.

The **Authorization Server (AS)** is decoupled — and that's intentional. It can be Auth0,
Okta, Keycloak, AWS Cognito, Entra ID, your own Hydra deployment, or anything else that
speaks OAuth 2.1. The MCP server only needs three things from it:

1. A way to discover its endpoints (RFC 8414 metadata).
2. A way to fetch its public signing keys (JWKS).
3. Tokens with an `aud` claim naming this MCP server.

```
┌─────────────────┐     ┌────────────────────┐     ┌───────────────────┐
│   MCP Client    │     │    MCP Server      │     │  Auth Server      │
│ (Claude, IDE,   │     │ (Resource Server,  │     │ (Auth0 / Keycloak │
│  Cursor, …)     │     │  validates tokens) │     │  / Okta / …)      │
└─────────────────┘     └────────────────────┘     └───────────────────┘
        │                        │                          │
        │  bearer token in       │                          │
        │  Authorization header  │                          │
        ├───────────────────────>│                          │
        │                        │                          │
        │                        │  fetch JWKS, verify sig, │
        │                        │  check aud + iss + exp   │
        │                        │                          │
        │                        │ (no per-request call to AS)
```

> **Key insight**: The MCP server should **never** call the AS on the hot path. Token
> verification is a local crypto operation against cached JWKS. If you find yourself making
> an HTTP call to validate every request, you've built it wrong — that's introspection
> (RFC 7662), which OAuth 2.1 still permits but which you should avoid for latency reasons.

---

## 2. What the Spec Mandates (`2025-11-25`)

The current MCP authorization spec is opinionated. These are not suggestions:

| Requirement | What it means |
|-------------|---------------|
| **OAuth 2.1** | No implicit flow. No resource owner password credentials grant. Both were retired for security reasons. |
| **PKCE for all clients** | Even confidential clients. Clients **must verify** the AS supports PKCE (via `code_challenge_methods_supported` in metadata) before initiating the flow. |
| **RFC 9728 PRM** | MCP server publishes Protected Resource Metadata at `/.well-known/oauth-protected-resource`. |
| **WWW-Authenticate on 401** | Must include `Bearer realm="..."` and `resource_metadata="<URL>"` so clients can bootstrap discovery from a single 401. |
| **Audience-restricted tokens** | `aud` claim must name the MCP server. Tokens without an `aud` (or with the wrong one) must be rejected. |

Recommended but not strictly required:

- **OpenID Connect Discovery 1.0** alongside RFC 8414 — most ASs publish both anyway.
- **OAuth Client ID Metadata Documents** (SEP-991) — replaces the heavyweight Dynamic
  Client Registration dance for ecosystems with many clients.
- **Refresh token rotation** — short-lived access tokens (5–15 min) plus rotating refresh
  tokens. Standard hygiene.

> **Rule**: If your server returns 401 without a `WWW-Authenticate` header naming the
> resource metadata URL, every spec-conformant client breaks. This is the single most
> common mistake when adapting an existing OAuth-protected API to MCP.

---

## 3. End-to-End Flow

Here's the full handshake from a cold-start client through the first authenticated
request. Assume the client has no prior knowledge of the AS.

```
Client          MCP Server          Auth Server
  │   POST /mcp/    │                    │
  │ ───────────────>│                    │
  │ 401 + WWW-Auth  │                    │
  │ <───────────────│                    │
  │  GET /.well-known/oauth-protected-resource
  │ ───────────────>│                    │
  │ { auth_servers: [...] }              │
  │ <───────────────│                    │
  │  GET /.well-known/oauth-authorization-server
  │ ──────────────────────────────────────>│
  │ { authorization_endpoint, token_endpoint, ... }
  │ <──────────────────────────────────────│
  │  PKCE: open browser → user authenticates
  │ ──────────────────────────────────────>│
  │  Auth code returned to client redirect
  │ <──────────────────────────────────────│
  │  POST /token (code + verifier)
  │ ──────────────────────────────────────>│
  │  Access token (JWT)                   │
  │ <──────────────────────────────────────│
  │  POST /mcp/  Authorization: Bearer ...│
  │ ───────────────>│                    │
  │  200 OK         │                    │
  │ <───────────────│                    │
```

Notice what's **not** in this diagram: the MCP server never talks to the Auth Server during
a request. Discovery happens once (and is cached). Verification is local. The AS is
essentially write-once-read-never from the MCP server's perspective.

> **Key insight**: A well-implemented MCP client treats the first 401 as bootstrap fuel, not
> an error. The 401 + `WWW-Authenticate` header is the entire handshake initiator — no
> out-of-band configuration needed.

---

## 4. The Two Well-Known Documents

Discovery is a two-hop chain: client hits the MCP server's PRM to learn which AS to use,
then hits the AS's metadata to learn the endpoints.

### `/.well-known/oauth-protected-resource` (RFC 9728)

Published by the **MCP server**. Tells clients which AS issues tokens for this resource:

```json
{
  "resource": "https://api.example.com",
  "authorization_servers": ["https://auth.example.com"],
  "scopes_supported": ["mcp.read", "mcp.write"],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "https://docs.example.com/mcp"
}
```

The `resource` value is what clients pass as the `resource` parameter in the auth request
(RFC 8707) — and what must appear in the token's `aud` claim. Mismatch → reject.

### `/.well-known/oauth-authorization-server` (RFC 8414)

Published by the **Auth Server**. Tells clients where to send users and how to exchange
codes for tokens:

```json
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/oauth/authorize",
  "token_endpoint": "https://auth.example.com/oauth/token",
  "registration_endpoint": "https://auth.example.com/oauth/register",
  "code_challenge_methods_supported": ["S256"],
  "grant_types_supported": ["authorization_code", "refresh_token"]
}
```

⚠️ The client **must** check `code_challenge_methods_supported` includes `S256` before
proceeding. If the AS only advertises `plain`, abort — that's not OAuth 2.1 compliant.

---

## 5. PKCE in 30 Seconds

PKCE (Proof Key for Code Exchange, RFC 7636) closes the auth-code interception window.
Mandatory for all clients in OAuth 2.1. The math:

```
1. Client generates code_verifier   = random 43–128 char string
2. Client computes code_challenge   = BASE64URL(SHA256(code_verifier))
3. Authorization request carries:
     code_challenge=<challenge>
     code_challenge_method=S256
4. Token exchange carries:
     code_verifier=<original verifier>
5. AS verifies SHA256(verifier) == challenge
   → if yes: issue token
   → if no:  reject (someone else intercepted the code)
```

Why it works: an attacker who steals the auth code from the redirect URI can't redeem it
without the verifier, which never left the original client. The challenge-in-request /
verifier-at-redemption pattern means a one-shot leak of the code is useless.

❌ **Never use `code_challenge_method=plain`** — it sends the verifier itself as the
challenge, defeating the purpose. Spec violation.

---

## 6. Python (FastMCP) — Token Verification

FastMCP ships an `OAuthResourceServerAuth` adapter that wires everything: it serves the
`.well-known/oauth-protected-resource` document, returns proper 401s with
`WWW-Authenticate` headers, and delegates token validation to a verifier you supply.

```python
from fastmcp import FastMCP
from fastmcp.server.auth import OAuthResourceServerAuth, BearerToken
import jwt

class JwtVerifier:
    """Verifies JWTs against a remote JWKS. Caches keys; no per-request AS call."""

    def __init__(self, jwks_url: str, audience: str, issuer: str):
        # PyJWKClient handles JWKS fetching, caching, and key rotation via `kid`
        self._jwks = jwt.PyJWKClient(jwks_url)
        self._audience = audience
        self._issuer = issuer

    async def verify(self, token: str) -> BearerToken:
        # Selects the right key from the JWKS based on the token's `kid` header
        signing_key = self._jwks.get_signing_key_from_jwt(token)
        claims = jwt.decode(
            token,
            signing_key.key,
            algorithms=["RS256"],         # never trust `alg` from the token header
            audience=self._audience,      # rejects tokens minted for other resources
            issuer=self._issuer,          # rejects tokens from other ASs
        )
        return BearerToken(
            subject=claims["sub"],
            scopes=claims.get("scope", "").split(),
            client_id=claims.get("client_id"),
            extra=claims,                 # full claim set available downstream
        )

mcp = FastMCP(
    "secure",
    auth=OAuthResourceServerAuth(
        verifier=JwtVerifier(
            jwks_url="https://auth.example.com/.well-known/jwks.json",
            audience="https://api.example.com",
            issuer="https://auth.example.com",
        ),
        # These three feed the .well-known/oauth-protected-resource document
        resource="https://api.example.com",
        authorization_servers=["https://auth.example.com"],
        scopes_supported=["mcp.read", "mcp.write"],
    ),
)
```

That's the whole server-side bootstrap. FastMCP handles:

- Serving `/.well-known/oauth-protected-resource` with the metadata above.
- Returning `401 Unauthorized` with the correct `WWW-Authenticate` header on missing or
  invalid tokens.
- Calling `verifier.verify()` for every request and rejecting with 401 on any exception.
- Surfacing the `BearerToken` to tools via `Context`.

> **Rule**: Pin the `algorithms` list explicitly. Never trust the `alg` field in the token
> header — `alg=none` and the `RS256→HS256` key confusion attack are infamous JWT
> footguns.

---

## 7. Python — Accessing the User Inside a Tool

Once a request is authenticated, the `BearerToken` lives on `Context.auth_info`. Tools
can read it for per-user logic, and FastMCP can enforce scopes declaratively.

```python
from fastmcp import Context

@mcp.tool
async def whoami(ctx: Context) -> dict:
    """Returns the authenticated subject — useful for client debugging."""
    auth = ctx.auth_info  # BearerToken from the verifier above
    return {"sub": auth.subject, "scopes": auth.scopes}

@mcp.tool(scopes=["mcp.write"])
async def delete_record(id: str, ctx: Context) -> dict:
    """Requires `mcp.write` scope; FastMCP enforces and 403s otherwise."""
    # If we got here, the token had mcp.write — no manual check needed
    ...
```

The scope check happens **before** your tool body executes. A token with only `mcp.read`
hitting `delete_record` gets a `403 Forbidden` automatically — you never see the call.

💡 For finer-grained authorization (per-row, per-tenant), do that check inside the tool
using `auth.subject` or `auth.extra` (custom claims like `tenant_id`, `org`, etc.).

---

## 8. TypeScript — `mcpAuthRouter` + `requireBearerAuth`

The TS SDK's auth surface is split into two pieces: `mcpAuthRouter` mounts all the
discovery endpoints, and `requireBearerAuth` is Express middleware that validates tokens
on protected routes.

```typescript
import { mcpAuthRouter } from "@modelcontextprotocol/sdk/server/auth/router.js";
import { requireBearerAuth } from "@modelcontextprotocol/sdk/server/auth/middleware/bearerAuth.js";
import {
  OAuthRegisteredClientsStore,
  OAuthServerProvider,
} from "@modelcontextprotocol/sdk/server/auth/provider.js";

const provider: OAuthServerProvider = {
  endpoints: {
    authorizationUrl: "https://auth.example.com/oauth/authorize",
    tokenUrl: "https://auth.example.com/oauth/token",
  },
  clientsStore: {
    /* Dynamic Client Registration / lookup — see §9 */
  },
  verifyAccessToken: async (token) => {
    // verifyJwt: your wrapper around jose.jwtVerify with JWKS caching
    const claims = await verifyJwt(token);
    return {
      token,
      clientId: claims.client_id,
      scopes: (claims.scope ?? "").split(" "),
      expiresAt: claims.exp,
    };
  },
};

// Mounts: /.well-known/oauth-protected-resource,
//         /.well-known/oauth-authorization-server (proxy),
//         /register (if DCR enabled), etc.
app.use(
  mcpAuthRouter({
    provider,
    issuerUrl: new URL("https://auth.example.com"),
    baseUrl: new URL("https://api.example.com"),
  }),
);

// Per-route enforcement — note `requiredScopes` is checked here, not in the verifier
app.post(
  "/mcp",
  requireBearerAuth({ verifier: provider, requiredScopes: ["mcp.read"] }),
  mcpHandler,
);
```

The `provider` interface is the single integration point. Implement `verifyAccessToken`
against your AS's JWKS, plug in a `clientsStore` if you support DCR, and the SDK handles
the rest of the spec compliance.

> **Key insight**: `mcpAuthRouter` is the TS equivalent of FastMCP's
> `OAuthResourceServerAuth` — it owns all the boilerplate so your handlers stay focused on
> business logic. Don't roll your own discovery endpoints; the SDK already gets the headers
> and edge cases right.

---

## 9. Dynamic Client Registration and SEP-991

Dynamic Client Registration (RFC 7591) lets a client POST its metadata to the AS at
runtime and get back a `client_id`. The AS persists the registration. This worked fine
when there were a handful of OAuth clients per ecosystem, but it doesn't scale to "every
IDE, every desktop app, every browser extension wants to talk to my MCP server."

Specifically, RFC 7591 has two pain points:

1. The AS becomes the bottleneck — every new client install is a write.
2. There's no way to revoke or update a registration without out-of-band coordination.

**SEP-991: OAuth Client ID Metadata Documents** flips this around. Instead of registering,
the client publishes a metadata document at a URL it owns:

```
https://claude.ai/.well-known/oauth-client-metadata
{
  "client_name": "Claude Desktop",
  "redirect_uris": ["claude://oauth/callback"],
  "scope": "mcp.read mcp.write",
  "token_endpoint_auth_method": "none"
}
```

The AS, on receiving an auth request with `client_id=https://claude.ai/.well-known/oauth-client-metadata`,
fetches the document, validates it, and trusts it for that flow. Think of it as **JWKS for
clients**: the client owns the source of truth, the AS just verifies and caches.

| Approach | Where state lives | Updates require |
|----------|-------------------|------------------|
| RFC 7591 DCR | AS database | API call to AS |
| SEP-991 CIMD | Client's well-known URL | Edit the JSON file |

Recommended for any MCP server expecting many client implementations (Claude Desktop,
ChatGPT, Cursor, custom IDEs). For a closed ecosystem with 2–3 known clients, plain
pre-registration is still simpler.

---

## 10. Brokered Patterns for Enterprise

Putting every client through the user-facing OAuth flow against your internal IdP gets
ugly fast in enterprise contexts:

- Each MCP server needs to be a registered client of the IdP.
- Long-lived refresh tokens proliferate across user devices.
- Audit and revocation are scattered.
- Per-tool scope mapping bleeds into the IdP's scope catalog.

The pattern that's emerging: run a **gateway** in front of your MCP servers that owns the
upstream credentials and exchanges client tokens for short-lived service tokens.

```
┌──────────┐   user OAuth   ┌──────────┐   service token   ┌─────────────┐
│  Client  │ ─────────────> │ Gateway  │ ────────────────> │ MCP Server  │
└──────────┘                │ (Kong /  │                   │ (validates  │
                            │  MCPX /  │                   │  gateway-   │
                            │  Lunar / │                   │  issued JWT)│
                            │  Context-│                   └─────────────┘
                            │  Forge)  │
                            └──────────┘
                                 │
                                 │  audit log, rate limit,
                                 │  scope mapping, key rotation
                                 ▼
                            ┌──────────┐
                            │   SIEM   │
                            └──────────┘
```

Benefits:

- **Reduced credential surface**: upstream API keys live in one place, not on every MCP
  server.
- **Centralized audit**: one log stream for "who called what tool, when, on whose behalf."
- **Token translation**: present external-facing OAuth scopes; map to internal RBAC at
  the gateway.
- **Independent rotation**: rotate the upstream credential without touching client
  configuration.

The trade-off is the gateway becomes a critical-path component. If it goes down, every
MCP server behind it is unreachable. Standard reverse-proxy HA patterns apply.

See [05_security.md](05_security.md) for the full gateway threat model — including
confused-deputy risks and what claims the gateway must add to outbound tokens.

---

## 11. Failure Modes

Things that break in production, ranked by how often they happen.

❌ **Returning 401 without `WWW-Authenticate`**
Spec-conformant clients use the header to discover the resource metadata URL. Without it,
clients can't bootstrap and have to be manually configured. Always include:

```
WWW-Authenticate: Bearer realm="mcp",
                  resource_metadata="https://api.example.com/.well-known/oauth-protected-resource"
```

❌ **Skipping audience validation on the JWT**
A token minted for `https://other-api.example.com` will pass signature verification (same
AS, same key) but was never meant for your server. Token confusion attack. Always check
`aud` matches your `resource` value exactly.

❌ **Allowing the implicit flow or password grant**
Both removed in OAuth 2.1. If your AS still has them enabled for legacy reasons, either
disable them or front it with a gateway that strips those grant types.

❌ **PKCE with `code_challenge_method=plain`**
Sends the verifier as the challenge, which is exactly what PKCE is meant to prevent. Spec
violation. Reject `plain` even if your AS offers it.

⚠️ **Hosting the AS on a different origin without proper CORS for `.well-known`**
Browser-based clients (web IDEs, ChatGPT-style hosts) need CORS on the discovery
endpoints. A common silent-failure mode: discovery works in `curl`, fails in the browser
with no useful console error because the preflight returns 404.

❌ **Not honoring `kid` / not rotating signing keys**
The `kid` header tells you which key from the JWKS to use. If you cache a single key and
the AS rotates, all new tokens fail. If you don't refresh the JWKS, expired keys live
forever. PyJWKClient and the `jose` library handle this — use them, don't roll your own.

⚠️ **Long-lived access tokens**
A 24-hour access token that gets stolen is a 24-hour breach. Aim for 5–15 minute access
tokens with rotating refresh tokens. The cost of refresh-on-expiry is negligible; the
blast radius reduction is not.

❌ **Trusting client-provided scope claims without verification**
The `scope` claim must come from the AS, never from the client. If your verifier accepts
unsigned token portions or you're parsing scopes from headers, that's a privilege
escalation primitive.

⚠️ **Forgetting to scope tokens to your resource (RFC 8707)**
Without `resource=https://api.example.com` in the auth request, the AS may issue a token
without the right `aud`. Some ASs default to a generic audience. Always pass `resource`
explicitly and validate `aud` strictly.

---

## 12. Quick Reference

| Task | Where it happens |
|------|------------------|
| Issue tokens | Authorization Server (Auth0, Keycloak, …) |
| Validate tokens | MCP Server (local crypto via JWKS) |
| Publish AS endpoints | `/.well-known/oauth-authorization-server` (on AS) |
| Publish resource metadata | `/.well-known/oauth-protected-resource` (on MCP server) |
| User login UI | Authorization Server |
| Per-tool scope enforcement | MCP Server (via SDK middleware) |
| Audit logging | MCP server + (optional) gateway |

| Spec | What it covers |
|------|----------------|
| OAuth 2.1 | Baseline auth flows; PKCE mandatory |
| RFC 7636 | PKCE mechanics |
| RFC 8414 | Auth Server metadata (`oauth-authorization-server`) |
| RFC 9728 | Resource Server metadata (`oauth-protected-resource`) |
| RFC 8707 | Resource indicators (`resource` parameter, audience binding) |
| RFC 7591 | Dynamic Client Registration (legacy) |
| SEP-991 | Client ID Metadata Documents (modern) |

---

## 13. Cross-References

- **Previous**: [01_streamable_http.md — Streamable HTTP Transport](01_streamable_http.md)
- **Next**: [03_scaling.md — Scaling: Stateful vs Stateless](03_scaling.md)
- **Related**:
  - [05_security.md — Security and Threat Model](05_security.md) — gateway patterns,
    confused deputy, prompt injection
  - [../python/06_middleware.md](../python/06_middleware.md) — building custom auth
    middleware on top of FastMCP
  - [../typescript/05_advanced.md](../typescript/05_advanced.md) — advanced
    `mcpAuthRouter` patterns and DCR implementation

---

**Next**: [03_scaling.md — Scaling: Stateful vs Stateless](03_scaling.md)
