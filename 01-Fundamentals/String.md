# String Internals

> How Go represents strings in memory -- immutable byte sequences with a lightweight header.

![[String-internals.excalidraw]]

---

## The StringHeader

Go strings are **NOT** null-terminated like C. A string is a **read-only slice of bytes** represented by a simple two-word struct:

```go
// reflect.StringHeader (runtime representation)
type StringHeader struct {
    Data uintptr  // pointer to underlying byte array
    Len  int      // length in bytes (NOT runes)
}
```

On a 64-bit system, a string header is always **16 bytes** regardless of content:

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    s := ""
    fmt.Println(unsafe.Sizeof(s)) // 16

    s = "Hello, World!"
    fmt.Println(unsafe.Sizeof(s)) // 16 — still 16, the header size never changes
}
```

- **Data** (`uintptr`, 8 bytes): A pointer to the first byte of the underlying array. This array lives in read-only memory for string literals.
- **Len** (`int`, 8 bytes): The number of bytes in the string. This is a byte count, not a character (rune) count.

You can inspect the header directly (though this is unsafe and should only be done for learning):

```go
package main

import (
    "fmt"
    "reflect"
    "unsafe"
)

func main() {
    s := "Hello"
    hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
    fmt.Printf("Data pointer: %#x\n", hdr.Data) // e.g., 0x49e3c8
    fmt.Printf("Length:       %d\n", hdr.Len)    // 5
}
```

> [!note]
> `reflect.StringHeader` is deprecated since Go 1.20. Use `unsafe.StringData(s)` and `len(s)` instead for the same information.

---

## Memory Layout

### String Literal Assignment

When you write `s := "Hello"`, two things are placed in memory:

1. **The header** (16 bytes) goes on the **stack** (or in the enclosing struct/slice).
2. **The byte data** `[H, e, l, l, o]` lives in the **read-only segment** of the binary (`.rodata`).

```
Stack                          .rodata (read-only memory)
+-----------+                  +---+---+---+---+---+
| Data: ----+----------------->| H | e | l | l | o |
| Len:  5   |                  +---+---+---+---+---+
+-----------+                  addr: 0x49e3c8
s (16 bytes)
```

### Assignment Copies the Header, Not the Data

```go
s := "Hello"
s2 := s       // copies 16 bytes (the header), NOT the 5 bytes of "Hello"
```

```
Stack                          .rodata
+-----------+
| Data: ----+------+           +---+---+---+---+---+
| Len:  5   |      +-------->>| H | e | l | l | o |
+-----------+      |           +---+---+---+---+---+
s                  |
                   |
+-----------+      |
| Data: ----+------+
| Len:  5   |
+-----------+
s2
```

Both `s` and `s2` share the **exact same backing byte array**. This is safe because strings are immutable -- neither can modify the bytes.

### String Interning for Literals

The Go compiler may **intern** identical string literals, meaning two variables with the same literal value could share the same `Data` pointer:

```go
a := "hello"
b := "hello"
// a and b may share the same Data pointer — compiler optimization
```

This is an implementation detail and should never be relied upon.

---

## UTF-8 Encoding

Go strings store **UTF-8 encoded bytes**. This is a fundamental design decision inherited from Plan 9 (Rob Pike and Ken Thompson co-invented UTF-8).

### Byte Length vs Rune Count

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    ascii := "Hello"
    fmt.Println(len(ascii))                    // 5 bytes
    fmt.Println(utf8.RuneCountInString(ascii)) // 5 runes

    japanese := "こんにちは"
    fmt.Println(len(japanese))                    // 15 bytes (3 bytes per rune)
    fmt.Println(utf8.RuneCountInString(japanese)) // 5 runes
}
```

Each CJK character in UTF-8 takes **3 bytes**. Emoji can take **4 bytes**.

### Iterating: Bytes vs Runes

```go
s := "Hello, 世界"

// Range iterates over RUNES — decodes UTF-8 automatically
for i, r := range s {
    fmt.Printf("index=%d rune=%c (U+%04X)\n", i, r, r)
}
// index=0  rune=H (U+0048)
// index=1  rune=e (U+0065)
// ...
// index=7  rune=世 (U+4E16)  <-- index jumps from 6 to 7
// index=10 rune=界 (U+754C)  <-- index jumps from 7 to 10

// Index loop iterates over raw BYTES
for i := 0; i < len(s); i++ {
    fmt.Printf("byte[%d] = 0x%02X\n", i, s[i])
}
// byte[7] = 0xE4  (first byte of 世)
// byte[8] = 0xB8
// byte[9] = 0x96
```

### Rune Conversion Allocates

```go
s := "Hello, 世界"
runes := []rune(s) // allocates a new []rune slice on the heap
// runes = ['H', 'e', 'l', 'l', 'o', ',', ' ', '世', '界']
// Each rune is 4 bytes (int32), so this uses 9 * 4 = 36 bytes
```

> [!warning]
> `[]rune(s)` creates a **brand new allocation**. For large strings, this can be expensive. Prefer `range` iteration when you just need to process runes sequentially.

---

## Immutability

Strings in Go are **immutable** -- once created, the underlying bytes cannot be changed.

### You Cannot Modify String Bytes

```go
s := "Hello"
s[0] = 'h' // compile error: cannot assign to s[0] (strings are immutable)
```

### Concatenation Allocates Every Time

```go
s := ""
for i := 0; i < 1000; i++ {
    s += "a" // allocates a NEW backing array on EVERY iteration
}
// This is O(n^2) — each concatenation copies all previous bytes + new bytes
```

