---
slug: tail-based-sampling
title: Tail Based Sampling
readTime: 20 min
orderIndex: 3
premium: false
---



# Tail-Based Sampling (Smart Trace Collection at Scale)

> Audience: observability engineers and SREs managing distributed tracing in high-throughput production systems.

This article assumes:
- Your system generates millions of traces per second - you can't store them all.
- The most important traces (failures, slow requests) are rare but critical.
- Head-based sampling makes blind decisions before seeing the full trace.
- Storage costs and query performance matter at scale.

---

## [CHALLENGE] Challenge: Your sampling strategy throws away the evidence

### Scenario
Your distributed tracing captures 1% of requests (head-based sampling).

Then an incident happens:
- Customer reports: "Checkout failed at 2:47 PM"
- You check traces: Nothing. The failed request wasn't in your 1% sample
- Support: "This is the 5th complaint today about checkout failures"
- You check error logs: Errors are there, but no traces to debug context

You're flying blind because your sampling threw away the evidence.

### Interactive question (pause and think)
What's wrong with "sample 1% of all requests randomly"?

1. 1% is too low (should sample more)
2. Random sampling treats all requests equally
3. You're sampling before you know which traces matter
4. All of the above

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (3) is the core problem, which leads to (2).

Random head-based sampling decides at the START of a request whether to trace it. But you don't know if the request will fail or be slow until it COMPLETES.

### Real-world analogy (security cameras)
Imagine a bank with security cameras that randomly record 1% of the day:

**Head-based sampling**: Decide at midnight which 1% of today to record. Might miss the robbery at 2 PM.

**Tail-based sampling**: Record everything temporarily, then at end of day, keep only footage with interesting events (robbery, suspicious activity). Delete boring footage.

### Key insight box
> Tail-based sampling makes sampling decisions AFTER seeing the complete trace, keeping important traces (errors, slow requests) while discarding boring ones.

### Challenge question
If you keep 100% of traces temporarily before sampling, where do you store them? How long can you afford to keep them?

---

## [MENTAL MODEL] Mental model - Sampling is an optimization, not a feature

### Scenario
Your VP asks: "Why do we sample traces at all? Just store everything."

You explain: "We generate 10 million traces/day. That's 50 TB/day. That's $500K/month storage."

### Interactive question (pause and think)
Why can't we just store all traces?

A. Storage costs too much
B. Query performance degrades with more data
C. Most traces are uninteresting (successful, fast requests)
D. All of the above

### Progressive reveal
Answer: D.

But the real question is: Can we keep the important traces and drop the boring ones?

### Mental model
Think of trace sampling as:

- **Signal vs noise problem**: Errors are signal, successful requests are (mostly) noise
- **Compression problem**: Keep data with high information content
- **Economic optimization**: Maximize observability value per dollar spent

The goal: Achieve 99% of debugging value with 1-10% of the data.

### Sampling strategies comparison

```text
┌─────────────────────────────────────────────────────────┐
│         SAMPLING STRATEGY COMPARISON                    │
└─────────────────────────────────────────────────────────┘

HEAD-BASED SAMPLING:
  Decision point: Start of request
  Information: Only knows: endpoint, user ID, nothing else
  Algorithm: Sample X% randomly

  Pros: Simple, low memory overhead
  Cons: Blindly discards important traces

  Example:
    10M traces/day → sample 1% → keep 100K traces
    But 10K errors → only 100 errors sampled (99% of errors lost!)

TAIL-BASED SAMPLING:
  Decision point: End of request (after complete)
  Information: Knows: duration, status, errors, all span data
  Algorithm: Keep if error OR slow OR interesting

  Pros: Keeps all important traces
  Cons: Higher memory overhead (buffer traces temporarily)

  Example:
    10M traces/day → 10K errors + 50K slow + 40K random = 100K traces
    Result: Keep 100% of errors, 100% of slow, 0.4% of boring

ADAPTIVE SAMPLING:
  Decision point: Continuous (adjusts over time)
  Information: Current error rates, latency distribution
  Algorithm: Sample more when error rate high, less when low

  Pros: Automatically adjusts to conditions
  Cons: Complex to implement correctly
```

### Key insight box
> Head-based sampling is economically efficient but informationally wasteful. Tail-based sampling inverts the trade-off.

### Challenge question
If tail-based sampling is better, why does anyone use head-based sampling?

---

## [WARNING] Understanding the tail-based sampling architecture

### Scenario
You decide to implement tail-based sampling. Where does the logic run?

### Architecture options

