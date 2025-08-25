> This file is **for myself** and does not need to be **seen by Irancell**.
> I use this file to be able to determine what should I done in each session.

# Sessions Summary

## Session 1: Payment System Best Practices Assessment

### 1. Core Architectural Patterns

#### **Microservices Design Principles**

| **Pattern**              | **Implementation**                                         | **Benefit**                           |
| ------------------------ | ---------------------------------------------------------- | ------------------------------------- |
| **Domain Separation**    | Payment, Wallet, Ledger, Notification as separate services | Clear boundaries, independent scaling |
| **API-First Design**     | OpenAPI 3.0 specs, async event contracts                   | Contract-driven development, testing  |
| **Stateless Services**   | JWT tokens, external session storage                       | Horizontal scaling, fault tolerance   |
| **Database-per-Service** | Each service owns its data                                 | Data consistency, service autonomy    |

#### **Event-Driven Architecture**

```
Payment Request → Validation → Authorization → Capture → Ledger Update → Notification
     ↓              ↓              ↓           ↓            ↓              ↓
  Event Bus ← → Event Bus ← → Event Bus ← → Event Bus ← → Event Bus ← → Event Bus
```

**Key Components:**

- **Message Bus:** Apache Kafka for event streaming (ordering, replay, durability)
- **Outbox Pattern:** Transactional event publishing with database
- **Saga Pattern:** Distributed transaction orchestration
- **CQRS:** Command/Query separation for read/write optimization

### 2. Resilience & Reliability Patterns

#### **Fault Tolerance Framework**

| **Pattern**           | **Configuration**                              | **Use Case**                |
| --------------------- | ---------------------------------------------- | --------------------------- |
| **Circuit Breaker**   | 50% failure rate, 10s timeout, 30s half-open   | PSP integration protection  |
| **Bulkhead**          | Separate thread pools per PSP                  | Failure isolation           |
| **Retry with Jitter** | Exponential backoff: 2^n + random(0,1000)ms    | Avoid thundering herd       |
| **Timeout Cascade**   | API Gateway: 30s, Service: 25s, PSP: 20s       | Prevent resource exhaustion |
| **Rate Limiting**     | Token bucket: 1000/min per client, 100/s burst | DDoS protection             |

#### **High Availability Design**

- **Multi-PSP Routing:** Primary/secondary with automatic failover
- **Geographic Distribution:** Active-passive across data centers
- **Health Checks:** Deep health endpoints (database, external APIs)
- **Graceful Degradation:** Read-only mode during PSP outages

### 3. Security Best Practices

#### **PCI-DSS Compliance Framework** _(Level 1 Requirements)_

| **Requirement**   | **Implementation**          | **Technical Control**                   |
| ----------------- | --------------------------- | --------------------------------------- |
| **PCI-DSS 3.4**   | Encrypt PAN transmission    | TLS 1.3, mTLS for PSP APIs              |
| **PCI-DSS 3.5.1** | Tokenization                | Format-preserving tokens, HSM storage   |
| **PCI-DSS 8.2**   | Multi-factor authentication | TOTP for admin access                   |
| **PCI-DSS 10.1**  | Audit trail                 | OpenTelemetry traces, tamper-proof logs |
| **PCI-DSS 11.4**  | Network security            | WAF, IDS/IPS, network segmentation      |

#### **Zero-Trust Security Model**

- **Authentication:** OAuth2/OIDC with short-lived tokens
- **Authorization:** RBAC with least-privilege principle
- **Network:** mTLS between all services
- **Data:** Encryption at rest (AES-256) and in transit (TLS 1.3)
- **Secrets Management:** HashiCorp Vault or cloud KMS

#### **Data Protection Strategy**

```
PAN Data Flow: Customer → Tokenization → Encrypted Storage → HSM
                   ↓
Log Data Flow: Application → PII Scrubbing → Encrypted Logs → Retention Policy
```

