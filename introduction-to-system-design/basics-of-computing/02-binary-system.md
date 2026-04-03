---
slug: binary-system
title: Binary System
readTime: 20 min
orderIndex: 2
premium: false
---

# Binary System

> Every image, video, and message on the internet is ultimately a sequence of 0s and 1s — understanding binary is understanding how computers think.


## **🔢 2. Binary System - The Language of Computers**

### **🎯 Challenge 2: The Light Switch Language**

**Scenario:** Imagine you're in a room with only a light switch. You can only communicate using ON and OFF.

**Challenge:** How would you represent:

* Numbers (0, 1, 2, 3...)
* Letters (A, B, C...)
* Colors (Red, Green, Blue...)
* Your vacation photos?

**Pause and think:** With just ON and OFF, can you really represent everything?

---

### **💡 The Binary Revelation**

**The Answer:** YES! Computers do exactly this with electricity:

BINARY \= The language of ON and OFF


Physical reality in computer:

  i) Voltage HIGH (5 volts) \= 1 (ON)

 ii) Voltage LOW (0 volts)  \= 0 (OFF)

Everything in your computer is just:

0 0 1 1 0 1 0 1 1 0 0 1 1 1 0...

But from these simple 0s and 1s, we get:

✓ Documents

✓ Photos

✓ Videos

✓ Music

✓ Games

✓ The entire internet!

**Key insight:** Binary is like Morse code - simple signals that combine to express complex ideas!

---

### **🎓 Understanding Binary: From Bits to Terabytes**

#### **The Hierarchy of Data**

🔍 THE BUILDING BLOCKS:


1️⃣ BIT (Binary Digit)
   Smallest unit of data
   Can be: 0 or 1

   Example: 1

   Real-world: A single light switch



2️⃣ BYTE (8 Bits)
   Basic unit of storage
   8 bits \= 1 byte

   Example: 01001000

   Can represent:

   i) Number 0-255

   ii) Single letter (H)

   iii)  Small instruction

   Real-world: Its like 8 light switches in a row


3️⃣ KILOBYTE (1,024 Bytes)
   1 KB ≈ 1 thousand bytes

   Examples:

   You have a short email: 2 KB

    A small text file: 5 KB

   Tiny image: 10 KB



4️⃣ MEGABYTE (1,024 KB)
   1 MB ≈ 1 million bytes

   Examples:

   A high-res photo: 3 MB

   A 1 minute MP3 song: 1 MB

   A short document: 0.5 MB

   A typical app install: 50 MB


5️⃣ GIGABYTE (1,024 MB)
   1 GB ≈ 1 billion bytes

   Examples:

    HD movie (1080p): 4 GB

    1 hour HD video: 2 GB

    Modern video game: 50 GB

    1000 songs: 1 GB

    Smartphone storage: 128 GB


6️⃣ TERABYTE (1,024 GB)
   1 TB ≈ 1 trillion bytes

   Examples:

   250 HD movies

   200,000 songs

   500,000 photos

   Laptop hard drive: 1-2 TB

   External backup drive: 4 TB


7️⃣ PETABYTE (1,024 TB)
   1 PB ≈ 1 quadrillion bytes

   Examples:
    Netflix's entire library: \~100 PB

    Large company data center: 10 PB

    Facebook's daily data: \~4 PB


**The Scale Visualization:**

From smallest to largest:

Bit          •                    (one dot)

Byte         ••••••••             (8 dots)

Kilobyte     [Small paragraph] ፨

Megabyte     [Entire book] 📖

Gigabyte     [Bookshelf - 100 books]📚

Terabyte     [Library - 100 bookshelves] 🚪🚪🚪🚪🚪🚪

Petabyte     [50 Libraries] 🏫🏫🏫🏫🏫🏫🏫🏫🏫🏫🏫🏫

---

### **🎮 Interactive Exercise: Binary to Decimal**

**Let's learn how binary represents numbers:**

DECIMAL SYSTEM (Base 10):

Positions:  1000s  100s  10s  1s
Number:        2     5    6   3

2×1000 \+ 5×100 \+ 6×10 \+ 3×1 \= 2563

We use 10 digits: 0,1,2,3,4,5,6,7,8,9


BINARY SYSTEM (Base 2):

Positions:  8s  4s  2s  1s
Number:      1   0   1   1

1×8 \+ 0×4 \+ 1×2 \+ 1×1 \= 11 (decimal)

We use 2 digits: 0, 1


EXAMPLES:

Binary → Decimal:

0001 \= 1

0010 \= 2

0011 \= 3

0100 \= 4

0101 \= 5

0110 \= 6

0111 \= 7

1000 \= 8

Pattern: Each position doubles!

   8  4  2  1

   ↓  ↓  ↓  ↓

   1  0  1  1  \= 8 \+ 2 \+ 1 \= 11

   16 8  4  2  1

   ↓  ↓  ↓  ↓  ↓

   1  0  0  1  1  \= 16 \+ 2 \+ 1 \= 19

**Try these yourself:**

Binary 1111 \= ?

Binary 1010 \= ?

Binary 0110 \= ?

(Answers below)

---

**ANSWERS:**

Binary 1111 \= 8 \+ 4 \+ 2 \+ 1 \= 15

