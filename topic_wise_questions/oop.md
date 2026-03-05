# Object-Oriented Programming — Senior Engineer Interview Bank

---

## SECTION 1: Core Principles (The "Four Pillars")

---

### Q1. What are the four pillars of OOP, and can you give a real-world analogy for each?

**Sample Answer:**

**Encapsulation** — Bundling data and the methods that operate on it into a single unit, while restricting direct access to internal state. Think of a car's engine: you interact with it via the gas pedal and ignition, not by manually firing pistons. The internals are hidden; the interface is clean.

**Abstraction** — Hiding complexity behind a simplified interface. When you use a microwave, you press "2 minutes" — you don't think about magnetron frequency or waveguide geometry. In code, an abstract class or interface defines *what* something does, not *how*.

**Inheritance** — A mechanism by which one class acquires the properties and behaviors of a parent class. A `SavingsAccount` and `CheckingAccount` both inherit from `BankAccount` — they share deposit/withdrawal logic but override how interest is calculated.

**Polymorphism** — The ability of different objects to respond to the same message in different ways. A `Shape` interface has a `draw()` method. A `Circle`, `Rectangle`, and `Triangle` all implement it differently. Code that loops over a list of `Shape` objects doesn't need to know which specific type it's working with.

---

### Q2. What's the difference between **abstraction** and **encapsulation**? Many candidates conflate them.

**Sample Answer:**

They're related but distinct.

**Encapsulation** is a *mechanism* — it's about *how* you protect data. You make fields private and expose only what's necessary via public methods. It's an implementation technique for information hiding.

**Abstraction** is a *design concept* — it's about *what* you expose. It's concerned with reducing complexity by defining a clean interface and hiding irrelevant details from the consumer.

A concrete example: a `LinkedList` class that makes its `Node` inner class private is practicing *encapsulation* — the internal node structure is hidden. When you expose only `add()`, `remove()`, `get()` methods, you're practicing *abstraction* — the caller doesn't need to think about nodes at all.

You can have encapsulation without full abstraction (e.g., a class that hides its fields but exposes a bloated, complex API). You can conceptually achieve abstraction through interfaces without encapsulating state at all (pure abstract types with no fields).

---

### Q3. Explain the difference between **method overloading** and **method overriding**. When would you use each?

**Sample Answer:**

**Overloading** (compile-time / static polymorphism): Multiple methods with the *same name* but *different parameter signatures* in the same class. The compiler resolves which one to call based on the argument types at compile time.

```java
public double calculate(int x) { ... }
public double calculate(double x) { ... }
public double calculate(int x, int y) { ... }
```

Use it when you want to provide convenience variants of an operation — same intent, different input types.

**Overriding** (runtime / dynamic polymorphism): A subclass provides its own implementation of a method already defined in its parent class, with the *exact same signature*. The JVM (or runtime) resolves which version to call based on the actual object type at runtime.

```java
class Animal { public String speak() { return "..."; } }
class Dog extends Animal { @Override public String speak() { return "Woof"; } }
```

Use it when a subclass needs to specialize or replace inherited behavior.

The critical difference: overloading is resolved at *compile time* based on declared types; overriding is resolved at *runtime* based on actual object type. This is why overriding is the engine behind polymorphism and overloading is not.

---

### Q4. What is the Liskov Substitution Principle (LSP) and why is it harder to follow than people think?

**Sample Answer:**

LSP states: if `S` is a subtype of `T`, then objects of type `T` in a program may be replaced with objects of type `S` without altering the correctness of the program.

In plain terms: a subclass should be fully substitutable for its parent class. Any code written against the parent should work correctly when handed a subclass instance.

**Why it's harder than it looks:**

The classic violation is the `Rectangle`/`Square` problem. Mathematically, a square *is* a rectangle. So you make `Square extends Rectangle`. But `Rectangle` has `setWidth()` and `setHeight()` independently. A `Square` must keep width == height, so it overrides both setters to set both dimensions simultaneously. Now:

```java
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(10);
assert r.getArea() == 50; // FAILS — Square gives 100
```

The behavior changed in a way the caller didn't expect. LSP is violated.

The subtlety is that LSP isn't just about method signatures — it's about *behavioral contracts*. Preconditions in a subtype must be no stronger than in the parent. Postconditions must be no weaker. Invariants must be preserved. This requires disciplined thinking about what guarantees a type promises, not just what methods it exposes.

The fix is often to invert the hierarchy — `Shape` → `Rectangle` and `Shape` → `Square` separately, with no inheritance between them.

---

### Q5. Walk me through the SOLID principles and give a real example of violating and fixing each.

**Sample Answer:**

**S — Single Responsibility Principle**
A class should have one and only one reason to change.

