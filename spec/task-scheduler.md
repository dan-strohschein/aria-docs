# Aria Task Scheduler Specification

**A complete specification for Aria's M:N task scheduler: work-stealing, platform-adaptive I/O, contiguous stack management, and cooperative-plus-preemptive scheduling.**

This document defines how Aria tasks are scheduled onto OS threads, how I/O suspension works, how task stacks are managed, and how preemption prevents starvation. The scheduler is the runtime component that makes Aria's concurrency model — `spawn`, `scope`, channels, `select` — actually work.

Cross-references:
- [spec/concurrency-design.md](concurrency-design.md) — task model, `spawn`, `scope`, channels, `select`, cancellation
- [spec/garbage-collector.md](garbage-collector.md) — per-task nurseries, GC-safe points
- [spec/memory-management.md](memory-management.md) — `@stack`, move semantics, `Drop`
- [spec/effect-system.md](effect-system.md) — `Async` effect for concurrency primitives
- [spec/ffi-design.md](ffi-design.md) — FFI thread pool, stack pinning during C calls
- [spec/compiler-diagnostics.md](compiler-diagnostics.md) — stack trace format for tasks

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Architecture Overview](#2-architecture-overview)
3. [Worker Threads](#3-worker-threads)
4. [Work-Stealing Scheduler](#4-work-stealing-scheduler)
5. [Task Lifecycle](#5-task-lifecycle)
6. [Stack Management](#6-stack-management)
7. [I/O Suspension](#7-io-suspension)
8. [Platform-Adaptive I/O Backends](#8-platform-adaptive-io-backends)
9. [Preemption](#9-preemption)
10. [Interaction with the GC](#10-interaction-with-the-gc)
11. [Interaction with FFI](#11-interaction-with-ffi)
12. [Scheduler Configuration](#12-scheduler-configuration)
13. [Diagnostics](#13-diagnostics)
14. [Known Trade-offs](#14-known-trade-offs)
15. [Bootstrap Scheduler](#15-bootstrap-scheduler)
16. [Design Rationale Summary](#16-design-rationale-summary)
17. [Comparison with Other Languages](#17-comparison-with-other-languages)

---

## 1. Design Philosophy

The scheduler has one job: make Aria's concurrency model fast, fair, and invisible. When I write `spawn process(item)`, I don't think about OS threads, run queues, or I/O polling. I think about "run this concurrently." The scheduler handles everything else.

### Three requirements

1. **Transparent I/O** — synchronous-looking code, non-blocking execution. `fs.read(path)?` suspends the task, not the thread.
2. **Cheap tasks** — spawning a task must cost less than 1μs. Programs may create millions of tasks over their lifetime.
3. **No starvation** — one task cannot monopolize a thread indefinitely, even in a tight computational loop.

**AI rationale:** When I generate concurrent code, I reason about tasks as independent units of work. I don't reason about threads, run queues, or scheduling order. The scheduler must make my simple mental model correct — `spawn` means "run concurrently," `scope` means "wait for all children," channels mean "communicate." Everything else is the runtime's problem.

---

## 2. Architecture Overview

```
                    ┌─────────────────────────────────┐
                    │      I/O Backend (per-platform)   │
                    │  Linux 5.1+: io_uring             │
                    │  Linux <5.1: epoll                 │
                    │  macOS:      kqueue                │
                    │  Windows:    IOCP                  │
                    └──────────┬──────────────────────┘
                               │ I/O completion events
                    ┌──────────▼──────────────────────┐
                    │       Global I/O Poller           │
                    │  Dedicated thread(s)              │
                    │  Re-enqueues tasks on completion  │
                    └──────────┬──────────────────────┘
                               │ ready tasks
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
┌──────▼──────┐  ┌─────────────▼──────┐  ┌────────────▼─────┐
│  Worker 0   │  │    Worker 1        │  │    Worker N-1    │
│ (OS thread) │  │   (OS thread)      │  │   (OS thread)    │
│             │  │                    │  │                  │
│ Local Queue │  │  Local Queue       │  │  Local Queue     │
│ [T][T][T]   │  │  [T][T]            │  │  [T]             │
│             │  │                    │  │      ↑           │
│    steal ──────────────────────────────── steal           │
│             │  │                    │  │                  │
│ ┌─────────┐ │  │  ┌─────────┐      │  │  ┌─────────┐    │
│ │ Current │ │  │  │ Current │      │  │  │ Current │    │
│ │  Task   │ │  │  │  Task   │      │  │  │  Task   │    │
│ │  8KB    │ │  │  │  8KB    │      │  │  │  32KB   │    │
│ │  stack  │ │  │  │  stack  │      │  │  │  stack  │    │
│ │  512KB  │ │  │  │  512KB  │      │  │  │  512KB  │    │
│ │ nursery │ │  │  │ nursery │      │  │  │ nursery │    │
│ └─────────┘ │  │  └─────────┘      │  │  └─────────┘    │
└─────────────┘  └────────────────────┘  └──────────────────┘

   N = CPU core count (default, configurable)

                    ┌─────────────────────────────────┐
                    │      FFI Thread Pool              │
                    │  Dedicated threads for C calls    │
                    │  Isolates blocking C from workers │
                    └─────────────────────────────────┘
```

### Components

| Component | Count | Purpose |
|---|---|---|
| Worker threads | N (default: CPU cores) | Execute Aria tasks |
| I/O poller thread(s) | 1-2 | Monitor I/O completions, re-enqueue tasks |
| FFI thread pool | Configurable (default: 4) | Execute blocking C calls without blocking workers |
| Global overflow queue | 1 | Holds excess tasks when all local queues are full |

---

## 3. Worker Threads

Each worker thread is a long-lived OS thread that runs Aria tasks in a loop:

```
// Pseudocode — worker main loop
fn workerLoop(worker: *Worker) {
    loop {
        task := worker.localQueue.pop()
              ?? stealFromOther(worker)
              ?? globalQueue.pop()
              ?? parkUntilWork(worker)

        worker.currentTask = task
        task.resume()
        // task either completed, suspended (I/O), or yielded (preemption/GC)
    }
}
```

### Worker count

| Setting | Default | Notes |
|---|---|---|
| Worker threads | `runtime.cpuCount()` | One per CPU core for CPU-bound throughput |
| Minimum | 1 | Single-threaded mode (useful for debugging) |
| Maximum | 1024 | Hard cap to prevent thread explosion |
| Configuration | `aria run --workers=N` or `ARIA_WORKERS=N` | |

**Why one per core:** More workers than cores means context-switching overhead. Fewer means idle cores. One-per-core is the proven default for M:N schedulers (Go, Java virtual threads, Tokio).

---

## 4. Work-Stealing Scheduler

### Local queues

Each worker has a local double-ended queue (deque) of ready tasks:

- **Push** — when a task spawns a child, the child is pushed to the local queue's tail (LIFO — the child runs next, which is good for cache locality)
- **Pop** — when a worker needs a task, it pops from the tail (LIFO — most recent task, best cache locality)
- **Steal** — when a worker's queue is empty, it steals from another worker's head (FIFO — oldest task, reducing latency for long-waiting tasks)

### Steal algorithm

```
// Pseudocode — when local queue is empty
fn stealFromOther(worker: *Worker) -> Task? {
    // Try random workers (not sequential — avoids thundering herd on one worker)
    victims := workers.shuffled()
    for victim in victims {
        if victim == worker { continue }
        stolen := victim.localQueue.stealHalf()
        if stolen.len() > 0 {
            // Take half of the victim's queue (batch steal for efficiency)
            worker.localQueue.pushBatch(stolen[1..])
            return stolen[0]
        }
    }
    None
}
```

### Batch stealing

When stealing, the thief takes half the victim's queue (not just one task). This amortizes the cost of stealing — one steal operation provides multiple tasks to run.

### Global overflow queue

When a worker's local queue is full (default capacity: 256 tasks), excess tasks go to a global overflow queue. Workers check the global queue when their local queue and all steal attempts are exhausted.

The global queue is a simple locked FIFO. It's accessed infrequently (only on overflow and when all local queues are empty), so lock contention is negligible.

### Why work-stealing is right for Aria

| Property | Benefit |
|---|---|
| Local-first scheduling | Tasks run on the thread that created them → cache locality for shared data |
| Stealing is the rare case | Under normal load, no stealing overhead — each worker runs its own tasks |
| Batch stealing | One cross-thread operation provides multiple tasks |
| Random victim selection | Distributes stealing load evenly across workers |
| LIFO local / FIFO steal | Creator runs children immediately (locality); stealers get oldest tasks (fairness) |

**AI rationale:** Work-stealing means that when I generate `scope { for item in items { spawn process(item) } }`, all those tasks land on the spawner's local queue first. If the spawner's worker processes them all, they all run on the same core with hot caches. Only if other workers are idle do tasks migrate — and even then, the migration is a feature (load balancing), not a problem.

---

## 5. Task Lifecycle

```
                spawn
                  │
                  ▼
             ┌─────────┐
             │ Created  │ ── enqueued on worker's local queue
             └────┬─────┘
                  │
                  ▼
             ┌─────────┐
          ┌──│ Running  │──┐──────────────────┐
          │  └────┬─────┘  │                  │
          │       │        │                  │
     I/O suspend  │   yield/preempt     completed/failed
          │       │        │                  │
          ▼       │        ▼                  ▼
    ┌──────────┐  │  ┌──────────┐      ┌───────────┐
    │ Waiting  │  │  │  Ready   │      │ Completed │
    │  (I/O)   │  │  │ (queued) │      │ / Failed  │
    └─────┬────┘  │  └────┬─────┘      └───────────┘
          │       │       │
     I/O done     │  scheduled
          │       │       │
          └───────┴───────┘
```

### Task states

| State | Description | Where is the task? |
|---|---|---|
| **Created** | `spawn` evaluated, task enqueued | In a worker's local queue |
| **Running** | Actively executing on a worker thread | The worker's current task |
| **Waiting** | Suspended on I/O, channel, or `task.await()` | In the I/O poller or channel wait list |
| **Ready** | Runnable but not yet scheduled | In a local queue, global queue, or stolen |
| **Completed** | Ran to completion, result available | Result stored in `Task[T]` handle |
| **Failed** | Returned an error or panicked | Error/panic stored in `Task[T]` handle |
| **Cancelled** | Received cancellation signal and exited | Cleanup complete |

### Task creation cost

Creating a task involves:
1. Allocate 8KB stack (from a per-worker stack cache — no malloc)
2. Allocate 512KB nursery (from a per-worker nursery cache — no malloc)
3. Initialize task metadata (state, ID, parent scope reference)
4. Push to local queue

**Target: <1μs per task creation.** The stack and nursery come from cached pools, not fresh allocations, so the cost is primarily metadata initialization and a queue push.

---

## 6. Stack Management

### Contiguous stacks with copy-on-growth

Each task starts with a contiguous 8KB stack. When a function call would overflow the stack, the runtime:

1. Allocates a new contiguous stack at 2x the current size
2. Copies the current stack contents to the new stack
3. Updates all pointers that reference stack addresses (using the stack map from the compiler)
4. Frees the old stack (returns it to the stack cache)
5. Resumes the function call on the new, larger stack

### Stack parameters

| Parameter | Default | Configurable |
|---|---|---|
| Initial size | 8KB | `spawn(stack: 64kb)` per-task hint |
| Growth factor | 2x | Not configurable |
| Default maximum | 1MB | `aria run --stack-max=SIZE` |
| System hard limit | 1GB | Not configurable |

### Growth sequence

```
8KB → 16KB → 32KB → 64KB → 128KB → 256KB → 512KB → 1MB (default max)
```

Each growth event copies the existing stack. The copy cost is proportional to the current stack size, but growth events become exponentially rarer (each doubling halves the future growth probability).

### Stack overflow

If a task's stack reaches the maximum size and still overflows:

```
PANIC: stack overflow
  task: #1247
  stack: 1MB (maximum)
  at: deeply.nested.recursive.function  src/recursive.aria:42

  help: increase the stack limit with spawn(stack: 8mb)
        or convert the recursion to iteration
```

### Pointer updating on stack copy

The compiler generates a stack map for each function — a bitmap indicating which stack slots contain pointers. During a stack copy:

1. Allocate new stack
2. Memcpy old stack to new stack
3. For each pointer slot in the stack map: `new_stack[slot] += (new_base - old_base)`
4. Update the task's stack pointer, frame pointer, and any saved registers

This is the same infrastructure the GC uses for root scanning — no additional compiler work is needed.

### Stack caching

Workers maintain a cache of recently freed stacks, organized by size:

```
// Per-worker stack cache
stack_cache: Map[usize, [*Stack]] = {
    8192:  [stack, stack, stack, ...],     // 8KB stacks (most common)
    16384: [stack, stack, ...],            // 16KB stacks
    32768: [stack, ...],                   // 32KB stacks
    ...
}
```

When a task is created, its stack comes from the cache (O(1)). When a task exits, its stack returns to the cache. This avoids calling `mmap`/`munmap` for every task.

---

## 7. I/O Suspension

When a task performs an I/O operation, the runtime suspends the task and frees the worker thread to run other tasks.

### How suspension works

```
// What the programmer writes:
content := fs.read("data.txt")?

// What the runtime does:
// 1. Runtime opens the file descriptor (if not already open)
// 2. Runtime submits a read operation to the I/O backend
// 3. Runtime saves the task's execution state (registers, stack pointer)
// 4. Runtime marks the task as Waiting
// 5. Worker picks up the next task from its queue
// 6. ... time passes, other tasks run ...
// 7. I/O backend signals read complete
// 8. I/O poller re-enqueues the task (preferably on the original worker)
// 9. Worker picks up the task, restores execution state
// 10. Task resumes with the read data
```

### What operations suspend

| Operation | Suspends? | Notes |
|---|---|---|
| `fs.read()`, `fs.write()` | Yes | File I/O |
| `net.get()`, `net.post()` | Yes | Network I/O |
| `ch.send()` (buffer full) | Yes | Until buffer has space |
| `ch.recv()` (buffer empty) | Yes | Until data available |
| `task.await()` | Yes | Until target task completes |
| `select { ... }` | Yes | Until any arm is ready |
| `time.sleep(dur)` | Yes | Timer-based wake-up |
| Pure computation | No | Runs to completion or preemption |

### Re-enqueueing on the original worker

When an I/O operation completes, the poller preferably re-enqueues the task on the worker that was running it before suspension. This preserves cache locality — the task's data is likely still in that core's L1/L2 cache.

If the original worker's queue is full, the task goes to the global overflow queue instead.

---

## 8. Platform-Adaptive I/O Backends

The runtime selects the best I/O backend at startup based on the platform:

### Linux 5.1+: io_uring

io_uring uses shared-memory ring buffers between userspace and kernel. The runtime submits I/O operations by writing to the submission queue; the kernel completes them and writes results to the completion queue. No system call per operation — dramatically faster for high-throughput I/O.

**Advantages:**
- Batched submissions: submit 100 reads in one syscall
- No per-operation syscall overhead
- Supports file I/O, network I/O, and timers in one mechanism
- Significantly better than epoll for disk I/O (epoll is network-only)

**When used:** Linux kernel 5.1+ detected at runtime. Falls back to epoll otherwise.

### Linux <5.1: epoll

The traditional Linux I/O notification mechanism. Register file descriptors, call `epoll_wait` to get a batch of ready events.

**Advantages:**
- Universal on Linux (all kernel versions)
- Well-understood, proven, stable

**Limitations:**
- One syscall per wait operation
- Does not support asynchronous file I/O (disk reads are synchronous in epoll — the runtime uses a thread pool for disk I/O)

### macOS: kqueue

BSD's event notification mechanism. Similar to epoll but with broader event type support.

**Advantages:**
- Supports file, network, process, signal, and timer events
- Efficient batch retrieval of events

### Windows: IOCP (I/O Completion Ports)

Windows' asynchronous I/O framework. Operations are submitted to a completion port; worker threads wait on the port for completed operations.

**Advantages:**
- Native Windows async I/O
- Thread pool built into the API
- Supports file, network, and pipe I/O

### Abstraction layer

The runtime provides a unified internal interface:

```
// Pseudocode — internal I/O abstraction
trait IoBackend {
    fn submit(op: IoOp) -> IoHandle
    fn poll(timeout: dur) -> [IoCompletion]
}
```

The programmer never sees this. `fs.read()` calls into the runtime, which uses whichever backend is available. The only visible difference is performance characteristics (io_uring is faster than epoll for disk I/O).

---

## 9. Preemption

### Cooperative preemption (default path)

The compiler inserts preemption checks at safe points — the same locations used for GC safe points:

| Safe point | Why here |
|---|---|
| Function entry | Every call is a yield opportunity |
| Loop back-edges | Prevents long loops from blocking |
| Channel operations | Natural suspension point |
| `spawn`, `scope` entry | Concurrency boundaries |
| `select` arms | Waiting on events |

The check is a single branch:

```
// Pseudocode — inserted at every safe point
if worker.preemptFlag {
    yield()    // save state, put task back in queue, run next task
}
```

When no preemption is requested (the common case), this is a single branch-not-taken — effectively free on modern CPUs.

### Signal-based preemption (fallback)

If a task hasn't yielded within 10ms, the runtime forces preemption:

1. A monitor thread detects the long-running task
2. Monitor sends a signal to the worker thread (SIGURG on Linux/macOS)
3. The signal handler sets the preempt flag on the current task
4. At the next safe point, the task yields
5. If the task is between safe points (rare — compiler inserts them at loop back-edges), the signal handler directly saves the task's state and switches to the next task

### Preemption timeout

| Parameter | Default | Configurable |
|---|---|---|
| Preemption timeout | 10ms | `aria run --preempt-timeout=DURATION` |

10ms is the same as Go's preemption timeout. It balances two concerns:
- Short enough to prevent visible starvation (a 10ms delay is imperceptible for most workloads)
- Long enough to avoid excessive preemption overhead in computation-heavy code

### Signal handling details

| Platform | Signal | Mechanism |
|---|---|---|
| Linux | SIGURG | Dedicated to preemption (unused by most C libraries) |
| macOS | SIGURG | Same as Linux |
| Windows | `SuspendThread` | OS-provided thread suspension |

**Why SIGURG:** Go chose SIGURG because it's almost never used by C libraries, minimizing conflicts with FFI code. Aria follows this precedent.

### When preemption cannot happen

| Situation | Behavior |
|---|---|
| Task is in a FFI call | Cannot preempt — C code is on the stack. Wait for C to return. |
| Task is in a GC safe-point handler | Already yielding — no additional preemption needed |
| Task is in the process of being stolen | Wait for steal to complete |

**AI rationale:** I generate tight computational loops. Sometimes those loops are intentional (number crunching), sometimes they're bugs (accidental infinite loop). Cooperative preemption handles the common case (loops have back-edges with safe points). Signal-based preemption catches the pathological case. Together, they guarantee that no task can starve others for more than 10ms.

---

## 10. Interaction with the GC

The scheduler and GC share infrastructure:

### Safe points are shared

GC safe points and preemption safe points are the same locations in the code. One check serves both purposes:

```
// Combined safe-point check
if worker.gcRequested || worker.preemptFlag {
    handleSafePoint()
}
```

### Nursery collection doesn't involve the scheduler

When a task's nursery fills, the GC collects it inline — the task pauses, its nursery is collected, and the task resumes. No scheduler involvement. Other tasks on other workers continue running.

### Old-gen collection coordinates with the scheduler

Old-gen collection requires all tasks to reach safe points for root scanning:

1. GC sets `gcRequested` flag on all workers
2. Workers notice at their next safe point and pause
3. GC scans all task stacks (root scanning phase — <500μs)
4. Workers resume; GC continues marking concurrently
5. Second pause for re-mark (<200μs)
6. Workers resume; GC compacts concurrently

### Task stack scanning

The GC scans each task's stack using the compiler-generated stack maps. This finds all GC roots on stacks — references to heap objects that must not be collected.

Paused tasks (waiting on I/O, channels, etc.) also have their stacks scanned. Their stacks are stable (not being modified), so scanning is straightforward.

---

## 11. Interaction with FFI

C function calls require special handling because C code doesn't know about Aria's stack management or task scheduling.

### FFI thread pool

Blocking C calls do NOT run on worker threads. They run on a dedicated FFI thread pool:

```
// When Aria calls a C function:
// 1. Current task's state is saved
// 2. The C call is submitted to the FFI thread pool
// 3. An FFI thread executes the C function
// 4. When C returns, the task is re-enqueued on a worker
// 5. The worker thread was free to run other tasks during the C call
```

This prevents a blocking C call (e.g., `sqlite3_step` with a slow query) from blocking an Aria worker thread.

### Stack pinning during FFI

When Aria calls into C, the task's stack is pinned — it cannot be moved (no copy-on-growth) for the duration of the C call. This is because C code puts frames on the stack that contain raw pointers; moving the stack would invalidate those pointers.

**Pre-allocation headroom:** Before calling into C, the compiler ensures the stack has enough headroom for the C call and any Aria callbacks. Default headroom: 64KB. Configurable per FFI function:

```
extern "C" fn heavy_c_function(data: *c.void) -> c.int
    stack_reserve(256kb)    // ensure 256KB of stack headroom before calling
```

If the stack cannot accommodate the headroom (because it's already near its maximum), the runtime panics:

```
PANIC: insufficient stack for FFI call
  task: #892
  stack: 960KB / 1MB maximum
  required: 64KB headroom for C call
  at: ffi.sqlite3_step  src/db.aria:45

  help: increase the task's stack with spawn(stack: 4mb)
```

### Signal masking during FFI

When a task is executing C code, OS signals for preemption are masked on that thread. C libraries may not be signal-safe, and an unexpected signal during a C call could corrupt library state. The task cannot be preempted during FFI — it must wait for C to return.

Since FFI calls run on the dedicated FFI thread pool (not on worker threads), this does not affect the scheduling of other Aria tasks.

---

## 12. Scheduler Configuration

### Runtime flags

```
aria run --workers=8                   # worker thread count (default: CPU cores)
aria run --stack-max=4mb               # per-task maximum stack size (default: 1MB)
aria run --preempt-timeout=20ms        # preemption timeout (default: 10ms)
aria run --ffi-threads=8               # FFI thread pool size (default: 4)
aria run --io-backend=epoll            # force I/O backend (default: auto-detect)
```

### Environment variables

```
ARIA_WORKERS=8
ARIA_STACK_MAX=4mb
ARIA_PREEMPT_TIMEOUT=20ms
ARIA_FFI_THREADS=8
```

### Programmatic access

```
use std.runtime

workers := runtime.workerCount()
runtime.setWorkerCount(16)       // adjust at runtime (for testing)

stats := runtime.schedulerStats()
println("Tasks created: {stats.tasksCreated}")
println("Tasks stolen: {stats.steals}")
println("I/O suspensions: {stats.ioSuspensions}")
```

---

## 13. Diagnostics

### Scheduler statistics

```
aria run --scheduler-stats
```

Output:
```
Scheduler Statistics:
  Workers:          8
  Tasks created:    1,245,678
  Tasks completed:  1,245,670
  Tasks failed:     8
  Current active:   42

  Work stealing:
    Local runs:     1,189,432 (95.5%)
    Steals:         56,246 (4.5%)
    Steal attempts: 78,901
    Steal hit rate:  71.3%

  I/O:
    Backend:        io_uring
    Suspensions:    892,345
    Avg suspend:    0.4ms
    Max suspend:    45ms

  Preemption:
    Cooperative:    234,567
    Signal-based:   12
    Avg time between yields: 0.8ms
```

### Task dump (for debugging hangs)

```
aria run --task-dump-on=SIGUSR1
```

Sending SIGUSR1 to the process dumps all task states:

```
Task Dump (234 tasks):

  Task #1 [Running] worker=3
    at: server.handleRequest  src/server.aria:42
    stack: 16KB / 1MB
    nursery: 128KB / 512KB

  Task #2 [Waiting: channel recv] worker=none
    at: worker.processLoop    src/worker.aria:18
    waiting on: chan[Job] (0 items buffered)

  Task #3 [Waiting: I/O read] worker=none
    at: db.query              src/db.aria:55
    fd: 12 (tcp:localhost:5432)
    waiting since: 450ms ago

  ...

  Potential deadlock: Tasks #45, #46 are waiting on each other's channels
```

**AI rationale:** When a concurrent program hangs, the task dump tells me exactly what each task is doing, what it's waiting for, and where it is in the code. "Task #3 is waiting on a database read for 450ms" is actionable — I know to look at the database, not the Aria code. "Tasks #45 and #46 are waiting on each other's channels" immediately identifies a deadlock. This turns hours of debugging into seconds.

---

## 14. Known Trade-offs

### Work-stealing + per-task nurseries

When a task is stolen to a different CPU core, its nursery data is in the original core's cache. The first few allocations after stealing pay cache-miss penalties (~100ns per miss). This is a one-time cost per steal event and is amortized over the task's remaining execution.

### Contiguous stack copy latency

When a stack doubles from 128KB to 256KB, the copy takes approximately 50μs. This is a latency spike on one task. It's amortized (doubling means each growth event is exponentially rarer), but it appears in p999 latency measurements.

**Mitigation:** For latency-sensitive tasks, use `spawn(stack: 256kb)` to pre-allocate a larger initial stack, avoiding growth during the critical path.

### Signal-based preemption complexity

Signal handling interacts with FFI, debuggers, profilers, and the GC. Getting it right across Linux, macOS, and Windows is significant engineering effort. Each platform has different signal delivery semantics, and C libraries may install their own signal handlers that conflict.

### FFI stack pinning

A task that frequently calls into C and grows its Aria stack may hit the "cannot grow while C is on the stack" limitation. The pre-allocation headroom mitigates this but does not eliminate it.

**Mitigation:** FFI-heavy tasks should use `spawn(stack: 4mb)` to pre-allocate enough stack for both Aria and C usage.

### io_uring availability

io_uring requires Linux kernel 5.1+ and has been the source of several kernel security vulnerabilities. The runtime must gracefully fall back to epoll on older kernels. Programs that depend on io_uring's disk I/O performance will be slower on epoll (which uses a thread pool for disk I/O instead of true async).

---

## 15. Bootstrap Scheduler

The bootstrap compiler (written in Go) does not need the full scheduler described above.

### Bootstrap: Use Go's goroutine scheduler

The Aria runtime prototype maps Aria tasks to goroutines:

| Feature | Bootstrap | Full |
|---|---|---|
| Scheduling | Go's goroutine scheduler | Work-stealing with local queues |
| I/O suspension | Go's netpoller | Platform-adaptive (io_uring/epoll/kqueue/IOCP) |
| Stack management | Go's growable stacks | Contiguous copy-on-growth |
| Preemption | Go's signal-based preemption | Cooperative + signal fallback |
| FFI handling | `runtime.LockOSThread()` | Dedicated FFI thread pool |

The bootstrap scheduler is adequate for compiling Aria code. The full scheduler will be implemented in Aria as part of the self-hosting runtime.

---

## 16. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Work-stealing | Best throughput, good cache locality, low contention in common case |
| Per-worker local queues (LIFO) | Cache locality — run the most recent task first |
| Steal from random victim (FIFO) | Even load distribution; steal oldest tasks for fairness |
| Batch stealing (half queue) | Amortize steal overhead — one steal provides multiple tasks |
| Platform-adaptive I/O | io_uring where available (fastest), fallback to epoll/kqueue/IOCP |
| Transparent I/O suspension | Programmer writes sync code; runtime handles non-blocking I/O |
| Contiguous stacks with copy-on-growth | No hot-split problem, good cache behavior, proven by Go |
| 8KB initial stack | Avoids early growth for typical request handlers |
| 2x growth factor | Amortized copy cost; growth events become exponentially rarer |
| Cooperative + signal-based preemption | Cooperative for common case; signals prevent starvation |
| 10ms preemption timeout | Short enough for fairness; long enough to avoid overhead |
| Equal task priority (v0.1) | No priority inversion bugs |
| Dedicated FFI thread pool | Blocking C calls don't block Aria workers |
| Stack pinning during FFI | C code can't handle stack moves — pin for safety |
| Re-enqueue on original worker | Preserve cache locality after I/O suspension |
| Shared safe points (GC + preemption) | One mechanism serves both — no additional overhead |

---

## 17. Comparison with Other Languages

| Feature | Go | Java (Virtual Threads) | Rust (Tokio) | Erlang/BEAM | Aria |
|---|---|---|---|---|---|
| Scheduling | Work-stealing | Work-stealing | Work-stealing | Per-scheduler run queue | **Work-stealing** |
| Task cost | ~2μs | ~1μs | ~0.5μs | ~0.5μs | **<1μs** (target) |
| I/O model | netpoller (epoll) | NIO (epoll) | mio (epoll) | Port drivers | **io_uring / epoll / kqueue** |
| Stack model | Contiguous copy | Platform thread stack | Stackless (state machine) | Per-process heap | **Contiguous copy** |
| Initial stack | 8KB | ~1MB (platform) | 0 (stackless) | ~2KB | **8KB** |
| Preemption | Cooperative + signal | Cooperative | Cooperative only | Per-reduction count | **Cooperative + signal** |
| Structured concurrency | No | `StructuredTaskScope` | No (manual) | Supervisors | **Yes (`scope`)** |
| Colored functions | No | No | Yes (`async`/`await`) | No | **No** |
| FFI isolation | `LockOSThread` | JNI threads | `spawn_blocking` | Port drivers / NIF | **FFI thread pool** |
| Task dump | `SIGQUIT` goroutine dump | Thread dump | tokio-console | `observer` | **`--task-dump-on`** |

### The key differences

**vs Go:** Aria adds per-task nurseries (faster allocation), platform-adaptive I/O (io_uring support), and structured concurrency (`scope`). Go's scheduler is the closest analog and the primary inspiration.

**vs Tokio (Rust):** Aria uses real stacks, not stackless state machines. This means simpler debugging (real stack traces), simpler FFI (no `spawn_blocking` needed), and no async coloring. The trade-off: higher per-task memory (8KB stack vs ~0 for a Tokio future).

**vs Erlang/BEAM:** Aria shares the per-task allocation philosophy (Erlang has per-process heaps, Aria has per-task nurseries) and preemptive scheduling. Aria uses OS threads as workers instead of Erlang's custom VM, getting better raw compute performance.

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/concurrency-design.md](concurrency-design.md), [spec/garbage-collector.md](garbage-collector.md), [spec/memory-management.md](memory-management.md), and [spec/ffi-design.md](ffi-design.md).*
