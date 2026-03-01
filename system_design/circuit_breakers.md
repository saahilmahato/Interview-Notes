# ⚡ Circuit Breakers — Complete Notes

---

## What is a Circuit Breaker?

A **Circuit Breaker** is a design pattern used in distributed systems to **prevent cascading failures**. It acts as a protective wrapper around a remote call (API, database, microservice) that can **automatically stop requests** when a service is failing — giving it time to recover.

The name comes from **electrical circuit breakers** in your home: when there's an overload, the breaker trips and cuts the power to prevent damage. Same idea here — when too many calls fail, the breaker "trips" and stops sending traffic.

> **Core Problem it Solves:** In a microservices architecture, if Service A calls Service B and Service B is down, Service A will keep retrying, consuming threads, memory, and time — eventually crashing itself too. This is called a **cascading failure**.

---

## The Three States

```
  [CLOSED] ──── failures exceed threshold ────► [OPEN]
     ▲                                              │
     │                                              │
     │                                     after timeout period
     │                                              │
     └──── success ──── [HALF-OPEN] ◄───────────────┘
```

### 1. 🟢 CLOSED (Normal Operation)
- Requests flow through **normally**.
- The breaker **counts failures** in a sliding window.
- If failures exceed a **threshold** (e.g., 5 failures in 10 seconds), the breaker trips to OPEN.
- This is the default/happy state.

### 2. 🔴 OPEN (Tripped — Failing Fast)
- **All requests are immediately rejected** without even trying.
- Returns a fallback response or error instantly.
- After a **timeout period** (e.g., 30 seconds), the breaker moves to HALF-OPEN.
- This gives the downstream service time to recover.

### 3. 🟡 HALF-OPEN (Testing the Waters)
- A **limited number of test requests** are allowed through.
- If they **succeed** → breaker moves back to CLOSED ✅
- If they **fail** → breaker goes back to OPEN 🔴
- This is the "probe" state.

---

## How It Works — Step by Step

```
Request comes in
      │
      ▼
Is breaker OPEN? ──► YES ──► Fail fast / return fallback
      │
      NO
      │
      ▼
Forward request to service
      │
      ▼
Did it succeed?
   │         │
  YES        NO
   │          │
   ▼          ▼
Reset      Increment
counter    failure count
           │
           ▼
    Threshold reached?
           │
          YES
           │
           ▼
       Trip to OPEN
```

---

## Key Configuration Parameters

| Parameter | Description | Example |
|---|---|---|
| **Failure Threshold** | How many failures before tripping | 5 failures |
| **Failure Rate Threshold** | % of failures before tripping | 50% |
| **Timeout / Sleep Window** | How long to stay OPEN before trying HALF-OPEN | 30 seconds |
| **Half-Open Max Calls** | Test requests allowed in HALF-OPEN | 3 requests |
| **Sliding Window Size** | Time or count window to measure failures | Last 10 calls / last 60s |

---

## Failure Types That Count

Not every error should trip the breaker. You typically count:

- **Network timeouts** ✅ (counts as failure)
- **5xx errors** ✅ (counts as failure)
- **Connection refused** ✅ (counts as failure)
- **4xx errors** ❌ (usually NOT a failure — it's a bad client request)
- **Business logic errors** ❌ (depends on use case)

---

## Fallback Strategies

When the breaker is OPEN, you need to do *something*. Options:

**1. Return a default/cached value**
```
"Sorry, recommendations are temporarily unavailable. Here are our top picks: [cached list]"
```

**2. Return an empty/degraded response**
```json
{ "recommendations": [], "status": "degraded" }
```

**3. Call a backup service**
```
Primary service failed → try secondary/redundant service
```

**4. Throw a meaningful error fast**
```
CircuitBreakerOpenException: "Service temporarily unavailable"
```

---

## Code Example (Pseudocode)

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30):
        self.state = "CLOSED"
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None

    def call(self, func, *args):
        if self.state == "OPEN":
            if time.now() - self.last_failure_time > self.timeout:
                self.state = "HALF-OPEN"   # Try again
            else:
                raise CircuitOpenError("Circuit is OPEN. Failing fast.")

        try:
            result = func(*args)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e

    def on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.now()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

