# Aria Pattern Matching Specification

**A complete, implementable specification for pattern matching in Aria — covering syntax, semantics, exhaustiveness checking, and every pattern kind.**

This document consolidates and extends the pattern matching sketches found throughout the Aria specification into a single authoritative reference. Every design decision is justified through the lens of AI code generation — the primary consumer of this language.

Cross-references:
- [high-level-design.md](../high-level-design.md) — enums, sum types, `match ok/err`, nullability
- [spec/formal-grammar.md](formal-grammar.md) — sections 3.14 (match expressions) and 3.16 (patterns)
- [spec/scoping-rules.md](scoping-rules.md) — section 8 (pattern binding scopes)
- [spec/language-spec-addendum.md](language-spec-addendum.md) — destructuring, bindings
- [spec/error-handling.md](error-handling.md) — `catch` arms, typed error patterns
- [spec/concurrency-design.md](concurrency-design.md) — `select` arms, channel patterns
- [spec/iteration-protocol.md](iteration-protocol.md) — `for` loop destructuring
- [spec/compiler-architecture.md](compiler-architecture.md) — Stage 4, exhaustiveness checking

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Pattern Taxonomy](#2-pattern-taxonomy)
3. [Match Expressions](#3-match-expressions)
4. [Exhaustiveness Checking](#4-exhaustiveness-checking)
5. [Pattern Matching in Bindings](#5-pattern-matching-in-bindings)
6. [Pattern Matching in `for` Loops](#6-pattern-matching-in-for-loops)
7. [Pattern Matching in `catch` and Error Handling](#7-pattern-matching-in-catch-and-error-handling)
8. [Pattern Matching in `select`](#8-pattern-matching-in-select)
9. [Guards](#9-guards)
10. [Nested and Composed Patterns](#10-nested-and-composed-patterns)
11. [Interaction with Other Language Features](#11-interaction-with-other-language-features)
12. [Compiler Implementation Notes](#12-compiler-implementation-notes)
13. [Design Rationale Summary](#13-design-rationale-summary)
14. [Token Cost Comparison](#14-token-cost-comparison)

---

## 1. Design Philosophy

Pattern matching is the primary control flow mechanism for branching on the structure and content of data in Aria. It is not a library feature, not syntactic sugar over if-chains — it is a core language primitive that the type checker, compiler, and runtime are all built around.

### Core Principles

#### Exhaustive by default — the compiler is your co-pilot

Every `match` expression in Aria is exhaustive. If you match a sum type with five variants and only cover four, the program does not compile. The compiler tells you which variants you missed and suggests the missing arms.

```go
type Status = | Active | Inactive | Suspended | Banned | PendingVerification

fn describe(s: Status) -> str = match s {
    .Active  => "account active"
    .Inactive => "account inactive"
    // COMPILE ERROR: non-exhaustive match
    // missing: .Suspended, .Banned, .PendingVerification
    // hint: add _ => ... or handle each variant explicitly
}
```

**AI rationale:** This is the single most important safety property in Aria for AI-generated code. When an AI generates a match expression, the compiler verifies that every possible case was handled. When a new variant is added to an enum, every match site in the entire codebase becomes a compile error — the compiler walks the AI through every place that needs updating. No runtime crash, no silent miss. This eliminates a whole class of bugs that are essentially impossible to prevent in languages without exhaustiveness checking.

#### Expression-oriented — match always returns a value

`match` is an expression, not a statement. Every arm must produce a value of the same type. This eliminates the mutable temporary variable pattern that generates incorrect code in Go and Java.

```go
// Aria — match as expression, no mutable temporary
label := match status {
    .Active  => "✓ Active"
    .Inactive => "○ Inactive"
    .Suspended => "⚠ Suspended"
    .Banned   => "✗ Banned"
    .PendingVerification => "… Pending"
}
```

```go
// Go equivalent — requires mutable temporary, AI often forgets the default case
var label string
switch status {
case Active:
    label = "✓ Active"
case Inactive:
    label = "○ Inactive"
// AI frequently forgets cases here — no compile error in Go
}
// label may be "" if AI forgot a case
```

**AI rationale:** Go's `switch` is not exhaustive. The AI can generate a switch with a missing case and the compiler says nothing. At runtime, `label` is the zero value — an empty string — and the bug manifests silently. In Aria, the compiler catches this at compile time with a specific error.

#### Token-efficient — no redundant keywords

Match arms require no `case` keyword, no `break`, no commas between arms. The `=>` separator is the minimum unambiguous token. Arms are newline-terminated; commas are accepted but not required.

```go
// Aria — minimal tokens
match shape {
    Circle{radius} => pi * radius * radius
    Rect{w, h}     => w * h
    Point          => 0.0
}
```

```go
// Rust equivalent — similar but requires comma after every arm
match shape {
    Circle { radius } => PI * radius * radius,
    Rect { w, h }     => w * h,
    Point             => 0.0,
}
```

#### No fallthrough — each arm is independent

Unlike C's `switch`, there is no implicit fallthrough between arms. Each arm is completely independent. If you want the same body for multiple patterns, use or-patterns (`A | B`).

```go
match day {
    .Monday | .Tuesday | .Wednesday | .Thursday | .Friday => "weekday"
    .Saturday | .Sunday => "weekend"
}
```

### Why Pattern Matching Is The Killer Feature for AI

When generating code in Go or Python, an AI must mentally track which cases it has handled and which it hasn't. There is no compile-time feedback. The result is:

- `switch` statements with missing cases (silent zero-value bugs)
- `if/else if` chains that are accidentally non-exhaustive
- Type assertions (`x.(SomeType)`) that panic at runtime
- Null pointer dereferences from unhandled `nil` returns

In Aria, every one of these failure modes is a compile error, not a runtime surprise. The AI generates match expressions, and the compiler guarantees the generated code handles every case.

---

## 2. Pattern Taxonomy

Aria has 14 pattern kinds. Every pattern can be nested arbitrarily inside other patterns.

### 2.1 Wildcard Pattern

**Syntax:** `_`

**Semantics:** Matches any value. Does not bind — the matched value is discarded.

```go
match response {
    Ok(data) => process(data)
    Err(_)   => log.warn("request failed")  // error value is discarded
}
```

**Exhaustiveness:** A wildcard covers all remaining cases. Use it as a catch-all arm, but be aware that adding new variants to a sum type will not produce a compile error at wildcard arms — the wildcard silently matches the new variant. For sum types where you want the compiler to enforce handling new variants, prefer exhaustive coverage without a wildcard.

```go
// WARNING: adding a new variant to Status won't produce an error here
match status {
    .Active => serve(request)
    _       => reject(request)  // silently catches .PendingVerification, etc.
}

// PREFERRED: explicit exhaustion — compile error when new variant added
match status {
    .Active             => serve(request)
    .Inactive           => reject(request)
    .Suspended          => reject(request)
    .Banned             => reject(request)
    .PendingVerification => reject(request)
}
```

### 2.2 Binding Pattern

**Syntax:** `name` or `mut name`

**Semantics:** Matches any value and binds it to the given name. The binding is immutable by default; `mut` makes it mutable within the arm body.

```go
match findUser(id) {
    Some(user)     => greet(user)          // `user` is bound, immutable
    Some(mut user) => { user.lastSeen = now(); save(user) }  // mutable binding
    None           => createUser(id)
}
```

**Scope:** Bindings are scoped to the arm body. They are not accessible in other arms or after the match expression. See [spec/scoping-rules.md](scoping-rules.md) section 8.

### 2.3 Literal Pattern

**Syntax:** Any literal value — integer, float, string, boolean, character, or duration.

**Semantics:** Matches only the exact value.

```go
match statusCode {
    200 => "OK"
    201 => "Created"
    404 => "Not Found"
    500 => "Internal Server Error"
    _   => "Other"  // required: integers are not sum types, wildcard or range needed
}

match enabled {
    true  => startService()
    false => stopService()
    // exhaustive — boolean has exactly two values
}

match greeting {
    "hello" => respond("hi")
    "bye"   => respond("goodbye")
    _       => respond("...")  // required: strings are not sum types
}
```

**Exhaustiveness:** Literal patterns on `bool` are exhaustive when both `true` and `false` are covered. Literal patterns on integers, floats, characters, and strings always require a wildcard or range pattern to be exhaustive (since the value space is not finite/enumerable).

### 2.4 Struct Pattern

**Syntax:** `TypeName { field1, field2: subpattern, .. }`

**Semantics:** Matches a struct (or struct-like enum variant) and destructures its fields. Fields can be:
- **Shorthand:** `field` — binds the field value to a name equal to the field name
- **Renamed:** `field: pattern` — matches the field value against a sub-pattern
- **Rest:** `..` — ignores all remaining fields (must appear last)

```go
type User {
    name: str
    age:  i32
    email: str
    role: Role
}

match user {
    // Shorthand — binds `name` and `age`, ignores `email` and `role`
    {name, age, ..} => println("{name} is {age}")

    // Renamed — binds email field as `addr`
    {email: addr, ..} => sendTo(addr)

    // Sub-pattern on a field
    {name, role: .Admin, ..} => grantAccess(name)
    {name, role: .User, ..}  => limitAccess(name)
}
```

For enum variants with struct syntax:

```go
type Shape =
    | Circle { radius: f64 }
    | Rect   { w: f64, h: f64 }
    | Point

match shape {
    Circle{radius}  => pi * radius * radius
    Rect{w, h}      => w * h
    Point           => 0.0
}
```

**Type checking:** Every named field in a struct pattern must exist on the matched type. Missing fields without `..` is a compile error. Mentioning a field that doesn't exist on the type is a compile error.

### 2.5 Variant Pattern

**Syntax:** `VariantName(subpattern)` or `TypeName.VariantName(subpattern)`

**Semantics:** Matches a specific enum variant and destructures its payload.

```go
match result {
    Ok(value)   => process(value)
    Err(IoError.NotFound{path}) => log.warn("missing: {path}")
    Err(e)      => propagate(e)
}

// Nested variant patterns
match event {
    MouseEvent.Click{x, y}        => handleClick(x, y)
    MouseEvent.Move{x, y}         => updateCursor(x, y)
    KeyEvent.Press{key}           => handleKey(key)
    KeyEvent.Release{key}         => handleKeyRelease(key)
    WindowEvent.Resize{w, h}      => handleResize(w, h)
    WindowEvent.Close             => shutdown()
}
```

**Qualified variant syntax** (`TypeName.VariantName`) is used when the type cannot be inferred from context, or when matching on a value of a generic or union type where multiple types share variant names.

### 2.6 Tuple Pattern

**Syntax:** `(p1, p2, p3)`

**Semantics:** Matches a tuple of the corresponding arity and applies each sub-pattern to the corresponding element.

```go
match coordinates {
    (0, 0)    => "origin"
    (x, 0)    => "on x-axis at {x}"
    (0, y)    => "on y-axis at {y}"
    (x, y)    => "at ({x}, {y})"
}

match getResult() {
    (true, value)  => process(value)
    (false, error) => handle(error)
}
```

**Arity checking:** The pattern must have the same arity as the tuple type. Mismatched arity is a compile error.

### 2.7 Array and List Pattern

**Syntax:** `[p1, p2, ..rest]`

**Semantics:** Matches an array or list. Elements are matched positionally from the left. `..rest` captures all remaining elements into a slice binding.

```go
match args {
    []             => println("no arguments")
    [cmd]          => runCommand(cmd)
    [cmd, ..flags] => runWithFlags(cmd, flags)
}

match bytes {
    [0xFF, 0xD8, ..rest] => decodeJpeg(rest)
    [0x89, 0x50, ..rest] => decodePng(rest)
    _                    => Err(UnknownFormat)
}
```

**Head-tail decomposition:**

```go
match items {
    []              => Ok([])
    [first, ..rest] => {
        processed := transform(first)?
        [processed, ..processAll(rest)?]
    }
}
```

**Fixed-length patterns:** A pattern with no `..` matches exactly that length. A mismatch is not a compile error (since list length is not a type-level property) but will not match at runtime.

### 2.8 Or-Pattern

**Syntax:** `pattern_a | pattern_b`

**Semantics:** Matches if either sub-pattern matches. Both sub-patterns must bind exactly the same names with exactly the same types.

```go
match day {
    .Monday | .Wednesday | .Friday => "MWF schedule"
    .Tuesday | .Thursday           => "TTh schedule"
    .Saturday | .Sunday            => "weekend"
}

// OK — both bind `radius: f64`
match shape {
    Circle{radius} | Ellipse{radius} => pi * radius * radius
}

// COMPILE ERROR — binding inconsistency: `Point` does not bind `radius`
match shape {
    Circle{radius} | Point => pi * radius * radius   // ERROR
}
```

**Binding consistency rule:** In `A | B`, every name bound by `A` must also be bound by `B` with the same type, and vice versa. This is enforced by the compiler. See [spec/scoping-rules.md](scoping-rules.md) section 8.

**With guards:** Or-patterns can be combined with guards. The guard applies to the entire or-pattern arm:

```go
match value {
    Circle{radius} | Ellipse{radius} if radius > 0.0 => pi * radius * radius
    _ => 0.0
}
```

### 2.9 Rest Pattern

**Syntax:** `..` or `..name`

**Semantics:** In struct patterns, `..` ignores all remaining fields. In array patterns, `..name` captures all remaining elements into a slice bound to `name`.

```go
// In a struct pattern — ignore remaining fields
{name, email, ..} := getUser()

// In an array pattern — capture tail
[head, ..tail] := items
```

`..` without a name simply ignores the remaining elements or fields without binding. `..name` binds the remaining elements to `name`.

**Placement:** In array patterns, `..name` must appear at most once and must appear last. Multiple rest patterns in a single array pattern is a compile error.

### 2.10 Range Pattern

**Syntax:** `start..end` (exclusive) or `start..=end` (inclusive)

**Semantics:** Matches integers or characters within the specified range.

```go
match statusCode {
    100..=199 => "informational"
    200..=299 => "success"
    300..=399 => "redirect"
    400..=499 => "client error"
    500..=599 => "server error"
    _         => "unknown"
}

match c {
    'a'..='z' => "lowercase"
    'A'..='Z' => "uppercase"
    '0'..='9' => "digit"
    _         => "other"
}

match port {
    0..=1023    => "privileged port"
    1024..=49151 => "registered port"
    49152..=65535 => "dynamic port"
    _           => "invalid"  // needed for exhaustiveness on u32
}
```

**Type support:** Range patterns are supported on all integer types (`i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`, `u128`, `isize`, `usize`) and on `char`. Range patterns on floats, strings, or other types are not supported.

**Exhaustiveness:** A set of range patterns is exhaustive when they collectively cover every value in the type's domain. For bounded types like `u8` (0–255), complete coverage is possible. For large integer types, a wildcard arm is almost always required.

### 2.11 Qualified Variant Pattern

**Syntax:** `TypeName.VariantName` or `TypeName.VariantName{fields}` or `TypeName.VariantName(payload)`

**Semantics:** Explicitly names the enum type containing the variant. Used when the type cannot be inferred from context alone.

```go
// Necessary when matching on a union of error types
match err {
    IoError.NotFound{path}        => log.warn("file not found: {path}")
    IoError.PermissionDenied{..}  => log.error("permission denied")
    ParseError.InvalidJson{line}  => log.error("bad json at line {line}")
    DbError.ConnectionFailed{..}  => reconnect()
}
```

Without the qualifier, the compiler may not be able to resolve which type's `NotFound` is meant when multiple types in scope have a variant with that name.

### 2.12 Dot-Shorthand Variant Pattern

**Syntax:** `.VariantName` or `.VariantName{fields}` or `.VariantName(payload)`

**Semantics:** When the type of the matched value is known from context, the type name can be omitted. The leading dot signals that this is a variant of the inferred type.

```go
// The compiler knows `status: Status`, so `.Active` is `Status.Active`
match status {
    .Active             => serve(request)
    .Inactive           => reject(request)
    .Suspended          => reject(request)
    .Banned             => reject(request)
    .PendingVerification => prompt(request)
}

// Enum field access in datetime matching
match dt.dayOfWeek {
    .Monday | .Wednesday | .Friday => "MWF schedule"
    .Tuesday | .Thursday           => "TTh schedule"
    .Saturday | .Sunday            => "weekend"
}
```

**Inference rule:** Dot-shorthand is valid when the type of the matched expression is a specific named enum type. If the type is ambiguous (e.g., a type variable, a union type), dot-shorthand is not allowed and the compiler requires a qualified variant.

### 2.13 Guard Patterns

**Syntax:** `pattern if condition`

**Semantics:** The guard is a Boolean expression evaluated after the structural pattern matches. If the guard evaluates to `false`, the arm does not match and the next arm is tried.

```go
match value {
    x if x > 0   => "positive"
    x if x < 0   => "negative"
    _             => "zero"
}

match user {
    {role: .Admin, name, ..} if name != "root" => grantAccess(name)
    {role: .Admin, ..}                          => restrictedAccess()
    _                                           => denyAccess()
}
```

See [Section 9](#9-guards) for complete guard semantics.

### 2.14 Trait-Test Pattern

**Syntax:** `binding if binding is TraitName`

**Semantics:** A special form of guard that checks whether a value implements a trait. Commonly used in error handling to dispatch on error categories.

```go
match err {
    e if e is Retryable  => scheduleRetry(e)
    e if e is UserFault  => return Err(UserError.fromCause(e))
    e                    => return Err(e)
}
```

Trait-test patterns narrow the type of the binding within the guard body and the arm body. After `e if e is Retryable`, `e` is known to implement `Retryable` and its methods are accessible without a cast.

---

## 3. Match Expressions

### 3.1 Syntax and Grammar

```
match_expr = "match" expression "{" { match_arm } "}" ;
match_arm  = pattern [ "if" expression ] "=>" ( expression | block ) [ "," ] ;
```

Arms are separated by newlines. Trailing commas are accepted but not required. The entire `match` construct is an expression that produces the value of the matched arm's body.

```go
// Match as expression
area := match shape {
    Circle{radius} => pi * radius * radius
    Rect{w, h}     => w * h
    Point          => 0.0
}

// Match as statement (return value discarded)
match level {
    .Debug   => log.debug(message)
    .Info    => log.info(message)
    .Warning => log.warn(message)
    .Error   => log.error(message)
}

// Multi-line arm bodies use a block
result := match request {
    Ok(req) if req.isValid() => {
        processed := process(req)?
        formatResponse(processed)
    }
    Ok(req) => Err(ValidationError.Invalid{reason: "request failed validation"})
    Err(e)  => Err(e)
}
```

### 3.2 Type Rules

When `match` is used as an expression, all arms must produce values of the same type. A type mismatch between arms is a compile error.

```go
// COMPILE ERROR: arm type mismatch
value := match flag {
    true  => 42         // i64
    false => "no"       // str — type mismatch!
}

// CORRECT: all arms produce the same type
value := match flag {
    true  => 42
    false => 0
}
```

When `match` is used as a statement (return value discarded), all arms must produce values of the same type or the unit type `()`. An arm that ends with a side-effecting call returning `()` and an arm that returns a value of type `T` is a type error.

### 3.3 Arm Evaluation Order

Arms are evaluated top-to-bottom. The first arm whose pattern matches (and whose guard, if any, evaluates to `true`) is selected. Subsequent arms are not evaluated.

```go
// The first matching arm wins
match n {
    0       => "zero"
    1       => "one"
    2..=9   => "single digit"
    10..=99 => "double digit"
    _       => "large"
}
```

**Unreachable arms:** If an arm can never match because a preceding arm already covers it completely, the compiler emits a warning.

```go
// WARNING: unreachable arm — `_` already covers everything
match n {
    _ => "any"
    5 => "five"   // WARNING: unreachable
}
```

### 3.4 Match as Statement vs. Expression

Match is an expression by default. When used in statement position (the result is discarded), it behaves as a statement. There is no syntactic difference — the usage context determines which mode applies.

```go
// Statement mode — result discarded
match status {
    .Active   => startTask()
    .Inactive => stopTask()
    .Suspended | .Banned => return Err(AccessDenied)
    .PendingVerification  => return Err(NotYetVerified)
}

// Expression mode — result used
message := match status {
    .Active   => "running"
    .Inactive => "stopped"
    .Suspended | .Banned => "blocked"
    .PendingVerification  => "pending"
}
```

### 3.5 Nested Match Expressions

Match expressions can be nested. Each nested match is independently exhaustive.

```go
response := match request {
    Ok(req) => match req.method {
        .GET    => handleGet(req)
        .POST   => handlePost(req)
        .PUT    => handlePut(req)
        .DELETE => handleDelete(req)
        _       => Err(MethodNotAllowed)
    }
    Err(e) => Err(NetworkError.fromCause(e))
}
```

---

## 4. Exhaustiveness Checking

Exhaustiveness checking is performed by the compiler at Stage 4 (Type Checking). See [spec/compiler-architecture.md](compiler-architecture.md) for the overall compilation pipeline.

### 4.1 How Exhaustiveness Works

The compiler builds a **coverage matrix** — for each possible value of the matched type, it determines whether at least one arm matches it. If any value is uncovered, the match is non-exhaustive and compilation fails.

The compiler reasons about exhaustiveness differently depending on the matched type:

#### Simple Enums

A simple enum (no associated data) is exhaustive when every variant is covered by at least one arm (or a wildcard covers the remainder).

```go
type Color = | Red | Green | Blue

// Exhaustive — all 3 variants covered
match color {
    .Red   => "#FF0000"
    .Green => "#00FF00"
    .Blue  => "#0000FF"
}

// COMPILE ERROR — non-exhaustive
// error: non-exhaustive match on `Color`
//   missing: .Blue
//   hint: add `.Blue => ...` or add a catch-all `_ => ...`
match color {
    .Red   => "#FF0000"
    .Green => "#00FF00"
}
```

#### Sum Types with Associated Data

Each variant must be covered. The associated data inside the variant does not affect variant-level exhaustiveness — matching `Circle{..}` covers the `Circle` variant regardless of what fields are ignored.

```go
type Shape =
    | Circle { radius: f64 }
    | Rect   { w: f64, h: f64 }
    | Point

// Exhaustive
match shape {
    Circle{..}  => "circle"
    Rect{..}    => "rectangle"
    Point       => "point"
}

// Also exhaustive — binding patterns cover any variant
match shape {
    Circle{radius} => pi * radius * radius
    Rect{w, h}     => w * h
    Point          => 0.0
}
```

#### Booleans

`true` and `false` must both be covered (or a wildcard).

```go
// Exhaustive
match flag {
    true  => enable()
    false => disable()
}
```

#### Option Types

`Some(x)` and `None` must both be covered.

```go
// Exhaustive
match findUser(id) {
    Some(user) => greet(user)
    None       => createUser(id)
}

// COMPILE ERROR
// error: non-exhaustive match on `Option[User]`
//   missing: None
match findUser(id) {
    Some(user) => greet(user)
}
```

#### Result Types

`Ok(x)` and `Err(e)` must both be covered.

```go
// Exhaustive
match readFile(path) {
    Ok(content) => process(content)
    Err(e)      => log.error("read failed: {e}")
}
```

#### Error Unions

When a function returns `T ! ErrorA | ErrorB | ErrorC`, matching on the error requires covering all error variants.

```go
// Function signature: fn initialize() -> App ! IoError | ParseError | DbError
match initialize() {
    Ok(app)              => run(app)
    Err(IoError.NotFound{path})     => log.error("missing file: {path}")
    Err(IoError.PermissionDenied{..}) => log.error("permission denied")
    Err(IoError.Timeout{after})     => retry(after)
    Err(IoError.ConnectionRefused{addr, port}) => alertOps(addr, port)
    Err(ParseError.InvalidJson{..}) => log.error("config syntax error")
    Err(ParseError.UnknownField{..}) => log.error("unknown config field")
    Err(DbError.ConnectionFailed{..}) => waitAndRetry()
    Err(DbError.QueryFailed{..})    => log.error("database error")
}

// Simpler — use wildcard per error type
match initialize() {
    Ok(app)      => run(app)
    Err(e: IoError)    => handleIo(e)
    Err(e: ParseError) => handleParse(e)
    Err(e: DbError)    => handleDb(e)
}
```

#### Nested Sum Types

Exhaustiveness is checked recursively through nested sum types.

```go
type Tree[T] = | Leaf | Node { value: T, left: Tree[T], right: Tree[T] }

fn depth(t: Tree[i64]) -> i64 = match t {
    Leaf            => 0
    Node{left, right, ..} => 1 + max(depth(left), depth(right))
}
```

#### Integer Ranges

For integer types, range patterns contribute to coverage. The compiler tracks which integers have been covered. A wildcard is almost always required unless every value in the type's domain is explicitly covered.

```go
// u8 has 256 possible values — explicit coverage is possible
match byte {
    0        => "null"
    1..=31   => "control"
    32..=126 => "printable"
    127      => "delete"
    128..=255 => "extended"
    // exhaustive! every u8 value is covered
}

// i64 has 2^64 possible values — wildcard is required
match n {
    0       => "zero"
    1..=100 => "small positive"
    _       => "other"  // required
}
```

#### String Values

Strings are not enumerable. A wildcard or binding pattern is always required.

```go
match greeting {
    "hello" | "hi" | "hey" => respond("hello!")
    "bye"   | "goodbye"    => respond("bye!")
    _                      => respond("...")  // required
}
```

#### Or-Patterns and Exhaustiveness

An or-pattern `A | B` counts as covering both `A` and `B` for exhaustiveness purposes.

```go
type Day = | Mon | Tue | Wed | Thu | Fri | Sat | Sun

// Exhaustive — or-patterns cover all 7 variants
match day {
    .Mon | .Tue | .Wed | .Thu | .Fri => "weekday"
    .Sat | .Sun                       => "weekend"
}
```

### 4.2 When Wildcards and Catch-All Bindings Are Required

A wildcard (`_`) or catch-all binding (`name`) is **required** when:
- The matched type has an unbounded value space (integers, strings, floats)
- The matched type is extensible (a variant from another module could be added)
- Some cases are handled with guards (guards do not count as coverage — see [Section 9.3](#93-guards-and-exhaustiveness))

A wildcard is **not required** (and is discouraged) when:
- Matching an enum defined in the current module where all variants are known
- You want the compiler to alert you when a new variant is added

### 4.3 Compiler Errors for Non-Exhaustive Matches

The compiler produces structured error messages for non-exhaustive matches:

```
error[E0301]: non-exhaustive match on `Result[Config, IoError | ParseError]`
  --> src/main.aria:24:5
   |
24 |     match readConfig(path) {
   |     ^^^^^ non-exhaustive
   |
   = missing arm for: `Err(ParseError._)`
   = hint: add the following arm:
   |         Err(e: ParseError) => ...
   |
   = note: `IoError` variants are fully covered
```

When multiple variants are missing:

```
error[E0301]: non-exhaustive match on `Status`
  --> src/server.aria:47:5
   |
47 |     match status {
   |     ^^^^^ non-exhaustive
   |
   = missing arms for: .Suspended, .Banned, .PendingVerification
   = hint: add the following arms:
   |         .Suspended          => ...
   |         .Banned             => ...
   |         .PendingVerification => ...
   |     or add a catch-all: _ => ...
```

### 4.4 The Killer Feature: Evolving Sum Types

When a new variant is added to a sum type, every match site in the codebase that doesn't use a wildcard becomes a compile error. The compiler directs the programmer (or AI) to every location that needs updating.

```go
// Before: Status has 4 variants
type Status = | Active | Inactive | Suspended | Banned

// Adding a new variant...
type Status = | Active | Inactive | Suspended | Banned | PendingVerification

// IMMEDIATELY, every match without a wildcard becomes a compile error:
// error[E0301]: non-exhaustive match on `Status`
//   --> src/auth.aria:15
//   --> src/dashboard.aria:42
//   --> src/report.aria:88
//   missing: .PendingVerification
```

**AI rationale:** This is the pattern matching "killer app" for AI code generation. When an AI generates code that works for today's enum, and a human adds a new variant tomorrow, the compiler finds every location the AI missed. The AI doesn't have to be perfect — the type system provides a safety net that catches any omissions.

---

## 5. Pattern Matching in Bindings

### 5.1 Irrefutable Patterns

An **irrefutable pattern** is one that always matches — it cannot fail. Irrefutable patterns can be used directly in `:=` bindings.

```go
// Tuple destructuring — always matches a (i64, i64)
(x, y) := getCoordinates()

// Struct destructuring — always matches a User
{name, email, ..} := getUser()

// Rename during destructure
{name: userName, email: userEmail} := getUser()

// Nested tuple destructuring
((a, b), c) := getNestedTuple()
```

**What makes a pattern irrefutable?**
- Wildcard `_` — always matches
- Binding `name` — always matches
- Struct patterns with `..` on any struct type — always matches
- Tuple patterns whose arity matches the type — always matches
- Or-patterns where at least one alternative is irrefutable — always matches

**What makes a pattern refutable?**
- Literal patterns (`42`, `"hello"`, `true`) — only matches one value
- Variant patterns (`Some(x)`, `Ok(v)`, `.Active`) — only matches one variant
- Range patterns (`0..=100`) — only matches a subset of values
- Guard patterns — may not match

Using a refutable pattern in a plain `:=` binding is a compile error.

```go
// COMPILE ERROR: refutable pattern in binding
Some(user) := findUser(id)   // ERROR: `findUser` may return `None`
```

### 5.2 Refutable Patterns with `else`

Refutable patterns can be used in bindings with a mandatory `else` branch. The `else` branch must diverge — it must `return`, `break`, `continue`, `panic`, or execute a block that always diverges.

```go
// Syntax: pattern := expr else divergent_block
Some(user) := findUser(id) else return None
// `user` is in scope here

Ok(data) := parseJson(input) else |err| {
    log.warn("parse failed: {err}")
    return defaultConfig
}
// `data` is in scope here

Circle{radius} := shape else {
    log.warn("expected circle, got {shape}")
    return 0.0
}
// `radius` is in scope here
```

**Scope rule:** Bindings introduced by the pattern are available in the current scope after the binding statement — they are not scoped to the `else` block. The `else` block cannot use pattern bindings (they don't exist in the `else` branch, since the pattern didn't match).

**Divergence requirement:** The compiler verifies that the `else` branch always diverges. A non-diverging `else` branch is a compile error.

```go
// COMPILE ERROR: else branch does not diverge
Some(user) := findUser(id) else {
    log.warn("user not found")
    // ERROR: what is `user` set to?
}
```

### 5.3 Destructuring in Function Parameters

Function parameters can use irrefutable patterns directly.

```go
// Tuple parameter destructuring
fn distance((x1, y1): (f64, f64), (x2, y2): (f64, f64)) -> f64 =
    sqrt((x2 - x1).pow(2) + (y2 - y1).pow(2))

// Struct parameter destructuring
fn greet({name, role, ..}: User) -> str = match role {
    .Admin => "Welcome, Admin {name}"
    .User  => "Hello, {name}"
    .Guest => "Hi there"
}

// Closure parameter destructuring
pairs := [(1, "a"), (2, "b"), (3, "c")]
labels := pairs |> map(fn(n, s) => "{n}:{s}")
```

Only irrefutable patterns are allowed in function parameters. Refutable patterns (variant patterns, literal patterns) in function parameters are a compile error.

---

## 6. Pattern Matching in `for` Loops

Destructuring patterns can be used directly in `for` loop variable bindings.

### 6.1 Simple Binding

```go
for item in collection {
    process(item)
}
```

### 6.2 Tuple Destructuring

```go
// Tuple pairs (key-value from map iteration)
for (key, value) in config.entries() {
    println("{key} = {value}")
}

// Tuples from zip
for (left, right) in lefts.zip(rights) {
    compare(left, right)
}
```

### 6.3 Struct Destructuring

```go
for {name, score, ..} in students {
    if score >= 90 { println("{name}: honors") }
}
```

### 6.4 Enumerated Destructuring

```go
for (index, item) in list.enumerate() {
    println("{index}: {item}")
}

// Combining enumerate with struct destructuring
for (index, {name, email, ..}) in users.enumerate() {
    println("{index}: {name} <{email}>")
}
```

### 6.5 Pattern Interaction with `Iterable`

The `for` loop desugars to calls to the `Iterable` and `Iterator` traits. The loop variable pattern is applied to each value yielded by the iterator's `.next()` method. See [spec/iteration-protocol.md](iteration-protocol.md) for the complete `Iterable`/`Iterator` protocol.

Refutable patterns in `for` loop variables silently skip non-matching elements (the `for` loop implicitly filters):

```go
// This is a filtered iteration — only `Some` values are processed
// `None` values are skipped (they don't match `Some(x)`)
for Some(x) in maybeValues {
    process(x)
}
```

---

## 7. Pattern Matching in `catch` and Error Handling

### 7.1 Typed `catch` Arms

`catch` uses match arm syntax to dispatch on specific error types or variants.

```go
fn loadConfig(path: str) -> Config ! ConfigError {
    content := readFile(path) catch |err| match err {
        IoError.NotFound{path}  => return Err(ConfigError.Missing{path})
        IoError.PermissionDenied{..} => return Err(ConfigError.Unreadable{path})
        e                       => return Err(ConfigError.Io{cause: e})
    }
    parseConfig(content)?
}
```

`catch` arms follow the same exhaustiveness rules as `match` arms: for a function that can fail with `IoError | ParseError`, a catch block must cover all variants of both error types, or use a wildcard.

### 7.2 Match on Result Directly

```go
match readFile("config.toml") {
    Ok(config)                      => startServer(config)
    Err(IoError.NotFound{path})     => log.fatal("config not found: {path}")
    Err(IoError.PermissionDenied{..}) => log.fatal("cannot read config")
    Err(e)                          => log.fatal("unexpected error: {e}")
}
```

### 7.3 Trait-Category Dispatch

Error categories encoded as traits enable pattern matching on the category of an error rather than its specific type. See [spec/error-handling.md](error-handling.md) section 6.

```go
match err {
    e if e is Retryable  => scheduleRetry(e)
    e if e is UserFault  => return Err(UserError.fromCause(e))
    e if e is Permanent  => alert(oncall, e)
    e                    => return Err(e)
}
```

For full error handling pattern reference, see [spec/error-handling.md](error-handling.md).

---

## 8. Pattern Matching in `select`

`select` uses a match-arm-like syntax for concurrent channel operations. Each arm is a receive, send, timeout, or default operation.

### 8.1 Basic Select

```go
select {
    msg from ch1  => process(msg)
    msg from ch2  => handleOther(msg)
    after 5s      => timeout()
}
```

### 8.2 Patterns in Receive Arms

The receive pattern (`msg from ch`) is a binding pattern applied to each received value. Full destructuring is supported:

```go
select {
    (id, payload) from requestCh  => handleRequest(id, payload)
    {level, message, ..} from logCh => writeLog(level, message)
    after 30s => flush()
}
```

### 8.3 Default and Timeout Arms

```go
select {
    msg from workQueue => process(msg)
    default            => idle()    // non-blocking — runs if no channel is ready
}

select {
    result from computeCh => use(result)
    after 10s             => return Err(Timeout{after: 10s})
}
```

`after` arms accept any duration expression. `default` arms make the `select` non-blocking.

### 8.4 Select Exhaustiveness

`select` is not subject to exhaustiveness checking — it models runtime channel availability, not a type domain. A `select` with no `default` and no `after` will block until one of its channel arms becomes ready.

For complete `select` semantics, see [spec/concurrency-design.md](concurrency-design.md).

---

## 9. Guards

### 9.1 Syntax

```
guard_arm = pattern "if" expression "=>" body ;
```

A guard is a Boolean expression that is evaluated after the structural pattern matches. If the guard is `false`, the arm is skipped and the next arm is tried.

```go
match value {
    n if n > 0    => "positive"
    n if n < 0    => "negative"
    _             => "zero"
}
```

### 9.2 Guard Scope

Guards have access to all bindings introduced by the pattern in that arm.

```go
match user {
    {name, age, role: .Admin, ..} if age >= 18 => grantAccess(name)
    {name, age, ..}               if age < 13  => restrictToKidsMode(name)
    {name, ..}                                  => defaultAccess(name)
}
```

The bindings `name`, `age`, and the structural match on `role` are all available in the guard expression.

### 9.3 Guards and Exhaustiveness

**Guards do not contribute to exhaustiveness checking.** A guarded arm is not counted as covering its pattern for the purposes of determining whether the match is exhaustive.

```go
// COMPILE ERROR: non-exhaustive despite the guard arm
match n {
    x if x > 0 => "positive"   // does NOT count as covering all integers
    0          => "zero"
    // missing: negative integers
}

// CORRECT: explicit coverage for all cases
match n {
    x if x > 0 => "positive"
    x if x < 0 => "negative"
    _          => "zero"       // required — guards don't count
}
```

**Rationale:** If a guarded arm were counted as exhaustive, the match would fail at runtime for values where the guard is `false`. Guarded arms provide filtering, not coverage.

### 9.4 Guard Side Effects

Guard expressions are generally expected to be pure (no side effects) for predictability, since the order and number of guard evaluations may depend on the pattern matching strategy. However, Aria does not enforce purity for guards at the language level — an impure guard is allowed. Be aware that:

- Guards may be evaluated more than once in some pattern compilation strategies
- Guards are evaluated lazily (only when the structural pattern matches)
- Guards on `select` arms follow channel-selection semantics, not match order

### 9.5 Trait-Test Guards

The `is` operator in a guard tests trait membership at runtime:

```go
// `e is TraitName` returns bool
match err {
    e if e is Retryable => scheduleRetry(e)
    e                   => return Err(e)
}
```

After `e if e is TraitName`, the compiler narrows `e`'s type within the arm body to `e: TraitName` (a trait object reference). The trait's methods are callable without further casting.

---

## 10. Nested and Composed Patterns

### 10.1 Arbitrary Nesting

All pattern kinds can be nested inside other patterns to any depth.

```go
// Struct inside variant inside Result
match fetchUser(id) {
    Ok({name, role: .Admin, permissions, ..}) => {
        log.info("admin {name} logged in")
        grantPermissions(permissions)
    }
    Ok({name, ..}) => {
        log.info("{name} logged in")
        grantDefaultAccess()
    }
    Err(DbError.NotFound{..}) => createGuestSession()
    Err(e)                    => return Err(e)
}
```

### 10.2 Nested Struct Destructuring in Match

```go
match response {
    Ok({status: 200, body, ..})    => processBody(body)
    Ok({status: 201, location, ..}) => redirect(location)
    Ok({status: 404, ..})          => handleNotFound()
    Ok({status, ..})               => handleOther(status)
    Err(e)                         => handleError(e)
}
```

### 10.3 Array Patterns with Nested Patterns

```go
match events {
    []                              => log.info("no events")
    [{type: .Click, x, y}, ..rest]  => { handleClick(x, y); processEvents(rest) }
    [{type: .Key, key, ..}, ..rest] => { handleKey(key); processEvents(rest) }
    [_, ..rest]                     => processEvents(rest)  // skip unknown events
}
```

### 10.4 Or-Patterns with Nested Sub-Patterns

Or-patterns can contain complex sub-patterns on each side, provided the binding sets are identical.

```go
// Both alternatives bind `radius: f64`
match shape {
    Circle{radius} | Ellipse{radius} => computeCircumference(radius)
    Rect{w, h}                       => 2.0 * (w + h)
    _                                => 0.0
}
```

### 10.5 Deep Exhaustiveness Checking

For deeply nested patterns, the compiler checks exhaustiveness at each level of nesting independently.

```go
type ApiResponse[T] = Ok { data: T, meta: Meta } | Err { code: i32, message: str }
type Meta = { cached: bool, version: i32 }

// Compiler verifies:
// 1. Top-level: Ok and Err are both covered
// 2. `data` field access is valid (it's in the `Ok` variant)
// 3. `meta.cached` sub-pattern is valid given `meta: Meta`
match response {
    Ok{data, meta: {cached: true, ..}}  => serveCached(data)
    Ok{data, meta: {cached: false, ..}} => serve(data)
    Err{code: 401, ..}                  => return Err(Unauthorized)
    Err{code: 403, ..}                  => return Err(Forbidden)
    Err{code, message}                  => return Err(ApiError{code, message})
}
```

---

## 11. Interaction with Other Language Features

### 11.1 Pipeline Operator

`match` can appear in a pipeline as the final operation, consuming the piped value as the matched expression.

```go
// Match at the end of a pipeline
result := fetchUsers()
    |> filter(.active)
    |> sortBy(.lastLogin)
    |> first()
    |> match {
        Some(user) => greet(user)
        None       => "no users found"
    }
```

A pipeline into match uses the anonymous pipe target — the piped value becomes the matched expression.

### 11.2 Closures

Closure parameters support irrefutable patterns:

```go
// Tuple destructuring in closure parameters
pairs |> map(fn(k, v) => "{k}={v}")

// Struct destructuring in closure parameters
users |> filter(fn({role, ..}) => role == .Admin)
users |> map(fn({name, email, ..}) => Contact{name, email})
```

### 11.3 Generics

Pattern matching works with generic types. The compiler instantiates the match for the concrete type at each call site.

```go
fn first[T](items: [T]) -> Option[T] = match items {
    []          => None
    [head, ..]  => Some(head)
}

fn unwrapOr[T](result: Result[T, _], default: T) -> T = match result {
    Ok(value) => value
    Err(_)    => default
}
```

### 11.4 Effects System

Pattern matching is transparent to the effects system. The effect context of a match arm is the union of the effects of the matched expression and the selected arm body.

```go
// If the match arm body has effect `IO`, the match expression has effect `IO`
fn processFile(path: str) -> str ! IoError {
    content := match openFile(path)? {
        {size: 0, ..} => return ""   // IO effect from openFile already propagated
        file          => file.readAll()?  // IO effect here
    }
    content
}
```

### 11.5 Type Narrowing

After a match arm's pattern matches, the compiler narrows the type of any bound values:

- A binding `x` in arm `Some(x)` is narrowed from `Option[T]` (the matched type) to `T`
- A binding `e` in arm `Err(e)` is narrowed from `Result[T, E]` to `E`
- After `e if e is Retryable`, `e` is narrowed to `dyn Retryable`

```go
match value {
    Some(x) => {
        // x: T (not Option[T])
        x.doSomething()
    }
    None => {
        // x is not in scope here
    }
}
```

---

## 12. Compiler Implementation Notes

### 12.1 Exhaustiveness Checking Algorithm

The compiler uses a **pattern matrix** algorithm (a variant of Maranget's algorithm) for exhaustiveness checking:

1. **Build the pattern matrix:** Each arm contributes a row; each column corresponds to one level of the matched type.
2. **Specialize:** For each constructor (variant, literal, struct), recursively compute coverage.
3. **Default matrix:** Collect arms that don't cover a specific constructor (wildcards, bindings).
4. **Useful check:** A pattern is useful if it matches at least one value not covered by preceding arms.
5. **Exhaustive check:** The match is exhaustive if the set of all uncovered cases is empty.

This algorithm handles or-patterns, guards (excluded from coverage), wildcards, nested patterns, and range patterns correctly.

### 12.2 Pattern Compilation Strategy

Aria compiles match expressions to **decision trees** rather than backtracking:

- Each decision in the tree tests one sub-value (one constructor, one field, one array element)
- No arm is tested more than once per input path
- The resulting machine code has no backtracking — every match runs in O(depth of pattern) tests

This guarantees that the cost of matching is proportional to the depth of the deepest pattern, not the number of arms.

### 12.3 Unreachable Arm Warnings

The compiler warns when an arm is unreachable — it can never match because a preceding arm already covers all values that would reach it.

```
warning[W0201]: unreachable match arm
  --> src/handler.aria:33:5
   |
33 |     .Active if isAdmin => adminView()
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   = note: this arm is covered by the preceding wildcard arm at line 30
```

Common causes:
- A wildcard or binding arm appearing before a more specific arm
- A literal arm duplicated
- An or-pattern that duplicates a variant from a preceding arm

### 12.4 Redundant Pattern Warnings

```
warning[W0202]: redundant pattern
  --> src/handler.aria:45:5
   |
45 |     .Active | .Active => handle()
   |               ^^^^^^^ this variant appears twice in the or-pattern
```

### 12.5 Compiler Error Messages

The compiler provides actionable error messages for pattern matching failures. Key error codes:

| Code | Meaning |
|---|---|
| `E0301` | Non-exhaustive match — missing arms listed, hints provided |
| `E0302` | Or-pattern binding inconsistency — names/types that differ are reported |
| `E0303` | Refutable pattern in irrefutable position (binding without `else`) |
| `E0304` | Pattern arity mismatch (wrong number of tuple/array elements) |
| `E0305` | Unknown field in struct pattern |
| `E0306` | Duplicate field in struct pattern |
| `W0201` | Unreachable match arm |
| `W0202` | Redundant pattern |

---

## 13. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Exhaustive by default | Compiler catches missed cases at compile time — critical for AI correctness; new enum variants become compiler errors at every match site |
| No fallthrough | Every arm is independent — eliminates C-style fall-through bugs; or-patterns are the intentional multi-arm pattern |
| Match as expression | Eliminates mutable temporaries and uninitialized variables; enables composition with pipelines and closures |
| Or-patterns require same bindings | Prevents accidental access of unbound names; if `A` binds `x` but `B` doesn't, using `x` in the arm body would be undefined behavior |
| Guards don't affect exhaustiveness | A guard can be `false`, so a guarded arm doesn't guarantee coverage; counting it as coverage would produce silent runtime failures |
| Range patterns | Natural for HTTP status codes, port numbers, byte values, character classes — avoids long chains of `or` conditions |
| Dot-shorthand for variants | Fewer tokens when type is inferrable from context; consistent with Aria's principle that every token must carry unique information |
| Wildcard is double-edged | `_` provides escape from exhaustiveness but silently absorbs new variants; the spec discourages it for module-local enums where variant evolution is tracked |
| Decision-tree compilation | Predictable O(depth) performance; no backtracking; enables the compiler to prove arm coverage at compile time |
| Pattern bindings are immutable by default | Consistent with Aria's default immutability; `mut` is opt-in and explicit |
| `else` branch must diverge | Ensures that after a refutable binding, the bound names are always valid — no "maybe bound" uncertainty |
| Trait-test guards narrow types | After `e if e is Retryable`, methods of `Retryable` are directly callable — no explicit downcast required |

---

## 14. Token Cost Comparison

This section compares Aria's pattern matching syntax against Go and Rust for common patterns. Aria is consistently more concise while maintaining or improving safety.

### 14.1 Simple Enum Matching

**Aria (13 tokens, exhaustiveness enforced):**
```go
match color {
    .Red   => "#FF0000"
    .Green => "#00FF00"
    .Blue  => "#0000FF"
}
```

**Go (22 tokens, not exhaustive):**
```go
switch color {
case Red:
    return "#FF0000"
case Green:
    return "#00FF00"
case Blue:
    return "#0000FF"
}
```

**Rust (19 tokens, exhaustiveness enforced):**
```rust
match color {
    Color::Red   => "#FF0000",
    Color::Green => "#00FF00",
    Color::Blue  => "#0000FF",
}
```

| Metric | Aria | Go | Rust |
|---|---|---|---|
| Tokens | 13 | 22 | 19 |
| Exhaustiveness check | ✅ Compile error | ❌ None | ✅ Compile error |
| Type qualifier required | ❌ `.Red` | ❌ `Red` | ✅ `Color::Red` |
| Arm separator | newline | `case:` keyword | `,` |

### 14.2 Nested Struct Matching

**Aria (27 tokens):**
```go
match response {
    Ok({status: 200, body, ..}) => process(body)
    Ok({status: 404, ..})       => notFound()
    Ok({status, ..})            => handleOther(status)
    Err(e)                      => handleError(e)
}
```

**Go (60+ tokens, no destructuring):**
```go
if response.Err != nil {
    handleError(response.Err)
} else if response.Status == 200 {
    process(response.Body)
} else if response.Status == 404 {
    notFound()
} else {
    handleOther(response.Status)
}
```

**Rust (35 tokens):**
```rust
match response {
    Ok(Response { status: 200, body, .. }) => process(body),
    Ok(Response { status: 404, .. })       => not_found(),
    Ok(Response { status, .. })            => handle_other(status),
    Err(e)                                 => handle_error(e),
}
```

| Metric | Aria | Go | Rust |
|---|---|---|---|
| Tokens | ~27 | ~60 | ~35 |
| Nested destructuring | ✅ | ❌ manual | ✅ |
| Exhaustiveness check | ✅ | ❌ | ✅ |
| Rest pattern (`..`) | ✅ | N/A | ✅ |

### 14.3 Option / Nullable Matching

**Aria (12 tokens):**
```go
match findUser(id) {
    Some(user) => greet(user)
    None       => createUser(id)
}
```

**Go (20 tokens, no Option type):**
```go
user, err := findUser(id)
if err != nil {
    createUser(id)
} else {
    greet(user)
}
```

**Rust (14 tokens):**
```rust
match find_user(id) {
    Some(user) => greet(user),
    None       => create_user(id),
}
```

| Metric | Aria | Go | Rust |
|---|---|---|---|
| Tokens | 12 | 20 | 14 |
| Null safety enforced | ✅ | ❌ | ✅ |
| Exhaustiveness | ✅ | ❌ | ✅ |

### 14.4 Error Handling with Match

**Aria (24 tokens):**
```go
match readFile(path) {
    Ok(content)              => process(content)
    Err(IoError.NotFound{path}) => log.warn("missing: {path}")
    Err(IoError.Timeout{..}) => retry()
    Err(e)                   => return Err(e)
}
```

**Go (45 tokens):**
```go
content, err := readFile(path)
if err != nil {
    var notFound *NotFoundError
    var timeout *TimeoutError
    if errors.As(err, &notFound) {
        log.Warn("missing:", notFound.Path)
    } else if errors.As(err, &timeout) {
        retry()
    } else {
        return err
    }
} else {
    process(content)
}
```

**Rust (28 tokens):**
```rust
match read_file(path) {
    Ok(content)                   => process(content),
    Err(IoError::NotFound { path }) => log::warn!("missing: {}", path),
    Err(IoError::Timeout { .. })  => retry(),
    Err(e)                        => return Err(e),
}
```

| Metric | Aria | Go | Rust |
|---|---|---|---|
| Tokens | ~24 | ~45 | ~28 |
| Typed error matching | ✅ | ❌ `errors.As` | ✅ |
| Exhaustiveness | ✅ | ❌ | ✅ |
| Type qualifier needed | ❌ | N/A | ✅ `IoError::` |

### 14.5 Range Matching

**Aria (22 tokens):**
```go
match statusCode {
    200..=299 => "success"
    400..=499 => "client error"
    500..=599 => "server error"
    _         => "other"
}
```

**Go (30 tokens):**
```go
switch {
case statusCode >= 200 && statusCode <= 299:
    return "success"
case statusCode >= 400 && statusCode <= 499:
    return "client error"
case statusCode >= 500 && statusCode <= 599:
    return "server error"
default:
    return "other"
}
```

**Rust (22 tokens):**
```rust
match status_code {
    200..=299 => "success",
    400..=499 => "client error",
    500..=599 => "server error",
    _         => "other",
}
```

| Metric | Aria | Go | Rust |
|---|---|---|---|
| Tokens | 22 | 30 | 22 |
| Range pattern syntax | ✅ `..=` | ❌ compound expr | ✅ `..=` |
| Exhaustiveness | ✅ | ❌ | ✅ |

---

## Appendix: Grammar Summary

```
// Match expression
match_expr   = "match" expression "{" { match_arm } "}" ;
match_arm    = pattern [ "if" expression ] "=>" ( expression | block ) [ "," ] ;

// Patterns
pattern = wildcard_pattern
        | binding_pattern
        | literal_pattern
        | range_pattern
        | struct_pattern
        | variant_pattern
        | tuple_pattern
        | array_pattern
        | or_pattern
        | rest_pattern
        | qualified_variant_pattern
        | dot_variant_pattern ;

wildcard_pattern         = "_" ;
binding_pattern          = [ "mut" ] IDENT ;
literal_pattern          = literal ;
range_pattern            = literal ( ".." | "..=" ) literal ;
struct_pattern           = [ IDENT "." ] IDENT "{" field_pattern_list "}" ;
variant_pattern          = IDENT "(" pattern_list ")" ;
tuple_pattern            = "(" pattern_list ")" ;
array_pattern            = "[" array_pat_list "]" ;
or_pattern               = pattern "|" pattern ;
rest_pattern             = ".." [ IDENT ] ;
qualified_variant_pattern = IDENT "." IDENT [ "{" field_pattern_list "}" | "(" pattern_list ")" ] ;
dot_variant_pattern      = "." IDENT [ "{" field_pattern_list "}" | "(" pattern_list ")" ] ;

field_pattern_list = field_pattern { "," field_pattern } [ "," ] ;
field_pattern      = IDENT
                   | IDENT ":" pattern
                   | ".." ;

pattern_list   = pattern { "," pattern } [ "," ] ;
array_pat_list = pattern { "," pattern } [ "," ] [ ".." IDENT ] ;

// Variable binding patterns
binding_stmt = pattern ":=" expression [ "else" ( block | "|" IDENT "|" block ) ] ;

// For loop patterns
for_stmt     = "for" pattern "in" expression block ;

// Select arms (concurrency)
recv_arm     = pattern "from" expression ;
```
