# 🌐 REST API Design — Complete Notes

---

## What is REST?

REST stands for **Representational State Transfer**. It is an **architectural style** — a set of rules and constraints — for designing networked applications. It was defined by Roy Fielding in his 2000 PhD dissertation.

REST is not a library, not a framework, not a protocol. It is a **design philosophy**. When an API follows REST principles, we call it a **RESTful API**.

A REST API lets clients (browsers, mobile apps, other servers) communicate with a backend server using standard HTTP to perform operations on **resources** — things like users, products, orders.

> Think of it like a restaurant. The menu (API) lists what you can order (endpoints). You place an order (HTTP request), and the kitchen (server) brings back your food (response). REST is the rules that define how the menu is organized and how orders are placed.

---

## REST vs HTTP — What is the Difference?

This is one of the most commonly misunderstood things. People use REST and HTTP interchangeably, but they are not the same thing.

**HTTP (HyperText Transfer Protocol)** is a **communication protocol**. It defines how data is transferred between a client and a server over the internet. It specifies things like request methods (GET, POST, PUT, DELETE), headers, status codes, and the structure of messages. HTTP is the transport layer — it is the pipe.

**REST** is an **architectural style** — a set of constraints and principles for how you design your API. REST says things like "use nouns for resources", "be stateless", "use a uniform interface". REST does not care about HTTP specifically — you could theoretically build a RESTful system over FTP or another protocol. In practice though, REST is almost always implemented over HTTP because HTTP naturally maps to REST concepts.

**The key distinction:**

- HTTP answers: *How do two machines talk to each other?*
- REST answers: *How should you design your API so it is clean, scalable, and consistent?*

A REST API **uses** HTTP as its foundation but adds design rules on top of it. For example, HTTP has no opinion on whether your URL should be `/getUser` or `/users/1`. REST says it should be `/users/1` because you should use nouns, not verbs. HTTP gives you the tools. REST gives you the rules for using those tools well.

**It is also possible to use HTTP without REST.** SOAP APIs use HTTP for transport but are not RESTful. RPC-style APIs (like some older APIs that use `POST /getUser`) use HTTP but violate REST principles.

---

## The 6 Constraints of REST

REST is formally defined by 6 architectural constraints. An API is truly "RESTful" only when it satisfies all of these.

**1. Client-Server Separation**
The client and server are independent. The client handles the UI and user experience. The server handles data storage and business logic. They communicate only through the API. This lets both sides evolve independently.

**2. Stateless**
Every request from the client must contain all the information the server needs to understand and process it. The server stores no session state between requests. There are no cookies, no server-side sessions. Each request stands completely on its own.

**3. Cacheable**
Responses must explicitly label themselves as cacheable or non-cacheable. When responses are cacheable, clients and intermediaries can reuse them, improving performance and reducing server load.

**4. Uniform Interface**
The API must have a consistent, standardized way of interacting. This is the most important REST constraint. It means using standard HTTP methods correctly, using resource-based URLs, and returning consistent response structures. It is what makes an API predictable.

**5. Layered System**
The client does not need to know if it is talking directly to the server or through intermediaries like load balancers, caches, or API gateways. The system can have multiple layers and the client is unaware of them.

**6. Code on Demand (Optional)**
Servers can optionally send executable code to clients, like JavaScript. This is the only optional constraint and is rarely discussed in the context of REST APIs.

---

## Core Concepts

### Resources

In REST, everything is a **resource**. A user is a resource. A product is a resource. An order is a resource. Resources are always **nouns**, never verbs.

```
✅  /users
✅  /orders/42
✅  /products/7/reviews

❌  /getUser
❌  /createOrder
❌  /deleteProduct
```

Resources are identified by **URIs (Uniform Resource Identifiers)** — the URL path.

### Representations

A resource is an abstract thing. What you send over the wire is a **representation** of that resource. The same resource can be represented in multiple formats — JSON, XML, HTML. JSON is the modern standard for REST APIs.

```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com"
}
```

### HTTP Methods

HTTP methods (also called verbs) express the action you want to perform on a resource.

| Method | Action | Example |
|---|---|---|
| GET | Read / fetch a resource | GET /users/1 |
| POST | Create a new resource | POST /users |
| PUT | Replace a resource entirely | PUT /users/1 |
| PATCH | Partially update a resource | PATCH /users/1 |
| DELETE | Delete a resource | DELETE /users/1 |

### Idempotency and Safety

**Safe** methods do not modify server state. GET is safe — reading data changes nothing.

