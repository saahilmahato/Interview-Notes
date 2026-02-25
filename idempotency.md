# 🔁 Idempotency — Complete Notes

---

## What is Idempotency?

**Idempotency** means: **performing the same operation multiple times produces the same result as doing it once.**

> "Do it once. Do it 100 times. The outcome is the same."

The word comes from Latin: *idem* (same) + *potens* (power).

---

## Simple Analogy

Think of an **elevator button**. If you press the "Floor 5" button 10 times, you still go to Floor 5. Pressing it again doesn't send you to Floor 50. That's idempotency.

Now think of a **vending machine**. Each press of the button dispenses one item. Press 10 times = 10 items. That is **NOT idempotent**.

---

## Formal Definition

An operation `f` is idempotent if:

```
f(f(x)) = f(x)
```

Or more generally:

```
f(x) applied N times = f(x) applied once
```

---

## Idempotency in HTTP Methods

HTTP methods have well-defined idempotency characteristics:

| Method | Idempotent? | Safe? | Description |
|--------|-------------|-------|-------------|
| `GET` | ✅ Yes | ✅ Yes | Just reads data. Same result every time. |
| `HEAD` | ✅ Yes | ✅ Yes | Like GET but no body. |
| `PUT` | ✅ Yes | ❌ No | Replaces a resource. Same data = same state. |
| `DELETE` | ✅ Yes | ❌ No | Delete once or 100 times — it's gone. |
| `POST` | ❌ No | ❌ No | Creates new resources each time. |
| `PATCH` | ❌ No* | ❌ No | Depends on implementation. |

