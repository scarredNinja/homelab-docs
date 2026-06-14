---
project_id: FinanceDev-2026
tags:
  - finance-dev
  - reference
status: Reference
phase: 'Phase 1: Foundation'
---

# 🗄️ Finance App — Database Schema

Full Prisma schema reference. Source of truth is `prisma/schema.prisma`. This note documents all models, their fields, relationships, and any important constraints or design notes.

---

## Model index

|Model|Purpose|
|---|---|
|`AppSettings`|Singleton app config (currency, alerts, preferences)|
|`Category`|Budget category reference data|
|`BudgetEnvelope`|Named spending buckets linked to a Category|
|`Transaction`|All financial transactions (manual, imported, scheduled)|
|`ScheduledTransaction`|Recurring and one-off future transactions|
|`Mortgage`|Loan details, repayment schedule, refix tracking|
|`SavingsGoal`|Named savings targets with progress tracking|
|`SavingsContribution`|Individual deposit history per goal|
|`Property`|Owned property with purchase details|
|`PropertyValuation`|Point-in-time value estimates for a property|
|`SolarConfig`|Solar system configuration and generation data|
|`BankAccount`|Akahu-linked bank accounts _(Phase 2)_|
|`AkahuSyncLog`|Sync run history per bank account _(Phase 2)_|

---

## AppSettings

Singleton — always exactly one row with `id = 1`. Use `upsert` on all reads; never `create` directly.

```prisma
model AppSettings {
  id                   Int      @id @default(1)
  currency             String   @default("NZD")
  currencySymbol       String   @default("$")
  dateFormat           String   @default("DD/MM/YYYY")
  defaultEnvelopeView  String   @default("grid")   // "grid" | "list"
  mortgageAlertDays    Int      @default(90)
  lowEnvelopeThreshold Float    @default(0.1)       // 10% remaining triggers warning
  akahuHistoricalDays  Int      @default(90)        // Phase 2 — initial sync lookback window (days)
  updatedAt            DateTime @updatedAt
}
```

> **`akahuHistoricalDays`**: Controls how many days back to fetch on the first sync run for each bank account (when `BankAccount.lastRefreshed` is null). Subsequent syncs always use `lastRefreshed - 2 days` regardless of this setting. Default 90 days; raise to 365 for a deep historical import.

---

## Category

Reference data for grouping envelopes and tagging transactions. System categories (`isSystem = true`) cannot be deleted via the API — they ship with the app seed.

```prisma
model Category {
  id           Int              @id @default(autoincrement())
  name         String
  colour       String           @default("#6b7280")
  sortOrder    Int              @default(0)
  isSystem     Boolean          @default(false)
  envelopes    BudgetEnvelope[]
  transactions Transaction[]
  scheduledTransactions ScheduledTransaction[]
  createdAt    DateTime         @default(now())
}
```

**Constraints:** `DELETE` blocked if any `BudgetEnvelope` references this category (API returns 409).

---

## BudgetEnvelope

Named spending buckets. Each envelope belongs to a Category and can optionally be linked to a bank account for auto-import.

```prisma
model BudgetEnvelope {
  id           Int       @id @default(autoincrement())
  name         String
  allocated    Float     @default(0)
  categoryId   Int?
  category     Category? @relation(fields: [categoryId], references: [id])
  transactions Transaction[]
  scheduledTransactions ScheduledTransaction[]
  bankAccounts BankAccount[]     // Phase 2
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
}
```

---

## Transaction

Every financial transaction. Source can be `manual`, `akahu`, `scheduled`, or `split`.

```prisma
model Transaction {
  id               Int                   @id @default(autoincrement())
  date             DateTime
  description      String
  amount           Float                 // negative = debit, positive = credit
  source           String                @default("manual")
  envelopeId       Int?
  envelope         BudgetEnvelope?       @relation(fields: [envelopeId], references: [id])
  categoryId       Int?
  category         Category?             @relation(fields: [categoryId], references: [id])
  scheduledTxnId   Int?
  scheduledTxn     ScheduledTransaction? @relation(fields: [scheduledTxnId], references: [id])
  openBankingRef   String?               @unique   // Akahu trans_ ID — dedup key
  isConfirmed      Boolean               @default(true)
  bankAccountId    Int?                            // Phase 2
  bankAccount      BankAccount?          @relation(fields: [bankAccountId], references: [id])
  akahuType        String?                         // EFTPOS | PAYMENT | DIRECT_DEBIT etc.
  akahuMerchant    String?                         // merchant.name from enriched data
  createdAt        DateTime              @default(now())
  updatedAt        DateTime              @updatedAt
}
```

