# Escape Analysis

> Go's compiler statically analyzes where values are used to decide stack vs heap allocation — `go build -gcflags="-m"` shows the decisions.

![[Escape-Analysis-internals.excalidraw]]

---

## How It Works

The compiler builds a graph of how values flow through the program. If a value can be proven to not outlive its stack frame, it stays on the stack. Otherwise, it "escapes" to the heap.

- Performed at **compile time** (not runtime)
- Conservative: if the compiler **can't prove** a value stays local, it escapes
- Per-function analysis with inlining considered

---

## Common Escape Patterns

```go
// ESCAPES: pointer returned to caller
func escape1() *int {
    x := 1
    return &x  // x must outlive escape1's stack frame
}

// ESCAPES: assigned to interface (boxing)
func escape2() {
    var i interface{} = 42  // 42 boxed on heap
    _ = i
}

// ESCAPES: closure captures by reference
func escape3() func() {
    x := 1
    return func() { x++ }  // x shared between scopes
}

// ESCAPES: sent to channel (crosses goroutine boundary)
func escape4(ch chan *int) {
    x := 1
    ch <- &x  // x may be read by another goroutine
}

// ESCAPES: slice too large for stack
func escape5() {
    s := make([]int, 1000000)  // ~8MB, too large for stack
    _ = s
}

// DOES NOT ESCAPE: known size, local use only
func noEscape() {
    s := make([]int, 10)  // small, stays on stack
    _ = s
}
```

---

## Debugging Escape Analysis

```bash
# Basic: shows what escapes and why
go build -gcflags="-m" ./...

# Verbose: shows detailed reasoning
go build -gcflags="-m -m" ./...
```

### Reading the Output

```
./main.go:5:2: moved to heap: x          # variable escapes
./main.go:6:9: &x escapes to heap        # pointer causes escape
./main.go:10:6: can inline noEscape      # inlining helps avoid escape
./main.go:12:14: make([]int, 10) does not escape  # stays on stack
```

---

## What Causes Escape

| Pattern | Escapes? | Reason |
|---------|----------|--------|
| Return pointer to local | Yes | Outlives function |
| Assign to `interface{}` | Yes | Boxing requires heap |
| Closure captures variable | Yes | Shared across scopes |
| Send pointer to channel | Yes | Crosses goroutine |
| `make([]T, n)` where n is large or variable | Yes | Size unknown or too big |
| `make([]T, n)` where n is small constant | No | Compiler proves it fits |
| Pass pointer to function that doesn't store it | No | Compiler traces flow |
| Return value (not pointer) | No | Copied to caller's frame |

---

## Optimization Tips

### Prefer value semantics for small types

```go
// GOOD: no escape, stays on stack
func process() Point {
    p := Point{X: 1, Y: 2}
    return p  // copied (16 bytes), no heap allocation
}

// BAD: forces heap allocation
func process() *Point {
    p := Point{X: 1, Y: 2}
    return &p  // escapes to heap
}
```

### Use pointer receivers for large structs

```go
// Large struct: pointer receiver avoids copying 1KB+ per call
type BigConfig struct {
    Data [1024]byte
}

func (c *BigConfig) Get() byte { return c.Data[0] }
```

### Pre-allocate slices with known capacity

```go
// GOOD: single allocation, known size
result := make([]int, 0, len(input))

// BAD: multiple allocations during growth
var result []int
```

### Avoid `interface{}` in hot paths

```go
// BAD: boxing causes heap allocation every call
func log(v interface{}) { ... }

// GOOD: use generics (Go 1.18+)
func log[T any](v T) { ... }
```

### Use `sync.Pool` for temporary allocations

```go
var bufPool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func process() {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)
    buf.Reset()
    // use buf...
}
```

---

## Inlining and Escape Analysis

Inlining can **prevent** escapes by bringing the callee's code into the caller:

```go
//go:inline
func newPoint(x, y int) *Point {
    return &Point{X: x, Y: y}  // would escape...
}

func main() {
    p := newPoint(1, 2)  // but if inlined, p stays on main's stack!
}
```

Use `go build -gcflags="-m"` to check if a function is inlined. Functions that are too complex (high cost) won't be inlined.

---

## Related

- [[Stack-vs-Heap]] — where stack and heap values live
- [[GC]] — heap allocations create GC pressure