> **Safe** = no side effects (doesn't change server state). All safe operations are idempotent, but not all idempotent operations are safe.

### Why is DELETE idempotent?

```
DELETE /users/123  → 200 OK (user deleted)
DELETE /users/123  → 404 Not Found (already deleted)
```

The **state of the server is the same** — user 123 doesn't exist. The response code may differ, but the *outcome* is identical.

### Why is POST NOT idempotent?

```
POST /orders  → Creates Order #1
POST /orders  → Creates Order #2  ← Different outcome!
```

---

## Idempotency in Programming

### Non-idempotent function:
```python
counter = 0

def increment():
    global counter
    counter += 1  # Each call changes state differently
```

### Idempotent function:
```python
def set_active(user):
    user.status = "active"  # Setting to active 100 times = same result
```

### Another example — setting vs incrementing:
```sql
-- NOT idempotent
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
-- Running twice adds 200!

-- Idempotent
UPDATE accounts SET balance = 500 WHERE id = 1;
-- Running twice still results in balance = 500
```

---

## Idempotency Key

When you **can't make an operation naturally idempotent** (like POST), you use an **Idempotency Key** — a unique token sent with the request.

### How it works:
1. Client generates a unique ID (UUID) for the request.
2. Client sends it as a header: `Idempotency-Key: uuid-1234`
3. Server stores the key + response.
4. If the same key comes again → server returns the **cached response**, doesn't re-execute.
5. After a TTL (e.g., 24 hours), the key expires.

```
Client                          Server
  |                               |
  |-- POST /pay (Key: abc-123) -->|
  |                               |-- Process payment
  |                               |-- Store {abc-123: success}
  |<-- 200 OK (charged $100) -----|
  |                               |
  [Network timeout, retry]        |
  |                               |
  |-- POST /pay (Key: abc-123) -->|
  |                               |-- Key exists! Return cached response
  |<-- 200 OK (charged $100) -----|
  |                               |
  [User NOT double-charged ✅]
```

---

## 🏦 Idempotency from a Database Perspective

### The Core Problem
Databases are the source of truth. Any duplicate write operation (due to retries, bugs, or network glitches) can corrupt data.

### Techniques to Achieve Idempotency in Databases:

#### 1. Upsert (INSERT or UPDATE)
```sql
-- If row exists → update. If not → insert. Same end state.
INSERT INTO users (id, email, status)
VALUES (1, 'user@example.com', 'active')
ON CONFLICT (id)
DO UPDATE SET email = EXCLUDED.email, status = EXCLUDED.status;
```

#### 2. Deduplication Table
```sql
-- Store processed request IDs
CREATE TABLE idempotency_keys (
    key         VARCHAR(255) PRIMARY KEY,
    response    JSONB,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Before processing, check:
SELECT response FROM idempotency_keys WHERE key = 'req-uuid-123';
-- If exists → return stored response
-- If not    → process, then INSERT into this table
```

#### 3. Conditional Updates
```sql
-- Only update if status hasn't changed
UPDATE orders
SET status = 'shipped'
WHERE id = 42 AND status = 'pending';
-- Running again: 0 rows updated, but state is still correct
```

#### 4. Natural Idempotency via Absolute Values
```sql
-- NOT idempotent (relative change)
UPDATE wallet SET balance = balance - 50 WHERE user_id = 1;

-- Idempotent (absolute value)
UPDATE wallet SET balance = 950 WHERE user_id = 1 AND balance = 1000;
```

#### 5. Optimistic Locking with Versioning
```sql
UPDATE accounts
SET balance = 900, version = version + 1
WHERE id = 1 AND version = 5;
-- If version doesn't match → reject (stale request)
```

---

## 💳 Idempotency from a Fintech Perspective

Fintech is where idempotency is **life or death**. A duplicate payment request = double charge = angry customer + compliance nightmare.

### Real-world Fintech Scenarios:

#### Scenario 1: Payment Processing
```
User clicks "Pay $100" → Request sent
→ Server processes payment
→ Network timeout before response
→ App retries the request
→ WITHOUT idempotency: User charged $200 ❌
→ WITH idempotency key: User charged $100 ✅
```

#### Scenario 2: Bank Transfer
```
POST /transfers
{
  "from": "acc_A",
  "to": "acc_B",
  "amount": 500,
  "idempotency_key": "txn-uuid-9876"
}
```
If this request is retried 3 times, money moves only **once**.

#### Scenario 3: Refunds
```
POST /refunds
{
  "charge_id": "ch_abc123",
  "amount": 50,
  "idempotency_key": "refund-uuid-xyz"
}
```
Same refund request retried = refund processed once, not three times.

---

### How Stripe Implements Idempotency

Stripe is the gold standard here. They require an `Idempotency-Key` header for all POST requests.

```bash
curl https://api.stripe.com/v1/charges \
  -H "Idempotency-Key: my-unique-key-123" \
  -d amount=2000 \
  -d currency=usd \
  -d source=tok_visa
```

- Keys are stored for **24 hours**.
- Same key = same response returned (no new charge).
- Different key = new operation.

---

### Fintech Idempotency Architecture

```
                    ┌──────────────────────────────────┐
Client Request      │          API Gateway             │
(with idem key) ───►│  Check Redis/DB for key          │
                    │   Key exists? → Return cached    │
                    │   Key new?   → Forward request   │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │       Payment Service            │
                    │  - Process transaction           │
                    │  - Lock account (pessimistic)    │
                    │  - Debit/Credit atomically       │
                    │  - Store result with idem key    │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │         Database                 │
                    │  - ACID transactions             │
                    │  - Unique constraints            │
                    │  - Idempotency keys table        │
                    └──────────────────────────────────┘
```

---

### Challenges in Fintech Idempotency

**1. Partial Failures**
What if payment succeeded but storing the idempotency key failed? You need to wrap both in a single **database transaction**.

```sql
BEGIN;
  INSERT INTO transactions (id, amount, ...) VALUES (...);
  INSERT INTO idempotency_keys (key, response) VALUES (...);
COMMIT;
-- Either both succeed or neither does
```

**2. Distributed Systems**
Multiple servers need to share idempotency key state → use **Redis** or a **distributed cache** with atomic operations.

```python
# Atomic check-and-set in Redis
result = redis.set(
    f"idem:{key}",
    "processing",
    nx=True,      # Only set if Not eXists
    ex=86400      # Expire in 24 hours
)
if not result:
    return get_cached_response(key)
```

**3. Key Collisions**
Always use UUIDs (v4) as idempotency keys. Never use predictable keys.

**4. Exactly-Once Delivery**
Message queues (Kafka, RabbitMQ) can deliver messages multiple times. Consumers must be idempotent — check if a message ID was already processed before acting.

---

## Idempotency vs Related Concepts

| Concept | Meaning |
|---|---|
| **Idempotent** | Same operation N times = same result as once |
| **Deterministic** | Same input always produces same output |
| **Atomic** | Operation either fully completes or not at all |
| **Immutable** | State cannot be changed after creation |
| **Safe** | No side effects (doesn't change state) |

> All safe operations are idempotent. Not all idempotent operations are safe.

---

## Quick Summary Table

| Where | Idempotent Example | Non-Idempotent Example |
|---|---|---|
| HTTP | `PUT /user/1 {name: "John"}` | `POST /users {name: "John"}` |
| SQL | `UPDATE SET status='active'` | `UPDATE SET count = count + 1` |
| Code | `set_flag(True)` | `increment_counter()` |
| API | GET request | Email send request |
| Fintech | Charge with idempotency key | Raw POST to charge endpoint |

---

---

# 🎤 Common Interview Questions & Answers

---

### Q1: What is idempotency? Explain with an example.

**Answer:**
Idempotency means performing an operation multiple times yields the same result as performing it once. The side effects are the same regardless of how many times the operation runs.

**Example:** Pressing an elevator button for Floor 5 ten times still takes you to Floor 5 — not Floor 50. In HTTP, `PUT /user/1 {name: "Alice"}` sets the name to Alice. Running it 100 times still results in name = Alice.

---

### Q2: Why is POST not idempotent but PUT is?

**Answer:**
`POST` creates a new resource each time it's called. Two identical `POST /orders` requests create two separate orders — different outcomes, so not idempotent.

`PUT` replaces a resource with exact data. `PUT /user/1 {name: "Alice"}` sets user 1's name to Alice every time. The 2nd, 3rd, 100th call doesn't change the final state — it's always Alice. That's idempotent.

---

### Q3: Is DELETE idempotent? The second call returns 404 — does that break idempotency?

**Answer:**
Yes, DELETE is idempotent. Idempotency is about the **server state**, not the **response code**. After the first DELETE, the resource is gone. After the second DELETE, the resource is still gone — the state is identical. The 404 is just informational; the intended outcome (resource doesn't exist) is the same.

---

### Q4: How would you prevent double charges in a payment system?

**Answer:**
Using **Idempotency Keys**:
1. The client generates a unique UUID per payment attempt.
2. Sends it as a header: `Idempotency-Key: uuid-xyz`.
3. The server checks a Redis/DB store for this key before processing.
4. If key exists → return the cached response (no new charge).
5. If key is new → process payment, store `{key: response}`, return response.
6. Wrap the payment processing and key storage in a single DB transaction to prevent partial failures.

---

### Q5: How do you handle idempotency in a distributed system with multiple servers?

**Answer:**
Use a **shared, centralized store** (like Redis) that all servers access:
- Use Redis's `SET key value NX` (set if Not eXists) which is atomic.
- This ensures only one server processes a request even if multiple receive the retry simultaneously.
- The first server to acquire the lock processes the request; others return the cached result.
- Use distributed locks (Redlock algorithm) for critical sections.

---

### Q6: What's the difference between idempotency and atomicity?

**Answer:**
- **Idempotency**: Running an operation N times gives the same result as once. Concerned with *repeatability*.
- **Atomicity**: An operation either fully completes or doesn't happen at all. Concerned with *all-or-nothing* execution.

They complement each other: In fintech, you want atomic transactions (so partial failures don't corrupt state) AND idempotent operations (so retries don't double-charge). You need both.

---

### Q7: How would you design an idempotent API endpoint for money transfers?

**Answer:**

```
POST /transfers
Headers: Idempotency-Key: <client-uuid>
Body: { from, to, amount, currency }

Server Logic:
1. Extract idempotency key from header.
2. Check DB/Redis: has this key been processed?
   - YES → Return stored response immediately.
   - NO  → Continue.
3. Begin DB transaction:
   a. Lock source and destination accounts.
   b. Check sufficient balance.
   c. Debit source, credit destination.
   d. Create transaction record.
   e. Store idempotency key + response.
4. Commit transaction.
5. Return response.
```

Key design decisions: UUID keys, 24hr TTL, atomic key storage with transaction, pessimistic locking on accounts.

---

### Q8: What is "exactly-once" delivery and how does idempotency help?

**Answer:**
In distributed messaging (Kafka, RabbitMQ), it's impossible to guarantee a message is delivered **exactly once** — only **at-least-once** or **at-most-once**. "Exactly-once" is achieved by combining at-least-once delivery with idempotent consumers.

The consumer tracks processed message IDs in a store. Before processing a message, it checks: "Have I processed this ID before?" If yes, skip it. This makes the consumer idempotent, achieving effectively exactly-once behavior even with duplicate deliveries.

---

### Q9: How would you make a database migration script idempotent?

**Answer:**

```sql
-- Non-idempotent (fails on second run)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Idempotent (safe to run multiple times)
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

-- For indexes:
CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);

-- For data:
INSERT INTO config (key, value)
VALUES ('max_retries', '3')
ON CONFLICT (key) DO NOTHING;
```

Always use `IF NOT EXISTS`, `ON CONFLICT DO NOTHING/UPDATE`, and check-before-execute patterns in migration scripts.

---

### Q10: Give a real-world example where lack of idempotency caused a critical bug.

**Answer:**
A classic fintech incident: A payment service sends a charge request to a bank. The bank processes it and charges the user, but the response is lost due to a network timeout. The payment service retries. Without idempotency keys, the bank charges again. The user is double-charged. The company faces chargebacks, regulatory scrutiny, and loss of user trust.

Another example: Knight Capital Group (2012) lost $440M in 45 minutes partly due to a deployment error that caused orders to be sent multiple times — a lack of proper idempotency and state management in their trading system.

---

## 🧠 Key Takeaways

1. Idempotency = **same result no matter how many times you call it**.
2. GET, PUT, DELETE are idempotent. POST is not.
3. Use **Idempotency Keys** (UUIDs) to make non-idempotent operations safe for retry.
4. In databases: use **upserts, conditional updates, deduplication tables**.
5. In fintech: idempotency prevents **double charges** and is **legally and financially critical**.
6. In distributed systems: use **Redis atomic operations** to share idempotency state.
7. Idempotency enables **safe retries**, which enables **fault-tolerant systems**.