**Key constraints:**

- `openBankingRef` is `@unique` — prevents duplicates on re-sync. Null values are excluded from uniqueness check by Postgres partial index behaviour.
- Splits: when a transaction is split, the original is deleted and N replacement records are inserted with the same `date` and `source`.
- `source: 'seed'` is used for test data records to distinguish them from real `'akahu'` imports — important for the "Unreviewed" filter (`envelopeId === null && source === 'akahu'`).

---

## ScheduledTransaction

Covers both recurring (mortgage repayments, rent, subscriptions) and one-off future transactions. Soft-deleted via `isActive = false`.

```prisma
model ScheduledTransaction {
  id             Int             @id @default(autoincrement())
  name           String
  amount         Float
  envelopeId     Int?
  envelope       BudgetEnvelope? @relation(fields: [envelopeId], references: [id])
  categoryId     Int?
  category       Category?       @relation(fields: [categoryId], references: [id])
  frequency      String          // "weekly" | "fortnightly" | "monthly" | "annually" | "once"
  nextDueDate    DateTime
  lastProcessed  DateTime?
  dayOfMonth     Int?            // 1–28, for monthly frequency only (capped at 28 to avoid Feb issues)
  dayOfWeek      Int?            // 0–6, for weekly/fortnightly
  source         String?         // "manual" | "open_banking" | "akut_detected"
  openBankingRef String?         // external ref for matching bank feed transactions
  mortgageId     Int?            @unique   // unique: one repayment schedule per mortgage
  mortgage       Mortgage?       @relation(fields: [mortgageId], references: [id])
  isActive       Boolean         @default(true)
  autoConfirm    Boolean         @default(false)
  notes          String?
  instances      Transaction[]
  createdAt      DateTime        @default(now())
  updatedAt      DateTime        @updatedAt
}
```

**Key constraints:**

- `@@unique([mortgageId])` enables the `upsert` pattern when syncing mortgage repayment schedule changes.
- `dayOfMonth` is capped at 28 in application logic to avoid February edge cases.

**Processing flow:** `POST /api/scheduled/[id]/process` → creates `Transaction`, advances `nextDueDate` via `calculateNextDueDate()`.

---

## Mortgage

Loan details plus refix tracking and repayment schedule. The repayment schedule fields sync to a linked `ScheduledTransaction`.

```prisma
model Mortgage {
  id                   Int                    @id @default(autoincrement())
  name                 String
  principal            Float
  interestRate         Float
  termYears            Int
  startDate            DateTime
  currentBalance       Float?                 // user-maintained actual balance
  nextRefixDate        DateTime?
  refixAlertDays       Int                    @default(90)
  repaymentAmount      Float?
  repaymentFrequency   String?                // "monthly" | "fortnightly"
  repaymentDayOfMonth  Int?                   // 1–28
  repaymentStartDate   DateTime?
  scheduledRepayments  ScheduledTransaction[]
  createdAt            DateTime               @default(now())
  updatedAt            DateTime               @updatedAt
}
```

**Fortnightly note:** `repaymentAmount` for fortnightly should be `(monthlyRepayment × 12) / 26` — 26 payments per year, not 24. See `src/lib/calculations/mortgage.ts`.

> **CDR note:** For ANZ, ASB, BNZ, and Westpac loan accounts, `meta.loan_details` (interest rate, term, repayment structure) is not available via official open banking APIs. Mortgage details are user-maintained.

---

## SavingsGoal

Named savings targets. `highWaterMark` tracks the peak `currentAmount` ever reached — used to show a "Rebuilding" badge if current drops below it.

```prisma
model SavingsGoal {
  id             Int                    @id @default(autoincrement())
  name           String
  targetAmount   Float
  currentAmount  Float                  @default(0)
  targetDate     DateTime?
  highWaterMark  Float                  @default(0)
  notes          String?
  contributions  SavingsContribution[]
  createdAt      DateTime               @default(now())
  updatedAt      DateTime               @updatedAt
}
```

