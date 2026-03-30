# API Design

An API (Application Programming Interface) is the contract between your system and its consumers. Good API design enables fast integration, graceful evolution, and a great developer experience. Bad API design creates technical debt that compounds forever.

---

## Why API Design Matters in System Design

In distributed systems, services communicate through APIs. Every API decision — naming conventions, data formats, versioning strategy — becomes load-bearing infrastructure. A poorly designed API:

- Creates breaking changes that cascade across teams
- Makes debugging distributed systems harder
- Forces clients to write workarounds that never die
- Slows down product development

---

## REST — The Web's Dominant Style

REST (Representational State Transfer) is an architectural style, not a protocol. It uses HTTP methods and URLs to model operations on resources.

### Core REST Principles

**1. Resources, not actions**
```
❌ Bad (action-based):     ✅ Good (resource-based):
POST /getUser              GET /users/42
POST /createOrder          POST /orders
POST /deleteComment        DELETE /comments/99
```

**2. HTTP Methods Map to CRUD**
```
GET    /articles          → List all articles
POST   /articles          → Create a new article
GET    /articles/42       → Get article #42
PUT    /articles/42       → Replace article #42 entirely
PATCH  /articles/42       → Partially update article #42
DELETE /articles/42       → Delete article #42
```

**3. Nested resources for relationships**
```
GET /users/42/posts           → All posts by user 42
GET /users/42/posts/7         → Post #7 by user 42
POST /users/42/posts          → Create a post for user 42
```

**4. Consistent response format**
```json
{
  "data": { "id": 42, "title": "System Design 101" },
  "meta": { "request_id": "req_xyz", "timestamp": "2024-01-15T10:30:00Z" }
}
```

**5. Errors should be informative**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "field": "email",
    "doc_url": "https://api.algoroq.com/docs/errors#VALIDATION_ERROR"
  }
}
```

### REST Best Practices

**Pagination:**
```
GET /articles?page=2&per_page=20        (offset-based, simpler)
GET /articles?after=cursor_xyz&limit=20 (cursor-based, better for real-time data)

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "cursor_abc",
    "has_more": true
  }
}
```

**Filtering and sorting:**
```
GET /articles?status=published&sort=created_at&order=desc
GET /users?age_min=18&age_max=65&city=NYC
```

**API Versioning:**
```
/v1/articles     ← URL path versioning (most common, most explicit)
/articles        + Header: API-Version: 2024-01-15  ← header versioning
/articles?v=2    ← query param versioning (least recommended)
```

---

## GraphQL — Query What You Need

GraphQL lets clients specify exactly what data they want. Instead of multiple endpoints, there's one endpoint with a flexible query language.

### The Problem GraphQL Solves

```
REST — Over-fetching:
GET /users/42
Returns: { id, name, email, bio, avatar, phone, address, ... }
App only needs: id, name

REST — Under-fetching (N+1):
GET /posts          → returns 10 posts, each with user_id
GET /users/1        → fetch user for post 1
GET /users/2        → fetch user for post 2
... 10 more requests!

GraphQL — Just right:
query {
  posts {
    title
    author {
      name         ← gets exactly this, nothing more
    }
  }
}
```

### GraphQL Query Example

```graphql
# Query — read data
query GetUserWithPosts {
  user(id: "42") {
    name
    email
    posts(limit: 5) {
      title
      publishedAt
      commentCount
    }
  }
}

# Mutation — write data
mutation CreatePost {
  createPost(input: {
    title: "System Design Basics"
    body: "..."
    authorId: "42"
  }) {
    id
    title
    createdAt
  }
}

# Subscription — real-time
subscription OnNewComment {
  commentAdded(postId: "123") {
    body
    author { name }
  }
}
```

### GraphQL Trade-offs

| Advantage | Disadvantage |
|-----------|-------------|
| No over/under-fetching | Complex queries can hammer DB (N+1) |
| Strongly typed schema | Caching is harder (no URLs) |
| Great for rapid frontend iteration | Learning curve vs REST |
| Self-documenting (introspection) | Rate limiting more complex |

---

## gRPC — High-Performance Internal APIs

gRPC uses Protocol Buffers (binary format) over HTTP/2. Used for internal service-to-service communication where performance matters.

### Protocol Buffers

```protobuf
// user.proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);  // server streaming
  rpc UpdateUser (stream UserUpdate) returns (UpdateResult); // client streaming
}