*Violation:* A `UserService` class that handles user authentication, sends welcome emails, logs activity, and updates the database. If email templates change, you touch the same class as when auth logic changes.

*Fix:* Split into `AuthService`, `UserEmailService`, `UserAuditLogger`, `UserRepository`.

---

**O — Open/Closed Principle**
Software entities should be open for extension, closed for modification.

*Violation:*
```java
double calculateDiscount(String customerType) {
    if (customerType.equals("VIP")) return 0.2;
    if (customerType.equals("Regular")) return 0.1;
    // Adding "Premium" means editing this method
}
```

*Fix:* Define a `DiscountStrategy` interface and let each customer type implement it. New types are added by creating new classes, not modifying existing ones.

---

**L — Liskov Substitution Principle** *(covered in Q4)*

---

**I — Interface Segregation Principle**
Clients should not be forced to depend on methods they don't use.

*Violation:* A fat `IWorker` interface with `work()`, `eat()`, `sleep()`. A `Robot` class is forced to implement `eat()` and `sleep()` even though robots don't do those things.

*Fix:* Split into `IWorkable`, `IFeedable`, `IRestable`. Classes implement only what applies to them.

---

**D — Dependency Inversion Principle**
High-level modules should not depend on low-level modules. Both should depend on abstractions.

*Violation:*
```java
class OrderService {
    MySQLDatabase db = new MySQLDatabase(); // hard dependency
}
```

*Fix:*
```java
class OrderService {
    private final DatabaseRepository db;
    public OrderService(DatabaseRepository db) { this.db = db; } // injected abstraction
}
```

Now you can swap `MySQLDatabase` for `PostgresDatabase` or a mock in tests without touching `OrderService`.

---

## SECTION 2: Inheritance, Composition, and Design

---

### Q6. "Favor composition over inheritance" — what does this mean and when would you still choose inheritance?

**Sample Answer:**

This is one of the most important design heuristics in OOP. Inheritance creates a very tight coupling — a subclass is bound to the internals of its parent, creating the "fragile base class" problem. If the parent changes, all subclasses can break. Inheritance also forces an "is-a" relationship that can be overly rigid.

Composition ("has-a") wires behaviors together as dependencies rather than class hierarchies. It's more flexible and testable.

**Example — the wrong way with inheritance:**
```java
class Logger extends FileWriter { ... }
```
`Logger` is not *really* a `FileWriter`. It just *uses* one.

**The right way with composition:**
```java
class Logger {
    private final Writer writer; // composed dependency
    public Logger(Writer writer) { this.writer = writer; }
}
```
Now you can swap in a `NetworkWriter`, `DatabaseWriter`, or a mock. No hierarchy needed.

**When inheritance IS appropriate:**
- There is a true, stable "is-a" relationship (a `Dog` genuinely *is* an `Animal`)
- You want to leverage polymorphism and runtime dispatch
- The parent class is designed with extension in mind (template method pattern)
- The hierarchy is shallow (1–2 levels deep) and unlikely to change

The rule of thumb I use: if you're inheriting behavior for code reuse alone, use composition. If you're inheriting to participate in a type hierarchy and polymorphic dispatch, inheritance is legitimate.

---

### Q7. What is the **diamond problem** in multiple inheritance and how do languages solve it?

**Sample Answer:**

The diamond problem occurs when a class inherits from two classes that both inherit from a common ancestor. The question becomes: which version of the ancestor's method does the final class inherit?

```
        A (method: greet())
       / \
      B   C   (both override greet())
       \ /
        D  ← which greet() does D get?
```

**C++ approach:** Allows true multiple inheritance. Uses *virtual inheritance* to ensure only one copy of `A`'s state exists. Without it, `D` would have two copies of `A`. The resolution is explicit but complex.

**Java's approach:** Doesn't allow multiple inheritance of classes at all. You can implement multiple *interfaces*, but interfaces (prior to Java 8) had no implementation. With default methods in Java 8+, a conflict is a compile error — the implementing class *must* override the conflicting method explicitly.

**Python's approach:** Uses the **C3 linearization algorithm** (MRO — Method Resolution Order). It defines a deterministic, consistent ordering of the class hierarchy so there's always a clear winner. You can inspect it with `ClassName.__mro__`.

**Scala/Kotlin approach:** Uses *traits/mixins* with deterministic linearization. The "last wins" rule applies based on the order traits are mixed in.

The real lesson is that the diamond problem reveals why deep multiple inheritance of implementation is dangerous — most modern language designers deliberately constrain it.

---

### Q8. What is the difference between an **abstract class** and an **interface**? When would you choose one over the other?

**Sample Answer:**

