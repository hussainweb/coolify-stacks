# Grounded Tools Docs MCP Server on Coolify

This stack runs [Grounded Tools Docs MCP Server](https://github.com/arabold/docs-mcp-server) (`@arabold/docs-mcp-server`), an MCP server that downloads, indexes, and provides searchable documentation to AI agents (Claude, Cursor, Cline, etc.).

When running in server mode, it provides:
- **Web Dashboard**: Browse and manage indexed documentation at `http://<your-domain>/`.
- **MCP SSE Server**: Connect AI assistants over Server-Sent Events at `http://<your-domain>/sse`.

---

## 🚀 Coolify Deployment Guide

### 1. Add Resource
1. In your Coolify dashboard, navigate to your Project and Environment.
2. Click **+ New Resource** → **Public Repository**.
3. Enter `https://github.com/hussainweb/coolify-stacks` and select branch `main`.
4. Select **Docker Compose** as the build pack.

### 2. Configure Paths
Under the **General** settings:
- **Base Directory**: `/stacks/docs-mcp`
- **Docker Compose Location**: `docker-compose.yaml`
- **Watch Paths**: `/stacks/docs-mcp/**`

> **Note**: Setting **Watch Paths** is essential in a monorepo setup. It ensures Coolify only triggers rebuilds for this application when files inside `/stacks/docs-mcp/` change.

### 3. Domains & SSL
1. In the **Domains** field, set your domain (e.g. `https://docs-mcp.example.com`).
2. Coolify's reverse proxy will route incoming traffic to container port `6280` automatically.
3. Save changes.

### 4. Storage & Persistence
The stack defines two persistent Docker volumes:
- `docs-mcp-data` (mounted to `/data`): Stores the SQLite search index, embeddings, and scraped document cache.
- `docs-mcp-config` (mounted to `/config`): Stores server configuration and settings.

### 5. Environment Variables
Configure any needed environment variables in Coolify's **Environment Variables** tab:

| Variable | Description | Default |
| :--- | :--- | :--- |
| `HOST` | Bind address | `0.0.0.0` |
| `PORT` | Listening port | `6280` |
| `DOCS_MCP_PROTOCOL` | Server protocol (`http` or `stdio`) | `http` |
| `DOCS_MCP_STORE_PATH` | Path for data index and cache | `/data` |
| `DOCS_MCP_SERVER_PUBLIC_URL` | Absolute public URL (e.g. `https://docs-mcp.example.com`). Required behind reverse proxies for Host validation & OAuth | None |
| `DOCS_MCP_SERVER_ALLOWED_ORIGINS` | Comma-separated list of additional allowed browser origins/hostnames | None |
| `DOCS_MCP_AUTH_ENABLED` | Enable OAuth2/OIDC authentication (`true` or `false`) | `false` |
| `DOCS_MCP_AUTH_ISSUER_URL` | Issuer / discovery URL of OIDC provider (e.g. `https://accounts.google.com`) | None |
| `DOCS_MCP_AUTH_AUDIENCE` | Expected JWT audience claim (e.g. `https://docs-mcp.example.com/mcp`) | None |
| `DOCS_MCP_SERVER_PUBLIC_ORIGIN` | *(Legacy alias)* External domain origin advertised in OAuth metadata | None |
| `OPENAI_API_BASE` | OpenAI-compatible API base URL (e.g. `http://ollama:11434/v1`) | None |
| `DOCS_MCP_EMBEDDING_MODEL` | Embedding model (e.g. `nomic-embed-text:latest`, `gemini:text-embedding-004`) | None (BM25 search) |
| `OPENAI_API_KEY` | API key (set to `ollama` for local Ollama) | `ollama` |
| `GOOGLE_API_KEY` | *(Optional)* Google Gemini API key | None |
| `ANTHROPIC_API_KEY` | *(Optional)* Anthropic API key | None |

> **Host Filtering & DNS Rebinding Protection (HTTP 403)**:
> In recent versions, `docs-mcp-server` enforces strict host header validation to guard against DNS rebinding. Any incoming request whose `Host` header does not match localhost/loopback, internal container names, `DOCS_MCP_SERVER_PUBLIC_URL`, or `DOCS_MCP_SERVER_ALLOWED_ORIGINS` will be rejected with **HTTP 403 Forbidden**.
> - **Behind a Reverse Proxy (Coolify / Traefik)**: Always set `DOCS_MCP_SERVER_PUBLIC_URL=https://<your-domain>` (e.g. `https://docs-mcp.example.com`) so the server permits requests arriving with your public domain in the `Host` header.
> - **Multiple / Alternate Domains**: Add any additional hostnames (e.g. `http://nas.local:6280`) to `DOCS_MCP_SERVER_ALLOWED_ORIGINS` as a comma-separated list.

> **Authentication (OAuth2 / OIDC) & Forward-Auth**:
> The Docs MCP server can act as an RFC 6749 / RFC 7591 OAuth 2.0 Resource Server:
> - **Scope of Built-in Auth**: Enabling `DOCS_MCP_AUTH_ENABLED=true` strictly validates Bearer tokens on the MCP HTTP endpoint (`/mcp`). It does **not** protect the web UI dashboard at `/` or the internal tRPC `/api` endpoints.
> - **Why TinyAuth Cannot Act as OIDC Issuer**: TinyAuth is a forward-auth proxy, not a full OIDC Identity Provider. It lacks an OIDC discovery endpoint (`/.well-known/openid-configuration`), Dynamic Client Registration (RFC 7591), and JWKS cryptographic endpoints. Pointing `DOCS_MCP_AUTH_ISSUER_URL` to TinyAuth will not work.
> - **Why Forward-Auth Reverse Proxies Break AI Clients**: Protecting the entire domain with TinyAuth (or Cloudflare Access / Authelia) sends an HTTP 302 redirect to an HTML login page when a client is unauthenticated. Non-browser AI clients (Claude Desktop, Cursor, CLI agents) do not have browser rendering engines and immediately fail when encountering HTML on `/mcp`.
> - **Recommended Strategies**:
>   1. **Private Network / Tailscale (Recommended)**: Serve Docs MCP over a private internal domain or Tailscale with `DOCS_MCP_AUTH_ENABLED=false`. Network access control protects both the Web UI and MCP endpoint without auth hurdles for AI agents.
>   2. **Reverse Proxy Path Split**: Protect `/` and `/api/*` with TinyAuth forward-auth or HTTP Basic Auth, while exempting `/mcp` (or protecting `/mcp` with token headers).
>   3. **External OIDC Provider**: Configure a true OIDC provider (Google OIDC or Keycloak) with `DOCS_MCP_AUTH_ENABLED=true` for clients supporting OAuth 2.1 code flows.

> **Using with Local Ollama**:
> To enable semantic vector embeddings using your homelab Ollama:
> 1. Set `OPENAI_API_BASE=http://<ollama-ip>:11434/v1` (point to Ollama's OpenAI-compatible `/v1` endpoint).
> 2. Set `DOCS_MCP_EMBEDDING_MODEL=nomic-embed-text:latest` (or any model pulled on your Ollama instance).
> 3. Leave `OPENAI_API_KEY=ollama` (required as non-empty by client validation).

> **Using with Google Gemini**:
> 1. Provide your `GOOGLE_API_KEY`.
> 2. Set `DOCS_MCP_EMBEDDING_MODEL=gemini:text-embedding-004` (or `gemini:embedding-001`).

> **How Provider Selection Works with Multiple Keys**:
> Setting credentials for multiple providers (e.g., having both `OPENAI_API_KEY` and `GOOGLE_API_KEY`) does **not** cause conflicts. The active provider is determined exclusively by the model prefix in **`DOCS_MCP_EMBEDDING_MODEL`**:
> - `gemini:...` routes to Google Gemini (reads `GOOGLE_API_KEY`).
> - `openai:...` or untagged local models (like `nomic-embed-text`) route to the OpenAI-compatible endpoint (reads `OPENAI_API_BASE` and `OPENAI_API_KEY`).
> - If `DOCS_MCP_EMBEDDING_MODEL` is left empty, it defaults to OpenAI's `text-embedding-3-small` (or full-text BM25 search if no keys are set).





---

## 🔌 Connecting MCP Clients

Once deployed and running behind HTTPS, you can connect your MCP clients using the Server-Sent Events (SSE) transport.

### Claude Desktop (`claude_desktop_config.json`)
```json
{
  "mcpServers": {
    "docs": {
      "url": "https://docs-mcp.example.com/sse"
    }
  }
}
```

### Cursor / Cline
- Transport Type: `SSE`
- Server URL: `https://docs-mcp.example.com/sse`

---

## 🧪 Local Verification

To run and test the stack locally on your workstation:

```bash
# Validate compose syntax
docker compose -f docker-compose.yaml config

# Start container locally with port 6280 published
docker compose -f docker-compose.yaml -f docker-compose.test.yaml up -d

# Check health
curl -fsS http://localhost:6280/

# Stop container
docker compose -f docker-compose.yaml -f docker-compose.test.yaml down
```
