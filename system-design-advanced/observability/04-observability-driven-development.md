---
slug: observability-driven-development
title: Observability Driven Development
readTime: 20 min
orderIndex: 1
premium: false
---




# Observability-Driven Development (ODD) - Building Observable Systems from Day One

> Audience: software engineers and architects who want to build systems that are debuggable, understandable, and operationally excellent.

This article assumes:
- Observability is not an afterthought - it shapes how you design systems.
- The hardest bugs to fix are the ones you can't reproduce or understand.
- Production is the only environment where real failures happen.
- Your code will be maintained by someone else (maybe future you) who doesn't understand it.

---

## [CHALLENGE] Challenge: You can't debug what you can't observe

### Scenario
3 AM page: "Checkout is failing for some users."

Your debugging session:
- **You**: "What error do they see?"
- **On-call**: "Just 'Something went wrong'"
- **You**: "Check the logs"
- **On-call**: "Which service? There are 12 in the checkout flow"
- **You**: "Check them all"
- **On-call**: "Nothing obvious in any of them"
- **You**: "Is there a trace?"
- **On-call**: "We don't have tracing"
- **You**: "Check metrics"
- **On-call**: "Which metric? We have 500"
- **You**: [4 AM] Still debugging, no progress

The system works 99% of the time. But that 1% is invisible.

### Interactive question (pause and think)
What failed here?

1. The on-call engineer wasn't skilled enough
2. The system wasn't designed to be observable
3. Need better monitoring tools
4. This is just how distributed systems work

Take 10 seconds.

### Progressive reveal (question -> think -> answer)
Answer: (2) - the system wasn't designed with observability in mind.

Better tools (3) help, but they can't compensate for a system that doesn't emit useful signals.

### Real-world analogy (car dashboard)
Imagine driving a car with no dashboard:

**No observability**: Car makes a weird noise. Is it engine, transmission, brakes? Open the hood and guess.

**Basic observability**: Dashboard shows: engine temp, RPM, speed. Better, but why is "check engine" light on?

**Full observability**: Dashboard shows: cylinder 3 misfiring, fuel injector clogged, O2 sensor voltage low. You know exactly what's wrong.

Observable systems are like cars with diagnostic ports - they tell you what's wrong.

### Key insight box
> Observability-Driven Development (ODD) means designing systems to explain their own behavior, not just work correctly.

### Challenge question
If observability is so important, why do most teams add it after the system is built?

---

## [MENTAL MODEL] Mental model - Observable systems explain themselves

### Scenario
You're building a new payment processing service. Two approaches:

**Traditional approach:**
1. Write code to process payments
2. Deploy to production
3. Add logging when bugs occur
4. Add metrics when performance degrades
5. Add tracing when you can't figure out why it's slow

**ODD approach:**
1. Define what observability you need (before writing code)
2. Write code that emits structured logs, metrics, traces
3. Deploy with observability built-in
4. Debug using the signals you designed
5. Iterate based on what you learn

### Interactive question (pause and think)
Which approach leads to better debuggability?

A. Traditional (add observability reactively)
B. ODD (design observability proactively)
C. Doesn't matter, same end result
D. Depends on the team

### Progressive reveal
Answer: B - proactive observability is exponentially better.

Reactive observability is like adding airbags after a car crash. Proactive is designing the car with airbags from the start.

### Mental model
Think of ODD as:

- **Test-Driven Development for operations**: Define how you'll debug before writing code
- **Design by contract for runtime**: System makes promises about what it will tell you
- **Defensive programming meets observability**: Anticipate what will go wrong and instrument for it

The mindset shift:
```yaml
traditional_thinking:
  question: "Does this code work?"

odd_thinking:
  questions:
    - "Does this code work?"
    - "How will I know if it breaks?"
    - "What will I see when it's slow?"
    - "Can I debug it at 3 AM without source code access?"
```

### Key insight box
> ODD treats observability as a first-class design concern, like security or performance, not an operational afterthought.

### Challenge question
You're in a code review. The PR has perfect logic but zero observability. Do you approve it?

---

## [WARNING] The three pillars are necessary but not sufficient

### Scenario
Your team says: "We have metrics, logs, and traces. We're observable!"

Then an incident happens. You have:
- 10,000 metrics (which one matters?)
- 1TB of logs per day (needle in haystack)
- 1 million traces (which trace explains the bug?)

