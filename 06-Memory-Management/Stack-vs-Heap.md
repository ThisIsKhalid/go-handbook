# Stack vs Heap

> Every Go value lives on either the stack or the heap. The compiler's escape analysis decides — and it matters for performance.

![[Stack-vs-Heap-internals.excalidraw]]

---

## Stack

- Per-goroutine, starts at **2KB**, grows dynamically (up to 1GB default max)
- **LIFO** allocation — push on function entry, pop on return
- Zero overhead: no GC involvement
- Stack-allocated values are fastest — allocation is just a pointer bump
- Stack can **grow** (2x reallocation + copy) and **shrink** (during GC, if using < 1/4)

```go
func stackOnly() int {
    x := 42        // stack — doesn't escape
    return x        // copied to caller's frame
}
```

---

## Heap

- Shared across all goroutines
- Managed by Go's memory allocator (`mcache` -> `mcentral` -> `mheap`)
- Subject to garbage collection (GC pressure)
- Required when value **outlives function scope**
- Slower: allocation requires finding free space, deallocation requires GC

```go
func heapEscape() *int {
    x := 42        // heap — escapes via pointer return
    return &x       // compiler detects escape
}
```

---

## What Goes Where?

The compiler performs **escape analysis** at compile time to decide placement:

```go
func stackOnly() int {
    x := 42        // stack — doesn't escape
    return x        // copied to caller's frame
}

func heapEscape() *int {
    x := 42        // heap — escapes via pointer return
    return &x       // compiler detects escape
}
```

| Scenario | Location | Why |
|----------|----------|-----|
| Local variable, no pointers shared | Stack | Lifetime bounded by function |
| Returned pointer | Heap | Outlives function scope |
| Assigned to interface | Heap | Compiler can't prove size at compile time |
| Closure captures by reference | Heap | Variable shared across scopes |
| Sent to channel | Heap | Crosses goroutine boundary |
| Too large for stack | Heap | Exceeds stack frame budget |

---

## Memory Allocator

Go's memory allocator is a hierarchy designed to minimize lock contention:

- **mcache**: per-P cache, no locking needed — fastest path
- **mcentral**: shared cache for each size class — one lock per size class
- **mheap**: manages large allocations and OS memory — global lock
- **Size classes**: 67 classes from 8 bytes to 32KB
- **Large objects** (>32KB): allocated directly from mheap

```
Goroutine needs memory
        |
        v
    mcache (per-P, lock-free)
        |  miss
        v
    mcentral (per-size-class lock)
        |  miss
        v
    mheap (global, request from OS via mmap)
```

### Size Classes

```
Class  0:      0 bytes  (special)
Class  1:      8 bytes
Class  2:     16 bytes
Class  3:     24 bytes
Class  4:     32 bytes
...
Class 66:  32768 bytes (32KB)
> 32KB:   large object, straight to mheap
```

Objects are rounded up to the next size class. A 17-byte allocation uses the 24-byte class (class 3).

---

## Stack vs Heap: Performance Impact

```go
// Benchmark: stack allocation
func BenchmarkStack(b *testing.B) {
    for i := 0; i < b.N; i++ {
        x := 42  // stack
        _ = x
    }
}

// Benchmark: heap allocation
func BenchmarkHeap(b *testing.B) {
    for i := 0; i < b.N; i++ {
        x := new(int)  // heap (escapes)
        _ = x
    }
}
// Stack: ~0.3 ns/op, 0 allocs
// Heap:  ~8 ns/op, 1 alloc  (25x slower + GC pressure)
```

---

## Related

- [[Escape-Analysis]] — how the compiler decides stack vs heap
- [[GC]] — garbage collection for heap objects
- [[Goroutine]] — each goroutine has its own stack
