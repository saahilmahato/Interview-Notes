# ☕ JVM Internals — Complete Cheatsheet

---

## 1. What is the JVM?

The **Java Virtual Machine (JVM)** is an abstract computing machine that provides a runtime environment for executing Java bytecode. It's the reason Java is **"Write Once, Run Anywhere"** — your `.java` source compiles to platform-independent `.class` bytecode, and the JVM on each OS translates that to native machine code.

```
MyApp.java  →  [javac compiler]  →  MyApp.class (bytecode)  →  [JVM]  →  Native Machine Code
```

The JVM doesn't just run Java. Kotlin, Scala, Groovy, Clojure — they all compile to the same bytecode and run on the JVM.

---

## 2. JVM Architecture — The Big Picture

```
┌──────────────────────────────────────────────────────────────┐
│                        JVM Architecture                       │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Class Loader Subsystem                     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                          ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   Runtime Data Areas                    │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────┐ ┌────┐ ┌───────┐  │ │
│  │  │  Method  │ │   Heap   │ │Stack │ │ PC │ │Native │  │ │
│  │  │  Area    │ │          │ │      │ │Reg │ │Stack  │  │ │
│  │  └──────────┘ └──────────┘ └──────┘ └────┘ └───────┘  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                          ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Execution Engine                           │ │
│  │   ┌──────────────┐  ┌───────┐  ┌──────────────────┐   │ │
│  │   │  Interpreter │  │  JIT  │  │  Garbage Collector│   │ │
│  │   └──────────────┘  └───────┘  └──────────────────┘   │ │
│  └─────────────────────────────────────────────────────────┘ │
│                          ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Native Method Interface (JNI)              │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Class Loader Subsystem

This is responsible for **loading, linking, and initializing** class files into the JVM.

### 3.1 The Three Built-in Class Loaders

| Loader | Loads | Location |
|---|---|---|
| **Bootstrap ClassLoader** | Core Java classes (`java.lang.*`, `java.util.*`) | `$JAVA_HOME/lib/rt.jar` (or modules in Java 9+) |
| **Extension / Platform ClassLoader** | Extension libraries | `$JAVA_HOME/lib/ext/` |
| **Application ClassLoader** | Your app's classes + classpath | `-classpath` / `-cp` |

### 3.2 Delegation Model (Parent-First)

When a class needs to be loaded, the child loader **always delegates to the parent first**. Only if the parent can't find it does the child try.

```
Request to load "com.myapp.Foo"
    ↓
Application ClassLoader → delegates to ↑
Extension ClassLoader   → delegates to ↑
Bootstrap ClassLoader   → tries first
    ↓ (not found)
Extension ClassLoader   → tries
    ↓ (not found)
