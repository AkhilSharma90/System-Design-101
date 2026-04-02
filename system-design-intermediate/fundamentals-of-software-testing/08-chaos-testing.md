---
slug: chaos-testing
title: Chaos Testing
readTime: 10 min
orderIndex: 8
premium: false
---

# Chaos Testing





## **🌪️ Chaos Testing: "Can It Survive Disasters?"**

### **What is Chaos Testing?**

Chaos testing (Chaos Engineering) intentionally **breaks things in production** to ensure system can handle failures.

Philosophy:
"Don't wait for things to break.
 Break them yourself and see what happens!"

                    - Netflix Chaos Monkey

### **Real-World Analogy:**

**Like fire drills in a building:**

Don't wait for real fire to test evacuation plan!

Chaos Test:
1. Pull fire alarm (simulate disaster)

2. Observe: Do people know where to go?

3. Measure: How long does evacuation take?

4. Find: Are there bottlenecks?

5. Improve: Fix issues before real emergency

Test your disaster response BEFORE disaster strikes!

### **Chaos Testing Principles:**

Netflix's Chaos Monkey:
Randomly terminates instances in production

Why?

- Forces engineers to build resilient systems

- Identifies weak points

- Proves system can handle failures

- No surprises during real outages

Philosophy:
"If we never test our backup systems,
 how do we know they work when we need them?"

### **Chaos Experiments:**

#### **1. Instance Failure**

Experiment: "Random Server Crash"

Hypothesis:
"If one application server crashes,
 the system should remain available with no customer impact"

Steady State:
- All servers healthy

- Response time: 100ms

- Error rate: 0%

Introduce Chaos:
→ Kill random application server
   (simulate hardware failure)

Observe:

✅ Load balancer detects failure (5 seconds)

✅ Traffic rerouted to healthy servers

✅ Response time: 120ms (slight increase)

✅ Error rate: 0% (no errors!)

⚠️ Auto-scaling triggered (new instance launched)

Result: System resilient! Experiment confirms hypothesis ✅

Learnings:

- Health check interval: 5 seconds (acceptable)

- Auto-scaling works

- No manual intervention needed

#### **2. Network Partition**

Experiment: "Network Split"


Setup: Microservices architecture

- Frontend

- API Gateway

- User Service

- Order Service

- Payment Service

Introduce Chaos:

→ Block network traffic between Order Service and Payment Service
   (simulate network failure)

Observe:

❌ Bad Design:

Order Service: Tries to call Payment

Payment Service: (unreachable)

Order Service: Hangs for 60 seconds

User: Sees timeout error 💥

Shopping cart: Lost! 💥

✅ Good Design:

Order Service: Tries to call Payment

Payment Service: (unreachable)

Order Service: Timeout after 3 seconds

Order Service: Enqueues order for retry

Order Service: Returns "Order processing" to user

User: Sees confirmation "We're processing your order"

Background job: Retries payment when network restored

Result:
Bad design exposed by chaos testing!
Fix: Implement circuit breaker, async processing

#### **3. Latency Injection**

Experiment: "Slow Database"

Normal: Database queries return in 10ms
Chaos: Add 1000ms latency to all database calls

Observe:

Endpoints affected:

GET /api/users → 1050ms (was 50ms) ❌

GET /api/products → 1020ms (was 30ms) ❌

GET /api/health → 5ms ✅ (doesn't hit DB)

Cascading effects:

- Request queues backing up

- Thread pool exhaustion

- Memory usage increasing

- Timeouts occurring

Improvements needed:

✅ Add database read replicas

✅ Implement caching layer

✅ Add query timeouts

✅ Circuit breaker for DB calls

#### **4. Resource Exhaustion**

Experiment: "Memory Leak"

Introduce Chaos:
→ Gradually consume memory (simulate leak)

Observe timeline:

T+0:   Memory: 2GB / 8GB (25%)

T+10:  Memory: 4GB / 8GB (50%)

T+20:  Memory: 6GB / 8GB (75%) ⚠️

T+25:  Memory: 7GB / 8GB (87%) ⚠️

       → Alerts triggered ✅

       → On-call engineer paged ✅

T+30:  Memory: 8GB / 8GB (100%) ❌

       → Process killed by OS ❌

       → No graceful degradation ❌

What should happen:

✅ Alert at 75% memory usage

✅ Graceful degradation (reject new requests)

✅ Auto-restart before OOM

✅ Load balancer health check fails

✅ Traffic diverted to healthy instances

Improvements:

✅ Implement memory monitoring

✅ Add graceful shutdown

✅ Configure OOM killer properly

### **Chaos Testing Tools:**

1. Chaos Monkey (Netflix)

   - Randomly terminates instances

   - Production testing

   - AWS-focused

2. Chaos Toolkit

   - Declarative chaos experiments

   - Multiple platforms

   - Hypothesis-driven

3. Gremlin

   - Commercial chaos engineering platform

   - CPU, memory, network, disk attacks

   - Safe rollback

4. Litmus (Kubernetes)

   - Chaos for Kubernetes

   - Pod failures, network chaos

   - GitOps-friendly

Example Chaos Toolkit experiment:
```json

 { "title": "Application should survive instance

   failure",  "description": "Kill a random

   instance and verify system remains healthy",  "steady-state-hypothesis":
   {    "title": "Application responds",    "probes": [      {    "type": "probe",        "name": "app-responds",
     "provider": {          "type": "http",
       "url": "https://myapp.com/health",

       "status": 200}}]  },  "method": [    {      "type": "action",

         "name": "terminate-instance",

         "provider": { "type": "python",
           "module": "chaosk8s.pod.actions",
           "func": "terminate_pods","arguments":

           { "label_selector": "app=myapp",

             "rand": true        }      }    }],

       "rollbacks": []}




 ```
### **Chaos Testing Stages:**

Stage 1: Sandbox Testing

  Environment: Local/Dev

  Impact: Zero user impact

  Goal: Learn the tools

Stage 2: Staging Testing

  Environment: Staging/QA

  Impact: Zero user impact

  Goal: Develop experiments

Stage 3: Production Testing (Off-hours)

  Environment: Production

  Time: 2 AM, low traffic

  Impact: Minimal user exposure

  Goal: Validate in real environment

Stage 4: Production Testing (Business hours)

  Environment: Production

  Time: Normal hours

  Impact: Real user exposure

  Goal: Prove resilience under realistic conditions

Note: Only reach Stage 4 after building confidence in Stages 1-3!

**Key characteristics:**

* 💥 **Intentional failures:** Break things on purpose
* 🏭 **Production testing:** Test in real environment
* 🔬 **Hypothesis-driven:** Scientific approach
* 🛡️ **Build resilience:** Expose weaknesses before real failures


---

## Key Takeaways

1. **Chaos testing deliberately injects failures to test system resilience** — kill processes, introduce network latency, fill disks
2. **Netflix pioneered chaos engineering with Chaos Monkey** — randomly terminating production instances to prove the system recovers
3. **Start chaos testing in non-production environments** — build confidence before injecting failures in production
4. **Define steady-state behavior first** — you need to know what "healthy" looks like before you can detect when chaos breaks it
5. **Chaos testing builds confidence in your redundancy and failover mechanisms** — untested failover is not real failover
