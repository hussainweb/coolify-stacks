# Coolify Stacks

Curated, production-tested [Coolify v4](https://coolify.io/) Docker Compose stacks for self-hosted tools and services.

This repository serves as a centralized monorepo for self-hosted service definitions, automated CI validation, and deploy triggers.

---

## 📦 Available Stacks

| Stack | Description | Upstream Image | Documentation |
| :--- | :--- | :--- | :--- |
| **`docs-mcp`** | Grounded Tools Docs MCP server (documentation fetch & search) | `ghcr.io/arabold/docs-mcp-server:latest` | [docs-mcp README](stacks/docs-mcp/README.md) |

---

## 🚀 How to Deploy in Coolify

Coolify supports deploying multiple distinct resources from a single Git repository by scoping each resource to a subfolder:

1. **Add a Resource in Coolify**:
   - Go to your Project / Environment.
   - Click **+ New Resource** → **Public Repository** (or **Private Repository**).
   - Enter `https://github.com/hussainweb/coolify-stacks` and branch `main`.
   - Choose **Docker Compose**.

2. **Configure General Settings**:
   - **Base Directory**: `/stacks/<stack-name>` (e.g. `/stacks/docs-mcp`).
   - **Docker Compose Location**: `docker-compose.yaml` (relative to the Base Directory).

3. **Configure Watch Paths (Crucial for Monorepo)**:
   - In the application's **General** settings, set **Watch Paths**:
     ```text
     /stacks/<stack-name>/**
     ```
   - This ensures Coolify will only rebuild or redeploy this specific service when changes are pushed to its directory.

4. **Domains & Ingress**:
   - Add your domain in Coolify (e.g., `https://docs-mcp.example.com`).
   - For proxied compose services, Coolify's reverse proxy (Traefik/Caddy) routes directly to the internal container port on the generated network. Do not expose host ports directly in the primary compose file.

---

## 🔄 CI/CD & Automated Deployments

Automated deployments are driven via GitHub Actions and Coolify's deploy webhook API (`POST /api/v1/deploy`):

- **Validation**: On every push and pull request, GitHub Actions validates Docker Compose syntax (`docker compose config`) and executes container smoke tests.
- **Path-filtered Deployments**: Pushes to `main` modifying files under `stacks/<stack-name>/**` trigger deployment specifically for that stack.
- **GitHub Secrets & Environments Setup**:
  1. Add `COOLIFY_API_TOKEN` as a **Repository Secret** (Settings → Secrets and variables → Actions → Repository secrets).
  2. Create a GitHub Environment for each stack (Settings → Environments → New environment, e.g. `docs-mcp`).
  3. Inside each environment, create the environment-scoped secret `COOLIFY_WEBHOOK_URL` containing the deploy webhook URL from Coolify (**Webhooks** tab → **Deploy Webhook**).
  4. Workflows automatically inherit the repo-wide token and pull the stack-specific webhook URL.

---

## 🛠️ Local Development

This repository uses [mise](https://mise.jdx.dev/) for local tasks:

```bash
# Validate compose syntax across all stacks
mise run validate

# Test compose configurations including test overrides
mise run test
```
