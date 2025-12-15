---
slug: stream-processing-patterns
title: Stream Processing Patterns
readTime: 10 min
orderIndex: 4
premium: false
---


## **💪 Stream Processing Patterns** ##

Stateless Processing:

# Transform each event independently
```python

def process_click_event(event):
  return {        'userId': event['userId'],        'page': event['page'],
  'timestamp': event['timestamp'],
  'device': detect_device(event['userAgent'])
  }
  # Process stream
  for event in stream:
    transformed = process_click_event(event)   output_stream.write(transformed)

```

Stateful Processing (Aggregations):
```python

from collections import defaultdict
from datetime import datetime, timedelta
# Count clicks per user in 5-minute windows
window_size = timedelta(minutes=5)
windows = defaultdict(lambda: {'count': 0, 'start': None})

def process_with_windowing(event):
  user_id = event['userId']
  timestamp = datetime.fromisoformat(event['timestamp'])
  # Get or create window
  if windows [user_id]['start'] is None:
    windows[user_id]['start'] = timestamp
    # Check if event in current window
    if timestamp - windows[user_id]['start'] <

        window_size:windows[user_id]['count']+= 1
    else:
        # Window complete, emit result
        yield {            'userId': user_id,           'clickCount': windows[user_id]['count'],
        'windowStart': windows[user_id]['start']
        }
        # Start new window
        windows[user_id] = {'count': 1, 'start': timestamp}
        # Process stream
        for event in stream:
            for result in process_with_windowing(event):
                output_stream.write(result)


```

Stream Joins:

# Join clicks with purchases
```python
clicks = stream('clicks')
purchases = stream('purchases')
def join_streams(click_event, purchase_event):

  if (click_event['userId'] == purchase_event['userId'] and  click_event ['productId'] == purchase_event['productId'] and        purchase_event ['timestamp'] - click_event['timestamp'] < timedelta(hours=1)):

  return {            'userId': click_event['userId'],            'productId': click_event['productId'],'clickTimestamp': click_event['timestamp'], 'purchaseTimestamp': purchase_event['timestamp'], 'timeToPurchase': purchase_event['timestamp'] - click_event['timestamp']}
  # Kafka Streams example
  builder = StreamsBuilder()
  clicks_stream = builder.stream('clicks')
  purchases_stream = builder.stream('purchases')

  joined = clicks_stream.join(    purchases_stream,

  joiner=lambda click, purchase: join_streams(click, purchase),
  window=JoinWindows.of(Duration.ofHours(1)))
  joined.to('conversion-events')

  ```
  Real-world parallel:

* Stateless \= Assembly line worker (each item independent)
* Stateful \= Cashier tallying sales (needs to remember)
* Joins \= Detective connecting clues

🛡️ Handling Late and Out-of-Order Events

The Problem:

Events arrive out of order:

Expected order:
\[Event 1 @ 10:00\] → \[Event 2 @ 10:01\] → \[Event 3 @ 10:02\]

Actual arrival:
\[Event 1 @ 10:00\] → \[Event 3 @ 10:02\] → \[Event 2 @ 10:01\] (late\!)

How to handle?

Solution 1: Watermarks

Define "how late is acceptable":

Watermark \= Current time \- 5 minutes

Events before watermark:

├─ On time → Process

└─ Late → Discard (or send to late events stream)


Example:
Current time: 10:10
Watermark: 10:05
Event with timestamp 10:03 arrives → Too late\! Discard
Event with timestamp 10:07 arrives → Process ✓

Solution 2: Grace Period

Wait for late events before finalizing window:

Window: 10:00 \- 10:05
Grace period: 2 minutes
Close window at: 10:07

Timeline:
10:05 \- Window "ends" but stays open
10:06 \- Late event arrives → Still accepted
10:07 \- Grace period over → Window closed, emit results

Solution 3: Event Time vs Processing Time

Event Time \= When event actually happened
Processing Time \= When system processes it

Use Event Time for correct results:

Event: Click at 9:59 AM
Arrives at: 10:01 AM (system was down)
Count for: 9:00-10:00 window (based on event time)
Not: 10:00-11:00 window (based on processing time)

Code:
window \=
event\['timestamp'\]  \# Event time ✓

\# Not: window \= now()  \# Processing time ✗

Real-world parallel:

* Watermarks \= Postal deadlines (no Christmas cards after Dec 20\)
* Grace period \= Late submissions accepted (with penalty)
* Event time \= When you wrote the letter vs. when it arrived

💡 Final Synthesis Challenge: The Time Machine

Complete this comparison: "Traditional databases are like a photograph of right now. Event streams are like..."

Your answer should include:

* Historical data
* Replay capability
* Multiple consumers
* Immutability

Take a moment to formulate your complete answer...

The Complete Picture: Event streams are like a movie recording of everything that ever happened:

✅ Never delete frames (immutable history)

✅ Can rewind and replay (time travel)

✅ Multiple people watch independently (parallel consumers)

 ✅ See what happened at any timestamp (temporal queries)

 ✅ Complete audit trail (compliance)

✅ Add new viewers anytime (new consumers)

✅ Fast-forward or slow-motion (read at any pace)

✅ Derive current state from full history (event sourcing)

This is why:

* LinkedIn uses Kafka for activity streams (invented it\!)
* Netflix uses Kafka for viewing history
* Uber uses streams for ride events
* Banks use event sourcing for transactions

Event streams transform ephemeral data into permanent, replayable business history\!

🎯 Quick Recap: Test Your Understanding Without looking back, can you explain:

1. How do event streams differ from traditional databases?
2. What is event sourcing and its benefits?
3. How do multiple consumers read from the same stream?
4. Why is event ordering important in streams?

Mental check: If you can answer these clearly, you've mastered event stream fundamentals\!

🚀 Your Next Learning Adventure Now that you understand Event Streams, explore:

Advanced Streaming:

* Stream processing with Kafka Streams
* Apache Flink for complex event processing
* Exactly-once semantics in streaming
* Stream-table duality

Event Sourcing Deep Dive:

* Snapshotting for performance
* Event versioning strategies
* Handling schema evolution
* Projections and read models

Stream Technologies:

* Kafka Connect (integrate external systems)
* Schema Registry (manage schemas)
* ksqlDB (SQL for streams)
* Debezium (CDC connector)

Real-World Patterns:

* Building event-sourced microservices
* Real-time analytics pipelines
* Stream processing at scale
* CQRS in production
