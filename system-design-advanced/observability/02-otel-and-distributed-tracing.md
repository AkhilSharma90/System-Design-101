---
slug: otel-and-distributed-tracing
title: Open Telemetry and Distributed Tracing
readTime: 20 min
orderIndex: 2
premium: false
---


# OpenTelemetry and Distributed Tracing at Scale (Observability in Production)

> Audience: platform engineers and SREs instrumenting microservices for production observability.

This article assumes:
- Your system has dozens to hundreds of microservices calling each other.
- A single user request might traverse 10-50 services.
- Manual logging is insufficient - you need automated correlation across services.
- Performance overhead matters - instrumentation can't slow down production traffic.

---

## [CHALLENGE] Challenge: Your microservices are a black box

### Scenario
Customer complaint: "Checkout failed at 3:42 PM with error 'Payment timeout'."

Your investigation:
- **Frontend logs**: "Called payment service, got timeout after 5s"
- **Payment service logs**: "Received request, called fraud detection, waiting..."
- **Fraud detection logs**: No logs at 3:42 PM (service restarted at 3:40 PM, logs lost)
- **Database logs**: Slow query at 3:41 PM, might be related?

You have 4 services with independent logs. No way to correlate them. No end-to-end visibility.

### Interactive question (pause and think)
How do you debug a request that touches 4 services when logs are disconnected?

1. Add request IDs manually to every log statement
2. Use distributed tracing to automatically link spans
3. Increase log verbosity everywhere
4. Just grep logs really hard

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (2) - distributed tracing solves correlation automatically.

Manual correlation (option 1) doesn't scale beyond 3-5 services. Option 3 makes the problem worse (more noise). Option 4 is pain.

### Real-world analogy (package tracking)
Shipping a package:

**Without tracking**: Package goes through sorting facilities, trucks, and delivery. If it's lost, you have no idea where.

**With tracking**: Every step scans the barcode. You see: "Picked up → Sorting facility → In transit → Out for delivery → Delivered." One tracking number links the entire journey.

Distributed tracing is package tracking for requests.

### Key insight box
> Distributed tracing automatically correlates logs, spans, and events across services using a shared trace ID, giving you end-to-end visibility.

### Challenge question
If distributed tracing is so valuable, why isn't every company using it in production?

---

## [MENTAL MODEL] Mental model - Traces, spans, and context propagation

### Scenario
User clicks "Buy Now" button. Request flows:

```
Frontend → API Gateway → Order Service → Payment Service → Database
```

Each service needs to know: "This request is part of the same user action."

### Interactive question (pause and think)
How do services know they're handling the same logical request?

A. They share a database that tracks requests
B. Each service generates a new request ID
C. A trace ID is passed through headers
D. Services don't know - logs are correlated later

### Progressive reveal
Answer: C.

The trace ID travels with the request through HTTP headers (or gRPC metadata).

### Core concepts

**Trace:**
- Represents one complete user request end-to-end
- Has a unique trace ID (e.g., `abc123`)
- Contains multiple spans

**Span:**
- Represents a single operation within a trace
- Has a span ID and parent span ID
- Contains: operation name, start time, duration, tags, logs

**Context propagation:**
- Passing trace ID and span ID between services
- Usually via HTTP headers: `traceparent: 00-abc123-def456-01`

### Visual: trace structure

