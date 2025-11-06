---
slug: basics-of-data-storage
title: Basics of Storage
readTime: 20 min
orderIndex: 1
premium: false
---




#  **Storage Fundamentals: Understanding Where Your Data Lives**

Let’s see what happens at every growing tech company.

Your startup is doing great. You've got 10,000 users, and they're uploading photos, videos, documents—everything. One morning, your CTO walks in and says: "We're spending $50,000 a month on storage. Why is it so expensive? Can we fix this?"

You look at the bill and realize: you've been storing everything the same way. Profile pictures from 2015 that nobody looks at? Same expensive, fast storage as today's viral videos. User documents? Same storage as your database that needs millisecond access times.

This is like storing your winter coat, your daily clothes, and your favorite t-shirt all in the same place—hanging in your closet. Sure, it works, but it's not optimal.

Today, you’ll learn about about storage—the different types, how they work, and most importantly, how to choose the right one for your data. By the end, you'll understand why some data belongs on $0.023/GB storage and other data needs $0.50/GB storage.

Let's dive in.

Actually before we dive we need to learn a few terms which

Will notoriously appear again and again in the explanations

i) AWS is Amazon Web Services is the Cloud Service Provided by Amazon

ii) Azure is Microsoft’s Cloud Service

iii) GCP is Google’s Cloud Service (Google Cloud Service)

iv) We are also going to use lot of terms like EBS,EC2,S3,Glacier, EFS,CDN you can come back and refer to this photo whenever you get confused

![img1](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436769/53_o39gys.png)

---

## **📦 Types of Storage: Block, File, and Object Storage**

### **The Library Analogy**

Imagine three different types of libraries, each organized completely differently:

**Library 1: The Block Storage Library**

The Warehouse:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Floor plan:
┌─────┬─────┬─────┬─────┬─────┐
│ 001 │ 002 │ 003 │ 004 │ 005 │
├─────┼─────┼─────┼─────┼─────┤
│ 006 │ 007 │ 008 │ 009 │ 010 │
├─────┼─────┼─────┼─────┼─────┤
│ 011 │ 012 │ 013 │ 014 │ 015 │
└─────┴─────┴─────┴─────┴─────┘

Each box (block):
\- Same size block  (512 bytes or 4KB)
\- Just a number (no label)
\- Can contain ANYTHING

Your book "War and Peace" is stored across:
\- Blocks 003, 004, 005, 007, 009
\- No one knows what's in each block
\- You need a separate map to track which blocks belong to your book

This is BLOCK STORAGE.

**Library 2: The File Storage Library**

The Traditional Library:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Organization:
📁 Fiction/
   📁 Tolstoy/
      📄 War and Peace.txt
      📄 Anna Karenina.txt
   📁 Hemingway/
      📄 The Old Man and the Sea.txt

📁 Non-Fiction/
   📁 History/
      📄 World War II.pdf

Each item:
\- Has a name
\- Lives in a folder (directory)
\- Organized hierarchically

You find your book by path:
/Fiction/Tolstoy/War and Peace.txt

This is FILE STORAGE.

**Library 3: The Object Storage Library**

The Amazon Warehouse:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Everything gets a unique ID:
📦 Item \#7f8e9a2b3c4d
   Content: "War and Peace" book
   Tags:
     \- author: "Tolstoy"
     \- genre: "fiction"
     \- year: 1869
     \- language: "English"
     \- size: 1.2MB
   Metadata: Last accessed, created date, etc.

No folders\! Just objects with IDs and tags.
You search by tags or retrieve by ID.

This is OBJECT STORAGE.

### **Block Storage: The Raw Blocks**

**What it actually is:**

Think of a Hard Drive:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Physical disk divided into blocks:

![img2](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436764/43_ru1dts.png)

Each block in a block storage has
\-  Fixed size (usually 4KB)
\- Addressable by number
\- No built-in organization
\- No metadata

It's like raw memory on disk\!

**How data is stored:**

