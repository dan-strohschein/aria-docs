# Aria Pattern Matching Specification

**A complete, implementable specification for pattern matching in Aria — the primary control flow mechanism for branching on structured data.**

This document formalizes and extends the pattern matching sketches scattered across the existing specs into a single exhaustive reference. Every design decision is justified through the lens of AI code generation — the primary consumer of this language.

Cross-references:
- [high-level-design.md](../high-level-design.md) — `match`, sum types, `Option`, `Result` sketches
- [spec/formal-grammar.md](formal-grammar.md) — §3.14 (match expressions), §3.16 (patterns)
- [spec/scoping-rules.md](scoping-rules.md) — §8 (pattern binding scopes)
- [spec/language-spec-addendum.md](language-spec-addendum.md) — destructuring in bindings and loops
- [spec/error-handling.md](error-handling.md) — `catch` arms, typed error matching
- [spec/concurrency-design.md](concurrency-design.md) — `select` arms, channel patterns
- [spec/numeric-overflow.md](numeric-overflow.md) — range patterns for integer types
- [spec/iteration-protocol.md](iteration-protocol.md) — `for` loop desugaring via `match`
- [spec/datetime-design.md](datetime-design.md) — dot-shorthand for enum variants
- [spec/compiler-architecture.md](compiler-architecture.md) — Stage 4 exhaustiveness checking

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Pattern Taxonomy](#2-pattern-taxonomy)
3. [Match Expressions](#3-match-expressions)
4. [Exhaustiveness Checking](#4-exhaustiveness-checking)
5. [Pattern Matching in Bindings](#5-pattern-matching-in-bindings)
6. [Pattern Matching in `for` Loops](#6-pattern-matching-in-for-loops)
7. [Pattern Matching in `catch` and Error Handling](#7-pattern-matching-in-catch-and-error-handling)
8. [Pattern Matching in `select` (Concurrency)](#8-pattern-matching-in-select-concurrency)
9. [Guards (`if` Clauses)](#9-guards-if-clauses)
10. [Nested and Composed Patterns](#10-nested-and-composed-patterns)
11. [Interaction with Other Language Features](#11-interaction-with-other-language-features)
12. [Compiler Implementation Notes](#12-compiler-implementation-notes)
13. [Design Rationale Summary](#13-design-rationale-summary)
14. [Token Cost Comparison](#14-token-cost-comparison)

---

## 1. Design Philosophy

Pattern matching is the **primary control flow mechanism** for branching on structured data in Aria. Every other language offers some form of conditional branching — `if/else`, `switch`, `cond` — but pattern matching in Aria is distinguished by three properties that make it uniquely suited to AI code generation.

### Exhaustive by Default

Every `match` expression is **exhaustive**: the compiler guarantees that every possible value of the matched type is covered. A `match` on a sum type that omits a variant is a compile error — not a runtime behavior difference, not a warning, a hard error.

```go
type Color = | Red | Green | Blue

// COMPILE ERROR: match is not exhaustive — Blue is not handled
fn colorName(c: Color) -> str = match c {
    Red   => "red"
    Green => "green"
    // Blue is missing — this is an error, not a default fall-through
}
```

This is the single most important property for AI code generation: the compiler catches missed cases, not the user at runtime. When an AI generates a match on a sum type, it cannot silently miss a variant. When a developer adds a new variant to a sum type, every existing match expression that doesn't cover it fails to compile — providing exhaustive, surgical notification of every place that needs updating.

**Comparison with other languages:**

| Language | Exhaustiveness on enums | Mechanism |
|---|---|---|
| Go | ❌ No | `switch` silently falls through or to `default` |
| Rust | ✅ Yes | `match` — same guarantee as Aria |
| Java (pre-21) | ❌ No | `switch` on enums is not exhaustively checked |
| Java 21+ | ⚠️ Partial | Pattern `switch` on sealed types — recent addition |
| Python | ❌ No | `match` is not exhaustive |
| Swift | ✅ Yes | `switch` on enums — same guarantee |
| Kotlin | ⚠️ Partial | `when` is exhaustive only as an expression |
| **Aria** | ✅ **Yes, always** | Every `match` on a sum type — compile error if incomplete |

### Expression-Oriented

Every `match` in Aria is an **expression** — it produces a value. There is no statement-only match form. This eliminates the need for mutable temporaries and enables compositional pipelines:

```go
// No temporary variable needed — match IS the value
label := match status {
    .Active  => "🟢 Active"
    .Pending => "🟡 Pending"
    .Disabled => "🔴 Disabled"
}

// match inline in a function call
sendEmail(subject: match priority {
    .High   => "[URGENT] {title}"
    .Normal => title
    .Low    => "[FYI] {title}"
})
```

### Token-Efficient

Aria's match syntax eliminates every redundant token:
- No `case` keyword before each arm
- No `break` to prevent fallthrough
- No `:` after each arm pattern
- No parentheses around the matched value
- Commas between arms are optional (newlines suffice)
- Dot-shorthand for enum variants when the type is inferred from context

**Why this matters for AI:** Every token that a model must generate is an opportunity for an error. Removing structural tokens that carry no semantic content reduces the bug surface for generated code.

---

## 2. Pattern Taxonomy

Aria supports the following pattern kinds. Each can appear in `match` arms, let bindings, `for` loop variables, function parameters, and `catch` arms.

### 2.1 Wildcard Pattern (`_`)

Matches any value and binds nothing. Used when the value is irrelevant.

```go
match result {
    Ok(value) => process(value)
    Err(_)    => log.warn("operation failed")   // don't need the error details
}

// As a catch-all in a non-exhaustive position
match code {
    200 => handleOk()
    404 => handleNotFound()
    _   => handleOther()    // required — integers are not exhaustively enumerable
}
```

**Type**: The wildcard pattern is compatible with any type. It contributes no bindings to the arm's scope.

### 2.2 Binding Pattern (`name`, `mut name`)

Matches any value and binds it to the given name. The binding is immutable by default; prefix with `mut` for a mutable binding.

```go
match event {
    Click{x, y} => handleClick(x, y)
    KeyPress{key} => handleKey(key)
    other => log.debug("unhandled event: {other}")  // `other` binds the whole event
}

// Mutable binding
match getCounter() {
    mut n => {
        n += 1
        update(n)
    }
}
```

**Type**: A binding pattern is compatible with any type `T`. The binding `name` has type `T` (immutable) or `mut T` (mutable).

**Shadowing**: A binding pattern at the top level of a `match` arm shadows any outer binding of the same name within that arm's scope only.

### 2.3 Literal Pattern

Matches a specific compile-time constant value. Supported for integers, floats, booleans, characters, string literals, and duration literals.

```go
// Integer literals
match port {
    80  => serveHttp()
    443 => serveHttps()
    _   => serveCustom(port)
}

// Boolean literals
match isValid {
    true  => proceed()
    false => reject("invalid input")
}

// String literals
match command {
    "quit"   => exit()
    "help"   => showHelp()
    "status" => showStatus()
    other    => unknownCommand(other)
}

// Duration literals
match retryDelay {
    0s  => retryImmediately()
    1s  => scheduleShort()
    _   => scheduleLong(retryDelay)
}
```

**Type rule**: The literal must be assignable to the type of the matched expression. A string literal pattern on an `i64` value is a compile error.

**Note on floats**: Float literal patterns are supported syntactically but produce a compiler warning — float equality comparison is almost never the right tool. Prefer range patterns or guard expressions for float matching.

### 2.4 Range Pattern

Matches a value within an inclusive or exclusive range. Supported for integer types and character types.

```go
// Inclusive range (..=)
match statusCode {
    200..=299 => handleSuccess()
    300..=399 => handleRedirect()
    400..=499 => handleClientError()
    500..=599 => handleServerError()
    _         => handleUnknown(statusCode)
}

// Character ranges
match c {
    'a'..='z' => lowercase(c)
    'A'..='Z' => uppercase(c)
    '0'..='9' => digit(c)
    _         => other(c)
}

// Half-open range (..)
match age {
    0..18   => minor()
    18..65  => adult()
    65..    => senior()    // 65 and above (open upper bound)
}
```

**Grammar**:
```
range_pattern = expression ".." [ "=" ] expression   (* lower..upper or lower..=upper *)
              | expression ".."                       (* lower.. open upper bound *)
```

**Type rule**: Both endpoints must be the same integer or character type. The half-open form `lower..` matches all values ≥ lower.

**Exhaustiveness with ranges**: Integer ranges contribute to exhaustiveness checking. The compiler tracks which values are covered and requires either full coverage or a wildcard/catch-all arm for integers.

### 2.5 Struct Pattern

Matches a struct value by destructuring its fields. Fields may be bound by name (shorthand) or mapped to a sub-pattern.

```go
type User = {
    name: str
    age: u32
    email: str
    active: bool
}

// Field shorthand — binds each named field
match user {
    {name, age, ..} => "{name} (age {age})"
}

// Field with sub-pattern
match user {
    {name, age: 0..=17, ..} => minorUser(name)
    {name, active: true, ..} => activeUser(name)
    {name, ..}               => inactiveUser(name)
}

// Fully specified (no rest pattern — all fields must be listed)
match point {
    {x: 0, y: 0} => origin()
    {x, y}       => point(x, y)
}
```

**Grammar**:
```
struct_pattern     = IDENT "{" field_pattern_list "}"
field_pattern_list = field_pattern { "," field_pattern } [ "," ]
field_pattern      = IDENT                (* bind field name directly *)
                   | IDENT ":" pattern    (* bind field with sub-pattern *)
                   | ".."                 (* rest — ignore remaining fields *)
```

**Rest pattern (`..`)**: When `..` appears in a struct pattern, it matches any remaining fields not explicitly listed. Without `..`, all fields must be listed; omitting any is a compile error.

**Type rule**: The pattern's field names must exist on the matched struct type. The `..` may appear at most once per struct pattern.

### 2.6 Variant Pattern

Matches a sum type variant with associated data. Variants with tuple-style data use `Variant(pattern)` syntax; variants with named fields use `Variant{field}` syntax (same as struct pattern).

```go
type Shape =
    | Circle { radius: f64 }
    | Rect   { w: f64, h: f64 }
    | Point

// Named-field variants (struct syntax)
fn area(s: Shape) -> f64 = match s {
    Circle{radius}  => 3.14159 * radius * radius
    Rect{w, h}      => w * h
    Point           => 0.0
}

// Tuple-style variants
type Result[T, E] =
    | Ok(T)
    | Err(E)

match result {
    Ok(value) => process(value)
    Err(e)    => handleError(e)
}

// Nested variant pattern
match response {
    Ok({status: 200, body, ..}) => processBody(body)
    Ok({status: 404, ..})       => handleNotFound()
    Ok({status, ..})            => handleOther(status)
    Err(e)                      => handleError(e)
}
```

**Grammar**:
```
variant_pattern = IDENT "(" pattern_list ")"    (* tuple-style *)
                | IDENT "{" field_pattern_list "}" (* named-field style *)
                | IDENT                            (* unit variant — no data *)
```

### 2.7 Qualified Variant Pattern

When the variant name is ambiguous (e.g., multiple sum types have a `None` variant), qualify the variant with the type name.

```go
match value {
    Option.Some(x)  => process(x)
    Option.None     => default()
}

match err {
    IoError.NotFound{path}         => log.warn("missing: {path}")
    IoError.PermissionDenied{path} => deny(path)
}
```

In most contexts, the type can be inferred and the qualification is optional. The compiler requires qualification only when the variant name is otherwise ambiguous.

### 2.8 Dot-Shorthand for Enum Variants

When the enum type is unambiguously inferred from context, variants may be written with a leading dot (`.Variant`) instead of the full qualified name.

```go
// The compiler infers the type of `status` from context — no need to write `Status.Active`
match status {
    .Active  => "active"
    .Pending => "pending"
    .Closed  => "closed"
}

// In calendar applications (from datetime-design.md)
match dt.dayOfWeek {
    .Monday | .Wednesday | .Friday => "MWF schedule"
    .Tuesday | .Thursday           => "TTh schedule"
    _                              => "weekend"
}

// In let bindings
.Active := currentStatus else return
```

**Type inference rule**: Dot-shorthand is valid when the type of the matched expression is a known sum type with a variant of that name. If the name is ambiguous across multiple sum types in scope, the compiler requires explicit qualification.

### 2.9 Tuple Pattern

Matches a tuple by position, binding each element to a sub-pattern.

```go
// Destructure a 2-tuple (Point)
match getCoordinates() {
    (0, 0) => origin()
    (x, 0) => onXAxis(x)
    (0, y) => onYAxis(y)
    (x, y) => generic(x, y)
}

// Nested tuples
match nested {
    ((a, b), c) => compute(a, b, c)
    _           => default()
}

// Ignoring elements
match triple {
    (first, _, last) => (first, last)
}
```

**Grammar**:
```
tuple_pattern = "(" pattern_list ")"
pattern_list  = pattern { "," pattern } [ "," ]
```

**Type rule**: The number of sub-patterns must exactly match the arity of the tuple type.

### 2.10 Array/List Pattern

Matches a list or array by position and optional tail. The rest pattern `..rest` captures the remaining elements as a slice.

```go
// Fixed-length match
match rgb {
    [r, g, b] => Color{r: r, g: g, b: b}
    _         => Color.black()
}

// Head and tail decomposition
match list {
    []            => empty()
    [only]        => singleton(only)
    [first, ..rest] => processHead(first, rest)
}

// First two elements with tail
match args {
    []                 => printUsage()
    [cmd]              => execute(cmd, [])
    [cmd, ..args]      => execute(cmd, args)
}

// Exact match with trailing ignore
match packet {
    [0xFF, 0xFE, ..] => unicode()
    [b, ..]          => ascii(b)
}
```

**Grammar**:
```
array_pattern   = "[" array_pat_list "]"
array_pat_list  = pattern { "," pattern } [ "," ] [ ".." [ IDENT ] ]
```

**Type rule**: The array/list pattern is compatible with any list, array, or slice type whose element type is compatible with the sub-patterns. The rest capture `..rest` binds a slice of the remaining elements.

**Exhaustiveness**: Array patterns on dynamically-sized lists require a wildcard or catch-all arm, since the compiler cannot enumerate all lengths. Array patterns on fixed-size arrays are exhaustible.

### 2.11 Or-Pattern

Matches if any of the alternatives match. All alternatives must bind exactly the **same names** with the **same types**.

```go
// Simple value alternatives
match ch {
    'a' | 'e' | 'i' | 'o' | 'u' => vowel(ch)
    _                            => consonant(ch)
}

// Variant alternatives (both must bind `radius: f64`)
match shape {
    Circle{radius} | Ellipse{radius} => pi * radius * radius
    Rect{w, h}                       => w * h
    Point                            => 0.0
}

// Enum variant shorthand
match status {
    .Active | .Pending => true
    .Closed | .Banned  => false
}
```

**Binding consistency rule**: In an or-pattern `A | B`, both `A` and `B` must introduce the same set of binding names, and each corresponding name must have the same type. Violating this is a compile error:

```go
// COMPILE ERROR: Point doesn't bind `radius`
match shape {
    Circle{radius} | Point => pi * radius * radius
}
```

**Grammar**:
```
or_pattern = pattern "|" pattern
```

Or-patterns associate left: `A | B | C` parses as `(A | B) | C`, which is semantically equivalent to a flat three-way or.

### 2.12 Rest Pattern

Matches zero or more remaining elements in a struct, array, or tuple. The named form `..rest` binds the collected remainder; the unnamed form `..` discards it.

```go
// In struct patterns — ignore remaining fields
match user {
    {name, ..} => greet(name)
}

// In array patterns — capture tail
match tokens {
    [head, ..tail] => process(head, tail)
    []             => empty()
}

// In tuple patterns — ignore trailing elements
match record {
    (id, name, ..) => (id, name)
}
```

**Note**: In struct patterns, `..` may appear only once and at the end of the field list. In array patterns, `..rest` may appear only once and must be the last element.

### 2.13 Guard Pattern (`pattern if condition`)

Adds a runtime condition to a match arm. The structural pattern is checked first; if it matches, the guard expression is evaluated. If the guard returns `false`, the arm is skipped and matching continues.

See [Section 9 — Guards](#9-guards-if-clauses) for full semantics and examples.

### 2.14 Negated Patterns

Negated patterns (matching "anything except X") are **not supported** as a distinct pattern kind. Use a guard expression for this instead:

```go
// There is no `!pattern` syntax. Use a guard:
match value {
    x if x != 0 => nonZero(x)
    _           => zero()
}
```

**Rationale**: Negated patterns complicate exhaustiveness checking and increase the cognitive load of reading match arms. Guards express the same intent more explicitly.

---

## 3. Match Expressions

### 3.1 Syntax

```
match_expr = "match" expression "{" { match_arm } "}"
match_arm  = pattern [ "if" expression ] "=>" expression_or_block [ "," ]
```

Arms are separated by newlines (preferred) or commas (optional). The matched expression is evaluated once; its value is then tested against each arm's pattern in order from top to bottom. The first matching arm (whose pattern matches **and** whose guard, if present, evaluates to `true`) is selected.

```go
match value {
    Pattern1 if guard1 => body1
    Pattern2           => body2
    Pattern3 if guard3 => body3
    _                  => default
}
```

### 3.2 Type Rules

A `match` expression used as a value requires that all arms produce values of the same type. The type of the `match` expression is the common type of all arm bodies.

```go
// All arms produce str — type of `label` is str
label := match status {
    .Active  => "active"
    .Pending => "pending"
    .Closed  => "closed"
}

// COMPILE ERROR: arms have mismatched types (str vs i64)
x := match flag {
    true  => "yes"
    false => 42        // type mismatch
}
```

When `match` is used as a statement (its return value is discarded, e.g., each arm produces `()`), type uniformity is not required.

### 3.3 Arm Evaluation Order

Arms are tested **top-to-bottom**. The first arm whose pattern matches (and whose guard, if any, returns `true`) is executed. Subsequent arms are not tested.

```go
match n {
    0     => "zero"
    1..=9 => "single digit"    // only reached if n != 0
    _     => "large"
}
```

### 3.4 Match as Statement vs. Expression

`match` can be used as either a statement or an expression:

```go
// As expression — produces a value
area := match shape {
    Circle{radius} => pi * radius * radius
    Rect{w, h}     => w * h
    Point          => 0.0
}

// As statement — side effects only, value discarded
match event {
    Login{user}  => audit.log("login: {user}")
    Logout{user} => audit.log("logout: {user}")
    _            => ()
}
```

When used as a statement, the expression is valid even if arms have different types, since the value is not used.

### 3.5 Nested Match Expressions

Match expressions can be nested. Each nested match is independently exhaustive.

```go
match outer {
    A(inner) => match inner {
        X => "A-X"
        Y => "A-Y"
    }
    B => "B"
}
```

Deep nesting is a readability warning. Prefer extracting inner matches into helper functions.

### 3.6 Side Effects in Match Arms

Match arm bodies are regular expressions; they may contain any expression, including those with side effects. Guards are evaluated for their boolean value only; side effects in guards are permitted but strongly discouraged — guards are evaluated in pattern-matching order and a guard that fires is not guaranteed to be the final answer (another arm may still match after a failed guard). See [Section 9](#9-guards-if-clauses) for details.

---

## 4. Exhaustiveness Checking

Exhaustiveness checking is performed at **Stage 4 (Type Checking)** of the Aria compiler pipeline (see [spec/compiler-architecture.md](compiler-architecture.md)). Every `match` expression must cover every possible value of the matched type. The compiler errors — never warns — on non-exhaustive matches.

### 4.1 Exhaustiveness for Sum Types

A `match` on a sum type is exhaustive if and only if every variant of the sum type is covered by at least one arm (accounting for guards — see §4.6).

```go
type Shape = | Circle{radius: f64} | Rect{w: f64, h: f64} | Point

// Exhaustive — all three variants covered
match shape {
    Circle{radius} => ...
    Rect{w, h}     => ...
    Point          => ...
}

// NOT exhaustive — Rect and Point are missing
// COMPILE ERROR: missing variants: Rect, Point
match shape {
    Circle{radius} => ...
}
```

**Compiler error format:**

```
error[E0004]: non-exhaustive match
  --> src/geometry.aria:12:5
   |
12 |     match shape {
   |     ^^^^^ patterns `Rect{..}` and `Point` not covered
   |
   = help: add the following arms:
             Rect{w, h} => todo()
             Point      => todo()
```

### 4.2 Exhaustiveness for Booleans

A `match` on `bool` is exhaustive when both `true` and `false` are covered.

```go
// Exhaustive
match flag {
    true  => enable()
    false => disable()
}

// Also exhaustive — wildcard covers the remaining case
match flag {
    true => enable()
    _    => ()
}
```

### 4.3 Exhaustiveness for `Option[T]`

`Option[T]` is a sum type with variants `Some(T)` and `None`. Both must be covered.

```go
match findUser(id) {
    Some(user) => greet(user)
    None       => createUser(id)
}
```

### 4.4 Exhaustiveness for `Result[T, E]`

`Result[T, E]` is a sum type with variants `Ok(T)` and `Err(E)`. Both must be covered.

```go
match readFile("config.toml") {
    Ok(config)  => startServer(config)
    Err(e)      => log.error("failed to read config: {e}")
}
```

### 4.5 Exhaustiveness for Integer Types

Integer types (`i8`, `i16`, `i32`, `i64`, `u8`, etc.) have a finite but very large value space. A match on an integer is exhaustive only when:
1. Every possible value is covered by literal or range patterns, **or**
2. A wildcard (`_`) or catch-all binding is present.

In practice, matches on integers almost always require a wildcard:

```go
// Exhaustive — wildcard covers values not listed
match port {
    80  => Http
    443 => Https
    _   => Custom(port)
}

// Also exhaustive via ranges covering the full type range
match b: u8 {
    0..=127 => ascii(b)
    128..=255 => extended(b)
}
```

The compiler tracks range coverage and can determine when the union of all arms covers the complete value space, making a wildcard unnecessary for range-complete matches.

### 4.6 Exhaustiveness for String Types

String values are infinite in number. A match on `str` always requires a wildcard or catch-all binding:

```go
match command {
    "start"  => startService()
    "stop"   => stopService()
    "status" => showStatus()
    other    => unknownCommand(other)    // required
}
```

### 4.7 Exhaustiveness for Nested Sum Types

The compiler checks exhaustiveness through nested patterns. For a nested sum type, the Cartesian product of all variants must be covered (with wildcards allowed at any level):

```go
type Outer = | A(Inner) | B
type Inner = | X | Y

// Exhaustive — all combinations covered
match outer {
    A(X) => ...
    A(Y) => ...
    B    => ...
}

// Also exhaustive — wildcard at inner level
match outer {
    A(inner) => match inner {
        X => ...
        Y => ...
    }
    B => ...
}
```

### 4.8 Exhaustiveness and Error Unions

Error union types (`IoError | ParseError | DbError`) behave as a flat sum type for exhaustiveness purposes. All variants across all error types must be covered:

```go
type AppError = IoError | ParseError | DbError

match err {
    IoError.NotFound{path}         => ...
    IoError.PermissionDenied{..}   => ...
    IoError.Timeout{..}            => ...
    ParseError.InvalidJson{line}   => ...
    ParseError.UnexpectedToken{..} => ...
    DbError.ConnectionFailed{..}   => ...
    DbError.Timeout{..}            => ...
}
```

Alternatively, use a wildcard to group unhandled cases:

```go
match err {
    IoError.NotFound{path} => recoverMissing(path)
    _                      => propagate(err)
}
```

### 4.9 Exhaustiveness and Or-Patterns

An or-pattern `A | B` counts as covering both variant `A` and variant `B` for exhaustiveness purposes.

```go
// Exhaustive — Red and Green covered by first arm; Blue by second
match color {
    .Red | .Green => warm()
    .Blue         => cool()
}
```

### 4.10 Guards Do Not Affect Exhaustiveness

A guarded arm (`pattern if condition => ...`) does **not** count as covering that pattern for exhaustiveness. The compiler assumes a guard might evaluate to `false` and requires the remaining cases to be covered by unguarded arms:

```go
// COMPILE ERROR: the guarded arm does not exhaust `Some(x)`
// The compiler cannot prove the guard is always true
match opt {
    Some(x) if x > 0 => positive(x)    // guard — does not count for exhaustiveness
    None             => nothing()
}
// ERROR: `Some(x)` where `x <= 0` is not handled

// Correct:
match opt {
    Some(x) if x > 0 => positive(x)
    Some(x)          => nonPositive(x)    // unguarded catch for the remaining Some cases
    None             => nothing()
}
```

### 4.11 Adding Variants — The Compiler as Change Guardian

When a new variant is added to a sum type, **every `match` expression in the entire codebase** that matches on that type and does not use a wildcard will fail to compile. This is intentional — it is the compiler acting as a mechanical code review, pointing at exactly the places that the developer needs to update.

```go
// Before: Color = | Red | Green | Blue
// After:  Color = | Red | Green | Blue | Yellow   ← new variant added

// All of these now fail to compile until Yellow is handled:
fn colorName(c: Color) -> str = match c { Red => ... Green => ... Blue => ... }
fn colorCode(c: Color) -> u32 = match c { Red => ... Green => ... Blue => ... }
fn isWarm(c: Color) -> bool   = match c { .Red | .Green => true  .Blue => false }
```

**AI rationale:** Adding a type variant in Aria creates a compile-time checklist of every place in the codebase that needs updating. An AI that adds a new variant can immediately run the compiler to discover all the places that require follow-up edits — this is a primitive but powerful form of AI-assisted refactoring.

---

## 5. Pattern Matching in Bindings

### 5.1 Irrefutable Patterns

A pattern is **irrefutable** if it is guaranteed to match for any value of the relevant type. Irrefutable patterns can appear in plain `:=` bindings without an `else` clause.

```go
// Tuple destructuring — always matches
(x, y) := getCoordinates()

// Struct destructuring — always matches
{name, email, ..} := getUser()

// Nested tuple/struct
((lat, lng), altitude) := getPosition()
```

**Irrefutable pattern kinds:**
- Wildcard (`_`)
- Binding (`name`, `mut name`)
- Tuple patterns where all sub-patterns are irrefutable
- Struct patterns with `..` where all explicitly listed fields have irrefutable sub-patterns
- Array patterns of a fixed-size array where all elements are irrefutable

**Refutable pattern kinds** (require `else` or a `match`):
- Literal patterns
- Range patterns
- Variant patterns (where the type has more than one variant)
- Array patterns on variable-length lists
- Or-patterns

### 5.2 Refutable Patterns with `else`

A refutable pattern in a binding requires an `else` branch that must **diverge** (return, break, continue, or panic). This guarantees that all bindings introduced by the pattern are valid after the binding statement.

```go
// Option binding — `user` is in scope after this line
Some(user) := findUser(id) else return None

// Result binding — `data` is in scope if parsing succeeds
Ok(data) := parseJson(input) else |err| {
    log.warn("parse failed: {err}")
    return defaultConfig
}

// Enum variant binding
Circle{radius} := shape else {
    log.warn("expected Circle, got {shape}")
    return 0.0
}
```

**The `|err|` syntax** in the `else` clause captures the non-matching value. In the `Ok(data)` example, `err` is bound to the `Err(e)` value for use in the else body. The binding name is chosen by the developer.

**Scope rule**: Bindings from a successful pattern match are introduced into the **current scope**, not a new nested scope. They are available from the point of the binding statement to the end of the enclosing block.

### 5.3 Destructuring in Function Parameters

Function parameters support irrefutable pattern destructuring:

```go
// Tuple parameter
fn distance((x1, y1): (f64, f64), (x2, y2): (f64, f64)) -> f64 {
    sqrt((x2 - x1).pow(2) + (y2 - y1).pow(2))
}

// Struct parameter (with field shorthand)
fn greet({name, age, ..}: User) -> str {
    "Hello, {name} (age {age})"
}

// Named parameter with destructuring
fn render(config: {width: u32, height: u32, ..}) -> Image {
    Image.new(config.width, config.height)
}
```

Only irrefutable patterns are valid in function parameters. Refutable patterns in parameters are a compile error.

---

## 6. Pattern Matching in `for` Loops

The `for` loop binds a pattern variable for each iteration. The pattern follows the same rules as let-binding patterns.

### 6.1 Simple Binding

```go
for item in collection {
    process(item)
}
```

### 6.2 Tuple Destructuring

```go
// Iterating over a map (yields (key, value) pairs)
for (key, value) in config.entries() {
    println("{key} = {value}")
}

// With index (enumerate yields (index, item) pairs)
for (i, item) in list.enumerate() {
    println("{i}: {item}")
}
```

### 6.3 Struct Destructuring

```go
for {name, score, ..} in students {
    if score >= 90 { println("{name}: honors") }
}
```

### 6.4 Nested Destructuring

```go
for (index, {name, email, ..}) in users.enumerate() {
    println("{index}: {name} <{email}>")
}

// Nested tuples
for ((lat, lng), label) in waypoints {
    renderMarker(lat, lng, label)
}
```

### 6.5 How `for` Desugars to `match`

The `for` loop is syntactic sugar over the `Iterator` protocol (see [spec/iteration-protocol.md](iteration-protocol.md)). The desugaring makes the match explicit:

```go
// Source
for x in collection {
    process(x)
}

// Desugars to:
{
    _iter := collection.iter()
    loop {
        match _iter.next() {
            Some(x) => process(x)
            None    => break
        }
    }
}
```

The pattern in the `for` loop header (here `x`) becomes the sub-pattern inside `Some(...)` in the desugared match. Tuple and struct destructuring work because the sub-patterns are composed naturally:

```go
// Source
for (k, v) in map.entries() { ... }

// Desugars to:
loop {
    match _iter.next() {
        Some((k, v)) => { ... }
        None         => break
    }
}
```

### 6.6 Where Clauses

A `for` loop with a `where` clause is equivalent to filtering before iteration — only items where the where expression is `true` are processed:

```go
for user in users where user.active {
    sendNotification(user)
}
```

The `where` clause is not a pattern — it's a boolean expression that can access the loop variable's bindings after destructuring.

---

## 7. Pattern Matching in `catch` and Error Handling

Error handling in Aria integrates with pattern matching at multiple levels. See [spec/error-handling.md](error-handling.md) for the complete error handling specification.

### 7.1 `catch` Arms

A `catch` block uses match arm syntax to handle error variants:

```go
fn processConfig(path: str) -> Config ! IoError | ParseError {
    content := readFile(path) catch |e| match e {
        IoError.NotFound{path}         => return Err(IoError.NotFound{path})
        IoError.PermissionDenied{..}   => panic("no permission to read config")
        _                              => return Err(e)
    }
    parseJson(content)?
}
```

Typed catch arms automatically have their exhaustiveness checked against the error type(s) being caught.

### 7.2 Inline `catch` Expression

```go
// catch with typed arms — exhaustive over `IoError`
result := readFile(path) catch |e| {
    IoError.NotFound{..}       => defaultContent
    IoError.PermissionDenied{..} => panic("permission denied")
    IoError.Timeout{after}     => retryAfter(after)
}
```

### 7.3 Pattern Matching on Error Categories via Guards

Error category traits (e.g., `Retryable`, `Transient`) can be tested in guards:

```go
match err {
    e if e is Retryable  => retry(op, attempts: 3)
    e if e is Permanent  => propagate(e)
    e                    => log.error("unexpected: {e}")
}
```

The `is` operator tests whether a value implements a given trait. See [Section 9.3](#93-trait-test-guards) for full semantics.

### 7.4 Match on `Result` Values

Matching on `Result` directly follows the standard variant pattern syntax:

```go
match readFile("config.toml") {
    Ok(config)                      => startServer(config)
    Err(IoError.NotFound{path})     => log.warn("config missing: {path}")
    Err(IoError.PermissionDenied{..}) => panic("cannot read config")
    Err(e)                          => return Err(e)
}
```

---

## 8. Pattern Matching in `select` (Concurrency)

`select` uses pattern-like arms for concurrent channel operations. See [spec/concurrency-design.md](concurrency-design.md) for the full concurrency specification.

### 8.1 Receive Arms

The `pattern from channel` syntax receives a value from a channel and binds it to the pattern:

```go
select {
    msg from requests  => handleRequest(msg)
    msg from responses => handleResponse(msg)
}
```

The pattern in a receive arm supports the same destructuring as any other pattern:

```go
select {
    {id, payload, ..} from requests  => process(id, payload)
    (code, body)      from responses => respond(code, body)
}
```

### 8.2 Timeout Arms

The `after <duration>` arm fires if no other arm becomes ready within the specified duration:

```go
select {
    msg from ch   => process(msg)
    after 5s      => timeout()
}
```

### 8.3 Default Arm

A `default` arm fires immediately if no other arm is ready (non-blocking select):

```go
select {
    msg from ch => process(msg)
    default     => doOtherWork()
}
```

### 8.4 Select as Expression

`select` is an expression. All arms must return the same type:

```go
result := select {
    val from resultCh => Ok(val)
    err from errorCh  => Err(err)
    after 30s         => Err(TimeoutError{})
}
```

### 8.5 Exhaustiveness in `select`

`select` does not require exhaustiveness in the same way as `match` — it always fires exactly one arm (the first that becomes ready). However:
- If a `default` arm is present, the select is non-blocking and always completes.
- If no `default` arm is present, the select blocks until at least one arm is ready.
- The compiler warns if all channel expressions are already-closed or otherwise unreachable.

---

## 9. Guards (`if` Clauses)

### 9.1 Syntax and Semantics

A guard adds a boolean condition to a match arm. The pattern must match first; then the guard is evaluated. If the guard is `false`, the arm is skipped.

```go
match n {
    x if x < 0  => negative(x)
    x if x == 0 => zero()
    x           => positive(x)   // unguarded catch-all
}
```

Guards have access to all bindings introduced by the pattern in their arm:

```go
match user {
    {name, age, ..} if age >= 18 => adult(name)
    {name, age, ..}              => minor(name, age)
}
```

### 9.2 Guards and Exhaustiveness

Guards do **not** contribute to exhaustiveness coverage. The compiler treats a guarded arm as potentially non-matching even when the pattern is irrefutable. This prevents a false sense of security from partial guards:

```go
// This is NOT exhaustive — the compiler requires another arm
match n {
    x if x >= 0 => positive(x)     // guard — might be false
    x if x < 0  => negative(x)     // guard — might be false
    // COMPILE ERROR: even though x >= 0 and x < 0 cover all cases,
    // the compiler cannot prove this at compile time.
    // A wildcard is required:
    _ => unreachable()    // or use a static assertion
}

// Correct — unguarded wildcard ensures exhaustiveness
match n {
    x if x > 0 => positive(x)
    x if x < 0 => negative(x)
    _          => zero()      // handles the x == 0 case (and satisfies exhaustiveness)
}
```

**Rationale**: Requiring the developer to acknowledge the remaining case (even if it is logically unreachable) makes the intent explicit and prevents subtle bugs when the guard conditions are later modified.

### 9.3 Trait-Test Guards

The `is` operator tests whether a value implements a trait. Combined with a guard, this enables dynamic dispatch on trait membership:

```go
match err {
    e if e is Retryable => {
        delay := e.retryDelay()    // method from the Retryable trait
        scheduleRetry(op, delay)
    }
    e if e is Permanent => log.error("permanent failure: {e}")
    e                   => propagate(e)
}
```

`e is Trait` evaluates to `bool`. When `true`, the binding `e` within the arm body is treated as implementing `Trait`, enabling trait method calls without explicit casting.

### 9.4 Guard Side Effects

Guards are evaluated as part of pattern matching, which tests arms in order. A guard with side effects may be evaluated multiple times if it appears in a context where the pattern could match but the guard might fail and a later arm also matches.

**Best practice**: Guards should be pure expressions. Side-effecting guards are permitted by the compiler but produce a warning:

```
warning[W0012]: guard expression has side effects
  --> src/main.aria:15:21
   |
15 |     x if fetchValue(x) > 0 => ...
   |          ^^^^^^^^^^^^^ consider using a local binding before the match
```

---

## 10. Nested and Composed Patterns

### 10.1 Arbitrary Nesting

Patterns can be nested to any depth. The compiler checks exhaustiveness and type-correctness recursively:

```go
type Response =
    | Ok { status: u16, body: str, headers: [Header] }
    | Err { code: u16, message: str }

match response {
    Ok{status: 200, body, ..}   => processBody(body)
    Ok{status: 201, body, ..}   => created(body)
    Ok{status: 404, ..}         => notFound()
    Ok{status: 400..=499, ..}   => clientError(response)
    Ok{status: 500..=599, ..}   => serverError(response)
    Ok{status, ..}              => unexpected(status)
    Err{code: 0, message}       => networkError(message)
    Err{code, message}          => appError(code, message)
}
```

### 10.2 Nested Struct in Variant

```go
match packet {
    Ok({status: 200, body: {data, metadata, ..}, ..}) => {
        processData(data, metadata)
    }
    Ok({status, ..}) => handleStatus(status)
    Err(e)           => handleError(e)
}
```

### 10.3 Array with Nested Struct

```go
match events {
    []                         => noEvents()
    [{kind: .Click, x, y}, ..rest] => {
        handleClick(x, y)
        processRemaining(rest)
    }
    [{kind, ..}, ..rest] => {
        log.debug("unhandled first event: {kind}")
        processRemaining(rest)
    }
}
```

### 10.4 Or-Pattern with Nested Structure

```go
match shape {
    Circle{radius} | Ellipse{radius} => area = pi * radius * radius
    Rect{w, h}                       => area = w * h
    Triangle{w, h}                   => area = w * h / 2.0
    Point                            => area = 0.0
}
```

### 10.5 Deeply Nested Exhaustiveness

The compiler checks exhaustiveness through all levels of nesting. Each nested match is independently exhaustive. In a flat match with nested patterns, the compiler verifies that the union of all nested patterns at each level is exhaustive:

```go
// The compiler ensures:
// 1. All `Response` variants are covered (Ok and Err)
// 2. Within Ok, all relevant status ranges are covered (via wildcard at the end)
// 3. Type safety at each sub-pattern
```

---

## 11. Interaction with Other Language Features

### 11.1 Pipeline Operator (`|>`)

Pattern matching integrates naturally with the pipeline operator. The common pattern is `match` as the final stage of a pipeline:

```go
result := fetchData(url)
    |> parseJson
    |> validate
    |> match {
        Ok(data)  => render(data)
        Err(e)    => renderError(e)
    }
```

When `match` follows `|>`, the piped value is the matched expression. All arms must produce the same type as the pipeline expects downstream.

### 11.2 Closures and Pattern Parameters

Closure parameters can use irrefutable destructuring patterns:

```go
// Tuple parameter in closure
pairs.map(fn((k, v)) => "{k}: {v}")

// Struct parameter in closure
users.filter(fn({active, ..}) => active)

// Nested
entries.map(fn((i, {name, score, ..})) => "{i}. {name}: {score}")
```

### 11.3 Generics

Pattern matching on generic types works through their concrete variants. The compiler specializes exhaustiveness checking for each instantiation:

```go
fn unwrapOr[T](opt: Option[T], default: T) -> T = match opt {
    Some(v) => v
    None    => default
}

// Works for any T:
unwrapOr(Some("hello"), "world")   // T = str
unwrapOr(Some(42), 0)              // T = i64
```

When matching on a value of type `T` (unconstrained generic), only binding and wildcard patterns are allowed — there are no structural patterns available for an unknown type.

### 11.4 Effects and `with`

Pattern matching in functions with effect constraints is transparent — the match itself does not introduce or restrict effects. However, arm bodies may perform effectful operations within the declared effect context:

```go
fn processItems(items: [Item]) -> () with [IO] {
    for item in items {
        match item.kind {
            .Text    => io.println(item.content)    // IO effect in arm body
            .Image   => io.writeFile(item.path, item.data)?
            .Unknown => ()
        }
    }
}
```

### 11.5 Type Narrowing

After a match arm's pattern binds a value, the compiler narrows the type within that arm's scope. This enables type-safe field access without explicit casts:

```go
match shape {
    Circle{radius} => {
        // type of `radius` is `f64` — narrowed from the Circle variant
        // `shape` is known to be Circle here — no cast needed
        pi * radius * radius
    }
    Rect{w, h} => {
        // type of `w` and `h` is `f64` — narrowed from the Rect variant
        w * h
    }
}
```

The narrowed type is only valid within the arm's scope. It is not available outside the match expression or in sibling arms.

### 11.6 String Interpolation

Patterns in string interpolation contexts are not supported — string interpolation (`"Hello, {name}"`) uses expression syntax, not pattern syntax.

---

## 12. Compiler Implementation Notes

This section documents the compiler's approach to pattern matching compilation and checking. It is normative for implementors.

### 12.1 Exhaustiveness Checking Algorithm

The Aria compiler uses a **pattern matrix algorithm** for exhaustiveness checking, similar to the algorithm described by Maranget (2007). The algorithm operates on a matrix where:
- Each row is a match arm (expressed as a vector of patterns, one per column if multiple values are being matched simultaneously)
- Each column corresponds to a scrutinee

The algorithm computes whether the union of all row patterns covers the full type space of the scrutinee. If not, it produces a **counterexample** — a specific value that is not covered — which is used to generate the error message.

```
// The counterexample algorithm produces the most specific uncovered value:
// For type Color = | Red | Green | Blue | Yellow
// With arms: Red => ... Green => ...
// Counterexample: Blue (first uncovered variant in declaration order)
error: patterns `Blue` and `Yellow` not covered
```

### 12.2 Pattern Compilation Strategy

The compiler compiles patterns to **decision trees** rather than backtracking automata. Decision trees guarantee:
- **Linear time** match execution (no exponential worst cases)
- **No repeated tests** of the same field at the same position
- **Optimal code generation** — each field is read from memory at most once

The decision tree is constructed by:
1. Choosing a column (scrutinee or field) to branch on — typically the leftmost column with the most specific patterns
2. Grouping arms by the top-level constructor at that column
3. Recursively building sub-trees for each group

### 12.3 Unreachable Arm Warnings

The compiler warns when a match arm can never be reached — either because a previous arm already covers all values that could reach it, or because the pattern is structurally impossible:

```go
match n {
    x if x > 0 => positive(x)
    x if x > 5 => large(x)     // WARNING: unreachable — `x > 0` above already handles this
    _          => zero_or_negative()
}

match color {
    .Red   => ...
    .Green => ...
    .Blue  => ...
    _      => ...    // WARNING: unreachable — all variants already covered above
}
```

```
warning[W0013]: unreachable match arm
  --> src/main.aria:8:5
   |
8  |     _      => default()
   |     ^ this arm is unreachable
   |
note: all variants of `Color` are already covered by the preceding arms
```

### 12.4 Redundant Pattern Warnings

A pattern is **redundant** (but not necessarily unreachable) when it duplicates coverage already provided by an earlier arm:

```go
match status {
    .Active   => "active"
    .Active   => "also active"    // WARNING: redundant — never reached
    .Pending  => "pending"
    _         => "other"
}
```

### 12.5 Common Error Messages

**Non-exhaustive match:**
```
error[E0004]: non-exhaustive match expression
  --> src/api.aria:25:9
   |
25 |         match response {
   |         ^^^^^ patterns `Err(..)` not covered
   |
   = help: add a catch-all arm, or add the following arm:
               Err(e) => todo()
```

**Or-pattern binding mismatch:**
```
error[E0031]: or-pattern arms bind different names
  --> src/shapes.aria:14:9
   |
14 |     Circle{radius} | Point => ...
   |     ^^^^^^^^^^^^^^^^ `Circle` binds `radius` but `Point` does not
   |
   = help: both sides of an or-pattern must bind the same names with the same types
```

**Refutable pattern in irrefutable position:**
```
error[E0005]: refutable pattern in irrefutable position
  --> src/main.aria:7:5
   |
7  |     Some(user) := findUser(id)
   |     ^^^^^^^^^^ pattern `None` is not handled
   |
   = help: use `else` to handle the non-matching case:
               Some(user) := findUser(id) else return None
```

**Type mismatch in match arms:**
```
error[E0308]: match arms have incompatible types
  --> src/handler.aria:18:20
   |
16 |     match code {
17 |         200 => processOk()     // returns `Response`
18 |         _   => 404             // ERROR: expected `Response`, found `i64`
   |                ^^^ expected `Response`, found `i64`
```

---

## 13. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Exhaustive by default | Compiler catches missed cases — critical for AI correctness. Silent non-exhaustive matches are the source of entire classes of bugs (null pointer, unhandled enum variant, unchecked error). |
| No fallthrough | Every arm is independent. No C-style fall-through bugs. No accidental execution of multiple arms. |
| Match as expression | Eliminates mutable temporaries, enables composition in pipelines, reduces the need for intermediate variables. |
| Guards do not affect exhaustiveness | A guarded arm might not fire. Counting it as coverage creates a false sense of security. The developer must explicitly handle the unguarded remainder. |
| Or-patterns require same bindings | Prevents accidental use of unbound variables. If `A` binds `x` but `B` does not, using `x` in the arm body would be undefined when `B` matches. |
| Range patterns | Natural for HTTP status codes, port numbers, ASCII ranges, protocol field values. Integer matching without ranges would require unmanageable lists of literals. |
| Dot-shorthand for variants | Fewer tokens when the type is inferrable. Removes redundant type qualification from the majority of match arms. |
| No negated patterns | Negated patterns complicate exhaustiveness checking and are always expressible as a guard. The cognitive simplification outweighs the minor convenience loss. |
| Float literal patterns warned | Float equality comparison is almost never correct (NaN, rounding). A warning prevents a common correctness pitfall. |
| Refutable let requires `else` that diverges | The binding is only valid if the pattern matches. The diverging `else` ensures the binding cannot be used in an invalid state. |
| Struct `..` rest pattern | Allows matching on relevant fields without committing to the struct's full field list. Makes match arms resilient to struct extensions — adding a new field does not break existing matches that use `..`. |
| Without `..`, all struct fields must be listed | Makes field omission an explicit choice (add `..` to opt out). Prevents silently ignoring a new field that should have been handled. |
| Guards can have side effects (with warning) | Full restriction would be unpractical — some guards legitimately call pure functions. The warning surfaces unintentional side effects. |
| Decision tree compilation | Linear-time match execution. No exponential blowup. Each field read at most once. Predictable performance for generated code. |
| New variant → compile errors everywhere | Mechanical code review. The compiler does the work of finding every place that needs updating. This is the killer feature for evolving AI-generated codebases. |

---

## 14. Token Cost Comparison

The following comparisons show token counts for common pattern matching tasks across Go, Rust, and Aria. Token counts are approximate (using GPT-4 tokenization) and represent typical generated code.

### 14.1 Simple Enum Match

**Task**: Match on a 3-variant enum and return a string.

**Go** — no exhaustiveness, uses `switch`:
```go
func colorName(c Color) string {
    switch c {
    case Red:
        return "red"
    case Green:
        return "green"
    case Blue:
        return "blue"
    default:
        return "unknown"  // required in Go, even with full enum coverage
    }
}
```
~30 tokens. Non-exhaustive — adding a variant does not break this code.

**Rust**:
```rust
fn color_name(c: Color) -> &'static str {
    match c {
        Color::Red   => "red",
        Color::Green => "green",
        Color::Blue  => "blue",
    }
}
```
~20 tokens. Exhaustive.

**Aria**:
```go
fn colorName(c: Color) -> str = match c {
    .Red   => "red"
    .Green => "green"
    .Blue  => "blue"
}
```
~15 tokens. Exhaustive. Dot-shorthand removes type qualification. No commas required between arms.

---

### 14.2 Nested Struct Match

**Task**: Match on a `Response` type with associated struct data.

**Go**:
```go
switch r := response.(type) {
case OkResponse:
    if r.Status == 200 {
        processBody(r.Body)
    } else if r.Status == 404 {
        handleNotFound()
    } else {
        handleOther(r.Status)
    }
case ErrResponse:
    handleError(r.Err)
}
```
~45 tokens. Requires type assertions. No field destructuring.

**Rust**:
```rust
match response {
    Response::Ok { status: 200, body, .. } => process_body(body),
    Response::Ok { status: 404, .. }       => handle_not_found(),
    Response::Ok { status, .. }            => handle_other(status),
    Response::Err(e)                       => handle_error(e),
}
```
~35 tokens. Exhaustive. Full field destructuring.

**Aria**:
```go
match response {
    Ok{status: 200, body, ..} => processBody(body)
    Ok{status: 404, ..}       => handleNotFound()
    Ok{status, ..}            => handleOther(status)
    Err(e)                    => handleError(e)
}
```
~25 tokens. Exhaustive. Dot-shorthand for variants. No commas between arms.

---

### 14.3 Error Handling with Match

**Task**: Match on a `Result` with typed error variants, handling 3 error cases.

**Go**:
```go
content, err := readFile(path)
if err != nil {
    var notFound *NotFoundError
    var permErr *PermissionError
    if errors.As(err, &notFound) {
        log.Warn("missing: " + notFound.Path)
        return nil, err
    } else if errors.As(err, &permErr) {
        panic("permission denied")
    } else {
        return nil, err
    }
}
```
~55 tokens. No exhaustiveness. `errors.As` is not type-safe.

**Rust**:
```rust
let content = match read_file(path) {
    Ok(c) => c,
    Err(IoError::NotFound { path }) => {
        log::warn!("missing: {}", path);
        return Err(e);
    }
    Err(IoError::PermissionDenied { .. }) => panic!("permission denied"),
    Err(e) => return Err(e),
};
```
~45 tokens. Exhaustive. Typed error variants.

**Aria**:
```go
content := readFile(path) catch |e| {
    IoError.NotFound{path}       => { log.warn("missing: {path}"); return Err(e) }
    IoError.PermissionDenied{..} => panic("permission denied")
    _                            => return Err(e)
}
```
~30 tokens. Exhaustive. Inline catch with typed arms.

---

### 14.4 Option / Nullable Matching

**Task**: Handle `Some(user)` vs. `None` from a lookup.

**Go** — no Option type, uses pointer nil:
```go
user := findUser(id)
if user == nil {
    createUser(id)
    return
}
greetUser(user)
```
~20 tokens. Not type-safe (any pointer can be nil).

**Rust**:
```rust
match find_user(id) {
    Some(user) => greet_user(&user),
    None       => create_user(id),
}
```
~15 tokens. Exhaustive.

**Aria**:
```go
match findUser(id) {
    Some(user) => greetUser(user)
    None       => createUser(id)
}
```
~12 tokens. Exhaustive. No commas required.

---

### 14.5 Range Matching

**Task**: Classify an HTTP status code into categories.

**Go**:
```go
switch {
case code >= 200 && code <= 299:
    handleSuccess(code)
case code >= 400 && code <= 499:
    handleClientError(code)
case code >= 500 && code <= 599:
    handleServerError(code)
default:
    handleOther(code)
}
```
~40 tokens. No exhaustiveness. Range conditions are verbose.

**Rust**:
```rust
match code {
    200..=299 => handle_success(code),
    400..=499 => handle_client_error(code),
    500..=599 => handle_server_error(code),
    _         => handle_other(code),
}
```
~25 tokens. Range patterns supported.

**Aria**:
```go
match code {
    200..=299 => handleSuccess(code)
    400..=499 => handleClientError(code)
    500..=599 => handleServerError(code)
    _         => handleOther(code)
}
```
~20 tokens. Range patterns. No commas between arms.

---

### 14.6 Token Cost Summary

| Task | Go | Rust | Aria | Aria savings vs. Go |
|---|---|---|---|---|
| Simple enum match (3 variants) | ~30 | ~20 | ~15 | -50% |
| Nested struct match | ~45 | ~35 | ~25 | -44% |
| Error handling with typed variants | ~55 | ~45 | ~30 | -45% |
| Option/nullable matching | ~20 | ~15 | ~12 | -40% |
| Range matching (4 ranges) | ~40 | ~25 | ~20 | -50% |
| **Average savings** | | | | **~46%** |

The token reduction comes from five sources:
1. **No `case` keyword** — one fewer token per arm
2. **No commas required** between arms — one fewer token per arm
3. **Dot-shorthand for variants** — saves `TypeName.` per arm where the type is inferred
4. **No `default` required** — exhaustiveness checking replaces the boilerplate safety net
5. **Inline destructuring** — `Ok{status: 200, body, ..}` vs. a type assertion + field access

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/formal-grammar.md](formal-grammar.md), [spec/error-handling.md](error-handling.md), [spec/concurrency-design.md](concurrency-design.md), and [spec/compiler-architecture.md](compiler-architecture.md).*
