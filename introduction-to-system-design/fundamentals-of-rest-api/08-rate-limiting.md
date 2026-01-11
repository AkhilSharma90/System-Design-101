---
slug: api-rate-limiting
title: Rate Limiting
readTime: 15 min
orderIndex: 8
premium: false
---

# **🚦 Rate Limiting: The Traffic Control System**

Okay, imagine you own a popular ice cream shop. On a hot summer day, if you let everyone in at once, what happens?

* The shop is overcrowded
* Service slows down
* Ice cream melts
* Staff gets overwhelmed
* Everyone has a bad experience

So what do smart shop owners do? They control the flow \- only let in a certain number of people at a time.

**That's rate limiting\!**

### **Why APIs Need Rate Limiting**

Let’s first see   what happens WITHOUT rate limiting:

Scenario: Your API can handle 1000 requests/second

Malicious User (or bug):

```javascript
 for (let i = 0; i < 1000000; i++) {

 makeAPIRequest()

// SPAM!
}
```

Result:

\- Your server gets 1 MILLION requests instantly

\- Server crashes

\- ALL users (good and bad) can't access your API

\- You're paying for all that bandwidth

\- Your database melts

Cost: Thousands of dollars
Downtime: Hours or days

**With rate limiting:**

After 100 requests from that user:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Response:

429 Too Many Requests

Retry-After: 60
```json

 { "error": "Rate limit exceeded",  "message": "You can make 100 requests per minute. Try again in 60 seconds."}

````

Result:

\- Bad user is blocked

\- Other users keep working fine

\- Server stays healthy

### **Common Rate Limit Strategies**

Let me explain the different approaches:

### **Strategy 1: Fixed Window (The Simple Clock)**

This is the easiest to understand. Imagine a stopwatch that resets every minute:

Example: 100 requests per minute

Minute 1 (00:00 \- 00:59):

━━━━━━━━━━━━━━━━━━━━━━━━━

User makes:


\- 50 requests at 00:30  ✓ Allowed (50/100)

\- 30 requests at 00:45  ✓ Allowed (80/100)

\- 20 requests at 00:50  ✓ Allowed (100/100)

\- 1 request at 00:55    ❌ BLOCKED (101/100)

Minute 2 (01:00 \- 01:59):

━━━━━━━━━━━━━━━━━━━━━━━━━

Counter resets to 0\!
\- 100 requests at 01:01  ✓ Allowed (100/100)

**The Problem:**

User is sneaky:

00:59 → Makes 100 requests  ✓

01:00 → Counter resets\!

01:01 → Makes 100 requests  ✓

Result: 200 requests in 2 seconds\! The "burst problem"

### **Strategy 2: Sliding Window (The Moving Average)**

This is smarter. Instead of fixed time blocks, it looks at a rolling time period:

Example: 100 requests per minute

Current time: 02:30 (2 minutes, 30 seconds)

Server looks back 60 seconds:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━

From 01:30 to 02:30:

Requests:

\- 01:35 → 20 requests

\- 01:50 → 30 requests

\- 02:10 → 40 requests

\- 02:25 → 10 requests

Total: 100 requests in last 60 seconds

New request at 02:30?

❌ BLOCKED (would be 101\)

At 02:36 (after the 01:35 requests expire):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Now counting: 01:36 to 02:36
Only 80 requests (the 20 from 01:35 dropped off)

New request?
✓ ALLOWED (now 81/100)

This prevents the "burst problem"\!

### **Strategy 3: Token Bucket (The Piggy Bank)**

Imagine a piggy bank:

The Rules:

━━━━━━━━━━━

1\. Piggy bank starts with 100 tokens

2\. Each API request costs 1 token

3\. New tokens added at rate of 10/minute

4\. Maximum capacity: 100 tokens

Example Timeline:

━━━━━━━━━━━━━━━━━

Time 00:00 → Bucket has 100 tokens

User makes 50 requests → Bucket now has 50 tokens

Time 01:00 → 10 tokens added → Bucket has 60 tokens

User makes 30 requests → Bucket now has 30 tokens

Time 02:00 → 10 tokens added → Bucket has 40 tokens

Time 10:00 → 10 tokens added every minute for 8 minutes

           → Bucket has 40 \+ 80 \= 100 tokens (can't exceed 100\!)

**The beauty:**

* Allows bursts if you have tokens saved up

* Gradually refills

* Can't exploit by timing resets

### **Real API Response with Rate Limiting**

Let’s see   what a real rate-limited response looks like:

Successful Request:

━━━━━━━━━━━━━━━━━━━

```bash
 GET /api/users


 Response:200 OK

 X-RateLimit-Limit: 1000        ← Your total limit

 X-RateLimit-Remaining: 742     ← How many you have left

 X-RateLimit-Reset: 1730123456← When it resets (Unix timestamp)
 ```
```json
 "users": {[...\]}
 ```
 The client can see:
 "Okay, I have 742 requests left. The limit resets at 1730123456 Let me pace myself


Rate Limit Exceeded:

━━━━━━━━━━━━━━━━━━━━

```bash
 GET  /api/users

 Response:

 429 Too Many Requests

 X-RateLimit-Limit: 1000

 X-RateLimit-Remaining: 0

 X-RateLimit-Reset: 1730123456

 Retry-After: 3600              ← Seconds until you can try again
