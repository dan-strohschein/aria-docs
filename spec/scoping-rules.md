# Aria v0.1 — Scoping Rules & Name Resolution

> This document defines Aria's scoping rules, name resolution algorithm, variable shadowing behavior,
> and the lifecycle of bindings throughout program execution.
> These rules are critical for compiler implementors and for understanding how Aria code behaves.

---

## 1. Scope Hierarchy

Aria uses **lexical (static) scoping**. The scope of a name is determined by where it appears in the source text, not by the runtime call stack.

Scopes are arranged in a hierarchy from outermost to innermost:

| Level | Scope | Contents |
|------:|-------|----------|
| 1 | **Universe scope** | Built-in types (`i64`, `u8`, `f64`, `str`, `bool`, `byte`, `dur`, `usize`, `Option`, `Result`, `Range`), built-in functions (`print`, `println`, `panic`, `assert`) |
| 2 | **Package scope** | All `pub` and `pub(pkg)` items from every module in the same package (directory) |
| 3 | **Module scope** | All top-level declarations in the current file: `fn`, `type`, `enum`, `const`, `trait`, `impl`, `test` bodies, `entry` block |
| 4 | **Import scope** | Names introduced by `use` declarations (available throughout the module) |
| 5 | **Function scope** | Function parameters; lives for the duration of the function call |
| 6 | **Block scope** | Any `{ }` block, including those introduced by `if`, `else`, `for`, `while`, `loop`, `match` arms, `scope`, `with`, `catch`, `select` arms |

Each inner scope can access names from all outer scopes. The inner scope's bindings take precedence over outer bindings of the same name (shadowing — see §2).

---

## 2. Variable Shadowing

**Shadowing IS allowed in Aria.** This is a deliberate design choice.

Shadowing creates a **new, independent binding** with the same name. The old binding becomes inaccessible within the new binding's scope. The old binding is not mutated — it simply becomes unreachable from that point.

```aria
x := 5
x := "hello"      // OK — new binding of type str; the previous i64 binding is now shadowed
                   // The string literal "hello" does not reference x at all;
                   // the new binding simply replaces the old one in the current scope

// Shadowing in a nested block:
y := 10
if condition {
    y := 20           // shadows outer y WITHIN this block
    println(y)        // prints 20
}
println(y)            // prints 10 — the outer y is unaffected
```

### Rules

1. Shadowing creates a NEW binding. It does **not** mutate the old one.
2. The old binding becomes inaccessible in the current scope from the point of the shadow declaration onward.
3. Shadowed bindings in **outer scopes** are NOT affected — they remain accessible after the inner scope ends.
4. Shadowing works at all scope levels: you can shadow function parameters, imported names, or module-level constants.
5. `mut` does NOT affect shadowing — you can shadow an immutable binding with a mutable one and vice versa.
6. The initializer expression of the shadowing binding may reference the **current value of the shadowed name** (it is evaluated before the new binding is created):

```aria
x := 5
x := x + 1    // OK: right-hand side sees the old x (5), then creates new x = 6
x := x * 2    // OK: right-hand side sees x = 6, creates new x = 12
```

> **Design Decision:** Shadowing in the same block is permitted (unlike Go, which only allows shadowing in inner blocks). This decision prioritizes AI code generation: when transforming a value through multiple stages, the same logical name can be reused at each stage without inventing synthetic names.

### Why Shadowing Matters for AI Code Generation

Without shadowing, transforming a value through stages requires inventing names:

```aria
// Without shadowing — must invent unique names
configRaw := fs.read("config.json")?
configStr := configRaw.trim()
configParsed := json.parse[Config](configStr)?
configValidated := validate(configParsed)?
```

With shadowing, the logical identity of "the config" is preserved:

```aria
// With shadowing — each step is the same logical entity
config := fs.read("config.json")?
config := config.trim()
config := json.parse[Config](config)?
config := validate(config)?
```

This produces 3–4x fewer unique name tokens over a typical program, directly reducing AI generation cost and error surface.

---

## 3. Block Scoping Rules

Every `{ }` block creates a new scope. The following constructs introduce new scopes:

| Construct | New Scope? | Bindings Introduced | Example |
|-----------|-----------|---------------------|---------|
| `fn` body | Yes | Parameters | `fn f(x: i64) { ... }` |
| `if` body | Yes | None (but pattern-binding forms possible in future) | `if cond { ... }` |
| `else` body | Yes | Independent from `if` scope | `else { ... }` |
| `match` arm | Yes, per arm | Pattern bindings from the arm's pattern | `Circle{r} => ...` |
| `for` body | Yes | Loop variable(s) from the pattern | `for x in list { ... }` |
| `while` body | Yes | None | `while cond { ... }` |
| `loop` body | Yes | None | `loop { ... }` |
| `scope` body | Yes | None (structured concurrency container) | `scope { ... }` |
| `with` body | Yes | Resource binding | `with f := open(path)? { ... }` |
| `catch` body | Yes | Error binding | `catch \|err\| { ... }` |
| `select` arm | Yes, per arm | Message binding from the `from` clause | `msg from ch => ...` |
| Block expression | Yes | Any `:=` bindings inside | `result := { ... }` |
| Closure body | Yes | Closure parameters + captured names | `fn(x) => ...` |
| `entry` body | Yes | Local bindings in the entry block | `entry { ... }` |
| `test` body | Yes | Local bindings in the test block | `test foo { ... }` |

Bindings created inside a block are **destroyed** (go out of scope) when the block exits. Resource cleanup (`with`, `defer`) is also triggered at block exit.

---

## 4. Name Resolution Algorithm

When the compiler encounters an identifier, it resolves it using the following algorithm:

```
resolve(name, current_scope):
  1. Search current_scope's local bindings (innermost block)
  2. If not found, move to the parent scope and repeat
     (repeat until module scope is reached)
  3. At module scope, search the import scope (names from `use` declarations)
  4. If not found in imports, search the package scope (pub(pkg) items from sibling modules)
  5. If not found in package scope, search the universe scope (built-ins)
  6. If not found anywhere → compile error: "undefined identifier: `name`"
```

### Conflict Resolution

**Import conflicts:** If two imports bring in the same name, it is a compile error. The programmer must use qualified paths or aliases:

```aria
use net.http           // brings in `http`
use mylib.http         // ERROR: `http` already imported from net
use mylib.http as mhttp  // OK: alias resolves the conflict
```

**Shadowing vs. mutation:** These are distinct operations:

| Operation | Syntax | What Happens |
|-----------|--------|--------------|
| Shadowing | `x := 6` (when `x` is already bound) | Creates a NEW binding; old `x` is inaccessible |
| Mutation | `x = 6` (when `x` is `mut`) | UPDATES the existing binding; same variable |

```aria
x := 5
x := 6      // shadowing — new binding

mut y := 5
y = 6       // mutation — same binding, new value
```

**Forward references:** Top-level declarations (functions, types, constants, traits) CAN reference each other regardless of declaration order in the file. This is resolved during a pre-pass before type-checking:

```aria
mod main

entry { greet("Aria") }    // OK — greet is defined below

fn greet(name: str) = println("Hello, {name}")
```

Inside function bodies, bindings MUST be declared before use:

```aria
fn f() {
    println(x)    // ERROR: x is not yet declared
    x := 5
}
```

**Self-reference in types:** Recursive types are permitted. The compiler resolves the type name from module scope:

```aria
type List[T] =
    | Cons { head: T, tail: List[T] }
    | Nil
```

**`self` in methods:** `self` is implicitly available in the body of any method inside an `impl` block. It refers to the receiver instance:

```aria
impl Greet for User {
    fn greet(self) = println("Hello, {self.name}")
}
```

`Self` (capital S) refers to the implementing type itself (useful in return types and constructors):

```aria
trait Builder {
    fn new() -> Self
}
```

**Qualified access:** Dotted paths in expressions resolve left-to-right through module and field namespaces:

```aria
std.fs.read("file.txt")    // `std` is package, `fs` is module, `read` is function
user.address.city          // `user` is variable, `address` is field, `city` is field
```

---

## 5. Closure Capture Rules

Closures capture variables from their enclosing lexical scope.

### Default: Capture by Immutable Reference

Closures capture variables by **immutable reference** by default. This means:
- The closure can read the captured variable
- The closure cannot modify the captured variable
- The captured variable must outlive the closure (compiler-enforced)

