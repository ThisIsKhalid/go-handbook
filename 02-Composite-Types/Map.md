# Map Internals

![[Map-internals.excalidraw]]

> Go maps are hash tables with a clever bucket-based design -- understanding the internals explains why maps aren't concurrent-safe and can't be compared.

---

## The hmap Struct

Every `map` variable is a pointer to an `hmap` struct (defined in `runtime/map.go`). This is why maps are **reference types** -- assigning a map copies the pointer, not the data.

```go
type hmap struct {
    count     int            // number of elements (len() returns this)
    flags     uint8          // iterator/write flags
    B         uint8          // log2 of number of buckets (can hold 2^B buckets)
    noverflow uint16         // approximate overflow bucket count
    hash0     uint32         // hash seed (randomized at creation)
    buckets   unsafe.Pointer // array of 2^B buckets
    oldbuckets unsafe.Pointer // previous bucket array (non-nil during growth)
    nevacuate  uintptr       // evacuation progress counter
    extra      *mapextra     // overflow bucket pointers
}
```

**Key takeaways:**

- `count` is what `len(m)` returns -- O(1) because it's stored directly.
- `B` determines the number of buckets: `2^B`. A map with `B=3` has 8 buckets.
- `hash0` is a random seed assigned at map creation -- this is why iteration order is randomized.
- `buckets` and `oldbuckets` coexist during incremental growth (more below).
- `flags` tracks whether an iterator is active or a goroutine is writing -- this is how the runtime detects concurrent map access.

```go
m := make(map[string]int)
// m is a *hmap under the hood
// m == nil is false, but count == 0
```

---

## Bucket Structure (bmap)

Each bucket is a `bmap` struct that holds up to **8 key-value pairs**:

```go
// Compile-time generated layout for bmap
type bmap struct {
    tophash [8]uint8     // top byte of hash for each slot
    // Followed in memory by:
    // keys     [8]KeyType
    // values   [8]ValueType
    // overflow *bmap
}
```

### Why 8?

The number 8 was chosen as a sweet spot -- small enough to scan linearly (cache-friendly), large enough to keep chains short.

### Memory Layout Within a Bucket

Keys and values are stored in **separate arrays** rather than interleaved `key1/value1/key2/value2`. This eliminates padding waste for differently-sized types:

```
Interleaved (wasteful):        Separated (Go's approach):
| key1 (1 byte) |              | key1 | key2 | ... | key8 |
| pad  (7 bytes)|              | val1 | val2 | ... | val8 |
| val1 (8 bytes)|
| key2 (1 byte) |              Saves 7 bytes of padding per entry
| pad  (7 bytes)|              for map[int8]int64
| val2 (8 bytes)|
```

### tophash Array

Each slot's `tophash` entry stores the **top 8 bits** of the full hash. This allows fast rejection -- if the tophash doesn't match, there's no need to compare the full key.

Special tophash values (0-4) indicate slot status:

```go
emptyRest  = 0  // this cell is empty, and no more non-empty cells above
emptyOne   = 1  // this cell is empty
evacuatedX = 2  // entry evacuated to first half of new table
evacuatedY = 3  // entry evacuated to second half of new table
evacuatedEmpty = 4 // cell is empty, bucket has been evacuated
minTopHash = 5  // minimum normal tophash value
```

### Overflow Pointer

When a bucket is full (all 8 slots occupied), new entries go into an **overflow bucket** -- a new `bmap` linked from the overflow pointer at the end of the struct. This forms a chain.

---

## Hash & Lookup

![[Map-growth.excalidraw]]

### Hash Function Selection

Go picks a hash function based on key type and CPU capabilities:

- **AES-NI available** (most modern x86): uses AES-based hash -- hardware-accelerated, fast and well-distributed.
- **No AES**: falls back to a platform-specific hash (e.g., wyhash on arm64).
- **String keys**: specialized string hash function.
- **Integer keys**: specialized integer hash.
- **Struct/array keys**: combines hashes of each field/element.

### Lookup Process

Given `m[key]`, the runtime:

1. **Hash the key**: `hash = hashfn(key, hmap.hash0)`
2. **Select bucket**: `bucket = hash & (2^B - 1)` (low B bits)
3. **Compute tophash**: `top = uint8(hash >> (64 - 8))` (high 8 bits)
4. **Scan bucket**:
   - Compare `top` against each `tophash[i]` in the bucket
   - On tophash match, compare full key with `keys[i]`
   - On full key match, return `values[i]`
5. **Follow overflow chain**: repeat step 4 for each overflow bucket
6. **During growth**: also check `oldbuckets` if the bucket hasn't been evacuated yet

```go
// Simplified pseudocode for map lookup
func mapaccess(h *hmap, key string) *ValueType {
    hash := hashfn(key, h.hash0)
    bucket := hash & bucketMask(h.B)
    top := tophash(hash)

    b := (*bmap)(add(h.buckets, bucket * bucketSize))
    for ; b != nil; b = b.overflow {
        for i := 0; i < 8; i++ {
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest {
                    return nil // no more entries
                }
                continue
            }
            if b.keys[i] == key {
                return &b.values[i]
            }
        }
    }
    return nil
}
```

### Lookup Performance

- **Average case**: O(1) -- most keys are found in the first bucket within 1-2 tophash comparisons.
- **Worst case**: O(n) -- degenerate hash collisions, but the growth policy prevents this.
- The tophash optimization avoids expensive full key comparisons ~97% of the time (255/256 chance of tophash mismatch for non-matching keys).

---

## Growth & Evacuation

Maps grow when either condition is met:

### Condition 1: Overloaded -- Load Factor > 6.5

```
load factor = count / (2^B)
```

When the average number of entries per bucket exceeds **6.5**, the map doubles in size:

- `B` increments by 1 (buckets double: `2^B` -> `2^(B+1)`)
- All entries must be redistributed

### Condition 2: Too Many Overflow Buckets

If there are too many overflow buckets (roughly as many as regular buckets) but the load factor is low, the map does a **same-size growth** -- same number of buckets, but entries are compacted.

This handles the pathological case of many insertions followed by many deletions leaving sparse overflow chains.

### Incremental Evacuation

Growth is **not done all at once** -- that would cause unpredictable latency spikes. Instead:

1. New bucket array is allocated (`hmap.buckets` points to it).
2. Old bucket array is saved (`hmap.oldbuckets`).
3. **Each subsequent write** operation evacuates 1-2 old buckets.
4. `nevacuate` tracks progress -- all buckets below this index are fully evacuated.
5. Once all old buckets are evacuated, `oldbuckets` is set to `nil`.

```go
// During growth, every write triggers evacuation
func mapassign(h *hmap, key string) *ValueType {
    // ... normal assignment logic ...

    if h.growing() {
        growWork(h, bucket) // evacuate the bucket we're writing to
    }
}
```

### Redistribution

During double-size growth, each old bucket's entries split into **exactly two** new buckets:

- Entry goes to bucket `i` or bucket `i + 2^oldB`
- Decided by one additional bit of the hash: `hash & (1 << oldB)`

```
Old bucket 0 (B=2):
  Entry hash ...00 → New bucket 0  (B=3)
  Entry hash ...100 → New bucket 4  (B=3)
```

---

## Iteration Randomization

Map iteration order is **intentionally randomized** in Go:

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

for k, v := range m {
    fmt.Println(k, v) // different order each run
}
```

### How It Works

- Each `range` iteration picks a **random start bucket** and a **random offset within that bucket**.
- The runtime uses `fastrand()` seeded from the system.
- This is a deliberate design choice to prevent developers from relying on insertion order.

### Why?

- Map layout changes with growth -- any "stable" order would break when the map resizes.
- Different Go versions may change the hash function -- relying on order would cause subtle bugs.
- Other languages (Python 3.7+) guarantee insertion order; Go explicitly does not.

---

## Concurrency

Maps are **NOT safe for concurrent read+write**. Even concurrent reads with concurrent writes will crash:

```go
m := make(map[string]int)

