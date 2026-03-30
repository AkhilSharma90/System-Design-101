---
slug: event-stream-patterns
title:  Event Stream Patterns
readTime: 10 min
orderIndex: 2
premium: false
---


# 📊 Event Stream Patterns

Event streams enable powerful architectural patterns that solve complex distributed system challenges. Let's explore four fundamental patterns and understand when to use each one.

---

## Pattern 1: Event Sourcing

### Overview

Event sourcing stores events as the source of truth, then derives the current state by replaying those events. Instead of storing the current state directly, you store all the changes that led to that state.

### Example: Bank Account

**Event Stream:**

```
[AccountCreated: initial: $0]
[MoneyDeposited: +$1000]
[MoneyWithdrawn: -$200]
[MoneyDeposited: +$500]
```

**Current State (computed):**

```
Balance = $0 + $1000 - $200 + $500 = $1,300 ✓
```

### Benefits

- ✅ **Complete audit trail** - Every change is recorded permanently
- ✅ **Time travel** - Reconstruct state at any point in history
- ✅ **No data loss** - Events are never deleted or modified
- ✅ **Can add new views anytime** - Project events into new models as needed

### Code Example

```python
class BankAccount:
    def __init__(self):
        self.balance = 0
        self.events = []

    def deposit(self, amount):
        """Record a deposit event"""
        event = {
            'type': 'MoneyDeposited',
            'amount': amount,
            'timestamp': now()
        }

        # Apply the event to update state
        self.apply(event)

        # Store locally
        self.events.append(event)

        # Persist to event stream
        stream.append(event)

    def withdraw(self, amount):
        """Record a withdrawal event"""
        event = {
            'type': 'MoneyWithdrawn',
            'amount': amount,
            'timestamp': now()
        }

        self.apply(event)
        self.events.append(event)
        stream.append(event)

    def apply(self, event):
        """Apply an event to update the current state"""
        if event['type'] == 'MoneyDeposited':
            self.balance += event['amount']
        elif event['type'] == 'MoneyWithdrawn':
            self.balance -= event['amount']

    @classmethod
    def rebuild_from_stream(cls, events):
        """Reconstruct account state from event history"""
        account = cls()
        for event in events:
            account.apply(event)
        return account


# Time travel example
events_at_noon = stream.read_until('2024-01-15T12:00:00Z')
account_state_at_noon = BankAccount.rebuild_from_stream(events_at_noon)

print(f"Balance at noon: ${account_state_at_noon.balance}")
```

### When to Use Event Sourcing

- **Financial systems** - Where audit trails are mandatory
- **Regulatory compliance** - When you need to prove what happened
- **Complex business logic** - Where understanding the sequence of events matters
- **Temporal queries** - When you need to answer "what was the state at time X?"

---

## Pattern 2: Change Data Capture (CDC)

### Overview

Change Data Capture monitors database changes and publishes them as events to a stream. This allows other systems to react to database changes in real-time without polling.

### How It Works

**Database Operation:**

```sql
UPDATE users SET email = 'new@example.com' WHERE id = 123;
```

**↓ CDC Captures the Change ↓**

**Event Stream Receives:**

```json
{
  "operation": "UPDATE",
  "table": "users",
  "before": {
    "id": 123,
    "email": "old@example.com"
  },
  "after": {
    "id": 123,
    "email": "new@example.com"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Consumers React to Database Changes

```
Database Change Event
├─ Search index updates (Elasticsearch)
├─ Cache invalidation (Redis)
├─ Data warehouse sync (Snowflake)
└─ Notification services (Email/SMS)
```

Each consumer processes the change independently and at their own pace.

### Benefits

- ✅ **Real-time data propagation** - Changes flow immediately to dependent systems
- ✅ **Decoupled systems** - Database doesn't need to know about consumers
- ✅ **Guaranteed delivery** - Events are not lost if consumers are temporarily down
- ✅ **Historical replay** - Can reprocess changes if needed

### CDC Technologies

- **Debezium** - Open-source CDC connector for various databases
- **AWS DMS** - Database Migration Service with CDC capabilities
- **Maxwell** - Reads MySQL binlogs and writes to Kafka
- **Oracle GoldenGate** - Enterprise CDC solution

### When to Use CDC

- **Search index synchronization** - Keep Elasticsearch in sync with your database
- **Cache invalidation** - Automatically invalidate cache entries when data changes
- **Data replication** - Sync data across multiple databases or data centers
- **Event-driven microservices** - Trigger actions based on database changes

---

## Pattern 3: CQRS (Command Query Responsibility Segregation)

### Overview

CQRS separates the write model (commands) from the read model (queries). The write side optimizes for consistency and validation, while the read side optimizes for query performance.

### Architecture Flow

**Write Side (Commands):**

```
Command: "Place Order"
   ↓
Validate business rules
   ↓
[OrderPlaced] → Event Stream
   ↓
Update Write DB (normalized)
```

**Read Side (Queries):**

```
Event: [OrderPlaced]
   ↓
Update Read DB (denormalized)
   ↓
Query: "Get order details"
   ↓
