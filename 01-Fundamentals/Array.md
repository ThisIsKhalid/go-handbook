# Array Internals

![[Array-internals.excalidraw]]

> Go arrays are fixed-size, contiguous blocks of memory -- the foundation that slices are built on.

---

## The Basics

An array in Go has a **fixed length that is part of its type**. This means `[5]int` and `[6]int` are completely different types and cannot be assigned to each other.

```go
var a [5]int             // zero-valued: [0, 0, 0, 0, 0]
b := [5]int{10, 20, 30, 40, 50}
c := [...]int{1, 2, 3}  // compiler infers [3]int
```

- The size **must** be a compile-time constant -- you cannot write `[n]int` where `n` is a variable.
- Arrays are **value types** -- assignment copies the entire array, not a reference.
- `unsafe.Sizeof([5]int{})` = **40 bytes** (5 elements x 8 bytes per `int` on 64-bit).

---

## Memory Layout

Arrays are laid out as a **contiguous block of memory** with no overhead. The variable itself *is* the data -- there is no hidden header struct like slices have.

```
Address:   0x00    0x08    0x10    0x18    0x20
         +-------+-------+-------+-------+-------+
  a  =>  |  10   |  20   |  30   |  40   |  50   |
         +-------+-------+-------+-------+-------+
```

**Key properties:**

- Element `i` lives at `base + i * sizeof(element)`.
- All elements are **zero-valued** on declaration (`0` for ints, `""` for strings, `nil` for pointers).
- **No header struct** -- unlike a slice, the array variable points directly at the first element.
- **Stack allocation** for small arrays; the compiler may move large arrays to the heap via escape analysis.

```go
// Small array -- stays on the stack
var small [4]int

// Large array -- likely escapes to the heap
var large [1_000_000]int
_ = large
```

---

## Value Semantics

Arrays in Go have **full value semantics**. Every assignment, every function call creates a complete copy.

```go
a := [5]int{10, 20, 30, 40, 50}
b := a        // COPIES all 5 elements -- b is independent
b[0] = 999    // a[0] is still 10
```

### Passing to functions copies everything

```go
func foo(a [1000]int) {
    // This parameter is a COPY -- 8,000 bytes copied on each call
    a[0] = 42 // does NOT affect the caller's array
}
```

### Use pointers for large arrays

```go
func foo(a *[1000]int) {
    // Only an 8-byte pointer is passed
    a[0] = 42 // DOES affect the caller's array
}
```

### Comparison

Arrays support `==` and `!=` which compare **all elements**:

```go
a := [3]int{1, 2, 3}
b := [3]int{1, 2, 3}
fmt.Println(a == b) // true
```

This also means arrays can be used as **map keys** (slices cannot):

```go
cache := map[[2]int]string{
    {0, 0}: "origin",
    {1, 1}: "diagonal",
}
```

---

## Array vs Slice

| Property        | Array `[5]int`          | Slice `[]int`                  |
| --------------- | ----------------------- | ------------------------------ |
| Size            | Fixed at compile time   | Dynamic, grows at runtime      |
| Type includes   | Length (`[5]int`)       | No length (`[]int`)            |
| Semantics       | Value type (copies)     | Reference type (shares backing)|
| Comparable      | Yes (`==` works)        | No (compile error)             |
| Map key         | Yes                     | No                             |
| Header          | None -- variable IS data| 24-byte struct (ptr, len, cap) |
| Memory          | Contiguous, inline      | Header + backing array         |

### Converting array to slice

```go
a := [5]int{10, 20, 30, 40, 50}
s := a[:]     // slice backed by a -- no copy, shares memory
s[0] = 999    // a[0] is now 999 too!
```

---

## Compile-Time Magic

The Go compiler applies several optimizations to arrays:

### Size must be constant

```go
const N = 5
var a [N]int   // OK -- N is a constant

n := 5
var b [n]int   // COMPILE ERROR -- n is not a constant
```

### Compiler-counted initializer

```go
a := [...]int{1, 2, 3}  // compiler infers [3]int
```

### Bounds check elimination (BCE)

When the compiler can prove an index is in-bounds, it **removes the runtime bounds check**:

```go
func sum(a [4]int) int {
    // The compiler knows len(a) == 4 at compile time
    // All these accesses have bounds checks eliminated
    return a[0] + a[1] + a[2] + a[3]
}
```

### Register optimization

Arrays with fewer than **4 elements** of simple types may be kept entirely in CPU registers -- no memory access at all:

```go
// This may live entirely in registers
pair := [2]int{x, y}
```

### `range` loop optimization

```go
for i, v := range a {
    // The compiler knows the exact iteration count
    // and can unroll or vectorize the loop
}
```

---

## Related

- [[Slice]] -- dynamic wrapper around arrays
- [[Struct]] -- similar value semantics and memory layout
