# Aria Memory Management Specification

**A complete specification for Aria's memory model: GC by default, manual control per-block, and the traits that bridge them.**

This document formalizes the memory management system sketched in [high-level-design.md](../high-level-design.md) (Pillar 4: Performance Is Opt-In Granular) and referenced throughout the spec directory. Aria's memory model is designed around one principle: **the default is safe and easy; manual control is available per-allocation when performance demands it.**

Cross-references:
- [high-level-design.md](../high-level-design.md) — `@stack`, `@arena`, `@inline`, memory control overview
- [spec/trait-system.md](trait-system.md) — `Drop`, `Clone`, `Send`, `Share` traits
- [spec/concurrency-design.md](concurrency-design.md) — move semantics for `spawn`, `Send`/`Share`
- [spec/ffi-design.md](ffi-design.md) — `@owned`, `@borrowed`, `@cffi` memory regions
- [spec/compiler-architecture.md](compiler-architecture.md) — escape analysis, ownership checking

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Default Mode: Garbage Collection](#2-default-mode-garbage-collection)
3. [The `@stack` Annotation](#3-the-stack-annotation)
4. [The `@arena` Annotation](#4-the-arena-annotation)
5. [The `@inline` Annotation](#5-the-inline-annotation)
6. [Pool Allocation](#6-pool-allocation)
7. [The `Drop` Trait](#7-the-drop-trait)
8. [Move Semantics and Ownership](#8-move-semantics-and-ownership)
9. [The `Clone` Trait](#9-the-clone-trait)
10. [Escape Analysis](#10-escape-analysis)
11. [FFI Memory](#11-ffi-memory)
12. [Concurrency and Memory](#12-concurrency-and-memory)
13. [Design Rationale Summary](#13-design-rationale-summary)
14. [Comparison with Other Languages](#14-comparison-with-other-languages)

---

## 1. Design Philosophy

Memory management is the eternal trade-off: safety vs. performance. GC languages (Go, Java) are safe but give up control. Manual languages (C, Rust) offer control but impose complexity on every function.

Aria rejects this binary choice. The default is a high-quality GC that handles 95% of code with zero annotations. The remaining 5% — hot paths, latency-sensitive code, embedded allocations — gets per-block manual control without changing languages, without `unsafe` blocks, and without a borrow checker.

### The Three Layers

| Layer | Annotation | Cost | Use Case |
|---|---|---|---|
| GC (default) | None | GC pauses, heap overhead | All general-purpose code |
| Stack | `@stack` | Compiler-verified lifetime | Short-lived values, tight loops |
| Arena | `@arena` | Bulk allocation | Request-scoped, batch processing |
| Inline | `@inline` | Embedded in parent | Cache-friendly data structures |
| Pool | `Pool[T]` | Reusable allocation | Hot-path object recycling |

**AI rationale:** The AI generates GC code by default — no annotations, no lifetime reasoning, no ownership complexity. When a performance annotation is needed, it's a single keyword (`@stack`, `@arena`, `@inline`) that the AI adds to an existing allocation without restructuring the surrounding code. This is fundamentally different from Rust, where choosing manual memory management restructures the entire function signature with lifetimes.

---

## 2. Default Mode: Garbage Collection

By default, all Aria allocations are managed by the garbage collector. No annotations are needed.

```
// All of these are GC-managed — zero annotations
x := Thing{...}
list := [1, 2, 3, 4, 5]
map := {"a": 1, "b": 2}
user := User{name: "Alice", email: "alice@example.com"}
```

### GC Characteristics

- **Concurrent** — the GC runs concurrently with application tasks; it does not stop the world for the full collection cycle
- **Generational** — new allocations are collected frequently (young generation); long-lived objects are collected less frequently (old generation)
- **Low-pause** — target pause times under 1ms for young-generation collections
- **Compacting** — the GC may compact the heap to reduce fragmentation (old generation only)

### GC tuning

```
// Runtime configuration (not language-level — set via environment or runtime API)
aria run --gc-target-pause=500us     // target pause time
aria run --gc-heap-max=4gb           // maximum heap size
aria run --gc-threads=4              // GC thread count
```

### What the GC manages

| Allocated by | GC-managed? |
|---|---|
| Default allocation (no annotation) | ✅ Yes |
| `@stack` allocation | ❌ No — compiler-managed |
| `@arena` allocation | ❌ No — arena-managed |
| `@inline` allocation | ❌ No — embedded in parent |
| `Pool[T]` allocation | ❌ No — pool-managed |
| FFI (`@cffi`) allocation | ❌ No — C-managed |

**AI rationale:** The GC is invisible to the AI in normal code generation. The AI writes allocations as bare expressions — `x := Thing{...}` — and the GC handles everything. This matches how the AI generates code in Go, Python, and Java, but with better pause characteristics.

---

## 3. The `@stack` Annotation

`@stack` allocates a value on the current function's stack frame. The value's lifetime is limited to the enclosing scope — it cannot escape.

```
fn processRequest(req: Request) -> Response {
    // Stack-allocated — freed when this function returns
    buffer := @stack Buffer.withCapacity(4096)

    // Use buffer normally
    buffer.write(req.body)
    parsed := parse(buffer)

    Response{data: parsed}
    // buffer is freed here — no GC involvement
}
```

### Lifetime rules

The compiler verifies that `@stack` values do not escape their scope:

```
fn bad() -> Buffer {
    buf := @stack Buffer.new()
    buf    // ❌ compile error: @stack value cannot escape function scope
}

fn also_bad() {
    buf := @stack Buffer.new()
    spawn process(buf)    // ❌ compile error: @stack value cannot cross task boundary
}
```

### When to use

- **Temporary buffers** — scratch space that lives for one function call
- **Small fixed-size data** — values that don't need heap allocation
- **Tight loops** — avoiding GC pressure in hot loops

```
fn sum_large_list(items: [f64]) -> f64 {
    mut total := 0.0
    for chunk in items.chunks(1000) {
        // Stack-allocated accumulator per chunk — no GC pressure
        acc := @stack Accumulator.new()
        for item in chunk { acc.add(item) }
        total += acc.result()
    }
    total
}
```

### Nested stack allocations

```
fn process() {
    a := @stack Thing{x: 1}     // allocated on stack
    {
        b := @stack Thing{x: 2} // also on stack, inner scope
        use(a, b)
    }
    // b is freed here
    use(a)
}
// a is freed here
```

**AI rationale:** `@stack` is the simplest performance annotation. The AI adds it to an existing allocation to avoid GC pressure — no restructuring needed. The compiler verifies the lifetime constraint, so the AI can't generate a use-after-free bug.

---

## 4. The `@arena` Annotation

`@arena` allocates values in a memory arena — a region that is freed all at once. This is ideal for request-scoped or batch-scoped work where many allocations share a lifetime.

```
fn handleRequest(req: Request) -> Response ! Error {
    // Create an arena for this request
    arena := Arena.new(64kb)
    defer arena.free()    // free everything at once when done

    // Allocate into the arena
    parsed := @arena(arena) parseRequest(req)?
    validated := @arena(arena) validate(parsed)?
    result := @arena(arena) process(validated)?

    // Build response (GC-allocated — outlives the arena)
    Response{data: result.toOwned()}
}
// arena.free() runs here — all @arena allocations freed in one operation
```

### Arena API

```
arena := Arena.new(64kb)       // initial capacity (grows if needed)
arena := Arena.new(1mb)        // larger arena
arena.free()                    // free all allocations at once
arena.reset()                   // reuse the arena's memory without freeing
arena.allocated() -> usize     // bytes currently allocated
arena.capacity() -> usize      // total capacity
```

### Allocation syntax

```
x := @arena(arena) Thing{...}
list := @arena(arena) [1, 2, 3]
```

### Lifetime rules

Arena-allocated values must not outlive the arena:

```
fn bad(arena: Arena) -> Thing {
    x := @arena(arena) Thing{...}
    x    // ⚠️ depends on arena lifetime — caller must ensure arena outlives the return value
}
```

The compiler tracks arena lifetimes and warns when arena-allocated values may escape. To safely return an arena-allocated value, copy it to the GC heap:

```
fn safe(arena: Arena) -> Thing {
    x := @arena(arena) Thing{...}
    x.toOwned()    // copies to GC heap — safe to return
}
```

### When to use

- **Request handlers** — allocate everything for one request into an arena, free at the end
- **Batch processing** — process a batch of records, free all working memory at once
- **Parsers** — AST nodes allocated into an arena, freed after processing

**AI rationale:** Arenas match the AI's natural reasoning about scope. "This function processes a request — allocate into an arena, free at the end" is a simple, mechanical transformation the AI can apply when told "optimize this handler."

---

## 5. The `@inline` Annotation

`@inline` embeds a value directly inside its parent struct, avoiding pointer indirection and a separate allocation.

```
type Response {
    headers: @inline Map[str, str]    // embedded, not heap-indirected
    body: [byte]
}
```

Without `@inline`, `headers` would be a pointer to a separately heap-allocated `Map`. With `@inline`, the map's storage is embedded directly in the `Response` struct — one allocation instead of two, and no pointer chase to access headers.

### When to use

- **Frequently accessed nested data** — eliminate pointer indirection on hot paths
- **Cache-friendly layouts** — keep related data contiguous in memory
- **Small fixed collections** — maps/lists that are always small enough to embed

```
type HttpClient {
    baseUrl: str
    headers: @inline Map[str, str]    // always accessed, keep inline
    timeout: dur
    retries: u8
}
```

### Restrictions

- `@inline` values cannot be shared by reference — they are part of their parent
- The parent struct becomes larger — consider whether the size increase is acceptable
- The `@inline` value is constructed and destroyed with its parent

**AI rationale:** `@inline` is a one-keyword optimization for a common performance pattern: eliminating pointer indirection in frequently accessed struct fields. The AI adds it to a field declaration — no restructuring needed.

---

## 6. Pool Allocation

`Pool[T]` provides a pool of reusable objects. Objects are borrowed from the pool, used, and returned — avoiding repeated allocation and deallocation.

```
pool := Pool[Buffer].new(
    size: 10,
    create: fn() => Buffer.withCapacity(4096),
    reset: fn(buf) { buf.clear() },     // optional: called on return
)

fn handleRequest(req: Request) -> Response {
    buf := pool.get()
    defer pool.put(buf)    // return to pool after function exits

    buf.write(req.body)
    processBuffer(buf)
}
```

### Pool API

```
pool := Pool[T].new(size: N, create: fn() -> T)
pool := Pool[T].new(size: N, create: fn() -> T, reset: fn(T))
obj := pool.get()              // borrow an object (blocks if pool empty)
obj := pool.tryGet()           // borrow or None if pool empty
pool.put(obj)                  // return an object to the pool
pool.size() -> u64             // current pool size
```

### When to use

- **Connection pools** — database connections, HTTP clients
- **Buffer pools** — reusable byte buffers for I/O
- **Object pools** — expensive-to-create objects on hot paths

See also: [spec/concurrency-design.md](concurrency-design.md) section 9.3 for `sync.Pool[T]`, the concurrent-safe pool variant.

---

## 7. The `Drop` Trait

The `Drop` trait provides deterministic cleanup — a `drop` method that runs when a value goes out of scope.

```
trait Drop {
    fn drop(mut ref self)
}
```

### Implementation

```
impl Drop for FileHandle {
    fn drop(mut ref self) {
        self.close()
    }
}

impl Drop for DatabaseConnection {
    fn drop(mut ref self) {
        self.disconnect()
    }
}
```

### Drop order

- Values are dropped in reverse declaration order (last declared, first dropped)
- Struct fields are dropped in declaration order
- `Drop` runs before the GC reclaims the memory

```
fn process() {
    a := FileHandle.open("a.txt")?     // dropped second
    b := FileHandle.open("b.txt")?     // dropped first
    // ... use a and b ...
}
// b.drop() runs, then a.drop()
```

### Drop and GC interaction

For GC-managed values that implement `Drop`:
1. The GC detects the value is unreachable
2. The runtime calls `drop()` before reclaiming the memory
3. `Drop` provides deterministic cleanup behavior, but the timing depends on when the GC collects

For `@stack` values: `drop()` is called at the end of the scope — fully deterministic.

For `@arena` values: `drop()` is called when `arena.free()` is called — deterministic at the arena level.

### The `with` statement for deterministic cleanup

For cases where deterministic timing matters, use `with`:

```
with file := fs.open("data.txt")? {
    content := file.readAll()?
    process(content)
}
// file.drop() runs here — deterministic, not dependent on GC
```

### `defer` for cleanup

```
fn process() {
    conn := db.connect(url)?
    defer conn.close()         // runs when function exits, regardless of how

    tx := conn.begin()?
    defer tx.rollback()        // runs if we don't commit

    tx.exec("INSERT ...")?
    tx.commit()?               // on success, commit before rollback defer
}
```

**AI rationale:** `Drop` + `with` + `defer` give the AI three levels of cleanup control. `Drop` is automatic and covers the common case. `with` is for scoped resources with deterministic timing. `defer` is for cleanup that must run at function exit. The AI picks the simplest one that satisfies the requirement.

---

## 8. Move Semantics and Ownership

Every value in Aria has a single owner. When a value is passed to a function, assigned to a variable, sent through a channel, or moved to a spawned task, ownership transfers.

### Assignment moves

```
a := Thing{...}
b := a          // a is moved to b
// a is no longer accessible — compile error if used
use(b)          // ✅ ok
use(a)          // ❌ compile error: a was moved
```

### Function arguments move

```
fn consume(thing: Thing) { ... }

x := Thing{...}
consume(x)      // x is moved into consume
// x is no longer accessible
```

### Channel sends move

```
data := buildData()
ch.send(data)    // data is moved into the channel
// data is no longer accessible
```

### Spawn moves

```
data := [1, 2, 3]
spawn processData(data)    // data is moved into the spawned task
// data is no longer accessible in this task
```

### When values don't move

The compiler optimizes common patterns:
- **Immutable values passed to functions that only read them** — the compiler may pass by reference instead of moving
- **Small values (primitives)** — copied instead of moved (optimization, not semantic change)
- **Values used after the function call** — the compiler checks and either errors or copies

The semantic model is always "move" — the optimizations are invisible to the programmer.

**AI rationale:** Move semantics prevent data races and use-after-free bugs by construction. The AI never generates code where two tasks access the same mutable data — the compiler rejects it. When the AI needs shared data, it must use `.clone()`, channels, or `sync` primitives — making the intent explicit.

---

## 9. The `Clone` Trait

`Clone` provides explicit deep copying. Aria never copies data implicitly — every copy is visible in the code.

```
trait Clone {
    fn clone(self) -> Self
}
```

### Usage

```
config := loadConfig()

scope {
    spawn taskA(config.clone())     // A gets its own copy
    spawn taskB(config.clone())     // B gets its own copy
    spawn taskC(config)             // C takes ownership of original
}
```

### Derivable

```
type User {
    name: str
    email: str
} derives [Clone]
```

All primitive types implement `Clone`. Structs can derive `Clone` if all fields implement `Clone`.

### Clone vs move

| Operation | What happens | When to use |
|---|---|---|
| Move (`x := y`) | Ownership transfers, original unusable | Default — most cases |
| Clone (`x := y.clone()`) | Deep copy, both usable | When you need two independent copies |

**AI rationale:** Explicit cloning means the AI never accidentally generates expensive hidden copies. Every `.clone()` in the code is a deliberate choice that the reviewer can evaluate. In Go, slice and map copies are silent and surprising — Aria makes them visible.

---

## 10. Escape Analysis

The compiler performs escape analysis to optimize allocations automatically. Values that don't escape their function can be stack-allocated even without `@stack`.

```
fn process(data: [i64]) -> i64 {
    // The compiler may stack-allocate this buffer automatically
    // because it doesn't escape the function
    buffer := Buffer.new()
    for item in data {
        buffer.write(item)
    }
    buffer.sum()
    // buffer doesn't escape — compiler can stack-allocate it
}
```

### What escape analysis does

1. Analyzes whether a value's reference can outlive the current function
2. If the value doesn't escape: allocates on the stack (no GC involvement)
3. If the value does escape: allocates on the GC heap (the default)

### Escape analysis is an optimization, not a guarantee

The programmer should not rely on escape analysis for correctness — it's a performance optimization. Use `@stack` when you need a guarantee that a value is stack-allocated.

**AI rationale:** Escape analysis means the AI's default GC code is often faster than expected — the compiler automatically avoids heap allocation for short-lived values. The AI doesn't need to manually optimize every allocation.

---

## 11. FFI Memory

When interacting with C libraries, memory follows different rules. See [spec/ffi-design.md](ffi-design.md) for the full FFI specification.

### `@cffi` — Non-moving memory region

C functions expect stable pointers. `@cffi` allocations are guaranteed not to move:

```
data := @cffi Buffer.new(1024)    // allocated in non-moving region
c_function(data.ptr())             // pointer remains valid until data is freed
```

### `@owned` — Aria owns the memory

```
extern "C" fn create_thing() -> *c.Thing @owned
// Aria takes ownership — will free when the value is dropped
```

### `@borrowed` — C owns the memory

```
extern "C" fn get_string(ctx: *c.Context) -> *const c.char @borrowed
// Aria borrows — must not free; valid only while ctx is alive
```

### Safety invariant

FFI memory annotations ensure that:
- Aria-owned C memory is freed exactly once
- C-owned memory is never freed by Aria
- Non-moving memory is never relocated by the GC
- Pointer stability is guaranteed for the duration of C calls

---

## 12. Concurrency and Memory

### `Send` and `Share` traits

Types that cross task boundaries must implement `Send`. Types that are shared between tasks must implement `Share`. See [spec/trait-system.md](trait-system.md) section 9 and [spec/concurrency-design.md](concurrency-design.md) section 8.

```
// Send: safe to move to another task
spawn process(data)     // data must be Send

// Share: safe to share between tasks
scope {
    spawn read(config)   // config must be Share (if accessed by multiple tasks)
    spawn read(config)
}
```

### Cross-task GC

The GC operates across all tasks — there is one heap shared by all tasks in the process. GC collections may pause all tasks briefly (concurrent collection minimizes this).

`@stack` and `@arena` values are task-local — they are not visible to other tasks and don't participate in cross-task GC.

---

## 13. Design Rationale Summary

| Decision | Rationale |
|---|---|
| GC by default | 95% of code doesn't need manual memory management; GC eliminates use-after-free and double-free bugs |
| `@stack` for compiler-verified stack allocation | Simple escape hatch for tight loops and temporary buffers |
| `@arena` for bulk allocation | Request-scoped memory management without individual free calls |
| `@inline` for embedded allocation | Cache-friendly layouts without changing allocation patterns |
| `Pool[T]` for object reuse | Hot-path optimization for expensive-to-create objects |
| `Drop` for deterministic cleanup | Resource management (files, connections) with guaranteed cleanup |
| Move semantics by default | Prevents data races and use-after-free at compile time |
| Explicit `Clone` | No hidden copies — every copy is visible and reviewable |
| Escape analysis | Automatic optimization — GC code is often stack-allocated transparently |
| FFI memory annotations | Safety guarantees at the C boundary without runtime overhead |

---

## 14. Comparison with Other Languages

| Feature | Go | Rust | Java | C | Aria |
|---|---|---|---|---|---|
| Default allocation | GC heap | Stack (moves) | GC heap | Manual (`malloc`) | **GC heap** |
| Manual stack allocation | No | Default for locals | No | `alloca` | **`@stack`** |
| Arena/region allocation | No (custom) | No (crates) | No | Manual | **`@arena`** |
| Inline/embedded allocation | No | Default for struct fields | No | Default for struct fields | **`@inline`** |
| Object pooling | `sync.Pool` | No (crates) | No (libraries) | Manual | **`Pool[T]`** |
| Deterministic cleanup | `defer` | `Drop` trait (RAII) | `try-with-resources` | Manual | **`Drop` + `with` + `defer`** |
| Move semantics | No (copies) | Yes (default) | No (references) | No | **Yes (default)** |
| Explicit copy | No (implicit) | `.clone()` | `.clone()` | `memcpy` | **`.clone()`** |
| Escape analysis | Yes | N/A (no GC) | Yes | N/A | **Yes** |
| Concurrency safety | Race detector (runtime) | Borrow checker (compile) | Locks (runtime) | None | **Send/Share (compile)** |

### The key difference from Rust

Rust requires lifetime annotations on every function that borrows data. This adds tokens to every signature and forces the AI to reason about lifetimes at every function boundary. Aria's GC handles the common case silently — only the 5% of code that needs manual control gets annotations, and those annotations (`@stack`, `@arena`) are simpler than Rust's lifetime system.

### The key difference from Go

Go's GC is the only option — there's no escape hatch for hot paths. Aria provides `@stack`, `@arena`, `@inline`, and `Pool[T]` for the cases where GC pause times or allocation pressure matter, without leaving the language.

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/trait-system.md](trait-system.md), [spec/concurrency-design.md](concurrency-design.md), and [spec/ffi-design.md](ffi-design.md).*
