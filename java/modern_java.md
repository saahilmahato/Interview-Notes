# ☕ Modern Java Features — Complete Cheatsheet

> Covers Java 8 through Java 21 (LTS). All the features you actually need to know.

---

## 📦 1. Lambda Expressions (Java 8)

A lambda is an **anonymous function** — a short block of code that can be passed around like a value.

**Syntax:**
```java
(parameters) -> expression
(parameters) -> { statements; }
```

**Examples:**
```java
// Old way
Runnable r = new Runnable() {
    public void run() { System.out.println("Hello"); }
};

// Lambda way
Runnable r = () -> System.out.println("Hello");

// With parameters
Comparator<String> c = (a, b) -> a.compareTo(b);

// Multi-line
Comparator<String> c = (a, b) -> {
    System.out.println("Comparing...");
    return a.compareTo(b);
};
```

**Key Rule:** Lambdas can only be used where a **Functional Interface** is expected (an interface with exactly one abstract method).

---

## 🔌 2. Functional Interfaces (Java 8)

An interface with **exactly one abstract method**. Annotate with `@FunctionalInterface` (optional but recommended).

**Built-in ones you must know:**

| Interface | Method | Use Case |
|---|---|---|
| `Function<T, R>` | `R apply(T t)` | Transform input to output |
| `Consumer<T>` | `void accept(T t)` | Do something with input, no return |
| `Supplier<T>` | `T get()` | Return something, no input |
| `Predicate<T>` | `boolean test(T t)` | Test a condition |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs, one output |
| `UnaryOperator<T>` | `T apply(T t)` | Same type in and out |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | Two of same type in, same type out |

```java
Function<String, Integer> length = s -> s.length();
Predicate<String> isEmpty = s -> s.isEmpty();
Supplier<String> greeting = () -> "Hello!";
Consumer<String> print = s -> System.out.println(s);

// Chaining
Function<String, String> trim = String::trim;
Function<String, Integer> trimAndLength = trim.andThen(String::length);
```

---

## 🔗 3. Method References (Java 8)

A shorthand for lambdas that just call an existing method.

```java
// Lambda              →  Method Reference
s -> s.toUpperCase()   →  String::toUpperCase
s -> System.out.println(s)  →  System.out::println
() -> new ArrayList<>()     →  ArrayList::new
```

**Four types:**

```java
// 1. Static method
Function<String, Integer> parseInt = Integer::parseInt;

// 2. Instance method on a specific object
String prefix = "Hello";
Predicate<String> startsWithHello = prefix::equals;

// 3. Instance method on an arbitrary object of the type
Function<String, String> upper = String::toUpperCase;

// 4. Constructor reference
Supplier<ArrayList<String>> listFactory = ArrayList::new;
```

---

## 🌊 4. Streams API (Java 8)

A stream is a **pipeline of operations** on a sequence of elements. It doesn't store data — it processes it lazily.

**Three stages:**
1. **Source** — where data comes from
2. **Intermediate operations** — transform the stream (lazy, return another stream)
3. **Terminal operation** — triggers execution, produces result

```
Source → filter() → map() → sorted() → collect()
                 (intermediate)          (terminal)
```

### Creating Streams
```java
Stream.of("a", "b", "c")
List.of(1, 2, 3).stream()
Arrays.stream(new int[]{1, 2, 3})
Stream.iterate(0, n -> n + 1)          // infinite
Stream.generate(Math::random)           // infinite
IntStream.range(0, 10)                  // 0 to 9
IntStream.rangeClosed(1, 10)            // 1 to 10
```

### Intermediate Operations (lazy)
```java
.filter(Predicate)          // keep matching elements
.map(Function)              // transform each element
.flatMap(Function)          // map then flatten (1-to-many)
.distinct()                 // remove duplicates
.sorted()                   // natural order
.sorted(Comparator)         // custom order
.limit(n)                   // take first n
.skip(n)                    // skip first n
.peek(Consumer)             // look at element without changing (debug)
.mapToInt/Long/Double()     // convert to primitive stream
```

