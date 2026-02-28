# 🧵 Multi-Threading in Java — Complete Notes

---

## 📌 What is a Thread?

Imagine you're cooking dinner. You're boiling water **AND** chopping vegetables at the same time. You're doing two things **concurrently**. That's what threads do in a program.

A **Thread** is the smallest unit of execution inside a program. A program can have **multiple threads running at the same time**, each doing its own task.

- **Process** = An entire running program (e.g., Chrome browser)
- **Thread** = A task inside that program (e.g., one tab loading, another playing video)

Java runs in a **JVM process**, and inside that JVM, you can spin up many threads.

---

## 📌 Why Multi-threading?

- **Performance**: Use multiple CPU cores instead of just one
- **Responsiveness**: UI doesn't freeze while background work happens
- **Efficiency**: While one thread waits (e.g., for a database), another can work

---

## 📌 Thread Lifecycle (States)

```
NEW → RUNNABLE → RUNNING → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED
```

| State | Meaning |
|---|---|
| **NEW** | Thread created but `.start()` not called yet |
| **RUNNABLE** | Ready to run, waiting for CPU |
| **RUNNING** | Actually executing right now |
| **BLOCKED** | Waiting to acquire a lock (synchronized) |
| **WAITING** | Waiting indefinitely (e.g., `wait()`) |
| **TIMED_WAITING** | Waiting for a specific time (e.g., `sleep(1000)`) |
| **TERMINATED** | Finished execution |

---

## 📌 Creating Threads — 3 Ways

### ✅ Way 1: Extend `Thread` class

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running: " + Thread.currentThread().getName());
    }
}

// Usage
MyThread t = new MyThread();
t.start(); // DO NOT call run() directly! That runs on the same thread.
```

> ⚠️ **Calling `run()` directly** is just a normal method call — no new thread is created. Always call `start()`.

---

### ✅ Way 2: Implement `Runnable` interface (Preferred)

```java
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

// Usage
Thread t = new Thread(new MyTask());
t.start();
```

**Why prefer Runnable over Thread?**
Because Java doesn't support multiple inheritance. If you extend `Thread`, you can't extend any other class. `Runnable` is just an interface — more flexible.

---

### ✅ Way 3: Lambda (since Java 8)

```java
Thread t = new Thread(() -> System.out.println("Lambda thread!"));
t.start();
```

---

### ✅ Way 4: `Callable` + `Future` (when you need a return value)

```java
import java.util.concurrent.*;

Callable<Integer> task = () -> {
    return 42; // can return a value, unlike Runnable
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);

Integer result = future.get(); // blocks until result is ready
System.out.println(result); // 42
executor.shutdown();
```

> `Runnable` → no return value, can't throw checked exceptions
> `Callable<T>` → returns a value of type T, can throw checked exceptions

---

## 📌 Important `Thread` Class Methods

| Method | What it does |
|---|---|
| `start()` | Starts the thread (calls `run()` in a new thread) |
| `run()` | Contains the task — don't call directly |
| `sleep(ms)` | Pauses current thread for given milliseconds |
| `join()` | Wait for another thread to finish before continuing |
| `interrupt()` | Signals a thread to stop what it's doing |
| `isInterrupted()` | Checks if thread was interrupted |
| `Thread.currentThread()` | Returns reference to currently running thread |
| `getName()` / `setName()` | Get/set thread name |
| `getPriority()` / `setPriority()` | 1 (MIN) to 10 (MAX), default is 5 |
| `isDaemon()` / `setDaemon(true)` | Daemon threads die when all non-daemon threads finish |
| `yield()` | Hints the scheduler to let other threads run (not reliable) |

### `sleep()` example:
```java
try {
    Thread.sleep(2000); // pauses this thread for 2 seconds
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // best practice: re-interrupt
}
```

### `join()` example:
```java
Thread t1 = new Thread(() -> System.out.println("T1 done"));
t1.start();
t1.join(); // main thread waits here until t1 finishes
System.out.println("Main continues");
```

---

## 📌 The Big Problem: Race Condition 🏁

When two threads access **shared data** at the same time and try to modify it, you get unpredictable results.

```java
class Counter {
    int count = 0;

