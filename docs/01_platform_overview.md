# Data Platform Overview

## Architecture Principle

```
Shared Raw Zone  →  Shared Staging (per-market)  →  Independent Project Pipelines  →  Project-Specific Gold
```

All projects share **two layers**:
1. **Raw Zone** — the file-based data lake (`/opt/data-lake/raw/`)
2. **Staging** — per-market DB schemas (`staging_au.*`, `staging_in.*`, `staging_us.*`) owned by the platform

After the Staging layer, every project is **fully independent** — its own pipeline, transform layer,
compute engines, financials schema, and application tables.

**Key guarantees**:
- A schema change, bug fix, or new feature in one project cannot break, affect, or be read by any other project.
- Common code is allowed only up to the Raw Zone and Staging layer. After Staging, no shared business logic or application code exists across projects.
- A failure in one project pipeline never stops or impacts any other project pipeline.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — RAW ZONE  (Shared — Platform-owned)                                      │
│  /opt/data-lake/raw/   —   immutable files, append-only                             │
│                                                                                     │
│  eodhd/AU/   eodhd/NSE+BSE/   eodhd/NYSE+NASDAQ/   asic/   nse-india/   sec/       │
│  finra/      charts/indices/   charts/forex/        charts/commodities/  charts/crypto/ │
└──────────────┬──────────────────┬────────────────────┬──────────────────────────────┘
               │  AU load job     │  India load job    │  US load job
               ▼                  ▼                    ▼
┌──────────────────┐  ┌───────────────────┐  ┌────────────────────┐
│  staging_au.*    │  │  staging_in.*     │  │  staging_us.*      │
│  Platform-owned  │  │  Platform-owned   │  │  Platform-owned    │
│                  │  │                   │  │                    │
│  eod_prices      │  │  eod_prices       │  │  eod_prices        │
│  fundamentals    │  │  fundamentals     │  │  fundamentals      │
│  shares_stats    │  │  shares_stats     │  │  shares_stats      │
│  dividends_raw   │  │  dividends_raw    │  │  dividends_raw     │
│  splits_raw      │  │  splits_raw       │  │  splits_raw        │
│  short_positions │  │  fii_dii_raw      │  │  sec_filings_raw   │
│                  │  │  promoter_        │  │  insider_trades_   │
│                  │  │  holdings_raw     │  │  raw               │
└───────┬──────────┘  └────────┬──────────┘  └──────────┬─────────┘
        │                      │                         │
        ▼                      ▼                         ▼
┌───────────────┐   ┌──────────────────┐   ┌────────────────────┐
│ ASX SCREENER  │   │ INDIA SCREENER   │   │  US SCREENER       │
│  Project A    │   │   Project B      │   │   Project C        │
│ ────────────  │   │ ──────────────── │   │  ────────────────  │
│  mkt_au.*     │   │  mkt_in.*        │   │  mkt_us.*          │
│  fin_au.*     │   │  fin_in.*        │   │  fin_us.*          │
│  screener_au.*│   │  screener_in.*   │   │  screener_us.*     │
└───────────────┘   └──────────────────┘   └────────────────────┘
        │                      │                         │
        └──────────────────────┴─────────────────────────┘
                               │  views into all three staging schemas
                               ▼
                    ┌──────────────────────────────────────┐
                    │  staging_charts.*  (Charting-owned)  │
                    │                                      │
                    │  VIEWS (no duplication):             │
                    │  au_eod_prices → staging_au.*        │
                    │  in_eod_prices → staging_in.*        │
                    │  us_eod_prices → staging_us.*        │
                    │  (only tables Charting needs)        │
                    │                                      │
                    │  OWN TABLES (Charting-specific):     │
                    │  indices, forex, commodities, crypto │
                    │  (loaded by Charting load job from   │
                    │   /opt/data-lake/raw/charts/)        │
                    └──────────────────┬───────────────────┘
                                       ▼
                              ┌─────────────────┐
                              │  CHARTING       │
                              │   Project D     │
                              │  ────────────   │
                              │  charts_mkt.*   │
                              │  charts.*       │
                              └─────────────────┘
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
- **Who writes**: Platform download pipeline + Charting download job (for indices/forex/commodities/crypto)
- **Who reads**: The staging load pipelines

