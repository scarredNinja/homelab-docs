---
project_id: FinanceDev-2026
tags:
  - finance-dev
  - reference
status: Reference
phase: 'Phase 1: Foundation'
---

# 📅 Finance App — Scheduled Transactions Design

Design document for the scheduled and recurring transactions system. Covers the data model, processing logic, frequency calculations, mortgage repayment sync, and the UI flows.

---

## Purpose

The scheduled transactions system handles two distinct use cases:

1. **Recurring transactions** — bills, subscriptions, mortgage repayments, salary. These repeat on a defined frequency and should auto-generate a `Transaction` record each time they're processed.
2. **One-off future transactions** — a known upcoming expense (e.g. insurance renewal, rates bill) that should appear in the upcoming payments dashboard widget before it's due.

Both are stored in the same `ScheduledTransaction` model. The `frequency` field distinguishes them — `"once"` is a one-off, everything else is recurring.

---

## Model summary

```prisma
model ScheduledTransaction {
  id             Int             @id @default(autoincrement())
  name           String
  amount         Float
  envelopeId     Int?
  categoryId     Int?
  frequency      String          // "weekly" | "fortnightly" | "monthly" | "annually" | "once"
  nextDueDate    DateTime
  lastProcessed  DateTime?
  dayOfMonth     Int?            // 1–28, monthly only
  dayOfWeek      Int?            // 0–6, weekly/fortnightly only
  source         String?         // "manual" | "open_banking" | "akut_detected"
  openBankingRef String?
  mortgageId     Int?            @unique
  isActive       Boolean         @default(true)
  autoConfirm    Boolean         @default(false)
  notes          String?
  instances      Transaction[]
}
```

See [[Finance App - Database Schema]] for the full definition including relations.

---

## Frequency types

|Value|Payments/year|Notes|
|---|---|---|
|`weekly`|52|`nextDueDate` advances by 7 days|
|`fortnightly`|26|`nextDueDate` advances by 14 days|
|`monthly`|12|Advances to same `dayOfMonth` next month|
|`annually`|1|Advances by 1 year|
|`once`|—|`isActive` set to `false` after processing — never repeats|

---

## `calculateNextDueDate` — `src/lib/calculations/schedule.ts`

The core pure function. Called whenever a scheduled transaction is processed or created.

```typescript
export function calculateNextDueDate(
  frequency: RepaymentFrequency,
  fromDate: Date = new Date(),
  dayOfMonth?: number
): Date {
  const next = new Date(fromDate)

  switch (frequency) {
    case 'weekly':
      next.setDate(next.getDate() + 7)
      break

    case 'fortnightly':
      next.setDate(next.getDate() + 14)
      break

    case 'monthly':
      next.setMonth(next.getMonth() + 1)
      // Snap to dayOfMonth if specified, capped at 28
      if (dayOfMonth) {
        next.setDate(Math.min(dayOfMonth, 28))
      }
      break

    case 'annually':
      next.setFullYear(next.getFullYear() + 1)
      break

    case 'once':
      // No advancement — caller should set isActive = false after processing
      break
  }

  return next
}
```

**Why cap `dayOfMonth` at 28?** February has 28 days in a common year. Setting day 29, 30, or 31 causes JavaScript's `Date` to roll over into March, silently shifting the payment date. Capping at 28 keeps monthly payments on a stable day year-round. If the user wants end-of-month, they use 28.

---

## `normalizeTransactionName` — `src/lib/calculations/schedule.ts`

Used by the Akut detection algorithm to group transactions by merchant before analysing intervals. See [[Finance App - Akut Detection Algorithm]] for full usage.

```typescript
export function normalizeTransactionName(description: string): string {
  return description
    .toLowerCase()
    .trim()
    // Strip date suffixes: "Netflix 01/04", "Netflix Apr 2026"
    .replace(/\b\d{1,2}[\/\-]\d{1,2}([\/\-]\d{2,4})?\b/g, '')
    .replace(/\b(jan|feb|mar|apr|may|jun|jul|aug|sep|oct|nov|dec)\w*\s*\d{0,4}\b/gi, '')
    // Strip reference numbers: "EFTPOS 12345 Countdown" → "eftpos countdown"
    .replace(/\b\d{4,}\b/g, '')
    // Strip card suffixes: "xx1234"
    .replace(/\bx+\d+\b/gi, '')
    // Collapse whitespace
    .replace(/\s+/g, ' ')
    .trim()
}
```

---

## Processing flow

When a user clicks "Mark as paid" on the scheduled page, or when a transaction is auto-matched from a bank import:

```
POST /api/scheduled/[id]/process
  │
  ├── 1. Fetch ScheduledTransaction by id
  │
  ├── 2. Create Transaction record
  │       date        = today (or matched bank transaction date)
  │       description = scheduled.name
  │       amount      = scheduled.amount  (negative for expenses)
  │       envelopeId  = scheduled.envelopeId
  │       categoryId  = scheduled.categoryId
  │       scheduledTxnId = scheduled.id
  │       source      = "scheduled"
  │
  ├── 3. Advance nextDueDate
  │       calculateNextDueDate(frequency, today, dayOfMonth)
  │
  ├── 4. Update ScheduledTransaction
  │       nextDueDate   = result from step 3
  │       lastProcessed = today
  │       isActive      = false  (if frequency === "once")
  │
  └── 5. Return { transaction, nextDueDate }
```

