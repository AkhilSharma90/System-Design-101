---
slug: logging-basics
title: Basics of Logging
readTime: 20 min
orderIndex: 1
premium: false

---




# **Logging Basics: Your System's Diary**

Imagine if you had amnesia—you wake up every day with no memory of yesterday. Terrifying, right? Without logging, your servers have amnesia. Let’s give your system a perfect memory.

### **The Crime Scene Investigation Analogy**

A crime happens in a building:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Without logs (No cameras, no witnesses):
Detective: "Something happened here..."
Evidence: None
Result: Unsolvable ❌

With logs (Cameras everywhere):
Detective: "Let me review the footage..."
Evidence:
\- 2:34 PM: Person A entered

\- 2:37 PM: Person B entered

\- 2:40 PM: Loud noise

\- 2:41 PM: Person B exited quickly

Result: Solvable\! ✓

Same with your app:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Without logs:

"The app crashed at 3 AM"

No idea why ❌

With logs:

"Let me check what happened..."

03:00:01 \- User 12345 placed order

03:00:02 \- Payment processing started

03:00:03 \- ERROR: Database connection timeout

03:00:04 \- System crashed

Result: Found the problem\! ✓

### **The Five Log Levels Explained**

Think of log levels like severity levels in a hospital:

**1\. DEBUG \- The Detailed Diary**

Hospital Analogy: Recording vitals of the patients every 5 minutes

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


"Patient's blood pressure: 120/80"

"Patient's heart rate: 72 bpm"

"Patient turned over at 2:15 PM"

Useful for: Detailed diagnosis
Too much for: Daily review

Code Example:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```js

logger.debug("User authentication started");
logger.debug("Checking user_id: 12345");

logger.debug("Database query: SELECT * FROM users WHERE id = 12345");

logger.debug("Query returned 1 result");

logger.debug("Password hash comparison starting");

logger.debug("Password matches\!");

logger.debug("Generating JWT token");

logger.debug("Token: eyJhbGc...");

logger.debug("Authentication completed successfully");

```
```bash
Output when enabled:
[DEBUG] 2025-10-19 14:30:01.234

- User authentication started

[DEBUG] 2025-10-19 14:30:01.235

Checking user_id: 12345

[DEBUG] 2025-10-19 14:30:01.236 - Database query: SELECT * FROM users...

[DEBUG] 2025-10-19 14:30:01.240

- Query returned 1 result

[DEBUG] 2025-10-19 14:30:01.241

Password hash comparison starting

[DEBUG] 2025-10-19 14:30:01.245

- Password matches

[DEBUG] 2025-10-19 14:30:01.246 - Generating JWT token

[DEBUG] 2025-10-19 14:30:01.250 - Token: eyJhbGc...

[DEBUG] 2025-10-19 14:30:01.251 -
```

Authentication completed successfully


**When to use DEBUG:**

* Development environment

* Troubleshooting specific issues

* Understanding code flow

* **Never in production** (too much data involved \!)

**2\. INFO \- The Normal Operations Log**

Hospital Analogy: Recording routine events about the patients

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


"Patient admitted at 9:00 AM"

"Medication administered at 10:00 AM"

"Patient discharged at 5:00 PM"

Useful for: Understanding normal flow
Not for: Every tiny detail

Code Example:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```js
 logger.info("User 12345 logged in successfully");
logger.info("User 12345 viewed product page: iPhone 15");
logger.info("User 12345 added item to cart");
logger.info("User 12345 initiated checkout");
logger.info("Payment processed successfully for order #789");
logger.info("User 12345 logged out");
```
```bash

Output:[INFO] 2025-10-19 14:30:01 - User 12345 logged in successfully [INFO]

2025-10-19 14:32:15 - User 12345 viewed product page: iPhone 15

[INFO] 2025-10-19 14:33:20

- User 12345 added item to cart

[INFO] 2025-10-19 14:35:10

- User 12345 initiated checkout

[INFO] 2025-10-19 14:35:45

- Payment processed for order #789

[INFO] 2025-10-19 14:36:00

- User 12345 logged out
```



