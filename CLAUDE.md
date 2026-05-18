# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A single-file ClickHouse hourly aggregation pipeline (`hourly_aggregation_pipeline.py`) that reads 5-minute granularity data from `ai_metrics_5m_v2` and aggregates it into `ai_service_features_hourly`.

## Dependencies

Only one external dependency: `requests`. No framework, no ORM.

## Running the Pipeline

**Locally:**
```bash
pip3 install requests
python3 hourly_aggregation_pipeline.py
```

**GitHub Actions (primary execution path):**
- Runs automatically at `25 * * * *` via `.github/workflows/hourly_pipeline.yml`
- Manual trigger: GitHub UI → Actions → "Hourly Aggregation Pipeline" → "Run workflow"
- Requires `CH_HOST` and `CH_PASSWORD` set as repository secrets (Settings → Secrets and variables → Actions)

**Cron (alternative, local server):**
```bash
25 * * * * cd /path/to/script && python3 hourly_aggregation_pipeline.py >> /var/log/hourly_aggregation/cron.log 2>&1
```

## Execution Flow

`run()` orchestrates the pipeline in this order:
1. `ensure_hourly_table()` / `ensure_state_table()` — idempotent DDL, safe to call every run
2. `get_latest_safe_hour_from_5min()` — upper bound (excludes in-flight hour)
3. `get_latest_hourly_hour()` — lower bound (last hour with both metrics written)
4. Cold-start fallback: if hourly table is empty, use `get_earliest_hour_from_5min()`
5. Batch or one-by-one loop: calls `aggregate_success_rate()` + `aggregate_latency()` per window
6. `_save_state()` — writes audit row to `hourly_pipeline_state`

## Key Architecture Principles

**Monotonic Processing with Partial Hours**
- Only the latest in-flight hour is protected (`max(ts) - 1 HOUR`)
- All older hours are aggregated even if incomplete — partial hours are by design (~81% of records have < 12 windows)
- No lookbacks or gap scanning — processes forward from the last completed hour
- Cold start (empty hourly table): begins from `min(toStartOfHour(ts))` in the source table
- Source of truth is always `ai_metrics_5m_v2`

**Batch Processing Strategy**
- Gap < 24 hours: process one-by-one
- Gap >= 24 hours: process in 24-hour batches, then remaining hours one-by-one
- Optimized for long downtime recovery (e.g. 1,980 hours in ~88 seconds)

**Two-Metric Invariant**
- An hour is only considered "complete" when BOTH `success_rate` AND `latency` metrics exist (`COUNT(DISTINCT metric) = 2`)
- This prevents skipping ahead if one metric fails and enables automatic recovery

**State Table is Audit-Only**
- `metrics.hourly_pipeline_state` (ClickHouse) stores one row per pipeline run — never read by the pipeline for processing decisions
- Pipeline determines processing range by querying the database directly (`ai_service_features_hourly` + `ai_metrics_5m_v2`)
- Each row records: `run_id`, `started_at`, `finished_at`, `status`, `source_latest_safe_hour`, `last_processed_hour_before_run`, `first_hour_processed`, `last_hour_processed`, `total_hours_processed`, `batch_mode`, `batch_count`, `duration_seconds`

**DateTime Handling**
- `ch_datetime()` (lines 73-90) handles ClickHouse DateTime values that may be returned as integers (Unix timestamps) or ISO strings over HTTP JSON — always use this when parsing timestamp fields from query results

**HTTP API Client**
- `ClickHouseClient` uses ClickHouse's HTTPS interface (port 443), not a native driver
- Both `execute()` and `execute_json()` pass `verify=False` — the server uses a self-signed certificate; `urllib3.disable_warnings()` suppresses the resulting noise at module load
- `execute()` — raw query, returns text
- `execute_json()` — automatically appends `FORMAT JSONEachRow`, returns list of dicts; do not include semicolons in queries passed to this method

## Critical Code Sections