```text
┌─────────────────────────────────────────────────────────┐
│              DISTRIBUTED TRACE STRUCTURE                │
└─────────────────────────────────────────────────────────┘

Trace ID: abc123 (one user request)

├─ Span 1: Frontend [0ms - 500ms]
│  ├─ span_id: 001
│  ├─ parent: null (root span)
│  ├─ operation: HTTP GET /checkout
│  └─ tags: {user_id: 12345, endpoint: /checkout}
│
├──┬─ Span 2: API Gateway [50ms - 450ms]
│  │  ├─ span_id: 002
│  │  ├─ parent: 001
│  │  ├─ operation: api_gateway.handle
│  │
│  ├──┬─ Span 3: Order Service [100ms - 400ms]
│  │  │  ├─ span_id: 003
│  │  │  ├─ parent: 002
│  │  │  ├─ operation: create_order
│  │  │
│  │  ├──┬─ Span 4: Payment Service [150ms - 350ms]
│  │  │  │  ├─ span_id: 004
│  │  │  │  ├─ parent: 003
│  │  │  │  ├─ operation: process_payment
│  │  │  │  ├─ tags: {payment_method: credit_card}
│  │  │  │
│  │  │  ├──── Span 5: Database Query [200ms - 300ms]
│  │  │  │     ├─ span_id: 005
│  │  │  │     ├─ parent: 004
│  │  │  │     ├─ operation: db.query
│  │  │  │     ├─ tags: {query: "INSERT INTO payments..."}
│  │  │  │     └─ duration: 100ms
│  │  │  │
│  │  │  └─ Payment span ends (200ms duration)
│  │  │
│  │  └─ Order span ends (300ms duration)
│  │
│  └─ API Gateway span ends (400ms duration)
│
└─ Frontend span ends (500ms duration)

Timeline view:
0ms    100ms   200ms   300ms   400ms   500ms
├───────┼───────┼───────┼───────┼───────┤
│ Frontend                              │
│       │ API Gateway            │      │
│       │       │ Order Service  │      │
│       │       │       │Payment │      │
│       │       │       │       │DB│    │
```

### Context propagation example

```go
package tracing

import (
    "context"
    "net/http"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("example-service")

// Server side: Extract trace context from incoming request
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    // Extract trace context from headers
    ctx := otel.GetTextMapPropagator().Extract(
        r.Context(),
        propagation.HeaderCarrier(r.Header),
    )

    // Start a new span (child of extracted context)
    ctx, span := tracer.Start(ctx, "handle_request")
    defer span.End()

    // Do work...
    result := processOrder(ctx)

    // Add span attributes
    span.SetAttributes(
        attribute.String("order_id", result.OrderID),
        attribute.Int("item_count", len(result.Items)),
    )

    w.Write([]byte("Success"))
}

// Client side: Inject trace context into outgoing request
func CallDownstream(ctx context.Context) error {
    req, _ := http.NewRequestWithContext(ctx, "GET", "http://payment-service/charge", nil)

    // Inject trace context into headers
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

    // Now downstream service can extract and continue the trace
    resp, err := http.DefaultClient.Do(req)
    return err
}
```

### Key insight box
> Context propagation is the magic that links spans across services. Without it, you have independent spans, not a distributed trace.

### Challenge question
What happens if one service in the chain doesn't propagate context? Does the trace break entirely?

---

## [WARNING] OpenTelemetry vs proprietary solutions - standards matter

### Scenario
Your team debates: "Should we use DataDog's tracing, New Relic's, or OpenTelemetry?"

### The vendor lock-in problem

```yaml
proprietary_approach:
  year_1:
    - Instrument services with DataDog SDK
    - Cost: $10K/month
    - Works great!

  year_2:
    - Cost increases to $50K/month (more services, more traces)
    - Decide to switch to Grafana Tempo (cheaper)
    - Problem: All instrumentation is DataDog-specific
    - Re-instrument: 6 months of engineering time

  result: Locked in to vendor

opentelemetry_approach:
  year_1:
    - Instrument services with OpenTelemetry SDK
    - Export to Jaeger backend
    - Cost: $2K/month (self-hosted)

  year_2:
    - Switch backend to Grafana Tempo
    - Change: 1 line in config (exporter endpoint)
    - Re-instrumentation: 0 days

  result: Freedom to choose backend
```

### OpenTelemetry architecture

```text
┌─────────────────────────────────────────────────────────┐
│         OPENTELEMETRY ARCHITECTURE                      │
└─────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                   │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │ Service A  │  │ Service B  │  │ Service C  │   │
│  │            │  │            │  │            │   │
│  │ OTel SDK   │  │ OTel SDK   │  │ OTel SDK   │   │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘   │
│        │ OTLP          │ OTLP          │ OTLP     │
└────────┼───────────────┼───────────────┼──────────┘
         │               │               │
         └───────────────┴───────────────┘
                         │
                         ▼
         ┌───────────────────────────────┐
         │   OpenTelemetry Collector     │
         │                               │
         │  ┌──────────────────────┐    │
         │  │  Receivers (OTLP)    │    │
         │  └──────────┬───────────┘    │
         │             │                 │
         │  ┌──────────▼───────────┐    │
         │  │  Processors          │    │
         │  │  - Batching          │    │
         │  │  - Sampling          │    │
         │  │  - Attributes        │    │
         │  └──────────┬───────────┘    │
         │             │                 │
         │  ┌──────────▼───────────┐    │
         │  │  Exporters           │    │
         │  │  - Jaeger            │    │
         │  │  - Tempo             │    │
         │  │  - Zipkin            │    │
         │  └──────────────────────┘    │
         └───────────────────────────────┘
                         │
         ┌───────────────┴────────────────┐
         │                                │
         ▼                                ▼
┌────────────────┐              ┌────────────────┐
│ Jaeger Backend │              │ Grafana Tempo  │
│ (vendor A)     │              │ (vendor B)     │
└────────────────┘              └────────────────┘

Benefits:
  - Vendor-neutral instrumentation
  - Swap backends without code changes
  - Standard protocol (OTLP)
  - Community support
```