### 4. Performance & Scaling Patterns

#### **Capacity Planning for 400 Concurrent Transactions**

| **Component**       | **Scaling Strategy**                       | **Target Performance**  |
| ------------------- | ------------------------------------------ | ----------------------- |
| **API Gateway**     | Horizontal pods, CPU-based autoscaling     | 1000+ req/s, p95 < 50ms |
| **Payment Service** | Stateless replicas, queue-based processing | 500 req/s, p95 < 200ms  |
| **Database**        | CockroachDB cluster, read replicas         | 10K IOPS, p95 < 10ms    |
| **Cache Layer**     | Redis Cluster, consistent hashing          | 100K ops/s, p95 < 1ms   |
| **Message Bus**     | Kafka partitioning, consumer groups        | 50K msg/s throughput    |

#### **Caching Strategy**

- **L1 (Application):** In-memory cache for configuration, feature flags
- **L2 (Distributed):** Redis for user sessions, rate limit counters
- **L3 (Database):** Query result caching, connection pooling
- **CDN:** Static content, API documentation

### 5. Integration & API Patterns

#### **PSP Integration Best Practices**

| **Aspect**               | **Standard Approach**             | **Irancell-Specific**                 |
| ------------------------ | --------------------------------- | ------------------------------------- |
| **Authentication**       | mTLS + HMAC signatures            | IP whitelisting + certificate pinning |
| **Request Format**       | JSON/REST with idempotency keys   | Standardized across PSPs              |
| **Response Handling**    | Async webhooks + polling fallback | 5-minute timeout, 3 retries           |
| **Error Classification** | Hard/soft failures, retry logic   | Circuit breaker per PSP               |

#### **Webhook Implementation**

```
Incoming Webhook → Signature Validation → Store & ACK → Async Processing → Business Logic
                        ↓                      ↓               ↓              ↓
                   IP Whitelist Check → Idempotency Check → DLQ for Failures → Event Publishing
```

**Configuration:**

- **Signature:** HMAC-SHA256 with rolling secrets
- **Retry Policy:** Exponential backoff (1s, 2s, 4s, 8s, 16s)
- **Timeout:** 30 seconds per webhook call
- **DLQ:** Manual review for failed webhook processing

### 6. Observability & Monitoring Patterns

#### **OpenTelemetry Implementation**

| **Signal Type** | **Collection Method**            | **Storage**                  | **Alerting**        |
| --------------- | -------------------------------- | ---------------------------- | ------------------- |
| **Metrics**     | Prometheus client libraries      | Prometheus + VictoriaMetrics | AlertManager        |
| **Logs**        | Structured JSON, correlation IDs | Loki                         | LogAlert rules      |
| **Traces**      | OTEL auto-instrumentation        | Tempo                        | Trace-based SLIs    |
| **Events**      | Business events, audit trail     | Custom event store           | Event-driven alerts |

#### **Key Metrics Framework**

```
Business Metrics: Transaction Success Rate, Revenue per Hour, Average Order Value
Technical Metrics: Request Latency, Error Rate, Throughput, Queue Depth
Infrastructure: CPU/Memory Usage, Database Connections, Network I/O
Security: Failed Auth Attempts, Suspicious Patterns, Compliance Violations
```

### 7. Compliance & Regulatory Framework

#### **Multi-Region Compliance Matrix**

| **Standard**       | **Scope**            | **Key Requirements**                    | **Implementation**               |
| ------------------ | -------------------- | --------------------------------------- | -------------------------------- |
| **PCI-DSS v4.0**   | Card data handling   | Encryption, access controls, monitoring | HSM, tokenization, audit logs    |
| **GDPR**           | EU customer data     | Right to erasure, data portability      | Data anonymization, consent mgmt |
| **PSD2**           | EU payment services  | Strong authentication, API standards    | 3DS2, secure APIs                |
| **Iran Data Laws** | Local data residency | Data sovereignty                        | In-country database replicas     |

