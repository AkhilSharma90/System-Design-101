---
slug: back-of-envelope-design
title: Back of Envelope Estimations
readTime: 20 min
orderIndex: 2
premium: false
---

# **Back of the Envelope Calculations - The Estimation Game**

### **🎯 Challenge 2: The Impossible Interview Question**

**Interviewer asks:** "Design a system that can handle WhatsApp-scale messaging. How much storage do you need?"

**Your first reaction:** "Umm... a lot? Maybe... 100 terabytes? Or is it petabytes? I have no idea! 😰"

**Pause and think:** How can you go from complete confusion to a reasonable estimate in 2 minutes?

**The Answer:** Back of the envelope calculations! These are quick, rough estimates that get you in the right ballpark.

**Key insight:** You don't need exact numbers. Being within 10x is often good enough for design decisions!

---

### **🧮 Interactive Exercise: The Napkin Math Foundations**

Before we dive in, let's build your mental toolkit with numbers you should memorize:

📏 POWERS OF 10 (Your new best friends)

──────────────────────────────────────

Thousand    \=        1,000 \= 10³   \= 1 KB

Million     \=    1,000,000 \= 10⁶   \= 1 MB

Billion     \=1,000,000,000 \= 10⁹   \= 1 GB

Trillion    \=        10¹²          \= 1 TB

⏱️ TIME CONVERSIONS

──────────────────────────────────────

1 day       \= 86,400 seconds   ≈ 100K seconds (rounded!)

1 month     ≈ 2.5 million seconds

1 year      ≈ 31 million seconds (remember: π × 10⁷ seconds!)


💾 DATA SIZES (rough estimates)

──────────────────────────────────────

Single character \= 1 byte

Short tweet \= 280 bytes ≈ 300 bytes

Typical web page \= 100 KB

YouTube video (1 min) \= 10 MB

High-res photo \= 2 MB

Movie (1080p, 2hrs) \= 4 GB

**Mental Model:** These rounded numbers make math easy! 86,400 seconds/day becomes 100K seconds. Close enough for estimates!

---

### **🎪 Real-World Challenge: Estimating Twitter's Storage**

**Scenario:** You're asked: "How much storage does Twitter need per year?"

**Step-by-step thinking:**

🤔 STEP 1: Break down the problem

──────────────────────────────────────

What data does Twitter store?

✓ Tweets (text)

✓ Images

✓ Videos

✓ User profiles

Let's estimate each!

📝 STEP 2: Make assumptions (and state them!)

──────────────────────────────────────

Daily active users: 200 million

Tweets per user per day: 2

Average tweet size: 300 bytes

20% of tweets have images (2 MB each)

5% of tweets have videos (10 MB each)

🧮 STEP 3: Calculate step by step

──────────────────────────────────────


Tweets per day:

200 million users × 2 tweets \= 400 million tweets/day


Text storage per day:

400 million tweets × 300 bytes \= 120 GB/day

Image storage per day:

400 million tweets × 20% × 2 MB \= 160 TB/day

Video storage per day:

400 million tweets × 5% × 10 MB \= 200 TB/day

TOTAL per day: 120 GB \+ 160 TB \+ 200 TB ≈ 360 TB/day

📅 STEP 4: Extrapolate to yearly

──────────────────────────────────────

360 TB/day × 365 days ≈ 131 PB/year

Add 20% for replication and backups:
131 PB × 1.2 ≈ 157 PB/year

**Final Answer:** "Twitter needs approximately **150-200 petabytes** of storage per year."

**Reality check:** Twitter's actual numbers are in this ballpark! You got it right with just napkin math! 🎉

---

### **🚰 Problem-Solving Exercise: Netflix Bandwidth**

**Challenge:** "How much internet bandwidth does Netflix need?"

**Think about it before looking at the solution...**

What information do you need?

* How many people stream at once?
* What video quality?
* How big is a video stream?

