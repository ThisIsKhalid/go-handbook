# Channel Internals

> Channels are Go's primary synchronization primitive -- built on a ring buffer with mutex protection and goroutine wait queues.

Channels are the backbone of Go's concurrency model. Under the hood, a channel is a runtime struct (`hchan`) that combines a **circular buffer** for data, a **mutex** for thread safety, and **wait queues** for parking goroutines that cannot proceed. Understanding these internals explains why channels behave the way they do -- and when to use alternatives.

![[Channel-internals.excalidraw]]

---

## The hchan Struct

Every channel created with `make(chan T)` or `make(chan T, n)` allocates an `hchan` on the heap:

```go
type hchan struct {
    qcount   uint           // current number of elements in buffer
    dataqsiz uint           // size of circular buffer (cap)
    buf      unsafe.Pointer // pointer to circular buffer array
    elemsize uint16         // size of each element
    closed   uint32         // closed flag
    elemtype *_type         // element type
    sendx    uint           // send index in circular buffer
    recvx    uint           // receive index in circular buffer
    recvq    waitq          // list of goroutines waiting to receive
    sendq    waitq          // list of goroutines waiting to send
    lock     mutex          // protects all fields
}
```

- `make(chan int, 5)` allocates an `hchan` + a contiguous buffer for 5 ints.
- `make(chan int)` (unbuffered): `dataqsiz = 0`, no buffer allocated -- sends and receives must rendezvous.
- The `lock` is a lightweight runtime mutex (not `sync.Mutex`), optimized for short critical sections.

```go
ch := make(chan int, 5)
// hchan: { qcount:0, dataqsiz:5, buf:0xc000..., elemsize:8, sendx:0, recvx:0 }
```

---

## Circular Buffer

The buffer is a fixed-size array allocated alongside the `hchan`. It works as a **ring buffer** using `sendx` and `recvx` as cursors:

```
buf (cap=5):  [ A ][ B ][ C ][ _ ][ _ ]
                         ^sendx    ^recvx wraps around

sendx = (sendx + 1) % dataqsiz
recvx = (recvx + 1) % dataqsiz
```

- `buf` points to an array of `dataqsiz` elements, each `elemsize` bytes.
- `sendx` advances when a value is sent into the channel.
- `recvx` advances when a value is received from the channel.
- Both indices wrap around using modular arithmetic: `sendx = (sendx + 1) % dataqsiz`.
- When `qcount == dataqsiz` (buffer full): the sending goroutine is parked in `sendq`.
- When `qcount == 0` (buffer empty): the receiving goroutine is parked in `recvq`.

For unbuffered channels (`dataqsiz = 0`), the buffer is never used -- data is always transferred directly between goroutines.

---

## Wait Queues (sudog)

Each channel has two wait queues -- doubly-linked lists of `sudog` structs:

```go
type waitq struct {
    first *sudog
    last  *sudog
}

type sudog struct {
    g     *g              // goroutine
    elem  unsafe.Pointer  // data to send/receive
    next  *sudog
    prev  *sudog
    // ... other fields
}
```

- When the channel is **full**: a sender creates a `sudog` containing a pointer to its data, then parks itself in `sendq`. The goroutine yields control to the scheduler.
- When the channel is **empty**: a receiver creates a `sudog`, parks in `recvq`, and yields.
- **Direct send optimization**: if a receiver is already waiting in `recvq` when a send occurs, the runtime copies data **directly** from the sender to the receiver's stack -- bypassing the buffer entirely. This is faster and avoids two copies.

```go
// Direct send: sender copies directly to waiting receiver
// Normal: sender → buf[sendx] → buf[recvx] → receiver  (2 copies)
// Direct: sender → receiver                              (1 copy)
```

---

## Send Operation

When you write `ch <- value`, the runtime executes `chansend()`:

1. **Lock** the channel mutex.
2. **If a receiver is waiting** in `recvq` -- dequeue the `sudog`, copy data directly to the receiver's `elem`, wake the receiver goroutine. **Done.**
3. **If buffer has space** (`qcount < dataqsiz`) -- copy the value to `buf[sendx]`, increment `sendx` (with wrap), increment `qcount`. **Done.**
4. **If buffer is full** -- create a `sudog` for the current goroutine, enqueue it in `sendq`, release the lock, and **park** (yield the goroutine to the scheduler).

```go
ch := make(chan int, 2)
ch <- 1  // step 3: buf=[1,_], sendx=1
ch <- 2  // step 3: buf=[1,2], sendx=0 (wraps)
ch <- 3  // step 4: buffer full, goroutine parks in sendq
```

---

## Receive Operation

When you write `value := <-ch`, the runtime executes `chanrecv()`:

1. **Lock** the channel mutex.
2. **If a sender is waiting** in `sendq` AND buffer is full -- receive from `buf[recvx]`, copy the sender's data into `buf[sendx]`, wake the sender. This maintains FIFO order.
3. **If buffer has data** (`qcount > 0`) -- copy from `buf[recvx]`, increment `recvx` (with wrap), decrement `qcount`. **Done.**
4. **If buffer is empty** -- create a `sudog` for the current goroutine, enqueue in `recvq`, release the lock, and **park**.

```go
ch := make(chan int, 2)
ch <- 10
ch <- 20

v := <-ch  // step 3: v=10, buf=[_,20], recvx=1
v = <-ch   // step 3: v=20, buf=[_,_], recvx=0 (wraps)
v = <-ch   // step 4: buffer empty, goroutine parks in recvq
```

---

## Select Statement

The `select` statement compiles down to `selectgo()` in the runtime:

```go
select {
case v := <-ch1:
    // ...
case ch2 <- val:
    // ...
default:
    // ...
}
```

How `selectgo()` works:

1. **Randomize** the case evaluation order (to prevent starvation).
2. **Lock ALL** channels involved, sorted by memory address to avoid deadlock.
3. **Check all cases** -- if any channel is ready, execute that case, unlock all, done.
4. **If none ready and `default` exists** -- execute `default`, unlock all, done.
5. **If none ready and no `default`** -- create a `sudog` for the goroutine, enqueue it in **every** channel's wait queue, unlock all, park.
6. **When woken** -- determine which case triggered, **dequeue from all other channels**, proceed.

The address-sorted locking prevents deadlocks when multiple goroutines `select` on overlapping sets of channels.

---

## Closing

When you call `close(ch)`, the runtime executes `closechan()`:

1. Set `closed = 1`.
2. **Wake ALL waiting receivers** in `recvq` -- they receive the zero value with `ok = false`.
3. **Wake ALL waiting senders** in `sendq` -- they **panic** with `"send on closed channel"`.

```go
ch := make(chan int, 1)
close(ch)

v, ok := <-ch  // v=0, ok=false -- receives zero value
ch <- 1         // PANIC: send on closed channel
close(ch)       // PANIC: close of closed channel
```

**Rules to remember:**

- Send to closed channel: **panics**.
- Receive from closed channel: returns zero value + `false`.
- Close an already-closed channel: **panics**.
- Close a nil channel: **panics**.
- The `closed` flag is a `uint32`, not a `bool`, to allow atomic reads without holding the lock for fast-path checks.

---

## Performance Notes

| Operation | Buffered (has space) | Buffered (full/empty) | Unbuffered |
|-----------|---------------------|-----------------------|------------|
| Send      | O(1), no park       | O(1) + park           | O(1) + park or direct |
| Receive   | O(1), no park       | O(1) + park           | O(1) + park or direct |
| Close     | O(n) wakeups        | O(n) wakeups          | O(n) wakeups |
| Select    | O(cases) + locking  | O(cases) + locking    | O(cases) + locking |

- Each send/receive involves **locking** the channel mutex -- contention under high concurrency.
- For high-throughput scenarios, consider buffered channels or `sync` primitives.
- The direct send/receive optimization makes unbuffered channels surprisingly efficient for rendezvous patterns.

---

## Related

- [[Goroutine]] -- channels coordinate goroutines
- [[Sync]] -- alternative synchronization primitives
