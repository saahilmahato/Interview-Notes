# ☕ Java — Complete Cheatsheet & Notes (Enhanced)

> **Changelog from original:** One factual correction (String Pool location), HashMap internals, double-checked locking Singleton, PECS for generics, Integer cache gotcha, modern Java features (sealed classes, pattern matching, text blocks), GC collector landscape, CompletableFuture, and deeper interview Q&A.

---

## 🧠 What is Java?

Java is a **statically typed, object-oriented, compiled + interpreted** language. You write `.java` source code → the Java compiler (`javac`) compiles it into **bytecode** (`.class` files) → the **JVM (Java Virtual Machine)** interprets/JIT-compiles that bytecode into native machine code at runtime.

This is why Java is **"Write Once, Run Anywhere"** — the bytecode is platform-neutral; only the JVM is platform-specific.

```
YourCode.java  →  javac  →  YourCode.class (bytecode)  →  JVM  →  OS / CPU
```

---

## 🏗️ JDK vs JRE vs JVM

| Term | Stands For | What it is |
|------|-----------|------------|
| **JVM** | Java Virtual Machine | Runtime that executes bytecode |
| **JRE** | Java Runtime Environment | JVM + standard libraries |
| **JDK** | Java Development Kit | JRE + compiler + dev tools |

> You **develop** with JDK. You **run** with JRE. The **JVM** does the actual execution.
> Note: Since Java 11, standalone JRE distributions were dropped. JDK is now the standard distribution.

---

## 📄 Anatomy of a Java Program

```java
// Single-line comment
/* Multi-line comment */
/** Javadoc comment */

package com.example;           // Package declaration (optional, must be first)

import java.util.Scanner;      // Import statement

public class HelloWorld {      // Class declaration

    public static void main(String[] args) {   // Entry point
        System.out.println("Hello, World!");   // Statement
    }
}
```

### What happens at low level:
- `package com.example` → tells the JVM this class lives in the `com/example/` directory hierarchy. The classloader uses this to locate `.class` files.
- `import java.util.Scanner` → a **compile-time directive**. It tells the compiler to look up the `Scanner` class from `java.util`. No runtime cost — it's just a namespace shortcut.
- `public class HelloWorld` → the JVM loads this class via the **ClassLoader** into the **Method Area** (part of JVM memory). The class name **must match** the filename.
- `public static void main(String[] args)` → the JVM specifically looks for this exact signature as the program entry point. `static` means no object is needed — the JVM can call it directly on the class.
- `System.out.println(...)` → `System` is a class, `out` is a static field of type `PrintStream`, `println` is a method call. At the bytecode level this becomes an `invokevirtual` instruction.

---

## 🔑 Keywords — Every One Explained

### Access Modifiers
| Keyword | Meaning | Low-level behavior |
|---------|---------|-------------------|
| `public` | Accessible from anywhere | JVM sets access flag `ACC_PUBLIC` on the class/member |
| `private` | Accessible only within the same class | JVM sets `ACC_PRIVATE`; access from outside throws `IllegalAccessError` |
| `protected` | Accessible within same package + subclasses | `ACC_PROTECTED` flag |
| *(default/package-private)* | No keyword, accessible within same package | No access flag set |

### Class & Object Keywords
| Keyword | Meaning | Low-level behavior |
|---------|---------|-------------------|
| `class` | Defines a class (blueprint for objects) | JVM loads this into **Method Area**. Contains method table (vtable), field info, constant pool |
| `interface` | Defines a contract (abstract method signatures) | Stored in Method Area; marked `ACC_INTERFACE`. No instance creation. Methods are abstract by default. Since Java 8: `default`/`static` methods allowed. Since Java 9: `private` methods allowed. |
| `abstract` | Class cannot be instantiated; method has no body | JVM sets `ACC_ABSTRACT`. Attempting `new AbstractClass()` → compile error |
| `extends` | Inherits from a superclass | JVM links the child's class structure to the parent's. Method calls walk up the **vtable** chain |
| `implements` | Class agrees to fulfill an interface contract | JVM creates an **itable** (interface method table) for dispatch |
| `new` | Creates a new object on the heap | JVM executes `new` bytecode instruction → allocates memory on **Heap**, runs constructor via `invokespecial` |
| `this` | Reference to the current object | At runtime, `this` is the first implicit argument passed to every instance method. Points to the object's address on the heap |
| `super` | Reference to the parent class | Allows calling parent's constructor/methods via `invokespecial` (bypasses virtual dispatch) |
| `instanceof` | Checks if object is an instance of a type | JVM checks the object's class metadata chain at runtime. Java 16+ supports **pattern matching**: `if (obj instanceof String s) { s.toUpperCase(); }` — eliminates explicit cast |
| `enum` | Defines a fixed set of named constants | Compiled to a final class extending `java.lang.Enum`. Each constant is a `public static final` instance |
| `record` | Immutable data class (Java 16 standard, preview since 14) | Compiler auto-generates constructor, getters, `equals`, `hashCode`, `toString`. All fields are implicitly `private final`. |
| `sealed` | Restricts which classes may extend/implement (Java 17+) | Pair with `permits`. Subclasses must be `final`, `sealed`, or `non-sealed`. Enables exhaustive pattern matching. |

