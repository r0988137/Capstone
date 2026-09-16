# AI Power Portfolio

Academic prototype of an AI-powered financial advisor: questionnaire → risk
profiling → portfolio recommendation → market news context → forecasting,
backed by a SQL star-schema data warehouse.

This repo currently covers **Weeks 1–3** of the build:

- Week 1 — repo/environment setup
- Week 2 — data warehouse design (star schema)
- Week 3 — ETL pipeline pulling historical price data from Yahoo Finance

## Project layout

```
ai_power_portfolio/
├── README.md
├── requirements.txt
├── .env.example
├── src/
│   ├── config/
│   │   └── settings.py        # env vars, ticker universe, paths
│   ├── db/
│   │   ├── schema.sql          # star schema DDL (SQLite-compatible)
│   │   └── init_db.py          # creates warehouse.db from schema.sql
│   └── etl/
│       ├── fetch_market_data.py   # pulls OHLCV history via yfinance
│       └── load_market_data.py    # transforms + loads into the warehouse
├── data/                        # warehouse.db lives here (gitignored)
├── tests/
│   └── test_etl.py
└── docs/
    └── star_schema.md           # schema explanation / ERD notes
```

## Setup

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # fill in ALPHA_VANTAGE_API_KEY if you'll use it later
```

## Run it

```bash
# 1. Create the warehouse (tables only, no data yet)
python -m src.db.init_db

# 2. Pull + load historical prices for the default ticker universe
python -m src.etl.load_market_data
```

This populates `data/warehouse.db` (SQLite) with `dim_date`, `dim_asset`, and
`fact_market_price` rows. The remaining warehouse tables
(`dim_user`, `fact_user_profile`, `fact_recommendation`, `fact_forecast`)
are created now (Week 2 deliverable) but stay empty until the questionnaire,
profiling, recommendation, and forecasting modules (Weeks 5–10) write to them.

## Why SQLite for now

SQLite keeps the prototype dependency-free and easy to demo. The schema is
plain ANSI SQL with no SQLite-only syntax beyond `AUTOINCREMENT`, so porting
to Postgres/MySQL later is a schema-only change — no ETL code changes needed.

## Notes on data source

The ETL defaults to `yfinance` (no API key required) for historical daily
OHLCV data. Alpha Vantage support is stubbed in `settings.py` for a future
switch if rate limits become an issue.

## Running in Google Colab

Open `notebooks/colab_quickstart.ipynb` in Colab (upload it, or open directly
from GitHub via File → Open notebook → GitHub tab). It:

1. Mounts your Google Drive so `warehouse.db` persists between sessions
   (Colab wipes local VM files when a runtime disconnects)
2. Clones this repo and installs `requirements.txt`
3. Points `WAREHOUSE_DB_PATH` at Drive instead of the ephemeral VM disk
4. Runs `src.db.init_db` and `src.etl.load_market_data`
5. Runs the test suite

Update `REPO_URL` in the first code cell to point at your GitHub repo before
running.

## Running the ETL on GitHub Actions

`.github/workflows/etl.yml` runs the pipeline daily on GitHub's own runners
(which have normal outbound internet access — Yahoo Finance is reachable
from there). It also runs on every push that touches `src/etl/` or `src/db/`.

Note: GitHub Actions runners are ephemeral, so each run starts from an empty
`warehouse.db` — the workflow uploads the resulting file as a build artifact
(kept 7 days) rather than persisting it long-term. For real continuity, swap
the warehouse for a hosted database (Supabase/Neon Postgres) once you're past
the prototype stage, or have the workflow commit `data/warehouse.db` back to
the repo / push it to Drive.
