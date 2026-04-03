---
slug: latency-throughput
title: Latency vs Throughput
readTime: 15 min
orderIndex: 6
premium: false
---

# Latency vs Throughput

> Two metrics that define system performance — how fast a single request completes and how many requests the system handles per second.




## **⚡ Latency vs Throughput: The Highway Story**

This is one of the most misunderstood concepts in system design. Let's  clear it up with a story.

### **The Road Trip Analogy**

Imagine you need to transport 100 people from City A to City B:

**Scenario 1: The Ferrari Approach (Low Latency)**

Use a Ferrari:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Speed: 200 mph (VERY FAST!)
Seats: 2 people per trip

Trip 1: Drive 2 people (30 minutes)

Trip 2: Drive 2 people (30 minutes)

Trip 3: Drive 2 people (30 minutes)

...
Trip 50: Drive last 2 people (30 minutes)

Total time: 25 hours

First person arrives: After 30 minutes ✓ (Low latency!)
Last person arrives: After 25 hours ❌ (Low throughput!)

**Latency:**

How long ONE person takes to arrive (30 minutes)

✓ **Throughput:** How many people arrive per hour (4 people/hour) ❌



**Scenario 2: The Bus Approach (High Throughput)**


Use a Bus:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Speed: 60 mph (slower)

Seats: 50 people per trip

Trip 1: Drive 50 people (1 hour)

Trip 2: Drive 50 people (1 hour)

Total time: 2 hours

First person arrives: After 1 hour ❌ (Higher latency!)


Last person arrives: After 2 hours ✓ (High throughput!)

**Latency:** How long ONE person takes to arrive (1 hour)

❌ **Throughput:** How many people arrive per hour (50 people/hour) ✓

### **The Key Insight**


![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1766473816/295_prpuky.png)


**Think of it this way:**

* **Latency:** The speed of ONE trip
* **Throughput:** The volume over time

### **Real System Examples**

**Example 1: Web Server Comparison**

Server A: Optimized for Low Latency

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Response time per request: 10ms (FAST!)
Concurrent requests: 100
Requests per second: 100 ÷ 0.01 \= 10,000/sec

Great for: Real-time applications, APIs
Use case: Stock trading, gaming

Server B: Optimized for High Throughput

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Response time per request: 100ms (slower)
Concurrent requests: 10,000
Requests per second: 10,000 ÷ 0.1 \= 100,000/sec

Great for: Batch processing, data pipelines
Use case: Analytics, video encoding

**Connection to TCP:** Remember TCP's flow control with sliding windows? That's managing throughput! TCP adjusts how much data flows based on network capacity. But each individual packet still has latency (round-trip time). TCP optimizes for reliable throughput, not necessarily lowest latency.

**Example 2: Database Query Design**

Let's see  a real scenario:

Task: Fetch 1 million user records

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Approach A: Low Latency (One at a time)

───────────────────────────────────────
```js
for (let i = 0; i < 1000000; i++) {
  const user = await db.query('SELECT * FROM users WHERE id = ?', i)
  process(user)
  // 1ms per user
}
````


Latency per user: 5ms (2ms query \+ 1ms process \+ 2ms network)
Throughput: 1,000,000 users ÷ 5ms \= 200 users/second
Total time: 5,000 seconds (83 minutes!) ❌

Approach B: High Throughput (Batches)

────────────────────────────────────
```js
const BATCH_SIZE = 10000;

for (let offset = 0; offset < 1000000; offset += BATCH_SIZE) {
  const users = await db.query(
    'SELECT * FROM users LIMIT ? OFFSET ?',
    BATCH_SIZE,
    offset
  )
  users.forEach(user => process(user))  // Process batch together
}
````



Latency for first batch: 100ms (slower per user!)
Throughput: 10,000 users/batch × 10 batches/sec \= 100,000 users/sec
Total time: 10 seconds (Much better!) ✓

**The Tradeoff:**

Individual user latency INCREASED (5ms → 100ms)
But overall throughput INCREASED 500x!

Sometimes you sacrifice latency for throughput!