**Option 1: Application-side tail sampling (doesn't work)**
```yaml
why_it_fails:
  problem: Application only sees its own spans, not complete trace

  example:
    request_flow: Frontend → API → Database

    frontend_span: 100ms, success
    api_span: 50ms, success
    database_span: 10 seconds, ERROR

    frontend_decision: "This span is fine, don't sample"
    result: Trace discarded, even though DB failed

  conclusion: Can't make tail decision without seeing full trace
```

**Option 2: Collector-side tail sampling (standard approach)**
```yaml
architecture:

  applications:
    - Generate spans (100% of requests)
    - Send to OpenTelemetry Collector

  collector_pipeline:
    1. Receive all spans
    2. Group spans by trace_id (assemble complete trace)
    3. Wait for trace to complete (or timeout)
    4. Evaluate sampling policy
    5. Keep or discard entire trace
    6. Forward kept traces to backend (Jaeger, Tempo, etc.)

  buffering:
    - Must buffer traces in memory until decision made
    - Typical buffer: 10-30 seconds
    - Memory usage: O(traces_in_flight × avg_trace_size)
```

### Visual: tail-based sampling pipeline

```text
┌─────────────────────────────────────────────────────────┐
│       TAIL-BASED SAMPLING ARCHITECTURE                  │
└─────────────────────────────────────────────────────────┘

┌──────────┐  ┌──────────┐  ┌──────────┐
│ Service  │  │ Service  │  │ Service  │
│    A     │  │    B     │  │    C     │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │ spans       │ spans       │ spans
     │ (100%)      │ (100%)      │ (100%)
     └─────────────┴─────────────┘
                   │
                   ▼
          ┌────────────────┐
          │  OTel Collector│
          │  (Tail Sampler)│
          └────────┬───────┘
                   │
         ┌─────────┴──────────┐
         │   BUFFERING        │
         │ trace_id → [spans] │
         │                    │
         │ abc123 → [s1,s2,s3]│ ◄─┐
         │ def456 → [s4,s5]   │   │ Wait for
         │ ghi789 → [s6,s7,s8]│   │ complete trace
         └─────────┬──────────┘   │ or timeout
                   │                │
         ┌─────────▼──────────┐ ◄──┘
         │  DECISION ENGINE   │
         │                    │
         │  IF error:   KEEP  │
         │  IF slow:    KEEP  │
         │  IF random:  KEEP  │
         │  ELSE:       DROP  │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │   KEPT TRACES      │
         │   (1-10% of total) │
         └─────────┬──────────┘
                   │
                   ▼
          ┌────────────────┐
          │   Backend      │
          │ (Jaeger/Tempo) │
          └────────────────┘

Key metrics:
  - Traces received: 10M/day
  - Buffered in memory: 1M traces (30 sec window)
  - Sampling decision: ~300K traces/min
  - Kept traces: 1M/day (10% sample rate)
  - Memory usage: ~50 GB (1M traces × 50 KB avg)
```

### Memory management challenges

```go
package tailsampling

import (
    "sync"
    "time"
)

type TraceBuffer struct {
    traces map[string]*Trace
    mu     sync.RWMutex

    maxAge      time.Duration
    maxSize     int
    evictionTicker *time.Ticker
}

type Trace struct {
    TraceID      string
    Spans        []*Span
    FirstSeen    time.Time
    LastSeen     time.Time
    Complete     bool
}

func NewTraceBuffer(maxAge time.Duration, maxSize int) *TraceBuffer {
    tb := &TraceBuffer{
        traces:     make(map[string]*Trace),
        maxAge:     maxAge,
        maxSize:    maxSize,
        evictionTicker: time.NewTicker(1 * time.Second),
    }

    go tb.evictionLoop()
    return tb
}

func (tb *TraceBuffer) AddSpan(span *Span) {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    trace, exists := tb.traces[span.TraceID]
    if !exists {
        trace = &Trace{
            TraceID:   span.TraceID,
            Spans:     []*Span{},
            FirstSeen: time.Now(),
        }
        tb.traces[span.TraceID] = trace
    }

    trace.Spans = append(trace.Spans, span)
    trace.LastSeen = time.Now()

    // Check if trace is complete
    if tb.isTraceComplete(trace) {
        trace.Complete = true
        tb.evaluateAndForward(trace)
        delete(tb.traces, span.TraceID)
    }
}

func (tb *TraceBuffer) evictionLoop() {
    for range tb.evictionTicker.C {
        tb.mu.Lock()

        now := time.Now()
        for traceID, trace := range tb.traces {
            age := now.Sub(trace.FirstSeen)

            // Evict old traces (timeout)
            if age > tb.maxAge {
                tb.evaluateAndForward(trace)
                delete(tb.traces, traceID)
            }
        }

        // Evict if buffer too large (memory pressure)
        if len(tb.traces) > tb.maxSize {
            tb.evictOldest()
        }

        tb.mu.Unlock()
    }
}

func (tb *TraceBuffer) isTraceComplete(trace *Trace) bool {
    // Heuristics for trace completion:
    // 1. Has root span
    // 2. All referenced span IDs are present
    // 3. No new spans for X seconds

    hasRoot := false
    spanIDs := make(map[string]bool)
    referencedIDs := make(map[string]bool)

    for _, span := range trace.Spans {
        spanIDs[span.SpanID] = true
        if span.ParentSpanID == "" {
            hasRoot = true
        } else {
            referencedIDs[span.ParentSpanID] = true
        }
    }

    // Check all parent references are resolved
    for refID := range referencedIDs {
        if !spanIDs[refID] && refID != "" {
            return false // Missing parent span
        }
    }

    return hasRoot
}
```

### Key insight box
> Tail-based sampling requires buffering complete traces in memory, which means memory usage scales with request rate and trace duration.

### Challenge question
What happens if a trace never completes (missing spans due to network issues)? How long do you wait before making a decision?

---

## [DEEP DIVE] Sampling policies - what to keep, what to discard

### Scenario
You have 10 million traces buffered. Which ones do you keep?

### Core sampling policies

**Policy 1: Always keep errors**
```yaml
policy: always_keep_errors
rule: IF any_span.status == ERROR THEN keep_trace

rationale:
  - Errors are rare (typically <1% of requests)
  - Errors are always interesting for debugging
  - Cost: minimal (keep ~100K traces if 1% error rate)

example:
  trace_abc123:
    spans:
      - frontend: success
      - api: success
      - database: ERROR (timeout)
    decision: KEEP (database error)
```

**Policy 2: Always keep slow requests**
```yaml
policy: always_keep_slow
rule: IF trace_duration > p99_latency THEN keep_trace

rationale:
  - Slow requests indicate performance problems
  - p99 threshold captures tail latency issues
  - Cost: keep ~1% of traces (by definition of p99)

example:
  p99_latency: 500ms

  trace_def456:
    duration: 3.2 seconds
    decision: KEEP (way above p99)

  trace_ghi789:
    duration: 450ms
    decision: DROP (below p99, not interesting)
```

**Policy 3: Probabilistic sampling for healthy traces**
```yaml
policy: probabilistic_sampling_healthy
rule: IF trace_is_healthy THEN sample_at_rate(1%)

rationale:
  - Keep some successful traces (baseline for comparison)
  - 1% of healthy traffic is statistically significant
  - Allows debugging "why is this slow compared to normal?"

example:
  10M traces, 9M healthy:
    - Keep 100% of errors (1M traces)
    - Keep 100% of slow (500K traces)
    - Keep 1% of healthy (90K traces)
    - Total kept: 1.59M traces (15.9% sample rate)
```

**Policy 4: Rate limiting per attribute**
```yaml
policy: rate_limit_per_endpoint
rule: Keep max N traces per endpoint per minute

rationale:
  - Prevent single hot endpoint from dominating storage
  - Ensure diverse trace coverage
  - Useful for multi-tenant systems

example:
  endpoints:
    /api/checkout: 1000 traces/min max
    /api/search: 500 traces/min max
    /api/healthcheck: 10 traces/min max (mostly noise)

  result:
    - Hot endpoints don't crowd out rare endpoints
    - /healthcheck doesn't waste storage
```

**Policy 5: String attribute matching**
```yaml
policy: keep_by_attribute
rules:
  - IF user_id == "test-user" THEN keep_trace (testing)
  - IF request_header.debug == "true" THEN keep_trace
  - IF span.name CONTAINS "critical_path" THEN keep_trace

use_cases:
  - Debug specific user's requests
  - Trace feature flag rollouts
  - Monitor critical business transactions
```

### Composite policy example

```yaml
tail_sampling_policy:
  # Ordered policies (first match wins)

  1_always_keep_errors:
    condition: any_span.status == ERROR
    sample_rate: 100%

  2_always_keep_slow:
    condition: trace.duration > 2 seconds
    sample_rate: 100%

  3_always_keep_specific_users:
    condition: user.tier == "enterprise"
    sample_rate: 10%

  4_rate_limit_per_endpoint:
    condition: true (applies to all)
    limits:
      - endpoint: /api/checkout
        max_traces_per_minute: 1000
      - endpoint: /api/*
        max_traces_per_minute: 100

  5_probabilistic_baseline:
    condition: true (catch-all)
    sample_rate: 0.5%

# Estimated retention:
# - 1M errors (100%)
# - 500K slow (100%)
# - 50K enterprise (10% of 500K)
# - 100K rate-limited (varies)
# - 40K probabilistic (0.5% of 8M remaining)
# Total: ~1.7M / 10M = 17% overall sample rate
```

### Implementation

```go
package policy

import (
    "time"
)

type SamplingPolicy interface {
    ShouldSample(trace *Trace) (bool, string)
}

// Policy 1: Always keep errors
type AlwaysKeepErrorsPolicy struct{}

func (p *AlwaysKeepErrorsPolicy) ShouldSample(trace *Trace) (bool, string) {
    for _, span := range trace.Spans {
        if span.Status == StatusError {
            return true, "error_detected"
        }
    }
    return false, ""
}

// Policy 2: Keep slow requests
type KeepSlowRequestsPolicy struct {
    Threshold time.Duration
}

func (p *KeepSlowRequestsPolicy) ShouldSample(trace *Trace) (bool, string) {
    duration := trace.EndTime.Sub(trace.StartTime)
    if duration > p.Threshold {
        return true, "slow_request"
    }
    return false, ""
}

// Policy 3: Probabilistic sampling
type ProbabilisticPolicy struct {
    SampleRate float64 // 0.0 to 1.0
}

func (p *ProbabilisticPolicy) ShouldSample(trace *Trace) (bool, string) {
    // Deterministic hash-based sampling
    hash := hashTraceID(trace.TraceID)
    threshold := uint64(p.SampleRate * float64(^uint64(0)))

    if hash < threshold {
        return true, "probabilistic"
    }
    return false, ""
}

// Policy 4: Rate limiting
type RateLimitPolicy struct {
    MaxTracesPerMinute int
    window             *RateLimitWindow
}

func (p *RateLimitPolicy) ShouldSample(trace *Trace) (bool, string) {
    if p.window.TryAcquire() {
        return true, "rate_limit_available"
    }
    return false, ""
}

// Composite policy evaluator
type CompositePolicy struct {
    Policies []SamplingPolicy
}

func (cp *CompositePolicy) Evaluate(trace *Trace) (bool, string) {
    for _, policy := range cp.Policies {
        if shouldSample, reason := policy.ShouldSample(trace); shouldSample {
            return true, reason
        }
    }
    return false, "no_policy_matched"
}

// Usage
func main() {
    policy := &CompositePolicy{
        Policies: []SamplingPolicy{
            &AlwaysKeepErrorsPolicy{},
            &KeepSlowRequestsPolicy{Threshold: 2 * time.Second},
            &ProbabilisticPolicy{SampleRate: 0.01}, // 1%
        },
    }

    shouldSample, reason := policy.Evaluate(trace)
    if shouldSample {
        forwardToBackend(trace, reason)
    } else {
        discardTrace(trace)
    }
}
```

### Key insight box
> Effective tail-based sampling uses composite policies: keep 100% of interesting traces (errors, slow) + small % of baseline traffic.

### Challenge question
How do you set the "slow request" threshold dynamically as your system's performance changes over time?

---

## [PUZZLE] The trace completion problem - when is a trace "done"?

### Scenario
You're buffering traces to make tail-based decisions. But how do you know when a trace is complete?

### Think about it
A trace might have:
- 3 spans (simple request)
- 300 spans (complex microservice call graph)
- Missing spans (network packet loss)

How long do you wait?

### Interactive question (pause and think)
When should you make a sampling decision?

A. After receiving root span
B. After X seconds of no new spans
C. After all parent-child references resolved
D. All of the above, with fallbacks

### Progressive reveal
Answer: D - you need multiple heuristics.

### Trace completion heuristics

**Heuristic 1: Root span detection**
```yaml
root_span:
  definition: Span with no parent_span_id

  logic:
    IF span.parent_span_id == null:
      trace_has_root = true

  problem:
    - Root span might arrive last (async processing)
    - Some traces legitimately have no root (orphaned spans)
```

**Heuristic 2: Reference resolution**
```yaml
reference_resolution:
  logic: |
    For each span:
      IF parent_span_id != null:
        Check if parent exists in trace

    IF all_parent_references_resolved:
      trace_complete = true

  problem:
    - Doesn't handle missing spans (network loss)
    - Circular references can cause infinite wait
```

**Heuristic 3: Inactivity timeout**
```yaml
inactivity_timeout:
  logic: |
    IF (now - trace.last_span_time) > 10 seconds:
      consider_trace_complete = true

  problem:
    - Long-running traces might be prematurely decided
    - Short timeout = incomplete traces
    - Long timeout = high memory usage
```

**Heuristic 4: Expected span count (if available)**
```yaml
expected_span_count:
  mechanism: Root span includes expected_child_count

  logic: |
    IF received_span_count == expected_span_count:
      trace_complete = true

  problem:
    - Requires instrumentation support
    - Count might be wrong (exceptions change flow)
```

### Practical completion strategy

```go
package completion

import (
    "time"
)

type TraceCompletionDetector struct {
    inactivityTimeout time.Duration
    maxTraceAge       time.Duration
}

func (tcd *TraceCompletionDetector) IsComplete(trace *Trace) (bool, string) {
    now := time.Now()

    // Rule 1: Max age exceeded (force decision)
    if now.Sub(trace.FirstSeen) > tcd.maxTraceAge {
        return true, "max_age_exceeded"
    }

    // Rule 2: Has root + all references resolved + inactive
    hasRoot := trace.HasRootSpan()
    allResolved := trace.AllReferencesResolved()
    inactive := now.Sub(trace.LastSeen) > tcd.inactivityTimeout

    if hasRoot && allResolved && inactive {
        return true, "references_resolved_and_inactive"
    }

    // Rule 3: Inactivity timeout (even without root)
    if now.Sub(trace.LastSeen) > tcd.inactivityTimeout * 2 {
        return true, "inactivity_timeout"
    }

    return false, "incomplete"
}

func (t *Trace) HasRootSpan() bool {
    for _, span := range t.Spans {
        if span.ParentSpanID == "" {
            return true
        }
    }
    return false
}

func (t *Trace) AllReferencesResolved() bool {
    spanIDs := make(map[string]bool)

    // Collect all span IDs
    for _, span := range t.Spans {
        spanIDs[span.SpanID] = true
    }

    // Check all parent references exist
    for _, span := range t.Spans {
        if span.ParentSpanID != "" && !spanIDs[span.ParentSpanID] {
            return false // Dangling reference
        }
    }

    return true
}
```

### Production configuration

```yaml
trace_completion_config:

  inactivity_timeout: 10 seconds
    rationale: Most requests complete within seconds

  max_trace_age: 30 seconds
    rationale: Force decision to prevent memory leak

  expected_behavior:
    - Fast traces (< 1s): decided in ~2-3 seconds
    - Normal traces (1-5s): decided in ~10-15 seconds
    - Slow traces (> 10s): decided when they complete or at 30s

  trade_offs:
    - Shorter timeout: faster decisions, more incomplete traces
    - Longer timeout: more complete traces, higher memory usage
    - Optimal: 2-3x expected p99 latency
```

### Key insight box
> Trace completion is probabilistic, not deterministic. Use multiple heuristics with timeouts to force decisions before memory exhaustion.

### Challenge question
A long-running async job generates spans over 5 minutes. How do you handle this without waiting 5 minutes to make a sampling decision?

---

## [WARNING] Scaling challenges - handling millions of traces

### Scenario
Your system does 100K requests/second. That's 8.6 billion traces/day.

Even with tail-based sampling keeping only 10%, that's 860 million traces to store.

### Scaling dimensions

**Dimension 1: Collector memory**
```yaml
memory_calculation:
  request_rate: 100K req/sec
  avg_trace_duration: 500ms

  traces_in_flight: 100K * 0.5s = 50K traces
  avg_trace_size: 50 KB (compressed spans)

  memory_needed: 50K * 50 KB = 2.5 GB

  with_30s_buffer:
    traces_buffered: 100K * 30s = 3M traces
    memory_needed: 3M * 50 KB = 150 GB

  conclusion: Need horizontal scaling
```

**Dimension 2: Collector throughput**
```yaml
throughput_calculation:
  spans_per_second: 500K (5 spans/trace avg)

  operations:
    - Receive: 500K spans/sec
    - Parse: 500K spans/sec
    - Group by trace_id: 500K lookups/sec
    - Evaluate policy: 100K traces/sec
    - Forward: 10K traces/sec (10% kept)

  bottleneck: trace_id grouping (hash map contention)
```

**Dimension 3: Trace distribution**
```yaml
problem: Traces split across collectors

  scenario:
    Collector-1 receives: span A1, A2
    Collector-2 receives: span A3, A4

    Neither collector sees complete trace!

  solution: Consistent hashing by trace_id
    - Route all spans with same trace_id to same collector
    - Load balancer: hash(trace_id) % num_collectors
```

### Horizontal scaling architecture

```text
┌─────────────────────────────────────────────────────────┐
│     SCALED TAIL-BASED SAMPLING ARCHITECTURE             │
└─────────────────────────────────────────────────────────┘

                      ┌──────────────┐
                      │Load Balancer │
                      │(Trace-aware) │
                      └──────┬───────┘
                             │
                   hash(trace_id) % N
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  Collector 1  │   │  Collector 2  │   │  Collector 3  │
│               │   │               │   │               │
│ Trace Buffer  │   │ Trace Buffer  │   │ Trace Buffer  │
│ 50 GB memory  │   │ 50 GB memory  │   │ 50 GB memory  │
│               │   │               │   │               │
│ Handles:      │   │ Handles:      │   │ Handles:      │
│ trace_ids     │   │ trace_ids     │   │ trace_ids     │
│ 0x000 - 0x555 │   │ 0x556 - 0xAAA │   │ 0xAAB - 0xFFF │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │    Backend    │
                    │ (Jaeger/Tempo)│
                    └───────────────┘

Scaling math:
  - 100K req/sec = 33K req/sec per collector
  - 150 GB buffer / 3 = 50 GB per collector
  - Add collectors as traffic grows
```

### Load balancer configuration

```yaml
# OpenTelemetry Collector load balancer config
receivers:
  otlp:
    protocols:
      grpc:

exporters:
  loadbalancing:
    protocol:
      otlp:
        timeout: 1s
    resolver:
      static:
        hostnames:
          - collector-1:4317
          - collector-2:4317
          - collector-3:4317
    routing_key: "traceID"  # CRITICAL: route by trace ID

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [loadbalancing]
```

### Memory pressure handling

```go
package memory

import (
    "runtime"
    "time"
)

type MemoryPressureManager struct {
    maxMemoryBytes  uint64
    evictionTicker  *time.Ticker
    buffer          *TraceBuffer
}

func (mpm *MemoryPressureManager) monitorMemory() {
    for range mpm.evictionTicker.C {
        var m runtime.MemStats
        runtime.ReadMemStats(&m)

        currentMemory := m.Alloc
        pressureRatio := float64(currentMemory) / float64(mpm.maxMemoryBytes)

        if pressureRatio > 0.9 {
            // Critical: evict aggressively
            mpm.evictTraces(0.3) // Drop 30% of buffered traces

        } else if pressureRatio > 0.7 {
            // Warning: evict moderately
            mpm.evictTraces(0.1) // Drop 10% of buffered traces
        }
    }
}

func (mpm *MemoryPressureManager) evictTraces(percentage float64) {
    traces := mpm.buffer.GetOldestTraces(percentage)

    for _, trace := range traces {
        // Make sampling decision NOW (can't wait)
        shouldKeep, reason := evaluatePolicy(trace)

        if shouldKeep {
            forwardToBackend(trace, reason)
        }

        mpm.buffer.Remove(trace.TraceID)
    }
}
```

### Key insight box
> Tail-based sampling at scale requires distributed collectors with trace-aware routing and aggressive memory management.

### Challenge question
How do you handle a traffic spike that 10x's your trace volume? Can you shed load gracefully?

---

## [DEEP DIVE] Tail sampling vs head sampling - when to use which

### Scenario
Your manager asks: "Should we use tail-based sampling for everything?"

### Trade-off analysis

```yaml
head_based_sampling:

  pros:
    - Simple to implement (decision at request start)
    - Low resource overhead (no buffering)
    - Stateless (no coordination needed)
    - Scales linearly with traffic

  cons:
    - Blind sampling (can't see errors/latency)
    - May miss critical traces
    - Fixed sample rate (not adaptive)

  best_for:
    - High-volume, low-error-rate systems
    - Cost-constrained environments
    - When most traces are similar

tail_based_sampling:

  pros:
    - Intelligent sampling (keeps important traces)
    - Catches 100% of errors/slow requests
    - Adaptive to system behavior

  cons:
    - Complex implementation
    - High memory overhead (buffering)
    - Requires collector coordination
    - Scaling challenges

  best_for:
    - Systems with rare but critical errors
    - Variable latency workloads
    - When debugging matters more than cost
```

### Hybrid approach (best of both worlds)

```yaml
hybrid_sampling_strategy:

  layer_1_head_sampling:
    location: Application (SDK)
    policy: Sample 10% of all requests
    rationale: Reduce collector load by 90%

  layer_2_tail_sampling:
    location: Collector
    input: 10% of traces (from head sampling)
    policy:
      - Keep 100% of errors
      - Keep 100% of slow
      - Keep 10% of healthy
    output: ~2-3% of original traces

  result:
    - 10% load on collector (vs 100% with pure tail)
    - Still catches most errors (some lost in head sample)
    - Cost: 2-3% storage (vs 10% with pure head)

  trade_off:
    - Miss ~10% of errors (those not in head sample)
    - But 10x cheaper than pure tail sampling
```

### Decision matrix

```yaml
choose_head_sampling_if:
  - Traffic > 1M requests/second
  - Error rate < 0.1%
  - Budget constrained
  - Debugging mostly via logs/metrics

choose_tail_sampling_if:
  - Error rate variable (0.1% - 5%)
  - Latency highly variable
  - Debugging requires full traces
  - Budget allows higher costs

choose_hybrid_if:
  - Traffic > 100K requests/second
  - Want error coverage without full cost
  - Can tolerate missing 5-10% of errors
```

### Key insight box
> Pure tail-based sampling is ideal for observability but expensive at scale. Hybrid approaches balance cost and coverage.

### Challenge question
Can you implement tail-based sampling at the application level (SDK) instead of collector, avoiding the buffering problem?

---

## [SYNTHESIS] Final synthesis - Design your tail-based sampling system

### Synthesis challenge
You're the observability lead for a global e-commerce platform.

Requirements:
- 500K requests/second peak (43 billion requests/day)
- Average 8 spans per trace = 4M spans/second
- Error rate: 0.5% (215 million errors/day)
- Slow requests (>2s): 1% (430 million/day)
- Must catch 99%+ of errors and slow requests for debugging
- Budget: $50K/month for trace storage

Constraints:
- Current: Head-based 1% sampling (430M traces/day)
- Problem: Missing 99% of errors (only 2.1M errors stored)
- Team: 3 observability engineers
- Must work with OpenTelemetry

### Your tasks (pause and think)
1. Calculate storage requirements for different sampling strategies
2. Design tail-based sampling policies
3. Plan collector architecture (how many instances?)
4. Handle trace completion (timeout strategy)
5. Define memory management approach
6. Estimate costs and justify to leadership

Write down your design.

### Progressive reveal (one possible solution)

**1. Storage requirements:**
```yaml
current_head_sampling_1_percent:
  traces_stored: 430M/day (1% of 43B)
  avg_trace_size: 10 KB
  storage_per_day: 4.3 TB
  retention: 7 days
  total_storage: 30 TB
  cost: $30K/month ($1/GB/month)

  problem:
    errors_total: 215M/day
    errors_stored: 2.1M/day (1% sample)
    error_coverage: 1% (99% of errors lost!)

pure_tail_sampling:
  policy:
    - Keep 100% errors: 215M traces
    - Keep 100% slow: 430M traces
    - Keep 1% healthy: 420M traces
    total: 1.065B traces/day

  storage_per_day: 10.6 TB (2.5x current)
  retention: 7 days
  total_storage: 74 TB
  cost: $74K/month

  result: Over budget, but 100% error coverage

optimized_tail_sampling:
  policy:
    - Keep 100% errors: 215M
    - Keep 100% slow: 430M
    - Keep 0.5% healthy: 210M
    total: 855M traces/day

  storage_per_day: 8.5 TB
  retention: 7 days
  total_storage: 60 TB
  cost: $60K/month

  compromise: Slightly over budget, negotiate or optimize

hybrid_head_tail:
  head_layer: 5% sampling (reduces to 2.15B traces/day)

  tail_layer_on_5_percent:
    - Keep 100% errors: ~10.75M (5% of 215M)
    - Keep 100% slow: ~21.5M (5% of 430M)
    - Keep 10% healthy: ~200M
    total: 232M traces/day

  storage_per_day: 2.3 TB
  total_storage: 16 TB
  cost: $16K/month (under budget!)

  trade_off: Miss 95% of errors in head sample

decision: Use optimized tail sampling, negotiate $10K budget increase
```

**2. Tail-based sampling policies:**
```yaml
sampling_policies:

  policy_1_always_errors:
    condition: any_span.status == ERROR
    sample_rate: 100%
    priority: 1 (highest)

  policy_2_always_slow:
    condition: trace.duration > 2 seconds
    sample_rate: 100%
    priority: 2

  policy_3_critical_endpoints:
    condition: endpoint IN ['/api/checkout', '/api/payment']
    sample_rate: 5%
    priority: 3

  policy_4_enterprise_users:
    condition: user.tier == 'enterprise'
    sample_rate: 2%
    priority: 4

  policy_5_baseline_healthy:
    condition: true (catch-all)
    sample_rate: 0.5%
    priority: 5 (lowest)

estimated_retention:
  errors: 215M (100%)
  slow: 430M (100%)
  critical_endpoints: ~20M (5% of 400M)
  enterprise: ~5M (2% of 250M)
  baseline: ~200M (0.5% of remaining)
  total: 870M traces/day ≈ 2% overall sample rate
```

**3. Collector architecture:**
```yaml
collector_design:

  calculation:
    spans_per_second: 4M
    avg_trace_duration: 500ms
    buffer_window: 30 seconds

    traces_in_flight: 500K req/sec * 30s = 15M traces
    avg_trace_size: 50 KB (in memory)
    memory_per_collector: 50 GB (conservative)

    collectors_needed: (15M * 50KB) / 50GB = 15 collectors

  deployment:
    collectors: 20 instances (20% overhead)
    instance_size: 64 GB RAM, 8 vCPU

  load_balancing:
    method: consistent_hashing by trace_id
    algorithm: hash(trace_id) % 20

  high_availability:
    - Deploy across 3 availability zones
    - 7 collectors per AZ
    - Can lose 1 AZ (still have 13 collectors)
```

**4. Trace completion strategy:**
```yaml
completion_detection:

  inactivity_timeout: 5 seconds
    rationale: p99 latency is 2 seconds, 2.5x buffer

  max_trace_age: 30 seconds
    rationale: Force decision to prevent memory leak

  completion_heuristics:
    1. Has root span
    2. All parent references resolved
    3. No new spans for 5 seconds

  expected_behavior:
    - Fast traces (<1s): decided in 6-7 seconds
    - Normal traces (1-3s): decided in 8-10 seconds
    - Slow traces (>3s): decided when complete or at 30s

  incomplete_traces:
    percentage: ~2% (network issues, missing spans)
    handling: Make best-effort decision at timeout
```

**5. Memory management:**
```yaml
memory_management:

  per_collector_limits:
    max_memory: 50 GB
    warning_threshold: 35 GB (70%)
    critical_threshold: 45 GB (90%)

  eviction_strategy:
    at_70_percent:
      - Evaluate oldest 10% of traces immediately
      - Force sampling decision

    at_90_percent:
      - Evaluate oldest 30% of traces
      - Even if incomplete, make decision

    at_95_percent:
      - Emergency mode: evaluate ALL traces
      - Clear buffer aggressively

  monitoring:
    - Memory usage per collector
    - Traces buffered per collector
    - Eviction rate (alerts if >5% traces evicted)
    - Incomplete trace rate (alerts if >10%)
```

**6. Cost justification:**
```yaml
cost_analysis:

  current_state_problems:
    - Missing 99% of errors (only 2.1M stored)
    - Average incident MTTR: 4 hours
    - Incidents per month: 10
    - Cost per incident: $50K (lost revenue + engineering time)
    - Monthly incident cost: $500K

  with_tail_sampling:
    - Capture 100% of errors (215M stored)
    - Expected MTTR reduction: 50% (better debugging)
    - Cost per incident: $25K (faster resolution)
    - Monthly incident cost: $250K
    - Savings: $250K/month

  roi_calculation:
    - Tail sampling infrastructure: $60K/month
    - Incident cost reduction: $250K/month
    - Net benefit: $190K/month
    - ROI: 317%

  justification_to_leadership:
    "Investing $30K/month more in observability saves
     $250K/month in incident costs. That's an 8x return.
     Plus, we capture 100% of errors instead of 1%."
```

### Key insight box
> Tail-based sampling is a business decision: pay more for storage to save more on incident costs through better debugging.

### Final challenge question
After implementing tail-based sampling, your storage costs are on target, but query performance degrades (too many traces to search). How do you optimize?

---

## Appendix: Quick checklist (printable)

**Tail-based sampling design:**
- [ ] Calculate expected trace volume (requests/sec × retention window)
- [ ] Estimate memory requirements (traces buffered × avg trace size)
- [ ] Define sampling policies (errors, slow, baseline)
- [ ] Choose completion detection strategy (timeout + heuristics)
- [ ] Plan collector scaling (horizontal based on traffic)
- [ ] Configure load balancing (trace-aware routing)

**Sampling policy checklist:**
- [ ] Always keep errors (100% of failures)
- [ ] Always keep slow requests (p99+ latency)
- [ ] Rate limit per endpoint (prevent hot endpoints)
- [ ] Keep baseline healthy traces (0.5-1% for comparison)
- [ ] Support debug mode (specific users/features)
- [ ] Test policies before production (dry-run mode)

**Trace completion detection:**
- [ ] Set inactivity timeout (2-3x p99 latency)
- [ ] Set max trace age (force decision limit)
- [ ] Implement root span detection
- [ ] Implement parent reference resolution
- [ ] Handle incomplete traces (timeout fallback)
- [ ] Monitor completion rate (alert if <90%)

**Memory management:**
- [ ] Set per-collector memory limits
- [ ] Implement eviction on memory pressure
- [ ] Monitor buffer size (traces in memory)
- [ ] Alert on high eviction rate
- [ ] Load test memory usage (before production)
- [ ] Plan for traffic spikes (burst capacity)

**Scaling considerations:**
- [ ] Use consistent hashing by trace_id
- [ ] Deploy collectors across availability zones
- [ ] Monitor collector CPU/memory utilization
- [ ] Auto-scale based on traffic
- [ ] Test failover (collector crashes)
- [ ] Measure end-to-end latency (span → decision → storage)

**Cost optimization:**
- [ ] Measure actual sample rate (% of traces kept)
- [ ] Analyze policy effectiveness (which policies trigger most?)
- [ ] Optimize trace completion timeout (balance completeness vs memory)
- [ ] Consider hybrid head+tail (reduce collector load)
- [ ] Monitor storage costs (alert on unexpected growth)
- [ ] Review policies quarterly (adjust based on traffic patterns)

**Red flags (redesign needed):**
- [ ] Memory pressure causes >10% trace eviction
- [ ] Incomplete trace rate >20%
- [ ] Collector CPU consistently >80%
- [ ] Missing critical errors in sampled data
- [ ] Storage costs exceed ROI from better debugging
- [ ] Query performance degraded despite sampling
