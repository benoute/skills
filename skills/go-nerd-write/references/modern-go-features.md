# Modern Go Features (1.18 -- 1.26)

---
description: >-
  Version-tagged list of modern Go features from the last 5 years. For each
  feature: what it replaces, when to use it, and the minimum go directive
  required. Target Go 1.26.
---

Use these features where they reduce code or improve correctness. Do not use
them for novelty. Each entry notes the minimum `go` directive needed in
`go.mod`.

## Go 1.18 (March 2022)

**Generics** -- Type parameters on functions and types.
- `any` replaces `interface{}` everywhere.
- `comparable` constraint for map keys.
- Use generics to eliminate duplicate code, not for premature abstraction.
- If a generic function is only instantiated with one type, keep it concrete.

**Fuzzing** -- `testing.F` and `(*testing.F).Fuzz` for fuzz testing.

## Go 1.19 (August 2022)

**Atomic types** -- `atomic.Int32`, `atomic.Int64`, `atomic.Uint32`,
`atomic.Uint64`, `atomic.Bool`, `atomic.Pointer[T]`.
- Replace `atomic.AddInt64(&x, 1)` with `x.Add(1)`.
- Type-safe, no need for `&` operator, harder to misuse.

**Doc comment revision** -- Links, lists, and headings in doc comments.

## Go 1.20 (February 2023)

**`errors.Join`** -- Combine multiple errors without third-party libraries.
```go
return errors.Join(err1, err2, err3)
```

**`context.WithCancelCause`** -- Cancel with a reason.
```go
ctx, cancel := context.WithCancelCause(parent)
cancel(fmt.Errorf("upstream timeout"))
// later: context.Cause(ctx) returns the reason
```

**`unsafe.SliceData`**, **`unsafe.StringData`**, **`unsafe.String`** --
Low-level slice/string manipulation without `reflect.StringHeader` hacks.

## Go 1.21 (August 2023)

**`log/slog`** -- Structured logging in the standard library.
- Replace `log.Printf("user=%s action=%s", user, action)` with
  `slog.Info("action", "user", user, "action", action)`.
- Supports JSON and text handlers out of the box.

**`slices` package** -- `slices.Sort`, `slices.SortFunc`, `slices.BinarySearch`,
`slices.BinarySearchFunc`, `slices.Contains`, `slices.Compact`, `slices.Index`,
`slices.Clone`, `slices.Grow`, `slices.Delete`, `slices.Insert`, etc.
- Replace `sort.Slice` with `slices.SortFunc`.
- Replace `sort.Search` with `slices.BinarySearchFunc`.
- Replace hand-rolled contains/index loops.

**`maps` package** -- `maps.Keys`, `maps.Values`, `maps.Clone`, `maps.Equal`,
`maps.Copy`, `maps.DeleteFunc`, `maps.Collect`.

**`cmp` package** -- `cmp.Or` (first non-zero value), `cmp.Compare`,
`cmp.Ordered` constraint.
- `cmp.Or(env, "default")` replaces `if env == "" { env = "default" }`.

**`min`/`max` builtins** -- Work with any ordered type.
- Delete custom `minInt`, `maxFloat64`, etc. helper functions.

**`clear` builtin** -- Zero a map or slice.

**`sync.OnceFunc`**, **`sync.OnceValue`**, **`sync.OnceValues`** -- Type-safe
once wrappers.
```go
// Before
var initOnce sync.Once
var client *http.Client
func getClient() *http.Client {
    initOnce.Do(func() { client = &http.Client{Timeout: 5 * time.Second} })
    return client
}

// After
var getClient = sync.OnceValue(func() *http.Client {
    return &http.Client{Timeout: 5 * time.Second}
})
```

## Go 1.22 (February 2024)

**Range over integers** -- `for i := range n` replaces `for i := 0; i < n; i++`.

**Loop variable per-iteration semantics** -- Each iteration of a `for` loop
gets its own copy of the loop variable. The classic goroutine-in-loop bug is
fixed. No more `v := v` inside the loop body.