    void increment() {
        count++; // This is NOT atomic! It's 3 steps: read → modify → write
    }
}
```

If two threads call `increment()` at the same time, they might both read `0`, both add 1, and both write back `1` — so count ends up as `1` instead of `2`. This is a **race condition**.

---

## 📌 `synchronized` Keyword 🔒

`synchronized` ensures **only one thread** can execute a block/method at a time. It uses a **monitor lock** (also called intrinsic lock or mutex).

Think of it like a **single-key bathroom** — only one person can be inside at a time.

### Synchronized Method:
```java
class Counter {
    int count = 0;

    synchronized void increment() { // lock on 'this' object
        count++;
    }
}
```

### Synchronized Block (more granular, better performance):
```java
class Counter {
    int count = 0;
    Object lock = new Object();

    void increment() {
        synchronized(lock) { // only this section is locked, not whole method
            count++;
        }
        // other code here runs without lock
    }
}
```

### Synchronized Static Method:
```java
class Counter {
    static int count = 0;

    static synchronized void increment() { // lock on Counter.class object
        count++;
    }
}
```

> 🔑 **Key rule**: Two threads can never hold the same lock at the same time. One must wait.

---

## 📌 `volatile` Keyword 👁️

### The Problem it Solves: CPU Caching

Modern CPUs cache variables in registers/cache for performance. So Thread A might be reading a stale/cached value of a variable that Thread B already updated in main memory.

```java
class Worker {
    boolean running = true; // Thread B updates this to false

    void work() {
        while (running) { // Thread A might never see the updated value!
            // do work
        }
    }
}
```

### Solution: `volatile`

```java
volatile boolean running = true;
```

`volatile` tells the JVM:
- **Always read this variable from main memory** (never from CPU cache)
- **Always write directly to main memory**
- Guarantees **visibility** across threads

### `volatile` vs `synchronized`

| | `volatile` | `synchronized` |
|---|---|---|
| Visibility | ✅ Yes | ✅ Yes |
| Atomicity | ❌ No | ✅ Yes |
| Performance | Faster | Slower |
| Use case | Simple flags | Compound operations |

> ⚠️ `volatile` does NOT make `count++` safe. `count++` is 3 operations. Use `synchronized` or `AtomicInteger` for that.

---

## 📌 `wait()`, `notify()`, `notifyAll()` — Thread Communication

These are methods on `Object` class (every Java object has them). They're used for **inter-thread communication**.

> 🔑 Must be called inside a `synchronized` block, otherwise you get `IllegalMonitorStateException`.

| Method | What it does |
|---|---|
| `wait()` | Releases the lock and waits until notified |
| `wait(ms)` | Waits for at most given milliseconds |
| `notify()` | Wakes up ONE waiting thread (random) |
| `notifyAll()` | Wakes up ALL waiting threads |

### Classic Producer-Consumer Example:

```java
class SharedBuffer {
    int data;
    boolean hasData = false;

    synchronized void produce(int value) throws InterruptedException {
        while (hasData) {
            wait(); // wait until consumer consumes
        }
        data = value;
        hasData = true;
        System.out.println("Produced: " + value);
        notify(); // wake up consumer
    }

    synchronized int consume() throws InterruptedException {
        while (!hasData) {
            wait(); // wait until producer produces
        }
        hasData = false;
        System.out.println("Consumed: " + data);
        notify(); // wake up producer
        return data;
    }
}
```

> ✅ Always use `wait()` inside a `while` loop, not `if`. Because of **spurious wakeups** — a thread can wake up without being notified.

---

## 📌 Deadlock 💀

A deadlock is when two (or more) threads are **waiting for each other** forever. Neither can proceed.

```
Thread 1 holds Lock A, waiting for Lock B
Thread 2 holds Lock B, waiting for Lock A
→ Both wait forever = DEADLOCK
```

```java
Object lockA = new Object();
Object lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized(lockA) {
        synchronized(lockB) { /* work */ } // t1 waits for lockB
    }
});