Application ClassLoader → loads it ✅
```

**Why?** This prevents malicious code from overriding core classes. You can't write your own `java.lang.String` and trick the JVM into loading it.

### 3.3 Loading → Linking → Initialization

**Loading** — Reads the `.class` file, creates a `Class` object in the Method Area.

**Linking** has three sub-steps:
- **Verification** — Ensures bytecode is structurally valid and doesn't violate JVM spec (no stack overflows, proper types, etc.)
- **Preparation** — Allocates memory for static variables and sets them to **default values** (`0`, `null`, `false`)
- **Resolution** — Resolves symbolic references (e.g., method names) into direct memory references

**Initialization** — Executes static initializers and assigns actual values to static variables. This is when `static { }` blocks run.

---

## 4. Runtime Data Areas (Memory Model)

### 4.1 Method Area (Metaspace)

- Shared across all threads
- Stores **class-level data**: class structure, method bytecode, constant pool, static variables, field/method metadata
- In Java 7 and earlier it was called **PermGen** (Permanent Generation) and was part of the heap with a fixed size
- In Java 8+, it became **Metaspace** — lives in **native memory** (off-heap), can grow dynamically

> **PermGen vs Metaspace** is a very common interview question. PermGen had a fixed max size (`-XX:MaxPermSize`) which caused `OutOfMemoryError: PermGen space`. Metaspace uses native memory and auto-grows, though you can cap it with `-XX:MaxMetaspaceSize`.

### 4.2 Heap

- Shared across all threads
- Where **all objects and arrays** live
- Subject to **Garbage Collection**
- Divided into generations (in most collectors):

```
┌───────────────────────────────────────────────────────┐
│                        HEAP                           │
│  ┌─────────────────────────┐  ┌─────────────────────┐ │
│  │       Young Generation  │  │   Old Generation    │ │
│  │  ┌───────┐ ┌──┐  ┌──┐  │  │   (Tenured Space)   │ │
│  │  │  Eden │ │S0│  │S1│  │  │                     │ │
│  │  └───────┘ └──┘  └──┘  │  │                     │ │
│  └─────────────────────────┘  └─────────────────────┘ │
└───────────────────────────────────────────────────────┘
```

- **Eden Space** — New objects are allocated here
- **Survivor Spaces (S0, S1)** — Objects that survive Minor GC bounce between these
- **Old/Tenured Generation** — Objects that have survived enough GC cycles are "promoted" here

### 4.3 JVM Stack (Per Thread)

- Each thread has its own stack
- Stores **Stack Frames** — one per method call
- Each frame contains:
  - **Local Variable Array** — method parameters and local vars
  - **Operand Stack** — working area for computations (like a scratch pad)
  - **Frame Data** — reference to constant pool, return address

```java
void foo() {
    int a = 5;       // stored in local variable array
    int b = 10;
    int c = a + b;   // a and b pushed to operand stack, ADD executed, result stored
}
```

- Stack grows with each method call. Too deep recursion → `StackOverflowError`
- Size tunable with `-Xss` (e.g., `-Xss512k`)

### 4.4 PC Register (Program Counter)

- One per thread
- Holds the address of the **currently executing JVM instruction**
- If a native method is running, the PC is undefined

### 4.5 Native Method Stack

- Supports native (non-Java) method calls via JNI
- Works like the JVM stack but for C/C++ code

---

## 5. How the JVM Executes Bytecode

### 5.1 What is Bytecode?

Bytecode is a set of instructions for the JVM's virtual processor. It's not machine code — it's an intermediate representation. Each instruction is 1 byte (hence "byte"code), with optional operands.

```java
// Java source
int add(int a, int b) {
    return a + b;
}
```

```
// Corresponding bytecode (javap -c output)
0: iload_1      // push local var 1 (a) onto operand stack
1: iload_2      // push local var 2 (b) onto operand stack
2: iadd         // pop two ints, add them, push result
3: ireturn      // return int from top of stack
```

### 5.2 Interpreter

The JVM starts by **interpreting** bytecode — reading each instruction and executing it one at a time. Simple, but slow because the same code path is translated over and over.

### 5.3 JIT Compiler (Just-In-Time)

The JIT compiler is the key to JVM performance. It monitors running code and identifies **"hot spots"** — code that runs frequently (loops, hot methods). It then **compiles that bytecode into native machine code** and caches it.

```
Method called 1st–Nth time   →  Interpreted
Method reaches "hot" threshold →  JIT compiles to native code
Method called (N+1)th time+  →  Executes native code directly (fast!)
```

HotSpot JVM (the reference JVM) has two JIT compilers:

| Compiler | Also Known As | Optimizes For |
|---|---|---|
| **C1** | Client Compiler | Fast startup, lighter optimizations |
| **C2** | Server Compiler | Max throughput, aggressive optimization |

**Tiered Compilation** (default since Java 8) uses both:
- Levels 0–1: Interpreted
- Level 2–3: C1 compiled (with profiling)
- Level 4: C2 compiled (fully optimized)

### 5.4 Key JIT Optimizations

**Inlining** — replaces a method call with the method body itself to eliminate call overhead.
```java
// Before inlining
int result = add(a, b);

