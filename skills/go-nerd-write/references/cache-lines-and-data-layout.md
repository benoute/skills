# Cache Lines and Data Layout

---
description: >-
  The cache-line mental model for writing performant Go. Covers memory
  hierarchy latencies, struct alignment, hot/cold field separation, SoA vs
  AoS, false sharing, and concrete benchmarks showing that data layout
  often matters more than algorithmic complexity.
---

## The 100-Cycle Rule

Every memory access that misses the CPU cache costs roughly 100x more than a
cache hit. This single fact drives most performance-sensitive design decisions.

| Level     | Latency       | Relative |
|-----------|---------------|----------|
| Register  | 1 cycle       | 1x       |
| L1 cache  | 3-4 cycles    | 3x       |
| L2 cache  | 12-15 cycles  | 12x      |
| L3 cache  | 40-50 cycles  | 40x      |
| DRAM      | 100-200 cycles| 100x     |

A single cache miss costs more than 100 register operations. When your code is
memory-bound (most Go code is), optimizing memory access patterns matters more
than reducing operation count.

## The Cache Line: 64 Bytes

The CPU never fetches a single byte. It fetches an entire **64-byte cache
line**. This has profound implications:

- Accessing `array[0]` loads bytes 0-63. The next 15 `int32` accesses are free.
- Accessing a 128-byte struct but touching only one 8-byte field wastes 94%
  of the fetched data.
- Sequential access enables the hardware prefetcher to fetch ahead. Random
  access (pointer chasing) defeats it.

**Design data layouts to use all 64 bytes of every cache line you fetch.**

## Big-O Is Necessary but Insufficient

Algorithmic complexity tells you how work scales. It says nothing about the
constant factor, which is dominated by cache behavior. Real-world examples from
"Data Structures in Practice" (D. Jiang):

| Scenario | "Better" algorithm | "Worse" algorithm | Winner | Speedup |
|----------|-------------------|-------------------|--------|---------|
| 500-entry lookup | Hash table O(1) | Linear scan O(n) | Linear scan | 3x |
| 500-entry lookup | Hash table O(1) | Binary search O(log n) | Binary search | 1.4x |
| 10K symbol lookup | Red-Black tree O(log n) | Sorted array O(log n) | Sorted array | 3x |
| 1M entry lookup | Red-Black tree O(log n) | B-tree O(log n) | B-tree (order 64) | 6.7x |
| Priority queue | Red-Black tree O(log n) | Array heap O(log n) | Array heap | 4x |
| Sequential sum | Linked list O(n) | Array O(n) | Array | 2.5x |
| Random access | Linked list O(n) | Array O(1) | Array | 30x |

The pattern: **contiguous, sequential, cache-friendly data structures beat
pointer-chasing structures**, even at the same or worse big-O.

For small N (< 50-100 elements), linear scan through a slice often beats a map.
The entire working set fits in L1/L2 cache and there is zero hashing overhead.

**When unsure which approach is faster, write a benchmark.** Do not guess.

## Struct Field Alignment and Padding

Go aligns struct fields to their natural alignment. Poorly ordered fields waste
memory to padding:

```go
// BAD: 24 bytes (with padding)
type Bad struct {
    A bool    // 1 byte
    // 7 bytes padding
    B int64   // 8 bytes
    C bool    // 1 byte
    // 3 bytes padding
    D int32   // 4 bytes
}

// GOOD: 16 bytes (no wasted padding)
type Good struct {
    B int64   // 8 bytes
    D int32   // 4 bytes
    A bool    // 1 byte
    C bool    // 1 byte
    // 2 bytes trailing padding
}
```

**Rule: order fields from largest to smallest alignment.** Group 8-byte fields,
then 4-byte, then 2-byte, then 1-byte.

Verify with `unsafe.Sizeof` and `unsafe.Alignof`:

```go
fmt.Println(unsafe.Sizeof(Bad{}))  // 24
fmt.Println(unsafe.Sizeof(Good{})) // 16
```

The `fieldalignment` analyzer (via `go vet` or `golang.org/x/tools`) can
detect sub-optimal ordering automatically.

## Hot/Cold Field Separation

Not all struct fields are accessed equally. Separate fields by access frequency:

```go
// BEFORE: 1520-byte struct, iterating to check 1-byte protocol field
// fetches 24 cache lines per packet
type Packet struct {
    Protocol  uint8
    SrcPort   uint16
    DstPort   uint16
    Payload   [1500]byte
    Checksum  uint32
    Timestamp int64
}

// AFTER: hot metadata (16 bytes, fits in 1 cache line)
// cold payload stored separately
type PacketHeader struct {
    Timestamp int64    // hot -- checked every packet
    SrcPort   uint16   // hot
    DstPort   uint16   // hot
    Protocol  uint8    // hot
    _         [3]byte  // padding for alignment
}

type PacketPayload struct {
    Data     [1500]byte
    Checksum uint32
}
```

A network driver applying this pattern went from 100K packets/sec to 950K
packets/sec -- a **9.5x improvement** from data layout alone.