Each `+` creates a new backing array, copies the old data, appends the new data, and the old array becomes garbage. For 1000 iterations, this means ~500,000 byte copies total.

### Use strings.Builder Instead

```go
var b strings.Builder
for i := 0; i < 1000; i++ {
    b.WriteString("a") // writes into a growing internal buffer
}
s := b.String() // one final string — efficient
```

`strings.Builder` uses a `[]byte` internally and doubles capacity when needed (amortized O(1) per write), then converts to string without copying at the end.

```go
// If you know the final size, pre-allocate:
var b strings.Builder
b.Grow(1000) // pre-allocate 1000 bytes
for i := 0; i < 1000; i++ {
    b.WriteByte('a')
}
```

### Unsafe Mutation (Do Not Do This)

```go
import "unsafe"

s := "Hello"
b := unsafe.Slice(unsafe.StringData(s), len(s))
b[0] = 'h' // UNDEFINED BEHAVIOR if s points to read-only memory
// May crash with SIGSEGV, may silently corrupt, may work by accident
```

> [!danger]
> Mutating a string's backing array through unsafe operations is **undefined behavior**. If the string literal is in read-only memory (`.rodata`), this will cause a segmentation fault. Even if it works on your machine, it can break on different OS/arch combinations.

---

## String vs []byte

| Property | `string` | `[]byte` |
|---|---|---|
| **Mutability** | Immutable | Mutable |
| **Header size** | 16 bytes (Data + Len) | 24 bytes (Data + Len + Cap) |
| **Concurrent safety** | Safe (immutable) | Requires synchronization |
| **Map key** | Yes | No |
| **I/O operations** | Must convert | Native (`io.Reader`/`io.Writer`) |

### Conversion Copies Data (Usually)

```go
s := "Hello"
b := []byte(s)  // allocates new []byte, copies 5 bytes
s2 := string(b) // allocates new string backing, copies 5 bytes
```

Each conversion copies the data because the type system must guarantee:
- `string` data will not be mutated (so `[]byte -> string` must copy)
- `[]byte` from a `string` should not alias read-only memory (so `string -> []byte` must copy)

### Compiler Optimizations That Avoid Copies

The Go compiler recognizes several patterns and **skips the copy**:

```go
// 1. Map lookup with []byte key — no copy
m := map[string]int{"hello": 1}
b := []byte("hello")
_ = m[string(b)] // compiler optimizes: no allocation

// 2. String comparison with []byte — no copy
if string(b) == "hello" { } // no allocation

// 3. Range over []byte converted to string — no copy
for i, r := range string(b) { } // no allocation

// 4. String concatenation with []byte — compiler may optimize
s := "prefix" + string(b) // may avoid intermediate string allocation
```

### Zero-Copy Conversion (Go 1.22+)

```go
import "unsafe"

// []byte to string without copy (CAREFUL: b must not be modified after)
s := unsafe.String(unsafe.SliceData(b), len(b))

// string to []byte without copy (CAREFUL: result must not be modified)
b := unsafe.Slice(unsafe.StringData(s), len(s))
```

> [!warning]
> Zero-copy conversions violate the type system's guarantees. Only use them in performance-critical code where you can guarantee the safety invariants.

---

## Gotchas

### 1. Substrings Retain the Entire Backing Array

```go
func getFirstTwo(s string) string {
    return s[:2] // returns a string header pointing into s's backing array
}

// Problem:
bigString := loadHugeFile() // 10 MB string
small := bigString[:5]      // "small" holds a reference to the 10 MB backing array
bigString = ""              // the 10 MB STILL can't be GC'd because "small" references it
```

**Fix: Copy the substring explicitly**

```go
func getFirstTwo(s string) string {
    return string([]byte(s[:2])) // force a copy into a new, small backing array
}
// Or use strings.Clone (Go 1.20+):
small := strings.Clone(bigString[:5])
```

### 2. String Comparison is O(n)

```go
a := "hello"
b := "hello"
_ = a == b // compares byte-by-byte, O(min(len(a), len(b)))
```

Go does **not** use pointer comparison as a shortcut (even if `a` and `b` share the same backing array). The comparison always walks the bytes.

> [!note]
> Exception: if `len(a) != len(b)`, the comparison returns `false` immediately (O(1)).

### 3. len() Returns Bytes, Not Characters

```go
s := "cafe\u0301" // "cafe" + combining acute accent = "cafe" (visually)
fmt.Println(len(s))                    // 6 bytes
fmt.Println(utf8.RuneCountInString(s)) // 5 runes
// But visually it's 4 characters! (e + combining accent = one grapheme cluster)
```

For true "character" counting, you need the `golang.org/x/text/unicode/norm` package to handle grapheme clusters.

### 4. String Indexing Returns a Byte, Not a Rune

```go
s := "Hello, 世界"
fmt.Println(s[7])    // 228 (0xE4) — first byte of 世, NOT the rune itself
fmt.Println(s[7:10]) // "世" — only if you know the exact byte boundaries
```

---

## Performance Tips

1. **Use `strings.Builder`** for concatenation in loops. Pre-allocate with `Grow()` if you know the final size.
2. **Avoid `[]rune` conversion** on large strings -- use `range` instead.
3. **Use `strings.Clone`** (Go 1.20+) to release large backing arrays from substrings.
4. **Prefer `[]byte`** for I/O-heavy code to avoid conversion overhead.
5. **Profile before using `unsafe`** -- the compiler optimizations often make manual tricks unnecessary.

---

## Related

- [[Slice]] -- strings are essentially immutable slices of bytes
- [[Array]] -- the underlying byte array
- [[Map]] -- strings are valid map keys; `[]byte` is not