**Abstract class:**
- Can have instance variables / state
- Can have constructors
- Can have concrete (implemented) methods
- A class can only extend *one* abstract class
- Expresses an "is-a" relationship with shared implementation

**Interface:**
- Traditionally only method signatures (contracts), though modern languages add default methods
- No instance state (only constants)
- A class can implement *many* interfaces
- Expresses a "can-do" capability contract

**When to use an abstract class:**
When you have a family of related classes that share *common state and behavior* that should not be duplicated. Example: `AbstractHttpHandler` might implement request parsing, logging setup, and error handling, leaving `handleRequest()` abstract for subclasses to implement.

**When to use an interface:**
When you want to define a *capability* that unrelated classes can share. `Serializable`, `Comparable`, `Runnable` — these cut across class hierarchies. A `Dog` and a `Transaction` might both be `Serializable` despite having nothing else in common.

**The nuanced answer for senior candidates:**
In modern Java (8+) and Kotlin, the line has blurred. Interfaces now support default methods and static methods, making them more powerful. My preference: default to interfaces for maximum flexibility unless you genuinely need shared state or a constructor — then use an abstract class. This keeps your design open to multiple implementations and easy to mock in tests.

---

### Q9. What is a **mixin** and how does it differ from traditional inheritance?

**Sample Answer:**

A mixin is a class (or module) that provides methods to other classes without being a true parent class — it's not meant to be instantiated on its own, and it doesn't define the "type" of the class it's mixed into. It's a horizontal code-sharing mechanism, not a vertical hierarchy.

In Python:
```python
class LogMixin:
    def log(self, message):
        print(f"[{self.__class__.__name__}] {message}")

class JsonSerializableMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class User(LogMixin, JsonSerializableMixin):
    def __init__(self, name): self.name = name
```

`User` gains logging and serialization without either being its "parent type." `LogMixin` and `JsonSerializableMixin` are pure capabilities.

**Key differences from traditional inheritance:**
- Mixins carry no semantic "is-a" meaning
- They're composable — you can combine many mixins freely
- They typically don't have their own constructor or state
- They're a form of horizontal reuse vs. vertical specialization

In Scala, this is done via `trait`. In Ruby, via `module`. In TypeScript, it's often achieved via intersection types and object composition patterns.

---

### Q10. Explain the **template method pattern** and how it leverages inheritance correctly.

**Sample Answer:**

The Template Method pattern defines the skeleton of an algorithm in a base class, deferring certain steps to subclasses. The base class calls abstract (or overrideable) methods at specific points in the algorithm flow.

```java
abstract class DataProcessor {
    // Template method — defines the algorithm skeleton
    public final void process() {
        readData();
        processData();
        writeOutput();
        cleanup();
    }

    protected abstract void readData();
    protected abstract void processData();

    protected void writeOutput() {
        System.out.println("Writing to default output...");
    }

    protected void cleanup() { /* default: do nothing */ }
}

class CSVDataProcessor extends DataProcessor {
    @Override protected void readData() { /* parse CSV */ }
    @Override protected void processData() { /* transform rows */ }
    @Override protected void writeOutput() { /* write to CSV */ }
}
```

The `process()` method is `final` — subclasses *cannot* change the algorithm's order or structure. They can only fill in or override specific steps.

**Why this is a legitimate use of inheritance:**
- The parent class genuinely *owns* the algorithm flow
- Subclasses are type-compatible — they *are* `DataProcessor`s
- The relationship is stable and the hierarchy is shallow

This is in contrast to inheritance-for-code-reuse, where there's no meaningful "is-a" relationship. Template Method is inheritance in service of polymorphism, which is its proper purpose.

---

## SECTION 3: Advanced OOP Concepts

---

### Q11. What is **covariance** and **contravariance** in the context of type systems and OOP?

**Sample Answer:**

These concepts describe how subtyping relationships between complex types relate to subtyping relationships between their component types.

**Covariance:** If `Cat` is a subtype of `Animal`, then `List<Cat>` is a subtype of `List<Animal>`. The type relationship is *preserved* in the same direction.

In Java, arrays are covariant:
```java
Animal[] animals = new Cat[3]; // legal — but dangerous
animals[0] = new Dog(); // throws ArrayStoreException at runtime!
```
This is why Java generics are *invariant* by default — to catch these errors at compile time.

**Contravariance:** The type relationship is *inverted*. A `Consumer<Animal>` can be used where a `Consumer<Cat>` is expected — because a consumer that can handle any `Animal` can certainly handle a `Cat`. The relationship flips.

**In Java with wildcards:**
- `List<? extends Animal>` — covariant, read-only safe (producer)
- `List<? super Cat>` — contravariant, write-safe (consumer)

This is the **PECS** principle — **Producer Extends, Consumer Super**.

