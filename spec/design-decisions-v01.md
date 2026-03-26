# Aria v0.1 Design Decisions

**Formal resolutions for open design questions identified during the pre-implementation review (2026-03-18).**

Each decision in this document was evaluated against Aria's five design pillars and the AI code generation use case. These are binding for v0.1 — they should be treated as spec-level constraints by compiler implementors.

Cross-references:
- [high-level-design.md](../high-level-design.md) — the five design pillars
- [spec/formal-grammar.md](formal-grammar.md) — grammar productions affected by these decisions
- [spec/trait-system.md](trait-system.md) — trait-related decisions
- [spec/memory-management.md](memory-management.md) — allocation-related decisions

---

## Decision 1: `struct` and `type` Keywords

**Resolution:** Both `struct` and `type` can declare struct types. They produce identical results.

```
// These are identical:
struct Point { x: f64, y: f64 }
type Point { x: f64, y: f64 }
```

The `type` keyword is also used for sum types and newtypes. The parser disambiguates by what follows:

| Declaration | Syntax | Disambiguated by |
|---|---|---|
| Struct (via `struct`) | `struct Name { fields }` | `struct` keyword |
| Struct (via `type`) | `type Name { fields }` | `{` directly after name |
| Sum type | `type Name = \| Variant ...` | `\|` after `=` |
| Newtype | `type Name = UnderlyingType` | Single type after `=`, no `\|` |
| Alias | `alias Name = Type` | `alias` keyword |

**Convention:** Use `struct` when declaring a plain data type. Use `type` when declaring sum types or newtypes. Both work for structs, but consistency helps the AI generate predictable code.

**Rationale:** Keeping both avoids a breaking change to existing examples (which all use `struct`) while maintaining `type` as the universal type-declaration keyword. The parser cost is zero — `struct` is unambiguous.

---

## Decision 2: Operator Overloading via Fine-Grained Traits

**Resolution:** Arithmetic and comparison operators map to individual traits. `Numeric` is a convenience supertrait.

### Operator traits

| Operator | Trait | Method |
|---|---|---|
| `a + b` | `Add` | `fn add(self, other: Self) -> Self` |
| `a - b` | `Sub` | `fn sub(self, other: Self) -> Self` |
| `a * b` | `Mul` | `fn mul(self, other: Self) -> Self` |
| `a / b` | `Div` | `fn div(self, other: Self) -> Self` |
| `a % b` | `Mod` | `fn mod(self, other: Self) -> Self` |
| `-a` | `Neg` | `fn neg(self) -> Self` |
| `a == b` | `Eq` | `fn eq(self, other: Self) -> bool` |
| `a < b` etc. | `Ord` | `fn cmp(self, other: Self) -> Ordering` |

### The `Numeric` supertrait

```
trait Numeric: Add + Sub + Mul + Div + Mod + Neg + Eq + Ord {
    fn zero() -> Self
    fn one() -> Self
}
```

All primitive numeric types (`i8`..`i64`, `u8`..`u64`, `f32`, `f64`) implement `Numeric`. User types can implement individual operator traits without implementing all of `Numeric`:

```
type Vector2 { x: f64, y: f64 }

impl Add for Vector2 {
    fn add(self, other: Vector2) -> Vector2 =
        Vector2 { x: self.x + other.x, y: self.y + other.y }
}

// Vector2 supports + but not *, /, etc.
```

### Derivable

For newtypes wrapping numeric types:

```
type Amount = i64 derives [Eq, Ord, Hash, Clone, Debug, Numeric]
// Amount + Amount works
// Amount * Amount works (may or may not make semantic sense — type author's choice)
```

For newtypes where only some operations make sense:

```
type Timestamp = i64 derives [Eq, Ord, Hash, Clone, Debug, Sub]
// Timestamp - Timestamp works (produces a duration-like value)
// Timestamp + Timestamp does NOT work — Add not derived
```

**Rationale:** Fine-grained operator traits let the AI (and the type author) control exactly which operations are available. `Numeric` is the convenience shorthand for "all arithmetic." This aligns with Pillar 5 (no implicit behavior) — every operator is an explicit trait.

**AI rationale:** When I see `[T: Add]`, I know exactly what I can do with `T` — add it. I don't get `*` or `/` for free. When I see `[T: Numeric]`, I get everything. The granularity prevents me from generating nonsensical operations.

---

## Decision 3: `as` Is Only for Import Aliases

**Resolution:** The `as` keyword is used exclusively for import aliases. It is NOT a type cast operator.

```
// ✅ Valid — import alias
use crypto.sha256 as sha

// ❌ NOT valid — use type conversion methods instead
n as u8           // NO — use n.trunc[u8]() or n.to[u8]()?
value as str      // NO — use value.toStr()
item as Display   // NO — use dyn Display for trait objects
```

