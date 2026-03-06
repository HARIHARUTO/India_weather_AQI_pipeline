# LEARNINGS

- Queue state interpretation matters:
  task state `None` is often scheduler timing, not model failure.
- Retries are only useful with operational discipline:
  restarting services during active tasks can create false-negative failures.
- Time alignment is core in multi-source analytics:
  hourly truncation fixed AQI-weather join quality.
- Data quality has to be explicit:
  quarantine + dbt tests are non-negotiable for trustworthy marts.
- One-run-at-a-time local operations improve signal:
  repeated manual triggers create noisy run queues and harder debugging.
