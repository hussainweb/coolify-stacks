# Agent Guidelines for Coolify Stacks

This repository is the centralized catalog of public, production-tested [Coolify v4](https://coolify.io/) Docker Compose stacks.

All AI agents working in this repository MUST strictly adhere to these conventions when creating, modifying, or maintaining stacks.

---

## 🏗️ Adding a New Stack Checklist

When adding a new stack (`stacks/<stack-name>/`), complete all of the following:

1. **Stack Directory Layout**:
   - `stacks/<stack-name>/docker-compose.yaml` (Coolify production definition)
   - `stacks/<stack-name>/docker-compose.test.yaml` (Local and CI port overrides)
   - `stacks/<stack-name>/README.md` (Detailed deployment, environment, and client guide)

2. **Compose Standards (Coolify Best Practices)**:
   - **Never publish `ports:` in `docker-compose.yaml`**: Coolify reverse proxies (Traefik/Caddy) route directly to internal container ports on Coolify's managed network. Put host port exposures exclusively in `docker-compose.test.yaml`.
   - **Never define a custom `networks:` block**: Coolify isolates and manages container networks automatically.
   - **Always pin specific image tags**: Never use `:latest`. Pin to a specific semver release tag (e.g. `3.2.1`) so Dependabot can track updates.
   - **Prevent non-TTY / stdio traps**: If the container auto-detects TTY vs stdio (e.g., CLI tools or MCP servers), explicitly force server/HTTP mode via `command:` or environment variables (e.g. `command: ["server", "--protocol", "http"]`).
   - **Operator-tunable environment variables**: Format every configurable option as `${VAR:-default}` or `${VAR:-}` so Coolify presents them as editable fields in the UI.
   - **Reliable in-container healthchecks**: Use tools guaranteed to exist in the image (e.g. Node one-liner `node -e "..."` or `wget --spider`), avoiding tools like `curl` which are missing from slim/distroless bases.
   - **Named persistent volumes**: Name volumes plainly (e.g. `<stack>-data:/data`); Coolify prefixes them with the resource UUID automatically.

3. **CI/CD Workflows**:
   - **Stack Workflow** (`.github/workflows/stack-<stack-name>.yml`):
     - Single workflow per stack containing both `test` and `deploy` jobs.
     - Triggers on `push` to `main`, `pull_request` to `main`, and `workflow_dispatch`.
     - Path filtering covers:
       - `stacks/<stack-name>/**`
       - `.github/workflows/stack-<stack-name>.yml`
       - `.github/actions/test-stack/**`
       - `.github/actions/deploy-stack/**`
     - **`test` Job**: Runs on both PR and push. Checks out the repository and invokes `./.github/actions/test-stack` supplying:
       - `stack_dir: stacks/<stack-name>`
       - `test_port: '<port>'`
       - `test_path: '/'` (or specific endpoint)
     - **`deploy` Job**: Runs only on push to `main` (`if: github.ref == 'refs/heads/main'`), depends on `test` (`needs: test`), targets `environment: production`, and invokes `./.github/actions/deploy-stack` supplying:
       - `coolify_base_url: ${{ secrets.COOLIFY_BASE_URL }}`
       - `coolify_api_token: ${{ secrets.COOLIFY_API_TOKEN }}`
       - `coolify_resource_uuid: '<uuid>'`

4. **Dependabot Updates**:
   - Add an entry in `.github/dependabot.yml`:
     ```yaml
     - package-ecosystem: "docker-compose"
       directory: "/stacks/<stack-name>"
       schedule:
         interval: "weekly"
     ```

5. **Catalog Registration**:
   - Register the new stack in root `README.md` under the **Available Stacks** table.

6. **Local Verification**:
   - Always run `mise run validate` and `mise run test` before committing.

---

## 🔒 Privacy & Secret Management

- **Public Repository Rule**: This repository is public. **NEVER** store private/internal hostnames, domains (e.g. `*.internal`, `*.home.arpa`), or IP addresses in GitHub Actions Variables (`vars.*`) or committed workflow files.
- **Coolify Secrets**:
  - `COOLIFY_BASE_URL`: Always stored as an encrypted GitHub Secret (automatically masked in logs).
  - `COOLIFY_API_TOKEN`: Stored as an encrypted GitHub Secret with deploy permissions.
- **Resource UUIDs**: Stored in the stack caller workflow (`coolify_resource_uuid: '...'`). A UUID without the base URL and API token exposes nothing sensitive.
