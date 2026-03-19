# Aria Effect System Specification

**A complete specification for effect declarations, purity verification, effect propagation, and the built-in effect catalog in Aria.**

This document formalizes the effect system sketched in [high-level-design.md](../high-level-design.md) (Effects System section) and referenced throughout the spec directory. The effect system is Aria's mechanism for tracking what a function *can do* — not just what it returns, but what side effects it performs. This is the "secret weapon" for AI code generation.

Cross-references:
- [high-level-design.md](../high-level-design.md) — `with [Io, Fs]` syntax, effects as secret weapon
- [ARIA_AI_GUIDE.md](../ARIA_AI_GUIDE.md) — effect tracking quick reference
- [spec/concurrency-design.md](concurrency-design.md) — `Async` effect, pure function restrictions
- [spec/ffi-design.md](ffi-design.md) — `Ffi` effect for C interop
- [spec/closures-capture-semantics.md](closures-capture-semantics.md) — closure effect inference
- [spec/formal-grammar.md](formal-grammar.md) — `effect_clause` production

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Effect Declarations on Functions](#2-effect-declarations-on-functions)
3. [Pure Functions](#3-pure-functions)
4. [Built-in Effects](#4-built-in-effects)
5. [Effect Propagation](#5-effect-propagation)
6. [Effect Stripping](#6-effect-stripping)
7. [Effect Polymorphism](#7-effect-polymorphism)
8. [Closures and Effects](#8-closures-and-effects)
9. [Effects and Concurrency](#9-effects-and-concurrency)
10. [Effect Inference](#10-effect-inference)
11. [Compile-Time Only — Zero Runtime Cost](#11-compile-time-only)
12. [Design Rationale Summary](#12-design-rationale-summary)
13. [Comparison with Other Languages](#13-comparison-with-other-languages)

---

## 1. Design Philosophy

In every mainstream language, functions are black boxes. When the AI calls `processOrder(order)`, it has no idea whether that function reads from disk, writes to a database, sends an HTTP request, or is pure computation. The AI must read the implementation — or the documentation — to find out.

Aria's effect system makes this information part of the function signature:

```
fn processOrder(order: Order) -> Receipt ! OrderError with [Io, Net] {
    // This function does I/O and network calls — it's in the signature
}

fn calculateTotal(items: [Item]) -> f64 {
    // No `with` clause — this function is pure. It cannot do I/O,
    // cannot access the network, cannot spawn tasks.
    items.map(.price).sum()
}
```

**AI rationale:** When generating code, the AI uses effect information to make correct decisions:
- Pure function? Safe to cache, reorder, parallelize, and call from any context.
- Has `Io` effect? Must be called from a context that also has `Io`.
- Has `Async` effect? Involves concurrency — handle with care.

Without effects, the AI must guess. With effects, the AI knows.

---

## 2. Effect Declarations on Functions

Effects are declared with the `with` keyword after the error clause (if present) or return type:

### Functions with effects

```
fn readFile(path: str) -> str ! IoError with [Io, Fs] {
    fs.read(path)?
}

fn fetchUrl(url: str) -> str ! HttpError with [Io, Net] {
    net.get(url)?
}

fn callCLibrary(data: [byte]) -> c.int with [Ffi] {
    c.process(data.ptr(), data.len())
}

fn spawnWorkers(items: [Item]) -> [Result] ! Error with [Async] {
    scope {
        items.map(fn(item) => spawn process(item))
             .map(fn(t) => t.await()?)
    }
}
```

### Functions without effects (pure)

```
fn add(a: i64, b: i64) -> i64 = a + b
fn double(x: f64) -> f64 = x * 2.0
fn formatName(first: str, last: str) -> str = "{first} {last}"
fn calculateArea(r: f64) -> f64 = 3.14159 * r * r
```

No `with` clause means the function is pure. The compiler verifies this — if the function body performs any effectful operation, the compiler requires a `with` clause.

### Multiple effects

```
fn initialize() -> App ! AppError with [Io, Fs, Net, Async] {
    config := readConfig("app.json")?            // Io, Fs
    db := connectDatabase(config.dbUrl)?          // Io, Net
    workers := startWorkers(config.workers)?      // Async
    App{config, db, workers}
}
```

### Grammar

```
effect_clause = "with" "[" IDENT { "," IDENT } "]" ;

fn_decl = [ visibility ] "fn" IDENT [ generic_params ]
          "(" [ param_list ] ")" [ "->" type ] [ error_clause ] [ effect_clause ]
          fn_body ;
```

---

## 3. Pure Functions

A function without a `with` clause is pure. The compiler enforces purity:

### What pure functions can do

- Arithmetic and logic operations
- Call other pure functions
- Create and manipulate local values
- Pattern match and destructure
- Return values

### What pure functions cannot do

- Read or write files (`Io`, `Fs`)
- Make network requests (`Net`)
- Call C functions (`Ffi`)
- Spawn tasks or use channels (`Async`)
- Print to stdout/stderr (`Io`)
- Access global mutable state

```
// ✅ Pure — only arithmetic
fn fibonacci(n: u64) -> u64 = match n {
    0 => 0
    1 => 1
    n => fibonacci(n - 1) + fibonacci(n - 2)
}

// ❌ Compile error — println requires Io effect
fn greet(name: str) {
    println("Hello, {name}")    // error: `println` has effect [Io], but `greet` is pure
}

// ✅ Fix: declare the effect
fn greet(name: str) with [Io] {
    println("Hello, {name}")
}
```

### Benefits of purity

Pure functions enable aggressive compiler optimizations:

| Optimization | Description | Requires purity? |
|---|---|---|
| Memoization | Cache results for repeated calls | ✅ Yes |
| Reordering | Evaluate in any order | ✅ Yes |
| Parallelization | Safe to run in parallel | ✅ Yes |
| Dead code elimination | Remove unused pure calls | ✅ Yes |
| Constant folding | Evaluate at compile time | ✅ Yes |

**AI rationale:** When the AI sees a pure function, it knows with certainty: this function has no side effects, its result depends only on its arguments, and calling it twice with the same arguments produces the same result. This enables the AI to reason about correctness locally — without reading the entire call graph.

---

## 4. Built-in Effects

Aria defines a set of built-in effects that cover the standard categories of side effects:

| Effect | Description | Operations |
|---|---|---|
| `Io` | General I/O operations | `println`, `readLine`, file read/write, stdin/stdout/stderr |
| `Fs` | Filesystem access | `fs.read`, `fs.write`, `fs.delete`, `fs.list`, directory operations |
| `Net` | Network operations | `net.get`, `net.post`, TCP/UDP sockets, DNS |
| `Ffi` | Foreign function calls | Any `extern "C"` function call |
| `Async` | Concurrency primitives | `spawn`, `scope`, `chan.send`, `chan.recv`, `select` |

### Effect hierarchy

Some effects imply others:

- `Fs` implies `Io` — filesystem operations are a subset of I/O
- `Net` implies `Io` — network operations are a subset of I/O

This means a function declared `with [Fs]` can call functions that require `with [Io]`, because `Fs` is a more specific form of `Io`.

```
fn processFiles(dir: str) -> [str] ! IoError with [Fs] {
    files := fs.list(dir)?         // requires Fs
    println("Found {files.len()} files")  // requires Io — satisfied by Fs ⊃ Io
    files.map(fn(f) => fs.read(f)?).collect()
}
```

### User-defined effects

Aria v0.1 does not support user-defined effect types. The built-in effects cover all categories of side effects. User-defined effects may be added in a future version.

---

## 5. Effect Propagation

Calling an effectful function requires the caller to declare the same effects (or a superset):

```
// readFile requires [Io, Fs]
fn readFile(path: str) -> str ! IoError with [Io, Fs] { ... }

// processConfig must declare at least [Io, Fs] because it calls readFile
fn processConfig() -> Config ! ConfigError with [Io, Fs] {
    content := readFile("config.json")?
    parseConfig(content)?
}

// ❌ Compile error — missing effects
fn processConfig() -> Config ! ConfigError {
    content := readFile("config.json")?    // error: readFile requires [Io, Fs]
}
```

### Propagation rules

1. A function that calls a function with effect `E` must also declare effect `E`
2. Effects accumulate — calling functions with `[Io]` and `[Net]` requires `[Io, Net]`
3. Pure functions can be called from any context (they add no effects)
4. The `entry` block implicitly has all effects — it's the program entry point

```
// entry can call any function regardless of effects
entry {
    config := readConfig("app.json")?     // Io, Fs
    data := fetchData(config.url)?        // Io, Net
    result := processData(data)           // pure — no effects
    saveResult(result)?                   // Io, Fs
}
```

**AI rationale:** Effect propagation is mechanical — the AI adds effects to the caller's signature based on what it calls. The compiler verifies correctness. This is simpler than Go's pattern of threading `context.Context` through every function — effects are declared, not passed.

---

## 6. Effect Stripping

A function can "strip" effects from its callers by encapsulating the effectful operation behind a pure interface. This is the pattern for creating safe wrappers.

```
// This function has effects internally but presents a pure interface
// by caching the result
fn getDefaultConfig() -> Config {
    // Uses a compile-time evaluated constant — no runtime I/O
    Config {
        host: "localhost"
        port: 8080
        timeout: 30s
    }
}
```

For runtime effect stripping, use memoization or initialization patterns:

```
// Initialize once, use the pure result
fn createLookupTable() -> Map[str, i64] with [Io, Fs] {
    data := fs.read("lookup.csv")?
    parseLookupTable(data)
}

// Pure function that uses the pre-loaded table
fn lookup(table: Map[str, i64], key: str) -> i64? {
    table.get(key)
}
```

The key pattern: perform effectful work at initialization, then pass the results to pure functions. This separates the "gather data" phase (effectful) from the "process data" phase (pure).

**AI rationale:** Effect stripping is how the AI structures programs for testability and reasoning. By pushing effects to the edges (entry point, initialization) and keeping the core logic pure, the AI generates code that is easier to test, cache, and parallelize.

---

## 7. Effect Polymorphism

Higher-order functions that accept callbacks may be polymorphic over effects — the function's effects depend on the callback's effects:

```
// map is pure if the callback is pure
// map has effect E if the callback has effect E
fn map[T, U](list: [T], f: fn(T) -> U) -> [U] {
    [f(x) for x in list]
}

// Pure callback → pure result
results := map(numbers, fn(x) => x * 2)

// Effectful callback → effectful result
results := map(urls, fn(url) => net.get(url)?)  // inherits [Io, Net]
```

The compiler infers the effects of `map` from the effects of the passed closure. This inference is automatic — no explicit effect parameters are needed.

### Effect polymorphism in practice

```
// sort is pure — the comparison function must be pure
fn sort[T](list: [T], cmp: fn(T, T) -> bool) -> [T] { ... }

// forEach may have effects — depends on the callback
fn forEach[T](list: [T], f: fn(T)) {
    for x in list { f(x) }
}

// Pure usage
sort(numbers, fn(a, b) => a < b)

// Effectful usage
forEach(files, fn(f) => fs.delete(f)?)   // inherits [Io, Fs]
```

---

## 8. Closures and Effects

Closures inherit the effects of their body. A closure that calls an `Io` function has the `Io` effect:

```
// This closure has effect [Io] because println has effect [Io]
logger := fn(msg: str) { println(msg) }

// This closure is pure
doubler := fn(x: i64) -> i64 => x * 2
```

### Closures and effect propagation

When a closure is passed to a higher-order function, the closure's effects propagate to the call site:

```
fn process(items: [str]) with [Io, Fs] {
    // This map call inherits [Io, Fs] from the closure
    results := items.map(fn(path) => fs.read(path)?)
}
```

### Closures in `spawn`

Closures passed to `spawn` carry the `Async` effect implicitly (since `spawn` itself requires `Async`):

```
fn startWorkers(items: [Item]) with [Async, Io] {
    scope {
        for item in items {
            spawn fn() {
                result := process(item)      // process must be compatible
                println("Done: {item}")       // Io effect
            }
        }
    }
}
```

---

## 9. Effects and Concurrency

The `Async` effect marks functions that use concurrency primitives.

### What requires `Async`

| Operation | Requires `Async`? |
|---|---|
| `spawn` | ✅ Yes |
| `scope { ... }` | ✅ Yes |
| `chan.send()` | ✅ Yes |
| `chan.recv()` | ✅ Yes |
| `select { ... }` | ✅ Yes |
| `task.await()` | ✅ Yes |
| `cancel.check()` | ✅ Yes |

### Pure functions cannot use concurrency

```
// ❌ Compile error — spawn requires Async, pure forbids it
fn calculate(x: f64, y: f64) -> f64 {
    spawn doSomething()    // error: spawn has Async effect
    x * x + y * y
}

// ✅ Declare the effect
fn calculateParallel(x: f64, y: f64) -> f64 with [Async] {
    scope {
        a := spawn fn() => x * x
        b := spawn fn() => y * y
        a.await() + b.await()
    }
}
```

### Effect restrictions summary

| Context | Allowed effects |
|---|---|
| Pure function (no `with`) | None |
| Function `with [Io]` | `Io` and pure |
| Function `with [Io, Net]` | `Io`, `Net`, and pure |
| Function `with [Async, Io]` | `Async`, `Io`, and pure |
| `entry { }` block | All effects (implicit) |
| `test { }` block | All effects (implicit) |

---

## 10. Effect Inference

The compiler can infer effects for functions that don't declare them explicitly. However, explicit declarations are preferred for public API surfaces.

### When inference applies

- **Private functions** — the compiler infers effects from the function body
- **Closures** — effects are always inferred
- **Public functions** — explicit `with` declarations are required (compiler verifies them)

### Inference rules

1. A function's inferred effects are the union of all effects in its body
2. A closure's effects are the union of effects in the closure body
3. If a public function is missing a `with` clause and calls effectful functions, it's a compile error
4. If a public function has a `with` clause, the compiler verifies it is sufficient

```
// Private function — effects inferred
fn loadInternal(path: str) -> str ! IoError {
    fs.read(path)?
    // compiler infers: with [Io, Fs]
}

// Public function — effects must be declared
pub fn loadConfig(path: str) -> Config ! ConfigError with [Io, Fs] {
    content := loadInternal(path)?
    parseConfig(content)?
}
```

**AI rationale:** Explicit effect declarations on public functions serve as machine-readable documentation. The AI reads the signature and knows exactly what the function can do — no need to read the implementation. For private functions, inference reduces token count without losing safety.

---

## 11. Compile-Time Only

Effects have **zero runtime cost**. They are purely a compile-time construct:

- No runtime effect handlers
- No effect wrapping or boxing
- No vtable dispatch for effects
- No runtime checks for effect violations
- Effects are erased after type checking — they don't appear in the compiled binary

The effect system is a static analysis tool. It constrains what code can be written where, but imposes no overhead at execution time.

### Comparison with algebraic effects

Koka and OCaml 5 provide *algebraic effects* — runtime handlers that can intercept and resume effectful operations. Aria's effect system is deliberately simpler:

| Feature | Koka / OCaml 5 | Aria |
|---|---|---|
| Effect handlers | Yes (runtime) | No |
| Effect resumption | Yes | No |
| Effect type checking | Yes | Yes |
| Runtime overhead | Yes (handler dispatch) | **None** |
| Complexity | High | **Low** |

Aria trades the power of algebraic effect handlers for simplicity and zero overhead. The effect system is a tracking mechanism, not a control flow mechanism.

**AI rationale:** Zero runtime cost means the AI can add effects to every function signature without worrying about performance. Effects are free — they only add compile-time safety.

---

## 12. Design Rationale Summary

| Decision | Rationale |
|---|---|
| `with [Effects]` syntax | Clear, grep-able, part of the function signature |
| No `with` = pure | Purity is the default; side effects must be declared |
| Built-in effect set | Covers all standard side-effect categories; user-defined effects add complexity for v0.1 |
| Effect propagation | Mechanical — caller must declare callees' effects; compiler verifies |
| `entry` has all effects | Program entry point is inherently impure; no annotations needed |
| Effect inference for private functions | Reduces token count for internal code |
| Explicit effects for public functions | Machine-readable documentation of function capabilities |
| Zero runtime cost | Effects are compile-time only — no performance concern |
| Simpler than algebraic effects | Tracking is enough; handlers add complexity the AI doesn't need |
| Closures inherit effects | Natural — a closure that does I/O has the I/O effect |

---

## 13. Comparison with Other Languages

| Feature | Go | Rust | Java | Haskell | Koka | Aria |
|---|---|---|---|---|---|---|
| Effect tracking | None | None | `throws` (checked exceptions) | Monads (`IO`, `State`) | Algebraic effects | **`with [Effects]`** |
| Purity verification | None | None | None | `IO` monad boundary | Yes | **Yes** |
| Runtime cost | N/A | N/A | Exception handling | Monad overhead | Handler dispatch | **Zero** |
| AI-readability | ❌ (black box) | ❌ (black box) | ⚠️ (`throws` only) | ⚠️ (monad stacks) | ✅ | **✅** |
| Effect propagation | Manual (context) | Manual | Automatic (`throws`) | Automatic (monads) | Automatic | **Automatic** |

### The key insight

In Go and Rust, every function is a black box — the AI must read the implementation to know what it does. In Java, `throws` only tracks exceptions, not I/O or concurrency. In Haskell, monads track effects but at significant complexity cost.

Aria's effect system sits in the sweet spot: **it tells the AI exactly what a function can do, with zero runtime cost and minimal syntax overhead.**

```
// Go — no information about what processOrder does
func processOrder(order Order) (Receipt, error)

// Aria — full information: does I/O, network, is async, can fail
fn processOrder(order: Order) -> Receipt ! OrderError with [Io, Net, Async]
```

The AI reads the Aria signature and knows: this function does I/O, makes network calls, and uses concurrency. In Go, the AI knows nothing until it reads every line of the implementation.

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/concurrency-design.md](concurrency-design.md), [spec/ffi-design.md](ffi-design.md), and [spec/closures-capture-semantics.md](closures-capture-semantics.md).*