**Projection logic** (computed in API, not stored):

```
avgMonthly      = sum of contributions in last 3 months / 3
remaining       = targetAmount - currentAmount
monthsToTarget  = remaining / avgMonthly  (null if avgMonthly ≤ 0)
projectedDate   = today + monthsToTarget months
```

---

## SavingsContribution

Immutable deposit/withdrawal history per goal. Written on every deposit via `POST /api/savings/[id]`.

```prisma
model SavingsContribution {
  id        Int         @id @default(autoincrement())
  goalId    Int
  goal      SavingsGoal @relation(fields: [goalId], references: [id])
  amount    Float
  note      String?
  date      DateTime    @default(now())
  createdAt DateTime    @default(now())
}
```

---

## Property

Owned property. First valuation is auto-inserted from `purchasePrice` on creation.

```prisma
model Property {
  id            Int                  @id @default(autoincrement())
  address       String
  purchasePrice Float
  purchaseDate  DateTime
  currentRV     Float?               // Council Rateable Value
  rvYear        Int?
  notes         String?
  valuations    PropertyValuation[]
  createdAt     DateTime             @default(now())
  updatedAt     DateTime             @updatedAt
}
```

---

## PropertyValuation

Point-in-time value estimates. Source identifies the type of valuation.

```prisma
model PropertyValuation {
  id         Int      @id @default(autoincrement())
  propertyId Int
  property   Property @relation(fields: [propertyId], references: [id], onDelete: Cascade)
  value      Float
  source     String   // "purchase" | "market_estimate" | "registered_valuation" | "rv"
  date       DateTime
  notes      String?
  createdAt  DateTime @default(now())
}
```

---

## SolarConfig

Solar system and generation config. Single record per household.

```prisma
model SolarConfig {
  id               Int      @id @default(autoincrement())
  systemSizeKw     Float?
  installDate      DateTime?
  exportRate       Float?   // $/kWh export tariff
  importRate       Float?   // $/kWh grid import rate
  // Monthly generation data stored as JSON or separate table — TBC
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt
}
```

---

## BankAccount _(Phase 2 — Akahu)_

Represents a connected bank account from Akahu. User maps each to a budget envelope for auto-assignment of imported transactions.

```prisma
model BankAccount {
  id            Int              @id @default(autoincrement())
  akahuId       String           @unique   // "acc_xxxxx" — stable Akahu identifier
  name          String                     // "ANZ Everyday"
  formattedName String?
  bankName      String?                    // "ANZ"
  type          String                     // "CHECKING" | "SAVINGS" | "CREDIT_CARD"
  balance       Float?                     // cached, updated on sync
  currency      String           @default("NZD")
  lastRefreshed DateTime?
  envelopeId    Int?
  envelope      BudgetEnvelope?  @relation(fields: [envelopeId], references: [id])
  transactions  Transaction[]
  syncLogs      AkahuSyncLog[]
  isActive      Boolean          @default(true)
  createdAt     DateTime         @default(now())
  updatedAt     DateTime         @updatedAt
}
```

---

## AkahuSyncLog _(Phase 2 — Akahu)_

One record per sync run. Lets you audit what happened and provides the `syncedFrom` watermark for the next run.

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
  syncedFrom      DateTime?   // start= param used (lastRefreshed - 2 days)
  syncedTo        DateTime?
}
```

---

## Delete/cascade order

When running `DELETE /api/seed` or similar full purges, delete in this dependency order to avoid FK violations:

```
SavingsContribution   → SavingsGoal
AkahuSyncLog          → BankAccount
Transaction           → (BudgetEnvelope, Category, ScheduledTransaction, BankAccount)
ScheduledTransaction  → (BudgetEnvelope, Category, Mortgage)
BankAccount           (no remaining dependants after Transaction and AkahuSyncLog)
BudgetEnvelope        → Category
PropertyValuation     → Property
Mortgage
SavingsGoal
SolarConfig
Property
AppSettings
```

> ⚠️ `AkahuSyncLog` must be deleted **before** `BankAccount`. `Transaction` must be deleted before both. The order above is FK-safe.

---

## Related notes

- [[Finance App - Architecture Overview]]
- [[Finance App - API Route Reference]]
- [[Finance App - Docker Compose Setup]]
