---
slug: Event Streams
title: Fundamentals of Event Streams
readTime: 10 min
orderIndex: 1
premium: false
---
## **Event Streams:** ##

 The Never-Ending Data River (Time Travel for Your Application)

🎯 Challenge 1: The Bank Statement Problem Imagine this scenario: You need to know your current bank balance.

Traditional Database Approach:

Database Table:

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1765696687/285_alobpa.png)

Questions you CAN'T answer:

❌ How did we get to $1,542?

❌ What transactions happened last month?

❌ Can we audit the account history?

❌ What if we need to recalculate?

Event Stream Approach:

Event Log (Immutable):
\[Deposited $1,000\] → \[Withdrew $200\] → \[Deposited $500\] → \[Withdrew $50\] → \[Paid $8 fee\] ...

Current balance \= $1,000 \- $200 \+ $500 \- $50 \- $8 \= $1,542 ✓

Questions you CAN answer:

✅ Replay all transactions to verify


✅ Query any time period

✅ Audit trail for compliance

✅ Rebuild balance at any point in time

✅ Multiple views (by month, by category, etc.)

Pause and think: What if instead of storing current state, you stored every event that ever happened?

The Answer: Event Streams are immutable, append-only logs of events\! They're like a video recording vs. a snapshot:

✅ Never delete events (permanent record)

✅ Events  are in chronological order (time-ordered)

✅ Can replay from any point (time travel\!)

✅ Multiple consumers read independently (parallel processing)

✅ Source of truth for what happened (audit trail)


Key Insight: Event streams transform "What is the current state?" into "What happened, in order?"

🎬 Interactive Exercise: Snapshot vs Event Stream

Database Snapshot (Current State):

Users Table:
![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1765696687/282_tjqmqb.png)

What you know:
\- Alice's current status is Active
\- Her current email

What you DON'T know:

❌ When did she join?

❌ Did she ever change her email?

❌ Was she ever inactive?

❌ What was her journey?

Event Stream (Complete History):

Event Log:
```bash

10:00 UserCreated:

{id: 1, name: "Alice", email: "alice@old.com"}

10:15 EmailUpdated:

{id: 1, email: "alice@new.com"}

10:30 AccountSuspended: {id: 1, reason:""payment_failed"}

11:00 PaymentReceived: {id: 1, amount: 50}

11:05 AccountReactivated: {id: 1}

12:00 EmailUpdated: {id: 1, email: "alice@example.com"}
````

Current State (computed from events):

\- Name: Alice

\- Email: alice@example.com (changed 2 times\!)

\- Status: Active (was suspended for 35 minutes\!)

What you KNOW:

✅ Complete timeline

✅ All state changes

✅ Can answer "what happened at 10:45?"

✅ Can rebuild state at any timestamp

✅ Perfect audit trail

Real-world parallel: Database is like a photograph (one moment). Event stream is like a video recording (the whole story).

🏗️ Core Event Stream Concepts

1. Events (The Facts):

An event is an immutable fact about what happened:

Event structure:

```json
{  "eventId": "evt-12345",  "eventType":
  "OrderPlaced",  "timestamp":

  "2024-01-15T10:30:00.000Z",  "streamId": "order-789",

  "data": {    "orderId": "789",    "customerId": "456",    "items": [...],    "total": 99.99  },

  "metadata": {    "userId": "user-123",    "source": "web-app",   "version": 1  }}


```

Characteristics:

├── Past tense ("OrderPlaced" not "PlaceOrder")

├── Immutable (can never be changed)

├── Timestamped (when it happened)

└── Self-contained (all necessary context)

2. Stream (The River):

A stream is an ordered sequence of events:

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1765696687/283_qrek44.png)

Time flows →

Features:

├── Append-only (can only add to end)

├── Immutable (can't modify past events)

├── Ordered (chronological)

└── Infinite (never "ends")

3. Offset/Position (The Bookmark):

Each event has a position in the stream:

Stream:
![img4](https://res.cloudinary.com/dretwg3dy/image/upload/v1765696686/280_emi8wm.png)

Each consumer tracks their own position\!

4. Consumers (The Readers):

Multiple independent readers:

![img5](https://res.cloudinary.com/dretwg3dy/image/upload/v1765696688/286_fdlpgk.png)

Each consumer:

├── Reads at their own pace

├── Can replay from beginning

├── Doesn't affect others

└── Maintains their own offset

Complete Flow:

Producer writes events:
Order Service → Stream "orders"

  \[OrderPlaced\]

  \[PaymentReceived\]

  \[OrderShipped\]

  \[OrderDelivered\]

Events stored permanently (configurable retention)

Consumers read independently:
 Email Service

 │ reads from position 0


 Analytics

 │ reads from position 2


 Data Warehouse

 │ reads all (batch processing)


 Audit System

 │ reads from position 1 (compliance)


![img6](https://res.cloudinary.com/dretwg3dy/image/upload/v1765696687/281_ixrelb.png)

New Consumer can join anytime:

Say Recommendation Engine Joins:

  ← Reads from beginning (builds full history)



  ← Or starts from now (only new events)

Real-world parallel:


* Event \= Transaction on bank statement


* Stream \= Bank statement (all transactions)

* Offset \= Line number you're reading

* Consumers \= Different people reading statement

🎮 Decision Game: Event Stream vs Traditional DB?

Match the use case to the best approach:

Scenarios:

A. Store user's current profile

B. Track all user actions for analytics

C. Shopping cart contents

D. Financial transaction ledger

E. Show real-time stock price

F. Audit trail for compliance

G. Simple CRUD application


H. Event sourcing system

Options:

1. Traditional Database (current state)
2. Event Stream (complete history)

Think about: Need history or just current state?

Answers:

A. User profile → Database (1)
   Only need current state, not history

B. User actions → Event Stream (2)
   Analytics needs complete history

C. Shopping cart → Database (1)
   Current items matter, not history

D. Financial ledger → Event Stream (2)
   Audit trail critical, can't lose transactions

E. Stock price → Database (1) \+ Stream (2)
   Current price in DB, history in stream

F. Audit trail → Event Stream (2)
   By definition, need complete history

G. CRUD app → Database (1)
   Simple create/read/update/delete

H. Event sourcing → Event Stream (2)
   Events ARE the source of truth

🚨 Common Misconception: "Event Streams Are Just Logs... Right?"

You might think: "Event streams are just application logs."

The Reality: Event streams are a first-class data source\!

Application Logs (Different Purpose):
```bash

2024-01-15 10:30:00 INFO User 123 logged in

2024-01-15 10:30:05 DEBUG Query took 45ms

2024-01-15 10:30:10 ERROR Connection timeout
```

Purpose: Debugging, troubleshooting
Format: Human-readable text
Structure: Unstructured or semi-structured
Retention: Days to weeks
Consumption: Humans, log analysis tools

Event Streams (Business Events):
```json

 {  "eventType": "UserLoggedIn",  "userId": "123"

   ,  "timestamp": "2024-01-15T10:30:00Z",

   "sessionId": "abc",  "device": "mobile"

 }
 ````



Purpose: Business logic, data processing
Format: Structured (JSON, Protobuf, Avro)
Structure: Well-defined schema
Retention: Months to forever
Consumption: Services, analytics, ML models

![img7](https://res.cloudinary.com/dretwg3dy/image/upload/v1765696687/284_rqqerk.png)

Real-world parallel:

* App logs \= Security camera footage (diagnostic)
* Event streams \= Business transaction receipts (business data)
