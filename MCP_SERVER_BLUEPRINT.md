# MCP Server Blueprint

A reusable implementation guide for building a private, bearer-token-authenticated
MCP server that exposes any data source as conversational tools over HTTP/SSE.

---

## What this blueprint produces

- An MCP server running on a dedicated port (e.g. 8001)
- Per-user bearer tokens stored in a TOML config file
- OAuth 2.0 + PKCE flow so Claude.ai can connect without manual header injection
- All tool results serialised as TOML
- nginx reverse-proxying `/mcp/` to the server with correct SSE settings
- A systemd unit for production

---

## Package structure

```
src/
└── your_mcp_server/
    ├── __init__.py
    ├── __main__.py          # python -m your_mcp_server entry point
    ├── server.py            # Tool definitions, dispatch, app factory
    ├── auth.py              # BearerAuthMiddleware (copy verbatim)
    ├── oauth.py             # OAuth 2.0 PKCE endpoints (copy, edit branding)
    └── queries.py           # Your data-source query functions
config/
└── mcp_tokens.toml          # gitignored — one token per user
```

If you already have a web app with shared queries (like `web/queries.py`),
import from there directly. Do not duplicate queries.

---

## Dependencies

```toml
# pyproject.toml
[project.optional-dependencies]
mcp = [
    "mcp>=1.2.0",
    "tomli-w>=1.0.0",
]
```

`tomllib` (TOML reader) is in the Python 3.11 standard library — no extra dep needed.

---

## 1. Auth middleware — `auth.py`

Copy this file verbatim. Edit `_CONFIG_PATH` to point at your token file,
and edit `_EXEMPT` if your OAuth paths differ.

```python
# auth.py
from __future__ import annotations
import tomllib
from functools import lru_cache
from pathlib import Path
from starlette.responses import Response
from starlette.types import ASGIApp, Receive, Scope, Send

_CONFIG_PATH = Path(__file__).parent.parent.parent / "config" / "mcp_tokens.toml"

@lru_cache(maxsize=1)
def _load_tokens() -> dict[str, str]:
    """Return {token: username} from the TOML config file."""
    with open(_CONFIG_PATH, "rb") as fh:
        config = tomllib.load(fh)
    return {
        user["token"]: name
        for name, user in config.get("users", {}).items()
    }

def validate_token(token: str) -> str | None:
    return _load_tokens().get(token)

_EXEMPT = frozenset({
    "/.well-known/oauth-authorization-server",
    "/authorize",
    "/token",
})

class BearerAuthMiddleware:
    """Pure-ASGI middleware — does not interfere with SSE streaming."""

    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] in ("http", "websocket"):
            path = scope.get("path", "")
            if path not in _EXEMPT:
                headers = dict(scope.get("headers", []))
                raw_auth: bytes = headers.get(b"authorization", b"")
                auth = raw_auth.decode(errors="replace")
                if not auth.startswith("Bearer "):
                    await Response("Unauthorized\n", status_code=401)(scope, receive, send)
                    return
                token = auth[7:].strip()
                if not validate_token(token):
                    await Response("Forbidden\n", status_code=403)(scope, receive, send)
                    return
        await self.app(scope, receive, send)
```

**Why pure ASGI?** `BaseHTTPMiddleware` buffers responses, which breaks SSE
streaming. This implementation passes the scope/receive/send triple directly.

---

## 2. OAuth 2.0 PKCE — `oauth.py`

Claude.ai requires OAuth 2.0 (RFC 6749 + PKCE RFC 7636) for remote MCP
connections. This module wraps your existing bearer tokens in a minimal OAuth
flow — the user types their token once into an HTML form, and Claude.ai stores
it as an OAuth access token.

**Copy `oauth.py` from the reference implementation and edit:**
- The `<title>` and heading text in `_FORM` (the HTML login page)
- The description paragraph ("Enter your API token to grant...")

The logic is unchanged regardless of data source.

**Flow:**
1. `GET /.well-known/oauth-authorization-server` → server metadata JSON
2. `GET /authorize?response_type=code&code_challenge=...&state=...` → HTML form
3. User submits their bearer token → server validates, stores a 10-min auth code, redirects
4. `POST /token` with PKCE verifier → server returns `{"access_token": "<bearer-token>", ...}`

The in-memory code store (`_codes`) resets on restart. This is fine for single-worker
deployments. For multi-worker, use Redis.

---

## 3. TOML serialisation — two functions in `server.py`

