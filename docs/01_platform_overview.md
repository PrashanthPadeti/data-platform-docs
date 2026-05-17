# Data Platform Overview

## Architecture Principle

```
Shared Raw Zone + Shared Staging  →  Project-Specific Transform  →  Project-Specific Gold
```

All projects share **two layers**:
1. **Raw Zone** — the file-based data lake (`/opt/data-lake/raw/`)
2. **Staging** — the shared DB schema (`staging.*`) where raw files land as relational tables

After the Staging layer, every project is **fully independent** — its own transform layer,
compute engines, financials schema, and application tables.

**Key guarantee**: A schema change, bug fix, or new feature in one project
cannot break, affect, or be read by any other project.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — RAW ZONE  (Shared — Platform-owned)                                   │
│  /opt/data-lake/raw/   —   immutable files, append-only                          │
│                                                                                  │
│  eodhd/exchange=AU/  eodhd/exchange=NSE/  eodhd/exchange=NYSE/                  │
│  eodhd/exchange=BSE/ eodhd/exchange=NASDAQ/ asic/ nse-india/ sec/ finra/        │
└─────────────────────────────────┬────────────────────────────────────────────────┘
                                  │  platform pipeline loads all exchanges
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2 — STAGING  (Shared — Platform-owned)                                    │
│  staging.*   —   all exchanges, all providers in one schema                      │
│                                                                                  │
│  staging.eod_prices          staging.fundamentals_raw    staging.shares_stats   │
│  staging.dividends_raw       staging.splits_raw          staging.short_positions │
│  staging.fii_dii_raw         staging.promoter_holdings_raw                       │
│  staging.sec_filings_raw     staging.insider_trades_raw                          │
└─────────────────────────────────┬────────────────────────────────────────────────┘
                                  │  each project reads independently (filtered by exchange)
              ┌───────────────────┼────────────────────┬──────────────────┐
              ▼                   ▼                    ▼                  ▼
   ┌─────────────────┐  ┌──────────────────┐  ┌────────────────┐  ┌────────────────┐
   │  ASX SCREENER   │  │  INDIA NIFTY     │  │  US SCREENER   │  │  CHARTING      │
   │   Project A     │  │  SCREENER        │  │   Project C    │  │   Project D    │
   │  ─────────────  │  │   Project B      │  │  ────────────  │  │  ────────────  │
   │  mkt_au.*       │  │  ─────────────── │  │  mkt_us.*      │  │  charts_mkt.*  │
   │  fin_au.*       │  │  mkt_in.*        │  │  fin_us.*      │  │                │
   │  screener_au.*  │  │  fin_in.*        │  │  screener_us.* │  │  charts.*      │
   └─────────────────┘  │  screener_in.*   │  └────────────────┘  └────────────────┘
                        └──────────────────┘
              │                   │                    │                  │
              └───────────────────┴────────────────────┴──────────────────┘
                                          │
                             ┌────────────▼────────────┐
                             │  SHARED INFRASTRUCTURE  │
                             │  users.*  notifications.│
                             │  ai.*  subscriptions.*  │
                             └─────────────────────────┘
