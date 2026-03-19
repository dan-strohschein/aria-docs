# Aria Type Conversion Specification

## Design Philosophy

Aria's type conversion system is built around one principle: **the syntax tells you the safety guarantee**.

Three conversion mechanisms. Each with a distinct syntax. Each with a clear contract. No implicit conversions, no silent data loss, no ambiguity about what a conversion does.

> **The rule**: If you can't tell from reading the source code whether a conversion is safe, the language has failed you.

Key principles:

- **Three mechanisms, zero ambiguity** — lossless, checked, and truncating conversions each have distinct syntax
- **The compiler enforces safety** — `T(x)` won't compile if data could be lost
- **Lossy intent is always explicit** — `.trunc[T]()` makes data loss visible in the code
- **No implicit conversions ever** — `i32 + i64` is a compile error

---

## The Three Conversion Mechanisms

### 1. Lossless Conversion: `T(x)`

**Syntax:** `TargetType(value)`

**Semantics:** Compile-time proven safe. The compiler verifies that no data can be lost. If the conversion could lose data, the code **does not compile**.

**Use cases:** Integer widening, `u8` → `i16`, `i32` → `i64`, `f32` → `f64`

**Backed by:** The `Convert` trait

```
// Integer widening — always lossless
a: i32 = 42
b: i64 = i64(a)         // ✅ compiles — i32 always fits in i64

// Float widening
f: f32 = 3.14
g: f64 = f64(f)         // ✅ compiles — f32 always fits in f64

// This does NOT compile:
c: i64 = 9999999999
d: i32 = i32(c)          // ❌ compile error — i64 may not fit in i32
```

**Rule:** `T(x)` is syntactic sugar for `x.convert()` where the `Convert[T]` trait is implemented. The compiler only provides `Convert` implementations for provably lossless conversions.

---

### 2. Checked Conversion: `x.to[T]()`

**Syntax:** `value.to[TargetType]()`

**Returns:** `Result[T, ConversionError]`

**Semantics:** Performs the conversion at runtime. Returns `Err(ConversionError)` if the value doesn't fit in the target type. Never panics, never silently loses data.

**Use cases:** Integer narrowing, string parsing, any conversion that might fail

**Backed by:** The `TryConvert` trait

```
// Integer narrowing — might fail
c: i64 = 9999999999
d := c.to[i32]()?          // propagate error if it doesn't fit
d := c.to[i32]()!          // panic if it doesn't fit (assert)

// Works when value fits
e: i64 = 42
f := e.to[i32]()?          // Ok(42)

// Float to int — checked
g: f64 = 3.0
h := g.to[i64]()?          // Ok(3) — no fractional part
i: f64 = 3.7
j := i.to[i64]()?          // Err(FractionalLoss) — has fractional part
```

**Rule:** `x.to[T]()` is syntactic sugar for `x.tryConvert()` where the `TryConvert[T]` trait is implemented.

---

### 3. Truncating Conversion: `x.trunc[T]()`

**Syntax:** `value.trunc[TargetType]()`

**Returns:** `T` (never fails)

**Semantics:** Performs the conversion, explicitly discarding data that doesn't fit. The caller accepts data loss. This is the "I know what I'm doing" escape hatch.

**Use cases:** Float to int (discard decimal), large int to small int (wrap/truncate), lossy transformations

```
// Float to int — truncate toward zero
f: f64 = 3.7
n := f.trunc[i64]()        // 3

f2: f64 = -3.7
n2 := f2.trunc[i64]()      // -3

// Integer narrowing — truncate (keep low bits)
big: i64 = 256
small := big.trunc[u8]()   // 0 (256 mod 256)

big2: i64 = 257
small2 := big2.trunc[u8]() // 1 (257 mod 256)
```

**Rule:** `x.trunc[T]()` makes the intent to lose data explicit in the source code. There is no silent truncation anywhere in Aria.

> See also: [numeric-overflow.md](numeric-overflow.md) for overflow behavior and wrapping/saturating arithmetic operators.

---

## String Conversions

String conversions are methods, not casts. They follow the same three-mechanism pattern where applicable.

> See also: [string-handling.md](string-handling.md) for the full `str` and `char` type specification.

### To String: `.toStr()`

