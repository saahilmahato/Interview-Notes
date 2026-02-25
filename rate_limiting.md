# 🚦 Rate Limiting — Complete Notes

---

## What is Rate Limiting?

Rate limiting is a technique used to **control how many requests a client can make to a server within a given time window**. It protects your system from being overwhelmed by too many requests — whether from a single user, a bot, or a denial-of-service attack.

Think of it like a nightclub bouncer: you can enter, but only so many people get in per hour. Once the limit is hit, you wait or get turned away.

---

## Why Do We Need Rate Limiting?

- **Prevent abuse** — stop bad actors from hammering your API
- **Ensure fair usage** — one user shouldn't starve others of resources
- **Protect backend infrastructure** — databases, services have limits
- **Cost control** — especially on pay-per-use services (LLMs, SMS, etc.)
- **Prevent DDoS attacks** — slow down or block flood attacks
- **SLA enforcement** — give different tiers different limits (free vs paid)

---

## Key Terminology

| Term | Meaning |
|---|---|
| **Limit** | Max number of requests allowed |
| **Window** | The time period for the limit (e.g., 1 minute) |
| **Throttling** | Slowing down requests instead of rejecting them |
| **Quota** | A longer-term limit (e.g., 10,000 requests/day) |
| **Burst** | Allowing temporarily exceeding the limit for short spikes |
| **429 Too Many Requests** | The HTTP status code returned when rate limited |

---

## Rate Limiting Algorithms

### 1. 🪣 Token Bucket
The most popular algorithm. Imagine a bucket that holds tokens:
- Tokens are **added at a fixed rate** (e.g., 10 tokens/second)
- Each request **consumes one token**
- If the bucket is empty → request is **rejected**
- Bucket has a **max capacity** — tokens don't overflow

**Pros:** Allows bursting up to bucket size. Smooth and flexible.  
**Cons:** Slightly complex to implement in distributed systems.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/sec

Request comes in → take 1 token
No token available → reject / wait
```

---

### 2. 🚰 Leaky Bucket
Requests flow in at any rate but **leak out (are processed) at a fixed rate**:
- Think of a bucket with a hole at the bottom
- Incoming requests fill the bucket
- Processed at a constant rate regardless of input
- If bucket overflows → requests are dropped

**Pros:** Smooths out bursty traffic. Predictable output rate.  
**Cons:** Doesn't handle bursts well — excess is always dropped.

---

### 3. 🔢 Fixed Window Counter
Divide time into fixed windows (e.g., every 60 seconds). Count requests in each window:

```
Window: 12:00:00 → 12:01:00 — allow 100 requests
At 100 requests → reject until 12:01:00
Window resets → counter goes back to 0
```

**Pros:** Simple and fast. Easy to implement with Redis `INCR + EXPIRE`.  
**Cons:** **Edge case problem** — a user can send 100 requests at 12:00:59 and 100 at 12:01:01, getting 200 requests in 2 seconds.

---

### 4. 📊 Sliding Window Log
Store a **timestamp log** of every request. On each new request:
- Remove timestamps older than the window
- Count remaining timestamps
- If count < limit → allow and add new timestamp

```
Limit: 5 requests per minute
Log: [12:00:10, 12:00:20, 12:00:40, 12:00:55]
New request at 12:01:05 → remove 12:00:10 (older than 1 min from now)
Count = 3 → allow
```

**Pros:** Very accurate. No boundary edge case.  
**Cons:** High memory usage — stores every request's timestamp.

---

### 5. 🪟 Sliding Window Counter (Hybrid)
A **practical middle ground** between Fixed Window and Sliding Window Log:

- Use two counters: current window and previous window
- Weight the previous window based on how far into the current window you are

```
Rate = prev_count × ((window_size - elapsed) / window_size) + curr_count
```

**Pros:** Low memory, accurate, fast.  
**Cons:** Approximate (not 100% precise).

---

## Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Complexity |
|---|---|---|---|---|
| Token Bucket | Low | High | ✅ Yes | Medium |
| Leaky Bucket | Low | High | ❌ No | Medium |
| Fixed Window | Very Low | Low | ✅ Yes | Easy |
| Sliding Window Log | High | Very High | ✅ Yes | Hard |
| Sliding Window Counter | Low | High | ✅ Yes | Medium |

---

## Where to Enforce Rate Limits?

```
Client → [API Gateway / Load Balancer] → [Service] → [Database]
              ↑
        Best place to rate limit (centralized)
