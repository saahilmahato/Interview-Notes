# Java — General Knowledge & Language Properties
### Senior/Elite Engineer Interview Questions & Model Answers

---

## 🔷 SECTION 1: JVM Architecture & Internals

---

**Q1. Explain the JVM architecture in depth. What are its core components and how do they interact at runtime?**

**Model Answer:**
The JVM is a stack-based abstract computing machine composed of several interconnected subsystems.

The **Class Loader Subsystem** is responsible for loading, linking, and initializing classes. Loading finds and reads the `.class` bytecode. Linking involves verification (ensuring bytecode is structurally correct), preparation (allocating memory for static fields and setting default values), and resolution (converting symbolic references into direct references). Initialization then runs static initializers and assigns static field values.

The **Runtime Data Areas** include:
- **Method Area (Metaspace in Java 8+):** Stores class-level data — field descriptors, method bytecodes, the runtime constant pool, and static variables. In Java 8, PermGen was replaced by Metaspace, which grows dynamically in native memory.
- **Heap:** Where all object instances and arrays live. Divided into Young Generation (Eden + two Survivor spaces) and Old Generation (Tenured). GC operates here.
- **Java Stack:** Each thread has its own stack. Each method invocation creates a **stack frame** containing: local variable array, operand stack, and a reference to the runtime constant pool. This is where primitive values and object references live.
- **PC Register:** Each thread has a program counter tracking the address of the currently executing JVM instruction.
- **Native Method Stack:** Supports native (C/C++) method execution via JNI.

The **Execution Engine** consumes bytecode. It contains the **Interpreter** (executes bytecode line by line), the **JIT Compiler** (identifies "hot spots" via profiling and compiles them to native machine code for performance), and the **Garbage Collector**.

The **JNI (Java Native Interface)** bridges Java code with native libraries.

What makes the JVM remarkable is that it provides a consistent execution model across hardware and OS boundaries — write once, run anywhere — while still achieving near-native performance through JIT compilation and adaptive optimization.

---

**Q2. What is the difference between JDK, JRE, and JVM? Where does each boundary sit?**

**Model Answer:**
These are nested abstractions.

**JVM (Java Virtual Machine)** is the runtime engine. It takes compiled bytecode (`.class` files) and executes it. It has no knowledge of the Java language itself — it only understands bytecode. You could compile Kotlin or Scala to bytecode and the JVM would run it identically.

**JRE (Java Runtime Environment)** = JVM + the standard class libraries (java.lang, java.util, java.io, etc.) + supporting files. It's the minimum you need to *run* a Java application. End users traditionally installed only the JRE.

**JDK (Java Development Kit)** = JRE + development tools: the compiler (`javac`), `javadoc`, `jar`, `jshell`, debugger, `jmap`, `jstack`, `jconsole`, etc. Developers need the JDK to *write and compile* Java code.

Since Java 9, with the introduction of the module system (Jigsaw), the distinction between JDK and JRE became less rigid. The JRE as a standalone distribution was deprecated and removed. You now use `jlink` to create custom runtime images containing only the modules your application needs, effectively replacing the old JRE concept with tailored, minimal runtimes.

---

**Q3. What is JIT compilation? How does tiered compilation work in HotSpot?**

**Model Answer:**
JIT (Just-In-Time) compilation is the process of translating JVM bytecode into native machine code at runtime, rather than interpreting it instruction by instruction. This trades startup cost for long-term execution speed.

HotSpot uses **Tiered Compilation** (default since Java 7, standard in Java 8+), which has five levels:

- **Level 0 — Interpreter:** All methods start here. Slow but immediate.
- **Level 1 — C1 (Client Compiler), no profiling:** Fast, simple compilation. Used for methods that will likely not be called often.
- **Level 2 — C1, limited profiling:** Compiles with basic invocation/backedge counters.
- **Level 3 — C1, full profiling:** Collects type profiles, branch profiles. Most commonly used before promotion to C2.
- **Level 4 — C2 (Server Compiler):** Aggressive optimizing compiler. Uses the profiling data from Level 3 to perform inlining, escape analysis, loop unrolling, lock elision, dead code elimination, etc.

The JVM promotes methods through these tiers based on how "hot" they are (measured by invocation count and loop back-edge count). C2-compiled code can approach and sometimes exceed equivalent C++ code in performance because it can make speculative optimizations based on observed runtime behavior — something a static compiler cannot do.

If a speculative optimization becomes invalid (e.g., a new subclass is loaded that breaks a monomorphic inline cache), the JVM performs **deoptimization** — it discards the compiled code and falls back to interpretation, possibly re-profiling for a better compilation.

---

**Q4. Explain how the Garbage Collector works. What are the main GC algorithms available in modern Java and their trade-offs?**

**Model Answer:**
Garbage Collection is the automatic reclamation of heap memory occupied by objects that are no longer reachable from any GC root (thread stacks, static fields, JNI references, etc.).

The foundational algorithm is **mark-and-sweep**: mark all live objects, then sweep (reclaim) the rest. Modern GCs add **compaction** to avoid fragmentation, and **generational collection** based on the weak generational hypothesis — most objects die young.

**The main modern GC algorithms:**

**Serial GC:** Single-threaded stop-the-world. Small heaps, single-core. Used in some embedded contexts.

**Parallel GC (Throughput Collector):** Multiple threads for young and old generation collection. Maximizes throughput. Stop-the-world pauses. Historically the default before Java 9.