```

---

## Layer Definitions

### Layer 1 — Raw Zone (Shared)

**Purpose**: Store every downloaded file from every data provider, unmodified.

- **Location**: `/opt/data-lake/raw/`
- **Owned by**: `datalake_ingest` service account
- **Format**: Provider's original format (JSON, JSON.GZ, CSV)
- **Mutation policy**: Append-only. Files are never modified or deleted after write.
- **Who writes**: Only the shared download pipeline (`/opt/data-lake/jobs/`)
- **Who reads**: The shared staging load pipeline

See [02_bronze_layer_guide.md](02_bronze_layer_guide.md) for the full file layout.

### Layer 2 — Staging (Shared)

**Purpose**: Load raw files into relational tables. One schema for all exchanges.

- **Schema**: `staging.*`
- **Owned by**: `datalake_ingest` service account (platform pipeline writes, project accounts read)
- **Content**: Raw provider data in relational form — no computed columns, no business logic
- **Filtering**: All exchanges coexist in the same tables; projects filter by `exchange` column
- **Who writes**: Platform staging load pipeline
- **Who reads**: Each project's transform pipeline, filtered to its own exchange(s)

**Staging tables**:

| Table | Content | Exchange filter |
|---|---|---|
| `staging.eod_prices` | OHLCV prices — all exchanges | `WHERE exchange = 'AU'` / `'NSE'` / `'NYSE'` etc. |
| `staging.fundamentals_raw` | Balance sheet, P&L, cashflow | `WHERE exchange = 'AU'` etc. |
| `staging.shares_stats` | Shares outstanding, ownership | `WHERE exchange = 'AU'` etc. |
| `staging.dividends_raw` | Dividend history | `WHERE exchange = 'AU'` etc. |
| `staging.splits_raw` | Stock split history | `WHERE exchange = 'AU'` etc. |
| `staging.short_positions` | ASIC short data | ASX Screener only |
| `staging.fii_dii_raw` | India FII/DII flows | India Screener only |
| `staging.promoter_holdings_raw` | India promoter SEBI disclosure | India Screener only |
| `staging.sec_filings_raw` | US 10-K/10-Q filings | US Screener only |
| `staging.insider_trades_raw` | US SEC Form 4 | US Screener only |

### Layer 3 — Market Transform (Independent per project)

**Purpose**: Clean, normalise, and enrich staging data into time-series tables.

Each project has its own transform schema — `mkt_au.*`, `mkt_in.*`, `mkt_us.*`, `charts_mkt.*`.
No project reads from another project's transform schema.

### Layer 4 — Financials (Independent per project)

**Purpose**: Normalised financial statements in the accounting standard of the market.

Each project has its own financials schema — `fin_au.*`, `fin_in.*`, `fin_us.*`.

### Layer 5 — Application / Gold (Independent per project)

**Purpose**: Denormalised Golden Record rebuilt nightly — the API reads this directly.

| Project | Schema | Key table |
|---|---|---|
| ASX Screener | `screener_au.*` | `screener_au.universe` |
| India Nifty Screener | `screener_in.*` | `screener_in.universe` |
| US Screener | `screener_us.*` | `screener_us.universe` |
| Charting | `charts.*` | `charts.ohlcv_cache`, `charts.saved_charts` |

---

## Project Details

### Project A: ASX Screener

**Market**: Australian Securities Exchange (ASX)
**Reads from staging**: `WHERE exchange = 'AU'`
**Additional staging sources**: `staging.short_positions` (ASIC)

**Unique features**:
- Franking credits → grossed-up dividend yield
- REIT-specific: NTA per share, WALE, gearing ratio
- Mining-specific: AISC per oz, reserve life
- Half-yearly reporting (H1/H2) — no Q1/Q2/Q3/Q4
- ASIC-sourced short position data

**Independent schemas**:
```
mkt_au.*          Market data (TimescaleDB)
fin_au.*          Financials (IFRS / Australian GAAP)
screener_au.*     Application layer — Golden Record
```

**Golden Record**: `screener_au.universe` — one row per stock, rebuilt nightly.

---

### Project B: India Nifty Screener

**Market**: National Stock Exchange (NSE) and Bombay Stock Exchange (BSE)
**Reads from staging**: `WHERE exchange IN ('NSE', 'BSE')`
**Additional staging sources**: `staging.fii_dii_raw`, `staging.promoter_holdings_raw`

**Unique features**:
- Promoter holdings % (mandatory SEBI disclosure) — key risk signal
- Pledged shares % (promoter pledge — insider stress indicator)
- FII/DII net flow tracking (Foreign/Domestic Institutional flows)
- Nifty index membership: Nifty 50, 100, 200, 500, Midcap 150, Smallcap 250
- Market cap tiers in INR per SEBI definition (Largecap top 100, Midcap 101-250, Smallcap 251+)
- Quarterly reporting mandatory (standalone + consolidated results)
- Circuit breakers: upper/lower circuit % — stock-specific daily price limits
- BSE/NSE dual-listing deduplication (same company, two exchange codes)
- No franking credits — dividend yield is pre-tax gross yield

**Independent schemas**:
```
mkt_in.*          Market data (TimescaleDB)
fin_in.*          Financials (Indian GAAP / Ind AS)
screener_in.*     Application layer — Golden Record
```

**Golden Record**: `screener_in.universe` — one row per stock, rebuilt nightly.

**India-only transform tables** (within `mkt_in.*`):
```
mkt_in.fii_dii_flows        FII/DII institutional buy/sell flows (daily)
mkt_in.promoter_holdings    SEBI-mandated promoter holding disclosures
mkt_in.pledged_shares       Promoter pledge data
mkt_in.circuit_limits       Per-stock upper/lower circuit bands
fin_in.quarterly_results    Quarterly P&L (standalone + consolidated)
```

---

### Project C: US Screener

**Market**: NYSE, NASDAQ, AMEX
**Reads from staging**: `WHERE exchange IN ('NYSE', 'NASDAQ', 'AMEX')`
**Additional staging sources**: `staging.sec_filings_raw`, `staging.insider_trades_raw`

**Unique features**:
- Quarterly reporting (10-Q mandatory via SEC)
- S&P 500 / Russell 1000/2000 / Nasdaq 100 index membership
- Market cap tiers in USD (Mega ≥$200B, Large $10-200B, Mid $2-10B, Small $300M-2B, Micro $50-300M, Nano <$50M)
- SEC Form 4 insider trading (mandatory disclosure within 2 business days)
- SEC 13F institutional ownership (quarterly, ≥$100M AUM threshold)
- FINRA bi-monthly short interest data (short float %, days-to-cover)
- GAAP vs non-GAAP EPS reconciliation
- No franking credits — straightforward dividend yield

**Independent schemas**:
```
mkt_us.*          Market data (TimescaleDB)
fin_us.*          Financials (US GAAP)
screener_us.*     Application layer — Golden Record
```

**Golden Record**: `screener_us.universe` — one row per stock, rebuilt nightly.

**US-only transform tables** (within `mkt_us.*`):
```
mkt_us.short_interest        FINRA bi-monthly short reports
mkt_us.insider_trades        SEC Form 4 insider buy/sell
mkt_us.institutional_13f     SEC 13F quarterly institutional holdings
fin_us.quarterly_results     10-Q data
```

---

### Project D: Charting (Multi-Market)

**Markets**: All (AU, IN, US — and any future market)
**Reads from staging**: All exchanges — `staging.eod_prices`, `staging.splits_raw`, `staging.dividends_raw`

**Design principle**: Market-agnostic. All instruments use a unified namespace:
`{MARKET}:{CODE}` — e.g. `AU:BHP`, `IN:RELIANCE`, `US:AAPL`

**Key differences from Screener projects**:
- Uses **split-adjusted** prices (for continuous chart lines across corporate actions)
- Screener projects use **unadjusted** prices (for accurate valuation ratios)
- Needs multi-timeframe pre-aggregated bars (1m, 5m, 15m, 1h, 4h, 1d, 1w, 1M)
- Needs corporate event annotations (dividends, splits, earnings dates) on chart timeline

**Independent schemas**:
```
charts_mkt.*      Chart-optimised transform layer
charts.*          Application layer
```

**Key tables**:
```
charts_mkt.ohlcv             Unified split-adjusted OHLCV, all markets
charts_mkt.corporate_events  Dividends + splits merged for chart annotations
charts_mkt.indicator_cache   Pre-computed indicator series
charts_mkt.multi_tf_bars     Pre-aggregated multi-timeframe bars
charts.saved_charts          User-saved chart configurations
charts.drawing_objects       Trendlines, annotations saved by users
charts.watchlists
charts.alert_levels
```

---

## Shared Infrastructure

These schemas are shared across all projects:

```
users.*             Auth, profiles, subscriptions, API keys
notifications.*     Email/push alerts, price alerts
ai.*                AI summaries, embeddings, analysis cache
subscriptions.*     Stripe billing, plan management
```

**Rule**: Project code may read `users.*` for auth only. All user-facing features
(watchlists, saved screens, alerts) are stored in **the project's own schema**.

---

## Database Role Isolation

```
Role                 READ access                              WRITE access
──────────────────────────────────────────────────────────────────────────────────
datalake_ingest      (none — file system only)               staging.*
screener_au_app      staging.* (exchange=AU only via app)    mkt_au.*, fin_au.*
                     mkt_au.*, fin_au.*, users.*              screener_au.*