### Why OpenTelemetry won

```yaml
before_opentelemetry:
  tracing_standards:
    - OpenTracing (one standard)
    - OpenCensus (another standard)
  problem: Two competing standards, fragmentation

after_merge:
  opentelemetry: Merged OpenTracing + OpenCensus
  backed_by: Google, Microsoft, AWS, Splunk, Datadog
  result: Industry convergence on one standard

adoption:
  kubernetes: Native OTel support
  cloud_providers: AWS X-Ray, GCP Trace support OTLP
  vendors: DataDog, New Relic, Honeycomb support OTel
```

### Key insight box
> OpenTelemetry is the HTTPS of observability - a standard that everyone implements, preventing vendor lock-in.

### Challenge question
If OpenTelemetry is vendor-neutral, how do vendors like DataDog make money? What's their moat?

---

## [DEEP DIVE] Instrumenting applications - auto vs manual

### Scenario
You need to add tracing to 50 microservices. Do you:
- Auto-instrument everything (zero code changes)
- Manually instrument critical paths (more control)

### Auto-instrumentation

```yaml
auto_instrumentation:
  how_it_works:
    - Language agent (Java, Python, Node.js, Go)
    - Bytecode injection or monkey-patching
    - Automatically traces: HTTP, gRPC, database, cache

  pros:
    - Zero code changes required
    - Quick to deploy
    - Covers common frameworks (Flask, Express, Spring)

  cons:
    - Limited control over span names
    - Can't add custom business context
    - Higher performance overhead
    - May miss custom protocols

  example_java:
    - Download OpenTelemetry Java agent JAR
    - Run: java -javaagent:opentelemetry-javaagent.jar -jar app.jar
    - Done! HTTP requests, DB queries auto-traced
```

**Auto-instrumentation example (Python):**

```python
# No code changes needed!
# Just install and configure

# Install:
# pip install opentelemetry-distro
# opentelemetry-bootstrap -a install

# Run with auto-instrumentation:
# opentelemetry-instrument python app.py

# app.py (unchanged)
from flask import Flask
app = Flask(__name__)

@app.route('/checkout')
def checkout():
    # This HTTP request is automatically traced
    # Database queries automatically traced
    result = db.query("SELECT * FROM orders")
    return "Success"
```

### Manual instrumentation

