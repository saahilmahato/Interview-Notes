# 📡 Event-Driven Architecture (EDA) — Complete Notes

---

## 🧠 What is Event-Driven Architecture?

Event-Driven Architecture is a software design pattern where **services communicate by producing and consuming events** rather than calling each other directly.

Think of it like a **newspaper subscription**:
- The newspaper (producer) prints the news (event)
- Subscribers (consumers) who care about it read it
- The newspaper doesn't know or care who reads it

> **Core idea:** Components are **decoupled**. They don't talk to each other directly — they talk through events.

---

## 🔑 Core Concepts

### 1. **Event**
A record that *something happened* in the system.

```json
{
  "eventId": "abc-123",
  "eventType": "order.placed",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "orderId": "ORD-456",
    "userId": "USR-789",
    "amount": 99.99
  }
}
```

Events are:
- **Immutable** — once emitted, they don't change
- **Past tense** — they describe something that *already happened* (`OrderPlaced`, not `PlaceOrder`)
- **Lightweight** — carry just enough data

---

### 2. **Producer (Publisher)**
The service that **detects a change** and emits an event. It doesn't know who will consume it.

Example: `OrderService` emits `OrderPlaced` when a user places an order.

---

### 3. **Consumer (Subscriber)**
The service that **listens for events** and reacts to them.

Example: `EmailService` listens for `OrderPlaced` and sends a confirmation email.

---

### 4. **Event Broker / Message Broker**
The **middleman** that receives events from producers and delivers them to consumers.

Popular brokers:
| Broker | Best For |
|---|---|
| **Apache Kafka** | High-throughput, event streaming, replay |
| **RabbitMQ** | Task queues, routing, low-latency |
| **AWS SNS/SQS** | Cloud-native, serverless |
| **Google Pub/Sub** | GCP ecosystem |
| **Azure Service Bus** | Enterprise messaging |
| **NATS** | Lightweight, IoT, edge |

---

### 5. **Event Channel / Topic**
A named pipe or stream where events of a specific type are published.

Example: `orders.created`, `payments.processed`, `users.signup`

---

## 🏗️ Architecture Patterns

### Pattern 1: Simple Event Notification
Producer fires an event. Consumer reacts. Consumer may call back for more data.

```
OrderService  →  [order.placed event]  →  Broker  →  EmailService
                                                    →  InventoryService
                                                    →  AnalyticsService
```

---

### Pattern 2: Event-Carried State Transfer
The event carries **all the data** consumers need — no callbacks required.

```json
{
  "eventType": "order.placed",
  "data": {
    "orderId": "ORD-456",
    "userEmail": "john@example.com",   ← carried in event
    "items": [...],                     ← carried in event
    "shippingAddress": {...}            ← carried in event
  }
}
```

✅ Consumers are fully autonomous — no tight coupling  
⚠️ Events become large and must be versioned carefully

---

### Pattern 3: Event Sourcing
Instead of storing the **current state**, you store the **full history of events**. State is derived by replaying events.

```
Events Log:
1. AccountCreated   { balance: 0 }
2. MoneyDeposited   { amount: 500 }
3. MoneyWithdrawn   { amount: 200 }

Current State = Replay all → balance: 300
```

✅ Full audit trail  
✅ Time travel (replay to any point in time)  
✅ Easy debugging  
⚠️ Complex to implement  
⚠️ Event schema changes are painful

---

### Pattern 4: CQRS (Command Query Responsibility Segregation)
Often paired with EDA. Separate the **write model** (commands) from the **read model** (queries).

```
User sends Command → Write DB (normalized)
                   → Event emitted
                   → Read DB updated (denormalized, optimized for queries)

User queries       → Read DB (fast, cheap)
```

---

### Pattern 5: Saga Pattern
Manages **distributed transactions** across multiple services using a chain of events.

**Choreography Saga** (decentralized):
```
OrderService → OrderPlaced
  PaymentService listens → PaymentProcessed
    InventoryService listens → InventoryReserved
      ShippingService listens → ShipmentCreated ✅
```

Each service reacts and emits the next event.

**Orchestration Saga** (centralized):
```
SagaOrchestrator → calls each service step by step
                 → handles failures and rollbacks
```

---

## 🔄 Push vs Pull

| | Push | Pull |
|---|---|---|
| **How it works** | Broker pushes events to consumers | Consumer polls for new events |
| **Latency** | Low | Higher |
| **Example** | Kafka with push consumers | SQS polling |

---

## ✅ Advantages

**Loose Coupling** — Services don't know about each other. You can change, scale, or replace them independently.

**Scalability** — Consumers can be scaled independently based on load.

**Resilience** — If a consumer is down, events queue up and are processed when it recovers.

**Flexibility** — Add new consumers without changing producers.

**Auditability** — Event logs give you a full history of what happened.

**Real-time processing** — React to things as they happen.

---

## ⚠️ Disadvantages & Challenges

**Eventual Consistency** — Data isn't immediately consistent across services. You have to design for this.