---

## Circuit Breaker vs Retry Pattern

These two patterns are **complementary but different**:

| | **Retry** | **Circuit Breaker** |
|---|---|---|
| **Purpose** | Handle transient/temporary failures | Prevent cascading failures |
| **Behavior** | Retries the failed call N times | Stops calling altogether |
| **Good for** | Momentary blips, network hiccups | Sustained outages |
| **Risk** | Can worsen an overloaded service | Adds complexity |

**Use them together:** Retry a few times, and if failures keep piling up, the circuit breaker trips to protect everything.

---

## Where Circuit Breakers Are Used

- **Microservices communication** (Service A calling Service B)
- **Third-party API calls** (payment gateway, SMS provider)
- **Database connections** (DB is overwhelmed)
- **Message queue consumers** (downstream processor is slow)

---

## Popular Implementations

| Library/Tool | Language/Platform |
|---|---|
| **Resilience4j** | Java (successor to Hystrix) |
| **Netflix Hystrix** | Java (now in maintenance mode) |
| **Polly** | .NET |
| **Opossum** | Node.js |
| **PyBreaker** | Python |
| **Istio / Envoy** | Service mesh (infrastructure level) |
| **AWS App Mesh** | AWS managed service mesh |

---

## Real-World Analogy

Think of Netflix during peak hours. If their recommendation service goes down, do they crash the entire homepage? No — they show a generic list of popular titles (fallback). The circuit breaker tripped on the recommendation service, the page still loads, and users still watch content. That's **graceful degradation** powered by a circuit breaker.

---

## Circuit Breaker in a Service Mesh

In modern architectures, you don't always implement circuit breakers in code. **Service meshes like Istio** implement them at the **infrastructure/sidecar proxy level**:

```
Service A ──► [Envoy Sidecar] ──► [Envoy Sidecar] ──► Service B
                    ▲
                Circuit Breaker logic lives here
                (no code changes needed)
```

This is powerful because it works across languages and teams without touching application code.

---

## Common Pitfalls

**1. Setting thresholds too low** → Breaker trips too easily on normal transient errors.

**2. Setting timeout too long** → Service recovers but traffic doesn't resume for minutes.

**3. Not monitoring breaker state** → You won't know it tripped until users complain.

**4. No fallback** → Circuit breaker trips and users get a 500 error anyway. Pointless.

**5. Shared breaker state in multi-instance deployments** → Each pod has its own breaker; they don't share state. Use a distributed cache (Redis) if you need coordinated state.

---

---

# 🎯 Common Interview Questions & Answers

---

**Q1: What is a Circuit Breaker pattern and why do we need it?**

**A:** A Circuit Breaker is a stability pattern that wraps calls to external services and monitors for failures. When failures exceed a threshold, it "trips open" and immediately rejects further calls without forwarding them, allowing the failing service time to recover. We need it to prevent cascading failures in distributed systems — where one failing service can cause a chain reaction of failures across the entire system by exhausting threads, connections, and memory.

---

**Q2: Explain the three states of a Circuit Breaker.**

**A:**
- **CLOSED:** Normal operation. Requests go through. Failures are counted. If they exceed the threshold, it trips to OPEN.
- **OPEN:** All requests are immediately rejected (fail fast). After a sleep timeout, it transitions to HALF-OPEN.
- **HALF-OPEN:** A limited number of probe requests are allowed through. If they succeed, the breaker closes. If they fail, it reopens.

---

**Q3: What is the difference between a Circuit Breaker and a Retry pattern?**

**A:** Retry is about handling transient failures — it retries a failed call N times with some backoff, hoping it was a momentary blip. Circuit Breaker is about sustained failures — it stops calling the service entirely once too many failures occur. They work best together: retry on transient failures, and if failures keep accumulating, the circuit breaker trips to stop hammering a service that is clearly down.

