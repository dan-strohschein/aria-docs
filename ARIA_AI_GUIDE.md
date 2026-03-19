# Aria Language — AI Code Generation Guide

> This file is designed to be consumed by AI coding assistants (GitHub Copilot, Cursor, Cline, etc.).
> Include this file in your AI context to enable correct Aria code generation.
> For the full language specification, see the [spec/](spec/) directory.

---

## Quick Facts

- **File extension:** `.aria`
- **Paradigm:** Expression-oriented procedural with functional composition and trait-based polymorphism ("Procedural bones, functional blood, no inheritance")
- **Entry point:** `entry { }` block in the main module
- **Module declaration:** `mod name`
- **No semicolons, no parentheses around conditions**
- **Indentation:** not significant (block-based with `{ }`)
- **Comments:** `//` line comments

---

## Type System Quick Reference

**Primitives:** `i8 i16 i32 i64 u8 u16 u32 u64 f32 f64 str bool byte`

- Default int: `i64`, default float: `f64`
- No implicit conversions ever
- No null — use `Option[T]` (sugar: `T?`)
- Result type: `Result[T, E]` (sugar: `T!E`)

**Composite types:**
```
// Sum types (enums)
type Shape = Circle(f64) | Rect(f64, f64) | Point

// Structs
struct Name { field: Type, field2: Type = default }

// Tuple types
(i64, str)

// Collections
[T]        // Array/list
{K: V}     // Map
{T}        // Set
```

**Traits:**
```
trait Name { fn method(self) -> Type }
```

**Generics:**
```
fn name[T: Trait](x: T) -> T
```

---

## Variable Declaration

```
x := value           // immutable (inferred type)
x: Type = value      // immutable (explicit type)
mut x := value       // mutable
const X = value      // constant
```

**No `let`, no `var`, no `val`.**

---

## Functions

```
fn name(param: Type, param2: Type) -> ReturnType {
    body
}

// Single-expression shorthand
fn add(a: i64, b: i64) -> i64 = a + b

// Methods (via trait implementations)
impl TraitName for TypeName {
    fn method(self) -> ReturnType { body }
}

// Inherent methods (no trait required)
impl TypeName {
    fn method(self) -> ReturnType { body }
}

// Closures / lambdas
double := fn(x: i64) -> i64 = x * 2
sorter := fn(a: str, b: str) -> bool = a < b
```

**No `func`, `function`, or `def` — it's always `fn`.**

---

## Control Flow

```
// If — expression-based, no parens around condition
result := if condition { value1 } else { value2 }

if x > 0 {
    println("positive")
} else if x < 0 {
    println("negative")
} else {
    println("zero")
}

// Match — exhaustive, expression-based
match value {
    Pattern1 => result1
    Pattern2(x) => result2
    _ => default
}

// For loop — collections
for item in collection { }

// For loop — range (exclusive end)
for i in 0..10 { }

// For loop — inclusive range
for i in 0..=10 { }

// While loop
while condition { }

// Infinite loop with break
loop {
    if done { break }
}
```

---

## Error Handling

- `?` operator propagates errors (adds automatic context)
- `catch` block handles errors inline
- `!` asserts success (panics on error — use only when failure is a bug)
- No try/catch, no exceptions

**Function signature with error return:**
```
fn read_config(path: str) -> Config!IoError {
    content := fs.read(path)?           // propagate on error
    parsed := json.parse[Config](content)?
    parsed
}
```

**Handling errors with `catch`:**
```
result := catch read_config("app.toml") {
    IoError.NotFound => default_config()
    err => { log.error(err); exit(1) }
}
```

**Custom error types:**
```
type AppError =
    | NotFound(str)
    | InvalidInput(str)
    | Internal(str)
```

---

## Concurrency

```
// Spawn a task (lightweight, like goroutines)
handle := spawn expensive_work(data)
result := handle.await()

// Structured concurrency — waits for all tasks
scope {
    spawn task1()
    spawn task2()
}

// Channels
ch := chan[i64]()          // unbuffered
ch := chan[i64](100)       // buffered (capacity 100)
ch.send(42)
value := ch.recv()

// Select over multiple channels
select {
    value := <-ch1 => handle_value(value)
    value := <-ch2 => handle_other(value)
    timeout(1s)   => handle_timeout()
}
```

---

## Pipeline Operator

Left-to-right transformation chains. `x |> f` desugars to `f(x)`. `x |> f(a, b)` desugars to `f(x, a, b)`.

```
result := data
    |> filter(is_valid)
    |> map(transform)
    |> sort_by(.score)
    |> take(10)

// With error propagation
processed := raw_input
    |> parse?
    |> validate?
    |> transform?
```

---

## Destructuring

```
// Tuple destructuring
(x, y, z) := get_coordinates()

// Struct destructuring
Point { x, y } := point
User { name, email } := user

// Ignore fields
(first, _, third) := triple

// List destructuring
[first, second, ..rest] := list

// In for loops
for (key, value) in map { }
for Point { x, y } in points { }

// In match arms
match result {
    Ok(User { name, email }) => println("Hello {name}")
    Err(e) => println("Error: {e}")
}
```

