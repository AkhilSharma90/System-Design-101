---
slug: high-cardinality-data-management
title: High Cardinality Data Management
readTime: 20 min
orderIndex: 4
premium: false
---

# High-Cardinality Data Management (The Curse of Infinite Dimensions)

> Audience: observability engineers dealing with metrics, logs, and traces explosion in high-scale systems.

This article assumes:
- Your metrics have attributes like user_id, request_id, container_id (millions of unique values).
- Traditional time-series databases explode when cardinality increases.
- Query performance degrades from seconds to minutes as cardinality grows.
- Storage costs scale linearly with cardinality - this gets expensive fast.

---

## [CHALLENGE] Challenge: Your metrics database just exploded

### Scenario
You add a new metric: `http_request_duration{user_id="..."}`.

Timeline:
- **Day 1**: 10K unique users, metrics database happy
- **Day 7**: 100K unique users, query latency increases to 5 seconds
- **Day 14**: 1M unique users, Prometheus OOM crashes
- **Day 30**: Database unusable, costs $50K/month

One innocent `user_id` label destroyed your observability.

### Interactive question (pause and think)
What went wrong?

1. Prometheus can't handle scale
2. user_id is high-cardinality (millions of unique values)
3. Needed more memory
4. Should have used a different database

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (2) - high-cardinality kills time-series databases.

More memory (option 3) delays the problem but doesn't solve it. Different database (option 4) might help, but understanding cardinality is key.

### Real-world analogy (library catalog)
Imagine cataloging books:

**Low cardinality**: Genre (fiction, non-fiction, science, history) = 10 categories. Easy to organize and search.

**High cardinality**: ISBN (every book has unique number) = millions of categories. Impossible to efficiently organize and search without specialized indexes.

Metrics databases are optimized for low cardinality (like genres), not high cardinality (like ISBNs).

### Key insight box
> Cardinality is the number of unique combinations of label values. High cardinality (millions+) causes exponential growth in storage and query cost.

### Challenge question
If high-cardinality metrics are so problematic, why do we need them? Can't we just avoid them?

---

## [MENTAL MODEL] Mental model - Cardinality is a combinatorial explosion

### Scenario
You have a metric with three labels:

```prometheus
http_requests_total{
    endpoint="/api/checkout",
    status="200",
    region="us-east"
}
```

### Interactive question (pause and think)
If you have:
- 100 endpoints
- 10 status codes
- 5 regions

How many unique time series can this metric create?

A. 115 (100 + 10 + 5)
B. 500 (100 × 5)
C. 5,000 (100 × 10 × 5)
D. It depends on actual combinations

### Progressive reveal
Answer: C (worst case), D (in practice).

Maximum cardinality = product of all label cardinalities.

### Cardinality mathematics

```text
┌─────────────────────────────────────────────────────────┐
│           CARDINALITY EXPLOSION MATH                    │
└─────────────────────────────────────────────────────────┘

Metric: http_requests_total

Low-cardinality labels:
  endpoint: 100 unique values
  status: 10 unique values
  region: 5 unique values

Max time series: 100 × 10 × 5 = 5,000 series
Storage per series: ~1 KB/hour
Total storage: 5,000 × 1 KB = 5 MB/hour = 3.6 GB/month
Cost: Manageable

Add high-cardinality label:
  user_id: 1,000,000 unique values

Max time series: 100 × 10 × 5 × 1,000,000 = 5 BILLION series
Storage: 5B × 1 KB = 5 TB/hour = 3.6 PB/month
Cost: IMPOSSIBLE

Even if only 1% of combinations exist:
  Actual series: 50 million
  Storage: 50M × 1 KB = 50 GB/hour = 36 TB/month
  Cost: $50K+/month
```

### Why time-series databases struggle

