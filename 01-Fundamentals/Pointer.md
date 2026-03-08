# Pointer Internals

![[Pointer-internals.excalidraw]]

> Pointers in Go are safe by default -- no arithmetic, no dangling references, with escape analysis deciding stack vs heap.

---

## The Basics

A pointer `*T` holds the memory address of a value of type `T`. The zero value of any pointer is `nil`.

```go
var p *int       // nil pointer -- points to nothing
x := 42
p = &x           // & takes the address of x
fmt.Println(*p)  // * dereferences: prints 42
*p = 100         // modify x through the pointer
fmt.Println(x)   // 100
```

On a 64-bit system, a pointer is always **8 bytes** regardless of what it points to:

```go
fmt.Println(unsafe.Sizeof((*int)(nil)))     // 8
fmt.Println(unsafe.Sizeof((*[1000]int)(nil))) // 8
fmt.Println(unsafe.Sizeof((*string)(nil)))  // 8
```

### What Go does NOT have

Unlike C/C++, Go pointers are **restricted**:

- **No pointer arithmetic** -- you cannot do `p++` or `p + 4`.
- **No dangling pointers** -- the garbage collector tracks all pointers. If a pointer exists, the data it points to will not be freed.
- **No manual free** -- memory is managed by the GC. You cannot (and need not) call `free()`.
- **No casting between unrelated pointer types** -- `*int` cannot be directly cast to `*float64`.

---

## Stack vs Heap

Go's compiler performs **escape analysis** to decide whether a variable lives on the stack (fast, automatically freed when function returns) or the heap (slower, garbage collected).

### Stack allocation (the fast path)

```go
func add(a, b int) int {
    sum := a + b   // sum lives on the stack
    return sum     // value is copied to caller -- sum is freed when add() returns
}
```

Variables that **do not escape** the function stay on the stack. Stack allocation is essentially free -- it just moves the stack pointer.

### Heap allocation (when pointers escape)

```go
func newInt(val int) *int {
    x := val    // x starts on the stack
    return &x   // &x escapes! compiler moves x to the heap
}
```

When the compiler sees that `&x` outlives the function, it allocates `x` on the heap instead. The pointer remains valid after the function returns because the GC tracks it.

### Viewing escape analysis

```bash
go build -gcflags="-m" .
# Output:
# ./main.go:4:2: moved to heap: x
```

Common reasons for heap escape:

- Returning a pointer to a local variable
- Storing a pointer in a global variable or a heap-allocated struct
- Sending a pointer to a channel
- Interface conversions (value may be boxed on the heap)
- Closures capturing local variables by reference
- Values too large for the stack

> [!note]
> Stack vs heap is a **compiler implementation detail**, not part of the Go spec. The compiler can (and does) change its escape analysis heuristics between versions.

---

## unsafe.Pointer

The `unsafe` package provides an escape hatch from Go's type-safe pointer system.

### Three pointer types

| Type | Can hold any address? | GC-tracked? | Arithmetic? |
|---|---|---|---|
| `*T` | No (only `*T`) | Yes | No |
| `unsafe.Pointer` | Yes (any pointer) | Yes | No |
| `uintptr` | Yes (any address) | **No** | Yes |

### unsafe.Pointer as a bridge

`unsafe.Pointer` can convert between any pointer type, serving as a bridge:

```go
func float64bits(f float64) uint64 {
    // *float64 -> unsafe.Pointer -> *uint64
    return *(*uint64)(unsafe.Pointer(&f))
}
```

### uintptr for arithmetic (dangerous)

```go
type Example struct {
    A int32
    B int64
}

e := Example{A: 1, B: 2}
// Access B via pointer arithmetic
pB := (*int64)(unsafe.Pointer(uintptr(unsafe.Pointer(&e)) + unsafe.Offsetof(e.B)))
fmt.Println(*pB) // 2
```