Read from optimized Read DB
```

### Example: E-commerce Order

**Write Model (Normalized):**

```sql
-- Optimized for consistency
orders: id, customer_id, status, created_at
order_items: id, order_id, product_id, quantity
products: id, name, price
```

**Read Model (Denormalized):**

```json
// Optimized for queries
{
  "orderId": "123",
  "customerName": "Alice Johnson",
  "customerEmail": "alice@example.com",
  "items": [
    {
      "productName": "Laptop",
      "price": 999.99,
      "quantity": 1
    }
  ],
  "totalAmount": 999.99,
  "status": "shipped",
  "trackingNumber": "ABC123"
}
```

### Benefits

- ✅ **Optimize reads separately from writes** - Each side uses appropriate data structures
- ✅ **Multiple read models** - Different views for different use cases
- ✅ **Scale reads independently** - Add more read replicas without affecting writes
- ✅ **Better performance** - Denormalized reads are faster, normalized writes maintain integrity

### Trade-offs

**Eventual consistency:** Read models may be slightly behind write models (typically milliseconds to seconds).

**Complexity:** Maintaining separate models adds architectural complexity.

### When to Use CQRS

- **High read/write ratio** - When reads vastly outnumber writes
- **Complex queries** - When queries require joins across many tables
- **Multiple views** - When different users need different data representations
- **Performance requirements** - When you need to scale reads independently

---

## Pattern 4: Stream Processing

### Overview

Stream processing analyzes events in real-time as they flow through the stream, enabling immediate reactions to patterns, anomalies, or thresholds.

### Example: User Activity Monitoring

**Input Event Stream:**

```
[Click: user=123, page=/home, timestamp=10:00:00] →
[Click: user=123, page=/products, timestamp=10:00:05] →
[Purchase: user=456, amount=99.99, timestamp=10:00:10] →
[Click: user=123, page=/cart, timestamp=10:00:15] →
...
```

**↓ Stream Processor ↓**

**Processing Logic:**

```
├─ Window: Last 5 minutes
├─ Count clicks per user
├─ Detect patterns (e.g., cart abandonment)
├─ Calculate conversion rates
└─ Generate alerts for anomalies
```

**↓ Output Stream ↓**

**Processed Events:**

```
[UserActiveAlert: user=123 had 50 clicks in 5 minutes]
[ConversionEvent: user=456 purchased after 3 clicks]
[AbandonmentAlert: user=789 added to cart but left]
```

### Stream Processing Operations

**Windowing:**
```python
# Tumbling window: Non-overlapping 5-minute windows
stream.window(size=5.minutes, type='tumbling')

# Sliding window: Overlapping windows
stream.window(size=5.minutes, slide=1.minute)

# Session window: Activity-based windows
stream.window(gap=30.minutes, type='session')
```

**Aggregations:**
```python
# Count events by key
stream.group_by('userId').count()

# Calculate averages
stream.group_by('productId').average('price')

# Custom aggregations
stream.group_by('userId').aggregate(custom_function)
```

**Joins:**
```python
# Join two streams
clicks_stream.join(
    purchases_stream,
    key='userId',
    window=5.minutes
)
```

### Stream Processing Technologies

```
├─ Apache Kafka Streams
│  └─ Simple library, tight Kafka integration
│
├─ Apache Flink
│  └─ Low latency, exactly-once processing
│
├─ Apache Spark Streaming
│  └─ Micro-batch processing, great for analytics
│
└─ AWS Kinesis Analytics
   └─ Managed service, SQL-based processing
```

### Use Cases

**Real-time Analytics:**
- Dashboard metrics
- Live leaderboards
- Traffic monitoring

**Fraud Detection:**
- Suspicious transaction patterns
- Unusual user behavior
- Account takeover detection

**IoT Data Processing:**
- Sensor data aggregation
- Anomaly detection
- Predictive maintenance

**Personalization:**
- Real-time recommendations
- Dynamic content
- A/B test results

### When to Use Stream Processing

- **Real-time insights** - When decisions need to be made immediately
- **Pattern detection** - When you need to identify trends across multiple events
- **Aggregations** - When you need to compute metrics over time windows
- **Complex event processing** - When one event triggers multiple downstream actions

---

## 🎯 Real-World Parallels

Understanding these patterns through everyday analogies:

### Event Sourcing
**Like:** An accounting ledger
**Why:** Every transaction is recorded; you can calculate balance at any point

### Change Data Capture (CDC)
**Like:** Security camera footage
**Why:** Captures all changes automatically; can review what happened

### CQRS
**Like:** A restaurant operation
**Why:** Separate kitchen (write) from dining area (read); each optimized for its purpose

### Stream Processing
**Like:** Real-time stock ticker
**Why:** Continuously processes new data and shows live results

---

## 🤔 Choosing the Right Pattern

### Decision Matrix

| Requirement | Best Pattern |
|-------------|--------------|
| Need complete audit trail | Event Sourcing |
| Sync multiple systems | CDC |
| Very high read load | CQRS |
| Real-time analytics | Stream Processing |
| Time travel queries | Event Sourcing |
| React to database changes | CDC |
| Multiple read views | CQRS |
| Pattern detection | Stream Processing |

### Can You Combine Patterns?

**Absolutely!** Many systems use multiple patterns together:

```
Event Sourcing + CQRS:
  Events are source of truth → Multiple optimized read models

CDC + Stream Processing:
  Database changes → Real-time analytics

Event Sourcing + Stream Processing:
  Event stream → Real-time aggregations and alerts
```

---

## 🎓 Key Takeaways

1. **Event Sourcing** preserves complete history by storing all state changes as events
2. **CDC** automatically captures and publishes database changes without application code changes
3. **CQRS** separates reads and writes for independent optimization and scaling
4. **Stream Processing** enables real-time analysis and reactions to event patterns
5. These patterns work together and can be combined based on your requirements
6. Choose patterns based on your specific needs: audit requirements, performance characteristics, and consistency requirements

---

## 📚 Further Reading

- **Event Sourcing:** Martin Fowler's article on Event Sourcing patterns
- **CDC:** Debezium documentation and use cases
- **CQRS:** Greg Young's CQRS documents
- **Stream Processing:** Kafka Streams documentation, Flink architecture guide
