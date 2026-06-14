---
project_id: FinanceDev-2026
phase: 2
tags:
  - open-banking
  - akahu
  - integration
  - finance-app
  - next-js
  - prisma
  - nz
  - created: '2026-04-01T00:00:00.000Z'
  - status: planning
status: Reference
---

# Akahu Open Banking Integration

> **Phase 2** — Personal app integration for transaction sync, account linking, and scheduled transaction matching.

## Overview

Akahu is an NZ open banking platform that aggregates bank account data. Since this is a personal app, there is no OAuth flow — authentication is a static User Access Token and App ID Token from `my.akahu.nz/apps`. This removes significant auth complexity compared to a multi-user implementation.

### Personal app constraints

|Feature|Personal App|Full App|
|---|---|---|
|Pricing|Free|Usage based|
|Users|1 (your account only)|Unlimited|
|Webhooks|✖|✅|
|Scheduled refresh|Daily|Configurable|
|Manual refresh rest period|1 hour|15 mins (default)|

### CDR transition (December 2025)

As of December 2025, ANZ, ASB, BNZ, and Westpac use regulated open banking APIs under the Consumer Data Right (CDR) legislation. Akahu is an accredited intermediary — your app continues to use the same Akahu API endpoints with no changes to auth or transaction data structures.

**One breaking change to be aware of:** The `meta.loan_details` field on loan accounts (interest rate, repayment structure, term, expiry) is **not available** for these four banks under the regulated standard. This field was only available on classic (reverse-engineered) connections. Mortgage details remain user-maintained in the app — do not plan to auto-populate the Mortgage page from Akahu.

---

## Architecture

### Integration layers

```
Your app (Next.js)         Your DB (Prisma)            Akahu API
─────────────────          ────────────────            ──────────
AppSettings              → AkahuCredentials            
  Store tokens + appId       Encrypted at rest         

GET /api/akahu/accounts  → BankAccount              → GET /v1/accounts
  Fetch + cache accounts     akahuId · name · type      Returns balance + type

POST /api/akahu/sync     → AkahuSyncLog             → GET /v1/transactions
  Manual or scheduled        lastSyncedAt per acct      start= lastSyncedAt

upsertTransaction()      → Transaction                 
  new / update / skip        openBankingRef = _id    

matchToScheduled()       → ScheduledTransaction        
  Amount + date window       Link if matched          

GET /api/akahu/pending   → NOT persisted            → GET /v1/transactions/pending
  Rebuild on each fetch      Display only               No stable _id

Link accounts UI         → BankAccount                 
  Map Akahu → Envelope       envelopeId FK            
```

> **Personal app note:** User + App tokens are static — no OAuth flow needed.

---

## Credentials

Store as environment variables, never in the database.

```bash
# .env (never committed)
AKAHU_USER_TOKEN=user_token_xxxxxx
AKAHU_APP_TOKEN=app_xxxxxxxxxxxxxxxxxx
CRON_SECRET=a_long_random_string_for_internal_cron_calls
```

The Settings page "Open banking" section validates these exist and tests them against `GET /v1/me` — it does not store them.

`CRON_SECRET` is used to authenticate calls from the Docker cron container to `GET /api/akahu/cron/sync`.

---

## Schema changes

### AppSettings addition

```prisma
model AppSettings {
  // ... existing fields ...
  akahuHistoricalDays  Int  @default(90)  // initial sync lookback; user-configurable in Settings UI
}
```

### New model: `BankAccount`

```prisma
model BankAccount {
  id            Int       @id @default(autoincrement())
  akahuId       String    @unique  // "acc_xxxxxx" — stable Akahu identifier
  name          String             // "ANZ Everyday"
  formattedName String?
  bankName      String?            // "ANZ"
  type          String             // "CHECKING" | "SAVINGS" | "CREDIT_CARD"
  balance       Float?             // cached, refreshed on sync
  currency      String    @default("NZD")
  lastRefreshed DateTime?          // from Akahu's refreshed.transactions timestamp

  // User maps each bank account to a budget envelope
  envelopeId    Int?
  envelope      BudgetEnvelope? @relation(fields: [envelopeId], references: [id])

  transactions  Transaction[]
  syncLogs      AkahuSyncLog[]
  isActive      Boolean   @default(true)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
}
```

