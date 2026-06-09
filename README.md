# TfL Analytics Platform

An end-to-end analytics engineering portfolio project ingesting real-time Transport for London (TfL) tube data, transforming it with dbt, and visualising it in Looker Studio — built entirely on free-tier infrastructure.

> **Target roles:** Senior Data Analyst / Analytics Engineer (UK)

---

## Architecture

```
TfL Unified API
      │
      ▼
Python ingest (tfl_ingest.py)
      │  batch load (load_table_from_file)
      ▼
BigQuery: raw dataset
      │
      ▼
dbt Core
  ├── staging   (stg_tfl__line_status, stg_tfl__arrivals)
  ├── intermediate (int_disruptions, int_arrivals_performance)
  └── marts     (dim_line, dim_station, fct_line_status, fct_disruptions, fct_arrivals)
      │
      ▼
Looker Studio dashboard (native BigQuery connector)
```

Orchestration runs locally via **Apache Airflow (Docker Compose)**. A **GitHub Actions cron** job acts as the fallback scheduler and runs `dbt test` on every pull request.

---

## Tech Stack

| Layer | Tool | Notes |
|---|---|---|
| Data source | [TfL Unified API](https://api.tfl.gov.uk) | Tube line status + arrivals endpoints |
| Language | Python 3.11 | Ingest scripts |
| Orchestration | Apache Airflow (Docker Compose) | Local; GitHub Actions cron as fallback |
| Warehouse | Google BigQuery | Free tier — batch loads only |
| Transformation | dbt Core | Staging → Intermediate → Marts |
| CI | GitHub Actions | dbt test on PRs + hourly ingest cron |
| Dashboard | Looker Studio | Native BigQuery connector |
| Docs | dbt docs → GitHub Pages | Auto-published on merge to main |

---

## BigQuery Dataset Structure

```
raw          ← Python batch loads land here
staging      ← dbt: typed, renamed, light cleaning
intermediate ← dbt: business logic (LAG-based disruption detection, arrival performance)
marts        ← dbt: analytics-ready fact & dimension tables
```

### dbt Models

**Staging**
- `stg_tfl__line_status` — source freshness: warn 75 min / error 2 hr
- `stg_tfl__arrivals` — source freshness: warn 75 min / error 2 hr

**Intermediate**
- `int_disruptions` — LAG-based status change detection per line
- `int_arrivals_performance` — scheduled vs actual arrival calculations

**Marts**
- `dim_line`, `dim_station` — slowly changing dimensions
- `fct_line_status` — incremental, partitioned by date
- `fct_disruptions` — incremental, partitioned by date
- `fct_arrivals` — incremental, partitioned by date

---

## Key Design Decisions

**No streaming inserts.** BigQuery ingestion uses `load_table_from_file` (batch loads) throughout. Streaming inserts incur per-row costs; batch loads are free within the free tier.

**All incremental models are date-partitioned.** Keeps query costs and scan sizes predictable as the table grows.

**Source freshness enforced in dbt.** Both staging sources warn at 75 minutes and error at 2 hours, catching scheduler failures before stale data reaches the marts.

**Airflow runs locally via Docker Compose.** No managed Airflow cost. GitHub Actions cron is the fallback so the pipeline stays alive even when the local machine is off.

---

## Project Phases

| Phase | Scope | Status |
|---|---|---|
| **Phase 1** | Repo setup, architecture design, README, BigQuery schema design | **In progress** |
| Phase 2 | Python ingest script (`tfl_ingest.py`), raw BigQuery loads | Planned |
| Phase 3 | dbt project — staging + intermediate models | Planned |
| Phase 4 | dbt marts (incremental fact tables, dimensions) | Planned |
| Phase 5 | Airflow DAGs + Docker Compose orchestration | Planned |
| Phase 6 | GitHub Actions CI (dbt test on PR + cron fallback) | Planned |
| Phase 7 | Looker Studio dashboard + dbt docs on GitHub Pages | Planned |

---

## Getting Started

### Prerequisites

- Python 3.11+
- Docker Desktop (for Airflow)
- A Google Cloud project with BigQuery enabled (free tier is sufficient)
- A [TfL API key](https://api-portal.tfl.gov.uk/) (free registration)
- dbt Core (`pip install dbt-bigquery`)

### Setup

```bash
git clone https://github.com/<your-username>/tfl-analytics-project.git
cd tfl-analytics-project

# Python environment
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# dbt
cd dbt
dbt deps
dbt debug   # confirms BigQuery connection
```

Credentials and environment variables are documented in `.env.example` (never committed).

---

## Repository Structure

```
tfl-analytics-project/
├── ingest/              # Python ingest scripts
│   └── tfl_ingest.py
├── dbt/                 # dbt project
│   ├── models/
│   │   ├── staging/
│   │   ├── intermediate/
│   │   └── marts/
│   ├── tests/
│   └── dbt_project.yml
├── airflow/             # DAGs + Docker Compose
│   ├── dags/
│   └── docker-compose.yml
├── .github/
│   └── workflows/       # CI: dbt test + ingest cron
├── .env.example
└── README.md
```

---

## Data Sources

- **Line status:** `GET /Line/Mode/tube/Status` — current status for all tube lines
- **Arrivals:** `GET /Line/{id}/Arrivals` — predicted arrivals per line

TfL API terms of service: data is free to use with attribution. See [tfl.gov.uk/corporate/terms-conditions](https://tfl.gov.uk/corporate/terms-conditions/tfl-api) for details.

---

## License

MIT
