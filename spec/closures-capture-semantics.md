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

## Named Functions as Values

Named functions are values. A function declared with `fn` can be passed directly wherever a matching `fn(Args) -> Return` type is expected — no wrapping closure needed.

```
fn isPositive(n: i64) -> bool = n > 0
fn double(n: i64) -> i64 = n * 2
fn descending(a: i64, b: i64) -> bool = a > b

// Pass named functions directly — no closure wrapper
positives := numbers.filter(isPositive)
doubled := numbers.map(double)
sorted := numbers.sort(descending)
```

The function name evaluates to a value of the corresponding function type:

| Declaration | Value type |
|---|---|
| `fn isPositive(n: i64) -> bool` | `fn(i64) -> bool` |
| `fn add(a: i64, b: i64) -> i64` | `fn(i64, i64) -> i64` |
| `fn greet(name: str) with [Io]` | `fn(str) with [Io]` |
| `fn readFile(path: str) -> str ! IoError with [Io, Fs]` | `fn(str) -> str ! IoError with [Io, Fs]` |

### Storing named functions

```
transforms: [fn(i64) -> i64] = [double, negate, abs]

dispatch: Map[str, fn(Request) -> Response ! Error] = {
    "GET": handleGet,
    "POST": handlePost,
    "DELETE": handleDelete,
}
```

### Named functions vs closures

A named function that captures nothing is interchangeable with a closure that captures nothing:

```
// These are equivalent:
numbers.filter(isPositive)
numbers.filter(fn(n) => n > 0)

// Both have type fn(i64) -> bool
```

The compiler may optimize named functions to direct calls (no boxing) when the target function is known at compile time.

**AI rationale:** When I generate pipeline code, I often define helper functions above and pass them by name. `items.filter(isValid).map(transform).sort(byScore)` reads as a sentence. Each function name is self-documenting. Wrapping each one in `fn(x) => isValid(x)` adds 5 tokens per usage and conveys zero additional information.

---

## Method References

A method can be referenced as a function value using `Type.method` syntax. The receiver becomes the first parameter of the resulting function.

### Instance method references

```
impl User {
    fn isActive(self) -> bool = self.active
    fn displayName(self) -> str = "{self.firstName} {self.lastName}"
    fn age(self) -> u8 = self.age
}

// User.isActive is a value of type fn(User) -> bool
activeUsers := users.filter(User.isActive)

// User.displayName is a value of type fn(User) -> str
names := users.map(User.displayName)

// Equivalent to:
activeUsers := users.filter(fn(u) => u.isActive())
names := users.map(fn(u) => u.displayName())
```

### Method reference types

The method reference `Type.method` produces a function where `self` is the first parameter:

| Method declaration | Reference | Resulting type |
|---|---|---|
| `fn isActive(self) -> bool` | `User.isActive` | `fn(User) -> bool` |
| `fn fullName(self) -> str` | `User.fullName` | `fn(User) -> str` |
| `fn distance(self, other: Point) -> f64` | `Point.distance` | `fn(Point, Point) -> f64` |
| `fn validate(self) -> bool ! Error` | `Email.validate` | `fn(Email) -> bool ! Error` |

### Trait method references

Trait methods can be referenced the same way:

```
trait Display {
    fn display(self) -> str
}

// Display.display is fn(T) -> str where T: Display
items.map(Display.display)

// Equivalent to:
items.map(fn(x) => x.display())
```

### Chaining with pipelines

Method references combine naturally with the pipeline operator:

```
// Read top-to-bottom: get users, keep active ones, extract names, sort
result := db.getUsers()?
    |> filter(User.isActive)
    |> map(User.displayName)
    |> sort(str.cmp)
```

### Static method references

Static methods (no `self`) are already function values:

```
impl Circle {
    fn new(radius: f64) -> Circle = Circle { radius }
}

// Circle.new is fn(f64) -> Circle
circles := radii.map(Circle.new)
```

**AI rationale:** Method references eliminate the most common closure boilerplate — `fn(x) => x.someMethod()`. In pipeline-heavy code, this saves 5-7 tokens per stage and makes the code read as a chain of transformations by name. When I generate `users.filter(User.isActive).map(User.displayName)`, every token carries meaning. Compare to `users.filter(fn(u) => u.isActive()).map(fn(u) => u.displayName())` — same semantics, 60% more tokens.

---

## Named Closures in Stack Traces

When an error occurs inside a closure, the stack trace includes context about where the closure was defined and how it was passed. Anonymous closures are identified by their call site, not just as `<closure>`.

### Stack trace format

