# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A single-file ClickHouse hourly aggregation pipeline (`hourly_aggregation_pipeline.py`) that reads 5-minute granularity data from `ai_metrics_5m_v2` and aggregates it into `ai_service_features_hourly`.

## Running the Pipeline

```bash
pip3 install requests
python3 hourly_aggregation_pipeline.py
```

**Cron (every hour at 25th minute):**
```bash
25 * * * * cd /path/to/script && python3 hourly_aggregation_pipeline.py >> /var/log/hourly_aggregation/cron.log 2>&1
```

## Key Architecture Principles

**Monotonic Processing with Partial Hours**
- Only the latest in-flight hour is protected (`max(ts) - 1 HOUR`)
- All older hours are aggregated even if incomplete — partial hours are by design (~81% of records have < 12 windows)
- No lookbacks or gap scanning — processes forward from the last completed hour
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
- `ch_datetime()` (lines 72-88) handles ClickHouse DateTime values that may be returned as integers (Unix timestamps) or ISO strings over HTTP JSON — always use this when parsing timestamp fields from query results

## Critical Code Sections

**Never modify:**
- Lines 73-89: `ch_datetime()` — handles int, string digit, and ISO string formats from ClickHouse HTTP API; treats epoch 0 and pre-2000 dates as None
- Lines 165-172: `get_latest_safe_hour_from_5min()` — protects the in-flight hour
- Lines 173-184: `get_latest_hourly_hour()` — enforces the two-metric invariant with `COUNT(DISTINCT metric) = 2`
- GROUP BY clauses in aggregation queries — ensures per-hour separation even in batch mode

**Safe to modify:**
- Batch size (currently 24 hours, line 332)
- `CH_STATE_TABLE` name (line 35)
- Database credentials (lines 30-33) — read from env vars `CH_HOST` and `CH_PASSWORD`, with hardcoded fallbacks

## Database Schema

**Source:** `metrics.ai_metrics_5m_v2` — 5-min windows. Fields used by the pipeline: `success_rate`, `success_target`, `response_success_rate`, `response_target_percent`, `total_count`, `response_breach_count`, `sum_response_time`, `p90_latency`, `project_id`. Additional fields present but not yet used: `application_name`, `success_count`, `error_count`, `error_rate`, `response_slo_seconds`, `avg_latency`, `p80_latency`, `p95_latency`, `burn_rate`, `eb_health`, `response_health`, `region`, `deploy_version`, `ingestion_time`, `processed_window`

**Target:** `metrics.ai_service_features_hourly` — `ReplacingMergeTree(updated_at)`, ordered by `(application_id, service_id, service, metric, ts_hour)`, partitioned by `toYYYYMM(ts_hour)`. Includes `project_id Int64` sourced from `ai_metrics_5m_v2`.

Each hour produces two independent rows: one with `metric='success_rate'` and one with `metric='latency'`. Both rows read ALL 5-minute windows but use different source fields. The latency row has non-NULL values for `response_breach_count`, `avg_latency`, and `p90_latency`; the success_rate row has NULL for those.

## Idempotency

`ReplacingMergeTree(updated_at)` ensures safe reprocessing — multiple concurrent runs will not create duplicates. Pipeline always starts from `last_completed_hour + 1`.
