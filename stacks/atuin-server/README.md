# Atuin Server on Coolify

This stack runs an [Atuin](https://atuin.sh/) sync server (`atuin-server`), enabling encrypted shell history synchronization and backup across all your terminals and workstations.

Atuin replaces your standard shell history with an SQLite database and records additional context (command duration, exit code, directory, hostname, timestamp). The self-hosted sync server provides:
- **End-to-End Encryption**: All history records are encrypted on the client side with a private encryption key before transmission; the server never sees raw commands.
- **Multi-Device Synchronization**: Instant background sync across macOS, Linux, and Windows shells (Bash, Zsh, Fish, Nushell).
- **Stateless & Scalable**: Lightweight Rust-based server supporting PostgreSQL or SQLite backends.
- **Full History Search**: Fast fuzzy search and filtering across all machines from your terminal.

---

## 🚀 Coolify Deployment Guide

### 1. Add Resource
1. In your Coolify dashboard, navigate to your **Project** and **Environment**.
2. Click **+ New Resource** → **Public Repository**.
3. Enter `https://github.com/hussainweb/coolify-stacks` and select branch `main`.
4. Select **Docker Compose** as the build pack.

### 2. Configure Paths & Settings
Under **General** settings:
- **Base Directory**: `/stacks/atuin-server`
- **Docker Compose Location**: `docker-compose.yaml`
- **Watch Paths**: `/stacks/atuin-server/**`
- **Preserve Repository**: Enable **"Preserve repository during deployment"**. This is critical because the stack mounts `./config` from the cloned repository. If left unchecked, Coolify removes the cloned repository after building, causing the bind mount to be empty and Atuin to fail.

Under **Advanced** settings:
- **Connect to Predefined Network**: Enable **"Connect to predefined network"** (select `coolify`). This connects Atuin to Coolify's shared Docker network so it can resolve and communicate with your PostgreSQL database container.

> **Note**: Setting **Watch Paths** is essential in a monorepo setup. It ensures Coolify only rebuilds or redeploys this application when files inside `/stacks/atuin-server/` change.

### 3. Domains & Reverse Proxy
1. In the **Domains** field, set your domain (e.g., `https://atuin.example.com`).
2. Coolify's reverse proxy (Traefik or Caddy) automatically routes HTTPS traffic directly to internal port `8888`.

---

## 🗄️ Database Setup Options

Atuin requires a database to store user accounts and encrypted history chunks. You have several deployment options in Coolify:

### Option A: Coolify-managed PostgreSQL (Recommended)
This approach leverages Coolify's built-in database management, automated backups, and scheduled snapshots:
1. In the same Coolify Project and Environment, click **+ New Resource** → **Database** → **PostgreSQL**.
2. Once created, check the **Service Settings** of the database to retrieve the internal database host, username, password, and database name.
3. In the Atuin service **Environment Variables** tab, define:
   - `POSTGRES_HOST`: The internal container name or service name of your Coolify PostgreSQL instance.
   - `POSTGRES_USER`: Database user.
   - `POSTGRES_PASSWORD`: Database password.
   - `POSTGRES_DB`: Database name.
4. In the Atuin service's **Advanced** tab, ensure **Connect to predefined network** (`coolify`) is enabled so the Atuin container can reach the PostgreSQL container on the internal Docker network.

Coolify dynamically constructs `ATUIN_DB_URI` using these variables.

### Option B: External PostgreSQL (Different Project or Remote Host)
If connecting to PostgreSQL in a different Coolify project or an external provider (Neon, Supabase, AWS RDS):
- Set `ATUIN_DB_URI` directly (e.g. `postgres://user:password@hostname:5432/dbname`).
- **Coolify Cross-Project Networking**: If the PostgreSQL container is in a *different* Coolify project on the same server, go to the **Advanced** tab of your Atuin service in Coolify and select **Connect to predefined network** (`coolify`). This enables communication between projects over Coolify's Docker bridge.

### Option C: SQLite (Single-File Database)
For personal or low-traffic installations, SQLite can be used without running a PostgreSQL instance:
1. Set `ATUIN_DB_URI=sqlite:///config/atuin.db`.
2. Ensure `/config` is mounted to a persistent volume (see Storage below).

### Option D: Self-Contained Setup (All-in-One Compose)
If you prefer running PostgreSQL within the same Docker Compose stack instead of managing a separate database resource:

```yaml
services:
  atuin:
    image: ghcr.io/atuinsh/atuin:18.23.0
    container_name: atuin
    command: start
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      ATUIN_HOST: "0.0.0.0"
      ATUIN_DB_URI: postgres://${POSTGRES_USER:-atuin}:${POSTGRES_PASSWORD:-password}@db:5432/${POSTGRES_DB:-atuin}
      ATUIN_CONFIG_DIR: /config
      RUST_LOG: ${RUST_LOG:-info,atuin_server=debug}
    volumes:
      - ./config:/config
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://127.0.0.1:8888/healthz || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  db:
    image: postgres:18-alpine
    container_name: atuin-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER:-atuin}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-password}
      - POSTGRES_DB=${POSTGRES_DB:-atuin}
    volumes:
      - postgres-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER:-atuin} -d $${POSTGRES_DB:-atuin}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
```

> **Note on PostgreSQL 18**: The default volume mount target for official Postgres 18 images is `/var/lib/postgresql` (without the `/data` suffix) due to upstream version-specific layout adjustments.

---

## 💾 Storage & Persistence

The stack mounts `./config:/config` into the container:
- In Coolify Git deployments, this mounts the repository's `config/` directory containing `server.toml`.
- If using SQLite (`sqlite:///config/atuin.db`) or if you want volume storage for persistent config changes, configure a **Persistent Storage** mount to `/config` in Coolify's **Storage** tab.

---

## ⚙️ Environment Variables

In the Atuin service settings in Coolify, go to the **Environment Variables** tab to configure your database connection and log levels:

| Variable | Description | Default |
| :--- | :--- | :--- |
| `POSTGRES_HOST` | PostgreSQL service name or host | `db` |
| `POSTGRES_USER` | Database username | `atuin` |
| `POSTGRES_PASSWORD` | Database password | `password` |
| `POSTGRES_DB` | Database name | `atuin` |
| `RUST_LOG` | Logging verbosity | `info,atuin_server=debug` |

Coolify uses these to construct `ATUIN_DB_URI` automatically (`postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:5432/${POSTGRES_DB}`).

---

## 🔒 Security & Post-Registration

Once you have created your initial user account, prevent unauthorized registrations by closing public registration:

1. Open `config/server.toml` (either in the repository or in Coolify).
2. Set `open_registration = false`.
3. Redeploy the service.

---

## 💻 Client Configuration

To connect your terminal shell to your new Atuin server:

### 1. Register Account
```bash
atuin register -u <username> -e <email> -p <password>
```

> **Important**: Atuin will output your client encryption key. Store this key securely (e.g. in your password manager); it is required to log in from other machines.

### 2. Login on Other Machines
```bash
atuin login -u <username> -e <email> -p <password> -k "<encryption-key>"
```

### 3. Configure Sync Address
Update `~/.config/atuin/config.toml`:
```toml
sync_address = "https://atuin.example.com"
```

### 4. Sync History
```bash
atuin sync
```

---

## 🧪 Local Verification

To test the stack locally using Docker Compose:

```bash
# Validate Compose syntax
mise run validate

# Run Compose configuration validation including test port and database overrides
mise run test

# Launch locally with test database and host port 8888 exposed
docker compose -f stacks/atuin-server/docker-compose.yaml -f stacks/atuin-server/docker-compose.test.yaml up -d

# Smoke test readiness endpoints
curl -fsS http://localhost:8888/
curl -fsS http://localhost:8888/healthz

# Stop and clean up test containers and volumes
docker compose -f stacks/atuin-server/docker-compose.yaml -f stacks/atuin-server/docker-compose.test.yaml down -v
```
