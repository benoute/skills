---
name: go-nerd-review
description: "Use when reviewing, auditing, or analyzing existing Go code — PRs, diffs, functions, files, or packages. Checks for performance issues (data layout, allocations, concurrency), missed modern Go idioms, unnecessary complexity and dead code, and comment quality."
---

# Reviewing Go Code

Review for the **end state**, not just the diff. Every review is an opportunity
to delete code, improve data layout, and modernize idioms.

Do not just check "is the new code correct?" -- ask "is the codebase smaller
and faster after this change?"

For detailed writing guidance, see the companion **go-write** skill.

## Review Process

Make multiple passes, each focused on one concern. Do not mix style nits with
architectural feedback.

1. **Performance** -- data layout, allocations, concurrency patterns.
2. **Modern Go** -- missed opportunities to use newer features.
3. **Comments** -- missing why-comments, useless what-comments, stale comments.
4. **Entropy** -- unnecessary types, interfaces, abstractions, dead code.

Lead with the most impactful finding. If the code is fundamentally the wrong
approach, say so before commenting on details.

## Pass 1: Performance

### Data layout

- **Struct field ordering:** Are fields ordered by descending alignment to
  minimize padding? Would reordering save memory? Use `unsafe.Sizeof` to
  check.
- **Hot/cold separation:** In performance-critical structs, are frequently
  accessed fields grouped at the top? Are rarely accessed fields (debug info,
  config, error details) separated?
- **SoA vs AoS:** For hot-loop iteration over many items touching few fields,
  would separate slices per field (Structure of Arrays) outperform the current
  slice of structs?
- **Cache line fit:** Do hot structs fit within 64 bytes (one cache line)?
  Exceeding 64 bytes means every access costs two cache line fetches.
- **Data structure choice:** Could a "simpler" structure (sorted slice, flat
  array) outperform a "fancier" one (map, tree) given the actual dataset size?
  For < 50-100 elements, linear scan through a slice often beats a map.

See the **go-write** skill's `references/cache-lines-and-data-layout.md` for
concrete numbers and benchmarks.

### Allocations

- **Hot-path allocations:** `fmt.Sprintf`, string concatenation with `+` in
  loops, `append` without pre-allocation, unnecessary `new`/`make`.
- **Slice pre-allocation:** Is `make([]T, 0, n)` used when the length is known
  or estimatable?
- **Escape analysis:** Are small values unnecessarily forced to the heap by
  taking their address? Would returning by value instead of pointer avoid an
  allocation?
- **`sync.Pool`:** For high-churn allocations (buffers, temporary objects), is
  `sync.Pool` considered?

### Concurrency

- **Shared state under mutex:** Could the problem be partitioned instead? If
  goroutines each process a known subset of items, a pre-allocated result slice
  with index partitioning eliminates all synchronization.
- **Channel where a result slice would do:** If the number of results is known
  upfront and each goroutine produces exactly one result, index partitioning is
  simpler and faster than collecting from a channel.
- **Unbuffered channel as synchronization:** If the consumer waits for all
  producers, `sync.WaitGroup` + result slice is usually clearer and faster.
- **`sync.Mutex` protecting a single field:** Use `atomic.Int64`, `atomic.Bool`,
  etc. (Go 1.19+).
- **False sharing risk:** Goroutines writing to adjacent small elements in a
  shared slice may cause cache-line contention. Flag when element size < 64
  bytes and the loop is performance-critical.
- **Goroutine leaks:** Unbuffered channels with early returns, missing
  cancellation via context, blocked sends with no receiver. Use the goroutine
  leak profile (`GOEXPERIMENT=goroutineleakprofile`, Go 1.26) in tests.

### Unsubstantiated performance claims

If code was structured in a non-obvious way "for performance," is there a
benchmark proving it? Performance-motivated code should have:

1. A comment explaining the tradeoff.
2. A benchmark test validating the assumption.

No benchmark = no performance claim. Flag it.

## Pass 2: Modern Go

Flag old patterns that have modern replacements. For each finding, cite the Go
version that introduced the replacement so the author can verify compatibility.

### Common modernizations

| Old pattern | Modern replacement | Since |
|---|---|---|
| `interface{}` | `any` | 1.18 |
| `atomic.AddInt64(&x, 1)` | `x.Add(1)` (atomic types) | 1.19 |
| `sort.Slice(s, less)` | `slices.SortFunc(s, cmp)` | 1.21 |
| `sort.Search(n, f)` | `slices.BinarySearchFunc(s, t, cmp)` | 1.21 |
| Custom `min`/`max` helpers | `min(a, b)` / `max(a, b)` builtins | 1.21 |
| `if x == "" { x = d }` | `cmp.Or(x, d)` | 1.21 |
| `log.Printf` (structured data) | `slog.Info(msg, key, val)` | 1.21 |
| `sync.Once` + separate func | `sync.OnceValue(func() T { ... })` | 1.21 |
| `for i := 0; i < n; i++` | `for i := range n` | 1.22 |
| `rand.Intn(n)` | `rand.IntN(n)` (math/rand/v2) | 1.22 |
| Index-based reflect iteration | Reflective iterators (`.Fields()`, `.Methods()`) | 1.26 |
| `runtime.SetFinalizer` | `runtime.AddCleanup` | 1.24 |
| `for i := 0; i < b.N; i++` | `for b.Loop()` | 1.24 |
| `wg.Add(1); go func(){ defer wg.Done(); ... }` | `wg.Go(func() { ... })` | 1.25 |
| `errors.As(err, &target)` | `errors.AsType[*T](err)` | 1.26 |
| `v := x; p = &v` (pointer to value) | `p = new(x)` | 1.26 |