### New model: `AkahuSyncLog`

```prisma
model AkahuSyncLog {
  id              Int         @id @default(autoincrement())
  bankAccountId   Int
  bankAccount     BankAccount @relation(fields: [bankAccountId], references: [id])
  startedAt       DateTime    @default(now())
  completedAt     DateTime?
  status          String      // "running" | "success" | "error"
  newCount        Int         @default(0)
  updatedCount    Int         @default(0)
  skippedCount    Int         @default(0)
  errorMessage    String?
  syncedFrom      DateTime?
  syncedTo        DateTime?
}
```

### Modify `Transaction`

Add fields:

```prisma
bankAccountId   Int?
bankAccount     BankAccount? @relation(fields: [bankAccountId], references: [id])
akahuType       String?      // EFTPOS | PAYMENT | DIRECT_DEBIT etc.
akahuMerchant   String?      // merchant.name from enriched data
// openBankingRef already planned — stores the Akahu _id "trans_xxxxxx"
```

> `openBankingRef` must have a unique constraint (nullable-safe). This is what prevents duplicates on every re-sync.

---

## Files to create

|File|Purpose|
|---|---|
|`src/lib/akahu/client.ts`|Typed HTTP wrapper with pagination|
|`src/lib/akahu/types.ts`|TypeScript interfaces for Akahu responses|
|`src/lib/akahu/sync.ts`|Core sync logic: upsert + match|
|`src/app/api/akahu/test/route.ts`|GET — validate credentials via `/me`|
|`src/app/api/akahu/accounts/route.ts`|GET — fetch + cache accounts|
|`src/app/api/akahu/sync/route.ts`|POST — trigger full sync|
|`src/app/api/akahu/sync/[accountId]/route.ts`|POST — sync single account|
|`src/app/api/akahu/cron/sync/route.ts`|GET — internal cron endpoint, validated via `CRON_SECRET`|
|`src/app/api/akahu/pending/route.ts`|GET — live pending txns _(deferred to Phase 3)_|
|`src/app/api/akahu/status/route.ts`|GET — last sync log per account|

---

## Pseudocode

### `src/lib/akahu/client.ts`

```typescript
const BASE_URL = 'https://api.akahu.io/v1'

function getHeaders() {
  return {
    'Authorization': `Bearer ${process.env.AKAHU_USER_TOKEN}`,
    'X-Akahu-Id': process.env.AKAHU_APP_TOKEN!,
    'Content-Type': 'application/json',
  }
}

async function akahuGet<T>(path: string, params?: Record<string, string>): Promise<T> {
  const url = new URL(`${BASE_URL}${path}`)
  if (params) Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v))

  const res = await fetch(url.toString(), { headers: getHeaders() })

  if (!res.ok) {
    const body = await res.json().catch(() => ({}))
    throw new Error(`Akahu API error ${res.status}: ${body.message ?? res.statusText}`)
  }

  const data = await res.json()
  if (!data.success) throw new Error(`Akahu returned success=false`)
  return data
}

export const akahu = {
  getMe: () => akahuGet<AkahuMeResponse>('/me'),
  getAccounts: () => akahuGet<AkahuAccountsResponse>('/accounts'),

  // Handles pagination internally — returns all transactions
  getTransactions: async (params: AkahuTransactionParams): Promise<AkahuTransaction[]> => {
    const all: AkahuTransaction[] = []
    let cursor: string | null = null

    do {
      const query = { ...params }
      if (cursor) query.cursor = cursor
      const page = await akahuGet<AkahuTransactionsResponse>('/transactions', query)
      all.push(...page.items)
      cursor = page.cursor?.next ?? null
    } while (cursor !== null)

    return all
  },

  // Pending: no pagination, no stable IDs — rebuild every time
  getPendingTransactions: () =>
    akahuGet<AkahuTransactionsResponse>('/transactions/pending'),

  // Only call this from a user-triggered "Force refresh" button
  requestRefresh: () =>
    fetch(`${BASE_URL}/refresh`, { method: 'POST', headers: getHeaders() }),
}
```

