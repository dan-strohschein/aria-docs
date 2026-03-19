# Aria Pattern Matching Specification

**A complete specification for match expressions, pattern types, exhaustiveness checking, and destructuring throughout the language.**

This document formalizes pattern matching as used throughout the Aria language. Pattern matching is not a single feature — it is a pervasive mechanism that appears in `match` expressions, variable bindings, `for` loops, function parameters, `catch` blocks, and error handling. Every use is governed by the same pattern language.

Cross-references:
- [high-level-design.md](../high-level-design.md) — `match` syntax, sum types, `Option`/`Result` matching
- [spec/formal-grammar.md](formal-grammar.md) — pattern productions (section 3.16)
- [spec/error-handling.md](error-handling.md) — `catch` blocks, `match` on `Result`, `Ok`/`Err` patterns
- [spec/language-spec-addendum.md](language-spec-addendum.md) — destructuring in bindings
- [spec/generics-type-parameters.md](generics-type-parameters.md) — matching on generic types (`Option[T]`, `Result[T,E]`)
- [spec/iteration-protocol.md](iteration-protocol.md) — destructuring in `for` loops

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [The `match` Expression](#2-the-match-expression)
3. [Pattern Types](#3-pattern-types)
4. [Guard Clauses](#4-guard-clauses)
5. [Nested Patterns](#5-nested-patterns)
6. [Exhaustiveness Checking](#6-exhaustiveness-checking)
7. [Patterns in Variable Bindings](#7-patterns-in-variable-bindings)
8. [Patterns in `for` Loops](#8-patterns-in-for-loops)
9. [Patterns in Function Parameters](#9-patterns-in-function-parameters)
10. [Patterns in `catch` Blocks](#10-patterns-in-catch-blocks)
11. [Irrefutable vs Refutable Patterns](#11-irrefutable-vs-refutable-patterns)
12. [Performance](#12-performance)
13. [Design Rationale Summary](#13-design-rationale-summary)
14. [Comparison with Other Languages](#14-comparison-with-other-languages)

---

## 1. Design Philosophy

Pattern matching is the mechanism by which Aria's type system delivers on its promise. Sum types declare what a value *can be*; pattern matching forces the programmer (or AI) to handle every case. Together, they eliminate the most common source of runtime errors in every other language: forgotten cases.

### Why exhaustive matching matters for AI

When the AI generates a `match` on a sum type with 5 variants and forgets one, the compiler rejects the code immediately. In Go, a missing case in a `switch` is a silent bug that only shows up at runtime. In Java, a missing case in an `if-else` chain is invisible.

```
// The compiler rejects this — missing Point
match shape {
    Circle(r) => pi * r * r
    Rect(w, h) => w * h
    Triangle(a, b, c) => herons(a, b, c)
    // ❌ compile error: non-exhaustive match — missing variant: Point
}
```

**AI rationale:** Exhaustive matching converts a runtime bug (forgotten case) into a compile-time error. The AI can mechanically generate all arms from the type definition, and the compiler verifies completeness. This eliminates an entire category of AI-generated bugs.

---

## 2. The `match` Expression

`match` is an expression — it evaluates to a value. Every arm must produce a value of the same type (or a compatible type through type inference).

### Basic syntax

```
result := match value {
    Pattern1 => expression1
    Pattern2 => expression2
    Pattern3 => expression3
}
```

### Match on sum types

```
type Shape =
    | Circle(f64)
    | Rect(f64, f64)
    | Point

fn area(s: Shape) -> f64 = match s {
    Circle(r) => 3.14159 * r * r
    Rect(w, h) => w * h
    Point => 0.0
}
```

### Match on enums

```
type Season = Spring | Summer | Autumn | Winter

fn describe(s: Season) -> str = match s {
    Spring => "flowers bloom"
    Summer => "sun shines"
    Autumn => "leaves fall"
    Winter => "snow falls"
}
```

### Match with block bodies

```
result := match command {
    Move { x, y } => {
        validate_coordinates(x, y)?
        execute_move(x, y)
    }
    Print { message } => {
        log.info("printing: {message}")
        print(message)
    }
    Quit => {
        cleanup()
        exit(0)
    }
}
```

### Match arms

Arms are separated by newlines. Commas between arms are optional:

```
// Both are valid:
match x {
    1 => "one"
    2 => "two"
    _ => "other"
}

match x {
    1 => "one",
    2 => "two",
    _ => "other",
}
```

### Grammar

```
match_expr = "match" expression "{" { match_arm } "}" ;
match_arm  = pattern [ "if" expression ] "=>" expression [ "," ] ;
```

---

## 3. Pattern Types

Aria supports the following pattern types, all of which can be nested and combined.

### 3.1 Wildcard Pattern (`_`)

Matches any value, binds nothing:

```
match value {
    Some(x) => use(x)
    _       => println("no value")
}
```

### 3.2 Binding Pattern

Matches any value and binds it to a name:

```
match value {
    x => println("got: {x}")    // x is bound to the matched value
}
```

Binding with `mut`:

```
match value {
    mut x => { x += 1; x }
}
```

### 3.3 Literal Pattern

Matches a specific literal value:

```
match count {
    0 => "none"
    1 => "one"
    2 => "two"
    n => "{n} items"
}
```

Supported literal types: integers, floats, strings, booleans, duration literals.

### 3.4 Variant Pattern

Matches a sum type variant, binding the associated data:

```
match shape {
    Circle(radius)      => use(radius)
    Rect(w, h)          => use(w, h)
    Point               => println("point")
}
```

Qualified variant names for disambiguation:

```
match error {
    IoError.NotFound{path}           => println("not found: {path}")
    IoError.Timeout{after}           => println("timeout: {after}")
    IoError.PermissionDenied{path, user} => deny(user, path)
}
```

### 3.5 Struct Destructuring Pattern

Matches struct fields by name:

```
match user {
    User { name, email, .. } => println("{name} <{email}>")
}

// With field renaming
match point {
    Point { x: px, y: py } => println("({px}, {py})")
}
```

`..` ignores remaining fields (rest pattern in struct context).

### 3.6 Tuple Pattern

Matches tuple elements by position:

```
match pair {
    (0, 0) => "origin"
    (x, 0) => "on x-axis at {x}"
    (0, y) => "on y-axis at {y}"
    (x, y) => "({x}, {y})"
}
```

### 3.7 Array/Slice Pattern

Matches array elements:

```
match items {
    []              => "empty"
    [x]             => "single: {x}"
    [first, second] => "pair: {first}, {second}"
    [first, ..rest] => "first: {first}, {rest.len()} more"
}
```

`..rest` binds the remaining elements to `rest`. `..` without a name ignores them.

### 3.8 Or-Pattern (`|`)

Matches if any of the alternatives match:

```
match season {
    Spring | Summer => "warm"
    Autumn | Winter => "cold"
}

match code {
    200 | 201 | 204 => "success"
    301 | 302       => "redirect"
    404             => "not found"
    _               => "other"
}
```

Both sides of `|` must bind the same variable names with the same types:

```
match shape {
    Circle(r) | Rect(r, _) => println("first dimension: {r}")
    // ✅ both sides bind `r` as f64
}
```

### 3.9 Named Pattern (`name @`)

Binds the entire matched value while also destructuring:

```
match shape {
    s @ Circle(r) if r > 10.0 => {
        log.info("large circle: {s}")
        r * r * pi
    }
    Circle(r) => r * r * pi
    _ => 0.0
}
```

### 3.10 Rest Pattern (`..`)

Matches zero or more remaining elements in arrays or fields in structs:

```
// In arrays
match items {
    [first, ..] => use(first)       // ignore rest
    [first, ..rest] => use(rest)    // bind rest
}

// In structs
match user {
    User { name, .. } => use(name)  // ignore other fields
}
```

---

## 4. Guard Clauses

A guard clause adds a boolean condition to a match arm. The arm matches only if both the pattern and the guard are satisfied:

```
fn describe(shape: Shape) -> str = match shape {
    Circle(r) if r > 10.0 => "large circle"
    Circle(r) if r > 1.0  => "medium circle"
    Circle(r)              => "small circle"
    Rect(w, h) if w == h   => "square ({w}×{h})"
    Rect(w, h)             => "rectangle ({w}×{h})"
    Point                  => "point"
}
```

### Guard expressions

The guard expression has access to all variables bound by the pattern:

```
match user {
    User { name, age, .. } if age >= 18 => "adult: {name}"
    User { name, age, .. } if age >= 13 => "teen: {name}"
    User { name, .. }                    => "child: {name}"
}
```

### Guards and exhaustiveness

Guards do not count toward exhaustiveness checking. A guarded arm is not guaranteed to match, so the compiler requires a fallback:

```
// ❌ compile error — guards don't guarantee exhaustiveness
match x {
    n if n > 0 => "positive"
    n if n < 0 => "negative"
    // missing: n == 0
}

// ✅ correct — unguarded arm covers the rest
match x {
    n if n > 0 => "positive"
    n if n < 0 => "negative"
    _          => "zero"
}
```

---

## 5. Nested Patterns

Patterns can be nested to arbitrary depth:

```
type Color =
    | Rgb(u8, u8, u8)
    | Named(str)
    | Transparent

type Stroke =
    | Solid(Color)
    | Dashed(Color, f64)
    | None

fn describe(stroke: Stroke) -> str = match stroke {
    Solid(Rgb(r, g, b))      => "solid rgb({r},{g},{b})"
    Solid(Named(name))       => "solid {name}"
    Solid(Transparent)       => "solid transparent"
    Solid(_)                 => "solid (other)"
    Dashed(Named(name), len) => "dashed {name} ({len}px)"
    Dashed(_, len)           => "dashed ({len}px)"
    None                     => "no stroke"
}
```

### Nested Option/Result matching

```
match findUser(id) {
    Some(User { name, email, .. }) => println("{name}: {email}")
    None                           => println("user not found")
}

match readFile("config.json") {
    Ok(content)                 => process(content)
    Err(IoError.NotFound{path}) => println("missing: {path}")
    Err(e)                      => println("error: {e}")
}
```

**AI rationale:** Nested patterns let the AI destructure complex data in a single match arm instead of chaining multiple matches. One `match` with nested patterns replaces what would be 3-4 nested `if` statements in Go — fewer tokens, fewer nesting bugs.

---

## 6. Exhaustiveness Checking

The compiler verifies that every `match` expression covers all possible values of the matched type.

### Sum types must be fully covered

```
type Shape = Circle(f64) | Rect(f64, f64) | Point

// ❌ compile error
match shape {
    Circle(r) => r * r * pi
    Rect(w, h) => w * h
    // missing: Point
}

// ✅ correct
match shape {
    Circle(r) => r * r * pi
    Rect(w, h) => w * h
    Point => 0.0
}
```

### Wildcard satisfies remaining cases

```
match shape {
    Circle(r) => r * r * pi
    _ => 0.0                  // covers Rect and Point
}
```

### Boolean exhaustiveness

```
match flag {
    true  => "yes"
    false => "no"
}
```

### Numeric and string types

Numeric and string types have infinite variants, so a wildcard or binding pattern is always required:

```
match count {
    0 => "none"
    1 => "one"
    n => "{n} items"     // required — covers all other values
}
```

### Compiler error messages

The compiler provides specific guidance when a match is non-exhaustive:

```
error: non-exhaustive match
  --> src/shapes.aria:15:5
   |
15 | match shape {
   |       ^^^^^ pattern `Point` not covered
   |
   = help: add a match arm for the missing variant:
           Point => /* expression */
```

---

## 7. Patterns in Variable Bindings

Patterns can appear on the left side of `:=` for destructuring:

### Tuple destructuring

```
(x, y) := getCoordinates()
(first, _, third) := getTriple()
(name, age, email) := getUserInfo()
```

### Struct destructuring

```
User { name, email, .. } := getUser()
Point { x, y } := origin
Config { host, port, .. } := loadConfig()?
```

### Refutable patterns in bindings

For patterns that might not match, use `else`:

```
Some(user) := findUser(id) else {
    return Err(NotFoundError{id: id})
}

Ok(data) := parseJson(input) else |err| {
    log.warn("parse failed: {err}")
    return defaultConfig
}
```

The `else` branch must diverge (return, break, continue, or panic) because the binding variable is not available in the else path.

---

## 8. Patterns in `for` Loops

Destructuring patterns work in `for` loop headers:

```
// Tuple destructuring (map iteration)
for (key, value) in config.entries() {
    println("{key} = {value}")
}

// Struct destructuring
for User { name, score, .. } in students {
    if score >= 90 { println("{name}: honors") }
}

// Enumerate with destructuring
for (index, User { name, email, .. }) in users.enumerate() {
    println("{index}: {name} <{email}>")
}

// Array destructuring
for [first, second, ..] in pairs {
    println("{first} -> {second}")
}
```

---

## 9. Patterns in Function Parameters

Function parameters can use destructuring patterns:

```
fn distance((x1, y1): (f64, f64), (x2, y2): (f64, f64)) -> f64 {
    ((x2 - x1).pow(2) + (y2 - y1).pow(2)).sqrt()
}

fn greet(User { name, .. }: User) {
    println("Hello, {name}!")
}
```

---

## 10. Patterns in `catch` Blocks

`catch` blocks use the same pattern syntax as `match` arms to handle error variants:

```
result := fetchUser(id) catch {
    UserError.NotFound{..}  => defaultUser
    UserError.Timeout{..}   => {
        log.warn("timeout fetching user {id}")
        retry(3, fn() => fetchUser(id)) catch { _ => defaultUser }
    }
    e => return Err(AppError.User{source: e})
}
```

Typed catch with single error:

```
content := readFile("config.json") catch |err| {
    log.warn("config not found: {err}")
    yield "{}"
}
```

See [spec/error-handling.md](error-handling.md) for the full catch specification.

---

## 11. Irrefutable vs Refutable Patterns

**Irrefutable patterns** always match — they can appear in `let` bindings and function parameters without an `else` clause:

- Binding patterns: `x`
- Wildcard: `_`
- Tuple destructuring of known-size tuples: `(a, b)`
- Struct destructuring: `Point { x, y }`

**Refutable patterns** may not match — they require `match`, `if let`, or an `else` clause:

- Variant patterns: `Some(x)`, `Ok(v)`, `Circle(r)`
- Literal patterns: `0`, `"hello"`, `true`
- Guard clauses: `x if x > 0`

```
// Irrefutable — always succeeds
(x, y) := getPoint()                      // ✅ tuple always has two elements

// Refutable — might not match, needs else
Some(user) := findUser(id) else return None  // ✅ handles None case
```

---

## 12. Performance

The compiler optimizes pattern matching for performance:

### Jump tables

When matching on integer or enum values with contiguous variants, the compiler generates a jump table — O(1) dispatch:

```
match color {
    Red   => 0xff0000
    Green => 0x00ff00
    Blue  => 0x0000ff
}
// Compiled as a jump table — no sequential comparisons
```

### Decision trees

For complex patterns with nested destructuring, the compiler generates a decision tree that minimizes the number of comparisons:

```
match shape {
    Circle(r) if r > 10.0 => ...
    Circle(r)              => ...
    Rect(w, h) if w == h   => ...
    Rect(w, h)             => ...
    Point                  => ...
}
// Decision tree: first check variant tag, then check guard
```

### No performance penalty

Pattern matching compiles to the same code that hand-written `if`/`else` chains would produce — there is no runtime overhead for the abstraction.

**AI rationale:** The AI can generate complex pattern matches without worrying about performance. The compiler turns them into optimal decision trees. The AI should prefer `match` over `if`/`else` chains for sum types — it's both more correct (exhaustive checking) and equally fast.

---

## 13. Design Rationale Summary

| Decision | Rationale |
|---|---|
| `match` is an expression | Eliminates mutable temporaries; every match produces a value |
| Exhaustive checking | Compile-time guarantee that no case is forgotten |
| No fallthrough | Each arm is independent — no accidental fallthrough bugs |
| Guards separate from patterns | Patterns match structure; guards filter by value — orthogonal concerns |
| Nested patterns | Deep destructuring in one expression instead of nested `if` chains |
| Patterns in bindings/loops | Destructuring everywhere reduces accessor boilerplate (`user.name` → `name`) |
| Or-patterns (`\|`) | Common cases grouped in one arm — fewer arms, fewer tokens |
| Named patterns (`name @`) | Bind the whole and destructure at the same time — no redundant binding |
| `..` rest pattern | Ignore what you don't need without listing every field |

---

## 14. Comparison with Other Languages

| Feature | Go | Rust | Java (21+) | Aria |
|---|---|---|---|---|
| Match as expression | No (`switch` is statement) | Yes | Partial (switch expr) | **Yes** |
| Exhaustive checking | No | Yes | Partial (sealed classes) | **Yes** |
| Nested patterns | No | Yes | Partial | **Yes** |
| Guard clauses | No | Yes (`if`) | Yes (`when`) | **Yes** |
| Destructuring in match | No | Yes | Partial (records) | **Yes** |
| Destructuring in bindings | No | Yes (`let`) | Partial (records) | **Yes** |
| Or-patterns | No | Yes (`\|`) | No | **Yes** |
| Rest patterns (`..`) | No | Yes (`..`) | No | **Yes** |
| Fallthrough | Yes (explicit) | No | No (break-based) | **No** |

### Token cost: matching a 4-variant sum type

| Language | Approach | Tokens |
|---|---|---|
| Go | `switch` + type assertion | ~25 |
| Rust | `match` | ~15 |
| Java | `switch` + pattern matching (21+) | ~20 |
| Aria | `match` | **~12** |

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/formal-grammar.md](formal-grammar.md), [spec/error-handling.md](error-handling.md), and [spec/language-spec-addendum.md](language-spec-addendum.md).*
