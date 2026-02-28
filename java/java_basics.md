# ☕ Java — Complete Cheatsheet & Notes

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
| `interface` | Defines a contract (abstract method signatures) | Stored in Method Area; marked `ACC_INTERFACE`. No instance creation. Methods are abstract by default |
| `abstract` | Class cannot be instantiated; method has no body | JVM sets `ACC_ABSTRACT`. Attempting `new AbstractClass()` → compile error |
| `extends` | Inherits from a superclass | JVM links the child's class structure to the parent's. Method calls walk up the **vtable** chain |
| `implements` | Class agrees to fulfill an interface contract | JVM creates an **itable** (interface method table) for dispatch |
| `new` | Creates a new object on the heap | JVM executes `new` bytecode instruction → allocates memory on **Heap**, runs constructor via `invokespecial` |
| `this` | Reference to the current object | At runtime, `this` is the first implicit argument passed to every instance method. Points to the object's address on the heap |
| `super` | Reference to the parent class | Allows calling parent's constructor/methods via `invokespecial` (bypasses virtual dispatch) |
| `instanceof` | Checks if object is an instance of a type | JVM checks the object's class metadata chain at runtime |
| `enum` | Defines a fixed set of named constants | Compiled to a final class extending `java.lang.Enum`. Each constant is a `public static final` instance |
| `record` | Immutable data class (Java 16+) | Compiler auto-generates constructor, getters, `equals`, `hashCode`, `toString` |

### Method & Variable Keywords
| Keyword | Meaning | Low-level behavior |
|---------|---------|-------------------|
| `static` | Belongs to the class, not instances | Stored in **Method Area**, not on the heap per-object. One copy shared across all instances |
| `final` | Cannot be changed (variable=constant, method=no override, class=no subclass) | For variables: value baked in at compile time if primitive literal. For methods: JVM can **devirtualize** calls (optimization). For classes: `ACC_FINAL` flag |
| `void` | Method returns nothing | Return type in bytecode is `V`. JVM still executes a `return` instruction |
| `return` | Exits method, optionally passing a value back | JVM executes `ireturn`/`areturn`/`dreturn` etc depending on type. Pops the stack frame |
| `native` | Method implemented in C/C++ via JNI | JVM invokes code outside the JVM through the **Java Native Interface** |
| `synchronized` | Only one thread can execute this at a time | JVM uses **monitor/mutex** (each object has one). `monitorenter` and `monitorexit` bytecode instructions |
| `volatile` | Variable read/written directly to main memory | Disables CPU caching/register optimization. Inserts **memory barriers**. Ensures visibility across threads |
| `transient` | Field excluded from serialization | ObjectOutputStream skips fields marked `transient` |
| `strictfp` | Ensures consistent floating-point math across platforms | Forces IEEE 754 compliance; deprecated in Java 17 |

### Flow Control Keywords
| Keyword | Meaning | Low-level behavior |
|---------|---------|-------------------|
| `if / else` | Conditional branching | Compiles to `ifeq`, `ifne`, `iflt` etc — conditional jump instructions |
| `switch` | Multi-way branch | Compiles to `tableswitch` (dense values, O(1)) or `lookupswitch` (sparse, O(log n)) |
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
| `finally` | Always executes, exception or not | JVM duplicates `finally` code (or uses `jsr`/`ret` in older bytecode) |
| `throw` | Manually throws an exception | JVM executes `athrow` instruction, unwinds call stack looking for handler |
| `throws` | Declares exceptions a method may throw | Compile-time contract. No runtime enforcement for unchecked exceptions |

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
| `var` | Local variable type inference (Java 10+). Compiler infers type at compile time — NOT dynamic typing |

---

## 🧮 Tokens in Java

Tokens are the **smallest meaningful units** the compiler reads.