---

## String Interpolation

```
greeting := "Hello, {name}! You are {age} years old."
debug_msg := "Value: {x * 2 + 1}"    // expressions work
```

---

## Inline Tests

```
fn add(a: i64, b: i64) -> i64 = a + b

test "addition works" {
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
}

test "handles edge cases" {
    result := catch risky_fn() {
        MyError.Expected => "handled"
        err => panic("unexpected: {err}")
    }
    assert result == "handled"
}
```

---

## Imports

```
use std.fs
use std.{json, http}
use my_package.module_name

// Import specific symbols
use std.math.{sqrt, PI}
```

---

## Visibility

- Everything is module-private by default
- `pub` makes items visible outside the module
- `pub(pkg)` makes items visible within the package only

```
pub struct User { ... }
pub fn create_user(...) -> User { ... }
pub(pkg) fn internal_helper(...) { ... }
fn private_fn(...) { ... }     // private (default)
```

---

## Resource Management

```
// Automatic cleanup with `with`
with file := fs.open("data.txt")? {
    content := file.read_all()?
    process(content)
}  // file automatically closed here

// Multiple resources
with conn := db.connect(url)?,
     tx   := conn.begin()? {
    tx.exec("INSERT ...")?
    tx.commit()?
}
```

---

## Structs — Construction and Update

```
struct Point { x: f64 = 0.0, y: f64 = 0.0 }
struct User  { name: str, email: str, active: bool = true }

// Construction
p := Point { x: 1.0, y: 2.0 }
u := User { name: "Alice", email: "alice@example.com" }

// Update syntax (immutable copy with changes)
p2 := p.{ x: 5.0 }           // p2 has x=5.0, y=2.0
u2 := u.{ active: false }
```

---

## Traits and Implementations

```
trait Display {
    fn display(self) -> str
}

trait Validate {
    fn validate(self) -> bool!ValidationError
}

struct Email { value: str }

impl Display for Email {
    fn display(self) -> str = self.value
}

impl Validate for Email {
    fn validate(self) -> bool!ValidationError {
        if !self.value.contains("@") {
            return err(ValidationError("missing @"))
        }
        true
    }
}

// Generic function with trait bound
fn show[T: Display](item: T) {
    println(item.display())
}

// Multiple trait bounds
fn process[T: Display + Validate](item: T) -> bool!ValidationError {
    item.validate()?
    println("Valid: {item.display()}")
    true
}
```

---

## Effect Tracking

Functions that perform I/O, mutate shared state, or call C code declare effects:

```
fn read_file(path: str) -> str with [Io] { ... }
fn call_c_lib() with [Ffi] { ... }
fn pure_fn(x: i64) -> i64 = x * 2    // no effects declared = pure
```

Pure functions can be optimized more aggressively and called safely in concurrent contexts.

---

## Common Patterns — DO vs DON'T

**DO:**
```
// Use expression-based if
status := if connected { "online" } else { "offline" }

// Use ? for error propagation
data := fs.read(path)?

// Use match exhaustively
match shape {
    Circle(r) => pi * r * r
    Rect(w, h) => w * h
    Point => 0.0
}

// Use pipeline for transformations
result := items |> filter(valid) |> map(transform) |> collect()

// Use Option[T] instead of null
fn find_user(id: i64) -> User? {
    if id > 0 { Some(users[id]) } else { None }
}
```

**DON'T:**
```
// ❌ No null — use Option[T]
// ❌ No exceptions — use Result[T,E] and ?
// ❌ No inheritance — use traits and composition
// ❌ No implicit conversions — be explicit
// ❌ No semicolons
// ❌ No parentheses around if/while/for conditions
// ❌ No `func`, `function`, `def` — it's `fn`
// ❌ No `let`, `var`, `val` — it's `:=` or `mut x :=`
// ❌ No `class` keyword — use `struct` + `impl`
```

---

## Derives (Auto-Implemented Traits)

```
@[derive(Eq, Hash, Debug, Json)]
struct UserId { value: i64 }

@[derive(Eq, Debug, Json, Validate)]
struct User {
    id:    UserId
    name:  str
    email: str
}
```

Available derives: `Eq`, `Hash`, `Debug`, `Json`, `Validate`, `Clone`, `Default`

---

## Complete Minimal Example

```
mod main

use std.{fs, json, log}

struct Config {
    host: str = "localhost"
    port: i64 = 8080
    debug: bool = false
}

fn load_config(path: str) -> Config!IoError {
    content := fs.read(path)?
    json.parse[Config](content)?
}

entry {
    config := catch load_config("app.toml") {
        IoError.NotFound => Config {}    // use defaults
        err => {
            log.error("Failed to load config: {err}")
            exit(1)
        }
    }

    log.info("Starting on {config.host}:{config.port}")
}
```