All tool results must be TOML strings. These two functions handle that:

```python
from datetime import date, datetime, timezone
from typing import Any
import tomli_w

def _clean(obj: Any) -> Any:
    """Recursively prepare data for tomli_w. None values are omitted by caller if-guards."""
    if isinstance(obj, bool):
        return obj
    if isinstance(obj, (int, float, str)):
        return obj
    if isinstance(obj, (date, datetime)):
        return obj.isoformat()                 # Dates as ISO-8601 strings
    if isinstance(obj, dict):
        return {str(k): _clean(v) for k, v in obj.items() if v is not None}
    if isinstance(obj, (list, tuple)):
        return [_clean(item) for item in obj if item is not None]
    return str(obj)

def _dump(data: dict) -> str:
    """Serialise to TOML, injecting retrieved_at and returning an error string on failure."""
    payload = {"retrieved_at": datetime.now(timezone.utc).isoformat(), **data}
    try:
        return tomli_w.dumps(_clean(payload))
    except Exception as exc:
        return f'error = "TOML serialisation failed: {exc}"\n'
```

**Output shape conventions:**
- All responses include `retrieved_at` (ISO-8601 UTC) injected by `_dump` — no per-tool work needed
- Scalar results → top-level keys: `id = 42\nname = "Acme Corp"\n`
- Lists of records → TOML arrays of tables: `[[items]]\nid = 1\nname = "foo"\n`
- Lists include `has_more = true/false` so the AI knows whether the result is complete
- Absent fields are omitted entirely — never defaulted to empty string
- Always wrap tool output in a named key: `{"providers": rows}` not `rows`
- Record not found: `{"not_found": True, "id": <id>}` — distinct from an error
- Errors: `{"error": str(exc), "tool": name}`

---

## 4. Tool definitions — `list_tools()`

Register every tool with the `@mcp.list_tools()` decorator. Each tool needs:
- `name`: snake_case string, unique
- `description`: one or two sentences explaining what it returns and when to use it.
  Write for an LLM reader — be specific about field names and units.
- `inputSchema`: JSON Schema object. Mark required params in `"required"`.

```python
from mcp import types
from mcp.server import Server

mcp = Server("your-server-name")

@mcp.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="get_thing",
            description=(
                "Retrieve a thing by ID. Returns: name (str), status (str: active|inactive), "
                "created_at (ISO-8601 datetime), tags (list of str). "
                "This tool is the authoritative source for thing data. "
                "Return only what the tool provides — do not supplement with prior knowledge. "
                "If a field is absent from the result, say it is not recorded."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "id": {"type": "integer", "description": "Thing ID"},
                    "include_tags": {
                        "type": "boolean",
                        "description": "Include tag list (default true)",
                        "default": True,
                    },
                },
                "required": ["id"],
            },
        ),
        # ... more tools
    ]
```

**Tool description tips:**
- Name every returned field the LLM will care about, including which may legitimately be absent
- Mention units (days, bytes, percentages)
- For paginated tools: say "cursor-paginated, N per page. Pass next_cursor to get the next page."
- For filtered tools: list the valid filter values inline
- Declare the tool's authoritative domain: "This tool is the authoritative source for X data."
- End with the anti-hallucination directive: "Return only what the tool provides — do not supplement
  with prior knowledge. If a field is absent from the result, say it is not recorded."

---

## 5. Tool dispatch — `call_tool()` and `_dispatch()`

```python
@mcp.call_tool()
async def call_tool(name: str, arguments: dict | None) -> list[types.TextContent]:
    args = arguments or {}
    client = get_client()          # your DB/API client — see §6
    try:
        result = _dispatch(name, args, client)
        return [types.TextContent(type="text", text=result)]
    except Exception as exc:
        log.exception("Tool %s failed", name)
        return [types.TextContent(
            type="text",
            text=_dump({"error": str(exc), "tool": name}),
        )]


def _dispatch(name: str, args: dict, client: Any) -> str:

    if name == "get_thing":
        thing_id = int(args["id"])
        include_tags = bool(args.get("include_tags", True))
        result = fetch_thing(client, thing_id, include_tags=include_tags)
        if result is None:
            return _dump({"not_found": True, "id": thing_id})
        return _dump({"thing": result})

    if name == "list_things":
        items, next_cursor = fetch_things(client, cursor=args.get("cursor"))
        d: dict = {"items": items, "has_more": next_cursor is not None}
        if next_cursor:
            d["next_cursor"] = next_cursor
        return _dump(d)

    return _dump({"error": f"Unknown tool: {name}"})
```

