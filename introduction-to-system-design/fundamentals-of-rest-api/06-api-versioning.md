---
slug: api-versioning
title: API Versioning
readTime: 15 min
orderIndex: 6
premium: false
---


# **API Versioning: Planning for Change**


Let’s see why versioning matters.

### **The Breaking Change Disaster**

Imagine you built a successful API. You have 1000 mobile apps using it. Your API returns user data like this:

Old API:
```bash
 GET /user/123
 ```

```json
 {  "name": "John Doe",  "email": "john@example.com"}
 ```


One day, you realize "Wait\! We need separate first and last names\!" So you change it to:

New API:
```bash
 GET /user/123
 ```
```json
 {  "first_name": "John",  "last_name": "Doe",  "email": "john@example.com"}
 ```

**What happens?**

All 1000 apps crash instantly\! They're expecting `name`, but now there's `first_name` and `last_name`. Their code breaks:

// Old app code
```javascript

console.log(user.name)  // undefined\! The field doesn't exist anymore\!
```

This is called a **breaking change**, and it's a disaster\!

### **The Solution: API Versioning**

Versioning is like saying "Hey, I'm making changes, but I'll keep the old version running for those who need it."

Let’s see  the right way:

 Version 1 (Old apps use this):
```bash
 GET /v1/user/123
 ```

 ```json

 {  "name": "John Doe",  "email": "john@example.com"}
 ```

 Version 2 (New apps use this):
 ```bash

 GET /v2/user/123
 ```
 ```json

 {  "first_name": "John",  "last_name": "Doe",  "email": "john@example.com"}

```

Now:

* Old apps keep working (using v1)

* New apps use the better format (using v2)

* You can gradually migrate users from v1 to v2

* Eventually, you deprecate v1 (after giving warning)

### **Method 1: URL Path Versioning (The Most Common)**

This is what most APIs do. The version is right in the URL:
```bash
GET /v1/users

GET /v2/users

GET /v3/users
```

Let’s  walk  through a real scenario:

Your Startup's Journey:

━━━━━━━━━━━━━━━━━━━━━

| 2023: Launch API v1
```bash
GET /v1/products
```

Returns:
```json
{"id": 1, "price": 19.99}
```

2024: Want to support multiple currencies

Launch API v2
```bash

GET /v2/products
```

Returns:
```json
{"id": 1, "price": {"amount": 19.99, "currency": "USD"}}
```

Keep v1 running for existing users

2025: Want to add tax calculation

Launch API v3
```bash
GET /v3/products
```
```json
 {        "id": 1,
           "price":
   {          "amount": 19.99,

               "currency": "USD",

               "tax": 1.60,

               "total": 21.59        }      }
```


2026: Announce v1 deprecation

"v1 will shut down in 6 months"

Give users time to upgrade

2026 (6 months later):

Turn off v1

Now only v2 and v3 are running

**Pros:**

* Very clear which version you're using

* Easy to test different versions

* Can run multiple versions simultaneously

**Cons:**

* URLs change when version changes

* Need to maintain multiple codebases

### **Method 2: Header Versioning (The Clean URL Approach)**

Some APIs keep the URL clean and put the version in headers:
```bash

GET /users
```

Accept: application/vnd.myapi.v1+json

vs
```bash

GET /users
```

Accept: application/vnd.myapi.v2+json

Let’s see  a conversation:

Client → Server

━━━━━━━━━━━━━━━

```bash
 "GET /products
 ```

 ```json

 Server: "You want v2? Here you go:"

 {          "id": 1,          "price": {            "amount": 19.99,           "currency": "USD"          }        }

```

Same URL, different version\!

Pros:

* Clean URLs (always just `/products`)

* RESTful purists prefer this

* Clients explicitly request version

Cons:

* Harder to test (can't just paste URL in browser)

* Not visible in logs easily

### **Method 3: Query Parameter Versioning**


```bash

GET /users?version=1
```

```bash
GET /users?version=2
```

This is less common but simple.

### **What Should Change Between Versions?**

 The rule of thumb:

**Breaking Changes (need new version):**

 Removing a field
 ```json

 v1: {"name": "John"}

 v2: {} ← Missing field\! BREAKING\

 ```

 Changing field type
 ```json
 v1: {"age": "25"}
 //returns age as string
 ```
 ```json

v2: {"age": 25}
//returns age as number

```
BREAKING

Changing field name
```json

v1: {"user_name": "john"}

v2: {"username": "john"}
```

BREAKING

❌ Removing an endpoint
```bash
v1: DELETE /users/123 exists


v2: DELETE /users/123 removed  ← BREAKING
```


**Non-Breaking Changes (same version):**

 ✅ Adding optional fields

v1: {"name": "John"}

v1.1: {"name": "John", "nickname": "Johnny"}

← Extra field, OK

Adding new endpoints

v1: GET /users exists

v1.1: GET /users/123/posts added  ← New endpoint, OK\

✅ Making required field optional

v1: email required

v1.1: email optional

More flexible, OK\

✅ Fixing bugs (that don't change contract)

v1: Returns wrong calculation

v1.1: Returns correct calculation  ← Same format, OK\! |

### **Real Example: Stripe's API**

Let’s see how a professional API (Stripe) does versioning:

Stripe's approach:

━━━━━━━━━━━━━━━━━

Every API call includes version:

```bash
POST /v1/charges
```

Stripe-Version: 2023-10-16

Stripe has DATED versions:

\- 2023-10-16

\- 2024-01-15

\- 2024-06-20

When you create an account, you're locked to that version.
You can upgrade when ready\!

### **The Deprecation Dance**

Here's how to retire an old version responsibly:

Step 1: Announce deprecation (12 months ahead)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```bash
GET /v1/users
```
Response:

Warning: API v1 is deprecated. It will be removed on 2026-10-19.

Please migrate to v2:

https://docs.myapi.com/v2\-migration

Step 2: Remind users (6 months ahead)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```bash
 GET /v1/users
```

 Response:Warning: API v1 will be removed in 6 months

 Only 20% of users have migrated. Start now

Step 3: Final warning (1 month ahead)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

```bash
 GET /v1/users
 ```

 Response:

 URGENT: API v1 will be removed in 30 days

 Migrate to v2 immediately

Step 4: Shutdown

━━━━━━━━━━━━━━━

```bash
 GET
 /v1/users
```
 Response:410 Gone
 ```json

 {  "error": "API v1 is no longer available",  "message":

 "Please use v2: https://api.myapi.com/v2/users"}
 ```