## Session 3 — SLOs, OTEL + LGTM Dashboards, and Alerting (PromQL)

### 1) SLOs → SLIs (what we’ll measure)

| Service      | SLO                                                 | SLI (Prom metrics)                                                                                                                            | Notes                                                                        |
| ------------ | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Payments API | **Availability 99.9% monthly**                      | `1 - (errors / total)` from `http_server_request_duration_seconds_count` + custom counters (`app_exceptions_total`, `request_timeouts_total`) | Errors = 5xx + timeouts + app exceptions (no declines)                       |
| Payments API | **Latency**: p50 ≤ 150ms, p95 ≤ 500ms, p99 ≤ 1200ms | `histogram_quantile()` on `http_server_request_duration_seconds_bucket`                                                                       | Excludes 3DS human steps                                                     |
| Platform     | **Error-rate alert** at **>4% for 5m**              | Error-rate PromQL (below)                                                                                                                     | Primary alert has **no traffic guard** (per instruction); advisory has guard |
| Adapters     | PSP call p95 ≤ 1200ms; error-rate < 2%              | `http_client_request_duration_seconds_bucket` by `peer=psp`                                                                                   | Use as **routing hints**                                                     |
| Webhooks     | End-to-end lag p95 ≤ 30s                            | Kafka consumer lag on `webhooks.in`                                                                                                           | Also alert on DLQ backlog                                                    |
| Kafka        | Consumer lag steady < 1,000 msgs/topic              | `kafka_consumergroup_lag` or equivalent                                                                                                       | Exporter depends on your Kafka exporter                                      |
| DB           | Commit latency p95 ≤ 50ms                           | CRDB/PG exporters (`sql_exec_latency_bucket`, `pg_stat*`)                                                                                     | Plus deadlock/error counters                                                 |
| Redis        | GET/SET p95 ≤ 5ms; mem < 80%                        | Redis exporter                                                                                                                                | Idempotency cache health                                                     |
| Gateway      | 429 rate < 1%                                       | `http_server_request_duration_seconds_count{status=~"429"}`                                                                                   | Detect rate-limit misconfigs                                                 |

**OTEL semantic conventions to emit**

- **HTTP server**: `http_server_request_duration_seconds_bucket` with labels: `http_response_status_code`, `route`, `method`, `job="payments"`.
- **HTTP client (adapters)**: `http_client_request_duration_seconds_bucket` with labels: `peer` (psp id), `http_response_status_code`.
- **Custom counters** (you must expose): `app_exceptions_total`, `request_timeouts_total`, `circuit_breaker_state{psp}`, `webhook_store_failures_total`.

---

### 2) Dashboards (Grafana panel blueprint)

**A. Executive SLO Overview**

- **Error rate (5m)**: primary vs target line (4%).
- **Latency p50/p95/p99** (Payments).
- **Traffic (RPS)**.
- **SLO burn** (see §4).

**B. Payments API**

- RPS: `sum(rate(http_server_request_duration_seconds_count{job="payments"}[5m]))`
- Error rate % (primary query below).
- Latency p95: `histogram_quantile(0.95, sum by (le) (rate(http_server_request_duration_seconds_bucket{job="payments"}[5m])))`
- Top routes by p95.
- Exceptions: `rate(app_exceptions_total{job="payments"}[5m])`

**C. Gateway**

- 2xx/4xx/5xx stacked.
- 429 percentage.
- Idempotency hits vs misses (from gateway metric `idempotency_cache_hits_total`).

**D. Adapters (per PSP)**

- p95 latency: `histogram_quantile(0.95, sum by (le,peer) (rate(http_client_request_duration_seconds_bucket{job="adapters"}[5m])))`
- Error % per PSP.
- Circuit breaker state over time (`circuit_breaker_state{psp}`).

**E. Webhooks & Reconciliation**