You're drowning in data but starving for insight.

### Beyond the three pillars

```text
┌─────────────────────────────────────────────────────────┐
│         OBSERVABILITY LAYERS                            │
└─────────────────────────────────────────────────────────┘

LAYER 1: DATA (Necessary but not sufficient)
  ├─ Metrics: What is happening (aggregates)
  ├─ Logs: What happened (events)
  └─ Traces: How it happened (causality)

LAYER 2: CONTEXT (Makes data interpretable)
  ├─ Correlation IDs: Link related events
  ├─ Request context: User, session, feature flags
  ├─ Build metadata: Version, commit, deploy time
  └─ Business context: Order ID, transaction ID

LAYER 3: STRUCTURE (Makes data queryable)
  ├─ Structured logging: JSON, not plain text
  ├─ Consistent naming: metric_name_unit convention
  ├─ Standard labels: environment, service, region
  └─ Semantic conventions: OpenTelemetry standards

LAYER 4: MEANING (Makes data actionable)
  ├─ SLIs/SLOs: What "healthy" looks like
  ├─ Alerts: What requires human attention
  ├─ Runbooks: How to fix common issues
  └─ Dashboards: What to look at first

LAYER 5: FEEDBACK (Makes data improve the system)
  ├─ Incident analysis: Learn from failures
  ├─ Capacity planning: Prevent future issues
  ├─ Performance optimization: Find bottlenecks
  └─ Feature flagging: Test impact before full rollout
```

### The context problem

```go
package badexample

// BAD: Log without context
log.Info("Payment failed")

// What payment? Why? For which user?
// This log is useless in production

// GOOD: Log with full context
log.WithFields(log.Fields{
    "user_id": userID,
    "order_id": orderID,
    "payment_id": paymentID,
    "amount": amount,
    "currency": currency,
    "payment_method": method,
    "error": err.Error(),
    "retry_attempt": retryCount,
}).Error("Payment processing failed")

// Now you can debug:
// - Who had the problem?
// - Which payment?
// - What was the error?
// - Was it a retry?
```

### The structure problem

```bash
# BAD: Unstructured logs (impossible to parse)
2024-01-04 10:23:45 INFO Payment for user 12345 failed with error timeout

# GOOD: Structured logs (queryable)
{
  "timestamp": "2024-01-04T10:23:45Z",
  "level": "error",
  "message": "Payment processing failed",
  "user_id": "12345",
  "error_type": "timeout",
  "duration_ms": 5000,
  "trace_id": "abc123"
}

# Can now query: "Show me all timeout errors for user 12345"
```

### Key insight box
> Three pillars give you data. Context, structure, and meaning give you understanding. ODD provides all five layers.

### Challenge question
You have perfect metrics, logs, and traces. But during an incident, you still can't figure out the root cause. What's missing?

---

## [DEEP DIVE] Designing observable code - practical patterns

### Scenario
You're writing a new feature: "Apply discount code to shopping cart."

How do you make it observable?

### Pattern 1: Structured logging with context

```go
package cart

import (
    "context"

    "github.com/sirupsen/logrus"
)

type Logger struct {
    *logrus.Entry
}

func (l *Logger) WithContext(ctx context.Context) *Logger {
    return &Logger{
        Entry: l.Entry.WithFields(logrus.Fields{
            "trace_id":   getTraceID(ctx),
            "request_id": getRequestID(ctx),
            "user_id":    getUserID(ctx),
        }),
    }
}

func ApplyDiscount(ctx context.Context, cartID string, code string) error {
    log := logger.WithContext(ctx).WithFields(logrus.Fields{
        "cart_id": cartID,
        "discount_code": code,
        "function": "ApplyDiscount",
    })

    log.Info("Applying discount code")

    // Validate discount code
    discount, err := validateDiscountCode(code)
    if err != nil {
        log.WithError(err).Warn("Invalid discount code")
        return ErrInvalidCode
    }

    log.WithFields(logrus.Fields{
        "discount_percent": discount.Percent,
        "discount_amount": discount.Amount,
        "discount_type": discount.Type,
    }).Info("Discount code validated")

    // Apply discount to cart
    originalTotal, err := getCartTotal(cartID)
    if err != nil {
        log.WithError(err).Error("Failed to get cart total")
        return err
    }

    newTotal := calculateDiscountedTotal(originalTotal, discount)

    log.WithFields(logrus.Fields{
        "original_total": originalTotal,
        "new_total": newTotal,
        "savings": originalTotal - newTotal,
    }).Info("Discount applied successfully")

    return nil
}
```

