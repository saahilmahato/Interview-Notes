# 📡 API Design: REST vs gRPC — Complete Cheatsheet

---

## 🧠 What is an API?

An **API (Application Programming Interface)** is a contract that defines how two systems communicate. When you design an API, you're deciding:

- **What** data can be requested/sent
- **How** it's structured
- **What protocol** is used to transfer it

---

## 🌐 REST (Representational State Transfer)

### What is REST?

REST is an **architectural style** (not a protocol) for building APIs over HTTP. It treats everything as a **resource** identified by a URL, and uses standard HTTP methods to operate on those resources.

> Think of it like a website — you visit URLs and perform actions on them.

### Core Principles (REST Constraints)

**1. Stateless** — Every request from client to server must contain all information needed to understand it. The server stores no session state between requests.

**2. Client-Server** — Client and server are independent. The client handles UI, the server handles data storage.

**3. Uniform Interface** — Consistent, standardized way to interact with resources (URLs + HTTP verbs).

**4. Cacheable** — Responses must define themselves as cacheable or non-cacheable.

**5. Layered System** — Client doesn't need to know if it's talking directly to the server or a proxy/load balancer.

**6. Code on Demand (optional)** — Server can send executable code to the client (e.g., JavaScript).

---

### HTTP Methods in REST

| Method | Purpose | Idempotent? | Safe? |
|--------|---------|-------------|-------|
| `GET` | Read a resource | ✅ Yes | ✅ Yes |
| `POST` | Create a resource | ❌ No | ❌ No |
| `PUT` | Replace a resource fully | ✅ Yes | ❌ No |
| `PATCH` | Partially update a resource | ❌ No | ❌ No |
| `DELETE` | Remove a resource | ✅ Yes | ❌ No |

> **Idempotent** = calling it multiple times has the same effect as calling it once.
> **Safe** = it doesn't modify state on the server.

---

### REST URL Design Best Practices

```
# ✅ Good REST URLs
GET    /users              → Get all users
GET    /users/42           → Get user with ID 42
POST   /users              → Create a new user
PUT    /users/42           → Replace user 42 entirely
PATCH  /users/42           → Update fields of user 42
DELETE /users/42           → Delete user 42

GET    /users/42/orders    → Get all orders for user 42
GET    /users/42/orders/7  → Get order 7 for user 42

# ❌ Bad REST URLs (avoid verbs in paths)
GET    /getUser/42
POST   /createUser
GET    /users/42/deleteUser
```

### HTTP Status Codes You Must Know

| Code | Meaning |
|------|---------|
| `200 OK` | Success |
| `201 Created` | Resource was created |
| `204 No Content` | Success, no body returned |
| `400 Bad Request` | Client sent invalid data |
| `401 Unauthorized` | Not authenticated |
| `403 Forbidden` | Authenticated but not allowed |
| `404 Not Found` | Resource doesn't exist |
| `409 Conflict` | Resource conflict (e.g., duplicate) |
| `422 Unprocessable Entity` | Validation failed |
| `429 Too Many Requests` | Rate limited |
| `500 Internal Server Error` | Server-side bug |
| `503 Service Unavailable` | Server down/overloaded |

---

### REST Request/Response Example

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "name": "Alice",
  "email": "alice@example.com"
}

---

HTTP/1.1 201 Created
Content-Type: application/json
Location: /users/99

{
  "id": 99,
  "name": "Alice",
  "email": "alice@example.com",
  "createdAt": "2025-01-01T10:00:00Z"
}
```

---

### REST Versioning Strategies

```
# 1. URL Path (most common)
GET /v1/users
GET /v2/users

# 2. Query Parameter
GET /users?version=2

# 3. Header (cleanest, but less visible)
GET /users
Accept: application/vnd.myapp.v2+json
```

---

### REST Pagination

```
# Offset-based (simple but slow at scale)
GET /users?page=2&limit=20

# Cursor-based (efficient for large datasets)
GET /users?cursor=eyJpZCI6MTAwfQ&limit=20

# Response includes next cursor
{
  "data": [...],
  "nextCursor": "eyJpZCI6MTIwfQ",
  "hasMore": true
}
```

---

## ⚡ gRPC (Google Remote Procedure Call)

### What is gRPC?

gRPC is a **high-performance RPC framework** developed by Google. Instead of thinking in resources (REST), you think in **function calls** — you call a method on a remote server as if it were a local function.

> Think of it like calling a function in your code, but that function lives on another server.

gRPC uses:
- **Protocol Buffers (protobuf)** — a binary serialization format to define services and messages
- **HTTP/2** — as the underlying transport protocol
- **Strongly typed contracts** — defined in `.proto` files

---

### Protocol Buffers (Protobuf)

This is the IDL (Interface Definition Language) you use to define your API contract.

```protobuf
// user.proto