### Skip flow

"Skip this occurrence" advances the date without creating a transaction. Useful for a mortgage payment that you know has already been taken directly.

```
POST /api/scheduled/[id]/skip
  │
  ├── Advance nextDueDate (same logic as process)
  └── Do NOT create a Transaction record
```

---

## Mortgage repayment sync

When a mortgage's repayment fields are updated via `PUT /api/mortgage/[id]`, the API automatically upserts the linked `ScheduledTransaction`. No separate API call is needed from the UI.

```typescript
// Inside PUT /api/mortgage/[id]
if (data.repaymentAmount && data.repaymentFrequency) {
  await prisma.scheduledTransaction.upsert({
    where: { mortgageId: mortgage.id },  // requires @@unique([mortgageId])
    update: {
      amount:      data.repaymentAmount,
      frequency:   data.repaymentFrequency,
      dayOfMonth:  data.repaymentDayOfMonth ?? null,
      nextDueDate: calculateNextDueDate(
                     data.repaymentFrequency,
                     data.repaymentStartDate ?? new Date(),
                     data.repaymentDayOfMonth
                   ),
      name: `${mortgage.name} repayment`,
    },
    create: {
      mortgageId:  mortgage.id,
      name:        `${mortgage.name} repayment`,
      amount:      data.repaymentAmount,
      frequency:   data.repaymentFrequency,
      dayOfMonth:  data.repaymentDayOfMonth ?? null,
      nextDueDate: calculateNextDueDate(
                     data.repaymentFrequency,
                     data.repaymentStartDate ?? new Date(),
                     data.repaymentDayOfMonth
                   ),
      source: 'manual',
    }
  })
}
```

**Why `@@unique([mortgageId])`?** Prisma's `upsert` requires a unique field or compound in the `where` clause. Without this constraint, the upsert fails with a Prisma validation error. The migration must include this alongside the `mortgageId` FK column.

---

## Fortnightly vs monthly repayment maths

Fortnightly repayments are **not** monthly ÷ 2. There are 26 fortnights in a year, not 24.

```typescript
// src/lib/calculations/mortgage.ts

export function calculateMonthlyRepayment(
  principal: number,
  annualRate: number,
  termYears: number
): number {
  const r = annualRate / 100 / 12
  const n = termYears * 12
  return (principal * r * Math.pow(1 + r, n)) / (Math.pow(1 + r, n) - 1)
}

export function calculateFortnightlyRepayment(
  principal: number,
  annualRate: number,
  termYears: number
): number {
  // Annual repayment is the same either way
  // Distribute over 26 fortnights, not 24
  const monthly = calculateMonthlyRepayment(principal, annualRate, termYears)
  return (monthly * 12) / 26
}
```

The mortgage form pre-fills the `repaymentAmount` field with the calculated value for the selected frequency. The user can override it.

---

## Akahu auto-matching

When a new transaction is imported from Akahu, `tryMatchToScheduled()` in `src/lib/akahu/sync.ts` looks for a candidate scheduled transaction:

```
Match criteria (all must pass):
  envelopeId  — same envelope as the scheduled transaction
  amount      — within ±1% of scheduled amount (banks can add small fees)
  nextDueDate — within ±5 days of the imported transaction date
  isActive    — true
```

If a match is found, the imported transaction is linked (`scheduledTxnId`) and `nextDueDate` is advanced. The transaction is flagged `source = 'akahu'` and shows an "Auto-matched" badge in the UI for user review.

The ±1% amount tolerance handles cases like a $500.00 mortgage payment arriving as $500.43 due to a small fee. The ±5 day date window handles weekends and public holidays shifting the actual payment date.

---

## Dashboard widget

The "Upcoming payments" card on the dashboard calls `GET /api/scheduled/due?days=14` and shows the next 5 results. Each row shows:

- Name
- Amount
- Due date (relative: "in 3 days", "tomorrow", "today")
- Envelope chip
- Overdue indicator if `nextDueDate` is in the past

---

## UI — Scheduled page layout

```
/scheduled
├── Upcoming (left column, next 30 days)
│   ├── Cards per scheduled item
│   │   ├── Name + frequency badge
│   │   ├── Amount + envelope chip
│   │   ├── Due date (relative)
│   │   ├── [Mark as paid] button → POST /process
│   │   └── [Skip] button → POST /skip
│   └── Overdue items — red left border
│
└── All scheduled (right column)
    ├── Table: name | frequency | amount | next due | last processed | active toggle
    ├── [Add scheduled transaction] button
    └── Edit / soft-delete per row
```

---

## Soft delete rationale

`isActive = false` rather than hard delete. This preserves the `scheduledTxnId` link on historical `Transaction` records — if you hard-deleted the schedule, those transactions would have a dangling FK (or cascade-delete the transaction history, which is worse). Soft delete lets you see that a transaction was generated by a now-inactive schedule.

---

## Related notes

- [[Finance App - Database Schema]]
- [[Finance App - API Route Reference]]
- [[Finance App - Akut Detection Algorithm]]
- [[Akahu Integration - Architecture & Pseudocode]]
