---
slug: primary-foreign-keys
title: Primary and Foreign Keys
readTime: 10 min
orderIndex: 4
premium: false
---



## **Primary Key vs Foreign Key \- The Relationship Builders**

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


```sql
 CREATE TABLE students (  student_id INT PRIMARY KEY,   first_name VARCHAR(50),  last_name VARCHAR(50),  email VARCHAR(100),  birth_date DATE)

```
\-- Sample data:

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764427380/208_uykzxc.png)

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

```sql
 CREATE TABLE enrollments (
enrollment_id INT PRIMARY KEY,
student_id INT, course_id INT, enrollment_date DATE,  grade VARCHAR(2), FOREIGN KEY (student_id) REFERENCES students(student_id),  FOREIGN KEY (course_id) REFERENCES courses(course_id))

```
**Visual relationship:**

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764427379/207_xje9pv.png)

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

 Scenario 1: Orphaned Records

 Try to insert enrollment for non-existent student
```sql
 INSERT INTO enrollments (student_id, course_id) VALUES (9999, 'CS101');

 ```
 \-- Student 9999 doesn't exist\!\-- Result: ❌ ERROR: Foreign key constraint violated\!\-- Database says: "Can't enroll a student that doesn't exist\!

 Scenario 2: Broken References
 \-- Try to delete a student who has enrollments
 ```sql

 DELETE FROM students WHERE student_id = 1001;
 ```

 \-- Result: ❌ ERROR: Foreign key constraint violated

 \-- Database says: "Can't delete student with existing enrollments\!"

 \-- (You'd have orphaned enrollments pointing to non-existent student)Options for handling deletions:

 \-- CASCADE: Delete enrollments when student is deleted


 FOREIGN KEY (student_id)
 REFERENCES students(student_id) ON DELETE CASCADE
 SET NULL: Set student_id to NULL in enrollments
 FOREIGN KEY (student_id) REFERENCES students(student_id)
 ON DELETE SET NULL;
 \-- RESTRICT: Prevent deletion (default)
 FOREIGN KEY (student_id) REFERENCES students(student_id)  ON DELETE RESTRICT;




**Mental model:** Foreign keys are like safety ropes in mountain climbing. They ensure you can't fall off a cliff (create invalid references) and keep everything connected safely\!

---

### **📊 Complete Example: E-Commerce Database**


 CUSTOMERS table

 ```sql
 CREATE TABLE customers (  customer_id INT PRIMARY KEY,   name VARCHAR(100),  email VARCHAR(100),  address VARCHAR(255));

 ```

 ORDERS table
```sql
 CREATE TABLE orders (  order_id INT PRIMARY KEY,         customer_id INT,  order_date DATE,  total_amount DECIMAL, FOREIGN KEY (customer_id) REFERENCES customers(customer_id));
 ```

  ORDER_ITEMS table
  ```sql
  CREATE TABLE order_items (  item_id INT PRIMARY KEY,    order_id INT,  orders  product_id INT,
  products  quantity INT,  price DECIMAL,   FOREIGN KEY (order_id) REFERENCES orders(order_id),  FOREIGN KEY (product_id) REFERENCES products(product_id));
  ```

 PRODUCTS table
 ```sql

 CREATE TABLE products (  product_id INT PRIMARY KEY,   name VARCHAR(200),  description TEXT,  price DECIMAL,  stock INT);

```
**The relationships:**

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764427379/206_ebbar7.png)

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