Example: Storing "HELLO WORLD" (11 bytes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Operating system:
1\. "I need to write 11 bytes"
2\. "Let me allocate blocks... I'll use block 42"
3\. Writes to block 42: "HELLO WORLD"
4\. Remembers: "This file uses block 42"

The block itself doesn't know:
\- What data it contains
\- What file it belongs to
\- Who owns it

The filesystem (ext4, NTFS, etc.) tracks all this\!

**Real-World Usage:**

Perfect for:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. Databases
   Why: Need raw, fast access
   Example:
   \- MySQL/PostgreSQL writing directly to blocks
   \- No filesystem overhead
   \- Can optimize exactly how data is laid out

2\. Virtual Machine Disks
   Why: VM needs to think it has a physical disk
   Example:
   \- AWS EBS (Elastic Block Store)
   \- Your VM sees: "I have a 100GB hard drive"
   \- Actually: 100GB of blocks on AWS infrastructure

3\. Boot Drives
   Why: Operating system needs direct block access
   Example:
   \- Your laptop's hard drive
   \- Server's boot volume

Examples in the Cloud:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

AWS EBS (Elastic Block Store provided by Amazon Cloud )
\- Attach to EC2(AWS’s computer) instance like a hard drive
\- Appears as /dev/xvda
\- Use it just like a physical disk

Azure Managed Disks (Microsoft Cloud)
\- It is similar concept to AWS EB2
\- Attach to VMs

Google Persistent Disks
\- Block storage for Compute Engine

**The Performance Characteristics:**

Block Storage Performance:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Speed: VERY FAST ⚡
\- Direct access to blocks
\- Low latency (\~1ms)
\- High IOPS (Input/Output Operations Per Second)

Use case example:
Database write: "Write 4KB to block 12345"
\- Direct operation: 1ms
\- No filesystem traversal
\- No metadata lookup

Perfect for: Applications needing speed
Cost: More expensive (you're paying for performance)

### **File Storage: The Familiar Filesystem**

**What it actually is:**

Your Computer's Files:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| C:\\Users\\YourName\\├── Documents\\│   ├── Resume.pdf│   ├── Cover Letter.docx│   └── Tax 2024\\│       ├── W2.pdf│       └── 1099.pdf├── Pictures\\│   ├── Vacation 2024\\│   └── Birthday Party\\└── Videos\\ |
| :---- |

This is file storage\!
\- Hierarchical in nature (folders in folders)
\- Each file has a path
\- Familiar and intuitive

**How it works under the hood:**

When you save "Resume.pdf":
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. Filesystem creates entry:
   /Users/YourName/Documents/Resume.pdf

2\. Stores metadata:


|   \- Size: 245KB   \- Created: 2025\-10\-19   \- Modified: 2025\-10\-19   \- Permissions: Read/Write   \- Owner: YourName |
| :---- |

3\. Allocates blocks (under the hood\!):
   \- Uses block storage beneath
   \- File uses blocks: 100, 101, 102, 103

4\. Updates directory structure:
   /Documents/ now contains "Resume.pdf"

The filesystem manages the complexity\!

**Real-World Usage:**

Perfect for:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. Shared File Systems
   Why: Multiple servers need to access same files
   Example:
   \- Company shared drive
   \- Web servers accessing same static files
   \- Media files accessed by multiple apps

2\. Application Data
   Why: Apps expect filesystem semantics
   Example:
   \- Log files: /var/log/app.log
   \- Config files: /etc/config.yaml
   \- Static assets: /public/images/

3\. User Home Directories
   Why: Natural organization
   Example:
   \- Each user has /home/username/
   \- Familiar folder structure

Examples in the Cloud:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

AWS EFS (Elastic File System)
\- Network file system (NFS)
\- Multiple EC2 instances can mount it
\- Appears like a regular directory

Azure Files
\- SMB file shares in the cloud
\- Windows-style file sharing

Google Filestore
\- Managed NFS for Compute Engine

**Network File Storage Example:**

Multiple Web Servers Sharing Files:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img3](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436767/52_xifyn5.png)


All servers see same files\!
Update index.html once → All servers see the change

Use case:
\- Static website assets
\- Uploaded user content
\- Shared configuration files

Benefit:
✓ No need to sync files between servers
✓ Central management
✓ Familiar filesystem operations

Code example:
// Server 1

| fs.writeFile('/var/www/html/images/new-photo.jpg', data); |
| :---- |

// Server 2 (immediately sees it\!)

| fs.readFile('/var/www/html/images/new-photo.jpg'); |
| :---- |

**Performance Characteristics:**

File Storage Performance:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Speed: FAST (but slower than block)
\- Need to traverse directory structure
\- Latency: \~10ms (network file systems)
\- Good throughput for sequential reads

Example operation:
Read /path/to/deep/folder/file.txt
1\. Look up "/path" → 1ms
2\. Look up "/path/to" → 1ms
3\. Look up "/path/to/deep" → 1ms
4\. Look up "/path/to/deep/folder" → 1ms
5\. Look up "file.txt" → 1ms
6\. Read file data → 5ms
Total: 10ms

Slower than block (10ms vs 1ms) but more convenient\!
Cost: Moderate

### **Object Storage: The Cloud-Native Solution**

**What it actually is:**

The Flat Namespace:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

No folders are involved \! Just objects with unique IDs:

| Object 1:  ID: 7f8e9a2b-3c4d-5e6f-7g8h-9i0j1k2l3m4n  Data: \[binary data of photo\]  Metadata:    \- key: "user-photos/john/vacation/beach.jpg"    \- content-type: "image/jpeg"    \- size: 2.5MB    \- uploaded: 2025\-10\-19    \- tags: {user: "john", type: "photo"}Object 2:  ID: a1b2c3d4-e5f6-g7h8-i9j0-k1l2m3n4o5p6  Data: \[binary data of video\]  Metadata:    \- key: "videos/2024/birthday.mp4"    \- content-type: "video/mp4"    \- size: 50MB |
| :---- |

Each object is self-contained\!

**The "key" looks like a path, but it's not\!**

Common Misconception:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This looks like a folder structure:

But it's actually just a STRING (the key)\!

There's no actual "user-photos" folder.
There's no actual "john" folder.
It's all flat\!

Think of it like hashtags:

| /user-photos/john/vacation/beach.jpg\#user-photos-john-vacation-beach.jpg |
| :---- |

So what are the "folders" in the  S3 console you may ask ?  It’s just for  UI convenience\!
They group objects by common prefix.

**Real-World Usage:**

Perfect for:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. Static Assets (Images, Videos, PDFs)
   Why: Cheap, scalable, durable
   Example:
   \- User profile pictures
   \- Product images for e-commerce
   \- Video streaming content

2\. Backups and Archives
   Why: Extremely durable
   Example:
   \- Database backups can be directly added to object storage
   \- Log archives can also be archived to object storage
   \- Historical data

3\. Big Data / Data Lakes
   Why:  It is a good option for big data as it has unlimited scale and it is very  cheap
   Example:
   \- Analytics data can be added to object storage
   \- Machine learning datasets can
   \- IoT sensor data can be streamed to object storage

4\. Static Website Hosting
   Why: Simple, cheap, fast with CDN
   Example:
   \- HTML/CSS/JS files
   \- SPA (Single Page Application)

Examples in the Cloud:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

AWS S3 (Simple Storage Service)
\- The original and most popular object storage
\- Buckets contain objects
\- Virtually unlimited storage

Azure Blob Storage
\- Similar to S3
\- Containers hold blobs

Google Cloud Storage
\- Buckets and objects
\- Similar model to S3

**Performance Characteristics:**

Object Storage Performance:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Speed: SLOWER ( block storage is optimized for throughput, not latency)
\- Latency: \~100ms first byte
\- High throughput: GB/s for large files
\- REST API (HTTP) overhead

Example the  operation:
GET [https://s3.amazonaws.com/bucket/key.jpg](https://s3.amazonaws.com/bucket/key.jpg)

Involves the following steps

1\. HTTP request → 20ms
2\. Authentication → 10ms
3\. Locate object → 30ms
4\. Start streaming → 40ms
Total: 100ms (vs 1ms for block storage)

BUT:
\- Unlimited scale ✓
\- Extremely cheap ✓
\- Built-in redundancy ✓
\- Global accessibility ✓

Cost: VERY CHEAP (\~$0.023/GB/month for S3)

**Connection to REST APIs:** Object storage is accessed via HTTP REST APIs\! Remember GET, PUT, DELETE? That's how you interact with S3. Each object has a URL\!

### **The Comparison Table**

 ![img4](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436769/58_a19dvf.png)

### **Real-World Architecture Example**

E-commerce Platform Storage Design:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Component 1: Database
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Storage Type: BLOCK STORAGE (AWS EBS)
Why block storage?: Needs fast, consistent access
What Data does it store? : User accounts, orders, inventory
Volume: 500 GB
Cost: $50/month

\[PostgreSQL\] → \[EBS Volume 500GB\]
               (Direct block access)

![img5](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436765/47_gqcxuc.png)

Component 2: Application Servers
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Storage Type: FILE STORAGE (AWS EFS)
Why file storage?: Multiple servers need same files
What data does it store? : Application logs, temp files, configs
Volume: 100 GB
Cost: $30/month

![img6](https://res.cloudinary.com/dretwg3dy/image/upload/v1762438956/54_koydvs.png)

Component 3: Product Images
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Storage Type: OBJECT STORAGE (AWS S3)
Why: Cheap, scalable, public URLs
Data: Product photos, user uploads
Volume: 5 TB
Cost: $115/month

![img7](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436767/55_vp5syz.png)

Component 4: Database Backups
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Storage Type: OBJECT STORAGE (S3 Glacier object storage provided by AWS)
Why: Extremely cheap for archives
Data: Daily database backups
Volume: 10 TB
Cost: $40/month (Glacier Deep Archive\!)

![img8](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436768/51_we1d9q.png)

Total Monthly Cost: $235
Total Storage: 15.6 TB

If everything was block storage: $1,560/month\! 😱
Savings: $1,325/month (85% savings\!)

---

## **💾 Memory vs Disk: The Speed Hierarchy**

### **The Kitchen Analogy**

We have already introduced this concept in the earlier section here is a bit detailed and elaborated part of the same concept

Let’s compare computer storage with something familiar: cooking in a kitchen.

Your Kitchen Layout:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Level 1: Your Hands (analogous to the CPU Registers)
\- Currently holding a knife and onion
\- Instant access (nanoseconds)
\- Very limited (2 items)

Level 2: Countertop ( like the RAM)
\- Ingredients you're actively using
\- Very fast access (microseconds)
\- Limited space (16GB worth)
\- Loses everything when power off (volatile\!)

Level 3: Kitchen Cabinets (SSD)
\- Ingredients you use today
\- Fast access (milliseconds)
\- More space (1TB worth)
\- Keeps everything when power off (persistent\!)

Level 4: Garage Storage (HDD)
\- Bulk items, rarely used ingredients
\- Slower access (seconds)
\- Lots of space (10TB worth)
\- Persistent storage

Level 5: Storage Unit (Cloud/Tape)
\- Things you barely use
\- Very slow access (minutes/hours)
\- Massive space (unlimited)
\- Persistent, archived

The Pattern:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Closer \= Faster but Less Space
Farther \= Slower but More Space

Remember the storage diagram we discussed on the first section (the fundamentals of computing)? Storage latency ranges from nanoseconds (RAM) to milliseconds (disk) to seconds (network storage). This affects EVERY operation in your system\!

### **RAM (Random Access Memory): The Speed King**

**What it actually is:**

Physical RAM Module:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img9](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436766/48_wtkdft.png)

Characteristics:
\- Random access (any address in same time)
\- Volatile (loses data when power off\!)
\- VERY fast
\- Limited capacity

**Speed Demonstration:**

RAM Access Time:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reading 1 byte from RAM: \~100 nanoseconds

How fast is that?

If RAM access was 1 second:
\- SSD access: 16 minutes
\- HDD access: 2 hours
\- Internet request: 4 months\!

Real numbers:
\- RAM: 100 ns (0.0001 ms)
\- SSD: 0.1 ms (1,000x slower)
\- HDD: 10 ms (100,000x slower)

**Real-World Example:**

**You’ll learn more about database queries and caching in future lessons but if you write code you could refer to these snippets below**

| // Database Query Without Cachingconst getUser \= async (userId) \=\> {  // Query database (on SSD)  const user \= await db.query(    'SELECT \* FROM users WHERE id \= ?',     \[userId\]  );  return user;};// Time: 50ms (database query)// If called 1000 times: 50 seconds\!// Database Query With RAM Cachingconst userCache \= new Map(); // In RAM\!const getUserCached \= async (userId) \=\> {  // Check RAM first  if (userCache.has(userId)) {    return userCache.get(userId); // From RAM\!  }    // Not in cache, query database  const user \= await db.query(    'SELECT \* FROM users WHERE id \= ?',     \[userId\]  );    // Store in RAM for next time  userCache.set(userId, user);    return user;}; |
| :---- |

// First call: 50ms (database)
// Subsequent calls: 0.0001ms (RAM)
// If called 1000 times: 50ms \+ (999 × 0.0001ms) \= 50ms\!
// 1000x faster\! ⚡

**The Volatility Problem:**

What Happens When Power is Lost:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RAM (Volatile):
Power ON:  \[Your data is here\!\]
Power OFF: \[Everything GONE\! ❌\]
Power ON:  \[Empty RAM, no data\]

Disk (Persistent):
Power ON:  \[Your data is here\!\]
Power OFF: \[Data stays on disk ✓\]
Power ON:  \[Your data is still here\! ✓\]

This is why:
\- You can't store databases ONLY in RAM
\- Cache can disappear on restart
\- Need to persist important data to disk

**RAM Usage Patterns:**

What Goes in RAM:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. Active Process Memory
   \- Running applications
   \- Operating system
   \- Currently executing code

2\. Cache
   \- Database query results
   \- Frequently accessed data
   \- Session data

3\. Buffers
   \- Data waiting to be written to disk
   \- Network packet buffers

4\. File System Cache
   \- Recently accessed files
   \- OS caches file contents

Monitoring RAM:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| $ free \-h              total        used        free      shared  buff/cache   availableMem:           16Gi       8.2Gi       2.1Gi       200Mi       5.7Gi       7.5Gi |
| :---- |

Used: 8.2GB (your applications)
Free: 2.1GB (unused)
Buff/cache: 5.7GB (OS caching files)
Available: 7.5GB (can be freed if needed)

### **SSD (Solid State Drive): The Modern Standard**

**What it actually is:**

How SSDs Work:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Flash Memory Cells:

![img10](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436766/49_psldow.png)

No moving parts\!
\- Electronic (like RAM)
\- Persistent (like HDD)
\- Fast (almost like RAM)

Best of both worlds\!

**Speed Profile:**

SSD Performance:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Random Read: 0.1ms (fast\!)
Sequential Read: 500 MB/s (very fast\!)
Random Write: 0.1ms
Sequential Write: 450 MB/s

IOPS (Input/Output Operations Per Second):
Consumer SSD: 100,000 IOPS
Enterprise SSD: 500,000+ IOPS

This is why modern computers feel fast\!

**Real-World Impact:**

Loading a Web Application:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

HDD System:
1\. Boot OS: 60 seconds
2\. Load browser: 20 seconds
3\. Start app: 15 seconds
Total: 95 seconds 😴

SSD System:
1\. Boot OS: 10 seconds
2\. Load browser: 2 seconds
3\. Start app: 3 seconds
Total: 15 seconds ⚡

6x faster\! This is why SSDs are standard now.

**Types of SSDs:**

SATA SSD (Consumer Grade):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Speed: 500 MB/s
Interface: SATA (same as old HDDs)
Cost: $0.10/GB
Use: Laptops, desktops

![img11](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436769/59_ietplb.png)

NVMe SSD (High Performance):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Speed: 3,500 MB/s (7x faster\!)
Interface: PCIe (direct to CPU)
Cost: $0.15/GB
Use: Gaming PCs, servers, databases

![img12](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436769/61_rx1nnr.png)

Cloud SSD (AWS gp3):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Speed: 3,000 IOPS baseline
Cost: $0.08/GB/month \+ IOPS
Use: Production databases, VMs

### **HDD (Hard Disk Drive): The Legacy Workhorse**

**What it actually is:**

Mechanical Hard Drive:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Physical Components:
    \[Spinning Platter\]
         │  7,200 RPM
         │
    \[Read/Write Head\]
         │  Moves across platter

Like a record player\!

**Why HDDs are Slow:**

Lets look at the steps involved in reading data from HDD:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: Seek Time (5-10ms)
   Move the read head to correct track
   \[Physical movement\! Slow\!\]

Step 2: Rotational Latency (4ms average)
   Wait for platter to spin to right position
   \[Mechanical\! Must wait for rotation\!\]

Step 3: Transfer Time (0.1ms)
   Actually read the data
   \[Finally\! Data transfer\!\]

Total: \~10ms per operation

Compare to SSD: 0.1ms
HDD is 100x slower\!

![img13](https://res.cloudinary.com/dretwg3dy/image/upload/v1762354300/32_yns27w.png)

Why Sequential is Better:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Random reads (jumping around):
\- Read block 100: Seek (10ms) \+ Read
\- Read block 5000: Seek (10ms) \+ Read
\- Read block 200: Seek (10ms) \+ Read
Total: 30ms for 3 blocks

Sequential reads (consecutive):
\- Read block 100: Seek (10ms) \+ Read
\- Read block 101: No seek\! \+ Read
\- Read block 102: No seek\! \+ Read
Total: 10ms for 3 blocks

3x faster when sequential\!

**Where HDDs Still Make Sense:**

HDD Advantages:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. Cost
   HDD: $0.02/GB
   SSD: $0.10/GB
   5x cheaper\!

2\. Capacity
   HDD: 20TB drives available
   SSD: 8TB typical max
   More space\!

3\. Longevity for Archives
   HDD: Can last 10+ years sitting
   SSD: Can lose data after years unpowered
   Better for cold storage\!

Good use cases:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Backups (sequential writes)
✓ Video surveillance (continuous write)
✓ Media archives (large files, rare access)
✓ Cold data storage
✓ Data warehouses (sequential scans)

Bad use cases:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

❌ Databases (random access)
❌ Operating system
❌ Virtual machines
❌ Active applications
❌ Anything needing low latency

### **The Complete Storage Hierarchy**

The Memory/Storage Pyramid (**we saw similar diagram in our computer fundamentals section this is the detailed/extended diagram for the original diagram**)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

![img14](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436766/46_pdclle.png)


---

## **🌡️ Hot vs Cold Data: The Temperature Model**

### **The Closet Organization Analogy**

Think about your closet:

Your Closet:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Front of Closet (Hot Data):

![img15](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436766/50_csiig8.png)

Back of Closet (Warm Data):
![img16](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436765/41_kxbxqo.png)

Storage Unit (Cold Data):

![img17](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436764/40_ptmfvp.png)

### **Defining Data Temperature**

**Hot Data: The Active Zone**

What Makes Data "Hot":
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Characteristics:
✓Data that is  accessed frequently (daily/hourly)
✓ Data which is modified often
✓ Time-sensitive
✓ Business-critical
✓ Needs low latency

Examples:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. In the case of  E-commerce:
   \- Current product inventory
   \- Shopping cart contents
   \- Today's orders
   \- Active user sessions

2\. Social Media:
   \- Recent posts (last 7 days)
   \- Current trending topics
   \- Active user profiles
   \- Real-time notifications

3\. Banking:
   \- Current account balances
   \- Today's transactions
   \- Active credit card sessions
   \- Real-time fraud detection data

Storage Choice:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ideal: RAM cache \+ SSD
Why: Need instant access
Cost: High ($0.10-0.50/GB/month)
Worth it: Users notice any delay\!

**Warm Data: The Middle Ground**

What Makes Data "Warm":
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Characteristics:
✓ Accessed occasionally (weekly/monthly)
✓ Rarely modified
✓ Still relevant
✓ Not time-critical
✓ Can tolerate slight delay

Examples:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. E-commerce:
   \- Order history (last 90 days)
   \- Product reviews
   \- Customer support tickets (recent)
   \- Returns/refunds data

2\. Social Media:
   \- Posts from last month
   \- User activity logs
   \- Analytics data
   \- Older notifications

3\. Banking:
   \- Transaction history (3-6 months)
   \- Archived statements
   \- Historical account data
   \- Completed loan applications

Storage Choice:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ideal: HDD or standard SSD
Why: To balance out cost and access
Cost: Moderate ($0.02-0.08/GB/month)
Worth it: Good enough performance

**Cold Data: The Archive**

What Makes Data "Cold":
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Characteristics:
✓ Rarely accessed (yearly or never)
✓ Never modified (immutable)
✓ Historical/compliance
✓ Can wait minutes/hours for access
✓ Needs to be kept (legal/audit)

Examples:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. E-commerce:
   \- Orders from 5 years ago
   \- Old product catalogs
   \- Deleted account backups
   \- Compliance audit logs

2\. Social Media:
   \- Deleted posts (retention policy)
   \- User data snapshots
   \- System logs from years ago
   \- Legal discovery data

3\. Banking:
   \- 7-year transaction history
   \- Closed account records
   \- Old audit reports
   \- Regulatory compliance archives

Storage Choice:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ideal storage: Glacier, tape, cold storage
Why:  Data is rarely accessed, needs to be cheap
Cost: Very low ($0.001-0.004/GB/month)
Worth it: 20x cheaper\! Storage at scale\!

### **Data Temperature Transitions**

The Lifecycle of Data:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Day 1: HOT 🔥
├─ User uploads photo
├─ Stored in: SSD with RAM cache
├─ Access: 1000+ times/day
└─ Cost: $0.10/GB/month

Day 30: WARM 🌤️
├─ Photo is 1 month old
├─ Moved to: Standard storage
├─ Access: 10 times/day
└─ Cost: $0.03/GB/month

Day 365: COLD ❄️
├─ Photo is 1 year old
├─ Moved to: Glacier
├─ Access: 1 time/year (if ever)
└─ Cost: $0.004/GB/month

Day 2555: FROZEN 🧊
├─ Photo is 7 years old
├─ Moved to: Deep archive/tape
├─ Access: Only if subpoenaed
└─ Cost: $0.001/GB/month

**Real-World Example: Instagram**

Instagram Photo Storage Strategy:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

When you upload a photo to instagram
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Day 0-7: HOT 🔥
Location: CDN (remember Amazon Cloudfront from the diagram) \+ SSD cache
Why: 90% of views happen in first week
Access pattern: Thousands of views
Storage: Multiple copies in RAM/SSD
Cost per GB: $0.50/month
User experience: \<50ms load time

Week 2-Month 3: WARM 🌤️
Location: Standard S3 storage
Why: Occasional views from followers
Access pattern: Dozens of views
Storage: Standard redundancy
Cost per GB: $0.023/month
User experience: \~200ms load time

Month 4+: COLD ❄️
Location: S3 Intelligent-Tiering
Why: Rarely viewed old photos
Access pattern: A few views per month
Storage: Automatic tier adjustment
Cost per GB: $0.01/month
User experience: \~500ms first access

Year 2+: FROZEN 🧊
Location: Glacier Deep Archive
Why: Historical archive (almost never viewed)
Access pattern: Almost never
Storage: Tape backup, rare access
Cost per GB: $0.001/month
User experience: 12 hours to retrieve (if ever needed)

Instagram's savings:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1 billion photos uploaded/day
Average photo: 2MB

If ALL photos were to be  stored HOT:
\- 2,000 TB uploaded daily
\- 730,000 TB per year
\- Cost: $365,000/day \= $133M/year 😱

With temperature-based storage:
\- Recent (HOT): 14,000 TB × $0.50 \= $7,000/day
\- Warm: 100,000 TB × $0.023 \= $2,300/day
\- Cold: 616,000 TB × $0.001 \= $616/day
\- Total: $10,000/day \= $3.6M/year ✓

Savings: $129M/year (97% reduction\!)

ed on access patterns

### **The 80/20 Rule in Storage**

Pareto Principle for Data:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Observation:
\- 20% of your data gets 80% of the access
\- 80% of your data gets 20% of the access

Translation:
\- Keep 20% (hot) on expensive fast storage
\- Keep 80% (cold) on cheap slow storage

Real Numbers:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total data: 100 TB

Naive approach (all hot storage):
\- 100 TB × $0.10/GB \= $10,000/month

Optimized approach:
\- 20 TB hot × $0.10/GB \= $2,000/month
\- 80 TB cold × $0.01/GB \= $800/month
\- Total: $2,800/month

Savings: $7,200/month (72% reduction\!)
Same user experience for 99% of requests\!

### **Decision Framework: Choosing Storage Temperature**

Decision Tree:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Question 1: How often is it accessed?
├─ Multiple times/day → HOT 🔥
├─ Weekly/Monthly → WARM 🌤️
└─ Yearly or less → COLD ❄️

Question 2: How fast must it load?
├─ \<100ms → HOT (SSD \+ cache)
├─ \<1s → WARM (Standard storage)
└─ Minutes OK → COLD (Archive)

Question 3: Is it modified?
├─ Yes, frequently → HOT
├─ Occasionally → WARM
└─ Never (immutable) → COLD

Question 4: How critical is it?
├─ Business-critical → HOT
├─ Important → WARM
└─ Archive/compliance → COLD

Question 5: How much does latency cost?
├─ $10/second delay → HOT
├─ $1/second delay → WARM
└─ Doesn't matter → COLD

### **Real-World Case Studies**

**Case Study 1: Netflix**

Netflix Content Storage:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

HOT 🔥 (10% of catalog, 90% of views):
├─ New releases
├─ Trending shows
├─ Popular movies
├─ Storage: CDN edge servers (SSD)
├─ Latency: \<50ms
└─ Cost: High, but worth it

WARM 🌤️ (40% of catalog, 9% of views):
├─ Recently popular shows
├─ Catalog titles
├─ Regional favorites
├─ Storage: Regional data centers
├─ Latency: \~200ms
└─ Cost: Moderate

COLD ❄️ (50% of catalog, 1% of views):
├─ Old seasons
├─ Niche documentaries
├─ Foreign language content
├─ Storage: Central S3
├─ Latency: \~1s (if not cached)
└─ Cost: Very low

Result:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
\- 90% of user requests: \<50ms (great UX\!)
\- Storage costs: 70% lower than all-hot
\- Users don't notice cold storage delays

**Case Study 2: Healthcare Records**

Patient Medical Records:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

HOT 🔥 (Active patients):
├─ Current patient records
├─ Recent lab results
├─ Today's appointments
├─ Active prescriptions
├─ Storage: SSD database
├─ Access: Doctors need instant access
└─ Retention: 90 days hot

WARM 🌤️ (Recent history):
├─ Last year's visits
├─ Historical lab results
├─ Imaging from past 2 years
├─ Storage: Standard S3
├─ Access: Occasional review
└─ Retention: 2 years warm

COLD ❄️ (Archives):
├─ 7-year history (legal requirement)
├─ Old x-rays and scans
├─ Closed patient records
├─ Storage: Glacier
├─ Access: Legal/audit only
└─ Retention: 7+ years

Compliance:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ HIPAA requires 7-year retention
✓ Must be retrievable (but can be slow)
✓ Cold storage meets legal requirements
✓ Massive cost savings

Savings: $500k/year for mid-size hospital

---

## **🎓 Putting It All Together: Complete Storage Architecture**

Let’s see data storage patterns in  a real production system:

Social Media Platform \- Complete Storage Design:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Layer 1: Cache (HOT 🔥)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[Redis Cluster\]
\- Type: RAM
\- Size: 100 GB
\- Latency: 0.1ms
\- Data:
  ✓ User sessions
  ✓ Feed cache (recent posts)
  ✓ Trending topics
  ✓ Active user profiles
\- TTL: 1-24 hours
\- Cost: $500/month

![img18](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436766/44_lgyzvx.png)

Layer 2: Primary Database (HOT 🔥)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[PostgreSQL on NVMe SSD\]
\- Type: Block Storage (EBS)
\- Size: 2 TB
\- Latency: 1ms
\- Data:
  ✓ User accounts
  ✓ Current posts (30 days)
  ✓ Active comments
  ✓ Real-time analytics
\- Backup: Hourly
\- Cost: $400/month

![img19](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436769/60_xzhrob.png)

Layer 3: Media Storage \- Recent (HOT 🔥)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[S3 Standard \+ CloudFront CDN\]
\- Type: Object Storage
\- Size: 50 TB
\- Latency: 50ms (CDN) / 200ms (S3)
\- Data:
  ✓ Photos from last 30 days
  ✓ Videos uploaded recently
  ✓ Profile pictures (active users)
\- Lifecycle: Move to warm after 30 days
\- Cost: $1,150/month

![img20](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436768/56_xuced2.png)

Layer 4: Media Storage \- Historical (WARM 🌤️)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[S3 Infrequent Access\] \- Type: Object Storage \- Size: 200 TB \- Latency:
300ms \- Data: ✓ Photos 30-365 days old ✓ Older videos ✓ Historical user content
\- Lifecycle: Move to cold after 1 year \- Cost: $2000/month (for 200TB)
![img21](https://res.cloudinary.com/dretwg3dy/image/upload/v1762436764/42_uz7dol.png)
Layer 5: Analytics Data (WARM 🌤️) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[Data Warehouse on HDD\]
\- Type: File Storage / Block Storage
\- Size: 100 TB
\- Latency: 10ms
\- Data:
  ✓ User behavior logs
  ✓ Engagement metrics
  ✓ A/B test results
  ✓ Business intelligence data
\- Access: Batch queries, reports
\- Cost: $800/month

Layer 6: Archives (COLD ❄️)

![img22](https://res.cloudinary.com/dretwg3dy/image/upload/v1762440179/57_ir8cpn.png)

\[S3 Glacier\]
\- Type: Object Storage (Archive)
\- Size: 500 TB
\- Latency: 5 hours
\- Data:
  ✓ Old posts (1+ year)
  ✓ Deleted account backups
  ✓ Audit logs
  ✓ Compliance data
\- Lifecycle: Keep for 7 years
\- Cost: $2,000/month

Layer 7: Deep Archive (FROZEN 🧊)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

\[S3 Glacier Deep Archive\]
\- Type: Object Storage (Tape-equivalent)
\- Size: 1 PB (1000 TB)
\- Latency: 12-48 hours
\- Data:
  ✓ Legal discovery data
  ✓ Historical system logs
  ✓ Old database backups
  ✓ Long-term archives
\- Retention: Permanent
\- Cost: $1,000/month

Total Storage: 1,853 TB
Total Cost: $7,850/month

If everything was HOT storage:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1,853 TB × $0.10/GB \= $185,300/month 😱

Actual cost: $7,850/month ✓
Savings: $177,450/month (96% reduction\!)

**The Request Flow:**

User Views a Post:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1\. Check Redis (Layer 1):
   "Is this post cached?"
   Hit: Return in 0.1ms ⚡ \[95% of requests end here\!\]
   Miss: Continue...

2\. Check PostgreSQL (Layer 2):
   "Query post from database"
   Found: Return in 5ms ✓ \[4% of requests\]
   Not found: Continue...

3\. Check S3 Standard (Layer 3):
   "Fetch from recent media storage"
   Found: Return in 200ms ✓ \[0.9% of requests\]
   Not found: Continue...

4\. Check S3 IA (Layer 4):
   "Fetch from historical storage"
   Found: Return in 300ms ✓ \[0.09% of requests\]
   Not found: Continue...

5\. Check Glacier (Layer 6):
   "Initiate restore (5 hours)"
   Found: User gets "This post is being retrieved" \[0.01% of requests\]

![img23](https://res.cloudinary.com/dretwg3dy/image/upload/v1762440474/final_ziwy35.png)


Result:
\- 95% of users: \<1ms (instant\!)
\- 4% of users: \<10ms (very fast)
\- 0.99% of users: \<300ms (acceptable)
\- 0.01% of users: Wait for archive restore (rare\!)

---

## **✅ Final Mastery Checklist**

You've mastered storage fundamentals if you can:

**Types of Storage:**

* \[ \] Explain block vs file vs object storage
* \[ \] Choose the right storage type for a use case
* \[ \] Understand the tradeoffs of each

**Memory vs Disk:**

* \[ \] Describe the storage hierarchy from CPU to cloud
* \[ \] Explain volatile vs persistent storage
* \[ \] Calculate cost/performance tradeoffs

**Hot vs Cold Data:**

* \[ \] Identify data temperature based on access patterns
* \[ \] Design lifecycle policies for data
* \[ \] Calculate storage cost optimizations

---

## **🚀 Interview-Ready Summary**

**Key Takeaways:**

1. **Storage types serve different purposes** \- Block for performance, File for sharing, Object for scale
2. **Speed costs money** \- RAM is 1000x faster but 100x more expensive than disk
3. **Data has temperature** \- Most data is cold, optimize accordingly
4. **The 80/20 rule** \- 20% of data gets 80% of access
5. **Lifecycle management** \- Automatically move data based on age/access

**When asked "How would you design storage for this system?"**

Answer framework:

1. "First, I'd identify the hot data that needs fast access..."
   * User sessions → RAM cache
   * Active records → SSD database
2. "Then categorize data by access patterns..."
   * Daily access → Hot (SSD \+ cache)
   * Weekly/monthly → Warm (Standard storage)
   * Yearly/never → Cold (Archive)
3. "Choose appropriate storage types..."
   * Database → Block storage
   * Shared files → File storage
   * Media/backups → Object storage
4. "Implement lifecycle policies..."
   * Auto-transition based on age
   * Monitor access patterns
   * Optimize costs continuously
5. "Calculate cost savings..."
   * All hot: $X/month
   * Tiered: $Y/month
   * Savings: $(X-Y)/month

**Common Interview Questions:**

Q: "Why not put everything in RAM?" A: "RAM is volatile (loses data on restart) and expensive. We need persistent storage for data, but use RAM for caching hot data."

Q: "When would you use object storage vs file storage?" A: "Object storage for static assets, backups, and unstructured data at scale. File storage when you need hierarchical organization or multiple servers sharing files."

Q: "How would you optimize storage costs?" A: "Implement data lifecycle policies, cache hot data in RAM/SSD, move warm data to standard storage, archive cold data, and delete expired data. Use the 80/20 rule—keep 20% hot, 80% cold."

**You're now ready to design production storage systems\!** 🎉

Remember: The best storage architecture isn't the fastest or the cheapest—it's the one that matches your access patterns. Fast where it matters, cheap where it doesn't.

Keep practicing by analyzing storage in systems you use daily. Ask yourself: "Why did they choose this storage solution?" That's how you develop storage intuition\!