**Debugging is hard** — Tracing a request across multiple services and async events is complex. Need distributed tracing (Jaeger, Zipkin).

**Event ordering** — Consumers may receive events out of order.

**Duplicate events** — Networks fail. The same event can be delivered more than once. Consumers must be **idempotent**.

**Schema evolution** — Events must be versioned. Breaking changes are dangerous.

**Complexity** — More moving parts = more things to monitor, deploy, and understand.

---

## 🔑 Key Design Principles

### Idempotency
Processing the same event twice should produce the same result.

```python
# Bad — charging twice if event is received twice
def handle_payment(event):
    charge_user(event.userId, event.amount)

# Good — check if already processed
def handle_payment(event):
    if not already_processed(event.eventId):
        charge_user(event.userId, event.amount)
        mark_as_processed(event.eventId)
```

---

### At-Least-Once vs At-Most-Once vs Exactly-Once Delivery

| Guarantee | Description | Risk |
|---|---|---|
| **At-Most-Once** | Event sent once, may be lost | Data loss |
| **At-Least-Once** | Retried until acknowledged, may duplicate | Duplicates |
| **Exactly-Once** | Delivered exactly once | Expensive, complex |

Most systems use **at-least-once** + **idempotent consumers**.

---

### Dead Letter Queue (DLQ)
When a consumer **fails to process** an event after retries, it goes to a DLQ for inspection.

```
Event → Consumer fails → Retry 3x → Dead Letter Queue → Alert/Manual fix
```

---

### Event Schema & Versioning
Use a **schema registry** (like Confluent Schema Registry with Avro) to manage event contracts.

```
v1: { orderId, userId, amount }
v2: { orderId, userId, amount, currency }  ← add optional fields (backward compatible)
```

---

## 🆚 EDA vs REST vs gRPC

| | REST | gRPC | EDA |
|---|---|---|---|
| **Communication** | Synchronous | Synchronous | Asynchronous |
| **Coupling** | Tight | Tight | Loose |
| **Latency** | Low | Very Low | Higher (async) |
| **Consistency** | Strong | Strong | Eventual |
| **Use case** | Request/response | High perf RPC | Decoupled workflows |
| **Failure handling** | Caller handles | Caller handles | Broker retries |

> **Real world:** Most systems use a mix. REST/gRPC for user-facing queries. EDA for background workflows.

---

## 🌍 Real-World Use Cases

- **E-commerce:** Order placed → payment processed → inventory updated → shipping triggered → email sent
- **Banking:** Transaction events trigger fraud detection, balance updates, notifications
- **Ride-sharing:** Driver location events update maps, ETAs, dispatch systems in real-time
- **IoT:** Sensor events trigger alerts, analytics, dashboards
- **Social media:** Post liked → notify followers → update feed rankings → analytics

---

## 🛠️ Tech Stack Example (Modern EDA)

```
Producers:  Spring Boot / Node.js / Python services
Broker:     Apache Kafka
Consumers:  Microservices / AWS Lambda
Schema:     Confluent Schema Registry (Avro/Protobuf)
Tracing:    Jaeger / Datadog
Monitoring: Prometheus + Grafana
DLQ:        Separate Kafka topic or SQS DLQ
```

---

---

# 💼 Interview Questions & Answers

---

**Q1: What is Event-Driven Architecture and why would you use it?**

**A:** EDA is a design pattern where services communicate through events — a producer emits an event when something happens, and consumers react to it asynchronously. You'd use it when you need loose coupling between services, high scalability, resilience to failures, or need to react to things in real-time. For example, in an e-commerce system, when an order is placed, multiple services (email, inventory, shipping) need to react — EDA handles this cleanly without the OrderService needing to know about any of them.

---

**Q2: What is the difference between a Message Queue and an Event Stream?**

**A:** A **message queue** (like RabbitMQ, SQS) delivers a message to **one consumer**, who then acknowledges and removes it. It's like a task list — once done, it's gone. An **event stream** (like Kafka) is a **log of events** that persists. Multiple consumer groups can independently read the same events at their own pace, and events can be replayed. Use queues for task distribution; use streams for event history, replay, and multiple independent consumers.

---

**Q3: What is Event Sourcing and how does it differ from traditional data storage?**

**A:** Traditionally, you store the **current state** of an entity in a database (e.g., a row with current balance). With Event Sourcing, you store the **sequence of events** that led to that state (Deposited $500, Withdrew $200). The current state is derived by replaying events. Benefits include a full audit trail, the ability to replay events to rebuild state or create new projections, and easier debugging. The downside is increased complexity and difficulty evolving event schemas.

---

**Q4: What is idempotency and why is it critical in EDA?**

**A:** Idempotency means that performing the same operation multiple times produces the same result. It's critical in EDA because message brokers typically guarantee **at-least-once delivery** — the same event can be delivered more than once (due to retries, network issues). If your consumer isn't idempotent, you might charge a user twice, send duplicate emails, etc. You ensure idempotency by tracking processed event IDs and skipping duplicates.

