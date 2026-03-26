# Aria Generics and Type Parameters Specification

**A complete specification for generic functions, generic types, trait bounds, type inference, and monomorphization in Aria.**

This document formalizes the generics system sketched in [high-level-design.md](../high-level-design.md) (Generics section) and used throughout the standard library and spec directory. Generics are the mechanism by which Aria achieves type-safe code reuse without sacrificing performance.

Cross-references:
- [high-level-design.md](../high-level-design.md) — generics syntax overview, `[T]` brackets
- [spec/trait-system.md](trait-system.md) — trait bounds, associated types, derives
- [spec/formal-grammar.md](formal-grammar.md) — `generic_params` production
- [spec/stdlib-design.md](stdlib-design.md) — `Option[T]`, `Result[T,E]`, `Map[K,V]`
- [spec/type-conversions.md](type-conversions.md) — `.to[T]()`, `.trunc[T]()` generic methods
- [spec/error-handling.md](error-handling.md) — `Result[T, E]` and generic error functions
- [spec/iteration-protocol.md](iteration-protocol.md) — `Iterator[T]` generic trait

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Generic Functions](#2-generic-functions)
3. [Generic Structs and Sum Types](#3-generic-structs-and-sum-types)
4. [Generic Traits](#4-generic-traits)
5. [Generic Impl Blocks](#5-generic-impl-blocks)
6. [Trait Bounds and Constraints](#6-trait-bounds-and-constraints)
7. [Where Clauses](#7-where-clauses)
8. [Type Inference](#8-type-inference)
9. [Monomorphization](#9-monomorphization)
10. [Parser Disambiguation: `[T]` Generics vs Array Literals](#10-parser-disambiguation)
11. [Phantom Types](#11-phantom-types)
12. [Design Rationale Summary](#12-design-rationale-summary)
13. [Comparison with Other Languages](#13-comparison-with-other-languages)

---

## 1. Design Philosophy

Generics exist to solve one problem: writing code once that works for many types, without sacrificing type safety or performance. In Aria, generics are monomorphized — the compiler generates specialized code for each concrete type used, so generic code runs at the same speed as hand-specialized code.

### Why `[T]` Instead of `<T>`

The single most impactful syntax decision in Aria's generics system is using square brackets `[T]` instead of angle brackets `<T>`.

```
// Aria — no ambiguity
fn map[T, U](list: [T], f: fn(T) -> U) -> [U]
a < b    // always comparison

// C++ / Rust — ambiguous
fn map<T, U>(list: Vec<T>, f: Fn(T) -> U) -> Vec<U>
a < b    // comparison or start of generic? depends on context
```

**AI rationale:** The `<`/`>` ambiguity in C++ and Rust (the "turbofish problem") causes parsing complexity, requires context-dependent lookahead, and forces Rust's `::< >` turbofish syntax. Aria eliminates this entire class of parser ambiguity by using `[T]`. The AI never generates code that the parser might misinterpret — `[` after a type/function name is always generics; `[` at the start of an expression is always an array.

---

## 2. Generic Functions

### Basic generic function

```
fn identity[T](x: T) -> T = x

fn first[T](list: [T]) -> T? {
    if list.len() == 0 { None }
    else { Some(list[0]) }
}
```

### Multiple type parameters

```
fn map[T, U](list: [T], f: fn(T) -> U) -> [U] {
    [f(x) for x in list]
}

fn zip[T, U](a: [T], b: [U]) -> [(T, U)] {
    mut result: [(T, U)] = []
    for i in 0..min(a.len(), b.len()) {
        result = result.append((a[i], b[i]))
    }
    result
}
```

### With trait bounds

```
fn largest[T: Ord](items: [T]) -> T? {
    if items.len() == 0 { return None }
    mut max := items[0]
    for item in items {
        if item > max { max = item }
    }
    Some(max)
}

fn sum[T: Numeric](list: [T]) -> T {
    list.fold(T.zero, fn(a, b) => a + b)
}

fn debug_largest[T: Ord + Display](items: [T]) {
    match largest(items) {
        Some(v) => println("Largest: {v.display()}")
        None    => println("Empty collection")
    }
}
```

### Generic functions with error types

```
fn try_map[T, U, E](items: [T], f: fn(T) -> U ! E) -> [U] ! E {
    mut results: [U] = []
    for item in items {
        results = results.append(f(item)?)
    }
    results
}

fn retry[T, E: Transient](attempts: u64, op: fn() -> T ! E) -> T ! E {
    mut last_err: E? = None
    for _ in 0..attempts {
        match op() {
            Ok(v) => return v
            Err(e) => last_err = Some(e)
        }
    }
    return Err(last_err!)
}
```

### Calling generic functions

Type parameters are usually inferred from arguments:

```
// Type inferred from arguments
result := map([1, 2, 3], fn(x) => x * 2)        // T=i64, U=i64
names := map(users, fn(u) => u.name)             // T=User, U=str

// Explicit type parameters when needed
parsed := json.parse[Config](content)?            // T=Config
id := 42.to[UserId]()?                            // T=UserId
```

### Grammar

```
fn_decl = [ visibility ] "fn" IDENT [ generic_params ]
          "(" [ param_list ] ")" [ "->" type ] [ error_clause ] [ effect_clause ]
          fn_body ;

generic_params = "[" generic_param { "," generic_param } "]" ;
generic_param  = IDENT [ ":" trait_bound ] ;
trait_bound    = IDENT { "+" IDENT } ;
```

---

## 3. Generic Structs and Sum Types

### Generic structs

```
struct Pair[A, B] {
    first:  A
    second: B
}

struct Stack[T] {
    items: [T]
}

struct Cache[K, V] {
    data: Map[K, V]
    ttl: dur
}
```

### Generic sum types

```
type Option[T] = Some(T) | None

type Result[T, E] = Ok(T) | Err(E)

type Tree[T] =
    | Leaf(T)
    | Branch { left: Tree[T], right: Tree[T] }
```

### Construction

```
pair := Pair { first: 42, second: "hello" }       // Pair[i64, str]
stack := Stack[i64] { items: [] }                  // explicit type parameter
tree := Tree.Branch {
    left: Tree.Leaf(1),
    right: Tree.Leaf(2),
}
```

### Standard library generic types

These are the most commonly used generic types, built into the language (Tier 0):

| Type | Definition | Sugar |
|---|---|---|
| `Option[T]` | `Some(T) \| None` | `T?` |
| `Result[T, E]` | `Ok(T) \| Err(E)` | `T ! E` in signatures |
| `[T]` | List/array of `T` | Built-in syntax |
| `Map[K, V]` | Hash map | `{K: V}` literal syntax |
| `Set[T]` | Hash set | `{T}` literal syntax |
| `chan[T]` | Typed channel | Built-in concurrency primitive |

---

## 4. Generic Traits

Traits can be parameterized over types:

```
trait From[T] {
    fn from(value: T) -> Self
}

trait Container[T] {
    fn insert(self, item: T) -> Self
    fn contains(self, item: T) -> bool where T: Eq
    fn size(self) -> i64
    fn is_empty(self) -> bool = self.size() == 0
}

trait Converter[Source, Target] {
    fn convert(source: Source) -> Target ! ConversionError
}
```

### Associated types vs type parameters

Associated types and type parameters serve different purposes. See [spec/trait-system.md](trait-system.md) section 7 for the detailed rules.

```
// Associated type: one Item per Iterator implementation
trait Iterator {
    type Item
    fn next(mut self) -> self.Item?
}

// Type parameter: converts FROM any type T
trait From[T] {
    fn from(value: T) -> Self
}
```

The rule of thumb: if the type is determined by the implementor, use an associated type. If the type is chosen by the caller or there are multiple valid implementations, use a type parameter.

---

## 5. Generic Impl Blocks

### Implementing for a generic type

```
impl[T] Stack[T] {
    fn new() -> Stack[T] = Stack { items: [] }

    fn push(self, item: T) -> Stack[T] =
        Stack { items: self.items.append(item) }

    fn pop(self) -> (Stack[T], T?) {
        if self.items.len() == 0 {
            (self, None)
        } else {
            n := self.items.len()
            (Stack { items: self.items.take(n - 1) }, Some(self.items[n - 1]))
        }
    }

    fn peek(self) -> T? {
        if self.items.len() == 0 { None }
        else { Some(self.items[self.items.len() - 1]) }
    }

    fn len(self) -> i64 = self.items.len()
}
```

### Implementing a trait for a generic type

```
impl[A, B] Display for Pair[A, B] where A: Display, B: Display {
    fn display(self) -> str = "({self.first.display()}, {self.second.display()})"
}

impl[T: Display] Display for Stack[T] {
    fn display(self) -> str {
        items_str := self.items.map(fn(x) => x.display()).join(", ")
        "Stack[{items_str}]"
    }
}
```

### Conditional implementations

Implement a trait only when type parameters satisfy additional bounds:

```
impl[T: Eq] Eq for Stack[T] {
    fn eq(self, other: Stack[T]) -> bool = self.items == other.items
}

impl[T: Hash] Hash for Stack[T] {
    fn hash(self, hasher: mut ref Hasher) {
        for item in self.items { item.hash(hasher) }
    }
}

impl[T: Clone] Clone for Stack[T] {
    fn clone(self) -> Stack[T] = Stack { items: self.items.clone() }
}
```

**AI rationale:** Conditional implementations let the AI write generic containers once, with capabilities that scale based on the element type. A `Stack[i64]` gets `Eq` and `Hash` automatically because `i64` implements them; a `Stack[MyType]` gets them only if `MyType` does. The AI doesn't need to generate separate implementations.

---

## 6. Trait Bounds and Constraints

### Single bound

```
fn show[T: Display](item: T) {
    println(item.display())
}
```

### Multiple bounds

```
fn process[T: Display + Validate](item: T) -> bool ! ValidationError {
    item.validate()?
    println("Valid: {item.display()}")
    true
}

fn sorted_unique[T: Ord + Eq + Hash](items: [T]) -> [T] {
    items.toSet().to[[T]]().sort()
}
```

### Bounds on multiple parameters

```
fn merge[K: Ord, V: Clone](a: Map[K, V], b: Map[K, V]) -> Map[K, V] {
    mut result := a.clone()
    for (k, v) in b { result = result.set(k, v.clone()) }
    result
}
```

### No bounds (unconstrained)

```
fn pair[T, U](a: T, b: U) -> (T, U) = (a, b)
fn wrap[T](value: T) -> [T] = [value]
```

Unconstrained type parameters can only be stored, moved, and returned — no methods can be called on them.

---

## 7. Where Clauses

For complex bounds that would make the function signature hard to read, use a `where` clause:

```
fn merge_sorted[T](a: [T], b: [T]) -> [T]
    where T: Ord + Clone
{
    // ...
}

fn serialize_entries[K, V](entries: [(K, V)]) -> str
    where K: Display + Ord,
          V: Serialize + Debug
{
    // ...
}
```

### Where clauses on impl blocks

```
impl[K, V] Display for Cache[K, V]
    where K: Display + Hash + Eq,
          V: Display
{
    fn display(self) -> str { ... }
}
```

### Where clauses on trait methods

```
trait Container[T] {
    fn insert(self, item: T) -> Self
    fn contains(self, item: T) -> bool where T: Eq
}
```

**AI rationale:** `where` clauses put complex bounds in a predictable location, separate from the parameter list. This makes signatures easier for the AI to parse and generate — the parameter list stays clean, and bounds are always on the line after the return type.

---

## 8. Type Inference

Aria infers type parameters from usage whenever possible. Explicit type annotations are required only when inference is ambiguous.

### Inference from arguments

```
// T is inferred from the argument type
largest([1, 2, 3])                    // T = i64
largest(["a", "b", "c"])              // T = str
map(users, fn(u) => u.name)          // T = User, U = str
```

### Inference from return context

```
// T inferred from the variable's type annotation
x: Option[str] = None                 // T = str
result: Result[Config, IoError] = Ok(config)  // T = Config, E = IoError
```

### When explicit annotation is required

```
// Ambiguous — parser can't determine T from None alone
x := None                              // ❌ compile error: cannot infer type
x: Option[str] = None                  // ✅ annotated

// Ambiguous — parse target type not inferrable
parsed := json.parse(content)?         // ❌ what type to parse into?
parsed := json.parse[Config](content)? // ✅ explicit type parameter

// Method with type parameter
id := 42.to[UserId]()?                 // explicit: which type to convert to?
n := big.trunc[u8]()                   // explicit: which type to truncate to?
```

### Inference rules

1. **Argument types propagate inward** — the type of a function argument constrains its type parameter
2. **Return context propagates outward** — the expected type at the call site constrains the return type parameter
3. **Bounds constrain but don't determine** — `[T: Numeric]` narrows the set of valid types but doesn't pick one
4. **Ambiguity is a compile error** — the compiler never guesses; it requires an annotation

**AI rationale:** Type inference means the AI generates fewer type annotations in typical code, reducing tokens. But when inference fails, the compiler error is clear and specific — "cannot infer type parameter T; add explicit annotation" — so the AI can fix it mechanically.

---

## 9. Monomorphization

Aria generics are monomorphized: the compiler generates specialized machine code for each concrete type used with a generic function or type.

### How it works

```
fn max[T: Ord](a: T, b: T) -> T = if a > b { a } else { b }

// Usage
x := max(3, 5)           // compiler generates: fn max_i64(a: i64, b: i64) -> i64
y := max(3.14, 2.72)     // compiler generates: fn max_f64(a: f64, b: f64) -> f64
z := max("hello", "world")  // compiler generates: fn max_str(a: str, b: str) -> str
```

### Properties

- **Zero runtime overhead** — generic code runs at exactly the same speed as hand-specialized code
- **No vtable, no boxing** — values are stored and passed by their concrete type
- **Larger binary size** — each specialization is a separate copy of the code
- **Compile-time only** — generics are fully resolved before code generation

### Trade-off: binary size vs runtime performance

Monomorphization trades binary size for runtime speed. For most programs, the binary size increase is negligible. For programs that use many generic types with many concrete type arguments, the increase can be significant.

The compiler may apply deduplication: if two specializations generate identical machine code (e.g., `max[i64]` and `max[u64]` on a platform where both are 64-bit integers with the same comparison semantics), the compiler may merge them.

**AI rationale:** Monomorphization means the AI never needs to worry about boxing or vtable overhead when using generics. Generic code is as fast as concrete code — the AI can use generics freely without performance concerns.

---

## 10. Parser Disambiguation

The `[` character serves dual duty in Aria: it begins generic parameters (after a type/function name) and it begins array literals/types (in expression/type position).

### Disambiguation rules

The parser resolves `[` based on the preceding token:

| Context | `[` means | Example |
|---|---|---|
| After `fn name` | Generic parameters | `fn map[T, U](...)` |
| After `type Name` | Generic parameters | `type Result[T, E] = ...` |
| After `trait Name` | Generic parameters | `trait Container[T]` |
| After `impl` | Generic parameters | `impl[T] Stack[T]` |
| After a type name in type position | Generic arguments | `Stack[i64]`, `Map[str, i64]` |
| After `.to` or `.trunc` | Generic arguments | `x.to[i32]()` |
| At the start of an expression | Array literal | `[1, 2, 3]` |
| In type position without preceding name | Array type | `[T]` means "array of T" |

### No ambiguity

Because Aria uses `[` for both generics and arrays, the parser must determine which is intended. The rule is context-dependent but unambiguous:

```
// These are never confused:
fn map[T, U](list: [T], f: fn(T) -> U) -> [U]
//     ^^^^  generic params
//                  ^^^               ^^^
//                  array type        array type

stack := Stack[i64].new()
//            ^^^^  generic argument

items := [1, 2, 3]
//       ^^^^^^^^^  array literal
```

The key insight: `[` is generic when it follows an identifier that names a type, function, trait, or impl. It is an array when it begins an expression or appears in type position without a preceding name.

**AI rationale:** Despite dual use, the disambiguation is mechanical and unambiguous. The AI can determine the meaning of `[` from the immediately preceding token — no lookahead beyond one token is needed.

---

## 11. Phantom Types

Phantom types are type parameters that appear in a type's generic signature but are not used in any field. They exist solely to make the type system distinguish between otherwise identical types.

```
// Unit markers — types with no fields, used only as phantom parameters
type Meters
type Feet
type Seconds

// Generic measurement — the Unit parameter is phantom
struct Measurement[Unit] {
    value: f64
}

impl[U] Measurement[U] {
    fn new(value: f64) -> Measurement[U] = Measurement { value }
    fn value(self) -> f64 = self.value
}

// Type-safe: meters + meters works, meters + feet is a compile error
fn add_meters(a: Measurement[Meters], b: Measurement[Meters]) -> Measurement[Meters] {
    Measurement { value: a.value + b.value }
}

// Explicit conversion required
fn feet_to_meters(feet: Measurement[Feet]) -> Measurement[Meters] {
    Measurement { value: feet.value * 0.3048 }
}

// This would be a compile error:
// add_meters(dist_meters, dist_feet)   // ❌ Measurement[Meters] + Measurement[Feet]
```

**AI rationale:** Phantom types give the AI dimensional analysis for free. When the AI generates physics, finance, or measurement code, the type system prevents unit-mixing bugs at compile time — no runtime overhead, no runtime checks needed.

---

## 12. Design Rationale Summary

| Decision | Rationale |
|---|---|
| `[T]` brackets for generics | Eliminates `<`/`>` parsing ambiguity and the turbofish problem entirely |
| Monomorphization | Zero runtime overhead — generic code is as fast as hand-specialized code |
| Type inference from arguments | Reduces token count by eliminating redundant type annotations |
| Explicit annotation when ambiguous | Compiler never guesses — prevents subtle type inference bugs |
| `where` clauses for complex bounds | Keeps function signatures readable; bounds in a predictable location |
| Phantom types | Compile-time dimensional analysis with zero runtime cost |
| Conditional impl blocks | Capabilities scale with element type — write once, get Eq/Hash/Clone where available |
| No higher-kinded types (v0.1) | Simplicity — HKT adds complexity that the AI rarely needs; can be added later |

---

## 13. Comparison with Other Languages

| Feature | Go (1.18+) | Rust | Java | TypeScript | Aria |
|---|---|---|---|---|---|
| Syntax | `[T Type]` | `<T: Trait>` | `<T extends I>` | `<T extends I>` | `[T: Trait]` |
| Implementation | Monomorphization + GCShape | Monomorphization | Type erasure | Type erasure | Monomorphization |
| Runtime overhead | Minimal (GCShape) | Zero | Boxing overhead | None (erased) | **Zero** |
| Inference | Limited | Good | Good | Good | **Good** |
| Associated types | No | Yes | No | No | **Yes** |
| Where clauses | No | Yes | No | No | **Yes** |
| Phantom types | No | Yes (`PhantomData`) | No | No | **Yes (no marker needed)** |
| Parser ambiguity | None (`[T]`) | Turbofish (`::< >`) | None (different context) | None | **None (`[T]`)** |

### Token cost: generic function with bounds

| Language | Code | Tokens |
|---|---|---|
| Go | `func Max[T constraints.Ordered](a, b T) T { if a > b { return a }; return b }` | ~20 |
| Rust | `fn max<T: Ord>(a: T, b: T) -> T { if a > b { a } else { b } }` | ~18 |
| Java | `public static <T extends Comparable<T>> T max(T a, T b) { return a.compareTo(b) > 0 ? a : b; }` | ~25 |
| Aria | `fn max[T: Ord](a: T, b: T) -> T = if a > b { a } else { b }` | **~15** |

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/trait-system.md](trait-system.md), [spec/formal-grammar.md](formal-grammar.md), and [spec/iteration-protocol.md](iteration-protocol.md).*