### `src/lib/akahu/sync.ts`

```typescript
interface SyncOptions {
  historicalDays?: number  // overrides AppSettings.akahuHistoricalDays for this run
}

export async function syncAccount(
  bankAccount: BankAccount,
  prisma: PrismaClient,
  options: SyncOptions = {}
) {
  // Resolve historical days: explicit param → AppSettings → hardcoded default
  const settings = await prisma.appSettings.findUnique({ where: { id: 1 } })
  const historicalDays = options.historicalDays ?? settings?.akahuHistoricalDays ?? 90

  const syncedFrom = bankAccount.lastRefreshed
    ? subtractDays(bankAccount.lastRefreshed, 2)
    : subtractDays(new Date(), historicalDays)

  // 1. Create sync log — track progress and allow resume
  const log = await prisma.akahuSyncLog.create({
    data: {
      bankAccountId: bankAccount.id,
      status: 'running',
      syncedFrom,
    }
  })

  try {
    // 2. Fetch — pagination handled by client
    const akahuTxns = await akahu.getTransactions({
      start: log.syncedFrom.toISOString(),  // exclusive
    })

    const accountTxns = akahuTxns.filter(t => t._account === bankAccount.akahuId)
    let newCount = 0, updatedCount = 0, skippedCount = 0

    // 3. Upsert each transaction sequentially
    for (const txn of accountTxns) {
      const result = await upsertTransaction(txn, bankAccount, prisma)
      if (result === 'created') newCount++
      else if (result === 'updated') updatedCount++
      else skippedCount++
    }

    // 4. Mark sync complete
    await prisma.akahuSyncLog.update({
      where: { id: log.id },
      data: { status: 'success', completedAt: new Date(), newCount, updatedCount, skippedCount }
    })

    await prisma.bankAccount.update({
      where: { id: bankAccount.id },
      data: { lastRefreshed: new Date() }
    })

    return { success: true, newCount, updatedCount, skippedCount }

  } catch (error) {
    await prisma.akahuSyncLog.update({
      where: { id: log.id },
      data: { status: 'error', completedAt: new Date(), errorMessage: String(error) }
    })
    throw error
  }
}

async function upsertTransaction(
  akahuTxn: AkahuTransaction,
  bankAccount: BankAccount,
  prisma: PrismaClient
): Promise<'created' | 'updated' | 'skipped'> {

  // openBankingRef = Akahu's stable _id — this is the dedup key
  const existing = await prisma.transaction.findUnique({
    where: { openBankingRef: akahuTxn._id }
  })

  const data = {
    date: new Date(akahuTxn.date),
    description: akahuTxn.description,
    amount: akahuTxn.amount,
    source: 'akahu',
    openBankingRef: akahuTxn._id,
    bankAccountId: bankAccount.id,
    akahuType: akahuTxn.type,
    akahuMerchant: akahuTxn.merchant?.name ?? null,
    envelopeId: existing?.envelopeId ?? bankAccount.envelopeId ?? null,
  }

  if (!existing) {
    await prisma.transaction.create({ data })
    await tryMatchToScheduled(data, prisma)
    return 'created'
  }

  // Only update if something meaningful changed (banks do mutate)
  const changed =
    existing.amount !== akahuTxn.amount ||
    existing.description !== akahuTxn.description

  if (!changed) return 'skipped'

  await prisma.transaction.update({
    where: { id: existing.id },
    data: { ...data, envelopeId: existing.envelopeId },
  })
  return 'updated'
}

async function tryMatchToScheduled(txn, prisma) {
  // Match criteria:
  //   - same envelope
  //   - amount within ±1% (banks can add small fees)
  //   - nextDueDate within ±5 days of transaction date
  const windowStart = subtractDays(txn.date, 5)
  const windowEnd   = addDays(txn.date, 5)

  const candidate = await prisma.scheduledTransaction.findFirst({
    where: {
      envelopeId: txn.envelopeId,
      amount: { gte: txn.amount * 0.99, lte: txn.amount * 1.01 },
      nextDueDate: { gte: windowStart, lte: windowEnd },
      isActive: true,
    }
  })

  // If matched: link transaction to scheduled, advance nextDueDate
  // Flag as 'auto-matched' for user review in the UI
  if (candidate) {
    // ... link + advance logic
  }
}
```