```

- **API Gateway** (e.g., Kong, AWS API Gateway, Nginx) — most common
- **Middleware** in your web framework (Express, Django, etc.)
- **Application layer** — in code itself
- **Service mesh** (e.g., Istio) — for microservice-to-service limits

---

## Rate Limiting in Distributed Systems

Single-server rate limiting is easy. In a distributed system (multiple servers), it gets tricky.

**The Problem:**
```
User sends 10 requests → split across 3 servers
Server A sees 4, Server B sees 3, Server C sees 3
Each thinks the user is fine → total 10 slips through undetected
```

**Solutions:**

**1. Centralized Counter (Redis)**
- All servers talk to a single Redis instance
- Use atomic operations: `INCR`, `EXPIRE`
- Very common in production

```redis
INCR user:123:requests
EXPIRE user:123:requests 60
```

**2. Sticky Sessions**
- Route the same user to the same server always
- Simple but breaks with server failures

**3. Approximate Algorithms**
- Accept slight inaccuracy for performance gains
- Sliding window counter works well here

---

## Rate Limit Keys (What to Limit By?)

You can rate limit by different identifiers:

- **IP address** — simple, but breaks with shared IPs (offices, NAT)
- **User ID** — more accurate, requires authentication
- **API Key** — common for developer APIs
- **Device ID** — for mobile apps
- **Endpoint** — e.g., `/login` gets stricter limits than `/feed`
- **Combination** — e.g., (user + endpoint)

---

## HTTP Headers for Rate Limiting

Best practice: tell the client about their limits via response headers.

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1693000860   ← Unix timestamp when window resets
Retry-After: 30                 ← Seconds to wait (sent with 429)
```

When limit is exceeded:
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/json

{ "error": "Rate limit exceeded. Try again in 60 seconds." }
```

---

## Rate Limiting vs Throttling

| | Rate Limiting | Throttling |
|---|---|---|
| **What happens** | Request is **rejected** (429) | Request is **slowed down / queued** |
| **User experience** | Hard stop | Gradual degradation |
| **Use case** | API protection | Streaming, background jobs |

---

## Common Rate Limiting Patterns

**1. Tiered Limits (by plan)**
```
Free tier:     100 requests/hour
Pro tier:    1,000 requests/hour
Enterprise: 10,000 requests/hour
```

**2. Endpoint-Specific Limits**
```
POST /login       → 5 requests/minute  (sensitive)
GET  /feed        → 100 requests/minute (read-heavy)
POST /send-email  → 10 requests/minute
```

**3. Penalty Box / Backoff**
If a user keeps violating limits, exponentially increase their lockout time.

**4. Graceful Degradation**
Instead of hard rejecting, serve cached/stale data under high load.

---

## Implementation Example (Node.js + Redis)

```javascript
const redis = require('redis');
const client = redis.createClient();

