# Aria Equality, Comparison, and Hashing Specification

**A complete specification for structural equality, ordering, hashing, and their interaction with the type system.**

This document formalizes the semantics of `==`, `!=`, `<`, `>`, `<=`, `>=` operators and the traits that power them. These traits are among the most frequently used in any program — getting them right eliminates an entire class of AI-generated bugs around identity confusion, accidental reference comparison, and hash-equality inconsistency.

Cross-references:
- [spec/trait-system.md](trait-system.md) — `Eq`, `Ord`, `Hash` trait declarations
- [spec/type-conversions.md](type-conversions.md) — no implicit conversions (relevant to comparison)
- [spec/generics-type-parameters.md](generics-type-parameters.md) — trait bounds using `Eq`, `Hash`
- [spec/stdlib-design.md](stdlib-design.md) — `Map[K, V]`, `Set[T]` require `Hash + Eq`
- [high-level-design.md](../high-level-design.md) — operator usage in examples

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [The `Eq` Trait — Structural Equality](#2-the-eq-trait)
3. [The `Ord` Trait — Total Ordering](#3-the-ord-trait)
4. [The `Hash` Trait](#4-the-hash-trait)
5. [Floating-Point Equality](#5-floating-point-equality)
6. [Equality on Sum Types](#6-equality-on-sum-types)
7. [Deriving Equality, Ordering, and Hashing](#7-deriving)
8. [Collections and Trait Requirements](#8-collections-and-trait-requirements)
9. [Design Rationale Summary](#9-design-rationale-summary)
10. [Comparison with Other Languages](#10-comparison-with-other-languages)

---

## 1. Design Philosophy

Equality is deceptively simple. In Java, `==` on objects is reference equality — a trap that produces bugs in every beginner's code and every AI's output. In JavaScript, `==` performs type coercion — `0 == ""` is `true`. In Go, `==` works on some types but panics at runtime on others (maps, slices).

Aria eliminates all of these traps with one rule: **`==` is always structural equality, and it only compiles if the type implements `Eq`.**

**AI rationale:** When I generate `a == b`, I mean "are these values the same?" — structural equality. I never mean "are these the same pointer?" or "can these be coerced to look equal?" Reference equality is a concept that exists for implementation reasons, not semantic ones. Aria removes it from the language entirely.

---

## 2. The `Eq` Trait

### Definition

```
trait Eq {
    fn eq(self, other: Self) -> bool
}
```

The `==` operator desugars to `a.eq(b)`. The `!=` operator desugars to `!a.eq(b)`.

### Rules

1. **`Eq` is not automatic** — types must derive or implement it explicitly
2. **`==` on a type that doesn't implement `Eq` is a compile error**
3. **`Eq` must be an equivalence relation**: reflexive (`a == a`), symmetric (`a == b` implies `b == a`), transitive (`a == b` and `b == c` implies `a == c`)
4. **No reference equality exists in the language** — there is no way to ask "are these the same object in memory?"

### Implementing Eq

```
type Point { x: f64, y: f64 }

// Manual implementation
impl Eq for Point {
    fn eq(self, other: Point) -> bool {
        self.x.approxEq(other.x, 1e-10) && self.y.approxEq(other.y, 1e-10)
    }
}

// Or derive for exact field-by-field comparison
type UserId { value: i64 } derives [Eq]
```

### Derived Eq semantics

When `Eq` is derived, the generated implementation compares fields in **declaration order**:

```
type User {
    name: str
    email: str
    age: u8
} derives [Eq]

// Generated equivalent:
// fn eq(self, other: User) -> bool =
//     self.name == other.name
//     && self.email == other.email
//     && self.age == other.age
```

### Compile error on missing Eq

```
type FileHandle { fd: i64 }
// No Eq derived or implemented

a := FileHandle{fd: 3}
b := FileHandle{fd: 3}
a == b    // ❌ compile error: FileHandle does not implement Eq
```

```
error: binary operation `==` requires trait `Eq`
  --> src/main.aria:5:3
  |
5 | a == b
  | ^^^^^^ FileHandle does not implement Eq
  |
  = help: add `derives [Eq]` to the type declaration
          or implement `impl Eq for FileHandle { ... }`
```

**AI rationale:** Requiring explicit `Eq` means I never accidentally compare things that shouldn't be compared — file handles, database connections, channels. If comparison compiles, it's meaningful.

---

## 3. The `Ord` Trait

### Definition

```
type Ordering = Less | Equal | Greater

trait Ord: Eq {
    fn cmp(self, other: Self) -> Ordering
}
```

`Ord` is a **supertrait of `Eq`** — implementing `Ord` requires implementing `Eq`. This ensures that `a.cmp(b) == Equal` is consistent with `a == b`.

### Operator desugaring

| Operator | Desugars to |
|---|---|
| `a < b` | `a.cmp(b) == Less` |
| `a > b` | `a.cmp(b) == Greater` |
| `a <= b` | `a.cmp(b) != Greater` |
| `a >= b` | `a.cmp(b) != Less` |

### Derived Ord semantics

Derived `Ord` compares fields in declaration order (lexicographic):

```
type Version {
    major: u32
    minor: u32
    patch: u32
} derives [Eq, Ord]

// Version{1, 2, 0} < Version{1, 3, 0}    // true (minor differs)
// Version{2, 0, 0} > Version{1, 9, 9}    // true (major differs)
```

### Types that are Eq but not Ord

Not everything has a natural ordering. Complex numbers, colors, user profiles — these can be equal but not ordered:

```
type Color { r: u8, g: u8, b: u8 } derives [Eq, Hash]
// No Ord — what would Color.Red < Color.Blue mean?

type ComplexNumber { real: f64, imag: f64 }
// No Eq (contains f64) — no Ord either
```

**AI rationale:** Separating `Eq` and `Ord` prevents me from generating nonsensical comparisons like `user1 < user2`. If a type has `Ord`, ordering is semantically meaningful. If it doesn't, the compiler tells me.

---

## 4. The `Hash` Trait

### Definition

```
trait Hash: Eq {
    fn hash(self, hasher: mut ref Hasher)
}
```

`Hash` is a **supertrait of `Eq`**. This enforces the critical invariant: **if `a == b`, then `hash(a) == hash(b)`**.

### Why Hash requires Eq

If two values are equal, they must hash to the same value. Otherwise, maps and sets break silently — you insert a key, look it up with an equal key, and get nothing. By making `Hash: Eq`, the compiler guarantees this consistency: you cannot implement `Hash` without implementing `Eq`.

### Derived Hash

```
type User {
    name: str
    email: str
} derives [Eq, Hash]

// Generated: hashes name, then email, combining into a single hash value
```

### Map and Set requirements

```
// Map keys must be Hash + Eq
config: Map[str, i64] = {"timeout": 30, "retries": 3}    // ✅ str is Hash + Eq

// Set elements must be Hash + Eq
ids: Set[UserId] = {UserId{1}, UserId{2}}                 // ✅ UserId derives Hash + Eq

// This fails at compile time
positions: Set[Point] = {Point{1.0, 2.0}}
// ❌ compile error: Point does not implement Hash
```

---

## 5. Floating-Point Equality

Floating-point numbers (`f32`, `f64`) do **not** implement `Eq` or `Hash`. This is a deliberate decision.

### Why

IEEE 754 defines `NaN != NaN`. This violates the reflexivity requirement of `Eq` (`a == a` must be `true`). Rather than introducing a `PartialEq` trait that weakens the contract (as Rust does), Aria takes a simpler approach: floats are not equatable.

### How to compare floats

```
// Approximate equality — the correct approach for floats
a: f64 = 0.1 + 0.2
b: f64 = 0.3
a.approxEq(b, epsilon: 1e-10)    // true

// Exact bit equality (rarely what you want, but available)
a.bitEq(b)                        // compares IEEE 754 bit representation
```

### Float methods

```
impl f64 {
    // Approximate equality within epsilon
    fn approxEq(self, other: f64, epsilon: f64 = 1e-10) -> bool {
        (self - other).abs() < epsilon
    }

    // Exact bit-level equality (NaN.bitEq(NaN) is true)
    fn bitEq(self, other: f64) -> bool

    // Classification
    fn isNan(self) -> bool
    fn isInfinite(self) -> bool
    fn isFinite(self) -> bool
}
```

### Floats as map keys — compile error

```
scores: Map[f64, str] = {3.14: "pi"}
// ❌ compile error: f64 does not implement Hash
//    help: use a newtype with a custom Hash implementation,
//          or use an integer key (multiply by precision factor)
```

### Ordered comparison for floats

Floats **do** support `<`, `>`, `<=`, `>=` via a built-in comparison that follows IEEE 754 ordering rules. However, they do not implement the `Ord` trait (because `Ord` requires `Eq`). The comparison operators on floats are built-in — they do not go through the `Ord` trait.

```
3.14 < 2.72     // false — built-in comparison, not Ord trait
3.14 > 2.72     // true
```

NaN comparisons always return `false`:

```
nan := f64.nan()
nan < 1.0       // false
nan > 1.0       // false
nan == 1.0      // ❌ compile error: f64 does not implement Eq
```

**AI rationale:** The "no Eq for floats" rule eliminates an entire class of bugs I generate: `if price == 0.0` (unsafe — floating point), `map[f64]` (broken with NaN keys). By forcing me to use `approxEq` for floats, the compiler makes me write correct numerical code. This is better than Rust's approach of having `PartialEq` — I never have to remember which equality trait to use.

---

## 6. Equality on Sum Types

Sum type equality compares the **variant tag first, then the fields**:

```
type Shape =
    | Circle(f64)
    | Rect(f64, f64)
    | Point

// With Eq derived (requires all field types to implement Eq)
// BUT: f64 doesn't implement Eq, so this doesn't compile:
// derives [Eq] for Shape  // ❌ compile error: f64 does not implement Eq

// For sum types with non-float fields, it works:
type Command =
    | Move { x: i64, y: i64 }
    | Print { message: str }
    | Quit
derives [Eq] for Command

Move{x: 1, y: 2} == Move{x: 1, y: 2}    // true
Move{x: 1, y: 2} == Move{x: 3, y: 4}    // false
Move{x: 1, y: 2} == Print{message: "hi"} // false (different variants)
Move{x: 1, y: 2} == Quit                  // false (different variants)
```

### Option and Result equality

`Option[T]` implements `Eq` when `T: Eq`:

```
Some(5) == Some(5)     // true
Some(5) == Some(6)     // false
Some(5) == None        // false
None == None           // true (both are the None variant)
```

`Result[T, E]` implements `Eq` when both `T: Eq` and `E: Eq`:

```
Ok(42) == Ok(42)       // true
Ok(42) == Err("fail")  // false
Err("a") == Err("a")   // true
```

---

## 7. Deriving

### Derivation requirements

| Trait | Requires |
|---|---|
| `derives [Eq]` | All fields implement `Eq` |
| `derives [Ord]` | All fields implement `Ord` (implies `Eq`) |
| `derives [Hash]` | All fields implement `Hash` (implies `Eq`) |

### Derivation generates field-by-field comparison in declaration order

```
type Record {
    id: i64
    name: str
    active: bool
} derives [Eq, Ord, Hash]

// Eq: compares id, then name, then active
// Ord: lexicographic on (id, name, active)
// Hash: hashes id, name, active in sequence
```

### Cannot derive when fields don't qualify

```
type Measurement {
    value: f64
    unit: str
} derives [Eq]
// ❌ compile error: cannot derive Eq for Measurement
//    field `value` has type `f64` which does not implement `Eq`
//    help: implement Eq manually using approxEq for the f64 field
```

---

## 8. Collections and Trait Requirements

| Collection | Key/Element requirement | Why |
|---|---|---|
| `[T]` (list) | None | Lists don't need equality or hashing |
| `Set[T]` | `T: Eq + Hash` | Sets use hash tables for O(1) lookup |
| `Map[K, V]` | `K: Eq + Hash` | Map keys use hash tables |
| `[T].contains(x)` | `T: Eq` | Linear scan with equality check |
| `[T].sort()` | `T: Ord` | Sorting requires ordering |
| `[T].dedup()` | `T: Eq` | Deduplication requires equality |
| `[T].unique()` | `T: Eq + Hash` | Hash-based deduplication |

```
// These compile:
users.contains(targetUser)      // ✅ if User: Eq
numbers.sort()                   // ✅ if i64: Ord (it is)
names.unique()                   // ✅ if str: Eq + Hash (it is)

// These don't:
floats.unique()                  // ❌ f64 doesn't implement Hash
handles.contains(myHandle)       // ❌ FileHandle doesn't implement Eq
```

**AI rationale:** When I generate `items.contains(x)`, the compiler immediately tells me whether that type supports the operation. No runtime surprise, no silent wrong behavior. The trait bound is the contract.

---

## 9. Design Rationale Summary

| Decision | Rationale |
|---|---|
| `==` is always structural equality | No reference equality bugs — the most common equality mistake in Java/JS |
| `Eq` must be derived or implemented | Prevents comparing types where equality is meaningless |
| No `PartialEq` — floats don't implement `Eq` | Simpler model; forces correct float comparison with `approxEq` |
| `Hash` requires `Eq` (supertrait) | Enforces hash-equality consistency — prevents broken maps/sets |
| `Ord` requires `Eq` (supertrait) | `a.cmp(b) == Equal` is consistent with `a == b` |
| `Ord` is separate from `Eq` | Not everything equatable is orderable (colors, complex numbers) |
| Derived comparison uses field declaration order | Predictable, deterministic, matches how the AI reads the type |
| Sum type `==` compares variant + fields | `Some(5) == Some(5)` is true — the only sensible behavior |
| Float comparison operators are built-in, not via `Ord` | Allows `<`/`>` on floats without pretending they have total ordering |

---

## 10. Comparison with Other Languages

| Feature | Go | Rust | Java | JavaScript | Aria |
|---|---|---|---|---|---|
| `==` meaning | Structural (some types) | Structural (`PartialEq`) | Reference (objects) | Coercing | **Structural (always)** |
| Compile-time safety | Partial (panics on maps) | Full | None (reference trap) | None | **Full** |
| Float equality | `==` (silent NaN issues) | `PartialEq` (NaN != NaN) | `==` (autoboxing trap) | `==` (coercion) | **No `==` for floats** |
| Hash-Eq consistency | Not enforced | Not enforced by compiler | Contract (honor system) | N/A | **Enforced (supertrait)** |
| Custom equality | Interface methods | `impl PartialEq` | `.equals()` override | N/A | **`impl Eq`** |
| Auto-derive | None | `#[derive(PartialEq)]` | None | N/A | **`derives [Eq]`** |

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/trait-system.md](trait-system.md), [spec/generics-type-parameters.md](generics-type-parameters.md), and [spec/stdlib-design.md](stdlib-design.md).*