**`math/rand/v2`** -- New random number generator. `rand.IntN(n)` replaces
`rand.Intn(n)`. `rand.N[T]` for generic random values.

**`net/http.ServeMux` pattern matching** -- `GET /api/users/{id}` patterns
directly in the default mux. May eliminate the need for third-party routers
in simple cases.

## Go 1.23 (August 2024)

**Range-over-func iterators** -- `iter.Seq[V]` and `iter.Seq2[K, V]` types.
Functions that yield values via callbacks can be used in `for...range` loops.
```go
// Before
fields := reflect.TypeOf(s)
for i := 0; i < fields.NumField(); i++ {
    field := fields.Field(i)
    use(field)
}

// After (with Go 1.26 reflect iterators, concept from 1.23)
for field := range reflect.TypeOf(s).Fields() {
    use(field)
}
```

**`unique` package** -- Interning/canonicalization for any comparable type.
Useful for deduplicating strings or reducing memory in long-lived maps.

**`structs.HostLayout`** -- Embed in a struct to guarantee host-compatible
memory layout for FFI/cgo interop.

**Iterator methods across stdlib** -- `slices.All`, `slices.Values`,
`slices.Backward`, `slices.Collect`, `slices.Chunk`, `maps.Keys`,
`maps.Values`, `maps.All`, `maps.Collect`, etc.

## Go 1.24 (February 2025)

**`testing.B.Loop`** -- The correct way to write benchmark loops.
```go
// Before (error-prone, compiler can optimize away)
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        foo()
    }
}

// After (runs once per -count, results kept alive)
func BenchmarkFoo(b *testing.B) {
    for b.Loop() {
        foo()
    }
}
```

**`weak` package** -- Weak pointers for cache-friendly data structures.
`weak.Pointer[T]` does not prevent garbage collection.

**`runtime.AddCleanup`** -- Preferred over `runtime.SetFinalizer`. Multiple
cleanups per object, works with interior pointers, no cycle leaks.

**`os.Root`** -- Directory-scoped filesystem access. Prevents path traversal
attacks without manual validation.

**Swiss Table maps** -- The default `map` implementation uses Swiss Tables.
Better performance for modifications, especially concurrent access to disjoint
keys.

**`maphash.Comparable`** -- Hash any comparable value. Enables building custom
hash-based structures.

**`encoding.TextAppender`/`BinaryAppender`** -- Append-based marshaling
interfaces. Avoid allocations when serializing to an existing buffer.

**Generic type aliases** -- `type MySlice[T any] = []T` now works.

**`sync.Map` performance** -- Rewritten with a hash trie. Disjoint key access
is much less likely to contend.

## Go 1.25 (August 2025)

**`testing/synctest`** -- Testing concurrent code with virtualized time.
Graduated from experimental. `synctest.Test` runs code in an isolated bubble
where time is fake and advances only when all goroutines block.

**`sync.WaitGroup.Go`** -- Combine `Add(1)` + `go func() { defer Done() ... }`
into one call.
```go
// Before
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func(item Item) {
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()

// After
var wg sync.WaitGroup
for _, item := range items {
    wg.Go(func() { process(item) })
}
wg.Wait()
```

**`runtime/trace.FlightRecorder`** -- Lightweight continuous tracing with
ring buffer. Snapshot last N seconds on significant events.

**`encoding/json/v2`** -- Experimental new JSON implementation. Faster
decoding, `omitzero` tag option. Enable with `GOEXPERIMENT=jsonv2`.

**`net/http.CrossOriginProtection`** -- Built-in CSRF protection using Fetch
metadata.

**Container-aware `GOMAXPROCS`** -- Defaults to cgroup CPU limit on Linux.
Runtime updates it dynamically.

**`reflect.TypeAssert`** -- Convert `reflect.Value` to a Go value without
allocation via `Value.Interface()`.

## Go 1.26 (February 2026)

**`new(expr)`** -- Allocate and initialize in one expression.
```go
// Before: need a helper or temporary variable
v := true
p := &v
// or: func ptr[T any](v T) *T { return &v }

// After
p := new(true)
cat := Cat{Name: "Mittens", Fed: new(true)}
```