### Method & Variable Keywords
| Keyword | Meaning | Low-level behavior |
|---------|---------|-------------------|
| `static` | Belongs to the class, not instances | Stored in **Method Area**, not on the heap per-object. One copy shared across all instances |
| `final` | Cannot be changed (variable=constant, method=no override, class=no subclass) | For variables: value baked in at compile time if primitive literal. For methods: JVM can **devirtualize** calls (optimization). For classes: `ACC_FINAL` flag |
| `void` | Method returns nothing | Return type in bytecode is `V`. JVM still executes a `return` instruction |
| `return` | Exits method, optionally passing a value back | JVM executes `ireturn`/`areturn`/`dreturn` etc depending on type. Pops the stack frame |
| `native` | Method implemented in C/C++ via JNI | JVM invokes code outside the JVM through the **Java Native Interface** |
| `synchronized` | Only one thread can execute this at a time | JVM uses **monitor/mutex** (each object has one). `monitorenter` and `monitorexit` bytecode instructions |
| `volatile` | Variable read/written directly to main memory | Disables CPU caching/register optimization. Inserts **memory barriers** (full fence). Guarantees **visibility** AND prevents **instruction reordering** around the access. Does NOT guarantee atomicity (e.g., `count++` is still unsafe). |
| `transient` | Field excluded from serialization | ObjectOutputStream skips fields marked `transient` |
| `strictfp` | Ensures consistent floating-point math across platforms | Forces IEEE 754 compliance; deprecated in Java 17 (all FP is IEEE 754 compliant by default now) |

### Flow Control Keywords
| Keyword | Meaning | Low-level behavior |
|---------|---------|-------------------|
| `if / else` | Conditional branching | Compiles to `ifeq`, `ifne`, `iflt` etc — conditional jump instructions |
| `switch` | Multi-way branch | Compiles to `tableswitch` (dense values, O(1)) or `lookupswitch` (sparse, O(log n)). Java 21 adds full **pattern matching in switch** (sealed class exhaustiveness checked at compile time). |
| `for` | Count-based loop | Compiles to: init → condition check (`ifeq` jump to end) → body → update → jump back |
| `while` | Condition-first loop | Same as for but without init/update |
| `do...while` | Body-first loop | Body executes unconditionally first, then condition check |
| `break` | Exit loop or switch | JVM `goto` instruction to after the loop |
| `continue` | Skip to next iteration | JVM `goto` to the loop update/condition check |
| `return` | Exit method | Pops current **stack frame** |

### Exception Keywords
| Keyword | Meaning | Low-level behavior |
|---------|---------|-------------------|
| `try` | Defines a block where exceptions might occur | JVM maintains an **Exception Table** in bytecode with try/catch ranges |
| `catch` | Handles a specific exception | JVM checks exception type against catch clauses top-to-bottom |
| `finally` | Always executes, exception or not | JVM duplicates `finally` code (or uses `jsr`/`ret` in older bytecode). One edge case: `finally` does NOT run if `System.exit()` is called. |
| `throw` | Manually throws an exception | JVM executes `athrow` instruction, unwinds call stack looking for handler |
| `throws` | Declares exceptions a method may throw | Compile-time contract only. No runtime enforcement for unchecked exceptions |

### Type Keywords
| Keyword | Meaning | Storage |
|---------|---------|---------|
| `byte` | 8-bit signed integer (-128 to 127) | Stack (local) / Heap (field) |
| `short` | 16-bit signed integer | Stack / Heap |
| `int` | 32-bit signed integer | Stack / Heap |
| `long` | 64-bit signed integer, suffix `L` | Stack / Heap (64-bit slot) |
| `float` | 32-bit IEEE 754 floating point, suffix `f` | Stack / Heap |
| `double` | 64-bit IEEE 754 floating point | Stack / Heap (64-bit) |
| `char` | 16-bit Unicode character (0 to 65535) | Stack / Heap |
| `boolean` | true/false (JVM uses `int` 0/1 internally) | Stack / Heap |

### Other Keywords
| Keyword | Meaning |
|---------|---------|
| `package` | Groups related classes (namespace) |
| `import` | Compile-time namespace shortcut |
| `assert` | Runtime condition check (disabled by default, enable with `-ea` JVM flag) |
| `null` | Literal meaning "no object reference". Dereferencing it → `NullPointerException` |
| `true / false` | Boolean literals |
| `var` | Local variable type inference (Java 10+). Compiler infers type at compile time — NOT dynamic typing. Cannot be used for fields, method parameters, or return types. |

---

## 🧮 Tokens in Java

Tokens are the **smallest meaningful units** the compiler reads.

| Token Type | Examples | Description |
|-----------|---------|-------------|
| **Keywords** | `int`, `class`, `public` | Reserved words with special meaning |
| **Identifiers** | `myVariable`, `HelloWorld` | Names you give to variables, methods, classes |
| **Literals** | `42`, `3.14`, `'A'`, `"hello"`, `true` | Fixed values written directly in code |
| **Operators** | `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `&&`, `\|\|`, `!` | Perform operations |
| **Separators** | `( ) { } [ ] ; , .` | Delimit code structure |
| **Comments** | `// ...`, `/* */`, `/** */` | Ignored by compiler, for humans |