| Token Type | Examples | Description |
|-----------|---------|-------------|
| **Keywords** | `int`, `class`, `public` | Reserved words with special meaning |
| **Identifiers** | `myVariable`, `HelloWorld` | Names you give to variables, methods, classes |
| **Literals** | `42`, `3.14`, `'A'`, `"hello"`, `true` | Fixed values written directly in code |
| **Operators** | `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `&&`, `||`, `!` | Perform operations |
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

// Switch Expression (Java 14+)
String label = switch (day) {
    case "MON", "TUE" -> "Weekday";
    case "SAT", "SUN" -> "Weekend";
    default -> "Unknown";
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
| Methods | Abstract + concrete | Abstract (default/static concrete Java 8+) |
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
┌─────────────────────────────────────────────────────┐
│  Method Area (Metaspace in Java 8+)                 │
│  Class metadata, static fields, constant pool       │
├─────────────────────────────────────────────────────┤
│  Heap                                               │
│  ┌──────────────────────────┬─────────────────────┐ │
│  │  Young Gen               │  Old Gen            │ │
│  │  Eden | S0 | S1          │  Long-lived objects │ │
│  └──────────────────────────┴─────────────────────┘ │
├─────────────────────────────────────────────────────┤
│  Stack (per thread)                                 │
│  Each method call = one stack frame                 │
│  Frame contains: local variables, operand stack     │
├─────────────────────────────────────────────────────┤
│  PC Register (per thread): current instruction addr │
│  Native Method Stack: for native (JNI) methods      │
└─────────────────────────────────────────────────────┘
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
}

// Custom Exception
public class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String msg) { super(msg); }
}

// try-with-resources (Java 7+) — auto-closes Closeable
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
}   // br.close() called automatically
```

---

## 🗃️ Collections Framework

```
Iterable
└── Collection
    ├── List (ordered, duplicates allowed)
    │   ├── ArrayList  — backed by dynamic array, O(1) get, O(n) insert
    │   └── LinkedList — doubly linked list, O(1) insert at head/tail, O(n) get
    ├── Set (no duplicates)
    │   ├── HashSet    — backed by HashMap, O(1) ops, no order
    │   ├── LinkedHashSet — insertion order maintained
    │   └── TreeSet    — sorted (Red-Black tree), O(log n) ops
    └── Queue
        ├── PriorityQueue — min-heap by default
        └── Deque (ArrayDeque, LinkedList)

Map (key-value, not Collection)
├── HashMap    — O(1) avg, no order, allows null key
├── LinkedHashMap — insertion order
├── TreeMap    — sorted by key (Red-Black tree)
└── Hashtable  — legacy, synchronized, no null keys
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

// Wildcards
List<? extends Number> readOnly   // can read as Number, can't add
List<? super Integer> writeOnly   // can add Integer, read as Object
```

**At compile time:** Generics are **erased** (type erasure). `Box<Integer>` becomes `Box` in bytecode. The compiler inserts casts automatically. This maintains backward compatibility with pre-generic Java.

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
```

### Functional Interfaces

```java
@FunctionalInterface  // exactly one abstract method
interface Transformer<T, R> { R transform(T input); }

// Built-in functional interfaces (java.util.function)
Function<String, Integer>   f = String::length;     // T → R
Predicate<String>           p = s -> s.isEmpty();   // T → boolean
Consumer<String>            c = System.out::println; // T → void
Supplier<String>            s = () -> "hello";       // () → T
BiFunction<Integer,Integer,Integer> add = (a,b) -> a+b;
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
```

Streams are **lazy** — intermediate operations are not executed until a terminal operation is called.

### Optional

```java
Optional<String> opt = Optional.ofNullable(maybeNull);
opt.isPresent();                        // check
opt.get();                              // throws if empty
opt.orElse("default");
opt.orElseGet(() -> compute());
opt.map(String::toUpperCase).orElse(""); // transform safely
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

// volatile — visibility guarantee
private volatile boolean running = true;

// Executor Service (preferred over raw threads)
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> doWork());
pool.shutdown();

