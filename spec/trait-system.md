# Aria Trait System Specification

**A complete specification for trait declarations, implementations, bounds, derives, and the built-in trait catalog in Aria.**

This document formalizes the trait system sketched in [high-level-design.md](../high-level-design.md) (Traits/Interfaces section) and referenced throughout the spec directory. Traits are Aria's sole mechanism for polymorphism — there are no classes, no inheritance, no interfaces-by-structural-conformance.

Cross-references:
- [high-level-design.md](../high-level-design.md) — trait syntax overview, `impl` blocks, `derives`
- [spec/generics-type-parameters.md](generics-type-parameters.md) — trait bounds on type parameters
- [spec/iteration-protocol.md](iteration-protocol.md) — `Iterable` and `Iterator` traits
- [spec/type-conversions.md](type-conversions.md) — `Convert` and `TryConvert` traits
- [spec/error-handling.md](error-handling.md) — `From`, `Transient`, `Retryable` traits
- [spec/concurrency-design.md](concurrency-design.md) — `Send` and `Share` marker traits
- [spec/memory-management.md](memory-management.md) — `Drop` and `Clone` traits
- [spec/closures-capture-semantics.md](closures-capture-semantics.md) — closure types and trait bounds

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Trait Declarations](#2-trait-declarations)
3. [Trait Implementations (`impl Trait for Type`)](#3-trait-implementations)
4. [Inherent Implementations (`impl Type`)](#4-inherent-implementations)
5. [Self Receivers](#5-self-receivers)
6. [Trait Bounds](#6-trait-bounds)
7. [Associated Types](#7-associated-types)
8. [Supertraits](#8-supertraits)
9. [Marker Traits](#9-marker-traits)
10. [The `derives` Mechanism](#10-the-derives-mechanism)
11. [Orphan Rules](#11-orphan-rules)
12. [Trait Objects (Dynamic Dispatch)](#12-trait-objects-dynamic-dispatch)
13. [Built-in Trait Catalog](#13-built-in-trait-catalog)
14. [Design Rationale Summary](#14-design-rationale-summary)
15. [Comparison with Other Languages](#15-comparison-with-other-languages)

---

## 1. Design Philosophy

Traits are Aria's answer to the question: how do you get polymorphism without inheritance? The answer is explicit interface contracts that types opt into, with no hierarchy, no method resolution order, and no diamond problem.

### Why Traits, Not Interfaces or Classes

In Java, an interface is a contract that classes implement. But classes also have inheritance, constructors, and mutable state — they are too many concepts in one. In Go, interfaces are satisfied implicitly — any type that happens to have the right methods satisfies the interface, even accidentally. Both models create problems for AI code generation.

**AI rationale:** Explicit `impl Trait for Type` blocks give the AI a searchable, unambiguous mapping from types to capabilities. When generating code that requires a type to be `Eq + Hash`, the AI can check whether `impl Eq for MyType` exists — no guessing, no accidental conformance, no inheritance chain to trace. This reduces a class of "I thought this type had that method" bugs to zero.

### The Core Rules

1. **Traits define capabilities** — a set of method signatures that a type can implement
2. **Implementation is always explicit** — `impl Trait for Type { ... }`
3. **No inheritance between types** — traits provide polymorphism without hierarchy
4. **Supertraits are allowed** — `trait B: A` means implementing `B` requires implementing `A`
5. **Default implementations are allowed** — a trait can provide a default method body
6. **Traits are the only generic constraint** — `[T: Trait]` is how you constrain type parameters
7. **Derives auto-generate implementations** — `derives [Eq, Hash]` generates boilerplate mechanically

---

## 2. Trait Declarations

A trait declares a set of methods that types can implement. Traits may contain required methods (no body) and default methods (with body).

### Basic trait declaration

```
trait Display {
    fn display(self) -> str
}

trait Validate {
    fn validate(self) -> bool ! ValidationError
}
```

Each method in a trait must have `self` as its first parameter (see [section 5](#5-self-receivers) for receiver variants).

### Traits with default methods

Default methods provide an implementation that types can use without overriding:

```
trait Describe {
    fn name(self) -> str

    // Default implementation — types can override this
    fn describe(self) -> str = "I am a {self.name()}"
}
```

Default methods may call other methods in the same trait. A type implementing the trait can override any default method by providing its own implementation.

```
impl Describe for Circle {
    fn name(self) -> str = "circle"
    // uses the default describe() — no need to implement it
}

impl Describe for Rectangle {
    fn name(self) -> str = "rectangle"

    // Override the default
    fn describe(self) -> str {
        if self.is_square() { "I am a square" }
        else { "I am a rectangle ({self.width}×{self.height})" }
    }
}
```

**AI rationale:** Default methods reduce boilerplate without inheritance. The AI generates fewer method bodies when reasonable defaults exist, and the relationship is explicit — no hidden method resolution order to trace.

### Generic traits

Traits can be parameterized over types:

```
trait Container[T] {
    fn insert(self, item: T) -> Self
    fn contains(self, item: T) -> bool where T: Eq
    fn size(self) -> i64
    fn is_empty(self) -> bool = self.size() == 0
}

trait From[T] {
    fn from(value: T) -> Self
}

trait Iterator[T] {
    fn next(mut self) -> T?
}
```

### Traits with effects

Trait methods can declare effects:

```
trait Serializable {
    fn serialize(self) -> [byte]
    fn deserialize(data: [byte]) -> Self ! DecodeError
}

trait DataSource {
    fn fetch(self, query: str) -> [byte] ! IoError with [Io]
}
```

### Grammar

```
trait_decl = [ "pub" ] "trait" IDENT [ generic_params ] [ ":" trait_bound ]
             "{" { trait_method } "}" ;

trait_method = "fn" IDENT [ generic_params ]
               "(" [ param_list ] ")" [ "->" type ] [ error_clause ] [ effect_clause ]
               [ "=" expression | block ] ;
```

---

## 3. Trait Implementations

A trait implementation provides concrete method bodies for a specific type. Every required method must be implemented; default methods may be overridden.

### Basic impl block

```
impl Display for Circle {
    fn display(self) -> str = "Circle(r={self.radius})"
}

impl Area for Circle {
    fn area(self) -> f64 = 3.14159 * self.radius * self.radius
}
```

### Generic impl blocks

When implementing a trait for a generic type, the impl block declares its own type parameters:

```
impl[T] Display for Stack[T] where T: Display {
    fn display(self) -> str {
        items_str := self.items.map(fn(x) => x.display()).join(", ")
        "Stack[{items_str}]"
    }
}

impl[A: Display, B: Display] Display for Pair[A, B] {
    fn display(self) -> str = "({self.first.display()}, {self.second.display()})"
}
```

### Implementing multiple traits

A type can implement any number of traits. Each implementation is a separate `impl` block:

```
impl Area for Rectangle {
    fn area(self) -> f64 = self.width * self.height
}

impl Perimeter for Rectangle {
    fn perimeter(self) -> f64 = 2.0 * (self.width + self.height)
}

impl Display for Rectangle {
    fn display(self) -> str = "Rectangle({self.width}×{self.height})"
}
```

### Blanket implementations

A trait can be implemented for all types that satisfy a bound:

```
impl[T: Display] Describe for T {
    fn name(self) -> str = self.display()
}
```

Blanket implementations are powerful but must not conflict with specific implementations. The compiler rejects overlapping implementations.

### Grammar

```
impl_decl = "impl" [ generic_params ] IDENT "for" type [ where_clause ]
            "{" { fn_decl } "}" ;
```

---

## 4. Inherent Implementations

Inherent `impl` blocks attach methods directly to a type without a trait. These are the type's "own" methods.

```
impl Circle {
    fn new(radius: f64) -> Circle = Circle { radius }
    fn diameter(self) -> f64 = self.radius * 2.0
    fn scale(self, factor: f64) -> Circle = Circle { radius: self.radius * factor }
}

impl Rectangle {
    fn new(width: f64, height: f64) -> Rectangle = Rectangle { width, height }
    fn is_square(self) -> bool = self.width == self.height
}
```

Inherent methods are called with dot syntax: `circle.diameter()`, `rect.is_square()`.

**Method resolution order:** When a method name exists in both an inherent impl and a trait impl, the inherent method takes priority. The trait method can always be called explicitly with `Trait.method(value)` syntax.

### Generic inherent impl blocks

```
impl[T] Stack[T] {
    fn new() -> Stack[T] = Stack { items: [] }
    fn push(self, item: T) -> Stack[T] = Stack { items: self.items.append(item) }
    fn peek(self) -> T? {
        if self.items.len() == 0 { None }
        else { Some(self.items[self.items.len() - 1]) }
    }
}
```

### Grammar

```
inherent_impl = "impl" [ generic_params ] type [ where_clause ]
                "{" { fn_decl } "}" ;
```

---

## 5. Self Receivers

Every trait method and inherent method takes `self` as its first parameter. The receiver determines how the value is accessed:

| Receiver | Meaning | Use case |
|---|---|---|
| `self` | Takes ownership (moves the value) | Consuming operations, builder patterns |
| `ref self` | Borrows immutably | Read-only access (most common) |
| `mut ref self` | Borrows mutably | In-place mutation |

```
trait Example {
    fn consume(self)             // takes ownership
    fn inspect(ref self) -> str  // read-only borrow
    fn modify(mut ref self)      // mutable borrow
}
```

In practice, the compiler infers the appropriate receiver semantics from usage. When you write `fn area(self) -> f64`, the compiler determines whether the value needs to be moved or can be borrowed. Explicit `ref` and `mut ref` annotations are available when you need to be precise.

**AI rationale:** A single `self` keyword covers the common case. The AI doesn't need to choose between `&self`, `&mut self`, and `self` on every method — the compiler infers the optimal receiver. Explicit annotations are available for performance-critical code but aren't required for correctness.

### Static methods (no receiver)

Methods without `self` are static — they're called on the type, not an instance:

```
impl Circle {
    fn new(radius: f64) -> Circle = Circle { radius }    // static: Circle.new(5.0)
    fn unit() -> Circle = Circle { radius: 1.0 }         // static: Circle.unit()
}
```

Static methods in traits use `Self` as the return type:

```
trait Default {
    fn default() -> Self
}

impl Default for Circle {
    fn default() -> Circle = Circle { radius: 0.0 }
}
```

---

## 6. Trait Bounds

Trait bounds constrain generic type parameters, requiring that the type implement specific traits.

### Inline bounds

```
fn show[T: Display](item: T) {
    println(item.display())
}

fn sum[T: Numeric](list: [T]) -> T {
    list.fold(T.zero, fn(a, b) => a + b)
}
```

### Multiple bounds with `+`

```
fn print_shape[T: Area + Perimeter + Display](shape: T) {
    println("{shape.display()}: area={shape.area()}, perimeter={shape.perimeter()}")
}

fn process[T: Display + Validate](item: T) -> bool ! ValidationError {
    item.validate()?
    println("Valid: {item.display()}")
    true
}
```

### `where` clauses

For complex bounds, use a `where` clause after the parameter list:

```
fn merge_sorted[T](a: [T], b: [T]) -> [T]
    where T: Ord + Clone
{
    // ...
}

fn serialize_map[K, V](map: Map[K, V]) -> str
    where K: Display + Ord,
          V: Serialize
{
    // ...
}
```

### Bounds on impl blocks

```
impl[T: Eq + Hash] Container[T] for HashSet[T] {
    fn insert(self, item: T) -> Self { ... }
    fn contains(self, item: T) -> bool { ... }
    fn size(self) -> i64 { ... }
}
```

**AI rationale:** Trait bounds are the AI's type-level documentation. When the AI sees `[T: Eq + Hash]`, it knows exactly what operations are available on `T` — no guessing, no reading documentation. The constraint is the contract.

---

## 7. Associated Types

Associated types are type members declared inside a trait. They are determined by the implementing type, not by the caller.

### Declaration

```
trait Iterable {
    type Item
    fn iter(self) -> Iterator[self.Item]
}

trait Collection {
    type Item
    fn len(self) -> u64
    fn get(self, idx: u64) -> self.Item?
}
```

### Implementation

```
impl Iterable for [i64] {
    type Item = i64
    fn iter(self) -> Iterator[i64] { ... }
}

impl Collection for [str] {
    type Item = str
    fn len(self) -> u64 { ... }
    fn get(self, idx: u64) -> str? { ... }
}
```

### When to use associated types vs type parameters

| Use associated types when... | Use type parameters when... |
|---|---|
| There is one natural type per implementation | Multiple valid types per implementation |
| The type is determined by the implementor | The type is chosen by the caller |
| Example: `Iterator` has one `Item` type | Example: `From[T]` converts from any `T` |

```
// Associated type — one Item per iterator
trait Iterator {
    type Item
    fn next(mut self) -> self.Item?
}

// Type parameter — converts from any T
trait From[T] {
    fn from(value: T) -> Self
}
```

**AI rationale:** Associated types reduce the number of type parameters the AI must specify at call sites. `iter.next()` doesn't need a type annotation — the `Item` type is fixed by the iterator's implementation.

---

## 8. Supertraits

A supertrait relationship means implementing the child trait requires implementing the parent trait first.

```
trait Retryable: Transient {
    fn retryAfter(self) -> dur?
    fn maxRetries(self) -> u64 = 3
}
```

`Retryable: Transient` means every type that implements `Retryable` must also implement `Transient`. The compiler enforces this — implementing `Retryable` without `Transient` is a compile error.

### Multiple supertraits

```
trait Printable: Display + Debug {
    fn prettyPrint(self)
}
```

### Supertrait method access

Inside a trait with supertraits, methods from the parent traits are available:

```
trait Saveable: Serialize + Validate {
    fn save(self) -> bool ! SaveError with [Io] {
        self.validate()?              // from Validate
        data := self.serialize()      // from Serialize
        io.writeFile("data.bin", data)?
        true
    }
}
```

**AI rationale:** Supertraits encode "is-a" relationships without inheritance. When the AI sees `T: Retryable`, it knows `T` is also `Transient` — no need to specify both bounds.

---

## 9. Marker Traits

Marker traits have no methods. They exist purely to tag types with properties that the compiler can check.

### Built-in marker traits

```
trait Send {}     // Safe to move to another task
trait Share {}    // Safe to share (read) across tasks simultaneously
```

### Auto-derivation

`Send` and `Share` are automatically derived by the compiler for types whose fields are all `Send` or `Share` respectively. You never write `impl Send for MyType {}` manually — the compiler either derives it or rejects the type.

| Type | Send? | Share? |
|---|---|---|
| All primitives (`i64`, `f64`, `bool`, `str`, etc.) | Auto | Auto |
| Structs with all-Send fields | Auto | Depends |
| Immutable types (no `mut` fields) | Auto | Auto |
| `chan[T]` | Auto | Auto |
| `sync.Mutex[T]` | Auto | Auto |
| `sync.Atomic[T]` | Auto | Auto |
| Types with `ref` captures | No | No |
| Raw FFI pointers | No | No |

### Negative implementations

When a type should explicitly not be `Send` or `Share`:

```
type ThreadLocalData {
    ptr: *c.void
} // Not Send — compiler sees raw pointer, won't auto-derive
```

**AI rationale:** Marker traits eliminate an entire class of concurrency bugs. The AI never accidentally sends a non-thread-safe value across a task boundary — the compiler rejects it.

---

## 10. The `derives` Mechanism

`derives` auto-generates trait implementations from the type's structure. It replaces what would be 10-50 lines of boilerplate per trait.

### Syntax

```
type User {
    name: str
    email: str
    age: u8
} derives [Eq, Hash, Debug, Clone, Default]
```

The `derives` clause appears after the type body. It takes a list of trait names.

> **Note on alternative syntax:** Some Aria documentation uses the attribute form `@[derive(Eq, Hash)]`. The canonical form is `derives [Eq, Hash]`; the attribute form may appear in older examples.

### Per-variant derives (sum types)

Sum type variants can have independent derives:

```
type IoError =
    | NotFound { path: str }         derives [Permanent, UserFault]
    | Timeout { after: dur }         derives [Transient, SystemFault]
    | PermissionDenied { path: str } derives [Permanent, UserFault]
```

Whole-type derives apply to all variants:

```
derives [Debug, Eq, Display] for IoError
```

### Derivable traits

| Trait | What it generates | Requirement |
|---|---|---|
| `Eq` | Field-by-field equality comparison | All fields must be `Eq` |
| `Ord` | Field-by-field ordering (lexicographic) | All fields must be `Ord` |
| `Hash` | Field-by-field hash combination | All fields must be `Hash` |
| `Clone` | Field-by-field deep copy | All fields must be `Clone` |
| `Debug` | Debug-format string representation | All fields must be `Debug` |
| `Display` | Human-readable string representation | Custom format or default |
| `Default` | All-defaults construction | All fields must have defaults |
| `Json` | JSON serialization/deserialization | All fields must be `Json` |

### Token cost comparison

```
// Go equivalent of Aria's `derives [Eq, Hash, Debug, Json]`:
// - func (u User) Equal(other User) bool { ... }        (~10 lines)
// - func (u User) Hash() uint64 { ... }                 (~8 lines)
// - func (u User) String() string { ... }                (~5 lines)
// - func (u User) MarshalJSON() ([]byte, error) { ... }  (~15 lines)
// - func (u *User) UnmarshalJSON(data []byte) error { ... } (~20 lines)
// Total: ~58 lines

// Aria: 1 line
type User { name: str, email: str, age: u8 } derives [Eq, Hash, Debug, Json]
```

**AI rationale:** `derives` eliminates the most common boilerplate the AI generates. Instead of producing 58 lines of mechanical code (each a potential bug site), the AI writes one clause. The compiler generates correct implementations guaranteed to stay in sync with the type definition.

---

## 11. Orphan Rules

To prevent conflicting implementations across packages, Aria enforces orphan rules:

**The rule:** You can only implement a trait for a type if at least one of them (the trait or the type) is defined in your package.

```
// OK — you own the type
impl Display for MyType { ... }

// OK — you own the trait
impl MyTrait for i64 { ... }

// Compile error — you own neither Display nor i64
impl Display for i64 { ... }
```

### Newtype pattern

To work around the orphan rule, wrap the foreign type:

```
type MyString { inner: str }

impl MyTrait for MyString {
    fn myMethod(self) -> str = self.inner.toUpper()
}
```

**AI rationale:** Orphan rules prevent the AI from generating conflicting implementations that would cause errors in downstream code. The constraint is simple and unambiguous — the AI can check it mechanically.

---

## 12. Trait Objects (Dynamic Dispatch)

When the concrete type is not known at compile time, trait objects provide dynamic dispatch.

### Syntax

```
fn log_all(items: [dyn Display]) {
    for item in items {
        println(item.display())
    }
}

fn process(handler: dyn Handler) {
    handler.handle()
}
```

`dyn Trait` is a trait object — a value whose concrete type is erased, replaced by a vtable pointer for dynamic method dispatch.

### Limitations

- Trait objects have runtime overhead (vtable indirection)
- Not all traits can be used as trait objects — the trait must be "object-safe":
  - No methods that return `Self`
  - No methods with generic type parameters
  - No associated types used in return position without constraints

### When to use

| Use static dispatch (`[T: Trait]`) when... | Use dynamic dispatch (`dyn Trait`) when... |
|---|---|
| Performance is critical | The concrete type varies at runtime |
| The type is known at compile time | You need heterogeneous collections |
| Monomorphization is acceptable | Binary size is a concern |

```
// Static dispatch — monomorphized, fastest, no runtime overhead
fn show[T: Display](item: T) { println(item.display()) }

// Dynamic dispatch — vtable lookup, runtime flexibility
fn show_any(item: dyn Display) { println(item.display()) }

// Heterogeneous collection — requires dynamic dispatch
items: [dyn Display] = [circle, rect, "hello", 42]
```

**AI rationale:** Static dispatch (`[T: Trait]`) should be the default the AI generates. Dynamic dispatch (`dyn Trait`) is reserved for cases where the type genuinely varies at runtime — heterogeneous collections and plugin systems. The AI can make this decision based on whether the collection contains mixed types.

---

## 13. Built-in Trait Catalog

### Core traits

| Trait | Methods | Purpose |
|---|---|---|
| `Display` | `fn display(self) -> str` | Human-readable string representation |
| `Debug` | `fn debug(self) -> str` | Debug-format string representation |
| `Eq` | `fn eq(self, other: Self) -> bool` | Equality comparison (`==`, `!=`) |
| `Ord` | `fn cmp(self, other: Self) -> Ordering` | Total ordering (`<`, `>`, `<=`, `>=`). Supertrait of `Eq` |
| `Hash` | `fn hash(self, hasher: mut ref Hasher)` | Hash value for use in maps and sets |
| `Clone` | `fn clone(self) -> Self` | Explicit deep copy |
| `Default` | `fn default() -> Self` | Default value construction |

### Conversion traits

| Trait | Methods | Purpose |
|---|---|---|
| `Convert[T]` | `fn convert(self) -> T` | Lossless conversion (`T(x)` syntax) |
| `TryConvert[T]` | `fn tryConvert(self) -> Result[T, ConversionError]` | Checked conversion (`.to[T]()` syntax) |
| `From[T]` | `fn from(value: T) -> Self` | Construct from another type |

### Iteration traits

| Trait | Methods | Purpose |
|---|---|---|
| `Iterable` | `type Item; fn iter(self) -> Iterator[self.Item]` | "I can be iterated" — enables `for` loops |
| `Iterator[T]` | `fn next(mut self) -> T?` | "I produce values one at a time" |

### Concurrency traits (marker)

| Trait | Methods | Purpose |
|---|---|---|
| `Send` | (none) | Safe to move to another task |
| `Share` | (none) | Safe to share across tasks |

### Resource traits

| Trait | Methods | Purpose |
|---|---|---|
| `Drop` | `fn drop(mut ref self)` | Deterministic cleanup when value goes out of scope |

### Error category traits

| Trait | Methods | Purpose |
|---|---|---|
| `Transient` | (none) | Error is temporary — retry may succeed |
| `Permanent` | (none) | Error is final — retry will not help |
| `UserFault` | (none) | Error was caused by caller input |
| `SystemFault` | (none) | Error was caused by environment |
| `Retryable` | `fn retryAfter(self) -> dur?; fn maxRetries(self) -> u64` | Supertrait of `Transient` with retry hints |

### Numeric trait

| Trait | Methods | Purpose |
|---|---|---|
| `Numeric` | Arithmetic operators, `zero`, `one` | Numeric types (`i8`..`i64`, `u8`..`u64`, `f32`, `f64`) |

---

## 14. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Explicit `impl Trait for Type` | No accidental interface conformance — the AI always knows what a type implements |
| No inheritance | Eliminates method resolution order complexity, diamond problem, and "which override am I calling?" bugs |
| Default methods allowed | Reduces boilerplate without inheritance — mechanical code reuse |
| `derives` for auto-generation | 1 line replaces 50+ lines of boilerplate; generated code is always correct |
| Supertraits | Encode "implements A implies implements B" without inheritance |
| Marker traits (Send, Share) | Concurrency safety checked at compile time, zero runtime cost |
| Associated types | Reduce type parameter noise at call sites |
| Orphan rules | Prevent conflicting implementations across packages |
| Static dispatch by default | Monomorphization = zero runtime overhead; dynamic dispatch is opt-in |
| Single `self` receiver | AI doesn't need to choose between `&self`/`&mut self`/`self` — compiler infers |

---

## 15. Comparison with Other Languages

| Feature | Go | Rust | Java | Aria |
|---|---|---|---|---|
| Polymorphism mechanism | Implicit interfaces | Traits | Classes + interfaces | Explicit traits |
| Implementation | Implicit (structural) | Explicit (`impl`) | Explicit (`implements`) | Explicit (`impl`) |
| Inheritance | None (embedding only) | None | Class hierarchy | **None** |
| Default methods | None | `default fn` | `default` methods | Default methods |
| Generic constraints | None (before 1.18) | `where T: Trait` | `<T extends Trait>` | `[T: Trait]` |
| Auto-derive | None | `#[derive(...)]` | None (use IDE) | `derives [...]` |
| Marker traits | None | `Send`, `Sync` | None | `Send`, `Share` |
| Associated types | None | `type Item` | None | `type Item` |
| Dynamic dispatch | Interface values | `dyn Trait` | Interface references | `dyn Trait` |
| Orphan rules | N/A | Enforced | N/A | Enforced |

### Token cost: implementing Display for a type

| Language | Code | Tokens |
|---|---|---|
| Go | `func (u User) String() string { return fmt.Sprintf("User{name: %s}", u.Name) }` | ~20 |
| Rust | `impl fmt::Display for User { fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result { write!(f, "User{{name: {}}}", self.name) } }` | ~30 |
| Java | `@Override public String toString() { return "User{name: " + name + "}"; }` | ~15 |
| Aria (manual) | `impl Display for User { fn display(self) -> str = "User\{name: {self.name}\}" }` | ~15 |
| Aria (derived) | `derives [Display]` on the type | **2** |

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/generics-type-parameters.md](generics-type-parameters.md), [spec/iteration-protocol.md](iteration-protocol.md), and [spec/error-handling.md](error-handling.md).*
