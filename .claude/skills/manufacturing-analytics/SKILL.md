---
name: manufacturing-analytics
description: Generates dbt-style SQL models, data quality YAML tests, and canonical metric definitions for manufacturing datasets. Use when working with manufacturing KPIs, defect analysis, OEE, downtime, or production pipelines.
dependencies: python>=3.8
---

# Manufacturing Analytics Skill

## What This Skill Does
Teaches Claude Code to produce production-ready dbt artifacts for manufacturing data.
Invoked whenever a user asks to: generate a dbt model, define a canonical KPI, write quality tests, or resolve conflicting metric definitions.

## Dataset Schema (manufacturing_defect_dataset.csv)
Exact column names to use in all SQL:
- machine_id, shift_id (synthesized: M-1..M-5, Morning/Afternoon/Night)
- production_volume (INT, 100–999)
- production_cost (FLOAT, 5600–20000)
- supplier_quality (FLOAT, 81–99)
- delivery_delay (INT, 0–5)
- defect_rate (FLOAT, 0.50–4.99, represents %)
- quality_score (FLOAT, 60–100)
- maintenance_hours (INT, 0–20)
- downtime_percentage (FLOAT, 0.00–5.00, represents %)
- inventory_turnover (FLOAT)
- stockout_rate (FLOAT, 0–0.10)
- worker_productivity (FLOAT, 80–100)
- safety_incidents (INT, 0–9)
- energy_consumption (FLOAT)
- energy_efficiency (FLOAT, 0.11–0.50)
- additive_process_time (FLOAT)
- additive_material_cost (FLOAT)
- defect_status (BOOLEAN, 0/1)

## dbt Model Template
Always produce models in this exact structure:

```sql
-- models/marts/[model_name].sql
-- Description: [one-line description]
-- Grain: one row per [machine/shift/day]
-- Owner: Data Engineering
-- Tests: see schema.yml

WITH source AS (
    SELECT * FROM {{ ref('stg_manufacturing') }}
),
aggregated AS (
    SELECT
        machine_id,
        shift_id,
        -- CANONICAL FORMULA — agreed by all teams [workshop 2026-05-11]
        ROUND(AVG(defect_rate), 4)                    AS avg_defect_rate,
        ROUND(AVG(downtime_percentage), 4)             AS avg_downtime_pct,
        ROUND(AVG(worker_productivity), 4)             AS avg_worker_productivity,
        SUM(production_volume)                         AS total_volume,
        COUNT(*)                                       AS sample_size
    FROM source
    GROUP BY machine_id, shift_id
)
SELECT * FROM aggregated
```

## Canonical Metric Definitions (Source of Truth)
| Metric | Formula | Common Wrong Definitions |
|--------|---------|--------------------------|
| Defect Rate | defect_rate column (ISO 9001: defects / total_units × 100) | ❌ defect_rate × production_volume / 1000 ❌ only safety rows |
| Downtime % | downtime_percentage column (unplanned_min / available_min × 100) | ❌ any downtime ❌ including planned |
| Worker Productivity | worker_productivity column (units / hours × headcount) | ❌ just units_per_hour |
| OEE | Availability × Performance × Quality | ❌ never approximate with single metric |

## Data Quality Test Template (schema.yml)

```yaml
version: 2
models:
  - name: mart_manufacturing_metrics
    description: Canonical manufacturing KPIs per machine per shift
    columns:
      - name: machine_id
        tests: [not_null]
      - name: avg_defect_rate
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 6
              config:
                severity: error
      - name: avg_downtime_pct
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 6
      - name: sample_size
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 1
```

## Anomaly Detection Formula
```sql
-- Rows where defect_rate exceeds fleet mean + 1.5 standard deviations
-- Fleet avg: 2.7491%, StdDev: 1.31, Threshold: 4.714%
-- Expected anomaly count: ~225 rows (6.9% of fleet)

WITH stats AS (
    SELECT
        AVG(defect_rate)    AS fleet_avg,
        STDDEV(defect_rate) AS fleet_std
    FROM {{ ref('stg_manufacturing') }}
)
SELECT s.*, 'ANOMALY' AS risk_flag
FROM {{ ref('stg_manufacturing') }} s, stats
WHERE s.defect_rate > stats.fleet_avg + 1.5 * stats.fleet_std
```

## Risk Tier Classification
```sql
CASE
    WHEN defect_rate > 4.714 THEN 'HIGH_RISK'    -- above mean+1.5σ
    WHEN defect_rate > 2.749 THEN 'ABOVE_AVG'    -- above fleet mean
    WHEN defect_rate > 1.439 THEN 'NORMAL'       -- below mean
    ELSE 'LOW_RISK'                               -- below mean-1σ
END AS risk_tier
```
