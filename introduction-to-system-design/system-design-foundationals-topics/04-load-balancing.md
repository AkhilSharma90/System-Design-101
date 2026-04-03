---
slug: load-balancing
title: Load Balancing
readTime: 20 min
orderIndex: 4
premium: false
---

# Load Balancing

> Distributing incoming traffic across multiple servers so no single machine becomes a bottleneck — the gateway to horizontal scaling.

## **Load Balancing Basics - The Traffic Director**

### **🎯 Challenge 4: The Restaurant Host Problem**

**Scenario:** You own a restaurant with 5 identical dining rooms. Customers arrive at the entrance.

**Without a host:**

Entrance
   │

            ▼

Room 1: ████████████ (packed, 30 min wait)

Room 2: ████         (some people)

Room 3: ████████████ (packed)

Room 4: ██           (almost empty!)

Room 5: ████████████ (packed)

Problem: Customers just pick random rooms!
Result: Long waits, unhappy customers, empty tables

**With a smart host:**

Entrance

   │

   ▼

  Host (sees all rooms, directs customers smartly)

   ├─→ Room 1: ██████  (balanced)

   ├─→ Room 2: ██████  (balanced)

   ├─→ Room 3: ██████  (balanced)

   ├─→ Room 4: ██████  (balanced)

   └─→ Room 5: ██████  (balanced)

Result: No waits, happy customers, efficient!

**This is exactly what a Load Balancer does!**

---

### **🏗️ What is Load Balancing?**

**Definition:** A load balancer is like a smart traffic director that distributes incoming requests across multiple servers.

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771695/262_rajp5f.png)

All servers: Happy and balanced!

**Key insight:** Load balancers prevent any single server from being overwhelmed!

---

### **🎮 Interactive Exercise: Load Balancing Algorithms**

**Scenario:** You have 3 servers. 9 requests arrive. How do you distribute them?

Let's explore different strategies:

#### **Algorithm 1: Round Robin 🔄**

**How it works:** "Take turns, like dealing cards"

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771695/254_ggpk1u.png)

Perfect equality!

**Pros:** Simple, fair, easy to implement **Cons:** Doesn't consider server load or speed

**Real-world example:** Like handing out homework to students in seat order

---

#### **Algorithm 2: Least Connections 📊**

**How it works:** "Send to whoever is least busy right now"

Starting state:

Server 1: 2 active connections

Server 2: 5 active connections

Server 3: 1 active connection

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764773305/load_balance_zj4bnd.png)

New request arrives:

→ Check all servers

→ Server 3 has fewest connections (1)

→ Send request to Server 3!

![img4](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771696/258_e36d0v.png)

**Pros:** Accounts for different request durations

**Cons:** Slightly more complex, needs to track connections


**Real-world example:** Like a grocery store checkout - you join the shortest line!

---

#### **Algorithm 3: Weighted Distribution ⚖️**

**How it works:** "Better servers get more work"

Server setup:

Server 1: Power rating 5 (newest, most powerful)

Server 2: Power rating 3 (medium)

Server 3: Power rating 2 (older, slower)

Total power: 10

Distribution ratio: 5:3:2

For 100 requests:

Server 1 gets: 50 requests (50%)

Server 2 gets: 30 requests (30%)

Server 3 gets: 20 requests (20%)

![img5](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771695/256_ax9fjr.png)

**Pros:** Utilizes hardware differences efficiently

**Cons:** Requires knowing server capabilities


**Real-world example:** Like a restaurant assigning more tables to experienced waiters

---

#### **Algorithm 4: IP Hash 🔐**

**How it works:** "Same client always goes to same server"

How hashing works:

User IP: 192.168.1.100

→ Hash function: hash(192.168.1.100) \= 347

→ 347 % 3 \= 2  (modulo number of servers)

→ Always routes to Server 2!

Same user's next request:

→ hash(192.168.1.100) \= 347  (same!)

→ 347 % 3 \= 2

→ Server 2 again!

Different user IP: 192.168.1.101

→ hash(192.168.1.101) \= 892

→ 892 % 3 \= 1

→ Routes to Server 1

![img6](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771695/261_cklgsr.png))

**Pros:**

