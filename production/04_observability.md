# Observability for MCP Servers

> **Who this is for**: SRE and platform engineers who'll own the pages, dashboards, and
> on-call rotation for an MCP server. Assumes you've read
> [03_scaling.md](03_scaling.md) and have a working OpenTelemetry collector and
> Prometheus stack to point things at.

---

## 1. Why MCP Observability Is Different

A traditional service is a closed loop: a client makes a request, your server hits a
database, you return a response. The trace fits on one screen.

```
   client ──▶ server ──▶ db ──▶ response
```

An MCP server is a relay in someone else's reasoning loop:

```
   user
    │
    ▼
   agent (LLM)            ← decides which tool to call
    │
    ▼
   host (Claude Desktop / Code / ChatGPT)
    │
    ▼
   MCP client
    │
    ▼
   MCP server (you)       ← receives tools/call
    │
    ▼
   tool handler
    │
    ▼
   upstream API / DB
    │
    ▼
   response ──▶ propagated back up the chain
```

Every layer can stall, retry, or mangle context. When a user complains *"Claude was slow
on that lookup"*, you need to answer four questions at once:

1. Did the model spend 25 seconds thinking, or did your tool spend 25 seconds executing?
2. If the tool was slow, was it your code or the upstream it called?
3. Was this one user, or are all sessions affected?
4. Did the agent retry the tool five times because the first response confused it?

> **Key insight**: Without **end-to-end distributed traces**, MCP debugging is guessing.
> A trace that links the host's tool-call decision to your span to the upstream HTTP
> call is the single highest-leverage piece of telemetry you can ship.

The rest of this file is how to get there: what to instrument, how to propagate context,
what dashboards to build, and the failure modes that will bite you in production.

---

## 2. What to Instrument — Four Golden Signals, Adapted

Google's golden signals (latency, traffic, errors, saturation) translate cleanly to MCP
once you remember that "request" means "tool call" and "user" means "agent session."

| Signal      | MCP form                                                                                  |
|-------------|-------------------------------------------------------------------------------------------|
| Latency     | Per-tool histograms — p50/p95/p99. Tools have wildly different baselines; **never** average across them. |
| Errors      | Per-tool error rate, broken down by class: `isError` results, exceptions, auth failures, validation. |
| Traffic     | Calls per tool per minute. Spikes per tool can indicate prompt injection or runaway agent loops. |
| Saturation  | Open SSE streams, concurrent tool executions, upstream connection-pool usage, DB pool depth. |

A few things worth calling out:

- **`isError: true` results are still successful HTTP responses.** They're tool-level
  errors that the model is supposed to see and react to. Track them separately from
  exceptions — they're a different signal.
- **Auth failures deserve their own counter.** They're almost always a misconfigured
  client (wrong token, expired credential, wrong scope). They're the leading indicator
  of an outage caused by a deploy on someone else's side.
- **Saturation on the SSE side is sneaky.** A leaked GET stream consumes a worker for
  hours and shows up as "we ran out of capacity at 200 RPS" when you should be doing 5k.

---

## 3. Structured Logs — What Every Line Should Carry

Plain-text logs are a debugging tax. Every log line your MCP server emits should be JSON
with the same minimum fields, so you can `WHERE tool_name = 'lookup' AND outcome != 'ok'`
in your log store and get a useful answer.

| Field         | Source                          | Why                                                     |
|---------------|---------------------------------|---------------------------------------------------------|
| `request_id`  | JSON-RPC `id`                   | Correlate request, response, and any logs in between    |
| `session_id`  | `Mcp-Session-Id` header         | Group all calls in one agent conversation               |
| `client_id`   | OAuth `sub` or session metadata | Attribute usage and rate limits                         |
| `method`      | JSON-RPC `method`               | `tools/call`, `resources/read`, `initialize`, etc.      |
| `tool_name`   | request params                  | The thing operators actually filter on                  |
| `trace_id`    | OTel context                    | Jump from log to trace in one click                     |
| `span_id`     | OTel context                    | Pinpoint the exact span that emitted the log            |
| `outcome`     | computed                        | `ok` / `error_validation` / `error_upstream` / `error_internal` |
| `duration_ms` | computed                        | Cheap latency signal even without metrics scraping      |

A FastMCP middleware that gets you most of the way there:

```python
import time
import structlog
from fastmcp.server.middleware import Middleware

logger = structlog.get_logger()

class StructuredLoggingMiddleware(Middleware):
    """Emit one structured line per JSON-RPC call with timing and outcome."""

    async def __call__(self, ctx, call_next):
        start = time.perf_counter()
        log = logger.bind(
            method=ctx.method,
            request_id=ctx.request_id,
            session_id=ctx.session_id,
            tool_name=getattr(ctx, "tool_name", None),
        )
        try:
            result = await call_next(ctx)
            log.info(
                "ok",
                ms=round((time.perf_counter() - start) * 1000, 1),
            )
            return result
        except Exception:
            # exception() captures stack trace; structlog renders it as JSON-safe data
            log.exception(
                "error",
                ms=round((time.perf_counter() - start) * 1000, 1),
            )
            raise
```

