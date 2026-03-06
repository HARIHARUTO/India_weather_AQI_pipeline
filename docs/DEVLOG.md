# DEVLOG

## Week 1: Foundation

- Finalized API-first direction for AQI + weather.
- Implemented ingestion loaders and BigQuery raw landing.

## Week 2: Modeling and Quality

- Built dbt staging/intermediate/marts.
- Added tests and quarantine-aware data quality flow.

## Week 3: Orchestration

- Added Airflow DAG for parallel ingestion and sequential dbt.
- Tuned retries and scheduler settings for local stability.

## 2026-03-06: Reliability Debug Session

- Observed queued runs and `None` task states.
- Identified `max_active_runs=1` behavior and trigger backlog effect.
- Confirmed one `SIGTERM` came from service restart during run.
- Validated repeated successful end-to-end DAG runs after stabilization.