### `src/app/api/akahu/sync/route.ts` (POST — manual trigger)

```typescript
export async function POST(request: Request) {
  const body = await request.json().catch(() => ({}))
  const historicalDays = typeof body.historicalDays === 'number'
    ? body.historicalDays
    : undefined

  const accounts = await prisma.bankAccount.findMany({ where: { isActive: true } })
  const results = []

  for (const account of accounts) {
    try {
      const result = await syncAccount(account, prisma, { historicalDays })
      results.push({ accountId: account.id, name: account.name, status: 'success', ...result })
    } catch (error) {
      results.push({ accountId: account.id, name: account.name, status: 'error', error: String(error) })
    }
  }

  const totals = results.reduce(
    (acc, r) => ({
      newCount:     acc.newCount     + (r.newCount ?? 0),
      updatedCount: acc.updatedCount + (r.updatedCount ?? 0),
      skippedCount: acc.skippedCount + (r.skippedCount ?? 0),
      errorCount:   acc.errorCount   + (r.status === 'error' ? 1 : 0),
    }),
    { newCount: 0, updatedCount: 0, skippedCount: 0, errorCount: 0 }
  )

  return Response.json({ accounts: results, totals })
}
```

### `src/app/api/akahu/cron/sync/route.ts` (GET — scheduled trigger)

```typescript
// Protected by CRON_SECRET header. Returns 401 if missing or wrong.
// Delegates to the same syncAccount() logic as the manual trigger.

export async function GET(request: Request) {
  const secret = request.headers.get('x-cron-secret')
  if (!secret || secret !== process.env.CRON_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const accounts = await prisma.bankAccount.findMany({ where: { isActive: true } })
  for (const account of accounts) {
    await syncAccount(account, prisma).catch(console.error)
  }

  return Response.json({ ok: true, timestamp: new Date().toISOString() })
}
```

---

## Akahu transaction type reference

|Type|Meaning|
|---|---|
|`CREDIT`|Money entered the account|
|`DEBIT`|Money left the account|
|`PAYMENT`|Payment to external account|
|`TRANSFER`|Internal transfer (same bank)|
|`STANDING ORDER`|Automatic payment|
|`EFTPOS`|EFTPOS terminal payment|
|`INTEREST`|Interest from provider|
|`FEE`|Fee from provider|
|`DIRECT DEBIT`|Direct debit|
|`DIRECT CREDIT`|Someone paying into the account|
|`ATM`|ATM deposit/withdrawal|
|`LOAN`|Loan-related payment|

---

## Gotchas and edge cases

### The 2-day mutation window

Always fetch from `lastSynced - 2 days`, not from `lastSynced` exactly. Banks silently mutate transaction details (date, description, balance) after posting. The upsert logic handles this — if the record exists but the amount or description changed, it updates in place.

### `start` is exclusive, `end` is inclusive

If last sync was `2026-03-31T12:00:00Z`, set `start=2026-03-29T12:00:00.000Z` to apply the 2-day buffer. Using `start=lastSynced` exactly would miss the cutoff millisecond.

### NZ timezone

Akahu returns all dates in UTC. A transaction dated `2026-04-01` in NZ time (NZDT, UTC+13) is `2026-03-31T11:00:00Z`. For filtering by NZ calendar date, include the offset in the query string: `start=2026-03-31T23:59:59.999+13:00`.

### Pending transactions — never persist them

_(Display deferred to Phase 3)_

