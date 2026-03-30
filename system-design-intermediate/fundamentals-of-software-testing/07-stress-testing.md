---
slug: stress-testing
title: Stress Testing
readTime: 10 min
orderIndex: 7
premium: false

---



## **💪 Stress Testing: "What Breaks the System?"**

### **What is Stress Testing?**

Stress testing pushes the system **beyond normal limits** to find breaking points.

Normal load: 1,000 users
Stress test: 10,000 → 50,000 → 100,000 users

Goal: Find where and how system fails

### **Real-World Analogy:**

**Like testing how much weight a bridge can hold:**

Design capacity: 100 cars
Stress test:

  ✓ 150 cars → Bridge holding ✅

  ✓ 200 cars → Bridge holding ✅

  ✓ 250 cars → Bridge sagging ⚠️

  ✗ 300 cars → Bridge collapses\! 💥

Now you know: Safety limit is 250 cars
Can post warning signs at 200 cars

### **Types of Stress Testing:**

#### **1\. Load Testing (Sustained)**

Gradually increase load:

Hour 1:  1,000 users  →  Response time: 100ms ✅

Hour 2:  5,000 users  →  Response time: 150ms ✅

Hour 3: 10,000 users  →  Response time: 200ms ✅

Hour 4: 20,000 users  →  Response time: 500ms ⚠️

Hour 5: 30,000 users  →  Response time: 2000ms ❌
                          Error rate: 5% ❌

Breaking point: \~25,000 concurrent users

#### **2\. Spike Testing (Sudden)**

Sudden traffic spike:

Normal:   1,000 users
  ↓
Spike:   50,000 users (in 10 seconds\!)
  ↓
Observe: How does system handle sudden surge?

Scenarios:

✅ Good: System auto-scales, handles load

⚠️ OK: System slows down but recovers

❌ Bad: System crashes, needs manual restart

#### **3\. Endurance Testing (Soak)**

Sustained load over long period:

Load: 10,000 users
Duration: 48 hours continuous

Watch for:

❌ Memory leaks (RAM usage grows over time)

❌ Connection pool exhaustion

❌ Disk space issues (logs filling up)

❌ Database connection leaks

✅ System remains stable

Example finding:
Hour 1:  Memory usage: 2GB

Hour 12: Memory usage: 4GB

Hour 24: Memory usage: 6GB ⚠️ Memory leak detected\!

### **Stress Test Example: News Website**

Scenario: Major news event (election, sports final)
Normal traffic: 10,000 concurrent users
Expected surge: 200,000 concurrent users

Stress Test Setup:

Tool: Apache JMeter or Gatling or k6

Test Script:
// Ramp up to peak load

\- Start: 10,000 virtual users

\- Every minute: Add 20,000 users

\- Peak: 200,000 users

\- Duration: 30 minutes at peak

\- Ramp down: Gradual decrease

User Behavior:

1\. Load homepage (80% of requests)

2\. Read article (15% of requests)

3\. Post comment (5% of requests)

Metrics to Monitor:

Performance:
\- Response time (p50, p95, p99)
\- Throughput (requests/second)
\- Error rate (%)

Resources:
\- CPU usage (%)
\- Memory usage (GB)
\- Disk I/O
\- Network bandwidth
\- Database connections

Results:

Phase 1: 0-50,000 users

  Response time: 100-200ms ✅

  CPU: 40% ✅

  Memory: Stable ✅

  Errors: 0% ✅

Phase 2: 50,000-100,000 users

  Response time: 200-400ms ⚠️

  CPU: 70% ⚠️

  Memory: Stable ✅

  Errors: 0.1% ⚠️

Phase 3: 100,000-150,000 users
  Response time: 400-1000ms ❌

  CPU: 95% ❌

  Memory: Growing slowly ⚠️

  Errors: 2% ❌

  Database: Connection pool exhausted\! 💥

Phase 4: 150,000+ users
  Response time: \>5000ms or timeout ❌

  CPU: 100% (maxed) ❌

  Memory: OOM errors ❌

  Errors: 25% ❌

  Database: Deadlocks occurring ❌

Finding: System breaks at \~120,000 concurrent users

Bottlenecks Identified:

1\. Database connection pool too small (max 100 connections)

2\. Application servers CPU-bound

3\. No caching layer for homepage

4\. No rate limiting

Recommendations:

✅ Increase database connection pool to 500

✅ Add Redis caching for homepage

✅ Scale to 5 application servers

✅ Implement CDN for static assets

✅ Add rate limiting (100 req/min per IP)

After fixes, re-test:
New capacity: 300,000 concurrent users\! ✅

### **Stress Testing Tools:**

1\. Apache JMeter
   \- GUI-based test design
   \- Extensive protocols support
   \- Distributed testing

2\. Gatling
   \- Code-based (Scala)
   \- Excellent reporting
   \- Real-time monitoring

3\. k6 (by Grafana)
   \- JavaScript-based scripts
   \- Developer-friendly
   \- CI/CD integration

4\. Locust (Python)
   \- Python scripts
   \- Distributed load generation
   \- Web UI

Example k6 script:

```js
 import http from 'k6/http';
 import { check, sleep } from 'k6';
 export let options = {  stages: [    { duration: '5m', target: 100 },
 // Ramp up to 100 users
 { duration: '10m', target: 100 },
 // Stay at 100 users
 { duration: '5m', target: 500 },
 // Ramp up to 500 users
 { duration: '10m', target: 500 },
 // Stay at 500 users
 { duration: '5m', target: 0 },
 // Ramp down  ],};
 export default function() {
  let response = http.get('https://example.com/');
  check(response, {    'status is 200': (r) => r.status === 200,
  'response time < 500ms': (r) => r.timings.duration < 500,  });  sleep(1);}
  ```

**Key characteristics:**

* 📈 **Beyond normal:** Push past expected limits
* 🔍 **Find breaking points:** Where does it fail?
* 📊 **Performance metrics:** Response time, throughput, errors
* 🎯 **Capacity planning:** Know your limits