screener_in_app      staging.* (exchange=NSE/BSE via app)    mkt_in.*, fin_in.*
                     mkt_in.*, fin_in.*, users.*              screener_in.*
screener_us_app      staging.* (exchange=NYSE/NASDAQ via app) mkt_us.*, fin_us.*
                     mkt_us.*, fin_us.*, users.*              screener_us.*
charts_app           staging.* (all exchanges via app)       charts_mkt.*, charts.*
                     charts_mkt.*, users.*
readonly             screener_au.universe,                   (none)
                     screener_in.universe,
                     screener_us.universe
```

No project role has SELECT on another project's transform or application schemas.
All project roles read `staging.*` but filter in application code to their own exchange.

---

## Pipeline Schedule

```
Time (UTC)       Job                              Owner
──────────────────────────────────────────────────────────────────
18:00 Mon-Fri    download_eodhd_au.py             Platform (data-lake)
18:00 Mon-Fri    download_asic_shorts.py          Platform (data-lake)
18:30 Mon-Fri    load_staging_au.py               Platform (data-lake)
──── India ───────────────────────────────────────────────────────
10:00 Mon-Fri    download_eodhd_nse_bse.py        Platform (data-lake)
10:30 Mon-Fri    download_nse_india_feeds.py      Platform (data-lake)
11:00 Mon-Fri    load_staging_in.py               Platform (data-lake)
──── US ──────────────────────────────────────────────────────────
21:30 Mon-Fri    download_eodhd_us.py             Platform (data-lake)
22:00 Mon-Fri    download_sec_form4.py            Platform (data-lake)
22:30 Mon-Fri    load_staging_us.py               Platform (data-lake)
──── Project pipelines (start AFTER staging is loaded) ───────────
19:00 Mon-Fri    asx_daily_pipeline.py            ASX Screener project
11:30 Mon-Fri    india_daily_pipeline.py          India Screener project
23:00 Mon-Fri    us_daily_pipeline.py             US Screener project
19:30 Mon-Fri    charting_daily_pipeline.py       Charting project
```

**Project pipelines start from the transform step** — they do NOT download files
or load staging. They read from `staging.*` and write to their own schemas.
