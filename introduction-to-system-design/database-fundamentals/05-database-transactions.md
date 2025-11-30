---
slug: database-transactions
title: Database Transactions
readTime: 10 min
orderIndex: 5
premium: false
---




##  **Database Transactions \- The All-or-Nothing Guarantee**

### **🎯 Challenge 1: The Concert Ticket Catastrophe**

**Scenario:** You're buying concert tickets online. The purchase involves multiple steps:

1. Check if seats are available (2 tickets for Section A)
2. Reserve those seats
3. Charge your credit card $200
4. Send confirmation email
5. Update ticket inventory

**Question:** Your internet crashes after Step 3 (card charged). What should happen?

A. Card charged, tickets not reserved → You paid but got nothing\! 😱 B. Tickets reserved, card not charged → Free tickets\! (Chaos for venue) C. Partial completion → Database is confused about what happened D. Entire purchase cancelled, card refunded → Everything rolls back safely

**Think carefully...** What protects both you and the venue?

### **The Answer: Database Transactions\!**

**A transaction is a group of operations that either ALL succeed or ALL fail together \- never partial\!**

Real-world parallel: Think of a transaction like a legal contract signing ceremony. You don't partially sign a contract\! Either all parties sign and it's binding, or someone refuses and nothing happens. There's no "half-signed" contract.

---

### **🎬 How Transactions Work: The Movie Analogy**

**Imagine filming a scene in a movie:**

**Without Transactions (Disaster\!):**

Director: "Action\!"
Actor 1: Delivers perfect line ✓
Actor 2: Forgets their line ✗
Actor 3: Hasn't even started

Director: "Cut\! But Actor 1's part is already filmed..."
Result: Unusable footage, must reshoot everything

**With Transactions (Protected\!):**

Director: "BEGIN TRANSACTION \- Action\!"
Actor 1: Delivers perfect line ✓
Actor 2: Forgets their line ✗

Director: "ROLLBACK\! Reset to starting positions"
Result: Like it never happened, start fresh\!

Second take:
Director: "BEGIN TRANSACTION \- Action\!"
Actor 1: Perfect ✓
Actor 2: Perfect ✓
Actor 3: Perfect ✓

Director: "COMMIT\! That's the take we're keeping\!"
Result: Scene is officially recorded

---

### **🎮 Transaction Commands: Your Control Panel**

**The Three Essential Commands:**

**1️⃣ BEGIN TRANSACTION (or START TRANSACTION)**

BEGIN TRANSACTION;
\-- Like saying "Start recording, this is one atomic operation"

**2️⃣ COMMIT**

COMMIT;
\-- Like saying "Save it\! Make all changes permanent"

**3️⃣ ROLLBACK**

ROLLBACK;
\-- Like saying "Undo everything\! Restore to how it was before"

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764474744/209_jyku0g.png)

---

### **💰 Real-World Example: The Bank Transfer**

**Scenario:** Transfer $500 from Alice to Bob

**Without Transaction (DISASTER ZONE):**

\-- Step 1: Deduct from Alice
```sql

UPDATE accounts SET balance = balance - 500 WHERE name = 'Alice';

```

\-- ✓ Success\! Alice now has $500 less

\-- Step 2: Add to Bob

\-- 💥 DATABASE CRASHES HERE\!

\-- Result: Alice lost $500, Bob never received it\! Money vanished\! 😱

**With Transaction (PROTECTED):**

BEGIN TRANSACTION;

\-- Step 1: Deduct from Alice
```sql
UPDATE accounts SET balance = balance - 500 WHERE name = 'Alice';

```

\-- Balance: Alice $1000 → $500 (temporary, not permanent yet\!)

\-- Step 2: Add to Bob
```sql
UPDATE accounts SET balance = balance + 500 WHERE name = 'Bob';
```

 Balance: Bob $500 → $1000 (temporary, not permanent yet\!)

\-- Check if both succeeded

IF both\_steps\_successful THEN

  COMMIT;

  \-- Make both changes permanent\!

  \-- Result: Alice $500, Bob $1000 ✓

ELSE

  ROLLBACK;

  \-- Undo both changes\!

  \-- Result: Back to original state: Alice $1000, Bob $500

END IF;

**Visualization:![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764474744/210_pjekrr.png)**

---

### **🔍 Investigation: Transaction States**

**A transaction goes through distinct states like a package in shipping:**

📦 PACKAGE ANALOGY:![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764474744/211_xrhxyq.png)



**In SQL terms:**

\-- State 1: ACTIVE (Transaction in progress)

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2
```

State 2a: COMMITTED (Success path)

COMMIT;

All changes now permanent and visible to everyone

OR State 2b:

ABORTED (Failure path)

ROLLBACK;  -- All changes discarded, database unchanged!


### **🎪 Interactive Exercise: Can This Be One Transaction?**

**Which of these should be a SINGLE transaction? Think about it:**

**Scenario A:** User registration

* Insert new user record
* Send welcome email
* Create default preferences
* Log registration event

**Should this be ONE transaction?** 🤔

**Answer:** NO\! Here's why:

\-- User record and preferences: YES, one transaction

BEGIN TRANSACTION
```sql
INSERT INTO users (name, email) VALUES ('Alice', 'alice@email.com');

INSERT INTO preferences (user\_id) VALUES (LAST\_INSERT\_ID());