See [02_bronze_layer_guide.md](02_bronze_layer_guide.md) for the full file layout.

### Layer 2 — Staging (Per-market, Platform-owned)

**Purpose**: Load raw files into relational tables. One schema per market for clear separation.

Each staging schema is loaded by an independent load job. Projects read only from their own market's staging schema.

#### `staging_au.*` — ASX (Australia)

- **Owned by**: Platform (`datalake_ingest`)
- **Loaded by**: Platform AU load job
- **Read by**: ASX Screener

| Table | Content |
|---|---|
| `staging_au.eod_prices` | OHLCV prices — ASX equities |
| `staging_au.fundamentals_raw` | Balance sheet, P&L, cashflow |
| `staging_au.shares_stats` | Shares outstanding, ownership |
| `staging_au.dividends_raw` | Dividend history |
| `staging_au.splits_raw` | Stock split history |
| `staging_au.short_positions` | ASIC short data (ASX-specific) |

#### `staging_in.*` — India (NSE + BSE)

- **Owned by**: Platform (`datalake_ingest`)
- **Loaded by**: Platform India load job
- **Read by**: India Screener

| Table | Content |
|---|---|
| `staging_in.eod_prices` | OHLCV prices — NSE + BSE equities |
| `staging_in.fundamentals_raw` | Balance sheet, P&L, cashflow |
| `staging_in.shares_stats` | Shares outstanding, ownership |
| `staging_in.dividends_raw` | Dividend history |
| `staging_in.splits_raw` | Stock split history |
| `staging_in.fii_dii_raw` | FII/DII institutional flows |
| `staging_in.promoter_holdings_raw` | Promoter SEBI disclosure |

#### `staging_us.*` — US (NYSE + NASDAQ + AMEX)

- **Owned by**: Platform (`datalake_ingest`)
- **Loaded by**: Platform US load job
- **Read by**: US Screener

| Table | Content |
|---|---|
| `staging_us.eod_prices` | OHLCV prices — NYSE + NASDAQ + AMEX |
| `staging_us.fundamentals_raw` | Balance sheet, P&L, cashflow |
| `staging_us.shares_stats` | Shares outstanding, ownership |
| `staging_us.dividends_raw` | Dividend history |
| `staging_us.splits_raw` | Stock split history |
| `staging_us.sec_filings_raw` | 10-K / 10-Q SEC filings |
| `staging_us.insider_trades_raw` | SEC Form 4 insider trades |

#### `staging_charts.*` — Charting (Multi-market)

- **Owned by**: Charting project (`charts_app`)
- **Loaded by**: Charting load job (for own tables) + views into platform staging
- **Read by**: Charting project only

| Object | Type | Content |
|---|---|---|
| `staging_charts.au_eod_prices` | VIEW → `staging_au.eod_prices` | ASX prices (no duplication) |
| `staging_charts.in_eod_prices` | VIEW → `staging_in.eod_prices` | India prices (no duplication) |
| `staging_charts.us_eod_prices` | VIEW → `staging_us.eod_prices` | US prices (no duplication) |
| `staging_charts.au_splits` | VIEW → `staging_au.splits_raw` | ASX splits (no duplication) |
| `staging_charts.in_splits` | VIEW → `staging_in.splits_raw` | India splits (no duplication) |
| `staging_charts.us_splits` | VIEW → `staging_us.splits_raw` | US splits (no duplication) |
| `staging_charts.indices` | TABLE (own data) | Global indices — S&P 500, FTSE, Nikkei, DAX, etc. |
| `staging_charts.forex` | TABLE (own data) | Forex pairs — AUD/USD, USD/INR, EUR/USD, etc. |
| `staging_charts.commodities` | TABLE (own data) | Gold, Oil, Iron Ore, etc. |
| `staging_charts.crypto` | TABLE (own data) | BTC, ETH, etc. |

