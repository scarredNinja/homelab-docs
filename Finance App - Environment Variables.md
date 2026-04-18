---

project_id: FinanceDev-2026 
tags:

- environment
- config
- security
- finance-app 
- created: 2026-04-01

---

# 🔐 Finance App — Environment Variables

Reference for all environment variables used by the app. Variables are loaded by Next.js at build time (public) or runtime (server-only). All sensitive values are server-only and must never be prefixed with `NEXT_PUBLIC_`.

---

## Quick reference

|Variable|Required|Phase|Where set|
|---|---|---|---|
|`DATABASE_URL`|✅ Yes|1|`.env` / Docker Compose|
|`NODE_ENV`|✅ Yes|1|Docker Compose (auto in dev)|
|`AKAHU_USER_TOKEN`|Phase 2|2|`.env` / Docker Compose|
|`AKAHU_APP_TOKEN`|Phase 2|2|`.env` / Docker Compose|

---

## Variables

### `DATABASE_URL`

PostgreSQL connection string. Used by Prisma for all DB operations.

```bash
# Docker Compose (app connects to the db service by hostname)
DATABASE_URL=postgresql://finance:finance@db:5432/financedb

# Local development (db running via Docker, app running via npm run dev)
DATABASE_URL=postgresql://finance:finance@localhost:5432/financedb
```

**Notes:**

- Username, password, and database name (`finance` / `finance` / `financedb`) match the values in `docker-compose.yml`. If you change them there, update this too.
- In Docker Compose the hostname is `db` (the service name). Outside Docker it is `localhost`.
- Prisma reads this at both `prisma migrate` time and runtime. It must be available in the runner stage of the Docker image — this is satisfied by passing it via `docker-compose.yml` environment block, not baking it into the image.

---

### `NODE_ENV`

Controls Next.js behaviour (optimisations, error detail, hot reload).

```bash
NODE_ENV=production    # Docker Compose
NODE_ENV=development   # Automatic when running npm run dev
```

**Notes:**

- Never set `NODE_ENV=development` in Docker Compose. Next.js will not optimise the build and error pages will expose stack traces.
- You do not need to set this in `.env` — `npm run dev` sets it automatically.

---

### `AKAHU_USER_TOKEN` _(Phase 2)_

Your personal Akahu User Access Token. This authenticates requests to the Akahu API on your behalf.

```bash
AKAHU_USER_TOKEN=user_token_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Where to get it:**

1. Log in at [my.akahu.nz](https://my.akahu.nz/)
2. Go to **Developers** → your personal app
3. Copy the **User Access Token**

**Notes:**

- This token grants access to your bank account data. Treat it like a password.
- Never commit it to source control. It belongs in `.env` (gitignored) and in the Docker Compose `environment` block only.
- For a personal app this token is static — it does not expire or rotate unless you regenerate it manually on the Akahu dashboard.
- Used server-side only in `src/lib/akahu/client.ts` via `process.env.AKAHU_USER_TOKEN`.

---

### `AKAHU_APP_TOKEN` _(Phase 2)_

Your Akahu App ID Token. Sent as the `X-Akahu-Id` header on every API request to identify your app.

```bash
AKAHU_APP_TOKEN=app_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Where to get it:**

1. Log in at [my.akahu.nz](https://my.akahu.nz/)
2. Go to **Developers** → your personal app
3. Copy the **App ID Token**

**Notes:**

- This is not a secret in the same way as the user token, but treat it as one anyway since it identifies your app registration.
- Required alongside `AKAHU_USER_TOKEN` on every Akahu API call.
- Used server-side only in `src/lib/akahu/client.ts` via `process.env.AKAHU_APP_TOKEN`.

---

## File setup

### `.env` (local development)

Used when running `npm run dev` outside Docker. This file is gitignored — never commit it.

```bash
# Database
DATABASE_URL="postgresql://finance:finance@localhost:5432/financedb"

# Phase 2 — uncomment when setting up Akahu
# AKAHU_USER_TOKEN=user_token_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# AKAHU_APP_TOKEN=app_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### `.env.example` (committed to source control)

Placeholder file showing what variables are needed, without real values. Keep this up to date when adding new variables.

```bash
# Database — required
DATABASE_URL=postgresql://USER:PASSWORD@localhost:5432/financedb

# Akahu open banking — required for Phase 2
# Get tokens from my.akahu.nz/apps → Developers
AKAHU_USER_TOKEN=
AKAHU_APP_TOKEN=
```

### `docker-compose.yml` (Docker deployment)

Variables are set inline in the `app.environment` block. The `db` hostname resolves inside the Docker network.

```yaml
app:
  environment:
    DATABASE_URL: postgresql://finance:finance@db:5432/financedb
    NODE_ENV: production
    # Phase 2 — uncomment and fill in when ready
    # AKAHU_USER_TOKEN: your_user_token_here
    # AKAHU_APP_TOKEN: your_app_token_here
```

---

## Security rules

- **Never prefix sensitive variables with `NEXT_PUBLIC_`** — anything with that prefix is embedded into the client-side JavaScript bundle and visible to anyone who visits the app.
- **Never log environment variables** — avoid `console.log(process.env)` in any server route or lib file.
- **Never commit `.env`** — confirm `.gitignore` includes `.env` and `.env.local`. The `.env.example` file with empty values is safe to commit.
- **Akahu tokens** — if either token is accidentally exposed, regenerate it immediately from the Akahu dashboard (`my.akahu.nz/apps`). The old token is invalidated as soon as you regenerate.

---

## Adding a new variable

1. Add it to `.env` locally with the real value.
2. Add a placeholder entry to `.env.example` with an empty value and a comment explaining where to get it.
3. Add it to the `app.environment` block in `docker-compose.yml`.
4. Access it in server-side code only via `process.env.YOUR_VARIABLE`.
5. Document it in this note.

---

## Related notes

- [[Finance App - Architecture Overview]]
- [[Finance App - Docker Compose Setup]]
- [[Akahu Integration - Architecture & Pseudocode]]