---

## 📦 Data Types Deep Dive

### Primitives vs Reference Types

**Primitives** store the **actual value** directly in memory (stack for locals).
**Reference types** store a **pointer/address** to an object on the heap.

```java
int x = 5;          // x IS the value 5 — stored directly on stack
String s = "hello"; // s is a REFERENCE (pointer) to a String object on the heap
```

### The Stack & Heap

```
Stack (per-thread, LIFO):          Heap (shared, garbage collected):
┌──────────────────┐               ┌──────────────────────────────┐
│ main() frame     │               │  String object "hello"       │
│  x = 5           │               │  Object { name="Alice" }     │
│  s ──────────────┼──────────────►│  int[] { 1, 2, 3 }           │
└──────────────────┘               └──────────────────────────────┘
```

- Stack frames are created on method calls and destroyed on return.
- Heap objects are cleaned up by the **Garbage Collector** when no references remain.

### Type Casting

```java
// Widening (automatic, safe — no data loss)
int i = 100;
long l = i;       // int → long, JVM uses i2l instruction

// Narrowing (explicit, may lose data)
double d = 9.99;
int i = (int) d;  // → 9, truncates decimal, JVM uses d2i instruction

// Object casting
Object obj = "Hello";          // Upcasting (always safe)
String s = (String) obj;       // Downcasting (runtime check via checkcast bytecode)

// Pattern matching instanceof (Java 16+) — preferred
if (obj instanceof String s) { // test + bind in one step, no explicit cast
    System.out.println(s.toUpperCase());
}
```

---

## 🔁 Control Flow

```java
// if-else
if (x > 0) {
    System.out.println("positive");
} else if (x < 0) {
    System.out.println("negative");
} else {
    System.out.println("zero");
}

// Ternary operator
String result = (x > 0) ? "positive" : "non-positive";

// Switch (traditional)
switch (day) {
    case "MON": System.out.println("Monday"); break;
    default:    System.out.println("Other");
}

// Switch Expression (Java 14+ standard, preview 12/13)
String label = switch (day) {
    case "MON", "TUE" -> "Weekday";
    case "SAT", "SUN" -> "Weekend";
    default -> "Unknown";
};

// Pattern matching in switch (Java 21+)
Object obj = 42;
String desc = switch (obj) {
    case Integer i when i > 0 -> "positive int: " + i;
    case Integer i            -> "non-positive int";
    case String s             -> "string: " + s;
    default                   -> "other";
};

// for
for (int i = 0; i < 10; i++) { ... }

// enhanced for (for-each)
for (String s : list) { ... }   // Uses Iterator under the hood

// while
while (condition) { ... }

// do-while
do { ... } while (condition);
```

---

## 🗒️ Modern Java Features (8 → 21)

### Text Blocks (Java 15+)
```java
// Before
String json = "{\n  \"name\": \"Alice\"\n}";

// Text block — preserves formatting, strips common leading whitespace
String json = """
        {
          "name": "Alice"
        }
        """;
```

### Sealed Classes (Java 17+)
```java
// Only listed subclasses can extend Shape
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double w, double h) implements Shape {}

// Compiler enforces exhaustiveness in switch
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.w() * r.h();
    case Triangle t  -> /* ... */ 0;
    // no default needed — compiler knows all cases
};
```

### Records (Java 16+)
```java
// Equivalent to a class with private final fields, canonical constructor,
// getters (named after fields, not getBean), equals, hashCode, toString
public record Point(int x, int y) {
    // Compact canonical constructor for validation
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("Negative coords");
    }
}

Point p = new Point(3, 4);
p.x();   // getter — NOT getX()
```

---

## 🎯 OOP Concepts

### Classes & Objects

```java
public class Person {
    // Fields (instance variables — stored per-object on heap)
    private String name;
    private int age;

    // Constructor — called when `new Person()` is used
    public Person(String name, int age) {
        this.name = name;   // `this` disambiguates field vs parameter
        this.age = age;
    }

    // Instance method — implicitly receives `this`
    public String getName() { return name; }

    // Static method — no `this`, called on class directly
    public static String species() { return "Homo Sapiens"; }

    // toString override
    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }
}

Person p = new Person("Alice", 30);
// `new` → allocates Person object on heap → calls constructor → returns reference
```

### Inheritance

```java
public class Animal {
    public void speak() { System.out.println("..."); }
}

public class Dog extends Animal {
    @Override
    public void speak() { System.out.println("Woof!"); }
}

Animal a = new Dog();  // Polymorphism: reference type = Animal, actual object = Dog
a.speak();             // Calls Dog.speak() — dynamic dispatch via vtable lookup
```

**At the JVM level:** Each object has a reference to its **class metadata** which includes a **vtable** (virtual method table). When `a.speak()` is called, the JVM looks up `speak` in the **actual object's class** (Dog), not the reference type — this is **dynamic dispatch** via `invokevirtual`.

### Abstract Classes vs Interfaces

| | Abstract Class | Interface |
|-|---------------|-----------|
| Instantiation | No | No |
| Fields | Yes (any) | Only `public static final` |
| Methods | Abstract + concrete | Abstract + `default`/`static` (Java 8+) + `private` (Java 9+) |
| Extends/Implements | `extends` (one only) | `implements` (multiple) |
| Constructor | Yes | No |
| Use when | Sharing code + IS-A | Defining capability/contract |