### Type conversion methods (the only way to convert types)

| Conversion | Syntax | Safety |
|---|---|---|
| Lossless widening | `T(x)` — e.g., `i64(myI32)` | Compile-time proven safe |
| Checked narrowing | `x.to[T]()` — returns `Result` | Runtime checked |
| Explicit truncation | `x.trunc[T]()` — returns `T` | Caller accepts data loss |
| To string | `x.toStr()` | Always safe |
| From string | `"42".parseInt[i64]()` | Returns `Result` |

See [spec/type-conversions.md](type-conversions.md) for the full specification.

### Why not `as` for casts?

1. **`as` is ambiguous about safety.** In Go and Rust, `x as T` can silently truncate data. You can't tell from reading the code whether the conversion is safe.
2. **Aria's three-mechanism system is self-documenting.** `T(x)` = safe, `.to[T]()` = checked, `.trunc[T]()` = lossy. The syntax tells you the safety level.
3. **One meaning per keyword.** `as` means "rename" (import alias). That's it.

**Note:** The `n as u8` usage in `examples/07-enums-and-pattern-matching.aria` was a bug and has been corrected to `n.trunc[u8]()`.

---

## Decision 4: Recursive Types Are Automatically Boxed

**Resolution:** The compiler automatically inserts heap indirection for recursive type fields. No programmer annotation is needed.

```
// This just works — no Box, no pointer annotation
type Tree[T] =
    | Leaf(T)
    | Branch { left: Tree[T], right: Tree[T] }

type LinkedList[T] =
    | Cons { head: T, tail: LinkedList[T] }
    | Nil
```

### How it works

1. The compiler detects that `Branch.left` and `Branch.right` have type `Tree[T]`, which is the type being defined — a recursive reference.
2. The compiler automatically stores recursive fields as GC-managed heap pointers.
3. The programmer never sees or manages this indirection — `branch.left` accesses the `Tree[T]` value directly.
4. For `@stack` and `@arena` allocations, recursive fields are still heap-allocated (in the arena or GC heap as appropriate), since stack allocation of infinite-depth recursive structures is impossible.

### Why automatic?

Requiring `Box` or `*` annotations for recursive types (as Rust does) adds tokens and forces the programmer to think about allocation for a structural property of the type. This violates Pillar 4 (GC by default, manual control opt-in) — the common case should be zero-annotation.

**AI rationale:** I generate recursive data structures — trees, linked lists, ASTs — frequently. Automatic boxing means I write `type Tree[T] = Leaf(T) | Branch { left: Tree[T], right: Tree[T] }` and it works. No `Box`, no `*`, no allocation annotation. The compiler handles it.

---

## Decision 5: Mutability Is on Bindings, Not Fields

**Resolution:** The `mut` keyword applies to variable bindings, not struct fields. A `mut` binding makes all fields of the bound value mutable. An immutable binding makes all fields read-only.

```
// Immutable binding — nothing can be changed
config := Config{host: "localhost", port: 8080}
config.port = 9090          // ❌ compile error: config is immutable

// Mutable binding — all fields can be changed
mut config := Config{host: "localhost", port: 8080}
config.port = 9090          // ✅ OK — config is mutable
config.host = "0.0.0.0"    // ✅ OK — all fields are mutable
```

### No per-field mutability

```
// ❌ This syntax does NOT exist in Aria:
type Counter { mut count: i64, name: str }
```

Mutability is a property of the **binding**, not the **type**. The same `Config` value can be bound immutably in one place and mutably in another.

### Deep immutability

Immutable bindings are **deeply immutable** — you cannot mutate nested fields either:

```
config := Config{server: Server{port: 8080}}
config.server.port = 9090   // ❌ compile error: config is immutable
```

### Mutable method receivers

`mut ref self` in method signatures means the method mutates the value. The caller must have a `mut` binding:

```
impl Counter {
    fn increment(mut ref self) {
        self.count += 1
    }
}

counter := Counter{count: 0}
counter.increment()           // ❌ compile error: cannot call mut method on immutable binding

mut counter := Counter{count: 0}
counter.increment()           // ✅ OK
```

### Why binding-level mutability?

1. **Simpler mental model** — one rule: `mut` = mutable, no `mut` = frozen
2. **Fewer tokens** — no per-field `mut` annotations
3. **Matches the functional default** — immutable by default, mutable by explicit choice
4. **No "partially mutable" confusion** — a value is either mutable or it isn't

**AI rationale:** When I generate code, I decide once whether a binding is mutable. I don't analyze each field separately. `mut config := Config{...}` — I can change anything. `config := Config{...}` — I can't change anything. One decision, applied uniformly.

