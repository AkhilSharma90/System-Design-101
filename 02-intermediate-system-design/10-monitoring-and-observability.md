# Monitoring and Observability

You can't improve what you can't measure. Monitoring and observability are what separate systems that quietly fail at 3am from systems that alert you before users notice anything is wrong.

---

## Monitoring vs Observability

These terms are often used interchangeably, but there's a meaningful distinction:

**Monitoring** answers: *"Is the system working?"*
- Pre-defined dashboards, alerts, thresholds
- Good for known failure modes
- "CPU is above 90%, alert the team"

**Observability** answers: *"Why is the system not working?"*
- Ability to ask arbitrary questions about system state
- Good for unknown/novel failures
- "I can explore any dimension of data to understand what happened"

A system is observable if you can understand its internal state purely from its external outputs (logs, metrics, traces).

---

## The Three Pillars

```
┌──────────────────────────────────────────────────────────────┐
│                  Observability Pillars                       │
│                                                              │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │
│  │    METRICS    │  │     LOGS      │  │    TRACES     │    │
│  │               │  │               │  │               │    │
│  │ Numeric       │  │ Timestamped   │  │ Request flow  │    │
│  │ aggregations  │  │ text records  │  │ across        │    │
│  │ over time     │  │ of events     │  │ services      │    │
│  │               │  │               │  │               │    │
│  │ "What's       │  │ "What         │  │ "Where is     │    │
│  │  happening?"  │  │  happened?"   │  │  the slowdown?"│   │
│  └───────────────┘  └───────────────┘  └───────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## Pillar 1: Metrics

Metrics are numeric measurements sampled over time. They're efficient, low-overhead, and perfect for alerting and dashboards.

### Metric Types

**Counter:** A value that only goes up. Reset to 0 on restart.
```
http_requests_total{method="GET", status="200"} = 150234
http_requests_total{method="POST", status="500"} = 47
```

**Gauge:** A value that can go up or down.
```
memory_usage_bytes = 4294967296   (4 GB)
active_connections = 342
queue_depth = 1205
```

**Histogram:** Distribution of a value in buckets.
```
http_request_duration_seconds_bucket{le="0.1"} = 45678   ← requests < 100ms
http_request_duration_seconds_bucket{le="0.5"} = 52341   ← requests < 500ms
http_request_duration_seconds_bucket{le="1.0"} = 53201   ← requests < 1s
http_request_duration_seconds_bucket{le="+Inf"} = 53240  ← all requests

→ Calculate p99: the 0.99 × 53240 = 52707th bucket value
```

### Prometheus Architecture

```
                    ┌──────────────┐
                    │   Prometheus │  Scrapes metrics on schedule
                    │   Server     │◀─── /metrics endpoint
                    └──────┬───────┘    on each service
                           │
                    ┌──────▼───────┐
                    │ Alertmanager │  Routes alerts → PagerDuty/Slack
                    └──────────────┘
                           │
                    ┌──────▼───────┐
                    │   Grafana    │  Dashboards and visualization
                    └──────────────┘
```

**PromQL example:**
```promql
# Request rate (per second, last 5 minutes)
rate(http_requests_total[5m])

# p99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Error rate
rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m])
```

---

## Pillar 2: Logs

Logs are time-stamped records of discrete events. They provide rich context for debugging.

### Structured vs Unstructured Logging

```
Unstructured (hard to query):
2024-01-15 10:23:45 ERROR user 42 failed to checkout - card declined

Structured JSON (queryable):
{
  "timestamp": "2024-01-15T10:23:45Z",
  "level": "ERROR",
  "event": "checkout_failed",
  "user_id": 42,
  "reason": "card_declined",
  "amount": 99.99,
  "trace_id": "abc123"
}

→ Can now query: level=ERROR AND user_id=42 AND event=checkout_failed
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

```
Services ──logs──▶ Logstash/Fluentd ──▶ Elasticsearch ──▶ Kibana
(JSON logs)        (parse, enrich,       (index and        (search,
                    aggregate)            store)            dashboards)
```

Modern alternative: **Grafana Loki** — like Prometheus but for logs. Labels-based, much cheaper storage.

### Log Levels

| Level | Use | Example |
|-------|-----|---------|
| DEBUG | Verbose, dev only | "Fetched user from cache" |
| INFO | Normal operations | "User 42 logged in" |
| WARN | Unexpected but handled | "Rate limit approaching for user 42" |
| ERROR | Request failed | "Payment processing failed: timeout" |
| FATAL | Service cannot continue | "Cannot connect to database on startup" |

---

## Pillar 3: Distributed Tracing

A single user request may touch 5+ services. Distributed tracing follows the request across all of them.

```
User Request (trace_id: abc123)

┌──────────────────────────────────────────────────────────┐
│  API Gateway (span 1: 150ms total)                       │
│  ├── User Service (span 2: 20ms)                        │
│  ├── Product Service (span 3: 45ms)                     │
│  │   └── DB Query (span 4: 40ms) ← the slow one!        │
│  ├── Cart Service (span 5: 30ms)                        │
│  └── Recommendation Service (span 6: 55ms)              │
└──────────────────────────────────────────────────────────┘

Timeline:
|──Gateway──|
   |──User──|
             |──────Product──────|
             |──DB Query──────────|  ← HIGH LATENCY
                     |──Cart──|
                         |──Recommend──|
```

