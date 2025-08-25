# MTN Irancell Payment Gateway - Observability & Alerting Configuration

## Table of Contents

- [SLO Framework](#slo-framework)
- [Alert Rules Configuration](#alert-rules-configuration)
- [LGTM Stack Setup](#lgtm-stack-setup)
- [OpenTelemetry Configuration](#opentelemetry-configuration)
- [Dashboard Specifications](#dashboard-specifications)
- [Escalation & Runbooks](#escalation--runbooks)
- [Log Aggregation Strategy](#log-aggregation-strategy)

---

## SLO Framework

### Service Level Objectives

| **SLI**                 | **SLO Target**     | **Error Budget**    | **Measurement Window** | **Alert Threshold**  |
| ----------------------- | ------------------ | ------------------- | ---------------------- | -------------------- |
| **API Availability**    | 99.9% uptime       | 43.2 min/month      | 30-day rolling         | <99.5% for 5min      |
| **Payment Latency**     | p95 ≤ 500ms        | 5% slow requests    | 1-hour rolling         | p95 >750ms for 10min |
| **Transaction Success** | 96% success rate   | 4% error budget     | 5-minute rolling       | >4% errors for 5min  |
| **PSP Integration**     | 99.5% availability | 21.6 min/month      | 24-hour rolling        | Circuit breaker open |
| **Data Durability**     | 99.999%            | 0 lost transactions | Continuous             | Any transaction loss |

### Error Budget Calculations

```promql
# API Availability SLI (target: 99.9%)
api_availability_sli =
  sum(rate(http_requests_total{code!~"5.."}[30d])) /
  sum(rate(http_requests_total[30d]))

# Payment Success Rate SLI (target: 96%)
payment_success_sli =
  sum(rate(payment_transactions_total{status="completed"}[5m])) /
  sum(rate(payment_transactions_total[5m]))

# Latency SLI (target: p95 ≤ 500ms)
latency_sli =
  histogram_quantile(0.95,
    rate(http_request_duration_seconds_bucket{service="payment-service"}[5m])
  ) <= 0.5

# Error Budget Remaining (monthly)
error_budget_remaining =
  1 - (
    (1 - api_availability_sli) / (1 - 0.999)
  )
```

---

## Alert Rules Configuration

### Primary Alert Rules (prometheus-alerts.yaml)

```yaml
groups:
  - name: payment_critical_alerts
    rules:
      # PRIMARY ALERT: 4% Error Rate Threshold
      - alert: PaymentErrorRateHigh
        expr: |
          (
            rate(http_requests_total{service=~"payment-service|wallet-service",code=~"4..|5.."}[5m]) +
            rate(http_timeouts_total{service=~"payment-service|wallet-service"}[5m]) +
            rate(psp_errors_total{service=~"payment-service"}[5m]) +
            rate(application_exceptions_total{service=~"payment-service|wallet-service"}[5m])
          ) / rate(http_requests_total{service=~"payment-service|wallet-service"}[5m]) > 0.04
        for: 5m
        labels:
          severity: critical
          service: payment-gateway
          team: fintech
          runbook: https://runbooks.irancell.com/payment-error-rate
        annotations:
          summary: "Payment error rate exceeded 4% threshold"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes. Immediate investigation required."
          dashboard: "https://grafana.irancell.com/d/payment-ops"

      # PSP Circuit Breaker Alert
      - alert: PSPCircuitBreakerOpen
        expr: |
          circuit_breaker_state{service="psp-adapter"} == 1
        for: 1m
        labels:
          severity: critical
          service: psp-integration
          team: fintech
          runbook: https://runbooks.irancell.com/psp-outage
        annotations:
          summary: "PSP {{ $labels.psp_name }} circuit breaker is open"
          description: "Circuit breaker has been open for {{ $labels.psp_name }} for over 1 minute. Auto-failover should be active."

      # Database Unavailability
      - alert: DatabaseConnectionFailure
        expr: |
          up{job="cockroachdb"} == 0
        for: 30s
        labels:
          severity: critical
          service: database
          team: fintech
          runbook: https://runbooks.irancell.com/database-outage
        annotations:
          summary: "CockroachDB cluster node down"
          description: "Database node {{ $labels.instance }} is unreachable"

  - name: payment_warning_alerts
    rules:
      # API Latency Warning
      - alert: PaymentLatencyHigh
        expr: |
          histogram_quantile(0.95, 
            rate(http_request_duration_seconds_bucket{service="payment-service"}[5m])
          ) > 0.75
        for: 10m
        labels:
          severity: warning
          service: payment-gateway
          team: fintech
          runbook: https://runbooks.irancell.com/payment-latency
        annotations:
          summary: "Payment API latency p95 exceeded 750ms"
          description: "p95 latency is {{ $value }}s over the last 10 minutes (SLO: 500ms)"

      # Transaction Success Rate Warning
      - alert: TransactionSuccessRateLow
        expr: |
          (
            rate(payment_transactions_total{status="completed"}[5m]) /
            rate(payment_transactions_total[5m])
          ) < 0.96
        for: 5m
        labels:
          severity: warning
          service: payment-gateway
          team: fintech
        annotations:
          summary: "Transaction success rate below 96%"
          description: "Success rate is {{ $value | humanizePercentage }} (SLO: 96%)"

      # High Database Connection Usage
      - alert: DatabaseConnectionsHigh
        expr: |
          (
            cockroachdb_sql_conns{application="payment-service"} /
            cockroachdb_sql_conns_max{application="payment-service"}
          ) > 0.8
        for: 5m
        labels:
          severity: warning
          service: database
          team: fintech
        annotations:
          summary: "Database connection pool usage high"
          description: "Connection usage is {{ $value | humanizePercentage }}"

      # Kafka Consumer Lag
      - alert: KafkaConsumerLagHigh
        expr: |
          kafka_consumer_lag_sum{topic=~"payment.*"} > 1000
        for: 5m
        labels:
          severity: warning
          service: event-processing
          team: fintech
        annotations:
          summary: "Kafka consumer lag is high"
          description: "Consumer lag is {{ $value }} messages for {{ $labels.topic }}"

      # Redis Memory Usage
      - alert: RedisMemoryHigh
        expr: |
          redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
          service: cache
          team: fintech
        annotations:
          summary: "Redis memory usage above 85%"
          description: "Redis instance {{ $labels.instance }} memory usage: {{ $value | humanizePercentage }}"

  - name: payment_info_alerts
    rules:
      # Deployment Notification
      - alert: PaymentServiceDeployed
        expr: |
          changes(deployment_version{service="payment-service"}[5m]) > 0
        labels:
          severity: info
          service: payment-gateway
          team: fintech
        annotations:
          summary: "Payment service deployed"
          description: "New version {{ $labels.version }} deployed to {{ $labels.environment }}"

      # Configuration Change
      - alert: FeatureFlagChanged
        expr: |
          changes(feature_flag_state[5m]) > 0
        labels:
          severity: info
          service: feature-flags
          team: fintech
        annotations:
          summary: "Feature flag configuration changed"
          description: "Flag {{ $labels.flag_name }} changed to {{ $labels.state }}"
```

### AlertManager Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: "smtp.irancell.com:587"
  smtp_from: "alerts@fintech.irancell.com"
  smtp_auth_username: "alerts@fintech.irancell.com"
  smtp_auth_password: "TBD-SMTP-PASSWORD"

route:
  group_by: ["alertname", "severity", "service"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: "default"
  routes:
    - match:
        severity: critical
      receiver: "pagerduty-critical"
      group_wait: 10s
      repeat_interval: 5m
    - match:
        severity: warning
      receiver: "slack-warnings"
      group_wait: 2m
      repeat_interval: 2h
    - match:
        severity: info
      receiver: "email-info"
      group_wait: 10m
      repeat_interval: 24h

receivers:
  - name: "default"
    email_configs:
      - to: "fintech-team@irancell.com"
        subject: "[ALERT] {{ .GroupLabels.alertname }}"
        body: |
          Alert: {{ .GroupLabels.alertname }}
          Service: {{ .GroupLabels.service }}
          Severity: {{ .GroupLabels.severity }}

          {{ range .Alerts }}
          Description: {{ .Annotations.description }}
          Runbook: {{ .Annotations.runbook }}
          {{ end }}

  - name: "pagerduty-critical"
    pagerduty_configs:
      - service_key: "TBD-PAGERDUTY-INTEGRATION-KEY"
        severity: "{{ .GroupLabels.severity }}"
        description: "{{ .CommonAnnotations.summary }}"
        details:
          service: "{{ .GroupLabels.service }}"
          runbook: "{{ .CommonAnnotations.runbook }}"

  - name: "slack-warnings"
    slack_configs:
      - api_url: "TBD-SLACK-WEBHOOK-URL"
        channel: "#fintech-alerts"
        title: "⚠️ {{ .CommonAnnotations.summary }}"
        text: |
          *Service:* {{ .GroupLabels.service }}
          *Description:* {{ .CommonAnnotations.description }}
          *Runbook:* {{ .CommonAnnotations.runbook }}
        actions:
          - type: button
            text: "View Dashboard"
            url: "{{ .CommonAnnotations.dashboard }}"
          - type: button
            text: "Runbook"
            url: "{{ .CommonAnnotations.runbook }}"

  - name: "email-info"
    email_configs:
      - to: "fintech-team@irancell.com"
        subject: "[INFO] {{ .CommonAnnotations.summary }}"
        body: |
          {{ .CommonAnnotations.description }}

          Service: {{ .GroupLabels.service }}
          Time: {{ .CommonAnnotations.timestamp }}
```

---

## LGTM Stack Setup

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: "irancell-payment-prod"
    region: "iran-central"

rule_files:
  - "prometheus-alerts.yaml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: "payment-services"
    static_configs:
      - targets:
          - "payment-service:8080"
          - "wallet-service:8080"
          - "ledger-service:8080"
          - "api-gateway:8080"
    scrape_interval: 15s
    metrics_path: /metrics

  - job_name: "psp-adapters"
    static_configs:
      - targets:
          - "psp-adapter-a:8080"
          - "psp-adapter-b:8080"
    scrape_interval: 30s

  - job_name: "infrastructure"
    static_configs:
      - targets:
          - "cockroachdb-1:8080"
          - "cockroachdb-2:8080"
          - "cockroachdb-3:8080"
          - "redis-master:6379"
          - "kafka-1:9092"
          - "kafka-2:9092"
          - "kafka-3:9092"

  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8888"]
```

### Loki Configuration

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  log_level: info

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h # 30 days

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 64
  ingestion_burst_size_mb: 128

chunk_store_config:
  max_look_back_period: 720h # 30 days

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
  compaction_interval: 10m
```

### Tempo Configuration

```yaml
# tempo-config.yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

ingester:
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 168h # 7 days

storage:
  trace:
    backend: local
    local:
      path: /tempo/blocks
    pool:
      max_workers: 100
      queue_depth: 10000

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics]
      generate_native_histograms: true
```

---

## OpenTelemetry Configuration

### OTEL Collector Configuration

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  prometheus:
    config:
      scrape_configs:
        - job_name: "payment-services"
          static_configs:
            - targets:
                [
                  "payment-service:8080",
                  "wallet-service:8080",
                  "ledger-service:8080",
                ]
          scrape_interval: 15s
          metrics_path: /metrics

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  resource:
    attributes:
      - key: service.environment
        value: production
        action: upsert
      - key: service.cluster
        value: irancell-payment
        action: upsert

  filter:
    metrics:
      exclude:
        match_type: regexp
        metric_names:
          - go_.*
          - process_.*
    logs:
      exclude:
        match_type: regexp
        bodies:
          - ".*health check.*"

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"

  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
    labels:
      attributes:
        service.name: "service"
        level: "level"

  tempo:
    endpoint: "tempo:4317"
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch, resource, filter]
      exporters: [prometheus]

    logs:
      receivers: [otlp]
      processors: [batch, resource, filter]
      exporters: [loki]

    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [tempo]

  extensions: [health_check, pprof]
```

### Application Instrumentation

```yaml
# Payment Service OTEL Configuration
OTEL_SERVICE_NAME: payment-service
OTEL_SERVICE_VERSION: 1.0.0
OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL: grpc
OTEL_RESOURCE_ATTRIBUTES: service.name=payment-service,service.version=1.0.0,deployment.environment=production

# Trace Sampling Configuration
OTEL_TRACES_SAMPLER: traceidratio
OTEL_TRACES_SAMPLER_ARG: 0.01 # 1% sampling for regular traffic

# Custom Sampling Rules
OTEL_TRACES_SAMPLER_RULES: |
  - service_name: payment-service
    operation_name: "*error*"
    sampling_rate: 1.0  # 100% for errors
  - service_name: payment-service  
    attribute: payment.amount
    value_threshold: 1000000  # High-value transactions
    sampling_rate: 1.0  # 100% for high-value
```

---

## Dashboard Specifications

### Executive Dashboard Panels

```json
{
  "dashboard_config": {
    "title": "MTN Irancell Payment Gateway - Executive View",
    "refresh": "30s",
    "time_range": "1h",
    "panels": [
      {
        "id": 1,
        "title": "Current Error Rate vs SLO",
        "type": "stat",
        "query": "100 * (rate(http_requests_total{code=~\"4..|5..\"}[5m]) + rate(http_timeouts_total[5m]) + rate(psp_errors_total[5m])) / rate(http_requests_total[5m])",
        "thresholds": [
          { "value": 0, "color": "green" },
          { "value": 2, "color": "yellow" },
          { "value": 4, "color": "red" }
        ],
        "unit": "percent"
      },
      {
        "id": 2,
        "title": "Transaction Throughput (TPS)",
        "type": "timeseries",
        "queries": [
          "rate(payment_transactions_total[1m])",
          "rate(wallet_operations_total[1m])",
          "rate(refund_transactions_total[1m])"
        ]
      },
      {
        "id": 3,
        "title": "PSP Health Matrix",
        "type": "table",
        "queries": [
          "circuit_breaker_state by (psp)",
          "avg_over_time(psp_response_time_seconds[5m]) by (psp)",
          "rate(psp_errors_total[5m]) by (psp)"
        ]
      },
      {
        "id": 4,
        "title": "Revenue Impact (Last 24h)",
        "type": "stat",
        "query": "sum(increase(payment_amount_total{status=\"completed\",currency=\"IRR\"}[24h]))",
        "unit": "currencyIRR"
      }
    ]
  }
}
```

### Operations Dashboard Panels

```json
{
  "ops_dashboard_config": {
    "title": "Payment Gateway - Operations Dashboard",
    "refresh": "15s",
    "panels": [
      {
        "title": "Service Health Overview",
        "type": "stat",
        "queries": [
          "up{job=~\"payment.*\"} * 100",
          "up{job=\"cockroachdb\"} * 100",
          "up{job=\"redis\"} * 100",
          "up{job=\"kafka\"} * 100"
        ]
      },
      {
        "title": "Error Rate by Service",
        "type": "timeseries",
        "query": "rate(http_requests_total{code=~\"5..\"}[5m]) by (service)"
      },
      {
        "title": "Queue Depths & Backlogs",
        "type": "timeseries",
        "queries": [
          "kafka_consumer_lag by (topic)",
          "redis_list_length{list=~\".*queue.*\"} by (list)",
          "outbox_pending_events by (service)"
        ]
      },
      {
        "title": "Database Performance",
        "type": "timeseries",
        "queries": [
          "rate(cockroachdb_sql_query_count[5m])",
          "cockroachdb_sql_ddl_duration_p99",
          "cockroachdb_sql_dml_duration_p99"
        ]
      }
    ]
  }
}
```

---

## Escalation & Runbooks

### Alert Severity & Escalation Matrix

| **Severity** | **Response Time** | **Escalation Path**                      | **Notification**        | **Auto-Actions**              |
| ------------ | ----------------- | ---------------------------------------- | ----------------------- | ----------------------------- |
| **Critical** | <5 minutes        | L1→L2 (15min)→L3 (30min)→Manager (60min) | PagerDuty + SMS + Slack | Circuit breaker, Auto-scaling |
| **Warning**  | <30 minutes       | L1 acknowledge (30min)→L2 (2hr)          | Slack + Email           | Resource scaling              |
| **Info**     | <2 hours          | L1 acknowledge (4hr)                     | Email only              | Log for analysis              |

### Runbook URL Framework

| **Alert**                     | **Runbook URL**                | **Primary Actions**                                             | **Escalation Trigger**   |
| ----------------------------- | ------------------------------ | --------------------------------------------------------------- | ------------------------ |
| **PaymentErrorRateHigh**      | `/runbooks/payment-error-rate` | Check PSP status, Review error logs, Analyze traffic patterns   | No improvement in 15min  |
| **PSPCircuitBreakerOpen**     | `/runbooks/psp-outage`         | Verify PSP health, Enable backup PSP, Create incident           | Manual failover required |
| **DatabaseConnectionFailure** | `/runbooks/database-outage`    | Check cluster status, Review slow queries, Scale connections    | Data unavailability      |
| **PaymentLatencyHigh**        | `/runbooks/payment-latency`    | Check resource usage, Review DB performance, Scale services     | SLO breach imminent      |
| **KafkaConsumerLagHigh**      | `/runbooks/kafka-lag`          | Scale consumers, Check broker health, Review partition strategy | Processing delays >10min |

### Runbook Template Structure

```markdown
# Runbook: Payment Error Rate High (>4%)

## Overview

- **Alert:** PaymentErrorRateHigh
- **Severity:** Critical
- **SLO Impact:** Transaction Success Rate (96% target)

## Immediate Actions (0-5 minutes)

1. Check PSP adapter health: `kubectl get pods -l app=psp-adapter`
2. Review error distribution: Grafana > Payment Ops > Error Breakdown panel
3. Verify traffic patterns: Check for unusual spikes or DDoS

## Investigation Steps (5-15 minutes)

1. **PSP Integration:**

   - Query: `rate(psp_errors_total[5m]) by (psp)`
   - Action: Check circuit breaker states
   - Failover: Enable backup PSP if primary failing

2. **Database Performance:**

   - Query: `cockroachdb_sql_ddl_duration_p99 > 1`
   - Action: Identify slow queries, check connection pool
   - Scale: Add read replicas if read-heavy

3. **Application Errors:**
   - Logs: `{service="payment-service"} |= "ERROR" | json`
   - Action: Check for deployment issues, config changes
   - Rollback: Previous version if recent deployment

## Escalation Criteria

- Error rate >6% for 10+ minutes
- Multiple PSPs failing simultaneously
- Database cluster degradation
- Revenue impact >$10K/hour

## Contact Information

- **L1 Oncall:** +98-XXX-XXX-XXXX
- **L2 Engineer:** +98-XXX-XXX-XXXX
- **L3 Architect:** +98-XXX-XXX-XXXX
- **Incident Commander:** +98-XXX-XXX-XXXX
```

---

## Log Aggregation Strategy

### Structured Logging Standards

```json
{
  "payment_service_log_format": {
    "timestamp": "2025-08-24T10:30:45.123Z",
    "level": "INFO|WARN|ERROR",
    "service": "payment-service",
    "version": "1.0.0",
    "trace_id": "abc123def456",
    "span_id": "789ghi012",
    "correlation_id": "req-uuid-12345",
    "transaction_id": "txn-uuid-67890",
    "message": "Payment processed successfully",
    "details": {
      "psp": "provider-a",
      "amount_masked": "***00",
      "currency": "IRR",
      "processing_time_ms": 245,
      "retry_count": 0
    },
    "pii_scrubbed": true
  }
}
```

### Log Queries for Common Scenarios

```
# Critical Error Investigation
payment_errors: '{service="payment-service"} |= "ERROR" | json | level="ERROR"'

# PSP Integration Issues
psp_timeouts: '{service="psp-adapter"} |= "timeout" | json | psp=~".*"'

# Fraud Alerts
fraud_detection: '{service="fraud-service"} |= "FRAUD_DETECTED" | json | risk_score > 80'

# High-Value Transaction Monitoring
high_value: '{service="payment-service"} | json | amount_masked=~".*000" and currency="IRR"'

# Webhook Processing Failures
webhook_failures: '{service="webhook-handler"} |= "signature_invalid" | json'

# Database Performance Issues
db_slow_queries: '{service="payment-service"} | json | db_query_time_ms > 1000'

# Security Audit Trail
security_events: '{service=~".*"} |= "SECURITY" | json | event_type=~"auth_failure|access_denied|suspicious_activity"'
```

### Log Retention Policy

| **Log Type**         | **Retention Period** | **Compression**    | **Archive Location** | **Access Control** |
| -------------------- | -------------------- | ------------------ | -------------------- | ------------------ |
| **Application Logs** | 30 days              | gzip               | Local filesystem     | Service accounts   |
| **Security Logs**    | 90 days              | gzip + encryption  | Secure archive       | Security team only |
| **Audit Logs**       | 7 years              | Strong compression | Cold storage         | Compliance team    |
| **Debug Traces**     | 7 days               | Standard           | Local filesystem     | Dev team access    |
| **Webhook Logs**     | 14 days              | gzip               | Local filesystem     | Integration team   |

---

## Custom Metrics Implementation

### Business Metrics Collection

```go
// Payment Service Custom Metrics
var (
    // Transaction metrics
    paymentCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "payment_transactions_total",
            Help: "Total number of payment transactions",
        },
        []string{"status", "psp", "currency", "type"},
    )

    // Revenue tracking (masked amounts)
    revenueCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "payment_amount_total",
            Help: "Total payment amounts processed",
        },
        []string{"currency", "psp", "status"},
    )

    // Processing time distribution
    processingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "payment_processing_duration_seconds",
            Help: "Time spent processing payments end-to-end",
            Buckets: []float64{0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0},
        },
        []string{"psp", "status", "type"},
    )

    // Circuit breaker state
    circuitBreakerState = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "circuit_breaker_state",
            Help: "Circuit breaker state (0=closed, 1=open, 2=half-open)",
        },
        []string{"service", "psp_name"},
    )

    // Queue depth monitoring
    queueDepth = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "queue_depth_current",
            Help: "Current queue depth for async processing",
        },
        []string{"queue_name", "service"},
    )
)
```

### Infrastructure Metrics

```yaml
# Key Infrastructure Queries

# API Gateway Performance
api_rps: 'rate(http_requests_total{service="api-gateway"}[1m])'
api_latency_p95: 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service="api-gateway"}[5m]))'

# Database Health
db_connections: "cockroachdb_sql_conns / cockroachdb_sql_conns_max"
db_query_rate: "rate(cockroachdb_sql_query_count[1m])"
db_slow_queries: "cockroachdb_sql_ddl_duration_p99 + cockroachdb_sql_dml_duration_p99"

# Cache Performance
redis_hit_rate: "rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))"
redis_memory_usage: "redis_memory_used_bytes / redis_memory_max_bytes"

# Message Queue Health
kafka_lag: "kafka_consumer_lag_sum by (topic, consumer_group)"
kafka_throughput: "rate(kafka_messages_consumed_total[1m]) by (topic)"

# Container Resources
cpu_usage: "rate(container_cpu_usage_seconds_total[5m]) by (container)"
memory_usage: "container_memory_usage_bytes / container_spec_memory_limit_bytes by (container)"
```

---

## Alert Testing & Validation

### Alert Rule Testing

```bash
# PromQL Testing Commands

# Test error rate calculation
promtool query instant 'localhost:9090' \
  '(rate(http_requests_total{code=~"5.."}[5m]) + rate(http_timeouts_total[5m])) / rate(http_requests_total[5m])'

# Validate latency percentiles
promtool query instant 'localhost:9090' \
  'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))'

# Check PSP circuit breaker state
promtool query instant 'localhost:9090' \
  'circuit_breaker_state{service="psp-adapter"}'

# Validate alert rule syntax
promtool check rules prometheus-alerts.yaml
```

### Chaos Testing Integration

```yaml
# Chaos Testing Scenarios for Alert Validation

chaos_scenarios:
  - name: "psp_timeout_simulation"
    target: "psp-adapter-a"
    action: "delay_requests"
    duration: "10m"
    parameters:
      delay: "5s"
      percentage: 50
    expected_alerts: ["PSPCircuitBreakerOpen", "PaymentLatencyHigh"]

  - name: "database_connection_exhaustion"
    target: "payment-service"
    action: "connection_leak"
    duration: "5m"
    expected_alerts: ["DatabaseConnectionsHigh"]

  - name: "high_error_rate_injection"
    target: "payment-service"
    action: "inject_http_errors"
    duration: "7m"
    parameters:
      error_rate: 6
      codes: [500, 502, 503]
    expected_alerts: ["PaymentErrorRateHigh"]
```
