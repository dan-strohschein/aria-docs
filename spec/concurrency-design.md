# Aria Concurrency Design

## Overview

This document formalizes Aria's concurrency model: a **structured concurrency system** built on lightweight tasks, typed channels, and scope-bound lifetimes. The design is a deliberate evolution past Go's goroutine model — keeping what Go got right (lightweight M:N tasks, channels as first-class values, select) and fixing what Go got wrong (goroutine leaks, silent error swallowing, unstructured task lifetimes).

Goroutine leaks and race conditions are the **#2 source of AI-generated bugs** after error handling mistakes. This spec exists because the language must make both impossible by construction.

This document is part of the core language spec. Related documents:

- [high-level-design.md](../high-level-design.md) — language overview, concurrency primitives sketched
- [spec/stdlib-design.md](stdlib-design.md) — `sync` module reference
- [spec/paradigm-design.md](paradigm-design.md) — expression-oriented procedural paradigm
- [spec/language-spec-addendum.md](language-spec-addendum.md) — pipeline operator and expressions

---

## Design Philosophy

### The Three Guarantees

1. **No task leaks — impossible by construction.** Every task spawned inside a `scope` block is joined before the scope exits. There is no equivalent of a fire-and-forget goroutine that outlives its creator (unless explicitly detached to a named runtime — see [Detached Tasks](#detached-tasks)).

2. **No data races at compile time.** The type system prevents sharing mutable state across task boundaries without explicit synchronization. A value that crosses a task boundary must be either immutable, atomically wrapped, or transferred via channel.

3. **Errors never silently disappear.** A task that fails with an error propagates that error to the parent scope. There is no equivalent of a goroutine crash that silently terminates without the parent knowing.

### Why This Matters for AI Code Generation

When I generate concurrent Go code, I produce bugs in a predictable pattern:

```go
// Go — bug I generate regularly: missing WaitGroup or wrong Add count
var wg sync.WaitGroup
for _, url := range urls {
    go func(u string) {    // goroutine leak if I forget wg.Add(1) before this
        defer wg.Done()    // or forget this
        fetch(u)           // errors silently dropped
    }(url)
}
wg.Wait()
```

The correct Aria equivalent is shorter and structurally impossible to get wrong:

```go
scope {
    for url in urls {
        spawn fetchAndProcess(url)    // errors propagate, lifetime is bounded
    }
}
```

There is no way to forget the join. There is no way to drop an error. The token count drops from ~8 lines to 3.

---

## 1. Task Model

### 1.1 Lightweight Tasks

Tasks are Aria's concurrency primitive — lightweight green threads scheduled onto OS threads by the Aria runtime. They are cheap to create (typical stack size: 8 KB, growable) and cost one M:N scheduler slot, not one OS thread.

```go
// Fire-and-forget spawn (within a scope — see section 2)
spawn fetchData(url)

// Spawn and bind the task handle
task := spawn fetchData(url)

// Spawn a block
task := spawn {
    data := fetchData(url)?
    process(data)?
}
```

A `spawn` expression returns a `Task[T]` where `T` is the return type of the spawned expression. If the spawned expression has no return value, `T` is `()` (unit).

### 1.2 Task Handles

```go
type Task[T] {
    // Methods available on a task handle
}

// Await a task's result (blocks the current task until done)
result := task.await()               // returns T

// Await with error handling (task can fail)
result := task.await() catch |err| { ... }

// Check if task is complete (non-blocking)
if task.done() {
    result := task.result()          // returns T? — None if not done yet
}

// Cancel a task (cooperative — see section 5)
task.cancel()
```

The type `Task[T]` is generic over the return type:

| Spawned expression | Task type |
|---|---|
| `spawn compute()` where `compute` returns `f64` | `Task[f64]` |
| `spawn processRequest(req)` where result is `Response ! HttpError` | `Task[Response ! HttpError]` |
| `spawn { doWork() }` where `doWork` returns `()` | `Task[()]` |

### 1.3 Task Lifecycle

```
Created → Running → Completed
                 ↘ Failed
                 ↘ Cancelled
```

- **Created**: the `spawn` expression has been evaluated; the task is queued for execution.
- **Running**: the runtime scheduler has assigned the task to a thread.
- **Completed**: the task ran to completion and produced a result value.
- **Failed**: the task encountered an unhandled error. The error is captured and propagated to the parent scope on join.
- **Cancelled**: the task received a cancellation signal and exited cooperatively (see section 5).

### 1.4 M:N Threading Model

Aria uses M:N scheduling: M tasks run on N OS threads (where N is typically the number of CPU cores).

- Tasks are multiplexed onto threads by the runtime scheduler.
- Blocking a task (e.g., waiting on I/O, `chan.recv()`, `task.await()`) yields the thread to other tasks — it does NOT block the OS thread.
- The runtime uses async I/O under the hood; the programmer sees synchronous code.
- No colors: `async`/`await` keywords are **not required** in Aria. Any function can `spawn`, any function can call blocking operations. The runtime handles it.

```go
// This is NOT async/await — it's synchronous code that suspends transparently
fn fetchAll(urls: [str]) -> [str] ! HttpError {
    results := scope {
        tasks := urls.map(fn(url) => spawn net.get(url)?)
        tasks.map(fn(t) => t.await())
    }
    results
}
```

**Why no async/await coloring**: The async function color problem is one of the biggest sources of AI bugs. In Rust/TypeScript/Python, forgetting `await` is a silent bug. In Aria, there is no color — tasks are implementation details the scheduler manages.

### 1.5 Stack Management

- Default stack size: 4 KB (half of Go's 8 KB default — tasks are expected to be more plentiful and shallower)
- Stacks are growable: the runtime doubles the stack when it overflows, up to a configurable limit (default 1 GB)
- Stack size hint: `spawn(stack: 64kb) heavyRecursion(depth)` — provides a hint, not a hard limit

```go
// Default stack
task := spawn processRequest(req)

// Stack size hint for known-deep recursion or large locals
task := spawn(stack: 256kb) parseGiantAst(tokens)
```

### 1.6 Detached Tasks

By default, every `spawn` must occur within a `scope` block that joins it. For the rare case of a truly background task that should outlive the scope (e.g., a metrics flusher, a background compactor), use `spawn.detach`:

```go
// Detach from structured lifetime — you accept responsibility
// for managing this task's lifetime
bgTask := spawn.detach backgroundMetricsFlusher()

// Detached tasks must be explicitly stopped
defer bgTask.cancel()
```

Detached tasks must be explicitly cancelled or awaited somewhere. The compiler emits a warning if a detached task handle is dropped without being joined or cancelled.

---

## 2. Structured Concurrency (`scope`)

### 2.1 What `scope` Does

A `scope` block is the key differentiator from Go. It provides a **structured lifetime guarantee**: every task spawned within a scope must complete (or be cancelled) before the scope expression returns.

```go
scope {
    a := spawn fetchUsers()
    b := spawn fetchOrders()
}
// This line is reached ONLY after BOTH tasks have completed
// a and b are no longer in scope here — their results were consumed
```

### 2.2 Accessing Task Results

Tasks spawned within a `scope` can be awaited for their results inside the scope body, or the scope itself can be an expression that returns the results:

```go
// Method 1: await inside scope
scope {
    a := spawn fetchUsers()
    b := spawn fetchOrders()
    users := a.await()
    orders := b.await()
    matchUsersToOrders(users, orders)
}

// Method 2: scope as expression returning a tuple
(users, orders) := scope {
    a := spawn fetchUsers()
    b := spawn fetchOrders()
    (a.await(), b.await())     // last expression is scope's value
}

// Method 3: named results
result := scope {
    a := spawn fetchUsers()
    b := spawn fetchOrders()
    Dashboard{
        users: a.await()
        orders: b.await()
    }
}
```

### 2.3 Error Propagation from Child Tasks

If any child task fails (returns an error), the error propagates to the parent scope on the first `await()` that encounters it, or when the scope exits:

```go
fn loadDashboard() -> Dashboard ! DbError {
    result := scope {
        a := spawn fetchUsers()           // Task[[]User ! DbError]
        b := spawn fetchOrders()          // Task[[]Order ! DbError]
        Dashboard{
            users: a.await()?             // ? propagates DbError up
            orders: b.await()?
        }
    }
    result
}
```

When using `?` inside a `scope` body, a failing `await()` cancels the remaining running sibling tasks, then propagates the error to the enclosing function.

**Error propagation semantics:**

| Situation | Behavior |
|---|---|
| Task completes with `Ok(v)` | `await()` returns `v` |
| Task completes with `Err(e)` and `?` is used | Sibling tasks cancelled; error propagates to function |
| Task completes with `Err(e)` and `catch` is used | Error handled locally; siblings continue |
| Multiple tasks fail simultaneously | First error propagates; others are attached as secondary errors |
| Task panics (unrecoverable) | Panic propagates to parent scope, terminates program unless caught at entry |

### 2.4 Cancellation Within Scope

When one task fails and `?` is used, sibling tasks receive a cancellation signal:

```go
scope {
    a := spawn riskyFetch(url1)     // starts running
    b := spawn riskyFetch(url2)     // starts running
    c := spawn riskyFetch(url3)     // starts running
    
    x := a.await()?   // if a fails, b and c receive cancel signal,
                      // wait for them to cooperatively stop,
                      // then the error propagates
    y := b.await()?
    z := c.await()?
    combine(x, y, z)
}
```

The cancellation is **cooperative** — tasks must check for cancellation at their natural yield points. See [section 5](#5-cancellation--timeouts) for details.

### 2.5 Scope as Expression

`scope` is an expression and can appear anywhere an expression is expected:

```go
// In a let binding
result := scope { ... }

// As a function argument
processResult(scope {
    a := spawn computeA()
    b := spawn computeB()
    (a.await(), b.await())
})

// In a pipeline
scope {
    a := spawn fetch(url1)
    b := spawn fetch(url2)
    [a.await()?, b.await()?]
}
|> filter(fn(r) => r.status == 200)
|> map(fn(r) => r.body)
```

### 2.6 Nested Scopes

Scopes can be nested. Inner scopes complete before outer scopes:

```go
scope {
    // Phase 1: fetch data concurrently
    (users, orders) := scope {
        a := spawn fetchUsers()
        b := spawn fetchOrders()
        (a.await()?, b.await()?)
    }
    
    // Phase 2: process concurrently (only starts after phase 1 completes)
    scope {
        spawn processUsers(users)
        spawn processOrders(orders)
    }
    
    // Both phases complete here
}
```

### 2.7 Comparison with Go

```go
// Go — correct WaitGroup usage (easy to get wrong)
var wg sync.WaitGroup
var mu sync.Mutex
var results []Result
var firstErr error

for _, item := range items {
    wg.Add(1)
    go func(i Item) {
        defer wg.Done()
        result, err := process(i)
        mu.Lock()
        if err != nil && firstErr == nil {
            firstErr = err
        }
        results = append(results, result)
        mu.Unlock()
    }(item)
}
wg.Wait()
if firstErr != nil {
    return nil, firstErr
}
```

```go
// Aria — structurally identical semantics, impossible to get wrong
results := scope {
    items.map(fn(item) => spawn process(item)).map(fn(t) => t.await()?)
}
```

| Metric | Go | Aria |
|---|---|---|
| Lines | ~17 | ~4 |
| Tokens | ~120 | ~25 |
| Can leak tasks | Yes | No — impossible |
| Can drop errors | Yes | No — `?` required |
| Can have data race on `results` | Yes | No — type system prevents |
| Correct on first generation | Often not | Yes |

---

## 3. Channels

### 3.1 Channel Types

Channels are typed, first-class values. A channel is a synchronized communication primitive between tasks.

```go
// Unbuffered channel — send blocks until a receiver is ready
ch := chan[str]()

// Buffered channel — send doesn't block until buffer is full
ch := chan[str](buffer: 10)

// Channel type annotation
fn producer(out: chan.Send[str]) { ... }
fn consumer(in: chan.Recv[str]) { ... }
```

Channel type sugar:

| Type | Meaning |
|---|---|
| `chan[T]` | Bidirectional channel (can send and receive) |
| `chan.Send[T]` | Send-only (directional) |
| `chan.Recv[T]` | Receive-only (directional) |

Directional channel types are enforced at compile time. A function taking `chan.Send[str]` cannot accidentally receive from it.

### 3.2 Send and Receive

```go
ch := chan[str](buffer: 5)

// Send — blocks if buffer full (or no receiver for unbuffered)
ch.send("hello")

// Send with error handling (channel closed)
ch.send("hello") catch |ClosedError| { return }

// Receive — blocks until a value is available
msg := ch.recv()           // returns str

// Receive on closed channel returns Option[T]
msg := ch.tryRecv()        // returns str? — None if closed and empty
```

### 3.3 Closing Channels

The **sender** closes the channel to signal "no more values will be sent":

```go
ch := chan[str](buffer: 5)
spawn {
    ch.send("a")
    ch.send("b")
    ch.send("c")
    ch.close()              // signals end of stream
}

// Receiver sees the close
for msg in ch {            // iterates until channel is closed
    process(msg)
}
```

Sending to a closed channel is a **panic** (programming error). Receiving from a closed, empty channel returns `None` (or exits a `for` loop).

### 3.4 Iterating Over Channels

```go
// for...in iterates until the channel is closed
for msg in ch {
    process(msg)
}

// Equivalent desugaring:
loop {
    match ch.tryRecv() {
        Some(msg) => process(msg)
        None => break
    }
}
```

### 3.5 Directional Channels in Function Signatures

Use directional channel types to express intent and prevent misuse:

```go
// Producer — only sends
fn generateNumbers(out: chan.Send[i64], count: i64) {
    for i in 0..count {
        out.send(i)
    }
    out.close()
}

// Consumer — only receives
fn sumNumbers(in: chan.Recv[i64]) -> i64 {
    total := 0i64
    for n in in { total += n }
    total
}

// Wiring them together
fn main() {
    ch := chan[i64](buffer: 100)
    scope {
        spawn generateNumbers(ch, 1000)
        total := sumNumbers(ch)       // ch auto-coerces to chan.Recv[i64]
        println("sum: {total}")
    }
}
```

### 3.6 Channel Interaction with Scope

Channels created inside a `scope` are naturally contained within it. Channels can also be passed across scope boundaries:

```go
// Channel defined outside, passed to inner scope
results := chan[ProcessedItem](buffer: 50)

scope {
    spawn producer(items, results)
    spawn consumer(results)
}
// scope exits after both producer and consumer finish
```

---

## 4. Select Statement

### 4.1 Basic Select

`select` waits on multiple channel operations simultaneously and executes the arm that becomes ready first:

```go
select {
    msg from ch1 => process(msg)
    msg from ch2 => handleOther(msg)
}
```

### 4.2 Timeout Arm

```go
select {
    msg from ch => process(msg)
    after 5s => {
        log.warn("timeout waiting for message")
        return Err(TimeoutError{})
    }
}
```

`after <duration>` creates a one-shot timeout arm. The duration expression can be any value of type `dur`:

```go
timeout := config.requestTimeout    // dur from config
select {
    result from ch => Ok(result)
    after timeout => Err(TimeoutError{after: timeout})
}
```

### 4.3 Default (Non-Blocking) Arm

A `default` arm fires immediately if no other arm is ready:

```go
select {
    msg from ch => process(msg)
    default => {
        doOtherWork()
    }
}
```

This makes `select` non-blocking — it always completes immediately.

### 4.4 Select as Expression

`select` is an expression and returns the value of the executed arm. All arms must return the same type (or a common supertype):

```go
// All arms return str
label := select {
    msg from primary => msg
    msg from fallback => "[fallback] {msg}"
    after 1s => "[timeout]"
}

// In an assignment with error type
result := select {
    val from resultCh => Ok(val)
    err from errorCh => Err(err)
    after 30s => Err(TimeoutError{})
}
```

### 4.5 Select in Loops

The most common pattern: processing messages until a shutdown signal:

```go
loop {
    select {
        req from requests => handleRequest(req)
        _ from shutdown => break
        after 60s => flushMetrics()
    }
}
```

### 4.6 Multiple Sends in Select

`select` can also wait on sends (when buffer might be full):

```go
select {
    ch1.send(value) => log("sent to ch1")
    ch2.send(value) => log("sent to ch2")
    after 1s => log("both full, dropping")
}
```

### 4.7 Grammar

```
select_expr := "select" "{" select_arm+ "}"
select_arm  := (recv_arm | send_arm | after_arm | default_arm) "=>" expr_or_block

recv_arm    := pattern "from" expr
send_arm    := expr "." "send" "(" expr ")"
after_arm   := "after" expr
default_arm := "default"
```

---

## 5. Cancellation & Timeouts

### 5.1 Cancellation Tokens

Cancellation in Aria is modeled as a `Cancel` token that flows through task hierarchies:

```go
// Create a cancellable scope
cancel := Cancel.new()
scope(cancel: cancel) {
    spawn fetchData(url, cancel)
    spawn processStream(stream, cancel)
}

// Cancel from outside (e.g., user interrupt, timeout)
cancel.trigger()

// The scope waits for running tasks to cooperatively stop
```

`Cancel` is passed explicitly — this makes it clear which tasks are cancellable and eliminates the "context threading" verbosity of Go's `context.Context`.

### 5.2 Checking for Cancellation

Tasks cooperatively check for cancellation at natural yield points:

```go
fn processLargeFile(path: str, cancel: Cancel) -> Stats ! IoError {
    reader := io.open(path)?
    defer reader.close()
    
    stats := Stats{}
    for line in reader.lines() {
        cancel.check()?          // returns Err(CancelledError) if cancelled
        stats = stats.process(line)
    }
    stats
}
```

`cancel.check()` returns `() ! CancelledError`. With `?`, a cancelled task unwinds naturally through error propagation.

### 5.3 Timeout Blocks

For scoped timeouts, use `timeout` combined with cancellation:

```go
// Timeout an entire scope
result := scope.timeout(5s) {
    spawn fetch(url1)
    spawn fetch(url2)
    // If not done within 5s, tasks are cancelled and TimeoutError is returned
}?

// Timeout a single operation
value := ch.recv().timeout(100ms)?
```

`scope.timeout(dur)` returns `Result[T, TimeoutError]` where `T` is the scope's return type.

### 5.4 Deadline Propagation

Cancellation tokens compose: a child cancel derives from a parent and is automatically triggered when the parent is:

```go
fn handleRequest(req: Request, requestCancel: Cancel) -> Response ! Error {
    // Create a child token for sub-operations
    fetchCancel := requestCancel.child()
    
    result := scope(cancel: fetchCancel) {
        a := spawn fetchUser(req.userId, fetchCancel)
        b := spawn fetchProfile(req.userId, fetchCancel)
        // If requestCancel fires, fetchCancel fires too
        Response{user: a.await()?, profile: b.await()?}
    }
    result
}
```

### 5.5 Cancellation and `defer`

`defer` runs when a function exits, including on cancellation. This makes cleanup reliable:

```go
fn processWithCleanup(cancel: Cancel) -> () ! Error {
    resource := acquireResource()?
    defer resource.release()        // runs even if cancelled
    
    scope {
        spawn processLoop(resource, cancel)
    }
    // defer fires here, whether we completed or were cancelled
}
```

### 5.6 `Cancel` API Summary

```go
// Create a root cancel token
cancel := Cancel.new()

// Create a child that fires when parent fires
child := cancel.child()

// Create a child that also fires after a timeout
child := cancel.childWithTimeout(5s)

// Fire the token (cooperative — running tasks check at yield points)
cancel.trigger()

// Check in a task — returns Err(CancelledError) if triggered
cancel.check()?

// Observe cancellation without checking (for select arms)
select {
    msg from ch => process(msg)
    _ from cancel.chan() => return Err(CancelledError{})
}
```

---

## 6. Synchronization Primitives

Channels should be the **first tool** for concurrent communication. Reach for `sync` primitives when you need shared mutable state that doesn't fit the channel model.

```go
use sync
```

### 6.1 Mutex

A mutual exclusion lock. Aria's `Mutex` is closure-based to prevent forgetting unlock:

```go
mu := sync.Mutex.new()

// Closure-based locking — unlock is guaranteed
mu.lock(fn(guard) {
    guard.data.count += 1
})

// The guard provides access to the protected data
// When the closure returns, the lock is automatically released
// There is NO way to forget to unlock
```

This is the key difference from Go's `mu.Lock()` / `mu.Unlock()` pair, which requires `defer mu.Unlock()` to be safe. In Aria, the borrow is structural.

```go
// Timed lock attempt (non-blocking)
locked := mu.tryLock(fn(guard) {
    guard.data.count += 1
})
if !locked {
    // could not acquire lock
}
```

### 6.2 Read-Write Mutex

```go
rwmu := sync.RWMutex.new()

// Read lock — multiple readers allowed simultaneously
rwmu.read(fn(guard) {
    value := guard.data.value
    process(value)
})

// Write lock — exclusive
rwmu.write(fn(guard) {
    guard.data.value = newValue
})
```

### 6.3 Atomic Operations

For lock-free shared counters and flags:

```go
// Create an atomic value
counter := sync.Atomic[i64].new(0)

// Common operations
counter.add(1)
counter.sub(1)
counter.store(42)
val := counter.load()

// Compare-and-swap (returns bool: was swap successful?)
swapped := counter.cas(expected: 0, new: 1)

// Fetch-and-add (returns old value)
old := counter.fetchAdd(1)
```

Supported atomic types: `i32`, `i64`, `u32`, `u64`, `bool`, `usize`, pointers.

### 6.4 WaitGroup

`scope` handles most cases where Go uses `WaitGroup`. Use `WaitGroup` only when the number of concurrent tasks is dynamic and not known at scope entry:

```go
wg := sync.WaitGroup.new()

// Dynamic task count
for item in dynamicQueue {
    wg.add(1)
    spawn {
        defer wg.done()
        process(item)
    }
}
wg.wait()
```

Prefer `scope` for the static case. `WaitGroup` is for dynamic dispatch patterns.

### 6.5 Once

Executes a function exactly once, even across concurrent calls:

```go
once := sync.Once.new()
config: Config? = None

fn getConfig() -> Config {
    once.do(fn() {
        config = Some(loadConfig())
    })
    config!         // guaranteed to be Some after once.do
}
```

### 6.6 Barrier

Synchronizes multiple tasks at a common point:

```go
barrier := sync.Barrier.new(workers: 4)

scope {
    for i in 0..4 {
        spawn {
            phase1()
            barrier.wait()    // all 4 tasks must reach here before any continues
            phase2()
        }
    }
}
```

### 6.7 When to Use What

| Situation | Use |
|---|---|
| Producer-consumer communication | `chan[T]` |
| Fan-out work distribution | `chan[T]` + `scope` |
| Multiple concurrent fetches | `scope` with `spawn` |
| Shared mutable counter | `sync.Atomic[T]` |
| Shared mutable data structure | `sync.Mutex` |
| Read-heavy shared data | `sync.RWMutex` |
| One-time initialization | `sync.Once` |
| Phase synchronization | `sync.Barrier` |
| Dynamic task count | `sync.WaitGroup` |

---

## 7. Effect System Integration

### 7.1 The `Async` Effect

Functions that spawn tasks or perform concurrent operations declare the `Async` effect:

```go
// This function can spawn tasks — requires Async effect
fn fetchAll(urls: [str]) -> [str] ! HttpError with [Async, IO] {
    scope {
        urls.map(fn(url) => spawn net.get(url)?)
            .map(fn(t) => t.await()?)
    }
}
```

### 7.2 What `Async` Restricts

The `Async` effect is **not** required to simply call `spawn`. Rather, it's a signal to the caller about concurrent behavior for composition purposes. The key restriction is on `pure` functions:

```go
// pure functions CANNOT use concurrent primitives
pure fn calculate(x: f64, y: f64) -> f64 {
    spawn doSomething()    // COMPILE ERROR: spawn has Async effect, pure forbids it
    x * x + y * y
}

// This is enforced: pure functions are truly pure
pure fn add(a: i64, b: i64) -> i64 = a + b
```

Effect restrictions for `pure`:

| Operation | Effect Required | Allowed in `pure`? |
|---|---|---|
| `spawn` | `Async` | ❌ No |
| `chan.send()` | `Async` | ❌ No |
| `chan.recv()` | `Async` | ❌ No |
| `scope { ... }` | `Async` | ❌ No |
| `net.get(...)` | `IO` | ❌ No |
| `io.readFile(...)` | `IO`, `Fs` | ❌ No |
| `print(...)` | `IO` | ❌ No |
| Arithmetic, logic | — | ✅ Yes |
| Pure function calls | — | ✅ Yes (if callee is also pure) |

### 7.3 Effect Propagation Across Task Boundaries

Effects declared on spawned functions flow into the scope's effect signature:

```go
// This scope spawns IO + Async operations, so it has IO effect
fn loadAll() -> [Data] ! IoError with [IO, Async] {
    scope {
        tasks := urls.map(fn(url) => spawn io.readFile(url))
        tasks.map(fn(t) => t.await()?)
    }
}
```

### 7.4 Effect Inference

The compiler infers effects for unmarked functions when possible. Explicit `with [...]` declarations serve as documentation and compiler-checked contracts:

```go
// Explicitly declared: compiler verifies this is correct
fn heavyCompute(data: [f64]) -> f64 with [Async] {
    scope {
        chunks := data.chunks(1000)
        tasks := chunks.map(fn(c) => spawn sumChunk(c))
        tasks.map(fn(t) => t.await()).sum()
    }
}
```

### 7.5 Error Types Across Task Boundaries

When a spawned task's error propagates back to the parent, the error type must be compatible with the parent's declared error type:

```go
// Task error type must be coercible to parent error type
fn processAll(items: [Item]) -> Summary ! ProcessError with [Async] {
    scope {
        // spawn process(item) returns Task[Result ! ProcessError]
        // The ? inside await propagates ProcessError, which matches our signature
        results := items.map(fn(item) => spawn process(item))
                        .map(fn(t) => t.await()?)
        Summary.from(results)
    }
}
```

---

## 8. Data Race Prevention

### 8.1 The `Send` and `Share` Traits

Types that cross task boundaries must implement appropriate traits:

```go
trait Send {
    // Marker trait: safe to move to another task
    // Automatically implemented for:
    //   - All primitive types (i64, f64, bool, str, etc.)
    //   - Types whose fields are all Send
    //   - chan[T] (channels are inherently thread-safe)
    //   - Immutable references
}

trait Share {
    // Marker trait: safe to share (read) from multiple tasks simultaneously
    // Automatically implemented for:
    //   - Immutable types (no mut fields)
    //   - sync.Atomic[T]
    //   - sync.Mutex (interior mutability wrapper)
}
```

The compiler automatically derives `Send` and `Share` for types where it's sound, and rejects code that would transfer non-`Send` types across task boundaries.

### 8.2 Ownership Transfer Across Task Boundaries

When you pass a value to `spawn`, ownership is **moved** into the task:

```go
data := [1, 2, 3, 4, 5]

spawn processData(data)   // data is moved into the task

// data is no longer accessible here — compiler error if accessed
println(data)             // COMPILE ERROR: data was moved to task
```

This prevents data races by construction: only one task owns the data at a time.

### 8.3 Sharing Immutable Data

Immutable data can be shared across tasks without ownership transfer:

```go
// config is immutable (no mut qualifier)
config := Config{timeout: 30s, retries: 3}

scope {
    // config is shared by reference — no copy, no race
    spawn processA(config)
    spawn processB(config)
    spawn processC(config)
}
```

The compiler verifies that `config` is not mutated after the spawn points, making shared read-only access safe.

### 8.4 Mutable Shared State

To share mutable state, wrap it in a `sync.Mutex` or use `sync.Atomic`:

```go
// Shared mutable counter — wrap in Atomic
counter := sync.Atomic[i64].new(0)

scope {
    for _ in 0..1000 {
        spawn { counter.add(1) }    // atomic — no race
    }
}
println(counter.load())    // 1000

// Shared mutable struct — wrap in Mutex
state := sync.Mutex.new(ServerState{connections: 0, requests: 0})

scope {
    spawn acceptLoop(state)
    spawn metricsLoop(state)
}
```

### 8.5 Channels Are Move Semantics

Sending a value through a channel moves it — the sender no longer owns it:

```go
data := buildExpensiveData()

ch.send(data)          // data is moved into the channel

// data is no longer accessible — compiler error
process(data)          // COMPILE ERROR: data was moved to channel
```

This ensures the receiver is the sole owner of the data, preventing any concurrent access.

### 8.6 The `Clone` Escape Hatch

When you genuinely need multiple owners, `.clone()` creates an independent copy:

```go
config := loadConfig()

scope {
    spawn taskA(config.clone())     // A gets its own copy
    spawn taskB(config.clone())     // B gets its own copy
    spawn taskC(config)             // C takes ownership of original
}
```

`Clone` is explicit — the compiler never silently copies data. If you see a `.clone()`, someone made a deliberate choice.

---

## 9. Concurrent Data Structures

These live in the `sync` module and are safe to use from multiple tasks without additional locking.

```go
use sync
```

### 9.1 `sync.Map[K, V]`

A concurrent hash map safe for use from multiple tasks simultaneously:

```go
cache := sync.Map[str, []byte].new()

// Insert
cache.set("key", data)

// Retrieve
value := cache.get("key")     // returns V?

// Remove
cache.delete("key")

// Get-or-insert (atomic)
value := cache.getOrInsert("key", fn() => computeExpensive())

// Iterate (snapshot iteration — consistent view)
for (key, value) in cache.snapshot() {
    process(key, value)
}
```

### 9.2 `sync.Queue[T]`

An unbounded concurrent FIFO queue:

```go
queue := sync.Queue[WorkItem].new()

// Multiple producers can push concurrently
queue.push(WorkItem{...})

// Multiple consumers can pop concurrently
item := queue.pop()           // returns T? — None if empty
item := queue.popWait()       // blocks until item available

// Size (approximate — may not be perfectly consistent under contention)
n := queue.len()
```

### 9.3 `sync.Pool[T]`

A concurrent object pool for reducing allocation pressure on hot paths:

```go
pool := sync.Pool[Buffer].new(
    create: fn() => Buffer.withCapacity(4096)
    reset: fn(buf) { buf.clear() }         // optional: called on return
)

fn handleRequest(req: Request) {
    buf := pool.get()
    defer pool.put(buf)          // returns to pool after function exits
    
    buf.write(req.body)
    processBuffer(buf)
}
```

The pool does not guarantee a specific capacity — it's a performance optimization, not a strict resource manager. Use `sync.Mutex` + a slice if you need guaranteed bounding.

---

## 10. Common Patterns

### 10.1 Fan-Out / Fan-In

Distribute work across many tasks, collect results:

```go
fn fanOut(items: [Item]) -> [Result] ! ProcessError {
    scope {
        // Fan-out: spawn a task per item
        tasks := items.map(fn(item) => spawn process(item))
        
        // Fan-in: collect all results
        tasks.map(fn(t) => t.await()?)
    }
}
```

With bounded concurrency (limit to N concurrent tasks):

```go
fn fanOutBounded(items: [Item], concurrency: u64) -> [Result] ! ProcessError {
    sem := chan[()]( buffer: concurrency)    // semaphore channel
    
    scope {
        tasks := items.map(fn(item) => spawn {
            sem.send(())              // acquire slot
            defer sem.recv()          // release slot
            process(item)?
        })
        tasks.map(fn(t) => t.await()?)
    }
}
```

### 10.2 Worker Pool

A fixed pool of workers consuming from a shared queue:

```go
fn workerPool(jobs: [Job], workers: u64) -> [Result] ! JobError {
    jobCh := chan[Job](buffer: jobs.len())
    resultCh := chan[Result](buffer: jobs.len())
    
    // Load jobs into channel
    for job in jobs { jobCh.send(job) }
    jobCh.close()
    
    scope {
        // Start fixed number of workers
        for _ in 0..workers {
            spawn {
                for job in jobCh {           // each worker pulls from shared queue
                    result := processJob(job)?
                    resultCh.send(result)
                }
            }
        }
    }
    // scope waits for all workers to drain jobCh
    resultCh.close()
    
    // Collect results
    [r for r in resultCh]
}
```

### 10.3 Producer-Consumer

Decouple production from consumption with a buffered channel:

```go
fn producerConsumer() -> () ! Error {
    ch := chan[RawData](buffer: 100)
    
    scope {
        // Producer
        spawn {
            defer ch.close()
            for item in dataSource() {
                ch.send(item)
            }
        }
        
        // Consumer
        spawn {
            for raw in ch {
                processed := transform(raw)?
                store(processed)?
            }
        }
    }
}
```

### 10.4 Pipeline Processing with Concurrency

Chain concurrent stages where each stage is a goroutine:

```go
fn concurrentPipeline(input: [RawRecord]) -> [EnrichedRecord] ! Error {
    // Stage channels
    parsed := chan[ParsedRecord](buffer: 50)
    validated := chan[ValidRecord](buffer: 50)
    enriched := chan[EnrichedRecord](buffer: 50)
    
    scope {
        // Stage 1: Parse
        spawn {
            defer parsed.close()
            for raw in input {
                parsed.send(parse(raw)?)
            }
        }
        
        // Stage 2: Validate
        spawn {
            defer validated.close()
            for record in parsed {
                validated.send(validate(record)?)
            }
        }
        
        // Stage 3: Enrich
        spawn {
            defer enriched.close()
            for record in validated {
                enriched.send(enrich(record)?)
            }
        }
        
        // Collect
        spawn { [r for r in enriched] }
    }
}
```

This is the **pipeline pattern**: each stage is a task reading from one channel and writing to the next. Backpressure is automatic — a slow downstream stage will slow upstream stages through channel blocking.

### 10.5 Graceful Shutdown

Use a cancel token to signal all tasks to stop:

```go
fn runServer(config: ServerConfig) -> () ! ServerError {
    cancel := Cancel.new()
    
    // Hook OS signals to trigger cancellation
    os.onSignal([.SIGTERM, .SIGINT], fn() { cancel.trigger() })
    
    scope(cancel: cancel) {
        spawn acceptConnections(config, cancel)
        spawn runMetrics(config.metricsPort, cancel)
        spawn runHealthCheck(config.healthPort, cancel)
        
        // Wait for cancel signal
        cancel.wait()    // blocks until triggered
        
        log.info("shutdown signal received, draining...")
    }
    // scope waits for all three tasks to finish their cleanup
    log.info("shutdown complete")
}
```

### 10.6 Rate Limiting

Use a ticker channel to implement rate-limited work:

```go
fn rateLimitedProcessor(items: [Item], rate: u64) -> () ! Error {
    // Ticker fires `rate` times per second
    ticker := time.ticker(interval: 1s / rate)
    defer ticker.stop()
    
    for item in items {
        ticker.wait()           // wait for next tick before processing
        spawn process(item)
    }
}
```

Or use a token bucket for bursting:

```go
fn withRateLimit(rate: u64, burst: u64, f: fn() -> () ! Error) -> () ! Error {
    bucket := sync.TokenBucket.new(rate: rate, burst: burst)
    
    loop {
        bucket.acquire()?    // blocks until token available
        f()?
    }
}
```

### 10.7 Broadcast / Pub-Sub

Fan a message out to multiple subscribers:

```go
type Broadcast[T] {
    subscribers: sync.Mutex[[[chan.Send[T]]]]
}

impl Broadcast[T] {
    fn new() -> Broadcast[T] = Broadcast{subscribers: sync.Mutex.new([])}
    
    fn subscribe(self) -> chan.Recv[T] {
        ch := chan[T](buffer: 10)
        self.subscribers.lock(fn(subs) {
            subs.push(ch)
        })
        ch
    }
    
    fn publish(self, msg: T) {
        self.subscribers.lock(fn(subs) {
            for sub in subs { sub.send(msg.clone()) }
        })
    }
}
```

---

## 11. Token Cost Comparison

### 11.1 Concurrent Fetch: Fan-Out

**Task**: Fetch N URLs concurrently, return all results.

```go
// Go — correct implementation
func fetchAll(urls []string) ([]string, error) {
    results := make([]string, len(urls))
    errs := make([]error, len(urls))
    var wg sync.WaitGroup
    for i, url := range urls {
        wg.Add(1)
        go func(idx int, u string) {
            defer wg.Done()
            resp, err := http.Get(u)
            if err != nil {
                errs[idx] = err
                return
            }
            defer resp.Body.Close()
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                errs[idx] = err
                return
            }
            results[idx] = string(body)
        }(i, url)
    }
    wg.Wait()
    for _, err := range errs {
        if err != nil { return nil, err }
    }
    return results, nil
}
```

```go
// Aria
fn fetchAll(urls: [str]) -> [str] ! HttpError {
    scope {
        urls.map(fn(url) => spawn net.get(url)?).map(fn(t) => t.await()?)
    }
}
```

| Metric | Go | Aria |
|---|---|---|
| Lines | ~20 | ~3 |
| Tokens | ~150 | ~25 |
| Can leak goroutines | Yes (if WaitGroup misused) | No |
| Can silently drop errors | Yes | No |
| Index synchronization bug possible | Yes | No |

### 11.2 Worker Pool

**Task**: Process N items with a bounded pool of W workers.

```go
// Go
func workerPool(jobs []Job, workers int) ([]Result, error) {
    jobCh := make(chan Job, len(jobs))
    resultCh := make(chan Result, len(jobs))
    errCh := make(chan error, 1)
    
    var wg sync.WaitGroup
    for w := 0; w < workers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobCh {
                result, err := processJob(job)
                if err != nil {
                    select {
                    case errCh <- err:
                    default:
                    }
                    return
                }
                resultCh <- result
            }
        }()
    }
    
    for _, job := range jobs {
        jobCh <- job
    }
    close(jobCh)
    
    wg.Wait()
    close(resultCh)
    
    select {
    case err := <-errCh:
        return nil, err
    default:
    }
    
    var results []Result
    for r := range resultCh {
        results = append(results, r)
    }
    return results, nil
}
```

```go
// Aria
fn workerPool(jobs: [Job], workers: u64) -> [Result] ! JobError {
    jobCh := chan[Job](buffer: jobs.len())
    resultCh := chan[Result](buffer: jobs.len())
    
    for job in jobs { jobCh.send(job) }
    jobCh.close()
    
    scope {
        for _ in 0..workers {
            spawn {
                for job in jobCh { resultCh.send(processJob(job)?) }
            }
        }
    }
    resultCh.close()
    [r for r in resultCh]
}
```

| Metric | Go | Aria |
|---|---|---|
| Lines | ~38 | ~13 |
| Tokens | ~280 | ~70 |
| Error handling correct | Requires careful design | Structural |
| Goroutine leak possible | Yes | No |

### 11.3 Select with Timeout

**Task**: Read from channel with timeout fallback.

```go
// Go
select {
case msg := <-ch:
    process(msg)
case <-time.After(5 * time.Second):
    handleTimeout()
}
```

```go
// Aria
select {
    msg from ch => process(msg)
    after 5s => handleTimeout()
}
```

| Metric | Go | Aria |
|---|---|---|
| Tokens | ~20 | ~10 |
| Time.After leak possible | Yes (timer not stopped) | No |

### 11.4 Producer-Consumer Pipeline

**Task**: Parse, validate, and store a stream of records concurrently.

```go
// Go — ~45 lines, 3 goroutines, manual channel management, 2 WaitGroups
// (omitted for brevity — the pattern requires ~45 lines minimum)
```

```go
// Aria
fn pipeline(records: [RawRecord]) -> () ! PipelineError {
    parsed := chan[ParsedRecord](buffer: 50)
    stored := chan[StoredRecord](buffer: 50)
    
    scope {
        spawn {
            defer parsed.close()
            for r in records { parsed.send(parseRecord(r)?) }
        }
        spawn {
            defer stored.close()
            for r in parsed { stored.send(validateAndStore(r)?) }
        }
        spawn {
            for r in stored { notifyDownstream(r)? }
        }
    }
}
```

| Metric | Go | Aria |
|---|---|---|
| Lines | ~45 | ~14 |
| Tokens | ~330 | ~80 |

### 11.5 Overall Token Savings

| Pattern | Go Tokens | Aria Tokens | Reduction |
|---|---|---|---|
| Concurrent fetch (N URLs) | ~150 | ~25 | **83%** |
| Worker pool (N jobs, W workers) | ~280 | ~70 | **75%** |
| Select with timeout | ~20 | ~10 | **50%** |
| Producer-consumer pipeline | ~330 | ~80 | **76%** |
| Mutex-protected counter | ~15 | ~8 | **47%** |
| Graceful shutdown | ~50 | ~20 | **60%** |
| **Average** | | | **~65%** |

For a service with heavy concurrent logic (a typical API server), this translates to **300-500 fewer tokens** per major concurrent subsystem — and structurally fewer bugs.

---

## Appendix: Grammar Summary

```
// Task creation
spawn_expr   := "spawn" ("(" spawn_opts ")")? (call_expr | block)
spawn_opts   := "stack" ":" size_expr ("," spawn_opts)?
               | "detach"

// Structured concurrency
scope_expr   := "scope" ("(" scope_opts ")")? block
scope_opts   := "cancel" ":" expr ("," scope_opts)?
               | "timeout" ":" expr

// Channels
chan_type     := "chan" "[" type "]" ("(" "buffer" ":" expr ")")?
chan_send_t   := "chan.Send" "[" type "]"
chan_recv_t   := "chan.Recv" "[" type "]"

// Select
select_expr  := "select" "{" select_arm+ "}"
select_arm   := (recv_arm | send_arm | after_arm | default_arm) "=>" (expr | block)
recv_arm     := pattern "from" expr
send_arm     := expr "." "send" "(" expr ")"
after_arm    := "after" expr
default_arm  := "default"

// Cancellation
cancel_expr  := "Cancel" "." "new" "(" ")"
             | cancel_val "." "child" "(" ")"
             | cancel_val "." "childWithTimeout" "(" expr ")"
             | cancel_val "." "trigger" "(" ")"
             | cancel_val "." "check" "(" ")" "?"
             | cancel_val "." "wait" "(" ")"
             | cancel_val "." "chan" "(" ")"
```

---

## Appendix: Design Decision Log

### Why No `async`/`await` Keywords?

The async/await coloring problem splits every function into "sync" and "async" variants, doubles the cognitive overhead, and introduces a class of bugs (forgetting `await`) that Aria's design eliminates entirely. By making all I/O suspension transparent at the runtime level, functions look and read as synchronous code while executing concurrently. The cost is a slightly more complex runtime; the benefit is a uniform, simpler language model.

### Why Closure-Based Mutex Instead of Guard Types?

Go's `sync.Mutex` requires paired `Lock()`/`Unlock()` calls. Even with `defer mu.Unlock()`, there are cases where unlock is missed (returning inside a switch, for example). Rust's `MutexGuard` RAII approach is safer but requires understanding the borrow checker. Aria's closure-based approach is structurally safe: the lock is always released when the closure returns, and there is no way to forget it. The ergonomic cost is minimal (a closure literal vs. a `defer`), and the safety gain is significant.

### Why Move Semantics for Channel Sends?

If channel sends copied by default, large data structures (byte slices, large structs) would pay unnecessary allocation and copy costs on every send. If they shared by reference, the sender could modify the data while the receiver is reading it. Move semantics are the correct default: the sender relinquishes ownership, the receiver gets exclusive access, and no copy is made. For cases where the sender needs to retain data, `.clone()` is explicit.

### Why Explicit Cancellation Tokens Instead of Context Threading?

Go's `context.Context` must be passed as the first argument to every function in a cancellable call chain — even functions that don't use it themselves but call functions that do. This "context threading" adds tokens and cognitive overhead to every function signature. Aria's `Cancel` token is still explicit (hidden cancellation is worse), but it's only passed to functions that actively check for cancellation, not threaded through every function in the call chain. Functions that don't check for cancellation don't need to accept or forward a `Cancel` parameter.