```yaml
time_series_db_design:

  prometheus_storage_model:
    - One series = one unique label combination
    - Each series stored separately (inverted index)
    - Query: scan all series matching labels

  low_cardinality_example:
    metric: http_requests_total{endpoint, status}
    series_count: 1,000
    query: SELECT where endpoint="/api/checkout"
    series_scanned: ~100 (10% of total)
    performance: Fast (milliseconds)

  high_cardinality_example:
    metric: http_requests_total{endpoint, status, user_id}
    series_count: 10,000,000
    query: SELECT where endpoint="/api/checkout"
    series_scanned: ~100,000 (1% of total, but huge absolute number)
    performance: Slow (minutes)
    memory: OOM (can't fit series in memory)

  problem:
    - Index size grows with cardinality
    - Query cost grows with cardinality
    - Memory requirements grow with cardinality
```

### Mental model
Think of cardinality as:

- **Dimensionality curse**: More dimensions = exponentially more space
- **Combinatorial explosion**: Labels multiply, not add
- **Query cost multiplier**: Each label filters, but high-cardinality labels don't filter much

### Key insight box
> Cardinality isn't additive (100 + 1M = 1M), it's multiplicative (100 × 1M = 100M). One high-cardinality label destroys everything.

### Challenge question
You have 5 labels, 4 are low-cardinality (10 values each), 1 is high-cardinality (1M values). Which label causes the problem?

---

## [WARNING] Identifying high-cardinality culprits

### Scenario
Your Prometheus is slow. You suspect high cardinality. How do you find the culprit?

### Cardinality analysis techniques

**Technique 1: Prometheus built-in metrics**
```promql
# Query cardinality per metric
topk(10, count by (__name__) ({__name__=~".+"}))

# Output:
# http_requests_total: 5,234,123 series
# cpu_usage: 1,234 series
# memory_usage: 567 series

# Find high-cardinality labels
count by (__name__, endpoint) (http_requests_total)

# Output shows which labels have most unique values
```

**Technique 2: Prometheus TSDB stats**
```bash
# Query Prometheus API
curl http://localhost:9090/api/v1/status/tsdb

# Response:
{
  "data": {
    "seriesCountByMetricName": [
      {
        "name": "http_requests_total",
        "value": 5234123
      },
      {
        "name": "cpu_usage",
        "value": 1234
      }
    ],
    "labelValueCountByLabelName": [
      {
        "name": "user_id",
        "value": 1000000
      },
      {
        "name": "endpoint",
        "value": 100
      }
    ]
  }
}

# Culprit: user_id label has 1M unique values
```

**Technique 3: Cardinality profiling script**
```python
import requests
import pandas as pd

def analyze_cardinality(prometheus_url):
    # Get all metrics
    response = requests.get(f"{prometheus_url}/api/v1/label/__name__/values")
    metrics = response.json()['data']

    cardinality_report = []

    for metric in metrics:
        # Count series for this metric
        query = f'count({{__name__="{metric}"}})'
        response = requests.get(
            f"{prometheus_url}/api/v1/query",
            params={'query': query}
        )
        series_count = int(response.json()['data']['result'][0]['value'][1])

        # Get labels for this metric
        response = requests.get(f"{prometheus_url}/api/v1/series?match[]={metric}")
        series = response.json()['data']

        # Analyze label cardinality
        label_cardinality = {}
        for s in series:
            for label, value in s.items():
                if label not in label_cardinality:
                    label_cardinality[label] = set()
                label_cardinality[label].add(value)

        cardinality_report.append({
            'metric': metric,
            'total_series': series_count,
            'labels': {k: len(v) for k, v in label_cardinality.items()}
        })

    return pd.DataFrame(cardinality_report).sort_values('total_series', ascending=False)

# Usage
report = analyze_cardinality("http://localhost:9090")
print(report.head(10))

# Output:
#            metric  total_series                        labels
# http_requests_total   5234123  {endpoint: 100, status: 10, user_id: 1000000}
# api_latency           234567   {endpoint: 100, user_id: 50000}
# db_queries            1234     {table: 20, operation: 5}
```

### Common high-cardinality culprits

```yaml
typical_high_cardinality_labels:

  identifiers:
    - user_id (millions of users)
    - request_id (every request unique)
    - trace_id (every trace unique)
    - container_id (ephemeral, millions over time)
    - pod_id (Kubernetes, constantly churning)

  timestamps:
    - timestamp (infinite cardinality)
    - created_at (every second is unique)

  unbounded_strings:
    - error_message (every error slightly different)
    - url (query params make each URL unique)
    - user_agent (thousands of variants)

  ip_addresses:
    - client_ip (millions of users)
    - server_ip (ephemeral cloud IPs)
```

