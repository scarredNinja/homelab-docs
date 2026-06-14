---
project_id: FinanceDev-2026
tags:
  - finance-dev
  - reference
status: Reference
phase: 'Phase 1: Foundation'
---

# 🤖 Finance App — Akut Detection Algorithm

Design document for the Akut recurring transaction detection feature. Covers the detection algorithm, confidence scoring, normalisation logic, and the UI confirm/dismiss flow.

---

## Purpose

Akut analyses historical transactions to identify recurring patterns — subscriptions, regular bills, automatic payments — and proposes creating a `ScheduledTransaction` for each one. The user reviews the proposals and either confirms (creating the schedule) or dismisses them.

The feature is entirely stateless at the API level. Detection results are never written to the database — they are computed on demand, held in React state, and discarded when the user navigates away or runs a fresh scan.

---

## API endpoint

```
POST /api/akut/detect-recurring
Body: { lookbackDays?: number }   // default 90
```

**Response:**

```typescript
interface DetectionResponse {
  proposed: DetectedPattern[]
  scannedTransactions: number
  lookbackDays: number
}

interface DetectedPattern {
  normalizedName: string           // cleaned merchant name used for grouping
  matchedTransactions: Transaction[] // the raw transactions that matched
  suggestedFrequency: RepaymentFrequency
  averageAmount: number            // negative for expenses
  amountVariance: number           // % variance across occurrences
  intervalVarianceDays: number     // std dev of intervals in days
  confidence: number               // 0–1 score
  alreadyScheduled: boolean        // true if a ScheduledTransaction already exists
  matchingScheduledId?: number     // if alreadyScheduled, link to edit it
}
```

---

## Algorithm — step by step

### Step 1: Fetch transactions

```typescript
const transactions = await prisma.transaction.findMany({
  where: {
    date: { gte: subtractDays(new Date(), lookbackDays) },
    amount: { lt: 0 },    // expenses only — credits rarely recur in a useful way
    isConfirmed: true,
  },
  orderBy: { date: 'asc' },
})
```

Only debit transactions are analysed. Credit transactions (salary, refunds) could theoretically be detected too, but produce noisy results for household finance use.

### Step 2: Normalise and group

```typescript
// Group transactions by their normalised description
const groups = new Map<string, Transaction[]>()

for (const txn of transactions) {
  const key = normalizeTransactionName(txn.description)
  if (!key) continue   // skip if normalisation produces empty string
  if (!groups.has(key)) groups.set(key, [])
  groups.get(key)!.push(txn)
}

// Discard groups with fewer than 3 occurrences — not enough signal
const candidates = [...groups.entries()].filter(([, txns]) => txns.length >= 3)
```

### Step 3: Analyse intervals

For each candidate group, compute the gaps between consecutive transactions:

```typescript
function analyseIntervals(transactions: Transaction[]): {
  medianInterval: number
  varianceDays: number
} {
  const sorted = [...transactions].sort((a, b) =>
    new Date(a.date).getTime() - new Date(b.date).getTime()
  )

  const intervals: number[] = []
  for (let i = 1; i < sorted.length; i++) {
    const days = daysBetween(sorted[i - 1].date, sorted[i].date)
    intervals.push(days)
  }

  const median = medianOf(intervals)
  const variance = stdDevOf(intervals)

  return { medianInterval: median, varianceDays: variance }
}
```

### Step 4: Map interval to frequency

```typescript
function inferFrequency(medianDays: number): RepaymentFrequency | null {
  if (medianDays >= 5  && medianDays <= 9)   return 'weekly'
  if (medianDays >= 12 && medianDays <= 16)  return 'fortnightly'
  if (medianDays >= 25 && medianDays <= 35)  return 'monthly'
  if (medianDays >= 58 && medianDays <= 68)  return 'monthly'  // bi-monthly edge case
  if (medianDays >= 350 && medianDays <= 380) return 'annually'
  return null  // no recognisable pattern — discard this group
}
```

Groups where `inferFrequency` returns `null` are silently dropped from results. These are irregular transactions that happen to share a description.

### Step 5: Score confidence

```typescript
function scoreConfidence(
  occurrences: number,
  intervalVarianceDays: number,
  amountVariancePct: number
): number {
  // Base score from occurrence count
  let score = occurrences === 3 ? 0.60
            : occurrences === 4 ? 0.75
            : 0.80   // 5+

  // Reward tight intervals
  if (intervalVarianceDays < 1) score += 0.15
  else if (intervalVarianceDays < 2) score += 0.10
  else if (intervalVarianceDays < 4) score += 0.05

  // Reward consistent amounts
  if (amountVariancePct < 0.01) score += 0.10   // < 1% variance
  else if (amountVariancePct < 0.05) score += 0.05  // < 5%

  return Math.min(score, 1.0)
}
```

### Step 6: Check for existing schedules

Before returning results, check whether each proposed pattern already has a matching `ScheduledTransaction`:

```typescript
const existing = await prisma.scheduledTransaction.findFirst({
  where: {
    name: { contains: pattern.normalizedName, mode: 'insensitive' },
    isActive: true,
  }
})

pattern.alreadyScheduled = !!existing
pattern.matchingScheduledId = existing?.id
```

This prevents the UI from proposing a duplicate schedule for something already being tracked.

---

## Normalisation — `normalizeTransactionName`

Lives in `src/lib/calculations/schedule.ts`. Strips everything that varies between occurrences of the same merchant, leaving a stable key for grouping.

