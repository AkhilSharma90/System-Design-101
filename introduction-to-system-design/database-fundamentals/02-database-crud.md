---
slug: database-crud
title: Database Crud Operations
readTime: 15 min
orderIndex: 2
premium: false
---
# **CRUD Operations - The Four Essential Database Actions**

### **🎯 Challenge 5: The Todo App**

**Scenario:** You're building a simple todo list app. What actions do users need?

Think about it:

* How do you add a new task?
* How do you see your tasks?
* How do you mark a task as complete?
* How do you remove a task?

**Answer: CRUD Operations!**

---

### **🔤 What is CRUD?**

**CRUD \= The four basic operations you can do with data**

* **C**reate - Add new records
* **R**ead - View/retrieve records
* **U**pdate - Modify existing records
* **D**elete - Remove records

**Real-world parallel:** Think of a contact list on your phone:

* 📝 **Create:** Add new contact
* 👁️ **Read:** View contact info
* ✏️ **Update:** Change phone number
* 🗑️ **Delete:** Remove old contact

---

### **📝 CREATE: Adding New Data (to an already created table i.e., you already have a table with datatypes for each field now you are adding data to it)**

**SQL Command:** `INSERT`

**Example: Adding a new user**
```sql
 INSERT INTO users (username, email, age) VALUES ('alice123', 'alice@email.com', 25)
 ```

**What happens:**


![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764418672/pre_post_db_insert_zpb0eb.png)

**Multiple records at once:**
```sql
 INSERT INTO products (name, price, category)VALUES ('Laptop', 999.99,
 'Electronics'), ('Mouse', 19.99, 'Electronics'), ('Notebook', 3.99,
 'Stationery');
 ```

**Real-world scenarios:**


* 🛒 Adding items to shopping cart
* 👤 User registration
* 📧 Posting a comment
* 📅 Creating a calendar event

**Mental model:** INSERT is like writing a new entry in a logbook - you're adding permanent information!

---

### **👁️ READ: Retrieving Data**

**SQL Command:** `SELECT`

**Example: View all users**

```sql
  SELECT * FROM users;
```

**Result:![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764410154/200_cnzn9c.png)**

**Select specific columns:**

```sql
 SELECT username, email FROM users;
 ```

**Result:**

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764410152/196_nua6ef.png)

**Filtering with WHERE:**

```sql
SELECT * FROM users WHERE age > 26
```

**Result:**

