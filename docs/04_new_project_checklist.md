# Starting a New Project — Checklist

Use this checklist when bootstrapping a new screener project (e.g. India Nifty,
US Screener) or any other project on this data platform.

Completing this checklist correctly ensures the new project is:
- Reading from the shared Staging layer (not downloading raw data itself)
- Fully isolated from all existing projects after the Staging layer
- Using the correct schema naming convention
- Schedulable without interfering with existing pipelines

---

## Understand What Is Shared vs What You Own

Before writing any code, be clear on what the platform provides and what your project owns:

| Layer | Who owns it | Your project does |
|---|---|---|
| Raw Zone (files) | Platform team | Nothing — read-only |
| Staging `staging.*` | Platform team | Read from it (filtered by exchange) |
| Market Transform `mkt_{market}.*` | **Your project** | Create tables, write, maintain |
| Financials `fin_{market}.*` | **Your project** | Create tables, write, maintain |
| Application `screener_{market}.*` | **Your project** | Create tables, write, maintain |

Your project pipeline **starts at the transform step** — it does NOT download files
or load staging. The platform pipeline handles that.

---

## Before You Start

- [ ] Read [01_platform_overview.md](01_platform_overview.md) — understand the full architecture
- [ ] Read [03_naming_conventions.md](03_naming_conventions.md) — confirm your market code and schema names
- [ ] Confirm which exchanges you need (e.g. `NSE`, `BSE`, `NYSE`, `NASDAQ`)
- [ ] Confirm the platform pipeline is already loading `staging.*` for your exchange — check with the platform team
- [ ] Confirm `staging.eod_prices` has data for your exchange: `SELECT MAX(date) FROM staging.eod_prices WHERE exchange = '{YOUR_EXCHANGE}';`

---

## Step 1 — Database Setup

### 1.1 Create your project's independent schemas

```sql
-- Replace {market} with your market code: in, us, etc.
-- Do NOT create stg_{market} — staging is shared and already exists
CREATE SCHEMA IF NOT EXISTS mkt_{market};
CREATE SCHEMA IF NOT EXISTS fin_{market};
CREATE SCHEMA IF NOT EXISTS screener_{market};
```

### 1.2 Create a dedicated DB role for your project

```sql
-- Create role (replace {market})
CREATE ROLE screener_{market}_app LOGIN PASSWORD 'your_strong_password';

-- Grant READ access to shared staging (project filters by exchange in app code)
GRANT USAGE  ON SCHEMA staging                    TO screener_{market}_app;
GRANT SELECT ON ALL TABLES IN SCHEMA staging      TO screener_{market}_app;

-- Grant FULL access to own independent schemas
GRANT USAGE, CREATE ON SCHEMA mkt_{market}        TO screener_{market}_app;
GRANT USAGE, CREATE ON SCHEMA fin_{market}        TO screener_{market}_app;
GRANT USAGE, CREATE ON SCHEMA screener_{market}   TO screener_{market}_app;

-- Grant READ access to shared infrastructure (auth only)
GRANT USAGE  ON SCHEMA users                      TO screener_{market}_app;
GRANT SELECT ON ALL TABLES IN SCHEMA users        TO screener_{market}_app;

-- IMPORTANT: Do NOT grant access to other projects' independent schemas
-- No access to mkt_au, fin_au, screener_au (or any other project's schemas)
```

### 1.3 Verify isolation

```sql
-- Your role CAN read staging (shared)
SET ROLE screener_{market}_app;
SELECT COUNT(*) FROM staging.eod_prices WHERE exchange = 'NSE';  -- should work

-- Your role CANNOT read another project's transform data
SELECT * FROM mkt_au.daily_prices LIMIT 1;  -- should return permission denied
RESET ROLE;
```

---

## Step 2 — Database Migrations

Create numbered migration files in `database/migrations/`:

```
001_extensions_and_schemas.sql        -- CREATE EXTENSION timescaledb; CREATE SCHEMA mkt_{market} etc.
002_mkt_{market}_companies.sql        -- Companies master table
003_mkt_{market}_timeseries.sql       -- TimescaleDB hypertables (daily_prices, daily_metrics etc.)
004_fin_{market}_tables.sql           -- Financial statement tables
005_screener_{market}_universe.sql    -- Golden Record table
```

Note: No staging migration — `staging.*` is owned by the platform, already exists.

Run migrations in order:

```bash
for f in database/migrations/*.sql; do
    psql $DATABASE_URL -f "$f" && echo "Applied: $f"
done
```

---

## Step 3 — Project Directory Structure