// After inlining (conceptually)
int result = a + b;  // no method call overhead
```

**Escape Analysis** — determines if an object "escapes" the current method/thread. If not, the JVM can allocate it on the stack (faster) instead of the heap, or even eliminate the allocation entirely.

**Loop Unrolling** — duplicates loop body to reduce loop control overhead.

**Dead Code Elimination** — removes code that can never execute.

**Devirtualization** — converts virtual method calls to direct calls when the JVM can prove only one type is involved.

**On-Stack Replacement (OSR)** — allows JIT to swap interpreted code with compiled code *while* a loop is still running.

---

## 6. Garbage Collection

GC automatically reclaims memory used by objects that are no longer reachable. "Reachable" means there's a chain of references from a **GC Root** to the object.

**GC Roots** include: local variables in active threads, static variables, JNI references.

### 6.1 Generational Hypothesis

Most objects die young. The heap is divided into generations to exploit this:
- **Minor GC** — cleans Young Generation, happens frequently, usually fast
- **Major/Full GC** — cleans Old Generation (and sometimes everything), expensive

### 6.2 GC Algorithms

**Mark-and-Sweep**
1. **Mark phase** — traverse from GC roots, mark all reachable objects
2. **Sweep phase** — reclaim unmarked (unreachable) memory
Problem: leaves fragmented memory.

**Mark-Sweep-Compact**
Adds a compaction step — moves live objects together, eliminating fragmentation.

**Copying Collector**
Splits memory into two halves. Copies live objects to the other half, then wipes the first. Eden → Survivor spaces use this approach.

### 6.3 Available GC Collectors

| Collector | Flag | Best For | STW? |
|---|---|---|---|
| **Serial GC** | `-XX:+UseSerialGC` | Small single-threaded apps | Yes |
| **Parallel GC** | `-XX:+UseParallelGC` | High throughput, batch jobs | Yes (parallel threads) |
| **CMS (deprecated)** | `-XX:+UseConcMarkSweepGC` | Low pause times | Mostly concurrent |
| **G1 GC** | `-XX:+UseG1GC` | Balanced latency/throughput (default Java 9+) | Short pauses |
| **ZGC** | `-XX:+UseZGC` | Ultra-low latency (<10ms pauses) | Almost fully concurrent |
| **Shenandoah** | `-XX:+UseShenandoahGC` | Low pause, similar to ZGC | Almost fully concurrent |

**Stop-The-World (STW)**: JVM pauses ALL application threads during GC. This is why GC tuning matters.

### 6.4 G1 GC Deep Dive

G1 (Garbage-First) divides the heap into equal-sized **regions** (~1–32 MB each). Regions are dynamically assigned as Eden, Survivor, Old, or Humongous (for large objects). It prioritizes collecting regions with the most garbage first — hence "Garbage-First."

```
┌────┬────┬────┬────┬────┬────┐
│ E  │ O  │ S  │ E  │ H  │ O  │   E=Eden, S=Survivor,
├────┼────┼────┼────┼────┼────┤   O=Old, H=Humongous
│ O  │ E  │ O  │ E  │ S  │ O  │
└────┴────┴────┴────┴────┴────┘
```

### 6.5 GC Phases (G1 Example)

1. **Young GC** — Evacuates Eden + Survivor to new Survivor/Old regions. STW.
2. **Concurrent Marking** — Marks live objects while app runs.
3. **Mixed GC** — Collects Young + some Old regions. STW but short.
4. **Full GC** — Fallback, single-threaded, long pause. You want to avoid this.

---

## 7. JVM Tuning & Flags

### 7.1 Heap Size

```bash
-Xms512m          # Initial heap size
-Xmx2g            # Maximum heap size
-Xmn256m          # Young generation size (or -XX:NewSize / -XX:MaxNewSize)
-XX:NewRatio=3    # Old:Young ratio (3 = 75% old, 25% young)
-XX:SurvivorRatio=8  # Eden:Survivor ratio (8 = 80% Eden, 10% S0, 10% S1)
```

**Best practice**: Set `-Xms` = `-Xmx` in production to avoid heap resizing pauses.

### 7.2 GC Tuning

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200       # Target max pause time (G1 will try to honor this)
-XX:G1HeapRegionSize=16m       # Region size (1MB–32MB, power of 2)
-XX:GCTimeRatio=12             # Goal: spend <1/(1+12) = ~7.7% of time in GC
-XX:+PrintGCDetails            # Verbose GC logs (older flag)
-Xlog:gc*:file=gc.log          # Modern GC logging (Java 9+)
-XX:ParallelGCThreads=4        # Threads for STW GC phases
-XX:ConcGCThreads=2            # Threads for concurrent phases
```