Thread t2 = new Thread(() -> {
    synchronized(lockB) {
        synchronized(lockA) { /* work */ } // t2 waits for lockA
    }
});
```

### How to Avoid Deadlock:
1. **Lock ordering** — always acquire locks in the same order
2. **Use `tryLock()`** with timeout from `ReentrantLock`
3. **Avoid nested locks** when possible
4. **Use higher-level abstractions** like `java.util.concurrent`

---

## 📌 `java.util.concurrent` Package 🚀

This package provides powerful, high-level tools. You should prefer these over raw `synchronized`.

---

### `ReentrantLock`

A more flexible alternative to `synchronized`. Same thread can acquire it multiple times (reentrant).

```java
import java.util.concurrent.locks.ReentrantLock;

ReentrantLock lock = new ReentrantLock();

void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock(); // ALWAYS unlock in finally!
    }
}
```

**Extra features over `synchronized`:**
- `tryLock()` — try to acquire, don't block if unavailable
- `tryLock(timeout, unit)` — try with timeout
- `lockInterruptibly()` — can be interrupted while waiting
- `ReentrantLock(true)` — fair lock (threads served in order)

---

### `ReadWriteLock` / `ReentrantReadWriteLock`

Multiple threads can **read** simultaneously, but only **one thread can write** (and no reads allowed during write).

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();

// Multiple readers at same time
rwLock.readLock().lock();
try { /* read data */ } finally { rwLock.readLock().unlock(); }

// Only one writer, blocks all readers
rwLock.writeLock().lock();
try { /* write data */ } finally { rwLock.writeLock().unlock(); }
```

---

### Atomic Classes (`java.util.concurrent.atomic`)

Thread-safe operations **without locks**, using CPU-level atomic instructions (CAS — Compare And Swap).

| Class | Use |
|---|---|
| `AtomicInteger` | Thread-safe int |
| `AtomicLong` | Thread-safe long |
| `AtomicBoolean` | Thread-safe boolean |
| `AtomicReference<T>` | Thread-safe object reference |

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet(); // thread-safe, no lock needed
counter.getAndAdd(5);
counter.compareAndSet(5, 10); // if value is 5, set to 10
```

---

### `ExecutorService` — Thread Pools 🏊

Creating raw threads is expensive. Thread pools **reuse** threads.

```java
ExecutorService executor = Executors.newFixedThreadPool(4); // pool of 4 threads

for (int i = 0; i < 10; i++) {
    executor.submit(() -> {
        System.out.println("Task by: " + Thread.currentThread().getName());
    });
}

executor.shutdown(); // stops accepting new tasks, finishes existing ones
executor.awaitTermination(10, TimeUnit.SECONDS); // wait for all to finish
```

### Types of Executors:

| Factory Method | Description |
|---|---|
| `newFixedThreadPool(n)` | Fixed number of threads |
| `newCachedThreadPool()` | Creates threads as needed, reuses idle ones |
| `newSingleThreadExecutor()` | Only one thread, tasks run sequentially |
| `newScheduledThreadPool(n)` | Run tasks after delay or periodically |

---

### `CountDownLatch`

One or more threads wait until a count reaches zero. **One-time use.**

```java
CountDownLatch latch = new CountDownLatch(3); // count = 3

// 3 worker threads each call latch.countDown()
executor.submit(() -> { doWork(); latch.countDown(); });
executor.submit(() -> { doWork(); latch.countDown(); });
executor.submit(() -> { doWork(); latch.countDown(); });

latch.await(); // main thread waits here until count = 0
System.out.println("All workers done!");
```

> Use case: Wait for all services to start before accepting requests.

---

### `CyclicBarrier`

A group of threads all wait for each other at a **barrier point**. **Reusable** (unlike `CountDownLatch`).

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All at barrier!"));

Runnable task = () -> {
    doPhase1();
    barrier.await(); // all 3 must reach here before anyone continues
    doPhase2();
};
```

---

### `Semaphore`

Controls access to a resource with a **fixed number of permits**. Like a parking lot with N spots.

```java
Semaphore semaphore = new Semaphore(3); // max 3 concurrent access

void accessResource() throws InterruptedException {
    semaphore.acquire(); // get a permit (blocks if 0 permits)
    try {
        // access shared resource
    } finally {
        semaphore.release(); // return the permit
    }
}
```

