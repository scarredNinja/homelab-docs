---

project_id: FinanceDev-2026 
tags:

- deployment
- docker
- devops
- finance-app 

created: 2026-04-01

---

# 🚀 Finance App — Deployment Notes

Operational reference for deploying, rebuilding, and maintaining the Finance App. Covers the current working state, branch history, known issues, and runbooks for common tasks.

---

## Current state

|Item|Detail|
|---|---|
|Status|Running|
|URL|`http://localhost:3000`|
|Branch|`claude/household-finance-app-0loXK`|
|Deployment method|Docker Compose|
|Database|PostgreSQL 16 (Docker volume `postgres_data`)|
|Claude Code version|v2.1.87 (installed `C:\Users\DJ\.local\bin`)|

---

## Branch inventory

|Branch|State|Notes|
|---|---|---|
|`claude/household-finance-app-0loXK`|✅ Working deployment|Current branch — use this|
|`phase-1-complete`|Tests + API hardening|No Settings or Categories UI built|
|`feature/api-hardening`|API hardening only|No UI features|

The `phase-1-complete` and `feature/api-hardening` branches have tests and API hardening work but the UI features (settings, categories) are unbuilt across all branches. New feature work from Phase 1 of the spec should be built on top of the current working branch.

---

## Claude Code setup (Windows)

Claude Code was installed to `C:\Users\DJ\.local\bin`. The PATH entry needs to be added manually if not already done:

1. Open **System Properties** → **Advanced** → **Environment Variables**
2. Under **User variables**, edit `Path`
3. Add: `C:\Users\DJ\.local\bin`
4. Click OK and restart any open terminals

Verify with: `claude --version`

---

## First-time setup

```bash
# 1. Clone / checkout the working branch
git checkout claude/household-finance-app-0loXK

# 2. Build and start everything
docker compose up --build

# App will be available at http://localhost:3000
# First run applies all Prisma migrations automatically
```

---

## Dockerfile — known issues resolved

Three issues were encountered and fixed during initial deployment. They are documented here so they are not re-introduced if the Dockerfile is modified.

### Issue 1: `dotenv` not available in runner stage

**Symptom:** App crashes on startup with `Cannot find module 'dotenv'`.

**Root cause:** The Dockerfile was selectively copying production packages into the runner stage but `dotenv` was classified as a dev dependency.

**Fix:** Copy the full `node_modules` directory into the runner stage rather than running a fresh `npm ci --production`.

```dockerfile
# Correct — copy full node_modules from builder
COPY --from=builder /app/node_modules ./node_modules
```

### Issue 2: `npx prisma` downloading a mismatched version

**Symptom:** Migration fails with `Could not find prisma/config` or version mismatch errors.

**Root cause:** When the runner stage ran `npx prisma migrate deploy`, `npx` downloaded the latest Prisma CLI rather than using the version pinned in `package.json`.

**Fix:** Copy the `prisma/` directory into the runner stage so the local installed CLI is used.

```dockerfile
COPY --from=builder /app/prisma ./prisma
```

### Issue 3: Prisma transitive deps missing

**Symptom:** Runtime errors referencing `@prisma/dev`, `valibot`, or other Prisma internal packages.

**Root cause:** Selective `node_modules` copy missed Prisma's transitive dependencies.

**Fix:** Same as Issue 1 — copy full `node_modules`. All three issues share the same root fix.

---

## Common runbooks

### Start the app

```bash
docker compose up
```

### Start with a full rebuild (after code or Dockerfile changes)

```bash
docker compose up --build
```

### Stop the app (preserve data)

```bash
docker compose down
```

### Stop and wipe all data

```bash
docker compose down -v
```

Deletes the `postgres_data` volume. All database content is permanently lost. Use this for a clean slate before testing migrations.

### View live logs

```bash
docker compose logs -f app     # App only
docker compose logs -f db      # Postgres only
docker compose logs -f         # Both services
```

### Restart just the app (not the DB)

```bash
docker compose restart app
```

### Force recreate containers without rebuilding image

```bash
docker compose up --force-recreate
```

---

## Database tasks

### Open Prisma Studio (visual DB browser)

Run this locally (not inside Docker). The app must be running so the DB port is exposed.

```bash
DATABASE_URL="postgresql://finance:finance@localhost:5432/financedb" npx prisma studio
```

Opens at `http://localhost:5555`.

### Connect via psql

```bash
docker compose exec db psql -U finance -d financedb
```

### Create a new migration (after schema changes)

Run locally, not inside Docker. Requires the DB to be running.

```bash
npx prisma migrate dev --name describe_your_change
```

This creates a new file under `prisma/migrations/` and applies it to the local DB. Commit the migration file to source control.

### Apply migrations in production (Docker)

Migrations run automatically on every container start via the `CMD` in the Dockerfile:

```dockerfile
CMD ["sh", "-c", "npx prisma migrate deploy && npm start"]
```

You do not need to run migrations manually. Just `docker compose up --build`.

### Reset the database (wipe + re-migrate)

```bash
# Stop containers and delete volume
docker compose down -v

# Restart — migrations run fresh on empty DB
docker compose up --build
```

---

## After adding a new environment variable

1. Add to `.env` locally (gitignored)
2. Add placeholder to `.env.example` (committed)
3. Add to `app.environment` in `docker-compose.yml`
4. Rebuild: `docker compose up --build`

See [[Finance App - Environment Variables]] for the full variable reference.

---

## After adding a new Prisma model or field

1. Edit `prisma/schema.prisma`
2. Run `npx prisma migrate dev --name your_migration_name` locally
3. Commit the generated migration file in `prisma/migrations/`
4. Rebuild the Docker image: `docker compose up --build`
5. Migrations apply automatically on container start
6. Update [[Finance App - Database Schema]] to reflect the change

---

## Checking migration status

```bash
# List applied migrations
docker compose exec app npx prisma migrate status
```

---

## Port reference

|Port|Service|Notes|
|---|---|---|
|`3000`|Next.js app|Main app URL|
|`5432`|PostgreSQL|Exposed to host for Prisma Studio and psql access|
|`5555`|Prisma Studio|Only active when running `npx prisma studio` locally|

---

## Data persistence

All data lives in the Docker named volume `postgres_data`. This volume survives:

- `docker compose down` (containers stopped, volume kept)
- `docker compose up --build` (image rebuilt, volume kept)
- Container crashes and restarts

The volume is destroyed by:

- `docker compose down -v` (explicit volume deletion)
- Manually running `docker volume rm financeapp_postgres_data`

**Backup recommendation:** Before any major migration or destructive change, dump the database:

```bash
docker compose exec db pg_dump -U finance financedb > backup_$(date +%Y%m%d).sql
```

Restore from dump:

```bash
docker compose exec -T db psql -U finance financedb < backup_20260401.sql
```

---

## Related notes

- [[Finance App - Docker Compose Setup]]
- [[Finance App - Environment Variables]]
- [[Finance App - Architecture Overview]]
- [[Finance App - Database Schema]]
- [[Finance Dev Change Log]]