### 7.3 Metaspace

```bash
-XX:MetaspaceSize=256m         # Initial metaspace (triggers first GC when hit)
-XX:MaxMetaspaceSize=512m      # Cap metaspace (prevent unbounded native memory growth)
```

### 7.4 JIT Tuning

```bash
-XX:+TieredCompilation         # Enable tiered compilation (default Java 8+)
-XX:CompileThreshold=10000     # How many invocations before JIT kicks in
-server                        # Use Server JVM (C2, better long-running perf)
-client                        # Use Client JVM (C1, better startup)
-XX:+PrintCompilation          # Log JIT compilation events
```

### 7.5 Stack & Thread

```bash
-Xss512k          # Stack size per thread (default ~512k–1m)
```

### 7.6 Useful Diagnostic Flags

```bash
-XX:+HeapDumpOnOutOfMemoryError             # Auto-dump heap on OOM
-XX:HeapDumpPath=/tmp/heap.hprof            # Where to dump
-XX:+PrintFlagsFinal                        # Print all JVM flags and their values
-XX:+UnlockDiagnosticVMOptions             # Unlock extra diagnostic flags
-verbose:gc                                 # Basic GC output
```

---

## 8. JVM Across Platforms

### 8.1 Platform Independence

The `.class` bytecode is **platform-independent**. The JVM itself is **platform-dependent** — there are different JVM implementations for Windows, macOS, Linux, etc. The JVM abstracts the OS and hardware.

```
MyApp.class (same file)
    ↓               ↓               ↓
JVM for Windows  JVM for Linux  JVM for macOS
    ↓               ↓               ↓
Windows x86_64  Linux x86_64   macOS ARM64
```

### 8.2 JVM Distributions

| Distribution | Vendor | Notes |
|---|---|---|
| **HotSpot** | Oracle / OpenJDK | Reference implementation |
| **OpenJ9** | Eclipse (IBM) | Low memory footprint, faster startup |
| **GraalVM** | Oracle | Polyglot, native image compilation |
| **Azul Zulu** | Azul Systems | TCK-certified OpenJDK builds |
| **Amazon Corretto** | AWS | Free LTS OpenJDK builds |
| **Temurin** | Eclipse Adoptium | Community-driven OpenJDK |

### 8.3 GraalVM & Native Image

GraalVM allows **Ahead-of-Time (AOT)** compilation — compiling Java all the way to a native binary at build time. The result starts instantly and uses less memory but loses JIT's runtime profile-based optimizations.

```
Java source → GraalVM native-image → native binary (no JVM needed at runtime!)
```

Popular in serverless/microservices (Quarkus, Micronaut use this).

---

## 9. Class File Structure

A `.class` file has a specific binary format:

```
magic number (0xCAFEBABE)    ← Identifies it as a Java class file
minor_version / major_version ← Java version (e.g., 52.0 = Java 8)
constant_pool                ← Literals, class/method/field names
access_flags                 ← public, abstract, final, etc.
this_class / super_class     ← Class hierarchy
interfaces[]                 ← Implemented interfaces
fields[]                     ← Field declarations
methods[]                    ← Method declarations + bytecode
attributes[]                 ← Extra info (SourceFile, LineNumber, etc.)
```

