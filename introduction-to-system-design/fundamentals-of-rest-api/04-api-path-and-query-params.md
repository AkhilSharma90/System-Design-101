---
slug: api-path-and-query-params
title: API Path and Query Parameters
readTime: 15 min
orderIndex: 4
premium: false
---




# Query Parameters vs Path Parameters: The Address System

Alright, let's talk about something that confuses a lot of beginners. When you're designing an API, you need to send information to the server, right? There are two main ways to do this in the URL itself: **path parameters** and **query parameters**.

Let's look at  a   story that makes it crystal clear.

### **The Library Analogy**

Imagine you walk into a huge library. You want to find a specific book. There are two ways to describe what you want:

**Method 1: The Shelf Address (Path Parameters)** "I want the book on Shelf 5, Row 3, Position 12"

* This is specific, structured, hierarchical
* You're navigating through a tree: Building → Floor → Shelf → Row → Position

**Method 2: The Description (Query Parameters)** "I want books about cooking, published after 2020, sorted by rating"

* These are filters and options
* You're describing characteristics you want

### **Path Parameters: The Hierarchical Structure**

Think of path parameters as **parts of the address itself**. Let me show you:

Example 1: Getting a Specific User

━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```bash
 GET /users/42
```

 Breaking it down:

```bash
 /users ← "Go to the users collection"
```

"Get the specific user with ID 42"

Notice how 42 is **part of the path**. It's like saying "Go to apartment building USERS, unit number 42."

Here's another example:

Example 2: Getting a Specific Post by a Specific User
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```bash

 GET /users/42/posts/7

```

Reading it hierarchically:

/users

"Start with users"

/42

"Find user with ID 42"/posts ← "Look at that user's posts"

/7

 "Get post with ID 7"




It's like a filing system: Cabinet → Drawer → Folder → Document

### **Let’s see  when to Use Path Parameters**

**Use path parameters when:**

1. You're identifying a specific resource
2. The parameter is required
3. The parameter is part of a hierarchy

Real examples from famous APIs:

GitHub API:

Reading: "Get issue \#123 from the react repo owned by facebook"

Twitter API (historical):

```bash
 GET /repos/facebook/react/issues/123
```

 owner   repo   type   issue

 ```bash

 GET /users/elonmusk/tweets/987654321
 ```

 username    type   tweet ID

Reading:

"Get tweet \#987654321 from user elonmusk"

### **Query Parameters: The Filters and Options**

Now, query parameters are different. They're like **adding filters to your search**. They come after a `?` in the URL.

Let me show you:

Example 1: Searching for Users
━━━━━━━━━━━━━━━━━━━━━━━━━━

```bash
GET /users?age=25\&city=Seattle\&sort=name ```


 Breaking it down:
 /users           "I want users"?
 "Now I'm adding filters " age=25

 "Only those aged 25"&
 "And also..." city=Seattle

 "Who live in Seattle"

 "And also..."sort=name

 "Sort them by name"

Think of query parameters as answering questions:

* "Which users?" → age=25
* "Where?" → city=Seattle
* "In what order?" → sort=name

### **The  Golden Rule**

**Ask yourself: "Is this identifying WHICH resource, or HOW I want to see it?"**

**WHICH → Path Parameter**

```bash

 GET /books/12345          ← WHICH book? \#12345

 ```

```bash

 GET /users/john/orders/99 ← WHICH user's WHICH order?
 ```


**HOW → Query Parameter**

```bash
 GET books?format=pdf\&lang=spanish

 ← HOW do I want books? PDF, Spanish

 GET /users?status=active\&role=admin

 ← HOW do I want to filter users?
 ```

### **Let’s Walk  Through a Real Scenario**

Imagine you're building a movie database API. A client wants to get reviews for a specific movie, but only 5-star reviews, sorted by date.

**Bad Design (mixing everything in query):**

```bash

GET /reviews?movie\_id=123\&rating=5\&sort=date

```

This works, but `movie_id` should be in the path because it identifies WHICH resource we're talking about.

**Good Design:**

```bash

 GET /movies/123/reviews?rating=5\&sort=date

 Path param    Query params    (WHICH movie)  (HOW to filter/sort)

```
See the difference? The movie ID is part of the structure (which movie's reviews), while rating and sort are filters (how to show those reviews).

### **Another Real Example: Blog Posts**

Let's say you're building a blog platform. Walk through this with me:

**Scenario 1: Get a specific post**

```bash
 GET /posts/456
 ```

 SimpleThe post ID is in the path."Show me THE post with ID 456"

**Scenario 2: Get all posts by an author**

 Option A:
 ```bash

 GET /authors/john/posts

 Option B:

 GET /posts?author=john
```

Which is better?

If author is FUNDAMENTAL to the structure

Option A

If you're FILTERING posts

Option B

Both can work\! It depends on your API design philosophy.
Most APIs choose Option A for cleaner structure.

**Scenario 3: Search posts with multiple filters**

```bash
 GET /posts?author=john\&category=tech\&status=published\&limit=10
 ```


Here, EVERYTHING is a filter:

\- author=john    ← Filter by author

\- category=tech  ← Filter by category

\- status=published <-Filter by status

\- limit=10       ← How many results |

### **Multiple Query Parameters: The Shopping Example**

Let’s see a really practical example. Imagine you're building Amazon's API. A user is searching for laptops:

```bash
 GET products?category=laptops&brand=dell&min_price=500&max_price=1500&ram=16GB&sort=price_low_to_high&page=2&limit=20

 ```


Let's read this like a sentence:
"Give me PRODUCTS (the resource)
 where category is laptops (filter)
 and brand is dell (filter)
 and price is between $500-$1500 (filter)
 and RAM is 16GB (filter)
 sorted by price from low to high (sorting)
 showing page 2 (pagination)
 with 20 items per page (pagination)"

Notice how ALL of these are optional\! If the user removes any parameter, the request still works \- it just shows more/different results.

### **Common Mistakes**

**Mistake 1: Putting everything in query params**

```bash
GET /api?resource=users\&id=42
```
Good:
```bash
GET /users/42
```

If it identifies the resource, put it in the path\!

**Mistake 2: Putting optional filters in the path**
```bash

 Bad:
GET /users/active/admin/seattle

Good:

GET /users?status=active\&role=admin\&city=seattle
```
Filters and options belong in query params\!

**Mistake 3: Making paths too deep**

```bash

 Bad:
 GET/companies/123/departments/456/teams/789/employees/111/tasks/222

 Good:

 GET /tasks/222

 Or:

 GET /employees/111/tasks/222


```
Don't make a crazy hierarchy\! Keep it reasonable \- usually 2-3 levels max.
