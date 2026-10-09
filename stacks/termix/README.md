# Termix on Coolify

This stack runs [Termix](https://github.com/termix-ssh/termix) alongside an [Apache Guacamole daemon (`guacd`)](https://guacamole.apache.org/) sidecar.

Termix is a modern, self-hosted, web-based server management platform providing:
- **SSH Terminals & Tunnels**: Tabbed/split-screen terminal sessions, SFTP file manager with direct server-to-server transfers, and dynamic SSH/SOCKS tunnels.
- **Remote Desktop**: In-browser RDP, VNC, and Telnet powered by the bundled `guacd` sidecar.
- **Fleet Management & Hardware Telemetry**: Live CPU, memory, disk, network, and GPU metrics across servers, plus scheduled automation routines.
- **Docker & Podman Management**: Container lifecycle controls, real-time container stats, and terminal attachment.
- **Comprehensive Authentication**: Local accounts, 2FA (TOTP), Passkeys (WebAuthn), OIDC/OAuth (GitHub, Google, Keycloak, Authentik), LDAP, and Trusted Proxy Authentication.

---

## 🚀 Coolify Deployment Guide

### 1. Add Resource
1. In your Coolify dashboard, navigate to your **Project** and **Environment**.
2. Click **+ New Resource** → **Public Repository**.
3. Enter `https://github.com/hussainweb/coolify-stacks` and select branch `main`.
4. Select **Docker Compose** as the build pack.

### 2. Configure Paths
Under **General** settings:
- **Base Directory**: `/stacks/termix`
- **Docker Compose Location**: `docker-compose.yaml`
- **Watch Paths**: `/stacks/termix/**`

> **Note**: Setting **Watch Paths** is essential in a monorepo setup. It ensures Coolify only rebuilds or redeploys this application when files inside `/stacks/termix/` change.

### 3. Domains & Reverse Proxy
1. In the **Domains** field, set your domain (e.g., `https://termix.example.com`).
2. Coolify's reverse proxy (Traefik or Caddy) routes traffic directly to internal port `8080`.
3. **WebSockets**: Termix terminal sessions, Guacamole video streams, and real-time telemetry require WebSocket support. Both Traefik and Caddy in Coolify support WebSockets natively without extra routing rules.

### 4. Storage & Persistence
The stack defines a named persistent volume:
- `termix-data` (mounted to `/app/data`): Persists the SQLite database, generated encryption keys, SSH configurations, host profiles, and SSL certificates.

---

## 🔐 Authentication Guide

Termix provides extensive enterprise-grade and homelab authentication options:

### 1. Local Authentication & MFA
- **Password Accounts**: Local username and password authentication with secure password hashing.
- **Two-Factor Authentication (2FA)**: Time-based One-Time Passwords (TOTP) supported via standard authenticator apps (e.g. 1Password, Bitwarden, Google Authenticator).
- **Passkeys (FIDO2 / WebAuthn)**: Native passwordless authentication using hardware keys (YubiKey), Touch ID, Windows Hello, or platform authenticators.
- **Registration Control**: Registration can be locked or controlled via `ALLOW_REGISTRATION=false` once your administrative account is set up.

### 2. OpenID Connect (OIDC) & Social SSO
Termix supports standard OIDC providers (Authentik, Keycloak, Authelia, Okta) as well as direct GitHub and Google OAuth:
- Configure providers via the web UI under **Admin Settings → SSO Providers**, or bootstrap via environment variables.
- Set `OIDC_FORCE_HTTPS=true` (enabled by default in this stack) when deploying behind Coolify's HTTPS reverse proxy so redirect callbacks match your public URL.
- Group-to-role mappings (`OIDC_ROLE_MAP` and `OIDC_ADMIN_GROUP`) allow automatically assigning user roles based on IDP group claims.

### 3. LDAP / Active Directory
- Connect to self-hosted LDAP or Active Directory servers to authenticate users against directory services.

### 4. Trusted Proxy Authentication (Forward Auth)
If you place Termix behind an identity-aware proxy (Authentik, Authelia, Cloudflare Access, or Tailscale):
- Set `TRUSTED_PROXY_AUTH_ENABLED=true`.
- Specify trusted proxy IP ranges via `TRUSTED_PROXY_AUTH_TRUSTED_PROXIES` (e.g., your Coolify proxy network range).
- Header mapping (`TRUSTED_PROXY_AUTH_USERNAME_HEADER` and `TRUSTED_PROXY_AUTH_ROLE_HEADER`) maps proxy identity headers to Termix users and roles.

---

## ⚙️ Environment Variables

Configure these in Coolify's **Environment Variables** tab as needed:

| Variable | Description | Default |
| :--- | :--- | :--- |
| `PORT` | Web interface internal port | `8080` |
| `PUID` | User ID for container execution | `1000` |
| `PGID` | Group ID for container execution | `1000` |
| `ENABLE_GUACAMOLE` | Enable Remote Desktop (RDP/VNC/Telnet) support | `true` |
| `GUACD_HOST` | Hostname of the Guacamole daemon sidecar | `guacd` |
| `GUACD_PORT` | Port of the Guacamole daemon sidecar | `4822` |
| `GUACD_TUNNEL_HOST` | Hostname guacd uses to reach Termix for jump tunnels | `termix` |
| `OIDC_FORCE_HTTPS` | Enforce HTTPS scheme in OIDC callback URLs | `true` |
| `ALLOW_REGISTRATION` | Toggle public user registration (`true` / `false`) | UI controlled |
| `ENABLE_TELEMETRY` | Daily anonymous heartbeat (`false` to disable) | `true` |
| `DATABASE_DIALECT` | Database backend (`sqlite`, `postgres`, `mysql`) | `sqlite` |
| `DATABASE_URL` | External DB connection string (when using Postgres/MySQL) | None |

> **Auto-Generated Secrets**:
> On first boot, Termix automatically derives and writes internal security keys (`JWT_SECRET`, `DATABASE_KEY`, `ENCRYPTION_KEY`) into `/app/data/.env`. These remain persistent across container updates within the `termix-data` volume.

---

## 🧪 Local Verification

To test the stack locally using Docker Compose:

```bash
# Validate Compose syntax
mise run validate

# Run Compose config validation including test port override
mise run test

# Launch locally with host port 8080 published
docker compose -f stacks/termix/docker-compose.yaml -f stacks/termix/docker-compose.test.yaml up -d

# Smoke test health endpoint
curl -fsS http://localhost:8080/health

# Stop and clean up
docker compose -f stacks/termix/docker-compose.yaml -f stacks/termix/docker-compose.test.yaml down -v
```
