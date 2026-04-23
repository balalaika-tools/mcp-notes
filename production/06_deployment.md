# Deploying MCP Servers

> **Who this is for**: Engineers and platform teams shipping an MCP server to real users.
> Assumes you've read [05_security.md](05_security.md) and have a working server locally.
> If you haven't yet thought through capacity and back-pressure, also see
> [03_scaling.md](03_scaling.md).

---

## 1. Two Distribution Shapes

Every MCP server is distributed in one of two shapes — sometimes both. The choice
shapes the rest of the deployment plan.

| Shape           | What ships                          | Where it runs                    | Who owns uptime |
|-----------------|-------------------------------------|----------------------------------|-----------------|
| **stdio**       | Code/binary (npm, PyPI, container)  | The user's machine, as a subprocess of the host | The user |
| **Remote HTTP** | A running service (URL)             | Your infrastructure              | You |

**stdio servers** are launched by the host (Claude Desktop, Cursor, VS Code, etc.) over
stdin/stdout. The user installs them locally — usually via a package manager. Distribution
looks like a CLI tool: publish to a registry, ship a `--help` flag, document the install
command. There is no service to monitor.

**Remote HTTP servers** are deployed as a long-running service implementing the
Streamable HTTP transport (see [01_streamable_http.md](01_streamable_http.md)). The user
configures their host with a URL and bearer token. You own the deploy, the scale, the
TLS, and the on-call rotation.

> **Key insight**: the same codebase can usually serve both. Branch on a `--http` flag
> or an env var at startup. Keep the tool implementations transport-agnostic and let the
> entry point pick the transport.

```python
# myserver/__main__.py
import argparse, os
from .server import build_server

def cli_main() -> None:
    ap = argparse.ArgumentParser(prog="mcp-weather")
    ap.add_argument("--http", action="store_true",
                    help="Run as HTTP server instead of stdio")
    ap.add_argument("--host", default="127.0.0.1")
    ap.add_argument("--port", type=int, default=8080)
    args = ap.parse_args()

    server = build_server()
    if args.http or os.getenv("MCP_TRANSPORT") == "http":
        server.run_http(host=args.host, port=args.port)
    else:
        server.run_stdio()
```

---

## 2. Containerizing a Python (FastMCP) Server

A production-ready Dockerfile for a FastMCP server. Multi-stage isn't strictly
necessary for Python (no compilation artifact to drop), but pinning, non-root, and an
explicit healthcheck are non-negotiable.

```dockerfile
FROM python:3.12-slim AS base
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev

COPY . .

# Non-root for sandbox depth
RUN useradd --no-create-home --shell /bin/false mcp && chown -R mcp:mcp /app
USER mcp

EXPOSE 8080
CMD ["uv", "run", "python", "-m", "myserver", "--http", "--host", "0.0.0.0", "--port", "8080"]

HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -fsSL http://127.0.0.1:8080/healthz || exit 1
```

Key choices, and why:

- **Pin the lock file**: `uv.lock` (or `poetry.lock`, `requirements.txt` with hashes)
  must be committed. `--frozen` refuses to resolve and fails if the lock is stale —
  exactly what you want in CI.
- **`--no-dev`**: keep pytest, mypy, ruff, etc. out of the runtime image. Smaller
  surface, smaller CVE footprint.
- **Non-root**: a tool-execution server is a high-value target. If anything inside the
  process does shell-out (subprocess, os.system), running as root means a CVE in your
  parser becomes root-on-the-container.
- **`PYTHONUNBUFFERED=1`**: without this, stdout is block-buffered and your logs
  appear minutes late — or never, if the process is killed. Always set it.
- **`HEALTHCHECK`**: Docker uses this for `docker ps` status and orchestrators use it
  for restart decisions. Kubernetes overrides it with its own probes (see §5).

---

