---
project_id: FinanceDev-2026
tags:
  - finance-dev
  - reference
status: Reference
phase: 'Phase 1: Foundation'
---

# 📡 Finance App — API Route Reference

All routes live under `src/app/api/`. They are Next.js App Router Route Handlers — each file exports named functions (`GET`, `POST`, `PUT`, `DELETE`). All responses are JSON. Errors return `{ error: string }` with the appropriate HTTP status code.

---

## Conventions

- All IDs in route segments are integers: `/api/envelopes/[id]` where `id` is a Prisma record ID.
- `PUT` routes accept partial updates — only fields present in the body are updated.
- Dates are ISO 8601 strings in request bodies and responses.
- Amounts are plain floats (NZD). Negative = debit/expense, positive = credit/income.
- The `AppSettings` singleton is always `id = 1`; the API uses `upsert` internally.

---

## Settings

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/settings`|—|Return settings row, upsert defaults if not exists|
|`PUT`|`/api/settings`|`Partial<AppSettings>`|Update one or more settings fields|

---

## Categories

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/categories`|—|List all, ordered by `sortOrder` ASC|
|`POST`|`/api/categories`|`{ name, colour, sortOrder? }`|Create new category|
|`PUT`|`/api/categories/[id]`|`{ name?, colour?, sortOrder? }`|Update category|
|`DELETE`|`/api/categories/[id]`|—|Delete — returns 409 if any envelopes reference it, 403 if `isSystem`|

---

## Budget Envelopes

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/envelopes`|—|List all envelopes with category and current month spend|
|`POST`|`/api/envelopes`|`{ name, allocated, categoryId? }`|Create envelope|
|`PUT`|`/api/envelopes/[id]`|`{ name?, allocated?, categoryId? }`|Update envelope|
|`DELETE`|`/api/envelopes/[id]`|—|Delete envelope|

---

## Transactions

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/transactions`|—|List transactions. Query params: `envelopeId`, `categoryId`, `source`, `startDate`, `endDate`, `unreviewed`|
|`POST`|`/api/transactions`|`{ date, description, amount, envelopeId?, categoryId? }`|Create manual transaction|
|`PUT`|`/api/transactions/[id]`|`{ date?, description?, amount?, envelopeId?, categoryId? }`|Update transaction|
|`DELETE`|`/api/transactions/[id]`|—|Delete transaction|
|`POST`|`/api/transactions/[id]/split`|`{ splits: [{ envelopeId, amount, description? }] }`|Split transaction — validates splits sum to original, deletes original, inserts N replacements|

**Split validation:** sum of `splits[].amount` must equal the original transaction `amount`. Returns 422 if not. Each split inherits the original's `date`, `source`, and `categoryId` unless overridden.

> **`unreviewed` filter:** returns transactions where `envelopeId === null && source === 'akahu'`. Transactions with `source === 'seed'` or `source === 'manual'` are excluded.

---

## Scheduled Transactions

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/scheduled`|—|List all, ordered by `nextDueDate` ASC. Query param: `activeOnly=true`|
|`POST`|`/api/scheduled`|`{ name, amount, frequency, nextDueDate, envelopeId?, categoryId?, dayOfMonth?, notes? }`|Create scheduled transaction|
|`PUT`|`/api/scheduled/[id]`|`Partial<ScheduledTransaction>`|Update|
|`DELETE`|`/api/scheduled/[id]`|—|Soft delete — sets `isActive = false`|
|`POST`|`/api/scheduled/[id]/process`|—|Mark as processed — creates `Transaction` record, advances `nextDueDate`|
|`POST`|`/api/scheduled/[id]/skip`|—|Skip this occurrence — advances `nextDueDate` without creating a transaction|
|`GET`|`/api/scheduled/due`|—|Query param: `days=7` (default). Returns scheduled transactions due within N days|

**Process logic:**

1. Create `Transaction` linked via `scheduledTxnId`
2. Advance `nextDueDate` using `calculateNextDueDate(frequency, dayOfMonth)`
3. Update `lastProcessed`

---

## Mortgage

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/mortgage`|—|List all mortgages|
|`POST`|`/api/mortgage`|`{ name, principal, interestRate, termYears, startDate }`|Create mortgage|
|`PUT`|`/api/mortgage/[id]`|`Partial<Mortgage>`|Update — if repayment fields change, upserts linked `ScheduledTransaction`|
|`DELETE`|`/api/mortgage/[id]`|—|Delete mortgage|
|`POST`|`/api/mortgage/[id]/scenarios`|`{ rates: number[], termYears?, extraRepayment? }`|**Stateless.** Returns `[{ rate, monthlyRepayment, fortnightlyRepayment, totalInterest, totalPaid, payoffMonths }]` for each rate — nothing written to DB|

**Repayment sync:** When `repaymentAmount` or `repaymentFrequency` is updated via `PUT`, the API automatically upserts the linked `ScheduledTransaction` (keyed on `mortgageId`). No separate API call needed.

---