message GetUserRequest {
  string user_id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}
```

**Why protobuf over JSON?**
- ~3-10x smaller payload
- ~5-10x faster serialization
- Strongly typed with schema evolution
- Code generation for all major languages

### gRPC Streaming

```
Unary:              Client ──request──▶ Server ──response──▶ Client
Server streaming:   Client ──request──▶ Server ──stream──▶ Client (ongoing)
Client streaming:   Client ──stream──▶ Server ──response──▶ Client
Bidirectional:      Client ◀──stream──▶ Server
```

---

## REST vs GraphQL vs gRPC

| Factor | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Protocol | HTTP/1.1 or 2 | HTTP/1.1 or 2 | HTTP/2 |
| Data format | JSON/XML | JSON | Protocol Buffers (binary) |
| Schema | Optional (OpenAPI) | Mandatory | Mandatory (proto) |
| Browser support | Universal | Universal | Limited (needs grpc-web) |
| Caching | Easy (HTTP cache) | Complex | Not built-in |
| Performance | Good | Good | Excellent |
| Learning curve | Low | Medium | Higher |
| Streaming | No (SSE/WS for push) | Subscriptions | Native streaming |
| Best for | Public APIs | Client-facing flexible APIs | Internal microservices |

---

## API Authentication

### API Keys
```
GET /articles
Authorization: Bearer sk_live_abc123xyz
```
Simple, stateless. Good for server-to-server. No user context.

### JWT (JSON Web Token)
```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo0Mn0.signature

Decoded payload: { "user_id": 42, "exp": 1700000000 }
```
Self-contained: server can verify without DB lookup. Suitable for user auth.

### OAuth 2.0
Standard for delegated authorization ("Login with Google").

```
User ──clicks "Login with Google"──▶ Your App
Your App ──redirect──▶ Google OAuth
User ──grants permission──▶ Google
Google ──authorization code──▶ Your App
Your App ──exchanges code──▶ Google (gets access token)
Your App ──uses access token──▶ Google APIs
```

---

## Idempotency

An operation is idempotent if performing it multiple times has the same effect as performing it once. Critical for safe retries.

```
GET    → Always idempotent (read doesn't change state)
DELETE → Idempotent (deleting twice = deleted)
PUT    → Idempotent (replacing twice = same result)
POST   → NOT idempotent by default (creating twice = two records)
PATCH  → Depends on implementation
```

### Idempotency Keys for POST
```
POST /payments
Idempotency-Key: client-generated-uuid-12345

{
  "amount": 100,
  "currency": "USD",
  "recipient_id": "user_42"
}
```

Server stores: `(idempotency_key → result)`. If the same key is received again, return the stored result without processing again. Prevents duplicate charges on retries.

---

## Rate Limiting Headers

When you rate-limit your API, communicate limits clearly:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1700000000

HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
```

---

## Backward Compatibility

Never break existing clients. Add, don't remove or rename.

```
✅ Safe changes (non-breaking):
  - Add a new optional field to response
  - Add a new endpoint
  - Add a new optional request parameter

❌ Breaking changes:
  - Remove a field from response
  - Rename a field
  - Change a field's type (string → integer)
  - Remove an endpoint
  - Change error response format
```

If you must make a breaking change, bump the API version:
- `/v1/` → `/v2/`
- Support `/v1/` for a deprecation period (minimum 6 months)
- Document migration guide
- Send deprecation notices in response headers

---

## Key Takeaways

1. **REST is resources, not actions** — use nouns in URLs, HTTP verbs for operations
2. **GraphQL solves over/under-fetching** — ideal for complex client-driven UIs
3. **gRPC excels at internal service communication** — binary protocol, streaming, strong types
4. **Idempotency keys make POST safe to retry** — essential for payments and critical operations
5. **Pagination should use cursors for real-time data** — offset-based breaks when data changes
6. **Communicate rate limits with headers** — don't silently fail clients
7. **Never break existing clients** — add fields, version when breaking changes are unavoidable

---

*Previous: [Networking Essentials ←](09-networking-essentials.md)*

*Completed the Introduction section! Next: [Intermediate System Design →](../02-intermediate-system-design/README.md)*