async function rateLimiter(userId, limit = 100, windowSecs = 60) {
  const key = `rate:${userId}`;
  
  const current = await client.incr(key);
  
  if (current === 1) {
    // First request in window, set expiry
    await client.expire(key, windowSecs);
  }
  
  if (current > limit) {
    throw new Error('Rate limit exceeded');
  }
  
  return { remaining: limit - current };
}
```

---

## System Design Considerations

When asked "design a rate limiter" in interviews, cover these:

1. **Requirements** — per user? per IP? per endpoint? global?
2. **Algorithm choice** — token bucket for flexibility, sliding window for accuracy
3. **Storage** — Redis (fast, atomic, TTL support, distributed)
4. **Where to place it** — API Gateway (before it hits your services)
5. **Failure mode** — if Redis goes down, do you fail open (allow all) or fail closed (block all)? Usually fail open to avoid downtime.
6. **Synchronization** — in distributed systems, use Lua scripts in Redis for atomic operations
7. **Headers** — always return rate limit info in response headers
8. **Monitoring** — track how often limits are hit, alert on spikes

---

---

# 🎯 Common Interview Questions & Answers

---

**Q1. What is rate limiting and why is it important?**

Rate limiting controls how many requests a client can make within a time window. It's important to protect backend systems from overload, prevent abuse and DDoS attacks, ensure fair resource distribution among users, and enforce business rules like API pricing tiers.

---

**Q2. What is the difference between the Token Bucket and Leaky Bucket algorithms?**

Token Bucket allows **burst traffic** up to the bucket's capacity. Tokens accumulate over time and each request consumes one. This is flexible and popular for APIs.

Leaky Bucket processes requests at a **constant outflow rate** regardless of how fast they arrive. Excess requests are dropped or queued. This smooths traffic but doesn't handle bursts well.

In practice, Token Bucket is more commonly used for API rate limiting because it's more forgiving of occasional bursts.

---

**Q3. What is the problem with Fixed Window rate limiting?**

The **boundary/edge case problem**. If the limit is 100 req/min, a user can send 100 requests at 12:00:59 and another 100 at 12:01:01 — getting 200 requests in just 2 seconds. The counter resets exactly on the minute boundary, allowing double the intended rate at the seam between windows.

Sliding window algorithms solve this problem.

---

**Q4. How would you implement rate limiting in a distributed system?**

The main challenge is that multiple servers need to share state. The most common solution is a **centralized Redis store**. All servers call Redis atomically using `INCR` + `EXPIRE`. Because Redis is single-threaded for commands, there are no race conditions. For even stronger atomicity, use **Lua scripts** in Redis to combine the check and increment into one atomic operation. Avoid per-server counters as they can allow users to exceed limits by distributing requests across nodes.

---

**Q5. What happens when the rate limiter itself goes down?**

This is a **fail-open vs fail-closed** decision:

- **Fail open** (allow all traffic) — service remains available, but you lose protection temporarily. Preferred for most consumer-facing APIs where availability is critical.
- **Fail closed** (block all traffic) — safer from abuse but your service goes down with the limiter. Preferred for highly sensitive endpoints like financial transactions.

You should also add **health checks, replication, and circuit breakers** around your rate limiter to minimize downtime.

---

**Q6. How do you rate limit by user when the user isn't authenticated (e.g., on a login endpoint)?**

Use **IP address** as the key. However, be careful: many users behind a corporate NAT share one IP. A better approach for login endpoints is a **combination**: rate limit by IP and also by the target username being logged into, preventing credential stuffing even from different IPs.

---

**Q7. What's the difference between rate limiting and throttling?**

Rate limiting **hard rejects** requests once the limit is hit, returning a 429 error. The client must wait.

Throttling **slows down** or queues requests rather than rejecting them outright. The client eventually gets a response, just slower. Throttling is more graceful but requires more server resources to queue pending requests.

---

**Q8. Design a rate limiter for a public API. Walk me through your design.**

A strong answer covers:

**Requirements clarification:** Are we limiting per user, IP, or API key? Per endpoint or globally? What's the scale (millions of users)?

**Algorithm:** Token Bucket — allows reasonable bursts, widely understood, easy to implement.

**Storage:** Redis — in-memory (fast), supports atomic `INCR`, has built-in key expiry, can be clustered for HA.

**Placement:** API Gateway layer — before requests reach your microservices, so even blocked requests don't consume application resources.

**Response:** Return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers on every response. Return `429` with `Retry-After` when limited.

**Failure mode:** Fail open — if Redis is unreachable, allow requests through and alert on-call.

**Tiering:** Store limits per API key in a config store/database. Look up the user's tier on each request (cached in Redis itself).

**Monitoring:** Track 429 rate per endpoint, alert if it spikes abnormally.

---

**Q9. What HTTP status code does rate limiting return and what headers should you send?**

Return **`429 Too Many Requests`**. Headers to include:
- `X-RateLimit-Limit` — the max requests allowed
- `X-RateLimit-Remaining` — how many are left in the current window
- `X-RateLimit-Reset` — Unix timestamp when the window resets
- `Retry-After` — seconds the client should wait before retrying

---

**Q10. How does the Sliding Window Counter algorithm work?**

It's a hybrid of Fixed Window and Sliding Window Log. You maintain counters for the **current** and **previous** fixed windows. When a new request comes in, you calculate a weighted estimate:

```
estimated_count = prev_count × (1 - elapsed_fraction) + curr_count
```

Where `elapsed_fraction` is how far into the current window you are. If this estimated count exceeds the limit, reject the request. This gives high accuracy with very low memory cost — just two counters per user rather than a full timestamp log.

---

**Q11. What are some real-world examples of rate limiting?**

- **Twitter/X API** — limits per endpoint (e.g., 500k tweets/month on free tier)
- **GitHub API** — 5,000 requests/hour for authenticated users
- **Stripe** — 100 read requests/second per secret key
- **AWS** — API Gateway throttling per stage and method
- **Cloudflare** — DDoS protection with automatic IP-based rate limiting
- **Google Maps API** — limits by requests per day tied to billing

---

> 💡 **Quick Memory Aid for Algorithms:**
> - **Token Bucket** = Fill up, spend tokens, bursty is OK
> - **Leaky Bucket** = Constant drip out, smooths everything
> - **Fixed Window** = Simple counter, resets on clock, edge case risk
> - **Sliding Log** = Super accurate, memory hungry
> - **Sliding Counter** = Best of both worlds, approximate but practical