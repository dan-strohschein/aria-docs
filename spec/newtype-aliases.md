# Aria Newtypes and Type Aliases Specification

**A specification for newtype declarations, type aliases, and the grammar rules that disambiguate them from sum types and structs.**

Newtypes are one of Aria's most valuable tools for AI code generation correctness. They prevent semantic type confusion — the class of bug where a `UserId` and an `OrderId` are both `i64` and get silently swapped. This spec defines how newtypes work, how they differ from aliases, and how the parser distinguishes all type declaration forms.

Cross-references:
- [high-level-design.md](../high-level-design.md) — type declaration syntax
- [spec/formal-grammar.md](formal-grammar.md) — `type_decl` production
- [spec/trait-system.md](trait-system.md) — derives and trait implementation for newtypes
- [spec/type-conversions.md](type-conversions.md) — conversion between newtypes and underlying types
- [spec/equality-comparison.md](equality-comparison.md) — equality for newtypes

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Type Declaration Grammar](#2-type-declaration-grammar)
3. [Newtypes](#3-newtypes)
4. [Type Aliases](#4-type-aliases)
5. [Newtype Construction and Extraction](#5-newtype-construction-and-extraction)
6. [Newtypes and Traits](#6-newtypes-and-traits)
7. [Common Newtype Patterns](#7-common-newtype-patterns)
8. [Design Rationale Summary](#8-design-rationale-summary)

---

## 1. Design Philosophy

The number one bug I generate in Go looks like this:

```go
func ProcessOrder(userID int64, orderID int64, amount int64) error {
    // I accidentally swap userID and orderID — both are int64, compiles fine
    db.Query("SELECT * FROM orders WHERE user_id = ? AND order_id = ?", orderID, userID)
}
```

This is not a skill issue. It's a type system issue. When three parameters have the same type, there is no compiler help against transposition. The function signature is a lie — it says `int64` three times when it means three semantically different things.

Newtypes solve this completely:

```
fn processOrder(userId: UserId, orderId: OrderId, amount: Amount) ! DbError {
    db.query("SELECT * FROM orders WHERE user_id = ? AND order_id = ?", userId, orderId)?
}
// Swapping userId and orderId is now a COMPILE ERROR
```

**AI rationale:** Newtypes are the cheapest, highest-impact correctness tool for AI code generation. They cost zero at runtime, they cost one line to declare, and they eliminate an entire class of parameter-swapping bugs. Every domain concept that maps to a primitive type should be a newtype.

---

## 2. Type Declaration Grammar

Aria uses the `type` keyword for three distinct declaration forms. The parser disambiguates them by what follows the `=` sign:

| Form | Syntax | Example | Distinguished by |
|---|---|---|---|
| **Struct** | `type Name { fields }` | `type User { name: str }` | `{` after name |
| **Sum type** | `type Name = \| Variant ...` | `type Shape = Circle \| Rect` | `\|` in body |
| **Newtype** | `type Name = UnderlyingType` | `type UserId = i64` | Single type, no `\|` or `{` |

Additionally, `alias` creates a type alias:

| Form | Syntax | Example |
|---|---|---|
| **Alias** | `alias Name = Type` | `alias Bytes = [u8]` |

### Grammar

```
type_decl   = [ visibility ] "type" IDENT [ generic_params ] "=" type_body [ derives_clause ]
            | [ visibility ] "type" IDENT [ generic_params ] struct_body [ derives_clause ] ;

type_body   = sum_variants          // | Variant1 | Variant2
            | type ;                // single type = newtype

alias_decl  = [ visibility ] "alias" IDENT [ generic_params ] "=" type ;
```

### Parser disambiguation

The parser determines the form as follows:

1. If `{` follows the name (no `=`): **struct**
2. If `=` is followed by `|`: **sum type**
3. If `=` is followed by a type expression (no `|`): **newtype**
4. If `alias` keyword: **alias**

This is always unambiguous with one token of lookahead after `=`.

---

## 3. Newtypes

A newtype creates a **distinct type** that wraps an underlying type. The newtype and its underlying type are **not interchangeable** — assignment, function calls, and operators require explicit conversion.

### Declaration

```
type UserId = i64
type OrderId = i64
type Email = str
type Amount = f64
type Timestamp = i64
type JsonString = str
```

### Distinctness

```
userId := UserId(42)
orderId := OrderId(42)

userId == orderId    // ❌ compile error: cannot compare UserId with OrderId

fn getUser(id: UserId) -> User ! DbError { ... }
getUser(orderId)     // ❌ compile error: expected UserId, got OrderId
getUser(42)          // ❌ compile error: expected UserId, got i64
getUser(userId)      // ✅ correct
```

### Zero-cost representation

Newtypes have **identical runtime representation** to their underlying type. `UserId` is stored as a bare `i64` in memory — no wrapper struct, no pointer indirection, no overhead. The distinction exists only at compile time.

### Generic newtypes

```
type NonEmpty[T] = [T]          // a list that the type system distinguishes
type Validated[T] = T           // a value that has been validated
type Sorted[T] = [T]            // a list guaranteed to be sorted
```

---

## 4. Type Aliases

An alias creates an **alternative name** for an existing type. The alias and the original type are **fully interchangeable** — they are the same type.

### Declaration

```
alias Bytes = [u8]
alias Headers = Map[str, str]
alias Handler = fn(Request) -> Response ! HttpError
alias Predicate[T] = fn(T) -> bool
```

### Interchangeability

```
data: Bytes = [0x48, 0x65, 0x6C, 0x6C, 0x6F]
raw: [u8] = data           // ✅ Bytes IS [u8] — fully interchangeable

fn process(data: [u8]) { ... }
process(data)               // ✅ Bytes is accepted where [u8] is expected
```

### When to use aliases vs newtypes

| Use a **newtype** when... | Use an **alias** when... |
|---|---|
| The type has semantic meaning beyond its structure | The name is just shorthand for a long type |
| You want the compiler to prevent mixing types | You want full interchangeability |
| Example: `UserId`, `Email`, `Amount` | Example: `Headers`, `Bytes`, `Handler` |

**AI rationale:** The distinction between newtypes and aliases is clear and useful. When I see `type UserId = i64`, I know I must use `UserId(42)` to construct it and cannot accidentally pass a bare `i64`. When I see `alias Bytes = [u8]`, I know it's just a convenient name. The `type` vs `alias` keyword makes my intent explicit.

---

## 5. Newtype Construction and Extraction

### Construction

A newtype is constructed by calling the type name as a function:

```
id := UserId(42)
email := Email("alice@example.com")
amount := Amount(99.99)
```

### Extraction

The underlying value is accessed via the `.value` field:

```
id := UserId(42)
raw := id.value          // 42: i64

email := Email("alice@example.com")
s := email.value         // "alice@example.com": str
```

### Conversion between newtype and underlying type

```
// Newtype → underlying: .value
raw := userId.value

// Underlying → newtype: constructor
userId := UserId(raw)
```

There is no implicit conversion in either direction. Both directions are explicit and visible in the code.

### Generic newtype construction

```
type NonEmpty[T] = [T]

items := NonEmpty([1, 2, 3])    // construct from [i64]
raw := items.value               // [1, 2, 3]: [i64]
```

---

## 6. Newtypes and Traits

Newtypes **do not inherit** any traits from their underlying type. You must derive or implement traits explicitly.

```
type UserId = i64

// UserId does NOT automatically get Eq, Ord, Hash, Display from i64
// You must opt in:
derives [Eq, Hash, Debug, Clone] for UserId

// Or implement manually:
impl Display for UserId {
    fn display(self) -> str = "user:{self.value}"
}
```

### Why newtypes don't inherit traits

If `UserId` inherited `Numeric` from `i64`, you could write `userId1 + userId2` — adding two user IDs is nonsensical. By requiring explicit trait derivation, the type author controls exactly which operations are available.

```
type UserId = i64 derives [Eq, Hash, Clone, Debug]
// ✅ Can compare, hash, clone, debug-print
// ❌ Cannot add, subtract, multiply — Numeric not derived

type Amount = i64 derives [Eq, Ord, Hash, Clone, Debug, Numeric]
// ✅ CAN do arithmetic — Amount + Amount makes sense
```

### Selective trait derivation

This is the power of newtypes: you choose which operations make semantic sense:

| Newtype | Sensible traits | NOT sensible |
|---|---|---|
| `UserId` | `Eq`, `Hash`, `Clone`, `Debug`, `Display` | `Numeric`, `Ord` |
| `Amount` | `Eq`, `Ord`, `Hash`, `Numeric`, `Clone`, `Debug` | (all sensible) |
| `Email` | `Eq`, `Hash`, `Clone`, `Debug`, `Display` | `Ord` (alphabetical ordering of emails?) |
| `Timestamp` | `Eq`, `Ord`, `Hash`, `Clone`, `Debug` | `Numeric` (adding timestamps is wrong) |

**AI rationale:** When I generate code using a newtype, I can only use operations that the type author explicitly allowed. `UserId + UserId` is a compile error, not a silent bug. This is the "pit of success" design — the correct code is the only code that compiles.

---

## 7. Common Newtype Patterns

### ID types

```
type UserId = i64 derives [Eq, Hash, Clone, Debug, Display]
type OrderId = i64 derives [Eq, Hash, Clone, Debug, Display]
type ProductId = i64 derives [Eq, Hash, Clone, Debug, Display]
```

### Validated strings

```
type Email = str derives [Eq, Hash, Clone, Debug]
type Url = str derives [Eq, Hash, Clone, Debug]
type PhoneNumber = str derives [Eq, Hash, Clone, Debug]

impl Email {
    fn parse(s: str) -> Email ! ValidationError {
        if !s.contains("@") {
            return Err(ValidationError{msg: "missing @"})
        }
        Email(s)
    }
}
```

### Units of measure

```
type Meters = f64
type Feet = f64
type Seconds = f64
type Kilograms = f64

// Cannot mix units — compile error
fn area(width: Meters, height: Meters) -> Meters {
    Meters(width.value * height.value)
}
```

### Permission/capability tokens

```
type AuthToken = str derives [Clone, Debug]
type ApiKey = str derives [Clone, Debug]

fn authenticate(token: AuthToken) -> User ! AuthError { ... }
fn callApi(key: ApiKey) -> Response ! ApiError { ... }

// Cannot accidentally pass an ApiKey where AuthToken is expected
```

### Semantic wrappers for collections

```
type SortedList[T] = [T]
type UniqueList[T] = [T]
type NonEmptyList[T] = [T]

// The type name documents the invariant
fn binarySearch[T: Ord](list: SortedList[T], target: T) -> T? { ... }
```

---

## 8. Design Rationale Summary

| Decision | Rationale |
|---|---|
| `type Foo = T` is a newtype (distinct) | Prevents semantic type confusion — the highest-value AI correctness tool |
| `alias Foo = T` is an alias (interchangeable) | Shorthand for long types without adding safety constraints |
| Newtypes are zero-cost | Same runtime representation — no overhead for safety |
| Newtypes don't inherit traits | Type author controls which operations are sensible |
| Construction via `Foo(value)` | Minimal syntax — one token overhead |
| Extraction via `.value` | Uniform, predictable access pattern |
| Parser disambiguation by `\|` and `{` | Unambiguous with one token of lookahead |
| No implicit conversion | Both directions are explicit and visible |

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/trait-system.md](trait-system.md), [spec/type-conversions.md](type-conversions.md), and [spec/formal-grammar.md](formal-grammar.md).*