```aria
counter := 0
f := fn() => counter + 1    // OK: reads counter by immutable reference
g := fn() => { counter += 1 }  // ERROR: cannot mutably capture `counter`
```

### What Gets Captured

Only variables actually **referenced** in the closure body are captured. The compiler performs escape analysis to determine the capture set:

```aria
x := 1
y := 2
z := 3
f := fn() => x + y    // captures x and y; z is NOT captured
```

### Lifetime Rule

A closure cannot outlive the variables it captures (unless using `move`):

```aria
fn make_adder(n: i64) -> fn(i64) -> i64 {
    fn(x) => x + n    // ERROR: n lives on the stack of make_adder; closure would outlive n
}

// Fix with move:
fn make_adder(n: i64) -> fn(i64) -> i64 {
    move fn(x) => x + n    // OK: n is moved into the closure
}
```

### `move` Closures

`move fn(params) => body` forces the closure to **take ownership** of all captured variables:

```aria
data := expensive_compute()
handle := spawn move fn() => process(data)   // data is moved into the spawned task
// data is no longer accessible here — it was moved
```

> **Design Decision:** `move` is required for closures passed to `spawn`. This is enforced by the compiler. A non-`move` closure cannot be spawned because the spawned task could outlive the capturing scope.

### Mutable Capture

Direct mutable capture is not allowed. For mutation across closure boundaries, use:
1. Channels (preferred for concurrent mutation)
2. `sync.Mutex` for shared mutable state
3. Return the mutated value from the closure and rebind it

```aria
// Preferred: channels
ch := chan[i64](buffer: 1)
spawn fn() => ch.send(compute())
result := ch.recv()

// Preferred: return + rebind
counter := 0
counter := increment(counter)    // pure function, returns new value
```

---

## 6. Scope and Concurrency

### `scope` Blocks (Structured Concurrency)

A `scope { }` block is a structured concurrency container. All tasks spawned inside a scope are guaranteed to complete before the scope exits:

```aria
scope {
    a := spawn fetchUsers()
    b := spawn fetchOrders()
}
// Here, both a and b are complete
// a.result and b.result are accessible after the scope
```

**Scoping rules for `scope`:**
- Variables declared inside the scope block are local to it
- `spawn` inside a scope creates a task associated with that scope
- Tasks within a scope CAN access outer scope variables IF they are **immutable**
- Tasks within a scope CANNOT mutate outer scope variables (compiler error)
- The scope waits for all tasks; if any task panics, the scope propagates the panic

```aria
shared := compute_shared_data()    // immutable
scope {
    a := spawn process(shared)     // OK: shared is immutable
    b := spawn analyze(shared)     // OK: shared is immutable
}
```

### `spawn` Outside a `scope`

`spawn` outside a `scope` block creates a **detached task**. The spawned task:
- Runs concurrently with the spawning code
- MUST use `move fn()` — it cannot capture by reference (it might outlive the capturing scope)
- Returns a `Task[T]` handle that can be awaited with `.await()`

```aria
handle := spawn move fn() => long_computation()
// ... do other work ...
result := handle.await()!
```

> **Design Decision:** `spawn` with a non-`move` closure outside a `scope` is a compile error. This prevents use-after-free bugs from tasks outliving their captured variables. Inside a `scope`, the compiler can verify the task does not outlive the scope, so non-`move` closures capturing immutable data are allowed.

---

## 7. `defer` Scoping

`defer` registers a cleanup action to run when the **enclosing block** exits, regardless of how it exits (normal completion, `return`, `break`, error propagation).

```aria
fn process() {
    f := fs.open("file.txt")!
    defer f.close()
    // ... use f ...
}   // f.close() is called here, when the function block exits
```

### Key Rules

1. **Scope:** `defer` executes when the **enclosing block** exits — not necessarily the function. A `defer` inside an `if` body executes when that `if` body's block exits:

```aria
fn f() {
    if condition {
        x := acquire_resource()!
        defer x.release()          // releases when this if-block exits
        use(x)
    }   // x.release() called here
    // ... code after if, x is already released ...
}
```