```java
abstract class Shape {
    abstract double area();       // Must be overridden
    void describe() { System.out.println("I am a shape"); }  // Concrete
}

interface Drawable {
    void draw();                            // Abstract
    default void show() { draw(); }        // Default method (Java 8+)
    static Drawable noOp() { return () -> {}; }  // Static method
    private void helper() { /* shared logic */ }  // Private method (Java 9+)
}
```

### Encapsulation

Keep fields `private`, expose via `getters/setters`. This protects internal state and allows validation.

```java
private int age;
public void setAge(int age) {
    if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
    this.age = age;
}
```

### Polymorphism

Same interface, different behavior. Two forms:
- **Compile-time (method overloading):** Same method name, different parameters → resolved at compile time.
- **Runtime (method overriding):** Subclass overrides parent method → resolved at runtime via vtable.

```java
// Overloading (compile-time)
void print(int x) { ... }
void print(String s) { ... }

// Overriding (runtime)
@Override
public void speak() { ... }
```

---

## 🧵 Memory Model

```
JVM Memory Areas:
┌──────────────────────────────────────────────────────┐
│  Method Area / Metaspace (Java 8+ — off-heap)        │
│  Class metadata, static fields, constant pool        │
│  (PermGen was removed in Java 8, replaced by         │
│   Metaspace which grows dynamically in native memory)│
├──────────────────────────────────────────────────────┤
│  Heap                                                │
│  ┌──────────────────────────┬─────────────────────┐  │
│  │  Young Gen               │  Old Gen            │  │
│  │  Eden | S0 | S1          │  Long-lived objects │  │
│  └──────────────────────────┴─────────────────────┘  │
│  ⚠️ String Pool is also on the HEAP (since Java 7)   │
│     (was in PermGen before Java 7)                   │
├──────────────────────────────────────────────────────┤
│  Stack (per thread)                                  │
│  Each method call = one stack frame                  │
│  Frame contains: local variables, operand stack      │
├──────────────────────────────────────────────────────┤
│  PC Register (per thread): current instruction addr  │
│  Native Method Stack: for native (JNI) methods       │
└──────────────────────────────────────────────────────┘
```

---

## 🚨 Exception Handling

### Exception Hierarchy

```
Throwable
├── Error (JVM-level, don't catch these: OutOfMemoryError, StackOverflowError)
└── Exception
    ├── RuntimeException (Unchecked — compiler doesn't force you to handle)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   └── IllegalArgumentException
    └── IOException (Checked — must handle or declare `throws`)
        ├── FileNotFoundException
        └── ...
```

```java
try {
    int[] arr = new int[5];
    arr[10] = 1;                        // throws ArrayIndexOutOfBoundsException
} catch (ArrayIndexOutOfBoundsException e) {
    System.out.println("Caught: " + e.getMessage());
} catch (Exception e) {
    System.out.println("Generic catch");
} finally {
    System.out.println("Always runs");  // Runs even if exception thrown
                                        // Does NOT run if System.exit() is called
}

// Multi-catch (Java 7+)
catch (IOException | SQLException e) { ... }

// Custom Exception
public class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String msg) { super(msg); }
}

// try-with-resources (Java 7+) — auto-closes Closeable
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
}   // br.close() called automatically, even if exception occurs
    // Multiple resources: closed in reverse order of declaration
```

---

## 🗃️ Collections Framework

```
Iterable
└── Collection
    ├── List (ordered, duplicates allowed)
    │   ├── ArrayList  — backed by dynamic array, O(1) get, O(n) insert/delete
    │   └── LinkedList — doubly linked list, O(1) insert at head/tail, O(n) get
    │                    (also implements Deque — common interview gotcha)
    ├── Set (no duplicates)
    │   ├── HashSet    — backed by HashMap, O(1) ops, no order
    │   ├── LinkedHashSet — insertion order maintained
    │   └── TreeSet    — sorted (Red-Black tree), O(log n) ops
    └── Queue
        ├── PriorityQueue — min-heap by default
        └── Deque (ArrayDeque, LinkedList)

Map (key-value, not Collection)
├── HashMap    — O(1) avg, no order, allows null key/values
│               (Java 8+: bucket converts from linked list → Red-Black tree
│                when a bucket exceeds 8 entries — "treeification")
├── LinkedHashMap — insertion order (or access order with flag)
├── TreeMap    — sorted by key (Red-Black tree), O(log n)
├── ConcurrentHashMap — thread-safe, segments locks (preferred over Hashtable)
└── Hashtable  — legacy, fully synchronized, no null keys/values — avoid
```

### HashMap Internals (Important for Interviews)
```java
// HashMap = array of "buckets" (Node<K,V>[])
// hash(key) → index into array
// Collisions: stored as linked list at that bucket
// Java 8+: when bucket size > 8 AND total capacity > 64 → converts to Red-Black Tree
//           → worst case O(n) → O(log n)
// Load factor default: 0.75 (rehashes when 75% full, doubles capacity)

// Iteration order is NOT guaranteed (use LinkedHashMap if needed)
```