Binary 1010 \= 8 \+ 0 \+ 2 \+ 0 \= 10

Binary 0110 \= 0 \+ 4 \+ 2 \+ 0 \= 6

---

### **🔤 How Binary Represents Text: ASCII and Unicode**

**Ever wonder how computers store letters?**

ASCII ENCODING:

Each letter \= 1 byte (8 bits)

Letter  →  Decimal  →  Binary

──────────────────────────────────

  A     →     65    →  01000001

  B     →     66    →  01000010

  C     →     67    →  01000011

  a     →     97    →  01100001

  b     →     98    →  01100010

  0     →     48    →  00110000

  !     →     33    →  00100001

Space   →     32    →  00100000

Example: The word "Hi!"

Hi   !
01001000 01101001 00100001

Total: 3 bytes (24 bits) to store "Hi!"

**Your name in binary:**

Example: "Bob"

B → 66 → 01000010

o → 111 → 01101111

b → 98 → 01100010

"Bob" \= 01000010 01101111 01100010

3 letters \= 3 bytes of storage

---

### **🎨 How Binary Represents Colors: RGB**

COLOR ENCODING:

Every pixel on your screen \= 3 bytes (24 bits)

Red: 1 byte (0-255)

Green: 1 byte (0-255)

Blue: 1 byte (0-255)

Examples:

i) **Pure Red:**

R: 255 (11111111)

G: 0   (00000000)

B: 0   (00000000)

Red pixel

ii) **Pure Green:**

R: 0   (00000000)

G: 255 (11111111)

B: 0   (00000000)

 Green pixel

iii) **Purple:**

R: 128 (10000000)

G: 0   (00000000)

B: 128 (10000000)

  Purple pixel

iv) **White:**

R: 255 (11111111)

G: 255 (11111111)

B: 255 (11111111)

 White pixel

v) **Black:**

R: 0   (00000000)

G: 0   (00000000)

B: 0   (00000000)

**Black pixel**

Your 1920×1080 monitor:
\= 2,073,600 pixels × 3 bytes per pixel \= 6,220,800 bytes ≈ 6 MB for ONE FRAME!

At 60 FPS:
6 MB × 60 \= 360 MB per second!
(This is why graphics cards need fast memory!)

---

### **🚨 Common Misconception: "KB, MB, GB are Exact Thousands"**

**You might think:** "1 KB \= 1,000 bytes exactly"

**The Reality:** It's actually 1,024!

**❌ MARKETING NUMBERS (Decimal):**
1 KB \= 1,000 bytes

1 MB \= 1,000 KB \= 1,000,000 bytes

1 GB \= 1,000 MB \= 1,000,000,000 bytes

(Hard drive manufacturers use this!)


**✅ COMPUTER NUMBERS (Binary):**

1 KB \= 1,024 bytes (2¹⁰)

1 MB \= 1,024 KB \= 1,048,576 bytes (2²⁰)

1 GB \= 1,024 MB \= 1,073,741,824 bytes (2³⁰)

(Computer systems use this!)


**WHY THE DIFFERENCE?**

Computers think in powers of 2:

2¹⁰ \= 1,024 (close to 1,000)

2²⁰ \= 1,048,576 (close to 1 million)

2³⁰ \= 1,073,741,824 (close to 1 billion)

THE RESULT:

You buy a "500 GB" hard drive:

Marketing: 500,000,000,000 bytes

Computer sees: 465 GB

You: "Where did my 35 GB go?!" 😡

**Reality: Marketing uses 1000, computers use 1024
That's a 7% difference!**

**Mental model:** Computer storage is like buying a "1 pound" of coffee that's actually 0.93 pounds because they use different measuring systems!

---

### **🎯 The Power of Binary: Everything is Numbers**

WHAT BINARY REPRESENTS:

**Text:**

"Hello" → 5 bytes

Each letter → specific number → binary

**Images:**

Photo.jpg → Millions of pixels

Each pixel → RGB values → binary

**Videos:**

Movie.mp4 → Sequence of images \+ audio

Each frame → pixels → binary

Audio → sound waves → numbers → binary

**Music:**

Song.mp3 → Sound wave samples

44,100 samples per second

Each sample → number → binary

**Programs:**

Chrome.exe → Machine instructions

Each instruction → number → binary

Everything is just different ways of interpreting

patterns of 0s and 1s!

**Mind-blowing fact:**

This entire article you're reading:
- Every letter: 1 byte

- Total: \~50,000 characters \= 50 KB

- In binary: 400,000 bits

- In binary: 400,000 individual 0s and 1s!


01000001 01110010 01100101... (and so on for 50KB!)

---


---

## Key Takeaways

1. **Computers use binary because transistors have two states** — on (1) and off (0), making binary the natural number system for hardware
2. **All data types are encoded as binary** — integers, floating-point numbers, characters (ASCII/UTF-8), and colors are all bit patterns
3. **Understanding binary helps you reason about storage sizes** — 1 byte = 8 bits = 256 possible values, which is why IP addresses max at 255
4. **Hexadecimal is a compact way to represent binary** — each hex digit maps to exactly 4 bits, making it useful for memory addresses and colors