- Inbound rate (`rate(webhooks_in_total[5m])`).
- **Consumer lag** per topic.
- **DLQ backlog**.
- Time from receive → processed (derive from timestamps or trace spans).

**F. Kafka**

- Broker health, partition under-replication.
- Topic bytes in/out, consumer lag.

**G. DB / Redis**

- DB exec latency histograms, errors, connection pool saturation.
- Redis ops/s, mem %, evictions, latency.

**H. Traces & Logs**

- **Tempo exemplars** attached to latency histograms.
- Error logs by service with correlation to trace ids.

---

### 3) `/observability-alerts.md` (drop-in content)

> Create a file named exactly `/observability-alerts.md` with the content below.

````md
## Observability Alerts (Prometheus + Alertmanager)

### 0) Recording rules (helpers)

```promql
# Payments totals (5m window)
sum:payments_http_total_5m =
  sum(increase(http_server_request_duration_seconds_count{job="payments"}[5m]))

# Payments 5xx (5m)
sum:payments_http_5xx_5m =
  sum(increase(http_server_request_duration_seconds_count{job="payments", http_response_status_code=~"5.."}[5m]))

# App-level exceptions (5m)
sum:payments_app_exceptions_5m =
  sum(increase(app_exceptions_total{job="payments"}[5m]))

# Request timeouts (service or adapter surfaced) (5m)
sum:payments_timeouts_5m =
  sum(increase(request_timeouts_total{job="payments"}[5m]))

# Technical errors (5m)
sum:payments_errors_5m = sum:payments_http_5xx_5m + sum:payments_app_exceptions_5m + sum:payments_timeouts_5m

# Error-rate %
ratio:payments_error_rate_5m = (sum:payments_errors_5m / sum:payments_http_total_5m) * 100
```
````

---

### 1) Primary alert — Error-rate > 4% for 5m (NO traffic guard)

```yaml
groups:
  - name: payments-api
    rules:
      - alert: PaymentsHighErrorRatePrimary
        expr: ratio:payments_error_rate_5m > 4
        for: 5m
        labels:
          severity: SEV-1
          service: payments
          runbook_url: "TBD/high-error-rate"
        annotations:
          summary: "Payments API technical error-rate > 4% (5m)"
          description: 'Error-rate (5m) is {{ $value | printf "%.2f" }}% (>4%). Investigate adapters, DB, recent deploys.'
```

> This matches Irancell’s request: **no minimum traffic condition**.

---

### 2) Advisory alert — Error-rate > 4% with traffic guard (reduce noise)

```yaml
- name: payments-api-advisory
  rules:
    - alert: PaymentsHighErrorRateAdvisory
      expr: ratio:payments_error_rate_5m > 4
        and sum:payments_http_total_5m > 500 # ~100 rps for 5m
      for: 5m
      labels:
        severity: SEV-2
        service: payments
        runbook_url: "TBD/high-error-rate"
      annotations:
        summary: "Payments error-rate > 4% (5m) with sufficient traffic"
        description: "Traffic gate active. Total req (5m): {{ $labels.instance }} passes threshold."
```

---

### 3) Latency SLO breaches (Payments)

```yaml
- name: payments-latency
  rules:
    - alert: PaymentsLatencyP95High
      expr: |
        histogram_quantile(0.95,
          sum by (le) (rate(http_server_request_duration_seconds_bucket{job="payments"}[5m]))
        ) > 0.5   # seconds
      for: 10m
      labels:
        severity: SEV-2
        runbook_url: "TBD/p95-latency-spike"
      annotations:
        summary: "Payments p95 latency > 500ms (10m)"
        description: "Check DB, adapters, or recent deploy."
    - alert: PaymentsLatencyP99High
      expr: |
        histogram_quantile(0.99,
          sum by (le) (rate(http_server_request_duration_seconds_bucket{job="payments"}[5m]))
        ) > 1.2
      for: 10m
      labels:
        severity: SEV-2
        runbook_url: "TBD/p99-latency-spike"
      annotations:
        summary: "Payments p99 latency > 1200ms (10m)"
        description: "Investigate outliers; link exemplars to Tempo."
```