**G1 (Garbage First):** Default since Java 9. Divides heap into equal-sized **regions** rather than contiguous young/old areas. Builds a "collection set" of regions with the most garbage. Designed for predictable pause times (soft real-time). Can run concurrently for marking. Much better pause control than Parallel GC, with acceptable throughput.

**ZGC:** Introduced in Java 11 (production-ready Java 15). Mostly concurrent — almost all work happens while application threads run. Stop-the-world pauses measured in single-digit milliseconds regardless of heap size (tested up to terabytes). Uses **load barriers** and **colored pointers** for concurrent compaction. Excellent for latency-sensitive applications.

**Shenandoah:** RedHat-developed, available in OpenJDK. Similar goals to ZGC — concurrent compaction, sub-millisecond pauses. Uses **Brooks forwarding pointers** for concurrent object movement.

**Trade-offs in a sentence:** Parallel GC wins on raw throughput. G1 is the balanced default. ZGC and Shenandoah minimize latency at the cost of slightly higher CPU overhead for concurrent work.

---

**Q5. What is Metaspace and why did it replace PermGen?**

**Model Answer:**
**PermGen (Permanent Generation)** was a fixed-size region of the JVM heap (prior to Java 8) where the JVM stored class metadata, interned strings, and static variables. The problem was that PermGen had a fixed maximum size (configured via `-XX:MaxPermSize`). In applications with heavy dynamic class loading — like application servers with hot redeploy, or frameworks that generate classes at runtime (Spring, Hibernate, CGLIB) — PermGen would fill up, causing the infamous `java.lang.OutOfMemoryError: PermGen space`. It was notoriously difficult to tune.

**Metaspace** (Java 8+) replaces PermGen. Key differences:
- It lives in **native memory** (outside the Java heap), not within the heap.
- It **grows dynamically** by default — no arbitrary fixed cap.
- You can still set `-XX:MaxMetaspaceSize` to cap it, but the default is essentially limited only by available native memory.
- Interned strings moved to the **main heap** (making them GC-eligible like normal objects).
- Static variables also moved to the heap, attached to their `Class` objects.

The practical benefit is eliminating the most common cause of PermGen OOM errors. The risk is that unbounded Metaspace growth can consume all native memory. In well-behaved applications (no class loader leaks), this is not a problem.

---

## 🔷 SECTION 2: Memory Model & Concurrency Semantics

---

**Q6. Explain the Java Memory Model (JMM). What problem does it solve and what guarantees does it provide?**

**Model Answer:**
Before the JMM (formalized in Java 5 via JSR-133), Java had no well-defined memory model, which meant that compiler optimizations, CPU instruction reordering, and CPU caching could produce completely unpredictable behavior in multi-threaded programs — with no contractual guarantees about what a thread would observe.

The JMM defines **when writes made by one thread are guaranteed to be visible to reads by another thread**, using the concept of **happens-before (HB) relationships**.

Key happens-before rules:
- **Program order rule:** Each action in a thread HB every action that comes after it in that thread.
- **Monitor lock rule:** An unlock of a monitor HB every subsequent lock of that monitor.
- **Volatile variable rule:** A write to a volatile field HB every subsequent read of that field.
- **Thread start rule:** A call to `Thread.start()` HB any actions in the started thread.
- **Thread termination rule:** Any action in a thread HB detection of that thread's termination (via `join()`).
- **Transitivity:** If A HB B and B HB C, then A HB C.

Without a HB relationship, the JMM makes no guarantees about visibility. A thread may see stale values from CPU caches or reordered writes.

The JMM also defines **atomicity** — reads and writes of most primitives (except `long` and `double` on 32-bit VMs) are atomic. And **volatile** additionally prevents instruction reordering around the volatile operation (acts as a memory fence).

Elite engineers understand that the JMM is about *visibility* and *ordering*, not about mutual exclusion — that's what locks are for. You can have a data race (two threads accessing a variable without synchronization) even if no thread is "in the middle of" a write.

---

**Q7. What does `volatile` guarantee and what doesn't it guarantee?**

**Model Answer:**
`volatile` provides two guarantees:

**1. Visibility:** A write to a volatile variable is immediately flushed to main memory. A read of a volatile variable always reads from main memory, bypassing CPU caches. This ensures that a write by Thread A is visible to Thread B after Thread B reads that variable.

**2. Ordering (happens-before):** The JMM guarantees that all writes before a volatile write (in program order) are visible to any thread that subsequently reads the volatile variable. This prevents both compiler and CPU reordering across the volatile operation.

**What `volatile` does NOT guarantee:**

**Atomicity of compound operations.** `volatile int counter; counter++` is NOT thread-safe. This is a read-modify-write compound operation. Two threads can both read the same value, both increment, and both write back — losing one increment. For this you need `AtomicInteger` or synchronization.

**Mutual exclusion.** `volatile` does not prevent two threads from entering a critical section simultaneously.

**Classic use case for `volatile`:** A simple boolean flag used to signal between threads — `volatile boolean running = true;` — where one thread writes it and another reads it in a loop. This is a perfect fit: single writer, visibility needed, no compound operation.

The classic misuse is using `volatile` for counters or check-then-act patterns, where the non-atomicity of compound operations causes races.

---

**Q8. What is the difference between `synchronized`, `ReentrantLock`, and `StampedLock`? When would you choose each?**

**Model Answer:**
All three provide mutual exclusion, but with different semantics and capabilities.

