---
slug: cache-eviction
title: Cache Eviction Techniques
readTime: 15 min
orderIndex: 3
premium: false
----



# Cache Eviction Policies \- When the Cache Gets Full
### **🎯 Challenge 3: The Parking Lot Problem**

**Scenario:** You manage a parking lot with 10 spaces (cache). All spaces are full. A new car arrives (new data). Which car do you remove to make space?

**Option A:** Remove the car that's been parked longest (first in, first out)

**Option B:** Remove the car that hasn't moved in the longest time (least recently used)

**Option C:** Remove the car that moves least frequently (least frequently used)

**Option D:** Remove a random car

**Question:** Which strategy makes the most sense for keeping the lot efficient?

### **The Answer: Cache Eviction Policies\!**

When cache is full, we need rules for what to remove. Let's explore each\!

---

### **🥇 LRU (Least Recently Used) \- Most Popular\!**

**Rule:** Remove the item that hasn't been accessed in the longest time.

**Mental model:** If you haven't used it recently, you probably won't need it soon.

**Visual example:**

**![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764582002/227_l7ri3p.png)**



---

**LRU Implementation:**
```js
 class LRUCache {

    constructor(capacity) {

    this.capacity = capacity;

    this.cache = new Map();

    //Maintains insertion order
    }

   get(key) {

      if (!this.cache.has(key)) {

        return null;

      // Cache miss
    }

  // Move to end (mark as recently used)
  const value = this.cache.get(key);

  this.cache.delete(key);

  this.cache.set(key, value);

  return value;

  // Cache hit
  }
  set(key, value) {

  // If exists, remove old entry
  if (this.cache.has(key)) {

        this.cache.delete(key)
  }
  // If full, remove least recently used
  // (first item)

  if (this.cache.size >= this.capacity) {

    const firstKey = this.cache.keys().next().value;

    this.cache.delete(firstKey);

    console.log(\`Evicted: ${firstKey} (LRU)\`);
  }
  //Add new entry at end
  function most_recent()
  {

      this.cache.set(key, value)
  }

// Usage


const lru = new LRUCache(3)

lru.set('A', 1);  // Cache: \[A\]

lru.set('B', 2);  // Cache: \[A, B\]

lru.set('C', 3);  // Cache: \[A, B, C\]

lru.get('A');     // Cache: \[B, C, A\]

//(A moved to end)

lru.set('D', 4);

// Cache:[C, A, D] (B evicted least recent)
````

**When LRU works best:**

✅ In the cases of temporal locality i.e. whenever the most recent data is most likely to  be used  (recent \= likely to reuse)

✅ In case of user sessions on a website recent activity by the user says that he is still logged in and actively using the website whereas the user who hasn’t had activity is more likely to log out or his/her session is going to expire  (recent activity matters)

✅ Reading sequential data (e.g., pagination)

✅ Most general-purpose scenarios

Examples:

\- Browser cache (recently visited pages)

\- Operating system page cache

\- Database query cache

---

### **📊 LFU (Least Frequently Used) \- Count Matters\!**

**Rule:** Remove the item accessed the fewest times.

**Mental model:** If it's rarely used, it's probably not important.

**Visual example:**

Cache (capacity: 4 items) with access counts:

A: 10 accesses

B: 2 accesses

C: 7 accesses

D: 15 accesses

Cache full\! New item E arrives (1 access).
Remove B (least frequently used \- only 2 accesses)

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764582002/225_kyo4x6.png)

---

**LFU Implementation:**

```js
 class LFUCache {
 constructor(capacity) {
 this.capacity = capacity;
 this.cache = new Map();
 this.frequencies = new Map();
 }
 get(key) {
  if (!this.cache.has(key)) {
      return null;  // Cache miss
}   // Increment frequency

this.frequencies.set(key, (this.frequencies.get(key) || 0+1);
return this.cache.get(key);  // Cache hit
}
set(key, value) {
  if (this.cache.size >= this.capacity && !this.cache.has(key)) {
  // Find least frequently used item
  let minFreq = Infinity;
  let lfuKey = null;
  for (let [k, freq] of this.frequencies) {
      if (freq < minFreq) {
        minFreq = freq;
        lfuKey = k;
      }
  }
  // Remove LFU item
  this.cache.delete(lfuKey);
  this.frequencies.delete(lfuKey);
  console.log(`Evicted: ${lfuKey} (LFU - ${minFreq} accesses)`);
}
this.cache.set(key, value);
this.frequencies.set(key, 1);
}
// Usage
const lfu = new LFUCache(3);
lfu.set('A', 1);  // A: freq=1
lfu.get('A');     // A: freq=2
lfu.get('A');     // A: freq=3
lfu.set('B', 2);  // B: freq=1
lfu.set('C', 3);  // C: freq=1
lfu.set('D', 4);  // Must evict: B or C (both freq=1)                  // Evicts B (first with freq=1)
```


**When LFU works best:**

✅ Content popularity matters (viral videos)

✅ Long-term access patterns

✅ Identifying "hot" data

✅ Media streaming platforms

Examples:
\- YouTube video cache (popular videos stay)

\- Spotify song cache

\- News article cache (trending stories)

**LFU Problem: The "Aging" Issue**

Problem:
  Old item: 1000 accesses (from 2 years ago)
  New item: 5 accesses (from today, trending\!)

LFU keeps old item, evicts new trending item\! ❌

Solution: Use "Time-aware LFU" with decay
  \- Reduce frequency counts over time

  \- Or use LRU \+ LFU hybrid

---

### **🎯 FIFO (First In, First Out) \- Simple Queue**

**Rule:** Remove the oldest item (first added), regardless of usage.

**Mental model:** Like a queue at a store \- first person in line leaves first.

**Visual example:**

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764582002/228_rfcxzo.png)

---

**FIFO Implementation:**
```js

 class FIFOCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.cache = new Map();
    this.queue = [];
    // Track insertion order
  }
  get(key) {
    return this.cache.get(key) || null;
    // Note: No reordering on access (unlike LRU)
    }
  set(key, value) {
    // If full and key doesn't exist, remove oldest
    if (this.cache.size>= this.capacity && \!this.cache.has(key)) {

      const oldestKey = this.queue.shift();  //Removefirst
      this.cache.delete(oldestKey);
      console.log(`Evicted: ${oldestKey} (FIFO - oldest)`);
  }
  // If new key, add to queue
  if (\!this.cache.has(key)) {
    this.queue.push(key)
  }
  this.cache.set(key, value);
  }
}
// Usage
const fifo = new FIFOCache(3);
fifo.set('A', 1);  // Queue: [A]
fifo.set('B', 2);  // Queue: [A, B]
fifo.set('C', 3);  // Queue: [A, B, C]
fifo.get('A');     // Queue unchanged: [A, B, C]