**When to use INFO:**

* Use it in production environment ✓
* Use to track important business events
* Keep audit trail
* Normal operations

**3\. WARN \- The Concerning But Not Critical**

Hospital Analogy: Potential problems

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


"Patient's temperature slightly elevated (99.5°F)"

"Patient hasn't eaten full meal"

"Unusual blood pressure reading"

Useful for: Catching issues early
Status: Watch closely, but not emergency

Code Example:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```js

 logger.warn("Database connection pool 80% full");

 logger.warn("API rate limit approaching (450/500 calls)");

 logger.warn("Disk space low: 15% remaining");

 logger.warn("Slow query detected: took 2.5s (threshold: 1s)");

 logger.warn("Failed login attempt \#3 for user: john@example.com");

 logger.warn("Cache miss rate high: 45% (normal: \<20%)");
 ```


 Output:
 ```bash

 [WARN] 2025-10-19 14:30:01 - Database connection pool 80% full

 [WARN] 2025-10-19 14:31:22 - API rate limit approaching (450/500)

 [WARN] 2025-10-19 14:32:10 - Disk space low: 15% remaining

 [WARN] 2025-10-19 14:33:45

 - Slow query detected: 2.5s

 [WARN] 2025-10-19 14:35:20

 - Failed login attempt

 \#3: john@example.com

 [WARN] 2025-10-19 14:36:15

 - Cache miss rate high: 45%
 ```

**When to use WARN:**

* Something unusual happened
* Not breaking, but concerning
* Might need attention soon
* Preview of future problems

**Real-World WARN Example:**

Scenario: E-commerce site

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| 14:00 \- [INFO\] Server started

14:15 \- \[INFO\] 100 requests/minute (normal)

14:30 \- \[WARN\] 300 requests/minute (unusual spike)

14:45 \- \[WARN\] 500 requests/minute (concerning)

15:00 \- \[ERROR\] 1000 requests/minute \- Server overloaded\! |

The WARNs gave you 30 minutes to act\!
Could have scaled up servers before ERROR\!

**4\. ERROR \- The Problem Log**

Hospital Analogy: Medical emergencies

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"Patient's heart rate dropped to 40 bpm"
"Medication allergy detected"
"Patient fell out of bed"

Status: Immediate attention required\!
Action: Doctors respond now

Code Example:


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```bash

14:00 - [INFO] Server started

14:15 - [INFO] 100 requests/minute (normal)

14:30 - [WARN] 300 requests/minute (unusual spike)

14:45 - [WARN] 500 requests/minute (concerning)

15:00 - [ERROR] 1000 requests/minute - Server overloaded\!t

at /app/database.js:45:12

at /app/server.js:120:5

[ERROR] 2025-10-19 14:32:15 - Payment processing failed

Error: Card declined - Insufficient funds

at PaymentProcessor.charge

at /app/payment.js:78:10

at /app/checkout.js:234:8

[ERROR] 2025-10-19 14:35:20 - External API returned 500


Error: Internal Server Error

at HttpClient.request

at /app/api-client.js:56:7
```

**When to use ERROR:**

* Operation failed
* User affected
* Needs immediate investigation
* Feature broken

**The Key Distinction:**

WARN vs ERROR:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| WARN:
"Payment took 5 seconds (slow but succeeded)" ⚠️

ERROR: "Payment failed \- user got error message" ❌

WARN: "Database query slow (800ms)" ⚠️

ERROR: "Database connection lost" ❌

WARN: "Disk 85% full" ⚠️

ERROR: "Disk 100% full \- can't write files" ❌

WARN: Things still work, but concerning

ERROR: Things broke, user impacted


**5\. FATAL/CRITICAL \- The System Killer**

Hospital Analogy: Code Blue

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"Patient cardiac arrest\!"

"Critical system failure"

Status: Life-threatening