Each span has: service name, operation, start time, duration, parent span ID, trace ID.

### OpenTelemetry

The standard for instrumentation — collects traces, metrics, and logs in one SDK:

```
App → OpenTelemetry SDK → OpenTelemetry Collector → Jaeger/Zipkin/Tempo
```

---

## SLIs, SLOs, and SLAs

The vocabulary of reliability:

### SLI — Service Level Indicator
A metric that measures the level of service. The "what we measure."

```
Common SLIs:
- Request success rate: successful_requests / total_requests
- Latency: % of requests completing under 200ms
- Error rate: errors / total_requests
- Availability: uptime / total_time
```

### SLO — Service Level Objective
A target value for an SLI. The "what we aim for."

```
Our SLOs:
- 99.9% of requests succeed
- 95% of requests complete in < 200ms
- 99.9% availability per month (43 minutes downtime allowed)
```

### SLA — Service Level Agreement
A contract with customers specifying SLOs and consequences for missing them. The "what we promise."

```
"If our API availability falls below 99.9% in any calendar month,
 customers receive a 10% service credit."
```

---

## Error Budgets

If your SLO is 99.9% uptime, your **error budget** is 0.1% = 43.8 minutes/month of allowed downtime.

```
Monthly error budget: 43.8 minutes

Week 1: 10 min incident    → 33.8 min remaining
Week 2: 5 min incident     → 28.8 min remaining
Week 3: 30 min incident    → 1.8 min remaining ← ALERT! Budget nearly exhausted
Week 3: Freeze risky deploys until month resets
```

**The power of error budgets:** They create alignment between reliability and feature velocity. If the error budget is healthy, ship features fast. If it's nearly exhausted, prioritize reliability.

---

## The Four Golden Signals

Google's SRE handbook defines four key signals to monitor for any service:

| Signal | What to Measure | Alert When |
|--------|----------------|------------|
| **Latency** | Request duration (p50, p95, p99) | p99 > 1s |
| **Traffic** | Requests per second | Sudden drops or spikes |
| **Errors** | Error rate (5xx, timeouts) | Error rate > 1% |
| **Saturation** | How full is the system? (CPU, memory, queue depth) | CPU > 80%, queue > 1000 |

### The USE Method (for Resources)

For every resource (CPU, memory, disk, network):
- **U**tilization — how busy is the resource? (%)
- **S**aturation — how much is waiting to use the resource?
- **E**rrors — are there errors being returned?

### The RED Method (for Services)

For every service:
- **R**ate — how many requests per second?
- **E**rrors — what percentage are failing?
- **D**uration — how long are requests taking?

---

## Alerting Best Practices

**Alert on symptoms, not causes:**
```
❌ Bad: "CPU is above 90%"    ← cause (may not affect users)
✅ Good: "Error rate > 1%"   ← symptom (users are affected)
✅ Good: "p99 latency > 2s"  ← symptom (users are experiencing this)
```

**Avoid alert fatigue:**
- Every alert should be actionable
- If an alert fires and the on-call engineer can't act on it → remove it
- Use severity levels: page immediately (P1), notify (P2), log only (P3)

**Alert on error budget burn rate:**
```
1h burn rate > 14.4×  → Page immediately  (budget exhausts in < 1 hour)
6h burn rate > 6×     → Page soon         (budget exhausts in < 6 hours)
1d burn rate > 3×     → Notify            (investigate soon)
```

---

## Health Checks

Every service should expose a health endpoint:

```
GET /health → 200 OK (when healthy)
{
  "status": "healthy",
  "checks": {
    "database": { "status": "healthy", "latency_ms": 5 },
    "redis":    { "status": "healthy", "latency_ms": 1 },
    "disk":     { "status": "healthy", "free_gb": 45 }
  },
  "version": "1.4.2",
  "uptime_seconds": 86400
}

GET /health → 503 Service Unavailable (when degraded)
```

**Readiness vs Liveness checks (Kubernetes):**
- **Liveness:** Is the process alive? If no → restart it
- **Readiness:** Is the service ready to serve traffic? If no → remove from load balancer

---

## Real-World Observability Stacks

| Company | Metrics | Logs | Traces | Alerting |
|---------|---------|------|--------|---------|
| Netflix | Atlas | Elasticsearch | Zipkin | PagerDuty |
| Uber | M3 (Prometheus-based) | ELK Stack | Jaeger | PagerDuty |
| Shopify | Prometheus + Datadog | Splunk | Jaeger | PagerDuty |
| Startups | Datadog or Grafana Cloud | Datadog or Loki | Datadog APM | PagerDuty/Opsgenie |

---

## Key Takeaways

1. **Observability = metrics + logs + traces** — each pillar answers different questions
2. **Structured logging enables powerful queries** — use JSON with consistent field names
3. **Distributed tracing shows where latency comes from** — essential for microservices debugging
4. **SLOs create clarity about reliability goals** — error budgets balance velocity and reliability
5. **Alert on user-visible symptoms, not internal causes** — high CPU is only a problem if users notice
6. **The four golden signals are your starting point** — latency, traffic, errors, saturation
7. **Health checks enable automatic traffic management** — fail liveness → restart; fail readiness → stop sending traffic

---

*Previous: [Blob Storage and Object Storage ←](09-blob-storage.md)*

*Completed the Intermediate section! Next: [Advanced System Design →](../03-advanced-system-design/README.md)*
