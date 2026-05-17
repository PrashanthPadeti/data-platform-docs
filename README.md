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

## The One Rule

> **All projects share the Bronze Raw Zone. After the Raw Zone, every project is fully independent.**
>
> A schema change, bug fix, or new feature in one project must never affect another project.

```
Shared Bronze (Raw Zone)
        │
        ├──→  ASX Screener     (stg_au → mkt_au → fin_au → screener_au)
        ├──→  India Screener   (stg_in → mkt_in → fin_in → screener_in)
        ├──→  US Screener      (stg_us → mkt_us → fin_us → screener_us)
        └──→  Charting         (stg_charts → charts_mkt → charts)
```

---

## Quick Reference

### Schema Map

| Project | Staging | Market | Financials | Application |
|---|---|---|---|---|
| ASX Screener | `stg_au.*` | `mkt_au.*` | `fin_au.*` | `screener_au.*` |
| India Screener | `stg_in.*` | `mkt_in.*` | `fin_in.*` | `screener_in.*` |
| US Screener | `stg_us.*` | `mkt_us.*` | `fin_us.*` | `screener_us.*` |
| Charting | `stg_charts.*` | `charts_mkt.*` | — | `charts.*` |
| Shared Infra | — | — | — | `users.*` `notifications.*` `ai.*` |

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

| Code | Exchange | Used in schemas, env vars, cache keys |
|---|---|---|
| `au` | ASX | `stg_au`, `mkt_au`, `au:screener:*` |
| `in` | NSE + BSE | `stg_in`, `mkt_in`, `in:screener:*` |
| `us` | NYSE + NASDAQ | `stg_us`, `mkt_us`, `us:screener:*` |

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