### Pattern 2: Metrics at decision points

```go
package cart

import (
    "github.com/prometheus/client_golang/prometheus"
    "time"
)

var (
    discountAttempts = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "discount_code_attempts_total",
            Help: "Total discount code application attempts",
        },
        []string{"result", "discount_type"},
    )

    discountValue = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "discount_value_dollars",
            Help: "Value of discounts applied",
            Buckets: prometheus.ExponentialBuckets(1, 2, 10),
        },
        []string{"discount_type"},
    )

    discountCodeValidationDuration = prometheus.NewHistogram(
        prometheus.HistogramOpts{
            Name: "discount_code_validation_duration_seconds",
            Help: "Time to validate discount code",
            Buckets: prometheus.DefBuckets,
        },
    )
)

func ApplyDiscountWithMetrics(ctx context.Context, cartID string, code string) error {
    start := time.Now()

    discount, err := validateDiscountCode(code)
    discountCodeValidationDuration.Observe(time.Since(start).Seconds())

    if err != nil {
        discountAttempts.WithLabelValues("invalid", "unknown").Inc()
        return ErrInvalidCode
    }

    discountAttempts.WithLabelValues("success", discount.Type).Inc()

    savings := calculateSavings(cartID, discount)
    discountValue.WithLabelValues(discount.Type).Observe(savings)

    return nil
}
```

### Pattern 3: Distributed tracing at boundaries

```go
package cart

import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
)

var tracer = otel.Tracer("cart-service")

func ApplyDiscountWithTracing(ctx context.Context, cartID string, code string) error {
    // Start span for entire operation
    ctx, span := tracer.Start(ctx, "apply_discount")
    defer span.End()

    span.SetAttributes(
        attribute.String("cart.id", cartID),
        attribute.String("discount.code", code),
    )

    // Child span for validation
    ctx, validateSpan := tracer.Start(ctx, "validate_discount_code")
    discount, err := validateDiscountCode(ctx, code)
    validateSpan.End()

    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "invalid discount code")
        return err
    }

    span.SetAttributes(
        attribute.String("discount.type", discount.Type),
        attribute.Float64("discount.percent", discount.Percent),
    )

    // Child span for applying discount
    ctx, applySpan := tracer.Start(ctx, "calculate_discounted_total")
    newTotal, err := calculateAndApplyDiscount(ctx, cartID, discount)
    applySpan.SetAttributes(
        attribute.Float64("cart.new_total", newTotal),
    )
    applySpan.End()

    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "failed to apply discount")
        return err
    }

    span.SetStatus(codes.Ok, "discount applied")
    return nil
}
```

### Pattern 4: Error enrichment

```go
package cart

import (
    "fmt"

    "github.com/pkg/errors"
)

var (
    ErrInvalidCode = errors.New("invalid discount code")
    ErrExpiredCode = errors.New("discount code expired")
    ErrMaxUsageExceeded = errors.New("discount code max usage exceeded")
)

func validateDiscountCode(code string) (*Discount, error) {
    discount, err := discountDB.Get(code)
    if err != nil {
        return nil, errors.Wrap(err, "failed to fetch discount code")
    }

    if discount == nil {
        return nil, errors.Wrapf(ErrInvalidCode, "code=%s not found", code)
    }

    if discount.ExpiresAt.Before(time.Now()) {
        return nil, errors.Wrapf(
            ErrExpiredCode,
            "code=%s expired at %s",
            code,
            discount.ExpiresAt,
        )
    }

    if discount.UsageCount >= discount.MaxUsage {
        return nil, errors.Wrapf(
            ErrMaxUsageExceeded,
            "code=%s used %d/%d times",
            code,
            discount.UsageCount,
            discount.MaxUsage,
        )
    }

    return discount, nil
}

// Error handling with full context
func ApplyDiscount(ctx context.Context, cartID string, code string) error {
    discount, err := validateDiscountCode(code)
    if err != nil {
        // Log error with full stack trace
        log.WithError(err).WithFields(logrus.Fields{
            "cart_id": cartID,
            "code": code,
            "stack": fmt.Sprintf("%+v", err),
        }).Error("Discount validation failed")

        return errors.Wrapf(err, "apply discount failed for cart=%s", cartID)
    }

    return nil
}
```

