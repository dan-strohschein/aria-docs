# Aria Garbage Collector Specification

**A complete specification for Aria's generational garbage collector with per-task nurseries, concurrent old-generation collection, and integration with Aria's manual memory annotations.**

This document defines the GC algorithm, its interaction with Aria's memory model, and the diagnostics it provides. The design exploits Aria's unique combination of per-task concurrency, immutable-by-default values, and manual memory annotations (`@stack`, `@arena`, `@inline`) to achieve allocation speeds and pause times that exceed Go and match Java's best collectors — with less GC traffic overall.

Cross-references:
- [spec/memory-management.md](memory-management.md) — `@stack`, `@arena`, `@inline`, `Drop`, move semantics
- [spec/concurrency-design.md](concurrency-design.md) — task model, `spawn`, `scope`, `Send`/`Share`
- [spec/trait-system.md](trait-system.md) — `Drop`, `Clone` traits
- [spec/compiler-architecture.md](compiler-architecture.md) — compilation pipeline, runtime components
- [spec/design-decisions-v01.md](design-decisions-v01.md) — bootstrap compiler in Go

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Architecture Overview](#2-architecture-overview)
3. [Per-Task Nurseries](#3-per-task-nurseries)
4. [Nursery Collection](#4-nursery-collection)
5. [Old Generation](#5-old-generation)
6. [Old Generation Collection](#6-old-generation-collection)
7. [Write Barriers](#7-write-barriers)
8. [GC-Safe Points](#8-gc-safe-points)
9. [Integration with Aria Features](#9-integration-with-aria-features)
10. [GC Configuration](#10-gc-configuration)
11. [Diagnostics and Profiling](#11-diagnostics-and-profiling)
12. [Performance Characteristics](#12-performance-characteristics)
13. [Bootstrap GC](#13-bootstrap-gc)
14. [Design Rationale Summary](#14-design-rationale-summary)
15. [Comparison with Other Languages](#15-comparison-with-other-languages)

---

## 1. Design Philosophy

Aria's GC is designed around one observation: **most allocations in Aria never reach the GC at all.**

The language provides `@stack` for temporaries, `@arena` for request-scoped work, `@inline` for embedded fields, move semantics for ownership transfer, and escape analysis for automatic stack promotion. These features divert the majority of allocations away from the GC heap. The GC manages only the **long-lived, shared, or escaping** subset — and it manages that subset extremely well.

### The three performance goals

1. **Allocation must be as fast as a stack push** — a bump pointer in a per-task nursery achieves ~2-3ns per allocation with zero contention
2. **Young collection must be invisible to other tasks** — per-task nursery collection pauses only the allocating task, not the entire program
3. **Old collection must never stop the world for more than 1ms** — concurrent marking and incremental compaction keep pause times bounded

### Why per-task nurseries

Aria's concurrency model is task-based. Tasks are the unit of execution, scheduling, and — with this design — allocation. Each task gets its own nursery, which means:

- Allocation requires no synchronization (single-threaded within each task)
- Young collection is per-task (other tasks keep running)
- The GC model aligns perfectly with the execution model

This is the key architectural insight. Go uses per-P (processor) caches shared across goroutines. Java uses per-thread TLABs tied to OS threads. Aria ties allocation to tasks — the natural unit of work.

**AI rationale:** When I generate code that allocates heavily inside a `scope` with many spawned tasks, each task's allocations are independent. One task's allocation rate doesn't slow down another task's allocation. I don't need to worry about GC contention between tasks — it doesn't exist at the young generation level.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                 Shared Old Generation                  │
│            (concurrent mark-compact, region-based)     │
│                                                        │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │
│  │Region 1│ │Region 2│ │Region 3│ │Region 4│ ...     │
│  │ (2MB)  │ │ (2MB)  │ │ (2MB)  │ │ (2MB)  │        │
│  └────────┘ └────────┘ └────────┘ └────────┘        │
└───────▲──────────▲──────────▲──────────▲─────────────┘
        │          │          │          │  promote
        │          │          │          │  surviving
        │          │          │          │  objects
┌───────┴───┐ ┌────┴─────┐ ┌─┴────────┐ ┌┴──────────┐
│  Task 1   │ │  Task 2  │ │  Task 3  │ │  Task 4  │
│  Nursery  │ │  Nursery │ │  Nursery │ │  Nursery │
│  (512KB)  │ │  (512KB) │ │  (512KB) │ │  (512KB) │
│ bump ptr  │ │ bump ptr │ │ bump ptr │ │ bump ptr │
└───────────┘ └──────────┘ └──────────┘ └──────────┘
```

### Two generations

| Generation | Location | Collection | Pause scope |
|---|---|---|---|
| **Young (nursery)** | Per-task, private | Copy surviving objects to old gen | This task only |
| **Old** | Shared across all tasks | Concurrent mark, incremental compact | Brief global pauses for root scanning |

### What lives where

| Allocation type | Where it lives |
|---|---|
| Default (`x := Thing{...}`) | Task nursery → promoted to old gen if it survives |
| `@stack` | Stack frame — never touches GC |
| `@arena(arena)` | Arena region — never touches GC |
| `@inline` field | Embedded in parent object — one GC object |
| `@cffi` | Non-moving C region — never touches GC |
| String literals | Old gen (interned at compile time, never collected) |
| Constants | Inlined at usage sites — no runtime allocation |

---

## 3. Per-Task Nurseries

Each Aria task receives its own nursery — a contiguous block of memory for young allocations.

### Allocation: bump pointer

Allocating in a nursery is a single pointer increment:

```
// Pseudocode — what the compiler generates for `x := Thing{field1: a, field2: b}`
fn nursery_alloc(task: *Task, size: usize) -> *void {
    ptr := task.nursery.cursor
    new_cursor := ptr + size
    if new_cursor > task.nursery.end {
        // Nursery full — trigger young collection
        nursery_collect(task)
        ptr = task.nursery.cursor
        new_cursor = ptr + size
    }
    task.nursery.cursor = new_cursor
    ptr
}
```

This is ~2-3 nanoseconds per allocation. No locks. No CAS. No free-list search. Just increment a pointer.

### Nursery sizing

| Parameter | Default | Configurable? |
|---|---|---|
| Initial nursery size | 512KB | Yes (`--gc-nursery=SIZE`) |
| Maximum nursery size | 4MB | Yes (`--gc-nursery-max=SIZE`) |
| Growth trigger | Nursery fills before 80% of objects die | Automatic |
| Shrink trigger | Nursery consistently <20% used at collection | Automatic |

### Nursery lifecycle

1. **Task created** — nursery allocated (512KB default)
2. **Allocations** — bump pointer increments within the nursery
3. **Nursery full** — young collection: copy survivors to old gen, reset nursery
4. **Task exits** — nursery freed entirely (no individual object deallocation)

### Task-local allocation means zero contention

Since each task has its own nursery, multiple tasks can allocate simultaneously without any synchronization. This is the fundamental performance advantage over Go's allocator (which uses per-P mcaches with occasional lock contention) and over traditional malloc (which requires thread synchronization).

```
// Each task allocates independently — zero contention
scope {
    // Task 1 allocates into nursery 1
    spawn {
        for _ in 0..1000 { data.append(Thing{...}) }
    }
    // Task 2 allocates into nursery 2
    spawn {
        for _ in 0..1000 { data.append(Thing{...}) }
    }
    // No synchronization between nurseries
}
```

---

## 4. Nursery Collection

When a task's nursery fills up, the GC collects it. Only the allocating task pauses — all other tasks continue running.

### Collection algorithm: semi-space copy

1. **Scan roots** — the task's stack and registers contain the root references
2. **Trace reachable objects** — follow references from roots through the nursery
3. **Copy survivors to old generation** — objects that are still reachable are copied to the shared old generation
4. **Reset nursery** — the entire nursery is reclaimed by resetting the bump pointer to the start

Dead objects cost nothing — they are simply overwritten when the nursery is reused. Only surviving objects pay the cost of being copied.

### Survival rate

In a well-designed generational GC, the vast majority of nursery objects die young. Typical survival rates:

| Workload | Survival rate | Notes |
|---|---|---|
| Web server request | 1-5% | Most objects are request-scoped temporaries |
| Data processing pipeline | 5-15% | Some objects accumulate in output collections |
| Long-running computation | 10-20% | More objects feed into long-lived data structures |

Aria's `@arena` and `@stack` annotations further reduce survival rates — temporaries that would have been nursery objects in other languages never enter the nursery at all.

### Nursery collection pause time

A nursery collection pauses only the collecting task. The pause time is proportional to the number of **surviving** objects (not the nursery size):

```
Pause ≈ (surviving_objects × copy_cost) + root_scan_cost
```

For a 512KB nursery with 5% survival rate:
- ~25KB of data copied to old generation
- Root scanning: ~10-50μs
- Total pause: **<100μs** (0.1ms)

This is a single-task pause — other tasks are unaffected.

---

## 5. Old Generation

The old generation is a shared heap that holds objects promoted from nurseries. It uses a region-based layout.

### Region structure

The old generation is divided into fixed-size regions (default: 2MB each):

```
┌────────┬────────┬────────┬────────┬────────┬────────┐
│Region 0│Region 1│Region 2│Region 3│Region 4│Region 5│ ...
│ (2MB)  │ (2MB)  │ (2MB)  │ (2MB)  │ (2MB)  │ (2MB)  │
│        │ 80%    │ 15%    │ 95%    │ 40%    │ 70%    │
│ (full) │ (live) │ (live) │ (live) │ (live) │ (live) │
└────────┴────────┴────────┴────────┴────────┴────────┘
```

Each region tracks its liveness ratio — the fraction of bytes that are live (reachable) objects. Regions with low liveness are candidates for compaction.

### Promotion

When a nursery object survives collection, it is copied into the old generation:

1. Find a region with free space (or allocate a new region)
2. Copy the object into the region
3. Update all references to the object (forwarding pointer in the nursery)

### Large object space

Objects larger than half a region size (default: >1MB) are allocated directly in the old generation, skipping the nursery. These are rare in practice — most large allocations in Aria use `@arena` or `@stack`.

---

## 6. Old Generation Collection

Old generation collection runs **concurrently** with application tasks. It never stops all tasks for the entire collection — only brief pauses for root scanning.

### Phase 1: Concurrent marking

The GC traces all reachable objects from roots (task stacks, global variables, module-level state) while tasks continue running:

1. **Initial root scan** — brief pause (~100-500μs) to snapshot all task stack roots
2. **Concurrent trace** — GC threads follow references from roots, marking reachable objects. Tasks continue running.
3. **Re-mark** — brief pause (~50-200μs) to process objects mutated during concurrent tracing (caught by write barriers)

### Phase 2: Incremental compaction

After marking, the GC knows which objects are live and which regions have low liveness. It compacts selectively:

1. **Select regions** — choose regions with the lowest liveness ratio (most garbage)
2. **Evacuate live objects** — copy live objects from selected regions to fresh regions
3. **Update references** — update all pointers to the evacuated objects
4. **Free emptied regions** — return the fully evacuated regions to the free pool

Compaction is incremental — only a few regions are compacted per cycle. This bounds the work per cycle and keeps pause times predictable.

### Compaction scheduling

| Old gen occupancy | Action |
|---|---|
| < 50% | No compaction needed — plenty of free space |
| 50-75% | Compact regions with <30% liveness |
| 75-90% | Compact regions with <50% liveness (more aggressive) |
| > 90% | Emergency: compact all regions, grow the heap |

### Old generation pause times

| Phase | Duration | Scope |
|---|---|---|
| Initial root scan | 100-500μs | All tasks briefly paused |
| Concurrent trace | 0ms (no pause) | GC threads only |
| Re-mark | 50-200μs | All tasks briefly paused |
| Incremental compaction | 0ms (concurrent) | GC threads only |
| **Total stop-the-world** | **<1ms** | **Two brief pauses per cycle** |

---

## 7. Write Barriers

Write barriers are bookkeeping operations that track when a reference is stored into an object. The GC needs write barriers to know about mutations that happen during concurrent marking.

### When write barriers are needed

| Situation | Write barrier? | Why |
|---|---|---|
| Immutable binding stores reference | No | Value is set once at construction — GC sees it during marking |
| `mut` binding reassigned | Yes (if stores a reference) | New reference must be seen by concurrent marker |
| Mutable struct field written | Yes (if stores a reference) | Same reason |
| Nursery object stores reference | No | Nursery is collected independently, not marked |
| Old gen object stores reference to nursery | Yes (remembered set) | Old-to-young reference must be tracked |

### Aria's advantage: fewer write barriers

Because Aria is immutable by default, the majority of objects are set once at construction and never mutated. These objects need **zero write barriers** — the GC sees all their references during marking.

Only `mut` bindings that store reference-typed values trigger write barriers. In typical Aria code, this is a small fraction of all stores.

### Barrier implementation

Aria uses a card-marking barrier (like Java's G1):

```
// Pseudocode — generated by compiler for `mut_ref.field = new_value`
fn write_barrier(object: *Object, field_offset: usize, new_value: *Object) {
    // Mark the card (512-byte region) as dirty
    card_index := (object as usize) / 512
    card_table[card_index] = DIRTY

    // If storing a young reference into an old object, add to remembered set
    if is_old(object) && is_young(new_value) {
        remembered_set.add(object, field_offset)
    }

    // Perform the actual write
    *(object + field_offset) = new_value
}
```

The card table is a byte array where each byte covers 512 bytes of heap. Marking a card dirty is a single byte store — very cheap.

---

## 8. GC-Safe Points

Old generation collection requires brief pauses where all tasks reach a consistent state (safe point) so the GC can scan roots. Tasks must reach safe points promptly.

### Where safe points are inserted

The compiler inserts safe point checks at:

| Location | Why |
|---|---|
| Function entry | Every function call is a safe point |
| Loop back-edges | Prevents long-running loops from blocking GC |
| Channel operations (`send`/`recv`) | Natural yield points |
| `spawn` and `scope` entry | Concurrency boundaries |
| `select` arms | Waiting on channels |
| `task.await()` | Blocking on task completion |

### Safe point check

A safe point check is a single conditional branch:

```
// Pseudocode — generated at every safe point
if gc_requested {
    park_and_wait_for_gc()
}
```

When no GC is requested (the common case), this is a single branch-not-taken — effectively free on modern CPUs with branch prediction.

### Preemption for uncooperative tasks

If a task doesn't reach a safe point within a timeout (e.g., tight computational loop without function calls), the runtime can use OS signals (like Go 1.14+) to asynchronously preempt the task and force it to a safe point.

---

## 9. Integration with Aria Features

### `@stack` values

Stack-allocated values are invisible to the GC. They live on the task's stack frame and are freed when the frame exits. The GC never traces, marks, or moves them.

However, if a `@stack` value contains a reference to a GC-managed object, that reference is a GC root. The GC must scan task stacks during root scanning to find these references.

### `@arena` values

Arena-allocated values are invisible to the GC. They live in the arena and are freed by `arena.free()`. Like `@stack`, arena values that contain references to GC objects create roots that must be scanned.

### `@inline` fields

Inline fields are embedded in their parent object. From the GC's perspective, the parent is a single object with a larger size. The inline field's references are traced as part of the parent.

### `Drop` trait

When the GC determines an object is unreachable and the object implements `Drop`:

1. The object is placed on a finalization queue (not immediately collected)
2. A finalization thread calls `drop()` on the object
3. After `drop()` returns, the object is eligible for collection in the next cycle

For `@stack` and `with`-scoped values, `drop()` is called deterministically at scope exit — the GC is not involved.

### Move semantics and `spawn`

When a value is moved into a spawned task, ownership transfers. If the value was in the spawner's nursery, it is either:
- Copied to the new task's nursery (if small)
- Promoted to the old generation (if the new task's nursery doesn't have space)

The original reference in the spawner is invalidated at compile time (move semantics), so no dangling reference exists.

### Immutable values

Immutable values (the default) are never mutated after construction. This means:
- No write barriers for immutable objects
- The GC can safely read immutable objects without synchronization during concurrent marking
- Immutable objects can be freely shared across tasks (they implement `Share` automatically)

---

## 10. GC Configuration

### Runtime flags

```
aria run --gc-nursery=1mb              # per-task nursery size (default: 512KB)
aria run --gc-nursery-max=8mb          # maximum nursery size after growth (default: 4MB)
aria run --gc-target-pause=500us       # target old-gen pause time (default: 1ms)
aria run --gc-heap-max=4gb             # maximum heap size (default: unlimited)
aria run --gc-threads=4                # GC thread count (default: CPU count / 4)
aria run --gc-region-size=4mb          # old gen region size (default: 2MB)
```

### Environment variables

```
ARIA_GC_NURSERY=1mb
ARIA_GC_HEAP_MAX=4gb
ARIA_GC_THREADS=4
```

### Programmatic access

```
use std.runtime

// Query GC statistics
stats := runtime.gcStats()
println("Total collections: {stats.collections}")
println("Total pause time: {stats.totalPause}")
println("Heap size: {stats.heapSize}")

// Force a collection (for testing/benchmarking — not recommended in production)
runtime.gc()
```

---

## 11. Diagnostics and Profiling

### GC statistics at exit

```
aria run --gc-stats
```

Output:
```
GC Statistics:
  Nursery collections:  12,450
  Avg nursery pause:    0.04ms
  Max nursery pause:    0.12ms
  Old gen collections:  23
  Avg old gen pause:    0.35ms (two phases)
  Max old gen pause:    0.82ms
  Total GC time:        1.2s (0.8% of runtime)
  Peak heap size:       256MB
  Total allocated:      18.4GB
  Promoted to old gen:  890MB (4.8% of allocated)
```

### GC event log

```
aria run --gc-log
```

Outputs a line per GC event:
```
[GC] nursery task=47 pause=0.03ms survived=12KB promoted=12KB
[GC] nursery task=12 pause=0.05ms survived=28KB promoted=28KB
[GC] old-gen phase=mark pause=0.28ms traced=45MB
[GC] old-gen phase=remark pause=0.08ms updated=1204
[GC] old-gen phase=compact regions=3 freed=6MB pause=0ms (concurrent)
```

### Allocation profiling

```
aria profile --alloc myprogram
```

Output:
```
Allocation Profile (top 10 by bytes):

  45.2MB  38%  server.handleRequest     src/server.aria:42
  18.7MB  16%  json.parse               std/json.aria:156
  12.1MB  10%  db.query                 src/db.aria:78
   8.3MB   7%  str.concat (interpolation) (built-in)
   5.1MB   4%  http.Response.new        std/http.aria:201
   ...

Total: 118.4MB across 2,340,000 allocations
```

### GC profiling (retention)

```
aria profile --gc myprogram
```

Shows what's keeping objects alive in the old generation:

```
Old Generation Retention (top 10 by bytes):

  12.3MB  cache: Map[str, Response]     src/cache.aria:15
          ← held by module-level variable `responseCache`

   8.1MB  [Connection]                  src/pool.aria:8
          ← held by Pool[Connection].items

   5.4MB  [User]                        src/session.aria:22
          ← held by sessionStore.activeSessions
  ...
```

**AI rationale:** When I generate code that leaks memory or causes GC pressure, these profiling tools tell me exactly where the problem is. `aria profile --alloc` shows me which function to add `@stack` or `@arena` to. `aria profile --gc` shows me which data structure is holding onto objects too long. One round of profiling → one targeted fix.

---

## 12. Performance Characteristics

### Allocation speed comparison

| Allocator | Cost per allocation | Contention | Notes |
|---|---|---|---|
| C `malloc` | 20-50ns | Moderate (lock) | Free-list search |
| Go (mcache) | ~25ns | Low (per-P) | Size-class lookup |
| Java (TLAB) | ~5ns | None (per-thread) | Bump pointer in TLAB |
| Rust (no GC) | 20-50ns | Moderate (global allocator) | `malloc` or custom |
| **Aria nursery** | **~2-3ns** | **None (per-task)** | Bump pointer, zero sync |

### Collection pause comparison

| GC | Young pause | Old pause | Stop-the-world? |
|---|---|---|---|
| Go | N/A (no generations) | 0.1-1ms | Brief STW for marking |
| Java G1 | 5-20ms | 10-50ms (mixed) | Yes (young), brief (old) |
| Java ZGC | <1ms | <1ms | Sub-ms STW phases |
| **Aria** | **<0.1ms (per-task)** | **<1ms (two brief phases)** | **Per-task young; brief global old** |

### Throughput overhead

| GC | % CPU on GC | Why |
|---|---|---|
| Go | 5-10% | Scans entire heap (no generations) |
| Java G1 | 3-8% | Generational but expensive barriers |
| Java ZGC | 3-5% | Low pause but higher memory overhead |
| **Aria** | **2-5%** | Less traffic (stack/arena divert), fewer barriers (immutable default) |

---

## 13. Bootstrap GC

The bootstrap compiler (written in Go) does not need the full GC described above. For the bootstrap phase:

### Bootstrap GC: Go's built-in GC

The Aria runtime prototype can use Go's own garbage collector during bootstrapping:

- Go's GC is concurrent and low-pause — adequate for bootstrapping
- No need to implement a custom GC in Go
- The self-hosting compiler (written in Aria) will implement the real GC

### What the bootstrap runtime provides

| Feature | Bootstrap | Full |
|---|---|---|
| GC algorithm | Go's concurrent mark-sweep | Generational + per-task nurseries |
| Per-task nurseries | No (use Go's allocator) | Yes |
| Compaction | No (Go doesn't compact) | Yes (incremental) |
| Write barriers | Go's write barriers | Aria-specific card marking |
| `@stack` support | Compiler-verified, uses Go stack | Compiler-verified, native stack |
| `@arena` support | Custom arena allocator in Go | Native arena implementation |
| GC diagnostics | Basic (`--gc-stats`) | Full profiling suite |

The bootstrap GC is intentionally minimal. Its job is to compile enough Aria code to build the self-hosting compiler, which will implement the full GC spec.

---

## 14. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Per-task nurseries | Aligns allocation with Aria's task model — zero contention, zero synchronization |
| Bump-pointer allocation | Fastest possible allocation (~2-3ns) — just increment a pointer |
| Semi-space nursery copy | Dead objects cost nothing — only survivors pay the copy cost |
| Region-based old generation | Enables incremental compaction without full-heap pauses |
| Concurrent marking | Old-gen collection runs alongside application tasks |
| Incremental compaction | Compact a few regions per cycle — bounded, predictable work |
| Card-marking write barriers | Cheap (one byte store) and only needed for mutable references |
| Immutable default = fewer barriers | Aria's immutable-by-default means most stores need no barrier |
| Deterministic Drop for resources | GC handles memory; Drop/with/defer handle resources — clear separation |
| Auto-sizing nurseries | Nurseries grow if survival rate is high, shrink if underused |
| Two brief STW phases for old gen | Root scan + remark — each <500μs, total <1ms |
| Allocation profiling built-in | AI needs "where is memory going?" answered in one command |

---

## 15. Comparison with Other Languages

| Feature | Go | Java G1 | Java ZGC | Rust | Aria |
|---|---|---|---|---|---|
| Generations | No | Yes (young + old) | Yes (colored ptrs) | N/A (no GC) | **Yes (nursery + old)** |
| Per-thread/task allocation | Per-P mcache | TLAB (per-thread) | TLAB | N/A | **Per-task nursery** |
| Allocation speed | ~25ns | ~5ns | ~5ns | ~20-50ns | **~2-3ns** |
| Young pause | N/A | 5-20ms (STW) | <1ms | N/A | **<0.1ms (per-task)** |
| Old pause | 0.1-1ms | 10-50ms | <1ms | N/A | **<1ms** |
| Compaction | No | Yes | Yes | N/A | **Yes (incremental)** |
| Write barriers | Yes (all stores) | Yes (all ref stores) | Yes (load barriers) | N/A | **Only mut ref stores** |
| Non-GC allocation | No | No | No | All manual | **@stack, @arena, @inline** |
| GC tuning knobs | Few | Many (~50 flags) | Moderate (~10 flags) | N/A | **Few (target-based)** |
| Profiling | pprof | JFR, VisualVM | JFR | valgrind, heaptrack | **Built-in (aria profile)** |

### The key differences

**vs Go:** Aria adds generations (Go doesn't have them), per-task nurseries (Go shares per-P), compaction (Go doesn't compact), and fewer write barriers (immutable default). Aria also diverts traffic away from the GC with `@stack`/`@arena`.

**vs Java G1:** Aria has faster allocation (per-task bump pointer vs TLAB with synchronization), lower young-gen pauses (per-task vs stop-the-world), and fewer write barriers (immutable default). Java G1 is more mature and battle-tested.

**vs Java ZGC:** Similar pause characteristics, but Aria's per-task nurseries avoid the load barrier overhead that ZGC pays on every reference load. Aria also diverts more traffic away from the GC.

**vs Rust:** Aria provides a GC for the 95% case where manual memory management is unnecessary. For the 5% case, `@stack`/`@arena`/`Pool[T]` provide manual control comparable to Rust — without lifetimes in every function signature.

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/memory-management.md](memory-management.md), [spec/concurrency-design.md](concurrency-design.md), and [spec/compiler-architecture.md](compiler-architecture.md).*