## Savings Goals

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/savings`|—|List all goals with projection data computed|
|`POST`|`/api/savings`|`{ name, targetAmount, targetDate? }`|Create goal|
|`PUT`|`/api/savings/[id]`|`Partial<SavingsGoal>`|Update goal details|
|`DELETE`|`/api/savings/[id]`|—|Delete goal and contributions|
|`POST`|`/api/savings/[id]/deposit`|`{ amount, note? }`|Deposit — updates `currentAmount`, inserts `SavingsContribution`, updates `highWaterMark` if exceeded|
|`POST`|`/api/savings/[id]/withdraw`|`{ amount, note? }`|Withdraw — same as deposit with negative amount|
|`GET`|`/api/savings/[id]/contributions`|—|Contribution history, ordered `date DESC`|

**Projection fields** returned on `GET /api/savings` per goal:

```json
{
  "avgMonthlyContribution": 500,
  "projectedDate": "2027-06-01",
  "monthsToTarget": 14,
  "isOnTrack": true
}
```

---

## Property

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/property`|—|List all properties|
|`POST`|`/api/property`|`{ address, purchasePrice, purchaseDate, currentRV?, rvYear?, notes? }`|Create property — auto-inserts first `PropertyValuation` from `purchasePrice`|
|`PUT`|`/api/property/[id]`|`Partial<Property>`|Update property details|
|`DELETE`|`/api/property/[id]`|—|Delete property and all valuations (cascade)|
|`GET`|`/api/property/[id]/valuations`|—|Valuation history, ordered `date DESC`|
|`POST`|`/api/property/[id]/valuations`|`{ value, source, date, notes? }`|Add new valuation|

**Source values:** `"purchase"` | `"market_estimate"` | `"registered_valuation"` | `"rv"`

---

## Solar

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/solar`|—|Get solar config and generation data|
|`POST`|`/api/solar`|`{ systemSizeKw, installDate, exportRate, importRate }`|Create/update solar config|
|`PUT`|`/api/solar/[id]`|`Partial<SolarConfig>`|Update solar config|

---

## Akahu — Open Banking _(Phase 2)_

|Method|Route|Body|Description|
|---|---|---|---|
|`GET`|`/api/akahu/test`|—|Calls `/v1/me` — returns `{ connected: true, email }` or error|
|`GET`|`/api/akahu/accounts`|—|Fetch accounts from Akahu, upsert `BankAccount` records, return list|
|`POST`|`/api/akahu/sync`|`{ historicalDays?: number }`|Sync all active bank accounts sequentially; returns aggregate result|
|`GET`|`/api/akahu/cron/sync`|Header: `X-Cron-Secret`|Internal endpoint for Docker cron container; 401 if secret missing/wrong|
|`POST`|`/api/akahu/sync/[accountId]`|—|Sync single bank account|
|`GET`|`/api/akahu/status`|—|Return last `AkahuSyncLog` per bank account|
|`GET`|`/api/akahu/pending`|—|_(Deferred to Phase 3)_ Fetch live pending transactions — not persisted, display only|

### POST /api/akahu/sync — request and response

**Request body (all optional):**
{ "historicalDays": 365 }

`historicalDays` overrides `AppSettings.akahuHistoricalDays` for this run only. Omit to use the stored setting (default 90). Only applies to accounts where `lastRefreshed` is null (first sync).

**Response:**
{
  "accounts": [
    { "accountId": 1, "name": "ANZ Everyday", "status": "success", "newCount": 12, "updatedCount": 1, "skippedCount": 47 }
  ],
  "totals": { "newCount": 12, "updatedCount": 1, "skippedCount": 47, "errorCount": 0 }
}

### GET /api/akahu/cron/sync — security

Protected by `X-Cron-Secret` header matched against `process.env.CRON_SECRET`. Returns 401 if absent or wrong. Uses AppSettings defaults (no body). Returns `{ ok: true, timestamp }`.

---

## Akahu — Recurring Detection _(Phase 2)_

|Method|Route|Body|Description|
|---|---|---|---|
|`POST`|`/api/akahu/detect-recurring`|`{ lookbackDays?: number }`|Analyse transactions for recurring patterns — **stateless, nothing written to DB**|

**Response shape:**

```json
{
  "proposed": [
    {
      "normalizedName": "Netflix",
      "matchedTransactions": [...],
      "suggestedFrequency": "monthly",
      "averageAmount": -18.99,
      "confidence": 0.92,
      "alreadyScheduled": false
    }
  ]
}
```

Confidence scoring:

- 3 occurrences = 0.6 base
- 4+ occurrences = 0.8 base
- Interval variance < 2 days = +0.15
- Amount variance < 5% = +0.10

---

## Seed / Test data

|Method|Route|Body|Description|
|---|---|---|---|
|`POST`|`/api/seed`|—|Create full test dataset — see below|
|`DELETE`|`/api/seed`|—|Purge all data in FK-safe order — does **not** drop tables|

**Seed creates:**

> ⚠️ Seeded transactions use `source: 'seed'` to distinguish them from Akahu-imported records. The "Unreviewed" filter (`envelopeId === null && source === 'akahu'`) will not match seed data.

- 1 `AppSettings` row
- 6 `Category` records (system + user)
- 10 `BudgetEnvelope` records
- ~60 `Transaction` records (3 months)
- 2 `Mortgage` records
- 4 `SavingsGoal` records with `SavingsContribution` history
- 1 `SolarConfig`
- 1 `Property` with 4 `PropertyValuation` records
- 4 `ScheduledTransaction` records

**Purge order (`DELETE /api/seed`):**

```
SavingsContribution → SavingsGoal
AkahuSyncLog        → BankAccount
Transaction
ScheduledTransaction
BankAccount
BudgetEnvelope      → Category
PropertyValuation   → Property
Mortgage
SavingsGoal
SolarConfig
Property
AppSettings
```

---

## Error responses

|Status|When|
|---|---|
|`400`|Missing required fields or invalid body|
|`401`|Unauthorized — cron secret missing or incorrect|
|`404`|Record not found|
|`409`|Conflict — e.g. deleting a Category that has envelopes|
|`403`|Forbidden — e.g. deleting a system category|
|`422`|Validation failed — e.g. split amounts don't sum to original|
|`500`|Unhandled server error|

---

## Related notes

- [[Finance App - Architecture Overview]]
- [[Finance App - Database Schema]]
- [[Finance App - Docker Compose Setup]]
- [[Akahu Integration - Architecture & Pseudocode]]
