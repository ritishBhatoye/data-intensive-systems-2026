# Operations & Production Guide

> Essential knowledge for running data-intensive systems in production. Covers monitoring, SRE practices, incident response, and operational excellence.

---

## 1. The Observability Stack

### 1.1 Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PILLARS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│      ┌─────────────────────────────────────────────────────────┐        │
│      │                                                         │        │
│      │                   METRICS                               │        │
│      │   "What happened?"                                      │        │
│      │                                                         │        │
│      │   • Counters (total requests)                         │        │
│      │   • Gauges (current connections)                     │        │
│      │   • Histograms (response times)                      │        │
│      │                                                         │        │
│      │   Tools: Prometheus, Datadog, CloudWatch, Mimir       │        │
│      │                                                         │        │
│      └─────────────────────────────────────────────────────────┘        │
│                                │                                        │
│      ┌─────────────────────────────────────────────────────────┐        │
│      │                                                         │        │
│      │                    LOGS                                 │        │
│      │   "Why did it happen?"                                 │        │
│      │                                                         │        │
│      │   • Structured JSON logs                              │        │
│      │   • Log levels (DEBUG, INFO, WARN, ERROR)            │        │
│      │   • Correlation IDs for tracing                       │        │
│      │                                                         │        │
│      │   Tools: Loki, ELK, Splunk, CloudWatch Logs           │        │
│      │                                                         │        │
│      └─────────────────────────────────────────────────────────┘        │
│                                │                                        │
│      ┌─────────────────────────────────────────────────────────┐        │
│      │                                                         │        │
│      │                   TRACES                               │        │
│      │   "How did it happen?"                                 │        │
│      │                                                         │        │
│      │   • Distributed request tracing                      │        │
│      │   • Latency breakdown by component                    │        │
│      │   • Service dependency map                            │        │
│      │                                                         │        │
│      │   Tools: Jaeger, Zipkin, Tempo, X-Ray                 │        │
│      │                                                         │        │
│      └─────────────────────────────────────────────────────────┘        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  OPENTELEMETRY: The open standard unifying all three                    │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │                                                              │        │
│  │  Application → OTel SDK → OTel Collector                   │        │
│  │                           ↓    ↓    ↓                      │        │
│  │                      Metrics  Logs  Traces                  │        │
│  │                                                              │        │
│  │  Vendor-neutral, supported by all major observability tools  │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. SLO/SLI/SLA: The Definitions