syntax = "proto3";

package user;

// Define the service (think: "what functions are available?")
service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse); // Server streaming
}

// Define the messages (think: "what data shapes are used?")
message GetUserRequest {
  int32 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
  string created_at = 4;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
}
```

From this `.proto` file, gRPC **auto-generates** client and server code in almost any language (Go, Python, Java, Node, etc.)

---

### gRPC Communication Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Unary RPC** | Client sends one request, server replies once | Simple CRUD |
| **Server Streaming** | Client sends one request, server streams responses | Live feed, large data |
| **Client Streaming** | Client streams data, server replies once | File upload, logs |
| **Bidirectional Streaming** | Both sides stream simultaneously | Chat, real-time collab |

```protobuf
service ChatService {
  // Unary
  rpc GetMessage (GetMessageRequest) returns (Message);

  // Server streaming
  rpc StreamMessages (StreamRequest) returns (stream Message);

  // Client streaming
  rpc SendLogs (stream LogEntry) returns (LogSummary);

  // Bidirectional streaming
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

---

### gRPC vs REST: Head-to-Head Comparison

| Feature | REST | gRPC |
|---------|------|------|
| **Protocol** | HTTP/1.1 (usually) | HTTP/2 |
| **Data Format** | JSON (text) | Protobuf (binary) |
| **Contract** | OpenAPI/Swagger (optional) | `.proto` file (required) |
| **Typing** | Loose (JSON) | Strongly typed |
| **Code Generation** | Manual or via tools | Auto-generated |
| **Streaming** | Limited (SSE, WebSockets) | Native 4 types |
| **Performance** | Slower (text parsing) | Faster (~7x smaller payload) |
| **Browser Support** | ✅ Native | ❌ Needs gRPC-Web proxy |
| **Human Readable** | ✅ Yes (JSON) | ❌ No (binary) |
| **Tooling/Ecosystem** | Mature, huge | Growing |
| **Learning Curve** | Low | Higher |
| **Error Handling** | HTTP status codes | gRPC status codes |
| **Best For** | Public APIs, web/mobile | Internal microservices |

---

### gRPC Status Codes

gRPC has its own status codes (different from HTTP):

| Code | Meaning |
|------|---------|
| `OK` | Success |
| `NOT_FOUND` | Resource not found |
| `INVALID_ARGUMENT` | Bad input |
| `PERMISSION_DENIED` | Not authorized |
| `UNAUTHENTICATED` | No credentials |
| `ALREADY_EXISTS` | Duplicate resource |
| `INTERNAL` | Server error |
| `UNAVAILABLE` | Server is down |
| `DEADLINE_EXCEEDED` | Timeout |

---

### Why gRPC is Faster

**Binary vs Text:** Protobuf encodes data in binary, which is much more compact than JSON.

```
JSON:  {"id": 42, "name": "Alice"}  →  27 bytes
Proto: same data                    →  ~8 bytes
```

**HTTP/2 features:**
- Multiplexing — multiple requests over one TCP connection
- Header compression (HPACK)
- Binary framing (more efficient than HTTP/1.1 text)
- Bidirectional streaming built-in

---

### When to Use What?

**Use REST when:**
- Building a public-facing API (developers expect REST)
- Browser clients need to call the API directly
- You need human-readable requests for debugging
- Simple CRUD operations
- Your team is small and speed of development matters

**Use gRPC when:**
- Internal microservice-to-microservice communication
- Performance is critical (low latency, high throughput)
- You need streaming (real-time data, live feeds)
- Strongly typed contracts across teams is important
- You're using polyglot systems (many languages)

---

### GraphQL — Honorable Mention

Worth knowing about as a third option:

- Query language for APIs where the **client specifies exactly what data it wants**
- Avoids over-fetching (getting too much data) and under-fetching (needing multiple requests)
- Single endpoint (`POST /graphql`)
- Best for: complex data graphs, frontend-driven apps, rapid iteration

```graphql
# Client asks for exactly what it needs
query {
  user(id: 42) {
    name
    email
    orders {
      id
      total
    }
  }
}
```

---

### API Security Best Practices

**Authentication & Authorization:**
- REST: JWT tokens in `Authorization: Bearer <token>` header, OAuth 2.0, API Keys
- gRPC: Metadata headers (same concepts, different transport), TLS + token-based auth

**General Best Practices:**
- Always use HTTPS/TLS
- Rate limiting to prevent abuse
- Input validation on every endpoint
- Never expose internal error details to clients
- Use short-lived tokens + refresh token pattern
- Implement proper CORS for browser-facing APIs

---

### REST API Design Checklist ✅

```
□ Use nouns for resources, not verbs in URLs
□ Use plural nouns (/users not /user)
□ Use correct HTTP methods (don't use GET to delete)
□ Return correct HTTP status codes
□ Version your API (/v1/...)
□ Implement pagination for list endpoints
□ Provide consistent error response format
□ Document with OpenAPI/Swagger
□ Authenticate with JWT/OAuth
□ Rate limit your endpoints
□ Use HTTPS only
```

---

## 🎯 Common Interview Questions & Answers

---

**Q1: What is REST and what are its constraints?**

REST is an architectural style for distributed systems built on HTTP. The 6 constraints are: **Stateless** (no session state on server), **Client-Server** (separation of concerns), **Uniform Interface** (consistent resource identification via URLs + HTTP verbs), **Cacheable** (responses define cacheability), **Layered System** (client unaware of proxies/load balancers), and **Code on Demand** (optional, server can send executable code). The most important constraint in practice is **statelessness** — it enables horizontal scaling because any server can handle any request.

---

**Q2: What is the difference between PUT and PATCH?**

**PUT** replaces the entire resource. If you send a PUT with only a `name` field, all other fields get wiped. **PATCH** is a partial update — you only send the fields you want to change. PUT is idempotent (same result if called multiple times). PATCH technically is not guaranteed to be idempotent (though it usually is in practice). Use PATCH when you want to update one or a few fields without affecting others.

---

**Q3: What is idempotency and why does it matter in API design?**

An operation is idempotent if performing it multiple times produces the same result as performing it once. GET, PUT, DELETE are idempotent. POST is not. This matters because in distributed systems, **requests can fail and be retried**. If a client sends a DELETE request and doesn't receive a response (network failure), it can safely retry because deleting something twice has the same result as deleting once. POST is dangerous to retry naively — you might create two records. This is why payment APIs use **idempotency keys**: a unique key sent with the request so the server knows "I already processed this, return the original result."

---

**Q4: What's the difference between authentication and authorization?**

**Authentication** = Who are you? (proving identity — username/password, token) **Authorization** = What can you do? (checking permissions — can this user access this resource?). In HTTP: `401 Unauthorized` actually means **unauthenticated** (you haven't proven who you are). `403 Forbidden` means **unauthorized** (we know who you are, but you don't have permission). This is a classic confusing naming issue in HTTP.

---

**Q5: How would you version a REST API?**

Three main strategies: **URL path versioning** (`/v1/users`) — most common, easy to understand and test. **Query parameter** (`/users?version=2`) — less clean but works. **Header-based** (`Accept: application/vnd.myapp.v2+json`) — cleanest technically, but less visible and harder to test in browser. I prefer URL path versioning because it's explicit, easy to route, and immediately visible. The key principle is: **never break existing clients** — maintain old versions as long as clients depend on them.

---

**Q6: What is gRPC and how does it differ from REST?**

gRPC is an RPC framework from Google that uses **Protocol Buffers** for serialization and **HTTP/2** as transport. Key differences: REST uses JSON (text, human-readable) over HTTP/1.1 while gRPC uses Protobuf (binary, compact) over HTTP/2. REST is resource-oriented ("operate on /users"), gRPC is action-oriented ("call CreateUser()"). gRPC auto-generates type-safe client/server code from `.proto` files, while REST typically requires manual client writing. gRPC has native support for 4 streaming patterns, REST needs workarounds like SSE or WebSockets. gRPC is ~7x more efficient in payload size and faster due to binary encoding and HTTP/2 multiplexing. The tradeoff: gRPC isn't natively supported in browsers and binary messages aren't human-readable, making debugging harder.

---

**Q7: What are the 4 gRPC communication patterns?**

**Unary RPC** — classic request/response, like a normal function call. **Server streaming** — client sends one request, server streams back multiple responses (e.g., subscribing to stock prices). **Client streaming** — client streams multiple messages, server replies once (e.g., uploading a large file in chunks). **Bidirectional streaming** — both client and server stream simultaneously (e.g., a real-time chat application). All four are defined in the `.proto` file, and gRPC handles the complexity underneath.

---

**Q8: How do you handle errors in REST vs gRPC?**

In **REST**, you use HTTP status codes (400, 404, 500, etc.) and typically return a JSON body with error details:
```json
{ "error": "USER_NOT_FOUND", "message": "User with id 42 does not exist", "code": 404 }
```
In **gRPC**, you use gRPC status codes (`NOT_FOUND`, `INVALID_ARGUMENT`, `INTERNAL`, etc.) along with a status message and optional error details (structured error metadata). Both approaches should: never expose internal stack traces, use consistent error formats, and provide actionable error messages.

---

**Q9: What is over-fetching and under-fetching in REST, and how does GraphQL solve it?**

**Over-fetching** = the API returns more data than the client needs. Example: `GET /users/42` returns 20 fields but you only need `name` and `email`. **Under-fetching** = one API call doesn't return enough data, so you need multiple calls. Example: you get a user, then have to call `/users/42/orders` separately to get their orders. GraphQL solves both by letting the client specify exactly what fields it wants in a single query. The tradeoff is complexity — REST is simpler, GraphQL requires a schema, resolvers, and more sophisticated server logic.

---

**Q10: What is rate limiting and how would you implement it in a REST API?**

Rate limiting caps how many requests a client can make in a time window to prevent abuse and protect server resources. Common algorithms: **Token Bucket** (smooth out bursts, add tokens over time, consume per request), **Sliding Window** (track request count in rolling time window), **Fixed Window** (simpler, count resets at fixed intervals). Implementation: use a Redis counter keyed by user ID or IP, increment on each request, check against limit. Return `429 Too Many Requests` with `Retry-After` and `X-RateLimit-Remaining` headers when exceeded. In microservices, this is often handled at the **API Gateway** level rather than in each service.

---

**Q11: Design a REST API for a Twitter-like system.**

```
# Users
POST   /v1/users                  → Register
GET    /v1/users/:id               → Get profile
PATCH  /v1/users/:id               → Update profile

# Tweets
POST   /v1/tweets                  → Create tweet
GET    /v1/tweets/:id              → Get tweet
DELETE /v1/tweets/:id              → Delete tweet

# Timeline
GET    /v1/users/:id/timeline      → Get user's tweets
GET    /v1/feed                    → Get home feed (auth required)

# Social
POST   /v1/users/:id/follow        → Follow user
DELETE /v1/users/:id/follow        → Unfollow user
GET    /v1/users/:id/followers     → Get followers
GET    /v1/users/:id/following     → Get following

# Interactions
POST   /v1/tweets/:id/likes        → Like tweet
DELETE /v1/tweets/:id/likes        → Unlike tweet
POST   /v1/tweets/:id/retweets     → Retweet
GET    /v1/tweets/:id/replies      → Get replies
```

All list endpoints should support cursor-based pagination. Real-time features (new tweets in feed) could use either Server-Sent Events or gRPC server streaming.

---

**Q12: What is HATEOAS and do you actually need it?**

HATEOAS (Hypermedia As The Engine Of Application State) is a REST constraint where responses include links to related actions the client can take next:
```json
{
  "id": 42,
  "name": "Alice",
  "_links": {
    "self": "/users/42",
    "orders": "/users/42/orders",
    "delete": { "href": "/users/42", "method": "DELETE" }
  }
}
```
The idea is the client doesn't need to hardcode URLs — it discovers them from responses. In theory, elegant. In practice, almost nobody uses it. Most real-world APIs rely on good documentation instead. Worth knowing the concept for interviews, but don't over-engineer real APIs with it.

---

## 🗂️ Quick Reference Summary

```
REST
├── Protocol: HTTP/1.1
├── Format: JSON
├── Style: Resource-oriented (/users/42)
├── Contract: Optional (OpenAPI)
├── Streaming: Limited
├── Best for: Public APIs, web/mobile
└── Verbs: GET, POST, PUT, PATCH, DELETE

gRPC
├── Protocol: HTTP/2
├── Format: Protobuf (binary)
├── Style: Action-oriented (GetUser())
├── Contract: Required (.proto)
├── Streaming: 4 native patterns
├── Best for: Internal microservices
└── Generated clients in any language

GraphQL
├── Protocol: HTTP
├── Format: JSON
├── Style: Query-oriented
├── Contract: Schema (SDL)
├── Best for: Complex frontends, flexibility
└── Single endpoint: POST /graphql
```

---

> 💡 **Golden Rule:** Use REST for public/external APIs because developers expect it. Use gRPC for internal service-to-service communication where performance matters. Use GraphQL when your frontend teams need flexibility and you have complex, interconnected data.