### **The Water Pipe Analogy**

This is a  way to explain it:

Latency \= How fast water flows through the pipe

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1766473816/297_goc0k5.png)

Throughput \= How much total water flows

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1766473816/296_f2j8u7.png)


The Tradeoff:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Thin pipe:  Fast flow (low latency), less volume (low throughput)
Thick pipe: Slow flow (high latency), more volume (high throughput)

You can have:
Option A: Thin pipe with high pressure → Low latency, low throughput
Option B: Thick pipe with normal pressure → Higher latency, high throughput
Option C: Thick pipe with high pressure → Low latency AND high throughput!
          (But this is expensive! 💰)

### **Real-World Optimization Examples**

**Case Study 1: Netflix Video Streaming**

Netflix's Challenge:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Serve 4K video to millions of users simultaneously

Their Solution:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Latency Optimization:
- CDN edge servers near users (low latency to start stream)

- First few seconds: High priority, low latency

- User sees video in \<1 second ✓

Throughput Optimization:

- Pre-encode videos in multiple qualities

- Stream large chunks (not individual frames)

- Massive bandwidth pipes

- Serves 250 million users simultaneously ✓

Result: Fast start (low latency) \+ sustained streaming (high throughput)

**Case Study 2: Google Search**

Google's Approach:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Latency Focus:

- Search results in \<100ms

- Users see results almost instantly

- Distributed data centers worldwide

- Aggressive caching

Throughput:

- Handles 8.5 billion searches/day

- 100,000 queries/second

- Scales horizontally

They optimize for BOTH:
Fast individual results \+ massive query volume

### **The Design Decision Framework**

When designing systems, ask yourself:

Decision Tree:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Question 1: What matters more to your users?

───────────────────────────────────────────


A. "I need results RIGHT NOW"

   → Optimize for LOW LATENCY

   Examples: Trading apps, games, real-time chat

B. "I need to process LOTS of data"

   → Optimize for HIGH THROUGHPUT

   Examples: Analytics, batch jobs, video encoding

Question 2: What's your bottleneck?

───────────────────────────────────

A. Network speed

   → Reduce latency: CDN, edge servers, compression

B. Processing capacity

   → Increase throughput: More workers, parallel processing

C. Database

   → Balance both: Caching (latency), read replicas (throughput)

Question 3: What's your budget?
───────────────────────────────

A. Limited
   → Pick one: Latency OR Throughput
   → Can't have both cheaply

B. Unlimited
   → Get both: Fast AND scalable
   → Expensive but possible

### **Common Mistakes**

**Mistake 1: Confusing the Two**

❌ Wrong thinking:
"My API is slow, so I'll add more servers"

Problem: If latency is the issue, more servers might not help!

Example:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Current: 1 server, 500ms per request
Add: 10 servers, still 500ms per request!

You increased throughput (10x more requests)
But latency didn't improve (still 500ms each)

✓ Correct approach for latency:

- Optimize the slow code

- Add caching

- Use faster database queries

- Reduce network hops

**Mistake 2: Optimizing the Wrong Thing**

Scenario: Batch Email System

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Current: Sends 100,000 emails in 1 hour
Request: "Make it faster!"

Engineer A: "I'll reduce latency!"
Result: Each email sends in 10ms instead of 36ms
Total time: Still \~50 minutes (minor improvement)

Engineer B: "I'll increase throughput!"
Result: Process 10 emails in parallel instead of 1
Total time: 6 minutes! (10x improvement!) ✓

For batch jobs: Throughput > Latency


---

## Key Takeaways

1. **Latency is how long a single request takes, throughput is how many requests per second** — you can optimize for one at the expense of the other
2. **P99 latency matters more than average latency** — 1% of users seeing 10-second load times is a real problem even if the average is 100ms
3. **Network latency dominates in distributed systems** — within a datacenter is ~0.5ms, cross-continent is ~150ms
4. **Caching, connection pooling, and async processing are the top latency reduction techniques** — measure before and after to verify improvement
5. **Throughput scales with parallelism** — more workers, more partitions, more replicas all increase total throughput