---

## Decision 6: Method Resolution Order

**Resolution:** Method resolution follows a strict priority order. Ambiguity is always a compile error with a fix suggestion.

### Resolution order

1. **Inherent methods** (from `impl Type { ... }`) — always checked first
2. **Trait methods** — checked second; if exactly one trait in scope provides the method, use it
3. **Ambiguity** — if multiple traits provide the same method name, compile error

### Disambiguation syntax

When two traits define the same method and a type implements both, use qualified call syntax:

```
trait Drawable { fn render(self) -> str }
trait Printable { fn render(self) -> str }

impl Drawable for Widget {
    fn render(self) -> str = "<svg>...</svg>"
}

impl Printable for Widget {
    fn render(self) -> str = "Widget(id={self.id})"
}

widget := Widget{id: 1}

// ❌ Compile error — ambiguous
widget.render()

// ✅ Disambiguate with Trait.method(value) syntax
Drawable.render(widget)     // "<svg>...</svg>"
Printable.render(widget)    // "Widget(id=1)"
```

### Compile error message

```
error[E0205]: ambiguous method call — `render` is defined by multiple traits
  --> src/widget.aria:15:5
   |
15 |     widget.render()
   |            ^^^^^^ both `Drawable` and `Printable` define `render`
   |
   = help: disambiguate with qualified syntax:
           Drawable.render(widget)
           Printable.render(widget)
```

### Inherent methods always win

```
impl Widget {
    fn render(self) -> str = "inherent"
}

impl Drawable for Widget {
    fn render(self) -> str = "drawable"
}

widget.render()              // "inherent" — inherent method wins, no ambiguity
Drawable.render(widget)      // "drawable" — explicit trait call still works
```

**AI rationale:** The common case (no conflict) is zero-overhead — I just call `widget.render()`. When there IS a conflict, the compiler gives me the exact disambiguation syntax to use. I never have to guess which method I'm calling — the resolution is deterministic and the error message tells me how to fix it.

---

## Decision 7: Closures Are GC-Boxed by Default

**Resolution:** Closures are heap-allocated (GC-managed) by default. The compiler may optimize to stack allocation or monomorphization when it can prove it's safe.

### What this means

```
// This closure is a GC-allocated object with a vtable
adder := fn(x: i64) -> i64 => x + 1

// Closures can be stored in collections (same type)
transforms: [fn(i64) -> i64] = [
    fn(x) => x + 1,
    fn(x) => x * 2,
    fn(x) => x - 3,
]

// Closures can be returned from functions
fn makeAdder(n: i64) -> fn(i64) -> i64 {
    fn(x) => x + n
}
```

### `fn(A) -> B` is a single uniform type

All closures with the same parameter and return types share the type `fn(A) -> B`. This is a boxed, dynamically dispatched type — similar to `dyn Fn` in Rust or `func` in Go.

### Compiler optimizations

The compiler may optimize away the boxing in these cases:
- **Inline closures passed to known functions** — `items.map(fn(x) => x * 2)` can be monomorphized
- **Non-escaping closures** — if the closure doesn't outlive its scope, it can be stack-allocated
- **Stateless closures** — `fn(x) => x * 2` (no captures) can be compiled as a plain function pointer

These are invisible optimizations — the programmer never sees them.

### Why boxing by default?

1. **Uniform type** — `fn(i64) -> i64` is one type, not a unique type per closure. This means closures can be stored in arrays, returned from functions, and passed around freely.
2. **Matches GC-default philosophy** — Pillar 4 says GC by default. Closures are values; values are GC'd.
3. **No three-trait complexity** — Rust's `Fn`/`FnMut`/`FnOnce` distinction is a constant source of AI errors. Aria has one closure type.
4. **Compiler can optimize** — the common case (inline, non-escaping) is optimized automatically.

**AI rationale:** I generate closures everywhere — `map`, `filter`, `sort`, callbacks. With boxing, `fn(T) -> U` is always the type. I never have to choose between `Fn`, `FnMut`, and `FnOnce`. I never get "expected impl Fn, found closure" errors. The compiler handles the optimization.

---

## Decision 8: `with` Resource Blocks — Formal Semantics

**Resolution:** `with` is a statement that binds a value, runs a block, and calls `drop()` on the value when the block exits — regardless of how it exits (normal completion, error propagation, or panic).

### Syntax

```
with resource := acquire()? {
    use(resource)
}
// resource.drop() called here — deterministic
```

### Multiple resources

```
with conn := db.connect(url)?,
     tx := conn.begin()? {
    tx.exec("INSERT ...")?
    tx.commit()?
}
// tx.drop() called first, then conn.drop() — reverse order
```

