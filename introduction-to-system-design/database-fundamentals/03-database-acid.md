---
slug: database-acid
title: Database Acid Properties
readTime: 15 min
orderIndex: 3
premium: false
---

##  **ACID Properties \- The Four Pillars of Database Reliability**

### **🎯 Challenge 2: The Bank Transfer Mystery**

**Scenario:** You transfer $100 from your account to your friend's account. Mid-transfer, the database server crashes\!

**What should happen to prevent disaster?** A. $100 disappears from both accounts (lost money\!) B. $100 removed from your account but never added to friend's (you lose $100\!) C. $100 added to friend's account but not removed from yours (free money\!) D. The entire transfer is cancelled, and both accounts return to original state

**Think carefully...** What protects users from database failures?

### **The Answer: ACID Properties Ensure Option D\!**

ACID is like a safety net for critical operations. Let's break down each letter:

---

###  A is for ATOMICITY: All or Nothing

**Real-world parallel:** Imagine buying a TV online:

**Without Atomicity (Disaster\!):**

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764423456/204_sajvm7.png)

Result: You paid, TV is reserved, but the system thinks it's still in stock. Chaos\!\! Isn’t it?

**With Atomicity (Protected\!):**

Transaction begins:

  Step 1: Charge credit card

  Step 2: Reserve TV

  Step 3: Update inventory

IF all steps succeed → COMMIT (make it permanent)

IF any step fails → ROLLBACK (undo everything)

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764423455/203_rhkdrw.png)

**The bank transfer example:**


```sql
 BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE user = 'you';  UPDATE accounts SET balance = balance + 100 WHERE user = 'friend';COMMIT;
```
**If crash happens mid-transaction:** Database automatically rolls back BOTH updates. Your $100 stays in your account, friend gets nothing. No money lost or duplicated\!

**Mental model:** Atomicity is like a light switch \- it's either completely ON or completely OFF. There's no half-lit room. A transaction is either fully completed or fully cancelled.

**Key insight:** Atomicity prevents partial operations that would leave data in an inconsistent state\!

---

###  C is for CONSISTENCY: Rules Must Be Followed

**Real-world parallel:** Imagine a university enrollment system with a rule: "No student can be enrolled in more than 5 courses."

**Without Consistency (Rule Breaking\!):**

Student already has 5 courses

System tries to add 6th course → ✓ ALLOWED

Rule violated\! Student overloaded\!

**With Consistency (Rules Enforced\!):**

Student already has 5 courses

System tries to add 6th course → ✗ REJECTED

Error: "Student cannot exceed 5 courses"

Student stays at 5 courses ✓

**Database rules (constraints):**

* Primary keys must be unique (we will learn about primary and foreign keys in upcoming sections)

* Foreign keys must reference existing records

* Balance cannot go negative

* Age must be a positive number

* Email must be unique

**Example with bank accounts:**

 Rule: Balance cannot go negative
 ```sql

 UPDATE accounts SET balance = balance - 1000 WHERE user = 'alice' AND balance = 500;

```
-- Database says NO\! This would make balance = -500
 Transaction REJECTED to maintain consistency

**Mental model:** Consistency is like a bouncer at a club checking IDs. If you don't meet the requirements (rules), you're not getting in\! The database is the bouncer ensuring all rules are followed.

**Key insight:** Consistency ensures the database moves from one valid state to another valid state, never breaking its own rules\!

---

### I is for ISOLATION: No Interference**

**The Coffee Shop Problem:**

**Scenario:** You and your friend both try to buy the last croissant at the same time.

**Without Isolation (Disaster\!):**

You:              Check stock → 1 croissant available ✓

Friend:           Check stock → 1 croissant available ✓

You:              Buy it\! → Success (stock \= 0\)

Friend:           Buy it\! → Success (stock \= \-1) 😱

Result: Two people bought one croissant\! Negative inventory\!

**With Isolation (Protected\!):**

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764423456/205_n5q96c.png)

**Database example \- Concert ticket sales:**

 User A and User B try to buy the last ticket simultaneously

User A:BEGIN TRANSACTION;
```sql
SELECT seats_remaining FROM events WHERE event_id = 123;
```
Returns: 1 seat