### Key insight box
> High-cardinality labels are usually identifiers (IDs, IPs, timestamps) or unbounded strings. Look for labels with >10K unique values.

### Challenge question
You find that `pod_id` has 100K unique values (Kubernetes pods constantly restart). Should you remove the label or handle it differently?

---

## [DEEP DIVE] Strategies for managing high-cardinality data

### Scenario
You've identified high-cardinality labels. Now what?

You can't just delete them - they provide valuable debugging context.

### Strategy 1: Don't use metrics for high-cardinality data

```yaml
wrong_approach:
  # DON'T DO THIS
  metric: http_request_duration_seconds
  labels:
    user_id: "user_12345"
    request_id: "req_abcdef"

  problem: Creates millions of time series

right_approach:
  # Use metrics for aggregates
  metric: http_request_duration_seconds
  labels:
    endpoint: "/api/checkout"
    status: "200"

  # Use traces for individual requests
  trace:
    span: http_request
    attributes:
      user_id: "user_12345"
      request_id: "req_abcdef"
      duration: 0.234

  # Use logs for debugging
  log:
    message: "Request completed"
    user_id: "user_12345"
    request_id: "req_abcdef"
    duration: 0.234
```

**When to use what:**
```yaml
use_metrics_for:
  - Aggregates (rates, counts, histograms)
  - Low-cardinality dimensions (endpoint, status, region)
  - Dashboards and alerts
  - Long-term trends

use_traces_for:
  - Individual request debugging
  - End-to-end latency breakdown
  - High-cardinality context (user_id, request_id)
  - Root cause analysis

use_logs_for:
  - Specific events
  - Error messages
  - Debugging context
  - Compliance/audit trail
```

### Strategy 2: Aggregate at write time

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
)

// BAD: High-cardinality metric
var badCounter = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
    },
    []string{"endpoint", "status", "user_id"}, // user_id = high cardinality!
)

// GOOD: Low-cardinality metric
var goodCounter = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
    },
    []string{"endpoint", "status"}, // Only low-cardinality labels
)

// Separate histogram for per-user analysis (sampled)
var userLatencyHistogram = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "http_request_duration_seconds_sampled",
        Buckets: prometheus.DefBuckets,
    },
    []string{"endpoint", "user_tier"}, // user_tier (not user_id!)
)

func RecordRequest(endpoint, status, userID string, duration float64) {
    // Always record aggregate
    goodCounter.WithLabelValues(endpoint, status).Inc()

    // Sample high-cardinality data (1% of requests)
    if shouldSample(userID) {
        userTier := getUserTier(userID) // "free", "paid", "enterprise"
        userLatencyHistogram.WithLabelValues(endpoint, userTier).Observe(duration)
    }
}

func shouldSample(userID string) bool {
    // Hash-based sampling (deterministic per user)
    hash := hashUserID(userID)
    return hash%100 < 1 // 1% sample rate
}
```

### Strategy 3: Use exemplars (metrics + traces bridge)

```yaml
exemplars:
  concept: Attach trace IDs to metric samples

  example:
    metric: http_request_duration_seconds
    value: 2.345
    exemplar:
      trace_id: "abc123def456"
      span_id: "789ghi"

  workflow:
    1. User sees slow p99 latency in metrics
    2. Clicks on exemplar
    3. Jumps directly to trace showing why that request was slow

  benefit:
    - Metrics for aggregate analysis (cheap, low cardinality)
    - Traces for individual debugging (expensive, high cardinality)
    - Exemplars bridge the two
```

### Strategy 4: Cardinality-aware databases

```yaml
specialized_databases:

  prometheus:
    cardinality_limit: ~1M series (practical)
    best_for: Low-cardinality metrics

  victoriametrics:
    cardinality_limit: ~10M series
    optimizations: Better compression, faster queries
    best_for: Medium-cardinality metrics

  m3db:
    cardinality_limit: ~100M series
    optimizations: Distributed, sharded storage
    best_for: High-cardinality metrics

  clickhouse:
    cardinality_limit: Billions of series
    tradeoff: Query latency higher, not real-time
    best_for: Analytical queries on high-cardinality data
