> This file is **for myself** and does not need to be **seen by Irancell**.
> I use this file to be able to determine the overall roadmap for myself.

## 1. Project Roadmap

| **Phase**                               | **Goals**                                             | **Key Outputs**                                      | **Dependencies**            |
| --------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------- | --------------------------- |
| **Session 1: Best Practices**           | Survey payment system patterns, compliance frameworks | Best practices summary, compliance checklist         | Session 0 approval          |
| **Session 2: Architecture Design**      | Core system design, component responsibilities        | Architecture diagram (Mermaid), component specs      | Assumptions confirmed       |
| **Session 3: SLO/Observability**        | Metrics, dashboards, alerting strategy                | Alert rules, LGTM+OTEL stack config, SLO definitions | Architecture baseline       |
| **Session 4: Capacity & Testing**       | Load testing approach, performance validation         | k6 test scenarios, capacity planning                 | Architecture + SLOs         |
| **Session 5: Frequent Issues Analysis** | Production issues identification, root cause patterns | Top 10 issues + mitigation strategies                | Technical design complete   |
| **Session 6: Risk Assessment**          | Security, compliance, operational risk analysis       | Risk register, security controls                     | Issues analysis complete    |
| **Session 7: PPT Assembly**             | Slide creation, narrative flow                        | Draft PowerPoint outline + content                   | All technical content ready |
| **Session 8: Refinement**               | Speaker notes, timing, Q&A prep                       | Final presentation package                           | Draft deck complete         |

## 2. Resolved Technical Decisions

### Database Strategy

**Primary Recommendation: CockroachDB**

- **Rationale:** Global distributed transactions, ACID compliance, horizontal scaling, resilient to node failures
- **Use case:** Core payment transactions, ledger entries, user accounts

**Alternative: PostgreSQL + Redis**

- **Rationale:** Mature ecosystem, familiar tooling, cost-effective
- **Trade-off:** More complex sharding/replication management

**Specialized Databases:**

- **Time-series data (metrics):** Prometheus/VictoriaMetrics
- **Logs:** Loki (part of LGTM stack)
- **Caching:** Redis Cluster
- **Search/Analytics:** OpenSearch (compliance logs, fraud detection)
- **Configuration:** Consul/etcd (feature flags, circuit breaker configs)

### API Gateway Architecture

**From-Scratch Design Components:**

- Rate limiting (per-client, per-endpoint)
- Authentication/authorization (OAuth2, mTLS)
- Request/response transformation
- Circuit breakers per PSP
- Idempotency key enforcement
- Request signing/validation
- IP whitelisting for webhooks

**Integration Readiness:**

- If existing gateway exists: API proxy pattern for gradual migration
- Standard interfaces: OpenAPI 3.0 specs, async event contracts

## 3. Working Assumptions

| **Category**               | **Assumption**                                               | **Rationale**                                                    |
| -------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------- |
| **Transaction Definition** | Auth+Capture, Refund, Reversal, CashIn/Out as complete flows | Per Irancell clarification: multi-step microservice transactions |
| **Load Profile**           | 400 concurrent any-type transactions = 300-500 req/s peak    | Mixed transaction types, some multi-step                         |
| **Error Calculation**      | `(5xx + 4xx + timeouts + PSP errors) / total_requests`       | All error types count toward 4% threshold                        |
| **Alert Window**           | Rolling 5-minute window, no minimum traffic threshold        | Per Irancell: 4% of total rejections/errors in any 5min window   |
| **Database Primary**       | CockroachDB for ACID + horizontal scaling                    | Embargo-resilient, self-managed clustering                       |
| **Observability**          | LGTM + OpenTelemetry for distributed tracing                 | Complete observability pipeline                                  |
| **Security Stance**        | Zero-trust, PCI-DSS Level 1, encryption everywhere           | Maximum security posture                                         |

## 4. Transaction Flow Definitions

| **Transaction Type**       | **Microservice Flow**                                        | **Completion Criteria**              |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------ |
| **Payment (Auth+Capture)** | Auth → Validation → Capture → Ledger → Notification          | Funds captured + ledger updated      |
| **Refund**                 | Validation → Refund → Ledger Reversal → Notification         | Refund processed + ledger updated    |
| **Reversal**               | Detection → Auto-Reversal → Ledger Correction → Alert        | System-initiated correction complete |
| **Cash-In (Wallet)**       | Validation → Balance Update → Ledger → Confirmation          | Wallet credited + ledger balanced    |
| **Cash-Out (Wallet)**      | Balance Check → Deduction → External Transfer → Confirmation | Funds withdrawn + ledger balanced    |

## 5. Error Rate Alerting Framework

**Alert Rule (PromQL-style):**

```promql
(
  rate(http_requests_total{code=~"4.."}[5m]) +
  rate(http_requests_total{code=~"5.."}[5m]) +
  rate(http_timeouts_total[5m]) +
  rate(psp_errors_total[5m])
) / rate(http_requests_total[5m]) > 0.04
```

**Alert Configuration:**

- **Severity:** Critical (immediate PagerDuty)
- **Window:** 5-minute rolling average
- **Threshold:** >4% error rate (no minimum traffic requirement)
- **Runbook:** `TBD: /runbooks/high-error-rate-investigation`

## 6. Integration Strategy

**PSP Integration Pattern:**

- **Interface:** REST APIs with mTLS authentication
- **Signatures:** HMAC-SHA256 for webhook validation
- **Retry Policy:** Exponential backoff (2^n seconds, max 5 attempts)
- **IP Whitelisting:** Configurable allowlists per PSP
- **Fallback:** Circuit breaker → secondary PSP routing

**Existing System Integration:**

- **Data Migration:** ETL pipelines with validation checkpoints
- **API Compatibility:** Adapter pattern for legacy endpoints
- **Gradual Cutover:** Blue-green deployment with traffic splitting