**Keep hot structs within a single cache line (64 bytes).** If your
most-accessed struct is 48 bytes, every access is one cache line fetch. If it
is 72 bytes, every access is two fetches.

## SoA vs AoS

**Array of Structures (AoS)** -- typical Go slice of structs:

```go
type Particle struct {
    X, Y, Z    float64
    VX, VY, VZ float64
    Mass       float64
    ID         int64
}
particles := make([]Particle, 10000)
```

Each cache line loads one particle (64 bytes = 1 particle). If you only need
positions (X, Y, Z), you fetch 64 bytes but use only 24 -- 37.5% utilization.

**Structure of Arrays (SoA)** -- separate slice per field:

```go
type Particles struct {
    X, Y, Z    []float64
    VX, VY, VZ []float64
    Mass       []float64
    ID         []int64
}
```

Each cache line loads 8 consecutive X values. If you are updating positions,
every byte is useful -- 100% utilization.

**Benchmark (1M particles, 1000 iterations):**
- AoS: 2,850 ms
- SoA: 1,200 ms (**2.4x faster**)

**When to use SoA:** operations access few fields across many items, hot loops,
SIMD-friendly workloads.

**When to use AoS:** operations access all fields of one item, small
collections, code clarity matters.

**When unsure:** benchmark both with your actual access pattern.

## False Sharing

When two goroutines write to memory locations in the **same 64-byte cache
line**, the CPU cache coherency protocol (MESI) bounces that line between
cores on every write. This is called false sharing and it serializes concurrent
code silently.

```go
// BAD: counterA and counterB share a cache line.
// Two goroutines writing to them will ping-pong the line between cores.
type Counters struct {
    CounterA atomic.Int64  // bytes 0-7
    CounterB atomic.Int64  // bytes 8-15, same cache line
}

// GOOD: each counter gets its own cache line.
// "Wasting" 112 bytes of padding makes concurrent code 5-10x faster.
type Counters struct {
    CounterA atomic.Int64
    _padA    [56]byte      // 64 - 8 = 56 bytes padding
    CounterB atomic.Int64
    _padB    [56]byte
}
```

The book "Data Structures in Practice" shows per-CPU counters (each on its own
cache line) were **8.9x faster** than a shared atomic counter across 8 cores.

False sharing also applies to pre-allocated result slices. If `sizeof(Result)`
is 8 bytes, goroutines writing `results[i]` and `results[i+1]` contend on the
same cache line. Mitigations:

- Batch index ranges: each goroutine handles `[start, end)`, not one index.
- Accumulate locally, write once at the end.
- Pad result elements to cache line boundaries for truly hot loops.

## Pre-allocate and Reuse

```go
// BAD: grows on demand, multiple reallocations
var results []Result
for _, item := range items {
    results = append(results, process(item))
}

// GOOD: single allocation, no copies
results := make([]Result, 0, len(items))
for _, item := range items {
    results = append(results, process(item))
}
```

Pre-allocation with `make([]T, 0, n)` is **3.75x faster** than grow-on-demand
for 1000-element slices. It avoids repeated reallocation and copying.

Use `sync.Pool` for objects that are frequently allocated and freed with known
lifetime patterns. This avoids GC pressure for high-churn allocations.

## Key Numbers to Remember

| Fact | Number |
|------|--------|
| Cache line size | 64 bytes |
| L1 cache hit | 3-4 cycles |
| DRAM access (cache miss) | 100-200 cycles |
| Array vs linked list traversal | 2.5x faster |
| Array vs linked list random access | 30x faster |
| SoA vs AoS (hot loop, few fields) | 2.4x faster |
| Hot/cold field separation (packet headers) | 9.5x faster |
| Pre-allocated vs grow-on-demand (1K elements) | 3.75x faster |
| Per-CPU counters vs shared atomic (8 cores) | 8.9x faster |
| Blocked Bloom filter vs naive (cache-aligned) | 5.3x faster |
| Sorted array vs Red-Black tree (read-heavy) | 3x faster |
| B-tree (order 64) vs Red-Black tree | 6.7x faster |
| Batch processing (32 items vs 1 at a time) | 1.6x faster |

## Always Benchmark

Never trust intuition about cache behavior. **Measure.**

- `go test -bench=. -benchmem` for Go benchmarks.
- `go tool pprof` for CPU and allocation profiling.
- `perf stat -e cache-misses,cache-references` on Linux for hardware cache
  counters.
- Compare approaches side by side in the same `_test.go` file.

## Further Reading

- [Data Structures in Practice](https://github.com/djiangtw/data-structures-in-practice-public) -- D. Jiang. Concrete benchmarks of cache-aware vs cache-hostile data structures.
- [CppCon 2014: Mike Acton "Data-Oriented Design and C++"](https://www.youtube.com/watch?v=rX0ItVEVjHc) -- The foundational talk on data-oriented design.
- [Go Performance Wiki](https://go.dev/wiki/Performance) -- Official Go performance resources.
