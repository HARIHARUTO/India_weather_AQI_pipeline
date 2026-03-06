# Technical Documentation: India AQI Weather Pipeline

## 1) Objective

Build a reliable, API-first pipeline to answer:

- How AQI behaves across major Indian cities over time.
- Whether rainfall, humidity, and wind correlate with AQI improvement or degradation.

Why this design:
hourly AQI and weather are time-series datasets with uneven quality, so the pipeline prioritizes validation, observability, and reproducible orchestration over ad-hoc analysis.

## 2) Data Sources

- AQI API (data.gov.in / CPCB resource)
  - Pollutant-level readings by city/station/timestamp.
- Weather API (OpenWeatherMap)
  - City-level weather observations (temperature, humidity, wind, rain).

Why these sources:
both are API-accessible and automation-friendly, avoiding scraper fragility and compliance risk.

## 3) End-to-End Data Flow

```text
data.gov.in AQI API ----\
                         +--> schema_validator.py --> raw_aqi (BigQuery) --> dbt staging
OpenWeatherMap API -----/                              |                     |
                                                      +--> invalid_records   +--> dbt intermediate
                                                                                     |
                                                                                     +--> dbt marts + features
                                                                                              |
                                                                                              +--> Looker Studio
```

1. Airflow triggers ingestion tasks in parallel.
2. Ingestion validates payload shape and values.
3. Valid rows go to `raw_aqi`; invalid rows go to `raw_aqi.invalid_records`.
4. dbt builds staging, intermediate, marts, and feature models.
5. dbt tests enforce model quality gates.
6. Final task logs run summary.

Why this flow:
separating validation, raw landing, and transformation isolates failures and makes root-cause debugging faster.

## 4) Operational Timeline (2026-03-06)

- Stack stabilized after folder/path cleanup.
- Multiple manual triggers initially queued due to `max_active_runs=1`.
- One `ingest_aqi_data` attempt received `SIGTERM` when services were restarted mid-run.
- Retries and later runs completed successfully end-to-end.
- Verified successful chains:
  - `ingest_aqi_data`
  - `ingest_weather_data`
  - `dbt_seed`
  - `dbt_run_staging`
  - `dbt_run_intermediate`
  - `dbt_run_marts`
  - `dbt_test`
  - `log_run_summary`

Why this matters:
this is real ops behavior, not only happy-path execution, and demonstrates recovery under scheduler/task interruption.

## 5) Warehouse Layers

- Raw dataset: `raw_aqi`
  - `aqi_readings`
  - `weather_readings`
  - `invalid_records`
- dbt dataset: `dbt_aqi`
  - `stg_*` cleaned models
  - `int_*` business logic joins
  - `mart_*` analytics tables
  - `features_city_air_quality` feature table

Warehouse optimization:

- Date/time marts partitioned on event or run timestamps.
- City-heavy marts clustered by `canonical_city`.

Why:
dashboard access patterns are mostly city + date filters; partition pruning and clustering reduce scanned bytes and improve query latency.

## 6) Validation and Quarantine Strategy

- Schema/value validation runs before BigQuery load.
- Invalid records are quarantined with raw payload and error metadata.
- Main raw tables accept only validated records.

Why:
failing fast at ingestion prevents silent corruption from propagating into dbt and dashboard outputs.

## 7) Airflow Orchestration

DAG: `india_aqi_weather_pipeline`

Task sequence:

- `ingest_aqi_data` + `ingest_weather_data` (parallel)
- `dbt_seed`
- `dbt_run_staging`
- `dbt_run_intermediate`
- `dbt_run_marts`
- `dbt_test`
- `log_run_summary`

Operational settings:

- retries + exponential backoff
- DAG-level SLA in default args
- schedule from `PIPELINE_SCHEDULE_CRON`
- `max_active_runs=1`

Why:
single active run avoids overlapping writes during local development and keeps failure analysis deterministic.

## 8) dbt Testing Strategy

Implemented checks and failure semantics:

- non-null key/timestamp/city checks:
  failure indicates upstream response drift or broken cleansing logic.
- AQI category validity:
  failure indicates mapping/range logic regression in intermediate models.
- no-future-timestamp check (with tolerance):
  failure indicates timestamp parsing or timezone handling defects.
- quality metrics freshness:
  failure indicates pipeline may have run but mart refresh path did not complete.

Why:
tests are used as release gates to catch semantic regressions, not only syntax errors.

## 9) Backfill Strategy

- `scripts/backfill_aqi.py`
  - date-range backfill with resume support using `scripts/completed_dates.txt`.
- `scripts/backfill_weather.py`
  - historical weather backfill via Open-Meteo archive endpoint.

Why:
historical coverage is required for meaningful trend/correlation analysis and for replay after outage windows.

## 10) Known Constraints and Accepted Tradeoffs

| Constraint | Why Accepted | Production Fix |
|---|---|---|
| Full refresh on some dbt workloads | Simpler behavior and compatible with free-tier development limits | Incremental models with `is_incremental()` and bounded lookback |
| File-based credentials in local Docker | Fast local setup and explicit mounting for container access | GCP Secret Manager + workload identity |
| Single local Airflow executor path | Predictable local debugging and low laptop resource usage | Celery/Kubernetes executor with multiple workers |
| Limited city coverage | OpenWeather free-tier call limits | Paid tier or alternate provider for broader city coverage |

## 11) Out of Scope

- Real-time streaming architecture (Kafka/Pub/Sub).
- ML model training and serving (feature table is prepared, model is not).
- Multi-region deployment and disaster recovery.
- Full national historical replay beyond configured backfill ranges.

## 12) Runbook

Start services:

```bash
docker compose up -d
```

Trigger DAG:

```bash
docker compose exec airflow-webserver airflow dags trigger india_aqi_weather_pipeline
```

List runs:

```bash
docker compose exec airflow-webserver airflow dags list-runs -d india_aqi_weather_pipeline --no-backfill
```

Check a run:

```bash
docker compose exec airflow-webserver airflow tasks states-for-dag-run india_aqi_weather_pipeline manual__2026-03-06T05:31:48+00:00
```

PowerShell helper (latest running run):

```powershell
$rid=(docker compose exec airflow-webserver airflow dags list-runs -d india_aqi_weather_pipeline --no-backfill | Select-String "running" | Select-Object -First 1).ToString().Split("|")[1].Trim(); Write-Host "RunId=$rid"; docker compose exec airflow-webserver airflow tasks states-for-dag-run india_aqi_weather_pipeline $rid
```

## 13) Troubleshooting Notes

- If all task states are `None`, check DAG run state first (`queued` vs `running`).
- If many runs are queued, stop triggering repeatedly and let current run complete.
- If task shows `SIGTERM`, check for scheduler/webserver/container restart events.
- Primary diagnostics:
  - `docker compose exec airflow-webserver airflow dags list-import-errors`
  - `docker compose logs --tail=200 airflow-scheduler`
  - task logs at `/opt/airflow/logs/dag_id=.../run_id=.../task_id=.../attempt=...log`

## 14) Interview Positioning

This project demonstrates:

- source contract validation
- layered warehouse modeling (staging, intermediate, marts)
- real orchestration debugging and recovery
- measurable data quality monitoring
- feature-ready outputs for downstream ML teams