**Idempotent** methods can be called multiple times and always produce the same result. DELETE /users/1 called ten times has the same effect as calling it once — the user is deleted.

| Method | Safe | Idempotent |
|---|---|---|
| GET | ✅ Yes | ✅ Yes |
| POST | ❌ No | ❌ No |
| PUT | ❌ No | ✅ Yes |
| PATCH | ❌ No | ❌ No |
| DELETE | ❌ No | ✅ Yes |

This matters because clients that experience a network failure need to know if it is safe to retry a request. Retrying a GET is always fine. Retrying a POST could create duplicates.

---

## URI Design Best Practices

**Use nouns, not verbs**
```
✅  GET /articles
❌  GET /getArticles
```

**Use plural nouns for collections**
```
✅  /users
✅  /products
❌  /user
```

**Use hierarchies to express relationships**
```
GET /users/42/orders        → all orders belonging to user 42
GET /users/42/orders/7      → order 7 belonging to user 42
```

**Use kebab-case (hyphens) in URLs, not camelCase or underscores**
```
✅  /blog-posts
❌  /blogPosts
❌  /blog_posts
```

**Keep URLs lowercase**
```
✅  /users
❌  /Users
```

**Never use file extensions**
```
✅  /products
❌  /products.json
```

**Use query strings for filtering, sorting, searching, and pagination**
```
GET /products?category=electronics&sort=price&order=asc&page=2&limit=20
GET /users?q=john&status=active
```

---

## HTTP Status Codes

Status codes communicate the outcome of a request. Use them correctly — never return `200 OK` with an error message in the body.

### 2xx — Success

| Code | Meaning | When to use |
|---|---|---|
| 200 OK | Request succeeded | GET, PUT, PATCH responses |
| 201 Created | Resource was created | Successful POST that creates something |
| 204 No Content | Succeeded but nothing to return | DELETE responses |

### 3xx — Redirection

| Code | Meaning |
|---|---|
| 301 Moved Permanently | Resource URL has permanently changed |
| 304 Not Modified | Client's cached version is still valid |

### 4xx — Client Errors

| Code | Meaning | When to use |
|---|---|---|
| 400 Bad Request | Malformed request syntax | Missing required fields, invalid JSON |
| 401 Unauthorized | Not authenticated | No token, expired token, invalid token |
| 403 Forbidden | Authenticated but not authorized | Valid token but insufficient permissions |
| 404 Not Found | Resource does not exist | Wrong ID or path |
| 405 Method Not Allowed | Wrong HTTP verb | Using DELETE on a read-only endpoint |
| 409 Conflict | State conflict | Trying to register with an email already in use |
| 422 Unprocessable Entity | Validation failed | Fields are present but invalid (e.g. age = -5) |
| 429 Too Many Requests | Rate limit exceeded | Client is making too many requests |

### 5xx — Server Errors

| Code | Meaning |
|---|---|
| 500 Internal Server Error | Something crashed on the server |
| 502 Bad Gateway | Upstream service failed |
| 503 Service Unavailable | Server is down or overloaded |

---

## Request and Response Design

### Request

```http
POST /users
Content-Type: application/json
Authorization: Bearer eyJhbGci...

{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "password": "secret123"
}
```

### Consistent Response Format

Pick a response structure and use it everywhere. This is what consistency in a "uniform interface" looks like in practice.

**Success:**
```json
{
  "status": "success",
  "data": {
    "id": 1,
    "name": "Jane Doe",
    "email": "jane@example.com"
  }
}
```

**Error:**
```json
{
  "status": "error",
  "code": 422,
  "message": "Validation failed",
  "errors": [
    { "field": "email", "message": "Email is already in use" },
    { "field": "password", "message": "Password must be at least 8 characters" }
  ]
}
```

Never expose internal stack traces, SQL errors, or database details in error responses. That is a security risk.

---

## Pagination

Never return all records at once. Always paginate large collections.

**Offset-based pagination** — simple and allows jumping to any page, but can miss or duplicate records when data changes mid-query.
```
GET /products?page=2&limit=20
```
```json
{
  "data": [...],
  "pagination": {
    "total": 200,
    "page": 2,
    "limit": 20,
    "totalPages": 10
  }
}
```

**Cursor-based pagination** — uses a pointer to the last seen item. More stable for real-time or frequently changing data, but you cannot jump to arbitrary pages.
```
GET /posts?cursor=eyJpZCI6MTAwfQ==&limit=20
```
```json
{
  "data": [...],
  "nextCursor": "eyJpZCI6MTIwfQ=="
}
```