### Grammar

```
with_stmt = "with" with_bindings block ;
with_bindings = with_binding { "," with_binding } ;
with_binding = pattern ":=" expression ;
```

### Semantics

1. Each binding expression is evaluated left-to-right
2. If any binding expression fails (returns `Err` and `?` is used), previously-bound resources are dropped in reverse order, and the error propagates
3. The block executes with all resources in scope
4. When the block exits (for any reason), resources are dropped in reverse binding order
5. `with` bindings are immutable by default; use `mut` for mutable access: `with mut file := ...`

### Difference from `defer`

| Feature | `with` | `defer` |
|---|---|---|
| Scope | Block-scoped — cleanup at end of `with` block | Function-scoped — cleanup at function exit |
| Binding | Creates a scoped binding | No binding |
| Multiple resources | Multiple bindings with ordered cleanup | Multiple `defer` statements (LIFO) |
| Use case | Scoped resource (file, transaction, lock) | Cleanup that must happen at function exit |

```
// with — cleanup at block end
with file := fs.open("data.txt")? {
    process(file)
}
// file closed HERE

doMoreWork()   // file is already closed

// defer — cleanup at function end
fn process() {
    file := fs.open("data.txt")?
    defer file.close()

    doWork(file)
    doMoreWork()   // file still open
}
// file closed HERE (at function exit)
```

**AI rationale:** `with` gives me scoped resource management — I know exactly when cleanup happens (at the `}` of the `with` block). `defer` defers to function exit, which is sometimes too late. When I generate code that opens a file, processes it, then does unrelated work, I use `with` to close the file immediately after processing — not at the end of the function.

---

## Decision 9: Bootstrap Compiler in Go

**Resolution:** The Phase 2 bootstrap compiler will be written in Go.

### Rationale

- **Developer velocity** — Go is fast to write; the bootstrap compiler is throwaway code
- **Good enough type system** — Go 1.18+ has generics, which helps with AST types
- **Concurrency for parallel compilation** — goroutines for per-file parsing and type checking
- **Easy cross-compilation** — `GOOS=linux GOARCH=amd64 go build` for CI
- **Familiar ecosystem** — standard library covers file I/O, JSON, testing, benchmarking
- **The bootstrap compiler is not the product** — the real Aria compiler will be written in Aria (self-hosting). The Go version only needs to compile enough Aria to self-host.

### What the bootstrap compiler must support

The bootstrap compiler implements a subset of Aria sufficient to compile the self-hosting compiler. This means:
- Lexer and parser for full Aria grammar
- Type checker (traits, generics, effects)
- Tier 1 backend only (fast codegen, no LLVM)
- Basic GC (stop-the-world is acceptable for bootstrap)
- Basic task scheduler (can be single-threaded initially)
- Enough stdlib to compile Aria code (io, str operations, collections)

### What the bootstrap compiler does NOT need

- LLVM backend (Tier 2) — optimization comes with self-hosting
- Full GC (concurrent, generational) — basic mark-sweep is fine
- Full stdlib — only what the self-hosting compiler uses
- Full diagnostics — basic error messages are sufficient
- Benchmarking, property-based testing, etc.

---

## Decision 10: No Integer Literal Type Suffixes

**Resolution:** Aria does not support integer literal suffixes (`42u8`, `3.14f32`). Bare integer literals are `i64`; bare float literals are `f64`. Use type annotations or conversions for other types.

### The only way to specify a non-default literal type

```
// Type annotation on the binding
x: u8 = 42
items: [u8] = [1, 2, 3, 255]
const MAX_RETRIES: u32 = 3

// Lossless conversion in an expression
buffer := Buffer.withCapacity(u32(4096))
```

### What is NOT valid

```
42u8        // ❌ syntax error — no suffixes
3.14f32     // ❌ syntax error — no suffixes
0xFFu16     // ❌ syntax error — no suffixes
```

### Rationale

1. **Type annotations are already shorter for the common case.** `items: [u8] = [1, 2, 3]` (one annotation) vs `[1u8, 2u8, 3u8]` (three suffixes).
2. **One way to express types.** Suffixes create a second encoding of type information that's already expressible with `: Type`. Two ways = inconsistency. (Pillar 1: every token carries meaning.)
3. **Simpler lexer.** The lexer doesn't need to check for type suffixes after every numeric literal. `42` is always a single `INT_LIT` token.
4. **Default types cover 90% of usage.** `42` is `i64`, `3.14` is `f64`. When a different type is needed, the context (binding annotation, function parameter type, generic inference) usually provides it.

---

*These decisions are part of the Aria v0.1 language specification. They are binding for the bootstrap compiler implementation and for all spec documents in this repository.*
