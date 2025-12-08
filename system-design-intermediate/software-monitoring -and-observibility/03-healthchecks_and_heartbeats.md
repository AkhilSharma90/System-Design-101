---
slug: healthchecks-and-heartbeats
title: Heartbeats and Healthchecks
readTime: 20 min
orderIndex: 3
premium: false


---


# **💓 Health Checks and Heartbeats: Is Your System Alive?**

Imagine if your heart stopped beating and nobody noticed for an hour. Terrifying\! That's why we have health checks and heartbeats.

### **Health Checks: The Wellness Checkup**

**What is a Health Check?**

The Doctor Visit Analogy:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Doctor asks:

\- Are you breathing? (Process running?)

\- Is your heart beating? (Database connected?)

\- Can you walk? (APIs responding?)

\- Do you feel okay? (Error rate normal?)

If all answers are "yes" → Healthy ✓

If any answer is "no" → Unhealthy ❌

Your Web Server:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Health check endpoint is usually : GET /health

It returns the status of the server in json format
```json

 {  "status": "healthy",
   "checks":
   {    "database": "connected",
     "cache": "connected",
     "disk_space": "45% free",
     "memory": "60% used"  },
   "timestamp": "2025-10-19T14:30:01Z"}
```

**Types of Health Checks:**

**1\. Shallow Health Check (Quick Check)**

Purpose: "Is the server responding?"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Implementation:
GET /health

Response time: 1-5ms

Checks: Just server process

Example:
```js

 app.get('/health', (req, res) => {res.status(200).json({ status: 'ok' });});
 ```
Use case:

\- Load balancer health check

\- Called every 5 seconds

\- Must be FAST

Pro: Fast, low overhead

Con: Doesn't check dependencies

**2\. Deep Health Check (Thorough Check)**

Purpose: "Is everything working?"

━━━━━━━━━━━━━━━━━━━━

 Implementation:
 GET /health/deep
 Response time: 100-500ms
 Checks: Everything
 ```js
 Example:app.get('/health/deep', async (req, res) => {

 const checks = {
  database: await checkDatabase(),
  redis: await checkRedis(),
  external_api: await checkExternalAPI(),
  disk_space: await checkDiskSpace(),
  memory: await checkMemory()
  };
  const allHealthy = Object.values(checks).every(check => check.status === 'healthy');
  res.status(allHealthy  200 : 503)
    .json({    status: allHealthy  'healthy' : 'unhealthy',
      checks: checks,   timestamp: new Date().toISOString()
    })})
 ```

Use case:
\- Deep health check is used for manual debugging

\- It can be used to provide detailed monitoring

\- Called less frequently

Pro: It is a more comprehensive form of health check
Con: Slower, more resource intensive

**Real Health Check Implementation:**

```js
 // Production-ready health check

app.get('/health', async (req, res) => {

  try {
    // Quick checks
    const checks = {
      server: 'up',
      timestamp: new Date().toISOString()
      };
    // Check database (with timeout!)
    try {
      await Promise.race([db.query('SELECT 1'),        timeout(1000) // 1 second max]);
      checks.database = 'connected';
      }
      catch (err) {
        checks.database = 'disconnected';
        return res.status(503).json({        status: 'unhealthy', checks      });
        }
        // Check Redis cache   try {
        await Promise.race([        redis.ping(),        timeout(500)]);
        checks.cache = 'connected';
        } catch (err) {
          checks.cache = 'disconnected';
          // Cache failure is not critical, just warn      logger.warn('Cache health check failed', err);
        }
        // All critical checks passed
        res.status(200).json({      status: 'healthy',      checks
        });}
        catch (err) {
        logger.error('Health check failed', err);    res.status(503).json({
        status: 'unhealthy',      error: err.message    });  }});
        ```

**How Load Balancers Use Health Checks:**

Load Balancer with 3 Servers:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Every 5 seconds, check each server:
```bash

 Time: 14:00:00
Server 1: GET health → 200 OK ✓
Server 2: GET health → 200 OK ✓
Server 3: GET /health → 200 OK ✓
Traffic: Distributed 33% / 33% / 33%
Time: 14:00:05
Server 1: GET /health → 200 OK ✓
Server 2: GET /health → 503 Unhealthy ❌
Server 3: GET /health → 200 OK ✓
Traffic: Distributed 50% / 0% / 50%
(Server 2 removed from rotation\!)
Time: 14:00:10
Server 1: GET /health → 200 OK ✓
Server 2: GET /health → 200 OK ✓ (Recovered!)
Server 3: GET /health → 200 OK ✓
Traffic: Back to 33% / 33% / 33%
```

