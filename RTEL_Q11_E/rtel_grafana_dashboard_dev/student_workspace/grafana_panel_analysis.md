# Grafana Panel Development & Threshold Analysis Report

## 1. Dashboard Architecture & Panel Overview
Document the 4 core observability panels defined in `grafana_dashboard_panels.json`:
- **Panel 1**:
  - *Title:* P95 API Latency (ms)
  - *Visualization Type:* `timeseries`
  - *PromQL Expression:* `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{env="$env", service=~"$service"}[5m])) by (le, service)) * 1000`
  - *Measured Metric:* 95th percentile HTTP request duration (latency) across selected services over a 5-minute rate window.
- **Panel 2**:
  - *Title:* CPU Utilization (%)
  - *Visualization Type:* `gauge`
  - *PromQL Expression:* `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle", env="$env"}[5m])) * 100)`
  - *Measured Metric:* Active CPU usage percentage per node instance, calculated by subtracting idle CPU time from 100%.
- **Panel 3**:
  - *Title:* HTTP Error Rate (%)
  - *Visualization Type:* `stat`
  - *PromQL Expression:* `sum(rate(http_requests_total{status=~"5..", env="$env", service=~"$service"}[5m])) / sum(rate(http_requests_total{env="$env", service=~"$service"}[5m])) * 100`
  - *Measured Metric:* Percentage of HTTP 5xx server error responses relative to total HTTP requests over a 5-minute window.
- **Panel 4**:
  - *Title:* Order Processing Throughput (req/sec)
  - *Visualization Type:* `bargauge`
  - *PromQL Expression:* `sum by (service) (rate(http_requests_total{env="$env", service=~"$service"}[1m]))`
  - *Measured Metric:* Per-service HTTP request rate (requests per second) calculated over a 1-minute window.

---

## 2. Query Transformations & Aggregations
1. **Explain the purpose and mode of the transformation applied to Panel 1:**
   - *Transformation Name:* `labelsToFields`
   - *Transformation Mode:* `columns`
   - *Operational Function:* Converts Prometheus metric labels (specifically `service`) into separate structured columns in the returned data frame, allowing Grafana to cleanly split and render each service's P95 latency as its own distinct time-series line.

2. **Why is a mathematical multiplier (`* 1000`) applied to the Prometheus query:**
   - *Raw Metric Units:* Seconds (`s`) — following Prometheus naming best practices (`http_request_duration_seconds_bucket`).
   - *Displayed Metric Units:* Milliseconds (`ms`).
   - *Explanation:* Multiplying the raw output by `1000` scales the values from seconds to milliseconds so they match the panel's configured display unit (`"unit": "ms"`) and align with the threshold steps (`500` ms and `1000` ms).

---

## 3. Threshold Color Schemes & Tiers
Complete the threshold evaluation matrix based on the dashboard specification:

| Panel Name | Metric Unit | Nominal (Green) | Warning (Yellow/Orange) | Critical (Red) |
|---|---|---|---|---|
| P95 API Latency | `ms` | `< 500` (`null` to `499`) | `>= 500` and `< 1000` (Yellow) | `>= 1000` (Red) |
| CPU Utilization | `percent` (`%`) | `< 75` (`null` to `74`) | `>= 75` and `< 90` (Orange) | `>= 90` (Red) |
| HTTP Error Rate | `percent` (`%`) | `< 1.0` (`null` to `0.99`) | `>= 1.0` and `< 5.0` (Yellow) | `>= 5.0` (Red) |

---

## 4. Dynamic Template Variables Configuration
Document the template variables configured in the dashboard:

- **Variable 1 (`$env`):**
  - *Type:* `custom`
  - *Configured Options:* `production` (default selected), `staging`, `development`
  - *Filtering Function:* Filters all Prometheus queries by environment label (`env="$env"`), enabling operators to switch the entire dashboard between deployment environments.

- **Variable 2 (`$service`):**
  - *Type:* `query` (sourced from `Prometheus-Prod`)
  - *Query Definition:* `label_values(http_requests_total{env="$env"}, service)`
  - *Multi-Value Enabled? (YES / NO):* **YES** (`"multi": true`)
  - *Include All Option? (YES / NO):* **YES** (`"includeAll": true`, defaulting to `"$__all"`)
  - *Regex Matcher in Queries:* `=~` (e.g., `service=~"$service"`), required to match multiple selected service values or the expanded regex pattern when "All" is selected.

---

## 5. Reliability & Alerting Recommendations
1. **Alert Rule Configuration:**
   - *Recommendation:* Configure **Grafana Unified Alerting rules** directly aligned with the dashboard's critical red thresholds—specifically firing high-severity alerts when **P95 API Latency exceeds 1000 ms** or **HTTP Error Rate exceeds 5.0%** for a sustained `for: 5m` evaluation duration (alongside warning alerts at 500 ms and 1.0%). Decouple alert queries from `$service` multi-value template variables by using static aggregation (`by (service)`) so alerts evaluate every microservice independently and route notifications to PagerDuty or Slack.

2. **Dashboard Auto-Refresh Interval:**
   - *Recommendation:* Increase the dashboard `"refresh"` interval from **`10s`** to **`30s` or `1m`**. Because Panels 1, 2, and 3 compute `rate(...[5m])` over 5-minute windows (and Prometheus scrape intervals are typically 15s–30s), polling every 10 seconds generates redundant query load on `Prometheus-Prod` without yielding meaningful new data points.