Pending transactions have no stable `_id`. Completely rebuild the pending view on every fetch — never write to the `Transaction` table. When a pending transaction settles, it appears in the next settled sync with a real `_id` via the normal upsert flow.

### `meta.loan_details` not available for big 4 banks (CDR)

For ANZ, ASB, BNZ, and Westpac loan accounts connected via official open banking APIs, the `meta.loan_details` field is not available. This provided: repayment structure, interest rate, expiry, and term. Do not attempt to auto-populate the Mortgage page from Akahu for these banks. Current balance on the Mortgage model remains user-maintained.

### Don't auto-trigger refresh

The personal app manual refresh rest period is 1 hour. Calling `POST /refresh` more often returns an error. Only trigger this from an explicit "Force refresh from bank" button — rely on Akahu's daily scheduled refresh for everything else.

### `openBankingRef` unique constraint

Add a partial unique index on `Transaction.openBankingRef` (where not null). Without this, a re-sync of overlapping dates will insert duplicate rows. In Prisma, use `@unique` on the field — Prisma handles null exclusion correctly in Postgres.

---

## Scheduled sync — Docker Compose cron service

```yaml
cron:
  image: alpine:3.19
  restart: unless-stopped
  depends_on:
    app:
      condition: service_healthy
  environment:
    CRON_SECRET: ${CRON_SECRET}
  command: >
    sh -c "
      echo 'Cron service started';
      while true; do
        echo \"[$(date -u)] Running Akahu sync...\";
        wget -qO- \
          --header=\"X-Cron-Secret: $$CRON_SECRET\" \
          http://app:3000/api/akahu/cron/sync \
          || echo 'Sync failed';
        sleep 3600;
      done
    "
```

Uses Alpine `wget` — no curl required. Runs hourly (`sleep 3600`). Requires the app healthcheck to be configured in `docker-compose.yml` so `depends_on: condition: service_healthy` works.

## Settings UI additions

Add an "Open banking" section to the Settings page:

- **Connection status** — GET `/api/akahu/test` on load, shows green/red status chip
- **Bank accounts list** — list from `/api/akahu/accounts` with name, bank, balance, last synced; envelope mapping dropdown per row
- **Sync button** — POST to `/api/akahu/sync`; shows progress then "X new, Y updated" summary from sync log
- **Pending toggle** — show/hide pending transactions on the transactions page
- **Historical import window** — number input (days); reads from and writes to `AppSettings.akahuHistoricalDays`

## Transactions page additions

- Bank/Akahu badge on transactions where `source === 'akahu'`
- **Unreviewed filter** — transactions with `envelopeId === null && source === 'akahu'`
- Bulk-assign: select multiple unreviewed → assign envelope
- Auto-matched badge on transactions linked via `tryMatchToScheduled()`

---

## Build order (learning path)

1. `GET /api/akahu/test` — validate tokens, get comfortable with auth headers and response format
2. `GET /api/akahu/accounts` — fetch accounts, inspect the response shape
3. Manual sync script (`ts-node`) — raw data exploration before adding upsert logic
4. `upsertTransaction()` as a pure function with unit tests — most critical piece, easy to test with fixture data
5. Full sync route + Settings UI connection status + bank accounts list
6. Cron endpoint + Docker cron service
7. Pending transactions display — **Phase 3, not Phase 2**

---

## References

- [Akahu getting started](https://developers.akahu.nz/docs/getting-started)
- [Personal apps guide](https://developers.akahu.nz/docs/personal-apps)
- [Accessing transactional data](https://developers.akahu.nz/docs/accessing-transactional-data)
- [Transaction model](https://developers.akahu.nz/docs/the-transaction-model)
- [Data refreshes](https://developers.akahu.nz/docs/data-refreshes)
- [Rate limits](https://developers.akahu.nz/docs/reference-rate-limits)
- [Demo bank](https://developers.akahu.nz/docs/demo-bank)
- [Transitioning to official open banking](https://developers.akahu.nz/docs/official-open-banking)
