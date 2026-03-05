# Golang — Senior Engineer Interview Questions & Model Answers

---

## 1. Fundamentals & Language Design

---

**Q1: Why does Go not have classes, and how does it achieve object-oriented behavior?**

**Model Answer:**
Go deliberately omits classes to avoid the complexity of deep inheritance hierarchies, which tend to produce tightly coupled, brittle code. Instead, Go achieves OO-like behavior through three mechanisms:

- **Structs** hold state (data fields).
- **Methods** are functions with a receiver, attached to any named type — not just structs.
- **Interfaces** define behavior contracts implicitly (structural typing / duck typing).

```go
type Animal interface {
    Sound() string
}

type Dog struct{ Name string }

func (d Dog) Sound() string { return "Woof" }

func Describe(a Animal) {
    fmt.Println(a.Sound())
}
```

`Dog` satisfies `Animal` without ever declaring it. This is called **implicit interface satisfaction**, and it's what makes Go's composition model so powerful. You compose behavior by embedding types and implementing interfaces, not by inheriting from a base class. The result is shallow, readable call graphs and extremely easy mocking in tests.

---

**Q2: What is the difference between `new` and `make` in Go?**

**Model Answer:**
- `new(T)` allocates zeroed storage for a value of type `T` and returns a `*T`. It works for any type.
- `make(T, ...)` allocates **and initializes** internal data structures for slices, maps, and channels only. It returns a value of type `T`, not a pointer.

Why the distinction? Slices, maps, and channels are reference types with hidden internal state (a slice header has a pointer, length, and capacity; a map has a hash table; a channel has a ring buffer and a mutex). Without initialization, they'd be nil and panic on use. `make` handles that initialization.

```go
p := new(int)       // *int, *p == 0
s := make([]int, 5) // []int of len 5, cap 5, all zeros
m := make(map[string]int) // initialized, ready to insert
```

Trying `new([]int)` gives you a pointer to a nil slice — usable but you'd still need to `make` before appending meaningfully.

---

**Q3: Explain how Go's garbage collector works and what "stop-the-world" means in its context.**

**Model Answer:**
Go uses a **tri-color, concurrent mark-and-sweep** GC running mostly in parallel with application goroutines, targeting very low pause times (sub-millisecond in modern versions).

**Tri-color marking:**
- **White** — not yet visited; will be collected if still white at end.
- **Grey** — discovered but children not yet scanned.
- **Black** — fully scanned; retained.

The GC starts by marking roots (stack variables, globals) grey, then progressively scans grey objects, turning them black, until no grey objects remain. Anything still white is unreachable and swept.

**Write barrier:** Because the application is mutating the heap concurrently, a write barrier intercepts pointer writes during marking to ensure the invariant "no black object points to a white object" is maintained.

**Stop-the-world (STW)** pauses are now extremely short — used only at two points:
1. Enabling write barriers at the start of the mark phase.
2. Finalizing marking at the end.

Tuning is done via `GOGC` (default 100 = trigger GC when heap doubles) and `GOMEMLIMIT` (added in Go 1.19). Engineers should be aware that allocation-heavy hot paths pressure the GC; using sync.Pool, pre-allocated buffers, or value semantics (avoiding heap escapes) reduces GC load.

---

**Q4: What is escape analysis and how does it affect performance?**

**Model Answer:**
Escape analysis is a **compile-time** analysis where the Go compiler determines whether a variable's lifetime is bounded to the stack frame or must be heap-allocated (it "escapes" to the heap).

Stack allocation is faster (just increment stack pointer, no GC pressure, better cache locality). Heap allocation is slower and adds GC load.

Common escape triggers:
- Returning a pointer to a local variable.
- Storing a local variable in an interface.
- Passing to a function the compiler can't inline.
- Appending to a slice whose size isn't statically known.

```go
func stackAlloc() int {
    x := 42   // stays on stack
    return x
}

func heapAlloc() *int {
    x := 42   // escapes to heap — returned pointer outlives frame
    return &x
}
```

You can inspect escape decisions with `go build -gcflags="-m"`. In hot paths, I proactively restructure code to avoid unnecessary escapes — for example, using `sync.Pool` for frequently allocated objects, accepting structs by value when they're small, and avoiding interface boxing in tight loops.

---

**Q5: What are the zero values in Go and why are they important?**

**Model Answer:**
Every type in Go has a zero value — the value a variable holds when declared without initialization:

| Type | Zero Value |
|---|---|
| `int`, `float64`, etc. | `0` |
| `bool` | `false` |
| `string` | `""` |
| pointer, slice, map, channel, func, interface | `nil` |
| struct | zero value of each field |