**Solution walkthrough:**

📊 ASSUMPTIONS

──────────────────────────────────────

Netflix subscribers: 250 million

Concurrent viewers (peak time): 10% \= 25 million

Average bitrate (1080p video): 5 Mbps

🔢 CALCULATION

──────────────────────────────────────

Bandwidth needed:

25 million viewers × 5 Mbps \= 125 million Mbps

\= 125,000 Gbps

\= 125 Tbps (terabits per second)

🎯 REALITY CHECK

──────────────────────────────────────

Netflix actually uses 100-200 Tbps during peak hours!
Our estimate was spot on! 🎯

**Mental Model:** Don't aim for perfection. Aim for "right order of magnitude."

If your answer is:

* ✅ 100-200 Tbps → Excellent estimate!
* ✅ 50-500 Tbps → Good ballpark
* ❌ 1 Tbps or 10,000 Tbps → Off by too much

---

### **🎯 The 5-Step Formula: Your Estimation Superpower**

**Use this every time:**

STEP 1: 🎯 CLARIFY & SCOPE

└─ What exactly are we estimating?

└─ What's included? What's excluded?

STEP 2: 📝 STATE ASSUMPTIONS

└─ Number of users

└─ Usage patterns

└─ Data sizes

└─ Write them down!

STEP 3: 🧮 CALCULATE IN CHUNKS

└─ Break into smaller pieces

└─ Estimate each piece

└─ Round numbers aggressively

STEP 4: 📊 SANITY CHECK

└─ Does this make sense?

└─ Compare to known systems

└─ Am I off by 1000x?

STEP 5: 🎁 ADD BUFFERS

└─ 2x for replication

└─ 1.5x for growth

└─ Better to overestimate

---

### **🎮 Practice Game: Quick Estimates!**

**Try these yourself before looking at answers:**

**Challenge A:** "How many queries per second does Google Search handle?"

**Challenge B:** "How much storage for 1 year of Zoom meeting recordings?"

**Challenge C:** "How many servers to handle Instagram's image uploads?"

**Think through each one...**

---

**ANSWERS:**

**Challenge A: Google Search QPS**

Assumptions:

- Google users worldwide: 4 billion

- Searches per person per day: 3

- Peak traffic: 2x average

Calculation:

Daily searches: 4B × 3 \= 12 billion

Per second: 12B ÷ 100K seconds ≈ 120,000 QPS

Peak: 120K × 2 \= 240,000 QPS

Answer: \~200,000-300,000 queries per second

Reality: Google handles \~100,000 QPS average, \~200,000 peak ✓


**Challenge B: Zoom Storage**

Assumptions:

- Daily meeting hours: 3 billion minutes

- Video bitrate: 2 Mbps

- Storage with compression: 1 MB/minute

Calculation:

Daily storage: 3B minutes × 1 MB \= 3 PB/day

Yearly: 3 PB × 365 \= 1,095 PB ≈ 1 exabyte

Answer: \~1 exabyte per year

**Challenge C: Instagram Servers**

Assumptions:

- Photos uploaded per day: 100 million

- Upload takes 2 seconds per photo

- Server handles 1000 req/sec

Calculation:

Total upload seconds per day:

100M photos × 2 sec \= 200M seconds

Servers needed:

200M seconds ÷ 86,400 seconds/day \= 2,315 server-days

To handle in 24 hours: 2,315 servers

Add 2x for peak load: \~5,000 servers

Answer: \~5,000 servers for uploads

---

### **💡 Common Pitfalls & How to Avoid Them**

❌ PITFALL 1: Trying to be too precise

└─ "It's exactly 247,394 requests per second"

└─ Fix: Round! "About 250K requests/second"

❌ PITFALL 2: Forgetting peak vs. average

└─ "We need 10 servers for average load"

└─ Fix: Always plan for 2-3x peak load

❌ PITFALL 3: Not stating assumptions

