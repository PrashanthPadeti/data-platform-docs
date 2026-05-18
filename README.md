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
| Charting | *(coming soon)* | Multi-market + Indices / Forex / Commodities / Crypto | 🔜 Planned |

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

> **Raw Zone and Staging are shared (platform-owned). Every layer after Staging is fully independent per project.**
>
> A schema change, bug fix, or new feature in one project must never affect another project.
>
> **Common code is allowed only up to the Raw Zone and Staging layer. After Staging, each project owns its own codebase, schemas, and application logic.**

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│  SHARED RAW ZONE  (Platform-owned)                                       │
│  /opt/data-lake/raw/                                                     │
│  eodhd/AU/   eodhd/NSE+BSE/   eodhd/NYSE+NASDAQ/   charts/indices/      │
│  asic/       nse-india/        sec/  finra/          forex/  crypto/     │
└────────────┬──────────────────┬──────────────────┬───────────────────────┘
             ▼                  ▼                  ▼
      AU load job          India load job      US load job
             ▼                  ▼                  ▼
      staging_au.*         staging_in.*        staging_us.*
      (Platform-owned)     (Platform-owned)    (Platform-owned)
             │                  │                  │
    ┌────────┘       ┌──────────┘       ┌──────────┘
    │                │                  │
    ▼                ▼                  ▼
ASX Screener   India Screener      US Screener
mkt_au.*       mkt_in.*            mkt_us.*
fin_au.*       fin_in.*            fin_us.*
screener_au.*  screener_in.*       screener_us.*
    │                │                  │
    └────────────────┴──────────────────┘
                     │  views into all three
                     ▼
             staging_charts.*   ◄── also loads own tables:
             (Charting-owned)       indices, forex, commodities, crypto
                     │              from /opt/data-lake/raw/charts/
                     ▼
              charts_mkt.*
              charts.*

                     │
        ┌────────────▼────────────┐
        │  SHARED INFRASTRUCTURE  │
        │  users.*  notifications.│
        │  ai.*  subscriptions.*  │
        └─────────────────────────┘
```

---

## What is Shared vs Independent

| Layer | Shared or Independent | Schema | Owned by |
|---|---|---|---|
| Raw Zone (files) | ✅ Shared | `/opt/data-lake/raw/` | Platform team |
| AU Staging (DB) | ✅ Shared | `staging_au.*` | Platform team |
| India Staging (DB) | ✅ Shared | `staging_in.*` | Platform team |
| US Staging (DB) | ✅ Shared | `staging_us.*` | Platform team |
| Charts Staging (DB) | 🔒 Independent | `staging_charts.*` | Charting project |
| Market Transform | 🔒 Independent | `mkt_au.*` / `mkt_in.*` / `mkt_us.*` | Each screener project |
| Financials | 🔒 Independent | `fin_au.*` / `fin_in.*` / `fin_us.*` | Each screener project |
| Application (Gold) | 🔒 Independent | `screener_au.*` / `screener_in.*` / `screener_us.*` | Each screener project |
| Charts Transform | 🔒 Independent | `charts_mkt.*` | Charting project |
| Charts Application | 🔒 Independent | `charts.*` | Charting project |
| Auth / Billing | ✅ Shared | `users.*` `notifications.*` `ai.*` | Platform team |

---

## Pipeline Execution Order

```
── PLATFORM PIPELINE (runs first) ────────────────────────────────────────
Step 1a  Download AU raw files          → /opt/data-lake/raw/eodhd/AU/
Step 1b  Download India raw files       → /opt/data-lake/raw/eodhd/NSE+BSE/
Step 1c  Download US raw files          → /opt/data-lake/raw/eodhd/NYSE+NASDAQ/

Step 2a  Load staging_au.*              (independent AU load job)
Step 2b  Load staging_in.*              (independent India load job)
Step 2c  Load staging_us.*              (independent US load job)

── CHARTING STAGING (runs alongside platform) ─────────────────────────────
Step 2d  Download Indices/Forex/Commodities/Crypto → /opt/data-lake/raw/charts/
Step 2e  Load staging_charts.* own tables          (Charting load job)

── PROJECT PIPELINES (each triggers independently after its staging is ready)
After 2a           → ASX Screener pipeline    reads staging_au.*
After 2b           → India Screener pipeline  reads staging_in.*
After 2c           → US Screener pipeline     reads staging_us.*
After 2a+2b+2c+2e  → Charting pipeline        reads staging_charts.*
```

**Failure isolation**: Each project pipeline runs as an independent process.
A failure in one project never stops or affects any other project pipeline.

---

## Code Separation Rule

```
┌─────────────────────────────────────────────────────┐
│  COMMON CODE ALLOWED                                │
│  Raw Zone download jobs                             │
│  Staging load jobs (staging_au, staging_in, us)     │
└──────────────────────────┬──────────────────────────┘
                           │
              ─────────────────────────
               CODE SEPARATION BOUNDARY
              ─────────────────────────
                           │
┌──────────────────────────▼──────────────────────────┐
│  NO SHARED CODE BELOW THIS LINE                     │
│  Each project has its own:                          │
│  - Transform pipeline                               │
│  - Compute engines                                  │
│  - Business logic                                   │
│  - Application code                                 │
│  - Schemas and tables                               │
└─────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Staging Schema Tables

| Schema | Contains | Exchange / Source |
|---|---|---|
| `staging_au.*` | ASX equities — prices, fundamentals, dividends, splits, shorts | ASX (AU) |
| `staging_in.*` | NSE/BSE equities — prices, fundamentals, FII/DII, promoter holdings | NSE + BSE (India) |
| `staging_us.*` | NYSE/NASDAQ equities — prices, fundamentals, SEC filings, insider trades | NYSE + NASDAQ + AMEX |
| `staging_charts.*` | Views → au/in/us staging + own tables: indices, forex, commodities, crypto | Multi-market |

### Project Schema Map (Independent Layers Only)

| Project | Staging (reads from) | Market Transform | Financials | Application |
|---|---|---|---|---|
| ASX Screener | `staging_au.*` | `mkt_au.*` | `fin_au.*` | `screener_au.*` |
| India Screener | `staging_in.*` | `mkt_in.*` | `fin_in.*` | `screener_in.*` |
| US Screener | `staging_us.*` | `mkt_us.*` | `fin_us.*` | `screener_us.*` |
| Charting | `staging_charts.*` | `charts_mkt.*` | — | `charts.*` |

### Market Codes

| Code | Exchange | Used in independent schemas, env vars, cache keys |
|---|---|---|
| `au` | ASX | `mkt_au`, `fin_au`, `screener_au`, `au:screener:*` |
| `in` | NSE + BSE | `mkt_in`, `fin_in`, `screener_in`, `in:screener:*` |
| `us` | NYSE + NASDAQ | `mkt_us`, `fin_us`, `screener_us`, `us:screener:*` |

### Raw Zone Paths

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
/opt/data-lake/raw/charts/indices/          ← Global indices
/opt/data-lake/raw/charts/forex/            ← Forex pairs
/opt/data-lake/raw/charts/commodities/      ← Commodities
/opt/data-lake/raw/charts/crypto/           ← Crypto
```

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
