# Struct Internals

![[Struct-internals.excalidraw]]

> Structs are Go's composite value type -- understanding alignment and padding is key to writing memory-efficient code.

---

## Memory Layout

Struct fields are laid out **sequentially in declaration order** in memory. Unlike C, Go does **not** reorder fields -- the programmer is responsible for choosing an efficient layout.

```go
type Example struct {
    A int32   // 4 bytes at offset 0
    B int64   // 8 bytes at offset 8 (not 4! padding inserted)
    C bool    // 1 byte at offset 16
}
// Total: 24 bytes (with 3 bytes padding after A, 7 bytes padding after C)
```

**Key properties:**

- Fields are placed at their **natural alignment boundary** (a field of size N must start at an address divisible by N).
- The compiler inserts **padding bytes** between fields to satisfy alignment requirements.
- The overall struct size is rounded up to a multiple of the largest field's alignment.
- `unsafe.Sizeof()` reports the total size including all padding.

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    type Example struct {
        A int32
        B int64
        C bool
    }
    var e Example
    fmt.Println("Size:", unsafe.Sizeof(e))          // 24
    fmt.Println("Align:", unsafe.Alignof(e))        // 8
    fmt.Println("Offset A:", unsafe.Offsetof(e.A))  // 0
    fmt.Println("Offset B:", unsafe.Offsetof(e.B))  // 8
    fmt.Println("Offset C:", unsafe.Offsetof(e.C))  // 16
}
```

---

## Alignment Rules

Alignment ensures the CPU can access fields efficiently. Misaligned access can be slow (or even crash on some architectures).

```go
type Bad struct {    // 24 bytes (with padding)
    a bool   // 1 byte + 7 padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 padding
}
type Good struct {   // 16 bytes (optimized)
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte + 6 padding
}
```

### Alignment sizes on 64-bit systems

| Type | Size | Alignment |
|---|---|---|
| `bool` | 1 byte | 1 |
| `int8` / `uint8` / `byte` | 1 byte | 1 |
| `int16` / `uint16` | 2 bytes | 2 |
| `int32` / `uint32` / `float32` | 4 bytes | 4 |
| `int64` / `uint64` / `float64` | 8 bytes | 8 |
| `int` / `uint` / `uintptr` | 8 bytes | 8 |
| `string` | 16 bytes | 8 |
| `slice` | 24 bytes | 8 |
| `pointer` | 8 bytes | 8 |
| `interface` | 16 bytes | 8 |

### Inspection tools

- `unsafe.Alignof(x)` -- alignment requirement of x's type
- `unsafe.Sizeof(x)` -- total size of x including padding
- `unsafe.Offsetof(s.f)` -- offset of field f within struct s

**General rule: order fields from largest to smallest** to minimize padding. For production code, use tools like [`fieldalignment`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/fieldalignment):

```bash
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
fieldalignment -fix ./...
```

---

## Embedding

Go supports **struct embedding** -- placing a type without a field name. This promotes the embedded type's fields and methods to the outer type.

```go
type Inner struct {
    X int
    Y int
}

func (i Inner) Sum() int {
    return i.X + i.Y
}

type Outer struct {
    Inner          // embedded -- NOT inheritance
    Z     int
}

func main() {
    o := Outer{Inner: Inner{X: 1, Y: 2}, Z: 3}
    fmt.Println(o.X)     // promoted field access (same as o.Inner.X)
    fmt.Println(o.Sum()) // promoted method call (same as o.Inner.Sum())
}
```

### Memory: inner struct fields are inlined

```
Outer in memory:
+-----+-----+-----+
|  X  |  Y  |  Z  |
+-----+-----+-----+
  Inner (inlined)
```

The embedded struct is **not a pointer** -- its fields are physically placed inside the outer struct. This means:

- **No indirection** -- accessing promoted fields is as fast as direct fields.
- **Value semantics** -- copying `Outer` copies `Inner` too.
- **Shadowing** -- if `Outer` declares a field `X`, it shadows `Inner.X` (but `o.Inner.X` still works).

### Embedding is NOT inheritance

```go
type Base struct{}
func (b Base) Method() { fmt.Println("Base") }

type Derived struct{ Base }
func (d Derived) Method() { fmt.Println("Derived") }

var d Derived
d.Method()       // "Derived" — Derived's method is called
d.Base.Method()  // "Base" — Base's method is still accessible
```

There is no virtual dispatch. Embedding is composition with syntactic sugar for delegation.

---

## Zero Value

Every struct field is initialized to its **zero value** upon declaration:

```go
type Config struct {
    Host    string  // ""
    Port    int     // 0
    Debug   bool    // false
    Logger  *log.Logger // nil
}

var c Config
// c is fully usable -- all fields are zero-valued
```

This enables the powerful pattern of **types that work without a constructor**:

```go
// sync.Mutex works at zero value -- no NewMutex() needed
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()

// bytes.Buffer works at zero value
var buf bytes.Buffer
buf.WriteString("hello")

// http.Client works at zero value (uses defaults)
var client http.Client
resp, err := client.Get("https://example.com")
```

> [!tip]
> Design your structs so the zero value is **useful and safe**. This is idiomatic Go -- constructors should only be needed when the zero value is not a valid state.

---

## Comparison

Structs are **comparable** if and only if **all their fields are comparable**:

```go
type Point struct {
    X, Y int
}
a := Point{1, 2}
b := Point{1, 2}
fmt.Println(a == b) // true -- compares all fields

// Comparable structs can be map keys
cache := map[Point]string{
    {0, 0}: "origin",
}
```

### Non-comparable structs

```go
type Data struct {
    Values []int // slices are NOT comparable
}
a := Data{[]int{1, 2}}
b := Data{[]int{1, 2}}
// fmt.Println(a == b) // COMPILE ERROR: struct containing []int cannot be compared
```

Types that make a struct non-comparable: `slice`, `map`, `func`, and any struct containing these.

For non-comparable structs, use `reflect.DeepEqual` (slow, uses reflection) or implement a custom `Equal` method.

---

## Struct Tags

Struct tags are string metadata attached to fields, read at runtime via reflection:

```go
type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email,omitempty" validate:"email"`
    Age   int    `json:"age" validate:"gte=0,lte=130"`
}
```

Tags do **not** affect memory layout or performance -- they are stored in type metadata, not in each struct instance.

---

## Related

- [[Array]] -- similar value semantics
- [[Interface]] -- struct methods satisfy interfaces
- [[Pointer]] -- `&struct` creates pointer, escapes to heap
