---

status: Completed
priority: High
due_date: 2026-06-01
completed_date: 2026-05-22
project_id: FinanceDev-2026 
phase: "Phase 1: Foundation" 
tags:

- finance-app
- next-js
- prisma
- schema
- settings
- categories

---

# 🏗️ Phase 1: Foundation

This phase covers all schema changes, the settings profile system, configurable budget categories, mortgage repayment scheduling, transaction categories and splitting, scheduled/recurring transactions, and the Akut AI recurring detection upgrade. These are the core data model and API changes that everything else depends on.

## 🎯 Goals

- Land all Prisma schema changes in a single migration so Phase 2 has a stable foundation.
- Build the Settings profile page with currency, alert thresholds, and category management.
- Implement configurable budget categories (add/edit/delete/reorder).
- Add category support to transactions and implement transaction splitting.
- Build the scheduled transactions system (one-off and recurring).
- Upgrade Akut to detect and propose recurring transaction patterns.
- Implement mortgage repayment scheduling (frequency, amount, day).

## 🔗 Action Items

```dataviewjs
// ── Tasks for this phase only ─────────────────────────────────────────────────
const PRIORITY_FALLBACK = 999;
const thisPhase = dv.current().phase;

if (!thisPhase) {
    dv.paragraph("⚠️ No `phase` field found in this file's frontmatter.");
} else {
    const pages = dv.pages('"10 - Projects"')
        .where(p =>
            p.phase === thisPhase &&
            !p.file.folder.includes("_Archive")
        );

    const tasks = pages
        .flatMap(p => p.file.tasks
            .where(t =>
                !t.completed &&
                !t.tags.includes("#Later") &&
                !t.tags.includes("#MuchLater")
            )
            .map(t => ({ task: t, file: p.file }))
        )
        .array();

    if (tasks.length === 0) {
        dv.paragraph("✅ No open tasks for this phase.");
    } else {
        tasks.sort((a, b) => {
            const pa = a.task.priority !== undefined ? Number(a.task.priority) : PRIORITY_FALLBACK;
            const pb = b.task.priority !== undefined ? Number(b.task.priority) : PRIORITY_FALLBACK;
            if (pa !== pb) return pa - pb;
            if (a.file.name !== b.file.name) return a.file.name.localeCompare(b.file.name);
            return (a.task.line ?? 0) - (b.task.line ?? 0);
        });

        dv.taskList(tasks.map(t => t.task), false);
    }
}
```

### Schema & Migration

- [ ] Add `AppSettings` model (currency, dateFormat, alertThresholds) [priority:: 1] #Schema #Prisma
- [ ] Add `ScheduledTransaction` model with frequency, nextDueDate, mortgageId, openBankingRef [priority:: 1] #Schema #Prisma
- [ ] Add `categoryId` FK to `Transaction` (nullable, backward compat) [priority:: 1] #Schema #Prisma
- [ ] Add `scheduledTxnId` FK to `Transaction` [priority:: 1] #Schema #Prisma
- [ ] Add `openBankingRef` field to `Transaction` with unique constraint [priority:: 1] #Schema #Prisma
- [ ] Add repayment fields to `Mortgage` (amount, frequency, dayOfMonth, startDate) [priority:: 1] #Schema #Prisma
- [ ] Add `isSystem` flag to `Category` model [priority:: 1] #Schema #Prisma
- [ ] Add `@@unique([mortgageId])` to `ScheduledTransaction` to enable upsert pattern [priority:: 1] #Schema #Prisma
- [ ] Write and test single migration file for all batch 2 schema changes [priority:: 1] #Schema #Migration
- [ ] Confirm `docker compose up --build` passes cleanly after migration [priority:: 1] #Schema #Testing

### Types & Shared Utilities

- [ ] Update `src/types/index.ts` — add AppSettings, ScheduledTransaction, extend Transaction and Mortgage [priority:: 2] #Types
- [ ] Create `src/lib/format.ts` — `formatCurrency()` and `formatDate()` using AppSettings [priority:: 2] #Utilities
- [ ] Create `src/lib/calculations/schedule.ts` — `calculateNextDueDate()` and `normalizeTransactionName()` [priority:: 2] #Utilities
- [ ] Add `calculateFortnightlyRepayment()` to `src/lib/calculations/mortgage.ts` (26 payments/year, not 24) [priority:: 2] #Utilities #Mortgage

### Settings API & Page

- [ ] Create `GET /api/settings` — upsert id=1 with defaults on first call [priority:: 2] #API #Settings
- [ ] Create `PUT /api/settings` — partial update, always upsert id=1 [priority:: 2] #API #Settings
- [ ] Build Settings page — General section (currency, date format, envelope view) [priority:: 3] #UI #Settings
- [ ] Build Settings page — Alerts section (mortgage refix alert days, low envelope threshold) [priority:: 3] #UI #Settings
- [ ] Build Settings page — Data section (move Load/Clear test data buttons from dashboard) [priority:: 3] #UI #Settings
- [ ] Add Settings nav item to Sidebar (gear icon) [priority:: 3] #UI #Sidebar

