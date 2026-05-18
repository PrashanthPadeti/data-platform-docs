# Naming Conventions

Consistent naming is critical when multiple projects share the same PostgreSQL
instance and the same codebase repository. This document is the authoritative
reference for all naming decisions across every project.

---

## Market Codes

These two-letter market codes are used everywhere — schema prefixes, file paths,
environment variables, API endpoints, and logging.

| Market Code | Exchange | Country |
|---|---|---|
| `au` | ASX (Australian Securities Exchange) | Australia |
| `in` | NSE + BSE (National/Bombay Stock Exchange) | India |
| `us` | NYSE + NASDAQ + AMEX | United States |

Future markets follow the same pattern (e.g. `uk` for LSE, `jp` for TSE).

---

## Database Schema Naming

### Shared vs Independent

| Layer | Schema | Owned by | Who writes | Who reads |
|---|---|---|---|---|
| Raw Zone (files) | `/opt/data-lake/raw/` | Platform | Platform pipeline + Charting | Platform pipeline + Charting |
| AU Staging | `staging_au.*` | Platform | Platform AU load job | ASX Screener |
| India Staging | `staging_in.*` | Platform | Platform India load job | India Screener |
| US Staging | `staging_us.*` | Platform | Platform US load job | US Screener |
| Charts Staging | `staging_charts.*` | Charting project | Charting load job | Charting project only |
| Market Transform | `mkt_au.*` / `mkt_in.*` / `mkt_us.*` | Each project | Each project | Same project only |
| Financials | `fin_au.*` / `fin_in.*` / `fin_us.*` | Each project | Each project | Same project only |
| Application (Gold) | `screener_au.*` / `screener_in.*` / `screener_us.*` | Each project | Each project | Same project + API |
| Charts Transform | `charts_mkt.*` | Charting | Charting project | Charting project + API |
| Charts Application | `charts.*` | Charting | Charting project | Charting project + API |
| Shared Infra | `users.*` `notifications.*` `ai.*` | Platform | Auth service | All projects (read-only) |

### Pattern for Independent Schemas

```
{market}_{layer}      ← screener projects
charts_{layer}        ← charting project
staging_{market}      ← platform staging schemas
```

### All Schemas

| Schema | Shared or Independent | Layer | Owned by |
|---|---|---|---|
| `staging_au` | ✅ **Shared** | Staging — ASX | Platform |
| `staging_in` | ✅ **Shared** | Staging — India | Platform |
| `staging_us` | ✅ **Shared** | Staging — US | Platform |
| `staging_charts` | 🔒 Independent | Staging — Charting-specific | Charting project |
| `mkt_au` | 🔒 Independent | Market transform | ASX Screener |
| `fin_au` | 🔒 Independent | Financials | ASX Screener |
| `screener_au` | 🔒 Independent | Application (Gold) | ASX Screener |
| `mkt_in` | 🔒 Independent | Market transform | India Screener |
| `fin_in` | 🔒 Independent | Financials | India Screener |
| `screener_in` | 🔒 Independent | Application (Gold) | India Screener |
| `mkt_us` | 🔒 Independent | Market transform | US Screener |
| `fin_us` | 🔒 Independent | Financials | US Screener |
| `screener_us` | 🔒 Independent | Application (Gold) | US Screener |
| `charts_mkt` | 🔒 Independent | Market transform | Charting |
| `charts` | 🔒 Independent | Application (Gold) | Charting |
| `users` | ✅ **Shared** | Auth / billing | Platform |
| `notifications` | ✅ **Shared** | Alerts / emails | Platform |
| `ai` | ✅ **Shared** | AI features | Platform |

### Rules

- Schema names are **lowercase**, **underscored**, no hyphens.
- Staging schemas use the `staging_{market}` pattern — one schema per market, owned by the platform.
- `staging_charts` is the exception — owned by the Charting project, contains views + own tables.
- Never use a generic name like `staging` without a market suffix.
- Never use `public` schema for application tables.
- Shared infrastructure schemas (`users`, `notifications`, `ai`) have **no market prefix** — they are intentionally cross-project.

