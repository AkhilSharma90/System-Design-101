---
slug: api-pagination
title: API Pagination
readTime: 15 min
orderIndex: 5
premium: false
---

#  Pagination: Handling Large Request Resources Like a Pro

Okay, imagine someone asked you to send them a list of every person in your city. You'd probably say "That's impossible\! There are too many\!"

That's exactly the problem pagination solves.

### **The Phone Book Story**

Remember old phone books? (Okay, maybe you don't, but work with me here\!) Phone books had thousands of listings. Did they print one giant list? No\! They:

1. Organized alphabetically
2. Added page numbers
3. Let you jump to sections (A-D, E-H, etc.)

**That's pagination\!**

### **Why Do We Need Pagination?**

Let's see this instance. You're building Instagram's API. Someone requests:

```bash
GET /posts
```

Your database has 500 MILLION posts.

Without pagination, you'd try to send:
```json

{
  "posts": \[
    {...}, {...}, {...}, ... (500 million objects)
  \]
}
```

Problems:

❌ Takes 10 minutes to load

❌ Uses 50 GB of bandwidth

❌ Crashes the user's app

❌ Kills your server

❌ Costs you thousands in bandwidth fees

**Solution: Only send a small "page" at a time\!**

### **Method 1: Offset Pagination (The Page Number System)**

This is the most intuitive method. It's like reading a book:

Page 1: Items 1-10

Page 2: Items 11-20

Page 3: Items 21-30

...and so on

Let’s see how this looks in an API:

Request for Page 1:
━━━━━━━━━━━━━━━━━━

```bash
 GET /posts?page=1\&limit=10
 ```
 Server thinks:"Page 1, limit 10... let me calculate:

 Offset \= (page \- 1\) × limit \= (1 \- 1\) × 10 \= 0

 So, fetch items from position 0 to 9"

 ```sql

 SQL: SELECT \* FROM posts LIMIT 10 OFFSET 0

 ```
 Response:
 ```json
 {  "posts": \[...10 posts...\],  "pagination": {    "current\_page": 1,    "total\_pages": 50000,    "total\_items": 500000,    "per\_page": 10  }} |
 ```

Now, the user wants page 2:

Request for Page 2:
━━━━━━━━━━━━━━━━━━
```bash
 GET /posts?page=2\&limit=10
 ```

Server calculates:

Offset \= (2 \- 1) × 10 \= 10

So, fetch items from position 10 to 19

```sql

SQL: SELECT \* FROM posts LIMIT 10 OFFSET 10
```

Response:
```json
{  "posts": [...10 different posts...],  "pagination": {   "current_page": 2,    "total_pages": 50000,    "total_items": 500000,   "per_page": 10  }}

```

### **Let’s see  a Complete Example**

Imagine you're building a blog API with 243 posts. A client wants 20 posts per page:

First Request:

```bash
 GET /posts?page=1\&limit=20
```


Your API returns:
```json

{  "data":

  {"id": 1, "title": "First Post"},    {"id": 2, "title": "Second Post"},    ... (18 more posts)  ],  "pagination": {    "page": 1,    "limit": 20,    "total_items": 243,    "total_pages": 13,  ← Math.ceil(243 / 20\) \= 13   "has\_next": true,   "has\_previous": false  },  "links": {    "first": "/posts?page=1\&limit=20",    "next": "/posts?page=2\&limit=20",   "last": "/posts?page=13\&limit=20"  }}
```
Notice how we are  giving the client helpful info:

* How many total items exist

* How many pages there are

* Whether there's a next/previous page

* Direct links to navigate

### **The Problem with Offset Pagination**

Let’s see why this method has issues. Imagine you're on page 5, looking at posts:

Page 5 shows posts 41-50

While you're reading, someone deletes post \#35

Now:

Post \#41 becomes post \#40

Post \#42 becomes post \#41


When you click "Next" to page 6:

```bash

GET /posts?page=6\&limit=10

```

(Expecting to see posts 51-60)

But now you see posts 50-59\!

You MISSED post \#50 because everything shifted\!

This is called the "shifting data problem."

### **Method 2: Cursor Pagination (The Bookmark System)**

Cursor pagination is like using a bookmark. Instead of remembering page numbers, you remember "where you left off."

Let see how:

First Request:
━━━━━━━━━━━━━━
```bash
 GET /posts?limit=10
```

 Response:
 ```json
 {  "data": [    {"id": 101, "title": "Post 1", "created\_at": "2025-10-15"},    {"id": 102, "title": "Post 2", "created\_at": "2025-10-16"},    ... (8 more posts)    {"id": 110, "title": "Post 10", "created\_at": "2025-10-19"}  \],  "pagination": {    "next\_cursor": "eyJpZCI6MTEwfQ==",  ← This is your bookmark\!    "has\_more": true  }} |
 ````


That `next_cursor` is encoded data saying "you left off at ID 110"

Second Request:
━━━━━━━━━━━━━━━
```bash

 GET /posts?limit=10\&cursor=eyJpZCI6MTEwfQ==

```
Server decodes cursor: "They left off at ID 110"
```sql

SQL: SELECT * FROM posts WHERE id > 110 LIMIT 10

```

```json
Response:
{  "data": \[    {"id": 111, "title": "Post 11"},    {"id": 112, "title": "Post 12"},    ... (8 more posts)    {"id": 120, "title": "Post 20"}  \],  "pagination": {    "next\_cursor": "eyJpZCI6MTIwfQ==",    "previous\_cursor": "eyJpZCI6MTExfQ==",    "has\_more": true  }}
```

### **Why Cursor Pagination Solves the Shifting Problem**

The same deletion scenario:

You're looking at items with cursor pointing to ID 110

Someone deletes item ID 35

Next request:

```bash
GET /posts?cursor=eyJpZCI6MTEwfQ==
```
Server: "Give me items AFTER ID 110"

Result: You still get items 111-120
Nothing shifted because you're using absolute identifiers\!

### **Real-World Example: Social Media Feed**

Think about your Instagram/Twitter feed. Let me walk you through what happens:

You open the app:
━━━━━━━━━━━━━━━━━

 App:
 ```bash

 GET  /feed?limit=20
 ```

 Server returns:
 ```json
 { "posts": [{"id": "post_999", "content": "..."},   {"id": "post_998", "content": "..."},    ... (18 more)    {"id": "post_980", "content": "..."}  \],  "next\_cursor": "post_980"}
 ```

You scroll down:
━━━━━━━━━━━━━━━━━
App:

```bash
GET /feed?limit=20\&cursor=post\_980
```


Server: "Show 20 posts AFTER post\_980"

Even if:
\- New posts are added
\- Posts are deleted
\- Posts are edited

You'll never see duplicates or miss posts\!

### **Which Pagination Method Should You Use?**

Let see a pagination technique:

**Use Offset (page numbers) when:**

* Users need to jump to specific pages (e.g., search results)
* Total count is important
* Data doesn't change frequently
* Example: Product catalogs, blog archives

**Use Cursor (bookmarks) when:**

* Data changes frequently
* Users mostly scroll forward
* Exact position isn't important
* Example: Social feeds, notifications, real-time data
