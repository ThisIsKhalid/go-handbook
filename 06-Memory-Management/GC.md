# Garbage Collection

> Go uses a concurrent, tri-color mark-and-sweep collector with sub-millisecond pauses.

![[GC-internals.excalidraw]]

---

## Tri-Color Algorithm

The GC categorizes every heap object into one of three colors:

- **White**: not yet visited — candidates for collection
- **Gray**: visited but children not yet scanned
- **Black**: visited and all children scanned

### Process

1. **Start**: all objects are white
2. **Root scan**: GC roots (globals, stacks, registers) are marked gray
3. **Mark loop**: pick a gray object, scan its pointers, mark referenced objects gray, mark the scanned object black
4. **Complete**: no gray objects remain — all white objects are unreachable garbage

```
White ──(discovered)──> Gray ──(scanned)──> Black

Remaining White = Garbage → Swept
```

---

## GC Phases

1. **Mark Setup** (STW): enable write barrier (~10-30us)
2. **Marking** (concurrent): traverse object graph, uses 25% of available CPU
3. **Mark Termination** (STW): finalize marking (~10-30us)
4. **Sweeping** (concurrent): reclaim white objects, runs concurrently with the mutator

### Timeline

```
  [STW]         [Concurrent]         [STW]        [Concurrent]
Mark Setup ──> Marking (25% CPU) ──> Mark Term ──> Sweeping
  ~10-30μs      bulk of GC work       ~10-30μs     lazy/background
```

- **STW** (Stop-The-World): all goroutines paused — kept extremely short
- **Concurrent phases**: application continues running alongside GC
- Total STW time is typically **< 1ms** even for large heaps

---

## Write Barrier

Required because marking runs **concurrently** with the program (mutator):

- **Problem**: while GC is marking, the program might move a pointer from a black object to a white object — the white object would never be scanned and would be incorrectly freed
- **Solution**: Dijkstra-style write barrier — when storing a pointer, shade the new target gray
- **Invariant**: a black object must never point directly to a white object (tri-color invariant)
- **When active**: only during the GC marking phase — zero cost when GC is not marking

```go
// Conceptual write barrier (compiler-inserted)
func writeBarrier(slot *unsafe.Pointer, new unsafe.Pointer) {
    shade(new)  // mark new target gray so GC will scan it
    *slot = new
}
```

---

## GC Pacing

The GC pacer decides **when** to start the next GC cycle:

- `GOGC=100` (default): trigger GC when heap grows to **2x** the size after last GC
- `GOGC=200`: trigger at **3x** — less frequent GC, more memory usage
- `GOGC=50`: trigger at **1.5x** — more frequent GC, less memory usage
- `GOGC=off`: disable GC entirely (dangerous, memory grows unbounded)

### GOMEMLIMIT (Go 1.19+)

```go
// Soft memory limit — GC adapts pacing to stay under the limit
GOMEMLIMIT=1GiB  // GC becomes more aggressive as memory approaches limit
```

- Works alongside `GOGC`, not a replacement
- Prevents OOM in memory-constrained environments (containers)
- GC will run more frequently to respect the limit

### Runtime Tuning

```go
import "runtime/debug"

debug.SetGCPercent(50)          // equivalent to GOGC=50
debug.SetMemoryLimit(1 << 30)   // 1GB soft limit
runtime.GC()                    // force immediate GC cycle
```

---

## GC Roots

The starting points for the mark phase:

- **Global variables**: package-level vars that contain pointers
- **Goroutine stacks**: each goroutine's stack is scanned using compiler-generated **stack maps**
- **Registers**: CPU registers that may hold pointers

### Stack Maps

The compiler generates metadata for each function describing which stack slots and registers contain pointers at each safe point. This allows precise (non-conservative) scanning.

---

## GC Metrics and Debugging

```go
// Runtime stats
var stats runtime.MemStats
runtime.ReadMemStats(&stats)
fmt.Printf("Heap alloc: %d MB\n", stats.HeapAlloc/1024/1024)
fmt.Printf("Num GC: %d\n", stats.NumGC)
fmt.Printf("GC pause: %v\n", stats.PauseNs[(stats.NumGC+255)%256])

// Trace
GODEBUG=gctrace=1 ./myapp
// Output: gc 1 @0.012s 2%: 0.019+0.42+0.024 ms clock, ...
```

### Reading gctrace Output

```
gc 5 @1.234s 3%: 0.018+1.2+0.021 ms clock, 0.072+0.8/1.1/0+0.084 ms cpu, 4->5->2 MB, 5 MB goal
    ^          ^   ^     ^   ^                                         ^    ^    ^
    |          |   |     |   mark term STW                             |    |    live after
    |          |   |     marking (concurrent)                          |    heap at mark
    |          |   mark setup STW                                      heap at GC start
    |          CPU% used by GC
    GC cycle number
```

---

## Reducing GC Pressure

- **Pre-allocate**: `make([]T, 0, n)` to avoid repeated growth
- **Reuse objects**: `sync.Pool` for temporary allocations
- **Value semantics**: avoid unnecessary pointers (fewer objects to scan)
- **Reduce pointer-heavy structures**: arrays of structs vs structs of arrays
- **Tune GOGC**: higher value = fewer GC cycles (trade memory for CPU)
- **Set GOMEMLIMIT**: prevent OOM while letting GC adapt

---

## Related

- [[Stack-vs-Heap]] — where allocations happen
- [[Escape-Analysis]] — what causes heap allocations
- [[Goroutine]] — goroutine stacks are GC roots
