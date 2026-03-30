# Message Queues

Message queues are the backbone of resilient, decoupled distributed systems. They allow services to communicate asynchronously — producers send messages without waiting for consumers to process them, enabling independent scaling, graceful degradation, and fault tolerance that's impossible with synchronous calls.

---

## Synchronous vs Asynchronous Communication

### Synchronous (Tight Coupling)

```
User Request
     │
     ▼
┌──────────┐   HTTP call   ┌──────────┐
│  Order   │──────────────▶│  Email   │  (must wait 500ms)
│  Service │               │  Service │
│          │◀──────────────│          │
└────┬─────┘   "sent!"     └──────────┘
     │         HTTP call
     ▼
┌──────────┐  (must wait)  ┌──────────┐
│          │──────────────▶│Inventory │
│          │◀──────────────│  Service │
└────┬─────┘               └──────────┘
     │
     ▼
  Response to user: 1200ms total

Problem: Email service goes down → Order creation FAILS
```

### Asynchronous (Loose Coupling)

```
User Request
     │
     ▼
┌──────────┐  publish   ┌─────────────┐
│  Order   │───────────▶│   Message   │
│  Service │            │    Queue    │
│          │◀────"ok"───│             │
└──────────┘            └──────┬──────┘
     │                         │  consume (independently)
     ▼                    ┌────┴────────┐
  Response to user: 50ms  │             │
  (just write to queue)   ▼             ▼
                     ┌────────┐   ┌──────────┐
                     │ Email  │   │Inventory │
                     │Service │   │ Service  │
                     └────────┘   └──────────┘

Email service goes down → Queue buffers messages → retries when it comes back
```

---

## Core Concepts

### Producers and Consumers

```
┌──────────┐              ┌──────────┐              ┌──────────┐
│          │   Message    │          │   Message    │          │
│ Producer │─────────────▶│  Queue   │─────────────▶│ Consumer │
│          │              │          │              │          │
└──────────┘              └──────────┘              └──────────┘
```

### Topics and Partitions (Kafka model)

```
Topic: "order-events"

Partition 0: [msg1] [msg4] [msg7] ...
Partition 1: [msg2] [msg5] [msg8] ...  ← Each partition is an ordered log
Partition 2: [msg3] [msg6] [msg9] ...

Consumer Group A (Order Service):
  Consumer A1 reads Partition 0
  Consumer A2 reads Partition 1
  Consumer A3 reads Partition 2

Consumer Group B (Analytics Service):
  Consumer B1 reads ALL partitions (independent of Group A)
```

### Consumer Groups

Multiple consumer groups can independently read the same topic. Each group maintains its own offset (position in the log). Adding consumers to a group increases throughput (one consumer per partition).

---

## Point-to-Point vs Publish-Subscribe

```
Point-to-Point (Queue):              Publish-Subscribe (Topic):

Producer ──▶ [Queue] ──▶ Consumer A  Publisher ──▶ [Topic] ──▶ Subscriber A
                                                           ──▶ Subscriber B
One consumer processes each message  ALL subscribers get EVERY message
```

| Model | Best For | Examples |
|-------|----------|---------|
| Point-to-Point | Task distribution (one worker handles each task) | Job queues, email sending |
| Pub-Sub | Event broadcasting (multiple services react to each event) | Order placed → notify email, inventory, analytics |

---

## Apache Kafka Deep Dive

Kafka is the most widely used event streaming platform. It's not just a queue — it's a distributed, persistent, replayable log.

### Kafka Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Cluster                            │
│                                                             │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐            │
│   │ Broker 1│      │ Broker 2│      │ Broker 3│            │
│   │         │      │         │      │         │            │
│   │ Part 0  │      │ Part 0  │      │ Part 0  │            │
│   │(leader) │      │(replica)│      │(replica)│            │
│   │         │      │         │      │         │            │
│   │ Part 1  │      │ Part 1  │      │ Part 1  │            │
│   │(replica)│      │(leader) │      │(replica)│            │
│   └─────────┘      └─────────┘      └─────────┘            │
│                                                             │
│   ZooKeeper / KRaft: cluster metadata, leader election      │
└─────────────────────────────────────────────────────────────┘
```

### Key Kafka Properties

**Persistent storage:** Messages are stored on disk, not just in memory. Retention is configurable (days, weeks, forever).

**Replay:** Consumers can seek to any offset and re-read historical messages. Invaluable for debugging, backfilling, and building new services on historical data.

**High throughput:** Kafka handles millions of messages/second through:
- Sequential disk writes (fast)
- Zero-copy I/O (kernel bypasses)
- Batching (multiple messages per network request)
- Compression

**Partitioned parallelism:** N partitions = N consumers in parallel = N× throughput.

### When to Use Kafka

- Event streaming and event sourcing
- Activity tracking (page views, clicks, user behavior)
- Log aggregation
- Stream processing (with Kafka Streams or Flink)
- When you need message replay
- When multiple independent consumers need the same events

---

## RabbitMQ

RabbitMQ is a traditional message broker — simpler than Kafka but with different strengths.

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Producer │───▶│ Exchange │───▶│  Queue   │───▶│ Consumer │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                    │
                Exchange types:
                - Direct: route by routing key
                - Fanout: broadcast to all queues
                - Topic: pattern-matching routing key
                - Headers: route by message headers
```