**`errors.AsType[E error](err) (E, bool)`** -- Type-safe, generic `errors.As`.
- No reflection, 3x faster, no runtime panics from wrong target types.
- Compile-time error if `E` doesn't implement `error`.
- Recommended drop-in replacement for `errors.As` in new code.

**Recursive type constraints** -- Generic types can reference themselves in
constraints. Enables self-referential patterns like `Ordered[T Ordered[T]]`.

**`go fix` modernizers** -- Automated code modernization. Rewrites old patterns
to use newer APIs. Run `go fix ./...` to apply.

**Green Tea garbage collector** -- Default in 1.26. Memory-aware scanning of
8 KiB spans, better locality, CPU scalability. 10-40% GC overhead reduction.

**`slog.NewMultiHandler`** -- Fan-out to multiple log handlers.

**`testing.T.ArtifactDir`** -- Directory for test artifacts (screenshots,
generated files) that survive test completion.

**Reflective iterators** -- `reflect.Type.Fields()`, `reflect.Type.Methods()`,
`reflect.Type.Ins()`, `reflect.Type.Outs()`, `reflect.Value.Fields()`,
`reflect.Value.Methods()`. Use in `for...range` loops.

**`bytes.Buffer.Peek`** -- Read next N bytes without advancing the buffer.

**Faster cgo** -- ~30% reduction in cgo call overhead.

**Faster small-object allocation** -- Size-specialized `mallocgc`. Up to 30%
faster for objects 1-512 bytes.

**Optimized `fmt.Errorf`** and **`io.ReadAll`** -- Reduced allocations.

**Goroutine leak profile** -- Experimental. `pprof.Lookup("goroutineleak")`
reports goroutines blocked on unreachable sync objects. Enable with
`GOEXPERIMENT=goroutineleakprofile`.

**`simd/archsimd`** -- Experimental. Low-level SIMD operations on amd64.
Enable with `GOEXPERIMENT=simd`.

**`runtime/secret`** -- Experimental. Run functions in secret mode that zeros
registers and stack on completion. For cryptographic key handling. Enable with
`GOEXPERIMENT=runtimesecret`.

**`crypto/hpke`** -- Hybrid Public Key Encryption (RFC 9180).

**Reader-less cryptography** -- Most `crypto/*` functions now ignore the
`io.Reader` parameter and use the system random source. Pass `nil` for rand.

## Migration Quick Reference

| Old pattern | Modern replacement | Since |
|---|---|---|
| `interface{}` | `any` | 1.18 |
| `atomic.AddInt64(&x, 1)` | `x.Add(1)` (atomic types) | 1.19 |
| `sort.Slice(s, less)` | `slices.SortFunc(s, cmp)` | 1.21 |
| `sort.Search(n, f)` | `slices.BinarySearchFunc(s, t, cmp)` | 1.21 |
| Custom `min`/`max` helpers | `min(a, b)` / `max(a, b)` builtins | 1.21 |
| `if x == "" { x = d }` | `cmp.Or(x, d)` | 1.21 |
| `log.Printf` structured | `slog.Info(msg, key, val)` | 1.21 |
| `sync.Once` + separate func | `sync.OnceValue(func() T { ... })` | 1.21 |
| `for i := 0; i < n; i++` | `for i := range n` | 1.22 |
| `v := v` in loop (capture fix) | (automatic per-iteration copy) | 1.22 |
| `rand.Intn(n)` | `rand.IntN(n)` (math/rand/v2) | 1.22 |
| Index-based field/method iteration | Range-over-func iterators | 1.23 |
| `runtime.SetFinalizer` | `runtime.AddCleanup` | 1.24 |
| `for i := 0; i < b.N; i++` | `for b.Loop()` | 1.24 |
| `wg.Add(1); go func() { defer wg.Done(); ... }` | `wg.Go(func() { ... })` | 1.25 |
| `errors.As(err, &target)` | `errors.AsType[*T](err)` | 1.26 |
| `v := val; ptr = &v` | `ptr = new(val)` | 1.26 |
| Manual code modernization | `go fix ./...` | 1.26 |