### Terminal Operations (eager — trigger execution)
```java
.collect(Collectors.toList())
.collect(Collectors.toSet())
.collect(Collectors.toMap(keyFn, valueFn))
.collect(Collectors.joining(", "))
.collect(Collectors.groupingBy(Function))
.collect(Collectors.partitioningBy(Predicate))
.forEach(Consumer)
.count()
.findFirst()        // returns Optional
.findAny()          // returns Optional (better for parallel)
.anyMatch(Predicate)
.allMatch(Predicate)
.noneMatch(Predicate)
.min(Comparator)    // returns Optional
.max(Comparator)    // returns Optional
.reduce(identity, BinaryOperator)
.toArray()
```

### Real Examples
```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Anna");

// Filter names starting with A, uppercase them, sort, collect
List<String> result = names.stream()
    .filter(n -> n.startsWith("A"))
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());
// [ALICE, ANNA]

// Sum of even numbers
int sum = IntStream.rangeClosed(1, 10)
    .filter(n -> n % 2 == 0)
    .sum();
// 30

// Group by first letter
Map<Character, List<String>> grouped = names.stream()
    .collect(Collectors.groupingBy(n -> n.charAt(0)));
// {A=[Alice, Anna], B=[Bob], C=[Charlie]}

// FlatMap example
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4));
List<Integer> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());
// [1, 2, 3, 4]

// Reduce example
int product = Stream.of(1, 2, 3, 4)
    .reduce(1, (a, b) -> a * b);
// 24
```

### Parallel Streams
```java
// Just add .parallel() — uses ForkJoinPool under the hood
long count = names.parallelStream()
    .filter(n -> n.length() > 3)
    .count();
```
⚠️ Not always faster. Good for CPU-bound tasks on large datasets. Bad for small lists or IO-bound tasks.

---

## 🎁 5. Optional (Java 8)

A container that **may or may not** contain a value. Eliminates `NullPointerException` when used correctly.

```java
// Creating
Optional<String> opt1 = Optional.of("hello");       // throws NPE if null
Optional<String> opt2 = Optional.ofNullable(null);  // safe, can be null
Optional<String> opt3 = Optional.empty();

// Checking and getting
opt1.isPresent()    // true
opt1.isEmpty()      // false (Java 11+)
opt1.get()          // "hello" — throws if empty, avoid using alone

// Safe access
opt1.orElse("default")                 // value or default
opt1.orElseGet(() -> "default")        // value or supplier result (lazy)
opt1.orElseThrow(() -> new RuntimeException("Not found"))

// Transform
opt1.map(String::toUpperCase)          // Optional<String> "HELLO"
opt1.flatMap(s -> Optional.of(s + "!"))
opt1.filter(s -> s.length() > 3)

// ifPresent
opt1.ifPresent(System.out::println);
opt1.ifPresentOrElse(
    s -> System.out.println("Found: " + s),
    () -> System.out.println("Not found")
);
```

**Best practice:** Use Optional as a return type for methods that might not return a value. Never use it as method parameters or fields.

---

## 🖥️ 6. Default & Static Interface Methods (Java 8)

Interfaces can now have **method implementations**.

```java
interface Greeter {
    String greet(String name);  // abstract (must implement)

    default String greetLoudly(String name) {   // optional to override
        return greet(name).toUpperCase();
    }

    static Greeter formal() {  // called on the interface itself
        return name -> "Good day, " + name;
    }
}

Greeter g = name -> "Hello, " + name;
g.greet("Alice");         // "Hello, Alice"
g.greetLoudly("Alice");   // "HELLO, ALICE"
Greeter.formal().greet("Bob");  // "Good day, Bob"
```

---

## 🕐 7. New Date/Time API (Java 8) — `java.time`

Replaces the broken `java.util.Date` and `Calendar`.