> **Rule**: bind once at the top of the handler, log at the bottom. Don't sprinkle
> `logger.info("step 3")` calls through the body — that's what spans are for.

---

## 4. Prometheus Metrics — A Starter Set

Three metrics get you 80% of the way to a real dashboard. Resist the urge to add more
until you've actually used these in an incident.

```python
from prometheus_client import Counter, Histogram, Gauge

TOOL_CALLS = Counter(
    "mcp_tool_calls_total",
    "MCP tool calls",
    ["server", "tool", "outcome"],
)

TOOL_LATENCY = Histogram(
    "mcp_tool_latency_seconds",
    "MCP tool latency",
    ["server", "tool"],
    # Buckets matter. Pick ranges that bracket your real p50 → p99.
    # Defaults are tuned for HTTP; many MCP tools are LLM-adjacent and slower.
    buckets=(0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30, 60),
)

ACTIVE_SESSIONS = Gauge(
    "mcp_active_sessions",
    "Active MCP sessions",
    ["server"],
)

ACTIVE_STREAMS = Gauge(
    "mcp_active_sse_streams",
    "Open server-push SSE streams",
    ["server"],
)
```

Wire them into your tool middleware so they update on every call:

```python
class MetricsMiddleware(Middleware):
    def __init__(self, server_name: str) -> None:
        self.server = server_name

    async def __call__(self, ctx, call_next):
        if ctx.method != "tools/call":
            return await call_next(ctx)

        tool = ctx.tool_name or "unknown"
        with TOOL_LATENCY.labels(self.server, tool).time():
            try:
                result = await call_next(ctx)
                outcome = "error_tool" if getattr(result, "isError", False) else "ok"
                TOOL_CALLS.labels(self.server, tool, outcome).inc()
                return result
            except ValidationError:
                TOOL_CALLS.labels(self.server, tool, "error_validation").inc()
                raise
            except UpstreamError:
                TOOL_CALLS.labels(self.server, tool, "error_upstream").inc()
                raise
            except Exception:
                TOOL_CALLS.labels(self.server, tool, "error_internal").inc()
                raise
```

⚠️ **Bucket choice is load-bearing.** A tool that calls an LLM has a p50 around 2s and a
p99 around 30s. The default Prometheus buckets top out at 10s and your "p99 latency"
panel will lie to you.

---

## 5. OpenTelemetry Tracing — Python

The single most useful thing you can ship. A span per JSON-RPC call, attributes for the
MCP-specific context, and auto-instrumentation for your downstream HTTP and DB calls.

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.asyncpg import AsyncPGInstrumentor
from fastmcp.server.middleware import Middleware

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter())
)

# Auto-instrument the libraries your tools actually use. Each one adds child spans
# automatically, which is how you get tool → upstream-API visibility for free.
HTTPXClientInstrumentor().instrument()
AsyncPGInstrumentor().instrument()

tracer = trace.get_tracer("mcp.server")


class TracingMiddleware(Middleware):
    async def __call__(self, ctx, call_next):
        # Adopt incoming context if the host propagated W3C headers (see Section 7).
        # If not, this still creates a useful root span.
        with tracer.start_as_current_span(
            ctx.method,
            attributes={
                "mcp.method": ctx.method,
                "mcp.request_id": str(ctx.request_id),
                "mcp.session_id": ctx.session_id or "",
                "mcp.tool.name": getattr(ctx, "tool_name", "") or "",
            },
        ) as span:
            try:
                result = await call_next(ctx)
                # isError is a tool-level failure, not an exception — mark the span
                # as ERROR so it shows up red in the trace UI.
                if getattr(result, "isError", False):
                    span.set_status(
                        trace.StatusCode.ERROR, "tool returned isError"
                    )
                return result
            except Exception as exc:
                span.record_exception(exc)
                span.set_status(trace.StatusCode.ERROR)
                raise
