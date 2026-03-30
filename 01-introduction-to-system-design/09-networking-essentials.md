# Networking Essentials

You don't need to be a network engineer to design great systems. But understanding how data travels across the internet will make you a dramatically better architect. This article covers exactly what you need.

---

## The Protocols That Matter

The internet is a stack of protocols. For system design, focus on these layers:

```
┌─────────────────────────────────────────┐
│  Application Layer  │  HTTP, HTTPS,      │  What your app speaks
│                     │  WebSocket, DNS    │
├─────────────────────┼────────────────────┤
│  Transport Layer    │  TCP, UDP          │  How data is delivered
├─────────────────────┼────────────────────┤
│  Network Layer      │  IP                │  How packets are routed
├─────────────────────┼────────────────────┤
│  Physical/Link      │  Ethernet, WiFi    │  How bits move physically
└─────────────────────┴────────────────────┘
```

---

## DNS — How Names Become Addresses

Every time you visit a website, DNS translates the domain name into an IP address.

### DNS Resolution Step by Step

```
User types: algoroq.com

1. Browser cache: "Do I know this?" → No
2. OS cache / /etc/hosts: "Do I know this?" → No
3. Recursive Resolver (your ISP): "Let me find out"
4. Root Name Server: "I don't know, ask .com TLD server"
5. TLD Name Server (.com): "I don't know, ask ns1.algoroq.com"
6. Authoritative Name Server: "algoroq.com is at 1.2.3.4"
7. Answer cached at each layer with TTL
8. Browser connects to 1.2.3.4

Full lookup: ~100ms (cold)
Subsequent lookups: ~1ms (cached)
```

### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| A | Domain → IPv4 address | `algoroq.com → 1.2.3.4` |
| AAAA | Domain → IPv6 address | `algoroq.com → 2001:db8::1` |
| CNAME | Domain → another domain | `www.algoroq.com → algoroq.com` |
| MX | Mail exchange server | `algoroq.com → mail.algoroq.com` |
| TXT | Text data (SPF, DKIM) | Spam prevention records |
| NS | Authoritative name server | Points to DNS servers for the domain |

### DNS for Load Balancing
**Round-robin DNS:** Return multiple IPs for the same domain — clients distribute across them.
**GeoDNS:** Return different IPs based on the client's location (user in Asia gets Asia DC IP).

---

## TCP vs UDP

Two transport protocols with fundamentally different trade-offs.

### TCP — Reliability Over Speed

```
   3-Way Handshake before any data:

   Client                              Server
     │                                   │
     │──── SYN (seq=100) ───────────────▶│
     │                                   │
     │◀─── SYN-ACK (seq=300, ack=101) ───│
     │                                   │
     │──── ACK (ack=301) ───────────────▶│
     │                                   │
     │   Connection Established!         │
     │   Now send data...               │
```

**TCP guarantees:**
- Delivery (retransmits lost packets)
- Order (reassembles out-of-order packets)
- No duplication

**TCP costs:**
- Handshake overhead (~1 RTT before data)
- Head-of-line blocking (a lost packet stalls all subsequent data)
- Higher latency than UDP

**Use TCP for:** HTTP, database connections, file transfers, email — anything where correctness matters.

### UDP — Speed Over Reliability

No handshake. No delivery guarantee. No ordering. Just send packets.

```
Client                    Server
  │                          │
  │── Packet 1 ─────────────▶│
  │── Packet 2 ─────────────▶│  (may arrive before Packet 1)
  │── Packet 3 ──────X        │  (lost, never retransmitted)
  │── Packet 4 ─────────────▶│
```

**Use UDP for:** Video streaming, gaming, DNS, VoIP — anything where speed > correctness and some packet loss is acceptable.

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection setup | Handshake required | None |
| Delivery guarantee | Yes | No |
| Ordering | Yes | No |
| Speed | Slower | Faster |
| Use case | HTTP, DB, files | Video, gaming, DNS |

---

## HTTP and HTTPS

HTTP is the application protocol that powers the web. HTTPS adds TLS encryption.

### HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|-----------|------|
| GET | Retrieve a resource | Yes | Yes |
| POST | Create a resource | No | No |
| PUT | Replace a resource | Yes | No |
| PATCH | Partially update a resource | No | No |
| DELETE | Remove a resource | Yes | No |

### HTTP Status Codes

| Range | Meaning | Examples |
|-------|---------|---------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 404 Not Found, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

### TLS Handshake (HTTPS)

```
Client                                    Server
  │── ClientHello (TLS versions, ciphers)──▶│
  │◀─ ServerHello + Certificate ────────────│
  │   (client verifies certificate)         │
  │── Key Exchange ────────────────────────▶│
  │   (both derive shared secret)           │
  │── Encrypted data ──────────────────────▶│
  │◀─ Encrypted response ───────────────────│
```