---

## Table Naming

### Pattern

```
{descriptive_noun}          (within an already-namespaced schema)
```

Tables do NOT repeat the schema name. The schema provides the namespace.

```sql
-- Correct
staging_au.eod_prices
mkt_au.daily_prices
screener_in.universe
fin_us.annual_pnl

-- Wrong — redundant prefix
staging_au.au_eod_prices    ← "au_" is redundant, schema already says au
mkt_au.au_daily_prices      ← "au_" is redundant
screener_in.india_universe  ← "india_" is redundant
```

### Standard Staging Table Names (same name across all staging schemas)

| Table | Description |
|---|---|
| `{stg}.eod_prices` | Raw EOD prices from provider |
| `{stg}.fundamentals_raw` | Raw financial statements |
| `{stg}.shares_stats` | Shares outstanding + ownership |
| `{stg}.dividends_raw` | Dividend history |
| `{stg}.splits_raw` | Stock split history |

### Market-Unique Staging Tables

| Table | Schema | Unique to |
|---|---|---|
| `short_positions` | `staging_au` | ASX Screener (ASIC data) |
| `fii_dii_raw` | `staging_in` | India Screener |
| `promoter_holdings_raw` | `staging_in` | India Screener |
| `sec_filings_raw` | `staging_us` | US Screener (SEC EDGAR) |
| `insider_trades_raw` | `staging_us` | US Screener (SEC Form 4) |

### Charting Staging Objects (`staging_charts.*`)

| Object | Type | Description |
|---|---|---|
| `au_eod_prices` | VIEW → `staging_au.eod_prices` | ASX prices — no data duplication |
| `in_eod_prices` | VIEW → `staging_in.eod_prices` | India prices — no data duplication |
| `us_eod_prices` | VIEW → `staging_us.eod_prices` | US prices — no data duplication |
| `au_splits` | VIEW → `staging_au.splits_raw` | ASX splits — no data duplication |
| `in_splits` | VIEW → `staging_in.splits_raw` | India splits — no data duplication |
| `us_splits` | VIEW → `staging_us.splits_raw` | US splits — no data duplication |
| `indices` | TABLE | Global indices — S&P 500, FTSE, Nikkei, DAX, etc. |
| `forex` | TABLE | Forex pairs — AUD/USD, USD/INR, EUR/USD, etc. |
| `commodities` | TABLE | Gold, Oil, Iron Ore, etc. |
| `crypto` | TABLE | BTC, ETH, etc. |

### Standard Transform/Application Table Names (same name, different schema per project)

| Table | Description |
|---|---|
| `{mkt}.companies_current` | SCD2 master company list |
| `{mkt}.daily_prices` | TimescaleDB OHLCV (unadjusted) |
| `{mkt}.daily_metrics` | Computed technical indicators |
| `{mkt}.weekly_metrics` | Weekly aggregated metrics |
| `{mkt}.monthly_metrics` | Monthly aggregated metrics |
| `{mkt}.quarterly_metrics` | Quarterly growth metrics |
| `{mkt}.yearly_metrics` | Annual CAGR + quality scores |
| `{mkt}.valuation_snapshot` | Snapshot PE/PB/EV ratios |
| `{mkt}.dividends` | Dividend history |
| `{mkt}.analyst_ratings` | Analyst consensus + targets |
| `{fin}.annual_pnl` | P&L financial statements |
| `{fin}.annual_balance_sheet` | Balance sheet |
| `{fin}.annual_cashflow` | Cash flow statement |
| `{screener}.universe` | Golden Record — one row per stock |

### Market-Unique Transform Tables (only exist for one project)

| Table | Schema | Unique to |
|---|---|---|
| `halfyearly_metrics` | `mkt_au` | ASX Screener (H1/H2 reporting) |
| `short_positions` | `mkt_au` | ASX Screener (ASIC data) |
| `fii_dii_flows` | `mkt_in` | India Screener |
| `promoter_holdings` | `mkt_in` | India Screener |
| `pledged_shares` | `mkt_in` | India Screener |
| `circuit_limits` | `mkt_in` | India Screener |
| `short_interest` | `mkt_us` | US Screener (FINRA) |
| `insider_trades` | `mkt_us` | US Screener (SEC Form 4) |
| `institutional_13f` | `mkt_us` | US Screener (SEC 13F) |