### Key insight box
> Observable code emits signals at every decision point: logs for events, metrics for aggregates, traces for causality, structured errors for debugging.

### Challenge question
How much observability is too much? When does instrumentation become noise?

---

## [PUZZLE] The correlation problem - linking signals across pillars

### Scenario
User reports: "My order failed at 2:45 PM."

You have:
- Logs showing errors at 2:45 PM (but which user?)
- Metrics showing error rate spike at 2:45 PM (but why?)
- Traces from 2:45 PM (but which trace is theirs?)

How do you connect the dots?

### Think about it
You need a way to correlate logs, metrics, and traces for the same user request.

### Interactive question (pause and think)
What's the key to correlation?

A. Use the same timestamp
B. Use the same request ID / trace ID
C. Store everything in one database
D. Hope you get lucky

### Progressive reveal
Answer: B - correlation IDs are the glue.

### Correlation ID strategy

```go
package correlation

import (
    "context"
    "net/http"

    "github.com/google/uuid"
)

type ContextKey string

const (
    RequestIDKey ContextKey = "request_id"
    TraceIDKey   ContextKey = "trace_id"
    UserIDKey    ContextKey = "user_id"
)

func WithRequestID(ctx context.Context) context.Context {
    requestID := uuid.New().String()
    return context.WithValue(ctx, RequestIDKey, requestID)
}

func GetRequestID(ctx context.Context) string {
    if rid := ctx.Value(RequestIDKey); rid != nil {
        return rid.(string)
    }
    return "unknown"
}

// Propagate through HTTP headers
func InjectHeaders(ctx context.Context, headers http.Header) {
    headers.Set("X-Request-ID", GetRequestID(ctx))
    headers.Set("X-Trace-ID", GetTraceID(ctx))
    headers.Set("X-User-ID", GetUserID(ctx))
}

func ExtractHeaders(r *http.Request) context.Context {
    ctx := r.Context()

    if requestID := r.Header.Get("X-Request-ID"); requestID != "" {
        ctx = context.WithValue(ctx, RequestIDKey, requestID)
    }

    if traceID := r.Header.Get("X-Trace-ID"); traceID != "" {
        ctx = context.WithValue(ctx, TraceIDKey, traceID)
    }

    return ctx
}
```

### Unified observability with correlation

```go
package observability

import (
    "context"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/sirupsen/logrus"
    "go.opentelemetry.io/otel/trace"
)

type ObservabilityContext struct {
    RequestID string
    TraceID   string
    SpanID    string
    UserID    string
}

func NewObservabilityContext(ctx context.Context) *ObservabilityContext {
    spanContext := trace.SpanContextFromContext(ctx)

    return &ObservabilityContext{
        RequestID: GetRequestID(ctx),
        TraceID:   spanContext.TraceID().String(),
        SpanID:    spanContext.SpanID().String(),
        UserID:    GetUserID(ctx),
    }
}

func (oc *ObservabilityContext) LogFields() logrus.Fields {
    return logrus.Fields{
        "request_id": oc.RequestID,
        "trace_id":   oc.TraceID,
        "span_id":    oc.SpanID,
        "user_id":    oc.UserID,
    }
}

func (oc *ObservabilityContext) MetricLabels() prometheus.Labels {
    return prometheus.Labels{
        "user_tier": getUserTier(oc.UserID), // Low-cardinality!
    }
}

// Usage in handler
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    obsCtx := NewObservabilityContext(ctx)

    // Log with correlation
    log.WithFields(obsCtx.LogFields()).Info("Request started")

    // Metric with correlation
    requestCounter.With(obsCtx.MetricLabels()).Inc()

    // Trace already has IDs from context
    ctx, span := tracer.Start(ctx, "handle_request")
    defer span.End()

    // Now: logs, metrics, traces all share correlation IDs
    // Can jump from metric → exemplar → trace → logs
}
```

### Exemplar: bridging metrics and traces

