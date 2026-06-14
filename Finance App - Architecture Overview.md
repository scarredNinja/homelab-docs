---
project_id: FinanceDev-2026
tags:
  - finance-dev
  - reference
status: Reference
phase: 'Phase 1: Foundation'
---

# 🏗️ Finance App — Architecture Overview

High-level map of the stack, page structure, data flow, and key design decisions.

---

## Stack

|Layer|Technology|Notes|
|---|---|---|
|Framework|Next.js 14 (App Router)|All pages use `src/app/` routing|
|Language|TypeScript|Strict mode|
|Database|PostgreSQL|Runs in Docker|
|ORM|Prisma|Schema at `prisma/schema.prisma`|
|Styling|Tailwind CSS|Utility-first, no component library|
|Charts|Recharts|Used on mortgage, solar, property, savings pages|
|Deployment|Docker Compose|See [[Finance App - Docker Compose Setup]]|

---

## Project structure

```
src/
├── app/
│   ├── page.tsx                  # Dashboard
│   ├── budget/page.tsx           # Budget envelopes
│   ├── transactions/page.tsx     # Transaction list + import
│   ├── scheduled/page.tsx        # Scheduled & recurring transactions
│   ├── savings/page.tsx          # Savings goals
│   ├── mortgage/page.tsx         # Mortgage calculator + refix
│   ├── property/page.tsx         # Property valuation tracking
│   ├── solar/page.tsx            # Solar generation tracking
│   ├── settings/page.tsx         # App settings + categories + open banking
│   └── api/                      # All API routes (see API Reference)
├── components/
│   └── layout/
│       └── Sidebar.tsx           # Nav sidebar
├── lib/
│   ├── calculations/
│   │   ├── mortgage.ts           # Repayment, amortisation, fortnightly
│   │   └── schedule.ts           # calculateNextDueDate, normalizeTransactionName
│   ├── akahu/
│   │   ├── client.ts             # Typed HTTP wrapper for Akahu API
│   │   ├── sync.ts               # syncAccount, upsertTransaction, tryMatchToScheduled
│   │   └── types.ts              # AkahuTransaction, AkahuAccount, response types
│   └── format.ts                 # formatCurrency, formatDate (respects AppSettings)
├── types/
│   └── index.ts                  # All shared TypeScript interfaces
prisma/
├── schema.prisma                 # Source of truth for DB schema
└── migrations/                   # Migration history
```

---

## Pages and their purpose

### Dashboard (`/`)

Central overview. Pulls summary data from most models.

- Budget envelope spend vs allocated
- Upcoming scheduled payments (next 14 days)
- Mortgage refix alert banner (if within alert window)
- Savings goal progress cards
- Property snapshot (current value vs purchase)
- Test data load/clear buttons (when no envelopes exist)

### Budget (`/budget`)

Manages envelopes and spending categories.

- Envelopes grouped by Category
- Monthly allocated vs spent per envelope
- Add/edit/delete envelopes
- Category management moved to Settings

### Transactions (`/transactions`)

Full transaction list with filtering and import.

- Filter by date range, category, envelope, source, reviewed status
- Split transaction across multiple envelopes
- Akahu-imported transactions flagged with bank badge
- Unreviewed filter for newly imported transactions
- Bulk envelope assignment
- Pending transactions section (live from Akahu, not persisted)

### Scheduled (`/scheduled`)

Recurring and one-off future transactions.

- Upcoming column (next 30 days) with Mark as Paid / Skip
- Full schedule list with add/edit/deactivate
- Overdue items highlighted in red

### Savings Goals (`/savings`)

Tracks progress toward named savings targets.

- Deposit / withdraw
- Projection to target date based on last 3 months average
- Contribution history accordion
- Rebuilding badge if current < high water mark

### Mortgage (`/mortgage`)

Loan calculators and refix planning.

- Overview: balance, rate, remaining term
- Amortisation schedule table
- Extra repayments calculator
- Rate scenarios tab (compare up to 4 rates)
- Repayment schedule (frequency, amount, day — synced to ScheduledTransaction)

### Property (`/property`)

Tracks property value over time.

- Summary card: purchase price, current estimate, council RV
- Gain/loss $ and % since purchase
- Area chart of valuation history
- Add manual valuations (market estimate, registered valuation, RV)

### Solar (`/solar`)

Generation and export tracking.

- Monthly generation vs export vs self-consumption
- Annual totals

### Settings (`/settings`)

App-wide configuration.

- General: currency, date format, default envelope view
- Alerts: mortgage refix alert threshold, low envelope warning threshold
- Categories: add/edit/delete/reorder — system categories are locked
- Open banking: Akahu connection status, bank account mapping, sync controls
- Data: load/clear test data

---

## Data flow

```
User action (UI)
  → Next.js page component (React state)
    → fetch() to /api/... route
      → Prisma query to PostgreSQL
        → JSON response
          → React state update → re-render
```

All API routes are in `src/app/api/`. They are Next.js Route Handlers (not pages/api). Each file exports named functions: `GET`, `POST`, `PUT`, `DELETE`.

Prisma Client is instantiated as a singleton in `src/lib/db.ts` to avoid connection pool exhaustion in development hot-reloads.

---

## Key design decisions

**Single-user, no auth** — This is a personal household finance app. There is no authentication layer. All data belongs to one household. If multi-user is ever needed it would require a full rewrite of the data model.

**AppSettings singleton** — The `AppSettings` table always has exactly one row (`id = 1`). All reads use `upsert` with defaults so there is always a row to return. Never use `create` directly.

**`openBankingRef` as dedup key** — The `Transaction.openBankingRef` field stores the Akahu `_id` (`trans_xxxxx`). It has a unique constraint. This is what prevents duplicates on every re-sync regardless of how many times the sync runs.

**Pending transactions are never persisted** — Akahu pending transactions have no stable ID and change frequently. They are fetched live, displayed, and discarded. They never touch the `Transaction` table.

**Scheduled transactions are soft-deleted** — `isActive = false` rather than hard delete. This preserves the link between historical `Transaction` records and the schedule that generated them.

**Fortnightly repayments = 26 payments/year** — Not monthly × 2 (which would be 24). Annual repayment is the same either way; the fortnightly amount is `(monthlyRepayment × 12) / 26`. This is implemented in `src/lib/calculations/mortgage.ts`.

**Categories are reference data** — `Category` is a lookup table. `BudgetEnvelope` and `Transaction` both FK into it. System categories (`isSystem = true`) cannot be deleted. User-created categories can be deleted only if no envelopes reference them (API returns 409 otherwise).

---

## Environment variables

|Variable|Purpose|Required|
|---|---|---|
|`DATABASE_URL`|PostgreSQL connection string|Yes|
|`AKAHU_USER_TOKEN`|Akahu User Access Token|Phase 2|
|`AKAHU_APP_TOKEN`|Akahu App ID Token|Phase 2|

See [[Finance App - Environment Variables]] for full setup instructions.

---

## Related notes

- [[Finance App - Database Schema]]
- [[Finance App - API Route Reference]]
- [[Finance App - Docker Compose Setup]]
- [[Akahu Integration - Architecture & Pseudocode]]
- [[Finance App - Environment Variables]]