```java
// Immutable, thread-safe classes
LocalDate date = LocalDate.now();              // 2024-01-15
LocalTime time = LocalTime.now();              // 14:30:00
LocalDateTime dateTime = LocalDateTime.now();  // 2024-01-15T14:30:00
ZonedDateTime zoned = ZonedDateTime.now(ZoneId.of("America/New_York"));
Instant instant = Instant.now();               // machine timestamp (epoch)

// Creating specific dates
LocalDate birthday = LocalDate.of(1990, Month.MAY, 15);
LocalDate parsed = LocalDate.parse("2024-01-15");

// Manipulation (returns new object — immutable!)
LocalDate nextWeek = date.plusDays(7);
LocalDate lastYear = date.minusYears(1);

// Comparing
date.isBefore(nextWeek);   // true
date.isAfter(nextWeek);    // false

// Duration and Period
Duration d = Duration.between(time, time.plusHours(2));  // time-based
Period p = Period.between(birthday, date);                // date-based
p.getYears();  // age

// Formatting
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy");
date.format(fmt);            // "15/01/2024"
LocalDate.parse("15/01/2024", fmt);
```

---

## 📝 8. Text Blocks (Java 15)

Multi-line strings without escape sequences.

```java
// Old way
String json = "{\n" +
              "  \"name\": \"Alice\",\n" +
              "  \"age\": 30\n" +
              "}";

// Text block
String json = """
        {
          "name": "Alice",
          "age": 30
        }
        """;
```
The indentation relative to the closing `"""` is stripped automatically.

---

## 🔒 9. Records (Java 16)

Compact, immutable data classes. Automatically generates constructor, getters, `equals()`, `hashCode()`, `toString()`.

```java
// Declaration
record Point(int x, int y) {}

// Use
Point p = new Point(3, 5);
p.x();          // 3 (accessor, not getX())
p.y();          // 5
p.toString();   // Point[x=3, y=5]

// Custom compact constructor (for validation)
record Range(int min, int max) {
    Range {   // compact constructor — no parameter list needed
        if (min > max) throw new IllegalArgumentException("min > max");
    }
}

// Can have methods
record Circle(double radius) {
    double area() { return Math.PI * radius * radius; }
}
```
Records **cannot** have mutable state, cannot extend classes (they extend `Record` implicitly), but **can** implement interfaces.

---

## 🔀 10. Sealed Classes (Java 17)

Restricts which classes can extend/implement a class or interface. Perfect for modeling **closed hierarchies**.

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}

record Circle(double radius) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
final class Triangle implements Shape {
    // ...
}

// Every permitted class must be: final, sealed, or non-sealed
```

The power comes with Pattern Matching (below) — the compiler knows the exhaustive set of subtypes.

---

## 🧩 11. Pattern Matching (Java 16–21)

### `instanceof` Pattern Matching (Java 16)
```java
// Old
if (obj instanceof String) {
    String s = (String) obj;  // redundant cast
    System.out.println(s.length());
}

// New
if (obj instanceof String s) {
    System.out.println(s.length());  // s is already String
}

// With condition
if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
}
```

### Pattern Matching in `switch` (Java 21)
```java
static String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "int: " + i;
        case String s  -> "string of length " + s.length();
        case null      -> "null value";
        default        -> "something else";
    };
}

// With sealed classes — exhaustive, no default needed!
static double area(Shape shape) {
    return switch (shape) {
        case Circle c     -> Math.PI * c.radius() * c.radius();
        case Rectangle r  -> r.w() * r.h();
        case Triangle t   -> t.base() * t.height() / 2;
    };
}

// Guarded patterns (when clause)
static String classify(Object o) {
    return switch (o) {
        case Integer i when i < 0  -> "negative int";
        case Integer i when i == 0 -> "zero";
        case Integer i             -> "positive int";
        default                    -> "not an int";
    };
}
```

---

## 🔄 12. Switch Expressions (Java 14)

Switch that **returns a value**. Cleaner than old switch statements.

```java
// Old switch statement (fall-through prone)
int day = 3;
String name;
switch(day) {
    case 1: name = "Monday"; break;
    case 2: name = "Tuesday"; break;
    default: name = "Other";
}

// New switch expression
String name = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 3, 4 -> "Midweek";    // multiple labels!
    default -> "Other";
};

// With yield for blocks
String name = switch (day) {
    case 1 -> "Monday";
    default -> {
        String result = "Day " + day;
        yield result;    // yield returns from a block
    }
};
```

---

## 🧵 13. Virtual Threads (Java 21) — Project Loom

Lightweight threads managed by the JVM, not the OS. Millions can run concurrently.

```java
// Create a virtual thread
Thread vt = Thread.ofVirtual().start(() -> System.out.println("Virtual!"));