---

**Q4: What happens when the circuit is OPEN? How do you handle requests?**

**A:** When the circuit is OPEN, the call should fail fast — meaning you don't even attempt to contact the downstream service. Instead you use a fallback strategy: return a cached response, a default value, call a backup service, or return a graceful error to the caller. The key is the failure is *immediate* — no waiting for timeouts.

---

**Q5: What is "fail fast" and why is it important?**

**A:** Fail fast means immediately returning an error when the circuit is OPEN instead of waiting for the real call to time out. This is important because thread pool exhaustion is a major cause of cascading failures. If each failed request takes 30 seconds to time out, a flood of requests will exhaust all available threads quickly. By failing in milliseconds, we free threads immediately and keep the calling service healthy.

---

**Q6: How would you implement a distributed Circuit Breaker across multiple instances of a service?**

**A:** Each service instance has its own in-memory circuit breaker, which means the state is not shared. For a distributed/coordinated breaker, you can store the failure count and state in a shared store like **Redis**, so all instances share the same view of the circuit. Alternatively, use a **service mesh** like Istio or Linkerd, which implements circuit breaking at the proxy level across all instances consistently.

---

**Q7: What metrics/events should you monitor for a Circuit Breaker?**

**A:** You should monitor:
- **Circuit state changes** (CLOSED → OPEN, OPEN → HALF-OPEN)
- **Failure rate** over time
- **Number of rejected calls** (when OPEN)
- **Latency distribution** of successful and failed calls
- **Fallback success/failure rate**

Alerts should fire when a circuit transitions to OPEN, because it signals a downstream service is unhealthy.

---

**Q8: What is the difference between a Circuit Breaker at the application level vs. at the infrastructure level?**

**A:** At the **application level** (e.g., Resilience4j, Polly), the circuit breaker is implemented in code, giving you fine-grained control and easy integration with business logic and fallbacks. At the **infrastructure level** (e.g., Istio, Envoy), the circuit breaker lives in the sidecar proxy — it's language-agnostic, requires no code changes, and is configured via the service mesh control plane. Infrastructure-level breakers are great for polyglot environments, but application-level breakers allow more customized fallback logic.

---

**Q9: What is the sliding window in a Circuit Breaker?**

**A:** A sliding window determines what "window" of calls is used to compute the failure rate. There are two types:
- **Count-based:** Looks at the last N requests (e.g., 10 calls). If 5 of the last 10 failed → trip.
- **Time-based:** Looks at all calls in the last N seconds (e.g., 60 seconds). If 50% failed → trip.

Time-based windows are generally better because they account for traffic volume — a count-based window can behave unpredictably during low-traffic periods.

---

**Q10: Can a Circuit Breaker replace load balancing or health checks?**

**A:** No — they serve different purposes. A **health check** proactively pings a service to detect if it's alive. A **load balancer** distributes traffic across healthy instances. A **circuit breaker** protects a caller when a downstream call is failing, regardless of whether the instance is technically "up." They are complementary: you'd use all three together in a production system.

---

**Q11: What are common mistakes when using Circuit Breakers?**

**A:**
- Thresholds too low, causing frequent false trips on normal transient errors
- No fallback defined, so tripping the breaker still returns a 500 to the user
- Not monitoring or alerting on state changes
- Counting 4xx client errors as failures (those are caller bugs, not service failures)
- Forgetting that each instance has its own state in a multi-pod deployment

---

**Q12: How does Netflix use Circuit Breakers?**

**A:** Netflix pioneered the pattern in microservices with **Hystrix**. Every service call is wrapped in a Hystrix command with a dedicated thread pool. If the pool fills up or failure rate spikes, the circuit trips and a fallback is served. For example, if the personalization service is down, Netflix falls back to showing popular titles. This ensures the site stays up even if individual services fail — which is the essence of resilience engineering.

---

*That covers Circuit Breakers end to end — the concept, implementation, pitfalls, and everything you'd need to ace an interview or implement it in production.* 🚀