# mozilla_telemetry_derived Dataset Guide

## Overview

The `mozilla_telemetry_derived` dataset contains derived tables from Mozilla's Firefox telemetry pipeline. In this BigQuery demo project (`bigquery-demo-461021`), **7 tables** are available as publicly exported copies. These correspond to a subset of the ~258 table definitions in the repo under `sql/moz-fx-data-shared-prod/telemetry_derived/`. Only tables with `public_bigquery: true` in their `metadata.yaml` are exported here.

The full production dataset lives in `moz-fx-data-shared-prod.telemetry_derived` and is not directly queryable from this project.

## Dataset-to-Repo Mapping

Each table in BigQuery maps to a directory in the repo:

```
sql/moz-fx-data-shared-prod/telemetry_derived/<table_name>/
  query.sql        # Main SQL query (Jinja2 template)
  metadata.yaml    # Owners, description, scheduling (DAG), labels
  schema.yaml      # Output schema definition
  checks.sql       # Data quality checks (optional)
  bigconfig.yml    # External monitoring config (rare)
```

Tables with `public_bigquery: true` and `public_json: true` labels in `metadata.yaml` are the ones exported to the public demo project.

## Available Tables

### 1. `ssl_ratios_v1` (Active, Daily since 2016-11-02)

**Purpose:** Percentages of Firefox page loads conducted over SSL, broken down by OS and country.

| Column | Type | Description |
|--------|------|-------------|
| `submission_date` | DATE | Partition key. Data available from 2016-11-02 to present. |
| `os` | STRING | Operating system: `Windows_NT`, `Darwin`, or `Linux`. |
| `country` | STRING | ISO country code (249 distinct values). |
| `non_ssl_loads` | INT64 | Count of non-SSL page loads. |
| `ssl_loads` | INT64 | Count of SSL page loads. |
| `reporting_ratio` | FLOAT64 | Ratio of clients reporting this metric. |

- **Partition filter required:** Yes (on `submission_date`)
- **Unique key:** `(submission_date, os, country)`
- **DAG:** `bqetl_ssl_ratios`
- **Owner:** `chutten@mozilla.com`
- **Query style:** Uses `metrics.calculate` Jinja framework, filtered to `sample_id = 42`, release channel only
- **Data quality:** Has `checks.sql` (not-null, min-row-count, uniqueness, range checks) and `bigconfig.yml` monitoring

### 2. `doh_adoption_rate_v1` (Active, Daily)

**Purpose:** Tracks DNS-over-HTTPS (DOH) adoption in Firefox through 9 distinct metrics.

| Column | Type | Description |
|--------|------|-------------|
| `submission_date` | DATE | Partition key. |
| `metric` | STRING | One of 9 DOH metrics (e.g., `doh.state_enabled`, `networking.doh_heuristics_result`). |
| `key` | STRING | Key/dimension associated with the metric value. |
| `value` | STRING | The metric value or extra field. |
| `country_code` | STRING | ISO country code; countries <5000 clients bucketed to `OTHER`. |
| `total_client_count` | INT64 | Sum of distinct client counts. |

- **Partition filter required:** Yes (on `submission_date`)
- **DAG:** `bqetl_ech_adoption_rate`
- **Owner:** `ascholtz@mozilla.com`
- **Sources:** `firefox_desktop.events_stream`, `firefox_desktop.metrics`
- **Country bucketing:** Countries with fewer than 5000 distinct clients are bucketed to `'OTHER'` (currently 10 distinct country codes)

**Available metrics:**
- `doh.evaluate_v2_heuristics` - DOH heuristics evaluation events
- `doh.state_disabled` / `doh.state_enabled` - DOH state changes
- `doh.state_manually_disabled` / `doh.state_policy_disabled` - Manual/policy disables
- `networking.doh_heuristics_attempts` / `networking.doh_heuristics_pass_count` / `networking.doh_heuristics_result` - Heuristics counters
- `security.doh.settings.provider_choice_value` - Selected DOH provider

### 3. `ech_adoption_rate_v1` (Active, Daily, ~1.1 GB)