2. **LIFO order:** Multiple `defer` statements execute in **last-in, first-out** order:

```aria
defer println("third")    // runs 3rd
defer println("second")   // runs 2nd
defer println("first")    // runs 1st
// Block exits → prints: first, second, third
```

3. **Argument evaluation:** `defer` evaluates the **expression** (function arguments) at the point of the `defer` statement, but executes the call at scope exit. This is consistent with Go's defer semantics:

```aria
i := 1
defer println(i)    // captures i = 1 at this point
i = 2
// When scope exits: prints 1, not 2
```

4. **Closures in defer:** To capture the *current* value at the time of execution (lazy evaluation), wrap the call in a closure and immediately invoke it via `defer`:

```aria
mut i := 1
defer fn() { println(i) }()    // defers the closure invocation; closure captures `i` by reference
i = 2
// When scope exits: prints 2 (the closure reads i at the time of execution, not at defer time)
```

> **Design Decision:** `defer fn() { ... }()` is the explicit opt-in for lazy evaluation. The trailing `()` causes the closure to be called (not just created) at scope exit. This pattern is intentional and mirrors Go's idiomatic workaround for the same problem.

5. **Error propagation:** `defer` runs even when a function returns an error via `?` propagation:

```aria
fn risky() -> str ! IoError {
    f := fs.open("file.txt")?
    defer f.close()             // still runs if next line errors
    content := f.read_all()?   // if this errors, defer still runs before propagating
    content
}
```

---

## 8. Pattern Binding Scopes

Pattern matching in `match` expressions introduces bindings local to each arm:

```aria
match shape {
    Circle{radius} => {
        // `radius` is available here
        pi * radius * radius
    }
    Rect{w, h} => {
        // `w` and `h` are available here
        // `radius` is NOT in scope here
        w * h
    }
    Point => 0.0
}
// None of `radius`, `w`, `h` are in scope here
```

### Or-Patterns

In or-patterns (`A | B`), both arms must bind the **same names** with the **same types**:

```aria
match value {
    Circle{radius} | Ellipse{radius} => pi * radius * radius   // OK: both bind `radius: f64`
    Circle{radius} | Point => 0.0                               // ERROR: Point doesn't bind `radius`
}
```

### Pattern Bindings in Let Statements

The `pattern := expr else { ... }` form (irrefutable-or-else binding) introduces bindings into the **current scope** (not a nested scope), available after the binding statement:

```aria
Some(user) := findUser(id) else return None
// `user` is in scope here and below
println(user.name)
```

---

## 9. For Loop Variable Scoping

The loop variable in a `for` loop is:
- Available **only inside the loop body**
- **Re-bound** each iteration (not mutated — each iteration creates a new binding)
- **Immutable** — the loop variable cannot be assigned to

```aria
for item in list {
    // `item` is available here
    // `item` is a fresh immutable binding for each iteration
    println(item)
}
// `item` is NOT in scope here

// This is an error:
for item in list {
    item = transformed(item)    // ERROR: `item` is immutable
    item := transformed(item)   // OK: shadows the loop variable within the body
}
```

### Destructuring in For Loops

Destructured bindings in `for` loop patterns have the same scoping as the loop variable:

```aria
for (key, value) in map.entries() {
    // `key` and `value` are available here only
}

for {name, score, ..} in students {
    // `name` and `score` are available here only
}
```

---

## 10. `with` Resource Scoping

`with` provides RAII-style resource management. The resource binding is:
- Available **only inside the `with` block**
- Cleaned up (drop/close called) when the block exits, whether normally or via error

```aria
with conn := db.connect(url)? {
    // `conn` is available here
    conn.query("SELECT ...")?
}
// `conn` is NOT in scope here; its cleanup has already run
```

The cleanup action is:
- Defined by the resource type implementing the `Disposable` trait (or `Close` convention)
- Called automatically — no explicit `conn.close()` needed
- Called even if the block exits via `?` error propagation or `return`

```aria
fn process_request(url: str) -> Response ! DbError {
    with conn := db.connect(url)? {
        // If this errors, conn.close() is still called
        result := conn.query("SELECT ...")?
        conn.commit()?
        result
    }
    // conn is closed here
}
```

---

## 11. Constant Scoping