**In Kotlin:** Types are explicitly marked. `out T` = covariant (can only be produced/returned). `in T` = contravariant (can only be consumed/accepted as parameter).

**Why it matters at the senior level:** Understanding variance is essential for writing correct generic APIs, avoiding runtime type errors, and understanding why certain type assignments are rejected by the compiler.

---

### Q12. What is **object slicing** and in which languages is it a concern?

**Sample Answer:**

Object slicing occurs in languages that support value semantics (like C++) when you assign a derived class object to a base class variable *by value*. The "extra" data introduced by the derived class is literally sliced off.

```cpp
class Animal {
public:
    string name;
    virtual string speak() { return "..."; }
};

class Dog : public Animal {
public:
    string breed;
    string speak() override { return "Woof"; }
};

Dog d;
d.name = "Rex";
d.breed = "Labrador";

Animal a = d; // SLICING — 'breed' is gone, speak() resolves to Animal::speak()
```

After assignment, `a` is just an `Animal`. The `breed` field is gone. If you call `a.speak()`, you get `"..."`, not `"Woof"`, because the vtable pointer has been replaced.

**The fix in C++:** Use pointers or references to base class rather than value copies:
```cpp
Animal* a = &d; // no slicing — polymorphism works correctly
```

**Why Java/C# don't have this problem:** Object variables in these languages are always *references* to heap-allocated objects. Assignment copies the reference, not the object. The full object is preserved.

This is a critical distinction between value semantics and reference semantics in OOP languages, and a common source of subtle bugs in C++ codebases.

---

### Q13. What is a **virtual dispatch table (vtable)** and how does it enable polymorphism?

**Sample Answer:**

A vtable (virtual function table) is a compiler-generated lookup table used to resolve calls to virtual/overridden methods at runtime. It's the mechanism behind dynamic dispatch.

**How it works:**

When a class defines virtual methods, the compiler creates a vtable for that class — essentially an array of function pointers, one per virtual method. Each object of that class has a hidden pointer (`vptr`) that points to its class's vtable.

```
Dog object in memory:
[ vptr ] → Dog's vtable: [ speak → Dog::speak, move → Animal::move, ... ]
[ name  ]
[ breed ]
```

When you call `animal->speak()` through a base pointer, the runtime:
1. Follows the `vptr` to the vtable
2. Looks up the function pointer for `speak`
3. Calls whatever function is pointed to

If the object is actually a `Dog`, the vtable points to `Dog::speak`. If it's a `Cat`, the vtable points to `Cat::speak`. The caller doesn't know or care.

**Performance implications:**
- Virtual dispatch involves an extra pointer indirection — typically 1-2 nanoseconds, usually negligible
- It inhibits inlining, which can matter in tight loops
- This is why C++ lets you choose — non-virtual methods have no overhead, virtual methods enable polymorphism

**In Java and C#:** All non-static, non-private, non-final methods are virtually dispatched by default. The JIT compiler can *devirtualize* calls when it can prove only one implementation will ever be called — often eliminating the overhead entirely.

---

### Q14. What is the difference between **shallow copy** and **deep copy**, and how do you implement deep copy correctly?

**Sample Answer:**

**Shallow copy:** Creates a new object but copies only the top-level field values. For reference fields, it copies the reference — both the original and the copy point to the *same* underlying object.

**Deep copy:** Recursively copies all objects reachable from the original. The copy is fully independent — modifying it doesn't affect the original.

```java
class Address {
    String street;
}

class Person {
    String name;
    Address address;

    // Shallow copy
    Person shallowCopy() {
        Person p = new Person();
        p.name = this.name;
        p.address = this.address; // same Address object!
        return p;
    }

    // Deep copy
    Person deepCopy() {
        Person p = new Person();
        p.name = this.name;
        Address a = new Address();
        a.street = this.address.street; // new Address object
        p.address = a;
        return p;
    }
}
```

**Common pitfalls in deep copy:**
- **Cycles:** Object A references B which references A. Naive deep copy recurses infinitely. Fix: keep a `visited` map of already-copied objects.
- **Shared references:** Sometimes you *want* two objects to share a reference (e.g., a shared configuration). Deep copy would break that intentional sharing.
- **Lazy initialization:** Copying before lazy fields are initialized may produce incorrect state.

**Better approaches in practice:**
- Serialization/deserialization (Jackson, Gson, Java serialization) — handles cycles if designed correctly
- Copy constructors with explicit deep-copy semantics
- Builder pattern that forces all fields to be explicitly set
- Immutable objects — the question becomes moot if objects can't be mutated

**My preference:** Design with immutability where possible. When mutability is necessary, use copy constructors over `clone()` (which is notoriously broken in Java) and document clearly what's shallow vs. deep.