```go
package main

import (
    "context"
    "database/sql"
    "net/http"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("checkout-service")

func HandleCheckout(w http.ResponseWriter, r *http.Request) {
    // Extract incoming trace context
    ctx := otel.GetTextMapPropagator().Extract(
        r.Context(),
        propagation.HeaderCarrier(r.Header),
    )

    // Start span for checkout operation
    ctx, span := tracer.Start(ctx, "checkout",
        trace.WithSpanKind(trace.SpanKindServer),
    )
    defer span.End()

    // Add custom business attributes
    userID := r.Header.Get("X-User-ID")
    span.SetAttributes(
        attribute.String("user.id", userID),
        attribute.String("checkout.type", "express"),
    )

    // Call payment service (manually instrument)
    paymentResult, err := processPayment(ctx, userID)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "payment failed")
        http.Error(w, "Payment failed", 500)
        return
    }

    // Add result attributes
    span.SetAttributes(
        attribute.String("payment.id", paymentResult.ID),
        attribute.Float64("payment.amount", paymentResult.Amount),
    )

    w.Write([]byte("Success"))
}

func processPayment(ctx context.Context, userID string) (*PaymentResult, error) {
    // Child span
    ctx, span := tracer.Start(ctx, "process_payment")
    defer span.End()

    // Simulate payment processing
    span.AddEvent("validating_card")

    // Call fraud detection
    fraudResult, err := checkFraud(ctx, userID)
    if err != nil {
        return nil, err
    }

    span.SetAttributes(
        attribute.Bool("fraud.detected", fraudResult.IsFraud),
        attribute.Float64("fraud.score", fraudResult.Score),
    )

    return &PaymentResult{ID: "pay_123", Amount: 99.99}, nil
}

func checkFraud(ctx context.Context, userID string) (*FraudResult, error) {
    ctx, span := tracer.Start(ctx, "fraud_check")
    defer span.End()

    // Database query (manually traced)
    ctx, dbSpan := tracer.Start(ctx, "db.query",
        trace.WithSpanKind(trace.SpanKindClient),
    )
    dbSpan.SetAttributes(
        attribute.String("db.system", "postgresql"),
        attribute.String("db.statement", "SELECT fraud_score FROM users WHERE id = $1"),
    )

    // Execute query
    var fraudScore float64
    err := db.QueryRowContext(ctx, "SELECT fraud_score FROM users WHERE id = $1", userID).Scan(&fraudScore)

    dbSpan.End()

    if err != nil {
        return nil, err
    }

    return &FraudResult{Score: fraudScore, IsFraud: fraudScore > 0.8}, nil
}
```

### Hybrid approach (best practice)

```yaml
recommended_strategy:

  layer_1_auto_instrumentation:
    what: HTTP servers, database clients, cache clients
    why: Covers 80% of traces with zero effort

  layer_2_manual_instrumentation:
    what: Business logic, critical paths, custom operations
    why: Add domain-specific context

  example:
    auto_traces:
      - HTTP request received (auto)
      - Database query to orders table (auto)
      - HTTP call to payment service (auto)

    manual_spans:
      - "calculate_tax" span with cart_value attribute
      - "apply_discount" span with coupon_code attribute
      - "reserve_inventory" span with sku attributes
```

### Key insight box
> Start with auto-instrumentation for coverage, add manual instrumentation for business context. The combination gives you both breadth and depth.

### Challenge question
You auto-instrument a legacy service and performance drops by 20%. How do you debug which instrumentation is causing overhead?

---

## [PUZZLE] Sampling strategies at the SDK level

### Scenario
You have 100K requests/second. Can't send all traces to backend.

Where do you sample: application (SDK) or collector?

### Think about it
- **SDK sampling**: Decide at request start (head-based)
- **Collector sampling**: Decide after trace completes (tail-based)

### Interactive question (pause and think)
Why might you sample at the SDK level instead of collector?