This is not cosmetic — it's a design philosophy. Zero values should be **useful**. Classic examples:
- `sync.Mutex` — zero value is an unlocked mutex, ready to use.
- `bytes.Buffer` — zero value is an empty buffer, ready to write to.
- `sync.WaitGroup` — zero value is valid, count = 0.

This eliminates entire classes of initialization bugs. You never need a constructor just to create a valid object. A well-designed Go type should have a meaningful zero value, and when it can't (e.g., a type requiring a connection string), the constructor pattern with `New*` functions is used, often returning errors.

---

## 2. Goroutines & Concurrency

---

**Q6: How does Go's scheduler work? What is the M:N threading model?**

**Model Answer:**
Go uses an **M:N scheduler** — M goroutines multiplexed onto N OS threads, managed by the Go runtime.

The scheduler has three core entities:
- **G** (Goroutine) — the logical unit of execution with its own tiny stack (starts at ~2–8KB, grows dynamically).
- **M** (Machine) — an OS thread.
- **P** (Processor) — a scheduling context. Holds a run queue of goroutines. `GOMAXPROCS` controls the number of Ps.

Each P has a local run queue. Ms pick Gs from their P's queue. When a P's queue is empty, it **work-steals** from other Ps, keeping all threads busy.

**Blocking system calls** are handled gracefully: when an M is about to block (e.g., file I/O), the runtime detaches its P and hands it to another M (or creates a new one), so goroutines on that P keep running. When the blocking call returns, the M tries to reacquire a P.

**Network I/O** is handled via `netpoller` — a platform-specific non-blocking I/O abstraction (epoll on Linux). Network goroutines park instead of blocking an OS thread.

This design lets Go run hundreds of thousands of goroutines on a handful of OS threads with very low overhead per goroutine.

---

**Q7: What is a data race in Go? How do you detect and prevent one?**

**Model Answer:**
A data race occurs when two goroutines concurrently access the same memory location and at least one access is a write, with no synchronization between them. The result is undefined behavior — not just incorrect values, but potential memory corruption.

**Detection:** Go ships with a built-in race detector based on ThreadSanitizer:
```bash
go test -race ./...
go run -race main.go
```
It instruments memory accesses at runtime and reports races with full stack traces for both conflicting accesses.

**Prevention — layered approach:**

1. **Channels** — communicate data ownership between goroutines instead of sharing memory.
2. **sync.Mutex / sync.RWMutex** — protect shared mutable state.
3. **sync/atomic** — lock-free operations on scalars.
4. **Immutability** — pass copies instead of pointers when the data is small.
5. **sync.Map** — for concurrent map access (though for most cases a mutex-guarded map is simpler and faster).
6. **Ownership discipline** — a goroutine "owns" data; transfers ownership via channels.

The mantra is *"Do not communicate by sharing memory; share memory by communicating."* In practice, I use channels for ownership transfer and mutexes for shared state where channel overhead isn't justified (e.g., a shared cache).

---

**Q8: Explain buffered vs unbuffered channels and when to use each.**

**Model Answer:**

**Unbuffered channel (`make(chan T)`):**
- A send blocks until a receiver is ready, and vice versa.
- Provides **synchronization guarantee**: when the send returns, you know the receiver got the value.
- Use for: signaling, rendezvous synchronization, guaranteed handoff.

**Buffered channel (`make(chan T, n)`):**
- A send blocks only when the buffer is full; a receive blocks only when it's empty.
- Decouples sender and receiver in time.
- Use for: rate-limiting work (semaphore pattern), smoothing bursty producers, pipeline stages where you want to allow some queuing.

```go
// Semaphore pattern — limit concurrency to 5
sem := make(chan struct{}, 5)
for _, job := range jobs {
    sem <- struct{}{}
    go func(j Job) {
        defer func() { <-sem }()
        process(j)
    }(job)
}
```

**Pitfall:** A buffered channel is not a free lunch. An unbuffered channel fails loudly when the receiver is missing; a buffered channel will just silently queue until it fills or your goroutine leaks. I default to unbuffered for synchronization and only add a buffer after profiling shows contention or latency issues.

---

**Q9: How do you handle goroutine leaks? What patterns prevent them?**

**Model Answer:**
A goroutine leak is a goroutine that is permanently blocked — waiting on a channel or mutex that will never be satisfied. Since goroutines hold their stack, large numbers of leaked goroutines exhaust memory.

