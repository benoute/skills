# Common Go Antipatterns

---
description: >-
  Concrete antipatterns to flag during Go code reviews. Each entry shows the
  problem, why it matters, and the fix. Organized by category: performance,
  modern idioms, and entropy.
---

## Performance Antipatterns

### 1. Pointer to small value type

**Problem:** Using `*bool`, `*int`, `*string` as struct fields when the zero
value is a valid state.

```go
// BAD: pointer indirection for a boolean, forces heap allocation
type Config struct {
    Verbose *bool
    Retries *int
}

// GOOD: use zero value as default, document the meaning
type Config struct {
    Verbose bool // false means quiet
    Retries int  // 0 means no retries
}
```

When a pointer is genuinely needed to distinguish "not set" from "zero value"
(e.g., JSON `omitempty`), use `new(expr)` (Go 1.26):

```go
config := Config{Verbose: new(true)} // instead of a ptr() helper
```

### 2. Mixed hot and cold struct fields

**Problem:** Frequently accessed fields interleaved with rarely accessed ones.
Each access fetches cache lines full of data you do not need.

```go
// BAD: 1-byte Protocol field buried in a 1520-byte struct
// Every access to Protocol loads 24 cache lines
type Packet struct {
    Protocol  uint8
    Payload   [1500]byte
    SrcPort   uint16
    Timestamp int64
    DstPort   uint16
}

// GOOD: hot fields in a compact struct (16 bytes, 1 cache line)
type PacketHeader struct {
    Timestamp int64
    SrcPort   uint16
    DstPort   uint16
    Protocol  uint8
    _         [3]byte
}
// Cold payload stored separately, only accessed when needed
type PacketPayload struct {
    Data [1500]byte
}
```

This pattern yielded a **9.5x speedup** in a real network driver.

### 3. Struct fields ordered by declaration convenience, not alignment

**Problem:** Padding waste from poorly ordered fields.

```go
// BAD: 24 bytes (8 bytes wasted on padding)
type Bad struct {
    Active bool    // 1 + 7 padding
    ID     int64   // 8
    Count  int32   // 4
    Flag   bool    // 1 + 3 padding
}

// GOOD: 16 bytes (2 bytes of trailing padding only)
type Good struct {
    ID     int64   // 8
    Count  int32   // 4
    Active bool    // 1
    Flag   bool    // 1
    // 2 bytes trailing padding
}
```

Rule: order fields from largest to smallest alignment (8, 4, 2, 1 byte).

### 4. `fmt.Sprintf` in hot paths

**Problem:** `fmt.Sprintf` uses reflection and allocates. In tight loops, this
dominates runtime.

```go
// BAD: allocates on every call
key := fmt.Sprintf("user:%d", userID)

// GOOD: no allocation for simple int-to-string
key := "user:" + strconv.Itoa(userID)

// GOOD: even better for repeated concatenation
var buf strings.Builder
buf.WriteString("user:")
buf.WriteString(strconv.Itoa(userID))
key := buf.String()
```

### 5. Append in a loop without pre-allocation

**Problem:** Each `append` beyond capacity triggers a reallocation and copy.
For N items, this means O(log N) allocations and O(N) total copies.

```go
// BAD: unknown number of reallocations
var results []Result
for _, item := range items {
    results = append(results, process(item))
}

// GOOD: single allocation
results := make([]Result, 0, len(items))
for _, item := range items {
    results = append(results, process(item))
}
```

Pre-allocation is **3.75x faster** for 1000-element slices.

### 6. `sync.Mutex` protecting a single counter or flag

**Problem:** A mutex for a single integer or boolean is heavyweight. Lock
acquisition involves goroutine scheduling overhead.

```go
// BAD: mutex for a single counter
type Stats struct {
    mu      sync.Mutex
    counter int64
}
func (s *Stats) Inc() {
    s.mu.Lock()
    s.counter++
    s.mu.Unlock()
}

// GOOD: atomic operation, no lock
type Stats struct {
    counter atomic.Int64
}
func (s *Stats) Inc() {
    s.counter.Add(1)
}
```

Use `atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]` (Go 1.19+).

### 7. Channel where index partitioning suffices

**Problem:** Sending N results through a channel when you could pre-allocate
`make([]Result, N)` and have each goroutine write to `results[i]`. The channel
version requires goroutine coordination, channel internal allocation, and
runtime scheduling overhead. It also loses input-output ordering.

```go
// BAD: unnecessary channel, order not preserved
ch := make(chan Result, len(items))
for _, item := range items {
    go func(item Item) { ch <- process(item) }(item)
}
results := make([]Result, 0, len(items))
for range len(items) {
    results = append(results, <-ch)
}

// GOOD: no synchronization, order preserved for free
results := make([]Result, len(items))
var wg sync.WaitGroup
for i, item := range items {
    wg.Go(func() {
        results[i] = process(item)
    })
}
wg.Wait()
```

Use channels when work count is dynamic, streaming, or fan-in/fan-out is
genuinely needed.

### 8. False sharing in concurrent code

**Problem:** Multiple goroutines writing to adjacent memory locations that
share a 64-byte cache line. The CPU bounces the line between cores on every
write.

