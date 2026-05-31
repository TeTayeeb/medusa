---
name: docker-setup
description: Set up Docker for a Medusa DTC starter project (pnpm monorepo with apps/backend + apps/storefront). Use when dockerizing Medusa, adding Docker Compose, configuring containers, or troubleshooting Docker connectivity issues. Contains exact file contents per the official Medusa documentation plus all known pitfalls and fixes.
---

# Docker Setup for Medusa DTC Starter

Complete guide for Dockerizing a Medusa v2 DTC starter monorepo (pnpm workspace with `apps/backend` + `apps/storefront`). Based on the official Medusa Docker installation documentation.

## When to Apply

Load this skill when:
- Setting up Docker for a new Medusa project
- Adding Docker Compose services (postgres, redis, medusa, storefront)
- Fixing Docker connectivity issues (SSL errors, HMR, host resolution)
- Creating or updating `Dockerfile`, `docker-compose.yml`, `start.sh`
- Debugging container startup failures

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Network                        │
│                                                          │
│  ┌──────────────┐    ┌──────────────┐                   │
│  │   postgres   │    │    redis     │                   │
│  │  :5432       │    │  :6379       │                   │
│  └──────┬───────┘    └──────┬───────┘                   │
│         │                   │                           │
│  ┌──────▼───────────────────▼──────┐                   │
│  │         medusa (backend)         │                   │
│  │  :9000 (API)  :5173 (Admin HMR) │                   │
│  └──────────────────┬──────────────┘                   │
│                     │                                   │
│  ┌──────────────────▼──────────────┐                   │
│  │       storefront (Next.js)       │                   │
│  │  :8000                           │                   │
│  └──────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

**Exposed ports on host:**
| Service    | Port  | URL                          |
|------------|-------|------------------------------|
| API        | 9000  | http://localhost:9000        |
| Admin      | 9000  | http://localhost:9000/app    |
| Admin HMR  | 5173  | (internal, no browser access)|
| Storefront | 8000  | http://localhost:8000        |
| PostgreSQL | 5432  | localhost:5432               |
| Redis      | 6379  | localhost:6379               |

---

## File Locations

All files go at the **repo root** (not inside `apps/backend`):

```
/                          ← repo root
├── Dockerfile             ← single image for both medusa + storefront
├── docker-compose.yml     ← 4 services
├── start.sh               ← medusa entrypoint
├── start-storefront.sh    ← storefront entrypoint
└── .dockerignore
```

---

## Complete File Contents

### `Dockerfile` (repo root)

```dockerfile
FROM node:20-alpine

WORKDIR /server

RUN npm install pnpm@10.11.1 -g

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml turbo.json ./
COPY apps/backend/package.json ./apps/backend/
COPY apps/storefront/package.json ./apps/storefront/

RUN pnpm install --frozen-lockfile

COPY . .

EXPOSE 9000 5173 8000

ENTRYPOINT ["./start.sh"]
```

> ⚠️ **CRITICAL: Use `WORKDIR /server`, NOT `/app`**
> Medusa Admin dashboard defaults to path `/app`. If WORKDIR is `/app`, the admin route conflicts with the working directory. This causes the admin UI to fail silently.

### `docker-compose.yml` (repo root)

```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: medusa_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: medusa-store
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - medusa_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: medusa_redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    networks:
      - medusa_network

  medusa:
    build: .
    container_name: medusa_backend
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    ports:
      - "9000:9000"
      - "5173:5173"
    # DATABASE_URL and REDIS_URL are set here as the source of truth for Docker.
    # Docker Compose gives `environment:` precedence over `env_file:`, so these
    # values always win. The .env file supplies all other variables (JWT_SECRET,
    # CORS, etc.). Do NOT put DATABASE_URL / REDIS_URL in apps/backend/.env when
    # running in Docker — they would be silently overridden anyway and cause confusion.
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/medusa-store
      - REDIS_URL=redis://redis:6379
    env_file:
      - apps/backend/.env
    volumes:
      - .:/server
      - /server/node_modules
      - /server/apps/backend/node_modules
    networks:
      - medusa_network

  storefront:
    build: .
    container_name: medusa_storefront
    restart: unless-stopped
    depends_on:
      - medusa
    ports:
      - "8000:8000"
    # See "Storefront env vars: server-side vs browser" section below before editing this.
    environment:
      - NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
    env_file:
      - apps/storefront/.env
    volumes:
      - .:/server
      - /server/node_modules
      - /server/apps/backend/node_modules
      - /server/apps/storefront/node_modules
      - /server/apps/storefront/.next
    entrypoint: ["./start-storefront.sh"]
    networks:
      - medusa_network

volumes:
  postgres_data:

networks:
  medusa_network:
    driver: bridge
```

