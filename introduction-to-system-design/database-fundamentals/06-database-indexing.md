---
slug: database-indexing
title: Database Indexing
readTime: 10 min
orderIndex: 6
premium: false
---





## **Database Indexing \- The Library Card Catalog**

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
```sql
 CREATE TABLE products (  product_id INT,  name VARCHAR(255),  price DECIMAL,  category VARCHAR(100),  description TEXT,  created_at DATE);

 ```
 Common queries:
 ```sql

 SELECT * FROM products WHERE product_id = 12345;

 SELECT * FROM products WHERE category = 'Electronics';

 SELECT * FROM products WHERE price > 100;

 SELECT * FROM products WHERE name LIKE 'phone';

 ```

 Question: Which columns should you index? Think about it...

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

 Index on single column
 ```sql

 CREATE INDEX idx_email ON users(email);
 ```

Composite index
```sql

CREATE INDEX idx_name ON users(last_name, first_name);

```

 Unique index

```sql

CREATE UNIQUE INDEX idx_username ON users(username);
```


---

### **🔥 Real-World Performance Example**

 Query without index:
 ```sql

SELECT * FROM orders WHERE customer_id = 12345;
```

Execution time: 8 seconds (full table scan)

Rows examined: 10,000,000

Same query WITH index

```sql
CREATE INDEX idx_customer ON orders(customer_id);

SELECT * FROM orders WHERE customer_id = 12345;
```

Execution time: 0.003 seconds

⚡ Rows examined: 50 (via index)

**Speed improvement: 2,667x faster\!**

**Visual analogy:**

**![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764476903/database_indexing_zrax60.png)**

---

## **🎯 Final Synthesis: Putting It All Together**

### **The Complete Picture: Building a Library System**

Let's apply everything we've learned\!

**1️⃣ Database Choice:**

* Need structured data (books, members, loans)
* Need relationships (members ←→ loans ←→ books)
* Need accuracy (can't lose loan records\!)
* **Choice: SQL Database with ACID properties** ✓

| 2️⃣ Table Design with Keys
```sql
CREATE TABLE books (  book_id INT PRIMARY KEY,          isbn VARCHAR(13),  title VARCHAR(200),  author VARCHAR(100), published_date DATE);


CREATE TABLE members (  member_id INT PRIMARY KEY,       name VARCHAR(100),  email VARCHAR(100),  join_date DATE);

CREATE TABLE loans (  loan_id INT PRIMARY KEY,             book_id INT,member_id INT,loan_date DATE,due_date DATE, returned_date DATE,FOREIGN KEY (book_id) REFERENCES books(book_id),  FOREIGN KEY (member_id) REFERENCES members(member_id));
```

3️⃣ Add Indexes for Performance
```sql

CREATE INDEX idx_book_isbn ON books(isbn)

CREATE INDEX idx_member_email ON members(email);

CREATE INDEX idx_loan_dates ON loans(loan_date, due_date)
```

4️⃣ CRUD Operations:

Create: New member joins
```sql
INSERT INTO members (member_id, name, email, join_date)VALUES (1, 'Alice Johnson', 'alice@email.com', '2025-10-15');
```

Read: Find overdue books
```sql

SELECT books.title, members.name, loans.due_date

FROM loans

JOIN books ON loans.book_id = books.book_id

JOIN members ON loans.member_id = members.member_id

WHERE loans.due_date < CURRENT_DATE   AND

loans.returned_date IS NULL;
```

Update: Return a book

UPDATE loans
```sql
SET returned_date = CURRENT_DATE WHERE loan_id = 123
```

Delete: Remove old loan records
```sql

DELETE FROM loans WHERE returned_date < '2020-01-01';
```


5️⃣ ACID Transaction: Loan a book

```sql
BEGIN TRANSACTION;

//Check if book is available

SELECT * FROM books WHERE book_id = 42 AND available = true;
// Create loan record

INSERT INTO loans (book\_id, member_id, loan_date, due_date)  VALUES (42, 1, CURRENT_DATE, CURRENT_DATE + 14);

 Mark book as unavailable
UPDATE books SET available = false WHERE book_id = 42;

COMMIT;
```
\-- All or nothing\!**

---

## **🏆 Quick Recap: Test Your Understanding**

**Without looking back, can you explain:**

1. **When would you choose SQL vs NoSQL?**
2. **What does each letter in ACID stand for and why does it matter?**
3. **Why would adding an index make a query faster?**
4. **What's the difference between a Primary Key and a Foreign Key?**
5. **What do the letters in CRUD stand for?**

**Mental check:** If you can answer these clearly, you've mastered database fundamentals\! 🎓