All primitive types and types that derive `Debug` or implement `Display` can convert to string:

```
n: i64 = 42
s := n.toStr()              // "42"

f: f64 = 3.14159
s := f.toStr()              // "3.14159"

b: bool = true
s := b.toStr()              // "true"

// Custom types — derive or implement Display
type User {
    name: str
    age: u8
} derives [Display]

user := User{name: "Alice", age: 30}
s := user.toStr()           // "User{name: Alice, age: 30}"
```

`.toStr()` is always lossless and infallible for types that implement it.

### From String: Parsing (Always Fallible)

String to any other type is always a checked operation — parsing can fail:

```
// Integer parsing
n := "42".parseInt[i64]()?           // Ok(42)
n := "hello".parseInt[i64]()?       // Err(ParseError)
n := "42".parseInt[u8]()?           // Ok(42)
n := "999".parseInt[u8]()?          // Err(ParseError) — out of range

// Float parsing
f := "3.14".parseFloat[f64]()?      // Ok(3.14)
f := "not a number".parseFloat[f64]()? // Err(ParseError)

// Bool parsing
b := "true".parseBool()?            // Ok(true)
b := "false".parseBool()?           // Ok(false)
b := "yes".parseBool()?             // Err(ParseError) — strict

// With radix (optional named parameter, defaults to 10)
n := "ff".parseInt[i64](radix: 16)? // Ok(255) — hexadecimal
n := "1010".parseInt[i64](radix: 2)? // Ok(10) — binary
```

**Rule:** There is no infallible string-to-number conversion. `"hello".parseInt[i64]()` returns a `Result`, it doesn't panic.

### Byte/String Conversions

```
// String to bytes — lossless (UTF-8 encoding)
s: str = "hello"
bytes := s.toBytes()                 // [u8] — UTF-8 bytes, lossless

// Bytes to string — checked (must be valid UTF-8)
data: [u8] = [72, 101, 108, 108, 111]
s := data.toStr()?                   // Ok("Hello")

invalid: [u8] = [0xff, 0xfe]
s := invalid.toStr()?                // Err(Utf8Error)

// Bytes to string — lossy (replace invalid sequences)
s := data.toStrLossy()               // replaces invalid UTF-8 with replacement char
```

---

## Collection Conversions

Collections use `.to[T]()` for conversions between collection types:

```
// List to Set (may lose duplicates — but Set is the target, so this is expected)
list: [i64] = [1, 2, 2, 3]
set := list.to[Set[i64]]()           // Set{1, 2, 3}

// Set to List (order is unspecified)
set: Set[str] = Set{"a", "b", "c"}
list := set.to[List[str]]()          // ["a", "b", "c"] (some order)

// Map entries to List
m: Map[str, i64] = {"a": 1, "b": 2}
entries := m.entries().to[List[(str, i64)]]()  // [(str, i64)]

// List of pairs to Map
pairs: [(str, i64)] = [("a", 1), ("b", 2)]
m := pairs.to[Map[str, i64]]()       // Map{"a": 1, "b": 2}
```

---

## Custom Type Conversions via Traits

### The `Convert` Trait (Lossless)

```
trait Convert[T] {
    fn convert(self) -> T
}

// Example: a custom Celsius type that can losslessly convert to Fahrenheit (f64 → f64)
type Celsius { value: f64 }
type Fahrenheit { value: f64 }

impl Convert[Fahrenheit] for Celsius {
    fn convert(self) -> Fahrenheit = Fahrenheit{value: self.value * 9.0 / 5.0 + 32.0}
}

// Now this works:
temp := Celsius{value: 100.0}
f := Fahrenheit(temp)               // Fahrenheit{value: 212.0}
```

**Important:** `Convert` should only be implemented when the conversion is truly lossless. The compiler provides built-in `Convert` implementations for numeric widening. User-defined `Convert` implementations are trusted — the type author asserts losslessness.

### The `TryConvert` Trait (Checked)

```
trait TryConvert[T] {
    fn tryConvert(self) -> Result[T, ConversionError]
}

// Example: a UserId that can be constructed from i64 only if positive
type UserId { value: i64 }

impl TryConvert[UserId] for i64 {
    fn tryConvert(self) -> Result[UserId, ConversionError] = {
        if self <= 0 { err(ConversionError.InvalidValue("UserId must be positive")) }
        else { ok(UserId{value: self}) }
    }
}

// Now this works:
id := 42.to[UserId]()?              // Ok(UserId{value: 42})
id := (-1).to[UserId]()?            // Err(InvalidValue)
```