Database LOCKS this row
```sql
UPDATE events SET seats_remaining = 0 WHERE event_id = 123;
COMMIT;
```
User B
(tries at the same time)

```sql
BEGIN TRANSACTION;  SELECT seats remaining FROM events WHERE event_id = 123;
```
Must WAIT for User A's transaction to complete

After A commits,

B sees: 0 seats

B's purchase fails ✓

**Isolation Levels (How strict is the isolation?):**

**Read Uncommitted** (Least strict)

* Can see changes from other transactions before they commit
* Like reading someone's draft email while they're still writing it
* Fast but risky\!

**Read Committed** (Common default)

* Can only see changes after other transactions commit
* Like only reading sent emails, not drafts
* Good balance

**Repeatable Read**

* Same query always returns same data during your transaction
* Like reading a frozen snapshot
* Very safe

**Serializable** (Most strict)

* Transactions run as if they were completely alone
* Like having exclusive access to the database
* Slowest but safest

**Mental model:** Isolation is like phone booth conversations \- you don't want two conversations interfering with each other\! Each transaction gets its own "booth" to work in.

**Key insight:** Isolation prevents concurrent transactions from interfering with each other and causing data corruption\!

---

### **🅳 D is for DURABILITY: Permanent and Safe**

**The Power Outage Scenario:**

**You just transferred $1,000 and got confirmation: "Transfer successful\!"**

**Suddenly:** ⚡ Power outage\! Server crashes\! 💥

**Question:** Is your transfer saved or lost?

**Without Durability (Nightmare\!):**

Transaction completed → "Success" shown
System crashes before writing to disk
Data stored only in RAM → LOST\!
You restart → Transfer never happened 😱

**With Durability (Protected\!):**

Transaction completed → IMMEDIATELY written to disk
Multiple copies stored (redundancy)
Even if system crashes → Data is PERMANENT
You restart → Transfer is there ✓

**How databases ensure durability:**

**Write-Ahead Logging (WAL):**

![img4](https://res.cloudinary.com/dretwg3dy/image/upload/v1764423455/202_hxrf0y.png)

**Mental model:** Durability is like signing a legal contract. Once it's signed and filed, even if the building burns down, the contract exists (because copies are stored safely). Your database transaction is the same\!

**Real-world parallel:** Think of saving a document:

* **Without Durability:** "Saved\!" but only in RAM. Crash \= lost work 😢
* **With Durability:** "Saved\!" and immediately written to hard drive. Crash \= work is safe ✓

**Key insight:** Durability guarantees that once a transaction is committed, it's permanent \- even if power fails, servers crash, or disasters strike\!

---

### **🎪 ACID Together: The Complete Safety System**

**The bank transfer example with ALL ACID properties:**

Transfer $100 from Alice to Bob:

🅰️ ATOMICITY:
   Both updates happen or neither happens

🅲 CONSISTENCY:
   Alice must have $100 (can't go negative)
   Total money in system stays the same

I ISOLATION:
   If Carol also transfers to Bob simultaneously,
   transactions don't interfere with each other

🅳 DURABILITY:
   Once confirmed, transfer is permanent
   even if server crashes immediately after

Protected by:

✓ All-or-nothing execution

✓ Rules enforced (balances valid)

✓ No interference from other transfers

✓ Permanent once confirmed

---

### **💡 Challenge: ACID or Not ACID?**

**Scenario 1:** Social media "likes" counter

* Thousands of people like simultaneously
* Occasional miscount of \+/- 1 is acceptable
* Speed is critical

**ACID needed?** 🤔

**Answer:** NO\!

* Eventual consistency is fine
* NoSQL databases often trade ACID for speed/scale
* Called "BASE" properties (Basically Available, Soft state, Eventually consistent)

---

**Scenario 2:** Medical prescription system

* Doctor prescribes medication
* Must update patient record AND pharmacy inventory
* No room for error

**ACID needed?** 🤔

**Answer:** YES\! ✓

* Patient safety depends on accuracy
* Both updates must succeed or fail together
* Need full ACID guarantees

---