![img4](https://res.cloudinary.com/dretwg3dy/image/upload/v1764410155/198_pdvbhi.png)

**Sorting results:**

```sql
 SELECT * FROM users ORDER BY age DESC;
 ```

**Result:**

![img5](https://res.cloudinary.com/dretwg3dy/image/upload/v1764410155/195_ejip7s.png)

**Advanced: Joining tables**

 SELECT users.username, orders.order_date, orders.total FROM users JOIN orders ON users.id = orders.user_id WHERE users.username = 'alice123';
**Real-world scenarios:**

* 📱 Loading your profile
* 📰 Viewing news feed
* 🔍 Searching for products
* 📊 Generating reports

**Mental model:** SELECT is like looking through a filing cabinet with superpowers - you can view, filter, and organize information instantly!

---

### **✏️ UPDATE: Modifying Existing Data**

**SQL Command:** `UPDATE`

**Example: Change user's email**

```sql
 UPDATE users SET email = 'alice.new@email.com' WHERE username = 'alice123';

```
**What happens:**

![img6](https://res.cloudinary.com/dretwg3dy/image/upload/v1764410153/197_jb96at.png)

**Update multiple columns:**
```sql
 UPDATE users SET email = 'bob.updated@email.com',age = 31WHERE username = 'bob456';
 ```
**Update multiple rows:**
```sql
 UPDATE products SET price = price * 0.9   10% discount WHERE category= 'Electronics';

```
**⚠️ WARNING: Always use WHERE clause!**

-- DANGEROUS! Updates ALL rows:
```sql
 UPDATE users SET age = 50;
 ```

-- Everyone is now 50! |

-- SAFE: Updates specific user:

```sql
 UPDATE users SET age = 50 WHERE username= 'bob456';
 ```


**Real-world scenarios:**

* 📝 Editing your profile
* ✅ Marking tasks as complete
* 📊 Updating inventory quantities
* 🔄 Changing order status

**Mental model:** UPDATE is like using correction fluid and writing over old information - the record stays, but the content changes!

---

### **🗑️ DELETE: Removing Data**

**SQL Command:** `DELETE`

**Example: Remove a user**

```sql
 DELETE FROM users WHERE username = 'alice123';
 ```

**What happens:**

![img7](https://res.cloudinary.com/dretwg3dy/image/upload/v1764410155/199_twa33u.png)


alice123 is GONE!

**Delete multiple rows:**
```sql
 DELETE FROM orders WHERE order_date < '2020-01-01';

```
Remove old orders

**⚠️ WARNING: Always use WHERE clause!**

 CATASTROPHIC! Deletes ALL data:
```sql
DELETE FROM users;
```

-- Table now empty! 😱

-- SAFE: Deletes specific user:

```sql
 DELETE FROM users WHERE id = 2;
 ```

**Soft Delete vs Hard Delete:**

**Hard Delete (permanent):**

```sql
 DELETE FROM users WHERE id = 2;
```
 -- User is completely removed from database

**Soft Delete (mark as deleted but keep data):**

-- Add deleted_at column to table

```sql
 UPDATE users SET deleted_at = NOW() WHERE id = 2;

```
 Later, when querying:
```sql
 SELECT * FROM users WHERE deleted_at IS NULL;

 ```
 -- Only active users
**Real-world scenarios:**

* 🗑️ Removing items from cart
* 🚫 Deleting your account
* 🧹 Cleaning up old data
* ❌ Canceling an order

**Mental model:** DELETE is like shredding a document - once it's gone, it's gone (unless you have backups!)

---

### **🎮 CRUD in Action: Complete Example**

**Scenario: Managing a blog**


 1️⃣ CREATE: Write new post
 ```sql
INSERT INTO posts (title, content, author_id, created_at)VALUES ('My First Post', 'Hello World\!', 1, NOW());
```

2️⃣ READ: View all posts
```sql
SELECT posts.title, posts.content, users.username FROM posts JOIN users ON posts.author_id = users.id ORDER BY posts.created_at DESC
```

3️⃣ UPDATE: Edit post
```sql
UPDATE posts
SET content = 'Updated content here\!',    updated_at = NOW()WHERE id = 1;
```

```sql
4️⃣ DELETE: Remove post

DELETE FROM posts WHERE id = 1;

```

### **🌐 CRUD in REST APIs**

**CRUD maps to HTTP methods:**

| CRUD Operation | HTTP Method | Example URL | Action |
| ----- | ----- | ----- | ----- |
| **Create** | POST | `POST /api/users` | Create new user |
| **Read** | GET | `GET /api/users/123` | Get user #123 |
| **Update** | PUT/PATCH | `PUT /api/users/123` | Update user #123 |
| **Delete** | DELETE | `DELETE /api/users/123` | Delete user #123 |

**Example API calls:**

CREATE

fetch('/api/posts', {  method: 'POST',  body: JSON.stringify({ title: 'New Post', content: '...' })})

READ

fetch('/api/posts/123');

UPDATE

fetch('/api/posts/123', {method: 'PUT',  body: JSON.stringify({ title: 'Updated Title' })});

DELETE

fetch('/api/posts/123', { method: 'DELETE' });

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
* Trust user input (always validate!)


---

## Key Takeaways

1. **CRUD (Create, Read, Update, Delete) covers all basic database operations** — every application ultimately performs some combination of these
2. **INSERT creates new records, SELECT reads them, UPDATE modifies them, DELETE removes them** — master these four SQL statements first
3. **Most applications are read-heavy** — optimize for reads with proper indexing and caching
4. **Soft deletes (marking as inactive) are safer than hard deletes** — you can always recover soft-deleted data
