---
slug: hld-lld
title: High Level and Low Level Design
readTime: 10 min
orderIndex: 1
premium: false
---

## **HLD vs LLD \- The Architect's Tale**

### **🎯 Challenge 1: The House Building Dilemma**

**Imagine this scenario:** You want to build your dream house. You meet with your builder and say: "I want a 3-bedroom house with a modern kitchen."

**Pause and think:** Would you expect the builder to start pouring concrete immediately? Or would you need something in between?

**The Two-Level Reality:** Just like building a house needs both an architectural blueprint AND detailed construction plans, software systems need:

* **High-Level Design (HLD)** \- The "what" and "why"
* **Low-Level Design (LLD)** \- The "how" in detail

**Key Insight:** You can't jump from idea to implementation without both levels of design\!

---

### **🖼️ Interactive Exercise: The Restaurant Analogy**

**Scenario:** You're opening a restaurant. Let's see the difference between HLD and LLD thinking:

**High-Level Design (HLD) Thinking:**

Question: "What kind of restaurant are we building?"

Your HLD answers:

✓ Italian restaurant

✓ Seats 50 customers

✓ Has kitchen, dining area, bar

✓ Takes reservations online

✓ Accepts credit cards

✓ Open 5pm-11pm

**Low-Level Design (LLD) Thinking:**

Question: "HOW exactly will online reservations work?"

Your LLD answers:

✓ Use PostgreSQL database with 'reservations' table

✓ Table has columns: id, customer\_name, phone, date, time, party\_size

✓ API endpoint: POST /api/reservations

✓ Validation: Check if time slot available before confirming

✓ Send confirmation email using SendGrid API

✓ Handle double-booking with database transaction locks

**Mental Model:**

* **HLD** \= The restaurant floor plan showing "Kitchen here, tables there, bar over here"
* **LLD** \= The detailed electrical wiring diagram showing "outlet here, wire gauge 14AWG, circuit breaker 20A"

---

### **🎨 Visual Comparison: Building a Social Media App**

Let's design a Twitter-like app to see both levels in action:

#### **HIGH-LEVEL DESIGN (30,000 foot view)**

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1764645948/236_xjdj2c.png)

Components: User Service, Tweet Service, Database
Data Flow: User → Posts Tweet → Stored in DB
Tech Stack: Node.js backend, React frontend, MongoDB

**HLD answers:**

* What are the main components? (User Service, Tweet Service, Database)
* How do they interact? (API calls between services)
* What technologies? (Node.js, MongoDB, React)

#### **LOW-LEVEL DESIGN (Ground level view)**

**Just for the "Post Tweet" feature:**
```js
| // LLD specifies the EXACT implementation
class TweetService {
  async postTweet(userId, content, mediaUrls) {
      // Step 1: Validate input
      if (!content || content.length \> 280) {

        throw new ValidationError("Invalid tweet length");
        }
      // Step 2: Check user exists and is not banned

      const user = await UserDB.findById(userId);

      if (!user || user.status === 'banned') {



      throw new UnauthorizedError();
      }
      // Step 3: Create tweet object
      const tweet = {
        id: generateUUID(),
        userId: userId,
        content: sanitizeHTML(content),
        mediaUrls: mediaUrls,
        timestamp: Date.now(),
        likes: 0,
        retweets: 0
        };
      // Step 4: Save to database
      await TweetDB.insert(tweet);
      // Step 5: Update user's timeline cache
      await TimelineCache.addTweet(userId, tweet);
      // Step 6: Fan out to followers
      await FanoutService.distributeToFollowers(userId, tweet);
      return tweet;
    }
  }
   Database Schema (LLD specifies exact structure)

   TweetsCollection {
   _id: ObjectId,
   userId: ObjectId (indexed),
   content: String (max 280 chars),
   mediaUrls: Array<String>,
   timestamp: ISODate (indexed),
   likes: Number (default 0),
   retweets: Number (default 0),
   hashtags: Array<String> (indexed)
   }
   ```

**LLD answers:**

* Exact function signatures
* Data validation logic
* Error handling
* Database schema with indexes
* Algorithm for each operation
* Edge cases handling

---

### **🚨 Common Misconception: "Can't I Just Code Without All This Design?"**

**You might think:** "This seems like extra work. Why not just start coding?"

**The Reality Check:**

Imagine building IKEA furniture:

* **Without HLD:** You don't look at the picture of the finished product → Build random parts → End up with a chair that looks like a table
* **Without LLD:** You see the picture but skip the step-by-step instructions → Parts don't fit → Screws in wrong places → Wobbly furniture

**Real-world comparison:**

No HLD \= Building a house without a floor plan
         Result: Bathroom ends up in the middle of the living room

No LLD \= Having the floor plan but no construction details
         Result: You know WHERE the bathroom goes, but not HOW to
         install the plumbing, what pipe sizes, what materials

**The Sweet Spot:**

1. **HLD first** → Make sure you're building the RIGHT thing
2. **LLD next** → Make sure you're building it the RIGHT way
3. **Code last** → Implementation is now straightforward

---

### **🎮 Decision Game: Which Level of Design?**

**For each statement, identify if it's HLD or LLD:**

**A.** "We'll use Redis for caching frequently accessed data"

**B.** "The cache key will be `user:{userId}:profile` with TTL of 3600 seconds"

**C.**
"The system will have three microservices: Auth, Orders, and Payments"

**D.**
"The Order service will use a state machine with states: PENDING → CONFIRMED → SHIPPED → DELIVERED"


**E.** "We'll use PostgreSQL for persistent storage"

**F.** "The users table will have a B-tree index on email column and a composite index on (last_name, first_name)"






**Think about each one... What level of detail is being specified?**

**Answers revealed:**

* **A: HLD** → Mentions technology choice, but not implementation details
* **B: LLD** → Specific key format, exact TTL value
* **C: HLD** → System architecture, component breakdown
* **D: LLD** → Exact state transitions and values
* **E: HLD** → Database technology choice
* **F: LLD** → Specific index types and columns

**Mental Model:**

* If your manager or client could understand it → **HLD**
* If only developers need this level of detail → **LLD**

---

### **📊 The Complete Picture: HLD vs LLD**

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1764645948/238_ybixwn.png)

                  LLD (Engineer)

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1764645947/237_ixdg9l.png)


---

### **🔑 Key Takeaway: The Two-Level Strategy**

**When designing a system:**

1. **Start with HLD**

   * Sketch the big picture
   * Identify major components
   * Choose technologies
   * Define interactions
   * Get stakeholder buy-in
2. **Drill down to LLD**

   * Design each component in detail
   * Specify algorithms
   * Define data structures
   * Plan error handling
   * Ready for coding

**Real-world parallel:**

* Planning a road trip: HLD \= Route from NYC to LA, stop in Chicago and Denver
* LLD \= Exact highways (I-80 West), exit numbers (Exit 126B), gas stations every 200 miles