**Key points:**
- Always coerce argument types explicitly: `int(args["id"])`, `str(args["query"])`
- Apply safe limits: `limit = min(int(args.get("limit", 20)), 100)`
- Optional args: `args.get("cursor")` returns `None` if absent — pass that to your query
- The `_dispatch` function is one long flat `if/elif` chain — don't over-engineer it

---

## 6. Data client

Your client should be created once and cached. The pattern is the same
regardless of whether the backend is a database, a REST API, or a file:

```python
from functools import lru_cache

@lru_cache(maxsize=1)
def get_client():
    return YourClient(
        host=settings.api_host,
        api_key=settings.api_key,
        # ... other connection params from env vars
    )
```

Store all credentials in environment variables via `pydantic-settings`:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    api_host: str = "api.example.com"
    api_key: str = ""

settings = Settings()
```

**For SQL databases** — always use parameterised queries; never format user
input into query strings:

```python
# SQLite / SQLAlchemy style
cursor.execute("SELECT * FROM things WHERE id = ?", (thing_id,))

# psycopg2 style
cursor.execute("SELECT * FROM things WHERE id = %s", (thing_id,))
```

**For REST APIs** — pass user-controlled values only as typed arguments to
the client, never interpolated into URL strings directly:

```python
# Good — library handles encoding
response = client.get("/things", params={"id": thing_id})

# Bad — open to injection / unexpected routing
response = httpx.get(f"https://api.example.com/things/{user_input}")
```

---

## 7. Cursor-based pagination

Never use OFFSET pagination — it allows bulk extraction by incrementing the offset.
Use an opaque cursor encoding the last-seen sort key.

```python
import base64, json

def encode_cursor(payload: dict) -> str:
    return base64.urlsafe_b64encode(json.dumps(payload).encode()).decode()

def decode_cursor(cursor: str) -> dict:
    return json.loads(base64.urlsafe_b64decode(cursor.encode()))

def fetch_things(client, cursor: str | None = None, page_size: int = 50):
    # Request one extra row to detect whether there's a next page
    params: dict = {"limit": page_size + 1}
    if cursor:
        c = decode_cursor(cursor)
        params["since_id"] = c["id"]   # or a timestamp, sort key, etc.

    rows = client.get("/things", params=params).json()["items"]

    next_cursor = None
    if len(rows) > page_size:
        rows = rows[:page_size]
        next_cursor = encode_cursor({"id": rows[-1]["id"]})

    return rows, next_cursor
```

The cursor payload can encode any sort key — an ID, a timestamp, a composite
value — whatever lets your backend express "give me the next page after this
item". The client sees only the opaque base64 string.

In `_dispatch`, always include `next_cursor` in the response dict when it is not None.
The LLM will pass it back as the `cursor` argument on the next call.

---

## 8. App factory — `create_app()`

```python
from contextlib import asynccontextmanager
from typing import Any
from mcp.server.streamable_http_manager import StreamableHTTPSessionManager
from starlette.applications import Starlette
from starlette.routing import Mount, Route
from .auth import BearerAuthMiddleware
from .oauth import handle_authorize, handle_metadata, handle_token

def create_app() -> Any:
    session_manager = StreamableHTTPSessionManager(
        app=mcp,
        event_store=None,
        json_response=False,   # return raw TOML text, not JSON-wrapped
        stateless=True,
    )

    @asynccontextmanager
    async def lifespan(app: Starlette):
        async with session_manager.run():
            yield

    async def handle_mcp(scope: Any, receive: Any, send: Any) -> None:
        # Mount("/") strips the leading "/" — restore it
        if scope.get("type") == "http" and not scope.get("path"):
            scope = {**scope, "path": "/"}
        await session_manager.handle_request(scope, receive, send)

    starlette_app = Starlette(
        routes=[
            Route("/.well-known/oauth-authorization-server", endpoint=handle_metadata),
            Route("/authorize", endpoint=handle_authorize, methods=["GET", "POST"]),
            Route("/token", endpoint=handle_token, methods=["POST"]),
            Mount("/", app=handle_mcp),
        ],
        lifespan=lifespan,
    )

    return BearerAuthMiddleware(starlette_app)
