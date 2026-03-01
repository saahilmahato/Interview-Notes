# 🏗️ Layered Architecture & Service-Oriented Design

---

## 📌 What is Layered Architecture?

Layered architecture (also called **N-Tier architecture**) is a design pattern where your application is divided into **horizontal layers**, each with a specific responsibility. Each layer only talks to the layer directly below or above it.

Think of it like a **burger** 🍔 — each layer has its own role, and you don't mix the bun with the patty logic.

```
┌──────────────────────────────┐
│       Presentation Layer     │  ← UI, API Controllers
├──────────────────────────────┤
│       Business Logic Layer   │  ← Services, Rules, Workflows
├──────────────────────────────┤
│       Data Access Layer      │  ← Repositories, ORM, Queries
├──────────────────────────────┤
│       Database Layer         │  ← PostgreSQL, MongoDB, etc.
└──────────────────────────────┘
```

---

## 🔍 The Layers Explained

### 1. Presentation Layer (UI / API Layer)
- The **entry point** of the application.
- Handles HTTP requests, renders UI, or exposes REST/GraphQL APIs.
- Should contain **zero business logic**.
- Examples: Express.js controllers, React components, Django views.

```js
// Controller — just receives request and delegates
app.post('/users', async (req, res) => {
  const user = await userService.createUser(req.body);
  res.status(201).json(user);
});
```

---

### 2. Business Logic Layer (Service Layer)
- The **brain** of the application.
- Contains all rules, validations, calculations, and workflows.
- Completely independent of how data is stored or how it's presented.

```js
// Service — pure business logic
class UserService {
  async createUser(data) {
    if (await this.userRepo.findByEmail(data.email)) {
      throw new Error('Email already exists');
    }
    const hashed = await bcrypt.hash(data.password, 10);
    return this.userRepo.save({ ...data, password: hashed });
  }
}
```

---

### 3. Data Access Layer (Repository Layer)
- Abstracts all **database interactions**.
- The service layer should never write raw SQL or ORM queries directly.
- Makes it easy to swap databases without touching business logic.

```js
// Repository — only talks to the DB
class UserRepository {
  async findByEmail(email) {
    return db.query('SELECT * FROM users WHERE email = $1', [email]);
  }
  async save(user) {
    return db.query('INSERT INTO users ...', [...]);
  }
}
```

---

### 4. Database Layer
- The actual data store: PostgreSQL, MySQL, MongoDB, Redis, etc.
- Only the Data Access Layer should ever touch this directly.

---

## ✅ Key Principles of Layered Architecture

| Principle | Meaning |
|---|---|
| **Separation of Concerns** | Each layer has one job |
| **Single Direction Flow** | Top layers call down, never the reverse |
| **Loose Coupling** | Layers talk through interfaces/abstractions |
| **High Cohesion** | Related code stays together in the same layer |
| **Testability** | Each layer can be unit tested in isolation |

---

## ⚡ Strict vs. Relaxed Layering

**Strict layering:** Layer N can only call Layer N-1. No skipping.

**Relaxed layering:** A layer can call any layer below it (e.g., Presentation calling the Repository directly). This is sometimes done for performance but breaks the clean separation.

> ✅ Prefer **strict layering** unless you have a very justified performance reason.

---

## 📌 What is Service-Oriented Design?

**Service-Oriented Design (SOD)** is about organizing your application around **services** — discrete, self-contained units that encapsulate a piece of business capability.

It answers the question: *"How do we divide behavior and responsibility into manageable, reusable, and independent units?"*

This is the foundation for both **monolithic service layers** and eventually **microservices**.

---

## 🔧 Core Concepts of Service-Oriented Design

### 1. Service
A service is a **class or module** that owns a specific piece of business logic. It should be:
- **Focused** — does one thing well (Single Responsibility)
- **Stateless** — doesn't hold mutable state between calls
- **Reusable** — can be called from multiple places

```
OrderService        → handles order creation, cancellation, status
PaymentService      → handles charging, refunds, payment methods
NotificationService → handles emails, SMS, push notifications
UserService         → handles registration, auth, profile updates
```

---

### 2. Dependency Injection (DI)
Services depend on abstractions (interfaces), not concrete implementations. This is injected from outside.

```js
class OrderService {
  constructor(orderRepo, paymentService, notificationService) {
    this.orderRepo = orderRepo;
    this.paymentService = paymentService;
    this.notificationService = notificationService;
  }

  async placeOrder(cartData) {
    const order = await this.orderRepo.create(cartData);
    await this.paymentService.charge(order);
    await this.notificationService.sendOrderConfirmation(order);
    return order;
  }
}
```