**`synchronized`:**
- Built into the language. Cleaner syntax.
- Intrinsic lock (monitor) acquired on `this`, a class object, or any object.
- Automatically released when the block exits (even on exception).
- No ability to try-lock without blocking, no timeout, no interruptible waiting.
- Non-fair (JVM makes no ordering guarantee on waiting threads, though HotSpot tends toward fairness in practice).
- Sufficient for the vast majority of use cases.

**`ReentrantLock`:**
- Explicit lock/unlock (must unlock in `finally` block — discipline required).
- Supports `tryLock()` (non-blocking), `tryLock(timeout)`, and `lockInterruptibly()` — thread can respond to interruption while waiting.
- Supports **fair mode** (`new ReentrantLock(true)`) — FIFO waiting order, preventing thread starvation.
- Supports **multiple Condition objects** (`lock.newCondition()`) — finer-grained wait/notify than a single monitor's `wait()`/`notifyAll()`.
- Better throughput than `synchronized` in some high-contention scenarios (historically; modern JVMs have closed this gap).
- Choose when you need: interruptible locking, timeouts, fairness, or multiple wait-sets.

**`StampedLock` (Java 8+):**
- A lock with three modes: write, read (pessimistic), and **optimistic read**.
- Optimistic read acquires no actual lock — it gets a "stamp" and proceeds, then validates the stamp before using the data. If invalidated (a write occurred), it retries with a pessimistic read lock. This is excellent for read-heavy workloads.
- NOT reentrant — a thread cannot acquire it again if it already holds it.
- Does NOT support `Condition`.
- More complex API, higher risk of misuse.
- Choose when you have a very read-heavy access pattern and need maximum throughput.

---

**Q9. Explain the double-checked locking pattern. What was wrong with it before Java 5 and how was it fixed?**

**Model Answer:**
Double-checked locking (DCL) is an attempt to lazily initialize a singleton with minimal synchronization overhead:

```java
private static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {                    // first check (no lock)
        synchronized (Singleton.class) {
            if (instance == null) {            // second check (with lock)
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```

**Pre-Java 5, this was broken.** The issue is subtle: `instance = new Singleton()` is not atomic. The JVM can reorder it as:
1. Allocate memory
2. Assign `instance` to point at that memory ← **this can happen before step 3**
3. Call the constructor to initialize the fields

So Thread A could be in the process of constructing the Singleton and write the reference to `instance` before the object is fully initialized. Thread B, doing the first null check, sees a non-null `instance`, skips synchronization, and uses a partially constructed object. This is a silent, intermittent bug — nearly impossible to reproduce.

**The fix (Java 5+):** Declare `instance` as `volatile`:

```java
private static volatile Singleton instance;
```

The volatile write to `instance` has a happens-before relationship with any subsequent volatile read. This means the constructor's writes are guaranteed to be visible before the reference is published. The reordering that caused the problem is prohibited by the volatile memory fence.

The **better pattern** most senior engineers prefer, however, is the **Initialization-on-demand holder idiom**:

```java
private static class Holder {
    static final Singleton INSTANCE = new Singleton();
}
public static Singleton getInstance() { return Holder.INSTANCE; }
```

This is thread-safe by the class loading contract (a class is initialized exactly once, atomically), has no synchronization overhead after initialization, and requires no `volatile`.

---

## 🔷 SECTION 3: Type System, Generics & Type Erasure

---

**Q10. Explain type erasure in Java generics. What are its implications and limitations?**

**Model Answer:**
Java generics were introduced in Java 5 as a compile-time mechanism only. The design decision was to implement generics via **type erasure** — generic type parameters are erased from the bytecode at compile time and replaced with their upper bounds (usually `Object`) or with the bounding type if bounded.

So `List<String>` and `List<Integer>` are both represented at runtime as just `List`. The compiler inserts **synthetic casts** at use sites to maintain type safety. The generic type information is present in class file metadata (for reflection via `getGenericType()`, etc.) but is not present in the actual runtime type of an object.

**Implications and limitations:**

1. **Cannot use primitives as type parameters.** `List<int>` is illegal. You must use `List<Integer>` with boxing. This is why primitive streams (`IntStream`, `LongStream`, `DoubleStream`) exist in the Stream API.

2. **Cannot create instances of type parameters.** `new T()` is illegal at runtime because `T` is erased.

3. **Cannot create generic arrays.** `new T[10]` is illegal. The runtime needs to know the component type to create an array.

4. **Cannot use `instanceof` with parameterized types.** `obj instanceof List<String>` is illegal (only `obj instanceof List<?>` is allowed).

5. **Cannot overload methods differing only in generic type parameter.** `void process(List<String>)` and `void process(List<Integer>)` both erase to `void process(List)` — collision.

6. **Heap pollution:** A variable of parameterized type can refer to an object that is not of that parameterized type. Varargs with generics (e.g., `@SafeVarargs`) relates to this.

**Why was erasure chosen?** Backward compatibility with pre-Java-5 code. Changing the JVM's type system to be reified (retaining full generic type information at runtime, like C# does) would have broken all existing libraries and JVM implementations.

---

**Q11. What is the difference between `? extends T` and `? super T`? Explain the PECS principle.**

**Model Answer:**
These are **bounded wildcards** that govern how a parameterized type can be used.