```

`json_response=False` is required — it tells the session manager not to
JSON-encode the tool result string a second time.

`stateless=True` means no session resumption; each request is independent.
Appropriate for single-worker, low-concurrency MCP servers.

---

## 9. Token config

The TOML file approach works well for up to ~30 users whose credentials
change infrequently — a team, a set of trusted clients, an internal tool.
It requires no database and is trivial to audit.

```toml
# config/mcp_tokens.toml  — gitignored, never commit
[users.alice]
token       = "a64hexstring..."   # python -c "import secrets; print(secrets.token_hex(32))"
description = "Alice Smith"

[users.bob]
token       = "b64hexstring..."
description = "Bob Jones"
```

Provide a `config/mcp_tokens.toml.example` with placeholder tokens.
Add `config/mcp_tokens.toml` to `.gitignore`.

Tokens are loaded once at startup via `@lru_cache`. Restart the process to
pick up changes.

**When to move to a database-backed token store:**
- More than ~30 users, or users are added/removed frequently
- You need token revocation without a process restart
- You want per-token rate limiting, usage logging, or expiry

Replace `_load_tokens()` with a query against a `tokens` table and drop the
`@lru_cache` (or use a short TTL cache instead). The rest of `auth.py` and
the middleware are unchanged.

---

## 10. `__main__.py`

```python
# __main__.py
import uvicorn
from .server import create_app

if __name__ == "__main__":
    uvicorn.run(
        "your_mcp_server.server:create_app",
        factory=True,
        host="127.0.0.1",
        port=8001,
        reload=False,
    )
```

---

## 11. nginx configuration

```nginx
upstream your_mcp {
    server 127.0.0.1:8001;
    keepalive 8;
}