---

### Q15. Explain **method hiding** vs. **method overriding**. Give a concrete example where they produce different results.

**Sample Answer:**

**Overriding** applies to *instance* methods and participates in dynamic dispatch. The method called is determined by the *runtime type* of the object.

**Hiding** (also called *shadowing*) applies to *static* methods (and fields). The method called is determined by the *declared/compile-time type* of the variable.

```java
class Parent {
    public static void staticMethod() { System.out.println("Parent static"); }
    public void instanceMethod() { System.out.println("Parent instance"); }
}

class Child extends Parent {
    public static void staticMethod() { System.out.println("Child static"); }  // HIDES
    @Override
    public void instanceMethod() { System.out.println("Child instance"); }      // OVERRIDES
}

Parent obj = new Child();

obj.staticMethod();   // "Parent static"  ← resolved at compile time by declared type
obj.instanceMethod(); // "Child instance" ← resolved at runtime by actual type
```

This is a common interview trap. Even though `obj` holds a `Child` at runtime, `staticMethod()` resolves to `Parent`'s version because static methods are not polymorphic.

**Why this matters:** It's a subtle source of bugs, especially when refactoring class hierarchies. If you promote an instance method to static without understanding this, polymorphic behavior silently disappears. Most IDEs will warn you, but understanding *why* it behaves this way requires understanding the vtable mechanism — static methods don't go through the vtable.

---

## SECTION 4: Design Patterns (OOP in Practice)

---

### Q16. Explain the **Strategy pattern** and why it's a direct application of OCP and DIP.

**Sample Answer:**

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. The client delegates to a strategy object rather than implementing the algorithm itself.

```java
interface SortStrategy {
    void sort(int[] data);
}

class QuickSort implements SortStrategy {
    public void sort(int[] data) { /* quicksort impl */ }
}

class MergeSort implements SortStrategy {
    public void sort(int[] data) { /* mergesort impl */ }
}

class DataProcessor {
    private SortStrategy strategy;

    public DataProcessor(SortStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }

    public void process(int[] data) {
        strategy.sort(data);
        // ... rest of processing
    }
}
```

**Why it embodies OCP:** `DataProcessor` never needs to change when you add a new sorting algorithm. You just add a new class implementing `SortStrategy`.

**Why it embodies DIP:** `DataProcessor` depends on the *abstraction* `SortStrategy`, not on any concrete implementation. The concrete strategies are injected from outside.

**Real-world applications:**
- Payment processing: `CreditCardStrategy`, `PayPalStrategy`, `CryptoStrategy`
- Compression: `GzipStrategy`, `ZipStrategy`, `LZ4Strategy`
- Pricing/discount engines
- Authentication: `JWTStrategy`, `OAuth2Strategy`, `APIKeyStrategy`

**When NOT to use it:** If there's only one algorithm and it's unlikely to change, Strategy adds unnecessary indirection. Premature abstraction is its own form of technical debt.

---

### Q17. What's the difference between the **Decorator pattern** and **inheritance** for extending behavior?

**Sample Answer:**

Both add behavior, but in fundamentally different ways.

**Inheritance** adds behavior *statically at compile time*. The subclass always has the extra behavior. The combination is baked in.

**Decorator** adds behavior *dynamically at runtime* by wrapping an object. You can stack decorators, remove them, or compose them in different combinations without creating a class explosion.

**The class explosion problem with inheritance:**

Suppose you have a `Coffee` class and want to add condiments: milk, sugar, whip. With inheritance you'd need: `CoffeeWithMilk`, `CoffeeWithSugar`, `CoffeeWithWhip`, `CoffeeWithMilkAndSugar`, `CoffeeWithMilkAndWhip`, ... 2^n combinations.

**With Decorator:**
```java
interface Coffee { double getCost(); String getDescription(); }

class SimpleCoffee implements Coffee {
    public double getCost() { return 1.0; }
    public String getDescription() { return "Coffee"; }
}

abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    public CoffeeDecorator(Coffee c) { this.coffee = c; }
}

class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee c) { super(c); }
    public double getCost() { return coffee.getCost() + 0.25; }
    public String getDescription() { return coffee.getDescription() + ", Milk"; }
}

class WhipDecorator extends CoffeeDecorator { ... }
```

Usage:
```java
Coffee myOrder = new WhipDecorator(new MilkDecorator(new SimpleCoffee()));
// Cost: 1.0 + 0.25 + 0.50 = 1.75
// Description: "Coffee, Milk, Whip"
```

**The tradeoff:** Decorators can produce complex object graphs that are hard to debug ("decorator hell"). Inheritance is simpler when the number of variations is small and fixed. The Decorator shines when behavior combinations are open-ended.