Constants follow the same scoping rules as variable bindings, but:
- Must be computable at **compile time** (literal values, arithmetic on other constants, const-fn calls)
- Are **immutable** by definition — `mut const` is a syntax error
- Are **inlined** by the compiler wherever they are used

```aria
// Module-level: visible throughout the module
const MAX_RETRIES = 3
pub const DEFAULT_TIMEOUT: dur = 30s

fn process() {
    // Block-level: visible only within this function
    const BATCH_SIZE = 100

    for i in 0..BATCH_SIZE { ... }
}
// BATCH_SIZE is NOT in scope here
```

Constants in inner scopes shadow constants from outer scopes, following the same shadowing rules as variables.

---

## 12. Import Scoping

Imports are processed before any other name resolution in the module. Imported names are available throughout the entire module.

```aria
use std.fs           // `fs` is available everywhere in this module
use std.{json, http} // `json` and `http` are available
use crypto.sha256 as sha  // `sha` is the alias; `sha256` is NOT a valid name

fn load() -> Config ! IoError | ParseError {
    raw := fs.read("config.json")?    // `fs` from import
    json.parse[Config](raw)?           // `json` from import
}
```

### Import Rules

1. Imports are **module-scoped** — they apply throughout the entire file, regardless of where the `use` declaration appears in the file
2. Imported names **can be shadowed** by local declarations (with a compiler warning, not an error)
3. Import **conflicts** (same name from two different imports) are **compile errors** — use aliases to resolve
4. **No glob imports**: `use std.*` is NOT valid. All imports must be explicit
5. Imports with `as` introduce only the alias; the original path is not a valid name

```aria
use std.fs
use mylib.fs    // ERROR: `fs` already imported; use an alias

use std.fs
use mylib.fs as myfs   // OK: `fs` is std.fs, `myfs` is mylib.fs
```

---

## 13. `impl` Block Scoping

Methods inside an `impl` block have access to:
- `self` — the receiver instance (implicitly bound)
- `Self` — the implementing type
- All module-scope names
- Their own parameters and local bindings

Methods in an `impl` block do **not** share a scope with each other — each method has its own independent function scope.

```aria
impl Cache[K, V] for HashMap[K, V] {
    fn get(self, key: K) -> V? {
        // `self`, `key` are in scope
        // `Self` refers to HashMap[K, V]
        self.inner.get(key)
    }

    fn set(mut self, key: K, value: V) {
        // `self`, `key`, `value` are in scope
        // `get` is NOT automatically in scope — use `self.get(key)` to call it
        self.inner.insert(key, value)
    }
}
```

---

## 14. Complete Example

This example demonstrates the interaction of all scoping rules:

```aria
mod analytics

use std.{io, time}
use std.collections.Map

const REPORT_VERSION = 2

pub fn generateReport(data: [Record], year: i64) -> Report ! AnalyticsError {
    // Function scope: `data` and `year` are parameters

    // Block-level constant (local to this function)
    const MAX_RECORDS = 10_000

    // Shadowing: `data` is filtered and re-bound as the same logical concept
    data := data |> filter(fn(r) => r.year == year)
    data := data |> take(MAX_RECORDS)

    // `with` scope: `timer` is only available inside the with block
    with timer := time.Stopwatch.start() {
        summaries := data
            |> groupBy(.region)
            |> map(fn((region, records)) => {
                // Closure scope: `region` and `records` captured by immutable reference
                // `total` is local to this closure
                total := records |> map(.revenue) |> sum
                RegionSummary{region: region, total: total}
            })
            |> sortBy(.total)
            |> reverse

        elapsed := timer.elapsed()

        Report{
            version:  REPORT_VERSION,    // module-level constant
            year:     year,
            summaries: summaries,
            generated: time.now(),
            elapsed:   elapsed,
        }
    }
    // `timer` is out of scope and cleaned up here
}

test "generateReport filters by year" {
    // Test scope: independent from the function above
    records := [
        Record{year: 2024, region: "North", revenue: 100.0},
        Record{year: 2023, region: "South", revenue: 200.0},
    ]
    report := generateReport(records, 2024)!
    assert report.summaries.len() == 1
    assert report.summaries[0].region == "North"
}
```