> ⚠️ **`environment:` takes precedence over `env_file:` in Docker Compose.**
> `DATABASE_URL` and `REDIS_URL` are defined in the `environment:` block — this is the **source of truth** for Docker. The `env_file` supplies other variables (JWT_SECRET, CORS settings, etc.). Do **not** also set `DATABASE_URL`/`REDIS_URL` in `apps/backend/.env` for Docker use; if you do, the `env_file` values are silently ignored for those keys.

> ⚠️ **Anonymous volumes for `node_modules` are required.**
> pnpm workspaces create symlinked `node_modules`. Without these anonymous volumes, the bind mount (`. :/server`) overwrites them with empty host directories, breaking all dependencies at runtime.

### `start.sh` (repo root)

```sh
#!/bin/sh
cd /server/apps/backend

echo "Running database migrations..."
pnpm medusa db:migrate

echo "Seeding database..."
pnpm seed || echo "Seeding failed, continuing..."

echo "Starting Medusa development server..."
pnpm dev
```

Make executable: `chmod +x start.sh`

> ⚠️ **Line endings must be LF (Unix), not CRLF (Windows)**
> On Windows, use `dos2unix start.sh` or configure `.gitattributes` with `start.sh text eol=lf`.

### `start-storefront.sh` (repo root)

```sh
#!/bin/sh
cd /server/apps/storefront

echo "Starting Next.js Starter Storefront development server..."
pnpm dev
```

Make executable: `chmod +x start-storefront.sh`

### `.dockerignore` (repo root)

```
node_modules
apps/backend/node_modules
apps/storefront/node_modules
apps/backend/.medusa
apps/storefront/.next
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.git
.gitignore
README.md
.env.test
.nyc_output
coverage
.DS_Store
*.log
dist
build
```

---

## Required: `medusa-config.ts` Changes

Two additions are **required** for Docker to work:

### 1. Disable SSL for containerized PostgreSQL

```typescript
projectConfig: {
  databaseUrl: process.env.DATABASE_URL,
  databaseDriverOptions: {
    ssl: false,
    sslmode: "disable",
  },
  // ...
}
```

Without this, the medusa container throws:
```
Error: self-signed certificate
```
even though no SSL is configured — the pg driver defaults to SSL in some environments.

### 2. Vite HMR config for Admin dashboard

```typescript
admin: {
  vite: () => {
    return {
      server: {
        host: "0.0.0.0",
        allowedHosts: [
          "localhost",
          ".localhost",
          "127.0.0.1",
        ],
        hmr: {
          port: 5173,      // HMR websocket port inside container
          clientPort: 5173, // Port browser connects to (must match docker-compose expose)
        },
      },
    }
  },
},
```

Without this, the Admin dashboard loads but shows a blank page or HMR connection errors in the browser console.

---

## Root `package.json` Scripts

Add convenience scripts to the root `package.json`:

```json
{
  "scripts": {
    "docker:up": "docker compose up --build -d",
    "docker:down": "docker compose down"
  }
}
```

---

## Prerequisites Before First Run

1. **`.env` files must exist** — docker-compose references them via `env_file`:
   ```bash
   cp apps/backend/.env.template apps/backend/.env
   cp apps/storefront/.env.template apps/storefront/.env
   ```

2. **Edit `apps/backend/.env`** — set only non-Docker variables here. `DATABASE_URL` and `REDIS_URL` are already provided by `docker-compose.yml`'s `environment:` block and will override anything in this file:
   ```env
   JWT_SECRET=your-secret-here
   COOKIE_SECRET=your-secret-here
   STORE_CORS=http://localhost:8000
   ADMIN_CORS=http://localhost:9000
   AUTH_CORS=http://localhost:9000
   ```

   > ⚠️ Do **not** add `DATABASE_URL` or `REDIS_URL` to this `.env` for Docker use. The `docker-compose.yml` `environment:` block sets them to the correct Docker service hostnames and takes precedence. Adding them here creates confusion about which value is active.