---

### Q18. What is the **Builder pattern** and when is it superior to constructor overloading or the JavaBeans pattern?

**Sample Answer:**

The Builder pattern constructs complex objects step by step, separating object construction from representation.

**Why constructors break down:**

```java
// Telescoping constructors — unreadable and error-prone
new HttpRequest("GET", "https://api.example.com", null, null, true, 5000, false);
// What does 'true' mean? What's the 5000? Which nulls are which?
```

**Why JavaBeans setters break down:**
```java
HttpRequest req = new HttpRequest();
req.setMethod("GET");
req.setUrl("https://api.example.com");
// Object is in an invalid state between the first and last setter call
// Cannot make the object immutable
// Not thread-safe during construction
```

**Builder solves all three:**
```java
HttpRequest request = new HttpRequest.Builder()
    .method("GET")
    .url("https://api.example.com")
    .timeout(5000)
    .followRedirects(true)
    .build(); // validation can happen here, object is immutable after
```

**Key advantages:**
- Named parameters via method chaining — self-documenting
- Object is either fully constructed or not at all — no intermediate invalid state
- The built object can be immutable (final fields set in constructor)
- `build()` can enforce invariants and throw exceptions on invalid combinations
- Easy to add optional parameters without combinatorial constructors

**When NOT to use it:** For simple objects with 2-3 required fields, a Builder is overkill. Use it when you have 4+ parameters, especially optional ones, or when you want to enforce immutability.

---

### Q19. What is the **Observer pattern**, and how does it relate to the **event-driven architecture** used in modern systems?

**Sample Answer:**

The Observer pattern (also called Publish-Subscribe) defines a one-to-many dependency: when a *subject* (publisher) changes state, all registered *observers* (subscribers) are notified automatically.

```java
interface Observer {
    void update(Event event);
}

class EventBus {
    private Map<String, List<Observer>> subscribers = new HashMap<>();

    public void subscribe(String eventType, Observer observer) {
        subscribers.computeIfAbsent(eventType, k -> new ArrayList<>()).add(observer);
    }

    public void publish(String eventType, Event event) {
        subscribers.getOrDefault(eventType, Collections.emptyList())
                   .forEach(o -> o.update(event));
    }
}
```

**How it maps to modern systems:**

The same pattern scales from in-process OOP to distributed systems:
- **In-process:** Java's `EventListener`, JavaScript's `addEventListener`, RxJava `Observable`
- **Message queues:** Kafka topics, RabbitMQ exchanges — publishers write to topics, consumers subscribe to them. The producer never knows who's listening.
- **Reactive frameworks:** RxJava, Project Reactor — streams are asynchronous observer chains
- **Frontend:** React's unidirectional data flow (Redux), component event propagation

**Key design considerations at the senior level:**
- **Push vs. pull models:** Push sends data in the notification; pull sends a signal and the observer fetches data. Push is simpler, pull reduces coupling but adds latency.
- **Synchronous vs. asynchronous notifications:** Synchronous can cause cascading failures and slow publishers. Async via queues decouples timing but introduces eventual consistency.
- **Memory leaks:** If observers are not deregistered, the subject holds strong references to them forever — a classic Android memory leak pattern.
- **Event ordering:** In distributed systems, events may arrive out of order. Designing observers to be idempotent and order-independent is critical.

---

### Q20. Explain the **Singleton pattern**, its typical implementation issues, and when you'd avoid it entirely.

**Sample Answer:**

Singleton ensures a class has at most one instance and provides a global access point to it.

**Classic lazy implementation:**
```java
public class DatabaseConnection {
    private static DatabaseConnection instance;

    private DatabaseConnection() {} // prevent external instantiation

    public static DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection(); // NOT thread-safe
        }
        return instance;
    }
}
```

**Thread-safe — double-checked locking:**
```java
private static volatile DatabaseConnection instance;

public static DatabaseConnection getInstance() {
    if (instance == null) {
        synchronized (DatabaseConnection.class) {
            if (instance == null) {
                instance = new DatabaseConnection();
            }
        }
    }
    return instance;
}
```

The `volatile` is essential — without it, the compiler can reorder instructions and another thread might see a partially-constructed object.

**The best Java approach — initialization-on-demand holder:**
```java
public class DatabaseConnection {
    private DatabaseConnection() {}

    private static class Holder {
        private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }

    public static DatabaseConnection getInstance() {
        return Holder.INSTANCE;
    }
}
```
This is lazy, thread-safe by the class loader guarantee, and has zero synchronization overhead after the first load.

**Why I'd avoid Singleton in modern code:**

