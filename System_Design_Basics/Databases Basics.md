# **Database Fundamentals: An Interactive Journey Through SQL, NoSQL, and Core Concepts**

## **🗄️ Part 1: SQL vs NoSQL \- The Great Database Divide**

### **🎯 Challenge 1: The Library Dilemma**

**Scenario:** You're designing a system to store data. You have two library options:

**Library A (Traditional):**

* Books organized in strict categories  
* Every book must fill out the same form: Title, Author, ISBN, Publication Year, Genre  
* Card catalog system with cross-references  
* If you want to add a new field (like "Translator"), you must reorganize the ENTIRE library

**Library B (Modern):**

* Books stored in flexible bins  
* Each book can have different information  
* Some have authors, some don't  
* Easy to add new types of information anytime  
* No strict card catalog, but ultra-fast search

**Pause and think:** Which library would you use for:

* A bank storing customer accounts?  
* A social media app storing user posts?  
* An e-commerce site storing product reviews?

### **The Answer: SQL vs NoSQL in a Nutshell**

**SQL (Structured Query Language) Databases \= Library A**

* Think: Excel spreadsheets with strict rules  
* Data organized in tables with fixed columns (schema)  
* Perfect for structured, predictable data  
* Examples: MySQL, PostgreSQL, Oracle, SQL Server

**NoSQL (Not Only SQL) Databases \= Library B**

* Think: Flexible containers adapting to any content  
* Data stored in various formats (documents, key-value pairs, graphs)  
* Perfect for flexible, rapidly changing data  
* Examples: MongoDB, Cassandra, Redis, DynamoDB

**Key Insight:** The choice isn't about "better" \- it's about the right tool for your specific problem\!

---

### **🏢 Interactive Comparison: The Office Building Analogy**

**SQL Database \= Corporate Office Building**

| EmpID (Number) | Name (Text) | Dept (Text) | Salary (Num) |  
|----------------|-------------|-------------|--------------|  
| 001            | Alice       | Sales       | 75000        |  
| 002            | Bob         | IT          | 80000        |  
| 003            | Carol       | Sales       | 70000        |

**Characteristics:**

* 🏗️ Fixed structure: Every employee MUST have these exact fields  
* 🔗 Relationships: Can link to DEPARTMENTS table using Dept field  
* 📋 Strict rules: Can't add "Favorite Color" unless you modify the entire table  
* ✅ Perfect for: Banking, inventory, financial records, HR systems

**NoSQL Database \= Co-Working Space**

Document Storage:  
{  
  "id": "001",  
  "name": "Alice",  
  "department": "Sales",  
  "salary": 75000,  
  "skills": \["negotiation", "communication"\],  
  "projects": \[...\]  
}

{  
  "id": "002",  
  "name": "Bob",  
  "department": "IT",  
  "certifications": \["AWS", "Docker"\],  
  "favorite\_language": "Python"  
  // Different fields\! No problem\!  
}

**Characteristics:**

* 🎨 Flexible structure: Each document can have different fields  
* 🚀 Easy evolution: Add new fields anytime without restructuring  
* 📦 Self-contained: Data stored together (less joining needed)  
* ✅ Perfect for: Social media, IoT sensors, content management, real-time analytics

---

### **🎮 Decision Game: Which Database Do You Choose?**

**Scenario 1: You're building a banking app**

* Need to track: account numbers, balances, transactions  
* Must be 100% accurate (no money lost\!)  
* Accounts always have the same fields  
* Need to see transaction history and account relationships

**Your choice?** Think about it...

**Answer: SQL Database\! ✓**

**Why?**