```go
package exemplar

import (
    "context"

    "github.com/prometheus/client_golang/prometheus"
    "go.opentelemetry.io/otel/trace"
)

var requestDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "http_request_duration_seconds",
    },
    []string{"endpoint", "status"},
)

func ObserveDuration(ctx context.Context, endpoint, status string, duration float64) {
    spanContext := trace.SpanContextFromContext(ctx)

    // Observe with exemplar (links metric to trace)
    requestDuration.WithLabelValues(endpoint, status).(prometheus.ExemplarObserver).
        ObserveWithExemplar(
            duration,
            prometheus.Labels{
                "trace_id": spanContext.TraceID().String(),
                "span_id":  spanContext.SpanID().String(),
            },
        )
}

// Now in Grafana:
// 1. See p99 latency spike in metric
// 2. Click exemplar dot on graph
// 3. Jump directly to trace showing why that request was slow
```

### Key insight box
> Correlation IDs (request_id, trace_id, user_id) are the thread that connects logs, metrics, and traces. Without them, you have isolated data islands.

### Challenge question
Your microservices span 3 languages (Go, Python, Java). How do you ensure consistent correlation ID propagation across all of them?

---

## [DEEP DIVE] ODD in practice - building an observable feature end-to-end

### Scenario
You're tasked with building: "Real-time inventory reservation system."

Requirements:
- Reserve inventory when user adds item to cart
- Release reservation after 15 minutes if not purchased
- Must handle 10K requests/second
- Need to debug: reservation leaks, double bookings, performance issues

### Step 1: Define observability requirements FIRST

```yaml
observability_requirements:

  questions_to_answer:
    - Which users are experiencing reservation failures?
    - Why are reservations being released early/late?
    - Are there inventory leaks (reserved but never released)?
    - What's the p99 latency of reservation operations?
    - Which SKUs have the most contention?

  signals_needed:
    logs:
      - Reservation created (user_id, sku, quantity, expiration)
      - Reservation released (reason: timeout vs purchase)
      - Reservation conflict (available vs requested)

    metrics:
      - reservation_attempts_total{result, sku_category}
      - reservation_duration_seconds{operation}
      - active_reservations_gauge{sku_category}
      - inventory_leaked_total{sku}

    traces:
      - Span: reserve_inventory (sku, quantity attributes)
      - Span: release_reservation (reason attribute)
      - Span: check_inventory (db_query_duration)
```

### Step 2: Implement with observability built-in

```go
package inventory

import (
    "context"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/sirupsen/logrus"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
)

var (
    reservationAttempts = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "reservation_attempts_total",
        },
        []string{"result", "sku_category"},
    )

    activeReservations = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "active_reservations",
        },
        []string{"sku_category"},
    )

    tracer = otel.Tracer("inventory-service")
)

type Reservation struct {
    ID         string
    UserID     string
    SKU        string
    Quantity   int
    ExpiresAt  time.Time
    CreatedAt  time.Time
}

func ReserveInventory(ctx context.Context, userID, sku string, qty int) (*Reservation, error) {
    ctx, span := tracer.Start(ctx, "reserve_inventory")
    defer span.End()

    span.SetAttributes(
        attribute.String("user.id", userID),
        attribute.String("inventory.sku", sku),
        attribute.Int("inventory.quantity", qty),
    )

    obsCtx := NewObservabilityContext(ctx)
    logger := log.WithFields(obsCtx.LogFields()).WithFields(logrus.Fields{
        "sku": sku,
        "quantity": qty,
    })

    logger.Info("Starting inventory reservation")

    // Check available inventory
    ctx, checkSpan := tracer.Start(ctx, "check_available_inventory")
    available, err := getAvailableInventory(ctx, sku)
    checkSpan.End()

    if err != nil {
        logger.WithError(err).Error("Failed to check inventory")
        span.RecordError(err)
        reservationAttempts.WithLabelValues("error_db", getCategory(sku)).Inc()
        return nil, err
    }

    span.SetAttributes(attribute.Int("inventory.available", available))

    if available < qty {
        logger.WithFields(logrus.Fields{
            "available": available,
            "requested": qty,
        }).Warn("Insufficient inventory")

        reservationAttempts.WithLabelValues("insufficient", getCategory(sku)).Inc()
        return nil, ErrInsufficientInventory
    }

    // Create reservation
    reservation := &Reservation{
        ID:        generateReservationID(),
        UserID:    userID,
        SKU:       sku,
        Quantity:  qty,
        ExpiresAt: time.Now().Add(15 * time.Minute),
        CreatedAt: time.Now(),
    }

    ctx, createSpan := tracer.Start(ctx, "persist_reservation")
    err = saveReservation(ctx, reservation)
    createSpan.End()

    if err != nil {
        logger.WithError(err).Error("Failed to persist reservation")
        span.RecordError(err)
        reservationAttempts.WithLabelValues("error_persist", getCategory(sku)).Inc()
        return nil, err
    }

    // Success metrics
    reservationAttempts.WithLabelValues("success", getCategory(sku)).Inc()
    activeReservations.WithLabelValues(getCategory(sku)).Inc()

    logger.WithFields(logrus.Fields{
        "reservation_id": reservation.ID,
        "expires_at": reservation.ExpiresAt,
    }).Info("Inventory reserved successfully")

    return reservation, nil
}
```

