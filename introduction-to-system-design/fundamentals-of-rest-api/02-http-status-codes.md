---
slug: http-status-codes
title: HTTP Status Codes
readTime: 15 min
orderIndex: 2
premium: false
---

# **🚦  HTTP Status Codes: The Traffic Lights of the Web**

Status codes are the API's way of saying "Here's what happened with your request."

### **The Universal Pattern:**

* **2xx \= Success** ✅ "All good\!"
* **3xx \= Redirect** ↪️ "Go look over there"
* **4xx \= Client Error** 🚫 "You messed up"
* **5xx \= Server Error** 💥 "We messed up"

---

## **✅ 2xx Success: "Mission Accomplished\!"**

### **200 OK \- The Universal Success**

**Restaurant:** "Here's your food, exactly as ordered"

**When you see it:** GET, PUT, PATCH requests succeeded

| GET /posts/42Response: 200 OKBody: { "id": 42, "title": "Hello World" } |
| :---- |

---

### **201 Created \- "Successfully Made\!"**

**Restaurant:** "Your order is placed and here's your receipt number"

**When you see it:** POST request successfully created something

| POST /postsResponse: 201 CreatedLocation: /posts/43Body: { "id": 43, "title": "New Post" } |
| :---- |

**Key Insight:** Often includes a `Location` header pointing to the new resource\!

---

### **204 No Content \- "Done, But Nothing to Show"**

**Restaurant:** "Your order is cancelled" (no food to bring back)

**When you see it:** DELETE succeeded, or update with no content to return

| DELETE /posts/42Response: 204 No Content(empty body) |
| :---- |

---

## **↪️ 3xx Redirects: "Not Here, Try There"**

### **301 Moved Permanently \- "We've Moved\!"**

**Restaurant:** "We permanently moved to a new location"

**Real-world:** A company changes their website address

| GET /old-blogResponse: 301 Moved PermanentlyLocation: /new-blog |
| :---- |

**Browser's reaction:** "I'll automatically go to the new location and remember this forever"

---

### **302 Found (Temporary Redirect)**

**Restaurant:** "We're temporarily serving from the food truck outside"

**Real-world:** Maintenance mode, temporary URL changes

| GET /postsResponse: 302 FoundLocation: /maintenance |
| :---- |

**Browser's reaction:** "I'll go there now, but I'll try the original URL again next time"

---

### **304 Not Modified \- "You Already Have This\!"**

**Restaurant:** "Same menu as yesterday, no need to print a new one"

**Real-world:** Caching \- you already have the latest version

| GET /posts/42If-None-Match: "etag-12345"Response: 304 Not Modified |
| :---- |

**Browser's reaction:** "Great\! I'll use my cached version"

---

## **🚫 4xx Client Errors: "You Did Something Wrong"**

### **400 Bad Request \- "I Don't Understand"**

**Restaurant:** "I can't understand your order"

**Real-world:** Malformed request, invalid JSON

POST /posts

| Body: { "title": }  ← Invalid JSON\!Response: 400 Bad Request{ "error": "Invalid JSON syntax" } |
| :---- |

**Mental model:** Like ordering "one of the purple" \- what are you asking for?

---

### **401 Unauthorized \- "Who Are You?"**

**Restaurant:** "Show me your ID before I serve you alcohol"

**Real-world:** Missing or invalid authentication

| GET /admin/usersResponse: 401 Unauthorized{ "error": "Authentication required" } |
| :---- |

**Common confusion:** Despite the name, 401 means "NOT AUTHENTICATED" (you haven't identified yourself)

---

### **403 Forbidden \- "I Know Who You Are, But You Can't Do This"**

**Restaurant:** "Sorry, this is the VIP section"

**Real-world:** Authenticated but lacking permissions

| DELETE /users/adminResponse: 403 Forbidden{ "error": "Insufficient permissions" } |
| :---- |

**Key difference from 401:**

* 401: "I don't know who you are" → Log in
* 403: "I know who you are, but you're not allowed" → Tough luck

---

### **404 Not Found \- "That Doesn't Exist"**

**Restaurant:** "We don't have that item on the menu"

**Real-world:** Resource doesn't exist

| GET /posts/999999Response: 404 Not Found{ "error": "Post not found" } |
| :---- |

**Most famous code on the internet\!** Everyone's seen this.

---

### **409 Conflict \- "That Creates a Problem"**

**Restaurant:** "You already have an order in progress"

**Real-world:** Duplicate resource, conflicting state

| POST /usersBody: { "email": "existing@email.com" }Response: 409 Conflict{ "error": "Email already exists" } |
| :---- |

---

### **429 Too Many Requests \- "Slow Down\!"**

**Restaurant:** "You're ordering too fast, give us a minute"

**Real-world:** Rate limiting

| GET /api/data (100th request in 1 minute)Response: 429 Too Many RequestsRetry-After: 60{ "error": "Rate limit exceeded" } |
| :---- |

---

## **💥 5xx Server Errors: "It's Not You, It's Us"**

### **500 Internal Server Error \- "We Broke Something"**

**Restaurant:** "The kitchen caught fire"

**Real-world:** Unhandled exception, code crashed

| GET /postsResponse: 500 Internal Server Error{ "error": "An unexpected error occurred" } |
| :---- |

**What to do:** Not your fault\! Report to developers, try again later.

---

### **502 Bad Gateway \- "The Kitchen Isn't Responding"**

**Restaurant:** "Our supplier didn't deliver ingredients"

**Real-world:** Upstream service failed (database down, microservice timeout)

| GET /postsResponse: 502 Bad Gateway |
| :---- |

**Mental model:** Like a chain of dominoes \- one service down affects others.

---

### **503 Service Unavailable \- "We're Temporarily Closed"**

**Restaurant:** "Closed for maintenance"

| Real-world: Planned maintenance, overloaded serverGET /api/dataResponse: 503 Service UnavailableRetry-After: 3600 |
| :---- |

---

## **🎯 Status Code Decision Tree**

**Did the request succeed?** → YES: 2xx (200, 201, 204\)

**NO → Did the client make a mistake?** → YES: 4xx

* Can't understand: 400
* Not logged in: 401
* Logged in but forbidden: 403
* Doesn't exist: 404
* Conflict: 409
* Too many requests: 429

**NO → Did the server make a mistake?** → YES: 5xx

* Server crashed: 500
* Upstream failed: 502
* Temporarily down: 503
