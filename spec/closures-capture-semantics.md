# Aria Closures and Capture Semantics

## Design Philosophy

Closures in Aria follow one rule: **capture by value (copy) by default, explicit `ref` for reference capture.** This eliminates the most common closure bug in programming — the loop capture problem — while keeping the syntax minimal.

Key principles:
- **Copy by default eliminates the loop closure bug** — each closure gets its own copy of captured variables
- **Explicit `ref` capture when mutation of the outer variable is needed** — no accidental aliasing
- **One closure type in signatures** — `fn(Args) -> Return`, not three traits like Rust's `Fn`/`FnMut`/`FnOnce`
- **`once` annotation for consuming closures** — when a closure consumes a captured value and can only be called once

---

## Closure Syntax

Two forms — short (single expression) and block:

```
// Short form (single expression, => arrow)
fn(x) => x * 2
fn(a, b) => a + b
fn() => 42

// Block form
fn(x) => {
    validated := validate(x)?
    transform(validated)
}

// Type annotations (usually inferred, available when needed)
fn(x: i64) -> i64 => x * 2

// No-argument closure
fn() => doSomething()
fn() => {
    setup()
    execute()
}
```

---

## Closure Type Syntax

In function signatures and type annotations:

```
fn(i64) -> str                    // takes i64, returns str
fn()                              // takes nothing, returns void
fn(str, str) -> bool              // comparator
fn once() -> Connection           // can only be called once (consumes captures)

// In function signatures
fn map[T, U](list: [T], f: fn(T) -> U) -> [U]
fn sort[T](list: [T], cmp: fn(T, T) -> bool) -> [T]
fn withResource[T](init: fn once() -> T, body: fn(T) -> void)
```

---

## Capture Semantics

### Default: Capture by Value (Copy)

When a closure references a variable from its enclosing scope, it captures a **copy** of that variable's current value at the point the closure is created.

```
x := 42
doubled := fn() -> i64 { x * 2 }    // captures a COPY of x (42)
x = 100                               // mutating x doesn't affect the closure
assert doubled() == 84                // still uses the captured copy (42 * 2)
```

**Why this is the default:**
- Eliminates the classic loop closure bug:
```
// In Go, this is a bug — all closures share the same `i`:
// for i := range items { go func() { use(i) }() }

// In Aria, this just works — each closure gets its own copy:
fns := for i in 0..5 { fn() => i }
assert fns[0]() == 0
assert fns[4]() == 4
// Each closure captured its own copy of i at creation time
```
- No lifetime concerns — the closure owns its data
- The compiler can optimize to reference capture when it proves no mutation occurs on either side (this is an optimization, not a semantic change)

### Explicit Reference Capture: `ref`

When a closure needs to read or mutate the original variable (not a copy), use explicit `ref` capture in the capture list:

```
// Capture list syntax: fn[captures](args) => body
counter := 0
increment := fn[ref counter]() { counter += 1 }
increment()
increment()
assert counter == 2    // the closure mutated the original variable

// Multiple ref captures
a := 0
b := 0
swap := fn[ref a, ref b]() {
    temp := a
    a = b
    b = temp
}
```

**Rules for `ref` capture:**
- The captured variable must outlive the closure — the compiler verifies this
- Multiple closures cannot hold `ref` captures to the same mutable variable simultaneously (prevents data races)
- A `ref` capture of an immutable variable is allowed (read-only reference)
- `ref` captures cannot be sent across `spawn` boundaries (would create data races)

### Mixed Capture

Closures can mix value and reference captures:

```
multiplier := 3                        // captured by value (copy)
count := 0                             // captured by reference
process := fn[ref count](x: i64) -> i64 {
    count += 1                         // mutates the original
    x * multiplier                     // uses the copied value
}
```

Variables not listed in the capture list are captured by value (the default). Only variables explicitly marked `ref` are captured by reference.

### `once` Closures

A closure annotated `once` can only be called a single time. This is for closures that consume (move) a captured value:

```
fn withConnection(factory: fn once() -> Connection) {
    conn := factory()     // consumes the closure
    // factory()          // ❌ compile error: closure already consumed
    use(conn)
}

// Creating a once closure
resource := acquireResource()
cleanup := fn once() => {
    resource.close()       // moves resource into the closure
}
// resource is no longer accessible here — it was moved into the closure
cleanup()                  // works once
// cleanup()              // ❌ compile error
```

**Rules for `once`:**
- The closure type must be declared `fn once(...)` in the signature
- The closure consumes captured values — they are moved, not copied
- Calling a `once` closure more than once is a compile error
- Useful for resource cleanup, initialization factories, and single-use callbacks

---

## Closures and Concurrency

```
// Value-captured closures can be spawned freely
x := 42
spawn fn() { use(x) }          // ✅ x is copied into the spawned task

// Ref-captured closures CANNOT be spawned
counter := 0
bad := fn[ref counter]() { counter += 1 }
spawn bad                       // ❌ compile error: ref captures cannot cross spawn boundary

// Use channels or atomics instead
counter := sync.Atomic[i64].new(0)
spawn fn() { counter.add(1) }  // ✅ Atomic is safe to share
```

---

## Closures and the Pipeline Operator

Closures compose naturally with pipelines (see `paradigm-design.md` for full pipeline operator semantics):

```
items
    |> filter(fn(x) => x.active)
    |> map(fn(x) => x.name)
    |> sortBy(fn(a, b) => a < b)
    |> take(10)
```

---

## Closures as Methods (Field Shorthand)

When passing a field accessor or single-method call, use the field shorthand syntax:

```
// Field shorthand — .fieldName is sugar for fn(x) => x.fieldName
names := users |> map(.name)          // equivalent to map(fn(u) => u.name)
prices := items |> map(.price)
actives := users |> filter(.active)   // works for bool fields

// Method shorthand
lengths := strings |> map(.len())     // equivalent to map(fn(s) => s.len())
```

---

## Type Inference

Closure parameter and return types are inferred from context in most cases:

```
// Inferred from map's signature: fn(User) -> str
names := users.map(fn(u) => u.name)

// Inferred from filter's signature: fn(User) -> bool
active := users.filter(fn(u) => u.active)

// Explicit types when inference can't determine (rare)
transform := fn(x: i64) -> str => x.toStr()
```

---

## Design Rationale Summary

| Decision | Rationale |
|---|---|
| Copy capture by default | Eliminates loop closure bug, no lifetime concerns |
| Explicit `ref` capture | Mutation intent is visible in the code |
| One closure type (`fn`) | No `Fn`/`FnMut`/`FnOnce` complexity — fewer tokens in bounds |
| `once` annotation | Consuming closures are rare and should be marked |
| No ref captures across spawn | Prevents data races at compile time |
| Field shorthand (`.name`) | Saves tokens on the most common closure pattern |

---

## Related Specs

- `high-level-design.md` — Closure syntax overview and use in `.map()` / `.filter()`
- `spec/paradigm-design.md` — Functional composition, pipeline operator
- `spec/iteration-protocol.md` — How closures integrate with iterator chains