```java
List<String> list = new ArrayList<>();
list.add("a");
list.get(0);
list.remove("a");

Map<String, Integer> map = new HashMap<>();
map.put("key", 1);
map.get("key");
map.getOrDefault("missing", 0);
map.putIfAbsent("key", 2);      // only puts if key not present
map.computeIfAbsent("key", k -> k.length()); // compute and put

// Iterating Map
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}
```

---

## ⚡ Generics

```java
// Generic class
public class Box<T> {
    private T value;
    public Box(T value) { this.value = value; }
    public T get() { return value; }
}

Box<Integer> intBox = new Box<>(42);
Box<String> strBox = new Box<>("hello");

// Generic method
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}

// Wildcards — PECS: Producer Extends, Consumer Super
List<? extends Number> producer = List.of(1, 2.0);  // can READ as Number, cannot add (except null)
List<? super Integer>  consumer = new ArrayList<>(); // can ADD Integer/subtypes, READ as Object

// PECS mnemonic:
// If you only READ from a collection → use <? extends T>  (it "produces" T values for you)
// If you only WRITE to a collection  → use <? super T>    (it "consumes" T values you provide)
// If you both read and write          → use explicit <T>
```

**At compile time:** Generics are **erased** (type erasure). `Box<Integer>` becomes `Box` in bytecode. The compiler inserts casts automatically. This maintains backward compatibility with pre-generic Java. Consequence: you cannot do `new T[]` or `instanceof Box<Integer>` at runtime.

---

## 🔄 Java 8+ Features

### Lambda Expressions

```java
// Before Java 8
Runnable r = new Runnable() {
    @Override public void run() { System.out.println("run"); }
};

// Lambda (anonymous function)
Runnable r = () -> System.out.println("run");

// With parameters
Comparator<String> c = (a, b) -> a.compareTo(b);

// Method reference
list.forEach(System.out::println);  // same as x -> System.out.println(x)

// Four types of method references:
// Static:          ClassName::staticMethod    e.g., Integer::parseInt
// Instance (obj):  instance::method           e.g., str::toUpperCase
// Instance (type): ClassName::instanceMethod  e.g., String::toUpperCase
// Constructor:     ClassName::new             e.g., Person::new
```

### Functional Interfaces

```java
@FunctionalInterface  // exactly one abstract method
interface Transformer<T, R> { R transform(T input); }

// Built-in functional interfaces (java.util.function)
Function<String, Integer>   f = String::length;         // T → R
Predicate<String>           p = s -> s.isEmpty();       // T → boolean
Consumer<String>            c = System.out::println;    // T → void
Supplier<String>            s = () -> "hello";          // () → T
BiFunction<Integer,Integer,Integer> add = (a,b) -> a+b;
UnaryOperator<String>       uo = String::toUpperCase;   // T → T (specialization of Function)
BinaryOperator<Integer>     bo = Integer::sum;          // (T,T) → T
```

### Stream API

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6);

int sumOfEvenSquares = numbers.stream()
    .filter(n -> n % 2 == 0)      // intermediate — lazy
    .map(n -> n * n)               // intermediate — lazy
    .reduce(0, Integer::sum);      // terminal — triggers evaluation

// collect
List<String> names = people.stream()
    .map(Person::getName)
    .filter(n -> n.startsWith("A"))
    .sorted()
    .collect(Collectors.toList());

// Grouping
Map<String, List<Person>> byCity = people.stream()
    .collect(Collectors.groupingBy(Person::getCity));

// Parallel stream (uses ForkJoinPool, caution with ordered/stateful ops)
numbers.parallelStream().filter(...).collect(...);

// flatMap — one-to-many (flattens nested collections)
List<String> words = sentences.stream()
    .flatMap(s -> Arrays.stream(s.split(" ")))
    .collect(Collectors.toList());
```

Streams are **lazy** — intermediate operations build a pipeline but are not executed until a terminal operation is called. Streams cannot be reused after a terminal operation.

### Optional

```java
Optional<String> opt = Optional.ofNullable(maybeNull);
opt.isPresent();                        // check
opt.isEmpty();                          // Java 11+, inverse of isPresent
opt.get();                              // throws NoSuchElementException if empty
opt.orElse("default");
opt.orElseGet(() -> compute());
opt.orElseThrow(() -> new RuntimeException("missing"));
opt.map(String::toUpperCase).orElse(""); // transform safely
opt.filter(s -> s.length() > 3);        // returns empty if predicate fails
opt.ifPresent(System.out::println);     // runs only if present
```

---

## 🔒 Concurrency Basics

```java
// Thread creation
Thread t = new Thread(() -> System.out.println("Thread running"));
t.start();  // start() creates new OS thread and calls run() on it
            // do NOT call run() directly — that runs on current thread

// synchronized — mutual exclusion
public synchronized void increment() { count++; }
// or
synchronized (lockObject) { ... }

// volatile — visibility guarantee (but NOT atomicity)
private volatile boolean running = true;

// Atomic operations — for single-variable thread safety without locks
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();     // atomic read-modify-write
counter.compareAndSet(5, 10);  // CAS (Compare-And-Swap)

// Executor Service (preferred over raw threads)
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> doWork());
pool.shutdown();      // stops accepting new tasks, waits for running tasks
pool.shutdownNow();   // interrupts running tasks

