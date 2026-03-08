# Slice Internals

> The most important data structure in Go — a dynamically-sized view into an array.

A slice is not an array. It is a **descriptor** — a small struct that lives on the stack and points to an underlying array on the heap. Understanding this distinction is the key to writing correct, performant Go code.

![[Slice-internals.excalidraw]]

---

## The SliceHeader

Every slice value in Go is represented at runtime by a `reflect.SliceHeader`:

```go
type SliceHeader struct {
    Data uintptr  // pointer to backing array
    Len  int      // number of elements in use
    Cap  int      // total capacity of backing array
}
```

- `unsafe.Sizeof([]int{})` = **24 bytes** (3 x 8 bytes on a 64-bit system)
- The header is always the same size regardless of the element type or number of elements.
- **Nil slice vs empty slice**:
  - `var s []int` — nil slice. `Data=0`, `Len=0`, `Cap=0`. `s == nil` is `true`.
  - `s := []int{}` — empty slice. `Data` points to a valid (but zero-length) address, `Len=0`, `Cap=0`. `s == nil` is `false`.
  - Both behave identically with `len()`, `cap()`, `range`, and `append`. The distinction matters only for `nil` checks and JSON marshaling (`null` vs `[]`).

```go
var nilSlice []int          // nil: Data=0, Len=0, Cap=0
emptySlice := []int{}       // empty: Data=<addr>, Len=0, Cap=0
madeSlice := make([]int, 0) // empty: Data=<addr>, Len=0, Cap=0

fmt.Println(nilSlice == nil)  // true
fmt.Println(emptySlice == nil) // false
```

---

## Memory Layout

The SliceHeader lives on the **stack** (24 bytes), while the backing array lives on the **heap**.

```go
s := make([]int, 3, 5)
// SliceHeader: { Data: 0xc0000b2000, Len: 3, Cap: 5 }
// Heap: [0][0][0][_][_]  (5 ints = 40 bytes allocated)
//        ^-accessible-^  ^-exist but inaccessible-^
```

- Elements at indices 0, 1, 2 are accessible via `s[i]`.
- Elements at indices 3 and 4 **exist in memory** but are not part of the slice — accessing `s[3]` panics with `index out of range`.
- These hidden elements become accessible only through `append` or re-slicing with `s[:5]` (since `cap` is 5).

```go
fmt.Println(len(s)) // 3
fmt.Println(cap(s)) // 5
s = s[:5]           // now Len=5, all 5 elements accessible
```

---

## Slicing Operations

### Basic slicing: `s[low:high]`

Creates a **new SliceHeader** pointing into the **same backing array**:

```go
original := []int{10, 20, 30, 40, 50}
sub := original[1:3] // sub = [20, 30]
// sub.Data = original.Data + 1*sizeof(int)
// sub.Len = 2, sub.Cap = 4 (from index 1 to end of original's capacity)
```

**Shared memory** — modifying `sub[0]` changes `original[1]`:

```go
sub[0] = 999
fmt.Println(original) // [10, 999, 30, 40, 50]
```

### Three-index slice: `s[low:high:max]`

Limits the **capacity** of the resulting slice to prevent `append` from overwriting elements in the original:

```go
safe := original[1:3:3] // Len=2, Cap=2 (not 4!)
// Now append(safe, 99) MUST allocate a new backing array
// instead of overwriting original[3]
```

This is critical when passing sub-slices to functions that may `append` to them.

### Gotcha: sub-slices share memory

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]        // b = [2, 3], shares backing array
b = append(b, 99)  // overwrites a[3]!
fmt.Println(a)     // [1, 2, 3, 99, 5] — surprise!
```

Use a three-index slice or `slices.Clone()` to avoid this.

---

## Append Internals

`append` is the only way to grow a slice. Its behavior depends on available capacity:

### Case 1: `len < cap` (capacity available)

```go
s := make([]int, 3, 5) // [0, 0, 0, _, _]
s = append(s, 42)       // [0, 0, 0, 42, _]  — Len=4, same Data pointer
```

- Writes the new value at `s[len]`
- Increments `Len` by 1
- **No allocation** — O(1) operation

### Case 2: `len == cap` (no capacity)

```go
s := []int{1, 2, 3, 4, 5} // Len=5, Cap=5
s = append(s, 6)            // NEW backing array allocated!
// Old: [1, 2, 3, 4, 5]        — becomes GC collectible
// New: [1, 2, 3, 4, 5, 6, _, _, _, _]  — Len=6, Cap=10
```

- Allocates a **new, larger** backing array
- Copies all existing elements to the new array
- Writes the new element
- Updates `Data` pointer and `Cap`
- Old backing array becomes eligible for garbage collection

### Growth Strategy (Go 1.18+)

```
if oldCap < 256:
    newCap = oldCap * 2                        // double