* Session persistence (user's data cached on one server)
* Predictable routing

**Cons:**

* Uneven distribution if many users from same IP range
* Server failure requires rehashing

**Real-world example:** Like having a regular doctor - you always see the same one who knows your history

---

### **💡 Types of Load Balancers: Layer 4 vs Layer 7**

🔌 LAYER 4 (Network Load Balancer)

═══════════════════════════════════════

Works at: TCP/UDP level

Sees: IP addresses, ports

Fast: Very (just forwards packets)

Smart: Not very

Example decision:

![img7](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771696/257_ikxhju.png)

Use when: Raw speed needed, simple TCP routing

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🌐 LAYER 7 (Application Load Balancer)

═══════════════════════════════════════

Works at: HTTP/HTTPS level

Sees: URLs, headers, cookies, content

Fast: Slower (must read HTTP)

Smart: Very!

Example decision:
"This request is for /api/users"
→ Route to API servers

"This request is for /images/cat.jpg"
→ Route to image servers

      Internet
         │
         ▼
    ┌─────────┐
    │ Layer 7 │  "GET /api/users"
    │   LB    │  → Route to API cluster
    └─────────┘
         │       "GET /static/logo.png"
         │       → Route to CDN servers
         │
    ┌────┴─────┐
    ▼          ▼
[API Servers] [Static Servers]

Use when: Need smart routing by content

**Real-world comparison:**

* **Layer 4:** Like a highway toll booth - just directs cars, doesn't care what's inside

* **Layer 7:** Like a smart receptionist - reads your request and directs you to the right department

### **🚨 Common Misconception: "Load Balancers Never Fail"**

**You might think:** "Great! Load balancer solves everything!"

**The Reality:** Load balancers themselves can fail or become bottlenecks!

**The Solution: Multiple Load Balancers!**

❌ SINGLE LOAD BALANCER (Risky!)

═════════════════════════════════

![img8](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771696/257_ikxhju.png)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ REDUNDANT LOAD BALANCERS (Safe!)

═════════════════════════════════

![img9](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771695/255_i2kk9d.png)

Safety: If LB 1 fails, LB 2 continues!

---

### **🎪 Real-World Load Balancing Architecture**

**Let's see how a real company like Netflix might structure their load balancing:**

🌍 GLOBAL ARCHITECTURE

═══════════════════════════════════════════

![img10](https://res.cloudinary.com/dretwg3dy/image/upload/v1764771696/260_rfic9f.png)

Each zone has:

├─ Layer 4 Load Balancer (Network LB) ← Third layer: TCP routing

├─ 50+ Web Servers

├─ 20+ API Servers

└─ 10+ Database Read Replicas

**Why multiple layers?**


1. **DNS Layer:** Routes by geography (latency optimization)

2. **Application Layer (L7):** Routes by URL path, HTTP headers

3. **Network Layer (L4):** Raw TCP connection distribution

---

### **🔍 Investigation: What Happens When a Server Dies?**

**Scenario:** You have 4 servers behind a load balancer. Server 3 crashes.

**Without Health Checks:**

Load Balancer: "Round robin time!"

Request 1 → Server 1 ✅ (works)

Request 2 → Server 2 ✅ (works)

Request 3 → Server 3 ❌ (DEAD! User sees error)

Request 4 → Server 4 ✅ (works)

Request 5 → Server 1 ✅ (works)

Request 6 → Server 2 ✅ (works)

Request 7 → Server 3 ❌ (STILL DEAD! Another error)

Result: 25% of requests fail! 😡

**With Health Checks (Smart!):**

Health Check Configuration:

- Check every 10 seconds

- Send HTTP GET to /health

- Expect 200 OK response

- If fails 3 times → Mark unhealthy

Timeline:
10:00:00 - Server 3 crashes

10:00:10 - Health check fails (1/3)

10:00:20 - Health check fails (2/3)

10:00:30 - Health check fails (3/3) → REMOVED!

Load Balancer: "Server 3 is out! Skip it!"

Request 1 → Server 1 ✅

Request 2 → Server 2 ✅

Request 3 → Server 4 ✅ (skipped dead Server 3!)

Request 4 → Server 1 ✅

Request 5 → Server 2 ✅

Result: 0% failures! 🎉

10:05:00 - Server 3 comes back online
10:05:10 - Health check succeeds (1/3)
10:05:20 - Health check succeeds (2/3)
10:05:30 - Health check succeeds (3/3) → ADDED BACK!

Load Balancer: "Welcome back, Server 3!"

**Key insight:** Health checks are CRITICAL for reliability!

---

---

### **🎯 Decision Game: Choose Your Algorithm**

**For each scenario, pick the best load balancing algorithm:**

**Scenario A:** Simple website, all servers identical

**Scenario B:** Video encoding - jobs take 5 seconds to 5 minutes

**Scenario C:** Shopping cart - need same user to hit same server

**Scenario D:** 1 new powerful server \+ 2 older slower servers

**Scenario E:** API with completely independent requests

**Think carefully...**

---

**ANSWERS:**

Scenario A: ROUND ROBIN ✓

Why: Servers identical, simple is best

Scenario B: LEAST CONNECTIONS ✓

Why: Job duration varies widely, need dynamic balancing

Scenario C: IP HASH ✓

Why: Session persistence required for cart data

Scenario D: WEIGHTED ✓

Why: Different server capabilities

Config: New server weight=5, old servers weight=2 each

Scenario E: ROUND ROBIN or RANDOM ✓

Why: Stateless, any algorithm works fine

---

### **🎪 The Complete Picture: Load Balancer Features**

MODERN LOAD BALANCER CAPABILITIES

═══════════════════════════════════════

✅ Traffic Distribution

   ├─ Round Robin

   ├─ Least Connections

   ├─ Weighted

   └─ IP Hash

✅ Health Monitoring

   ├─ HTTP health checks

   ├─ TCP health checks

   ├─ Custom health endpoints

   └─ Automatic server removal/re-addition

✅ SSL/TLS Termination

   ├─ Handle HTTPS encryption

   ├─ Decrypt at load balancer

   └─ Send plain HTTP to backend
      (reduces server CPU load)

✅ Sticky Sessions

   ├─ Cookie-based routing

   └─ Keep user on same server

✅ Rate Limiting

   ├─ Protect from DDoS

   └─ Throttle aggressive clients

✅ Geographic Routing

   ├─ Route EU users to EU servers

   └─ Minimize latency

✅ Content-Based Routing

   ├─ /api/* → API servers

   ├─ /images/* → Image servers

   └─ /admin/* → Admin servers

✅ Metrics & Logging

   ├─ Request counts

   ├─ Response times

   └─ Error rates

---

### **🔑 Key Takeaways: Load Balancing**

📝 REMEMBER:

═══════════════════════════════════════

1️⃣ Load balancers distribute traffic across servers

2️⃣ Multiple algorithms for different needs:

   • Round Robin → Simple rotation

   • Least Connections → Dynamic balancing

   • Weighted → Different server capabilities

   • IP Hash → Session persistence

3️⃣ Health checks prevent routing to dead servers

4️⃣ Layer 4 \= Fast & simple (TCP/IP)
   Layer 7 \= Smart & content-aware (HTTP)

5️⃣ Load balancers themselves need redundancy!

6️⃣ Essential for horizontal scaling

## Load Balancing Algorithms Comparison

| Algorithm | How it works | Best for | Drawback |
|-----------|-------------|----------|----------|
| Round Robin | Rotate through servers sequentially | Equal servers, stateless requests | Ignores server load |
| Least Connections | Route to server with fewest active connections | Varying request durations | Slightly more overhead |
| Weighted Round Robin | Rotate with proportional weight per server | Mixed hardware capabilities | Must configure weights manually |
| IP Hash | Hash client IP to pick server | Session stickiness without cookies | Uneven if IP distribution is skewed |
| Least Response Time | Route to fastest responding server | Latency-sensitive applications | Requires constant health monitoring |
| Random | Pick a server randomly | Simple systems with equal servers | No intelligence at all |

## Layer 4 vs Layer 7

| Feature | Layer 4 (Transport) | Layer 7 (Application) |
|---------|--------------------|-----------------------|
| Operates on | TCP/UDP packets | HTTP requests |
| Speed | Faster (no content inspection) | Slower (reads headers/body) |
| Routing decisions | IP + port only | URL path, headers, cookies, body |
| Use case | Raw throughput, non-HTTP | Content-based routing, API gateways |
| Example | AWS NLB | AWS ALB, Nginx, HAProxy |