3. **Edit `apps/storefront/.env`** — see "Storefront env vars: server-side vs browser" below for which value to use:
   ```env
   NEXT_PUBLIC_BASE_URL=http://localhost:8000
   ```

---

## Storefront Env Vars: Server-Side vs Browser

`NEXT_PUBLIC_` variables in Next.js are **embedded at build time** and sent to the browser. The browser runs on the user's machine — it cannot resolve Docker service names like `medusa`.

| Context | Can resolve `medusa`? | Correct URL |
|---|---|---|
| Inside storefront container (SSR/fetch) | ✅ Yes | `http://medusa:9000` |
| User's browser (client-side JS) | ❌ No | `http://localhost:9000` |

**The Medusa Next.js Starter uses `NEXT_PUBLIC_MEDUSA_BACKEND_URL` for client-side SDK calls** (from the browser). Therefore:

```env
# apps/storefront/.env — for browser (client-side) requests
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
```

And in `docker-compose.yml`, use the same `localhost` value (not the Docker service name):

```yaml
environment:
  - NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
```

> ⚠️ If your storefront also makes **server-side** requests (SSR, `getServerSideProps`, Route Handlers) that need to call the backend, those must use `http://medusa:9000`. Use a separate non-`NEXT_PUBLIC_` variable for SSR requests, e.g. `MEDUSA_BACKEND_URL=http://medusa:9000`, and reference it only in server-side code.

---

```bash
# Start all services (builds image if needed)
pnpm docker:up
# or: docker compose up --build -d

# View logs
docker compose logs -f
docker compose logs -f medusa   # medusa only

# Stop all services (keeps volumes/data)
pnpm docker:down
# or: docker compose down

# Stop and delete all data (clean slate)
docker compose down -v
```

---

## First-Time Admin User

After containers start (wait for migrations to finish):

```bash
docker compose exec medusa sh -c \
  "cd /server/apps/backend && pnpm medusa user -e admin@example.com -p supersecret"
```

Then log in at: **http://localhost:9000/app**

---

## Common Errors and Fixes

### `self-signed certificate` / SSL error on DB connect
**Fix:** Add `databaseDriverOptions: { ssl: false, sslmode: "disable" }` to `medusa-config.ts`.

### Admin dashboard blank page / HMR websocket error
**Fix:** Add `admin.vite()` config with `host: "0.0.0.0"` and `hmr: { port: 5173, clientPort: 5173 }`.

### `Cannot find module` / dependency errors at startup
**Fix:** Ensure anonymous volumes for `node_modules` are in `docker-compose.yml`. These prevent the bind mount from overwriting pnpm-linked dependencies.

### `WORKDIR /app` conflict with admin path
**Fix:** Always use `WORKDIR /server`. The admin dashboard uses `/app` as its default route prefix — same name as WORKDIR causes a path conflict.

### `start.sh: not found` or permission denied
**Fix:** Ensure `start.sh` has execute permission (`chmod +x start.sh`) AND uses LF line endings (not CRLF).

### Storefront can't reach backend
**Fix:** Set `NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://medusa:9000` (use Docker service name, not `localhost`).

### Database migrations fail on first start
**Cause:** postgres container not ready to accept connections yet when medusa starts.
**Fix:** Add a `healthcheck` to the postgres service and use `condition: service_healthy` in medusa's `depends_on`:

```yaml
services:
  postgres:
    image: postgres:15-alpine
    # ... other config ...
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  medusa:
    # ... other config ...
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
```

This is already included in the `docker-compose.yml` template above.

### Port already in use
**Fix:** Stop local postgres/redis first, or change port mappings in `docker-compose.yml`.

---

## Development Mode Note

This Docker setup runs Medusa in **development mode** (`pnpm dev`), not production. This is the approach per the official Medusa documentation for the DTC starter. It includes:
- Hot-module replacement for the Admin dashboard
- Auto-restart on file changes
- Source maps and verbose logging

For production deployments, a separate multi-stage production Dockerfile would be needed.

---

## pnpm Version

The official Medusa DTC starter uses `pnpm@10.11.1` (declared in root `package.json` `packageManager` field). The Dockerfile installs this exact version:

```dockerfile
RUN npm install pnpm@10.11.1 -g
```

If the project's `packageManager` field changes, update this line to match.
