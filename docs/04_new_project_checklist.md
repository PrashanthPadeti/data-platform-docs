# Starting a New Project — Checklist

Use this checklist when bootstrapping a new screener project (e.g. India Nifty,
US Screener) or any other project on this data platform.

Completing this checklist correctly ensures the new project is:
- Reading from the correct per-market Staging schema (not downloading raw data itself)
- Fully isolated from all existing projects after the Staging layer
- Using the correct schema naming convention
- Schedulable without interfering with existing pipelines

---

## Understand What Is Shared vs What You Own

Before writing any code, be clear on what the platform provides and what your project owns:

| Layer | Who owns it | Your project does |
|---|---|---|
| Raw Zone (files) | Platform team | Nothing — read-only |
| Staging `staging_au.*` | Platform team | Read from it (ASX Screener only) |
| Staging `staging_in.*` | Platform team | Read from it (India Screener only) |
| Staging `staging_us.*` | Platform team | Read from it (US Screener only) |
| Staging `staging_charts.*` | **Charting project** | Create views + load own tables (Charting only) |
| Market Transform `mkt_{market}.*` | **Your project** | Create tables, write, maintain |
| Financials `fin_{market}.*` | **Your project** | Create tables, write, maintain |
| Application `screener_{market}.*` | **Your project** | Create tables, write, maintain |

Your project pipeline **starts at the transform step** — it does NOT download files
or load the platform staging schemas. The platform pipeline handles that.

**Code separation rule**: No shared business logic or application code is allowed
across projects. Your project's code is fully independent after the staging layer.

---

## Before You Start

- [ ] Read [01_platform_overview.md](01_platform_overview.md) — understand the full architecture
- [ ] Read [03_naming_conventions.md](03_naming_conventions.md) — confirm your market code and schema names
- [ ] Confirm which staging schema your project reads from (`staging_au`, `staging_in`, or `staging_us`)
- [ ] Confirm the platform pipeline is already loading your market's staging schema — check with the platform team
- [ ] Verify staging data exists: `SELECT MAX(date) FROM staging_{market}.eod_prices;`

---

## Step 1 — Database Setup

### 1.1 Create your project's independent schemas

```sql
-- Replace {market} with your market code: in, us, etc.
-- Do NOT create staging_{market} — that is platform-owned and already exists
CREATE SCHEMA IF NOT EXISTS mkt_{market};
CREATE SCHEMA IF NOT EXISTS fin_{market};
CREATE SCHEMA IF NOT EXISTS screener_{market};
```

### 1.2 Create a dedicated DB role for your project

```sql
-- Create role (replace {market})
CREATE ROLE screener_{market}_app LOGIN PASSWORD 'your_strong_password';

-- Grant READ access to your market's staging schema
GRANT USAGE  ON SCHEMA staging_{market}               TO screener_{market}_app;
GRANT SELECT ON ALL TABLES IN SCHEMA staging_{market} TO screener_{market}_app;

-- Grant FULL access to own independent schemas
GRANT USAGE, CREATE ON SCHEMA mkt_{market}            TO screener_{market}_app;
GRANT USAGE, CREATE ON SCHEMA fin_{market}            TO screener_{market}_app;
GRANT USAGE, CREATE ON SCHEMA screener_{market}       TO screener_{market}_app;

-- Grant READ access to shared infrastructure (auth only)
GRANT USAGE  ON SCHEMA users                          TO screener_{market}_app;
GRANT SELECT ON ALL TABLES IN SCHEMA users            TO screener_{market}_app;

-- IMPORTANT: Do NOT grant access to other projects' staging or independent schemas
-- No access to staging_au (if you're not ASX), staging_in, staging_us (cross-market)
-- No access to mkt_au, fin_au, screener_au (or any other project's schemas)
```

### 1.3 Verify isolation

```sql
-- Your role CAN read your market's staging
SET ROLE screener_{market}_app;
SELECT COUNT(*) FROM staging_{market}.eod_prices;  -- should work

-- Your role CANNOT read another project's staging
SELECT * FROM staging_au.eod_prices LIMIT 1;       -- should return permission denied (if not AU project)

-- Your role CANNOT read another project's transform data
SELECT * FROM mkt_au.daily_prices LIMIT 1;         -- should return permission denied
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

Note: No staging migration — `staging_{market}.*` is owned by the platform, already exists.

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
│       ├── daily_compute.py        ← reads mkt_{market}.*, writes mkt_{market}.computed_metrics
│       └── yearly_compute.py       ← reads fin_{market}.*, writes mkt_{market}.yearly_metrics
├── scripts/
│   └── {provider}/
│       └── v1/
│           ├── transform_daily_prices.py   ← reads staging_{market}.eod_prices, writes mkt_{market}.*
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

# Market identifier (used in logging, cache keys)
MARKET_CODE={market}          # in, us, etc.

# Redis (use project-specific DB number to avoid key collisions)
REDIS_URL=redis://localhost:6379/{db_number}
# AU=1, IN=2, US=3, Charts=4
```