---

## Column Naming

### Monetary Values — Always Specify the Unit

Monetary columns include the unit in the column name to prevent ambiguity:

| Suffix | Meaning | Examples |
|---|---|---|
| _(no suffix)_ | Per-share value (AUD/INR/USD) | `eps`, `dps_ttm`, `book_value_per_share` |
| `_aud` / `_inr` / `_usd` | Absolute value in currency | `revenue_aud`, `market_cap_usd` |
| `_m` | AUD/INR/USD millions | `revenue_m`, `market_cap_m`, `net_debt_m` |

All screener.universe Golden Record tables use **millions** for absolute monetary
values (consistent with how analysts reference them).

### Ratio and Percentage Fields

| Type | Convention | Scale stored | Example |
|---|---|---|---|
| Decimal ratio | No suffix | 0.15 = 15% | `dividend_yield`, `net_margin`, `roe` |
| 0-100 scale | `_pct` suffix | 75.0 = 75% | `franking_pct`, `short_pct`, `rsi_14` |
| Multiplier | `_ratio` suffix | 1.5 = 1.5x | `current_ratio`, `dma50_ratio` |
| Score (0-9) | `_score` or `_f_score` | integer | `piotroski_f_score` |

**Exception list** (0-100 scale despite no `_pct` suffix — grandfathered):
`percent_insiders`, `percent_institutions`, `adx_14`, `rsi_21`, `stoch_k`, `stoch_d`

New columns must follow the `_pct` convention. The exceptions above are not
extended to new columns.

### Date and Time Columns

| Column type | Convention |
|---|---|
| Calendar date | `DATE` type, suffix `_date` e.g. `price_date`, `ex_div_date` |
| Timestamp with tz | `TIMESTAMPTZ`, suffix `_at` e.g. `universe_built_at`, `created_at` |
| Fiscal year | `INTEGER` e.g. `fiscal_year = 2025` |
| Fiscal quarter | `SMALLINT` e.g. `quarter = 3` |
| Period end | `DATE` e.g. `period_end_date` |

### Boolean Flags

Boolean columns use `is_` or `above_` or `has_` prefixes:

```
is_reit, is_miner, is_asx200, is_mega
above_sma50, above_sma200, above_vwap
golden_cross, death_cross         ← exceptions (established names)
new_52w_high, new_52w_low         ← exceptions (established names)
rsi_overbought, rsi_oversold      ← exceptions (established names)
macd_bullish_cross, macd_bearish_cross
```

New boolean columns must use `is_`, `above_`, or `has_` prefixes.

### CAGR and Growth Rate Columns

```
{metric}_growth_1y      1-year growth (YoY)
{metric}_cagr_3y        3-year CAGR
{metric}_cagr_5y        5-year CAGR
{metric}_growth_hoh     Half-on-half growth (ASX only)
{metric}_growth_yoy_q   Year-on-year quarterly growth
avg_{metric}_3y         3-year rolling average
avg_{metric}_5y         5-year rolling average
```

---

## File and Directory Naming

### Raw Zone Files

```
{dataset}_{exchange}_{date}_{run_id}.{ext}

eod_prices_AU_2026-05-16_083002.json.gz
fundamentals_NSE_2026-05-01_130000.json.gz
short_positions_AU_2026-05-14_090000.csv
form4_insider_NYSE_2026-05-16_233500.json.gz
indices_GLOBAL_2026-05-16_230000.json.gz
forex_2026-05-16_230000.json.gz
commodities_2026-05-16_230000.json.gz
crypto_2026-05-16_230000.json.gz
```

### Script Files

```
{action}_{target}.py

download_eod_prices.py
load_staging_au.py
load_staging_in.py
load_staging_us.py
load_staging_charts.py
transform_daily_prices.py
compute_daily_metrics.py
build_screener_universe.py
```

