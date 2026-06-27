# Satsang Vihar FMS — Community Data & Analytics Platform

## Overview

A full-stack data management and analytics system built for a non-profit religious organization that tracks monthly financial contributions ("Istavrity") from 100+ member families across multiple villages. The project replaced manual registers and ad-hoc Excel sheets with a structured database system — first as a Python desktop app, then migrated to a full web application with a custom client-side analytics engine.

## The Problem

Before this system, the organization had no reliable way to:
- Track month-over-month contribution history per family
- Generate monthly financial reports without manual compilation
- Reprint receipts without re-entering data
- Identify which villages or families were falling behind on contributions

## Data Architecture

**Per-month database strategy:** each month's data lives in its own SQLite file (desktop) or IndexedDB partition (web) — isolating historical records so past months are never accidentally modified, while still supporting cross-month historical queries.

```
Excel File → Python import → SQLite (current month)
                                  ↓ (end of month)
                          Monthly Snapshot DB (archive)
                                  ↓ (migration to web)
                  serialize_sqlite_to_json.py (reads all 16 DBs)
                                  ↓
                  clean_database_json.py (dedup, schema fix)
                                  ↓
                  initial_data.json (~8.4 MB consolidated)
                                  ↓
                  IndexedDB (browser-native store, partitioned by month)
```

## Tech Stack

**Desktop:** Python, Tkinter, SQLite, ReportLab (PDF receipts), openpyxl/pandas (Excel import), Google Drive API (OAuth2 backup)
**Web:** Vanilla JavaScript (ES6+, 16 modules), IndexedDB, SheetJS (Excel import/export), custom dual-theme (light/dark) CSS system

## Analytics Engine

A standalone, self-contained analytics dashboard (`analytics.html`) processes all monthly database partitions client-side — no server required — to produce:

- **Historical growth tracking** — month-over-month collection growth, new registrations, and average contribution per active family
- **Village health summaries** — collection totals and active-family ratios grouped by village
- **Declining-village trend detection** — tracks active family counts per village over time (e.g. `Jul: 15 → Aug: 14 → Sep: 12`) to flag falling participation
- **Inactive cohort segmentation** — identifies families inactive in over 70% of recorded months
- **Data-quality audits** — flags missing fields (address, village, etc.) relative to the rest of the dataset
- **Risk alerts** — automatic flags when a village's contributions drop >15% month-over-month or inactivity exceeds 25%, with suggested next steps

## Engineering Challenges & Solutions

**Heavy client-side aggregation without blocking the UI** — built an asynchronous aggregation pipeline that reads all IndexedDB partitions, caches mappings in memory, and renders results incrementally.

**Migrating 16 SQLite databases with inconsistent schemas** — early databases had fewer columns than later ones. Wrote Python inspection scripts to detect schema drift, then a consolidation script that merges all databases with backward-compatible defaults for missing fields.

**Theme-flash on load** — reading the saved theme from `localStorage` asynchronously caused a visible flash of the wrong theme on page load; fixed with a tiny blocking script at the very top of `<head>` that sets the theme attribute before anything renders.

**IndexedDB transaction lifecycle bugs** — `await`-ing between IndexedDB operations was invalidating transactions; rewrote all write paths to batch operations within a single active transaction.

## Scale

| Metric | Value |
|---|---|
| Historical months managed | 11+ (Jul 2025 – May 2026) |
| SQLite databases consolidated | 16 |
| Consolidated dataset size | ~8.4 MB JSON |
| JavaScript modules | 16 (170KB+) |
| Report/analytics views | 8 distinct types |
| Python scripts (ETL, reporting, backup) | 50+ |

---
*Migrated from a Python desktop app to a full-stack web SPA with a custom client-side analytics engine — developed solo.*