Views are created by the Charting project. Only the specific tables Charting needs are exposed as views — not the full staging schema.

### Layer 3 — Market Transform (Independent per project)

**Purpose**: Clean, normalise, and enrich staging data into time-series tables.

Each project has its own transform schema — `mkt_au.*`, `mkt_in.*`, `mkt_us.*`, `charts_mkt.*`.
No project reads from another project's transform schema.

### Layer 4 — Financials (Independent per project)

**Purpose**: Normalised financial statements in the accounting standard of the market.

Each project has its own financials schema — `fin_au.*`, `fin_in.*`, `fin_us.*`.
Charting does not have a financials schema (not required for charting).

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
**Reads from staging**: `staging_au.*`
**Additional staging sources**: `staging_au.short_positions` (ASIC)

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
**Reads from staging**: `staging_in.*`
**Additional staging sources**: `staging_in.fii_dii_raw`, `staging_in.promoter_holdings_raw`

**Unique features**:
- Promoter holdings % (mandatory SEBI disclosure) — key risk signal
- Pledged shares % (promoter pledge — insider stress indicator)
- FII/DII net flow tracking (Foreign/Domestic Institutional flows)
- Nifty index membership: Nifty 50, 100, 200, 500, Midcap 150, Smallcap 250
- Market cap tiers in INR per SEBI definition (Largecap top 100, Midcap 101-250, Smallcap 251+)
- Quarterly reporting mandatory (standalone + consolidated results)
- Circuit breakers: upper/lower circuit % — stock-specific daily price limits
- BSE/NSE dual-listing deduplication (same company, two exchange codes)

**Independent schemas**:
```
mkt_in.*          Market data (TimescaleDB)
fin_in.*          Financials (Indian GAAP / Ind AS)
screener_in.*     Application layer — Golden Record
```

**Golden Record**: `screener_in.universe` — one row per stock, rebuilt nightly.

---

### Project C: US Screener

**Market**: NYSE, NASDAQ, AMEX
**Reads from staging**: `staging_us.*`
**Additional staging sources**: `staging_us.sec_filings_raw`, `staging_us.insider_trades_raw`

**Unique features**:
- Quarterly reporting (10-Q mandatory via SEC)
- S&P 500 / Russell 1000/2000 / Nasdaq 100 index membership
- Market cap tiers in USD (Mega ≥$200B, Large $10-200B, Mid $2-10B, Small $300M-2B, Micro $50-300M, Nano <$50M)
- SEC Form 4 insider trading (mandatory disclosure within 2 business days)
- SEC 13F institutional ownership (quarterly, ≥$100M AUM threshold)
- FINRA bi-monthly short interest data (short float %, days-to-cover)
- GAAP vs non-GAAP EPS reconciliation

**Independent schemas**:
```
mkt_us.*          Market data (TimescaleDB)
fin_us.*          Financials (US GAAP)
screener_us.*     Application layer — Golden Record
```

**Golden Record**: `screener_us.universe` — one row per stock, rebuilt nightly.

---

### Project D: Charting (Multi-Market)

**Markets**: All — ASX, India, US + Global Indices, Forex, Commodities, Crypto
**Reads from**: `staging_charts.*` (views into `staging_au.*`, `staging_in.*`, `staging_us.*` + own tables)

**Design principle**: Market-agnostic. All instruments use a unified namespace:
`{MARKET}:{CODE}` — e.g. `AU:BHP`, `IN:RELIANCE`, `US:AAPL`, `FX:AUDUSD`, `CRYPTO:BTC`