The **Constant Pool** is like a symbol table — strings, class names, method signatures are stored here and referenced by index throughout the rest of the file.

---

## 10. The Java Memory Model (JMM)

The JMM defines how threads interact through memory. It's not about heap layout — it's about **visibility and ordering guarantees**.

### Key Concepts

**Visibility problem**: Without synchronization, changes made by one thread may not be visible to another (due to CPU caches, registers).

**`volatile`** — guarantees visibility. Writes to a volatile variable are immediately flushed to main memory. Reads always come from main memory. Also prevents instruction reordering around the variable.

**`synchronized`** — guarantees both visibility and atomicity. Acquiring a monitor flushes cached values; releasing writes changes back.

**Happens-Before Relationship**: If action A *happens-before* action B, then A's effects are visible to B. Rules include:
- Within a thread, code executes in program order
- Monitor unlock happens-before subsequent lock
- Write to volatile happens-before subsequent read
- Thread start/join

**Instruction Reordering**: CPUs and compilers reorder instructions for performance. The JMM defines when this is allowed. The `volatile` keyword and synchronization create **memory barriers** that prevent problematic reorderings.

---

## 11. JVM Startup & Shutdown

### Startup Sequence

1. OS loads the JVM binary
2. JVM initializes runtime data areas
3. Bootstrap class loader loads core classes (`java.lang.Object`, etc.)
4. JVM creates the main thread
5. `main(String[] args)` method is invoked
6. Application runs

### Shutdown Sequence

1. Last non-daemon thread finishes, OR `System.exit()` is called
2. JVM invokes all registered shutdown hooks (`Runtime.addShutdownHook(...)`)
3. If `runFinalizersOnExit` is enabled, finalizers run
4. JVM halts

---

## 12. Safepoints

A **safepoint** is a point in code where the JVM can safely pause all threads (for GC, deoptimization, etc.). The JVM doesn't stop threads randomly — it asks them to reach the nearest safepoint.

Examples of safepoint-inducing operations:
- GC stop-the-world phases
- Class redefinition (hot-swap)
- Deoptimization (removing JIT-compiled code)
- Thread dumps

Loops with counted iterations may not have safepoints in them by default, potentially causing **"safepoint stalls"** — one thread delays others from stopping.

---

## 13. Object Memory Layout

Every Java object in memory has:

```
┌─────────────────┐
│   Mark Word     │  8 bytes — hash code, lock state, GC age
├─────────────────┤
│   Klass Pointer │  4–8 bytes — pointer to class metadata
├─────────────────┤
│  (Array Length) │  4 bytes — only for arrays
├─────────────────┤
│   Instance Data │  actual fields
├─────────────────┤
│    Padding      │  to align to 8-byte boundary
└─────────────────┘
```

The **Mark Word** is overloaded — it stores different things depending on lock state:
- Unlocked: hash code + GC age
- Biased locking: thread ID
- Thin lock: lock record pointer
- Fat lock (inflated): monitor pointer

**Compressed OOPs** (`-XX:+UseCompressedOops`, default on 64-bit JVMs with heap ≤32GB) — uses 4-byte references instead of 8-byte, saving significant memory.

---

## 14. String Pool (Interning)

String literals are stored in a special area called the **String Pool** (part of the heap since Java 7, was in PermGen before).

```java
String a = "hello";          // goes to string pool
String b = "hello";          // returns same reference from pool
String c = new String("hello"); // creates new object on heap, NOT in pool

a == b  // true (same reference)
a == c  // false (different objects)
a.equals(c) // true

String d = c.intern(); // explicitly adds to pool, returns pooled reference
a == d  // true
```

---

