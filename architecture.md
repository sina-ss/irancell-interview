# Payments Platform — Architecture Document

> Author: Sina Sepahvand
> Scope: Payments API + Adapters + Wallet + Ledger + Recon + OTEL + LGTM

---

## Diagram Placeholders

**System Architecture:** `/architecture.mmd`
[![architecture](https://github.com/sina-ss/irancell-interview/blob/main/images/architecture.png?raw=true)](https://github.com/sina-ss/irancell-interview/blob/main/images/architecture.png?raw=true)
**Sequence (Happy Path + Failover):** `/sequence-happy-path.mmd`
[![sequence-happy-path](https://github.com/sina-ss/irancell-interview/blob/main/images/sequence-happy-path.png?raw=true)](https://github.com/sina-ss/irancell-interview/blob/main/images/sequence-happy-path.png?raw=true)
**Network & Security Zones:** `</network-security.mmd`
[![network-security](https://github.com/sina-ss/irancell-interview/blob/main/images/network-security.png?raw=true)](https://github.com/sina-ss/irancell-interview/blob/main/images/network-security.png?raw=true)
## 1. Core Component Responsibilities

**Edge & Control**

- **WAF + DDoS + LB:** volumetric protection, IP reputation, TLS offload, L7 rules.
- **API Gateway (from scratch):** TLS 1.3, mTLS internal, OAuth2 (client-cred) for services, HMAC for webhooks, **rate limits/quotas**, **Idempotency-Key enforcement**, canonical request/response logging (PAN/PII stripped), request signing to adapters.
- **IdP (Keycloak):** OIDC/SAML, MFA, RBAC, service accounts with least privilege.
- **Feature Flags / Kill-switches:** per-PSP toggles; routing/timeout/CB thresholds; audited changes.

**Core Business**

- **Payments Service (orchestrator):** request validation (in-process policy library), routing, PSP selection, saga state machine, **transactional outbox**, compensations (void/refund/reversal), **adapter-SLO aware routing**.
- **Wallet Service:** balances, reservations/holds, cash-in/out; derives state from ledger or maintains operational shadow state reconciled to ledger.
- **Fraud Service (edge):** velocity checks, device fingerprint hook, rule engine integration points (no ML build-out).

**Integration**

- **PSP/Bank Adapters (A/B + Bank):** per-connector **timeouts**, **retries + jitter**, **circuit breakers**, **bulkheads**, response normalization, HMAC/mTLS, per-PSP health metrics.
- **Webhook Ingest:** allowlist + mTLS/HMAC verification, **store-then-ack**, ordered processing, **Kafka topics** with **DLQ & replay**.

**Financial Integrity**

- **Ledger Service (double-entry):** append-only journal, **idempotent on `event_id`**, derived balances, audit queries.
- **Reconciliation:** T+0 streaming from events/webhooks; T+1 SFTP files (MinIO-backed), hash/manifest verification, break detection → DLQ + operator UI.

**Data & Observability**

- **CockroachDB _or_ PostgreSQL** (see §3), **Redis** (idempotency/locks/rate), **Kafka** (events + webhooks), **MinIO** (payloads/files).
- **OpenTelemetry + LGTM:** OTEL collectors → Prometheus (metrics) + Tempo (traces) + Loki (logs) + Grafana (viz) + Alertmanager (routing).

**Security Services**

- **Vault (Secrets/KMS):** PSP creds, HMAC keys, certs; short-lived tokens.
- **HSM:** tokenization/detokenization keys, PAN protection, PKCS#11.

---

## 2. Data Flow Patterns

**Idempotency (edge + service)**

- Header: `Idempotency-Key: <uuid>`; **fingerprint** = `(method, path, merchant_id, amount_minor, currency[, payment_method])`.
- **Fast path:** Redis lookup → return cached response **iff fingerprint matches**.
- **Write-through backstop:** persist `{key, fingerprint, status_code, body_hash, expires_at}` to OLTP to survive Redis loss (TTL 48–72h).

**Outbox → Kafka**

- Within the same DB TX as business write: insert to `outbox_events`.
- Relayer publishes to Kafka **exactly-once producer** semantics; consumers are **idempotent** by `event_id`.

**Retries & Breakers**

- Bounded exponential backoff with jitter; **2 attempts** after initial try; cap total adapter budget to **≤8s**.
- Breaker opens on 20%+ error/timeouts in 60s **or** 10 consecutive failures; half-open probe every 15–30s; Payments respects breaker state for routing.

**Webhook “store-then-process”**

- Verify allowlist + signature → **persist raw payload** (MinIO) + **enqueue Kafka event**; ack 200 immediately; processors handle ordering & DLQ.

**Ledger write model**

- Append-only double-entry; **no updates**; corrections as new entries.
- **Idempotent** on `(source_system, source_event_id)` unique constraint.

---

## 3. Database Schema Design (CockroachDB and PostgreSQL)

> Both designs model the same entities; the **storage/layout** differs for locality, partitioning, and replication.

### 3.1 Core Entities (common logical model)

- **payments** — business transaction header/state.
- **outbox_events** — TX-local message log.
- **ledger_entries** — journal rows (DR/CR).
- **accounts** — ledger accounts (platform, PSP clearing, merchant, wallet).
- **webhooks_in** — raw inbound events (normalized metadata).
- **reconciliation_breaks** — mismatches and their lifecycle.
- **idempotency_keys** — key→fingerprint→result cache (durable).

### 3.2 CockroachDB (Cluster A active, Cluster B passive via CDC/backup)

**Keys & locality**

- Use **UUID v7** for write distribution; consider **hash-sharded** primary indexes on hot tables.
- Table localities: keep write-heavy tables “REGIONAL BY TABLE” (DC_A); **CDC** to DC_B for RPO ≤ 5m; use **Follower Reads** for analytics where feasible.

**DDL sketch (selected)**

```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  merchant_id UUID NOT NULL,
  amount_minor INT8 NOT NULL,
  currency CHAR(3) NOT NULL,
  state STRING NOT NULL,                -- INITIATED|AUTHORIZED|CAPTURED|DECLINED|FAILED|REFUNDED|REVERSED
  psp_ref STRING,
  idempotency_key STRING,
  fingerprint_hash BYTES NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_payments_idem ON payments (idempotency_key, fingerprint_hash);
CREATE INDEX idx_payments_state_created ON payments (state, created_at);
CREATE INDEX idx_payments_merchant_created ON payments (merchant_id, created_at);

CREATE TABLE outbox_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agg_type STRING NOT NULL,             -- "payment"
  agg_id UUID NOT NULL,
  event_type STRING NOT NULL,           -- "AUTH_APPROVED", ...
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  published_at TIMESTAMPTZ,
  status STRING NOT NULL DEFAULT 'NEW', -- NEW|PUBLISHED|FAILED
  retry_count INT4 DEFAULT 0
) locality regional by table; -- keep close to writers

CREATE TABLE ledger_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL,               -- idempotency anchor
  txn_id UUID,
  account_id UUID NOT NULL,
  direction STRING NOT NULL,            -- DR|CR
  amount_minor INT8 NOT NULL,
  currency CHAR(3) NOT NULL,
  description STRING,
  created_at TIMESTAMPTZ DEFAULT now(),
  CONSTRAINT uq_ledger_event UNIQUE (event_id, account_id, direction)
);

CREATE TABLE idempotency_keys (
  key STRING PRIMARY KEY,
  fingerprint_hash BYTES NOT NULL,
  status_code INT4 NOT NULL,
  body_hash BYTES NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE webhooks_in (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider STRING NOT NULL,
  topic STRING NOT NULL,
  sig_valid BOOL NOT NULL,
  seq_id STRING,
  received_at TIMESTAMPTZ DEFAULT now(),
  payload JSONB NOT NULL,
  processed_at TIMESTAMPTZ,
  status STRING NOT NULL DEFAULT 'NEW'
);

CREATE TABLE reconciliation_breaks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source STRING NOT NULL,
  ref STRING NOT NULL,
  amount_delta INT8 NOT NULL,
  currency CHAR(3) NOT NULL,
  detected_at TIMESTAMPTZ DEFAULT now(),
  status STRING NOT NULL DEFAULT 'OPEN',
  notes STRING
);
```

### 3.3 PostgreSQL 16 (Primary + read replicas; monthly partitioning)

**Partitioning & indexes**

- **payments, ledger_entries, webhooks_in** partitioned **BY RANGE (created_at)** monthly.
- **B-tree** on `(idempotency_key, fingerprint_hash)`, `(state, created_at)`, `(merchant_id, created_at)`.
- Replica read scaling for GETs/reporting; primary for writes.

**DDL sketch (differences)**

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
-- For UUID v7 use a custom extension or fall back to v4; keep random distribution.
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  merchant_id UUID NOT NULL,
  amount_minor BIGINT NOT NULL,
  currency CHAR(3) NOT NULL,
  state TEXT NOT NULL,
  psp_ref TEXT,
  idempotency_key TEXT,
  fingerprint_hash BYTEA NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ
) PARTITION BY RANGE (created_at);

-- Create monthly partitions (automation via maintenance job)
-- Indexes as in CRDB example.

-- Other tables mirror CRDB with the same uniqueness/idempotency rules.
```

**Outbox**

- Use a **single TX** to write business row + outbox row.
- A **reliable relayer** (dequeue with `FOR UPDATE SKIP LOCKED`) publishes to Kafka; mark as `PUBLISHED`.

---

## 4. API Specification Framework

**Standards**

- **REST/JSON**, versioned base path `/v1`.
- **Auth:** OIDC OAuth2 (Client Credentials) for first-party clients; HMAC + allowlist for inbound webhooks; mTLS between services.
- **Idempotency:** `Idempotency-Key` required for POST/PUT/PATCH; semantics per §2.
- **Rate Limits:** 429 with `X-RateLimit-*` headers.

**Core Endpoints (selected)**

- `POST /v1/payments` — create (auth+capture or auth-only).
- `GET /v1/payments/{id}` — status.
- `POST /v1/payments/{id}/capture` — capture full/partial.
- `POST /v1/payments/{id}/void` — void uncaptured auth.
- `POST /v1/payments/{id}/refund` — refund (full/partial).
- `POST /v1/wallets/{wallet_id}/cashin` / `.../cashout` — wallet ops.
- `POST /v1/webhooks/psp/{provider}` — inbound webhooks (store-then-ack).

**Headers**

```
Authorization: Bearer <JWT>
Idempotency-Key: <uuid>
Content-Type: application/json
X-Request-Signature: <HMAC>   # for webhooks
```

**Error Model**

```json
{
  "error_code": "ADAPTER_TIMEOUT",
  "message": "Upstream PSP timed out",
  "retryable": true,
  "correlation_id": "trace/span id"
}
```

**Versioning & Compatibility**

- **URI versioning** + **backward-compatible** response fields; new fields are additive.
- **Deprecation policy:** `Sunset` header + changelog.

---

## 5. Capacity Planning for 400 Concurrent Transactions

**Assumptions**

- “400 concurrent” = **400 VUs**; sustained throughput target **≈300 rps** (adjustable 250–350).
- Internal SLOs (excluding human 3DS): **p50 ≤ 150 ms**, **p95 ≤ 500 ms**, **p99 ≤ 1200 ms**.

**Little’s Law sanity check**
If λ = 300 rps, and concurrent users N = 400, then average time in system `W = N/λ ≈ 1.33 s`.
→ Satisfiable when client think time and non-critical async steps overlap; **service latency** p95 remains ≤ 500 ms.

### 5.1 Resource Allocation Matrix (initial sizing)

| Layer        | Component          |             Replicas | CPU/Pod |   Mem/Pod | Notes                                           |
| ------------ | ------------------ | -------------------: | ------: | --------: | ----------------------------------------------- |
| Edge         | WAF/LB             |              managed |      — |        — | Provider/infra service                          |
| Gateway      | API Gateway        |                 4–6 |  1 vCPU | 1.5–2 GB | HPA @60% CPU; sticky for idempotency cache hits |
| Control      | Keycloak           |                    2 |       1 |      2 GB | External DB; session cache in Redis (short TTL) |
| Orchestrator | Payments           |                 6–8 |    1–2 |   2–3 GB | HPA; DB pool 60–100; outbox relayer x2         |
| Domain       | Wallet             |                 3–4 |       1 |      2 GB | DB pool 40–60                                  |
| Domain       | Fraud              |                 2–3 |       1 |      2 GB | Cache heavy, low write                          |
| Integration  | Adapters (per PSP) |                 3–4 |  0.5–1 |   1–2 GB | Separate pools, CB/bulkhead                     |
| Async        | Webhook Ingest     |                 3–4 |  0.5–1 |   1–2 GB | Kafka producers; DLQ consumer x2                |
| Finance      | Ledger             |                 3–4 |       1 |      2 GB | Append-only; index/write optimized              |
| Events       | Kafka              |            3 brokers |       4 |  8–16 GB | 6–8 partitions/topic to start; ISR=3           |
| Cache        | Redis Cluster      |          3 primaries |       2 |  8–16 GB | AOF on; 25–50k OPS headroom                    |
| OLTP         | CRDB**A**    |              5 nodes |    4–8 | 16–32 GB | NVMe SSD; 15–20k TPS write headroom            |
| OLTP         | CRDB**B**    |           3–5 nodes |    4–8 | 16–32 GB | CDC/backup from A                               |
| Object       | MinIO              |                 3–4 |       2 |  8–16 GB | Erasure coding; versioning on                   |
| Obs          | OTEL               |                 2–3 |     0.5 | 0.5–1 GB | Daemonset/sidecar mix                           |
| Obs          | Prometheus         |               2 (HA) |       4 |  8–16 GB | Remote write optional                           |
| Obs          | Loki               | 3 (ingester/querier) |    2–4 |  8–16 GB | Retention 30–90 days hot                       |
| Obs          | Tempo              |                    3 |    2–4 |  8–16 GB | 10% parent-based sampling                       |
| Obs          | Alertmanager       |                    2 |     0.5 |      1 GB | HA mesh                                         |
| Sec          | Vault              |                 2–3 |       2 |      8 GB | Integrated HSM where available                  |
| Sec          | HSM                |              HA pair |      — |        — | Network HSM latency budget < 5 ms               |

### 5.2 Performance Projections (starting targets)

- **Gateway throughput:** \~1.2–1.8k rps cluster (4–6 pods) with p95 < 10 ms overhead.
- **Payments service:** 150–250 rps per pod p95 ≤ 40 ms (sans adapter/DB).
- **Adapter call windows:** connect 300–500 ms, request 2.5–3.5 s, **max 2 retries**; CB prevents global tail amplification.
- **DB write:** 2–4 ms index hit; 10–20 ms OLTP roundtrip median under load (NVMe + tuned pools).
- **Kafka:** 10–30 ms produce acks=all; consumers lag < 1s steady state.

> Tune via k6: ramp to **400 VUs**, hold 15 min @ target RPS, validate p95/99 and **error-rate alert** trigger logic.

---

## 6. Security Architecture Implementation

**Network & Transport**

- Segmented zones (DMZ / App / Integration / Data).
- **mTLS** for service↔service; **TLS 1.3** at the edge.
- PSP links: **mTLS + HMAC**; Webhooks: **allowlist + signature + store-then-ack**.

**Data Protection**

- **PAN never logged**; tokenization via **HSM**; Vault-managed keys; encrypt at rest (DB, MinIO, Kafka at rest as needed).
- **PCI-aligned logging**: centralized, tamper-evident; 12-month retention (90-day hot).
- **Secrets**: short-lived tokens, periodic rotation, sealed storage; no secrets in images.

**Access & Change**

- RBAC everywhere; least-privilege service roles; just-in-time operator access.
- CI/CD signed artifacts; **GitOps** for config; change windows with approval gates.

**Hardening**

- Host baseline (CIS), image scanning, SBOMs, CVE gates; dependency pinning.
- Time sync (NTP), clock-skew checks in signatures.
- Rate limits + anomaly detection at gateway.

---

## 7. Integration Patterns & Protocols

**Synchronous**

- REST/JSON over HTTPS; **OAuth2** for first-party, **mTLS** for adapters, **HMAC** per-PSP.

**Asynchronous**

- **Kafka** topics for: `payments.events`, `webhooks.in`, `recon.*`; DLQs per stream; replay tools.

**Batch**

- **SFTP** for T+1; file naming convention + SHA-256 manifests; idempotent file loads; break detection pipeline.

**Resilience patterns**

- Retries with jitter; **circuit breakers**; **bulkheads** (conn pools, worker pools); **feature flags** to drain/disable a bad PSP.

---

## 8. Deployment & Infrastructure Patterns

**Topology**

- **Active–Passive DCs**: DC_A active; DC_B passive via **CDC/backup** for CRDB; Kafka mirror; MinIO bucket sync.
- **Cutover**: DNS or anycast plan with runbook to hit **RTO ≤ 30 min**; periodic DR drills.

**Platform**

- **Kubernetes (preferred)**:

  - Namespaces by domain; **pod disruption budgets**, **pod anti-affinity**, separate **node pools** for data vs stateless.
  - **HPA** (CPU 60%, RPS/latency SLOs), **VPA** for baseline tuning.
  - **Blue/green** + **canary** rollouts (gateway + payments + adapters).
  - **StatefulSets** for Kafka/MinIO; operators where available (CRDB/PG).
  - **NetworkPolicies**: default-deny; explicit egress to PSPs.

**Ops**

- **Backups**: nightly full + 15-min incrementals; restore validation; protected timestamps (CRDB).
- **Config**: ConfigMaps/Secrets from Git; feature flags service with local cached fallback.
- **Observability**: OTEL sidecars/daemonsets; SLO dashboards (RPS, p95/p99, error rate, CB state, queue depth, consumer lag).
- **Runbooks**: high error rate, PSP outage, queue backlog, p99 spike, DR cutover.

---

### Appendices

**A) Example Idempotency (pseudocode)**

```pseudo
key = req.header["Idempotency-Key"]
fp  = hash(method, path, merchant_id, amount_minor, currency, payment_method)

rec = redis.get(key)
if rec and rec.fingerprint == fp:
  return rec.response

resp = process_payment()
persist_idempotency(key, fp, resp.status, hash(resp.body), ttl=72h)  // write-through to DB
redis.setex(key, {fingerprint:fp, response:resp}, 72h)
return resp
```

**B) Example Adapter Retry Budget**

- Try #1; if timeout/5xx → backoff `200ms * 2^i + jitter(0–100ms)`, max 2 retries, total ≤ 8s; then open breaker.