// Future — async result
Future<Integer> future = pool.submit(() -> compute());
Integer result = future.get();  // blocks until done
```

---

## 📝 String Internals

```java
String s1 = "hello";          // stored in String Pool (Metaspace)
String s2 = "hello";          // same reference as s1
String s3 = new String("hello"); // new object on heap

s1 == s2      // true  (same pool reference)
s1 == s3      // false (different objects)
s1.equals(s3) // true  (content comparison)

// Strings are IMMUTABLE — every modification creates a new object
String s = "hello";
s.concat(" world");   // s is still "hello"!
s = s.concat(" world"); // now s points to new object

// For mutable strings
StringBuilder sb = new StringBuilder("hello");  // not thread-safe
sb.append(" world");  // modifies in place

StringBuffer sbf = new StringBuffer("hello"); // thread-safe (synchronized)
```

---

## 🗂️ Common Java Patterns

```java
// Singleton
public class Config {
    private static Config instance;
    private Config() {}
    public static synchronized Config getInstance() {
        if (instance == null) instance = new Config();
        return instance;
    }
}

// Builder
Person person = new Person.Builder()
    .name("Alice")
    .age(30)
    .build();

// Factory
Shape shape = ShapeFactory.create("circle");
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

`HashMap` is not synchronized (not thread-safe), allows one `null` key and multiple `null` values, and is generally faster. `Hashtable` is synchronized (thread-safe), does not allow `null` keys or values, and is considered legacy. For thread-safe maps today, prefer `ConcurrentHashMap` over `Hashtable`.

---

### Q3. What is the difference between `ArrayList` and `LinkedList`?

`ArrayList` is backed by a dynamic array. Random access (`get(i)`) is O(1), but insertion/deletion in the middle is O(n) because elements must be shifted. `LinkedList` is a doubly-linked list. Insertion/deletion at head or tail is O(1), but random access is O(n) because it must traverse from one end. Use `ArrayList` for most cases; `LinkedList` only when you frequently insert/remove at the ends.

---

### Q4. What is the JVM, JRE, and JDK?

JVM (Java Virtual Machine) is the runtime engine that executes bytecode. JRE (Java Runtime Environment) includes the JVM plus the standard class libraries — it's what end users need to run Java programs. JDK (Java Development Kit) includes the JRE plus development tools like the compiler (`javac`), debugger, and documentation tools — it's what developers need to write and compile Java code.

---

### Q5. What are checked vs unchecked exceptions?

**Checked exceptions** are subclasses of `Exception` (but not `RuntimeException`). The compiler forces you to either catch them or declare them with `throws` (e.g., `IOException`, `SQLException`). **Unchecked exceptions** are subclasses of `RuntimeException` — the compiler doesn't force handling (e.g., `NullPointerException`, `ArrayIndexOutOfBoundsException`). `Error` (like `OutOfMemoryError`) is also unchecked and represents unrecoverable JVM-level problems.

---

### Q6. What is the difference between `String`, `StringBuilder`, and `StringBuffer`?

`String` is **immutable** — every operation creates a new String object. `StringBuilder` is **mutable** and not thread-safe — best for single-threaded string manipulation (most common case). `StringBuffer` is **mutable and thread-safe** (synchronized methods) — use only when multiple threads share the same buffer. In practice: use `StringBuilder` in loops, `String` for constants, `StringBuffer` almost never.

---

### Q7. What is method overloading vs method overriding?

**Overloading** is having multiple methods with the same name but different parameter lists in the same class. It's resolved at **compile time** (static polymorphism). **Overriding** is a subclass providing a specific implementation of a method already defined in its parent class with the same signature. It's resolved at **runtime** via dynamic dispatch (runtime polymorphism). Overriding requires `@Override` annotation (best practice) and cannot have a more restrictive access modifier.

---

### Q8. What is the `static` keyword?

`static` means the member **belongs to the class**, not to any instance. Static fields are shared across all instances — only one copy exists in the Method Area. Static methods can be called without creating an object. Static members cannot access instance members directly because there's no `this` reference. Static blocks run once when the class is first loaded by the ClassLoader.

---