```

### Strategy 5: Label transformation/aggregation

```go
package transform

import (
    "regexp"
    "strings"
)

// Transform high-cardinality labels to low-cardinality

func TransformURL(url string) string {
    // BAD: url="/api/users/12345/orders/67890"
    // Cardinality: millions of unique URLs

    // GOOD: url="/api/users/:id/orders/:id"
    // Cardinality: ~100 unique endpoints

    re := regexp.MustCompile(`/\d+`)
    return re.ReplaceAllString(url, "/:id")
}

func TransformUserAgent(userAgent string) string {
    // BAD: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36..."
    // Cardinality: thousands of unique strings

    // GOOD: "Chrome", "Firefox", "Safari", "Other"
    // Cardinality: ~10

    if strings.Contains(userAgent, "Chrome") {
        return "Chrome"
    } else if strings.Contains(userAgent, "Firefox") {
        return "Firefox"
    } else if strings.Contains(userAgent, "Safari") {
        return "Safari"
    }
    return "Other"
}

func TransformErrorMessage(err string) string {
    // BAD: "Connection timeout to host 192.168.1.45 after 30.234 seconds"
    // Cardinality: infinite (IPs, timestamps)

    // GOOD: "connection_timeout"
    // Cardinality: ~50 error types

    if strings.Contains(err, "timeout") {
        return "connection_timeout"
    } else if strings.Contains(err, "refused") {
        return "connection_refused"
    }
    return "unknown_error"
}
```

### Key insight box
> Managing high cardinality requires multiple strategies: use the right tool (metrics vs traces), aggregate, sample, or transform labels to reduce cardinality.

### Challenge question
Your CEO wants a dashboard showing "response time per customer." How do you build this without exploding cardinality?

---

## [PUZZLE] The sampling dilemma - keeping enough data

### Scenario
You decide to sample high-cardinality metrics: keep 1% of user_id values.

Problem: Enterprise customer (0.01% of users) generates error. Their data wasn't sampled. You can't debug.

### Think about it
Random sampling might miss important users. How do you sample intelligently?

### Interactive question (pause and think)
Which sampling strategy catches the most important data?

A. Random sampling (1% of all users)
B. Stratified sampling (1% per user tier)
C. Always keep errors + sample healthy traffic
D. Dynamic sampling based on anomalies

### Progressive reveal
Answer: C or D, depending on use case.

Random sampling (A) misses rare but important events. Stratified (B) is better but still probabilistic.

### Intelligent sampling strategies

**Strategy 1: Error-weighted sampling**
```go
package sampling

func ShouldSampleMetric(userID string, isError bool) bool {
    if isError {
        return true // Always keep errors (100%)
    }

    // Sample healthy traffic (1%)
    hash := hashUserID(userID)
    return hash%100 < 1
}
```

**Strategy 2: Tier-based sampling**
```go
func ShouldSampleByTier(userID string) bool {
    tier := getUserTier(userID)

    switch tier {
    case "enterprise":
        return true // 100% of enterprise users
    case "paid":
        return hash(userID)%10 < 1 // 10% of paid users
    case "free":
        return hash(userID)%100 < 1 // 1% of free users
    default:
        return false
    }
}
```

**Strategy 3: Reservoir sampling (keep diverse sample)**
```go
type ReservoirSampler struct {
    reservoir []string
    size      int
    count     int
}

func NewReservoirSampler(size int) *ReservoirSampler {
    return &ReservoirSampler{
        reservoir: make([]string, 0, size),
        size:      size,
    }
}

func (rs *ReservoirSampler) Add(userID string) bool {
    rs.count++

    if len(rs.reservoir) < rs.size {
        rs.reservoir = append(rs.reservoir, userID)
        return true
    }

    // Replace random element with probability size/count
    j := rand.Intn(rs.count)
    if j < rs.size {
        rs.reservoir[j] = userID
        return true
    }

    return false
}