**Purpose:** Measures Encrypted Client Hello (ECH) adoption in Firefox through SSL handshake metrics.

| Column | Type | Description |
|--------|------|-------------|
| `submission_date` | DATE | Partition key. |
| `metric` | STRING | One of 4 ECH metrics. |
| `label` | STRING | Sub-label (e.g., `GREASE`, `REAL` for `http3_ech_outcome`). |
| `key` | STRING | Histogram bucket key. |
| `handshakes` | INT64 | Count of SSL handshake events. |
| `total_client_count` | INT64 | Distinct client count. |
| `country_code` | STRING | ISO code; countries <5000 handshakes/clients bucketed to `OTHER`. |

- **Partition filter required:** Yes (on `submission_date`)
- **Largest table** in the dataset (~18M rows, ~1.1 GB)
- **DAG:** `bqetl_ech_adoption_rate`
- **Owner:** `efilho@mozilla.com`
- **Source:** `firefox_desktop.metrics`

**Available metrics:**
- `http3_ech_outcome` - HTTP/3 ECH outcomes (labeled: GREASE, REAL)
- `ssl_handshake_privacy` - SSL handshake privacy level histogram
- `ssl_handshake_result_ech` - ECH handshake results
- `ssl_handshake_result_ech_grease` - ECH GREASE handshake results

### 4. `client_probe_processes_v1` (Active, Unpartitioned)

**Purpose:** Maps telemetry metric names to the browser processes where they are observed. Used by the Glean Dictionary search service.

| Column | Type | Description |
|--------|------|-------------|
| `metric` | STRING | Telemetry metric name (e.g., `places_bookmarks_count`). |
| `processes` | ARRAY\<STRING\> | Processes where the metric is observed (e.g., `["parent", "content", "gpu"]`). |

- **No partitioning** - small reference table (1,261 rows)
- **DAG:** `bqetl_main_summary`
- **Owner:** `wlachance@mozilla.com`
- **Source:** `telemetry.client_probe_counts`

### 5. `deviations_v1` (Deprecated)

**Purpose:** Historical anomaly detection data tracking deviations in desktop DAU and active hours across geographies.

| Column | Type | Description |
|--------|------|-------------|
| `date` | DATE | Partition key. Range: 2020-01-31 to 2020-12-31. |
| `metric` | STRING | Either `desktop_dau` or `mean_active_hours_per_client`. |
| `deviation` | FLOAT64 | Deviation score (positive = above expected, negative = below). |
| `ci_deviation` | FLOAT64 | Confidence interval of the deviation. |
| `geography` | STRING | Country or sub-region code (742 distinct values, e.g., `US`, `FR:IDF:75:Paris`). |

- **Status:** Deprecated (deletion date: 2024-05-31)
- **Static data** - no query.sql in the repo; data covers only 2020
- **Geography granularity** can go down to city level (e.g., `FR:IDF:75:Paris`)

### 6. `italy_covid19_outage_v1` (Active, Static, Unpartitioned)

**Purpose:** Aggregated Firefox Desktop data measuring Italy's internet quality during the COVID-19 pandemic (Jan-Mar 2020).

| Column | Type | Description |
|--------|------|-------------|
| `date` | DATE | Date of measurement (2020-01-01 to 2020-03-31). |
| `proportion_undefined` | FLOAT64 | Proportion of undefined network failures. |
| `proportion_timeout` | FLOAT64 | Proportion of timeout failures. |
| `proportion_abort` | FLOAT64 | Proportion of aborted connections. |
| `proportion_unreachable` | FLOAT64 | Proportion of unreachable errors. |
| `proportion_terminated` | FLOAT64 | Proportion of terminated connections. |
| `proportion_channel_open` | FLOAT64 | Proportion of channel open errors. |
| `avg_dns_success_time` | FLOAT64 | Average successful DNS resolution time (ms). |
| `avg_dns_failure_time` | FLOAT64 | Average failed DNS resolution time (ms). |
| `count_dns_failure` | FLOAT64 | Average count of DNS failures per client. |
| `avg_tls_handshake_time` | FLOAT64 | Average TLS handshake time (ms). May be NULL for some days. |