**Detection:** `runtime.NumGoroutine()` in tests, or tools like `goleak` (Uber's library), or a `/debug/pprof/goroutine` endpoint.

**Prevention patterns:**

1. **Context cancellation** — the single most important pattern:
```go
func worker(ctx context.Context, ch <-chan Work) {
    for {
        select {
        case w, ok := <-ch:
            if !ok { return }
            process(w)
        case <-ctx.Done():
            return
        }
    }
}
```

2. **Always close channels you own** — enables range loops and blocked receivers to unblock.

3. **Use errgroup** (`golang.org/x/sync/errgroup`) for groups of goroutines with coordinated cancellation.

4. **Set deadlines/timeouts on blocking operations** — never do `<-ch` without a `ctx.Done()` case in production code.

5. **Goroutine ownership** — the goroutine that starts another goroutine is responsible for ensuring it can terminate. Document the termination contract.

In code review, I look for goroutines started in a loop without a cancel mechanism — a classic source of leaks under load.

---

**Q10: What is `sync.Once` and when would you use it over `init()`?**

**Model Answer:**
`sync.Once` ensures a function is executed exactly once across all goroutines, even under concurrent calls:

```go
var (
    instance *DB
    once     sync.Once
)

func GetDB() *DB {
    once.Do(func() {
        instance = connect()
    })
    return instance
}
```

**vs `init()`:**
- `init()` runs at program startup, always. `sync.Once` is **lazy** — runs on first call.
- `init()` runs before `main`, so it can't depend on runtime configuration (env vars parsed at startup, flags, etc.).
- `sync.Once` is appropriate for expensive initialization that might not always be needed (e.g., only if a certain code path is hit).
- `sync.Once` handles errors better — you can store the error and return it; `init()` must panic or silently swallow.

**Also important:** `sync.Once`'s `Do` only calls `f` once even if the first call panics — subsequent calls are no-ops. This is a subtlety to watch out for if initialization can fail.

---

**Q11: What are the differences between `sync.Mutex` and `sync.RWMutex`? When does `RWMutex` hurt performance?**

**Model Answer:**
- `sync.Mutex` — exclusive lock. Only one goroutine can hold it at a time, regardless of read/write.
- `sync.RWMutex` — distinguishes readers from writers. Many goroutines can hold a read lock simultaneously; a write lock is exclusive.

`RWMutex` wins when reads vastly outnumber writes and the critical section is long enough that reader contention is meaningful.

**Where `RWMutex` hurts:**
1. **Write-heavy workloads** — writers must wait for all current readers to drain, then block all new readers. Under high write frequency, this causes writer starvation and overhead.
2. **Short critical sections** — the overhead of `RWMutex` (it has more internal state: reader count, pending writers) can exceed the cost of the actual operation. A plain `Mutex` with a nanosecond-long critical section will often outperform.
3. **NUMA / cache coherence** — `RWMutex` uses an atomic counter for reader counts. On multi-socket systems, this causes cache line bouncing between sockets on every `RLock`/`RUnlock`, which can be worse than a well-contested mutex.

I always benchmark before choosing `RWMutex`. A common pattern I've seen that's wrongly assumed to be faster is `RWMutex` on a map that's written to frequently — profiling usually reveals it's slower.

---

## 3. Interfaces & Type System

---

**Q12: How do nil interfaces differ from interfaces holding nil pointers in Go?**

**Model Answer:**
This is one of Go's most infamous gotchas. An interface value internally has two components: `(type, value)`.

- A **nil interface** has `(nil, nil)` — both type and value are nil. A nil check returns `true`.
- An **interface holding a nil pointer** has `(*MyType, nil)` — the type is set, the value is nil. A nil check returns `false`.

```go
type MyError struct{}
func (e *MyError) Error() string { return "err" }

func getError(fail bool) error {
    var e *MyError // nil pointer
    if fail {
        e = &MyError{}
    }
    return e // WRONG — always returns non-nil interface if *MyError type is known
}

err := getError(false)
if err != nil {
    fmt.Println("This prints even though e was nil!") // BUG
}
```

**Fix:** Return `nil` explicitly rather than a typed nil:
```go
func getError(fail bool) error {
    if fail {
        return &MyError{}
    }
    return nil // returns true nil interface
}
```

In practice: never return a typed nil as an interface. Return the interface type directly as `nil`.

---

**Q13: What is an empty interface (`interface{}` / `any`) and what are the tradeoffs of using it?**

**Model Answer:**
`interface{}` (aliased as `any` since Go 1.18) is satisfied by every type. It's Go's equivalent of `Object` in Java — a type-erased container.

**Legitimate uses:**
- Encoding/decoding arbitrary JSON (`map[string]interface{}`).
- Containers before generics (now largely replaceable).
- Variadic logging/formatting functions (`fmt.Println`).

**Tradeoffs and costs:**
1. **No compile-time type safety** — errors surface at runtime via type assertions.
2. **Boxing cost** — storing a concrete value in an interface allocates a copy on the heap (for non-pointer types). In hot paths this adds GC pressure.
3. **Loss of readability** — callers must document and remember the expected concrete types.
4. **Type assertion overhead** — runtime type checking is slower than static dispatch.

**Type assertion vs type switch:**
```go
// Type assertion — panics if wrong, use two-value form
v, ok := x.(string)

// Type switch — preferred for multiple types
switch v := x.(type) {
case string:
    fmt.Println(v)
case int:
    fmt.Println(v * 2)
}
```

With generics available in Go 1.18+, most uses of `interface{}` for type-generic containers should be replaced with type parameters.

---

**Q14: Explain generics in Go — what problems do they solve and what are their current limitations?**

**Model Answer:**
Introduced in Go 1.18, generics allow writing functions and types parameterized by type, eliminating the need for `interface{}` casts or code generation for type-generic algorithms.

```go
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

doubled := Map([]int{1, 2, 3}, func(x int) int { return x * 2 })
```

**Constraints** (`~T`, interfaces as constraints):
```go
type Number interface {
    ~int | ~float64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

**Problems solved:**
- Type-safe data structures (trees, heaps, sets) without code generation.
- Generic algorithms (sort, filter, reduce) without `interface{}` boxing.
- Replacing `reflect`-heavy code for common patterns.

**Current limitations:**
1. No generic methods on types (only functions and type declarations can be parameterized).
2. No higher-kinded types.
3. Type inference is limited — sometimes you must specify type parameters explicitly.
4. No specialization — the compiler may not generate optimal code per-type (implementation uses GC shapes, sharing code for pointer types).
5. Interface method sets and constraints interact in non-obvious ways.
6. Compile times increase with heavy generic use.

My philosophy: reach for generics when you find yourself copy-pasting identical logic for different types, but don't over-abstract prematurely — Go's strength is readability.

---

## 4. Memory & Performance

---

**Q15: Explain slices in detail — header structure, capacity growth, and common pitfalls.**

**Model Answer:**
A slice is a three-word struct: `(pointer to array, length, capacity)`. It is a view into an underlying array.

```go
s := make([]int, 3, 6)
// pointer -> [0, 0, 0, 0, 0, 0]
// len = 3, cap = 6
```

**Append and capacity growth:**
When `append` exceeds capacity, the runtime allocates a new, larger backing array and copies. Go's growth strategy: roughly double capacity for small slices, ~1.25x for large ones (exact formula changed in Go 1.18 to be smoother).

**Pitfalls:**

1. **Slice shares backing array — mutation aliasing:**
```go
a := []int{1, 2, 3, 4, 5}
b := a[:3]
b[0] = 99 // also modifies a[0]!
```

2. **Memory retention via slice:**
```go
func firstByte(data []byte) []byte {
    return data[:1] // keeps entire large data slice alive in GC
}
// Fix:
func firstByte(data []byte) []byte {
    result := make([]byte, 1)
    copy(result, data[:1])
    return result
}
```

3. **Unexpected sharing after append:**
```go
a := make([]int, 3, 5)
b := a[:3]
b = append(b, 4) // writes into a's backing array at index 3!
```

4. **Three-index slice** (`a[low:high:max]`) caps the capacity, preventing accidental writes into the original backing array.

---

**Q16: What is `sync.Pool` and when should you use it?**

**Model Answer:**
`sync.Pool` is a pool of temporary objects that can be reused to reduce GC pressure. Objects in the pool may be collected at any GC cycle (there's no guarantee of retention).

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func handler(w http.ResponseWriter, r *http.Request) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)
    // use buf...
}
```

**When to use:**
- Frequently allocated, short-lived, expensive-to-allocate objects: `bytes.Buffer`, large byte slices, encoder/decoder instances.
- You've profiled and confirmed allocation is the bottleneck.

**When NOT to use:**
- Objects with state that must not leak between uses (always reset before putting back).
- Objects with finalizers (they won't run).
- As a general-purpose object cache — `sync.Pool` is not a cache. Objects can vanish on GC. Use a proper LRU or TTL cache for persistent object reuse.

**Critical:** Always `Reset()` or otherwise reinitialize before reuse. I've seen bugs where a pooled `bytes.Buffer` was put back without resetting and the next caller got stale data.

---

**Q17: How does Go handle memory alignment and what are the implications for struct layout?**

**Model Answer:**
The Go compiler aligns struct fields according to their size — an `int64` must start at an 8-byte boundary, an `int32` at 4, etc. Padding bytes are inserted to satisfy alignment.

```go
// Poorly laid out — 24 bytes
type Bad struct {
    a bool    // 1 byte + 7 padding
    b float64 // 8 bytes
    c bool    // 1 byte + 7 padding
}

// Well laid out — 16 bytes
type Good struct {
    b float64 // 8 bytes
    a bool    // 1 byte
    c bool    // 1 byte + 6 padding
}
```

Use `unsafe.Sizeof` or `go vet` tooling (e.g., `fieldalignment` analyzer) to detect wasteful layouts.

**Practical implications:**
- In memory-intensive code (large slices of structs), layout affects cache efficiency significantly.
- False sharing on cache lines: if two goroutines write to fields in the same 64-byte cache line, they thrash each other's L1 cache. The fix is padding to `cache line size` (typically 64 bytes):

```go
type Counter struct {
    value int64
    _     [56]byte // pad to 64-byte cache line
}
```

This is critical in performance-sensitive concurrent code — I've seen 3–5x speedups from fixing false sharing in tight hot loops.

---

## 5. Error Handling

---

**Q18: How does Go's error handling philosophy differ from exceptions, and what are the idiomatic patterns?**

**Model Answer:**
Go treats errors as values — `error` is just an interface:
```go
type error interface {
    Error() string
}
```

Functions return errors explicitly. The caller decides what to do. This is a deliberate rejection of exceptions, which tend to create invisible control flow paths that make it hard to reason about what a function can do.

**Idiomatic patterns:**

1. **Sentinel errors** — comparable errors for expected conditions:
```go
var ErrNotFound = errors.New("not found")
if err == ErrNotFound { ... }
```

2. **Custom error types** — carry context:
```go
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}
```

3. **Error wrapping** (`fmt.Errorf` with `%w`, Go 1.13+):
```go
if err != nil {
    return fmt.Errorf("loading config: %w", err)
}
```

4. **`errors.Is` and `errors.As`** for unwrapping:
```go
if errors.Is(err, ErrNotFound) { ... }

var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Println(valErr.Field)
}
```

**My philosophy:** Add context at every layer (wrap with `%w`), check for specific types/sentinels at decision points, and never swallow errors without at minimum logging them. Panic is reserved for truly unrecoverable programmer errors, not operational failures.

---

**Q19: What is the difference between `panic`/`recover` and errors? When is it appropriate to `panic`?**

**Model Answer:**
`panic` unwinds the stack, running deferred functions, until a `recover` catches it or the program crashes. It is not for ordinary error handling.

**When `panic` is appropriate:**
1. **Programmer errors** — impossible states that indicate a bug, not a runtime condition:
   - Index out of bounds (the runtime panics automatically).
   - Nil pointer dereference.
   - Calling a function that requires initialization that wasn't done.
2. **Package initialization** — if `init()` encounters a state that makes the package unusable, panicking is acceptable.
3. **Unreachable code** in switch/select default branches to catch logic errors.

**`recover` use cases:**
- HTTP servers: recover from panics in handlers to return 500 instead of crashing the process.
- Plugin systems: isolate panics in user-provided code.
- Always re-panic if the value is not an expected type (don't silently swallow unknown panics).

```go
func safeHandler(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("panic: %v\n%s", rec, debug.Stack())
                w.WriteHeader(http.StatusInternalServerError)
            }
        }()
        h.ServeHTTP(w, r)
    })
}
```

The key insight: panics are for bugs; errors are for expected failure modes. Library code should never expose panics to callers for operational conditions — return an error.

---

## 6. Testing

---

**Q20: What testing strategies and tooling does Go provide natively, and what do you layer on top?**

**Model Answer:**
**Native:**
- `testing` package — unit tests (`TestXxx`), benchmarks (`BenchmarkXxx`), examples (`ExampleXxx`), fuzz tests (`FuzzXxx`, Go 1.18+).
- `go test -race` — race detector.
- `go test -cover` — coverage.
- `testing/iotest`, `net/http/httptest`, `testing/fstest` — helpers for common scenarios.
- Table-driven tests — idiomatic Go pattern:

```go
func TestAdd(t *testing.T) {
    cases := []struct {
        a, b, want int
    }{
        {1, 2, 3},
        {0, 0, 0},
        {-1, 1, 0},
    }
    for _, tc := range cases {
        t.Run(fmt.Sprintf("%d+%d", tc.a, tc.b), func(t *testing.T) {
            if got := Add(tc.a, tc.b); got != tc.want {
                t.Errorf("got %d, want %d", got, tc.want)
            }
        })
    }
}
```

**Layered on top:**
- `testify/assert` — cleaner assertions (though I keep its use judicious — the stdlib is often enough).
- `gomock` or `moq` — interface-based mocking.
- `goleak` — goroutine leak detection in tests.
- `httpexpect` or `testcontainers-go` — integration test helpers.
- Property-based testing: `rapid` library.

**My philosophy:** Fast unit tests for logic-heavy code, integration tests with real dependencies (Postgres, Redis) in Docker via `testcontainers`. I avoid mocking at the database layer when possible — integration tests catch far more real bugs. Benchmarks live alongside unit tests and run in CI to catch performance regressions.

---

**Q21: How does fuzzing work in Go 1.18+ and when would you use it?**

**Model Answer:**
Fuzzing is automated adversarial testing — the engine generates random mutations of seed inputs and runs them against your function, looking for panics, crashes, or invariant violations.

```go
func FuzzParseURL(f *testing.F) {
    // Seed corpus
    f.Add("https://example.com/path?q=1")
    f.Add("http://localhost:8080")

    f.Fuzz(func(t *testing.T, input string) {
        u, err := url.Parse(input)
        if err != nil {
            return // errors are fine; panics are not
        }
        // Invariant: re-parsing the string form should give equal result
        u2, err := url.Parse(u.String())
        if err != nil || u.String() != u2.String() {
            t.Fatalf("round-trip failed for %q", input)
        }
    })
}
```

Run with `go test -fuzz=FuzzParseURL -fuzztime=60s`.

**When to use:**
- Parsers (JSON, XML, binary protocols) — extremely high value.
- Cryptographic code.
- Compression/decompression routines.
- Any function that accepts user-supplied bytes — fuzzing finds edge cases humans never think of.

Failing inputs are automatically saved as corpus entries in `testdata/fuzz/` and will be run as regular unit tests on future `go test` invocations.

---

## 7. Concurrency Patterns

---

**Q22: Implement a worker pool in Go.**

**Model Answer:**
```go
func WorkerPool(ctx context.Context, jobs <-chan Job, numWorkers int) <-chan Result {
    results := make(chan Result)

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    select {
                    case results <- process(job):
                    case <-ctx.Done():
                        return
                    }
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

Key points I'd highlight in an interview:
- Workers exit when `jobs` is closed or context is cancelled — no leaks.
- Results channel is closed only after all workers are done (`wg.Wait` in a separate goroutine).
- The double `select` on result send ensures a context cancel doesn't block if the downstream consumer is slow.
- The caller controls backpressure by controlling how fast they consume `results`.

---

**Q23: What is the pipeline pattern and how do you handle cancellation across stages?**

**Model Answer:**
A pipeline chains goroutines as stages, where each stage receives from an input channel and sends to an output channel. Data flows through, each stage transforming it.

```go
func generator(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    for n := range square(ctx, generator(ctx, 1, 2, 3, 4, 5)) {
        fmt.Println(n)
    }
}
```

**Cancellation propagation:** `ctx.Done()` is checked at every send *and* every receive. When `cancel()` is called, all stages drain their input and return, closing their output. Downstream stages then see their input closed and also return. The whole pipeline tears down cleanly.

**Fan-out:** multiple goroutines reading from the same channel. **Fan-in** (merge): one goroutine reading from multiple channels and forwarding to one output. These are composable primitives.

---

## 8. Standard Library & Ecosystem

---

**Q24: How does `context.Context` work? What should and shouldn't be stored in a context?**

**Model Answer:**
`context.Context` carries deadlines, cancellation signals, and request-scoped values across API boundaries and goroutines.

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error  // Canceled or DeadlineExceeded
    Value(key any) any
}
```

**Cancellation chain:** `context.WithCancel`, `WithTimeout`, and `WithDeadline` create child contexts. Cancelling a parent cancels all children. This enables one-call cancellation of entire subtrees of work (e.g., cancelling an HTTP request cancels DB queries and downstream RPC calls spawned to serve it).

**What to store in context values (`context.WithValue`):**
✅ Appropriate: request IDs, trace spans, auth tokens, logger instances — cross-cutting, request-scoped, does not affect function logic.
❌ Inappropriate: database connections, optional function parameters, anything that would be better as a function parameter or struct field.

The `Value` method accepts an `any` key — use unexported types to avoid key collisions:
```go
type contextKey int
const requestIDKey contextKey = iota

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}
func RequestID(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(requestIDKey).(string)
    return id, ok
}
```

**Rule:** Context is always the first parameter, named `ctx`, never stored in a struct.

---

**Q25: Walk me through how you'd design a rate limiter in Go.**

**Model Answer:**
I'd start with the token bucket algorithm — tokens accumulate at a fixed rate up to a burst capacity; each request consumes one token.

**Using `golang.org/x/time/rate`** (the standard approach):
```go
limiter := rate.NewLimiter(rate.Every(time.Second/100), 10) // 100 req/s, burst 10

func (s *Server) handle(w http.ResponseWriter, r *http.Request) {
    if err := limiter.Wait(r.Context()); err != nil {
        http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
        return
    }
    // handle request
}
```

**Per-user rate limiting** (distributed consideration):
```go
type UserLimiter struct {
    mu       sync.Mutex
    limiters map[string]*rate.Limiter
}

func (ul *UserLimiter) Get(userID string) *rate.Limiter {
    ul.mu.Lock()
    defer ul.mu.Unlock()
    if l, ok := ul.limiters[userID]; ok {
        return l
    }
    l := rate.NewLimiter(rate.Every(time.Second), 20)
    ul.limiters[userID] = l
    return l
}
```

**For distributed rate limiting** (multiple service instances): use Redis with sliding window logs or the token bucket pattern via Lua scripts for atomic check-and-decrement. `go-redis` with `Eval` for atomic Lua is the standard approach. Libraries like `go-redis/redis_rate` provide this out of the box.

I'd also design the middleware to return `Retry-After` headers and distinguish between "slow down" (429) and "blocked" (403) scenarios.

---

## 9. Advanced Topics

---

**Q26: What is `unsafe.Pointer` and when would you ever use it?**

**Model Answer:**
`unsafe.Pointer` is a pointer type that bypasses Go's type system. You can convert any pointer to `unsafe.Pointer` and back, and convert `unsafe.Pointer` to `uintptr` (for pointer arithmetic). Operations using it are excluded from Go's memory safety guarantees and are not protected by garbage collector move semantics if done incorrectly.

**Legitimate uses:**
1. **Atomic operations on non-integer types** — `sync/atomic` requires `unsafe.Pointer` for pointer CAS operations.
2. **Zero-copy string↔[]byte conversion** — avoid allocation when you can guarantee immutability:
```go
func StringToBytes(s string) []byte {
    return *(*[]byte)(unsafe.Pointer(
        &struct {
            string
            Cap int
        }{s, len(s)},
    ))
}
```
(Now better done with `unsafe.SliceData` / `unsafe.StringData` in Go 1.20.)

3. **Interfacing with C code** via cgo.
4. **Reading struct fields by offset** in low-level serialization code.

**Rules (`go/vet` and `go tool vet` check these):**
- Never store `uintptr` derived from `unsafe.Pointer` across function calls without converting back — the GC may move objects.
- Only the blessed conversion patterns documented in the `unsafe` package are safe.
- Use `go:linkname` and `unsafe` only as a last resort, document the invariants exhaustively, and wrap it in a single package with a clean safe API.

---

**Q27: How does Go handle reflection, and what are the performance implications?**

**Model Answer:**
Go's `reflect` package allows inspecting and manipulating types and values at runtime. It's built on the interface's `(type, value)` pair and exposes them as `reflect.Type` and `reflect.Value`.

```go
func PrintFields(v interface{}) {
    rv := reflect.ValueOf(v)
    if rv.Kind() == reflect.Ptr {
        rv = rv.Elem()
    }
    rt := rv.Type()
    for i := 0; i < rv.NumField(); i++ {
        fmt.Printf("%s: %v\n", rt.Field(i).Name, rv.Field(i).Interface())
    }
}
```

**Performance implications:**
- Reflection bypasses compiler optimizations — no inlining, no escape analysis.
- Type lookups and value extraction involve hash map lookups internally.
- Method calls via `reflect.Value.Call` are 10–100x slower than direct calls.
- Every `Interface()` call may cause a heap allocation (boxing).

**When to use reflection:**
- Serialization/deserialization frameworks (`encoding/json`, `encoding/gob`).
- Dependency injection containers.
- ORM mapping.
- Test utilities.

**Mitigation:** Cache reflect type information. Libraries like `json-iterator` or `encoding/json` in Go 1.18+ use cached `reflect.Type` lookups and even code generation to avoid repeated reflection costs. For hot paths, generate code at startup or use `go:generate` to eliminate reflection entirely.

---

**Q28: Explain how `defer` works under the hood and the performance cost in tight loops.**

**Model Answer:**
`defer` pushes a function call onto a per-goroutine defer list. When the surrounding function returns (normally or via panic), deferred functions execute LIFO.

**Under the hood (Go 1.14+ optimization):**
Prior to 1.14, every `defer` allocated a heap structure. Since 1.14, the compiler uses **open-coded defers** for most cases — it emits inline code at every return site instead of using the heap-allocated defer chain. This reduces `defer` overhead to near zero for most uses.

```go
// Before Go 1.14: heap allocation per defer call
// After Go 1.14: often compiled to inline code
func readFile(name string) error {
    f, err := os.Open(name)
    if err != nil { return err }
    defer f.Close() // now very cheap
    // ...
}
```

**When open-coded defers don't apply:**
- Defers in loops — the compiler can't statically bound the count.
- Functions with very many defer calls.

```go
// Performance problem: allocates defer frame each iteration
for _, file := range files {
    defer file.Close() // WRONG: also closes on function return, not loop iteration
}

// Correct and efficient:
for _, file := range files {
    func() {
        defer file.Close()
        process(file)
    }()
}
```

**Closure capture with defer:**
```go
for i := 0; i < 3; i++ {
    defer fmt.Println(i) // evaluates i NOW — prints 2, 1, 0
}

for i := 0; i < 3; i++ {
    defer func() { fmt.Println(i) }() // captures reference — prints 3, 3, 3!
}
```

---

## 10. System Design / Architecture in Go

---

**Q29: How would you structure a large Go codebase? What are your opinions on package design?**

**Model Answer:**
My preferred structure for a service:

```
/cmd
  /myservice        # main package — only wires dependencies, no logic
/internal           # private packages — not importable externally
  /domain           # pure domain types and interfaces (no external imports)
  /service          # business logic — depends on domain interfaces
  /repository       # data access implementations
  /handler          # HTTP/gRPC handlers
  /config           # configuration loading
/pkg                # truly reusable, importable packages
/migrations
/scripts
```

**Package design principles:**
1. **Packages are namespaces, not modules** — a package should have a single, coherent purpose. Avoid mega-packages and "util" dumping grounds.
2. **Depend on interfaces, not implementations** — define interfaces in the package that *uses* them (not the package that implements them). This avoids import cycles and makes testing trivial.
3. **Keep `main` thin** — wire dependencies in `main`, nothing else. All logic lives in testable packages.
4. **Use `internal`** to enforce package boundaries at compile time — internal packages cannot be imported by outside modules.
5. **Flat > deep** — prefer flat package hierarchies. Deep nesting (5+ levels) is a code smell.
6. **No circular imports** — Go enforces this. If you have one, you need to extract an interface or merge packages.

The domain layer has no external imports (no database drivers, no HTTP libraries) — just Go types and interfaces. This makes the business logic layer independently testable.

---

**Q30: How would you approach building a high-throughput HTTP service in Go? What are the key bottlenecks to watch for?**

**Model Answer:**
`net/http` is production-ready out of the box — Go's standard library HTTP server handles concurrency, connection pooling, and graceful shutdown well.

**Starting point — `net/http` with good defaults:**
```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}
```
Always set timeouts — the zero value means no timeout, which leads to goroutine and FD leaks under slow clients.

**Key bottlenecks and mitigations:**

1. **JSON serialization** — `encoding/json` uses reflection. For hot paths, use `json-iterator/go` or `easyjson` (code generation). Profile first.

2. **Memory allocation in handlers** — every request allocating large buffers is GC-heavy. Use `sync.Pool` for buffers, pre-allocate response slices.

3. **Database connection pool** — `database/sql` manages a pool; tune `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime` based on observed query latencies.

4. **Context propagation** — ensure DB/RPC calls respect request context deadlines. A slow downstream shouldn't hold a goroutine forever.

5. **Graceful shutdown:**
```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

6. **Profiling** — expose `net/http/pprof` in development; `go tool pprof` heap and CPU profiles are invaluable. CPU profiling shows hot functions; heap profiles show allocation sources.

7. **Middleware design** — keep middleware O(1); avoid per-request allocations in logging/tracing middleware by reusing buffer pools.

For extreme throughput, `fasthttp` is an alternative that avoids some allocations, but it deviates from `net/http` semantics. I'd reach for it only after exhausting optimization of the standard library stack.

---

That covers the core areas a senior Go engineer would be expected to master — language semantics, runtime internals, concurrency patterns, performance, testing, and architecture. The best answers will always tie back to **why** Go made the design choices it did, not just **what** the behavior is.