// Future — async result
Future<Integer> future = pool.submit(() -> compute());
Integer result = future.get();          // blocks until done
future.get(5, TimeUnit.SECONDS);        // with timeout

// CompletableFuture (Java 8+) — non-blocking async composition
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchFromDB())      // runs on ForkJoinPool
    .thenApply(data -> transform(data))    // chain non-blocking
    .thenCompose(x -> callAnotherService(x)) // flatMap for async
    .exceptionally(ex -> "default");       // handle errors

// Waiting on multiple futures
CompletableFuture.allOf(cf1, cf2, cf3).join();  // wait for all
CompletableFuture.anyOf(cf1, cf2, cf3).join();  // wait for first

// ReentrantLock — more flexible than synchronized
ReentrantLock lock = new ReentrantLock();
lock.lock();
try { ... } finally { lock.unlock(); } // always unlock in finally

// CountDownLatch — wait for N events
CountDownLatch latch = new CountDownLatch(3);
latch.countDown();    // called by each thread when done
latch.await();        // main thread waits until count hits 0
```

---

## 📝 String Internals

```java
// ⚠️ CORRECTION: String Pool is on the HEAP (since Java 7), NOT in Metaspace.
// Before Java 7 it was in PermGen, which is now gone.

String s1 = "hello";          // stored in String Pool (part of the Heap)
String s2 = "hello";          // same reference as s1 — pool deduplicates
String s3 = new String("hello"); // forced new object on heap, bypasses pool

s1 == s2      // true  (same pool reference)
s1 == s3      // false (different objects)
s1.equals(s3) // true  (content comparison)

// Intern: explicitly put a heap string into the pool
String s4 = s3.intern(); // returns pooled reference
s1 == s4      // true

// Strings are IMMUTABLE — every modification creates a new object
String s = "hello";
s.concat(" world");   // s is still "hello" — result is discarded!
s = s.concat(" world"); // now s points to new object

// For mutable strings
StringBuilder sb = new StringBuilder("hello");  // not thread-safe, prefer this
sb.append(" world");  // modifies in place, returns `this` for chaining
sb.insert(0, "Hi ");
sb.delete(0, 3);
sb.reverse();
sb.toString();

StringBuffer sbf = new StringBuffer("hello"); // thread-safe (synchronized methods)
                                               // use only when truly sharing between threads