COMMIT;
```

\-- Email and logging: SEPARATE operations
\-- (External services, can fail independently)
send\_email('alice@email.com', 'Welcome\!');
log\_event('user\_registered', user\_id);

**Mental model:** Only database operations that MUST succeed together should be in one transaction. External services (email, APIs) should be separate\!

---

**Scenario B:** E-commerce checkout

* Deduct items from inventory
* Create order record
* Process payment
* Update customer points

**Should this be ONE transaction?** 🤔

**Answer:** MOSTLY YES, but payment is tricky\!
```sql
 BEGIN TRANSACTION;
  UPDATE products SET stock = stock- 1 WHERE id = 123;
  INSERT INTO orders (customer_id, total) VALUES (1, 99.99);

  UPDATE customers SET points = points + 10 WHERE id = 1;  IF (stock < 0)
  THEN    ROLLBACK;
  ELSE
  COMMIT;
  END IF;
  END TRANSACTION;
  ```


\-- Payment processing happens separately
\-- (External payment gateway, different rules)

**Key insight:** Transactions are for database consistency, not for orchestrating external systems\!

---

### **🚨 Common Transaction Pitfalls**

| ❌ Mistake 1: Transaction Too Long

Bad transaction example:
```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;

User thinks for 10 minutes... ⏰

Meanwhile, account is LOCKED! 🔒

Other users can't access this account

COMMIT;
```

Good transaction example:

 Get data first (outside transaction)

```sql

SELECT balance FROM accounts WHERE id = 1;

```

// User thinks... no locks held

// Only use transaction for the actual update

```sql

BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;

COMMIT;
```
 Done in milliseconds\! ⚡ |




**Mental model:** Transactions are like holding your breath underwater \- keep them SHORT\!

---

**❌ Mistake 2: Forgetting to COMMIT or ROLLBACK**

**Bad transaction example:**

```sql
 BEGIN TRANSACTION;

 UPDATE products SET price = 99.99 WHERE id = 1;

 Oops Forgot to COMMIT or ROLLBACK

 Transaction stays ACTIVE forever

 Locks held indefinitely 🔒😱

 Good transaction example:

 BEGIN TRANSACTION;

 UPDATE products SET price = 99.99 WHERE id = 1;

 COMMIT;
 ```

 \-- Always end your transactions\!

 Or use try-catch:
 ```javascript

 try {
  await db.query('BEGIN TRANSACTION');

  await db.query('UPDATE products SET price = 99.99 WHERE id = 1');

  await db.query('COMMIT');}

  catch (error) {
  await db.query('ROLLBACK');  // Clean up on error

  throw error;

  }
  ```

---

**❌ Mistake 3: Nested Transactions (Not What You Think\!)**

**Confusing:**
```sql
 BEGIN TRANSACTION;

 Transaction 1

 UPDATE table1 SET value = 1;

 BEGIN TRANSACTION;

  //This doesn't create Transaction 2

 UPDATE table2 SET value = 2;

 COMMIT;

 // This doesn't commit anything yet

 COMMIT;

 // This commits EVERYTHING (both updates)
 ```


**Most databases don't support true nested transactions\!** Instead, they flatten to one transaction.

**Better approach \- Savepoints:**

```sql
 BEGIN TRANSACTION

   UPDATE table1 SET value = 1;

   SAVEPOINT my_savepoint;

   //Bookmark this point

   UPDATE table2 SET value = 2;

   // Oops, this failed!

   ROLLBACK TO my_savepoint;

   Undo only UPDATE table2

   UPDATE table3 SET value = 3;

  // This still happens

   COMMIT;

   //Commits UPDATE table1 and UPDATE table3
   ```


---

### **💡 Transaction Isolation Levels: How Isolated Are You?**

**Remember isolation from ACID? Here's the depth:**

**The Dinner Party Analogy:**

**Read Uncommitted (Least Isolated)**

You: Reading someone's plate while they're still cooking on it
Problem: You might see raw chicken\! (Dirty Read)
Real issue: You read data that might get rolled back

**Read Committed (Default for most databases)**

You: Only eat dishes that are served (finished cooking)
Protection: No dirty reads
Problem: If chef replates the dish, it looks different (Non-repeatable Read)

**Repeatable Read**

You: Your plate is photographed at start
Protection: Your view stays consistent throughout meal
Problem: New dishes might appear (Phantom Reads)

**Serializable (Most Isolated)**

You: Eat alone, one person at a time
Protection: Complete isolation, as if transactions run in sequence
Problem: Slowest performance

**Example showing the differences:**

\-- Scenario: Two people checking same account balance

```sql
 Transaction A (You)


 BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

 SELECT balance FROM accounts WHERE id = 1;
 // Shows: $1000

 // Wait 5 seconds...

 SELECT balance FROM accounts WHERE id = 1;

// Might show: $500 (if Transaction B committed)

// This is Non-Repeatable Read

 COMMIT;
 ```

  Transaction B (Someone else, simultaneous)
  ```sql

  BEGIN TRANSACTION;

  UPDATE accounts SET balance = 500 WHERE id = 1;

  COMMIT;

  With Repeatable Read:

  Transaction A


  BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

  SELECT balance FROM accounts WHERE id = 1;

  // Shows: $1000

  //Wait 5 seconds...

  SELECT balance FROM accounts WHERE id = 1;

  // Still shows: $1000 (consistent snapshot)

  COMMIT;

---

### **🎯 Quick Recap: Transaction Essentials**

**Test yourself \- without looking back:**

1. What happens if a transaction is interrupted mid-execution?
2. Why should transactions be kept short?
3. What's the difference between COMMIT and ROLLBACK?
4. Can you have transactions within transactions?

**Answers:**

1. If not committed, all changes are automatically rolled back (ACID's Atomicity\!)
2. Long transactions hold locks, blocking other users and increasing deadlock risk
3. COMMIT makes changes permanent, ROLLBACK discards all changes
4. Most databases don't support true nesting; use savepoints instead