---

### `BlockingQueue`

Thread-safe queue. Producer blocks if full, consumer blocks if empty. Perfect for producer-consumer.

```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

// Producer
queue.put(item); // blocks if queue is full

// Consumer
Integer item = queue.take(); // blocks if queue is empty
```

Implementations: `ArrayBlockingQueue`, `LinkedBlockingQueue`, `PriorityBlockingQueue`

---

### `CompletableFuture` (Java 8+)

Async programming with a fluent API. Chain operations, combine results.

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> "Hello")         // runs in ForkJoinPool
    .thenApply(s -> s + " World")       // transform result
    .thenApply(String::toUpperCase);

String result = future.get(); // "HELLO WORLD"

// Combine two futures
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");
CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + " " + b);
```

---

## 📌 Thread-Safe Collections

| Regular | Thread-Safe Alternative |
|---|---|
| `ArrayList` | `CopyOnWriteArrayList` |
| `HashMap` | `ConcurrentHashMap` |
| `HashSet` | `CopyOnWriteArraySet` |
| `LinkedList` | `ConcurrentLinkedQueue` |

### `ConcurrentHashMap`
Locks only a **segment** of the map (not the whole map like `Hashtable`). Much better performance.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.computeIfAbsent("b", k -> 2); // atomic operation
```

---

## 📌 `ThreadLocal`

Each thread gets its own **isolated copy** of a variable. No sharing, no synchronization needed.

```java
ThreadLocal<Integer> threadId = ThreadLocal.withInitial(() -> 0);

Thread t1 = new Thread(() -> {
    threadId.set(1);
    System.out.println(threadId.get()); // prints 1
});

Thread t2 = new Thread(() -> {
    threadId.set(2);
    System.out.println(threadId.get()); // prints 2
});
```

> Use case: Database connections, user sessions per request in web servers.
> ⚠️ Always call `threadId.remove()` when done (especially in thread pools) to avoid memory leaks.

---

## 📌 Java Memory Model (JMM) — Simplified

The JMM defines how threads interact through memory.

Key concepts:
- **Happens-before relationship**: If action A happens-before action B, then A's results are visible to B.
- `synchronized` establishes happens-before
- `volatile` writes happen-before subsequent volatile reads
- `Thread.start()` happens-before any action in the started thread
- All actions in a thread happen-before `Thread.join()`

---

## 📌 `ForkJoinPool` (Java 7+)

For **divide-and-conquer** algorithms. Break a big task into smaller tasks (fork), then combine results (join).

```java
class SumTask extends RecursiveTask<Long> {
    int[] arr;
    int start, end;

    @Override
    protected Long compute() {
        if (end - start <= 1000) {
            // small enough, compute directly
            long sum = 0;
            for (int i = start; i < end; i++) sum += arr[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();  // run async
        return right.compute() + left.join(); // wait for left
    }
}
```

---

## 📌 Quick Reference: What to use When?

| Scenario | Tool |
|---|---|
| Simple counter | `AtomicInteger` |
| Protect a block of code | `synchronized` or `ReentrantLock` |
| Lots of reads, few writes | `ReentrantReadWriteLock` |
| Wait for N tasks to complete | `CountDownLatch` |
| All threads reach checkpoint | `CyclicBarrier` |
| Limit concurrent access | `Semaphore` |
| Producer-consumer | `BlockingQueue` |
| Thread-safe map | `ConcurrentHashMap` |
| Async task with result | `CompletableFuture` |
| Thread-local data | `ThreadLocal` |
| Run tasks in background | `ExecutorService` |

---

---

# 🎯 Common Interview Questions & Answers

---

### Q1. What is the difference between `Thread` and `Runnable`?

**Answer:**
`Thread` is a class, `Runnable` is a functional interface. When you extend `Thread`, you can't extend any other class (single inheritance). When you implement `Runnable`, you keep the option to extend another class. Also, `Runnable` represents a **task** (what to do), while `Thread` represents the **runner** (mechanism of execution). Best practice is to always implement `Runnable` or `Callable` and pass it to a `Thread` or `ExecutorService`.

