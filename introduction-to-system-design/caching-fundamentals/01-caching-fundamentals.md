---
slug: caching-fundamentals
title: Fundamentals of Caching
readTime: 10 min
orderIndex: 1
premium: false
---

#  What is Caching? \- Your Speed Multiplier**

### **🎯 Challenge 1: The Library Puzzle**

**Scenario:** You're a librarian. A student asks for "Harry Potter and the Sorcerer's Stone" \- a book requested 20 times per day.

**Method A: The Thorough Approach**

* Walk to back storage room (5 minutes)
* Search through shelves (3 minutes)
* Retrieve book (2 minutes)
* Walk back to front desk (5 minutes)
* Total: 15 minutes per request
* Daily time: 15 min × 20 requests \= **300 minutes (5 hours\!)**

**Method B: The Smart Approach**

* Keep a copy of popular books at the front desk
* Student asks → grab from desk (10 seconds\!)
* Total: 10 seconds per request
* Daily time: 10 sec × 20 requests \= **200 seconds (3 minutes\!)**

**Question:** Which method serves students better? How much time do you save?

**Answer:** Method B saves 297 minutes (almost 5 hours\!) daily \- that's **99.97% faster\!**

### **The Answer: This is Caching\!**

**Definition:** Caching stores frequently accessed data in a fast, nearby location so you don't have to retrieve it from the slow, distant source every time.

---

### **🧠 The Memory Hierarchy: Why Caching Exists**

**Your computer has different storage speeds:**

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764489179/217_raogsx.png)

**Speed comparison in human terms (**all theoretical comparisons only)

If accessing RAM takes 1 second: ()

* L1 Cache: 0.02 seconds (blink of an eye)
* SSD: 16 minutes (theoretically)
* HDD: 1.5 days
* Network: 1+ weeks

**Key insight:** Caching moves data from slow storage to fast storage\!

---

### **🏪 Real-World Caching Analogy: The Grocery Store**

**Without Cache (Always go to the warehouse):**

You need milk:
1\. You drive to warehouse (30 min) 🚗
2\. You search for milk (10 min) 🔍
3\. You drive back  home (30 min) 🚗
Total: 70 minutes

You need bread:
1\. You drive to warehouse again (30 min) 🚗
2\. You search for bread (10 min) 🔍
3\.  You drive back home (30 min) 🚗
Total: 70 minutes

Daily time for 5 items: 350 minutes\! 😰

**With Cache (Local grocery store):**

You need milk:
1\. You walk to corner store (2 min) 🚶
2\. You grab the  milk (1 min) 🥛
3\. You walk back  home (2 min) 🚶
Total: 5 minutes

You need bread:
1\. You walk to corner store (2 min) 🚶
2\. You grab the  bread (1 min) 🍞
3\. You walk back  home (2 min) 🚶
Total: 5 minutes

Daily time for 5 items: 25 minutes\! ⚡

**Mental model:** The corner store is your cache \- it stocks popular items locally for quick access. The warehouse is your database \- it has everything but it's far away\!

---

### **💻 Caching in Web Applications**

**![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764489178/216_o9zu73.png)**

---

### **🎪 Types of Caching: Where Data Gets Stored**

**1\. Browser Cache**

**Location**: It is located in our computer
**Stores**:  It basically stores images, CSS, JavaScript, HTML
**Speed**: Instant (already on your device)
**Example**: Facebook logo doesn't re-download every page load

**2\. CDN Cache**

**Location:** It is located at  edge servers worldwide across different locations around the world
**Stores**: It generally stores static content (images, videos, files)
**Speed**:  It is very fast (geographically close)
**Example**: Netflix videos served from nearby servers

**3\. Server Cache**

**Location**: It is a web server memory (Redis, Memcached)
**Stores**: It generally stores database query results, session data, computed values
**Speed:** Fast (in-memory)
**Example**: User profile data, product listings

**4\. Database Cache**

**Location**: It is located at the database memory
**Stores**: It queries  results, frequently accessed rows
**Speed**: Fast (avoids disk I/O)
**Example**: Most-read blog posts