TLS adds ~1 RTT to connection setup — this is why connection reuse (keep-alive, HTTP/2 multiplexing) matters.

---

## HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.1
- One request per TCP connection (with keep-alive: sequential)
- Head-of-line blocking: request 2 waits for request 1 to complete
- Text-based protocol

### HTTP/2
- Multiple requests over one TCP connection (multiplexing)
- No head-of-line blocking at HTTP layer
- Binary protocol (more efficient)
- Server push (server sends resources before client asks)
- Header compression (HPACK)

```
HTTP/1.1:          HTTP/2:
Req 1 ──▶         Req 1 ──▶
         ◀── Res1  Req 2 ──▶  (all at once!)
Req 2 ──▶         Req 3 ──▶
         ◀── Res2            ◀── Res 1, 2, 3 (interleaved)
Req 3 ──▶
         ◀── Res3
```

### HTTP/3 (QUIC)
- Runs over UDP instead of TCP
- Built-in TLS 1.3
- Eliminates TCP head-of-line blocking
- Better performance on lossy networks (mobile)
- Connection migration (switching from WiFi to cellular without re-handshake)

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Protocol | TCP | TCP | UDP (QUIC) |
| Multiplexing | No | Yes | Yes |
| Head-of-line blocking | Yes | Partial | No |
| Header compression | No | HPACK | QPACK |
| Server push | No | Yes | Yes |
| Adoption | Universal | ~60% of web | Growing |

---

## WebSockets

HTTP is request-response. WebSockets provide persistent, bidirectional connections.

```
HTTP (request-response):              WebSocket (persistent):

Client ──request──▶ Server            Client ──upgrade──▶ Server
Client ◀──response── Server           Client ◀────────────▶ Server
Client ──request──▶ Server                    (open channel,
Client ◀──response── Server                   push data anytime)
```

**WebSocket handshake:**
```
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

**Use WebSockets for:** Chat, live feeds, multiplayer games, real-time dashboards, collaborative editing.

---

## Long Polling vs SSE vs WebSockets

When you need server-to-client data push, you have several options:

### Long Polling
Client makes a request, server holds it open until data is available, then client immediately requests again.

```
Client ──request──▶ Server (holds for up to 30s)
               ◀──data── (when available)
Client ──request──▶ Server (immediately again)
```

### Server-Sent Events (SSE)
One-directional: server pushes data over a persistent HTTP connection.

```
Client ──GET /stream──▶ Server
                ◀── event: update\ndata: {...}\n\n
                ◀── event: update\ndata: {...}\n\n  (ongoing)
```

### Comparison

| Feature | Long Polling | SSE | WebSocket |
|---------|-------------|-----|-----------|
| Direction | Bidirectional | Server → Client | Bidirectional |
| Protocol | HTTP | HTTP | WebSocket |
| Complexity | Medium | Low | Medium |
| Browser support | Universal | Universal | Universal |
| Reconnection | Manual | Automatic | Manual |
| Use case | Simple updates | Notifications, feeds | Chat, gaming |

---

## What Happens When You Type google.com?

The classic interview question — and a great end-to-end view of networking:

```
1. Browser checks cache → no cached IP

2. OS DNS lookup:
   → Checks /etc/hosts → not there
   → Queries ISP's recursive resolver
   → Resolver queries root → .com TLD → google.com NS
   → Returns: google.com = 142.250.80.46

3. Browser initiates TCP connection to 142.250.80.46:443
   → SYN → SYN-ACK → ACK (3-way handshake, ~30ms)

4. TLS handshake
   → ClientHello → ServerHello + cert → Key exchange
   → Encrypted session established (~60ms more)

5. Browser sends HTTP GET request:
   GET / HTTP/2
   Host: google.com
   Accept: text/html

6. Google's load balancer receives request
   → Routes to nearest frontend server

7. Server processes request
   → Fetches personalized search page
   → Returns HTTP 200 with HTML

8. Browser parses HTML
   → Finds <link> and <script> tags
   → Makes parallel requests for CSS, JS (via HTTP/2 multiplexing)
   → Renders page

Total time: ~300ms (first visit), ~50ms (cached)
```

---

## Key Takeaways

1. **DNS translates names to IPs** — cache TTL controls how often clients re-resolve
2. **TCP is reliable but slower** — use for anything needing correctness
3. **UDP is fast but unreliable** — use for real-time where packet loss is acceptable
4. **HTTP/2 multiplexing eliminates sequential requests** — enables parallel asset loading
5. **HTTP/3 runs over UDP via QUIC** — better performance on mobile/lossy networks
6. **WebSockets enable bidirectional real-time communication** — prefer SSE for server-only push
7. **TLS adds latency** — connection pooling and keep-alive amortize the cost

---

*Previous: [Databases — SQL vs NoSQL ←](08-databases-sql-vs-nosql.md) · Next: [API Design →](10-api-design.md)*