```go
// BAD: both atomics on the same cache line
type Counters struct {
    Reads  atomic.Int64 // bytes 0-7
    Writes atomic.Int64 // bytes 8-15, same cache line
}

// GOOD: each counter on its own cache line
type Counters struct {
    Reads  atomic.Int64
    _padR  [56]byte     // pad to 64-byte boundary
    Writes atomic.Int64
    _padW  [56]byte
}
```

Per-CPU/per-goroutine counters with cache-line padding were **8.9x faster**
than shared atomic counters across 8 cores.

## Modern Idiom Antipatterns

### 9. Manual `errors.As` with target variable

**Problem:** `errors.As` requires declaring a target variable, uses reflection,
and can panic at runtime if the target type is wrong.

```go
// BAD: verbose, uses reflection, runtime panic risk
var connErr *net.OpError
if errors.As(err, &connErr) {
    handleConnError(connErr)
}

// GOOD: type-safe, no reflection, compile-time checked (Go 1.26)
if connErr, ok := errors.AsType[*net.OpError](err); ok {
    handleConnError(connErr)
}
```

`errors.AsType` is 3x faster than `errors.As` and catches type mistakes at
compile time.

### 10. Custom `min`/`max` helper functions

**Problem:** Hand-rolled `minInt`, `maxFloat64`, etc. that duplicate a builtin.

```go
// BAD: unnecessary code
func minInt(a, b int) int {
    if a < b { return a }
    return b
}
result := minInt(x, y)

// GOOD: builtin, works with any ordered type (Go 1.21)
result := min(x, y)
```

Delete the helpers. `min`/`max` work with all ordered types.

### 11. `b.N` loop in benchmarks

**Problem:** The classic `for i := 0; i < b.N; i++` pattern runs the entire
benchmark function multiple times to calibrate `b.N`. Expensive setup/teardown
runs repeatedly. The compiler can also optimize away the loop body.

```go
// BAD: setup runs multiple times, compiler may eliminate work
func BenchmarkProcess(b *testing.B) {
    data := expensiveSetup() // runs many times
    for i := 0; i < b.N; i++ {
        process(data)
    }
}

// GOOD: setup runs once, results kept alive (Go 1.24)
func BenchmarkProcess(b *testing.B) {
    data := expensiveSetup() // runs once
    for b.Loop() {
        process(data)
    }
}
```

### 12. Manual WaitGroup boilerplate

**Problem:** `wg.Add(1)` + `go func() { defer wg.Done(); ... }` is verbose
and error-prone (forgetting `Add` or `Done`).

```go
// BAD: 5 lines of boilerplate per goroutine
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func(item Item) {
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()

// GOOD: 1 line per goroutine (Go 1.25)
var wg sync.WaitGroup
for _, item := range items {
    wg.Go(func() { process(item) })
}
wg.Wait()
```

## Entropy Antipatterns

### 13. Interface with single implementation

**Problem:** An interface exists but only one type implements it. No test
doubles use it. It adds a layer of indirection for zero benefit.

```go
// BAD: interface with one implementation, never mocked
type UserRepository interface {
    FindByID(id string) (*User, error)
    Save(u *User) error
}
type postgresUserRepo struct { db *sql.DB }

// GOOD: use the concrete type directly
type UserRepository struct { db *sql.DB }
func (r *UserRepository) FindByID(id string) (*User, error) { ... }
func (r *UserRepository) Save(u *User) error { ... }
```

If a caller only needs one method, accept a small interface at the call site.
Do not create a God-interface upfront.

### 14. Getter that returns a field with no logic

**Problem:** A method that returns a struct field with no validation,
computation, or side effects. Pure boilerplate.

```go
// BAD: 3 lines that add nothing
func (s *Server) Port() int { return s.port }

// GOOD: export the field
type Server struct {
    Port int
}
```

Add getters/setters only when there is validation, side effects, or the value
is computed.

### 15. Thin wrapper with no added logic

**Problem:** A function that delegates to another function with the same
signature. Adds indirection and cognitive overhead.

```go
// BAD: wrapper that does nothing
func SendMessage(client *Client, msg string) error {
    return client.Send(msg)
}

// GOOD: callers call client.Send(msg) directly
```

### 16. Builder pattern where a struct literal suffices

**Problem:** Builder types add 30-50 lines of boilerplate for something a
struct literal does in 5 lines.

```go
// BAD: 40+ lines of builder
s := NewServerBuilder().
    WithAddr(":8080").
    WithTimeout(30 * time.Second).
    Build()

// GOOD: 4 lines, same result
s := &Server{
    Addr:    ":8080",
    Timeout: 30 * time.Second,
}
```

Functional options are warranted for public library APIs with complex defaults.
For internal code, struct literals are almost always sufficient.

### 17. Premature generics

**Problem:** A generic function or type that is only instantiated with one
type. Adds cognitive overhead and type parameter noise for no code reduction.

```go
// BAD: generic with one instantiation
func Transform[T any](items []T, fn func(T) T) []T { ... }
// only ever called as Transform[string](...)

// GOOD: concrete until a second type appears
func TransformStrings(items []string, fn func(string) string) []string { ... }
```

Generics should delete duplicate code, not anticipate hypothetical future
types.