```

> **Note**: **FastMCP 3.x ships built-in OpenTelemetry** for tool calls, resource reads,
> and JSON-RPC framing — enabling it is usually a config flag, not the manual middleware
> above. The example is here so you understand what the framework is doing and can
> customize attributes (e.g. add `mcp.tool.input.size` for capacity planning).

---

## 6. OpenTelemetry Tracing — TypeScript

Same shape, different SDK. The TypeScript MCP SDK exposes `extra.requestId` and
`extra.sessionId` on every tool handler — wire them onto the span the same way.

```typescript
import { NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { trace, SpanStatusCode } from "@opentelemetry/api";

const provider = new NodeTracerProvider();
provider.addSpanProcessor(new BatchSpanProcessor(new OTLPTraceExporter()));
provider.register();

const tracer = trace.getTracer("mcp.server");

server.registerTool(
  "lookup",
  {
    description: "Look up a record by id",
    inputSchema: { id: z.string() },
  },
  async (input, extra) =>
    tracer.startActiveSpan("tool.lookup", async (span) => {
      span.setAttributes({
        "mcp.tool.name": "lookup",
        "mcp.request_id": String(extra.requestId),
        "mcp.session_id": extra.sessionId ?? "",
      });
      try {
        return await doLookup(input);
      } catch (e) {
        span.recordException(e as Error);
        span.setStatus({ code: SpanStatusCode.ERROR });
        throw e;
      } finally {
        // Always end the span — leaked spans are silently dropped by the batch processor
        // and you'll wonder why some tool calls have no traces.
        span.end();
      }
    }),
);
```

Pair this with `@opentelemetry/auto-instrumentations-node` to get spans for `fetch`,
`pg`, `mysql2`, `redis`, etc., for free.

---

## 7. Trace Context Propagation Across the Agent Boundary

This is the part most teams skip and then regret. Without propagation, your MCP server
trace and your upstream API trace are two disconnected fragments — useless for
end-to-end debugging.

The W3C Trace Context spec defines two HTTP headers:

| Header        | Purpose                                                       |
|---------------|---------------------------------------------------------------|
| `traceparent` | The trace ID, parent span ID, and sampling flag               |
| `tracestate`  | Vendor-specific extensions (e.g. AWS X-Ray, Datadog tags)     |

**Hosts (Claude Desktop, Claude Code, ChatGPT) increasingly forward these headers when
configured to.** Your transport-level middleware should extract them *before* creating
its own span, so the host's trace ID becomes the parent of yours:

```python
from opentelemetry.propagate import extract

# In your HTTP transport handler, before any tracer.start_as_current_span()
ctx_in = extract(dict(request.headers))

with tracer.start_as_current_span("mcp.request", context=ctx_in) as span:
    # span.context.trace_id == the host's trace_id
    # All child spans (your tool handler, the auto-instrumented httpx call, the asyncpg
    # query) inherit this — full distributed trace from agent to upstream.
    ...
```

When you then make upstream HTTP calls, the auto-instrumented HTTPX (or `fetch` in
TypeScript) **injects the same `traceparent` header onward**, so the upstream's spans
also become children of the host's trace.

End-to-end picture:

```
   ┌─ host span (Claude Code) ───────────────────────────┐
   │  traceparent: 00-aaaa-1111-01                       │
   │                                                     │
   │   ┌─ mcp.request (your server) ───────────────────┐ │
   │   │  parent: 1111                                 │ │
   │   │                                               │ │
   │   │   ┌─ tool.lookup ─────────────────────────┐   │ │
   │   │   │   ┌─ HTTP GET upstream.example.com ┐  │   │ │
   │   │   │   │  traceparent: 00-aaaa-3333-01  │  │   │ │
   │   │   │   └─────────────────────────────── ┘  │   │ │
   │   │   │   ┌─ pg.query SELECT ...           ┐  │   │ │
   │   │   │   └─────────────────────────────── ┘  │   │ │
   │   │   └───────────────────────────────────────┘   │ │
   │   └───────────────────────────────────────────────┘ │
   └─────────────────────────────────────────────────────┘
```

> **Rule**: extract context at the transport layer, not in your tool handler. By the
> time you're inside a tool, the spans above you have already been created — too late
> to attach them to the host's trace.

---

## 8. GenAI Semantic Conventions

OpenTelemetry has an emerging set of GenAI semantic conventions that standardize span
naming and attributes for AI-shaped systems. They're still in flux, but adopting them
now saves you a rename later when your APM vendor adds GenAI dashboards out of the box.

| Span name                                  | Use for                                  |
|--------------------------------------------|------------------------------------------|
| `mcp.tool/<tool_name>` or `gen_ai.tool.execution` | A tool execution                  |
| `mcp.resource.read`                        | A resource read                          |
| `mcp.prompt.get`                           | A prompt fetch                           |

Standard attributes:

| Attribute              | Value                       |
|------------------------|-----------------------------|
| `gen_ai.system`        | `"mcp"`                     |
| `gen_ai.operation.name`| `"tool"` / `"resource"` / `"prompt"` |
| `gen_ai.tool.name`     | e.g. `"get_user"`           |
| `gen_ai.tool.call.id`  | The tool call id from the host |
| `mcp.method`           | `"tools/call"`, etc.        |
| `mcp.session_id`       | Your session id             |
| `error.type`           | `"validation"`, `"upstream"`, `"internal"` |
| `exception.*`          | Standard OTel exception attrs (set automatically by `record_exception`) |

💡 Don't put the tool's input or output in span attributes. Inputs may contain user data
(PII, secrets), outputs can be large, and spans aren't designed for that. Use a
separate, sampled audit pipeline if you need full request/response capture.

---

## 9. Sample Dashboard Panels (Grafana)

A good MCP dashboard answers "is anything weird?" in one glance. Six panels do most of
the work:

| Panel                                          | What it tells you                                            |
|------------------------------------------------|--------------------------------------------------------------|
| Top tools by p95 latency                       | Regressions show up here first                               |
| Error rate by tool, stacked by error class     | Prompt injection or runaway agents stand out                 |
| Active sessions / open SSE streams             | Capacity pressure and stream leaks                           |
| Auth failures per minute                       | Leading indicator of misconfigured clients                   |
| Tool calls per session per minute              | Healthy loops cluster at 1–3; spikes to 100+ = infinite loop |
| Upstream pool saturation                       | The most common root cause of "MCP is slow"                  |

The "tool calls per session per minute" panel deserves elaboration. A well-behaved agent
calls 1–3 tools per minute during a conversation. A panicking or jailbroken agent will
hammer the same tool dozens of times in a row — sometimes thousands per minute. This
panel is your earliest warning that something has gone wrong upstream of you, and it's
trivial to compute:

```promql
sum by (session_id) (rate(mcp_tool_calls_total[1m])) * 60
```

⚠️ Alert on `> 30 calls/min/session` for at least two minutes. Page on `> 200/min/session`
— at that rate you're being abused or there's a host bug, and either way you want
someone looking at it now.

---

## 10. Sampling and Cost

Telemetry isn't free. A naive setup will quintuple your observability bill the first
month an agent goes viral.

| Signal | Strategy                                                                                     | 
|--------|----------------------------------------------------------------------------------------------|
| Traces | Trace 100% of errors and slow requests. Head-sample successful traces at ~10%.               |
| Logs   | Structured JSON to stdout in production. Let your log shipper handle ingestion and rotation. |
| Metrics| Cheap. Keep all of them, but watch label cardinality (see Section 11).                       |

> **Rule**: head-based sampling is fine for "show me what a normal request looks like."
> **Tail-based sampling** in the OTel collector is what you actually want — it buffers
> the whole trace and only keeps it if something interesting happened (an error, a
> latency outlier, a specific user cohort). Configure it once in the collector and your
> SDKs stay simple.

A reasonable starting policy in your collector:

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow
        type: latency
        latency: { threshold_ms: 1000 }
      - name: probabilistic
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }
```

---

## 11. Failure Modes

The mistakes that will burn you, in roughly the order they happen:

❌ **Logging the entire tool input/output.** Leaks user data, balloons log volume, and
makes your log store unsearchable. Sample, redact, or just don't.

❌ **Spans without parent context.** If you forget to extract `traceparent` at the
transport layer, every request becomes its own root trace and you lose the connection
to the host. Always extract from headers before creating your first span.

❌ **Counters with high-cardinality labels.** `user_id`, `request_id`, or `session_id` as
Prometheus labels will OOM your metrics backend. Stick to `tool` and `outcome`. If you
need per-user breakdowns, use logs with structured fields — they're built for high
cardinality.

⚠️ **Logging large content blocks.** Resource reads can return megabytes. Tool outputs
can include images encoded as base64. Truncate aggressively (1–2 KB max) and store full
payloads separately if you really need them.

❌ **Forgetting to instrument the GET stream.** If you only wrap `tools/call`, your
server-push SSE paths have no traces and no spans. Wrap the transport, not just the
RPC handler.

❌ **Histogram buckets sized for HTTP, not LLMs.** Default buckets top out at 10s. An
LLM-backed tool routinely exceeds that. Your "p99 latency" panel will read `+Inf` and
you'll think your dashboard is broken when actually it's correct and damning.

⚠️ **Not alerting on auth failures.** A spike in 401s means a client deploy went wrong
or a credential rotated. You want to know within minutes, not when the customer files a
ticket.

❌ **One log line per "step."** This is what spans are for. Bind a logger at the top of
the handler, log once at the bottom with the outcome and duration. Use spans for
intermediate timing.

---

## Cross-references

- Previous: [03_scaling.md](03_scaling.md)
- Next: [05_security.md](05_security.md)
- Related: [../python/06_middleware.md](../python/06_middleware.md),
  [../typescript/05_advanced.md](../typescript/05_advanced.md)

---

**Next**: [05_security.md — Security: Threat Model, Gateways, Sandboxing](05_security.md)