// Using Executors (preferred)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));  // Blocks virtual, not OS thread!
            return "done";
        });
    }
}
```
**Why it matters:** Traditional threads are expensive OS resources (~1MB stack). Virtual threads are cheap (~few KB). You can now write simple blocking code that scales like reactive code, without callbacks or reactive frameworks.

---

## 📌 14. `var` — Local Variable Type Inference (Java 10)

Let the compiler infer the type. Only for **local variables**.

```java
var list = new ArrayList<String>();   // ArrayList<String>
var map = new HashMap<String, List<Integer>>();
var stream = list.stream();

// Works great with complex generics and for-each
for (var entry : map.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
}
```
⚠️ Cannot use with fields, method parameters, or return types. The type is still static — this is NOT dynamic typing.

---

## 📚 15. Collectors Deep Dive

```java
// groupingBy with downstream collector
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));

// groupingBy with mapping
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.mapping(Employee::name, Collectors.toList())
    ));

// partitioningBy
Map<Boolean, List<String>> partition = names.stream()
    .collect(Collectors.partitioningBy(s -> s.length() > 4));
// {true=[Alice, Charlie], false=[Bob, Anna]}

// joining
String result = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie, Anna]"

// toUnmodifiableList (Java 10)
List<String> immutable = names.stream()
    .collect(Collectors.toUnmodifiableList());

// teeing (Java 12) — two collectors, then merge
var stats = Stream.of(1, 2, 3, 4, 5)
    .collect(Collectors.teeing(
        Collectors.summingInt(Integer::intValue),
        Collectors.counting(),
        (sum, count) -> "Sum=" + sum + ", Count=" + count
    ));
```

---

## 📬 16. CompletableFuture (Java 8)

Async programming without blocking threads.

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchFromDB())       // runs in ForkJoinPool
    .thenApply(data -> process(data))       // transform result
    .thenApply(String::toUpperCase)
    .exceptionally(ex -> "Error: " + ex.getMessage());

future.get(); // block and get (or thenAccept for non-blocking)

// Combine two futures
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> " World");

CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + b);

// Wait for all
CompletableFuture.allOf(f1, f2).join();

// First to complete
CompletableFuture.anyOf(f1, f2).join();
```

---

## 🔤 17. String Enhancements

```java
// Java 11
"  hello  ".strip()         // Unicode-aware trim (unlike trim())
"  hello  ".stripLeading()
"  hello  ".stripTrailing()
"".isBlank()                // true (whitespace-only = blank)
"a\nb\nc".lines()           // Stream<String>
"abc".repeat(3)             // "abcabcabc"

// Java 12
"hello".indent(4)           // adds spaces to each line

// Java 15
String.formatted("Hello %s", "world")  // instance method alternative to String.format()
```

---

## 🗂️ 18. Collection Factory Methods (Java 9)

```java
List<String> list = List.of("a", "b", "c");       // immutable
Set<String> set = Set.of("x", "y", "z");           // immutable
Map<String, Integer> map = Map.of("a", 1, "b", 2); // immutable, up to 10 entries
Map<String, Integer> map2 = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2)
    // unlimited entries
);
```
⚠️ These are **truly immutable** — any mutation throws `UnsupportedOperationException`.

---

## 🔄 19. Map Enhancements (Java 8+)

```java
Map<String, Integer> map = new HashMap<>();

// getOrDefault
map.getOrDefault("key", 0);

// putIfAbsent
map.putIfAbsent("key", 100);

// computeIfAbsent — compute and put if key missing
map.computeIfAbsent("key", k -> k.length());

// computeIfPresent — update if key exists
map.computeIfPresent("key", (k, v) -> v + 1);

// compute — always compute
map.compute("key", (k, v) -> v == null ? 1 : v + 1);

// merge — great for counting/accumulating
map.merge("key", 1, Integer::sum);  // if absent put 1, else add 1

// forEach
map.forEach((k, v) -> System.out.println(k + "=" + v));

// replaceAll
map.replaceAll((k, v) -> v * 2);
```