---

## ConversionError Type

```
type ConversionError =
    | Overflow          // value too large for target type
    | Underflow         // value too small (negative) for unsigned target
    | FractionalLoss    // float has fractional part, target is integer
    | InvalidValue(str) // custom validation failure
    | Utf8Error         // invalid UTF-8 in byte-to-string conversion
    | ParseError(str)   // string parsing failure
```

---

## Rules Summary

1. **No implicit conversions** — `i32 + i64` is a compile error. Use `i64(a) + b`.
2. **`T(x)` is compile-time safe** — won't compile if data could be lost.
3. **`.to[T]()` is runtime safe** — returns `Result`, never silently loses data.
4. **`.trunc[T]()` is explicitly lossy** — caller accepts data loss.
5. **String parsing is always fallible** — returns `Result`, not a panic.
6. **No `as` keyword** — eliminates silent truncation bugs.
7. **No implicit truthiness** — `0` is not `false`, empty string is not `false`, `None` is not `false`. Use explicit comparison or matching.
8. **Custom conversions use traits** — `Convert` for lossless, `TryConvert` for checked.

---

## Token Comparison with Other Languages

| Operation | Go | Rust | Aria |
|---|---|---|---|
| Widen i32 → i64 | `int64(x)` | `x as i64` or `i64::from(x)` | `i64(x)` |
| Narrow i64 → i32 (safe) | `int32(x)` (silent truncation!) | `x as i32` (silent truncation!) | `x.to[i32]()?` |
| Narrow i64 → i32 (explicit lossy) | `int32(x)` (same syntax as safe!) | `x as i32` (same syntax as safe!) | `x.trunc[i32]()` (different syntax!) |
| Int to string | `strconv.Itoa(n)` | `n.to_string()` | `n.toStr()` |
| String to int | `strconv.Atoi(s)` + 3-line error check | `s.parse::<i64>()?` | `s.parseInt[i64]()?` |
| Float to int | `int(f)` (silent truncation!) | `f as i64` (silent truncation!) | `f.trunc[i64]()` (explicit) |
| Bytes to string | `string(bytes)` (copies) | `String::from_utf8(bytes)?` | `bytes.toStr()?` |

Key insight: Go and Rust use the **same syntax** for safe widening and unsafe narrowing. Aria uses **different syntax** — making the safety guarantee visible in every conversion.

---

## Interaction with Other Language Features

**Error propagation (`?`):**
```
fn processInput(raw: str) -> ProcessedData ! ConversionError | ProcessError {
    id := raw.parseInt[i64]()?           // ConversionError propagates
    data := fetch(id)?                    // ProcessError propagates
    data
}
```

**Pipeline operator:**
```
result := rawInput
    |> .trim()
    |> .parseInt[i64]()?
    |> processId?
```

**Pattern matching on ConversionError:**
```
match value.to[i32]() {
    Ok(n) => use(n)
    Err(Overflow) => log("value too large")
    Err(Underflow) => log("value too small")
    Err(e) => log("conversion failed: {e}")
}
```

---

## Design Rationale Summary

| Decision | Rationale |
|---|---|
| Three distinct conversion syntaxes | Safety guarantee is visible in the syntax — no guessing |
| `T(x)` compile-time only for lossless | Silent data loss is impossible with the simple syntax |
| `.to[T]()` returns Result | Conversions that can fail always surface the error |
| `.trunc[T]()` for explicit lossy | Intent to lose data is always visible in the code |
| No `as` keyword | Eliminates the single largest source of silent truncation bugs |
| String parsing always returns Result | `"hello".parseInt()` can't panic — must handle the error |
| No implicit truthiness | Types are what they appear to be — `0` is an integer, not a boolean |
| Custom conversions via traits | Extensible system that follows the same three-mechanism pattern |

> See also: [high-level-design.md](../high-level-design.md) for base type definitions, [numeric-overflow.md](numeric-overflow.md) for overflow and wrapping behavior, [string-handling.md](string-handling.md) for string type details.