**`? extends T` (upper bounded wildcard):** The wildcard represents some type that is T or a subtype of T. You can **read** from this collection (you'll always get at least a `T`), but you **cannot write** to it (the compiler doesn't know the exact subtype, so it can't guarantee type safety on write).

**`? super T` (lower bounded wildcard):** The wildcard represents some type that is T or a supertype of T. You can **write** a `T` into this collection (it can always accept a `T`), but you can only **read** it as `Object` (the exact supertype is unknown).

**PECS: Producer Extends, Consumer Super** — coined by Joshua Bloch.

- If a structure **produces** (i.e., you read data from it), use `extends`.
- If a structure **consumes** (i.e., you write data into it), use `super`.

Example from `Collections.copy`:
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src)
```
`src` produces `T`s (you read from it) → `extends`. `dest` consumes `T`s (you write into it) → `super`.

**Why does this matter?** Without wildcards, generics are **invariant** — `List<String>` is NOT a subtype of `List<Object>`, even though `String` is a subtype of `Object`. This is correct! If it were allowed, you could add an `Integer` to a `List<String>` via the `List<Object>` reference and corrupt the type system. Wildcards restore some of the covariance/contravariance flexibility in a type-safe way.

---

**Q12. What are raw types? Why are they dangerous and when is it acceptable to use them?**

**Model Answer:**
A **raw type** is a generic class or interface used without any type parameter — e.g., `List` instead of `List<String>`. Raw types exist for backward compatibility with pre-Java-5 code.

**Why dangerous:**
- Using raw types opts out of the entire generics type-checking system. You lose all compile-time safety.
- Operations that would be caught at compile time on parameterized types become runtime `ClassCastException` failures.
- The compiler emits "unchecked" warnings but still allows it — so you can silently introduce type corruptions that fail far from the source of the bug.

Example:
```java
List rawList = new ArrayList();
rawList.add("hello");
rawList.add(42); // no error
String s = (String) rawList.get(1); // ClassCastException at runtime
```

**When might raw types appear legitimately?**
- In legacy code or when interfacing with pre-generics libraries.
- In `instanceof` checks (you must use a raw type or unbounded wildcard there).
- In extremely rare reflection-based code where the type truly is unknown and `Class<?>` or raw is the most honest representation.

**Raw type vs. `<?>`:** Both lose type parameter info, but `List<?>` (unbounded wildcard) still carries the "this is a parameterized type" constraint — you can't add to a `List<?>` (except null). A raw `List` silently allows adds with no checking whatsoever. In new code, always prefer `<?>` over raw.

---

## 🔷 SECTION 4: Object Model & Core Language

---

**Q13. Explain the Java object model. How is an object structured in memory on the HotSpot JVM?**

**Model Answer:**
In HotSpot, every Java object on the heap has a **header** followed by its instance fields.

**Object Header (typically 12–16 bytes):**
- **Mark Word (8 bytes on 64-bit):** Multipurpose. At rest, contains the identity hash code and GC age bits. Under contention, contains a pointer to the inflated monitor (lock). During GC marking or copying, contains forwarding pointers. The mark word is overloaded across different states.
- **Class Pointer / Klass Word (4 or 8 bytes):** A pointer to the `Klass` structure in Metaspace representing the object's type. With **Compressed OOPs** (on by default for heaps ≤ 32GB), this is compressed to 4 bytes.

**Instance Fields:** Laid out after the header. HotSpot reorders fields to minimize padding/alignment gaps — longs and doubles first (8-byte aligned), then ints/floats, then shorts/chars, then booleans/bytes, then object references. This can mean the field order in memory doesn't match the source declaration order.

**Arrays** have an extra 4-byte length field in the header.

**Object alignment:** HotSpot aligns objects to 8-byte boundaries by default, meaning objects smaller than 8 bytes are padded. This alignment is why **Compressed OOPs** work — an address divisible by 8 can be stored in 32 bits to represent a 35-bit address space (up to 32GB).

This knowledge is critical for understanding memory overhead (a `Integer` object costs ~16 bytes vs. 4 bytes for an `int`), for using tools like JOL (Java Object Layout), and for reasoning about **false sharing** in concurrent code (two fields in the same CPU cache line from different logical objects being on the same cache line, causing cache invalidation between threads).

---

**Q14. What is the difference between `==` and `.equals()`? What is the contract of `.equals()` and `hashCode()`?**

**Model Answer:**
**`==`** compares **reference equality** for objects — do both variables point to the exact same object in memory? For primitives, it compares values directly.

**`.equals()`** compares **logical equality** — defined by the class itself. By default (inherited from `Object`), it also uses reference equality (`==`), but classes like `String`, `Integer`, and most value types override it to compare content.

**The `equals()` contract (reflexive, symmetric, transitive, consistent, null-handling):**
- **Reflexive:** `x.equals(x)` must be true.
- **Symmetric:** `x.equals(y)` iff `y.equals(x)`.
- **Transitive:** If `x.equals(y)` and `y.equals(z)`, then `x.equals(z)`.
- **Consistent:** Repeated calls with unchanged objects return the same result.
- **Null:** `x.equals(null)` must return false (not throw NPE).

**The `hashCode()` contract:**
- If `x.equals(y)` is true, then `x.hashCode() == y.hashCode()` **must** be true.
- If `x.equals(y)` is false, hash codes *may* differ (collision is allowed, but distinct codes improve hash collection performance).

**The critical pitfall:** If you override `equals()` without overriding `hashCode()`, objects that are logically equal will have different hash codes. This breaks `HashMap`, `HashSet`, `Hashtable` — you'll be able to `put` a key and then never `get` it back because the lookup uses a different bucket.

**Subtle violation of symmetry in subclassing:** If `Point` equals by coordinates, and `ColorPoint extends Point` equals by coordinates AND color, then `point.equals(colorPoint)` might be true but `colorPoint.equals(point)` might be false — a symmetry violation. This is why equality and inheritance don't play well together, and why **composition over inheritance** is preferred for value types.

---

**Q15. Explain the significance of `String` immutability. How is the String pool implemented and how does `intern()` work?**

**Model Answer:**
`String` in Java is **immutable** — once a `String` object is created, its character sequence cannot be changed. This is a deliberate design choice with several benefits:

- **Thread safety:** An immutable `String` can be freely shared between threads without synchronization.
- **Hashability:** `String` is the most common `HashMap` key. Because its content never changes, its hash code can be cached (and Java does cache it — computed lazily on first `hashCode()` call, stored in the object).
- **Security:** Method parameters of type `String` (database URLs, file paths, class names for class loading) cannot be mutated by the callee. Critical for the security model.
- **Class loading:** The JVM uses `String` for class names internally. Immutability prevents attacks that could substitute class names mid-load.

**The String Pool (String Intern Pool):**
String literals in source code are automatically interned. When the JVM loads a class, string literal constants go into the **string pool** — a canonical set of `String` objects. If two classes use the literal `"hello"`, they reference the same `String` object.

Prior to Java 7, the pool was stored in PermGen (with fixed size). From Java 7+, it moved to the **main heap**, making interned strings eligible for GC and removing the PermGen pressure.

**`String.intern()`:** Explicitly places a `String` into the pool and returns the canonical instance. If a `String` with the same content is already in the pool, it returns that existing instance.

```java
String a = new String("hello"); // forces heap allocation, not pool
String b = "hello";             // pool instance
a == b;          // false
a.intern() == b; // true
```

**Performance note:** `intern()` was historically used to save memory for large sets of duplicate strings. In modern Java, it can be a bottleneck (pool access requires a hash lookup with some synchronization). Alternative approaches like `String.valueOf()` with caches, or `Guava`'s `Interner`, may be more appropriate.

---

**Q16. What are the differences between checked and unchecked exceptions? What is the philosophical argument about each?**

**Model Answer:**
**Checked exceptions** are subclasses of `Exception` (but not `RuntimeException`). They must be either caught or declared in the method signature with `throws`. The compiler enforces this.

**Unchecked exceptions** are subclasses of `RuntimeException` or `Error`. They do not need to be declared or caught. `Error` represents JVM-level failures (OutOfMemoryError, StackOverflowError) that applications typically cannot recover from.

**The philosophical argument for checked exceptions:**
Checked exceptions represent **recoverable exceptional conditions** that the caller should be aware of and handle. The compiler forcing you to acknowledge them creates a contract: "this operation can fail in this specific way, and you must decide what to do about it." This was a deliberate design choice by James Gosling — make the "exceptional paths" as visible as "normal paths."

**The philosophical argument against checked exceptions (and for unchecked):**
In practice, checked exceptions have a significant cost:
- They leak implementation details into the API (a `FileInputStream` throws `IOException` — if you later change the implementation, the checked exception might be wrong).
- They are often handled poorly — developers write empty `catch` blocks or wrap everything in `RuntimeException` just to satisfy the compiler, defeating the purpose.
- They scale poorly. If you have 10 layers of abstraction, checked exceptions must be declared or re-wrapped at every layer — causing "exception pollution."
- Functional interfaces (`Function`, `Predicate`, etc.) cannot throw checked exceptions, making lambda-based code awkward when operations can fail.

**Modern best practices:**
Most modern Java frameworks (Spring, Hibernate, JUnit) use unchecked exceptions almost exclusively. Checked exceptions are still appropriate for cases where the caller truly *must* handle the failure — e.g., `InterruptedException`, `ParseException` for user-supplied input. For programming errors (null access, out-of-bounds, type mismatch) unchecked is always correct — these represent bugs, not recoverable conditions.

---

**Q17. What are the four types of references in Java (`Strong`, `Soft`, `Weak`, `Phantom`) and when would you use each?**

**Model Answer:**
Reference strength controls how the GC treats objects.

**Strong Reference:** The default. `Object o = new Object()`. As long as any strong reference exists, the object will never be GC'd. The GC root reachability analysis follows strong references.

**Soft Reference (`java.lang.ref.SoftReference`):** The GC *may* collect a softly-referenced object, but only under memory pressure. The JVM is guaranteed to clear all soft references before throwing `OutOfMemoryError`. Useful for **memory-sensitive caches** — you get "as much caching as available memory allows" for free.

**Weak Reference (`java.lang.ref.WeakReference`):** The GC *will* collect a weakly-referenced object at the next GC cycle if no strong or soft references exist. It doesn't matter if memory is plentiful. Used in **canonicalizing mappings** (`WeakHashMap` — if the key has no other strong reference, the entry is removed) and for **attaching metadata to objects without preventing their collection** (like thread-local storage or listener registries where you don't want to force object retention).

**Phantom Reference (`java.lang.ref.PhantomReference`):** The object has already been finalized and is ready for reclamation. You cannot even retrieve the object from a phantom reference — `get()` always returns null. Used for **post-mortem cleanup actions** — scheduling cleanup *after* finalization, as a more reliable alternative to `finalize()`. Java 9+ `Cleaner` API is built on phantom references.

**Reference Queues:** Soft, weak, and phantom references can be registered with a `ReferenceQueue`. When the GC is about to (or has) reclaim the object, it enqueues the reference, allowing your code to take action.

**The spectrum in a sentence:** Strong → always kept. Soft → kept unless memory-pressured. Weak → kept until next GC. Phantom → already dead, notification hook.

---

## 🔷 SECTION 5: Modern Java Features

---

**Q18. Explain the Java Module System (JPMS / Project Jigsaw). What problems does it solve and what are its key concepts?**

**Model Answer:**
JPMS (Java Platform Module System), introduced in Java 9, addresses two fundamental problems that grew over Java's history:

1. **The Classpath Hell / JAR Hell problem:** The classpath is flat and unstructured. Any class can access any public class in any JAR. There's no encapsulation between JARs. Dependency conflicts (two JARs with the same class) are silently resolved in classpath order. You can't know what a library's actual dependencies are until runtime.

2. **No platform encapsulation:** Internal JDK classes (like `sun.misc.Unsafe`, internal APIs in `com.sun.*`) were accessible to anyone. This made it impossible for the JDK team to evolve or refactor internals without breaking applications.

**Key concepts:**

- **module-info.java:** A module descriptor at the root of a module. Declares: the module name, what it `requires` (dependencies), what it `exports` (publicly accessible packages), and optionally `opens` (for reflection), `uses` (services), `provides ... with ...` (service implementations).

- **Strong encapsulation:** Only `exports`ed packages are accessible to other modules. Even `public` classes in non-exported packages are inaccessible.

- **Reliable configuration:** Module dependencies are explicit and checked at startup (not at first use). Missing or duplicate modules cause startup failures rather than runtime `ClassNotFoundException`.

- **Platform module graph:** The JDK itself is modularized (java.base, java.sql, java.xml, etc.). Applications can now exclude unneeded platform modules, enabling much smaller runtime images via `jlink`.

- **Layers and class loader architecture:** The module system introduces a **module graph** that sits above the class loader hierarchy.

**Practical reality:** JPMS adoption has been uneven. Many large applications and frameworks use the **unnamed module** (classpath mode) to avoid migration friction. But for runtime image creation and embedding (especially in containers), `jlink` with JPMS is highly valuable. The strong encapsulation benefits are most realized by library authors.

---

**Q19. What are records in Java (Java 16+)? How do they differ from a regular class with the same structure?**

**Model Answer:**
**Records** are a special-purpose class declaration for immutable data carriers. The declaration:

```java
record Point(int x, int y) {}
```

automatically generates: a canonical constructor, private final fields `x` and `y`, public accessor methods `x()` and `y()`, `equals()`, `hashCode()`, and `toString()` — all based on the component list.

**How they differ from a regular class:**

- All components are **implicitly `private final`**. Records enforce immutability of their state.
- You cannot add mutable instance fields to a record (you can add static fields/methods, and add extra constructors or instance methods).
- Records are **implicitly `final`** — you cannot subclass a record.
- Records cannot extend any class (though they can implement interfaces).
- The generated `equals()` and `hashCode()` use all components by default — no partial equality traps.
- The compiler generates a **compact constructor** option — you can validate/normalize values without re-stating parameters.

**What records are not:** They are not just "Lombok's `@Data`." They carry a semantic meaning — a record is a transparent, nominal, immutable data carrier. The compiler and future language features may use this semantic (e.g., pattern matching deconstruction with sealed hierarchies is designed to work with records).

**Records and serialization:** Records implement `Serializable` normally. The deserialization path is guaranteed to go through the canonical constructor (unlike regular classes where the constructor is bypassed), which is a security improvement.

---

**Q20. Explain sealed classes (Java 17). How do they complement pattern matching and records?**

**Model Answer:**
**Sealed classes** allow a class or interface to restrict which other classes may extend or implement it. The declaration uses `sealed` with a `permits` clause:

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
```

