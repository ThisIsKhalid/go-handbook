# Goroutine & Scheduler Internals

> Goroutines are Go's lightweight threads — 2KB initial stack, multiplexed onto OS threads by a work-stealing scheduler.

---

## The G-M-P Model

- **G (Goroutine)**: the goroutine struct — stack, instruction pointer, status
- **M (Machine)**: OS thread — executes Go code, syscalls
- **P (Processor)**: logical processor — holds run queue, manages scheduling
- `GOMAXPROCS` = number of P's (default = CPU cores)

![[Goroutine-internals.excalidraw]]

---

## Goroutine Struct (simplified)

```go
type g struct {
    stack       stack    // stack bounds [lo, hi)
    stackguard0 uintptr  // for stack growth check
    m           *m       // current M executing this G
    sched       gobuf    // saved context (SP, PC, etc.)
    status      uint32   // idle, runnable, running, syscall, waiting, dead
    goid        uint64   // goroutine ID
    // ...
}
```

---

## Stack Management

- **Initial stack**: 2KB (vs 1-8MB for OS threads)
- **Stack growth**: detected by `stackguard0` check in function prologues
- **When stack grows**: allocate 2x new stack, copy everything, update all pointers
- **Stack can shrink** during GC (if using < 1/4 of allocated)
- **Max stack**: 1GB (default)

---

## Scheduling

- Each P has a **local run queue** (256 G's max)
- **Global run queue** for overflow
- **Work stealing**: idle P steals half of another P's queue
- **Schedule points**: channel ops, mutex, syscalls, function calls, `runtime.Gosched()`

---

## Preemption

- **Go 1.14+**: asynchronous preemption via signals (`SIGURG`)
- **Before 1.14**: only cooperative (at function calls)
- Tight loops without function calls could block other goroutines (pre-1.14)

---

## Syscall Handling

- M entering syscall: **P detached**, can be picked up by another M
- If no idle M available: **new M created** (but limited by `GOMAXPROCS` for P's)
- Network I/O: uses **netpoller** (epoll/kqueue), doesn't block M

---

## Goroutine States

- `_Gidle` -> `_Grunnable` -> `_Grunning` -> `_Gwaiting` -> `_Grunnable` -> ...
- `_Gsyscall`: blocked in syscall
- `_Gdead`: finished, cached for reuse

---

## Goroutine Lifecycle

```go
go func() { ... }()  // -> runtime.newproc()
```

1. Allocate G struct (or reuse from free list)
2. Allocate 2KB stack
3. Set up `sched` (PC = function entry)
4. Put on current P's local run queue
5. If idle P exists, wake an M to run it

---

## Related

- [[Channel]] — goroutine parking/waking
- [[Sync]] — sync primitives interact with scheduler
- [[06-Memory-Management/Stack-vs-Heap]] — goroutine stack is on heap!