### Step 3: Add diagnostic endpoints

```go
package inventory

// Diagnostic endpoint for debugging
func DiagnosticReport(ctx context.Context) (*DiagnosticReport, error) {
    report := &DiagnosticReport{}

    // Count active reservations
    report.ActiveReservations = countActiveReservations(ctx)

    // Find expired reservations (leaks)
    report.ExpiredReservations = findExpiredReservations(ctx)

    // Calculate inventory discrepancies
    report.InventoryDiscrepancies = checkInventoryConsistency(ctx)

    log.WithFields(logrus.Fields{
        "active": report.ActiveReservations,
        "expired": len(report.ExpiredReservations),
        "discrepancies": len(report.InventoryDiscrepancies),
    }).Info("Diagnostic report generated")

    return report, nil
}

// Health check endpoint
func HealthCheck(ctx context.Context) error {
    if err := pingDatabase(ctx); err != nil {
        return errors.Wrap(err, "database unreachable")
    }

    expired := countExpiredReservations(ctx)
    if expired > 1000 {
        log.WithField("expired_count", expired).Warn("Excessive expired reservations")
        return errors.Errorf("inventory leak detected: %d expired", expired)
    }

    return nil
}

// Background monitoring
func MonitorInventoryHealth(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            report, err := DiagnosticReport(ctx)
            if err != nil {
                log.WithError(err).Error("Diagnostic report failed")
                continue
            }

            if len(report.ExpiredReservations) > 100 {
                log.WithField("count", len(report.ExpiredReservations)).
                    Error("Inventory leak detected")

                // Auto-remediate
                releaseExpiredReservations(ctx, report.ExpiredReservations)
            }

        case <-ctx.Done():
            return
        }
    }
}
```

### Key insight box
> ODD means designing observability INTO the feature from day one, not bolting it on after production issues surface.

### Challenge question
You've built perfect observability into your feature. But other teams don't follow these patterns. How do you scale ODD across an organization?

---

## [WARNING] Common ODD anti-patterns to avoid

### Scenario
Your team claims to practice ODD, but observability is still poor during incidents.

### Anti-patterns catalog

**Anti-pattern 1: Log everything (noise)**
```go
// BAD: Excessive logging
func ProcessOrder(order *Order) {
    log.Info("Starting order processing")
    log.Info("Validating order")
    log.Info("Order validated")
    log.Info("Checking inventory")
    log.Info("Inventory checked")
    log.Info("Processing payment")
    log.Info("Payment processed")
    log.Info("Order processed")
}
// 99% of these logs are useless noise

// GOOD: Log at decision points and errors
func ProcessOrder(order *Order) {
    logger := log.WithField("order_id", order.ID)

    if err := validateOrder(order); err != nil {
        logger.WithError(err).Error("Order validation failed")
        return err
    }

    if !hasInventory(order) {
        logger.Warn("Insufficient inventory")
        return ErrNoInventory
    }

    logger.Info("Order processed successfully")
}
```

**Anti-pattern 2: Metrics without purpose**
```go
// BAD: Metric that's never queried
var obscureCounter = prometheus.NewCounter(
    prometheus.CounterOpts{
        Name: "some_internal_thing_total",
    },
)
// No dashboard uses this
// No alerts on it
// Just wasting storage

// GOOD: Metrics tied to SLOs
var checkoutSuccessRate = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "checkout_attempts_total",
    },
    []string{"result"},
)
// Used in SLO dashboard
// Alerts when success_rate < 99%
```