// Result: Uniform sample of users, updated as new users arrive
```

**Strategy 4: Anomaly-based sampling**
```go
func ShouldSampleAnomaly(userID string, latency float64) bool {
    // Get historical p99 latency for this endpoint
    p99 := getHistoricalP99()

    // If this request is anomalous (>3x p99), always keep it
    if latency > 3*p99 {
        return true
    }

    // Otherwise, sample normally
    return hash(userID)%100 < 1
}
```

### Combining strategies

```go
type SmartSampler struct {
    errorSampler    *AlwaysKeepErrorsSampler
    tierSampler     *TierBasedSampler
    anomalySampler  *AnomalyBasedSampler
    randomSampler   *RandomSampler
}

func (ss *SmartSampler) ShouldSample(req *Request) (bool, string) {
    // Priority 1: Always keep errors
    if req.IsError {
        return true, "error"
    }

    // Priority 2: Always keep enterprise users
    if req.UserTier == "enterprise" {
        return true, "enterprise_tier"
    }

    // Priority 3: Keep anomalies
    if req.Latency > 3*ss.anomalySampler.GetP99() {
        return true, "anomaly"
    }

    // Priority 4: Probabilistic sampling (1%)
    if ss.randomSampler.Sample(req.UserID) {
        return true, "random"
    }

    return false, "dropped"
}
```

### Key insight box
> Intelligent sampling keeps important data (errors, anomalies, VIPs) at 100% while sampling less important data. Don't rely on pure random sampling.

### Challenge question
You sample 1% of users but need to report "accurate total request count." How do you estimate the true count from sampled data?

---

## [WARNING] Query performance degradation patterns

### Scenario
Your metrics database has high cardinality. Queries that took seconds now take minutes.

What's happening under the hood?

### Performance degradation mechanics

```yaml
low_cardinality_query:
  metric: http_requests_total{endpoint, status}
  total_series: 1,000

  query: rate(http_requests_total{endpoint="/api/checkout"}[5m])
  series_matched: 10 (1% of total)
  data_points: 10 series × 300 points = 3,000 points
  query_time: 50ms

high_cardinality_query:
  metric: http_requests_total{endpoint, status, user_id}
  total_series: 10,000,000

  query: rate(http_requests_total{endpoint="/api/checkout"}[5m])
  series_matched: 100,000 (1% of total, but huge absolute number)
  data_points: 100,000 series × 300 points = 30,000,000 points
  query_time: 60 seconds (1200x slower!)
  memory: 2 GB (OOM risk)
```

### Query optimization techniques

**Technique 1: Pre-aggregation (recording rules)**
```yaml
# Prometheus recording rule
groups:
  - name: http_aggregates
    interval: 30s
    rules:
      # Pre-aggregate: remove high-cardinality labels
      - record: http_requests_total:rate5m
        expr: |
          sum by (endpoint, status) (
            rate(http_requests_total[5m])
          )

      # Now query the pre-aggregated metric (low cardinality)
      # Instead of querying the raw high-cardinality metric
```

**Technique 2: Query result caching**
```yaml
query_caching:
  cache: Memcached or Redis
  ttl: 60 seconds

  flow:
    1. Check cache for query result
    2. If hit: return cached result
    3. If miss: execute query, cache result

  benefit:
    - Repeated dashboard queries cached
    - Reduces load on metrics database
```

**Technique 3: Downsampling**
```yaml
downsampling:
  raw_data: 10-second resolution, 7 days retention
  1m_aggregates: 1-minute resolution, 30 days retention
  5m_aggregates: 5-minute resolution, 1 year retention

  query_strategy:
    - Last 24 hours: query raw data
    - Last 7 days: query 1m aggregates
    - Last year: query 5m aggregates

  benefit:
    - Long-term queries faster (fewer data points)
    - Storage reduced (discard old high-res data)
```

**Technique 4: Cardinality limits**
```yaml
# Prometheus configuration
global:
  metric_relabel_configs:
    - source_labels: [__name__]
      regex: 'http_requests_total'
      action: keep
      # Only keep metrics matching regex

    - source_labels: [user_id]
      action: drop
      # Drop high-cardinality label entirely

