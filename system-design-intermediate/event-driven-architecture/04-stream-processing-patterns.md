---
slug: stream-processing-patterns
title: Stream Processing Patterns
readTime: 10 min
orderIndex: 4
premium: false
---

## 💪 Stream Processing Patterns

Stream processing involves continuously ingesting, transforming, and analyzing data as it flows through your system. Understanding these core patterns is essential for building robust real-time data pipelines.

---

### Stateless Processing

Stateless processing transforms each event independently without needing to remember previous events. This is the simplest and most scalable pattern since each event can be processed in isolation.

```python
def process_click_event(event):
    """Transform a raw click event into an enriched format."""
    return {
        'userId': event['userId'],
        'page': event['page'],
        'timestamp': event['timestamp'],
        'device': detect_device(event['userAgent'])
    }

# Process stream - each event handled independently
for event in stream:
    transformed = process_click_event(event)
    output_stream.write(transformed)
```

---

### Stateful Processing (Aggregations)

Stateful processing maintains information across multiple events, enabling aggregations, windowing, and pattern detection. This pattern is more complex but necessary for analytics like counting, averaging, or detecting trends.

```python
from collections import defaultdict
from datetime import datetime, timedelta

# Count clicks per user in 5-minute windows
window_size = timedelta(minutes=5)
windows = defaultdict(lambda: {'count': 0, 'start': None})

def process_with_windowing(event):
    """Aggregate click counts within time windows."""
    user_id = event['userId']
    timestamp = datetime.fromisoformat(event['timestamp'])

    # Get or create window for this user
    if windows[user_id]['start'] is None:
        windows[user_id]['start'] = timestamp

    # Check if event falls within current window
    if timestamp - windows[user_id]['start'] < window_size:
        windows[user_id]['count'] += 1
    else:
        # Window complete - emit aggregated result
        yield {
            'userId': user_id,
            'clickCount': windows[user_id]['count'],
            'windowStart': windows[user_id]['start']
        }
        # Start new window
        windows[user_id] = {'count': 1, 'start': timestamp}

# Process stream with windowing
for event in stream:
    for result in process_with_windowing(event):
        output_stream.write(result)
```

---

### Stream Joins

Stream joins combine events from multiple streams based on matching criteria and time windows. This pattern is essential for correlating related events, such as tracking user journeys from click to purchase.

```python
from datetime import timedelta

# Define source streams
clicks = stream('clicks')
purchases = stream('purchases')

def join_streams(click_event, purchase_event):
    """Join click and purchase events to track conversions."""
    # Match by user, product, and time proximity
    if (click_event['userId'] == purchase_event['userId'] and
        click_event['productId'] == purchase_event['productId'] and
        purchase_event['timestamp'] - click_event['timestamp'] < timedelta(hours=1)):

        return {
            'userId': click_event['userId'],
            'productId': click_event['productId'],
            'clickTimestamp': click_event['timestamp'],
            'purchaseTimestamp': purchase_event['timestamp'],
            'timeToPurchase': purchase_event['timestamp'] - click_event['timestamp']
        }

# Kafka Streams implementation
builder = StreamsBuilder()
clicks_stream = builder.stream('clicks')
purchases_stream = builder.stream('purchases')

joined = clicks_stream.join(
    purchases_stream,
    joiner=lambda click, purchase: join_streams(click, purchase),
    window=JoinWindows.of(Duration.ofHours(1))
)

joined.to('conversion-events')
```

---

### Real-World Analogies

| Pattern | Analogy | Use Case |
|---------|---------|----------|
| **Stateless** | Assembly line worker | Data transformation, enrichment, filtering |
| **Stateful** | Cashier tallying sales | Aggregations, counting, windowed analytics |
| **Joins** | Detective connecting clues | Conversion tracking, user journey analysis |

---

## 🛡️ Handling Late and Out-of-Order Events

In distributed systems, events often arrive out of order due to network delays, system failures, or geographic distribution. Handling this correctly is crucial for accurate stream processing.

### The Problem

Events rarely arrive in the order they occurred:

```
Expected order:
[Event 1 @ 10:00] → [Event 2 @ 10:01] → [Event 3 @ 10:02]

Actual arrival:
[Event 1 @ 10:00] → [Event 3 @ 10:02] → [Event 2 @ 10:01] (late!)
```