// This will crash with: "fatal error: concurrent map read and map write"
go func() {
    for {
        m["key"] = 1
    }
}()
go func() {
    for {
        _ = m["key"]
    }
}()
```

### How Detection Works

- The `flags` field in `hmap` has a `hashWriting` bit.
- `mapassign` sets this flag before writing, clears it after.
- `mapaccess` checks this flag -- if set, it panics immediately.
- The **race detector** (`-race` flag) provides even more precise detection.

### Solutions

| Approach | When to Use |
|---|---|
| `sync.Mutex` or `sync.RWMutex` | General purpose, wrapping map in a struct |
| `sync.Map` | Read-heavy workloads, keys are stable |
| Channel-based access | When map operations are part of a larger workflow |
| Sharded maps | High-contention workloads needing parallelism |

```go
// Common pattern: mutex-protected map
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (s *SafeMap) Get(key string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}

func (s *SafeMap) Set(key string, value int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = value
}
```

---

## Key Requirements

Map keys must satisfy the **comparable** constraint -- they must support `==` and `!=`:

### Valid key types

```go
map[int]string           // integers
map[string]int           // strings
map[[3]int]bool          // arrays (fixed size, comparable elements)
map[struct{X, Y int}]int // structs (if all fields are comparable)
map[*MyType]int          // pointers (compared by address)
map[interface{}]int      // interfaces (compared by underlying type+value)
```

### Invalid key types (compile error)

```go
map[[]int]string         // slices -- not comparable
map[map[string]int]bool  // maps -- not comparable
map[func()]int           // functions -- not comparable
```

### Interface Key Gotcha

Using `interface{}` (or `any`) as a key compiles, but panics at runtime if the underlying type is not comparable:

```go
m := map[any]int{}
m[42] = 1            // OK -- int is comparable
m[[]int{1, 2}] = 2   // PANIC: runtime error: hash of unhashable type []int
```

### NaN Keys

`float64` keys with NaN values are allowed but create "phantom" entries -- they can be inserted but never retrieved:

```go
m := map[float64]string{}
m[math.NaN()] = "ghost"
fmt.Println(m[math.NaN()]) // "" -- NaN != NaN, so lookup always fails
fmt.Println(len(m))         // 1 -- the entry exists but is inaccessible
```

---

## Make & Initialization

```go
// nil map -- reads return zero values, writes panic
var m map[string]int
_ = m["key"]   // returns 0, ok=false
m["key"] = 1   // PANIC: assignment to entry in nil map

// Empty map -- fully functional, zero elements
m = make(map[string]int)     // no size hint
m = make(map[string]int, 100) // hint: pre-allocate for ~100 entries

// Literal syntax
m = map[string]int{
    "alice": 1,
    "bob":   2,
}
```

### Size Hints

`make(map[K]V, hint)` pre-allocates enough buckets for `hint` entries. This avoids early growth:

- `hint <= 8`: 1 bucket (fits in a single bmap, may be stack-allocated)
- `hint > 8`: `2^B` buckets where `B = ceil(log2(hint / 6.5))`

Pre-sizing matters for performance when you know the approximate final size:

```go
// Without hint: multiple growths as map fills
m := make(map[string]int)       // starts small, grows 3-4 times to hold 10000 entries

// With hint: one allocation, no growth
m := make(map[string]int, 10000) // pre-allocates ~2048 buckets
```

---

## Performance Characteristics

| Operation | Average | Notes |
|---|---|---|
| Lookup `m[k]` | O(1) | Hash + scan 8-slot bucket |
| Insert `m[k] = v` | O(1) amortized | May trigger evacuation work |
| Delete `delete(m, k)` | O(1) | Clears slot, sets tophash to empty |
| Iteration `range m` | O(n) | Visits every bucket and overflow chain |
| `len(m)` | O(1) | Reads `hmap.count` directly |

### Memory Overhead

- Each bucket: `8 + 8*keySize + 8*valueSize + pointerSize` bytes
- The hmap struct itself: ~56 bytes on 64-bit
- Typical overhead per entry: ~30-50% beyond raw key+value size

---

## Related

- [[Slice]] -- buckets are stored as arrays internally
- [[Interface]] -- interface keys use hash of underlying type
- [[Sync]] -- `sync.Map` for concurrent access patterns