else:
    newCap = oldCap + oldCap/4 + 192           // ~1.25x + constant
```

| Old Cap | New Cap | Growth Factor |
|---------|---------|---------------|
| 4       | 8       | 2.00x         |
| 64      | 128     | 2.00x         |
| 256     | 512     | 2.00x         |
| 512     | 832     | 1.63x         |
| 1024    | 1472    | 1.44x         |
| 4096    | 5312    | 1.30x         |

The transition is smooth — no hard cutoff. The `+192` constant prevents pathological cases near the boundary.

![[Slice-growth.excalidraw]]

### Always reassign

```go
s = append(s, val) // correct — append may return a new slice
append(s, val)     // BUG — new slice is discarded if reallocation happened
```

---

## Copy Semantics

### Assignment copies the header

```go
s1 := []int{1, 2, 3}
s2 := s1 // copies 24 bytes (the header) — SHARES the backing array

s2[0] = 999
fmt.Println(s1[0]) // 999 — both see the same data
```

### `copy()` copies elements

```go
src := []int{1, 2, 3}
dst := make([]int, len(src))
n := copy(dst, src) // copies min(len(dst), len(src)) elements
// dst is now [1, 2, 3], completely independent of src
```

- Returns the number of elements copied.
- Does **not** allocate — you must pre-allocate `dst`.
- Copies `min(len(dst), len(src))` elements.

### `slices.Clone()` (Go 1.21+)

```go
import "slices"

clone := slices.Clone(original) // independent copy
```

Under the hood, this is equivalent to:

```go
clone := append([]T(nil), original...)
```

### Full deep copy pattern

```go
// Shallow clone (elements are copied, but if they're pointers, the pointees are shared)
clone := slices.Clone(s)

// Deep clone (for slices of pointers or structs with pointer fields)
clone := make([]*MyStruct, len(s))
for i, v := range s {
    copied := *v
    clone[i] = &copied
}
```

---

## Common Patterns & Gotchas

### Memory leak: holding capacity

```go
// BAD: s still holds reference to large backing array
s = s[:0] // Len=0 but Cap unchanged, backing array not GC'd

// GOOD: release backing array
s = nil
```

This is especially dangerous when slicing a large slice to keep a small piece:

```go
// BAD: big's 1MB backing array stays alive
small := big[0:2]

// GOOD: copy the elements you need
small := slices.Clone(big[0:2])
```

### Slice of large structs

```go
type BigStruct struct {
    Data [1024]byte
    // ... lots of fields
}

// BAD: append copies the entire 1KB+ struct each time
items := []BigStruct{}

// GOOD: store pointers, append only copies 8 bytes
items := []*BigStruct{}
```

### Pre-allocate when size is known

```go
// BAD: multiple allocations and copies during growth
var result []int
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// GOOD: single allocation
result := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    result = append(result, i)
}
```

### `append` may or may not allocate — never assume

```go
func process(s []int) {
    s = append(s, 42) // might modify caller's array, might not!
}

// If the caller's slice has spare capacity, this overwrites
// data the caller might still be using. If it doesn't,
// a new array is allocated and the caller sees nothing.
// This is a source of subtle bugs.
```

### Clearing a slice

```go
// Go 1.21+
clear(s) // zeros all elements, keeps len and cap

// Pre-1.21
for i := range s {
    s[i] = zero // zero value of element type
}
```

---

## Performance Notes

| Operation | Time Complexity | Allocates? |
|-----------|----------------|------------|
| `s[i]`   | O(1)           | No         |
| `len(s)` | O(1)           | No         |
| `cap(s)` | O(1)           | No         |
| `append` (has cap) | O(1) | No         |
| `append` (no cap)  | O(n) | Yes        |
| `copy`   | O(n)           | No         |
| `s[a:b]` | O(1)           | No         |
| `slices.Contains` | O(n) | No         |

---

## Related

- [[Array]] — the fixed-size backing storage
- [[String]] — strings are essentially read-only slices of bytes
- [[Map]] — uses slices internally for bucket arrays