**Anti-pattern 3: Inconsistent naming**
```go
// BAD: Different names across services
// Service A: "request_id"
// Service B: "requestId"
// Service C: "req_id"
// Impossible to grep logs

// GOOD: Standardized naming
// All services: "request_id", "trace_id", "user_id"
```

**Anti-pattern 4: Dropped context**
```go
// BAD: Creating new context loses correlation
func CallDownstream() {
    ctx := context.Background() // Lost trace!
    http.Get("http://downstream/api")
}

// GOOD: Propagate context
func CallDownstream(ctx context.Context) {
    req, _ := http.NewRequestWithContext(ctx, "GET", "http://downstream/api", nil)
    InjectHeaders(ctx, req.Header)
    http.DefaultClient.Do(req)
}
```

**Anti-pattern 5: PII in logs**
```go
// BAD: Sensitive data
log.WithFields(logrus.Fields{
    "email": user.Email,
    "password": password, // NEVER!
    "credit_card": card.Number, // NEVER!
}).Info("User login")

// GOOD: Hash or redact PII
log.WithFields(logrus.Fields{
    "user_id": user.ID,
    "email_hash": hash(user.Email),
}).Info("User login")
```

### Key insight box
> ODD anti-patterns: excessive logging (noise), vanity metrics (unused), inconsistent naming (chaos), dropped context (broken correlation), PII exposure (compliance violation).

### Challenge question
Your logs accidentally contained customer emails. How do you scrub this PII from historical data?

---

## [SYNTHESIS] Final synthesis - ODD transformation for a team

### Synthesis challenge
You're the engineering lead for 15 developers building a fintech app.

Current state:
- No distributed tracing
- Unstructured text logs (grep debugging)
- Metrics exist but unused
- Incidents take 4+ hours to debug
- Developers hate on-call

Requirements:
- Reduce MTTR from 4 hours to 30 minutes
- Make on-call sustainable
- Pass SOC2 audit (need audit trail)
- Maintain development velocity

### Your tasks (pause and think)
1. Define ODD standards
2. Create observability onboarding checklist
3. Plan retrofit for existing services
4. Measure ODD adoption
5. Make ODD part of code review
6. Justify investment to leadership

Write down your transformation plan.

### Progressive reveal (solution)

**1. ODD standards:**
```yaml
observability_standards:

  structured_logging:
    format: JSON (logrus)
    required_fields:
      - timestamp (ISO 8601)
      - level
      - message
      - service_name
      - request_id
      - trace_id
      - user_id (hashed)

  metrics:
    naming: "{service}_{metric}_{unit}"
    required_per_service:
      - {service}_requests_total{endpoint, status}
      - {service}_request_duration_seconds{endpoint}
      - {service}_errors_total{error_type}
    labels: max 5, low cardinality only

  distributed_tracing:
    sdk: OpenTelemetry
    sampling: 10% head, tail-based collector
    required_spans:
      - HTTP requests (auto)
      - Database queries (auto)
      - Business operations (manual)

  context_propagation:
    headers:
      - X-Request-ID
      - X-Trace-ID
      - X-User-ID (hashed)
    must_propagate: all HTTP/gRPC calls
```

**2. Observability checklist:**
```markdown
# New Feature Observability Checklist

Before merging:

## Logging
- [ ] Errors logged with context (user_id, operation, details)
- [ ] State transitions logged (created, updated, deleted)
- [ ] Logs are structured JSON
- [ ] No PII in logs

## Metrics
- [ ] Success/failure counter
- [ ] Latency histogram (if >100ms)
- [ ] Low-cardinality labels only

## Tracing
- [ ] Spans for network calls
- [ ] Spans for expensive operations
- [ ] Business attributes in spans
- [ ] Context propagated downstream

## Testing
- [ ] Can debug locally with logs
- [ ] Metrics in Prometheus
- [ ] Traces in Tempo
- [ ] Correlation IDs present

## Documentation
- [ ] Runbook updated
- [ ] Dashboard created (if critical)
- [ ] Alerts defined (if SLO-critical)
```

