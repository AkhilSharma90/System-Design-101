---
slug: event-stream-patterns
title:  Event Stream Patterns
readTime: 10 min
orderIndex: 2
premium: false
---

## **📊 Event Stream Patterns**##

Pattern 1: Event Sourcing

Store events as source of truth, derive state:

Event Stream:

\[AccountCreated: initial: $0\]

\[MoneyDeposited: \+$1000\]

\[MoneyWithdrawn: \-$200\]

\[MoneyDeposited: \+$500\]

Current State (computed):

Balance \= $0 \+ $1000 \- $200 \+ $500 \= $1,300 ✓

Benefits:

✅ Complete audit trail

✅ Time travel (state at any point)

✅ No data loss

✅ Can add new views anytime

Code example:
```python

 class BankAccount:
  def __init__(self):
    self.balance = 0
    self.events = []
  def deposit(self, amount):
    event = {
    'type': 'MoneyDeposited',
    'amount': amount,
    'timestamp': now()
    }
    self.apply(event)
    self.events.append(event)
    stream.append(event)
    # Store in stream
   def apply(self, event):
    if event['type'] == 'MoneyDeposited':

    self.balance += event['amount']

    elif event['type'] == 'MoneyWithdrawn':

      self.balance -= event['amount']

  @classmethod
  def rebuild_from_stream(cls, events):

    account = cls()
    for event in events:
      account.apply(event)
      return account

# Time travel
events_at_noon = stream.read_until('2024-01-15T12:00:00Z')
account_state_at_noon = BankAccount.rebuild_from_stream(events_at_noon)
```


Pattern 2: Change Data Capture (CDC)

Capture database changes as events:

Database:
```sql

UPDATE users SET email = 'new@example.com' WHERE id = 123;
```
           ↓

CDC captures change:

           ↓

Event Stream:
```json
{
  "operation": "UPDATE",
  "table": "users",
  "before": {"id": 123, "email": "old@example.com"},
  "after": {"id": 123, "email": "new@example.com"},
  "timestamp": "2024-01-15T10:30:00Z"
}
```

Consumers react to database changes:

├─ Search index updates

├─ Cache invalidation

├─ Data warehouse sync

└─ Notification services

Pattern 3: CQRS (Command Query Responsibility Segregation)

Separate write and read models:

Write Side (Commands):

Command: "Place Order"

   ↓

[OrderPlaced] → Event Stream

   ↓

Update Write DB (normalized)

Read Side (Queries):

Event: [OrderPlaced]

   ↓

Update Read DB (denormalized)

   ↓

Query: "Get order"

   ↓

Read from optimized Read DB

Benefits:

✅ Optimize reads separately from writes

✅ Multiple read models for different views

✅ Scale reads independently

Pattern 4: Stream Processing

Process events in real-time:

Event Stream:

\[Click\] → \[Click\] → \[Purchase\] → \[Click\] → ...


   ↓
Stream Processor:

├─ Window: Last 5 minutes

├─ Count clicks per user

├─ Detect patterns

└─ Generate alerts

Output Stream:

\[UserActiveAlert: user 123 had 50 clicks\]

\[ConversionEvent: user 456 purchased\]

Technologies:

├─ Apache Kafka Streams

├─ Apache Flink

├─ Apache Spark Streaming

└─ AWS Kinesis Analytics

Real-world parallel:

* Event Sourcing = Accounting ledger (all transactions)

* CDC = Security camera (captures all changes)

* CQRS = Restaurant (separate kitchen and dining)

* Stream Processing = Real-time stock ticker
