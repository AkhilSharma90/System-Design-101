---
slug: database-read-write
title: Database Read Write Operations
readTime: 12 min
orderIndex: 7
premium: false
---

##  **Read vs Write Operations \- Understanding the Flow**

### **🎯 Challenge 2: The Library Rush**

**Scenario:** A library has 1000 students:

* 950 students are browsing books (reading data)
* 50 students are checking out books (writing data)

**Question:** Which operation is more demanding on the library system?

Think about it:

* Reading is just looking at information
* Writing requires updating records, checking availability, locking resources
* Which needs more coordination?

### **The Answer: Writes Are Much More Expensive\!**

**The fundamental truth:** Reads are cheap, writes are expensive.

---

### **📚 Understanding Read Operations**

**What happens during a READ:**

```sql

 SELECT * FROM users WHERE username = 'alice';
 ```

**Behind the scenes:**

1\. Find data location (using index if available) ⚡

2\. Read from disk/memory 📖

3\. Return result ✓


No locks (in most cases)

No validation needed

No disk writes

Multiple reads can happen simultaneously

**Mental model:** Reading is like looking at a museum painting \- everyone can look at the same time, no one interferes with each other\!

**Types of Reads:**

```sql

 SELECT \* FROM users WHERE username = 'alice';
 ```

**Read characteristics:**

* ✅ Fast and efficient

* ✅ Can happen in parallel

* ✅ Don't block other reads

* ✅ Cheap to scale (add read replicas)

* ❌ Can block writes (in some isolation levels)

---

### **✏️ Understanding Write Operations**

**What happens during a WRITE:**
```sql
 UPDATE users SET email = 'new@email.com' WHERE id = 123;
 ```

**Behind the scenes (so much more complex\!):**

1. BEGIN TRANSACTION 🔄

2. Acquire LOCK on the row 🔒

3. Read current data 📖

4. Validate constraints (unique, foreign keys, etc.) ✓

5. Write to transaction log (WAL) 📝

6. Update indexes 🗂️

7. Write actual data change 💾

8. Release locks 🔓

9. COMMIT transaction ✅

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764481458/214_w8h8ne.png)

Blocks concurrent writes to same row

Requires validation

Multiple disk writes

Slower than reads

**Mental model:** Writing is like editing a shared Google Doc - only one person can edit a specific paragraph at a time, changes must be saved, version history updated, etc.

**Types of Writes:**


INSERT (simplest write):

```sql
INSERT INTO users (name, email) VALUES ('Bob', 'bob@email.com');
```

 Must:- Generate new ID

 Check constraints (email unique?)

Update indexes

  Write to log

UPDATE (complex write):

```sql
UPDATE products SET price = 99.99 WHERE category = 'Electronics';
```

 Must:
- Find all matching rows

- Lock them

- Update each one

- Update all relevant indexes

- Maintain consistency



  DELETE (most complex):
  ```sql

  DELETE FROM users WHERE id = 123;
  ```

   Must:

   Check foreign key constraints (does anything reference this?)

   Lock the row

   Remove from indexes

   Mark as deleted

   Handle cascade deletes


**Write characteristics:**

* ❌ Slower than reads

* ❌ Require locks (blocking)

* ❌ Multiple validation steps

* ❌ Multiple disk writes

* ❌ Harder to scale

* ✅ Ensure data integrity

---

### **📊 The 80/20 Rule (or 95/5 in reality\!)**

**Real-world application traffic:**

📊 Typical Web Application:

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764481458/215_rxilqb.png)

Examples:

\- Social media: 99% reading posts, 1% creating posts

\- E-commerce: 95% browsing, 5% purchasing

\- News sites: 99.9% reading, 0.1% commenting

**Why this matters for architecture:**

**Read-Heavy Application (like Twitter):**

Strategy: Optimize for reads\!

\- Use caching heavily (Redis, Memcached)

\- Add read replicas

\- Denormalize data for faster reads

\- Use CDNs for static content

**Write-Heavy Application (like logging systems):**

Strategy: Optimize for writes\!