* Banking needs ACID guarantees (we'll explore this next\!)  
* Fixed structure works perfectly (all accounts have same fields)  
* Relationships are crucial (accounts → transactions → customers)  
* Accuracy \> flexibility

---

**Scenario 2: You're building a social media feed**

* Some posts have images, some videos, some just text  
* Some have location tags, some don't  
* New features added frequently (polls, stories, reactions)  
* Millions of posts per day  
* Need ultra-fast reads

**Your choice?** Think about it...

**Answer: NoSQL Database\! ✓**

**Why?**

* Every post can have different fields (flexible\!)  
* Easy to add new features without restructuring  
* Scales horizontally (add more servers easily)  
* Optimized for fast reads at massive scale

---

### **🚨 Common Misconception: "NoSQL Means No Structure\!"**

**You might think:** "NoSQL \= chaos, just dump data randomly\!"

**The truth:** NoSQL still has structure, just flexible structure\!

**Bad NoSQL approach:**

// Chaos\! Hard to work with\!  
{ "user": "alice", "data": "some stuff" }  
{ "person": "bob", "info": 123 }  
{ "x": "y", "random": true }

**Good NoSQL approach:**

// Flexible but consistent\!  
{  
  "user\_id": "alice123",  
  "post\_type": "image",  
  "content": "...",  
  "created\_at": "2025-10-15",  
  "tags": \["travel", "sunset"\]  
}

{  
  "user\_id": "bob456",  
  "post\_type": "video",  
  "content": "...",  
  "created\_at": "2025-10-15",  
  "duration": 45,  
  "thumbnail": "..."  
  // Different fields, but still organized\!  
}

**Mental model:** Think of SQL as a strict form you must fill out completely, and NoSQL as a flexible journal where each entry can be different but still meaningful\!

---

### **📊 Visual Comparison Table**

| Aspect | SQL | NoSQL |
| ----- | ----- | ----- |
| **Structure** | Fixed schema (tables, rows, columns) | Flexible schema (documents, key-value, graph) |
| **Scaling** | Vertical (bigger servers) | Horizontal (more servers) |
| **Transactions** | ACID compliant (strong consistency) | Eventually consistent (usually) |
| **Best For** | Complex queries, relationships, accuracy | Flexibility, speed, massive scale |
| **Learning Curve** | Steeper (need to learn SQL language) | Gentler (often uses JSON-like formats) |
| **When to Use** | Banking, ERP, CRM, inventory | Social media, IoT, real-time analytics, catalogs |

**Real-world analogy:**

* **SQL:** Like a meticulous accountant \- precise, structured, perfect for financial records  
* **NoSQL:** Like a creative scrapbook \- flexible, fast, perfect for diverse content

---

## **🛡️ Part 2: ACID Properties \- The Four Pillars of Database Reliability**

### **🎯 Challenge 2: The Bank Transfer Mystery**

**Scenario:** You transfer $100 from your account to your friend's account. Mid-transfer, the database server crashes\!

**What should happen to prevent disaster?** A. $100 disappears from both accounts (lost money\!) B. $100 removed from your account but never added to friend's (you lose $100\!) C. $100 added to friend's account but not removed from yours (free money\!) D. The entire transfer is cancelled, and both accounts return to original state

**Think carefully...** What protects users from database failures?

### **The Answer: ACID Properties Ensure Option D\!**

ACID is like a safety net for critical operations. Let's break down each letter:

---

### **🅰️ A is for ATOMICITY: All or Nothing**

**Real-world parallel:** Imagine buying a TV online:

**Without Atomicity (Disaster\!):**

![][image1]

Result: You paid, TV is reserved, but system thinks it's still in stock. Chaos\!

**With Atomicity (Protected\!):**

Transaction begins:  
  Step 1: Charge credit card  
  Step 2: Reserve TV  
  Step 3: Update inventory  
    
IF all steps succeed → COMMIT (make it permanent)  
IF any step fails → ROLLBACK (undo everything)

![][image2]

**The bank transfer example:**

BEGIN TRANSACTION;  
  UPDATE accounts SET balance \= balance \- 100 WHERE user \= 'you';  
  UPDATE accounts SET balance \= balance \+ 100 WHERE user \= 'friend';  
COMMIT;

**If crash happens mid-transaction:** Database automatically rolls back BOTH updates. Your $100 stays in your account, friend gets nothing. No money lost or duplicated\!

**Mental model:** Atomicity is like a light switch \- it's either completely ON or completely OFF. There's no half-lit room. A transaction is either fully completed or fully cancelled.

**Key insight:** Atomicity prevents partial operations that would leave data in an inconsistent state\!

---

### **🅱️ C is for CONSISTENCY: Rules Must Be Followed**

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

* Primary keys must be unique  
* Foreign keys must reference existing records  
* Balance cannot go negative  
* Age must be a positive number  
* Email must be unique

**Example with bank accounts:**

\-- Rule: Balance cannot go negative  
UPDATE accounts SET balance \= balance \- 1000   
WHERE user \= 'alice' AND balance \= 500;

\-- Database says NO\! This would make balance \= \-500  
\-- Transaction REJECTED to maintain consistency

**Mental model:** Consistency is like a bouncer at a club checking IDs. If you don't meet the requirements (rules), you're not getting in\! The database is the bouncer ensuring all rules are followed.

**Key insight:** Consistency ensures the database moves from one valid state to another valid state, never breaking its own rules\!

---

### **🅾️ I is for ISOLATION: No Interference**

**The Coffee Shop Problem:**

**Scenario:** You and your friend both try to buy the last croissant at the same time.

**Without Isolation (Disaster\!):**

You:              Check stock → 1 croissant available ✓  
Friend:           Check stock → 1 croissant available ✓  
You:              Buy it\! → Success (stock \= 0\)  
Friend:           Buy it\! → Success (stock \= \-1) 😱

Result: Two people bought one croissant\! Negative inventory\!

**With Isolation (Protected\!):**

You:              BEGIN TRANSACTION  
You:              Check stock → 1 croissant available  
You:              LOCK this croissant  
Friend:           BEGIN TRANSACTION    
Friend:           Check stock → Must WAIT (locked by You)  
You:              Buy it\! COMMIT → stock \= 0  
You:              RELEASE LOCK  
Friend:           Check stock → 0 croissants available  
Friend:           Cannot buy → "Out of stock" error ✓

**Database example \- Concert ticket sales:**

\-- User A and User B try to buy the last ticket simultaneously

\-- User A:  
BEGIN TRANSACTION;  
  SELECT seats\_remaining FROM events WHERE event\_id \= 123;  
  \-- Returns: 1 seat  
  \-- Database LOCKS this row  
  UPDATE events SET seats\_remaining \= 0 WHERE event\_id \= 123;  
COMMIT;

\-- User B (tries at the same time):  
BEGIN TRANSACTION;  
  SELECT seats\_remaining FROM events WHERE event\_id \= 123;  
  \-- Must WAIT for User A's transaction to complete  
  \-- After A commits, B sees: 0 seats  
  \-- B's purchase fails ✓

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

Step 1: Write transaction to LOG FILE (fast, on disk)  
Step 2: Tell user "Success\!"  
Step 3: Update actual database (can happen later)

If crash occurs:  
  \- Check log file  
  \- Replay any completed transactions  
  \- Database recovers to correct state\!

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
     
🅾️ ISOLATION:  
   If Carol also transfers to Bob simultaneously,  
   transactions don't interfere with each other  
     
🅳 DURABILITY:  
   Once confirmed, transfer is permanent  
   even if server crashes immediately after

**Visual representation:**

Alice: $500  →  \[ACID TRANSACTION\]  →  Alice: $400  
Bob:   $200  →  \[ACID TRANSACTION\]  →  Bob:   $300

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

## **🔍 Part 3: Database Indexing \- The Library Card Catalog**

### **🎯 Challenge 3: The Phonebook Problem**

**Scenario:** You need to find "John Smith" in a phonebook with 100,000 names.

**Method A:** Start at page 1, check every single name until you find John Smith

* Time: Could take 50,000 comparisons on average\! 😰

**Method B:** Use alphabetical organization:

* Open to middle (probably around "M")  
* Too far? Go left  
* Not far enough? Go right  
* Repeat until found  
* Time: About 17 comparisons\! ⚡

**Question:** How is Method B so much faster?

### **The Answer: Indexing\!**

**A database without indexes** \= Reading every single row to find what you need

**A database with indexes** \= Jump directly to the data you want

---

### **📚 What Is an Index?**

**Think of it like:**

* 📖 Index at the back of a textbook  
* 🗂️ Card catalog in a library  
* 📇 Contacts list sorted by name on your phone

**Example: Finding a book in a library**

**Without Index:**

Walk through EVERY aisle  
Check EVERY book title  
Found it after searching 50,000 books\! ⏰ 2 hours

**With Index (Card Catalog):**

Look up "Pride and Prejudice" in catalog  
Card says: "Aisle 12, Shelf 5, Position 3"  
Walk directly there ⚡ 2 minutes

---

### **🗄️ How Database Indexes Work**

**Imagine a USERS table without index:**

Finding user with email "bob@example.com":

Row 1: alice@example.com ❌  
Row 2: charlie@example.com ❌  
Row 3: david@example.com ❌  
...  
Row 50,000: bob@example.com ✓ FOUND\!

Time: Scanned 50,000 rows (SLOW\! 😰)

**Same table WITH index on email:**

Email Index (automatically sorted):  
alice@example.com    → Row 1  
bob@example.com      → Row 50,000 ✓   
charlie@example.com  → Row 2  
david@example.com    → Row 3  
...

Database uses index:  
1\. Binary search in index (fast\!)  
2\. Index points to Row 50,000  
3\. Jump directly to that row

Time: Scanned \~17 entries (FAST\! ⚡)

---

### **🎮 Interactive Exercise: When to Add an Index?**

**You have a PRODUCTS table with 1 million products:**

CREATE TABLE products (  
  product\_id INT,  
  name VARCHAR(255),  
  price DECIMAL,  
  category VARCHAR(100),  
  description TEXT,  
  created\_at DATE  
);

**Common queries:**

1. `SELECT * FROM products WHERE product_id = 12345;`  
2. `SELECT * FROM products WHERE category = 'Electronics';`  
3. `SELECT * FROM products WHERE price > 100;`  
4. `SELECT * FROM products WHERE name LIKE '%phone%';`

**Question:** Which columns should you index? Think about it...

---

### **🎯 Indexing Guidelines: When to Index**

**✅ GOOD candidates for indexing:**

* **Primary keys** (usually auto-indexed)  
  * Reason: Constantly used for lookups  
* **Foreign keys** (columns that reference other tables)  
  * Reason: Used in JOIN operations  
* **Columns frequently used in WHERE clauses**  
  * Example: `WHERE email = '...'`  
  * Reason: Speeds up filtering  
* **Columns used for sorting (ORDER BY)**  
  * Example: `ORDER BY created_at DESC`  
  * Reason: Avoids sorting entire table

**❌ POOR candidates for indexing:**

* **Rarely queried columns**  
  * Example: `description` text field  
  * Reason: Index overhead not worth it  
* **Columns with many duplicates**  
  * Example: `gender` (only M/F values)  
  * Reason: Index doesn't narrow down much  
* **Very small tables**  
  * Example: Table with 50 rows  
  * Reason: Full scan is already fast

**💰 The Cost of Indexes:**

**Benefits:**

* ⚡ Dramatically faster SELECT queries  
* 🎯 Efficient searching and filtering  
* 📊 Faster sorting and joining

**Costs:**

* 💾 Extra storage space (indexes take up room)  
* ⏱️ Slower INSERT/UPDATE/DELETE (must update indexes too)  
* 🔧 Maintenance overhead

**Mental model:** Think of indexes like shortcuts in a video game. They get you places faster, but they take time to build and maintain\!

---

### **📊 Index Types: Quick Overview**

**Primary Index (Clustered):**

* Determines physical order of data on disk  
* Like books sorted on library shelves  
* One per table

**Secondary Index (Non-Clustered):**

* Separate structure pointing to data  
* Like card catalog pointing to books  
* Multiple per table

**Unique Index:**

* Ensures no duplicate values  
* Like social security numbers

**Composite Index:**

* Index on multiple columns together  
* Like phonebook sorted by (LastName, FirstName)

**Example:**

\-- Index on single column  
CREATE INDEX idx\_email ON users(email);

\-- Composite index  
CREATE INDEX idx\_name ON users(last\_name, first\_name);

\-- Unique index  
CREATE UNIQUE INDEX idx\_username ON users(username);

---

### **🔥 Real-World Performance Example**

**Query without index:**

SELECT \* FROM orders   
WHERE customer\_id \= 12345;

\-- Execution time: 8 seconds (full table scan)  
\-- Rows examined: 10,000,000

**Same query WITH index:**

CREATE INDEX idx\_customer ON orders(customer\_id);

SELECT \* FROM orders   
WHERE customer\_id \= 12345;

\-- Execution time: 0.003 seconds\! ⚡  
\-- Rows examined: 50 (via index)

**Speed improvement: 2,667x faster\!**

**Visual analogy:**

Without Index:          With Index:  
                          
📄📄📄📄📄📄📄           🗂️ Index  
📄📄📄📄📄📄📄           ↓  
📄📄📄📄📄📄📄           📄 (Jump directly here\!)  
📄📄✓📄📄📄📄  
📄📄📄📄📄📄📄  
(Must check all)        (Check index, then 1 row)

---

## **🔑 Part 4: Primary Key vs Foreign Key \- The Relationship Builders**

### **🎯 Challenge 4: The Student ID Mystery**

**Scenario:** You're organizing a school database. How do you ensure:

* Each student is unique?  
* You can find any student quickly?  
* You can link students to their classes?  
* You can link students to their grades?

**Think about it:** What system would you use to identify and connect students?

### **The Answer: Keys\!**

---

### **🏆 Primary Key: The Unique Identifier**

**Real-world parallel:** Your Social Security Number, Passport Number, or Student ID

**Definition:** A column (or set of columns) that uniquely identifies each row in a table.

**Rules for Primary Keys:**

1. ✅ Must be UNIQUE (no duplicates)  
2. ✅ Cannot be NULL (must have a value)  
3. ✅ Should never change  
4. ✅ One per table

**Example: STUDENTS table**

CREATE TABLE students (  
  student\_id INT PRIMARY KEY,  \-- This is the primary key\!  
  first\_name VARCHAR(50),  
  last\_name VARCHAR(50),  
  email VARCHAR(100),  
  birth\_date DATE  
);

\-- Sample data:  
student\_id | first\_name | last\_name | email  
\-----------+------------+-----------+-------------------  
1001       | Alice      | Johnson   | alice@school.edu  
1002       | Bob        | Smith     | bob@school.edu  
1003       | Carol      | Davis     | carol@school.edu

**Why is student\_id the primary key?**

* 🆔 Unique: No two students have same ID  
* 🚫 Not null: Every student must have an ID  
* 🔒 Stable: ID never changes even if name or email changes  
* ⚡ Fast: Database automatically indexes primary keys

**Bad primary key choices:**

* ❌ **Name:** Not unique (many "John Smith"s exist)  
* ❌ **Email:** Could change when student graduates  
* ❌ **Phone number:** Can change  
* ❌ **Age:** Definitely changes\!

---

### **🔗 Foreign Key: The Relationship Connector**

**Real-world parallel:** A reference number on a package pointing to your address

**Definition:** A column that references the primary key of another table, creating a relationship.

**Example: ENROLLMENTS table**

CREATE TABLE enrollments (  
  enrollment\_id INT PRIMARY KEY,  
  student\_id INT,                    \-- Foreign key\!  
  course\_id INT,                     \-- Foreign key\!  
  enrollment\_date DATE,  
  grade VARCHAR(2),  
    
  FOREIGN KEY (student\_id) REFERENCES students(student\_id),  
  FOREIGN KEY (course\_id) REFERENCES courses(course\_id)  
);

**Visual relationship:**

STUDENTS Table:                ENROLLMENTS Table:  
student\_id | name              enrollment\_id | student\_id (FK) | course\_id (FK)  
\-----------+-------            \--------------+-----------------+---------------  
1001       | Alice             1             | 1001            | CS101  
1002       | Bob               2             | 1001            | MATH200  
1003       | Carol             3             | 1002            | CS101  
                                4             | 1003            | ART150

COURSES Table:  
course\_id | course\_name  
\----------+-------------  
CS101     | Intro to Programming  
MATH200   | Calculus II  
ART150    | Digital Art

**The relationship:**

Alice (student\_id: 1001\) is enrolled in:  
  → CS101 (via enrollment\_id 1\)  
  → MATH200 (via enrollment\_id 2\)

CS101 has these students:  
  → Alice (student\_id: 1001\)  
  → Bob (student\_id: 1002\)

---

### **🎮 Interactive Comparison Game**

**Match the key type to its role:**

| Scenario | Primary Key or Foreign Key? |
| ----- | ----- |
| Social Security Number in PEOPLE table | ? |
| Customer ID in ORDERS table referencing CUSTOMERS | ? |
| Order ID in ORDERS table | ? |
| Product ID in ORDER\_ITEMS table referencing PRODUCTS | ? |

**Think about each one...**

**Answers:**

1. **Primary Key** \- Uniquely identifies each person  
2. **Foreign Key** \- Links order to a customer  
3. **Primary Key** \- Uniquely identifies each order  
4. **Foreign Key** \- Links order item to a product

---

### **🛡️ Foreign Key Constraints: Protecting Data Integrity**

**What Foreign Keys Prevent:**

**Scenario 1: Orphaned Records**

\-- Try to insert enrollment for non-existent student:  
INSERT INTO enrollments (student\_id, course\_id)   
VALUES (9999, 'CS101');  \-- Student 9999 doesn't exist\!

\-- Result: ❌ ERROR: Foreign key constraint violated\!  
\-- Database says: "Can't enroll a student that doesn't exist\!"

**Scenario 2: Broken References**

\-- Try to delete a student who has enrollments:  
DELETE FROM students WHERE student\_id \= 1001;

\-- Result: ❌ ERROR: Foreign key constraint violated\!  
\-- Database says: "Can't delete student with existing enrollments\!"  
\-- (You'd have orphaned enrollments pointing to non-existent student)

**Options for handling deletions:**

\-- CASCADE: Delete enrollments when student is deleted  
FOREIGN KEY (student\_id) REFERENCES students(student\_id)  
  ON DELETE CASCADE;

\-- SET NULL: Set student\_id to NULL in enrollments  
FOREIGN KEY (student\_id) REFERENCES students(student\_id)  
  ON DELETE SET NULL;

\-- RESTRICT: Prevent deletion (default)  
FOREIGN KEY (student\_id) REFERENCES students(student\_id)  
  ON DELETE RESTRICT;

**Mental model:** Foreign keys are like safety ropes in mountain climbing. They ensure you can't fall off a cliff (create invalid references) and keep everything connected safely\!

---

### **📊 Complete Example: E-Commerce Database**

\-- CUSTOMERS table  
CREATE TABLE customers (  
  customer\_id INT PRIMARY KEY,      \-- Primary Key  
  name VARCHAR(100),  
  email VARCHAR(100),  
  address VARCHAR(255)  
);

\-- ORDERS table  
CREATE TABLE orders (  
  order\_id INT PRIMARY KEY,         \-- Primary Key  
  customer\_id INT,                  \-- Foreign Key → customers  
  order\_date DATE,  
  total\_amount DECIMAL,  
    
  FOREIGN KEY (customer\_id) REFERENCES customers(customer\_id)  
);

\-- ORDER\_ITEMS table  
CREATE TABLE order\_items (  
  item\_id INT PRIMARY KEY,          \-- Primary Key  
  order\_id INT,                     \-- Foreign Key → orders  
  product\_id INT,                   \-- Foreign Key → products  
  quantity INT,  
  price DECIMAL,  
    
  FOREIGN KEY (order\_id) REFERENCES orders(order\_id),  
  FOREIGN KEY (product\_id) REFERENCES products(product\_id)  
);

\-- PRODUCTS table  
CREATE TABLE products (  
  product\_id INT PRIMARY KEY,       \-- Primary Key  
  name VARCHAR(200),  
  description TEXT,  
  price DECIMAL,  
  stock INT  
);

**The relationships:**

CUSTOMERS (1) ──→ (Many) ORDERS  
   One customer can have many orders

ORDERS (1) ──→ (Many) ORDER\_ITEMS  
   One order can have many items

PRODUCTS (1) ──→ (Many) ORDER\_ITEMS  
   One product can appear in many orders

---

### **💡 Quick Comparison Summary**

| Aspect | Primary Key | Foreign Key |
| ----- | ----- | ----- |
| **Purpose** | Uniquely identify rows | Link to other tables |
| **Uniqueness** | Must be unique | Can have duplicates |
| **Null Values** | Cannot be NULL | Can be NULL (optional relationship) |
| **Quantity** | One per table | Multiple per table |
| **Auto-Indexed** | Yes, always | Should be indexed |
| **Example** | Student ID | Student ID in Enrollments table |

**Mental model:**

* **Primary Key** \= Your name on your ID badge  
* **Foreign Key** \= Your name on your visitor's guest list (referencing you)

---

## **✏️ Part 5: CRUD Operations \- The Four Essential Database Actions**

### **🎯 Challenge 5: The Todo App**

**Scenario:** You're building a simple todo list app. What actions do users need?

Think about it:

* How do you add a new task?  
* How do you see your tasks?  
* How do you mark a task as complete?  
* How do you remove a task?

**Answer: CRUD Operations\!**

---

### **🔤 What is CRUD?**

**CRUD \= The four basic operations you can do with data**

* **C**reate \- Add new records  
* **R**ead \- View/retrieve records  
* **U**pdate \- Modify existing records  
* **D**elete \- Remove records

**Real-world parallel:** Think of a contact list on your phone:

* 📝 **Create:** Add new contact  
* 👁️ **Read:** View contact info  
* ✏️ **Update:** Change phone number  
* 🗑️ **Delete:** Remove old contact

---

### **📝 CREATE: Adding New Data**

**SQL Command:** `INSERT`

**Example: Adding a new user**

INSERT INTO users (username, email, age)   
VALUES ('alice123', 'alice@email.com', 25);

**What happens:**

Before:  
users table:  
id | username | email           | age  
\---+----------+-----------------+----  
1  | bob456   | bob@email.com   | 30

After INSERT:  
users table:  
id | username  | email            | age  
\---+-----------+------------------+----  
1  | bob456    | bob@email.com    | 30  
2  | alice123  | alice@email.com  | 25  ← NEW\!

**Multiple records at once:**

INSERT INTO products (name, price, category)  
VALUES   
  ('Laptop', 999.99, 'Electronics'),  
  ('Mouse', 19.99, 'Electronics'),  
  ('Notebook', 3.99, 'Stationery');

**Real-world scenarios:**

* 🛒 Adding items to shopping cart  
* 👤 User registration  
* 📧 Posting a comment  
* 📅 Creating a calendar event

**Mental model:** INSERT is like writing a new entry in a logbook \- you're adding permanent information\!

---

### **👁️ READ: Retrieving Data**

**SQL Command:** `SELECT`

**Example: View all users**

SELECT \* FROM users;

**Result:**

id | username  | email            | age  
\---+-----------+------------------+----  
1  | bob456    | bob@email.com    | 30  
2  | alice123  | alice@email.com  | 25  
3  | carol789  | carol@email.com  | 28

**Select specific columns:**

SELECT username, email FROM users;

**Result:**

username  | email  
\----------+------------------  
bob456    | bob@email.com  
alice123  | alice@email.com  
carol789  | carol@email.com

**Filtering with WHERE:**

SELECT \* FROM users WHERE age \> 26;

**Result:**

id | username | email           | age  
\---+----------+-----------------+----  
1  | bob456   | bob@email.com   | 30  
3  | carol789 | carol@email.com | 28

**Sorting results:**

SELECT \* FROM users ORDER BY age DESC;

**Advanced: Joining tables**

SELECT users.username, orders.order\_date, orders.total  
FROM users  
JOIN orders ON users.id \= orders.user\_id  
WHERE users.username \= 'alice123';

**Real-world scenarios:**

* 📱 Loading your profile  
* 📰 Viewing news feed  
* 🔍 Searching for products  
* 📊 Generating reports

**Mental model:** SELECT is like looking through a filing cabinet with superpowers \- you can view, filter, and organize information instantly\!

---

### **✏️ UPDATE: Modifying Existing Data**

**SQL Command:** `UPDATE`

**Example: Change user's email**

UPDATE users   
SET email \= 'alice.new@email.com'   
WHERE username \= 'alice123';

**What happens:**

Before:  
id | username  | email            | age  
\---+-----------+------------------+----  
2  | alice123  | alice@email.com  | 25

After UPDATE:  
id | username  | email                | age  
\---+-----------+----------------------+----  
2  | alice123  | alice.new@email.com  | 25  ← CHANGED\!

**Update multiple columns:**

UPDATE users   
SET email \= 'bob.updated@email.com',  
    age \= 31  
WHERE username \= 'bob456';

**Update multiple rows:**

UPDATE products   
SET price \= price \* 0.9  \-- 10% discount  
WHERE category \= 'Electronics';

**⚠️ WARNING: Always use WHERE clause\!**

\-- DANGEROUS\! Updates ALL rows:  
UPDATE users SET age \= 50;  \-- Everyone is now 50\!

\-- SAFE: Updates specific user:  
UPDATE users SET age \= 50 WHERE username \= 'bob456';

**Real-world scenarios:**

* 📝 Editing your profile  
* ✅ Marking tasks as complete  
* 📊 Updating inventory quantities  
* 🔄 Changing order status

**Mental model:** UPDATE is like using correction fluid and writing over old information \- the record stays, but the content changes\!

---

### **🗑️ DELETE: Removing Data**

**SQL Command:** `DELETE`

**Example: Remove a user**

DELETE FROM users WHERE username \= 'alice123';

**What happens:**

Before:  
id | username  | email            | age  
\---+-----------+------------------+----  
1  | bob456    | bob@email.com    | 30  
2  | alice123  | alice@email.com  | 25  ← TO BE DELETED  
3  | carol789  | carol@email.com  | 28

After DELETE:  
id | username | email           | age  
\---+----------+-----------------+----  
1  | bob456   | bob@email.com   | 30  
3  | carol789 | carol@email.com | 28  
                                    ← alice123 is GONE\!

**Delete multiple rows:**

DELETE FROM orders   
WHERE order\_date \< '2020-01-01';  \-- Remove old orders

**⚠️ WARNING: Always use WHERE clause\!**

\-- CATASTROPHIC\! Deletes ALL data:  
DELETE FROM users;  \-- Table now empty\! 😱

\-- SAFE: Deletes specific user:  
DELETE FROM users WHERE id \= 2;

**Soft Delete vs Hard Delete:**

**Hard Delete (permanent):**

DELETE FROM users WHERE id \= 2;  
\-- User is completely removed from database

**Soft Delete (mark as deleted but keep data):**

\-- Add deleted\_at column to table  
UPDATE users   
SET deleted\_at \= NOW()   
WHERE id \= 2;

\-- Later, when querying:  
SELECT \* FROM users WHERE deleted\_at IS NULL;  \-- Only active users

**Real-world scenarios:**

* 🗑️ Removing items from cart  
* 🚫 Deleting your account  
* 🧹 Cleaning up old data  
* ❌ Canceling an order

**Mental model:** DELETE is like shredding a document \- once it's gone, it's gone (unless you have backups\!)

---

### **🎮 CRUD in Action: Complete Example**

**Scenario: Managing a blog**

**1️⃣ CREATE: Write new post**

INSERT INTO posts (title, content, author\_id, created\_at)  
VALUES ('My First Post', 'Hello World\!', 1, NOW());

**2️⃣ READ: View all posts**

SELECT posts.title, posts.content, users.username  
FROM posts  
JOIN users ON posts.author\_id \= users.id  
ORDER BY posts.created\_at DESC;

**3️⃣ UPDATE: Edit post**

UPDATE posts   
SET content \= 'Updated content here\!',  
    updated\_at \= NOW()  
WHERE id \= 1;

**4️⃣ DELETE: Remove post**

DELETE FROM posts WHERE id \= 1;

---

### **🌐 CRUD in REST APIs**

**CRUD maps to HTTP methods:**

| CRUD Operation | HTTP Method | Example URL | Action |
| ----- | ----- | ----- | ----- |
| **Create** | POST | `POST /api/users` | Create new user |
| **Read** | GET | `GET /api/users/123` | Get user \#123 |
| **Update** | PUT/PATCH | `PUT /api/users/123` | Update user \#123 |
| **Delete** | DELETE | `DELETE /api/users/123` | Delete user \#123 |

**Example API calls:**

// CREATE  
fetch('/api/posts', {  
  method: 'POST',  
  body: JSON.stringify({ title: 'New Post', content: '...' })  
});

// READ  
fetch('/api/posts/123');

// UPDATE  
fetch('/api/posts/123', {  
  method: 'PUT',  
  body: JSON.stringify({ title: 'Updated Title' })  
});

// DELETE  
fetch('/api/posts/123', { method: 'DELETE' });

---

### **💡 CRUD Best Practices**

**✅ DO:**

* Always use WHERE clauses with UPDATE and DELETE  
* Validate input data before creating  
* Consider soft deletes for important data  
* Use transactions for related operations  
* Add indexes on frequently queried columns

**❌ DON'T:**

* Run UPDATE or DELETE without WHERE (unless intentional)  
* Store sensitive data without encryption  
* Forget to back up before bulk operations  
* Expose database directly to users  
* Trust user input (always validate\!)

---

## **🎯 Final Synthesis: Putting It All Together**

### **The Complete Picture: Building a Library System**

Let's apply everything we've learned\!

**1️⃣ Database Choice:**

* Need structured data (books, members, loans)  
* Need relationships (members ←→ loans ←→ books)  
* Need accuracy (can't lose loan records\!)  
* **Choice: SQL Database with ACID properties** ✓

**2️⃣ Table Design with Keys:**

CREATE TABLE books (  
  book\_id INT PRIMARY KEY,           \-- Primary Key  
  isbn VARCHAR(13),  
  title VARCHAR(200),  
  author VARCHAR(100),  
  published\_date DATE  
);

CREATE TABLE members (  
  member\_id INT PRIMARY KEY,         \-- Primary Key  
  name VARCHAR(100),  
  email VARCHAR(100),  
  join\_date DATE  
);

CREATE TABLE loans (  
  loan\_id INT PRIMARY KEY,           \-- Primary Key  
  book\_id INT,                       \-- Foreign Key  
  member\_id INT,                     \-- Foreign Key  
  loan\_date DATE,  
  due\_date DATE,  
  returned\_date DATE,  
    
  FOREIGN KEY (book\_id) REFERENCES books(book\_id),  
  FOREIGN KEY (member\_id) REFERENCES members(member\_id)  
);

**3️⃣ Add Indexes for Performance:**

CREATE INDEX idx\_book\_isbn ON books(isbn);  
CREATE INDEX idx\_member\_email ON members(email);  
CREATE INDEX idx\_loan\_dates ON loans(loan\_date, due\_date);

**4️⃣ CRUD Operations:**

**Create: New member joins**

INSERT INTO members (member\_id, name, email, join\_date)  
VALUES (1, 'Alice Johnson', 'alice@email.com', '2025-10-15');

**Read: Find overdue books**

SELECT books.title, members.name, loans.due\_date  
FROM loans  
JOIN books ON loans.book\_id \= books.book\_id  
JOIN members ON loans.member\_id \= members.member\_id  
WHERE loans.due\_date \< CURRENT\_DATE   
  AND loans.returned\_date IS NULL;

**Update: Return a book**

UPDATE loans   
SET returned\_date \= CURRENT\_DATE  
WHERE loan\_id \= 123;

**Delete: Remove old loan records**

DELETE FROM loans   
WHERE returned\_date \< '2020-01-01';

**5️⃣ ACID Transaction: Loan a book**

BEGIN TRANSACTION;

  \-- Check if book is available  
  SELECT \* FROM books WHERE book\_id \= 42 AND available \= true;  
    
  \-- Create loan record  
  INSERT INTO loans (book\_id, member\_id, loan\_date, due\_date)  
  VALUES (42, 1, CURRENT\_DATE, CURRENT\_DATE \+ 14);  
    
  \-- Mark book as unavailable  
  UPDATE books SET available \= false WHERE book\_id \= 42;

**COMMIT;  \-- All or nothing\!**

---

## **🏆 Quick Recap: Test Your Understanding**

**Without looking back, can you explain:**

1. **When would you choose SQL vs NoSQL?**  
2. **What does each letter in ACID stand for and why does it matter?**  
3. **Why would adding an index make a query faster?**  
4. **What's the difference between a Primary Key and a Foreign Key?**  
5. **What do the letters in CRUD stand for?**

**Mental check:** If you can answer these clearly, you've mastered database fundamentals\! 🎓

---

## **🚀 Your Next Learning Adventure**

**Now that you understand database fundamentals, explore:**

**Immediate Next Steps:**

* Database Normalization (1NF, 2NF, 3NF) \- Organizing data efficiently  
* SQL JOINs (INNER, LEFT, RIGHT, FULL) \- Combining data from multiple tables  
* Subqueries and CTEs \- Advanced query techniques  
* Database Transactions in depth \- Concurrent operations

**Advanced Topics:**

* Query Optimization \- Making your queries lightning fast  
* Stored Procedures & Triggers \- Automating database logic  
* Database Replication \- Scaling reads across multiple servers  
* Sharding & Partitioning \- Scaling writes and storage

**Practical Applications:**

* Design a complete e-commerce database  
* Build a social media backend  
* Optimize a slow-running application  
* Implement caching strategies

**Remember:** Every major application uses databases. Master these fundamentals, and you'll understand how the digital world stores and manages information\! 💪

#  **Database: Transactions, Operations, Pooling & Backups**

## **🔄 Part 1: Database Transactions \- The All-or-Nothing Guarantee**

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

---

### **💰 Real-World Example: The Bank Transfer**

**Scenario:** Transfer $500 from Alice to Bob

**Without Transaction (DISASTER ZONE):**

\-- Step 1: Deduct from Alice  
UPDATE accounts SET balance \= balance \- 500 WHERE name \= 'Alice';  
\-- ✓ Success\! Alice now has $500 less

\-- Step 2: Add to Bob  
\-- 💥 DATABASE CRASHES HERE\! 

\-- Result: Alice lost $500, Bob never received it\! Money vanished\! 😱

**With Transaction (PROTECTED):**

BEGIN TRANSACTION;

\-- Step 1: Deduct from Alice  
UPDATE accounts SET balance \= balance \- 500 WHERE name \= 'Alice';  
\-- Balance: Alice $1000 → $500 (temporary, not permanent yet\!)

\-- Step 2: Add to Bob    
UPDATE accounts SET balance \= balance \+ 500 WHERE name \= 'Bob';  
\-- Balance: Bob $500 → $1000 (temporary, not permanent yet\!)

\-- Check if both succeeded  
IF both\_steps\_successful THEN  
  COMMIT;  \-- Make both changes permanent\!  
  \-- Result: Alice $500, Bob $1000 ✓  
ELSE  
  ROLLBACK;  \-- Undo both changes\!  
  \-- Result: Back to original state: Alice $1000, Bob $500  
END IF;

**Visualization:**

TIME FLOW →

Database State Before Transaction:  
┌──────────┬─────────┐  
│ Account  │ Balance │  
├──────────┼─────────┤  
│ Alice    │ $1000   │  
│ Bob      │ $500    │  
└──────────┴─────────┘

BEGIN TRANSACTION (Work in Progress)  
┌──────────┬─────────┬────────────┐  
│ Account  │ Balance │ Temp Value │  
├──────────┼─────────┼────────────┤  
│ Alice    │ $1000   │ $500 📝    │  
│ Bob      │ $500    │ $1000 📝   │  
└──────────┴─────────┴────────────┘  
         (not visible to others)

COMMIT ✓ (Success\!)  
┌──────────┬─────────┐  
│ Account  │ Balance │  
├──────────┼─────────┤  
│ Alice    │ $500    │ ← Permanent\!  
│ Bob      │ $1000   │ ← Permanent\!  
└──────────┴─────────┘

Or ROLLBACK ✗ (Failure\!)  
┌──────────┬─────────┐  
│ Account  │ Balance │  
├──────────┼─────────┤  
│ Alice    │ $1000   │ ← Restored\!  
│ Bob      │ $500    │ ← Restored\!  
└──────────┴─────────┘

---

### **🔍 Investigation: Transaction States**

**A transaction goes through distinct states like a package in shipping:**

📦 PACKAGE ANALOGY:

BEGIN TRANSACTION  
    ↓  
\[ACTIVE\] 📝  
"Package being prepared"  
\- Operations in progress  
\- Changes not permanent yet  
    ↓  
Decision Point ⚡  
    ↓                    ↓  
\[COMMIT\] ✅          \[ROLLBACK\] ❌  
"Shipped\!"          "Cancelled\!"  
    ↓                    ↓  
\[COMMITTED\] 🎉      \[ABORTED\] 🗑️  
"Delivered\!"        "Returned to sender"

**In SQL terms:**

\-- State 1: ACTIVE (Transaction in progress)  
BEGIN TRANSACTION;  
UPDATE accounts SET balance \= balance \- 100 WHERE id \= 1;  
UPDATE accounts SET balance \= balance \+ 100 WHERE id \= 2;

\-- State 2a: COMMITTED (Success path)  
COMMIT;    
\-- All changes now permanent and visible to everyone\!

\-- OR State 2b: ABORTED (Failure path)  
ROLLBACK;    
\-- All changes discarded, database unchanged\!

---

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
BEGIN TRANSACTION;  
  INSERT INTO users (name, email) VALUES ('Alice', 'alice@email.com');  
  INSERT INTO preferences (user\_id) VALUES (LAST\_INSERT\_ID());  
COMMIT;

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

BEGIN TRANSACTION;  
  \-- Reduce inventory  
  UPDATE products SET stock \= stock \- 1 WHERE id \= 123;  
    
  \-- Create order  
  INSERT INTO orders (customer\_id, total) VALUES (1, 99.99);  
    
  \-- Update points  
  UPDATE customers SET points \= points \+ 10 WHERE id \= 1;  
    
  \-- Check if inventory went negative  
  IF (stock \< 0\) THEN  
    ROLLBACK;  \-- Can't sell what we don't have\!  
  ELSE  
    COMMIT;  
  END IF;  
END TRANSACTION;

\-- Payment processing happens separately  
\-- (External payment gateway, different rules)

**Key insight:** Transactions are for database consistency, not for orchestrating external systems\!

---

### **🚨 Common Transaction Pitfalls**

**❌ Mistake 1: Transaction Too Long**

**Bad:**

BEGIN TRANSACTION;  
  UPDATE accounts SET balance \= balance \- 100 WHERE id \= 1;  
  \-- User thinks for 10 minutes... ⏰  
  \-- Meanwhile, account is LOCKED\! 🔒  
  \-- Other users can't access this account\!  
COMMIT;

**Good:**

\-- Get data first (outside transaction)  
SELECT balance FROM accounts WHERE id \= 1;  
\-- User thinks... no locks held

\-- Only use transaction for the actual update  
BEGIN TRANSACTION;  
  UPDATE accounts SET balance \= balance \- 100 WHERE id \= 1;  
COMMIT;  \-- Done in milliseconds\! ⚡

**Mental model:** Transactions are like holding your breath underwater \- keep them SHORT\!

---

**❌ Mistake 2: Forgetting to COMMIT or ROLLBACK**

**Bad:**

BEGIN TRANSACTION;  
  UPDATE products SET price \= 99.99 WHERE id \= 1;  
  \-- Oops\! Forgot to COMMIT or ROLLBACK  
  \-- Transaction stays ACTIVE forever  
  \-- Locks held indefinitely 🔒😱

**Good:**

BEGIN TRANSACTION;  
  UPDATE products SET price \= 99.99 WHERE id \= 1;  
COMMIT;  \-- Always end your transactions\!

**Or use try-catch:**

try {  
  await db.query('BEGIN TRANSACTION');  
  await db.query('UPDATE products SET price \= 99.99 WHERE id \= 1');  
  await db.query('COMMIT');  
} catch (error) {  
  await db.query('ROLLBACK');  // Clean up on error\!  
  throw error;  
}

---

**❌ Mistake 3: Nested Transactions (Not What You Think\!)**

**Confusing:**

BEGIN TRANSACTION;  \-- Transaction 1  
  UPDATE table1 SET value \= 1;  
    
  BEGIN TRANSACTION;  \-- This doesn't create Transaction 2\!  
    UPDATE table2 SET value \= 2;  
  COMMIT;  \-- This doesn't commit anything yet\!  
    
COMMIT;  \-- This commits EVERYTHING (both updates)

**Most databases don't support true nested transactions\!** Instead, they flatten to one transaction.

**Better approach \- Savepoints:**

BEGIN TRANSACTION;  
  UPDATE table1 SET value \= 1;  
    
  SAVEPOINT my\_savepoint;  \-- Bookmark this point  
    UPDATE table2 SET value \= 2;  
    \-- Oops, this failed\!  
  ROLLBACK TO my\_savepoint;  \-- Undo only UPDATE table2  
    
  UPDATE table3 SET value \= 3;  \-- This still happens  
COMMIT;  \-- Commits UPDATE table1 and UPDATE table3

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

\-- Transaction A (You)  
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;  
  SELECT balance FROM accounts WHERE id \= 1;  
  \-- Shows: $1000  
    
  \-- Wait 5 seconds...  
    
  SELECT balance FROM accounts WHERE id \= 1;  
  \-- Might show: $500 (if Transaction B committed)  
  \-- This is Non-Repeatable Read\!  
COMMIT;

\-- Transaction B (Someone else, simultaneous)  
BEGIN TRANSACTION;  
  UPDATE accounts SET balance \= 500 WHERE id \= 1;  
COMMIT;

**With Repeatable Read:**

\-- Transaction A  
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;  
  SELECT balance FROM accounts WHERE id \= 1;  
  \-- Shows: $1000  
    
  \-- Wait 5 seconds...  
    
  SELECT balance FROM accounts WHERE id \= 1;  
  \-- Still shows: $1000 (consistent snapshot\!)  
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

---

## **📖 Part 2: Read vs Write Operations \- Understanding the Flow**

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

SELECT \* FROM users WHERE username \= 'alice';

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

**Simple Read:**

SELECT email FROM users WHERE id \= 123;  
\-- Fast\! Direct lookup

**Aggregate Read (more expensive):**

SELECT COUNT(\*) FROM orders WHERE status \= 'pending';  
\-- Slower\! Must scan many rows

**Join Read (even more expensive):**

SELECT users.name, orders.total  
FROM users  
JOIN orders ON users.id \= orders.user\_id;  
\-- Slowest\! Must combine data from multiple tables

**Read characteristics:**

* ✅ Fast and efficient  
* ✅ Can happen in parallel  
* ✅ Don't block other reads  
* ✅ Cheap to scale (add read replicas)  
* ❌ Can block writes (in some isolation levels)

---

### **✏️ Understanding Write Operations**

**What happens during a WRITE:**

UPDATE users SET email \= 'new@email.com' WHERE id \= 123;

**Behind the scenes (so much more complex\!):**

1\. BEGIN TRANSACTION 🔄  
2\. Acquire LOCK on the row 🔒  
3\. Read current data 📖  
4\. Validate constraints (unique, foreign keys, etc.) ✓  
5\. Write to transaction log (WAL) 📝  
6\. Update indexes 🗂️  
7\. Write actual data change 💾  
8\. Release locks 🔓  
9\. COMMIT transaction ✅

Blocks concurrent writes to same row  
Requires validation  
Multiple disk writes  
Slower than reads

**Mental model:** Writing is like editing a shared Google Doc \- only one person can edit a specific paragraph at a time, changes must be saved, version history updated, etc.

**Types of Writes:**

**INSERT (simplest write):**

INSERT INTO users (name, email) VALUES ('Bob', 'bob@email.com');

\-- Must:  
\- Generate new ID  
\- Check constraints (email unique?)  
\- Update indexes  
\- Write to log

**UPDATE (complex write):**

UPDATE products SET price \= 99.99 WHERE category \= 'Electronics';

\-- Must:  
\- Find all matching rows  
\- Lock them  
\- Update each one  
\- Update all relevant indexes  
\- Maintain consistency

**DELETE (most complex):**

DELETE FROM users WHERE id \= 123;

\-- Must:  
\- Check foreign key constraints (does anything reference this?)  
\- Lock the row  
\- Remove from indexes  
\- Mark as deleted  
\- Handle cascade deletes

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

Reads:  ████████████████████ 95%  
Writes: █ 5%

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

SELECT \* FROM users WHERE id \= 12345;

Time: 0.5-2ms ⚡  
Disk I/O: 1 read  
Locks: None (Read Committed)  
Concurrent: Unlimited simultaneous reads

**Write Operation:**

UPDATE users SET last\_name \= 'Smith' WHERE id \= 12345;

Time: 5-50ms ⏱️  
Disk I/O: 3-5 writes (data, log, indexes)  
Locks: Row-level lock 🔒  
Concurrent: Blocks other writes to same row  
Validation: Check constraints, update indexes

**Visual comparison:**

READ:  ⚡ ────✓ (Fast\! One hop)

WRITE: 🔄 ──→ 🔒 ──→ ✓ ──→ 📝 ──→ 💾 ──→ 🔓 ──→ ✅  
       (Start) (Lock) (Validate) (Log) (Write) (Unlock) (Done)  
         
Write is 10-25x slower than read\!

---

### **🚀 Optimization Strategies**

**For Read-Heavy Systems:**

**1\. Caching**

First request:  
User → Database → 50ms ⏱️

With Cache:  
User → Cache → 1ms ⚡  
(50x faster\!)

Only hit database on cache miss

**2\. Read Replicas**

Primary Database (handles writes)  
    ↓ (replicates to)  
Read Replica 1  Read Replica 2  Read Replica 3  
    ↓               ↓               ↓  
Users          Users           Users

Distribute read load across replicas\!

**3\. Indexing**

\-- Without index: Scan 1,000,000 rows ⏱️  
SELECT \* FROM users WHERE email \= 'alice@email.com';

\-- With index: Jump to specific row ⚡  
CREATE INDEX idx\_email ON users(email);

---

**For Write-Heavy Systems:**

**1\. Batch Writes**

// Bad: 1000 individual writes  
for (let i \= 0; i \< 1000; i++) {  
  await db.query('INSERT INTO logs (message) VALUES (?)', \[msg\[i\]\]);  
}  
// Time: 1000 × 5ms \= 5000ms 😰

// Good: 1 batched write  
await db.query(  
  'INSERT INTO logs (message) VALUES (?), (?), ... (all 1000)',  
  messages  
);  
// Time: 50ms ⚡ (100x faster\!)

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

---

## **🏊 Part 3: Connection Pooling \- The Efficiency Multiplier**

### **🎯 Challenge 3: The Restaurant Seating Problem**

**Scenario:** You own a restaurant with 10 tables:

**Method A: New Table Setup Each Time**

* Customer arrives  
* Staff sets up a new table: tablecloth, silverware, plates, glasses  
* Customer eats  
* Staff completely breaks down table  
* Next customer arrives  
* Repeat setup process...

**Method B: Keep Tables Ready**

* Set up 10 tables in advance  
* Customer arrives → sit immediately  
* Customer leaves → quick cleanup  
* Next customer → sit immediately  
* Tables stay ready all day

**Question:** Which method serves more customers per hour?

### **The Answer: Connection Pooling \= Method B\!**

**Definition:** Connection pooling maintains a collection of ready-to-use database connections instead of creating new ones for each request.

---

### **🔌 The Connection Problem**

**Without Connection Pooling (Wasteful\!):**

Web Server receives request:  
1\. Create new database connection  
   \- TCP handshake (3 round trips)  
   \- Authentication (username/password)  
   \- SSL/TLS setup (if encrypted)  
   \- Initialize session variables  
   \- Time: 50-100ms 😰

2\. Execute query  
   \- Time: 5ms ⚡

3\. Close connection  
   \- Cleanup resources  
   \- Time: 10ms

Total: 65-115ms (90% wasted on setup/teardown\!)

For 100 requests: Create 100 connections\! 💥

**Visual representation:**

Request 1: ──\[Setup 50ms\]──\[Query 5ms\]──\[Close 10ms\]──  
Request 2:                                              ──\[Setup 50ms\]──\[Query 5ms\]──\[Close 10ms\]──  
Request 3:                                                                                          ──\[Setup 50ms\]...

Total time: 65ms \+ 65ms \+ 65ms \= 195ms

---

**With Connection Pooling (Efficient\!):**

Application Startup:  
1\. Create pool of 10 connections (one-time cost)  
   \- All connections ready and waiting  
   \- Time: 500ms once 🎉

Web Server receives request:  
1\. Get connection from pool (already connected\!)  
   \- Time: \<1ms ⚡

2\. Execute query  
   \- Time: 5ms ⚡

3\. Return connection to pool (reuse it\!)  
   \- Time: \<1ms

Total: \~6ms (10x faster\!)

For 100 requests: Reuse same 10 connections\! 🎉

**Visual representation:**

Pool: \[C1\]\[C2\]\[C3\]...\[C10\] (always ready)

Request 1: \[Borrow C1\]──\[Query 5ms\]──\[Return C1\]  
Request 2:             \[Borrow C2\]──\[Query 5ms\]──\[Return C2\]  
Request 3:                         \[Borrow C1\]──\[Query 5ms\]──\[Return C1\] (reused\!)

Total time: 6ms \+ 6ms \+ 6ms \= 18ms (10x faster\!)

---

### **🎯 How Connection Pooling Works**

**The Pool Lifecycle:**

┌─────────────────────────────────────┐  
│     CONNECTION POOL                  │  
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐       │  
│  │ C1 │ │ C2 │ │ C3 │ │ C4 │ ...   │  
│  └────┘ └────┘ └────┘ └────┘       │  
│   IDLE   IDLE   IDLE   IDLE         │  
└─────────────────────────────────────┘

User Request Arrives ↓

┌─────────────────────────────────────┐  
│     CONNECTION POOL                  │  
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐       │  
│  │ C1 │ │ C2 │ │ C3 │ │ C4 │ ...   │  
│  └────┘ └────┘ └────┘ └────┘       │  
│   BUSY   IDLE   IDLE   IDLE         │  
└─────────────────────────────────────┘  
      ↓  
   Request uses C1 to query database

Query Complete ↓

┌─────────────────────────────────────┐  
│     CONNECTION POOL                  │  
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐       │  
│  │ C1 │ │ C2 │ │ C3 │ │ C4 │ ...   │  
│  └────┘ └────┘ └────┘ └────┘       │  
│   IDLE   IDLE   IDLE   IDLE         │  
└─────────────────────────────────────┘  
      ↑  
   C1 returned to pool, ready for reuse\!

---

### **🎪 Code Example: With and Without Pooling**

**Without Pooling (Bad):**

// Every request creates new connection  
app.get('/users/:id', async (req, res) \=\> {  
  // Create new connection (SLOW\! 50-100ms)  
  const connection \= await mysql.createConnection({  
    host: 'localhost',  
    user: 'root',  
    password: 'password',  
    database: 'myapp'  
  });  
    
  // Execute query (5ms)  
  const \[rows\] \= await connection.query(  
    'SELECT \* FROM users WHERE id \= ?',  
    \[req.params.id\]  
  );  
    
  // Close connection (10ms)  
  await connection.end();  
    
  res.json(rows);  
  // Total: \~65ms per request 😰  
});

**With Pooling (Good):**

// Create pool once at startup  
const pool \= mysql.createPool({  
  host: 'localhost',  
  user: 'root',  
  password: 'password',  
  database: 'myapp',  
  connectionLimit: 10,      // Max 10 concurrent connections  
  waitForConnections: true,  // Wait if pool is full  
  queueLimit: 0             // No limit on waiting queue  
});

// Every request reuses connections from pool  
app.get('/users/:id', async (req, res) \=\> {  
  // Get connection from pool (FAST\! \<1ms)  
  const connection \= await pool.getConnection();  
    
  try {  
    // Execute query (5ms)  
    const \[rows\] \= await connection.query(  
      'SELECT \* FROM users WHERE id \= ?',  
      \[req.params.id\]  
    );  
      
    res.json(rows);  
  } finally {  
    // Return connection to pool (don't close it\!)  
    connection.release();  
  }  
  // Total: \~6ms per request ⚡ (10x faster\!)  
});

---

### **⚙️ Pool Configuration: The Critical Settings**

**Key parameters to tune:**

const poolConfig \= {  
  // Minimum connections always maintained  
  connectionLimit: 10,  
    
  // Wait for connection if pool exhausted?  
  waitForConnections: true,  
    
  // Maximum waiting requests (0 \= unlimited)  
  queueLimit: 0,  
    
  // Idle timeout (close unused connections)  
  idleTimeout: 60000,  // 60 seconds  
    
  // Connection lifetime (refresh periodically)  
  maxLifetime: 1800000,  // 30 minutes  
    
  // Test connections before using  
  enableKeepAlive: true,  
  keepAliveInitialDelay: 0  
};

---

### **🎮 Interactive Exercise: What Pool Size?**

**Scenario:** Your web application has:

* 1000 requests per second  
* Each query takes 10ms on average  
* Database can handle 100 concurrent connections max

**Question:** What should your pool size be? Think about it...

**Calculation:**

Concurrent requests at any moment:  
\= Requests per second × Query time  
\= 1000 req/s × 0.01s  
\= 10 concurrent requests

Safe pool size: 10-20 connections  
(Add buffer for spikes)

Don't use 100\! That wastes database resources.

**Mental model:** Think of pool size like lanes on a highway. Too few \= traffic jams. Too many \= wasted pavement. Find the sweet spot\!

---

### **🚨 Common Pooling Mistakes**

**❌ Mistake 1: Pool Size Too Large**

// BAD: Huge pool  
const pool \= createPool({ connectionLimit: 500 });

Problems:  
\- Database overwhelmed (100+ connection limit exceeded)  
\- Memory waste (each connection uses RAM)  
\- Slower performance (too much context switching)  
\- Database may reject connections

**✅ Good: Right-sized pool**

// GOOD: Appropriate pool size  
const pool \= createPool({   
  connectionLimit: 20  // Based on load testing  
});

Benefits:  
\- Efficient resource usage  
\- Database stays healthy  
\- Predictable performance

**Rule of thumb:**

Pool size \= (Expected concurrent users × Avg query time) \+ Buffer

Example:  
\- 50 concurrent users  
\- 20ms average query time  
\- Pool size: (50 × 0.02) \+ 5 \= \~6-10 connections

---

**❌ Mistake 2: Forgetting to Release Connections**

// BAD: Connection leak\!  
app.get('/users', async (req, res) \=\> {  
  const connection \= await pool.getConnection();  
  const \[rows\] \= await connection.query('SELECT \* FROM users');  
  res.json(rows);  
  // OOPS\! Forgot to release connection\!  
});

Result after 10 requests:  
Pool: \[BUSY\]\[BUSY\]\[BUSY\]...\[BUSY\] (all 10 stuck\!)  
Next request: ⏰ TIMEOUT\! No connections available\!

**✅ Good: Always release**

// GOOD: Always release (use try-finally\!)  
app.get('/users', async (req, res) \=\> {  
  const connection \= await pool.getConnection();  
  try {  
    const \[rows\] \= await connection.query('SELECT \* FROM users');  
    res.json(rows);  
  } finally {  
    connection.release();  // Always execute\!  
  }  
});

---

**❌ Mistake 3: Creating Multiple Pools**

// BAD: New pool for each request  
app.get('/users', async (req, res) \=\> {  
  const pool \= createPool({ connectionLimit: 10 });  // DON'T\!  
  // This defeats the purpose of pooling\!  
});

**✅ Good: One pool, shared globally**

// GOOD: Create pool once at startup  
const pool \= createPool({ connectionLimit: 10 });

// Reuse everywhere  
app.get('/users', async (req, res) \=\> {  
  const connection \= await pool.getConnection();  
  // ...  
});

app.get('/orders', async (req, res) \=\> {  
  const connection \= await pool.getConnection();  // Same pool\!  
  // ...  
});

---

### **📊 Real-World Performance Impact**

**Benchmark:  Let us benchmark 1000 requests with and without pooling** 

**Without Pooling:**

Average response time: 65ms  
Total time: 65 seconds  
Database connections created: 1000  
CPU usage: High (connection churn)  
Memory: Stable but inefficient

**With Pooling (10 connections):**

Average response time: 6ms ⚡  
Total time: 6 seconds (10x faster\!)  
Database connections created: 10 (reused)  
CPU usage: Low (no overhead)  
Memory: Efficient

**Visual comparison:**

  ![][image3]                

                  10x faster\!

---

### **💡 Best Practices Summary**

**✅ DO:**

* Create pool once at application startup  
* Right-size based on load testing (usually 10-50 connections)  
* Always release connections (use try-finally)  
* Monitor pool statistics (active, idle, waiting)  
* Set reasonable timeouts  
* Implement health checks

**❌ DON'T:**

* Create new pools repeatedly  
* Make pool too large (waste) or too small (bottleneck)  
* Forget to release connections  
* Use same pool for different databases (isolate\!)  
* Ignore pool exhaustion errors

**Mental model:** Connection pooling is like having a fleet of taxis ready. Instead of calling a new taxi (slow), you grab one that's already waiting (fast), use it, and return it for the next person\!

---

## **💾 Part 4: Database Backup Types \- Your Safety Net**

### **🎯 Challenge 4: The Photo Album Disaster**

**Scenario:** You have 10,000 family photos:

**Backup Strategy A:**

* Monday: Copy all 10,000 photos (2 hours)  
* Tuesday: Copy all 10,000 photos again (2 hours)  
* Wednesday: Copy all 10,000 photos again (2 hours)  
* ...every day

**Backup Strategy B:**

* Monday: Copy all 10,000 photos (2 hours)  
* Tuesday: Copy only the 10 new photos taken today (1 minute)  
* Wednesday: Copy only the 5 new photos taken today (30 seconds)  
* ...

**Backup Strategy C:**

* Monday: Copy all 10,000 photos (2 hours)  
* Tuesday: Copy 10 new \+ record "changed since Monday" (1 minute)  
* Wednesday: Copy 5 new \+ record "changed since Tuesday" (30 seconds)  
* Saturday: Copy all changes since Monday (100 photos, 10 minutes)

**Question:** Which strategy is most efficient for regular backups? What if you need to restore?

### **The Answer: These are Full, Incremental, and Differential Backups\!**

---

### **💽 Backup Type 1: FULL BACKUP**

**Definition:** Copy EVERYTHING \- the entire database from scratch.

**How it works:**

┌──────────────────────┐  
│    FULL BACKUP       │  
│                      │  
│  📄 All tables       │  
│  📄 All rows         │  
│  📄 All data         │  
│  📄 All indexes      │  
│  📄 Everything\!      │  
│                      │  
│  Size: 100 GB        │  
│  Time: 2 hours       │  
└──────────────────────┘

**Real-world analogy:** Moving to a new house \- you pack and move EVERY single item you own.

---

**Example: Full Backup Command**

\# MySQL full backup  
mysqldump \--all-databases \--single-transaction \> full\_backup\_2025-10-15.sql

\# PostgreSQL full backup  
pg\_dump mydb \> full\_backup\_2025-10-15.sql

\# Creates complete snapshot of entire database

**What's included:**

✅ All database structures (tables, views, procedures)  
✅ All data in all tables  
✅ All indexes  
✅ User accounts and permissions  
✅ Database configuration

Result: Complete standalone backup\!

---

**✅ Advantages:**

* **Simple**: One file \= complete database  
* **Independent**: Don't need any other backups to restore  
* **Fast restore**: Just restore one file  
* **Easy to verify**: One backup to test

**❌ Disadvantages:**

* **Large size**: 100 GB database \= 100 GB backup  
* **Slow**: Takes hours for large databases  
* **Resource intensive**: High I/O and CPU during backup  
* **Storage expensive**: Daily full backups eat storage fast

**When to use:**

✓ Weekly baseline backup  
✓ Before major changes (migrations, upgrades)  
✓ Small databases (\<10 GB)  
✓ Regulatory compliance requirements  
✓ Long-term archival

---

### **📦 Backup Type 2: INCREMENTAL BACKUP**

**Definition:** Copy only data that changed SINCE THE LAST BACKUP (of any type).

**How it works:**

Sunday (Full):     ████████████████████ 100 GB  
Monday (Incr):     ██ 2 GB (changed since Sunday)  
Tuesday (Incr):    █ 1 GB (changed since Monday\!)  
Wednesday (Incr):  ██ 2 GB (changed since Tuesday\!)  
Thursday (Incr):   █ 1 GB (changed since Wednesday\!)

Total storage: 106 GB

**Chain dependency:**

FULL (Sun) → INCR (Mon) → INCR (Tue) → INCR (Wed) → INCR (Thu)  
    ↑           ↑            ↑            ↑            ↑  
   Need      Need Mon     Need Tue     Need Wed     Need Thu  
   this      to restore   to restore   to restore   to restore

**Real-world analogy:** Packing for a trip where you add only new items each day \- Day 1: Full suitcase, Day 2: Add toothbrush, Day 3: Add sunglasses.

---

**Example: Incremental Backup**

\# MySQL incremental (using binary logs)  
\# Full backup Sunday  
mysqldump mydb \> full\_sunday.sql

\# Monday: backup binary logs (changes only)  
mysqlbinlog mysql-bin.000001 \> incremental\_monday.sql

\# Tuesday: backup new binary logs (changes since Monday)  
mysqlbinlog mysql-bin.000002 \> incremental\_tuesday.sql

**Visual timeline:**

Database changes over time:

Sunday:    \[Table1: 1000 rows\]\[Table2: 500 rows\] \= FULL BACKUP  
Monday:    \[Table1: \+50 rows\] \= INCR (50 rows)  
Tuesday:   \[Table2: \+20 rows\] \= INCR (20 rows)  
Wednesday: \[Table1: \+100 rows\]\[Table2: \+30 rows\] \= INCR (130 rows)

---

**✅ Advantages:**

* **Fast**: Only backs up changes (minutes vs hours)  
* **Small**: Each backup is tiny (1-5 GB vs 100 GB)  
* **Efficient**: Minimal resource usage  
* **Frequent**: Can run every hour if needed

**❌ Disadvantages:**

* **Complex restore**: Need ALL backups in chain  
* **Fragile**: If one backup corrupts, chain breaks  
* **Slower restore**: Must apply each backup sequentially  
* **Chain management**: Must track dependencies

**Restore process (the pain\!):**

To restore Wednesday data:

Step 1: Restore FULL (Sunday)     ⏱️ 2 hours  
Step 2: Apply INCR (Monday)       ⏱️ 10 minutes  
Step 3: Apply INCR (Tuesday)      ⏱️ 5 minutes  
Step 4: Apply INCR (Wednesday)    ⏱️ 8 minutes

Total restore time: \~2.5 hours  
(And if any backup is corrupt, restore FAILS\!)

**When to use:**

✓ Frequent backups (hourly/every few hours)  
✓ Large databases with small daily changes  
✓ When storage space is limited  
✓ Between differential backups  
✓ Continuous data protection

---

### **🔄 Backup Type 3: DIFFERENTIAL BACKUP**

**Definition:** Copy all data that changed SINCE THE LAST FULL BACKUP (not since last backup).

**How it works:**

Sunday (Full):     ████████████████████ 100 GB  
Monday (Diff):     ██ 2 GB (changed since Sunday)  
Tuesday (Diff):    ███ 3 GB (changed since Sunday\!)  
Wednesday (Diff):  █████ 5 GB (changed since Sunday\!)  
Thursday (Diff):   ███████ 7 GB (changed since Sunday\!)

Total storage: 117 GB (more than incremental)

**Chain dependency:**

FULL (Sun) → DIFF (Mon) ✓ Can restore from these two  
    ↑          
FULL (Sun) → DIFF (Tue) ✓ Can restore from these two  
    ↑  
FULL (Sun) → DIFF (Wed) ✓ Can restore from these two  
    ↑  
FULL (Sun) → DIFF (Thu) ✓ Can restore from these two

Only need: FULL \+ Latest DIFF (much simpler\!)

**Real-world analogy:** Writing a book \- you keep the original manuscript (full backup) and each day you save a copy with ALL changes made since the original (not just today's changes).

---

**Example: Differential Backup**

\# Sunday: Full backup  
mysqldump mydb \> full\_sunday.sql

\# Monday: Differential (all changes since Sunday)  
mysqldump \--where="modified\_date \>= 'Sunday'" mydb \> diff\_monday.sql

\# Tuesday: Differential (all changes since Sunday, not Monday\!)  
mysqldump \--where="modified\_date \>= 'Sunday'" mydb \> diff\_tuesday.sql

\# Each diff contains cumulative changes since Sunday

**Visual timeline:**

Database changes over time:

Sunday:    \[1000 rows total\] \= FULL BACKUP

Monday:    \[+50 new rows\]  
           DIFF \= 50 rows

Tuesday:   \[+20 more new rows\]  
           DIFF \= 50 \+ 20 \= 70 rows (cumulative\!)

Wednesday: \[+100 more new rows\]  
           DIFF \= 50 \+ 20 \+ 100 \= 170 rows (cumulative\!)

---

**✅ Advantages:**

* **Simple restore**: Only need FULL \+ Latest DIFF  
* **Fast restore**: Two backups \= quick recovery  
* **Safer**: Less fragile than incremental chain  
* **Good balance**: Faster than full, simpler than incremental

**❌ Disadvantages:**

* **Growing size**: Each diff gets bigger over time  
* **More storage**: Larger than incremental  
* **Redundancy**: Backs up same changes multiple times

**Restore process (much simpler\!):**

To restore Wednesday data:

Step 1: Restore FULL (Sunday)       ⏱️ 2 hours  
Step 2: Apply DIFF (Wednesday only) ⏱️ 30 minutes

Total restore time: 2.5 hours  
(Only need 2 backups \- much simpler\!)

**When to use:**

✓ Daily backups (good middle ground)  
✓ Medium-to-large databases  
✓ When restore speed matters  
✓ When simplicity is important  
✓ Most common strategy for production\!

---

### **📊 Side-by-Side Comparison**

**Scenario: 100 GB database, 2 GB changes per day**

| Day | Full | Incremental | Differential |
| ----- | ----- | ----- | ----- |
| **Sun** | 100 GB | 100 GB | 100 GB |
| **Mon** | 100 GB | 2 GB | 2 GB |
| **Tue** | 100 GB | 2 GB | 4 GB |
| **Wed** | 100 GB | 2 GB | 6 GB |
| **Thu** | 100 GB | 2 GB | 8 GB |
| **Fri** | 100 GB | 2 GB | 10 GB |
| **Sat** | 100 GB | 2 GB | 12 GB |
| **TOTAL** | **700 GB** | **112 GB** ✓ | **142 GB** |

**Restore complexity:**

Full:         ⭐ Easiest (1 file)  
Differential: ⭐⭐ Easy (2 files: Full \+ Latest Diff)  
Incremental:  ⭐⭐⭐⭐⭐ Complex (7 files: Full \+ 6 Incrementals)

**Restore time:**

Full:         2 hours (restore 1 file)  
Differential: 2.5 hours (restore Full \+ Diff)  
Incremental:  3+ hours (restore Full \+ all Incrementals)

---

### **🎪 Real-World Backup Strategy: The 3-2-1 Rule**

**Best practice: Combine all three types\!**

**![][image4]**

┌**The 3-2-1 Rule:**  
3 \= Keep 3 copies of data  
    \- Production database  
    \- Local backup  
    \- Offsite backup

2 \= Use 2 different storage types  
    \- Local disk/SAN  
    \- Cloud storage (S3, Azure Blob)

1 \= Keep 1 copy offsite  
    \- Different geographic location  
    \- Protected from local disasters

---

### **🎮 Interactive Exercise: Design a Backup Strategy**

**Your database:**

* Size: 500 GB  
* Daily changes: 10 GB  
* Business hours: 8am \- 8pm  
* Recovery Time Objective (RTO): 2 hours  
* Recovery Point Objective (RPO): 4 hours max data loss

**Design a backup strategy. Consider:**

1. When to run full backups?  
2. When to run differential backups?  
3. When to run incremental backups?  
4. Where to store backups?

**Think about it...**

---

**Sample Solution:**

STRATEGY:

FULL BACKUP:  
\- Sunday 2am (weekly)  
\- Time: 4 hours (runs overnight)  
\- Retention: 4 weeks

DIFFERENTIAL BACKUP:  
\- Daily at 2am (Mon-Sat)  
\- Time: 1 hour  
\- Retention: 1 week

INCREMENTAL BACKUP:  
\- Every 4 hours during business hours (8am, 12pm, 4pm, 8pm)  
\- Time: 15 minutes each  
\- Retention: 3 days

STORAGE:  
\- Local: Fast SSD (last 7 days)  
\- Remote: Cloud (S3) for everything  
\- Offsite: Different region for weekly fulls

WHY THIS WORKS:  
✓ RPO: 4 hours (matches incremental schedule)  
✓ RTO: \~2 hours (restore Full \+ Diff \+ few Incrementals)  
✓ Balance: Not too much storage, not too risky  
✓ 3-2-1 compliant

---

### **🚨 Backup Disasters (What Can Go Wrong)**

**Horror Story 1: Untested Backups**

❌ Problem:  
   \- Ran backups daily for 2 years  
   \- Never tested restore  
   \- Database crashed  
   \- Tried to restore: ALL BACKUPS CORRUPTED\! 😱

✅ Solution:  
   \- Test restore monthly  
   \- Automate restore tests  
   \- Verify backup integrity

**Horror Story 2: No Offsite Storage**

❌ Problem:  
   \- All backups on same server as database  
   \- Server room flooded  
   \- Lost database AND all backups\! 💦

✅ Solution:  
   \- Store backups in different location  
   \- Use cloud storage  
   \- 3-2-1 rule\!

**Horror Story 3: Backup Windows Missed**

❌ Problem:  
   \- Full backup takes 6 hours  
   \- Business needs 24/7 uptime  
   \- Can't find backup window  
   \- Backup fails/incomplete

✅ Solution:  
   \- Use online backups (snapshot technology)  
   \- Implement incremental forever  
   \- Use read replicas for backups

---

### **💡 Best Practices Checklist**

**✅ DO:**

* Test restores regularly (monthly minimum)  
* Automate backup process  
* Monitor backup success/failure  
* Encrypt backups (especially offsite)  
* Document restore procedures  
* Use 3-2-1 rule  
* Set retention policies (how long to keep)  
* Verify backup integrity  
* Time backups during low-traffic periods

**❌ DON'T:**

* Assume backups work (test them\!)  
* Store all backups in one location  
* Keep backups forever (costs add up)  
* Forget to backup configuration files  
* Run backups during peak hours  
* Ignore backup failure alerts  
* Use same credentials for backup and production

![][image5]

---

### **🎯 Quick Decision Guide: Which Backup Type?**

Choose FULL when:  
✓ Starting new backup cycle (weekly baseline)  
✓ Small database (\<10 GB)  
✓ Before major changes  
✓ Regulatory requirements

Choose DIFFERENTIAL when:  
✓ Daily backups (best for most cases)  
✓ Need balance of speed and simplicity  
✓ Restore speed is important  
✓ Medium changes daily

Choose INCREMENTAL when:  
✓ Very frequent backups (hourly)  
✓ Large database with small changes  
✓ Storage space is limited  
✓ Between differential backups

---

## **🎓 Complete System Design: Putting It All Together**

### **The Production-Ready Database Architecture**

Let's synthesize everything we've learned into a complete system\!

**Scenario: E-commerce Database**

* 1TB database  
* 10,000 reads/second (95%)  
* 500 writes/second (5%)  
* 24/7 availability required  
* Max 1-hour data loss acceptable  
* 2-hour recovery time acceptable

---

**Architecture:**

![][image6]  
---

**How it works:**

**1\. Connection Pooling:**

// Pool configuration  
const pool \= createPool({  
  connectionLimit: 50,  
  waitForConnections: true,  
  queueLimit: 0  
});

// Each API server shares this pool  
// 3 API servers × 50 connections \= max 150 concurrent  
// Handles 10,500 req/s easily

**2\. Read/Write Split:**

// Route writes to primary  
async function createOrder(order) {  
  const conn \= await primaryPool.getConnection();  
  try {  
    await conn.beginTransaction();  // ACID transaction  
    await conn.query('INSERT INTO orders...');  
    await conn.query('UPDATE inventory...');  
    await conn.commit();  
  } catch (error) {  
    await conn.rollback();  // Atomic\!  
    throw error;  
  } finally {  
    conn.release();  
  }  
}

// Route reads to replicas (load balanced)  
async function getOrders(userId) {  
  const conn \= await replicaPool.getConnection();  // Read replica  
  try {  
    return await conn.query('SELECT \* FROM orders WHERE user\_id \= ?', \[userId\]);  
  } finally {  
    conn.release();  
  }  
}

**3\. Backup Schedule:**

\# Sunday 2am \- FULL (4 hours)  
0 2 \* \* 0 /backup/full.sh

\# Mon-Sat 2am \- DIFFERENTIAL (1 hour)  
0 2 \* \* 1-6 /backup/differential.sh

\# Every hour \- INCREMENTAL (10 minutes)  
0 \* \* \* \* /backup/incremental.sh

\# Verify backups daily  
0 4 \* \* \* /backup/verify.sh

---

**Performance metrics achieved:**

┌──────────────────────────────────────────────┐  
│ PERFORMANCE RESULTS                           │  
├──────────────────────────────────────────────┤  
│ Read Operations:                              │  
│  • Average latency: 5ms ⚡                   │  
│  • 95th percentile: 12ms                     │  
│  • Throughput: 10,000/sec (distributed)      │  
│                                               │  
│ Write Operations:                             │  
│  • Average latency: 25ms                     │  
│  • 95th percentile: 50ms                     │  
│  • Throughput: 500/sec (primary handles)     │  
│                                               │  
│ Connection Pool:                              │  
│  • Average wait time: \<1ms                   │  
│  • Connection reuse: 1000+ queries/conn      │  
│  • Pool exhaustion: Never (well-sized)       │  
│                                               │  
│ Recovery:                                     │  
│  • RPO: 1 hour (hourly incrementals)        │  
│  • RTO: 2 hours (tested monthly)            │  
│  • Last 12 restore tests: 12/12 success ✓   │  
└──────────────────────────────────────────────┘

---

## **🏆 Final Knowledge Check**

**Without looking back, can you explain:**

1. **Transactions:** Why does a bank transfer need to be atomic? What happens during rollback?

2. **Read vs Write:** Why are writes more expensive than reads? How do you optimize each?

3. **Connection Pooling:** What problem does it solve? What's the downside of a pool that's too large?

4. **Backups:** You have Sunday Full \+ Monday Diff \+ Tuesday Diff. Database crashes Wednesday morning. What do you restore? How long does it take?

---

**Answers:**

1. **Atomicity ensures money isn't lost or duplicated.** Rollback undoes ALL operations, returning database to pre-transaction state. Without transactions, partial completions create inconsistencies.

2. **Writes require locks, validation, logging, and multiple disk writes.** Reads just fetch data. Optimize reads with caching, replicas, and indexes. Optimize writes with batching, async processing, and write-optimized storage.

3. **Connection pooling eliminates expensive connection setup/teardown overhead**, reusing connections across requests. Too large wastes database resources and memory; proper sizing based on concurrent load is critical.

4. **Restore Sunday Full (2 hours) \+ Tuesday Diff (30 min) \= 2.5 hours.** Don't need Monday Diff because Tuesday Diff contains all changes since Sunday. Only need latest differential\!

---

## **🚀 Your Next Learning Adventures**

**Master these fundamentals first, then explore:**

**Immediate Next Steps:**

* Database Replication topologies (Primary-Replica, Multi-Primary)  
* Sharding strategies (Horizontal partitioning)  
* Query optimization and execution plans  
* Database monitoring and observability

**Advanced Topics:**

* Distributed databases (Cassandra, CockroachDB)  
* Event sourcing and CQRS  
* Database migration strategies  
* Disaster recovery planning  
* Point-in-time recovery

**Practical Projects:**

* Build a multi-region database architecture  
* Implement automated backup/restore pipeline  
* Create connection pool monitoring dashboard  
* Design a high-availability system

**Remember:** These concepts are the foundation of reliable, scalable systems. Master them, and you'll understand how every major application maintains data integrity, performance, and reliability\! 💪

---

**🎓 You've now mastered:** ✅ Transactions (ACID guarantees) ✅ Read/Write optimization ✅ Connection pooling efficiency ✅ Backup strategies (Full, Incremental, Differential)

**Congratulations\! You're ready to design production-grade database systems\!** 🎉

[image1]: <data:image/png;base64,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>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,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>

[image4]: <data:image/png;base64,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>

[image5]: <data:image/png;base64,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>

[image6]: <data:image/png;base64,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>