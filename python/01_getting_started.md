# Getting Started with FastMCP 3.x in Python

> **Who this is for**: Python developers who know the language well but haven't touched
> FastMCP. Assumes you've read [../fundamentals/03_primitives.md](../fundamentals/03_primitives.md)
> so you already know what tools, resources, and prompts are conceptually. By the end of
> this file you'll have a server running over both stdio and Streamable HTTP, a working
> client, and a Claude Desktop integration.

---

## 1. Install

FastMCP is the canonical Python framework for the Model Context Protocol. It's maintained
by [PrefectHQ](https://github.com/jlowin/fastmcp), the latest release at the time of
writing is **3.2.4** (April 14, 2026), and it requires **Python 3.11+**.

```bash
# Preferred: uv (fast, lockfile-based, handles the venv for you)
uv add fastmcp

# Or plain pip
pip install fastmcp

# Upgrade an existing install
python -m pip install --upgrade fastmcp
```

Verify the install:

```bash
python -c "import fastmcp; print(fastmcp.__version__)"
# 3.2.4
```

> **Tip**: If you're starting a fresh project, `uv init && uv add fastmcp` gets you a
> working `pyproject.toml`, lockfile, and venv in one step. The rest of this file assumes
> a uv-managed project, but everything works identically with `pip` + `venv`.

---

## 2. Hello-World Server

Create `server.py`:

```python
from fastmcp import FastMCP

mcp = FastMCP("weather-demo")

@mcp.tool
def get_temperature(city: str) -> str:
    """Return the current temperature in a city."""
    return f"It is 22°C in {city}."

if __name__ == "__main__":
    mcp.run()  # defaults to stdio
```

Five lines worth understanding:

| Line | What it does |
|------|-------------|
| `FastMCP("weather-demo")` | Creates the server instance. The string is the server name advertised to clients during the MCP handshake. |
| `@mcp.tool` | Registers the function as an MCP tool. FastMCP introspects the function at decoration time. |
| `city: str` type hint | Becomes the tool's input JSON Schema (`{"type": "string"}`). Missing or wrong type hints will raise at startup. |
| `-> str` return hint | Becomes the tool's output schema. Return a `dict`, `list`, `pydantic.BaseModel`, or `dataclass` for structured output. |
| Docstring | Becomes the tool's `description` field — the text the LLM actually reads when deciding whether to call the tool. **Treat it as production copy.** |
| `mcp.run()` | Starts the event loop. With no arguments it speaks stdio, blocking on stdin until the client closes. |

> **Key insight**: FastMCP is "schema-from-signature." Your Python function *is* the
> contract. There's no separate schema file, no manual JSON-Schema dictionaries. Type
> hints + docstring → tool definition. This is also why poor docstrings translate
> directly into poor tool-selection behavior from the model.

---

## 3. Run It Locally

### Stdio (the default)

```bash
python server.py
```

That's it — but the process now sits silently waiting for a client to speak MCP framed
JSON-RPC over its stdin. Pressing keys in your terminal will **not** produce useful
output; stdio servers are meant to be spawned as subprocesses by hosts like Claude
Desktop, Cursor, or your own client code.

### Inspect with the official MCP Inspector

The fastest way to poke at a server during development is the official inspector, which
launches a local web UI and a stdio client wired to your script:

```bash
npx @modelcontextprotocol/inspector python server.py
```

It opens `http://127.0.0.1:6274` in your browser. From there you can:

- Browse the registered tools, resources, and prompts.
- Invoke tools with arbitrary arguments and see the raw JSON-RPC traffic.
- View server-emitted log messages and notifications.

Use it for every server you build. It's the MCP equivalent of `curl` + Postman.

---

## 4. Streamable HTTP Server

Stdio is great for desktop hosts spawning local subprocesses. For anything remote — a
deployed service, a multi-tenant integration, anything behind a load balancer — you want
**Streamable HTTP**, the modern HTTP+SSE transport that replaced the older "HTTP+SSE
dual endpoint" design.

```python
if __name__ == "__main__":
    mcp.run(transport="http", host="127.0.0.1", port=8080)
```

Run it:

```bash
python server.py
# INFO     Starting MCP server 'weather-demo' with transport http
# INFO     Uvicorn running on http://127.0.0.1:8080
```

The MCP endpoint is mounted at **`http://127.0.0.1:8080/mcp/`** by default (note the
trailing slash). Clients POST JSON-RPC requests to that path and receive responses as
either a single JSON body or a streamed `text/event-stream` depending on whether the
server needs to send progress notifications.

| Argument | Purpose |
|----------|---------|
| `transport="http"` | Streamable HTTP. (Use `"stdio"` or `"sse"` for the legacy transports.) |
| `host` | Bind address. Use `"0.0.0.0"` to expose externally. |
| `port` | TCP port. Pick something that won't collide with your dev server. |
| `path` | Override the default `/mcp/` mount path. |

⚠️ Don't expose an HTTP MCP server to the open internet without authentication. FastMCP
ships auth middleware (OAuth, bearer tokens, custom verifiers) — see the auth chapter
later in this directory.

---

## 5. A Complete Hello-World Client

FastMCP includes a high-level `Client` that handles all three transports through a
single API. Save as `client.py`:

```python
import asyncio
from fastmcp import Client

async def main() -> None:
    async with Client("python server.py") as client:
        # discover tools
        tools = await client.list_tools()
        for t in tools:
            print(t.name, "—", t.description)

        # call one
        result = await client.call_tool("get_temperature", {"city": "Lisbon"})
        print(result.data)  # FastMCP unpacks structured output for you

asyncio.run(main())
```

Run it:

```bash
python client.py
# get_temperature — Return the current temperature in a city.
# It is 22°C in Lisbon.
```

The `Client` constructor is overloaded to **infer the transport from its argument**:

| Input | Transport chosen |
|-------|------------------|
| `"python server.py"` (or any path / shell command string) | Spawns a stdio subprocess |
| `"http://host:port/mcp/"` | Streamable HTTP |
| `"http://host:port/sse"` | Legacy HTTP+SSE (auto-detected by suffix) |
| A `FastMCP` instance directly | **In-memory transport** — no subprocess, no socket |

The in-memory case is gold for testing:

```python
from fastmcp import Client
from server import mcp  # import your FastMCP instance directly

async def test_temperature():
    async with Client(mcp) as client:
        result = await client.call_tool("get_temperature", {"city": "Lisbon"})
        assert "22" in result.data
```

No subprocess, no port, no network. Just function calls through the same code path the
real client would use. This is the recommended way to write unit tests for MCP servers.

> **Rule**: `result.data` is the unwrapped Python value FastMCP reconstructed from the
> tool's structured output. `result.content` is the raw list of MCP `Content` blocks
> (text, image, resource references) if you need to inspect the wire format.

---

## 6. Async Tools

FastMCP accepts both sync and async functions. The framework runs sync tools on a worker
thread automatically (via `anyio.to_thread.run_sync`), so a slow `requests.get()` won't
block the event loop. You only need `async def` when you actually want concurrency
inside the tool:

```python
import httpx

@mcp.tool
async def fetch_status(url: str) -> dict:
    """Fetch and parse a URL."""
    async with httpx.AsyncClient() as client:
        r = await client.get(url, timeout=5.0)
        return {"status": r.status_code, "headers": dict(r.headers)}
```

A few notes that bite people in production:

- **Return types matter.** Returning a `dict` produces structured output; the client gets
  both the JSON object *and* a text rendering. Returning a primitive (`str`, `int`,
  `bool`) produces text-only output.
- **Pydantic models are first-class.** Annotate the return as `MyModel` and FastMCP will
  call `.model_dump()` for you and emit the model's JSON Schema as the output schema.
- **Don't share mutable state across tool calls without locking.** Tools may run
  concurrently when the client pipelines requests, especially over HTTP.

```python
from pydantic import BaseModel

class Forecast(BaseModel):
    city: str
    temp_c: float
    summary: str

@mcp.tool
async def get_forecast(city: str) -> Forecast:
    """Return today's forecast for a city."""
    # ... call your weather API ...
    return Forecast(city=city, temp_c=22.0, summary="Sunny")
```

The client's `result.data` will be a real `Forecast` instance if the model is importable
on the client side, or a `dict` matching the schema otherwise.

---

## 7. Wire It Up to Claude Desktop

Claude Desktop reads its MCP server list from a JSON config file:

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

Add an entry under `mcpServers`:

```json
{
  "mcpServers": {
    "weather-demo": {
      "command": "uv",
      "args": ["run", "--directory", "/abs/path/to/project", "python", "server.py"]
    }
  }
}
```

The `--directory` flag tells uv where the project's `pyproject.toml` lives so it picks up
the right venv. Use absolute paths — Claude Desktop launches the command from its own
working directory and won't resolve relative paths the way your shell does.

Then:

1. Save the config.
2. Quit Claude Desktop completely (⌘Q on macOS — closing the window isn't enough; the
   helper process keeps running).
3. Relaunch.
4. Click the hammer icon in the chat input — `weather-demo` and its `get_temperature`
   tool should appear.

If it doesn't show up, check the logs:

```bash
tail -f ~/Library/Logs/Claude/mcp*.log
```

Common failure modes:

| Symptom | Cause |
|---------|-------|
| Server never appears, no log entries | JSON syntax error in the config file. Validate it. |
| "spawn uv ENOENT" | `uv` isn't on the `PATH` Claude Desktop sees. Use the absolute path: `/Users/you/.local/bin/uv`. |
| Server crashes on startup | Print to **stderr**, not stdout — stdout is the MCP transport channel. Any `print()` to stdout corrupts the JSON-RPC stream. |

> **Rule**: For stdio servers, **never** write to stdout from your tool code, dependencies,
> or any logging configuration. Send all logs to stderr (`logging.basicConfig(stream=sys.stderr)`)
> or, better, use FastMCP's structured logging which routes through MCP's own log channel.

---

## 8. Other Hosts: Cursor, VS Code, Claude Code

The config schema is broadly similar across hosts — `command` + `args` with optional
`env` — but the file location and a few fields differ:

| Host | Config location |
|------|----------------|
| Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Cursor | `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (project) |
| VS Code (Copilot Chat) | `.vscode/mcp.json` in the workspace |
| Claude Code | `~/.claude.json` (`mcpServers` key) or `claude mcp add` CLI |

For Streamable HTTP servers, the entry collapses to a single URL:

```json
{
  "mcpServers": {
    "weather-demo": {
      "url": "http://127.0.0.1:8080/mcp/"
    }
  }
}
```

See [../reference/01_ecosystem.md](../reference/01_ecosystem.md) for the full host-by-host
comparison including auth, environment variables, and feature support matrices.

---

## 9. Next Steps

You now have:

- A FastMCP server that exposes one tool over stdio and HTTP.
- A FastMCP client that talks to it over either transport.
- An in-memory testing pattern.
- A working Claude Desktop integration.

The next file goes deep on tools — parameter validation with Pydantic, structured
output, error handling, progress reporting, cancellation, and the `Context` object that
gives tools access to logging, sampling, and the host's filesystem.

Related reading:

- Previous: [../fundamentals/05_transports.md](../fundamentals/05_transports.md) — what
  stdio vs. Streamable HTTP actually do at the wire level.
- TypeScript equivalent: [../typescript/01_getting_started.md](../typescript/01_getting_started.md)
- Host comparison: [../reference/01_ecosystem.md](../reference/01_ecosystem.md)

---

**Next**: [02_tools.md — Tools In Depth](02_tools.md)