## 3. Containerizing a TypeScript Server

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build   # tsc → dist/

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json /app/package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
USER node
EXPOSE 8080
CMD ["node", "dist/server.js"]
HEALTHCHECK --interval=10s --timeout=3s CMD wget -q -O- http://127.0.0.1:8080/healthz || exit 1
```

The multi-stage build matters more here. The `build` stage has TypeScript, type
declarations, and dev tooling; the `runtime` stage has only the compiled JS plus
production deps. Final images typically land at 100–150 MB instead of 400+ MB.

- **`npm ci` not `npm install`**: `ci` requires a clean lock file and refuses to
  modify it. Reproducible builds.
- **`--omit=dev`**: drops `@types/*`, test runners, linters.
- **`USER node`**: the official `node` image ships a `node` user (UID 1000) ready to
  use. Don't reinvent it.

> **Rule**: never `apk add` build tools in the runtime stage. If you need them
> (e.g., `node-gyp`), do it in the build stage and copy artifacts forward.

---

## 4. Health Checks

Implement two endpoints. They answer different questions and feed different
controllers.

| Endpoint    | Question                          | Failure means              | Controller |
|-------------|-----------------------------------|----------------------------|------------|
| `/healthz`  | Is the process alive?             | Restart the container      | Liveness probe |
| `/ready`    | Are dependencies reachable?       | Stop sending traffic       | Readiness probe |

```python
from fastapi import APIRouter
from fastapi.responses import JSONResponse

api = APIRouter()

@api.get("/healthz")
async def healthz() -> dict[str, bool]:
    # Liveness: only fails if the process itself is broken (deadlocked event
    # loop, heap exhausted, etc.). Don't check dependencies here — a flaky DB
    # would cause a restart loop that doesn't fix anything.
    return {"ok": True}

@api.get("/ready")
async def ready() -> JSONResponse:
    # Readiness: dependencies. If they're unreachable, take this pod out of
    # the load balancer rotation until they recover.
    try:
        async with state.db.acquire() as conn:
            await conn.fetchval("SELECT 1")
        return JSONResponse({"ok": True})
    except Exception:
        return JSONResponse({"ok": False}, status_code=503)
```

The distinction is load-bearing on Kubernetes: a readiness failure stops new traffic
without killing the pod, giving the dependency time to recover. A liveness failure
restarts the container — appropriate for "process wedged," catastrophic for "DB had a
30-second blip."

⚠️ A common mistake is wiring `/healthz` to check the database. If the DB has a brief
outage, every pod restarts simultaneously, the connection pool resets cold, and
recovery takes much longer than it should.

---

## 5. Kubernetes — Full Manifest Pattern

Deployment + Service + HPA. This is the baseline for any serious remote MCP
deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: weather-mcp }
spec:
  replicas: 3
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  template:
    spec:
      terminationGracePeriodSeconds: 600   # let SSE streams drain
      containers:
        - name: server
          image: ghcr.io/example/weather-mcp:1.4.2
          ports: [{ containerPort: 8080 }]
          env:
            - name: DATABASE_URL
              valueFrom: { secretKeyRef: { name: db, key: url } }
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: 1,    memory: 512Mi }
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 30
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 30"] }   # allow LB deregistration
---
apiVersion: v1
kind: Service
metadata: { name: weather-mcp }
spec:
  selector: { app: weather-mcp }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: weather-mcp }
spec:
  scaleTargetRef: { kind: Deployment, name: weather-mcp, apiVersion: apps/v1 }
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
```

Things worth highlighting:

- **`terminationGracePeriodSeconds: 600`** — SSE streams hold connections open for
  minutes. The default 30 s grace period kills in-flight tool calls. Set this to
  longer than your longest expected tool latency.
- **`maxUnavailable: 0`** — never reduce capacity during a deploy. Surge first,
  drain second.
- **`preStop: sleep 30`** — gives the load balancer time to notice the pod is
  Terminating and stop routing new connections to it before SIGTERM lands. Without
  this, every rolling deploy drops a handful of in-flight requests.
- **Readiness `periodSeconds: 5`, liveness `periodSeconds: 30`** — readiness needs
  to react fast (traffic decisions); liveness shouldn't be twitchy (restart
  decisions).
- **HPA `minReplicas: 3`** — survive a node drain without dropping below quorum.

For **stateful servers** (long-lived sessions with `Mcp-Session-Id`), the basic
Service load balancer round-robins requests, which breaks affinity. Layer on either:

- **Istio DestinationRule** with `consistentHash: { httpHeaderName: Mcp-Session-Id }`
- **NGINX Ingress** with `nginx.ingress.kubernetes.io/upstream-hash-by:
  "$http_mcp_session_id"`
- **AWS ALB** with sticky sessions enabled (cookie-based, less ideal)

See [03_scaling.md](03_scaling.md) for the full session-routing discussion.

---

## 6. Hosting Platforms — Selection Guide

| Platform | Best for | Caveats |
|----------|----------|---------|
| **Cloud Run / App Runner** | Stateless HTTP servers; per-request billing | No long-lived SSE — cap at the platform's request timeout |
| **Cloudflare Workers** | Stateless, low-latency, global; using the `agents` framework | 30s CPU limit; no native FastMCP — TS only |
| **Cloudflare Containers** | Stateful + persistent SSE; Durable Objects for session affinity | Newer platform; pricing model differs |
| **Fly.io** | Stateful; per-region VMs; sticky-by-machine-id | Some operational hand-rolling |
| **AWS Lambda** | Pure stateless RPC tools | No SSE; cold starts hurt latency-sensitive tools |
| **Kubernetes (EKS/GKE/AKS)** | Anything; full control | You own all the operational complexity |
| **Smithery / Mintlify / Anthropic-hosted MCP** | Just-publish-a-server SaaS | Vendor lock-in, less customization |

The decision usually comes down to two axes: **statefulness** and **operational
appetite**.

```
                  STATELESS                       STATEFUL (sessions / SSE)
              ┌─────────────────────┐         ┌─────────────────────────────┐
   Low-ops    │ Cloud Run           │         │ Cloudflare Containers       │
   (managed)  │ Cloudflare Workers  │         │ Fly.io                      │
              │ Lambda (RPC only)   │         │ Smithery / hosted-MCP       │
              └─────────────────────┘         └─────────────────────────────┘
              ┌─────────────────────┐         ┌─────────────────────────────┐
   High-ops   │ Kubernetes          │         │ Kubernetes + sticky ingress │
   (DIY)      │ Bare VMs            │         │ Bare VMs + HAProxy          │
              └─────────────────────┘         └─────────────────────────────┘
```

💡 If you're not sure whether you need stateful, start stateless. The streamable HTTP
spec lets you upgrade later without breaking clients.

---

## 7. Distributing stdio Servers

For the install-locally distribution shape, treat the server like any other CLI tool.

**Python (PyPI)**

```toml
# pyproject.toml
[project.scripts]
mcp-weather = "myserver:cli_main"
```

```bash
uv build
uv publish
```

End-user adds to their host config:

```jsonc
// claude_desktop_config.json
{
  "mcpServers": {
    "weather": {
      "command": "uvx",
      "args": ["mcp-weather"],
      "env": { "WEATHER_API_KEY": "..." }
    }
  }
}
```

`uvx` (and `pipx`) creates an isolated venv per server, so dependency conflicts
between servers can't happen.

**TypeScript (npm)**

```json
{
  "name": "@example/mcp-weather",
  "bin": { "mcp-weather": "./dist/cli.js" },
  "files": ["dist"]
}
```

```bash
npm publish --access public
```

```jsonc
{
  "mcpServers": {
    "weather": {
      "command": "npx",
      "args": ["-y", "@example/mcp-weather"]
    }
  }
}
```

Either way, ship these flags:

- `--help` — usage and all flags
- `--version` — exact version string (matches `serverInfo.version` in the protocol
  handshake; users will reference it in bug reports)
- `MCP_LOG_LEVEL=debug` env var — verbose logging for debugging install issues

⚠️ stdio servers must **never** write logs to stdout — that's the protocol channel.
Always log to stderr or a file. This is the most common bug in first-time MCP
servers.

---

## 8. The Official MCP Registry

Introduced in 2025 and now the canonical discovery mechanism.

- **Browse**: registry.modelcontextprotocol.io
- **Scale (as of 2026)**: ~6,400 servers in the official registry; tens of
  thousands more across community registries (MCP.so, MCPMarket, Smithery)
- **What you publish**: the package itself (PyPI / npm / OCI image) plus a
  metadata file describing transport(s), install command, configuration schema,
  and required permissions
- **Who consumes it**: Claude Desktop, ChatGPT, Cursor, VS Code, and most other
  hosts increasingly install directly from the registry — users browse a catalog
  rather than hand-editing JSON config

A minimal registry manifest:

```yaml
# server.yaml
name: example/weather
version: 1.4.2
description: NOAA weather data and forecasts
transports:
  - type: stdio
    install: { type: pypi, package: mcp-weather }
    command: { run: uvx, args: [mcp-weather] }
  - type: http
    url: https://weather-mcp.example.com
config:
  - name: WEATHER_API_KEY
    required: true
    secret: true
    description: NOAA API token
```

For the broader landscape — gateways, proxies, registries, host capabilities — see
[../reference/01_ecosystem.md](../reference/01_ecosystem.md).

---

## 9. Configuration Management

The configuration story differs by distribution shape.

**stdio servers**: configuration arrives as env vars set by the host config (the
`env` block in `claude_desktop_config.json`, etc.). Read them at startup; fail loudly
if required ones are missing.

```python
import os, sys

API_KEY = os.environ.get("WEATHER_API_KEY")
if not API_KEY:
    print("ERROR: WEATHER_API_KEY env var is required", file=sys.stderr)
    sys.exit(1)
```

**HTTP servers**: env vars are still the right primitive, but never bake values into
images or commit them to git. Use a real secret store:

- AWS Secrets Manager + IRSA (IAM roles for service accounts)
- HashiCorp Vault with the agent injector
- Doppler / Infisical for SaaS shops
- Kubernetes `Secret` resources (acceptable for moderate-sensitivity values; see
  caveats below)

For Kubernetes, **mount secrets as files** rather than env vars when:

- The secret is large (>4 KB)
- It's rotated frequently — file-mounted secrets update in-place; env-var secrets
  require a pod restart
- It contains structured data (PEM keys, kubeconfigs)

```yaml
volumeMounts:
  - name: tls-certs
    mountPath: /etc/tls
    readOnly: true
volumes:
  - name: tls-certs
    secret:
      secretName: weather-tls
      defaultMode: 0400
```

> **Rule**: secrets in env vars leak into crash dumps, debug endpoints, error
> trackers, and `kubectl describe`. Secrets in mounted files don't.

---

## 10. Day-2 Operations Checklist

Things that are easy to skip on day one and painful to retrofit on day ninety.

- ✅ **Structured logs to stdout** — JSON lines, picked up by your logging stack
  (CloudWatch, Loki, Datadog). Include `request_id`, `session_id`, `tool_name`.
- ✅ **Prometheus `/metrics` endpoint** — request rate, error rate, latency
  histograms per tool. Scraped by your monitoring. See
  [04_observability.md](04_observability.md).
- ✅ **OTel traces exported to your APM** — span per tool call, child spans for
  upstream dependencies. Critical for debugging "why did this tool take 12 seconds?"
- ✅ **Health checks wired to LB and Kubernetes probes** — both `/healthz` and
  `/ready`, doing different jobs.
- ✅ **Rolling deploys with `maxSurge: 1, maxUnavailable: 0`** — never reduce
  capacity mid-deploy.
- ✅ **`terminationGracePeriodSeconds` > longest expected tool latency** — otherwise
  in-flight calls get SIGKILL'd during deploys.
- ✅ **Image scanning + dependency updates in CI** — Trivy or Grype for the image,
  Renovate or Dependabot for the lock file. Catch CVEs before users do.
- ✅ **Quarterly DR test** — kill a replica during business hours; simulate a region
  failure; pull the plug on a critical dependency. If you've never tested it, it
  doesn't work.
- ✅ **Versioning hygiene** — semver on the package, semver on the container tag
  (never deploy `:latest`), and report `serverInfo.version` in the MCP handshake so
  clients can include it in bug reports.

---

## 11. Failure Modes

Things that will absolutely happen if you skip them.

- ❌ **Running as root in the container** — host CVEs become root CVEs. The fix
  costs one line in the Dockerfile.
- ❌ **Ignoring SIGTERM and letting K8s SIGKILL** — drops in-flight tool calls,
  truncates streaming responses, leaves users staring at half-rendered tables.
  Handle SIGTERM, stop accepting new requests, drain in-flight ones, then exit.
- ❌ **Baking secrets into the image** — they end up in registries forever, in
  every developer's pulled cache, in image-scan reports. Rotation becomes
  impossible. Use a secret store.
- ❌ **No version pinning in the lock file** — non-deterministic builds. Two CI runs
  twelve hours apart produce different artifacts. Debugging is a coin flip.
- ❌ **One pod, no HPA** — the first traffic spike (or the first node drain) drops
  the service. Three-pod minimum, autoscale on CPU.
- ⚠️ **Cloud Run / Lambda for stateful servers** — long-lived SSE doesn't fit the
  request/response platform model. Cloud Run caps streams at the request timeout
  (60 minutes max); Lambda has no SSE at all. For stateful MCP, pick Cloudflare
  Containers, Fly, or Kubernetes.

```
┌─────────────────────────────────────────────────────────────────┐
│  SIGTERM handling (the one bug you will actually hit)           │
│                                                                 │
│  K8s sends SIGTERM ──► server has terminationGracePeriodSeconds │
│                        to:                                      │
│                          1. stop accepting new connections      │
│                          2. let in-flight tool calls finish     │
│                          3. close SSE streams cleanly           │
│                          4. exit 0                              │
│                                                                 │
│  If you don't ──► SIGKILL at the grace deadline ──► dropped    │
│                   requests, truncated streams, angry users      │
└─────────────────────────────────────────────────────────────────┘
```

A minimal SIGTERM handler in Python:

```python
import asyncio, signal

async def main() -> None:
    server = build_http_server()
    stop = asyncio.Event()

    def _on_signal() -> None:
        # Log first, then trigger drain
        logger.info("received SIGTERM, draining")
        stop.set()

    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, _on_signal)

    serve_task = asyncio.create_task(server.serve())
    await stop.wait()
    await server.shutdown(timeout=300)   # drain in-flight tool calls
    serve_task.cancel()
```

> **Key insight**: deployment quality is measured during the deploy, not after it.
> If you can't deploy at peak traffic without dropping a single request, you have
> work to do — regardless of how green your dashboards look at 3 AM.

---

**Next**: [Reference: Ecosystem](../reference/01_ecosystem.md)
