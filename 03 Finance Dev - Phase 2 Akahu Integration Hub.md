---

status: Planned 
priority: Medium 
due_date: 2026-06-01 
project_id: FinanceDev-2026 
phase: "Phase 2: Akahu Integration" 
tags:

- open-banking
- akahu
- integration
- finance-app
- next-js
- prisma
- nz

---

# 🏦 Phase 2: Akahu Open Banking Integration

This phase integrates Akahu as the open banking data source for the app. Since this is a personal app, there is no OAuth flow — authentication uses static tokens from `my.akahu.nz/apps`. The integration covers account syncing, transaction import with upsert/dedup logic, pending transaction display, scheduled transaction matching, and the Settings UI for managing the connection.

> **Dependency:** Phase 1 must be complete before starting this phase. Specifically: the `openBankingRef` unique constraint on `Transaction`, the `ScheduledTransaction` model, and the Settings page scaffold are all required.

## 🎯 Goals

- Store Akahu credentials securely as environment variables (never in DB).
- Sync bank accounts from Akahu and let the user map each to a budget envelope.
- Import settled transactions with full upsert/dedup logic using Akahu's stable `_id`.
- Display pending transactions live (no DB persistence — rebuild on every fetch).
- Auto-match imported transactions to scheduled transactions within a ±5-day window.
- Surface sync status and controls in the Settings page.

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

- [ ] Add `BankAccount` model (akahuId, name, bankName, type, balance, envelopeId FK, lastRefreshed) [priority:: 1] #Schema #Prisma
- [ ] Add `AkahuSyncLog` model (bankAccountId, status, newCount, updatedCount, skippedCount, syncedFrom, syncedTo) [priority:: 1] #Schema #Prisma
- [ ] Add `bankAccountId` FK to `Transaction` (nullable) [priority:: 1] #Schema #Prisma
- [ ] Add `akahuType` and `akahuMerchant` fields to `Transaction` [priority:: 1] #Schema #Prisma
- [ ] Confirm `openBankingRef` has `@unique` constraint (landed in Phase 1) [priority:: 1] #Schema #Prisma
- [ ] Write and test migration file for Phase 2 schema additions [priority:: 1] #Schema #Migration

### Types

- [ ] Create `src/lib/akahu/types.ts` — AkahuTransaction, AkahuAccount, AkahuTransactionType, response wrappers [priority:: 1] #Types
- [ ] Update `src/types/index.ts` — add BankAccount, AkahuSyncLog [priority:: 1] #Types

### Akahu Client

- [ ] Create `src/lib/akahu/client.ts` — typed HTTP wrapper with auth headers [priority:: 1] #Akahu #Client
- [ ] Implement `getMe()` — validate credentials [priority:: 1] #Akahu #Client
- [ ] Implement `getAccounts()` — fetch all connected accounts [priority:: 1] #Akahu #Client
- [ ] Implement `getTransactions()` — paginated loop until `cursor.next === null` [priority:: 1] #Akahu #Client
- [ ] Implement `getPendingTransactions()` — single fetch, no pagination [priority:: 1] #Akahu #Client
- [ ] Implement `requestRefresh()` — POST /refresh, only called from explicit UI button [priority:: 2] #Akahu #Client

### Sync Logic

- [ ] Create `src/lib/akahu/sync.ts` — `syncAccount()` with SyncLog creation and error handling [priority:: 1] #Akahu #Sync
- [ ] Implement `upsertTransaction()` — create/update/skip logic keyed on `openBankingRef` [priority:: 1] #Akahu #Sync
- [ ] Apply 2-day overlap on `syncedFrom` to catch bank mutations to recent transactions [priority:: 1] #Akahu #Sync
- [ ] Implement `tryMatchToScheduled()` — match on envelope + amount ±1% + date ±5 days [priority:: 2] #Akahu #Sync
- [ ] Write unit tests for `upsertTransaction()` with fixture data (new, update, skip cases) [priority:: 2] #Akahu #Testing

### API Routes

- [ ] Create `GET /api/akahu/test` — call `/me`, return connection status [priority:: 1] #API #Akahu
- [ ] Create `GET /api/akahu/accounts` — fetch from Akahu, upsert BankAccount records, return list [priority:: 1] #API #Akahu
- [ ] Create `POST /api/akahu/sync` — iterate active BankAccounts sequentially, call syncAccount() [priority:: 1] #API #Akahu
- [ ] Create `POST /api/akahu/sync/[accountId]` — single account sync [priority:: 2] #API #Akahu
- [ ] Create `GET /api/akahu/pending` — fetch live from Akahu, return without persisting [priority:: 2] #API #Akahu
- [ ] Create `GET /api/akahu/status` — return last AkahuSyncLog per BankAccount [priority:: 2] #API #Akahu

### Settings UI — Open Banking Section

- [ ] Add "Open banking" section to Settings page [priority:: 2] #UI #Settings #Akahu
- [ ] Connection status card — GET `/api/akahu/test` on load, green/red chip [priority:: 2] #UI #Settings
- [ ] Bank accounts list — name, bank, balance, last synced, envelope mapping dropdown per row [priority:: 2] #UI #Settings
- [ ] Sync button — POST `/api/akahu/sync`, show progress, display "X new, Y updated" result [priority:: 2] #UI #Settings
- [ ] Force refresh button — POST `/api/akahu/accounts` to re-pull from Akahu (respects 1hr rest period) [priority:: 3] #UI #Settings
- [ ] Pending transactions toggle — show/hide pending on the main transactions page [priority:: 3] #UI #Settings

### Transactions Page — Akahu Additions

- [ ] Add Akahu/bank badge on transactions where `source === 'akahu'` [priority:: 3] #UI #Transactions
- [ ] Add "Unreviewed" filter — `envelopeId === null && source === 'akahu'` [priority:: 3] #UI #Transactions
- [ ] Add bulk-assign — select multiple unreviewed transactions → assign envelope [priority:: 3] #UI #Transactions
- [ ] Add "Auto-matched" badge on transactions linked via `tryMatchToScheduled()` [priority:: 3] #UI #Transactions
- [ ] Add pending transactions section (live fetch, separate from settled list) [priority:: 3] #UI #Transactions

### Environment & Config

- [ ] Add `AKAHU_USER_TOKEN` and `AKAHU_APP_TOKEN` to `.env.example` with placeholder values [priority:: 1] #Config
- [ ] Confirm env vars are excluded from Docker image build args (runtime only) [priority:: 1] #Config #Docker
- [ ] Add Akahu env vars to deployment notes [priority:: 2] #Config #Documentation

### Verification

- [ ] GET `/api/akahu/test` returns success with real tokens
- [ ] GET `/api/akahu/accounts` returns accounts and creates BankAccount records
- [ ] Map one BankAccount to an envelope
- [ ] POST `/api/akahu/sync` — confirm transactions import, check for duplicates on re-run
- [ ] Re-run sync — confirm existing transactions show `skipped`, no duplicates
- [ ] Modify a synced transaction description manually in DB — re-run sync, confirm `updated`
- [ ] Check pending transactions display and confirm nothing written to Transaction table
- [ ] Confirm auto-match fires for a transaction within ±5 days of a scheduled item
- [ ] `docker compose up --build` passes after Phase 2 migration

## 📝 Spoke Notes & Documentation

- [[Akahu Integration - Architecture & Pseudocode]]
- [[Finance App - Database Schema]]
- [[Finance App - Environment Variables]]
- [[Finance App - API Route Reference]]

---

## ➡️ Next Phase

- [[03 Finance Dev - Phase 3 Hub]]