series_limit: 1000000
  # Hard limit on total series
  # Prevents cardinality explosion
```

### Key insight box
> High-cardinality degrades query performance exponentially. Mitigate with pre-aggregation, caching, downsampling, and cardinality limits.

### Challenge question
Your dashboard queries 100 high-cardinality metrics simultaneously. How do you make it load in < 5 seconds?

---

## [SYNTHESIS] Final synthesis - Design a high-cardinality observability system

### Synthesis challenge
You're the observability architect for a multi-tenant SaaS platform.

Requirements:
- 1 million users across 10K organizations
- Users want "per-user dashboards" showing their activity
- Need to debug individual user issues
- Compliance: 1-year data retention
- Budget: $30K/month for observability

Constraints:
- Current: Prometheus with user_id labels (10M series, constantly OOM)
- Query latency: 2+ minutes for dashboards
- Team: frustrated, want to remove observability entirely

### Your tasks (pause and think)
1. Analyze current cardinality problem
2. Design metrics strategy (what goes in metrics?)
3. Design traces/logs strategy (what doesn't go in metrics?)
4. Choose appropriate databases
5. Implement sampling strategy
6. Optimize query performance

Write down your architecture.

### Progressive reveal (one possible solution)

**1. Cardinality analysis:**
```yaml
current_problem:

  problematic_metrics:
    http_requests_total:
      labels: {endpoint, status, user_id, org_id}
      cardinality: 100 endpoints × 10 statuses × 1M users = 1B potential series
      actual_series: 10M (active users)
      problem: user_id and org_id are high-cardinality

    api_latency_seconds:
      labels: {endpoint, user_id}
      cardinality: 100M potential series
      actual_series: 5M

  total_series: 15M+ active series
  prometheus_limit: ~1M series (practical limit)
  conclusion: 15x over capacity, unsustainable
```

**2. Metrics strategy (low-cardinality only):**
```yaml
metrics_design:

  keep_in_metrics:
    # Aggregate metrics (no user_id)
    http_requests_total:
      labels: {endpoint, status, org_tier}
      # org_tier = "free", "paid", "enterprise" (3 values)
      cardinality: 100 × 10 × 3 = 3,000 series ✓

    api_latency_seconds:
      labels: {endpoint, status}
      cardinality: 100 × 10 = 1,000 series ✓

    # Sampled per-user metrics (1% sample)
    user_request_count_sampled:
      labels: {org_tier, user_cohort}
      # user_cohort = hash(user_id) % 100 (100 buckets)
      cardinality: 3 × 100 = 300 series ✓
      sample_rate: 1% (deterministic per user)

  total_metrics_series: ~5,000 series (sustainable!)
```

**3. Traces/logs strategy (high-cardinality):**
```yaml
move_to_traces:

  per_user_debugging:
    tool: Distributed tracing (Tempo)
    data:
      - Every request has trace with user_id, org_id
      - Span attributes include all high-cardinality data
      - Tail sampling keeps errors + slow requests

    query:
      - User reports issue
      - Query: "Find traces for user_id=12345 in last hour"
      - See individual requests with full context

    cardinality: No limit (traces are documents, not time series)

  per_user_dashboards:
    implementation:
      - Pre-aggregate user stats in separate database (ClickHouse)
      - Daily batch job: sum(requests), avg(latency) per user
      - Users query their own aggregated stats

    cardinality: 1M users × 365 days = 365M rows (manageable)
    storage: Column-store optimized for analytics
```

**4. Database choices:**
```yaml
multi_database_architecture:

  prometheus:
    purpose: Real-time metrics (low-cardinality)
    retention: 7 days
    series: ~5,000
    cost: $1K/month (self-hosted)

  tempo:
    purpose: Distributed traces (high-cardinality per-request)
    retention: 30 days
    traces: ~100M/day (tail sampled)
    cost: $10K/month (S3 storage + compute)

  clickhouse:
    purpose: User analytics (pre-aggregated)
    retention: 1 year
    rows: ~365M (1M users × 365 days)
    cost: $8K/month (managed service)

  loki:
    purpose: Logs (unstructured, high-cardinality)
    retention: 30 days
    logs: ~1TB/day
    cost: $5K/month (S3 storage)

  total_cost: $24K/month (under $30K budget)