\- Batch writes together

\- Use write-optimized databases

\- Async processing

\- Eventual consistency

---

### **🎮 Interactive Exercise: Read or Write?**

**Classify these operations:**

| Operation | Read or Write? | Why? |
| ----- | ----- | ----- |
| `SELECT COUNT(*) FROM users` | ? | ? |
| `UPDATE users SET last_login = NOW() WHERE id = 1` | ? | ? |
| `SELECT * FROM posts ORDER BY created_at` | ? | ? |
| `INSERT INTO logs (message) VALUES ('error')` | ? | ? |

**Think about each one...**

**Answers:**

1. **READ** \- Counting doesn't modify data

2. **WRITE** \- Modifying data, requires transaction, locks

3. **READ** \- Just retrieving and sorting data

4. **WRITE** \- Adding new data, requires transaction

---

### **🔥 Performance Comparison: Real Numbers**

**Scenario: Database with 1 million users**

**Read Operation:**
```sql

SELECT * FROM users WHERE id = 12345;
```

Time: 0.5-2ms ⚡
Disk I/O: 1 read
Locks: None (Read Committed)
Concurrent: Unlimited simultaneous reads

**Write Operation:**
```sql

UPDATE users SET last_name = 'Smith' WHERE id = 12345;
```

Time: 5-50ms ⏱️
Disk I/O: 3-5 writes (data, log, indexes)
Locks: Row-level lock 🔒
Concurrent: Blocks other writes to same row
Validation: Check constraints, update indexes

**Visual comparison:**

READ:  ⚡ ────✓ (Fast\! One hop)

Write is 10-25x slower than read\!


![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764481458/213_cfax8z.png)

---

### **🚀 Optimization Strategies**

**For Read-Heavy Systems:**

**1\. Caching (we shall learn more about caching in the upcoming sections)**

First request:
User → Database → 50ms ⏱️

With Cache:
User → Cache → 1ms ⚡
(50x faster\!)

Only hit database on cache miss

**2\. Read Replicas** (read replicas will be discussed in upcoming lessons as well)

Primary Database (handles writes)
    ↓ (replicates to)
Read Replica 1  Read Replica 2  Read Replica 3
    ↓               ↓               ↓
Users          Users           Users

Distribute read load across replicas\!

**3\. Indexing**

 Without index: Scan 1,000,000 rows ⏱️

```sql

SELECT * FROM users WHERE email = 'alice@email.com';
```

 With index: Jump to specific row ⚡

 ```sql

CREATE INDEX idx\_email ON users(email);
```



**For Write-Heavy Systems:**

**1\. Batch Writes**

| // Bad example:

1000 individual writes
```javascript
for (let i = 0; i < 1000; i++) {

  await db.query('INSERT INTO logs (message) VALUES (?)', \[msg[i]]);}

  // Time: 1000 × 5ms = 5000ms 😰
  ```

// Good: 1 batched write

```javascript
await db.query(  'INSERT INTO logs (message) VALUES (?), (?), ... (all 1000)',  messages);

// Time: 50ms ⚡ (100x faster\!)
```


**2\. Async Processing**

User Action (POST /order)
    ↓

Quick Response: "Order received\!" ✅
    ↓

Background Job Queue 📋
    ↓

Process writes asynchronously

**3\. Write-Optimized Storage**

Traditional Database: Update in place

\- Read old value

\- Modify

\- Write back

\- Update indexes

(Slow\! Many disk seeks)

Log-Structured Storage: Append only

\- Just write new entry at end

\- Never modify existing data

\- Periodically compact

(Fast\! Sequential writes)

---

### **🎯 Mental Models Summary**

**Reading \= Looking at a book**

* Many people can look at the same time
* No permission needed
* Fast and easy
* Doesn't change anything

**Writing \= Editing a shared document**

* Need to coordinate with others
* Requires permission (locks)
* Must validate changes
* Changes need to be saved
* Slower and more complex

**Key Insight:** Design your application architecture around the read/write ratio\! Most applications are read-heavy, so optimize for reads first.