> The `OrderService` doesn't care *how* payment is done — it just calls the interface. You can swap `StripePaymentService` for `PayPalPaymentService` without touching `OrderService`.

---

### 3. Interface / Contract
A service exposes a clear **contract** — what inputs it takes and what outputs it returns. Internal implementation is hidden.

```ts
interface PaymentService {
  charge(order: Order): Promise<PaymentResult>;
  refund(orderId: string): Promise<void>;
}

class StripePaymentService implements PaymentService { ... }
class PayPalPaymentService implements PaymentService { ... }
```

---

### 4. Orchestration vs. Choreography

**Orchestration:** One central service (orchestrator) controls the flow and calls other services in sequence. Easy to trace, but creates coupling to the orchestrator.

```
OrderService → calls PaymentService → calls InventoryService → calls NotificationService
```

**Choreography:** Services react to **events**. No central controller. Each service listens for events and does its own thing. More decoupled, but harder to trace.

```
OrderPlaced event emitted
  → PaymentService listens and charges
  → InventoryService listens and reserves stock
  → NotificationService listens and sends email
```

---

## 🏛️ How Layered Architecture + SOD Work Together

```
┌──────────────────────────────────────────┐
│            Presentation Layer            │
│    (Controllers / API Routes / UI)       │
└──────────────┬───────────────────────────┘
               │ calls
┌──────────────▼───────────────────────────┐
│          Service Layer (SOD)             │
│  OrderService  PaymentService  UserSvc   │
└──────────────┬───────────────────────────┘
               │ calls
┌──────────────▼───────────────────────────┐
│         Data Access Layer                │
│   OrderRepo    PaymentRepo    UserRepo   │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│            Database                      │
└──────────────────────────────────────────┘
```

---

## 🆚 Layered Architecture vs. Microservices

| Aspect | Layered (Monolith) | Microservices |
|---|---|---|
| Deployment | Single unit | Independent services |
| Communication | In-process function calls | HTTP / message queues |
| Complexity | Low-medium | High |
| Scalability | Scale whole app | Scale per service |
| Team size | Small-medium | Large, multiple teams |
| Good for | Startups, MVPs | Large-scale systems |

> Layered + SOD inside a monolith is a **great starting point**. You can later split services into microservices if needed because your boundaries are already clean.

---

## 🚨 Common Mistakes to Avoid

**1. Fat Controllers** — Putting business logic in the controller. Never do this.

**2. Anemic Services** — Services that are just wrappers around repos with no real logic. This usually means logic leaked into controllers or repos.

**3. Circular Dependencies** — ServiceA depends on ServiceB which depends on ServiceA. Fix with events or restructuring.

**4. Skipping Layers** — Controller calling the database directly. Breaks the entire pattern.

**5. God Service** — One `AppService` that does everything. Break it up.

---

## 📐 SOLID in the Context of Service-Oriented Design

| Principle | Application |
|---|---|
| **S** - Single Responsibility | Each service owns one business domain |
| **O** - Open/Closed | Add new services without modifying existing ones |
| **L** - Liskov Substitution | `StripeService` can replace `PayPalService` if they implement same interface |
| **I** - Interface Segregation | Small, focused interfaces per service |
| **D** - Dependency Inversion | Services depend on abstractions, not implementations |

---

## 💡 Real World Example — E-Commerce App

```
Presentation Layer:
  POST /orders          → OrderController

Service Layer:
  OrderService.placeOrder()
    → validates cart
    → calls InventoryService.reserve()
    → calls PaymentService.charge()
    → calls NotificationService.sendConfirmation()
    → calls OrderRepository.save()

Data Access Layer:
  OrderRepository.save()
  InventoryRepository.decrementStock()

Database:
  orders table, inventory table, payments table
```

Everything is clean, testable, and replaceable.

---

---

# 🎯 Common Interview Questions & Answers

---

### Q1. What is layered architecture and why do we use it?

**Answer:** Layered architecture divides an application into horizontal layers — Presentation, Business Logic, Data Access, and Database — where each layer has a single responsibility and communicates only with adjacent layers. We use it because it promotes separation of concerns, makes code easier to maintain and test, allows teams to work on different layers independently, and makes it easy to swap out implementations (e.g., changing the database) without affecting other parts of the system.

---

### Q2. What is the difference between the Service Layer and the Repository Layer?

**Answer:** The **Service Layer** contains business logic — rules, workflows, validations, and decisions. It orchestrates what needs to happen. The **Repository Layer** is purely about data persistence — it abstracts database queries and operations. The service says *"what"* needs to happen; the repository says *"how"* to persist it. Services call repositories, never the other way around.

---

### Q3. What does "separation of concerns" mean in practice?

