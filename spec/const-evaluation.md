# Aria Compile-Time Constants Specification

**A specification for `const` declarations, constant expressions, and compile-time evaluation in Aria v0.1.**

This document defines what values can be computed at compile time and how they interact with the rest of the language. The design is deliberately minimal for v0.1 — compile-time function execution is deferred to a future version.

Cross-references:
- [high-level-design.md](../high-level-design.md) — `const` usage in examples
- [spec/formal-grammar.md](formal-grammar.md) — `const_decl` production
- [spec/numeric-overflow.md](numeric-overflow.md) — overflow behavior in const arithmetic

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Const Declarations](#2-const-declarations)
3. [Constant Expressions](#3-constant-expressions)
4. [Const Structs and Arrays](#4-const-structs-and-arrays)
5. [Const and the Type System](#5-const-and-the-type-system)
6. [What Const Cannot Do (v0.1)](#6-what-const-cannot-do)
7. [Future: Compile-Time Functions](#7-future-compile-time-functions)
8. [Design Rationale Summary](#8-design-rationale-summary)

---

## 1. Design Philosophy

Constants are the simplest optimization: compute once at compile time, inline everywhere. The question is how far to push compile-time evaluation. Zig's `comptime` is a full compile-time interpreter. Rust's `const fn` has evolved over seven years and still has restrictions. Both added significant complexity to their compilers.

For Aria v0.1, the answer is: **keep it simple.** Constants support literals, arithmetic, and composition of other constants. No compile-time function execution. This gives the bootstrap compiler a tractable feature to implement while covering the vast majority of real-world `const` usage.

**AI rationale:** When I generate `const MAX_RETRIES = 3`, I need to know this is inlined everywhere and evaluated at compile time. I don't need to generate compile-time Fibonacci — I need lookup tables, configuration constants, and mathematical constants. The v0.1 design covers all of these.

---

## 2. Const Declarations

### Syntax

```
const NAME = expression
const NAME: Type = expression
```

The type annotation is optional when the type can be inferred from the expression.

### Examples

```
const MAX_RETRIES = 3
const DEFAULT_TIMEOUT: dur = 30s
const PI = 3.14159265358979
const APP_NAME = "aria-server"
const MAX_CONNECTIONS: u32 = 1024
const BUFFER_SIZE = 64 * 1024          // 65536 — computed at compile time
```

### Visibility

Constants follow the same visibility rules as all declarations:

```
pub const API_VERSION = "v2"           // visible to importers
pub(pkg) const INTERNAL_KEY = "abc"    // visible within the package
const PRIVATE_LIMIT = 100              // module-private
```

### Naming convention

Constants use `UPPER_SNAKE_CASE` by convention. The compiler emits a warning for constants that don't follow this convention.

---

## 3. Constant Expressions

A constant expression is an expression that can be fully evaluated at compile time. The following are valid in constant context:

### Allowed in const expressions

| Expression type | Example | Notes |
|---|---|---|
| Integer literals | `42`, `0xFF`, `0b1010` | Any integer literal |
| Float literals | `3.14`, `1e10` | Any float literal |
| String literals | `"hello"` | Including interpolation of other consts |
| Bool literals | `true`, `false` | |
| Duration literals | `30s`, `5m`, `100ms` | |
| Size literals | `64kb`, `1mb` | |
| Arithmetic | `MAX * 2 + 1` | `+`, `-`, `*`, `/`, `%` on numeric consts |
| Comparison | `MAX > MIN` | `==`, `!=`, `<`, `>`, `<=`, `>=` |
| Logical | `A && B`, `!C` | `&&`, `||`, `!` |
| Bitwise | `FLAGS \| MASK` | `&`, `\|`, `^`, `~`, `<<`, `>>` |
| String concatenation | Implicit via interpolation | `"prefix_{SUFFIX}"` |
| References to other consts | `OTHER_CONST + 1` | Must reference previously declared consts |
| Parenthesized expressions | `(A + B) * C` | Grouping |
| Negation | `-MAX_VALUE` | Unary minus on numeric const |

### Not allowed in const expressions (v0.1)

| Expression type | Example | Why not |
|---|---|---|
| Function calls | `sqrt(2.0)` | Requires compile-time function execution |
| Method calls | `"hello".len()` | Requires compile-time method dispatch |
| Variable references | `x + 1` | Variables are runtime values |
| Collection operations | `[1,2,3].len()` | Requires compile-time method dispatch |
| Conditional expressions | `if A > B { A } else { B }` | Adds control flow to const evaluator |
| Match expressions | `match X { ... }` | Adds control flow to const evaluator |
| Closures | `fn() => 42` | Runtime construct |

### Const string interpolation

String interpolation in const context only interpolates other const values:

```
const APP = "aria"
const VERSION = "0.1.0"
const USER_AGENT = "{APP}/{VERSION}"    // "aria/0.1.0" — computed at compile time

const COUNT = 42
const MSG = "count is {COUNT}"          // "count is 42" — const integer formatted
```

### Overflow in const context

Arithmetic overflow in a const expression is a **compile error**, not a runtime panic or wrap:

```
const TOO_BIG: u8 = 200 + 200
// ❌ compile error: constant expression overflows u8 (400 > 255)
```

This catches overflow bugs before the program ever runs.

---

## 4. Const Structs and Arrays

Constants can be arrays or structs, as long as all values are const expressions:

### Const arrays

```
const PRIMES = [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
const VOWELS = ["a", "e", "i", "o", "u"]
const POWERS_OF_TWO = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]
```

### Const structs

```
const DEFAULT_CONFIG = Config {
    host: "localhost"
    port: 8080
    timeout: 30s
}

const ORIGIN = Point { x: 0, y: 0 }
```

### Const maps

```
const HTTP_STATUS = {
    200: "OK",
    201: "Created",
    204: "No Content",
    400: "Bad Request",
    404: "Not Found",
    500: "Internal Server Error",
}
```

**AI rationale:** Const arrays and maps are how I generate lookup tables — HTTP status codes, error message maps, configuration defaults. These are common patterns that should be evaluated at compile time with zero runtime cost.

---

## 5. Const and the Type System

### Const values are inlined

Every reference to a const is replaced with the const's value at compile time. There is no runtime storage for constants — they exist only in the compiled code at each usage site.

```
const MAX = 100

fn check(n: i64) -> bool = n <= MAX
// Compiled as: fn check(n: i64) -> bool = n <= 100
```

### Const values are immutable

Constants cannot be reassigned, mutated, or shadowed by a mutable binding in the same scope:

```
const X = 42
X = 43              // ❌ compile error: cannot assign to constant
mut X := 43         // ❌ compile error: cannot shadow constant with mutable binding
```

A local variable may shadow a constant in an inner scope:

```
const X = 42
fn example() {
    x := X + 1      // x is 43, using the constant
    // X is still 42 — the constant is not affected
}
```

### Const type annotations

When the type is ambiguous, annotate it:

```
const ZERO = 0              // inferred as i64 (default integer type)
const ZERO_U8: u8 = 0       // explicitly u8
const ZERO_F: f64 = 0.0     // explicitly f64
```

---

## 6. What Const Cannot Do (v0.1)

The following are explicitly **not supported** in v0.1. These are deferred, not rejected — they may appear in future versions.

### No compile-time function execution

```
// ❌ Not supported in v0.1
const FACTORIAL_10 = factorial(10)
const TABLE = buildLookupTable(256)
```

**Workaround:** Pre-compute the value and write it as a literal:

```
const FACTORIAL_10 = 3628800
```

### No compile-time conditionals

```
// ❌ Not supported in v0.1
const MAX = if TARGET_64BIT { i64.max } else { i32.max }
```

### No const function parameters

```
// ❌ Not supported in v0.1
fn createBuffer(const size: usize) -> Buffer { ... }
```

### No const generics

```
// ❌ Not supported in v0.1
type FixedArray[const N: usize, T] { data: [T] }
```

---

## 7. Future: Compile-Time Functions

A future version of Aria may introduce `comptime fn` for compile-time function execution:

```
// Hypothetical v0.2+ syntax
comptime fn factorial(n: u64) -> u64 = match n {
    0 => 1
    n => n * factorial(n - 1)
}

const FACT_10 = factorial(10)    // evaluated at compile time
```

This is deferred because:
1. The bootstrap compiler is written in Go — adding a compile-time Aria interpreter is significant work
2. The v0.1 `const` covers 90%+ of real-world constant needs
3. Getting const generics and comptime right requires experience with the language

**AI rationale:** I'd rather have a simple, reliable `const` now than a complex `comptime` that might have edge cases. The v0.1 design lets me generate correct constant declarations immediately. Compile-time function execution can be added once the compiler is mature enough to support it.

---

## 8. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Simple const for v0.1 | Covers 90%+ of use cases; avoids bootstrap compiler complexity |
| Literals + arithmetic + const refs | All common patterns: config values, limits, lookup tables |
| Const arrays and structs | Lookup tables and default configurations at zero runtime cost |
| Overflow is compile error | Catches bugs before runtime — stronger than runtime panic |
| Always inlined | Zero runtime storage; const is a compile-time concept only |
| No comptime fn in v0.1 | Deferred to v0.2 — requires Aria interpreter in the compiler |
| No const generics in v0.1 | Deferred — complex interaction with type system |
| String interpolation of consts | Natural syntax for building constant strings |

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/formal-grammar.md](formal-grammar.md), [spec/numeric-overflow.md](numeric-overflow.md), and [high-level-design.md](../high-level-design.md).*