### Q9. What is garbage collection in Java?

Garbage collection (GC) is the JVM's automatic memory management process. When an object has no more references pointing to it, it becomes eligible for GC. The GC runs in the background and reclaims that memory. Java uses **generational GC**: most objects die young, so the heap is split into Young Generation (Eden + Survivor spaces) and Old Generation. Minor GC cleans Young Gen frequently; Major/Full GC cleans Old Gen less frequently. You cannot force GC (`System.gc()` is just a hint), and you cannot control when it runs.

---

### Q10. What is the difference between `abstract class` and `interface`?

An abstract class can have state (fields), constructors, abstract and concrete methods. A class can only extend **one** abstract class. An interface traditionally defines a contract (abstract methods only), but since Java 8 can also have `default` and `static` methods, and since Java 9 can have `private` methods. A class can implement **multiple** interfaces. Use an abstract class when you want to share code among closely related classes. Use an interface to define a capability that unrelated classes can share.

---

### Q11. What is `final`, `finally`, and `finalize`?

These are three completely different things that just sound alike:
- `final` is a **keyword**: makes a variable constant, a method non-overridable, or a class non-subclassable.
- `finally` is a **block** in try-catch: always executes after try/catch, used for cleanup.
- `finalize` is a **method** in `Object`: called by the GC before an object is collected. It is deprecated since Java 9 and should not be used.

---

### Q12. What is autoboxing and unboxing?

Autoboxing is the automatic conversion of a primitive to its wrapper class (`int` → `Integer`). Unboxing is the reverse. The compiler inserts these conversions automatically.

```java
List<Integer> list = new ArrayList<>();
list.add(5);          // autoboxing: 5 → Integer.valueOf(5)
int x = list.get(0);  // unboxing: Integer → int
```

Beware of `NullPointerException` when unboxing a `null` Integer, and performance issues in tight loops due to object creation.

---

### Q13. What are the four pillars of OOP?

**Encapsulation** — bundling data and methods together, hiding internal state (private fields + public methods). **Abstraction** — exposing only essential details, hiding complexity (abstract classes, interfaces). **Inheritance** — a class acquires properties of another class (`extends`), promoting reuse. **Polymorphism** — the same interface behaves differently depending on the actual object type (overriding at runtime, overloading at compile time).

---

### Q14. What is a `volatile` variable?

`volatile` ensures that reads and writes to a variable go directly to **main memory**, bypassing CPU caches. Without it, threads may see stale cached values. `volatile` guarantees **visibility** but not **atomicity** — so `count++` is still not thread-safe even if `count` is volatile because `++` is three operations (read, increment, write). Use `AtomicInteger` for atomic operations, or `synchronized` for compound operations.

---

### Q15. What is the Java Memory Model (JMM)?

The JMM defines the rules for how threads interact through memory. It specifies what values a thread is guaranteed to see when reading a variable written by another thread. Key concepts: **happens-before** relationship (if action A happens-before B, then B sees A's changes), **visibility** (without synchronization, threads may see stale data), and **ordering** (compiler/CPU may reorder instructions for optimization — JMM defines when this is allowed). `synchronized`, `volatile`, `final` fields, and `Thread.start()/join()` all establish happens-before relationships.

---

### Bonus: What happens when you run `java MyClass`?

1. JVM starts up.
2. **Bootstrap ClassLoader** loads core Java classes.
3. **Application ClassLoader** finds and loads `MyClass.class`.
4. The class is **linked**: verified (bytecode valid?), prepared (static fields get defaults), resolved (symbolic references → direct references).
5. **Static initializer blocks** run.
6. JVM locates `public static void main(String[] args)` and calls it.
7. Execution begins. The JIT compiler profiles hot code paths and compiles bytecode to native machine code on the fly for performance.
8. When `main` returns, JVM runs **shutdown hooks**, finalizes objects, and exits.

---

*That's a comprehensive Java reference. Master these concepts and you'll handle both fundamentals and deep JVM internals questions with confidence.*