---

## 🏗️ 20. Structured Concurrency (Java 21 — Preview)

Groups related tasks so they live and die together.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user  = scope.fork(() -> fetchUser(id));
    Future<Order> order  = scope.fork(() -> fetchOrder(id));
    
    scope.join();           // wait for all
    scope.throwIfFailed();  // propagate errors
    
    return new Response(user.resultNow(), order.resultNow());
}
// If either task fails, the other is cancelled automatically
```

---

## ✅ Common Interview Questions & Answers

---

**Q1: What is the difference between `map()` and `flatMap()` in streams?**

`map()` transforms each element 1-to-1, returning a `Stream<Stream<T>>` if the mapping function returns a stream. `flatMap()` transforms each element and **flattens** the result into a single stream.

```java
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4));
nested.stream().map(List::stream);      // Stream<Stream<Integer>> — not useful
nested.stream().flatMap(List::stream);  // Stream<Integer> — [1, 2, 3, 4]
```

---

**Q2: What is the difference between `findFirst()` and `findAny()`?**

`findFirst()` always returns the **first element** in encounter order. `findAny()` may return **any element** — it's better for parallel streams since it doesn't enforce ordering, so it can be faster. Both return `Optional`.

---

**Q3: What's the difference between `Optional.of()` and `Optional.ofNullable()`?**

`Optional.of(value)` throws `NullPointerException` if value is null. Use it when you're sure the value isn't null. `Optional.ofNullable(value)` safely creates an empty Optional if the value is null.

---

**Q4: Why should you not use `Optional` as method parameters?**

It forces callers to wrap values in Optional unnecessarily, creating noise. APIs should accept plain values and handle nullability internally. Optional is designed as a **return type** to signal "this might not have a value."

---

**Q5: Are streams lazy? Explain.**

Yes. Intermediate operations (`filter`, `map`, etc.) are **lazy** — they build a pipeline but don't execute. Execution only happens when a **terminal operation** (`collect`, `count`, `forEach`, etc.) is called. This allows optimizations like short-circuiting:

```java
Stream.of(1, 2, 3, 4, 5)
    .filter(n -> { System.out.println("filtering " + n); return n > 3; })
    .findFirst();  // Only processes until it finds first match (4), stops there
```

---

**Q6: What is the difference between intermediate and terminal operations?**

Intermediate operations return a new `Stream` and are lazy (e.g., `filter`, `map`, `sorted`). Terminal operations trigger the pipeline to execute and return a non-stream result or produce a side effect (e.g., `collect`, `count`, `forEach`). A stream can only be consumed by **one terminal operation**.

---

**Q7: Can you reuse a stream?**

No. Once a terminal operation has been called on a stream, it is **closed/consumed**. Using it again throws `IllegalStateException`. You must create a new stream.

---

**Q8: What's the difference between `Comparable` and `Comparator`?**

`Comparable` is implemented by the class itself (`compareTo()`) — it defines the natural ordering. `Comparator` is an external comparison strategy — it's a functional interface used to define custom/multiple orderings without modifying the class.

```java
// Comparator as lambda
list.sort((a, b) -> a.getName().compareTo(b.getName()));
// Or using Comparator.comparing
list.sort(Comparator.comparing(Person::getName).thenComparing(Person::getAge));
```

---

**Q9: What's the difference between a `record` and a regular class?**

A `record` is a special kind of class that is: **immutable** (all fields are final), **transparent** (components are part of the public API), and auto-generates a canonical constructor, accessor methods (not `getX()`, just `x()`), `equals()`, `hashCode()`, and `toString()`. Records cannot extend other classes and cannot have non-final instance fields.

---

**Q10: What are virtual threads and why do they matter?**

Virtual threads (Java 21) are lightweight threads managed by the JVM rather than the OS. Traditional platform threads are expensive (each one maps to an OS thread, ~1MB stack). Virtual threads are very cheap (~few KB), allowing you to run **millions concurrently**. When a virtual thread blocks (e.g., on I/O), the JVM parks it and reassigns the carrier (OS) thread to another virtual thread. This lets you write simple, imperative blocking code that scales as well as reactive/async code without the complexity.

---

**Q11: What's the difference between `sealed` classes and `final` classes?**

A `final` class cannot be subclassed at all. A `sealed` class **controls exactly which classes** can extend/implement it using the `permits` clause. Permitted subclasses must be `final`, `sealed`, or `non-sealed`. Sealed classes enable exhaustive pattern matching in switch expressions.

---

**Q12: What is `reduce()` in streams?**

`reduce()` combines all elements into a single result using a `BinaryOperator`.

```java
// With identity (initial value) — always returns T
int sum = Stream.of(1, 2, 3).reduce(0, Integer::sum);  // 6