**3. Retrofit plan:**
```yaml
retrofit_timeline:

  phase_1_quick_wins: # 2 weeks
    - Deploy OpenTelemetry Collector
    - Auto-instrument all services
    - Standardize log format
    - Deploy Grafana + Tempo
    impact: 30% MTTR improvement

  phase_2_manual: # 6 weeks
    - Add business spans per service
    - Structured logging on error paths
    - Recording rules
    - Service dashboards
    impact: 50% MTTR improvement

  phase_3_optimization: # 4 weeks
    - Tail-based sampling
    - Top 10 incident runbooks
    - SLOs and alerts
    - Team training
    impact: 70% MTTR improvement

  total: 3 months
  result: 4 hours → 1.2 hours MTTR
```

**4. Adoption metrics:**
```yaml
measure_adoption:

  leading_indicators:
    - % services with structured logs (target: 100%)
    - % services with tracing (target: 100%)
    - % PRs passing checklist (target: 95%)

  lagging_indicators:
    - MTTR: 4 hours → 30 minutes
    - MTTD: 20 minutes → 2 minutes
    - On-call satisfaction: 3/10 → 7/10
    - % incidents debugged without escalation: 20% → 80%
```

**5. Code review culture:**
```yaml
code_review_for_observability:

  checklist:
    - "Can I debug this at 3 AM without source code?"
    - "Are errors logged with context?"
    - "Are metrics emitted?"
    - "Is context propagated?"
    - "Any high-cardinality labels?"
    - "Is PII redacted?"

  automated_checks:
    - Linter: require structured logging
    - Linter: require context.Context
    - CI: fail on high-cardinality metrics
    - CI: fail on PII patterns
```

**6. Leadership justification:**
```yaml
business_case:

  current_costs:
    - Incidents: 10/month × 4 hours × 3 engineers × $100/hr = $12K/month
    - Customer churn: 1% due to downtime = $50K/month
    - Developer attrition: unsustainable on-call
    total: $62K/month

  investment:
    - Tools: $10K/month (Grafana Cloud, Tempo)
    - Engineering: 6 engineer-months one-time
    - Ongoing: 10% dev time = 1.5 engineers
    total: $15K/month recurring

  return:
    - 70% MTTR reduction: $8K/month saved
    - Fewer outages: $25K/month saved
    - Better retention: $10K/month saved
    total: $43K/month benefit

  roi: $43K - $15K = $28K/month net benefit
  payback: 6 months

  pitch:
    "Investing $15K/month saves $43K/month in incident costs
     and churn. That's 3x ROI, plus developers won't quit."
```

### Key insight box
> ODD transformation is cultural change: measure impact (MTTR), enforce standards (code review), demonstrate ROI (business case).

### Final challenge question
After 6 months, adoption is 60% (not 100%). Some teams resist, saying "observability slows us down." How do you overcome this?

---

## Appendix: Quick checklist (printable)

**ODD feature checklist:**
- [ ] Define observability requirements before coding
- [ ] Add structured logging with context
- [ ] Emit metrics at decision points
- [ ] Instrument with distributed tracing
- [ ] Propagate correlation IDs
- [ ] Write debugging runbook
- [ ] Create dashboard (if critical)
- [ ] Define alerts (if SLO-critical)

**Observability standards:**
- [ ] All logs structured JSON
- [ ] All logs include request_id, trace_id
- [ ] All metrics low-cardinality
- [ ] All traces include business attributes
- [ ] All errors logged with context
- [ ] All PII redacted/hashed

**Code review checklist:**
- [ ] Can I debug at 3 AM?
- [ ] Errors logged with context?
- [ ] Metrics emitted?
- [ ] Context propagated?
- [ ] No high-cardinality labels?
- [ ] No PII exposure?

**Team adoption:**
- [ ] Training for all engineers
- [ ] Observability advocate per team
- [ ] Automated linting
- [ ] In definition of done
- [ ] Monthly reviews

**Measurement:**
- [ ] Track MTTR (target <30 min)
- [ ] Track MTTD (target <5 min)
- [ ] Track % services observable
- [ ] Track on-call satisfaction
- [ ] Track self-service debugging rate

**Red flags:**
- [ ] MTTR increasing (observability degrading)
- [ ] New features without observability
- [ ] High-cardinality explosion
- [ ] PII in logs/traces
- [ ] Checklist bypassed
- [ ] No log/metric/trace correlation