**Answer:** It means that each piece of code has exactly one reason to exist and one reason to change. In a layered architecture, a controller shouldn't format database queries, a service shouldn't render HTML, and a repository shouldn't contain business rules. If your business rule changes, only the service changes. If your database changes, only the repository changes. Nothing else needs to be touched.

---

### Q4. What is Dependency Injection and why is it important in service-oriented design?

**Answer:** Dependency Injection is a pattern where a class receives its dependencies from the outside rather than creating them internally. Instead of `new StripePaymentService()` inside `OrderService`, the `PaymentService` is injected into `OrderService` via its constructor. This is important because it decouples the classes, makes them testable (you can inject a mock in tests), and allows implementations to be swapped without modifying the consumer class.

---

### Q5. What is the difference between orchestration and choreography?

**Answer:** In **orchestration**, one central service controls the flow and explicitly calls other services in a defined sequence. It's easy to read and trace but creates a tight coupling to the orchestrator. In **choreography**, services are decoupled and react to events independently — a service publishes an event and other services subscribe and react. Choreography is more scalable and decoupled but harder to trace and debug. Orchestration is better for simple, critical flows; choreography is better for distributed, high-scale systems.

---

### Q6. How would you handle cross-cutting concerns like logging and authentication in a layered architecture?

**Answer:** Cross-cutting concerns that apply across multiple layers are best handled with **middleware** (at the presentation layer) or **decorators/interceptors** at the service level. Authentication is typically a middleware or guard at the controller level that validates the request before it even reaches the service. Logging can be applied at the controller level for request/response logs, and at the service level for business-level logs. Some teams use **AOP (Aspect-Oriented Programming)** or **decorators** to inject these behaviors without polluting the core logic.

---

### Q7. What is the problem with "fat controllers" and how do you fix it?

**Answer:** A fat controller is one that contains business logic directly — validation, calculations, database calls — instead of delegating to a service. The problems are that it can't be reused (tied to HTTP), it's hard to test (needs HTTP context), and it violates single responsibility. The fix is to move all business logic into a service class. The controller should only: parse the request, call the appropriate service method, and return the response. It should have no `if` statements about business rules.

---

### Q8. How does layered architecture support testability?

**Answer:** Because each layer is independent and communicates through clear interfaces, you can test each layer in isolation by mocking the layer below it. You can unit test a `UserService` by mocking `UserRepository` — no real database needed. You can test a controller by mocking `UserService` — no real business logic runs. This means tests are fast, reliable, and focused. Integration tests can then verify that layers work correctly together.

---

### Q9. When would you NOT use layered architecture?

**Answer:** For very simple scripts or CRUD applications with no real business logic, layered architecture can be overkill. Also, strict layering can add latency if data has to pass through many transformation steps. For read-heavy systems requiring extreme performance, you might bypass layers using the **CQRS (Command Query Responsibility Segregation)** pattern where queries can go directly to a read-optimized store. Always choose architecture based on the complexity and scale of the problem.

---

### Q10. What is the Anemic Domain Model anti-pattern?

**Answer:** An Anemic Domain Model is when your service layer is full of business logic and your domain/model objects are just plain data containers with getters and setters — they have no behavior. This is considered an anti-pattern in Domain-Driven Design because the objects that *represent* the business concepts carry none of the business rules. The alternative is a **Rich Domain Model** where the domain objects themselves contain relevant behaviors (e.g., `order.cancel()`, `user.changePassword()`), and services orchestrate them rather than implementing all logic themselves.

---

### Q11. How would you prevent circular dependencies between services?

**Answer:** Circular dependencies (`ServiceA → ServiceB → ServiceA`) are a sign of poor boundary design. Solutions include: (1) **Extract shared logic** into a third service that both depend on. (2) **Use events** — instead of ServiceA calling ServiceB directly, ServiceA emits an event that ServiceB reacts to. (3) **Refactor boundaries** — if two services are too coupled, maybe they belong in the same service. (4) In DDD terms, reconsider your bounded contexts and aggregate boundaries.

---

### Q12. Explain the difference between horizontal and vertical scaling in the context of a layered monolith vs microservices.

**Answer:** A layered monolith scales **horizontally** by running multiple identical instances of the whole application behind a load balancer. You can't scale just the payment processing independently — the entire app scales together. In a microservices architecture (which builds on SOD principles), each service is independently deployable, so you can scale just the services under high load. If your `PaymentService` is the bottleneck, you scale only that service, which is more resource-efficient. This is the main scalability argument for eventually splitting a well-designed layered monolith into microservices.

---

> 💡 **Pro tip for interviews:** Always emphasize *why* you'd use a pattern, not just *what* it is. Interviewers want to see that you understand trade-offs.