---

### Q2. What happens if you call `run()` instead of `start()`?

**Answer:**
The `run()` method executes on the **current thread**, not a new thread. No new thread is created. It's just a regular method call. Only `start()` creates a new thread and then calls `run()` on that thread.

---

### Q3. What is a race condition?

**Answer:**
A race condition occurs when two or more threads access shared mutable data concurrently and the outcome depends on the timing/order of execution. For example, `count++` is not atomic — it involves read, increment, and write. If two threads do this simultaneously, one update can be lost. Fix it with `synchronized`, `AtomicInteger`, or `ReentrantLock`.

---

### Q4. What is the difference between `synchronized` method and `synchronized` block?

**Answer:**
A synchronized method locks the entire method using `this` as the monitor (or `ClassName.class` for static). A synchronized block lets you specify **which object** to lock and locks only a portion of code, giving better performance and more flexibility. Synchronized blocks are generally preferred because they minimize the time a lock is held.

---

### Q5. What is `volatile` and when do you use it?

**Answer:**
`volatile` ensures that reads and writes to a variable go directly to **main memory**, not from CPU cache. This guarantees **visibility** — all threads see the latest value. Use it for simple flags (`boolean running`) or single-variable state. However, `volatile` does NOT provide atomicity for compound operations like `count++`. For that, use `AtomicInteger` or `synchronized`.

---

### Q6. What is a deadlock? How do you prevent it?

**Answer:**
A deadlock occurs when Thread A holds Lock 1 and waits for Lock 2, while Thread B holds Lock 2 and waits for Lock 1 — both wait forever.

Prevention strategies:
1. Always acquire locks in a **consistent global order**
2. Use `tryLock(timeout)` from `ReentrantLock` instead of blocking forever
3. Minimize lock scope and avoid nested locks
4. Use higher-level concurrency utilities from `java.util.concurrent`

---

### Q7. Difference between `wait()` and `sleep()`?

| | `wait()` | `sleep()` |
|---|---|---|
| Class | `Object` | `Thread` |
| Releases lock? | ✅ Yes | ❌ No |
| Where called | Inside `synchronized` block | Anywhere |
| Woken up by | `notify()` / `notifyAll()` | After timeout / interrupt |
| Purpose | Inter-thread communication | Pause execution |

---

### Q8. What is the difference between `notify()` and `notifyAll()`?

**Answer:**
`notify()` wakes up **one** randomly selected thread waiting on the object's monitor. `notifyAll()` wakes up **all** waiting threads — they then compete for the lock and only one proceeds at a time. It's safer to use `notifyAll()` in most cases to avoid situations where the wrong thread is woken up and the system stalls.

---

### Q9. What is a daemon thread?

**Answer:**
A daemon thread is a **background thread** that serves other threads (e.g., garbage collector). When all non-daemon (user) threads finish, the JVM exits even if daemon threads are still running. Set with `thread.setDaemon(true)` before calling `start()`. You should not use daemon threads for critical tasks like I/O since they can be killed abruptly.

---

### Q10. What is `ThreadLocal`?

**Answer:**
`ThreadLocal` provides each thread with its **own independent copy** of a variable. Threads cannot see each other's copies. It's like giving each thread its own locker. Common use cases: storing user sessions, database connections, or SimpleDateFormat instances per thread. Important: always call `remove()` when done, especially in thread pools, to prevent memory leaks.

---

### Q11. What is the difference between `CountDownLatch` and `CyclicBarrier`?

| | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Reusable? | ❌ One-time | ✅ Yes, resets |
| Who decrements? | Any thread | Each participating thread |
| Direction | Many → one (wait for N) | All → one point (all meet) |
| Callback on reach? | ❌ No | ✅ Yes (Runnable) |

---

### Q12. What is `ConcurrentHashMap` and how does it differ from `Hashtable`?

**Answer:**
Both are thread-safe, but `Hashtable` locks the **entire map** on every operation, causing huge contention. `ConcurrentHashMap` uses **segment-level locking** (in Java 7) and **CAS + synchronized on individual buckets** (in Java 8), allowing multiple threads to read and write to different parts simultaneously. `ConcurrentHashMap` has much better performance and never throws `ConcurrentModificationException`. Also, `Hashtable` doesn't allow null keys/values; `ConcurrentHashMap` doesn't either. `HashMap` allows null but is not thread-safe.

