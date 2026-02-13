---
name: go-nerd-write
description: "Use when writing, generating, or implementing Go code — functions, types, handlers, services, or any .go files. Produces performant, minimal Go with data-oriented design, proper struct layout, cache-line awareness, and modern Go 1.26 idioms. Biases toward fewer types, fewer allocations, and useful comments that explain why."
---

# Writing Go Code

The goal is the smallest, fastest Go code that a future engineer can understand
without you standing next to them.

Three pillars:

1. **Performance through data orientation.** Think about how data flows through
   CPU caches, not just algorithmic complexity. A cache-friendly O(n) can
   destroy a cache-hostile O(1).
2. **Minimalism through entropy reduction.** Less code, fewer types, fewer
   abstractions. Measure the end state, not the effort.
3. **Clarity through purposeful comments.** Explain *why*, never *what*. Document
   tradeoffs, not syntax.

For reviewing existing Go code, see the companion **go-review** skill.

## Before Writing

**Load at least one reference from `references/`.**

1. List the files in the `references/` directory.
2. Read the one(s) most relevant to the task at hand.
3. State which you loaded and its core principle.

**Do not proceed until you have done this.** The references contain concrete
numbers and examples that should inform your design decisions.

## The Three Questions

Before writing any new type, function, or package, ask:

### 1. What is the smallest set of types and functions that solves this?

Not "what's the cleanest architecture" -- what's the smallest *result*.

- Could this be 2 functions instead of 14?
- Could this be 0 functions (delete the feature)?
- Does a built-in or stdlib type already do this?

### 2. Does this result in fewer allocations and less total code?

Count lines and allocations before and after. If after > before, reject it.

- "Better organized" but more code = more entropy.
- "More flexible" but more allocations = slower.
- "Cleaner separation" but more packages = more entropy.

### 3. What existing code can we delete, inline, or simplify?

Every change is an opportunity to delete. Ask:

- What does this make obsolete?
- What was only needed because of what we are replacing?
- What is the maximum we could remove?

## Struct Design

Struct layout is a performance decision, not just an organizational one.

**Group fields by access pattern:**

- **Hot fields first** -- fields accessed together in tight loops go at the top.
- **Cold fields last** -- config, metadata, debug info go at the bottom.
- Within each group, **order fields by descending alignment** (8-byte, 4-byte,
  2-byte, 1-byte) to minimize padding.

**Fit hot structs in a single cache line (64 bytes)** when possible. Every cache
line fetch that doesn't contribute useful data is wasted bandwidth.

**Prefer value types over pointers for small structs.** Fewer indirections =
better cache behavior. A `*T` where `T` is 16 bytes adds a pointer chase for
no benefit.

**SoA vs AoS:** When iterating many items but touching few fields, separate
slices per field (Structure of Arrays) can give 2-2.5x speedups over a slice of
structs (Array of Structures). When accessing all fields of one item, keep them
together. Choose based on the actual access pattern.

See `references/cache-lines-and-data-layout.md` for the full mental model,
concrete numbers, and examples.

## Allocation Awareness

- **Pre-allocate slices** with `make([]T, 0, n)` when the length is known or
  estimatable. This alone can yield 3-4x speedups over grow-on-demand.
- **Prefer stack allocation.** Small values, avoid taking the address of locals
  unnecessarily (forces heap escape).
- **Use `strings.Builder`** or `[]byte` buffer for string concatenation in loops.
- **Avoid `fmt.Sprintf` in hot paths** -- use `strconv.Itoa`, `strconv.AppendInt`,
  `strconv.FormatFloat`, etc.
- **`sync.Pool`** for frequently allocated/freed objects of known lifetime.
- **Batch processing** -- processing N items at a time amortizes per-item
  overhead and improves prefetcher behavior. The book "Data Structures in
  Practice" shows 1.6x speedup from batching 32 items vs 1 at a time.

## Concurrency Without Sharing

The hierarchy, from most preferred to least:

### 1. Pre-allocated result slice with index partitioning

When you know the work items upfront (which is common), allocate the output
slice once and assign each goroutine a non-overlapping index range. No
synchronization needed at all. Fastest and simplest to reason about.

```go
results := make([]Result, len(items))
var wg sync.WaitGroup
for i, item := range items {
    wg.Go(func() {
        results[i] = process(item) // no lock, no channel -- disjoint indices
    })
}
wg.Wait()
// results is fully populated, order preserved for free
```

**Watch for false sharing:** if `sizeof(Result)` is small (< 64 bytes),
adjacent goroutines writing to `results[i]` and `results[i+1]` contend on the
same cache line. Mitigations:

- Batch index ranges so each goroutine handles `[start, end)`, not one index.
- Have each goroutine accumulate locally and write once at the end.
- Pad result elements to cache line boundaries if the loop is truly hot.

### 2. Channels

When work count is dynamic, streaming, or fan-in/fan-out is needed. Use
buffered channels sized to expected output count. Simple and idiomatic.

### 3. `atomic` operations

When sharing a single counter or flag. Use `atomic.Int64`, `atomic.Bool`, etc.
(Go 1.19+). Beware false sharing if multiple atomics are in the same struct --
pad to 64-byte boundaries.

### 4. `sync.Mutex`

Last resort. Complex shared state that cannot be partitioned. Keep critical
sections as small as possible.

**The key insight:** if you can partition the problem space, you can partition
the output space, and then there is nothing to synchronize.