The permitted subclasses must be in the same package (or module), and each must be declared `final`, `sealed`, or `non-sealed`.

**The core value proposition:**

Sealed classes give the compiler (and the human reader) **exhaustive knowledge of the type hierarchy**. This is the key property that makes them powerful in combination with pattern matching.

Without sealed classes, if you're switching over a type hierarchy with `instanceof` chains, you must always handle an "unknown subclass" case, because anyone could subclass `Shape`. With sealed classes, the compiler knows *all* possible subtypes, and can verify **exhaustiveness** in `switch` expressions:

```java
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
    // No default needed — compiler verifies exhaustiveness
};
```

**The trio: Sealed classes + Records + Pattern Matching** forms Java's approach to **algebraic data types (ADTs)**, a concept familiar from functional languages like Haskell, Scala, and Rust's enums. A sealed interface with record implementations is essentially a **sum type** — a type that is *exactly one* of a fixed set of product types.

This is architecturally significant. It enables a more functional, data-oriented style of modeling — rather than behavior-heavy class hierarchies with polymorphic dispatch, you model data variants and switch over them. The compiler helps ensure you handle all cases.

---

**Q21. What is the difference between `Optional.orElse()` and `Optional.orElseGet()`? Why does it matter?**

**Model Answer:**
This is a deceptively important distinction about **eagerness vs. laziness**.

