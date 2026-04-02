---
slug: client-server-architecture
title: Client Server Architecture
readTime: 10 min
orderIndex: 1
premium: false
---

# Client Server Architecture




## **👥 Client-Server Architecture: The Restaurant Revisited**

Remember our restaurant analogy from the REST API section? Let's dive deeper into the fundamental architecture pattern that powers the entire internet.

### **The Historical Context**

Let’s look at history to understand why client-server architecture exists.

**Before Client-Server (1970s): Mainframe Era**

The Old Way:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544683/307_qmabqw.png)

Terminals: Just keyboards and screens

- No processing power

- No storage

- Just display what mainframe sends

Problem:

❌ Expensive mainframe does EVERYTHING

❌ Terminals are "dumb" (can't do anything alone)

❌ No mainframe \= no work for anyone

**Client-Server Revolution (1980s-Present)**

The New Way:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544683/306_npmmv6.png)

Clients: Smart devices

✓ Can process data locally

✓ Have their own storage

✓ Work independently (sometimes)

Server: Centralized resource

✓ Manages shared data

✓ Coordinates between clients

✓ Enforces business rules

### **Understanding the Roles**

**The Client's Job:**

What Clients Do:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. User Interface

   - Display information

   - Accept user input

   - Validate forms

2. Request Making

   - Send requests to server

   - Handle responses

   - Manage errors

3. Local Processing

   - Format data for display

   - Client-side validation

   - Caching for speed

4. State Management

   - Remember user preferences

   - Track session

   - Store temporary data

Example: Your Web Browser
![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544684/310_jt32bo.png)

**Connection to TCP and HTTP:** When your browser (client) makes a request, it uses HTTP over TCP. The TCP 3-way handshake establishes the connection, then HTTP request/response flows over that reliable TCP connection. The browser is the client, the web server is the server (we have already discussed these concepts in detail before)

**The Server's Job:**

What Servers Do:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Data Management

   - Store in database

   - Retrieve data

   - Update records

2. Business Logic

   - Process payments

   - Validate rules

   - Calculate results

3. Security

   - Authentication

   - Authorization

   - Encryption

4. Coordination

   - Handle multiple clients

   - Manage concurrency

   - Ensure consistency

Example: Facebook's Server

![img4](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544684/311_v8bb8j.png)

### **Different Flavors of Client-Server**

**Flavor 1: Thin Client (Web Apps)**

Thin Client Architecture:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img5](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544683/318_gghhmp.png)

User clicks "Send Email":

1. Browser: "POST /send" → Server

2. Server: Validates email, sends it, updates database

3. Server: Generates new HTML

4. Browser: Displays new HTML

Almost everything happens on server!

**Flavor 2: Thick Client (Desktop Apps)**

Thick Client Architecture:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img6](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544683/308_zc2pss.png)
Example: Microsoft Word with OneDrive

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User types document:

1. Word: Processes locally (spell check, formatting)

2. Word: Auto-saves to local disk

3. Word: Syncs to OneDrive every few minutes

4. Server: Just stores the file

Most work happens on client!

**Flavor 3: Hybrid (Modern Web Apps/SPAs)**

Hybrid Architecture (Best of Both):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img7](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544685/317_bzn2cs.png)

Example: Modern Gmail

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User clicks "Send Email":

1. React app: Validates form locally ✓

2. React app: Shows "Sending..." immediately

3. React app: POST /api/send → Server

4. Server: Actually sends email, updates DB

5. Server: Returns JSON {success: true}

6. React app: Updates UI (smooth transition)

Work split intelligently between client and server!

### **Real-World Example: Netflix**

Let me show you how Netflix uses client-server architecture:

Netflix Architecture:

The Flow:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Client (TV): "Show me action movies"
   → Server: Returns personalized list

2. Client: User clicks "Play"
   → Server: "Here's the video URL"

3. Client: Connects to nearby CDN
   ← CDN: Streams video chunks

4. Client: Handles playback, buffering, quality
   (Smart client does a lot!)

5. Client: Periodically reports back
   → Server: "User watching minute 24"
   (For "Continue Watching" feature)

![img8](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544683/309_cn07a0.png)

### **The Communication Patterns**

**Pattern 1: Request-Response (Synchronous)**

Classic Web Request:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
![img9](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544684/313_wqui17.png)

Timeline: \~100ms
Client blocks waiting for response

**Connection to Our Previous Topics:**

* This is **synchronous communication** (we covered this earlier!)

* Uses **HTTP methods** (GET in this example)

* Can return various **status codes** (200 OK)

* May include **authentication** (JWT or session)

**Pattern 2: Long Polling**

Chat Application:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img10](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544684/312_dfjmsh.png)

Server holds connection open until data available

**Pattern 3: WebSockets (Bidirectional)**

Real-Time Chat:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img11](https://res.cloudinary.com/dretwg3dy/image/upload/v1766546133/web_sockets_oykqms.png)

Persistent connection, messages flow both ways
No request-response limitation!

**Pattern 4: Server-Sent Events (SSE)**

Live Stock Prices:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img12](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544684/314_ncbspc.png)

Server pushes updates to client continuously
Client doesn't need to request each time

### **Security in Client-Server**

**The Trust Boundary**

The Rule:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img13](https://res.cloudinary.com/dretwg3dy/image/upload/v1766544684/315_qvlopz.png)

Bad Example (Trusting Client):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Client: "I'm paying $10 for this $100 item"

Server: "Okay!" ❌

Result: Business loses money!

Good Example (Validate on Server):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Client: "I'm buying item\_id: 42"

Server: Looks up item in database

Server: "That's $100"

Server: Validates payment for $100 ✓

Result: Business is protected!

**Validation Rules:**

Client-Side Validation:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Purpose: User experience

- Immediate feedback

- Prevent typos


- Guide user

Example:
"Email must contain @"
"Password needs 8 characters"

BUT: Can be bypassed!

Server-Side Validation:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Purpose: Security & data integrity

- Enforce business rules

- Prevent attacks

- Ensure data quality

Example:

Check email actually exists in database

Verify password against hash

Validate payment with bank

REQUIRED: Cannot be bypassed!


The Pattern:

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Validate on client (UX)

2. Validate on server (Security)

3. Never trust client alone!