# OAuth discovery — no /mcp/ prefix, served from domain root
location = /.well-known/oauth-authorization-server {
    proxy_pass http://your_mcp;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}
location = /authorize {
    proxy_pass http://your_mcp;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}
location = /token {
    proxy_pass http://your_mcp;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}

# Redirect /mcp → /mcp/ preserving POST method
location = /mcp {
    return 308 https://$host/mcp/;
}

# MCP server — SSE requires buffering off and long timeouts
location /mcp/ {
    proxy_pass http://your_mcp/;
    proxy_buffering          off;
    proxy_cache              off;
    proxy_read_timeout       86400s;
    proxy_send_timeout       86400s;
    chunked_transfer_encoding on;
    proxy_http_version       1.1;
    proxy_set_header         Connection "";
    proxy_set_header         Host $host;
    proxy_set_header         X-Forwarded-Proto $scheme;
}
```

**Critical SSE settings:**
- `proxy_buffering off` — nginx must not buffer the event stream
- `proxy_read_timeout 86400s` — SSE connections are long-lived (idle ≠ dead)
- `proxy_http_version 1.1` + `Connection ""` — enables keep-alive to upstream
- `308` (not `301`) on the `/mcp` → `/mcp/` redirect — preserves POST method

---

## 12. systemd service

```ini
# /etc/systemd/system/your-mcp.service
[Unit]
Description=Your MCP Server
After=network.target

[Service]
Type=simple
User=your-service-user
WorkingDirectory=/opt/your-app

ExecStart=/opt/your-app/venv/bin/uvicorn \
    your_mcp_server.server:create_app \
    --factory \
    --host 127.0.0.1 \
    --port 8001 \
    --workers 1

Restart=on-failure
RestartSec=5
TimeoutStopSec=10

StandardOutput=journal
StandardError=journal
SyslogIdentifier=your-mcp

[Install]
WantedBy=multi-user.target
```

**Always use `--workers 1`.** SSE connections are long-lived; multiple workers
would spread connections across processes with no shared state.

---

## 13. Connecting from Claude.ai

**Option A — direct bearer token:**
1. Settings → Model Context Protocol → Add remote server
2. URL: `https://yourdomain.com/mcp/`
3. Custom header — Name: `Authorization`, Value: `Bearer <token>`

**Option B — OAuth flow (preferred for Claude.ai):**
1. Settings → Model Context Protocol → Add remote server
2. URL: `https://yourdomain.com/mcp/`
3. Claude.ai will detect the OAuth metadata endpoint and prompt the user
   to log in — they paste their bearer token into your `/authorize` form

---

## 14. Connecting from the Claude API

```python
import anthropic

client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-opus-4-7",
    max_tokens=4096,
    tools=[{
        "type": "mcp",
        "server": {
            "type": "url",
            "url": "https://yourdomain.com/mcp/",
            "authorization_token": "your-token",
        },
    }],
    messages=[{"role": "user", "content": "Your question here"}],
    betas=["mcp-client-2025-04-04"],
)
```

---

## 15. Data accuracy requirements for AI assistants

This server provides **live, authoritative data**. When an AI assistant is
connected to it, the following rules apply.

### Always call the tool first
Never answer a factual question from training data. Names, identifiers, ownership,
and configuration change continuously. Training data is a snapshot — it is stale
by definition. Call the tool, then answer.

### Report only what the tool returns
- Present field values exactly as returned. Do not correct, embellish, or
  restate them in your own words if doing so changes their meaning.
- If a field is absent from the result, say it is not recorded — do not infer
  or guess a value.
- If `not_found = true` is in the result, say the record does not exist — do not
  substitute a "likely" answer from memory.
- If the tool returns an error, say so. Do not substitute a "likely" answer from memory.

### Do not blend tool results with prior knowledge
A common failure mode is mixing live data with training-data assumptions — e.g.
returning the correct name from the tool but adding an incorrect affiliation from
memory. The tool result is the whole answer. Prior knowledge is not a supplement.

### Confidence is not correctness
High model confidence in a recalled fact does not make it accurate. For any
domain served by this tool, the tool is the authoritative source — not the
model's weights.

### Paginated results are incomplete by default
If the result includes `has_more = true` or a `next_cursor`, explicitly tell the
user the list is paginated and may be incomplete. Do not imply you have shown all
records.

### Follow-up questions require a new tool call
Do not answer follow-up questions ("what's their phone number?", "when did that
change?") from memory. If the initial call did not return the needed field, call
the tool again or say the data is not available from this source.

### Know the tool's scope
Each tool's description states what domain it is authoritative for. For questions
outside that domain, say the tool does not cover it — do not answer from memory
even if you have relevant training data.

---

## Checklist for a new MCP server

- [ ] Copy `auth.py` verbatim, update `_CONFIG_PATH`
- [ ] Copy `oauth.py`, edit the HTML form title/description
- [ ] Write `server.py`: `_clean`, `_dump`, tool list, `call_tool`, `_dispatch`, `create_app`
- [ ] Write your query functions (or import from existing module)
- [ ] Create `config/mcp_tokens.toml.example` with placeholder tokens
- [ ] Add `config/mcp_tokens.toml` to `.gitignore`
- [ ] Add `mcp>=1.2.0` and `tomli-w>=1.0.0` to dependencies
- [ ] Write `__main__.py`
- [ ] Add nginx location blocks (buffering off, long timeouts)
- [ ] Write systemd unit with `--workers 1`
- [ ] Generate real tokens and populate `mcp_tokens.toml`
- [ ] `systemctl enable --now your-mcp`
- [ ] Test: `curl -H "Authorization: Bearer <token>" https://yourdomain.com/mcp/`
- [ ] Tool descriptions declare the authoritative domain and end with the anti-hallucination directive
- [ ] `_dispatch` returns `not_found = true` (not an error) when a record is absent
- [ ] List tool responses include `has_more = true/false`

---

## Common mistakes

| Mistake | Fix |
|---|---|
| Using `BaseHTTPMiddleware` for auth | Use pure ASGI middleware — `BaseHTTPMiddleware` breaks SSE |
| `json_response=True` on session manager | Set `False` — otherwise TOML is double-encoded |
| `--workers 4` in uvicorn | Use `--workers 1` for SSE |
| Missing `proxy_buffering off` in nginx | SSE will hang or buffer until connection closes |
| Short `proxy_read_timeout` | Idle SSE connections will be killed by nginx |
| `308` omitted on `/mcp` → `/mcp/` redirect | `301`/`302` downgrades POST to GET, breaking MCP handshake |
| OFFSET-based pagination | Use cursor tokens — OFFSET enables bulk extraction |
| Interpolating user input into SQL/URLs | Use parameterised queries or typed client methods |
| Committing `mcp_tokens.toml` | Keep gitignored; use `.example` file instead |
| Forgetting to restart after token changes | Tokens are `lru_cache`d — restart process to reload |
| `None → ""` in `_clean` (old pattern) | Omit absent fields — the AI can't distinguish "not recorded" from empty string |
| Missing `has_more` in list responses | Always include it so the AI knows whether the result is complete |
| Returning `error` for not-found records | Use `not_found = true` — the AI should say "doesn't exist", not "something went wrong" |
| Tool description omits authoritative domain | State it explicitly — the AI needs to know when not to fall back on training data |
