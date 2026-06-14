---
project_id: FinanceDev-2026
tags:
  - finance-dev
  - reference
created: '2026-04-01T00:00:00.000Z'
status: Reference
phase: 'Phase 1: Foundation'
---

# 🐳 Finance App — Docker Compose Setup

Deployment reference for running the Finance App locally via Docker Compose. The app runs at `http://localhost:3000`.

---

## Services

|Service|Image|Port|Purpose|
|---|---|---|---|
|`app`|Built from `Dockerfile`|`3000:3000`|Next.js app + Prisma|
|`db`|`postgres:16-alpine`|`5432:5432`|PostgreSQL database|
|`cron`|`alpine:3.19`|—|Hourly Akahu sync _(Phase 2)_|

---

## docker-compose.yml

```yaml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: finance
      POSTGRES_PASSWORD: finance
      POSTGRES_DB: financedb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U finance -d financedb"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://finance:finance@db:5432/financedb
      NODE_ENV: production
      # Phase 2 — add when setting up Akahu
      # AKAHU_USER_TOKEN: your_user_token_here
      # AKAHU_APP_TOKEN: your_app_token_here
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/api/akahu/test > /dev/null 2>&1 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s
    depends_on:
      db:
        condition: service_healthy

  # Phase 2 — Akahu scheduled sync
  # Uncomment when Phase 2 Akahu integration is active
  #
  # cron:
  #   image: alpine:3.19
  #   restart: unless-stopped
  #   depends_on:
  #     app:
  #       condition: service_healthy
  #   environment:
  #     CRON_SECRET: your_long_random_secret_here
  #   command: >
  #     sh -c "
  #       echo 'Cron service started';
  #       while true; do
  #         echo \"[$(date -u)] Running Akahu sync...\";
  #         wget -qO-
  #           --header=\"X-Cron-Secret: $$CRON_SECRET\"
  #           http://app:3000/api/akahu/cron/sync
  #           || echo '[cron] Sync failed';
  #         sleep 3600;
  #       done
  #     "

volumes:
  postgres_data:
```

---

## Dockerfile

The build uses a multi-stage approach. The key lesson from the initial deployment: copy the full `node_modules` into the runner stage — do not cherry-pick packages. Prisma's transitive deps (`@prisma/dev`, `valibot`, etc.) are required at runtime.

```dockerfile
# ── Stage 1: Dependencies ─────────────────────────────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

# ── Stage 2: Builder ──────────────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Generate Prisma client before building
RUN npx prisma generate
RUN npm run build

# ── Stage 3: Runner ───────────────────────────────────────────────────────────
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# Copy full node_modules — do not selectively copy packages
# Prisma transitive deps (valibot, @prisma/dev, etc.) are needed at runtime
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/prisma ./prisma

# dotenv must be available in the runner stage for Prisma
# (included via full node_modules copy above)

EXPOSE 3000

# Run migrations then start the app
CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]
```

### Known Dockerfile issues resolved

Three issues were hit and fixed during the initial deployment:

1. **`dotenv` not available in runner stage** — fixed by copying full `node_modules` instead of only production deps.
2. **`npx prisma` downloading a mismatched version** — fixed by copying the `prisma/` directory into the runner stage so the local CLI is used, not a fresh download.
3. **Prisma transitive deps missing** — fixed by copying full `node_modules` rather than selectively picking packages.

---

## Environment variables

### Required

|Variable|Value|Notes|
|---|---|---|
|`DATABASE_URL`|`postgresql://finance:finance@db:5432/financedb`|Set in `docker-compose.yml`|
|`NODE_ENV`|`production`|Set in `docker-compose.yml`|

### Phase 2 — Akahu (add when ready)

|Variable|Where to get it|Notes|
|---|---|---|
|`AKAHU_USER_TOKEN`|`my.akahu.nz/apps` → Developers page|Your personal user access token|
|`AKAHU_APP_TOKEN`|`my.akahu.nz/apps` → Developers page|Your personal app ID token|
|`CRON_SECRET`|Generate with `openssl rand -hex 32`|Shared secret for cron → app auth; set in both app and cron environment blocks|

Add these to the `app.environment` block in `docker-compose.yml`. Never commit them to source control.

### Local development (`.env`)

For `npm run dev` outside Docker:

```bash
DATABASE_URL="postgresql://finance:finance@localhost:5432/financedb"
# Phase 2
# AKAHU_USER_TOKEN=user_token_xxxxx
# AKAHU_APP_TOKEN=app_xxxxxxxxxxxxx
```

`.env` is gitignored. An `.env.example` with placeholder values should be committed.

---

## Cron service details _(Phase 2)_

The `cron` service uses an Alpine Linux container running a `while true` shell loop, calling `GET /api/akahu/cron/sync` every hour.

**Why Alpine + wget?** Zero extra dependencies, simple to read, easy to adjust interval (`sleep 3600`), logs appear in `docker compose logs -f cron`.

**Security:** Authenticated via `X-Cron-Secret` header. The route returns 401 if the header is missing or incorrect.

**Adjusting frequency:** Change `sleep 3600` to any number of seconds. Syncing more often than hourly won't yield fresher data — Akahu's personal app refresh schedule is daily unless you explicitly call `POST /refresh`.

**Viewing logs:**
```bash
docker compose logs -f cron
```

---

## Common commands

### Start everything

```bash
docker compose up --build
```

First run builds the image, runs `prisma migrate deploy`, and starts the app. Subsequent runs skip the image build if nothing has changed.

```bash
docker compose up
```

Start without rebuilding (faster if code hasn't changed).

### Stop

```bash
docker compose down
```

Stops containers but preserves the `postgres_data` volume (your data is safe).

```bash
docker compose down -v
```

Stops containers **and deletes the volume** — all data is wiped. Use this for a clean slate.

### Rebuild after schema changes

```bash
docker compose up --build
```

The `CMD` in the Dockerfile runs `prisma migrate deploy` on every container start, so new migrations are applied automatically.

### View logs

```bash
docker compose logs -f app       # App logs only
docker compose logs -f db        # Postgres logs only
docker compose logs -f cron      # Cron sync logs (Phase 2)
docker compose logs -f           # All services
```

### Run Prisma commands against the running DB

```bash
# Open Prisma Studio (DB GUI) against the Docker postgres
DATABASE_URL="postgresql://finance:finance@localhost:5432/financedb" npx prisma studio

# Generate client after schema changes (run locally, not in Docker)
npx prisma generate

# Create a new migration (run locally)
npx prisma migrate dev --name describe_your_change
```

### Connect to the database directly

```bash
docker compose exec db psql -U finance -d financedb
```

---

## Deployment notes

**This is a local-only personal app.** It is not exposed to the internet. The Postgres password in `docker-compose.yml` is intentionally simple — change it if you ever expose this outside your LAN.

**Data persistence:** All data lives in the `postgres_data` Docker named volume. As long as you don't run `docker compose down -v`, your data survives container rebuilds and restarts.

**Branch:** The working deployment is on branch `claude/household-finance-app-0loXK`. The `phase-1-complete` and `feature/api-hardening` branches contain tests and API hardening but their UI features (settings, categories) are unbuilt.

**PATH note (Windows):** Claude Code was installed to `C:\Users\DJ\.local\bin` — PATH still needs updating manually via System Properties → Environment Variables if not already done.

---

## Related notes

- [[Finance App - Architecture Overview]]
- [[Finance App - Database Schema]]
- [[Finance App - API Route Reference]]
- [[Finance App - Environment Variables]]