## 15. Common Interview Questions & Answers

---

**Q1: What is the difference between JDK, JRE, and JVM?**

**JVM (Java Virtual Machine)** is the runtime engine that executes bytecode. It's an abstract machine that handles memory, GC, and bytecode execution. **JRE (Java Runtime Environment)** = JVM + standard class libraries. It's what you need to *run* Java programs. **JDK (Java Development Kit)** = JRE + development tools (compiler `javac`, `javap`, `jstack`, etc.). It's what you need to *develop* Java programs.

---

**Q2: What is the difference between PermGen and Metaspace?**

PermGen was the method area in Java 7 and earlier. It was part of the heap with a **fixed maximum size** (`-XX:MaxPermSize`, default ~64–256 MB). When too many classes were loaded (e.g., in app servers), it caused `OutOfMemoryError: PermGen space`. In Java 8, it was replaced by **Metaspace**, which lives in **native memory** (off-heap). It auto-grows as needed, significantly reducing OOM issues. You can still cap it with `-XX:MaxMetaspaceSize` to prevent runaway class loading from consuming all native memory.

---

**Q3: What happens when you call `new Object()` in Java?**

1. JVM checks if `Object` class is already loaded; if not, loads it via the class loader
2. JVM allocates memory for the object on the heap (typically in Eden space)
3. Memory is zeroed out (default values)
4. The object header (mark word + klass pointer) is initialized
5. The constructor is called to initialize fields
6. A reference to the object is returned

---

**Q4: What is JIT compilation and why is it better than pure interpretation?**

Pure interpretation re-translates the same bytecode every single time it runs. JIT identifies hot code paths and compiles them to **native machine code once**, storing the result. Subsequent executions run the native code directly — much faster. JIT also applies sophisticated runtime optimizations (inlining, escape analysis) that are only possible with runtime profiling data that a static compiler doesn't have.

---

**Q5: Explain the difference between Minor GC, Major GC, and Full GC.**

**Minor GC** collects only the Young Generation (Eden + Survivors). It runs frequently, is fast, and causes short STW pauses. Objects that survive enough Minor GCs are promoted to Old Gen. **Major GC** collects the Old Generation. It's slower because Old Gen is larger and contains long-lived objects. **Full GC** collects the entire heap (Young + Old) plus Metaspace. It's the most expensive and causes the longest pauses. Full GC often indicates a problem: memory leak, undersized heap, or survivor space thrashing.

---

**Q6: What is Stop-The-World (STW) and why is it necessary?**

STW is a JVM pause where **all application threads are suspended**. It's necessary for operations that require a consistent view of the heap — like finding all GC roots, compacting memory, or updating object references. Without it, objects could be moving in memory while a thread holds a reference to their old location, causing corruption. Modern collectors like ZGC minimize STW by doing most work concurrently, keeping pauses under ~10ms regardless of heap size.

---

**Q7: What is escape analysis and what optimizations does it enable?**

Escape analysis determines whether an object can be accessed outside the method or thread that created it. If an object **does not escape**:
- **Stack allocation** — the object can be allocated on the stack instead of the heap, bypassing GC entirely
- **Scalar replacement** — the object can be broken into individual primitive fields (no object allocated at all)
- **Lock elision** — if a synchronized block can be proven to only be accessed by one thread, the lock is removed entirely

---

**Q8: How does the class loader delegation model prevent security issues?**

When any class loader receives a load request, it first delegates to its parent. This means the **Bootstrap ClassLoader always gets first chance** to load any class. If someone puts a malicious `java/lang/String.class` on the application classpath, the Application ClassLoader will try to load it — but it delegates to Bootstrap first, which loads the real `java.lang.String` from the JDK. The malicious class is never loaded. This is called the **parent-first (delegation) model**.

---

**Q9: What is the difference between `volatile` and `synchronized`?**

