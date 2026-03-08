# Sync Primitives Internals

> Go's sync package provides low-level synchronization — each primitive has a carefully optimized internal structure.

![[Sync-internals.excalidraw]]

---

## sync.Mutex

```go
type Mutex struct {
    state int32   // locked|woken|starving|waiterCount
    sema  uint32  // semaphore for blocking
}
```

- State packs 4 things into one int32: locked bit, woken bit, starvation bit, waiter count (upper 29 bits)
- **Normal mode**: goroutines compete via spinning and semaphore — new arrivals can steal lock
- **Starvation mode**: after 1ms of waiting, switch to FIFO — hand lock directly to longest-waiting goroutine
- Spinning: active wait on multi-core, avoids semaphore overhead for short critical sections

---

## sync.RWMutex

```go
type RWMutex struct {
    w           Mutex   // held for writes
    writerSem   uint32  // semaphore for writers
    readerSem   uint32  // semaphore for readers
    readerCount atomic.Int32  // number of active readers (negative = writer waiting)
    readerWait  atomic.Int32  // departing readers when writer waiting
}
```

- Multiple readers OR one writer
- Writer sets readerCount to negative → new readers block
- Writer waits for existing readers (tracked by readerWait) to finish
- Readers unblock writer when readerWait reaches 0

---

## sync.WaitGroup

```go
type WaitGroup struct {
    state atomic.Uint64  // counter(high 32) + waiter(low 32)
    sema  uint32
}
```

- `Add(n)` increments counter, `Done()` decrements (= `Add(-1)`)
- `Wait()` increments waiter count and blocks on semaphore
- When counter hits 0: wake all waiters
- Panic if counter goes negative

---

## sync.Once

```go
type Once struct {
    done atomic.Uint32
    m    Mutex
}
```

- Fast path: `atomic.Load(&done) == 1` → return immediately (no lock!)
- Slow path: lock mutex, check again, call function, `atomic.Store(&done, 1)`
- Guarantees function runs exactly once, even with concurrent calls

---

## sync.Pool

- Per-P private and shared pools (reduces contention)
- `Get()`: check private → check shared → check other P's shared → call New
- `Put()`: store in private (one slot) or shared
- GC clears pools every cycle — NOT for long-lived caching
- Great for: temporary buffers, encoders/decoders, fmt.Fprintf

---

## sync.Map

```go
type Map struct {
    mu    Mutex
    read  atomic.Pointer[readOnly]  // lock-free reads
    dirty map[any]*entry
    misses int
}
```

- Two maps: `read` (lock-free, atomic) and `dirty` (mutex-protected)
- Reads: check `read` first (no lock), fallback to `dirty` (with lock)
- After enough misses: promote `dirty` → `read`
- Best for: append-only workloads, keys written once but read many times
- NOT a general replacement for `map` + `Mutex`

---

## Related

- [[Channel]] — higher-level synchronization
- [[Goroutine]] — scheduler interaction with sync primitives