**Connection to Previous Topics:** This eliminates SPOF\! If one server fails, load balancer detects it via health checks and routes traffic to healthy servers.

### **Heartbeats: The Continuous Pulse**

**What is a Heartbeat?**

The Difference:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Health Check (Pull):

    Monitor ────────\> Server

    "Are you alive?"

    Server responds (or doesn't)

Heartbeat (Push):

    Server ────────\> Monitor

    "I'm alive\!"

    Server sends regularly


**Real-World Heartbeat Example:**

Background Worker Process:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```js
 // Worker sends heartbeat every 30 seconds
setInterval(async () => {
  try {
    await redis.set(`heartbeat:worker:${workerId}`, Date.now(),
    'EX', 60  // Expires in 60 seconds);
    logger.debug('Heartbeat sent');
    } catch (err) {
      logger.error('Failed to send heartbeat', err);}}, 30000);
    // Monitor checks heartbeats setInterval(async () => {  const workers = await getWorkerList();
    for (const worker of workers) {
      const heartbeat = await redis.get(`heartbeat:worker:${worker.id}`);
      if (!heartbeat) {
        // No heartbeat received!
        logger.error(`Worker ${worker.id} is dead - no heartbeat`);
        alertOps(`Worker ${worker.id} is down!`);
        restartWorker(worker.id);
        } else {
          const age = Date.now() - parseInt(heartbeat);

          if (age > 45000) {

           // 45 seconds old

            logger.warn(`Worker ${worker.id} heartbeat stale: ${age}ms\`);     }    }  }}, 10000);
            // Check every 10 seconds |
            }


```
**Heartbeat Patterns:**

Pattern 1: Simple Timestamp
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Worker → Redis: SET worker:123:heartbeat 1697712345000

Monitor checks: "Last heartbeat 20 seconds ago" ✓

Pattern 2: Detailed Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Worker →
```json

 Monitor: {
worker_id: "123",
status: "processing",
current_job: "send_emails",
jobs_completed: 450,
memory_usage: "45%",
timestamp: 1697712345000

}
```


Monitor knows: What worker is doing right now\!

Pattern 3: Dead Man's Switch
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Worker updates key every 30 seconds
Key expires after 60 seconds

If worker dies: Key expires automatically
Monitor: "Key missing \= worker dead" ❌

### **Real System Design Example**

Distributed Task Queue System:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Architecture:
            [Queue]

            ← Tasks waiting

               │
    ┌──────────┼──────────┐
    ↓          ↓          ↓
[Worker 1] [Worker 2] [Worker 3]

Health Checks:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Each worker exposes: GET /health

Returns:

```json
 {  "status": "healthy",  "queue_connection": "connected", "current_jobs": 3,  "total_processed": 1024}
 ```


Load balancer checks every 10 seconds

Heartbeats:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Every 30 seconds, each worker:

 Redis SET heartbeat:worker1
 ```json
 {
 timestamp: 1697712345000,
 jobs_in_progress:[{id: "job_123", started: 1697712300000},
 {id: "job_456", started: 1697712320000}]}
 ```

TTL: 60 seconds (auto-expires if worker dies)

Monitor Dashboard:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Worker 1: ✓ Alive (last heartbeat: 15s ago)
          Processing 2 jobs

Worker 2: ✓ Alive (last heartbeat: 8s ago)
          Processing 3 jobs

Worker 3: ❌ DEAD (last heartbeat: 75s ago)
          ALERT SENT
          Auto-restart initiated
          Jobs reassigned to Worker 1 and 2

### **The Complete Monitoring Stack**

Real Production Setup:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


Layer 1: Infrastructure Health

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Server CPU, memory, disk

✓ Network connectivity

✓ Load balancer health checks

Tool: AWS CloudWatch, Prometheus

Layer 2: Application Health

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ /health endpoints

✓ Error rates

✓ Response times

Tool: DataDog APM, New Relic

Layer 3: Business Metrics

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Orders per minute

✓ Revenue tracking

✓ User signups

Tool: Custom dashboard, Grafana

Layer 4: Logs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Application logs

✓ Error traces

✓ Audit logs

Tool: ELK Stack, Splunk

Layer 5: Alerts

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ PagerDuty (on-call)

✓ Slack notifications

✓ Email alerts

Tool: PagerDuty, OpsGenie

---