**Never modify:**
- Lines 76-93: `ch_datetime()` — handles int, string digit, and ISO string formats from ClickHouse HTTP API; treats epoch 0 and pre-2000 dates as None
- Lines 168-174: `get_latest_safe_hour_from_5min()` — protects the in-flight hour
- Lines 176-187: `get_latest_hourly_hour()` — enforces the two-metric invariant with `COUNT(DISTINCT metric) = 2`
- GROUP BY clauses in aggregation queries — ensures per-hour separation even in batch mode

**Safe to modify:**
- Batch size (currently 24 hours, line 335)
- `CH_STATE_TABLE` name (line 38)
- Database credentials (lines 33-36) — `CH_HOST` and `CH_PASSWORD` read from env vars; `CH_USERNAME` is hardcoded as `"wm_test"` and has no env var override
- `_save_state()` (lines 369-400) — pure audit logging; never affects processing decisions

**Security note:** Lines 33 and 36 contain hardcoded default values for `CH_HOST` and `CH_PASSWORD` used as fallbacks when env vars are unset. In production (GitHub Actions), these are overridden by repository secrets. Do not remove the `os.environ.get()` calls or change them to hardcoded-only values.

## Extending the Schema (Adding a New Column)

Edit exactly three places and keep the column order in INSERT SELECT matching the DDL order:

1. **`ensure_hourly_table()`** — add column to the `CREATE TABLE IF NOT EXISTS` DDL
2. **`aggregate_success_rate()`** — add the corresponding SELECT expression at the same position; use `NULL` for latency-only columns
3. **`aggregate_latency()`** — add the corresponding SELECT expression at the same position; use `NULL` for success_rate-only columns

The INSERT SELECT is positional (no column name list), so order must match the DDL exactly. Mismatched position silently inserts into the wrong column.

Also update the `GROUP BY` in both aggregation methods if the new column is a dimension (not a measure).

## Database Schema

**Source:** `metrics.ai_metrics_5m_v2` — 5-min windows. Fields used by the pipeline: `success_rate`, `success_target`, `response_success_rate`, `response_target_percent`, `total_count`, `response_breach_count`, `sum_response_time`, `p90_latency`, `project_id`. Additional fields present but not yet used: `application_name`, `success_count`, `error_count`, `error_rate`, `response_slo_seconds`, `avg_latency`, `p80_latency`, `p95_latency`, `burn_rate`, `eb_health`, `response_health`, `region`, `deploy_version`, `ingestion_time`, `processed_window`

**Target:** `metrics.ai_service_features_hourly` — `ReplacingMergeTree(updated_at)`, ordered by `(application_id, service_id, service, metric, ts_hour)`, partitioned by `toYYYYMM(ts_hour)`. Includes `project_id Int64` sourced from `ai_metrics_5m_v2`.

Each hour produces two independent rows: one with `metric='success_rate'` and one with `metric='latency'`. Both rows read ALL 5-minute windows but use different source fields. The latency row has non-NULL values for `response_breach_count`, `avg_latency`, and `p90_latency`; the success_rate row has NULL for those.

`avg_latency` is computed as `SUM(sum_response_time) / SUM(total_count)` (true weighted average). `p90_latency` is a weighted average of per-window p90s (`SUM(p90_latency * total_count) / SUM(total_count)`) — an approximation, not a true p90 across all requests.

`ReplacingMergeTree` deduplication is asynchronous — immediate reads after INSERT may still return duplicate rows until ClickHouse merges the parts in the background.

## Tests

There is no test suite. Correctness is validated by running the pipeline against the real database and inspecting `ai_service_features_hourly` and `hourly_pipeline_state` directly.

## Idempotency

`ReplacingMergeTree(updated_at)` ensures safe reprocessing — multiple concurrent runs will not create permanent duplicates. Pipeline always starts from `last_completed_hour + 1`.

Both `ai_service_features_hourly` and `hourly_pipeline_state` are created with `CREATE TABLE IF NOT EXISTS` on every run, so no manual schema setup is needed.