### Log Files

```
{pipeline}_{market}.log

daily_pipeline_au.log
daily_pipeline_in.log
daily_pipeline_us.log
charting_pipeline.log
```

---

## Code and Environment Variables

### Environment Variables

```
# Market-prefixed where market-specific
DATABASE_URL_SYNC_AU=postgresql://...
DATABASE_URL_SYNC_IN=postgresql://...
DATABASE_URL_SYNC_US=postgresql://...
DATABASE_URL_SYNC_CHARTS=postgresql://...

# Provider API keys (shared, not market-specific)
EODHD_API_KEY=...
NSE_INDIA_API_KEY=...
SEC_EDGAR_USER_AGENT=...

# Data lake path (shared)
DATA_LAKE_PATH=/opt/data-lake/raw

# Redis (shared or per-project)
REDIS_URL=redis://localhost:6379/0
REDIS_URL_AU=redis://localhost:6379/1
REDIS_URL_IN=redis://localhost:6379/2
REDIS_URL_US=redis://localhost:6379/3
REDIS_URL_CHARTS=redis://localhost:6379/4
```

### Redis Cache Key Namespaces

```
au:screener:*           ASX Screener cache
in:screener:*           India Screener cache
us:screener:*           US Screener cache
charts:*                Charting cache
```

Redis databases (0-15) are allocated per project to prevent key collisions:
- DB 0: Shared / session
- DB 1: ASX Screener
- DB 2: India Screener
- DB 3: US Screener
- DB 4: Charting

### API Endpoint Namespacing

```
/api/v1/au/screener          ASX Screener
/api/v1/in/screener          India Screener
/api/v1/us/screener          US Screener
/api/v1/charts/              Charting
/api/v1/auth/                Shared auth
```

---

## Instrument Namespace (Charting Project)

The Charting project handles instruments from multiple markets and asset classes.
It uses a unified instrument key to avoid collisions:

```
{ASSET_CLASS}:{CODE}

AU:BHP          BHP Group — ASX equity
US:BHP          BHP Group ADR — NYSE equity
IN:RELIANCE     Reliance Industries — NSE equity
US:AAPL         Apple Inc — NASDAQ equity
IN:TCS          Tata Consultancy — NSE equity
AU:CBA          Commonwealth Bank — ASX equity
IDX:SPX         S&P 500 Index
IDX:FTSE        FTSE 100 Index
IDX:N225        Nikkei 225 Index
FX:AUDUSD       AUD/USD Forex pair
FX:USDINR       USD/INR Forex pair
CMD:GOLD        Gold (spot)
CMD:OIL         Crude Oil (WTI)
CRYPTO:BTC      Bitcoin
CRYPTO:ETH      Ethereum
```

Screener projects do NOT need this prefix (they are single-market, equities only).

---

## Git Repository Structure

Each project lives in its own repository. The data lake lives in a separate repository.

```
data-platform-docs/             ← this repo — architecture source of truth
  README.md
  docs/

data-lake/                      ← shared, platform-owned
  jobs/
    download_eodhd_au.py
    download_eodhd_nse_bse.py
    download_eodhd_us.py
    load_staging_au.py
    load_staging_in.py
    load_staging_us.py
  docs/

asx-screener/                   ← Project A
  backend/
  frontend/
  scripts/
  compute/
  jobs/
  database/migrations/
  docs/platform/ → link to data-platform-docs

nifty-screener/                 ← Project B (future)
  ...
  docs/platform/ → link to data-platform-docs

us-screener/                    ← Project C (future)
  ...

charting/                       ← Project D (future)
  jobs/
    download_charts_data.py     ← Charting-owned download job
    load_staging_charts.py      ← Charting-owned staging load
    charting_pipeline.py        ← Charting transform + build pipeline
  ...
```

The `data-platform-docs` repository is the **source of truth** for all architecture decisions.
All projects should link to or copy these docs at project creation and keep them in sync.
Architecture changes must be communicated to all project teams.
