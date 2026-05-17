# Market Screener — Data Platform Documentation

This repository is the **single source of truth** for the shared data platform
architecture used across all screener and charting projects.

Every project team must read and follow these standards before writing any code.

---

## Projects on this Platform

| Project | Repository | Market | Status |
|---|---|---|---|
| ASX Screener | [asx-screener](https://github.com/PrashanthPadeti/asx-screener) | Australian Securities Exchange | ✅ Live |
| India Nifty Screener | *(coming soon)* | NSE / BSE | 🔜 Planned |
| US Screener | *(coming soon)* | NYSE / NASDAQ / AMEX | 🔜 Planned |
| Charting | *(coming soon)* | Multi-market | 🔜 Planned |

---

## Architecture Documents

| # | Document | Read this when… |
|---|---|---|
| [01 — Platform Overview](docs/01_platform_overview.md) | Starting anything. Understand the full picture first. |
| [02 — Bronze Layer Guide](docs/02_bronze_layer_guide.md) | Reading or writing to the shared raw data lake. |
| [03 — Naming Conventions](docs/03_naming_conventions.md) | Naming a schema, table, column, file, env var, or cache key. |
| [04 — New Project Checklist](docs/04_new_project_checklist.md) | Bootstrapping a new screener or charting project. |
| [05 — Pipeline Standards](docs/05_pipeline_standards.md) | Writing any ingestion, transform, compute, or build job. |

---

## The Rule

> **Raw Zone and Staging are shared. Every layer after Staging is fully independent per project.**
>
> A schema change, bug fix, or new feature in one project must never affect another project.

```
┌─────────────────────────────────────────────┐
│  SHARED (Platform-owned)                    │
│                                             │
│  Raw Zone  →  staging.*                     │
│  (files)      (DB schema)                   │
└──────────────────────┬──────────────────────┘
                       │  each project reads independently
         ┌─────────────┼─────────────┬──────────────────┐
         ▼             ▼             ▼                  ▼
  ┌────────────┐ ┌───────────┐ ┌──────────┐ ┌──────────────┐
  │ ASX        │ │  INDIA    │ │    US    │ │  CHARTING    │
  │ SCREENER   │ │ SCREENER  │ │ SCREENER │ │              │
  │            │ │           │ │          │ │              │
  │  mkt_au.*  │ │  mkt_in.* │ │ mkt_us.* │ │ charts_mkt.* │
  │  fin_au.*  │ │  fin_in.* │ │ fin_us.* │ │              │
  │ screener   │ │ screener  │ │ screener │ │  charts.*    │
  │   _au.*    │ │   _in.*   │ │   _us.*  │ │              │
  └────────────┘ └───────────┘ └──────────┘ └──────────────┘
         │             │             │                  │
         └─────────────┴─────────────┴──────────────────┘
                                │
                   ┌────────────▼────────────┐
                   │  SHARED INFRASTRUCTURE  │
                   │  users.*  notifications.│
                   │  ai.*  subscriptions.*  │
                   └─────────────────────────┘
```

---

## Quick Reference

### What is Shared vs Independent

| Layer | Shared or Independent | Schema | Owned by |
|---|---|---|---|
| Raw Zone (files) | ✅ Shared | `/opt/data-lake/raw/` | Platform team |
| Staging (DB) | ✅ Shared | `staging.*` | Platform team |
| Market Transform | 🔒 Independent | `mkt_au.*` / `mkt_in.*` / `mkt_us.*` | Each project |
| Financials | 🔒 Independent | `fin_au.*` / `fin_in.*` / `fin_us.*` | Each project |
| Application (Gold) | 🔒 Independent | `screener_au.*` / `screener_in.*` / `screener_us.*` | Each project |
| Auth / Billing | ✅ Shared | `users.*` `notifications.*` `ai.*` | Platform team |

### Shared Staging Schema Tables

All exchanges land in the same `staging.*` schema. Projects filter by exchange:

| Table | All exchanges stored | Projects read with |
|---|---|---|
| `staging.eod_prices` | AU, NSE, BSE, NYSE, NASDAQ | `WHERE exchange = 'AU'` |
| `staging.fundamentals_raw` | All exchanges | `WHERE exchange = 'AU'` |
| `staging.shares_stats` | All exchanges | `WHERE exchange IN ('AU')` |
| `staging.dividends_raw` | All exchanges | `WHERE exchange = 'AU'` |
| `staging.splits_raw` | All exchanges | `WHERE exchange = 'AU'` |
| `staging.short_positions` | ASX (ASIC) only | ASX Screener only |
| `staging.fii_dii_raw` | India (NSE) only | India Screener only |
| `staging.promoter_holdings_raw` | India only | India Screener only |
| `staging.sec_filings_raw` | US only | US Screener only |
| `staging.insider_trades_raw` | US only | US Screener only |

### Project Schema Map (Independent Layers Only)

| Project | Market Transform | Financials | Application |
|---|---|---|---|
| ASX Screener | `mkt_au.*` | `fin_au.*` | `screener_au.*` |
| India Screener | `mkt_in.*` | `fin_in.*` | `screener_in.*` |
| US Screener | `mkt_us.*` | `fin_us.*` | `screener_us.*` |
| Charting | `charts_mkt.*` | — | `charts.*` |

### Raw Zone Path

```
/opt/data-lake/raw/eodhd/exchange=AU/       ← ASX
/opt/data-lake/raw/eodhd/exchange=NSE/      ← India NSE
/opt/data-lake/raw/eodhd/exchange=BSE/      ← India BSE
/opt/data-lake/raw/eodhd/exchange=NYSE/     ← US NYSE
/opt/data-lake/raw/eodhd/exchange=NASDAQ/   ← US NASDAQ
/opt/data-lake/raw/asic/                    ← ASX short positions
/opt/data-lake/raw/nse-india/               ← India NSE direct feeds
/opt/data-lake/raw/sec/                     ← US SEC EDGAR
/opt/data-lake/raw/finra/                   ← US FINRA short interest
```

### Market Codes

| Code | Exchange | Used in independent schemas, env vars, cache keys |
|---|---|---|
| `au` | ASX | `mkt_au`, `fin_au`, `screener_au`, `au:screener:*` |
| `in` | NSE + BSE | `mkt_in`, `fin_in`, `screener_in`, `in:screener:*` |
| `us` | NYSE + NASDAQ | `mkt_us`, `fin_us`, `screener_us`, `us:screener:*` |

---

## Starting a New Project

1. Read **[01 — Platform Overview](docs/01_platform_overview.md)**
2. Follow **[04 — New Project Checklist](docs/04_new_project_checklist.md)** step by step
3. Copy this `docs/` folder into your new project repo for local reference
4. Create your schemas using the naming convention in **[03 — Naming Conventions](docs/03_naming_conventions.md)**

---

## Updating This Documentation

Architecture changes must be:
1. Discussed and agreed across all affected project teams
2. Updated here first (this repo is the source of truth)
3. Communicated to all project teams before implementation

Do not change architecture conventions in a single project's repo without updating this doc.

---

## Repository Structure

```
data-platform-docs/
├── README.md                           ← This file (start here)
└── docs/
    ├── 01_platform_overview.md         ← Full architecture, all projects
    ├── 02_bronze_layer_guide.md        ← Shared raw zone standards
    ├── 03_naming_conventions.md        ← All naming rules
    ├── 04_new_project_checklist.md     ← Bootstrap guide for new projects
    └── 05_pipeline_standards.md        ← Pipeline coding standards
```
