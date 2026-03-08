# Go Data Structures Deep Dive

> Interactive visual guide to how Go works under the hood.
> Every data structure explained with **Excalidraw diagrams** showing memory layout, internal structs, and runtime behavior.

---

## 01 — Fundamentals

| Topic | What You'll Learn |
|---|---|
| [[01-Fundamentals/String]] | `StringHeader{Data *byte, Len int}`, UTF-8 encoding, immutability, string interning |
| [[01-Fundamentals/Array]] | Contiguous memory, value semantics, copy-on-assign, compile-time size |
| [[01-Fundamentals/Slice]] | `SliceHeader{Data, Len, Cap}`, backing array, `append()` growth strategy, slice tricks |
| [[01-Fundamentals/Pointer]] | Stack vs heap addresses, `unsafe.Pointer`, `uintptr`, pointer arithmetic |

## 02 — Composite Types

| Topic | What You'll Learn |
|---|---|
| [[02-Composite-Types/Struct]] | Memory alignment, padding, field ordering optimization, struct embedding |
| [[02-Composite-Types/Map]] | `hmap` → buckets → `bmap`, hash function, overflow chains, incremental evacuation |
| [[02-Composite-Types/Function-Closure]] | Stack frames, closure capture (by reference), `funcval` struct, defer stack |

## 03 — Interface System

| Topic | What You'll Learn |
|---|---|
| [[03-Interface-System/Interface]] | `eface` vs `iface`, `itab` caching, type assertion cost, interface satisfaction |

## 04 — Concurrency

| Topic | What You'll Learn |
|---|---|
| [[04-Concurrency/Channel]] | `hchan` struct, circular ring buffer, `sendq`/`recvq` sudog queues, select |
| [[04-Concurrency/Goroutine]] | G/M/P model, goroutine stack (2KB→1GB), stack copying, work stealing, preemption |

## 05 — Sync Primitives

| Topic | What You'll Learn |
|---|---|
| [[05-Sync-Primitives/Sync]] | `sync.Mutex` (normal vs starvation), `RWMutex`, `WaitGroup`, `Once`, `Pool`, `sync.Map` |

## 06 — Memory Management

| Topic | What You'll Learn |
|---|---|
| [[06-Memory-Management/Stack-vs-Heap]] | Stack allocation, heap escape, `go build -gcflags="-m"` |
| [[06-Memory-Management/Escape-Analysis]] | When and why values escape to heap, common escape patterns |
| [[06-Memory-Management/GC]] | Tri-color mark-and-sweep, write barriers, GC pacing, `GOGC` tuning |

---

## How to Use This Vault

1. **Open in Obsidian** — Install the [Excalidraw plugin](https://github.com/zsviczian/obsidian-excalidraw-plugin) to view interactive diagrams
2. **Follow the links** — Each topic links to related concepts (e.g., Slice → Array → Memory)
3. **Zoom into diagrams** — Excalidraw diagrams are interactive: pan, zoom, and explore memory layouts
4. **Run the code** — Each topic includes Go code snippets you can copy and run

## Color Legend (Diagrams)

| Color | Meaning |
|---|---|
| 🔵 Blue `#a5d8ff` | Runtime struct headers (SliceHeader, hmap, hchan) |
| 🟢 Green `#b2f2bb` | Actual data / memory contents |
| 🟡 Yellow `#ffec99` | Pointers / references |
| 🔴 Red `#ffc9c9` | Important warnings / gotchas |
| 🟣 Purple `#d0bfff` | Goroutines / concurrency primitives |
| ⚪ Gray `#dee2e6` | Unused / available capacity |