Action: All hands on deck, emergency protocol

Code Example:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```js

logger.fatal("Unable to bind to port 3000 - port already in use");

logger.fatal("Critical configuration missing - cannot start server");

logger.fatal("Out of memory - application crashing");

logger.fatal("Database cluster completely unavailable");

logger.fatal("Unrecoverable error in main thread", error);
```

Output:
```bash

[FATAL] 2025-10-19 14:30:01 - Unable to bind to port 3000

Application cannot start. Exiting with code 1

[FATAL] 2025-10-19 14:32:15 - Out of memory

Error: JavaScript heap out of memory

Process will terminate.

[FATAL] 2025-10-19 14:35:20 - Database cluster unavailable

Cannot operate without database. Shutting down.

```

**When to use FATAL:**

* Application cannot continue
* About to crash/exit
* System-level failure
* Complete service outage

### **Structured Logging: The Modern Way**

Old Way (String Logging):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

logger.info("User john@example.com placed order 789 for $99.99");

Problem: Hard to search, parse, analyze

New Way (Structured Logging):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```js
logger.Info({
event: "order_placed",
user_id: 12345,
user_email: "john@example.com",
order_id: 789,
amount: 99.99,
currency: "USD",
items_count: 3,
timestamp: "2025-10-19T14:30:01.234Z"});
````

Output (JSON):
```json
{  "level": "info",
  "event": "order_placed",
  "user_id": 12345,
  "user_email": "john@example.com",
  "order_id": 789,
  "amount": 99.99,
  "currency": "USD",
  "items_count": 3,
  "timestamp": "2025-10-19T14:30:01.234Z",
  "host": "server-3",
  "service": "checkout-service"}
}
```


Benefits:

✓ Easy to search: "Show all orders \> $100"

✓ Easy to aggregate: "Total revenue today?"

✓ Machine-readable: Tools can parse automatically

✓ Consistent format

**Real-World Debugging Example:**

Problem: "Some users can't checkout\!"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

With string logs (hard):

❌ Grep through millions of lines

❌ Parse text manually

❌ Hard to find patterns

With structured logs (easy):

✓ Query: event = "checkout_failed"

✓ Filter: last 24 hours

✓ Group by: error_type

Results:

```js
 {  "payment_declined": 45occurrences,
   "timeout": 12 occurrences,
   "invalid_address": 8 occurrences,
   "inventory_unavailable": 3occurrences
 }
 ````



Found it ! "payment_declined" is the main issue!

Drill deeper:
payment_declined errors →
40 from same credit card processor
→ Their API is down!
→ Switch to backup processor ✓

### **Log Best Practices**

**What to Log:**

✓ DO Log:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ User actions: login, purchase, upload

✓ System events: startup, shutdown, deployment

✓ External API calls: request, response, latency

✓ Errors with context: what failed, why, when

✓ Performance metrics: slow queries, high memory

✓ Security events: failed logins, suspicious activity

❌ DON'T Log:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

❌ Passwords (NEVER\!)

❌ Credit card numbers

❌ Social security numbers

❌ API keys, secrets, tokens

❌ Personal health information

❌ Anything sensitive/private

Example of what NOT to do:

logger.info(\`User logged in with  password: ${password}\`); ❌❌❌


THIS IS A SECURITY BREACH\!

**Log Retention:**

How Long to Keep Logs:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

DEBUG logs: 1-7 days (huge volume)

INFO logs: 30-90 days (moderate volume)

WARN logs: 90-365 days (important patterns)

ERROR logs: 1-2 years (compliance, analysis)

AUDIT logs: 7+ years (legal requirements)


Storage cost example:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1 million requests/day

Each request: 2 log entries

Each entry: 1 KB

Daily logs: 2 GB

Monthly logs: 60 GB

Yearly logs: 730 GB

At $0.023/GB/month (AWS S3):

Monthly cost: $16.80

Yearly cost: $201.60

Worth it for debugging? YES\! ✓

---
