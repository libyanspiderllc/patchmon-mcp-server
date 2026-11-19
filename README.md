# PatchMon MCP Server

An MCP (Model Context Protocol) server that connects to a [PatchMon](https://patchmon.net)
instance via its Integration API, so an LLM client (Claude Desktop, Claude
Code, etc.) can read host inventory, patch status, and system details.

Built against PatchMon's documented Integration API:
https://patchmon.net/docs/patchmon-api-integrations-guide/integration-api-documentation

Uses the standalone [`fastmcp`](https://github.com/jlowin/fastmcp) library
(v2.x) to implement the MCP server. This is a separate project from, but
protocol-compatible with, Anthropic's official `mcp` Python SDK — any MCP
client (Claude Desktop, Claude Code, MCP Inspector) works with it
identically.

## Tools exposed

| Tool | Scope needed | Description |
|---|---|---|
| `list_hosts` | `host:get` | List all hosts, optionally filtered by host group, optionally with inline stats |
| `get_host_stats` | `host:get` | Package/repo statistics for one host |
| `get_host_info` | `host:get` | OS, agent version, host groups for one host |
| `get_host_network` | `host:get` | IP, gateway, DNS, interfaces |
| `get_host_system` | `host:get` | CPU/RAM/disk/kernel/uptime/reboot status |
| `get_host_packages` | `host:get` | Installed packages, optionally updates-only |
| `get_host_package_reports` | `host:get` | Agent check-in report history |
| `get_host_agent_queue` | `host:get` | Agent job queue + job history |
| `get_host_notes` | `host:get` | Free-text notes on a host |
| `get_host_integrations` | `host:get` | Integration status (e.g. Docker) |
| `get_host_overview` | `host:get` | Combined info + stats + system in one call |
| `find_hosts_needing_security_updates` | `host:get` | Hosts with pending security updates |

## 1. Create a PatchMon API credential

1. Log in to your PatchMon instance as an administrator.
2. Go to **Settings → Integrations → Auto-Enrollment & API**.
3. Click **New Token**, choose usage type **API**.
4. Grant only the `host:get` scope.
5. Copy the **Token Key** (`patchmon_ae_...`) and **Token Secret** — the
   secret is shown only once.

## 2. Install dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## 3. Configure

The server reads its configuration from environment variables:

| Variable | Required | Default | Description |
|---|---|---|---|
| `PATCHMON_URL` | yes | — | Base URL of your PatchMon instance, e.g. `https://patchmon.example.com` |
| `PATCHMON_API_KEY` | yes | — | Token key (`patchmon_ae_...`) |
| `PATCHMON_API_SECRET` | yes | — | Token secret |
| `PATCHMON_API_VERSION` | no | `v1` | API version path segment |
| `PATCHMON_VERIFY_SSL` | no | `true` | Set to `false` to skip TLS verification (dev/self-signed only) |
| `PATCHMON_TIMEOUT` | no | `30` | Request timeout in seconds |
| `MCP_AUTH_TOKEN` | for HTTP | — | Shared bearer token for MCP clients |
| `HOST` | no | `127.0.0.1` | Standalone HTTP bind address or Compose host interface |
| `MCP_PORT` | no | `8000` | Host port published by Docker Compose |

## 4. Run standalone (for testing)

```bash
export PATCHMON_URL="https://patchmon.example.com"
export PATCHMON_API_KEY="patchmon_ae_abc123"
export PATCHMON_API_SECRET="your_secret_here"
python server.py
```

The script defaults to stdio for clients that launch it as a subprocess.

To run Streamable HTTP directly, export a separate `MCP_AUTH_TOKEN` and start
the server. Generate a token with `openssl rand -hex 32` and save it for your MCP
client.

```bash
export MCP_AUTH_TOKEN="<your-generated-token>"
python server.py --transport http
```

Configure your MCP client to use Streamable HTTP at `http://127.0.0.1:8000/mcp`.
Set its `Authorization` header to `Bearer <your-generated-token>`. The server
rejects requests with a missing or invalid token. Use `--host` to override
`HOST` and `--port` to change the standalone listening port.

## Run with Docker Compose

1. Copy the configuration template and fill in the PatchMon HTTPS URL and
   credentials for a dedicated `host:get` token. Set `MCP_AUTH_TOKEN` to a
   separate secret generated with `openssl rand -hex 32`.

   ```bash
   cp .env.example .env
   chmod 600 .env
   ```

2. Build and start the server.

   ```bash
   docker compose up --build -d
   ```

3. Set your MCP client's Streamable HTTP URL to `http://127.0.0.1:8000/mcp`
   and its `Authorization` header to `Bearer <your-generated-token>`. View
   logs with `docker compose logs -f patchmon-mcp` and stop with
   `docker compose down`.

Set `HOST` in `.env` to the Docker host IP to publish on. The default
`127.0.0.1` allows connections from the Docker host, and `0.0.0.0` publishes
on all IPv4 interfaces. Set `MCP_PORT` to change the published port. For remote
access, use an HTTPS reverse proxy that forwards the `Authorization` header
so the bearer token is encrypted in transit.

The PatchMon URL must be reachable from inside the container. Use a DNS
name reachable from Docker, or use `host.docker.internal` on Docker Desktop
for a service running on the host.

## 5. Connect it to Claude Desktop

Add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "patchmon": {
      "command": "/absolute/path/to/.venv/bin/python",
      "args": ["/absolute/path/to/server.py"],
      "env": {
        "PATCHMON_URL": "https://patchmon.example.com",
        "PATCHMON_API_KEY": "patchmon_ae_abc123",
        "PATCHMON_API_SECRET": "your_secret_here"
      }
    }
  }
}
```

Restart Claude Desktop and the `patchmon` tools will be available.

## 6. Connect it to Claude Code

```bash
claude mcp add patchmon \
  --env PATCHMON_URL=https://patchmon.example.com \
  --env PATCHMON_API_KEY=patchmon_ae_abc123 \
  --env PATCHMON_API_SECRET=your_secret_here \
  -- /absolute/path/to/.venv/bin/python /absolute/path/to/server.py
```

## Notes on safety

- Use a dedicated `host:get` token and keep TLS verification enabled.
- Credentials are read from environment variables only — never hard-code
  them into the server or commit them to version control.
- Consider restricting the PatchMon API credential to specific IPs and
  setting an expiration date (see PatchMon's Security Best Practices).
