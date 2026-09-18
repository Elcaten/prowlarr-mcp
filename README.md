# prowlarr-mcp

Part of the [arr-mcps](https://github.com/arr-mcps/arr-mcps) collection.
MCP server exposing [Prowlarr](https://github.com/Prowlarr/Prowlarr)'s API v1
(REST) as tools, so an LLM can manage your indexers, applications, download
clients, indexer proxies, notifications, tags, and custom filters — plus run
cross-indexer searches and issue system commands. The full API surface is
covered, **read and write**.

Built with [FastMCP](https://gofastmcp.com).

## Enabling the API on your Prowlarr server

Generate an API key in Prowlarr **Settings > General** (Authentication section).
Every `/api/v1` endpoint requires it, sent as the `X-Api-Key` header.

## Install

Download a wheel from the [latest release](https://github.com/arr-mcps/prowlarr-mcp/releases/latest)
and install it as a `uv` tool (no repo checkout needed):

```bash
uv tool install prowlarr_mcp-*.whl
```

This puts a `prowlarr-mcp` command on your PATH. Register it with Claude Code:

```bash
claude mcp add prowlarr \
  --env PROWLARR_URL=http://localhost:9696 \
  --env PROWLARR_API_KEY=<key> \
  -- prowlarr-mcp
```

### Docker

Images are published to GHCR on push to `main` (`:latest`) and on `v*` tags
(`:0.2.1`, `:0.2`). The container speaks HTTP on `0.0.0.0:8080/mcp`. Do not set
`FASTMCP_HOST=127.0.0.1` in the container — that would only listen inside it.

```bash
docker run --rm -p 127.0.0.1:8080:8080 \
  -e PROWLARR_URL=http://prowlarr:9696 \
  -e PROWLARR_API_KEY=<key> \
  -e PROWLARR_MCP_TOKEN=<mcp-token> \
  ghcr.io/arr-mcps/prowlarr-mcp:latest
```

Or with [compose.yaml](compose.yaml) (binds `127.0.0.1:8080` so Caddy on the
host can proxy it without publishing to the internet):

```bash
cp .env.example .env   # set PROWLARR_URL, PROWLARR_API_KEY, PROWLARR_MCP_TOKEN
docker compose up -d
```

```yaml
services:
  prowlarr-mcp:
    image: ghcr.io/arr-mcps/prowlarr-mcp:latest
    restart: unless-stopped
    env_file: .env
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      FASTMCP_TRANSPORT: http
      FASTMCP_HOST: 0.0.0.0
      FASTMCP_PORT: "8080"
```

Optional: `PROWLARR_MCP_GROUPS=prowlarr_search,prowlarr_history` in `.env`. The
first image from a public repo may stay private on GHCR until you set the
package public.

### From source

```bash
uv sync
cp .env.example .env   # fill in PROWLARR_URL and PROWLARR_API_KEY
```

```bash
claude mcp add prowlarr \
  --env PROWLARR_URL=http://localhost:9696 \
  --env PROWLARR_API_KEY=<key> \
  -- uv run --directory /path/to/prowlarr-mcp prowlarr-mcp
```

## Config

| Env var | Required | Default |
|---|---|---|
| `PROWLARR_URL` | yes | - |
| `PROWLARR_API_KEY` | yes* | none (no auth header sent if unset) |
| `PROWLARR_MCP_TOKEN` | when HTTP | - |
| `PROWLARR_MCP_GROUPS` | no | all groups |
| `FASTMCP_TRANSPORT` | no | `stdio` |
| `FASTMCP_HOST` | no | `127.0.0.1` |
| `FASTMCP_PORT` | no | `8000` |

\* Every Prowlarr API endpoint requires auth; practically you must set
`PROWLARR_API_KEY`, but the server still starts without one so errors
surface from the API rather than at startup.

`PROWLARR_MCP_TOKEN` is a separate layer: FastMCP bearer auth on the MCP
HTTP endpoint (`Authorization: Bearer <token>`). It is required when
`FASTMCP_TRANSPORT` is `http`, `sse`, or `streamable-http`, and ignored
for stdio (local `claude mcp add` keeps working without it). HTTP listens
at `http://$FASTMCP_HOST:$FASTMCP_PORT/mcp`. A FastMCP client passes the
token as `auth="<token>"` (no `Bearer ` prefix).

`PROWLARR_MCP_GROUPS` limits which portmanteau tools are registered, e.g.
`prowlarr_search,prowlarr_history`. Unset or empty registers all 10 groups.

```bash
FASTMCP_TRANSPORT=http \
FASTMCP_HOST=127.0.0.1 \
FASTMCP_PORT=8000 \
PROWLARR_MCP_TOKEN=<mcp-token> \
PROWLARR_URL=http://localhost:9696 \
PROWLARR_API_KEY=<key> \
  prowlarr-mcp
```

## Tools

**10 resource-scoped tools**, each covering multiple Prowlarr API v1
operations (116 total) via an `operation` parameter. Call a tool with
`operation` set to one of its listed operations and an `arguments` dict
matching that operation's parameters — the tool's own description
(visible to your MCP client) lists every operation, its signature, and a
one-line doc. This keeps the full API surface available while costing a
fraction of the context budget of registering all 116 operations as
separate tools.

| Tool | Operations | Kind |
|---|---|---|
| `prowlarr_system` | 27 | reads + writes |
| `prowlarr_applications` | 17 | reads + writes |
| `prowlarr_indexers` | 14 | reads + writes |
| `prowlarr_config` | 13 | reads + writes |
| `prowlarr_download_clients` | 11 | reads + writes |
| `prowlarr_indexer_proxies` | 9 | reads + writes |
| `prowlarr_notifications` | 9 | reads + writes |
| `prowlarr_tags` | 7 | reads + writes |
| `prowlarr_search` | 6 | reads + writes |
| `prowlarr_history` | 3 | read-only |

Example: `prowlarr_indexers(operation="prowlarr_get_indexer", arguments={"id": 3})`.
Operation-level naming (`prowlarr_<verb>_<resource>`) is preserved as the
`operation` value, so the full operation list is still discoverable from each
group tool's description at runtime.

## Development

```bash
make help  # list all commands
```

| Command | Does |
|---|---|
| `make sync` | `uv sync` |
| `make test` | Offline tests - one per endpoint, mocked HTTP |
| `make test-integration` | Tests against the live instance (needs `PROWLARR_URL`/`PROWLARR_API_KEY`) |
| `make build` | Build wheel + sdist into `dist/` |
| `make bump-patch` / `bump-minor` / `bump-major` | Bump the version in `pyproject.toml` + `uv.lock` |
| `make clean` | Remove build artifacts |

The release workflow (`.github/workflows/release.yml`) builds and publishes to
[Releases](https://github.com/arr-mcps/prowlarr-mcp/releases) whenever a `v*`
tag is pushed - so the usual flow is `make bump-patch`, commit, then tag and
push.

The integration suite is read-only by default (a plain `make test-integration`
never mutates your instance). Setting `PROWLARR_ALLOW_WRITES=1` additionally
runs a scratch Tag create+delete round-trip.