Note: No `DATA_LAKE_PATH`, `EODHD_API_KEY`, or `EXCHANGE_CODES` needed —
your project does not touch raw files and reads from a market-specific staging schema
that already contains only your market's data.

---

## Step 5 — Transform Layer

Your transform script reads from `staging_{market}.*` and writes to `mkt_{market}.*`.

No `WHERE exchange = ?` filter needed — the staging schema already contains only your market's data.

Key pattern:

```python
# Read from YOUR MARKET'S staging schema — no exchange filter needed
cur.execute("""
    SELECT asx_code, date, open, high, low, close, volume
    FROM staging_{market}.eod_prices
    WHERE date = %s
""", (target_date,))

rows = cur.fetchall()

# Write into YOUR OWN transform schema
cur.executemany("""
    INSERT INTO mkt_{market}.daily_prices (ticker, time, open, high, low, close, volume)
    VALUES (%s, %s, %s, %s, %s, %s, %s)
    ON CONFLICT (ticker, time) DO UPDATE SET ...
""", rows)
```

**Never write to `staging_{market}.*`** — it is owned by the platform.

The ticker column name for each market:

| Market | Staging schema | Ticker column name | Example value |
|---|---|---|---|
| AU | `staging_au` | `asx_code` | `BHP` |
| IN | `staging_in` | `nse_code` | `RELIANCE` |
| US | `staging_us` | `ticker` | `AAPL` |

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

See [05_pipeline_standards.md](05_pipeline_standards.md) for the full standard column list.

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
Step 1   [PLATFORM] Download raw files              → /opt/data-lake/raw/
Step 2   [PLATFORM] Load staging_{market}.*         → staging_{market}.*
─────────────────────────────────────────────────────────────────────────
Step 3   [YOUR PROJECT] Transform daily prices      → mkt_{market}.daily_prices
Step 4   [YOUR PROJECT] Daily compute engine        → mkt_{market}.computed_metrics
Step 4b  [YOUR PROJECT] Technical compute           → mkt_{market}.daily_metrics
Step 4c  [YOUR PROJECT] Yearly compute              → mkt_{market}.yearly_metrics
Step 5   [YOUR PROJECT] Build Golden Record         → screener_{market}.universe
```

Your pipeline must run as an independent process. A failure in your pipeline
must not stop or affect any other project's pipeline.

---

## Step 10 — Crontab Registration

Add your pipeline to the **server crontab** — run AFTER the platform staging pipeline completes
for your market:

```bash
crontab -e
```

```
# {Market} Screener daily pipeline (starts after platform staging_{market} load completes)
{minute} {hour} * * 1-5 cd /opt/{market}-screener && /opt/{market}-screener/venv/bin/python jobs/daily_pipeline.py >> /opt/{market}-screener/logs/daily_pipeline_{market}.log 2>&1
```

Check platform pipeline schedule in [01_platform_overview.md](01_platform_overview.md)
to ensure your pipeline runs after `staging_{market}.*` is fully loaded.

---

## Step 11 — Verification Checklist

- [ ] `SELECT COUNT(*) FROM staging_{market}.eod_prices;` returns rows
- [ ] `SELECT COUNT(*) FROM screener_{market}.universe;` returns expected number of stocks
- [ ] `SELECT * FROM screener_{market}.universe LIMIT 5;` shows sensible data (no all-nulls rows)
- [ ] API returns 200 on `POST /api/v1/{market}/screener` with empty filters
- [ ] DB role isolation verified — role cannot read `mkt_au.*` or any other project's schemas
- [ ] DB role CAN read `staging_{market}.*` (needed for transform step)
- [ ] DB role CANNOT read other markets' staging schemas (e.g. `staging_au` if you're not ASX)
- [ ] Crontab entry runs after platform staging pipeline for your market (check timing in overview doc)
- [ ] Redis cache key prefix uses market code (not shared with other projects)
- [ ] `.env` uses project-specific DB role
- [ ] No download scripts or staging load scripts in your project — platform owns those
- [ ] Build script uses TCP keepalives
- [ ] No `EXCHANGE_CODES` env var — schema separation replaces exchange filtering

---

## What NOT to Do

| Do not | Reason |
|---|---|
| Create `staging_{market}.*` schemas | Platform staging already exists — don't recreate it |
| Write to `staging_{market}.*` | Platform owns staging — your project only reads it |
| Download raw files in your pipeline | Raw zone and staging load belong to the platform |
| Use `WHERE exchange = ?` filters on staging | Staging schemas are already per-market — no filter needed |
| Read from another project's staging | e.g. India project must not read `staging_au.*` |
| Read from `mkt_au.*` in the India project | Cross-project dependency — breaks isolation |
| Use another project's DB role | Gives access to the wrong project's data |
| Share a log file with another project | Makes debugging impossible |
| Use `public` schema | No namespacing — collisions guaranteed |
| Skip TCP keepalives on the build script | Long-running upsert will SSL-timeout |
| Add download jobs to your project's crontab | Downloads belong to the platform pipeline |
| Share business logic or application code with another project | Violates code separation rule |