**`orElse(T other)`:** The argument `other` is evaluated **eagerly** — it's a direct value expression that is computed regardless of whether the `Optional` is empty or present.

**`orElseGet(Supplier<T> supplier)`:** The `Supplier` is only called **lazily** — only if the `Optional` is empty.

This matters when the default value is expensive to compute:

```java
// BAD: expensiveOperation() is ALWAYS called, even if optional is present
optional.orElse(expensiveOperation());

// GOOD: expensiveOperation() is only called if optional is empty
optional.orElseGet(() -> expensiveOperation());
```

If `expensiveOperation()` involves a database call, file I/O, object construction, or any non-trivial work, using `orElse` wastes that work whenever the Optional is present (which is the common case in a well-designed system).

The same principle applies to `orElseThrow()` vs `orElseThrow(Supplier<X>)` — except `orElseThrow()` without a supplier always throws `NoSuchElementException`, while the supplier form lets you construct a meaningful exception lazily.

This distinction is also a microcosm of a broader principle in Java 8+ APIs: **use `Supplier` when the value is expensive or has side effects and should only be computed on demand.**

---

**Q22. Explain the Stream API's design philosophy. What is the difference between intermediate and terminal operations, and what does lazy evaluation buy you?**

**Model Answer:**
The Stream API represents a **pipeline of aggregate operations over a sequence of elements**. It's declarative — you describe *what* you want rather than *how* to compute it, and the runtime optimizes the execution.

