# DECISIONS

## DECISION-001: API-first ingestion over scraping

- Decision: use data.gov.in + weather APIs.
- Why: predictable interfaces, lower breakage, better compliance.
- Tradeoff: less control than custom scraping targets.

## DECISION-002: BigQuery + dbt layered modeling

- Decision: raw -> staging -> intermediate -> marts/features.
- Why: traceability, testability, and clear ownership boundaries.
- Tradeoff: more upfront model structure.

## DECISION-003: Airflow orchestration

- Decision: orchestrate ingestion + dbt in one DAG.
- Why: retries, observability, and reproducible task dependencies.
- Tradeoff: heavier local setup than simple scripts.

## DECISION-004: Quarantine table before transformation

- Decision: reject malformed records from primary raw tables.
- Why: protect downstream models from schema drift and bad values.
- Tradeoff: more ingestion code and error handling paths.