## Modern Go Idioms

Target **Go 1.26**. Use modern features where they reduce code or improve
correctness. Do not use them for novelty.

High-impact changes to prefer:

- `errors.AsType[*T](err)` over `errors.As(err, &target)` -- type-safe, faster,
  no reflection (1.26).
- `new(expr)` for optional pointer fields -- `Fed: new(true)` instead of a
  helper function (1.26).
- Range-over-func iterators (`iter.Seq`, `iter.Seq2`) where they simplify
  iteration (1.23+).
- `slices`, `maps`, `cmp` packages over hand-rolled equivalents (1.21+).
- `min`/`max` builtins over custom functions (1.21+).
- `log/slog` for structured logging (1.21+).
- `b.Loop()` for benchmarks instead of `for i := 0; i < b.N; i++` (1.24+).
- `sync.WaitGroup.Go` instead of manual `Add`/`Done` with goroutines (1.25+).
- Run `go fix` to auto-apply modernizers (1.26).

Use generics when they eliminate duplicate code, not for premature abstraction.
If a generic function is only used with one type today, make it concrete.

See `references/modern-go-features.md` for the complete version-tagged list
from Go 1.18 through 1.26.

## Comments That Matter

### Why, not what

Never `// increment i` or `// close the connection`. Always explain the
reasoning behind non-obvious decisions.

### Struct fields

Document every exported struct field. For unexported fields: document if the
meaning is not obvious from the name and type. Include units, invariants, valid
ranges.

### Performance annotations

When code is shaped by performance requirements, say so:

```go
// Fields ordered by descending alignment to minimize padding.
// Hot fields (accessed per-request) are grouped first.
type Request struct {
    Timestamp int64   // 8 bytes -- hot
    Size      int32   // 4 bytes -- hot
    Flags     uint16  // 2 bytes -- hot
    Priority  uint8   // 1 byte  -- hot
    // -- cold fields below, accessed only during error handling --
    TraceID   [16]byte
    UserAgent string
}
```

### Benchmark-backed decisions

When code is shaped by a benchmark result, document the evidence:

```go
// SoA layout chosen over AoS after benchmarking with 10K particles:
//   BenchmarkSoA: 480 ns/op, 0 allocs/op
//   BenchmarkAoS: 1150 ns/op, 0 allocs/op
// See particle_test.go for the benchmark.
type Particles struct {
    X, Y, Z    []float64
    VX, VY, VZ []float64
}
```

This is the gold standard for performance comments: the *why* (cache locality),
the *evidence* (benchmark numbers), and the *proof location* (test file).

### Section comments

In functions longer than ~30 lines, use section headers to delineate logical
phases:

```go
func (s *Server) handleRequest(ctx context.Context, req *Request) error {
    // --- Validate and normalize inputs ---
    ...

    // --- Route to handler ---
    ...

    // --- Write response ---
    ...
}
```

### Interface contracts

Document what implementations must guarantee, not just the method signatures.

### Delete stale comments

A wrong comment is worse than no comment. If the code changed but the comment
did not, delete or update the comment.

## When in Doubt, Benchmark

When choosing between two approaches and you cannot determine which will perform
better from first principles alone, **write a benchmark test before committing
to either.** This applies to:

- SoA vs AoS layout for a specific workload
- `sync.Pool` vs direct allocation
- Map vs sorted slice for a given dataset size
- Channel vs index-partitioned result slice
- Pre-allocated buffer size tradeoffs
- Any struct layout change (padding, field reordering, hot/cold split)

The benchmark should:

- Use `testing.B` with `b.Loop()` (Go 1.24+).
- Test with realistic data sizes, not toy inputs.
- Report allocations with `b.ReportAllocs()`.
- Compare both approaches side by side in the same `_test.go` file.

```go
func BenchmarkMapLookup(b *testing.B) {
    m := buildMap(1000)
    b.ResetTimer()
    for b.Loop() {
        _ = m[key]
    }
}

func BenchmarkSliceScan(b *testing.B) {
    s := buildSortedSlice(1000)
    b.ResetTimer()
    for b.Loop() {
        _ = slices.BinarySearchFunc(s, key, cmp)
    }
}
```

**Never ship a "performance optimization" without a benchmark proving it is
faster.** Intuition about cache behavior is notoriously unreliable.

## Red Flags

- **"We need a new type for this"** -- Could a built-in or existing type work?
- **"Let's add an interface"** -- Does it have more than one implementation?
  Will it ever?
- **"This needs a builder pattern"** -- Does it? Or does a struct literal
  suffice?
- **"Better organized" but more code** -- More files/types/packages = more
  entropy.
- **"Let's wrap this"** -- Thin wrappers that add no logic add only
  indirection.
- **"For performance"** -- Where is the benchmark? No benchmark, no claim.

## When This Does Not Apply

- Prototyping / throwaway code.
- Code generation output.
- Test helpers where clarity trumps performance.
- Code in a framework with strong conventions (do not fight it).
- Regulatory / compliance requirements that mandate certain structures.

## Reference Materials

See `references/` for detailed guidance:

- `cache-lines-and-data-layout.md` -- The cache-line mental model, struct
  alignment, hot/cold separation, SoA vs AoS, false sharing, concrete numbers.
- `modern-go-features.md` -- Complete version-tagged list of modern Go features
  from 1.18 through 1.26.
- `entropy-for-go.md` -- Reducing-entropy principles adapted specifically to
  Go idioms.
