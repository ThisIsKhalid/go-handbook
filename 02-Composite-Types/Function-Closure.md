# Function & Closure Internals

![[Function-internals.excalidraw]]

> Functions in Go are first-class values. Closures capture variables by reference through a hidden struct allocated on the heap.

---

## Function Values

At runtime, a function value is a pointer to a `funcval` struct (defined in `runtime/runtime2.go`):

```go
type funcval struct {
    fn uintptr  // pointer to machine code
    // ... captured variables follow
}
```

**Key properties:**

- A function value is a **pointer to a `funcval` struct** -- the variable itself is just 8 bytes on 64-bit systems.
- **Direct calls** (`foo()`) don't use `funcval` at all -- the compiler emits a direct jump to the function's machine code. Zero indirection.
- **Indirect calls** through a function value (`f := foo; f()`) go through the `funcval` pointer -- one extra dereference to read `fn`, then a call to that address.
- When there are no captured variables, multiple references to the same function share a single `funcval` (compiler optimization).

```go
func greet() { fmt.Println("hello") }

func main() {
    greet()       // direct call -- no funcval, just: CALL greet(SB)

    f := greet    // f is *funcval{fn: &greet_code}
    f()           // indirect call -- load f.fn, then CALL
}
```

---

## Stack Frames

Each function call pushes a **frame** on the goroutine's stack:

```
High address
+---------------------------+
| caller's frame            |
+---------------------------+
| return address            |
+---------------------------+
| arguments (caller writes) |   <- Go 1.16 and earlier: args on stack
| return values             |
| local variables           |
| saved registers           |
+---------------------------+
| callee's frame            |
+---------------------------+
Low address
```

**Key details:**

- Each goroutine has its own stack -- starting at 2KB, growing as needed (see [[Goroutine]]).
- **Arguments and returns** were passed on the stack in Go <= 1.16. Since **Go 1.17**, arguments and returns use **registers** (ABI redesign) -- significantly faster.
- Go uses **caller-saved** registers -- the caller is responsible for saving registers it needs across a call.
- **No frame pointer** by default. Enable with `GOEXPERIMENT=framepointer` for compatibility with profiling tools like `perf`. Go 1.21+ enables frame pointers by default on amd64.
- The runtime can walk stacks without frame pointers by using `.gopclntab` (PC-to-line table) metadata embedded in the binary.

---

## Closures

A closure is a function literal that **captures variables from its enclosing scope**:

```go
func counter() func() int {
    n := 0
    return func() int {  // closure captures &n
        n++
        return n
    }
}

func main() {
    c := counter()
    fmt.Println(c()) // 1
    fmt.Println(c()) // 2
    fmt.Println(c()) // 3
}
```

### How closures work internally

A closure is a `funcval` struct with captured variable pointers appended after the `fn` field:

```go
// What the compiler generates for the closure above:
type closureVar struct {
    fn uintptr   // pointer to the anonymous function's code
    n  *int      // pointer to captured variable n
}
```

**Critical properties:**

- Variables are captured by **REFERENCE** (pointer), **not by value**. The closure holds a `*int` to `n`, not a copy of `n`.
- Captured variables **escape to the heap** -- escape analysis detects that `n` outlives `counter()`'s stack frame and allocates it on the heap.
- **Multiple closures** over the same variable share the **same heap location**:

```go
func twoClosures() (func(), func() int) {
    n := 0
    increment := func() { n++ }     // captures &n
    read := func() int { return n }  // captures same &n
    return increment, read
}

func main() {
    inc, read := twoClosures()
    inc()
    fmt.Println(read()) // 1 -- both closures see the same n
}
```

### Common gotcha: closures in loops

```go
// BUG: all closures capture the same &i
funcs := make([]func(), 5)
for i := 0; i < 5; i++ {
    funcs[i] = func() { fmt.Println(i) }
}
for _, f := range funcs {
    f() // prints 5, 5, 5, 5, 5
}

// FIX (Go < 1.22): shadow the variable
for i := 0; i < 5; i++ {
    i := i // new variable per iteration
    funcs[i] = func() { fmt.Println(i) }
}

// FIX (Go 1.22+): loop variables are per-iteration by default
for i := 0; i < 5; i++ {
    funcs[i] = func() { fmt.Println(i) } // works correctly
}
```

---

## Defer Internals

`defer` schedules a function call to run when the surrounding function returns:

```go
type _defer struct {
    started   bool
    heap      bool
    openDefer bool
    sp        uintptr    // stack pointer at defer
    pc        uintptr    // program counter at defer
    fn        func()     // function to call
    _panic    *_panic
    link      *_defer    // linked list
}
```

### Defer mechanics

- Defers form a **linked list** (LIFO -- last deferred runs first).
- `_defer.link` points to the previously deferred call, creating a stack.
- Arguments are **evaluated at defer time**, not at execution time:

```go
func example() {
    x := 0
    defer fmt.Println(x) // captures x=0 NOW
    x = 42
    // prints 0, not 42
}
```

### Three implementations (by optimization level)

| Type | When Used | Overhead |
|---|---|---|
| **Open-coded** (Go 1.14+) | Simple defers, no loops | ~0 ns -- compiled inline with conditional branches |
| **Stack-allocated** | Known at compile time, single execution path | ~35 ns -- `_defer` struct on stack |
| **Heap-allocated** | Defers in loops: `for { defer f() }` | ~90 ns -- `_defer` struct on heap, linked list |

```go
// Open-coded defer (fastest) -- compiler inlines the deferred call
func fast() {
    defer mu.Unlock()  // becomes: if !returned { mu.Unlock() }
    mu.Lock()
    // ...
}

// Heap-allocated defer (slowest) -- unknown count at compile time
func slow() {
    for _, f := range files {
        defer f.Close()  // each iteration allocates a _defer on heap
    }
}
```

---

## Method Values

A **method value** binds a method to a specific receiver, creating a closure:

```go
type MyStruct struct {
    val int
}

func (s MyStruct) GetVal() int {
    return s.val
}

func main() {
    s := MyStruct{val: 42}
    f := s.GetVal  // method value -- captures receiver
    fmt.Println(f()) // 42
}
```

### Internal representation

A method value is syntactic sugar for a closure over the receiver:

```go
// f := s.GetVal is equivalent to:
f := func() int {
    return MyStruct.GetVal(s)  // s is captured
}

// The funcval struct looks like:
// funcval{fn: GetVal_wrapper_code, receiver: s}
```

**Key details:**

- For **value receivers**, the receiver is copied at the time the method value is created.
- For **pointer receivers**, the pointer is captured -- changes to the struct are visible through the method value.
- Method expressions (`MyStruct.GetVal`) take the receiver as an explicit first argument -- no closure needed.

```go
// Method value (closure over receiver)
f := s.GetVal     // f is func() int

// Method expression (receiver becomes first param)
g := MyStruct.GetVal  // g is func(MyStruct) int
g(s)
```

---

## Related

- [[Goroutine]] -- `go f()` launches goroutine with function
- [[Interface]] -- method tables point to function code
- [[06-Memory-Management/Escape-Analysis]] -- closures force heap escape
