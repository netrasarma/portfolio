# LeafLedger Pro — Offline-First Agricultural ERP

## Overview

LeafLedger Pro is a multi-platform, offline-first financial management system built for **tea garden agents** (middlemen) across Assam, Tripura, and Arunachal Pradesh in Northeast India. It digitizes the entire tea-leaf supply chain: daily collection at remote farms, payment and advance tracking, factory reconciliation, and monthly statement generation — replacing handwritten paper ledgers (*khata* books) with a synchronized digital system across mobile, desktop, and the cloud.

Built solo, full-stack: backend, desktop UI, mobile apps, and cloud infrastructure.

## The Problem

Tea garden agents buy leaves daily from 20–150 smallholder farmers, aggregate them for factory delivery, and settle payments monthly — minus any cash advances given through the month. Running this on paper caused real financial loss:

- **Manual arithmetic errors** on weight/rate calculations led to an estimated 3–5% of monthly revenue lost to over/under-payment
- **Cash advances tracked in separate notebooks**, frequently forgotten and never deducted at settlement
- **No audit trail** — disputes between agents and farmers over weights or rates had no record to resolve them
- **Zero connectivity in the field** — tea gardens often have no cellular signal, ruling out standard cloud-based apps
- **No accountability for field collectors** — no way to trace who recorded a given transaction

## The Solution — System Architecture

```
┌───────────────────┬───────────────────┬────────────────┐
│   COLLECTOR APP   │    AGENT MOBILE   │  DESKTOP PANEL │
│   (Field Staff)   │   (Field Agent)   │ (Admin/Office) │
├───────────────────┼───────────────────┼────────────────┤
│ Offline pickup    │ KPI Dashboards    │ Bulk billing   │
│ Bluetooth print   │ Rate Management   │ A4 PDF Gen     │
│ Local SQLite      │ Local SQLite      │ SQLite DB      │
└─────────┬─────────┴─────────┬─────────┴────────┬───────┘
          └───────────┬───────┴──────────────────┘
                  (Bi-directional Sync)
                       ▼
              ┌───────────────────┐
              │ Vercel API Proxy  │  (shields DB credentials)
              └─────────┬─────────┘
                        ▼
              ┌───────────────────┐
              │  Supabase Cloud   │  (PostgreSQL, Auth, RLS)
              └───────────────────┘
```

**Core features:**
- **Automated calculations** — collectors enter gross weight and moisture deduction; net weight and payable amount are calculated instantly against preconfigured rate tables
- **Integrated advance ledger** — cash advances are logged digitally and auto-deducted at monthly settlement
- **Instant field receipts** — the Collector App prints a physical receipt via Bluetooth thermal printer (ESC/POS protocol) the moment a weighment is saved
- **Offline-first sync** — all devices write to local SQLite; a sync engine queues changes and pushes them to cloud PostgreSQL once connectivity returns

## Tech Stack

`Python` · `SQLite` · `PostgreSQL` (via Supabase) · `React Native` · `Vercel` (serverless API proxy)

## Engineering Challenges & Solutions

**Bi-directional sync without conflicts** — every record carries `created_at`, `updated_at`, and a client-generated `sync_id`. On sync, the most recent `updated_at` wins; a tombstone table tracks deletions so they propagate cleanly across devices.

**Securing cloud credentials on distributed desktop clients** — rather than embedding Supabase keys in client code, a serverless proxy on Vercel validates a rotation-capable auth token and holds the real database secrets server-side in environment variables.

**Low-cost phone authentication for rural users** — standard SMS OTP services (Twilio, Firebase Phone Auth) were too expensive for the target market. Built a pseudo-email mapping layer (`9876543210` → `phone@llpro.com`) so phone numbers could use Supabase's free email-auth tier with an MPIN as the password.

**SQLite write conflicts during background sync** — solved with WAL (Write-Ahead Logging) mode, a thread-safe `SyncManager` singleton, and a retry decorator with exponential backoff for locked-database errors.

**Silent auto-updates on Windows** — Windows locks running executables, so the updater spawns as an independent subprocess, waits for the main app to release file handles, renames the locked `.exe`, and replaces it — all without admin privileges.

## Scale & Impact

| Metric | Value |
|---|---|
| Daily weighment transactions | 50–200 across 1–3 field collectors |
| Daily leaf volume managed | 1,500–6,000 kg |
| Farmers managed per agent | 20–150 |
| Monthly trade reconciled per agent | up to ₹1,500,000 |
| Database schema | 11-table normalized relational design |
| Admin time for monthly billing | reduced from 24–36 hours to seconds per farmer |

**Human impact:** Physical Bluetooth-printed receipts gave largely paper-record-free farmers concrete proof of their earnings for the first time — reducing disputes and rebuilding trust between farmers and purchasing agents.

---
*Solo-developed, full-stack: Python backend, mobile apps, desktop UI, and cloud sync infrastructure.*