`volatile` guarantees **visibility** — changes by one thread are immediately visible to all others — and prevents reordering of operations around the volatile variable. But it does NOT guarantee atomicity. `i++` on a volatile `i` is still not thread-safe (it's read-modify-write, three operations). `synchronized` guarantees both **visibility and atomicity**. It ensures only one thread executes the block at a time and that all memory changes are properly flushed/synchronized. Use `volatile` for simple flags; use `synchronized` (or `Atomic*` classes) for compound actions.

---

**Q10: What causes `OutOfMemoryError` and what are the types?**

`java.lang.OutOfMemoryError: Java heap space` — heap is full, GC can't free enough. Fix: increase `-Xmx`, fix memory leaks, reduce object creation.

`java.lang.OutOfMemoryError: GC overhead limit exceeded` — JVM spending >98% of time in GC but recovering <2% of heap. Classic sign of a memory leak.

`java.lang.OutOfMemoryError: Metaspace` — too many classes loaded, Metaspace exhausted. Fix: increase `-XX:MaxMetaspaceSize`, check for class loader leaks.

`java.lang.OutOfMemoryError: Direct buffer memory` — NIO direct buffers exceeded. Fix: `-XX:MaxDirectMemorySize`.

`java.lang.OutOfMemoryError: unable to create new native thread` — OS can't create more threads. Fix: reduce thread count, increase `-Xss`, increase OS thread limits.

---

**Q11: What is the Happens-Before guarantee?**

It's the foundation of the Java Memory Model. If action A happens-before action B, then the JVM **guarantees** that B can see all of A's effects (writes). Key happens-before relationships: program order within a thread, unlocking a monitor → locking that same monitor, writing to a volatile → reading that volatile, `Thread.start()` → any action in that thread, any action in a thread → `Thread.join()` returns. Without a happens-before chain between two threads, you have no visibility guarantee — even if code *looks* like it should work.

---

**Q12: What is Tiered Compilation?**

Tiered Compilation (default since Java 8) combines the fast startup of C1 with the peak performance of C2. Code progresses through levels:
- **Level 0**: Interpreted
- **Level 1**: C1 compiled, no profiling (simple methods)
- **Level 2**: C1 compiled, limited profiling
- **Level 3**: C1 compiled, full profiling (collects data for C2)
- **Level 4**: C2 compiled using profile data from Level 3 (maximum optimization)

A method might be called thousands of times at Level 3 while C2 compiles it in the background, then switches to Level 4.

---

**Q13: How does G1 differ from the Parallel GC?**

Parallel GC is purely throughput-oriented — it uses multiple threads but still causes full STW pauses proportional to heap size. G1 divides the heap into fixed-size regions (instead of contiguous young/old areas), works concurrently to track garbage, and picks the most garbage-dense regions to collect. It gives you a **pause time target** (`-XX:MaxGCPauseMillis`) that it tries to meet by only collecting as many regions as possible within that time budget. Parallel GC is better for pure throughput (batch); G1 is better for mixed workloads needing predictable latency.

---

**Q14: What is deoptimization?**

JIT compilers make **speculative optimizations** based on observed runtime behavior. For example, if only one implementation of an interface has been seen, the JIT may inline a direct call. But if later a second implementation is loaded, that assumption is violated. The JVM performs **deoptimization** — it discards the compiled code and falls back to the interpreter (or re-compiles with updated assumptions). This is transparent to the application but can cause brief performance dips.

---

**Q15: What is String interning and when would you use it?**

String interning puts a String into the **String Pool** so all identical String literals share the same reference. Literals are interned automatically. `new String("hello")` is not interned unless you call `.intern()`. Use interning when you have a large number of repeated strings and want to save memory. Beware: in Java 6 and earlier, interning put strings in PermGen which could cause OOM. In Java 7+, the pool is in the heap, so it's GC'd normally. Interning is generally recommended only for specific high-memory-pressure scenarios.

---

*Happy coding! 🚀*