---

### 4) Adapter health & circuit breakers

```yaml
- name: adapters
  rules:
    - alert: PSPAdapterErrorRateHigh
      expr: |
        ( sum(increase(http_client_request_duration_seconds_count{job="adapters", http_response_status_code=~"5.."}[5m])) /
          sum(increase(http_client_request_duration_seconds_count{job="adapters"}[5m])) ) * 100 > 2
      for: 10m
      labels:
        severity: SEV-2
        runbook_url: "TBD/gateway-outage"
      annotations:
        summary: "Adapter technical error-rate > 2% (10m)"
    - alert: PSPCircuitBreakerOpen
      expr: max_over_time(circuit_breaker_state{state="open"}[1m]) == 1
      for: 2m
      labels:
        severity: SEV-1
        runbook_url: "TBD/gateway-outage"
      annotations:
        summary: "Circuit breaker OPEN for a PSP"
        description: "Failover/routing engaged or needed. Check adapter metrics."
```

_(If your breaker metric is numeric, e.g., `circuit_breaker_state{psp}` with 2=open, adjust expr to `== 2`.)_

---

### 5) Webhooks & Kafka

```yaml
- name: webhooks-kafka
  rules:
    - alert: WebhookDLQBacklogHigh
      expr: |
        sum(kafka_topic_partition_current_offset{topic=~".*webhooks.*-dlq"}) -
        sum(kafka_consumergroup_current_offset{topic=~".*webhooks.*-dlq"}) > 1000
      for: 10m
      labels:
        severity: SEV-2
        runbook_url: "TBD/queue-backlog"
      annotations:
        summary: "Webhook DLQ backlog > 1k messages (10m)"
    - alert: WebhookConsumerLag
      expr: sum(kafka_consumergroup_lag{topic=~".*webhooks.*"}) > 5000
      for: 10m
      labels:
        severity: SEV-2
        runbook_url: "TBD/queue-backlog"
      annotations:
        summary: "Webhook consumer lag > 5k"
```

---

### 6) DB / Redis

```yaml
- name: database
  rules:
    - alert: DBCommitLatencyHigh
      expr: histogram_quantile(0.95, sum by (le) (rate(sql_exec_latency_bucket{op="commit"}[5m]))) > 0.05
      for: 10m
      labels: { severity: SEV-2, runbook_url: "TBD/db-latency" }
      annotations:
        summary: "DB commit p95 > 50ms"
    - alert: DBErrors
      expr: increase(db_errors_total[5m]) > 0
      for: 5m
      labels: { severity: SEV-1, runbook_url: "TBD/db-errors" }
      annotations:
        summary: "DB errors detected"
- name: redis
  rules:
    - alert: RedisHighLatency
      expr: histogram_quantile(0.95, sum by (le) (rate(redis_command_duration_seconds_bucket[5m]))) > 0.005
      for: 10m
      labels: { severity: SEV-2, runbook_url: "TBD/redis-latency" }
      annotations:
        summary: "Redis p95 > 5ms"
    - alert: RedisHighMemory
      expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
      for: 15m
      labels: { severity: SEV-2, runbook_url: "TBD/redis-memory" }
      annotations:
        summary: "Redis memory > 80%"
```

---

### 7) Gateway saturation & 429s

```yaml
- name: gateway
  rules:
    - alert: GatewayHigh429Rate
      expr: |
        ( sum(increase(http_server_request_duration_seconds_count{job="gateway", http_response_status_code="429"}[5m])) /
          sum(increase(http_server_request_duration_seconds_count{job="gateway"}[5m])) ) * 100 > 1
      for: 15m
      labels: { severity: SEV-3, runbook_url: "TBD/rate-limit-tuning" }
      annotations:
        summary: "429 rate > 1% (15m)"
```