fifo.get('A');     // Still unchanged (FIFO doesn't care about access)
fifo.set('D', 4);  // Queue: \[B, C, D\] (A evicted \- oldest) |
```


**When FIFO works best:**

✅ Simple requirements

✅ Time-based expiration matters

✅ All data equally likely to be accessed

✅ Predictable access patterns

Examples:
\- Log file rotation
\- Message queues
\- Simple rate limiting

**FIFO Problems:**

❌ Doesn't consider access patterns

❌ May evict frequently used items

❌ No locality optimization

❌ Generally worse hit ratio than LRU/LFU

---

### **🎪 Comparison: Which Policy Wins?**
```bash
 Test scenario:
 Access pattern: A, B, C, D, E, A, B, A, F
 Cache capacity: 3 items
 LRU (Least Recently Used):
 Access A: [A]
 Access B: [A, B]
 Access C: [A, B, C]
 Access D: [B, C, D]
 (Evict A - least recent)
 Access E: [C, D, E] (Evict B - least recent)
 Access A: [D, E, A] (Evict C - least recent)
 Access B: [E, A, B] (Evict D - least recent)
 Access A: [E, A, B] (A already in cache - move to front)
 Access F: [A, B, F] (Evict E - least recent)
 Hits: 1 (when accessing A second time)
 Misses: 8
 Hit Ratio: 11.1%
 LFU (Least Frequently Used):
 Access A: [A(1)]
 Access B: [A(1), B(1)]
 Access C: [A(1), B(1), C(1)]
 Access D: [A(1), C(1), D(1)] (Evict B \- break tie by order)
 Access E:[A(1), D(1), E(1)] (Evict C)
 Access A: [A(2), D(1), E(1)] (A frequency increases)
 Access B: [A(2), E(1), B(1)] (Evict D - least frequent)
 Access A: [A(3), E(1), B(1)] (A frequency increases)
 Access F: [A(3), B(1), F(1)] (Evict E - least frequent)
 Hits: 2 (A accessed twice)
 Misses: 7
 Hit Ratio: 22.2%
 FIFO (First In, First Out):
 Access A: [A]
 Access B: [A, B]
 Access C: [A, B, C]
 Access D: [B, C, D]
 (Evict A - oldest)
 Access E: [C, D, E] (Evict B - oldest)
 Access A: [D, E, A] (Evict C - oldest)
 Access B: [E, A, B] (Evict D - oldest)
 Access A: [E, A, B] (A already in cache - no change)
 Access F: [A, B, F] (Evict E - oldest)
 Hits: 1 (when accessing A second time)
 Misses: 8
 Hit Ratio: 11.1%
 Winner: LFU (in this specific pattern) |

```
---

### **📊 Real-World Performance Comparison**

**General performance (typical workloads):**

![img4](https://res.cloudinary.com/dretwg3dy/image/upload/v1764582002/226_q9efo8.png)

LRU wins most scenarios\! ✓

**Complexity:**

| Policy | Get Complexity | Set Complexity | Implementation |
| ----- | ----- | ----- | ----- |
| LRU | O(1) | O(1) | Moderate (doubly-linked list \+ hash map) |
| LFU | O(1)\* | O(log n)\* | Complex (multiple data structures) |
| FIFO | O(1) | O(1) | Simple (queue \+ hash map) |

---

### **💡 Choosing the Right Policy**

Choose LRU when:

✓ General-purpose caching

✓ Temporal locality important

✓ Simple to implement

✓ Good performance for most workloads

Choose LFU when:

✓ Popularity matters (hot content)

✓ Long-term patterns

✓ Willing to handle complexity

✓ Media/content streaming

Choose FIFO when:

✓ Simplicity is critical

✓ Time-based expiration

✓ All items equally likely to be accessed

✓ Learning/prototyping

**Most common choice: LRU** (best balance of performance and simplicity\!)