A. Reduce network bandwidth (don't send unsampled traces)
B. Reduce collector load
C. Lower latency (no collector buffering)
D. All of the above

### Progressive reveal
Answer: D - SDK sampling reduces load on everything downstream.

### SDK sampling strategies

**Strategy 1: AlwaysOn sampler**
```go
import "go.opentelemetry.io/otel/sdk/trace"

// Sample 100% of traces (development/low-traffic)
sampler := trace.AlwaysSample()
```

**Strategy 2: Probabilistic sampler**
```go
// Sample 1% of traces
sampler := trace.TraceIDRatioBased(0.01)

// How it works:
// - Hash trace ID
// - If hash < (0.01 * max_uint64), sample
// - Deterministic per trace_id
```

**Strategy 3: Parent-based sampler**
```go
// If parent span was sampled, sample this span too
sampler := trace.ParentBased(
    trace.TraceIDRatioBased(0.01),
)

// Ensures: If trace is sampled, ALL spans in trace are sampled
// Avoids: Partial traces with missing spans
```

**Strategy 4: Custom sampler (rate limiting)**
```go
package sampling

import (
    "sync"
    "time"

    "go.opentelemetry.io/otel/sdk/trace"
)

type RateLimitingSampler struct {
    maxTracesPerSecond int
    window             time.Duration
    counter            int
    windowStart        time.Time
    mu                 sync.Mutex
}

func NewRateLimitingSampler(maxTracesPerSecond int) *RateLimitingSampler {
    return &RateLimitingSampler{
        maxTracesPerSecond: maxTracesPerSecond,
        window:             1 * time.Second,
        windowStart:        time.Now(),
    }
}

func (s *RateLimitingSampler) ShouldSample(p trace.SamplingParameters) trace.SamplingResult {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now()

    // Reset window if expired
    if now.Sub(s.windowStart) > s.window {
        s.counter = 0
        s.windowStart = now
    }

    // Check if under limit
    if s.counter < s.maxTracesPerSecond {
        s.counter++
        return trace.SamplingResult{
            Decision: trace.RecordAndSample,
        }
    }

    return trace.SamplingResult{
        Decision: trace.Drop,
    }
}

func (s *RateLimitingSampler) Description() string {
    return "RateLimitingSampler"
}
```

### SDK vs Collector sampling trade-offs

```yaml
sdk_sampling:
  pros:
    - Reduces network bandwidth (95%+ reduction)
    - Reduces collector CPU/memory load
    - Lower end-to-end latency
    - Works without collector

  cons:
    - Head-based only (can't see errors/latency)
    - All instances must have same config
    - Hard to change sampling rate (requires redeployment)

collector_sampling:
  pros:
    - Tail-based (intelligent sampling)
    - Centralized configuration
    - Can adjust sampling without redeploying apps

  cons:
    - All traces sent to collector (network bandwidth)
    - Collector must buffer traces (memory)
    - Higher latency (buffering delay)

best_practice:
  - SDK: 10-20% sampling (reduce network load)
  - Collector: Tail-based sampling on that 10-20%
  - Result: 1-2% final sample rate with intelligent selection
```

### Key insight box
> SDK sampling is economic (reduce costs), collector sampling is intelligent (keep important traces). Use both in layers.

### Challenge question
How do you ensure consistent sampling across multiple SDK languages (Go, Python, Java) without configuration drift?

---

## [DEEP DIVE] Performance overhead - measuring the cost of observability

### Scenario
You add OpenTelemetry to production. Suddenly p99 latency increases from 200ms to 250ms.

Is the 25% latency increase worth the observability gain?

### Sources of overhead

```yaml
overhead_breakdown:

  span_creation:
    cost: 1-5 microseconds per span
    factors:
      - Memory allocation
      - Timestamp capture
      - Attribute assignment

  context_propagation:
    cost: 10-50 microseconds per HTTP call
    factors:
      - Header injection/extraction
      - Context copying

  span_export:
    cost: 100-1000 microseconds (batched)
    factors:
      - Serialization (protobuf)
      - Network I/O to collector
      - Batching delays
    mitigation: Asynchronous export (non-blocking)

  total_typical_overhead:
    low_traffic: 1-5% latency increase
    high_traffic: 5-10% latency increase
    high_span_count: 10-20% latency increase
```

### Measuring overhead

```go
package benchmark

import (
    "context"
    "testing"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/sdk/trace"
)

func BenchmarkWithoutTracing(b *testing.B) {
    ctx := context.Background()

    for i := 0; i < b.N; i++ {
        doWork(ctx)
    }
}

func BenchmarkWithTracing(b *testing.B) {
    // Setup OpenTelemetry
    tp := trace.NewTracerProvider()
    defer tp.Shutdown(context.Background())

    tracer := tp.Tracer("benchmark")
    ctx := context.Background()

    for i := 0; i < b.N; i++ {
        ctx, span := tracer.Start(ctx, "do_work")
        doWork(ctx)
        span.End()
    }
}

func doWork(ctx context.Context) {
    // Simulate 100ms of work
    time.Sleep(100 * time.Millisecond)
}

// Results:
// BenchmarkWithoutTracing-8    10000    100000 ns/op
// BenchmarkWithTracing-8       9500     105000 ns/op
// Overhead: 5% (5000 ns per operation)
```

### Optimization strategies

**Optimization 1: Reduce span cardinality**
```go
// BAD: High cardinality span names
span := tracer.Start(ctx, fmt.Sprintf("query_user_%s", userID))
// Creates millions of unique span names

// GOOD: Low cardinality span names
span := tracer.Start(ctx, "query_user")
span.SetAttributes(attribute.String("user.id", userID))
// One span name, user ID in attributes
```

**Optimization 2: Sampling**
```go
// Sample 10% in production, 100% in development
sampler := trace.ParentBased(
    trace.TraceIDRatioBased(getSamplingRate()),
)

func getSamplingRate() float64 {
    if os.Getenv("ENV") == "production" {
        return 0.10
    }
    return 1.0
}
```

**Optimization 3: Asynchronous export**
```go
// BAD: Synchronous export (blocks request)
exporter := otlptracegrpc.New(ctx,
    otlptracegrpc.WithInsecure(),
)

// GOOD: Batched async export
exporter := otlptracegrpc.New(ctx,
    otlptracegrpc.WithInsecure(),
)

bsp := trace.NewBatchSpanProcessor(exporter,
    trace.WithBatchTimeout(5*time.Second),
    trace.WithMaxQueueSize(2048),
    trace.WithMaxExportBatchSize(512),
)
```

**Optimization 4: Conditional instrumentation**
```go
// Only trace slow operations
func QueryDatabase(ctx context.Context, query string) error {
    start := time.Now()

    result := executeQuery(query)
    duration := time.Since(start)

    // Only create span if query was slow
    if duration > 100*time.Millisecond {
        _, span := tracer.Start(ctx, "slow_query")
        span.SetAttributes(
            attribute.String("db.query", query),
            attribute.Int64("duration_ms", duration.Milliseconds()),
        )
        span.End()
    }

    return result
}
```

### Key insight box
> Tracing overhead is typically 1-10% latency increase. Optimize by sampling, async export, and low-cardinality span names.

### Challenge question
Your service creates 50 spans per request. Overhead is 20%. How do you decide which spans to keep and which to remove?

---

## [WARNING] Common OpenTelemetry anti-patterns

### Scenario
Your team has instrumented everything. But traces are useless.

### Anti-patterns catalog

**Anti-pattern 1: Too many spans (noise)**
```go
// BAD: Span for every tiny operation
func ProcessOrder(ctx context.Context) {
    ctx, span1 := tracer.Start(ctx, "validate_input")
    validateInput()
    span1.End()

    ctx, span2 := tracer.Start(ctx, "calculate_tax")
    calculateTax()
    span2.End()

    ctx, span3 := tracer.Start(ctx, "round_total")
    roundTotal()
    span3.End()

    // 50 spans for one request - impossible to read
}

// GOOD: Spans for meaningful operations
func ProcessOrder(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "process_order")
    defer span.End()

    validateInput()    // No span (too small)
    calculateTax()     // No span (too small)
    roundTotal()       // No span (too small)

    // Only span for expensive operations
    saveToDatabase(ctx)  // This has its own span
}
```

**Anti-pattern 2: High-cardinality span names**
```go
// BAD: Unique span name per user
span := tracer.Start(ctx, "user_" + userID)

// Creates millions of unique span names
// Makes aggregation impossible

// GOOD: Fixed span name, user in attribute
span := tracer.Start(ctx, "user_lookup")
span.SetAttributes(attribute.String("user.id", userID))
```

**Anti-pattern 3: Forgetting to propagate context**
```go
// BAD: Context not propagated
func HandleRequest(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "handle")
    defer span.End()

    // New context created, trace is lost!
    callDownstream(context.Background())
}

// GOOD: Context propagated
func HandleRequest(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "handle")
    defer span.End()

    callDownstream(ctx)  // ctx carries trace info
}
```

**Anti-pattern 4: Sensitive data in spans**
```go
// BAD: PII in span attributes
span.SetAttributes(
    attribute.String("user.email", "user@example.com"),
    attribute.String("credit_card", "4111111111111111"),
)

// GOOD: Redact or hash sensitive data
span.SetAttributes(
    attribute.String("user.email_hash", hash("user@example.com")),
    attribute.String("credit_card_last4", "1111"),
)
```

**Anti-pattern 5: Blocking on span export**
```go
// BAD: Synchronous export
exporter := otlpgrpc.New(ctx)
tp := trace.NewTracerProvider(
    trace.WithSyncer(exporter),  // Blocks on every span!
)

// GOOD: Async batched export
exporter := otlpgrpc.New(ctx)
tp := trace.NewTracerProvider(
    trace.WithBatcher(exporter),  // Batches in background
)
```

### Key insight box
> Common mistakes: too many spans, high cardinality, dropped context, PII exposure, synchronous export. Avoid these to keep traces useful and performant.

### Challenge question
Your traces contain passwords in span attributes (accidental logging). How do you scrub existing traces from the backend?

---

## [SYNTHESIS] Final synthesis - Instrument a production system

### Synthesis challenge
You're the observability lead for a microservices e-commerce platform.

Requirements:
- 50 microservices (Go, Python, Node.js)
- 200K requests/second peak
- Need to debug: checkout failures, slow searches, payment timeouts
- Compliance: No PII in traces (GDPR)
- Performance: < 5% latency overhead acceptable

Constraints:
- Team: 5 engineers, limited OpenTelemetry experience
- Budget: $20K/month for tracing backend
- Current: No distributed tracing (only logs)
- Timeline: 3 months to full rollout

### Your tasks (pause and think)
1. Choose instrumentation strategy (auto vs manual)
2. Select sampling approach (SDK, collector, or both)
3. Design trace backend architecture
4. Plan PII redaction strategy
5. Define performance testing approach
6. Create rollout plan (which services first?)

Write down your architecture.

### Progressive reveal (one possible solution)

**1. Instrumentation strategy:**
```yaml
instrumentation_approach:

  phase_1_auto_instrumentation:
    services: All 50 microservices
    method: Language-specific auto-instrumentation
    coverage:
      - HTTP requests/responses
      - Database queries (Postgres, MongoDB)
      - Cache operations (Redis)
      - Message queues (Kafka)
    effort: 1 week (deploy agents, configure exporters)

  phase_2_manual_instrumentation:
    services: 5 critical services (checkout, payment, search)
    method: Manual SDK instrumentation
    additions:
      - Business context (cart_value, payment_method)
      - Custom operations (fraud_check, tax_calculation)
      - Error details (failure_reason, retry_attempts)
    effort: 4 weeks (code changes, testing)

  languages:
    go: Auto-instrument with otel/instrumentation packages
    python: opentelemetry-bootstrap + opentelemetry-instrument
    nodejs: @opentelemetry/auto-instrumentations-node
```

**2. Sampling approach:**
```yaml
sampling_strategy:

  sdk_layer:
    sampler: ParentBased(TraceIDRatioBased(0.10))
    reason: Reduce network load by 90%
    config_management: Environment variable OTEL_TRACES_SAMPLER_ARG

  collector_layer:
    sampler: Tail-based sampling
    policies:
      - Always keep errors (100%)
      - Always keep slow (>2s duration, 100%)
      - Rate limit per endpoint (1000 traces/min)
      - Probabilistic baseline (1% of healthy traces)

  expected_retention:
    incoming: 200K req/sec × 10% SDK = 20K traces/sec to collector
    collector_output:
      - Errors (0.5%): 1K traces/sec
      - Slow (1%): 2K traces/sec
      - Baseline (0.1%): 200 traces/sec
    total_stored: 3.2K traces/sec = 1.6% of original traffic

  cost:
    traces_per_day: 3.2K × 86400 = 276M traces/day
    storage_cost: $15K/month (Grafana Tempo)
    under_budget: Yes ($20K/month available)
```

**3. Trace backend architecture:**
```yaml
backend_architecture:

  choice: Grafana Tempo (cost-effective)

  deployment:
    collectors:
      - 10 OTel Collector instances
      - Load balanced by trace_id (consistent hashing)
      - 32 GB RAM each (buffering for tail sampling)

    tempo:
      - S3 backend for storage (cheap)
      - Retention: 30 days (compliance requirement)
      - Query: Grafana dashboards

  data_flow:
    applications (50 services)
      ↓ OTLP
    OTel Collectors (10 instances, tail sampling)
      ↓ OTLP
    Tempo (storage)
      ↑ query
    Grafana (visualization)

  high_availability:
    - Collectors across 3 availability zones
    - Tempo multi-instance with S3 backend
    - No single point of failure
```

**4. PII redaction strategy:**
```yaml
pii_redaction:

  layer_1_sdk:
    - Never instrument: passwords, credit cards, SSNs
    - Hash: email addresses, phone numbers
    - Truncate: IP addresses (keep first 2 octets)

  layer_2_collector:
    processor: attributes/redact
    rules:
      - Delete attributes matching: .*password.*
      - Delete attributes matching: .*credit_card.*
      - Hash attributes: user.email, user.phone

  example_config:
    processors:
      attributes/redact:
        actions:
          - key: user.password
            action: delete
          - key: user.email
            action: hash
          - key: http.request.header.authorization
            action: delete

  validation:
    - Automated tests for PII detection
    - Manual audit of sample traces monthly
    - GDPR compliance review quarterly
```

**5. Performance testing:**
```yaml
performance_testing:

  baseline_measurement:
    environment: Load test in staging
    metric: p50, p99, p999 latency
    load: 10K req/sec sustained
    without_tracing: Record baseline

  instrumentation_overhead:
    phase_1_auto_only:
      expected_overhead: 3-5%
      acceptance: < 5% p99 latency increase

    phase_2_auto_plus_manual:
      expected_overhead: 5-8%
      acceptance: < 10% p99 latency increase

  testing_approach:
    1. Instrument 1 service, measure overhead
    2. If acceptable, instrument next 5 services
    3. If acceptable, instrument all services
    4. Continuous monitoring in production

  rollback_plan:
    - If overhead > 10%: disable manual instrumentation
    - If overhead > 15%: disable auto-instrumentation
    - Always: Feature flag to disable tracing per service
```

**6. Rollout plan:**
```yaml
rollout_timeline:

  month_1_infrastructure:
    week_1:
      - Deploy OTel Collectors (10 instances)
      - Deploy Grafana Tempo
      - Setup Grafana dashboards

    week_2:
      - Auto-instrument 5 non-critical services (staging)
      - Load test, measure overhead
      - Fix any issues

    week_3:
      - Auto-instrument 5 non-critical services (production)
      - Monitor for 1 week
      - Validate traces are useful

    week_4:
      - Auto-instrument remaining 40 services
      - Monitor, tune sampling rates

  month_2_manual_instrumentation:
    - Manually instrument checkout service
    - Manually instrument payment service
    - Manually instrument search service
    - Add business context, custom spans

  month_3_optimization:
    - Review sampling policies (adjust based on data)
    - Optimize high-overhead services
    - Train team on using traces for debugging
    - Document runbooks for common trace patterns

  success_metrics:
    - 95% of services instrumented
    - < 5% latency overhead
    - MTTR reduced by 30% (faster debugging)
    - 100% of errors captured in traces
```

### Key insight box
> Rollout distributed tracing incrementally: auto-instrument for coverage, manual instrumentation for depth, measure overhead continuously.

### Final challenge question
After 6 months, your trace storage costs doubled (traffic grew). How do you reduce costs without losing observability?

---

## Appendix: Quick checklist (printable)

**Instrumentation checklist:**
- [ ] Choose SDK language (Go, Python, Java, Node.js)
- [ ] Configure auto-instrumentation (HTTP, DB, cache)
- [ ] Add manual spans for critical business logic
- [ ] Propagate context across service boundaries
- [ ] Test context propagation (end-to-end traces)
- [ ] Avoid high-cardinality span names

**Sampling configuration:**
- [ ] Configure SDK sampling (10-20% recommended)
- [ ] Configure collector tail sampling (errors, slow)
- [ ] Set sampling consistently across services
- [ ] Monitor actual sample rate (vs expected)
- [ ] Adjust sampling based on costs

**Performance optimization:**
- [ ] Measure baseline latency (before tracing)
- [ ] Measure overhead after instrumentation
- [ ] Use async batched export (not synchronous)
- [ ] Reduce span count (only meaningful operations)
- [ ] Load test with tracing enabled

**PII and security:**
- [ ] Identify sensitive attributes (passwords, credit cards)
- [ ] Configure attribute redaction in collector
- [ ] Hash or truncate PII (emails, IPs)
- [ ] Audit traces for PII leakage
- [ ] Document PII handling policy

**Backend setup:**
- [ ] Deploy OTel Collectors (load balanced)
- [ ] Choose backend (Jaeger, Tempo, vendor)
- [ ] Configure retention policy
- [ ] Setup Grafana dashboards
- [ ] Test collector failover

**Operational readiness:**
- [ ] Create runbooks for common trace queries
- [ ] Train team on trace debugging
- [ ] Monitor collector health (CPU, memory)
- [ ] Monitor trace ingestion rate
- [ ] Alert on trace export failures

**Red flags (fix immediately):**
- [ ] Traces missing spans (context not propagated)
- [ ] High-cardinality explosion (millions of unique span names)
- [ ] PII in traces (compliance violation)
- [ ] > 10% latency overhead (optimize instrumentation)
- [ ] Collector OOM crashes (reduce buffer or add instances)
- [ ] Traces incomplete (spans arriving out of order)