### Automated modernization

Suggest running `go fix ./...` (Go 1.26) to auto-apply many of these
transformations. This catches patterns that are tedious to spot manually.

### Generics opportunities

- Are there 2+ functions with identical logic differing only in types? They
  should be a single generic function.
- Conversely, is a generic function only instantiated with one type? It should
  be concrete.

## Pass 3: Comments

### Missing why-comments

Flag non-obvious code paths, performance-motivated decisions, workarounds, and
tricky algorithms that lack explanation. If a reviewer has to ask "why is this
done this way?" the code needs a comment.

Specific things that always need a why-comment:

- Struct field ordering choices motivated by alignment or cache behavior.
- Use of `unsafe` or `//go:linkname`.
- Intentional deviation from the obvious approach.
- Error handling that swallows or transforms errors non-obviously.
- Magic numbers, buffer sizes, timeout values.
- Concurrency invariants (which goroutine owns what, ordering guarantees).

### Useless what-comments

Flag comments that restate the code:

```go
// BAD
// Close the connection
conn.Close()

// Increment the counter
counter++

// Return the error
return err
```

These add noise and maintenance burden with zero information value. Suggest
deletion.

### Exported symbols

Every exported type, function, method, and constant should have a doc comment
starting with the symbol name. This is not just convention -- it is what
`go doc` and `pkg.go.dev` display.

### Struct field documentation

Exported struct fields should have comments, especially when the name alone
does not convey semantics. Document units, invariants, valid ranges, and
relationships between fields.

```go
type Config struct {
    // MaxRetries is the maximum number of retry attempts before giving up.
    // Zero means no retries. Negative values are treated as zero.
    MaxRetries int

    // Timeout is the per-request timeout. Includes connection and read time.
    Timeout time.Duration
}
```

### Stale comments

Flag comments that do not match the code they describe. A wrong comment is
worse than no comment.

### Section comments

In functions longer than ~30 lines, suggest section headers if logical phases
are not delineated. The reader should be able to scan section headers to
understand the function's flow without reading every line.

## Pass 4: Entropy

### Type count

- Could types be merged? Is there a type only used in one place?
- Are there types that differ only in one or two fields? Could they be unified
  with an optional field or a type parameter?

### Interface audit

- Does each interface have 2+ implementations?
- Is the interface tested directly, or only through a concrete type?
- Does the interface have 5+ methods? It is probably a concrete type in
  disguise.

### Package structure

- Are there packages with 1-2 files that could be folded into their parent?
- Are there packages with only one exported symbol?
- Does the package name echo the parent (`server/serverutil`)?

### Dead code

- Unused exported symbols (use `go vet` / `staticcheck`).
- Unreachable branches.
- TODO comments older than 6 months with no associated issue.
- Commented-out code blocks.

### Abstraction layers

- Is there a function that just calls another function? Inline it.
- Is there a type that wraps another type with no added behavior? Delete it.
- Is there a getter/setter pair for a field with no validation or side effects?
  Export the field directly.

### Obsoleted code

Did this change make something obsolete that was not cleaned up? Check for:

- Old helper functions replaced by new ones.
- Types or interfaces that lost their only consumer.
- Test utilities that no longer apply.

## Output Format

Free-form review comments. Lead each finding with its category:

- `[perf]` -- Performance issue (data layout, allocations, concurrency).
- `[modern]` -- Modern Go feature opportunity. Always cite the Go version.
- `[comment]` -- Comment quality issue (missing, useless, stale).
- `[entropy]` -- Unnecessary code, abstraction, or complexity.

Include `file:line` references for each finding.

For modernization suggestions, always cite the minimum Go version required so
the author can verify compatibility with their `go.mod`.

End with a brief summary: total findings by category and overall assessment.

## When This Does Not Apply

- Generated code (protobuf, stringer, etc.).
- Vendored dependencies.
- Test fixtures and test data files.
- Code explicitly marked as legacy/deprecated with a migration plan and
  tracking issue.

## Reference Materials

See `references/` for detailed checklists and antipatterns:

- `review-checklist.md` -- Structured pass-by-pass checklist.
- `common-antipatterns.md` -- Concrete antipatterns with bad/good examples.

For cache-line details and modern Go features, see the companion **go-write**
skill's `references/` directory.