1. **It's global state** — hidden dependencies, making code hard to reason about
2. **It kills testability** — you can't swap it out for a mock without hacks
3. **It violates DIP** — classes reach out and grab their dependency instead of receiving it
4. **It makes multithreading treacherous** — shared mutable singleton state is a concurrency nightmare

**The modern alternative:** Use a dependency injection container (Spring, Guice, Dagger) with singleton *scope*. The object is still created once, but it's injected rather than grabbed. Classes remain testable, dependencies are explicit, and the DI framework handles lifecycle.

---

## SECTION 5: Expert-Level & Systems Thinking

---

### Q21. How does the concept of **immutability** relate to OOP, and how do immutable objects improve correctness and concurrency?

**Sample Answer:**

An immutable object's state cannot change after construction. Every field is set once, in the constructor, and never again.

```java
public final class Money {
    private final long amount;
    private final Currency currency;

    public Money(long amount, Currency currency) {
        if (amount < 0) throw new IllegalArgumentException("Amount cannot be negative");
        this.amount = amount;
        this.currency = Objects.requireNonNull(currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amount + other.amount, this.currency); // new object
    }
    // No setters
}
```

**Benefits:**

**Correctness:**
- Objects satisfy their invariants from construction forever. No method can put them into an invalid state.
- Defensive copying is unnecessary — sharing is safe because no one can mutate the shared object.
- Value equality semantics become natural — two `Money(100, USD)` objects are interchangeable.

**Concurrency:**
- Immutable objects are *inherently thread-safe*. No synchronization needed — there's nothing to race on.
- They can be freely shared across threads without locks.
- This is why the functional programming community embraces immutability — shared mutable state is the root of most concurrency bugs.

**The design implication:**
Instead of `account.setBalance(newBalance)`, you have `account.withBalance(newBalance)` which returns a *new* `Account`. Your operations become transformations, not mutations. This is the foundation of event sourcing and functional reactive programming.

**Tradeoffs:**
- Garbage pressure: you create more short-lived objects. In practice, JVM generational GC handles this very well.
- Some domains are naturally mutable (counters, caches) — forcing immutability there adds complexity.
- Use `record` types in Java 14+, `data class` in Kotlin, or `@Value` with Lombok for convenient immutability.

---

### Q22. What is the **Law of Demeter**, and how does violating it produce fragile designs?

**Sample Answer:**

The Law of Demeter (LoD), or "principle of least knowledge," states: a method should only call methods on objects it directly owns — not on objects returned by those objects.

In plain English: **don't talk to strangers.** Don't chain through object graphs you don't own.

**The classic violation — "train wreck":**
```java
// BAD
double price = order.getCustomer().getLoyaltyAccount().getCurrentTier().getDiscount();
```

This method knows about `Order`, `Customer`, `LoyaltyAccount`, `Tier`, and their relationships. If any of those change — if `LoyaltyAccount` is renamed, or if the discount logic moves — this line breaks.

**The fix:**
```java
// GOOD
double price = order.getDiscountRate();
// Order asks Customer, Customer asks LoyaltyAccount internally
```

`order` is a single point of contact. The method talks to one object. The object graph navigation is hidden.

**Why the chain form is fragile:**
- It creates *structural coupling* — your code knows the internal structure of remote objects
- It violates encapsulation — the `Customer` and `Tier` implementation details leak everywhere
- Changes ripple broadly — modify one hop in the chain and many callers break
- It's nearly impossible to mock in tests without stubbing a full chain

**Legitimate exception:** Fluent interfaces (builders, stream pipelines) and DSLs intentionally chain calls, but they operate on *the same object* throughout — `stream.filter(...).map(...).collect(...)` always returns the same `Stream` type. This is not a LoD violation because you're not navigating to a stranger.

---

### Q23. How would you design a system to support **plugin architecture** using OOP principles?

**Sample Answer:**

A plugin architecture allows third-party code to extend a system without modifying its core. This is a direct application of OCP, DIP, and interface-based design.

**Core design:**

1. **Define a plugin contract (interface):**
```java
public interface PaymentPlugin {
    String getName();
    boolean supports(PaymentRequest request);
    PaymentResult process(PaymentRequest request);
}
```

2. **Plugin registry:**
```java
public class PluginRegistry {
    private final List<PaymentPlugin> plugins = new ArrayList<>();

    public void register(PaymentPlugin plugin) {
        plugins.add(plugin);
    }

    public PaymentPlugin find(PaymentRequest request) {
        return plugins.stream()
            .filter(p -> p.supports(request))
            .findFirst()
            .orElseThrow(() -> new NoPluginFoundException(request));
    }
}
```

3. **Plugin discovery:**
Use Java's `ServiceLoader` for classpath-based discovery, or a directory scanner for JAR-based plugins:
```java
ServiceLoader<PaymentPlugin> loader = ServiceLoader.load(PaymentPlugin.class);
loader.forEach(registry::register);
```