Use offset pagination for admin panels and dashboards. Use cursor pagination for feeds, timelines, and large datasets.

---

## Versioning

APIs change over time. Versioning lets you make breaking changes without breaking existing clients.

**URL Versioning** — most common and recommended
```
/api/v1/users
/api/v2/users
```

**Header Versioning**
```
Accept: application/vnd.myapi.v2+json
```

**Query Parameter Versioning**
```
/users?version=2
```

URL versioning is the most explicit. It is easy to test in a browser, easy to route in code, and immediately obvious to developers reading the URL. It is the industry standard.

---

## Authentication and Authorization

**Authentication** means: who are you?
**Authorization** means: what are you allowed to do?

**API Keys** — a static key sent in a header. Simple and good for server-to-server communication.
```
X-API-Key: your-secret-key
```

**Bearer Tokens / JWT** — the most common pattern for REST APIs. A token is issued on login and sent with every subsequent request.
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**OAuth 2.0** — delegated authorization. Used when your app needs to access another service on behalf of a user (e.g. "Login with Google", "Connect your GitHub account").

**Basic Auth** — base64-encoded username and password in the header. Acceptable only over HTTPS and generally not recommended for production APIs.

---

## Rate Limiting

Protect your API from abuse and overload by limiting how many requests a client can make in a time window. When exceeded, return `429 Too Many Requests`.

Include rate limit information in response headers so clients can behave accordingly:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1700000000
```

---

## Idempotency Keys

For non-idempotent operations like POST (e.g. processing a payment), clients can include an idempotency key to make retries safe. If the server sees the same key twice, it returns the original response without repeating the action.

```
POST /payments
Idempotency-Key: a8098c1a-f86e-11da-bd1a-00112444be1e
```

This is critical in payment, order, and email systems where duplicate actions are catastrophic.

---

## HATEOAS

HATEOAS stands for **Hypermedia As The Engine Of Application State**. It is a REST constraint where responses include hyperlinks to related actions and resources, making the API self-discoverable.

```json
{
  "id": 1,
  "name": "Jane Doe",
  "_links": {
    "self": { "href": "/users/1" },
    "orders": { "href": "/users/1/orders" },
    "delete": { "href": "/users/1", "method": "DELETE" }
  }
}
```

Most real-world APIs do not implement full HATEOAS, but including relevant links is considered a good practice.

---

## Handling Actions That Don't Fit CRUD

Sometimes you need to express an action that does not map cleanly to create, read, update, or delete — like "publish a post" or "send a verification email".

The cleanest approach is to model the action as a sub-resource:
```
POST /posts/42/publish
POST /users/1/email-verification/resend
POST /orders/99/cancel
```

Alternatively, for simple state transitions, use PATCH:
```
PATCH /posts/42
{ "status": "published" }
```

---

## Quick Reference Cheat Sheet

```
POST    /users            →  Create user           →  201 Created
GET     /users            →  List all users         →  200 OK
GET     /users/:id        →  Get one user           →  200 OK  /  404 Not Found
PUT     /users/:id        →  Replace user entirely  →  200 OK
PATCH   /users/:id        →  Partially update user  →  200 OK
DELETE  /users/:id        →  Delete user            →  204 No Content