### 2.1 Understanding the Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SLO / SLI / SLA HIERARCHY                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   SLA (Service Level Agreement)                                         │
│   ════════════════════════════                                          │
│   • Contract with customers                                              │
│   • Legal consequences if missed                                        │
│   • Example: "99.9% uptime, or 0.1% credit"                           │
│   • Usually less strict than SLO                                        │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                  │   │
│   │   SLO (Service Level Objective)                                │   │
│   │   ════════════════════════════════════                         │   │
│   │   • Internal goal that team commits to                         │   │
│   │   • Stricter than SLA (buffer for safety)                     │   │
│   │   • Example: "99.95% latency under 200ms"                    │   │
│   │                                                                  │   │
│   │   ┌─────────────────────────────────────────────────────────┐  │   │
│   │   │                                                          │  │   │
│   │   │   SLI (Service Level Indicator)                        │  │   │
│   │   │   ════════════════════════════                         │  │   │
│   │   │   • Actual metric being measured                       │  │   │
│   │   │   • Example: "p99 latency of /api/timeline"          │  │   │
│   │   │                                                          │  │   │
│   │   └─────────────────────────────────────────────────────────┘  │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  EXAMPLE:                                                                │
│                                                                          │
│  SLA: 99.9% availability (8.76 hours downtime/year)                   │
│       │                                                                │
│  SLO: 99.95% availability (4.38 hours downtime/year)                 │
│       │  (5% error budget for SLO to meet SLA)                       │
│       │                                                                │
│  SLI: "requests with successful response / total requests"            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ERROR BUDGET:                                                          │
│  • Error budget = 100% - SLO%                                          │
│  • Example: 99.95% SLO → 0.05% error budget                          │
│  • Daily: 0.05% × 86,400 = 43.2 seconds allowed failures per day     │
│                                                                          │
│  BURN RATE:                                                             │
│  • How fast we're consuming error budget                               │
│  • Target burn rate: < 5x (don't consume 1 day budget in 1 hour)      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Monitoring Dashboards

### 3.1 Dashboard Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION DASHBOARD TEMPLATE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                      OVERVIEW ROW                                │   │
│  │                                                                   │   │
│  │   ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │   │
│  │   │   RPS      │ │  Latency   │ │ Error Rate │ │   Uptime   │ │   │
│  │   │            │ │  p50/p99   │ │   (5xx)    │ │  (24h)     │ │   │
│  │   │ 12,340/s   │ │ 45ms/180ms │ │   0.12%    │ │  99.97%    │ │   │
│  │   └────────────┘ └────────────┘ └────────────┘ └────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    LATENCY ROW                                    │   │
│  │                                                                   │   │
│  │   Response Time Distribution (by percentile)                      │   │
│  │   ─────────────────────────────────────                          │   │
│  │   p50: 45ms    p90: 89ms    p95: 145ms    p99: 180ms          │   │
│  │                                                                   │   │
│  │   ┌─────────────────────────────────────────────────────────┐   │   │
│  │   │  Timeline (ms)                                          │   │   │
│  │   │  ████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░  │   │   │
│  │   │  0      50      100     150     200     250     300   │   │   │
│  │   └─────────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    ERROR ROW                                       │   │
│  │                                                                   │   │
│  │   Error Rate by Status Code                                      │   │
│  │   ─────────────────────────────────────                          │   │
│  │   2xx: ████████████████████████████████████████████████ 99.3%    │   │
│  │   4xx: ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 0.5%    │   │
│  │   5xx: █░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 0.2%    │   │
│  │                                                                   │   │
│  │   Top 5 Errors (last 24h)                                       │   │
│  │   ─────────────────────────                                       │   │
│  │   1. 500: Database connection timeout         1,234 errors     │   │
│  │   2. 503: Service unavailable               456 errors        │   │
│  │   3. 429: Rate limit exceeded               234 errors        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                   DEPENDENCIES ROW                                │   │
│  │                                                                   │   │
│  │   Service Dependency Health                                     │   │
│  │   ────────────────────────────                                   │   │
│  │   API Gateway: ● Healthy          PostgreSQL: ● Healthy         │   │
│  │   Redis Cache: ● Healthy          Kafka: ● Healthy              │   │
│  │   Elasticsearch: ● Healthy       S3: ● Healthy                │   │
│  │                                                                   │   │
│  │   External Dependencies                                          │   │
│  │   ────────────────────────                                       │   │
│  │   Stripe API: ● Healthy (latency: 45ms)                        │   │
│  │   AWS Route53: ● Healthy                                        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Alerting Best Practices

### 4.1 Alert Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ALERT HIERARCHY                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   P1: CRITICAL (Page immediately)                              │   │
│  │   ════════════════════════════════════                          │   │
│  │   • Service completely down                                     │   │
│  │   • SLO breach imminent                                         │   │
│  │   • Data corruption/loss                                        │   │
│  │   • Security breach                                             │   │
│  │                                                                  │   │
│  │   Action: On-call engineer paged within 15 minutes             │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                             │                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   P2: HIGH (Notify within 1 hour)                              │   │
│  │   ════════════════════════════════════                          │   │
│  │   • Error rate > 1%                                            │   │
│  │   • Latency > 500ms                                            │   │
│  │   • Dependency degraded                                         │   │
│  │                                                                  │   │
│  │   Action: Slack notification, engineer responds in 1 hour       │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                             │                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   P3: MEDIUM (Notify within 24 hours)                         │   │
│  │   ══════════════════════════════════════                       │   │
│  │   • Disk usage > 80%                                           │   │
│  │   • Certificate expiring in 30 days                           │   │
│  │   • Build warnings increasing                                   │   │
│  │                                                                  │   │
│  │   Action: Ticket created, triaged in daily standup              │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                             │                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   P4: LOW (Informational)                                     │   │
│  │   ══════════════════════════════════════                       │   │
│  │   • Deprecations                                               │   │
│  │   • Optimization opportunities                                  │   │
│  │   • Post-incident follow-up                                    │   │
│  │                                                                  │   │
│  │   Action: Review in weekly ops meeting                         │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  GOLDEN SIGNALS (Google SRE):                                           │
│  ────────────────────────────                                           │
│                                                                          │
│  1. LATENCY - How long does it take?                                    │
│     Alert on: p99 > 2x baseline, sustained for 5 minutes              │
│                                                                          │
│  2. TRAFFIC - How many requests?                                       │
│     Alert on: > 2x baseline, or < 50% baseline (sudden drop)          │
│                                                                          │
│  3. ERRORS - Are requests failing?                                     │
│     Alert on: 5xx rate > 0.1%, or any increase in 4xx for 5 minutes  │
│                                                                          │
│  4. SATURATION - Is the system full?                                   │
│     Alert on: CPU > 80%, Memory > 85%, Disk > 90%                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Incident Response

### 5.1 Incident Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INCIDENT RESPONSE LIFECYCLE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐    │
│   │  DETECT  │────▶│ RESPOND  │────▶│ MITIGATE │────▶│ RESOLVE  │    │
│   └──────────┘     └──────────┘     └──────────┘     └──────────┘    │
│        │                │                │                │            │
│        ▼                ▼                ▼                ▼            │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐    │
│   │  Alert   │     │  Triage  │     │  Fix or  │     │ Post-    │    │
│   │  fires   │     │  Assess  │     │  work-   │     │ mortem   │    │
│   │          │     │  Scope   │     │  around  │     │          │    │
│   └──────────┘     └──────────┘     └──────────┘     └──────────┘    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 1: DETECT (Time to Detect - TTD)                                 │
│  ─────────────────────────────────                                     │
│                                                                          │
│  Sources:                                                               │
│  • Automated alerts (thresholds)                                        │
│  • Customer reports                                                    │
│  • On-call engineer noticing                                           │
│  • Monitoring dashboards                                               │
│                                                                          │
│  Target TTD: < 5 minutes for critical issues                           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 2: RESPOND (Time to Respond - TTR start)                        │
│  ─────────────────────────────────────                                 │
│                                                                          │
│  Actions:                                                               │
│  1. Acknowledge alert                                                  │
│  2. Assess severity (SEV1-4)                                          │
│  3. Declare incident (if SEV2+)                                       │
│  4. Join incident Slack channel                                       │
│  5. Assign incident commander                                         │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 3: MITIGATE (Stop bleeding)                                      │
│  ──────────────────────────────                                        │
│                                                                          │
│  Actions:                                                               │
│  1. Identify root cause                                               │
│  2. Apply immediate fix (rollback, scale, disable feature)            │
│  3. Communicate status to stakeholders                                │
│  4. Update status page                                                │
│                                                                          │
│  Priority: Restore service FIRST, debug SECOND                          │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 4: RESOLVE (Full recovery)                                      │
│  ────────────────────────                                              │
│                                                                          │
│  Actions:                                                               │
│  1. Verify fix works                                                   │
│  2. Confirm monitoring looks healthy                                  │
│  3. Update stakeholders                                               │
│  4. Schedule post-mortem (within 48 hours)                            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STEP 5: POST-MORTEM (Learn)                                           │
│  ────────────────────────                                               │
│                                                                          │
│  Template:                                                              │
│  1. Summary (what happened)                                           │
│  2. Impact (who was affected)                                          │
│  3. Root cause (why it happened)                                      │
│  4. Resolution (how it was fixed)                                     │
│  5. Lessons learned (how to prevent recurrence)                       │
│  6. Action items (specific tickets)                                    │
│                                                                          │
│  Key principle: Blameless post-mortem!                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Runbook Template

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RUNBOOK TEMPLATE                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  # Runbook: High Error Rate on API Gateway                              │
│                                                                          │
│  ## Alert Condition                                                     │
│  ─────────────────                                                      │
│  • Alert: Error rate > 1% for 5 minutes                                 │
│  • Metric: sum(rate(http_requests{status=~"5.." })) /                  │
│            sum(rate(http_requests)) > 0.01                            │
│                                                                          │
│  ## Impact                                                              │
│  ───────                                                                │
│  • Users seeing 500 errors when calling API                            │
│  • Affects all endpoints                                               │
│                                                                          │
│  ## Diagnosis Steps                                                    │
│  ─────────────────                                                      │
│                                                                          │
│  1. Check if database is healthy                                       │
│     ```                                                                 │
│     kubectl exec -it postgres-0 -- pg_isready                         │
│     ```                                                                 │
│                                                                          │
│  2. Check recent deployments                                           │
│     ```                                                                 │
│     kubectl rollout history deployment/api-gateway                    │
│     ```                                                                 │
│                                                                          │
│  3. Check error logs                                                   │
│     ```                                                                 │
│     kubectl logs -l app=api-gateway --tail=100 | grep ERROR          │
│     ```                                                                 │
│                                                                          │
│  4. Check database connections                                          │
│     ```                                                                 │
│     psql -c "SELECT count(*) FROM pg_stat_activity"                  │
│     ```                                                                 │
│                                                                          │
│  ## Mitigation Steps                                                    │
│  ─────────────────                                                      │
│                                                                          │
│  OPTION A: Rollback (if recent deployment)                             │
│  ```                                                                    │
│  kubectl rollout undo deployment/api-gateway                         │
│  ```                                                                    │
│                                                                          │
│  OPTION B: Scale up (if resource exhaustion)                          │
│  ```                                                                    │
│  kubectl scale deployment/api-gateway --replicas=10                  │
│  ```                                                                    │
│                                                                          │
│  OPTION C: Restart (if stuck)                                          │
│  ```                                                                    │
│  kubectl rollout restart deployment/api-gateway                      │
│  ```                                                                    │
│                                                                          │
│  ## Escalation                                                         │
│  ────────────                                                           │
│  • If not resolved in 15 minutes: Page on-call manager               │
│  • If database issue: Escalate to DBA team                            │
│                                                                          │
│  ## Related Alerts                                                      │
│  ─────────────────                                                      │
│  • High CPU on database                                                │
│  • Connection pool exhaustion                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Cost Optimization

### 6.1 Cloud Cost Breakdown

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COST OPTIMIZATION FRAMEWORK                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  MONTHLY SPEND BREAKDOWN (EXAMPLE):                                    │
│  ════════════════════════════════                                      │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │   Compute:           ████████████████████░░░░░░░  $45,000 (45%) │   │
│  │   Storage:           ████████████░░░░░░░░░░░░░░░  $25,000 (25%) │   │
│  │   Network:           ██████░░░░░░░░░░░░░░░░░░░░░  $10,000 (10%) │   │
│  │   Database:         ████████░░░░░░░░░░░░░░░░░░░░  $12,000 (12%) │   │
│  │   External Services: ███░░░░░░░░░░░░░░░░░░░░░░░░   $5,000 (5%)  │   │
│  │   Other:            ███░░░░░░░░░░░░░░░░░░░░░░░░░   $3,000 (3%)  │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  COMPUTE OPTIMIZATION:                                                  │
│  ───────────────────                                                   │
│                                                                          │
│  1. Right-sizing                                                       │
│     • Analyze actual CPU/memory usage                                  │
│     • Downsize instances running at <30% utilization                   │
│     • Savings: 20-40%                                                  │
│                                                                          │
│  2. Reserved Instances / Savings Plans                                 │
│     • Commit to 1-3 years for baseline workload                        │
│     • Savings: 30-60%                                                  │
│                                                                          │
│  3. Spot Instances                                                     │
│     • Fault-tolerant workloads (batch, stateless)                      │
│     • Savings: 60-90%                                                  │
│                                                                          │
│  4. Auto-scaling                                                       │
│     • Scale down during off-peak                                       │
│     • Savings: 20-40%                                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  STORAGE OPTIMIZATION:                                                 │
│  ───────────────────                                                   │
│                                                                          │
│  1. Tiered Storage                                                     │
│     • Hot: SSD for recent data                                         │
│     • Cold: Glacier for archives                                       │
│     • Savings: 50-80%                                                  │
│                                                                          │
│  2. Compression                                                        │
│     • Enable for all storage classes                                   │
│     • Savings: 20-40%                                                  │
│                                                                          │
│  3. Lifecycle Policies                                                │
│     • Auto-archive old data                                            │
│     • Delete unused data                                               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  DATABASE OPTIMIZATION:                                                │
│  ─────────────────────                                                  │
│                                                                          │
│  1. Connection Pooling                                                │
│     • Reuse connections, reduce max connections                        │
│                                                                          │
│  2. Read Replicas                                                     │
│     • Offload reads from primary                                       │
│                                                                          │
│  3. Serverless                                                         │
│     • Use Aurora Serverless, Lambda for variable workloads            │
│     • Savings: 30-50% for sporadic workloads                           │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  NETWORK OPTIMIZATION:                                                │
│  ───────────────────                                                   │
│                                                                          │
│  1. CDN Usage                                                          │
│     • Cache static content at edge                                     │
│     • Savings: 30-50% on egress                                        │
│                                                                          │
│  2. Private Subnets                                                   │
│     • Keep services internal, reduce public IPs                       │
│                                                                          │
│  3. Compression                                                        │
│     • Enable for all API responses                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  COST MONITORING:                                                      │
│  ────────────────                                                      │
│                                                                          │
│  • Daily spend tracking by service                                     │
│  • Anomaly alerts on spend                                             │
│  • Monthly cost review meeting                                         │
│  • Engineering incentives tied to cost efficiency                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Performance Tuning Patterns

### 7.1 Database Tuning

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DATABASE PERFORMANCE PATTERNS                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. CONNECTION POOLING                                                 │
│  ─────────────────────                                                 │
│                                                                          │
│  Problem: Creating connections is expensive                            │
│  Solution: Pool connections at application layer                        │
│                                                                          │
│  Configuration:                                                         │
│  • Min connections: 5-10 (keep warm)                                   │
│  • Max connections: Based on DB max + application threads            │
│  • Idle timeout: 5-10 minutes                                         │
│  • Connection lifetime: 30 minutes                                     │
│                                                                          │
│  Tools: PgBouncer, HikariCP, R2DBC                                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  2. INDEXING                                                           │
│  ────────────                                                          │
│                                                                          │
│  The RIGHT way:                                                        │
│  1. Analyze slow query logs                                            │
│  2. EXPLAIN ANALYZE the query                                          │
│  3. Add covering index if needed                                       │
│  4. Test with production-like data                                     │
│                                                                          │
│  The WRONG way:                                                        │
│  • Index every column "just in case"                                   │
│  • Not considering write cost                                           │
│                                                                          │
│  Index Guidelines:                                                     │
│  • B-tree for equality + range                                         │
│  • GIN for full-text search, JSON                                     │
│  • BRIN for time-series (append-only)                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  3. QUERY OPTIMIZATION                                                 │
│  ───────────────────                                                   │
│                                                                          │
│  Common Issues:                                                        │
│  • N+1 queries: Use JOIN or batch fetching                            │
│  • Missing WHERE: Full table scans                                      │
│  • Functions on columns: Can't use index                              │
│  • OR conditions: Split into UNION                                    │
│                                                                          │
│  EXPLAIN ANALYZE patterns:                                              │
│  • Seq Scan: OK for small tables, bad for large                      │
│  • Index Scan: Good for selective queries                             │
│  • Bitmap Heap: Multiple index scans merged                           │
│  • Nested Loop: Expensive for large tables                            │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  4. CACHING STRATEGY                                                  │
│  ───────────────────                                                   │
│                                                                          │
│  Redis patterns:                                                       │
│  • Cache-aside: App checks cache first                                │
│  • Write-through: Update cache on write                               │
│  • Write-behind: Async DB writes                                      │
│  • TTL: Set appropriately per data type                               │
│                                                                          │
│  Cache invalidation:                                                   │
│  • TTL: Simple but stale data possible                                 │
│  • Events: More complex, immediate invalidation                       │
│  • Versioning: Increment version on updates                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Summary Checklist

### 8.1 Production Readiness Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION READINESS CHECKLIST                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  MONITORING:                                                            │
│  ────────────                                                           │
│  □ Dashboards for key metrics (QPS, latency, errors)                   │
│  □ SLIs defined for all critical services                              │
│  □ SLOs defined with error budgets                                     │
│  □ Alerts configured with proper thresholds                            │
│  □ Runbooks for all alerts                                             │
│  □ On-call rotation established                                        │
│                                                                          │
│  LOGGING:                                                               │
│  ────────                                                               │
│  □ Structured logging (JSON)                                           │
│  □ Correlation IDs for request tracing                                 │
│  □ Appropriate log levels (not DEBUG in prod)                          │
│  □ Sensitive data filtered                                             │
│  □ Retention policy defined                                             │
│                                                                          │
│  TRACING:                                                               │
│  ────────                                                               │
│  □ Distributed tracing enabled                                         │
│  □ Service dependency map visible                                       │
│  □ Key operations traced                                               │
│  □ Sampling configured appropriately                                    │
│                                                                          │
│  INCIDENT RESPONSE:                                                    │
│  ─────────────────                                                      │
│  □ Incident commander defined                                          │
│  □ Communication channels established                                   │
│  □ Status page process defined                                          │
│  □ Post-mortem template exists                                          │
│                                                                          │
│  SCALING:                                                               │
│  ─────────                                                              │
│  □ Horizontal scaling defined                                          │
│  □ Auto-scaling configured                                             │
│  □ Capacity planning documented                                        │
│  □ Load testing performed                                              │
│                                                                          │
│  RESILIENCE:                                                           │
│  ──────────                                                             │
│  □ Circuit breakers configured                                         │
│  □ Timeouts defined                                                    │
│  □ Retry policies defined                                              │
│  □ Bulkheads configured                                                │
│  □ Fallback strategies defined                                         │
│                                                                          │
│  SECURITY:                                                              │
│  ────────                                                               │
│  □ TLS enabled                                                         │
│  □ Secrets management (not in code)                                   │
│  □ Network policies defined                                            │
│  □ Audit logging enabled                                                │
│                                                                          │
│  COST:                                                                  │
│  ─────                                                                  │
│  □ Cost monitoring in place                                            │
│  □ Right-sizing performed                                              │
│  □ Reserved instances for baseline                                     │
│  □ Cleanup policies for unused resources                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