**5\. Application Cache**

**Location**: Application memory
**Stores**: Object instances, computed results
**Speed**: Very fast (in-process)
**Example**: Configuration settings, lookup tables

---

### **🎮 Interactive Exercise: What Should You Cache?**

**Decide if each scenario should use caching:**

| Scenario | Cache It? | Why? |
| ----- | ----- | ----- |
| User's account balance in banking app | ? | ? |
| Product catalog with 10,000 items | ? | ? |
| Weather forecast for a city | ? | ? |
| Live stock prices | ? | ? |
| User's profile picture | ? | ? |

**Think about each one...**

---

**Answers:**

1. **Account Balance: ❌ NO** (or very short TTL (time to live))

   * Account Balances changes frequently
   * It must always be accurate
   * Cache for 1-2 seconds max if needed
   * **Mental model:** Money must be precise\!
2. **Product Catalog: ✅ YES**

   * A product catalog changes infrequently
   * It is read far more than written
   * We can cache it  for 5-60 minutes
   * **Mental model:** Products don't change every second
3. **Weather Forecast: ✅ YES**

   * Weather forecast updates every 30-60 minutes
   * It is usually the same data for all users in city
   * Cache for 30 minutes
   * **Mental model:** Weather doesn't change instantly
4. **Live Stock Prices: ❌ NO** (or 1-second cache)

   * Stock prices changes every millisecond
   * Users expect real-time prices to carry out high frequency trade
   * Maybe 1-second cache for high traffic
   * **Mental model:** "Live" means fresh\!
5. **Profile Picture: ✅ YES**

   * Rarely changes
   * Same for all viewers
   * Cache for 24 hours or longer
   * **Mental model:** People don't change photos often

---

### **📋 The Caching Decision Framework**

**Cache when data is:**

* ✅ Read frequently (high traffic)
* ✅ Expensive to generate (complex queries)
* ✅ Relatively stable (doesn't change often)
* ✅ Identical for multiple users
* ✅ Acceptable if slightly stale

**Don't cache when data is:**

* ❌ Written more than read
* ❌ Must be 100% fresh (financial transactions)
* ❌ User-specific and rarely reused
* ❌ Cheap to generate
* ❌ Privacy-sensitive

**The Golden Rule:**

Cache value \= **(Access frequency × Generation cost) / Freshness requirement**

If Cache value \> Threshold → Cache it\!

---

### **💡 Simple Caching Example**

 Without Cache:
// Every request hits database
```javascript
app.get('/products', async (req, res) => {

  // Query takes 100ms
  const products = await db.query('SELECT \* FROM products');

  res.json(products);}

// 100 requests = 100 database queries = 10 seconds total

// With Cache:
 const cache = {}
 app.get('/products', async (req, res) => {

  // Check cache first

  if (cache['products']) {

    // Cache hit! Return immediately (1ms)

         return res.json(cache['products'])
  }

  // Cache miss - query database

  const products = await db.query('SELECT * FROM products');
  // Store in cache for future requests/

  cache['products'] = products;

  res.json(products);

  // 100 requests = 1 database query + 99 cache hits = 0.2 seconds! ⚡ 50x  faster



---
```

### **🚨 Common Caching Mistakes**

| ❌ Mistake 1: Caching Everything

// BAD: Cache user's shopping cart

```javascript
cache.set(`cart:${userId}`, cart);
````

// Changes every second

// BAD: Cache current time
```javascript

cache.set('current_time', new Date());
```

// Always stale

// Mental model: Don't cache what changes constantly !❌ Mistake

2: Never Expiring Cache

// BAD: Cache forever
```javascript
cache.set('products', data);
```

// What if product prices change?

// GOOD: Set expiration (TTL - Time To Live)

```javascript

cache.set('products', data, { ttl: 300 });  // 5 minutes ❌

```

Mistake 3: Cache Stampede

PROBLEM: Cache expires, 1000 requests hit database simultaneously

// All 1000 requests see cache miss

All 1000 query database at once

// Database overload\! 💥

// SOLUTION: Use locking or cache warming