---

### 8) Certificates, Vault & HSM (sanity)

```yaml
- name: security
  rules:
    - alert: TLSCertExpiringSoon
      expr: min(cert_expiry_seconds{job=~"gateway|payments"}) < 1209600 # 14 days
      for: 1h
      labels: { severity: SEV-3, runbook_url: "TBD/cert-rotation" }
      annotations:
        summary: "TLS cert expiring < 14 days"
    - alert: VaultUnsealedOrSealed
      expr: vault_sealed == 1 or vault_active == 0
      for: 5m
      labels: { severity: SEV-1, runbook_url: "TBD/vault" }
      annotations:
        summary: "Vault sealed or inactive"
```

````

---

### 4) SLO burn alerts (optional, advanced)

If you want budget-based paging (SRE style) for **99.9% monthly availability**, error-budget is **43.2 min**/month. A simple two-window burn:

- **Fast burn** (2h) and **slow burn** (24h) windows:
```promql
burn_fast = (sum(rate(errors[2h])) / sum(rate(total[2h]))) / 0.001
burn_slow = (sum(rate(errors[24h])) / sum(rate(total[24h]))) / 0.001
````

Alert if `burn_fast > 14` **or** `burn_slow > 1`, etc. (Tune to taste.)

---

### 5) OTEL, Logs & Redaction (PCI-aligned)

**Emit IDs**

- Inject `trace_id`, `span_id`, `correlation_id` into logs; **logfmt/JSON** only.

**Redaction (OTEL Collector `transform` processor)**

- Drop/replace sensitive attributes before they hit Loki:

```yaml
processors:
  transform:
    error_mode: ignore
    log_statements:
    - context: log
      statements:
      - replace_all(pattern: "(?P<PAN>\\b\\d{13,19}\\b)", replacement: "[REDACTED_PAN]")
      - replace_all(pattern: "(?i)authorization: Bearer [^\\s]+", replacement: "authorization: [REDACTED]")
      - delete_key(key: "card_number")
      - delete_key(key: "cvv")
exporters:
  loki: ...
```

_(Regex is illustrative; tune to your formats. Enforce denylist at the gateway too.)_

**Exemplars**

- Enable exemplars in Prometheus and Grafana to link **p95 buckets → Tempo traces**.

---

### 6) Test hooks (k6 profile → alert smoke)

- Ramp: `0 → 400 VUs` in 5 min; hold 15 min @ target RPS **≈300**; random 1–3s think time.
- Inject chaos: spike adapter latency to force breaker; drop 1% packets to simulate timeouts.
- Expected: **Primary 4% alert** triggers only when we push real 5xx/timeout spikes; advisory triggers during guarded loads.

---

### 7) Runbook placeholders (link these in alerts)

- `TBD/high-error-rate`: check deploy diff, adapters panel, DB errors, breaker state, rollback if needed.
- `TBD/p95-latency-spike`: inspect traces (slow spans), DB locks, GC pauses, hot routes.
- `TBD/gateway-outage`: WAF/LB health, rate limits, PSP reachability, switch PSP flags.
- `TBD/queue-backlog`: DLQ size, consumer restarts, poison message pattern, replay tool.
- `TBD/db-latency`: hot ranges/partitions, contention, long transactions, connection pool.

---

### 8) Alert routing (Alertmanager sketch)

```yaml
route:
  receiver: pagerduty
  group_by: [service]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 2h
  routes:
    - matchers: [severity="SEV-1"]
      receiver: pagerduty
    - matchers: [severity="SEV-2"]
      receiver: oncall-email
    - matchers: [severity="SEV-3"]
      receiver: slack

receivers:
  - name: pagerduty
    pagerduty_configs:
      - routing_key: ${PD_KEY}
  - name: oncall-email
    email_configs:
      - to: "noc@example.com"
  - name: slack
    slack_configs:
      - channel: "#payments-alerts"
```