```

---

## 🗑️ Garbage Collection Deep Dive

### Generational GC
- **Young Generation** — new objects born here. Minor GC is frequent and fast.
  - **Eden**: all new objects start here
  - **Survivor 0/S1**: objects that survive one GC cycle are copied here
  - After N cycles (threshold), objects promoted to Old Gen
- **Old Generation** — long-lived objects. Major/Full GC is infrequent but slow (Stop-The-World).
- **Metaspace** — class metadata. Not garbage collected in the traditional sense; grows/shrinks dynamically.

### Modern GC Collectors

| Collector | Flag | Best For | Pause |
|-----------|------|----------|-------|
| Serial | `-XX:+UseSerialGC` | Single-core, small heaps | STW |
| Parallel (Throughput) | `-XX:+UseParallelGC` | Batch workloads, max throughput | STW |
| G1GC (default Java 9+) | `-XX:+UseG1GC` | Balanced latency/throughput, large heaps | Short STW |
| ZGC (Java 15+ production) | `-XX:+UseZGC` | Ultra-low latency (<1ms pause), huge heaps | Near-zero |
| Shenandoah | `-XX:+UseShenandoahGC` | Low latency (OpenJDK, not Oracle JDK) | Near-zero |

**Key point for interviews:** You cannot force GC (`System.gc()` is a hint, not a command). You cannot deterministically control *when* it runs. Design your code to minimize allocation pressure (object pooling, avoid large temp objects in loops).

---

## 🗂️ Common Java Patterns

```java
// Singleton — double-checked locking (thread-safe, performant)
public class Config {
    // volatile prevents reordering of constructor + reference assignment
    private static volatile Config instance;
    private Config() {}
    public static Config getInstance() {
        if (instance == null) {               // First check (no lock)
            synchronized (Config.class) {
                if (instance == null) {       // Second check (under lock)
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}

// Even better: Initialization-on-demand holder (lazy, thread-safe, no volatile)
public class Config {
    private Config() {}
    private static class Holder {
        static final Config INSTANCE = new Config();
    }
    public static Config getInstance() { return Holder.INSTANCE; }
}

// Builder
Person person = new Person.Builder()
    .name("Alice")
    .age(30)
    .build();

// Factory
Shape shape = ShapeFactory.create("circle");

// Strategy
interface SortStrategy { void sort(int[] arr); }
class QuickSort implements SortStrategy { ... }
class MergeSort implements SortStrategy { ... }

// Observer (also see java.util.Observer — deprecated; use custom event bus or reactive streams)
interface EventListener { void onEvent(Event e); }
```

---

---

# 🎤 Common Java Interview Questions & Answers

---

### Q1. What is the difference between `==` and `.equals()`?

`==` compares **references** (memory addresses) for objects, or actual values for primitives. `.equals()` compares **content/logical equality**. For Strings, `"abc" == "abc"` may be true due to string pooling, but you should never rely on this. Always use `.equals()` for object comparison.

```java
String a = new String("hello");
String b = new String("hello");
a == b        // false — different heap objects
a.equals(b)   // true  — same content
```

---

### Q2. What is the difference between `HashMap` and `Hashtable`?

`HashMap` is not synchronized (not thread-safe), allows one `null` key and multiple `null` values, and is generally faster. `Hashtable` is synchronized (thread-safe), does not allow `null` keys or values, and is considered legacy. For thread-safe maps today, prefer `ConcurrentHashMap` over `Hashtable`. `ConcurrentHashMap` uses **segment-level locking** (Java 7) or **CAS + synchronized per-bucket** (Java 8+) rather than locking the entire map, giving far better throughput.

---

### Q3. What is the difference between `ArrayList` and `LinkedList`?

`ArrayList` is backed by a dynamic array. Random access (`get(i)`) is O(1), but insertion/deletion in the middle is O(n) because elements must be shifted. `LinkedList` is a doubly-linked list. Insertion/deletion at head or tail is O(1), but random access is O(n) because it must traverse from one end. Use `ArrayList` for most cases; `LinkedList` only when you frequently insert/remove at the ends and don't need random access. In practice, `ArrayDeque` is a better `Queue`/`Deque` than `LinkedList` due to better cache locality.

---

### Q4. What is the JVM, JRE, and JDK?

JVM (Java Virtual Machine) is the runtime engine that executes bytecode. JRE (Java Runtime Environment) includes the JVM plus the standard class libraries — it's what end users need to run Java programs. JDK (Java Development Kit) includes the JRE plus development tools like the compiler (`javac`), debugger, and documentation tools — it's what developers need to write and compile Java code. Since Java 11, Oracle stopped distributing a standalone JRE; the JDK is the standard distribution.

---

### Q5. What are checked vs unchecked exceptions?

**Checked exceptions** are subclasses of `Exception` (but not `RuntimeException`). The compiler forces you to either catch them or declare them with `throws` (e.g., `IOException`, `SQLException`). **Unchecked exceptions** are subclasses of `RuntimeException` — the compiler doesn't force handling (e.g., `NullPointerException`, `ArrayIndexOutOfBoundsException`). `Error` (like `OutOfMemoryError`) is also unchecked and represents unrecoverable JVM-level problems. The debate: checked exceptions enforce handling at compile time but lead to verbose catch blocks and leaky abstractions; many modern frameworks (Spring, Kotlin) prefer unchecked exceptions.

---

### Q6. What is the difference between `String`, `StringBuilder`, and `StringBuffer`?

`String` is **immutable** — every operation creates a new String object. `StringBuilder` is **mutable** and not thread-safe — best for single-threaded string manipulation (most common case). `StringBuffer` is **mutable and thread-safe** (synchronized methods) — use only when multiple threads share the same buffer. In practice: use `StringBuilder` in loops, `String` for constants, `StringBuffer` almost never. Note: the compiler already optimizes `String` concatenation with `+` inside a single expression, but repeated `+` in loops creates many intermediate objects — use `StringBuilder` there.

---

### Q7. What is method overloading vs method overriding?

**Overloading** is having multiple methods with the same name but different parameter lists in the same class. It's resolved at **compile time** (static polymorphism). **Overriding** is a subclass providing a specific implementation of a method already defined in its parent class with the same signature. It's resolved at **runtime** via dynamic dispatch (runtime polymorphism). Overriding requires `@Override` annotation (best practice) and cannot have a more restrictive access modifier. A tricky distinction: overloading is determined by the **declared type** of the reference; overriding is determined by the **actual runtime type** of the object.

---

### Q8. What is the `static` keyword?

`static` means the member **belongs to the class**, not to any instance. Static fields are shared across all instances — only one copy exists in the Method Area. Static methods can be called without creating an object. Static members cannot access instance members directly because there's no `this` reference. Static blocks run once when the class is first loaded by the ClassLoader. Common pitfall: `static` mutable fields are a shared global state and need synchronization in multi-threaded contexts.

---

### Q9. What is garbage collection in Java?

Garbage collection (GC) is the JVM's automatic memory management process. When an object has no more references pointing to it (reachable from GC roots: stack vars, static fields, JNI refs), it becomes eligible for GC. The GC runs in the background and reclaims that memory. Java uses **generational GC**: most objects die young, so the heap is split into Young Generation (Eden + Survivor spaces) and Old Generation. Minor GC cleans Young Gen frequently; Major/Full GC cleans Old Gen less frequently. Modern collectors like G1GC, ZGC, and Shenandoah reduce Stop-The-World pauses significantly. You cannot force GC (`System.gc()` is just a hint), and you cannot control when it runs.

---

### Q10. What is the difference between `abstract class` and `interface`?

An abstract class can have state (fields), constructors, abstract and concrete methods. A class can only extend **one** abstract class. An interface traditionally defines a contract (abstract methods only), but since Java 8 can also have `default` and `static` methods, and since Java 9 can have `private` methods. A class can implement **multiple** interfaces. Use an abstract class when you want to share code among closely related classes. Use an interface to define a capability that unrelated classes can share. With records and sealed classes, interfaces now carry even more design weight in modern Java.

---

### Q11. What is `final`, `finally`, and `finalize`?

These are three completely different things that just sound alike:
- `final` is a **keyword**: makes a variable constant, a method non-overridable, or a class non-subclassable.
- `finally` is a **block** in try-catch: always executes after try/catch, used for cleanup. Edge case: does NOT run if `System.exit()` is called or the JVM crashes.
- `finalize` is a **method** in `Object`: called by the GC before an object is collected. It is **deprecated since Java 9** and **removed in Java 18**. Use `Cleaner` (Java 9+) or try-with-resources instead.

---

### Q12. What is autoboxing and unboxing?

Autoboxing is the automatic conversion of a primitive to its wrapper class (`int` → `Integer`). Unboxing is the reverse. The compiler inserts these conversions automatically.

```java
List<Integer> list = new ArrayList<>();
list.add(5);          // autoboxing: 5 → Integer.valueOf(5)
int x = list.get(0);  // unboxing: Integer → int
```

Beware of:
1. `NullPointerException` when unboxing a `null` Integer.
2. Performance issues in tight loops due to object creation.
3. **Integer cache gotcha:** `Integer.valueOf()` caches instances for values **-128 to 127**. This means `Integer a = 127; Integer b = 127; a == b` is `true`, but `Integer a = 128; Integer b = 128; a == b` is `false`. Always use `.equals()` for Integer comparison.

```java
Integer a = 127, b = 127;
a == b   // true  — same cached object
Integer c = 128, d = 128;
c == d   // false — different heap objects
```

---

### Q13. What are the four pillars of OOP?

**Encapsulation** — bundling data and methods together, hiding internal state (private fields + public methods). **Abstraction** — exposing only essential details, hiding complexity (abstract classes, interfaces). **Inheritance** — a class acquires properties of another class (`extends`), promoting reuse. **Polymorphism** — the same interface behaves differently depending on the actual object type (overriding at runtime, overloading at compile time). Java also supports a fifth principle often added: **Composition over Inheritance** — prefer building complex behavior by composing objects rather than deep inheritance chains.

---

### Q14. What is a `volatile` variable?

`volatile` ensures that reads and writes to a variable go directly to **main memory**, bypassing CPU caches. It provides two guarantees:
1. **Visibility** — changes by one thread are immediately visible to all other threads.
2. **Ordering** — prevents instruction reordering around the volatile access (acts as a memory barrier).

`volatile` does NOT guarantee **atomicity** — `count++` is still not thread-safe even if `count` is volatile because `++` is three operations (read, increment, write). Use `AtomicInteger` for atomic operations, or `synchronized` for compound operations. Common valid use: a `volatile boolean` flag to signal threads to stop.

---

### Q15. What is the Java Memory Model (JMM)?

The JMM defines the rules for how threads interact through memory. It specifies what values a thread is guaranteed to see when reading a variable written by another thread. Key concepts:
- **happens-before** relationship: if action A happens-before B, then B sees A's changes. Established by: `synchronized` exit/entry, `volatile` write/read, `Thread.start()`, `Thread.join()`, `final` field initialization.
- **Visibility**: without synchronization, threads may see stale data due to CPU caching/register optimization.
- **Ordering**: compiler and CPU may reorder instructions for optimization — the JMM defines what reorderings are allowed, ensuring sequential consistency *within* a thread and happens-before semantics *across* threads.
- **Data race**: when two threads access the same variable concurrently without synchronization, and at least one write, the result is **undefined behavior** per the JMM.

---

### Q16. Explain HashMap's internal working.

`HashMap` maintains an array of `Node<K,V>[]` (buckets). On `put(k, v)`:
1. Compute `hash(k)` — spreads bits to reduce collisions.
2. Index = `(n-1) & hash` where `n` is array length (power of 2, enables fast modulo via bitwise AND).
3. If bucket empty → insert. If collision → compare key via `equals()`, update if same key, else append to chain.
4. **Java 8+:** When a bucket's chain exceeds 8 entries AND total table size ≥ 64 → convert chain to a **Red-Black Tree** (O(n) worst-case → O(log n)).
5. When load factor (entries/capacity) > 0.75 → **rehash**: double the array, redistribute all entries.

Key interview points: hash collisions are handled via chaining; `equals` and `hashCode` must be consistent (if `a.equals(b)` then `a.hashCode() == b.hashCode()`); breaking this contract causes keys to be "lost" in the map.

---

### Bonus: What happens when you run `java MyClass`?

1. JVM starts up.
2. **Bootstrap ClassLoader** loads core Java classes (`java.lang.*` etc.).
3. **Application ClassLoader** finds and loads `MyClass.class`.
4. The class is **linked**: verified (bytecode valid? type-safe?), prepared (static fields get default values: 0, null, false), resolved (symbolic references → direct memory references).
5. **Static initializer blocks** run (in order of declaration).
6. JVM locates `public static void main(String[] args)` and calls it.
7. Execution begins. The **JIT compiler** profiles hot code paths (C1 → C2 tiered compilation) and compiles bytecode to native machine code on the fly for performance.
8. When `main` returns, JVM runs **shutdown hooks** (registered via `Runtime.addShutdownHook()`), finalizes objects, and exits.

---

*Master these concepts and you'll handle both fundamentals and deep JVM internals questions with confidence.*