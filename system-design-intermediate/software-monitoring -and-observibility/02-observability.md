---
slug: basics-of-observability
title: Basics of Observability
readTime: 20 min
orderIndex: 2
premium: false

---


# **What is Observability? Seeing Inside Your System**

Logging is like having a diary. Observability is like having X-ray vision, a time machine, and a crystal ball all together.

### **The Difference Between Monitoring and Observability**

Monitoring (The Known Unknowns):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"I know what can go wrong, so I check for it"

Examples:

\- Is the server up? ✓/❌

\- Is CPU usage \<80%? ✓/❌

\- Is disk space \>20%? ✓/❌

\- Are errors \<1%? ✓/❌

Like a car dashboard:

🌡️ Engine temperature: Normal

⛽ Fuel: 3/4 tank

⚠️ Check engine: Off

Problem: Only shows what you expected to check\!

Observability (The Unknown Unknowns):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"I can investigate anything, even problems I didn't anticipate"

Examples:

\- Why is THIS specific user seeing errors?

\- What happened in the 5 minutes before the crash?

\- Which code path led to this slow request?

\- How are these services interacting?

Like a flight recorder:

📊 Every instrument reading

🎙️ Every conversation

📹 Every action

⏱️ Every timestamp

You can investigate ANYTHING after the fact\!

### **The Three Pillars of Observability**

**Pillar 1: Logs (What happened)**

The Story:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 [INFO] User 12345 clicked "Buy Now"

 [INFO] Checking inventory for product 789

 [WARN] Inventory low: only 2 items left

 [INFO] Reserving item for user 12345

 [ERROR] Payment processing failed: Card declined

 [INFO] Rolling back inventory reservation

Logs tell you THE STORY of what happened.

**Connection to Previous Section:** This is why we learned log levels\! Logs are the foundation of observability.

**Pillar 2: Metrics (How much/many)**

The Numbers:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

requests\_per\_second: 1250

average\_response\_time: 145ms

error\_rate: 0.5%

cpu\_usage: 45%

memory\_usage: 3.2GB

active\_users: 10,543

Metrics tell you QUANTITATIVE data.

Time-series data you can graph\!

**Pillar 3: Traces (The journey)**

The Path:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Request ID: abc123

User Request → API Gateway (20ms)

    └→ Auth Service (50ms)

        └→ Database query (30ms)

    └→ Product Service (100ms)

        └→ Database query (80ms)

        └→ Cache check (5ms)

    └→ Payment Service (200ms)

        └→ External API (180ms) ← SLOW\!

    └→ Return Response

Total: 370ms

Traces show you THE PATH through your system.

### **Real-World Observability Example**

Problem Report: "Checkout is slow\!"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: Check Metrics (Is it slow?)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Average checkout time:

\- Yesterday: 500ms ✓

\- Today: 3000ms ❌

Confirmed\! 6x slower\!

Step 2: Check Logs (What's happening?)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Filter: service=checkout, time=last_hour

[WARN] Payment service timeout (2000ms exceeded)

[WARN] Payment service timeout (2000ms exceeded)

[WARN] Payment service timeout (2000ms exceeded)... 127 more warnings |

Pattern found\! Payment service timing out\!

Step 3: Check Traces (Where exactly?)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Sample trace of slow request:

Checkout Request (total: 3200ms)

├─ Validate cart (50ms) ✓

├─ Check inventory (80ms) ✓

├─ Calculate tax (30ms) ✓

└─ Process payment (3000ms) ❌

    ├─ Call Stripe API (2900ms) ❌❌❌

    │   └─ HTTP request timeout

    └─ Retry logic (100ms)

Found it ! Stripe API is slowing down\!

Step 4: Check External Service Status

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Visit: status.stripe.com

Status: "Degraded Performance - Investigating"

Confirmed! It's Stripe, not us!

Step 5: Fix

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Options:
A. Wait for Stripe to fix ❌ (users suffering now)

B. Switch to backup payment processor ✓

C. Show better error message ✓

Implemented:
\- Failover to backup processor

\- Better timeout handling (1000ms instead of 3000ms)

\- User-friendly error: "Payment processing slow, trying backup..."

Result: Checkout time back to 600ms ✓

### **Observability Tools**

**Popular Observability Platforms:**

1\. DataDog

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ All-in-one: Logs \+ Metrics \+ Traces

✓ Great dashboards

✓ Easy setup

❌ Expensive at scale

Use for: Production systems

2\. Prometheus \+ Grafana

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Open source (free\!)

✓ Powerful querying

✓ Great for metrics

❌ Logs/traces separate tools

Use for: Cost-conscious startups

3\. ELK Stack (Elasticsearch, Logstash, Kibana)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Powerful log search

✓ Open source

✓ Great for debugging

❌ Complex to set up

Use for: Log-heavy applications

4\. New Relic

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ APM (Application Performance Monitoring)

✓ Easy to get started

✓ Good for beginners

❌ Can get expensive

Use for: Quick setup needed

---