**When to use RabbitMQ:**
- Complex routing logic (route messages based on content)
- When you need message acknowledgment and delivery guarantees
- When consumers delete messages after processing (no replay needed)
- Lower volume, higher per-message logic

### Kafka vs RabbitMQ

| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| Model | Log (pull-based) | Queue (push-based) |
| Message retention | Configurable (days/forever) | Deleted after consumption |
| Replay | Yes | No |
| Throughput | Millions/sec | Tens of thousands/sec |
| Routing | By partition | Complex exchange routing |
| Consumer model | Pull (consumer tracks offset) | Push (broker tracks) |
| Use case | Event streaming, big data | Task queues, complex routing |

---

## Delivery Guarantees

### At-Most-Once
Message is delivered at most once. May be lost. Fastest.

```
Producer sends → Consumer processes → ACK sent
If consumer crashes before ACK: message is lost
```
**Use for:** Metrics, logs where losing some data is acceptable.

### At-Least-Once
Message is delivered at least once. May be duplicated. Most common.

```
Producer sends → Consumer processes → ACK sent
If ACK is lost: producer retries → consumer may process twice
```
**Use for:** Most use cases. Consumers must be idempotent.

### Exactly-Once
Message is processed exactly once. Hardest, most expensive.

```
Requires: idempotent producers + transactional consumers
Kafka supports this with transactions (producer_id + sequence numbers)
```
**Use for:** Payment processing, financial ledgers.

---

## Dead Letter Queues (DLQ)

When a message can't be processed (repeated failures), send it to a Dead Letter Queue for investigation.

```
Normal Queue:
   [msg1] [msg2] [msg3(failing)] [msg4]
                      │ (failed 3 times)
                      ▼
              ┌──────────────┐
              │  Dead Letter │  ← Engineers investigate
              │    Queue     │    and replay or discard
              └──────────────┘
```

**Without DLQ:** One bad message blocks the entire queue ("poison pill" problem).

---

## Message Ordering

Ordering guarantees vary by system:

| Scope | Kafka | SQS Standard | SQS FIFO |
|-------|-------|-------------|----------|
| Global ordering | No | No | No |
| Per-partition ordering | Yes | N/A | N/A |
| Per-queue ordering | N/A | No | Yes |

**Kafka's approach:** Messages with the same partition key go to the same partition → ordered within that partition.
```
Messages for user "alice" → always Partition 2 → processed in order
Messages for user "bob"   → always Partition 0 → processed in order
```

---

## Backpressure and Flow Control

When consumers are slower than producers, queues fill up.

```
Producer: 10,000 msg/sec ──▶ [Queue: filling up...] ──▶ Consumer: 5,000 msg/sec

Solutions:
1. Scale consumers (add more instances)
2. Backpressure: signal producer to slow down
3. Drop messages (only if data loss is acceptable)
4. Alert and page on-call when queue depth grows
```

**Monitor queue depth!** A growing queue is an early warning system.

---

## Real-World Usage

### Uber (Kafka)
- 1 trillion messages per day
- Real-time trip updates, driver locations, payments
- Event sourcing for the entire driver/rider matching system

### LinkedIn (Kafka — the birthplace)
- Created Kafka for their activity stream: 4.5 billion events/day
- Feed updates, notifications, analytics

### Netflix
- Uses SQS for inter-service communication
- Studio content processing pipeline uses message queues for video transcoding jobs

---

## Key Takeaways

1. **Message queues decouple producers and consumers** — services can scale and fail independently
2. **Kafka is a distributed log** — persistent, replayable, high-throughput event streaming
3. **RabbitMQ excels at complex routing** — use when you need fanout, topic, or header-based routing
4. **At-least-once is the safe default** — make consumers idempotent to handle duplicate delivery
5. **Dead letter queues catch poison pills** — without them, one bad message blocks the queue forever
6. **Per-partition ordering in Kafka** — use a consistent partition key for ordered processing
7. **Monitor queue depth** — it's your canary in the coal mine for consumer health

---

*Previous: [Consistent Hashing ←](02-consistent-hashing.md) · Next: [Database Replication →](04-database-replication.md)*