```
// Given this code:
fn processUsers(users: [User]) -> [str] ! Error {
    users
        .filter(fn(u) => u.active)             // line 3
        .map(fn(u) => u.name.toUpper()?)       // line 4 — error here
        .collect()
}
```

Stack trace on error:

```
Error: StringError.InvalidUtf8
  at: closure passed to map    src/users.aria:4:26
      fn(u) => u.name.toUpper()?
                      ^^^^^^^^^^
  in: processUsers              src/users.aria:2:5
  in: handleRequest             src/server.aria:45:12
```

### What the trace shows

1. **"closure passed to map"** — the compiler knows this closure was the argument to `.map()` and names it accordingly
2. **The closure source text** — the actual closure expression is shown
3. **The specific sub-expression that failed** — `u.name.toUpper()?` with a pointer to the error site
4. **The containing function** — `processUsers` with file and line

### Named closures bound to variables

When a closure is assigned to a named binding, the stack trace uses that name:

```
validator := fn(email: str) -> bool ! ValidationError {
    if !email.contains("@") {
        return Err(ValidationError{msg: "missing @"})
    }
    true
}

emails.map(validator)    // if this fails:
```

Stack trace:
```
Error: ValidationError{msg: "missing @"}
  at: validator                 src/validate.aria:3:16
  in: emails.map(validator)     src/validate.aria:9:8
```

The name `validator` appears because the closure was bound to it.

### Named function references in traces

When a named function or method reference is used, the trace shows the real function name:

```
users.filter(User.isActive)    // if this fails:
```

Stack trace:
```
Error: ...
  at: User.isActive             src/models.aria:15:5
  in: users.filter(User.isActive)  src/handler.aria:8:12
```

### JSON diagnostic format

In structured output (`--format=json`), closures include their context:

```json
{
  "frame": {
    "function": "closure",
    "context": "passed to map",
    "source_text": "fn(u) => u.name.toUpper()?",
    "file": "src/users.aria",
    "line": 4,
    "column": 26
  }
}
```

**AI rationale:** When I run tests and a failure occurs inside a closure, `<anonymous>` in the stack trace tells me nothing. "closure passed to map at users.aria:4" tells me exactly where to look. Showing the closure source text means I don't even need to open the file — the diagnostic contains the failing code. This turns a multi-round debugging session into a single round: read trace → see the closure → fix it.

---

## Partial Application (Deferred to v0.2)

Partial application using `_` placeholders is **not supported in Aria v0.1**. It is deferred to v0.2 pending real-world usage data.

### What it would look like

```
// Hypothetical v0.2 syntax:
fn multiply(a: i64, b: i64) -> i64 = a * b

doubler := multiply(2, _)           // fn(i64) -> i64 — first arg bound to 2
doubler(5)                           // 10

// In pipelines:
items |> map(multiply(2, _)) |> filter(greaterThan(_, threshold))
```

### Why deferred

1. The closure form works today and is explicit: `fn(x) => multiply(2, x)`
2. `_` is already used as the wildcard pattern in `match` — adding it as a placeholder in expressions creates parser ambiguity
3. The token savings (~5 per usage) are real but not large enough to justify the complexity in v0.1
4. Real-world Aria code will reveal whether pipelines are common enough to warrant this sugar

### The v0.1 alternative

Use a closure:

```
// These are the v0.1 equivalents:
items |> map(fn(x) => multiply(2, x))
items |> filter(fn(x) => greaterThan(x, threshold))
```

Or define named helpers:

```
fn doubleIt(x: i64) -> i64 = multiply(2, x)
items |> map(doubleIt)
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
| Named functions as values | `filter(isPositive)` — no wrapper needed, reads as prose |
| Method references (`Type.method`) | `map(User.displayName)` — eliminates `fn(x) => x.method()` boilerplate |
| Contextual closure names in traces | "closure passed to map" not "`<anonymous>`" — one-round debugging |
| Partial application deferred to v0.2 | Closure form works; `_` ambiguity with wildcard needs careful design |

---

## Related Specs

- [high-level-design.md](../high-level-design.md) — Closure syntax overview and use in `.map()` / `.filter()`
- [spec/paradigm-design.md](paradigm-design.md) — Functional composition, pipeline operator
- [spec/iteration-protocol.md](iteration-protocol.md) — How closures integrate with iterator chains
- [spec/trait-system.md](trait-system.md) — Trait method references
- [spec/effect-system.md](effect-system.md) — Effect inference on closures
- [spec/compiler-diagnostics.md](compiler-diagnostics.md) — Structured stack trace format
- [spec/design-decisions-v01.md](design-decisions-v01.md) — Decision 7: closures are GC-boxed by default
