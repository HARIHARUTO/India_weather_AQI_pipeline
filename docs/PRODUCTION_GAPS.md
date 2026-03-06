# PRODUCTION_GAPS

## Current Gaps

- Secrets are file-based in local Docker.
- Some dbt paths still run full refresh behavior.
- Single local scheduler/executor path.
- Manual monitoring for anomalies.

## Production Upgrades

- Move secrets to GCP Secret Manager + workload identity.
- Convert heavy marts to incremental models with bounded lookback.
- Use multi-worker executor (Celery/Kubernetes).
- Add alerting for:
  - task failures
  - quarantine spikes
  - freshness breaches
  - missing-city thresholds