4. **Isolation:** Each plugin runs in its own `ClassLoader` if you need to prevent version conflicts (like OSGi does).

5. **Plugin lifecycle:**
```java
public interface Plugin {
    void onLoad(PluginContext context);
    void onUnload();
}
```

**Design principles at work:**
- OCP: The core never changes to add a new plugin
- DIP: Core depends on `PaymentPlugin` abstraction, not any concrete payment provider
- ISP: Keep plugin interfaces narrow — a plugin shouldn't need to implement methods it doesn't care about

**Real-world examples of this pattern:** IntelliJ IDEA plugins, Webpack loaders, Jenkins plugins, Minecraft mods, VS Code extensions, Gradle tasks.

---

### Q24. What are the tradeoffs between **anemic domain models** and **rich domain models**?

**Sample Answer:**

This is a classic debate in enterprise software design.

**Anemic Domain Model:** Domain objects are just data containers — plain POJOs with getters and setters. All business logic lives in separate service classes.

```java
// Anemic
class Order {
    private List<OrderLine> lines;
    private OrderStatus status;
    // Just getters and setters
}

class OrderService {
    public void placeOrder(Order order) {
        // ALL logic here
        if (order.getLines().isEmpty()) throw new Exception("...");
        order.setStatus(OrderStatus.PLACED);
        // ...
    }
}
```

**Rich Domain Model:** Domain objects carry both state *and* behavior. Business rules live in the objects they're about.

```java
// Rich
class Order {
    private List<OrderLine> lines;
    private OrderStatus status;

    public void place() {
        if (lines.isEmpty()) throw new EmptyOrderException();
        if (status != OrderStatus.DRAFT) throw new InvalidStateException();
        this.status = OrderStatus.PLACED;
        // emit domain event, calculate totals, etc.
    }

    public Money totalAmount() {
        return lines.stream().map(OrderLine::lineTotal).reduce(Money.ZERO, Money::add);
    }
}
```

**Arguments for anemic models:**
- Simpler to understand at a glance
- Easier for ORMs to work with (direct field mapping)
- Familiar to developers coming from procedural or data-centric backgrounds
- Works well for CRUD-heavy systems with little real business logic

**Arguments for rich models:**
- Business rules are colocated with the data they govern — enforced at the object level
- Invariants are harder to violate — the object controls its own state transitions
- Code reads closer to the business domain
- Aligns with Domain-Driven Design (DDD) principles

**My take:** Anemic models are often a symptom of treating OOP as a data-modeling tool rather than a behavior-modeling tool. For complex domains with real business rules, rich models reduce scattered logic and enforce invariants. For simple CRUD services with no real domain complexity, an anemic model may be perfectly appropriate — don't add complexity where there's no complexity to manage.

---

### Q25. How do you think about **coupling and cohesion** as metrics for OOP design quality?

**Sample Answer:**

These two metrics are arguably the most important measures of design quality in any paradigm, especially OOP.

**Cohesion** measures how focused a class is. High cohesion means every element of a class belongs together — they all serve a single, well-defined purpose. A `CustomerReportGenerator` that also handles database connections and sends emails has low cohesion.

**Coupling** measures how much one class knows about or depends on another. Tight coupling means changes in one class ripple to others. Loose coupling means classes interact through stable abstractions.

**The goal: high cohesion, low coupling.**

**Types of coupling (from worst to least bad):**
- **Content coupling:** One class directly modifies another's internal state
- **Common coupling:** Classes share global state
- **Control coupling:** One class passes a flag to another to control its behavior
- **Data coupling:** Classes share data through parameters only (acceptable)
- **Message coupling:** Classes interact only through method calls on interfaces (ideal)

**Measuring cohesion — LCOM (Lack of Cohesion in Methods):**
If you take all the methods of a class and look at which instance variables each method uses, a highly cohesive class has high overlap. If some methods use fields A and B and others use fields C and D with no overlap, the class might actually be two classes pretending to be one.

**Practical heuristics:**
- If you can't describe what a class does in one sentence without "and," it has low cohesion
- If changing a requirement requires touching many files, you have tight coupling
- If your unit tests require extensive mocking, classes are too tightly coupled
- If a class has more than 5-7 injected dependencies, it's probably violating SRP and has low cohesion

The path from bad design to good design is almost always: break large, low-cohesion classes into smaller, focused ones, and wire them together through abstractions (reducing coupling). This is OOP done right.

---

*These 25 questions cover the full spectrum from foundational concepts to systems-level design thinking. A candidate who answers all of them at this depth has a genuine, internalized understanding of OOP — not just memorized definitions.*