> [!danger]
> `uintptr` is **NOT** a pointer -- the GC does not track it. If a GC cycle runs between converting `unsafe.Pointer` to `uintptr` and back, the object may be moved and your `uintptr` becomes a **dangling reference**. The conversion must happen in a **single expression**.

```go
// SAFE: single expression
p := (*int64)(unsafe.Pointer(uintptr(unsafe.Pointer(&e)) + offset))

// UNSAFE: uintptr stored in a variable -- GC may invalidate it
addr := uintptr(unsafe.Pointer(&e)) + offset  // GC can run here!
p := (*int64)(unsafe.Pointer(addr))            // addr may be stale
```

### Valid unsafe.Pointer patterns (from the Go spec)

1. Convert `*T1` to `*T2` (via `unsafe.Pointer`)
2. Convert `unsafe.Pointer` to `uintptr` (but do not store it)
3. Pointer arithmetic with `uintptr` (in a single expression)
4. `reflect.Value.Pointer()` or `reflect.Value.UnsafeAddr()` conversion
5. Conversion of `unsafe.Pointer` to/from `syscall.Syscall` arguments

---

## Pointer Receivers

Methods in Go can have either **value receivers** or **pointer receivers**:

```go
type Counter struct {
    n int
}

// Value receiver -- gets a COPY of Counter
func (c Counter) Value() int {
    return c.n
}

// Pointer receiver -- gets the ORIGINAL Counter
func (c *Counter) Increment() {
    c.n++ // modifies the original
}
```

### Auto-dereferencing

Go automatically takes the address or dereferences when calling methods:

```go
c := Counter{n: 0}
c.Increment()     // Go automatically takes &c
fmt.Println(c.n)  // 1

p := &Counter{n: 0}
p.Value()          // Go automatically dereferences *p
fmt.Println(p.n)   // auto-deref: same as (*p).n
```

### Method sets matter for interfaces

This is where the distinction becomes critical:

```go
type Incrementer interface {
    Increment()
}

var c Counter
// var _ Incrementer = c   // COMPILE ERROR: Counter does not implement Incrementer
var _ Incrementer = &c     // OK: *Counter has Increment()
```

| Receiver type | Method set includes |
|---|---|
| `T` (value) | Only value receiver methods |
| `*T` (pointer) | **Both** value and pointer receiver methods |

The reason: if you have a value (not addressable), Go cannot reliably take its address to call a pointer receiver method. But if you have a pointer, Go can always dereference it to call a value receiver method.

### When to use pointer receivers

- The method needs to **modify** the receiver
- The struct is **large** (avoid copying)
- **Consistency** -- if one method needs a pointer receiver, make them all pointer receivers

---

## The new() and & Built-ins

```go
// These are equivalent:
p1 := new(int)       // allocates int on heap, returns *int (zero-valued)
p2 := &int{}         // INVALID -- cannot take address of non-composite literal

// For structs:
p3 := new(MyStruct)       // returns *MyStruct with zero-valued fields
p4 := &MyStruct{X: 1}     // returns *MyStruct with X=1 (composite literal)
```

`new(T)` is rarely used in practice -- `&T{}` is more idiomatic for structs because it allows field initialization.

---

## Nil Pointer Safety

```go
var p *int
// fmt.Println(*p)  // PANIC: runtime error: invalid memory address or nil pointer dereference

// Always check for nil when pointer may be nil
if p != nil {
    fmt.Println(*p)
}
```

Methods can be called on nil pointer receivers (this is valid and sometimes useful):

```go
type List struct {
    Value int
    Next  *List
}

func (l *List) Len() int {
    if l == nil {
        return 0  // nil list has length 0
    }
    return 1 + l.Next.Len()
}
```

---

## Related

- [[Slice]] -- contains a pointer to backing array
- [[String]] -- contains a pointer to byte array
- [[Interface]] -- stores a pointer to data
- [[06-Memory-Management/Escape-Analysis]] -- decides where pointers live
