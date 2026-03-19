# Aria Initialization and Zero Values

## Design Philosophy

Aria takes a firm stance: **no implicit zero values, no uninitialized variables.** Every variable must be explicitly initialized, either with a value, a default, or via the `late` mechanism with compiler-verified definite assignment. This eliminates null pointer panics, zero-value surprises, and "forgot to initialize" bugs.

Key principles:
- **All struct fields must be initialized** — unless they have explicit defaults
- **No Go-style implicit zero values** — a zero-valued struct is not automatically valid
- **The `Default` trait is opt-in** — only types that explicitly declare defaults have them
- **`late` for deferred initialization** — with compiler-verified definite assignment
- **Primitives have known defaults, but only when explicitly requested**

---

## Struct Initialization

### All Fields Required (No Defaults)

```
type Point {
    x: f64
    y: f64
}

p := Point{}                // ❌ compile error: x and y not initialized
p := Point{x: 1.0}         // ❌ compile error: y not initialized
p := Point{x: 1.0, y: 2.0} // ✅ all fields provided
```

### Fields with Defaults

```
type Server {
    host: str = "localhost"
    port: u16 = 8080
    timeout: dur = 30s
    logger: Logger = .default()
}

s := Server{}                    // ✅ all fields have defaults
s := Server{port: 9090}         // ✅ override port, rest use defaults
s := Server{host: "0.0.0.0", port: 443, timeout: 60s}  // ✅ mix
```

**Rule**: A struct can only be constructed with `T{}` (no arguments) if **every field** has a default value. If even one field lacks a default, the compiler requires it.

### Mixed Fields

```
type DatabaseConfig {
    url: str                     // NO default — must be provided
    maxConns: u16 = 10           // has default
    timeout: dur = 30s           // has default
    sslMode: bool = true         // has default
}

c := DatabaseConfig{}                        // ❌ compile error: url not initialized
c := DatabaseConfig{url: "postgres://..."}   // ✅ url provided, rest use defaults
```

---

## Variable Initialization

### No Uninitialized Variables

```
x: i64                          // ❌ compile error: variable must be initialized
x: i64 = 42                    // ✅ explicitly initialized
x := 42                        // ✅ type inferred, initialized
```

**Rule**: Every variable binding must have an initializer. There are no uninitialized variables in Aria. Period.

### `late` for Deferred Initialization

For cases where initialization genuinely depends on control flow, use `late`:

```
late db: Database

if config.usePostgres {
    db = connectPostgres(config.pgUrl)?
} else {
    db = connectSqlite(config.sqlitePath)?
}

// The compiler verifies: db is DEFINITELY assigned before this point
use(db)  // ✅ compiler proves all paths assign db
```

**Rules for `late`:**
- `late` declares a variable without initializing it
- The compiler performs **definite assignment analysis**: it verifies that every possible execution path assigns the variable before it is read
- Using a `late` variable before all paths have assigned it is a **compile error**
- `late` variables can only be assigned once — they are not mutable after assignment
- `late` is NOT a mechanism for nullable types — it's a mechanism for control-flow-dependent initialization

```
late x: i64

if condition {
    x = 42
}
// Using x here is a compile error — the else branch doesn't assign x

late y: i64

if condition {
    y = 42
} else {
    y = 0
}
// Using y here is ✅ — both branches assign y
```

---

## The `Default` Trait

```
trait Default {
    fn default() -> Self
}
```

### Auto-Derivable

When all fields have defaults, `Default` can be derived:

```
type Config {
    debug: bool = false
    logLevel: str = "info"
    maxRetries: u8 = 3
} derives [Default]

c := Config.default()        // ✅ all fields use their defaults
```

### Manual Implementation

```
impl Default for Logger {
    fn default() -> Logger = Logger{
        level: .Info
        output: io.stdout
        format: .Text
    }
}

logger := Logger.default()
```

### Primitive Defaults

Primitives have known default values, accessible via `T.default()` or when used as struct field defaults:

| Type | Default value |
|---|---|
| `i8`..`i64` | `0` |
| `u8`..`u64` | `0` |
| `f32`, `f64` | `0.0` |
| `bool` | `false` |
| `str` | `""` |
| `[T]` | `[]` (empty list) |
| `Map[K,V]` | `{}` (empty map) |
| `Set[T]` | `{}` (empty set) |
| `Option[T]` | `None` |

**These defaults are only used when explicitly requested:**
1. A struct field declares `= .default()` or a specific default value
2. You call `T.default()` explicitly
3. A collection method needs a default (like `map.getOrDefault(key)`)
4. A `derives [Default]` generates them

They are **never applied implicitly** to uninitialized variables or struct fields.

---

## The `.default()` Convention

Any type can provide a `.default()` static method via the `Default` trait. This integrates with struct field defaults:

```
type Server {
    logger: Logger = .default()     // calls Logger.default()
    cache: Cache = .default()       // calls Cache.default()
    port: u16                       // no default — must be provided
}
```

The `.default()` shorthand (without the type name) works in struct field defaults because the type is known from the field declaration.

---

## Record Update Syntax and Initialization

The record update syntax (`.{...}`) creates a modified copy. All fields are initialized because they come from the source:

```
s := Server{port: 8080}
s2 := s.{port: 9090}           // ✅ all fields initialized — copied from s, port overridden
s3 := s.{timeout: 60s}         // ✅ all fields initialized
```

---

## Design Rationale Summary

| Decision | Rationale |
|---|---|
| No implicit zero values | Zero-valued structs are not automatically valid — prevents "forgot to initialize" bugs |
| All fields required unless defaulted | The compiler catches missing fields at compile time |
| `late` with definite assignment | Handles control-flow initialization without introducing null |
| `Default` trait is opt-in | Only types that explicitly declare defaults have them |
| Primitive defaults only when explicit | `0` is not automatically assigned to `i64` variables |
| `.default()` convention | Clean syntax for default construction |

---

## Comparison with Other Languages

| Feature | Go | Rust | Aria |
|---|---|---|---|
| Uninitialized vars | Zero-valued implicitly | Compile error (must init) | Compile error (must init) |
| Struct missing fields | Zero-valued implicitly | Compile error | Compile error |
| Default values on fields | Not supported | Not built-in (use `Default`) | Built into struct syntax |
| Null/nil | Yes (nil) | No (Option) | No (Option) |
| Deferred init | Just use zero value | `let x; x = ...` with analysis | `late x; x = ...` with analysis |

---

## Related Specs

- `high-level-design.md` — Core language design and type system overview
- `spec/stdlib-design.md` — Collection types (`[T]`, `Map[K,V]`, `Set[T]`) and their defaults
- `spec/language-spec-addendum.md` — Destructuring syntax used in struct patterns