```

 ```json
 {  "error": "Rate limit exceeded",  "message": "You've made 1000 requests. Limit resets in 1 hour."}
 ```


### **Different Limits for Different Users**

Professional APIs often have tiered limits:

Free Tier:

━━━━━━━━━━


\- 100 requests/hour

\- 1000 requests/day

Example response:
```bash
X-RateLimit-Limit: 100

X-RateLimit-Remaining: 45
```

Pro Tier ($29/month):

━━━━━━━━━━━━━━━━━━━━


\- 10,000 requests/hour

\- No daily limit

Example response:
```bash
X-RateLimit-Limit: 10000

X-RateLimit-Remaining: 8542
```

Enterprise Tier (Custom):

━━━━━━━━━━━━━━━━━━━━━━━


\- Unlimited requests


\- Dedicated servers

Example response:
```bash

X-RateLimit-Limit: unlimited

X-RateLimit-Remaining: unlimited
```

### **How to Design Your Rate Limits**

Let’s lay out a solid design Framework:

**Step 1: Calculate your capacity**

Your server can handle:

\- 10,000 requests/second

\- You have 1000 users

\- Average user makes 100 requests/day

Math: 10,000 req/sec × 60 × 60 \= 36 million req/hour capacity

      1000 users × 100 req/day \= 100,000 req/day needed

You have PLENTY of capacity\!

Safe rate limit: 1000 requests/hour per user
(Far below capacity, but generous for users)

**Step 2: Set tiered limits**

Free: 100/hour  (prevents abuse, encourages upgrade)

Pro: 10,000/hour (reasonable for paid users)

Enterprise: Custom (negotiate based on needs)

**Step 3: Monitor and adjust**

After 1 month:

\- 90% of users never hit limit ✓

\- 5% hit limit occasionally → Probably okay

\- 5% hit limit constantly → Contact them (might be bugs or need upgrade)

### **Rate Limiting Best Practices**

Here are the rules to keep in mind:

**1\. Always return helpful headers**

✓ X-RateLimit-Limit

✓ X-RateLimit-Remaining

✓ X-RateLimit-Reset

✓ Retry-After (when blocked)

**2\. Give clear error messages**

❌ "Too many requests"

✓ "You've made 1000/1000 requests. Limit resets in 45 minutes at 3:00 PM."

**3\. Different limits for different endpoints**

GET /users → 10,000/hour (reading is cheap)

POST /users → 100/hour (writing is expensive)

POST /send-email → 10/hour (external service costs money)

**4\. Document your limits**

Your API docs should clearly state:

\- What the limits are

\- How they're calculated

\- What headers to check

\- What happens when exceeded

---

## **🎓 Putting It All Together: A Complete API**

Let me show you how ALL these concepts work together in a real system:

Real-World Scenario: Photo Sharing API

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1763360912/api_gateway_wvdmoo.png)

Example Request Flow:

━━━━━━━━━━━━━━━━━━━━━

1\. Get photos from a specific user, filtered by date:


```bash

 GET /v2/users/john/photos?from=2025-01-01\&limit=20\&cursor=photo\500
 ↑            ↑                ↑           ↑
 Path param   Query params    Pagination    Pagination

```

2\. Gateway checks:

   ✓ Authentication (Bearer token)

   ✓ Rate limit (500/1000 remaining)

   ✓ Routes to Photos API v2

3\. Photos API returns:

```bash
 200 OK


 X-RateLimit-Remaining: 499
 ```
 ```json

 {     "photos": [20 photos],     "pagination": {

 "next_cursor": "photo_520",


 "has_more": true     }

 }
 ```


4 Client continues scrolling:
```bash

   GET /v2/users/john/photos?cursor=photo\_520\&limit=20

```


## **✅ Final Checklist: Are You Ready?**

You've mastered advanced API concepts if you can:

**API Gateway:**

* \[ \] Explain why gateways centralize authentication
* \[ \] Describe how routing works
* \[ \] Understand when to use a gateway

**Parameters:**

* \[ \] Choose between path and query parameters correctly
* \[ \] Design clean, logical URLs
* \[ \] Understand hierarchical structure

**Pagination:**

* \[ \] Implement offset pagination
* \[ \] Implement cursor pagination
* \[ \] Choose the right method for your use case

**Versioning:**

* \[ \] Identify breaking vs non-breaking changes
* \[ \] Version your APIs properly
* \[ \] Deprecate old versions responsibly

**Rate Limiting:**

* \[ \] Calculate appropriate limits
* \[ \] Return helpful rate limit headers
* \[ \] Implement tiered limits

Congratulations\! You now understand professional API design\! 🎉