- **Only 91 rows** (one per day)
- **No DAG** - static dataset, not incrementally updated
- **Owner:** `aplacitelli@mozilla.com`
- **Sources:** `telemetry.clients_daily`, `telemetry.health`, `telemetry.main`

### 7. `regrets_reporter_study_v1` (Deprecated)

**Purpose:** YouTube Regrets Reporter study data capturing user feedback on video recommendations.

| Column | Type | Description |
|--------|------|-------------|
| `rejected_video_title` | STRING | Title of the rejected video. |
| `rejected_video_channel` | STRING | Channel of the rejected video. |
| `recommendation_title` | STRING | Title of the recommended video. |
| `recommendation_channel` | STRING | Channel of the recommended video. |
| `rejected_video_id` | STRING | YouTube video ID of rejected video. |
| `recommendation_id` | STRING | YouTube video ID of recommendation. |
| `feedback_type` | STRING | One of: `dislike`, `dont_recommend`, `not_interested`, `remove_from_history`. |
| `rejection_first_time` | STRING | Timestamp of first rejection. |
| `recommendation_last_time` | STRING | Timestamp of last recommendation. |
| `rejected_image` | STRING | Thumbnail URL of rejected video. |
| `recommendation_image` | STRING | Thumbnail URL of recommendation. |

- **Status:** Deprecated (deletion date: 2024-05-31, Bug 1788672)
- **Only 37 rows** across 31 distinct video channels
- **No query.sql** in the repo; data is static

## Query Patterns

### Partitioned Tables (ssl_ratios_v1, doh_adoption_rate_v1, ech_adoption_rate_v1, deviations_v1)

Always filter on the partition column first:

```sql
SELECT submission_date, os, country, ssl_loads, non_ssl_loads, reporting_ratio
FROM `bigquery-demo-461021.mozilla_telemetry_derived.ssl_ratios_v1`
WHERE submission_date >= '2025-01-01'
  AND submission_date < '2025-02-01'
  AND os = 'Windows_NT'
ORDER BY ssl_loads DESC
LIMIT 10;
```

### DOH/ECH Metric Filtering

Both tables store multiple metrics in a single table. Always filter by `metric`:

```sql
SELECT submission_date, country_code, total_client_count
FROM `bigquery-demo-461021.mozilla_telemetry_derived.doh_adoption_rate_v1`
WHERE submission_date = '2025-06-01'
  AND metric = 'doh.state_enabled'
ORDER BY total_client_count DESC;
```

### Unpartitioned Tables (client_probe_processes_v1, italy_covid19_outage_v1, regrets_reporter_study_v1)

These are small enough to scan fully without partition filters:

```sql
SELECT metric, processes
FROM `bigquery-demo-461021.mozilla_telemetry_derived.client_probe_processes_v1`
WHERE 'content' IN UNNEST(processes);
```

## Relationship to Broader telemetry_derived

The repo defines ~258 tables in `telemetry_derived`, organized into functional groups:

| Group | ~Count | Description |
|-------|--------|-------------|
| Uninstalls analysis | 28 | Uninstall metrics by various dimensions (country, OS, addon, etc.) |
| Firefox Health Indicators | 24 | Health dashboard metrics (bookmarks, video, searches, etc.) |
| Clients lifecycle | 20+ | `clients_daily`, `clients_first_seen`, `clients_last_seen` family |
| Experiments | 15 | Experiment enrollment, population, search aggregates |
| Newtab | 12 | New tab page interactions and Merino recommendations |
| Active users / retention | 10+ | DAU/WAU/MAU aggregates and retention cohorts |
| GLAM | 7 | Histogram/scalar aggregates for the GLAM frontend |
| Crashes | 6 | Crash counts, signatures, symbolication |

Only tables labeled `public_bigquery: true` are exported to this demo project. The remaining tables require access to `moz-fx-data-shared-prod`.