---

**Q5: What is the Saga Pattern and when would you use it?**

**A:** The Saga pattern manages long-running distributed transactions across multiple services. Instead of a two-phase commit (which requires all services to be available simultaneously), a Saga breaks the transaction into a sequence of local transactions, each publishing an event that triggers the next step. If a step fails, compensating transactions are triggered to undo previous steps. Use it when you have a multi-step business process spanning multiple services, like order checkout (payment → inventory → shipping).

---

**Q6: What's the difference between Choreography and Orchestration in Sagas?**

**A:** In **Choreography**, each service knows what to do when it receives an event and emits the next event. There's no central coordinator — services react and chain together. It's simple but can become hard to track as complexity grows. In **Orchestration**, a central Saga Orchestrator tells each service what to do step by step and handles failures. It's easier to trace and reason about but introduces a central point of coupling. Choreography is better for simple flows; orchestration for complex, multi-step business processes.

---

**Q7: How do you handle failures in EDA?**

**A:** Several strategies: (1) **Retries with exponential backoff** — retry failed processing with increasing delays. (2) **Dead Letter Queue (DLQ)** — after max retries, move the event to a DLQ for manual inspection or alerting. (3) **Circuit breaker** — stop hammering a failing dependency and fail fast. (4) **Idempotent consumers** — safe to retry without side effects. (5) **Compensation events** — emit an event that undoes a previous action (used in Sagas). The broker itself should be clustered and replicated for high availability.

---

**Q8: What is CQRS and how does it relate to EDA?**

**A:** CQRS (Command Query Responsibility Segregation) separates the write model from the read model. Commands mutate state; queries read state from a separate, optimized read database. EDA ties in naturally: when a command is processed, an event is emitted, which updates the read model asynchronously. This allows the read model to be tailored for fast queries (denormalized, cached) while the write model stays clean. The tradeoff is eventual consistency — the read model is slightly behind the write model.

---

**Q9: How do you handle event schema evolution without breaking consumers?**

**A:** (1) **Backward-compatible changes only** — add optional fields, never remove or rename required ones. (2) **Use a schema registry** (like Confluent) with Avro or Protobuf, which enforces compatibility rules. (3) **Version your events** — include a `version` or `schemaVersion` field so consumers can handle multiple versions. (4) **Dual-publish** — during migration, publish both old and new event formats. (5) Consumers should be **tolerant readers** — ignore unknown fields, use defaults for missing ones.

---

**Q10: How do you ensure event ordering in EDA?**

**A:** Ordering is per-partition in Kafka — events in the same partition are ordered. Use a **partition key** (like `orderId` or `userId`) so all events for the same entity land in the same partition and are processed in order. The tradeoff is that a single partition is consumed by one consumer, limiting parallelism for that entity. In RabbitMQ, you'd use a single queue for ordered processing. Globally ordered streams are very expensive and usually unnecessary — design your system to handle out-of-order events where possible using timestamps or sequence numbers.

---

**Q11: How is EDA different from REST in terms of consistency?**

**A:** REST APIs are typically **synchronous** and give you **strong consistency** — you call a service, it does the work, it responds. EDA is **asynchronous** and gives you **eventual consistency** — you emit an event, consumers process it later, and the system converges to a consistent state over time. EDA is better for scalability and resilience; REST is better when you need an immediate, consistent response. Many real systems use both: REST for user-facing queries, EDA for background workflows.

---

**Q12: What is a Dead Letter Queue and why is it important?**

**A:** A Dead Letter Queue (DLQ) is a special queue where messages are sent when they **cannot be processed successfully** after a configured number of retries. It's important because without it, a bad message (poison pill) can block processing indefinitely, or you could silently lose data. With a DLQ, you can: alert on failures, inspect problematic events, fix the bug, and reprocess them. It's a critical safety net in any production EDA system.

---

**Q13: What's a "poison pill" message?**

**A:** A poison pill is a message that consistently causes the consumer to crash or throw an exception — it can never be processed successfully. Without protection, it would be retried forever, blocking all other messages. The solution is to set a **max retry count** and route failed messages to a **DLQ**. Consumers should also have robust error handling so one bad message doesn't take down the whole consumer.

---

**Q14: When would you NOT use Event-Driven Architecture?**

**A:** EDA adds significant complexity and is overkill in some scenarios: (1) **Simple CRUD apps** with no inter-service communication needs. (2) When you need **strong, immediate consistency** — eventual consistency is unacceptable (e.g., financial double-entry bookkeeping without careful design). (3) **Small teams or projects** where the operational overhead of running a broker, monitoring, DLQs, and schema registry is too heavy. (4) When **synchronous response** is required — the user is waiting for an immediate result. Use the simplest architecture that solves your problem.

---

*These notes cover everything from fundamentals to advanced patterns and real interview scenarios. Review the Saga pattern, idempotency, and CQRS — those are the most commonly deep-dived topics in senior engineering interviews.* 🚀