GET  /users?role=admin                    →  Filter
GET  /users?sort=name&order=desc          →  Sort
GET  /users?page=1&limit=20              →  Paginate
GET  /users?q=john                        →  Search
```

---

## Interview Questions and Answers

---

**Q1. What is REST and what makes an API RESTful?**

REST is an architectural style defined by Roy Fielding with 6 constraints: client-server separation, statelessness, cacheability, uniform interface, layered system, and optional code-on-demand. An API is RESTful when it follows these constraints — using standard HTTP methods on resource-based URLs, returning consistent responses, and treating each request as self-contained with no server-side session state. The most important constraints in practice are statelessness and the uniform interface.

---

**Q2. What is the difference between REST and HTTP?**

HTTP is a communication protocol — it defines how data is transferred between a client and a server. REST is an architectural style — a set of design principles for building APIs. REST does not require HTTP, but in practice REST APIs are always built on top of HTTP because HTTP naturally maps to REST concepts (methods map to actions, URLs map to resources, status codes communicate outcomes). You can use HTTP without REST — SOAP and RPC-style APIs use HTTP as a transport without being RESTful. Think of it this way: HTTP is the pipe, REST is the set of rules for how to use that pipe well.

---

**Q3. What is the difference between PUT and PATCH?**

PUT replaces the entire resource. If you send a PUT request with missing fields, those fields are wiped. PATCH performs a partial update — only the fields you include are changed. If a user has `name`, `email`, and `age`, and you PATCH with `{ "age": 30 }`, only age is updated. If you PUT with `{ "age": 30 }`, name and email would be removed. PUT is idempotent. PATCH is not guaranteed to be idempotent.

---

**Q4. What is the difference between 401 and 403?**

`401 Unauthorized` means the client is not authenticated — they have not provided credentials, or their token is missing, invalid, or expired. `403 Forbidden` means the client is authenticated but does not have permission to access the resource. Simple memory trick: 401 means "I don't know who you are." 403 means "I know who you are, but you cannot come in."

---

**Q5. What is statelessness in REST and why does it matter?**

Statelessness means the server stores no session state between requests. Every request must include all the information needed to process it — authentication tokens, user context, everything. The server is essentially amnesiac — it has no memory of previous requests. This matters because it makes the API horizontally scalable. Any server instance can handle any request since none of them store session data. It also makes the API more predictable and easier to debug.

---

**Q6. What is idempotency? Which HTTP methods are idempotent?**

An operation is idempotent if calling it multiple times produces the same result as calling it once. GET, PUT, and DELETE are idempotent. POST and PATCH are not. This matters for reliability — if a client retries a GET or DELETE because of a network failure, it is safe. But retrying a POST could create duplicate records. To make POST operations safe to retry, you can use idempotency keys.

---

**Q7. How would you handle API versioning?**

The three main approaches are URL versioning (`/api/v1/`), header versioning (`Accept: application/vnd.api.v2+json`), and query parameter versioning (`?version=2`). I prefer URL versioning because it is explicit, easy to test in a browser, and easy to route. The goal is to never introduce breaking changes in an existing version. When you must break the contract, bump the version, clearly deprecate the old one with a sunset date, and provide migration documentation.

---

**Q8. How do you design pagination for a REST API?**

Two main approaches exist. Offset-based pagination uses page and limit query parameters — it is simple and allows jumping to any page, but can return duplicates or miss records when data changes between requests. Cursor-based pagination uses a pointer to the last seen item — it is more stable for real-time or frequently updated data but does not allow jumping to arbitrary pages. I use offset for admin dashboards where data is relatively stable, and cursor-based for feeds and large datasets.

---

**Q9. How do you handle errors properly in a REST API?**

Use the correct HTTP status code for every situation — 4xx for client errors, 5xx for server errors. Return a consistent error response body with a message, an error code, and optionally field-level validation errors. Never return 200 OK with an error in the body, as this defeats the purpose of status codes and breaks API clients. Never expose internal stack traces, SQL queries, or database details in error responses as this is a serious security risk.

---

**Q10. What is HATEOAS?**

HATEOAS stands for Hypermedia As The Engine Of Application State. It is a REST constraint where API responses include hyperlinks to related resources and available actions, making the API self-discoverable. Clients do not need to hardcode URLs — they simply follow links in responses. In practice, very few production APIs implement full HATEOAS because of the complexity it adds, but including relevant links in responses is a good design practice.

---

**Q11. How would you design an endpoint for an action like "publish a post"?**

Publishing does not map cleanly to standard CRUD operations. The most common clean approach is to model it as a sub-resource: `POST /posts/42/publish`. This clearly communicates intent. Alternatively, for simpler state transitions, you can use PATCH: `PATCH /posts/42` with `{ "status": "published" }`. I prefer the sub-resource approach when the action has meaningful side effects like sending notifications or triggering background jobs, and PATCH when it is a simple state field change.

---

**Q12. What is the difference between REST and GraphQL?**

REST uses multiple endpoints — one per resource — and the server decides what data each endpoint returns. GraphQL uses a single endpoint where the client specifies exactly what data it needs, eliminating over-fetching (getting more data than needed) and under-fetching (needing multiple requests to get related data). REST is simpler, widely understood, and the right default for most APIs. GraphQL excels when clients have very different data requirements (e.g., mobile needs less data than web) or when you need to aggregate data from multiple resources efficiently. They are complementary — many systems use both.

---

**Q13. How do you secure a REST API?**

Security needs to be layered. Use HTTPS for all traffic to encrypt data in transit. Use JWT Bearer tokens or OAuth 2.0 for authentication. Validate authorization on every request — never trust the client. Implement rate limiting to prevent abuse and DDoS. Validate and sanitize all inputs to prevent injection attacks. Configure CORS to restrict which origins can call your API. Never return sensitive data like passwords, full card numbers, or internal system details in responses. Log suspicious activity and set up alerting.

---