// Without identity — returns Optional<T>
Optional<Integer> max = Stream.of(1, 2, 3).reduce(Integer::max);
```

---

**Q13: What's the difference between `stream()` and `parallelStream()`?**

`stream()` processes elements **sequentially** in a single thread. `parallelStream()` splits the work across multiple threads using `ForkJoinPool.commonPool()`. Parallel is not always faster — it has overhead for splitting/merging and can cause issues with shared mutable state. Use it only for CPU-intensive, stateless operations on large datasets.

---

**Q14: Explain `Collectors.groupingBy()` with a downstream collector.**

`groupingBy()` groups stream elements by a classifier function into a `Map`. A downstream collector further processes each group:

```java
// Count employees per department
Map<String, Long> countPerDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

// Average salary per department
Map<String, Double> avgSalary = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.averagingDouble(Employee::getSalary)
    ));
```

---

**Q15: What is the difference between `orElse()` and `orElseGet()` in Optional?**

`orElse(value)` always evaluates the argument, even if the Optional is present. `orElseGet(supplier)` is **lazy** — the supplier is only called if the Optional is empty. Prefer `orElseGet()` when the fallback is expensive to compute.

```java
Optional<String> opt = Optional.of("hello");
opt.orElse(expensiveMethod());        // expensiveMethod() is ALWAYS called
opt.orElseGet(() -> expensiveMethod()); // expensiveMethod() NOT called here
```

---

**Q16: What's the purpose of `peek()` in streams?**

`peek()` is an intermediate operation that lets you look at elements as they pass through the pipeline without modifying them. It's mainly used for **debugging**:

```java
stream.filter(x -> x > 2)
      .peek(x -> System.out.println("After filter: " + x))
      .map(x -> x * 2)
      .peek(x -> System.out.println("After map: " + x))
      .collect(Collectors.toList());
```
⚠️ Don't rely on `peek()` for side effects in production code — it may not be called if the terminal operation doesn't need all elements.

---

**Q17: What's the difference between `String::trim` and `String::strip` (Java 11)?**

`trim()` removes characters `<= '\u0020'` (ASCII space), while `strip()` uses Unicode whitespace definitions and is aware of all Unicode whitespace characters. Always prefer `strip()` for modern applications.

---

## 🗺️ Quick Reference: Java Version → Feature

| Version | Key Features |
|---|---|
| Java 8 | Lambdas, Streams, Optional, Date/Time API, Default Methods, Method Refs |
| Java 9 | `List.of()`, `Map.of()`, `Stream.iterate()` with predicate, `Optional.ifPresentOrElse()` |
| Java 10 | `var` keyword, `Collectors.toUnmodifiableList()` |
| Java 11 | `String.strip()`, `isBlank()`, `lines()`, `repeat()`, `Optional.isEmpty()` |
| Java 12 | `Collectors.teeing()`, Switch Expressions (preview) |
| Java 14 | Switch Expressions (stable), Records (preview) |
| Java 15 | Text Blocks (stable), Sealed Classes (preview) |
| Java 16 | Records (stable), `instanceof` Pattern Matching (stable) |
| Java 17 | Sealed Classes (stable) — LTS |
| Java 21 | Virtual Threads, Pattern Matching in switch, Sequenced Collections, Structured Concurrency — LTS |

---

> 💡 **Pro tip for interviews:** Always explain the *why* behind each feature, not just the *what*. Interviewers love when you say "Virtual threads matter because they allow you to scale without reactive programming complexity" rather than just defining them.