---

### Solution 1: Watermarks

Watermarks define "how late is acceptable" by establishing a threshold beyond which events are considered too old to process. This allows the system to make progress while handling reasonable delays.

```
Watermark = Current time - 5 minutes

Events relative to watermark:
├─ On time    → Process normally
└─ Late       → Discard (or route to late-events stream)

Example:
  Current time: 10:10
  Watermark: 10:05

  Event with timestamp 10:03 arrives → Too late! Discard
  Event with timestamp 10:07 arrives → Process ✓
```

---

### Solution 2: Grace Period

Grace periods keep windows open for a defined time after they "end," allowing late-arriving events to still be included before final results are emitted.

```
Window: 10:00 - 10:05
Grace period: 2 minutes
Close window at: 10:07

Timeline:
  10:05 - Window "ends" but stays open for grace period
  10:06 - Late event arrives → Still accepted ✓
  10:07 - Grace period expires → Window closed, emit final results
```

---

### Solution 3: Event Time vs Processing Time

Distinguishing between when an event *happened* versus when it was *processed* is fundamental to accurate stream processing.

```
Event Time      = When the event actually occurred
Processing Time = When the system processes it

Example:
  Event: Click at 9:59 AM
  Arrives at: 10:01 AM (network delay)

  ✓ Count for 9:00-10:00 window (based on event time)
  ✗ Not 10:00-11:00 window (based on processing time)
```

```python
# Correct: Use event time for windowing
window = event['timestamp']  # Event time ✓

# Incorrect: Using processing time leads to inaccurate results
# window = now()  # Processing time ✗
```

---

### Real-World Analogies

| Solution | Analogy |
|----------|---------|
| **Watermarks** | Postal deadlines (no Christmas cards accepted after Dec 20) |
| **Grace Period** | Late assignment submissions (accepted with conditions) |
| **Event Time** | When you wrote the letter vs. when it was delivered |

---

## 💡 Final Synthesis: The Time Machine

**Complete this comparison:** "Traditional databases are like a photograph of right now. Event streams are like..."

### The Complete Picture

**Event streams are like a movie recording of everything that ever happened:**

| Capability | Description |
|------------|-------------|
| ✅ Never delete frames | Immutable history preserved forever |
| ✅ Rewind and replay | Time travel to any point in history |
| ✅ Multiple viewers | Parallel consumers read independently |
| ✅ Timestamp queries | See exactly what happened at any moment |
| ✅ Complete audit trail | Full compliance and debugging capability |
| ✅ Add new viewers anytime | New consumers can join and catch up |
| ✅ Variable playback speed | Read at any pace (batch or real-time) |
| ✅ Derive current state | Rebuild state from complete history |

### Industry Examples

| Company | Use Case |
|---------|----------|
| **LinkedIn** | Activity streams (invented Kafka!) |
| **Netflix** | Viewing history and recommendations |
| **Uber** | Real-time ride events and tracking |
| **Banks** | Transaction event sourcing for audit trails |

> Event streams transform ephemeral data into permanent, replayable business history!

---

## 🎯 Quick Recap: Test Your Understanding

Without looking back, can you explain:

1. **How do event streams differ from traditional databases?**
2. **What is event sourcing and its benefits?**
3. **How do multiple consumers read from the same stream?**
4. **Why is event ordering important in streams?**

*If you can answer these clearly, you've mastered event stream fundamentals!*

---

## 🚀 Your Next Learning Adventure

Now that you understand Event Streams, explore these advanced topics:

### Advanced Streaming
- Stream processing with Kafka Streams
- Apache Flink for complex event processing
- Exactly-once semantics in streaming
- Stream-table duality

### Event Sourcing Deep Dive
- Snapshotting for performance optimization
- Event versioning strategies
- Handling schema evolution gracefully
- Projections and read models

### Stream Technologies
- Kafka Connect (integrate external systems)
- Schema Registry (manage event schemas)
- ksqlDB (SQL interface for streams)
- Debezium (Change Data Capture connector)

### Real-World Patterns
- Building event-sourced microservices
- Real-time analytics pipelines
- Stream processing at scale
- CQRS in production environments