```typescript
export function normalizeTransactionName(description: string): string {
  return description
    .toLowerCase()
    .trim()
    // Strip DD/MM, DD/MM/YYYY, DD-MM-YYYY date patterns
    .replace(/\b\d{1,2}[\/\-]\d{1,2}([\/\-]\d{2,4})?\b/g, '')
    // Strip month names + optional year: "Apr 2026", "April", "apr"
    .replace(/\b(jan|feb|mar|apr|may|jun|jul|aug|sep|oct|nov|dec)\w*\s*\d{0,4}\b/gi, '')
    // Strip standalone 4+ digit numbers (transaction refs, card numbers)
    .replace(/\b\d{4,}\b/g, '')
    // Strip masked card suffixes: "xx1234", "xxxx 5678"
    .replace(/\bx+\s*\d+\b/gi, '')
    // Strip common noise prefixes
    .replace(/^(eftpos|visa|mastercard|direct debit|ap )\s*/i, '')
    // Collapse whitespace
    .replace(/\s+/g, ' ')
    .trim()
}
```

**Examples:**

|Raw description|Normalised|
|---|---|
|`Netflix 01/04`|`netflix`|
|`EFTPOS 84721 Countdown`|`countdown`|
|`AP ANZ MORTGAGE Apr 2026`|`anz mortgage`|
|`DIRECT DEBIT CONTACT ENERGY 00293847`|`contact energy`|
|`Spotify NZ xx1234`|`spotify nz`|

---

## Confidence score guide

|Score|What it means|
|---|---|
|0.90 – 1.00|Very high — 5+ occurrences, tight intervals, consistent amount|
|0.75 – 0.89|High — 4+ occurrences, minor variance|
|0.60 – 0.74|Medium — 3 occurrences, reasonable consistency|
|Below 0.60|Not returned — insufficient signal|

The UI renders confidence as a progress bar. Colour: green ≥ 0.85, amber ≥ 0.65, red below (though red shouldn't appear given the floor).

---

## UI flow — Akut page

The detection section is a separate panel on the existing Akut page, not a new page.

```
[Scan for recurring transactions]  ← triggers POST, shows spinner
Last scanned: 3 minutes ago

┌─ Detected: Netflix ──────────────────────────────────────────┐
│  Monthly · avg $18.99 · 6 occurrences                        │
│  Confidence ████████████░░  92%                              │
│  Last seen: 01 Apr · 01 Mar · 01 Feb                         │
│                                                              │
│  [Create schedule]   [Dismiss]                               │
└──────────────────────────────────────────────────────────────┘

┌─ Detected: Contact Energy ───────────────────────────────────┐
│  Monthly · avg $143.20 · 4 occurrences · ±$12.40 variance    │
│  Confidence ████████░░░░░  75%                               │
│  Last seen: 28 Mar · 27 Feb · 31 Jan                         │
│                                                              │
│  [Create schedule]   [Dismiss]                               │
└──────────────────────────────────────────────────────────────┘

┌─ Already scheduled: ANZ Mortgage ────────────────────────────┐
│  Fortnightly · $892.00 · ✅ Already tracked                  │
│  [Edit schedule →]                                           │
└──────────────────────────────────────────────────────────────┘
```

### Confirm flow

Clicking "Create schedule" opens the `ScheduledTransaction` create dialog, pre-filled with:

```
name        ← pattern.normalizedName (user can edit)
amount      ← pattern.averageAmount
frequency   ← pattern.suggestedFrequency
nextDueDate ← estimated next occurrence (last matched date + one interval)
```

On save, the created `ScheduledTransaction` gets `source = 'akut_detected'`.

After confirming, the API call also backfills `scheduledTxnId` on the historical transactions that were matched:

```typescript
await prisma.transaction.updateMany({
  where: {
    id: { in: pattern.matchedTransactions.map(t => t.id) }
  },
  data: { scheduledTxnId: newSchedule.id }
})
```

This means historical transactions are properly linked and won't appear as unmatched noise in future scans.

### Dismiss flow

Dismiss removes the card from the UI (local React state only — nothing written to DB). If the user runs a fresh scan, the same pattern will reappear. There is no persistent ignore list in Phase 1. A future enhancement could add a `AkutIgnore` table to persist dismissals.

---

## Stateless rationale

Detection results are not stored because:

1. They are derived entirely from existing `Transaction` data — recomputing is cheap.
2. Storing them would require a cleanup job or TTL to avoid stale results.
3. The "confirm" action writes to `ScheduledTransaction` immediately — there's no intermediate state that needs to persist.
4. Dismissals are rare enough that losing them on page reload is acceptable in Phase 1.

---

## Edge cases

**Bi-monthly billing** — some utilities bill every 2 months (~60 day interval). The interval map handles this but confidence will be lower since you get fewer occurrences in a 90-day lookback. Extend `lookbackDays` to 180 in the UI as an option.

**Amount variance on utilities** — power, gas, and water bills vary significantly by season. The algorithm uses a looser ±5% amount check for confidence but still groups them by normalised name regardless of amount.

**Public holidays shifting payment dates** — a monthly payment due on the 1st may arrive on the 3rd if there's a holiday. The ±4 day interval variance tolerance handles this without losing confidence points.

**Same merchant, multiple accounts** — if "Countdown" appears as both a weekly grocery spend and occasional large shop, the grouping will merge them. The normalised name grouping is intentionally coarse — false positives are filtered by the interval analysis (irregular spending won't produce a clean weekly pattern).

---

## Related notes

- [[Finance App - Scheduled Transactions Design]]
- [[Finance App - Database Schema]]
- [[Finance App - API Route Reference]]
- [[Akahu Integration - Architecture & Pseudocode]]
