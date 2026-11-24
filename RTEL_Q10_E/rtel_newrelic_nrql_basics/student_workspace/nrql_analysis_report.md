# New Relic NRQL Querying & Performance Audit Report

## 1. Top Slowest Transactions Audit
Identify the top slow database and web transactions recorded across the environment from `nrql_slow_transactions.json`:

- **Transaction 1**:
  - *Name:* `WebTransaction/SpringController/checkout` (`order-service-prod`)
  - *Max Duration (s):* 5.12
  - *Database Duration (s):* 4.60
  - *HTTP Status / Error:* `504` (`Lock wait timeout exceeded; try restarting transaction`)
  - *Primary Bottleneck:* Pessimistic row-level database locking caused by `SELECT * FROM orders WHERE customer_id = ? FOR UPDATE`.

- **Transaction 2**:
  - *Name:* `WebTransaction/Express/chargeCard` (`payment-gateway-prod`)
  - *Max Duration (s):* 6.89
  - *External Duration (s):* 6.55
  - *HTTP Status / Error:* `500` (`GatewayTimeoutException: Payment upstream provider unresponsive`)
  - *Primary Bottleneck:* Upstream third-party API latency on `POST https://api.stripe.com/v1/charges`.

- **Transaction 3**:
  - *Name:* `WebTransaction/FastAPI/searchInventory` (`catalog-service-prod`)
  - *Max Duration (s):* 3.80
  - *Database Duration (s):* 3.55
  - *HTTP Status / Error:* `500` (`QueryExecutionTimeout: Full table scan exceeded 3000ms threshold`)
  - *Primary Bottleneck:* Unindexed wildcard leading-percent query (`SELECT * FROM inventory WHERE title LIKE '%boots%' AND stock > 0`) forcing a full table scan.

---

## 2. NRQL Query Formulations
Document the NRQL queries constructed to extract slow transaction performance data:

### Query 1: Top 10 Slowest Transactions by Average Duration
```sql
SELECT average(duration) AS 'Avg Duration (s)', 
       max(duration) AS 'Max Duration (s)', 
       average(databaseDuration) AS 'Avg DB (s)', 
       average(externalDuration) AS 'Avg External (s)' 
FROM Transaction 
WHERE appName IN ('order-service-prod', 'payment-gateway-prod', 'catalog-service-prod') 
FACET appName, name 
ORDER BY average(duration) DESC 
LIMIT 10 
SINCE 7 days ago
```

### Query 2: Duration Percentile Aggregations (P50, P90, P95, P99)
```sql
SELECT percentile(duration, 50, 90, 95, 99) 
FROM Transaction 
WHERE appName IN ('order-service-prod', 'payment-gateway-prod', 'catalog-service-prod') 
FACET appName, name 
SINCE 7 days ago
```

### Query 3: Time-Series Error Rate Trends
```sql
SELECT percentage(count(*), WHERE error IS true) AS 'Error Rate (%)' 
FROM Transaction 
WHERE appName IN ('order-service-prod', 'payment-gateway-prod', 'catalog-service-prod') 
FACET appName 
TIMESERIES AUTO 
SINCE 7 days ago
```

---

## 3. Latency Percentile Distribution
Based on the aggregated metrics in the dataset, record the percentile thresholds:

| Metric Dimension | P50 (ms) | P90 (ms) | P95 (ms) | P99 (ms) | Operational Assessment |
|---|---|---|---|---|---|
| `order-service-prod` (`checkout`) | 850 | 3200 | 4820 | 5120 | **Critical:** Severe tail latency (P95/P99 > 4.8s) driven by DB row lock contention, causing 504 timeouts and a 3.85% error rate. |
| `catalog-service-prod` (`searchInventory`) | 350 | 1950 | 2950 | 3800 | **Degraded:** P99 breaches the 3000ms query timeout threshold due to full table scans on wildcard searches (2.10% error rate). |
| `payment-gateway-prod` (`chargeCard`) | 650 | 2500 | 3450 | 6890 | **Critical:** Extreme P99 spike (6.89s) caused by external Stripe API hangs without fast-fail timeouts, resulting in the highest error rate (4.50%). |

---

## 4. Root Cause Diagnosis & Remediation Strategy

1. **Database Row Lock Contention (`orders` table):**
   - *Recommendation:* Replace pessimistic locking (`SELECT ... FOR UPDATE`) with **optimistic concurrency control** (using a `version` column or `updated_at` timestamp check on `UPDATE`). Ensure an index exists on `customer_id` so lock scopes do not escalate across rows, and keep transaction blocks strictly limited to the state mutation rather than wrapping external network calls.

2. **External Gateway Latency (`Stripe API`):**
   - *Recommendation:* Implement a **Circuit Breaker pattern** (e.g., via Resilience4j or opossum) alongside strict **connection and read timeouts** (e.g., 2,500ms–3,000ms) on outbound HTTP calls to `api.stripe.com`. Pair this with idempotency keys and exponential backoff retries for transient failures to prevent thread pool exhaustion.

3. **Full Table Scan on Catalog (`LIKE '%...'`):**
   - *Recommendation:* Replace leading-wildcard SQL queries (`LIKE '%shoes%'`) with a dedicated **Full-Text Search index** (e.g., PostgreSQL `GIN`/`tsvector` or MySQL `FULLTEXT`) or offload catalog search to an inverted-index search engine like **Elasticsearch / OpenSearch**. Add a composite index on `(stock, title)` to quickly filter out zero-inventory items.