### Categories

- [ ] Create `GET /api/categories` — list all, ordered by sortOrder [priority:: 2] #API #Categories
- [ ] Create `POST /api/categories` — create with name, colour, sortOrder [priority:: 2] #API #Categories
- [ ] Create `PUT /api/categories/[id]` — update name/colour/sortOrder [priority:: 2] #API #Categories
- [ ] Create `DELETE /api/categories/[id]` — 409 if envelopes exist, block if isSystem [priority:: 2] #API #Categories
- [ ] Build Settings page — Categories section (inline add/edit/delete/reorder, lock icon on system categories) [priority:: 3] #UI #Settings #Categories
- [ ] Update Budget page envelope dialog — category dropdown pulls from `/api/categories` [priority:: 3] #UI #Budget

### Transaction Categories & Splitting

- [ ] Create `POST /api/transactions/[id]/split` — validate splits sum to original, replace with N records [priority:: 2] #API #Transactions
- [ ] Add category column to transactions table (colour dot + name) [priority:: 3] #UI #Transactions
- [ ] Add category filter to transactions page (multi-select) [priority:: 3] #UI #Transactions
- [ ] Add "Split" button per transaction row + split dialog [priority:: 3] #UI #Transactions
- [ ] Add "Scheduled" badge on transactions linked to a ScheduledTransaction [priority:: 3] #UI #Transactions

### Scheduled Transactions

- [ ] Create `GET /api/scheduled` — list all, ordered by nextDueDate ASC [priority:: 2] #API #Scheduled
- [ ] Create `POST /api/scheduled` — create manual scheduled transaction [priority:: 2] #API #Scheduled
- [ ] Create `PUT /api/scheduled/[id]` — update [priority:: 2] #API #Scheduled
- [ ] Create `DELETE /api/scheduled/[id]` — soft-delete (isActive = false) [priority:: 2] #API #Scheduled
- [ ] Create `POST /api/scheduled/[id]/process` — create Transaction, advance nextDueDate [priority:: 2] #API #Scheduled
- [ ] Create `GET /api/scheduled/due` — due in next N days (default 7) [priority:: 2] #API #Scheduled
- [ ] Build Scheduled Transactions page — Upcoming column + All Scheduled list [priority:: 3] #UI #Scheduled
- [ ] Add overdue highlight (red left border) on past-due scheduled items [priority:: 3] #UI #Scheduled
- [ ] Add "Skip this occurrence" button (advance date, no Transaction created) [priority:: 3] #UI #Scheduled
- [ ] Add Scheduled nav item to Sidebar (calendar icon) [priority:: 3] #UI #Sidebar
- [ ] Add "Upcoming payments" card to Dashboard (next 5 from `/api/scheduled/due?days=14`) [priority:: 3] #UI #Dashboard

### Mortgage Repayment Scheduling

- [ ] Update mortgage form dialog — add Frequency (Monthly/Fortnightly), Amount, Start Date, Day of Month [priority:: 2] #UI #Mortgage
- [ ] Update `PUT /api/mortgage/[id]` — accept repayment fields, upsert linked ScheduledTransaction [priority:: 2] #API #Mortgage
- [ ] Verify fortnightly calculation uses 26 payments/year not 24 [priority:: 2] #Mortgage #Testing

### Akut Recurring Detection

- [ ] Create `POST /api/akut/detect-recurring` — group by normalised description, analyse intervals, score confidence [priority:: 2] #API #Akut
- [ ] Add "Detected recurring" section to Akut page — confidence bar, sample transactions, Create/Dismiss actions [priority:: 3] #UI #Akut
- [ ] Add "Scan for recurring" button (stateless — results held in React state, not persisted) [priority:: 3] #UI #Akut
- [ ] After confirm: set `source: 'akut_detected'` on ScheduledTransaction, backfill `scheduledTxnId` on historical transactions [priority:: 3] #Akut

### Seed Data Update

- [ ] Update `POST /api/seed` — add AppSettings, ScheduledTransaction records to seed payload [priority:: 3] #Seed
- [ ] Update `DELETE /api/seed` — add new models to purge order [priority:: 3] #Seed

## 📝 Spoke Notes & Documentation

- [[Finance App - Architecture Overview]]
- [[Finance App - Database Schema]]
- [[Finance App - API Route Reference]]
- [[Finance App - Scheduled Transactions Design]]
- [[Finance App - Akut Detection Algorithm]]

---

## ➡️ Next Phase

- [[03 Finance Dev - Phase 2 Akahu Integration Hub]]