**Key differences from Screener projects**:
- Uses **split-adjusted** prices (for continuous chart lines across corporate actions)
- Screener projects use **unadjusted** prices (for accurate valuation ratios)
- Needs multi-timeframe pre-aggregated bars (1m, 5m, 15m, 1h, 4h, 1d, 1w, 1M)
- Needs corporate event annotations (dividends, splits, earnings dates) on chart timeline
- Also handles non-equity instruments: indices, forex, commodities, crypto

**Own staging load job**: Charting downloads and loads its own Indices/Forex/Commodities/Crypto
data from `/opt/data-lake/raw/charts/` into `staging_charts.*` own tables.

**Independent schemas**:
```
staging_charts.*  Charting staging (views + own tables) — Charting-owned
charts_mkt.*      Chart-optimised transform layer
charts.*          Application layer
```

**Key tables**:
```
charts_mkt.ohlcv             Unified split-adjusted OHLCV, all markets + indices/forex/crypto
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
Role                  READ access                              WRITE access
──────────────────────────────────────────────────────────────────────────────────────
datalake_ingest       (none — file system only)               staging_au.*
                                                               staging_in.*
                                                               staging_us.*

screener_au_app       staging_au.*, users.*                   mkt_au.*, fin_au.*
                      mkt_au.*, fin_au.*                      screener_au.*

screener_in_app       staging_in.*, users.*                   mkt_in.*, fin_in.*
                      mkt_in.*, fin_in.*                      screener_in.*

screener_us_app       staging_us.*, users.*                   mkt_us.*, fin_us.*
                      mkt_us.*, fin_us.*                      screener_us.*

charts_app            staging_au.*, staging_in.*,             staging_charts.*
                      staging_us.*, users.*                   charts_mkt.*, charts.*
                      charts_mkt.*

readonly              screener_au.universe,                   (none)
                      screener_in.universe,
                      screener_us.universe
```

No project role has SELECT on another project's transform or application schemas.

---

## Pipeline Schedule

```
Time (UTC)       Job                                Owner
──────────────────────────────────────────────────────────────────────
── AU ────────────────────────────────────────────────────────────────
18:00 Mon-Fri    download_eodhd_au.py               Platform (data-lake)
18:00 Mon-Fri    download_asic_shorts.py            Platform (data-lake)
18:30 Mon-Fri    load_staging_au.py                 Platform (data-lake)

── India ─────────────────────────────────────────────────────────────
10:00 Mon-Fri    download_eodhd_nse_bse.py          Platform (data-lake)
10:30 Mon-Fri    download_nse_india_feeds.py        Platform (data-lake)
11:00 Mon-Fri    load_staging_in.py                 Platform (data-lake)

── US ────────────────────────────────────────────────────────────────
21:30 Mon-Fri    download_eodhd_us.py               Platform (data-lake)
22:00 Mon-Fri    download_sec_form4.py              Platform (data-lake)
22:30 Mon-Fri    load_staging_us.py                 Platform (data-lake)

── Charting (own staging load) ───────────────────────────────────────
22:30 Mon-Fri    download_charts_data.py            Charting project
23:00 Mon-Fri    load_staging_charts.py             Charting project

── Project pipelines (start AFTER staging is loaded) ─────────────────
19:00 Mon-Fri    asx_daily_pipeline.py              ASX Screener project
11:30 Mon-Fri    india_daily_pipeline.py            India Screener project
23:00 Mon-Fri    us_daily_pipeline.py               US Screener project
23:30 Mon-Fri    charting_daily_pipeline.py         Charting project
```

**Project pipelines start from the transform step** — they do NOT download platform files
or load platform staging. They read from their staging schema and write to their own schemas.

**Charting pipeline** starts after all platform staging loads AND its own staging load are complete.

**Failure isolation**: Each pipeline runs as an independent cron job. A failure in one project
does not stop or affect any other project's pipeline.