```

**5. Sampling strategy:**
```yaml
sampling_approach:

  metrics_sampling:
    aggregate_metrics: 100% (no sampling, low cardinality)
    per_user_metrics: 1% deterministic sample

  trace_sampling:
    sdk_layer: 10% head sampling (reduce network)
    collector_layer: Tail sampling
      - 100% of errors
      - 100% of requests >2s
      - 1% of healthy requests
    final_rate: ~2% of original traffic

  logs_sampling:
    error_logs: 100% (always keep)
    info_logs: 10% (sampled)
    debug_logs: 1% (heavily sampled)
```

**6. Query performance optimization:**
```yaml
performance_improvements:

  prometheus:
    - Recording rules for common queries
    - Pre-aggregate by org_tier every 1m
    - Cache dashboard queries (60s TTL)
    - Query result <1s (from 2+ minutes)

  tempo:
    - Index by user_id, org_id, trace_id
    - Query: "traces for user_id=X" in <5s
    - Exemplars link metrics → traces

  clickhouse:
    - Partition by date (efficient time-range queries)
    - Materialize user_id → org_id mapping
    - Pre-aggregate daily stats
    - Query: "user dashboard" in <2s

  unified_query:
    - Grafana dashboard combines:
      - Prometheus: overall system metrics
      - ClickHouse: per-user aggregates
      - Tempo: drill-down to individual traces
```

### Key insight box
> High-cardinality requires multi-database architecture: metrics for aggregates, traces for per-request debugging, analytics DB for pre-aggregated user stats.

### Final challenge question
After implementing this architecture, your COO asks: "Why do we need 4 databases? Can't we just use one?" How do you explain the trade-offs?

---

## Appendix: Quick checklist (printable)

**Cardinality audit:**
- [ ] List all metrics and their labels
- [ ] Count unique values per label (identify high-cardinality)
- [ ] Calculate max cardinality (multiply label cardinalities)
- [ ] Identify labels with >10K unique values
- [ ] Flag metrics with >100K total series

**Cardinality reduction:**
- [ ] Remove high-cardinality labels (user_id, request_id, etc.)
- [ ] Transform labels (normalize URLs, aggregate errors)
- [ ] Use low-cardinality aggregates (user_tier not user_id)
- [ ] Move high-cardinality data to traces/logs
- [ ] Implement sampling for necessary high-cardinality metrics

**Database selection:**
- [ ] Prometheus/VictoriaMetrics: low-cardinality metrics
- [ ] Tempo/Jaeger: high-cardinality traces
- [ ] ClickHouse/BigQuery: analytical queries on high-cardinality
- [ ] Loki/Elasticsearch: logs
- [ ] Use exemplars to bridge metrics and traces

**Query optimization:**
- [ ] Create recording rules (pre-aggregation)
- [ ] Implement query caching (reduce repeated queries)
- [ ] Downsample old data (reduce resolution over time)
- [ ] Set cardinality limits (prevent explosion)
- [ ] Monitor query latency (alert on degradation)

**Sampling strategy:**
- [ ] Define sampling rate per metric
- [ ] Use intelligent sampling (errors, anomalies, VIPs)
- [ ] Implement deterministic sampling (consistent per entity)
- [ ] Document what's sampled and what's not
- [ ] Validate sample representativeness

**Operational:**
- [ ] Monitor cardinality growth (trend over time)
- [ ] Alert on cardinality spikes (new labels added)
- [ ] Review metrics quarterly (remove unused)
- [ ] Educate team on cardinality impact
- [ ] Document cardinality guidelines

**Red flags:**
- [ ] Metrics with >1M series
- [ ] Labels with >100K unique values
- [ ] Query latency >10s
- [ ] Prometheus OOM crashes
- [ ] Storage costs growing >50%/month
- [ ] New labels added without review