```
{market}-screener/
├── backend/
│   ├── app/
│   │   ├── api/v1/routes/
│   │   │   └── screener.py         ← API endpoint
│   │   ├── schemas/
│   │   │   └── screener.py         ← Pydantic models
│   │   └── core/
│   │       └── cache.py
│   └── migrations/
├── compute/
│   └── engine/
│       ├── daily_compute.py        ← reads staging.*, writes mkt_{market}.computed_metrics
│       └── yearly_compute.py       ← reads staging.*, writes mkt_{market}.yearly_metrics
├── scripts/
│   └── {provider}/
│       └── v1/
│           ├── transform_daily_prices.py   ← reads staging.eod_prices, writes mkt_{market}.*
│           └── build_screener_universe.py  ← reads mkt/fin, writes screener_{market}.universe
├── jobs/
│   └── daily_pipeline.py           ← orchestrates transform + compute + build steps only
├── database/
│   └── migrations/
├── docs/
│   └── platform/                   ← link or copy from data-platform-docs
└── .env
```

Notice: **no download scripts, no staging load scripts** — those belong to the platform.

---

## Step 4 — Environment Variables

Add to your project's `.env`:

```bash
# Database (your project-specific role — NOT the platform role)
DATABASE_URL_SYNC=postgresql://screener_{market}_app:password@localhost:5432/asx_screener

# Market identifier (used in logging, cache keys, exchange filter)
MARKET_CODE={market}          # in, us, etc.
EXCHANGE_CODES=NSE,BSE        # comma-separated list of exchange codes in staging

# Redis (use project-specific DB number to avoid key collisions)
REDIS_URL=redis://localhost:6379/{db_number}
# AU=1, IN=2, US=3, Charts=4
```

Note: No `DATA_LAKE_PATH` or `EODHD_API_KEY` needed — your project does not touch raw files.

---

## Step 5 — Transform Layer

Your transform script reads from `staging.*` (filtering by exchange) and writes
to `mkt_{market}.*`.

Key pattern:

```python
EXCHANGE_CODES = os.getenv("EXCHANGE_CODES", "NSE,BSE").split(",")

# Read from SHARED staging — always filter by exchange
cur.execute("""
    SELECT asx_code, date, open, high, low, close, volume
    FROM staging.eod_prices
    WHERE exchange = ANY(%s) AND date = %s
""", (EXCHANGE_CODES, target_date))

rows = cur.fetchall()

# Write into YOUR OWN transform schema
cur.executemany("""
    INSERT INTO mkt_{market}.daily_prices (nse_code, time, open, high, low, close, volume)
    VALUES (%s, %s, %s, %s, %s, %s, %s)
    ON CONFLICT (nse_code, time) DO UPDATE SET ...
""", rows)
```

**Never write to `staging.*`** — it is owned by the platform.

The ticker column name for each market:

| Market | Ticker column name | Example value |
|---|---|---|
| AU | `asx_code` | `BHP` |
| IN | `nse_code` | `RELIANCE` |
| US | `ticker` | `AAPL` |

---

## Step 6 — Compute Engine

The compute engine calculates derived metrics (margins, CAGR, quality scores).
It reads from `mkt_{market}.*` + `fin_{market}.*` and writes back to `mkt_{market}.*`.

Copy the ASX Screener compute engines as a starting point:
- `compute/engine/yearly_compute.py` → adapt for your market's accounting standard
- `compute/engine/daily_compute.py` → mostly universal, minor column name changes

Market-specific additions (beyond the universal set):
- **India**: Add `promoter_holdings_pct`, `pledged_pct`, `fii_net_buy` to `mkt_in.yearly_metrics`
- **US**: Add `insider_buy_signal`, `institutional_ownership_change` to `mkt_us.yearly_metrics`

---

## Step 7 — Golden Record

Your project's Golden Record is `screener_{market}.universe`.

It must follow the same pattern as `screener_au.universe`:
- One row per instrument
- All columns pre-joined and denormalised
- Rebuilt nightly via `build_screener_universe.py`
- Single `INSERT ... ON CONFLICT DO UPDATE SET` query (not multiple queries)
- TCP keepalives on the psycopg2 connection:

```python
conn = psycopg2.connect(
    DATABASE_URL,
    keepalives=1,
    keepalives_idle=30,
    keepalives_interval=10,
    keepalives_count=5,
    options="-c statement_timeout=0",
)
```

Standard columns every screener universe must have:

```sql
-- Identity
ticker, company_name, sector, industry, stock_type, status,
is_large, is_mid, is_small, is_micro, is_nano, market_cap_tier,

-- Price
price, price_date, volume, avg_volume_20d, market_cap, high_52w, low_52w,

-- Valuation
pe_ratio, forward_pe, price_to_book, price_to_sales, ev_to_ebitda,
price_to_fcf, fcf_yield,

-- Dividends
dividend_yield, dps_ttm, payout_ratio,

-- Profitability
gross_margin, ebitda_margin, net_margin, operating_margin, roe, roa, roce,

-- Growth
revenue_growth_1y, revenue_growth_3y_cagr, earnings_growth_1y,
eps_cagr_5y, ebitda_growth_1y, fcf_growth_1y,

-- Balance Sheet
debt_to_equity, current_ratio, net_debt_to_ebitda, interest_coverage,

-- Technicals
rsi_14, macd, macd_signal, sma_50, sma_200, adx_14,
above_sma50, above_sma200, golden_cross, death_cross,
dma50_ratio, dma200_ratio, relative_volume,

-- Returns
return_1w, return_1m, return_3m, return_6m, return_1y, return_ytd,
drawdown_from_ath, volatility_20d, beta_1y,

-- Quality
piotroski_f_score, altman_z_score,

-- Metadata
universe_built_at
```

Add market-unique columns on top of this standard set.

---

## Step 8 — API Backend

Copy the ASX Screener backend pattern:
- `POST /api/v1/{market}/screener` — run a screen
- `GET  /api/v1/{market}/screener/fields` — list filterable fields
- `GET  /api/v1/{market}/screener/presets` — pre-built templates

The `ALLOWED_FIELDS` registry approach (field → SQL column mapping) is mandatory.
Never accept raw column names from API clients.

Cache key prefix must use your market code:

```python
CACHE_PREFIX = f"{MARKET_CODE}:screener:"   # "in:screener:" or "us:screener:"
```

---

## Step 9 — Pipeline Orchestration

Your `jobs/daily_pipeline.py` starts from **Step 3 (Transform)**. Steps 1 and 2
(download + staging load) are handled by the platform pipeline — not your project.

```
Step 1   [PLATFORM] Download raw files            → /opt/data-lake/raw/
Step 2   [PLATFORM] Load staging                  → staging.*
─────────────────────────────────────────────────────────────────────
Step 3   [YOUR PROJECT] Transform daily prices    → mkt_{market}.daily_prices
Step 4   [YOUR PROJECT] Daily compute engine      → mkt_{market}.computed_metrics
Step 4b  [YOUR PROJECT] Technical compute         → mkt_{market}.daily_metrics
Step 4c  [YOUR PROJECT] Yearly compute            → mkt_{market}.yearly_metrics
Step 5   [YOUR PROJECT] Build Golden Record       → screener_{market}.universe
```

---

## Step 10 — Crontab Registration

Add your pipeline to the **server crontab** — run AFTER the platform staging pipeline completes:

```bash
crontab -e
```

```
# {Market} Screener daily pipeline (starts after platform staging load completes)
{minute} {hour} * * 1-5 cd /opt/{market}-screener && /opt/{market}-screener/venv/bin/python jobs/daily_pipeline.py >> /opt/{market}-screener/logs/daily_pipeline_{market}.log 2>&1
```

Check platform pipeline schedule in [01_platform_overview.md](01_platform_overview.md)
to ensure your pipeline runs after staging is loaded for your market.

---

## Step 11 — Verification Checklist

- [ ] `SELECT COUNT(*) FROM staging.eod_prices WHERE exchange = '{EXCHANGE}';` returns rows
- [ ] `SELECT COUNT(*) FROM screener_{market}.universe;` returns expected number of stocks
- [ ] `SELECT * FROM screener_{market}.universe LIMIT 5;` shows sensible data (no all-nulls rows)
- [ ] API returns 200 on `POST /api/v1/{market}/screener` with empty filters
- [ ] DB role isolation verified — role cannot read `mkt_au.*` or any other project's schemas
- [ ] DB role CAN read `staging.*` (needed for transform step)
- [ ] Crontab entry runs after platform staging pipeline (check timing in overview doc)
- [ ] Redis cache key prefix uses market code (not shared with other projects)
- [ ] `.env` uses project-specific DB role
- [ ] No download scripts or staging load scripts in your project — platform owns those
- [ ] Build script uses TCP keepalives

---

## What NOT to Do

| Do not | Reason |
|---|---|
| Create `stg_{market}.*` schemas | Staging is shared — `staging.*` already exists |
| Write to `staging.*` | Platform owns staging — your project only reads it |
| Download raw files in your pipeline | Raw zone and staging load belong to the platform |
| Read from `mkt_au.*` in the India project | Cross-project dependency — breaks isolation |
| Use another project's DB role | Gives access to the wrong project's data |
| Share a log file with another project | Makes debugging impossible |
| Use `public` schema | No namespacing — collisions guaranteed |
| Skip TCP keepalives on the build script | Long-running upsert will SSL-timeout |
| Add download jobs to your project's crontab | Downloads belong to the platform pipeline |