**Intermediate operations** (e.g., `filter`, `map`, `flatMap`, `distinct`, `sorted`, `limit`, `skip`, `peek`) return a new `Stream`. They are **lazy** — calling them doesn't process any elements. They build up a description of the pipeline.

**Terminal operations** (e.g., `collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`, `toList`) trigger actual computation. The pipeline is evaluated only when a terminal operation is invoked.

**What laziness buys:**

1. **Short-circuit evaluation:** `findFirst()` after `filter()` will only process elements until the first match is found — it doesn't process the entire source. Similarly, `anyMatch()` stops at the first match.

2. **Loop fusion:** The JVM can merge multiple intermediate operations into a single pass over the data. `stream.filter(p).map(f).collect(...)` doesn't create intermediate collections — all three operations execute in a single traversal of elements.

3. **Infinite streams:** Laziness enables working with infinite sources (`Stream.generate()`, `Stream.iterate()`) — you can create an infinite stream and `limit()` it, because elements are only pulled as needed.

**Key design principles:**
- Streams are **not data structures** — they don't store elements.
- Streams are **single-use** — once consumed by a terminal operation, they cannot be reused.
- Streams encourage **non-interfering, stateless** operations for correctness in both sequential and parallel contexts.
- **Parallel streams** (`parallelStream()`) use the common ForkJoinPool to process elements concurrently — powerful, but requires that operations are stateless and side-effect-free to produce correct results.

---

## 🔷 SECTION 6: Class Loading & Reflection

---

**Q23. Explain the class loading process in depth. What are the built-in class loaders and what is the parent delegation model?**

**Model Answer:**
Class loading in Java is a three-phase process — Loading, Linking, and Initialization — performed by a `ClassLoader`.

**The built-in class loaders (Java 8):**
- **Bootstrap ClassLoader:** Written in native code. Loads the core JDK classes from `rt.jar` (java.lang, java.util, etc.). Has no Java representation — `String.class.getClassLoader()` returns null.
- **Extension ClassLoader:** Loads JARs from `$JAVA_HOME/lib/ext`. A `URLClassLoader` implemented in Java.
- **Application (System) ClassLoader:** Loads classes from the application classpath. Your application code.

**(Java 9+ change:** Bootstrap and Extension were replaced by the module-aware **Platform ClassLoader**, and the hierarchy was updated to reflect the module system.)

**Parent Delegation Model:**
When a class loader is asked to load a class, it first **delegates to its parent** rather than attempting to load it directly. Only if the parent (and its parent, all the way up to Bootstrap) fails to find the class does the current loader attempt to load it.

**Why parent delegation?** It ensures that core Java classes (like `java.lang.String`) are always loaded by the Bootstrap ClassLoader. Without it, malicious or accidental code could place a class named `java.lang.String` on the classpath and have it loaded instead of the real one — a security and consistency disaster.

**When is parent delegation violated?**
- **OSGi, application servers, and plugin systems** break parent delegation deliberately. Each plugin/webapp gets its own class loader that loads its classes first, enabling class isolation (different versions of the same library in different webapps). This is how Tomcat allows two webapps to use different versions of a library.
- **JDBC's ServiceLoader pattern** requires the thread context class loader, because the Bootstrap ClassLoader loads `java.sql.DriverManager` but needs to load vendor-specific driver classes from the application classpath.

---

**Q24. What is reflection in Java? What are its power and its costs?**

**Model Answer:**
Reflection is the ability of a program to inspect and manipulate its own structure and behavior at runtime — accessing class metadata, invoking methods, reading/writing fields, creating instances — all by name as strings, bypassing compile-time type checking.

**Power:**
- Framework-level capabilities: Dependency injection (Spring), ORM (Hibernate), JSON serialization (Jackson), and test runners (JUnit) all rely heavily on reflection to work with arbitrary user classes.
- Dynamic proxies: `java.lang.reflect.Proxy` creates implementations of interfaces at runtime without compile-time class generation.
- Plugin systems: Loading and introspecting classes from JARs discovered at runtime.
- Tooling: IDEs, debuggers, and profilers use reflection.