└─ Just writing numbers without context

└─ Fix: Always say "Assuming X users..."

❌ PITFALL 4: Skipping sanity checks

└─ Calculating you need 1 million servers

└─ Fix: "Wait, that's more than Google. Recheck!"

❌ PITFALL 5: Analysis paralysis

└─ Spending 20 minutes on perfect numbers

└─ Fix: 2-5 minutes max. Rough is good enough!

---

### **🎯 Quick Reference: Estimation Cheat Sheet**

COMMON BENCHMARKS TO REMEMBER

═════════════════════════════════════

📱 Typical smartphone:

   Storage: 128 GB

   RAM: 6 GB

   Screen size: 6 inches, 2400×1080

💻 Typical web server:

   Handles: 1000-5000 requests/sec

   RAM: 16-64 GB

   Disk: 500 GB - 2 TB

🌐 Web page load:

   Size: 2 MB average

   Time: 2-3 seconds

   50-100 HTTP requests

📹 Video streaming:

   480p: 1 Mbps

   720p: 2.5 Mbps

   1080p: 5 Mbps

   4K: 25 Mbps

👥 Social media usage:

   Average user posts: 1-2 per day

   Average user views: 100-200 items/day

   Active time: 30-60 minutes/day

🗄️ Database queries:

   Simple SELECT: 1 ms

   JOIN query: 10-100 ms

   Full table scan: seconds to minutes

---

### **🎪 The Grand Finale: Complex Estimation**

**Ultimate Challenge:** "Design YouTube's infrastructure. Estimate everything."

**Your approach:**

🎯 SCOPE

─────────

What to estimate:

✓ Storage (videos)

✓ Bandwidth (uploads \+ downloads)

✓ Servers (processing)

✓ Database (metadata)

📝 ASSUMPTIONS

──────────────

Daily active users: 2 billion

Videos watched per user: 10

Average video: 5 minutes, 1080p

Upload/watch ratio: 1:1000

New videos per day: 500,000

💾 STORAGE CALCULATION

──────────────────────

Video size: 5 min × 5 Mbps ÷ 8 \= 3.75 MB/sec × 300 sec \= 1.1 GB


New storage per day:

500K videos × 1.1 GB \= 550 TB/day

Yearly growth: 550 TB × 365 \= 200 PB/year

🌐 BANDWIDTH CALCULATION

────────────────────────

Views per day: 2B users × 10 videos \= 20B views

Data downloaded per day:

20B × 1.1 GB \= 22,000 PB \= 22 EB/day

Per second: 22 EB ÷ 100K sec \= 220 TB/sec \= 1,760 Tbps

⚙️ SERVER CALCULATION

──────────────────────

Video encoding:

500K videos/day need transcoding

Each video takes 10 minutes to encode

One server encodes 144 videos/day (24 hrs)

Servers needed: 500K ÷ 144 \= 3,500 encoding servers

🎯 FINAL ANSWER

───────────────

Storage: 200 PB new per year (+ historical data)

Bandwidth: \~2,000 Tbps (petabits per second!)

Encoding servers: \~4,000

Streaming servers: \~50,000 (estimated separately)

Reality: YouTube stores \~1 exabyte total, uses \~1,000 Tbps

Our estimates are in the right ballpark! 🎉


---

## Key Takeaways

1. **Back-of-envelope calculations give you order-of-magnitude estimates in minutes** — you don't need exact numbers, just the right ballpark
2. **Know your powers of 2** — 2^10 ≈ 1K, 2^20 ≈ 1M, 2^30 ≈ 1B, 2^40 ≈ 1T
3. **Convert daily numbers to per-second** — divide by 86,400 (seconds/day) or approximate: 1M/day ≈ 12/sec
4. **Storage estimation: multiply users × data per user** — 1 billion users × 1 KB each = 1 TB
5. **Always state your assumptions clearly** — the estimation process matters more than the exact answer
