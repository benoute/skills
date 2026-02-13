# Review Checklist

---
description: >-
  Structured pass-by-pass checklist for reviewing Go code. Use this as a
  systematic guide to ensure nothing is missed. Not every item applies to
  every review -- skip items that are irrelevant to the change.
---

## Performance

### Data layout

- [ ] Struct fields ordered by descending alignment to minimize padding?
- [ ] Hot fields (accessed in tight loops) grouped at the top of the struct?
- [ ] Cold fields (config, metadata, error details) separated from hot fields?
- [ ] Hot structs fit within 64 bytes (one cache line)?
- [ ] SoA layout considered for hot loops touching few fields across many items?
- [ ] Data structure appropriate for actual dataset size? (Linear scan for
      < 50-100 elements? Sorted slice instead of map for read-heavy workloads?)

### Allocations

- [ ] Slices pre-allocated with `make([]T, 0, n)` when size is known or
      estimatable?
- [ ] No `fmt.Sprintf` in hot paths? (Use `strconv` functions instead.)
- [ ] No string concatenation with `+` in loops? (Use `strings.Builder`.)
- [ ] No unnecessary heap escapes? (Small values returned by value, not pointer.)
- [ ] `sync.Pool` considered for high-churn allocations?
- [ ] Buffers reused where possible instead of reallocating?

### Concurrency

- [ ] Concurrent code: could shared state be eliminated via index-partitioned
      result slices?
- [ ] Channels used only when work count is dynamic or streaming?
- [ ] Buffered channels sized appropriately (not unbuffered by default)?
- [ ] `sync.Mutex` not used where `atomic` would suffice for a single field?
- [ ] False sharing assessed for tight concurrent loops with small result types?
- [ ] No goroutine leak risks? (Unbuffered channels + early returns, missing
      context cancellation, blocked sends with no receiver.)
- [ ] `sync.WaitGroup.Go` used instead of manual `Add`/`Done` boilerplate?

### Benchmarks

- [ ] Performance-motivated design choices backed by benchmark tests?
- [ ] Benchmarks use `b.Loop()` (not raw `b.N` loops)?
- [ ] Benchmarks call `b.ReportAllocs()`?
- [ ] Benchmarks test realistic data sizes, not toy inputs?
- [ ] Both alternatives benchmarked side by side in the same test file?

## Modern Go Idioms

### Language features

- [ ] `any` used instead of `interface{}`?
- [ ] `for i := range n` instead of `for i := 0; i < n; i++`?
- [ ] `min`/`max` builtins instead of custom helper functions?
- [ ] `clear` builtin instead of manual zeroing loops?
- [ ] `new(expr)` for optional pointer fields instead of helper functions?
- [ ] Generics used to eliminate duplicate code (not for premature abstraction)?

### Standard library

- [ ] `errors.AsType[*T](err)` instead of `errors.As(err, &target)`?
- [ ] `slices.SortFunc` instead of `sort.Slice`?
- [ ] `slices.BinarySearchFunc` instead of `sort.Search`?
- [ ] `slices.Contains` / `slices.Index` instead of hand-rolled loops?
- [ ] `maps.Keys` / `maps.Values` / `maps.Clone` instead of manual iteration?
- [ ] `cmp.Or(x, default)` instead of `if x == zero { x = default }`?
- [ ] `log/slog` instead of `log.Printf` for structured logging?
- [ ] `sync.OnceValue` / `sync.OnceFunc` instead of `sync.Once` + globals?
- [ ] `sync.WaitGroup.Go` instead of `wg.Add(1); go func(){ defer wg.Done() }`?
- [ ] `runtime.AddCleanup` instead of `runtime.SetFinalizer`?
- [ ] `math/rand/v2` instead of `math/rand`?
- [ ] Reflective iterators (`.Fields()`, `.Methods()`) instead of index loops?

### Tooling

- [ ] `go fix ./...` suggested for automated modernization?
- [ ] `go vet` / `staticcheck` passing?

## Comments

### Presence

- [ ] All exported types, functions, methods, and constants have doc comments?
- [ ] Doc comments start with the symbol name?
- [ ] Exported struct fields have comments (especially for units, invariants,
      valid ranges)?
- [ ] Non-obvious code paths have why-comments?
- [ ] Performance-motivated code shapes are annotated with the reasoning and
      benchmark evidence?
- [ ] Magic numbers, buffer sizes, and timeout values are explained?
- [ ] Concurrency invariants are documented (which goroutine owns what)?

### Absence

- [ ] No "what" comments on obvious code? (`// close the connection` above
      `conn.Close()`.)
- [ ] No commented-out code blocks?
- [ ] No stale comments that do not match the code?
- [ ] No redundant comments that repeat function/type names?

### Structure

- [ ] Functions longer than ~30 lines have section headers?
- [ ] Interface contracts document what implementations must guarantee?

## Entropy

### Types and interfaces

- [ ] No single-use interfaces (only one implementation, no test double)?
- [ ] No interfaces with 5+ methods (probably a concrete type in disguise)?
- [ ] No types used in only one place that could be inlined?
- [ ] No types that differ in only 1-2 fields (could be unified)?

### Functions and packages

- [ ] No thin wrappers that add no logic (just delegate to another function)?
- [ ] No getters/setters for fields with no validation or side effects?
- [ ] No packages with 1-2 files that could be folded into parent?
- [ ] No packages with only one exported symbol?

### Dead code

- [ ] No unused exported symbols?
- [ ] No unreachable branches?
- [ ] No TODO comments older than 6 months without associated issues?
- [ ] No commented-out code?

### Change hygiene

- [ ] Change does not leave obsolete code behind?
- [ ] Old helper functions replaced by this change are deleted?
- [ ] Types or interfaces that lost their only consumer are removed?
- [ ] Test utilities that no longer apply are cleaned up?