**Costs:**

1. **Performance:** Reflective method invocation is significantly slower than direct invocation, though the gap has narrowed with JVM improvements. The reflective call cannot be inlined by JIT as aggressively. In hot paths, use `MethodHandle` (introduced in Java 7) — handles are JIT-friendly and approach direct call performance.

2. **Security:** Reflection can access private fields and methods via `setAccessible(true)`. The module system (Java 9+) restricts this — unless a package is `opens`, reflective access to its private members is denied by default.

3. **Type safety loss:** Reflection bypasses generics and type checking. Errors become runtime exceptions rather than compile-time errors.

4. **Fragility:** Code that calls methods by string name breaks silently if the method is renamed during refactoring. Compilers and IDEs can't help you.

5. **GraalVM native image:** Reflective access must be declared in configuration metadata for native compilation to work, because the static analysis cannot track reflection dynamically.

**Modern alternatives:** `MethodHandle` and `VarHandle` (Java 9+) provide reflection-like capabilities with better JIT optimization. The `invokedynamic` bytecode instruction powers dynamic language runtimes and lambda implementation on the JVM.

---

## 🔷 SECTION 7: Performance, Profiling & Tooling

---

**Q25. What is a memory leak in Java? How can it occur in a GC-managed language, and how do you diagnose it?**

**Model Answer:**
A Java memory leak is a situation where **objects that are no longer logically needed by the application are still reachable** from GC roots, preventing the GC from collecting them. The GC can only collect objects it can prove are unreachable — it cannot know about application-level "logical unreachability."

**Common causes:**

1. **Static collections:** A static `Map` or `List` that accumulates entries and is never cleaned. Since static fields are GC roots, everything they reference is always reachable.

2. **Listeners and callbacks not deregistered:** An event source holds a reference to registered listeners. If you add a listener and never remove it, the listener (and everything it references transitively) is retained for the source's lifetime.

3. **ThreadLocal leaks in thread pools:** `ThreadLocal` values are held as long as the thread lives. In thread pools, threads live forever. If a `ThreadLocal` holds a large object and you don't explicitly call `remove()`, it leaks for the thread's lifetime.

4. **Caches without eviction:** An unbounded `HashMap` used as a cache. Use `WeakHashMap`, or a cache with an eviction policy (Guava Cache, Caffeine).

5. **Inner class references to outer class:** A non-static inner class (or anonymous class) holds an implicit reference to its enclosing outer class instance. If the inner class outlives the outer (e.g., used in a callback, stored in a collection), it prevents the outer from being GC'd.

6. **Class loader leaks:** In hot-redeploy scenarios, if any object loaded by the old class loader is referenced from a long-lived object (e.g., a static field of a library), the entire old class loader — and all classes it loaded — is retained in Metaspace. This is the classic "PermGen/Metaspace OOM after multiple redeploys."

**Diagnosis:**
- **Heap dump analysis:** Take a heap dump (`jmap -dump:format=b,file=heap.hprof <pid>`, or via JMX, or on OOM with `-XX:+HeapDumpOnOutOfMemoryError`). Analyze with Eclipse MAT or VisualVM/JDK Mission Control — look at retained heap size, GC roots holding large object graphs.
- **GC logs:** Monitoring GC behavior over time. A memory leak shows as heap usage growing each GC cycle with the post-GC baseline drifting upward.
- **Profilers:** JFR (Java Flight Recorder), Async-Profiler, YourKit to observe allocation rates and object retention.

---

**Q26. What is false sharing and how do you address it in Java?**

**Model Answer:**
**False sharing** is a performance pathology in multi-core systems where two threads on different cores are each writing to different variables that happen to reside on the **same CPU cache line** (typically 64 bytes). Even though the threads are writing to *logically independent data*, the cache coherence protocol (MESI) treats the entire cache line as the unit of coherence. Each write by one core invalidates the cache line for all other cores, forcing them to reload it, causing excessive cache invalidation traffic and reducing effective parallelism to near-serial performance.

**How to detect it:**
Performance is much worse than single-threaded despite good CPU utilization. Low throughput at high core counts. Profilers with hardware counter support (perf, VTune, Async-Profiler) will show high cache miss rates on specific variables.

**Solutions in Java:**

1. **Padding:** Surround the hot field with enough dummy fields to ensure it occupies a full cache line alone:
```java
// 8 bytes padding before + 8 bytes value + 48 bytes padding after = 64 bytes
```

2. **`@Contended` annotation (Java 8+, HotSpot):** `@sun.misc.Contended` (or `@jdk.internal.vm.annotation.Contended`) tells the JVM to automatically pad the annotated field or class to avoid false sharing. Requires JVM flag `-XX:-RestrictContended` for user-code use.

**Famous example:** `java.util.concurrent.atomic.LongAdder` uses a striped array of `Cell` objects, each marked with `@Contended`, allowing different threads to update different cells without false sharing, dramatically outperforming `AtomicLong` under contention.

False sharing is a great example of how hardware architecture forces its way into application-level design, even in a managed language.

---

This covers the full landscape of elite-level Java general knowledge — from JVM internals and the memory model, through generics and the type system, to modern language features and performance considerations. A candidate who answers these with depth and nuance, draws connections between topics, and proactively mentions trade-offs and edge cases would represent genuine mastery of the Java platform.