---

### Q13. What is `CompletableFuture` and how is it different from `Future`?

**Answer:**
`Future` (Java 5) represents a pending async result but is **blocking** — you must call `get()` to retrieve the result, which blocks the thread. `CompletableFuture` (Java 8) is non-blocking, supports **chaining** operations with `thenApply`, `thenCompose`, `thenCombine`, error handling with `exceptionally`, and can be completed manually. It's far more powerful for building async pipelines.

---

### Q14. What is the Java Memory Model (JMM)?

**Answer:**
The JMM defines the rules for how threads interact through memory. The key concept is **happens-before**: if action A happens-before action B, then A's effects are visible to B. Synchronization actions (entering/exiting `synchronized`, reading/writing `volatile`) create happens-before relationships. Without these, the JVM and CPU are free to reorder instructions for optimization, which can cause threads to see stale data.

---

### Q15. What are spurious wakeups and how do you handle them?

**Answer:**
A spurious wakeup is when a thread wakes up from `wait()` without being notified. This can happen due to JVM or OS-level behavior. To handle this, always use `wait()` inside a `while` loop (not `if`):

```java
synchronized(lock) {
    while (!condition) { // re-check condition after every wakeup
        lock.wait();
    }
    // proceed
}
```

---

### Q16. What is `Livelock`? How is it different from Deadlock?

**Answer:**
In a deadlock, threads are **frozen** waiting for each other. In a livelock, threads are **actively running** but keep responding to each other in a way that prevents any progress. Imagine two people in a hallway both stepping aside the same way repeatedly. Neither is blocked, but neither gets through. Fix it by introducing randomness or priority in decision-making.

---

### Q17. What is thread starvation?

**Answer:**
Thread starvation happens when a thread is perpetually denied access to resources it needs because other threads with higher priority keep getting it first. The starving thread is runnable but never actually runs. Fix it by using **fair locks** (`new ReentrantLock(true)`), which serve threads in FIFO order.

---

### Q18. Explain `ForkJoinPool` and work-stealing.

**Answer:**
`ForkJoinPool` is designed for recursive divide-and-conquer tasks. It uses **work-stealing**: each thread has its own deque of tasks. When a thread finishes its tasks, it **steals** tasks from the tail of another busy thread's deque. This keeps all threads busy and maximizes CPU utilization. `CompletableFuture.supplyAsync()` uses `ForkJoinPool.commonPool()` by default.

---

### Q19. What is the difference between `synchronized` and `ReentrantLock`?

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Explicit unlock? | ❌ Auto-release | ✅ Must call `unlock()` |
| Try without blocking? | ❌ No | ✅ `tryLock()` |
| Interruptible wait? | ❌ No | ✅ `lockInterruptibly()` |
| Fair mode? | ❌ No | ✅ `new ReentrantLock(true)` |
| Condition variables? | `wait/notify` | `Condition` object |
| Simpler syntax? | ✅ Yes | ❌ More verbose |

---

### Q20. What is `Exchanger`?

**Answer:**
`Exchanger<T>` is a synchronization point where **two threads swap data** with each other. Both threads must call `exchange()`, and each receives what the other sent. Useful in pipeline designs where two stages of a pipeline need to exchange buffers.

```java
Exchanger<String> exchanger = new Exchanger<>();
// Thread 1: String result = exchanger.exchange("Hello from T1");
// Thread 2: String result = exchanger.exchange("Hello from T2");
// After exchange: T1 gets "Hello from T2" and vice versa
```

---

> 💡 **Pro Tips for Interviews:**
> - Always mention thread safety when discussing shared state
> - Know when to use `volatile` vs `synchronized` vs `Atomic`
> - Understand that `synchronized` has overhead — use the finest granularity possible
> - Prefer `java.util.concurrent` over manual `wait/notify` in real code
> - Mention that premature optimization is bad — synchronize correctly first, then optimize