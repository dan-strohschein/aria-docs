# Aria Numeric Types and Overflow Behavior Specification

This document formally specifies integer and floating-point types, overflow semantics, and numeric conversion rules for the Aria v0.1 language. These design decisions should be considered part of the core language spec alongside [high-level-design.md](../high-level-design.md).

---

## Integer Types

### Available Integer Types

| Type | Width | Range |
|---|---|---|
| `i8` | 8-bit signed | −128 to 127 |
| `i16` | 16-bit signed | −32,768 to 32,767 |
| `i32` | 32-bit signed | −2,147,483,648 to 2,147,483,647 |
| `i64` | 64-bit signed | −9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
| `i128` | 128-bit signed | ±1.7 × 10³⁸ |
| `u8` | 8-bit unsigned | 0 to 255 |
| `u16` | 16-bit unsigned | 0 to 65,535 |
| `u32` | 32-bit unsigned | 0 to 4,294,967,295 |
| `u64` | 64-bit unsigned | 0 to 18,446,744,073,709,551,615 |
| `u128` | 128-bit unsigned | 0 to 3.4 × 10³⁸ |
| `isize` | Platform pointer-sized signed | Platform-dependent |
| `usize` | Platform pointer-sized unsigned | Platform-dependent |

### Default Integer Type

Untyped integer literals default to `i64`.

```go
x := 42        // i64 — the default
y: u8 = 42     // narrowed by explicit annotation
z := 42_u8     // narrowed by suffix annotation
```

**Why `i64`?** It covers loop counters, file sizes, Unix timestamps, array lengths, and most business domain values without requiring the programmer to think about width. For 95% of code, choosing between `i32` and `i64` is a false decision — just use `i64` and move on. For the 5% of code where width matters (serialization, FFI, SIMD, embedded), explicit annotation makes intent clear.

### Literal Syntax

```go
// Decimal (default)
x := 1_000_000   // underscores for readability

// Hexadecimal
a := 0xFF
b := 0xDEAD_BEEF

// Binary
c := 0b1010_0101

// Octal
d := 0o755

// Suffix annotation
e := 42_i32
f := 255_u8
g := 1_000_000_i64
```

---

## Overflow Behavior

### Default: Trap on Overflow

The default arithmetic operators (`+`, `-`, `*`) **trap (runtime panic) on integer overflow** in all build modes. There is no debug/release behavioral difference.

```go
x: u8 = 255
y := x + 1    // runtime trap: integer overflow
```

The trap points at the exact source line where overflow occurred. The error message identifies the type, the operation, and the values involved.

**Why trap by default?** Integer overflow is almost never intentional. When it occurs silently (C, Go, Rust release mode), it produces incorrect results that propagate invisibly, often surfacing as corrupted data or security vulnerabilities far from the overflow site. Trapping at the exact line converts a silent data corruption bug into an immediate, locatable runtime error. The cost of this safety is zero for code that doesn't overflow — there are no extra tokens, no wrapping annotations, nothing to add.

### Explicit Wrapping Arithmetic

Use `+%`, `-%`, `*%` when modular/wrapping overflow is intentional.

```go
x: u8 = 255
a := x +% 1    // 0       — wraps around
b := x *% 2    // 254     — wraps around (255 * 2 = 510, 510 % 256 = 254)

y: u8 = 0
c := y -% 1    // 255     — wraps around (underflow)
```

**Use cases:** hash functions, checksums, circular buffer indices, cryptographic primitives, intentional modular arithmetic.

```go
// FNV-1a hash (wrapping is intentional and documented)
fn fnv1a(data: [u8]) -> u64 {
    hash: u64 = 14695981039346656037_u64
    for byte in data {
        hash = hash ^ u64(byte)       // XOR — bitwise, no overflow possible
        hash = hash *% 1099511628211_u64  // wrapping multiply — intentional
    }
    hash
}
```

### Explicit Saturating Arithmetic

Use `+|`, `-|`, `*|` when clamping to the type's min/max is the desired behavior.

```go
x: u8 = 255
a := x +| 1    // 255   — saturates at max

y: u8 = 0
b := y -| 1    // 0     — saturates at min

z: i8 = 100
c := z *| 2    // 127   — saturates at i8 max (127)
```

**Use cases:** audio mixing (prevent clipping), graphics blending (prevent component wrap), protocol fields with defined max values.

```go
// Audio mixing: sum samples without wrapping
fn mixSamples(a: i16, b: i16) -> i16 {
    a +| b    // saturates to i16 max/min instead of wrapping
}

// Bounded counter: never exceeds type range
fn incrementSaturating(counter: mut u32) {
    counter = counter +| 1
}
```

### Checked Arithmetic

For code that must handle overflow gracefully rather than trapping, use the checked arithmetic methods which return `Result`.

| Method | Return Type | Description |
|---|---|---|
| `.checkedAdd(rhs)` | `Result[T, OverflowError]` | Addition, returns Err on overflow |
| `.checkedSub(rhs)` | `Result[T, OverflowError]` | Subtraction, returns Err on overflow |
| `.checkedMul(rhs)` | `Result[T, OverflowError]` | Multiplication, returns Err on overflow |
| `.checkedDiv(rhs)` | `Result[T, DivisionError]` | Division, returns Err on overflow or divide-by-zero |
| `.checkedNeg()` | `Result[T, OverflowError]` | Negation, returns Err on overflow (e.g., `i8.min.checkedNeg()`) |

```go
x: i32 = i32.max
result := x.checkedAdd(1)
// result is Err(OverflowError)

match x.checkedAdd(delta) {
    Ok(sum) => processSum(sum)
    Err(_)  => handleOverflow()
}

// With ? propagation
safe_sum := x.checkedAdd(y)?    // propagates OverflowError up
```

### Operator Summary

| Operator | Overflow Behavior |
|---|---|
| `+`, `-`, `*` | Trap (runtime error) |
| `+%`, `-%`, `*%` | Wrap (modular arithmetic) |
| `+|`, `-|`, `*|` | Saturate (clamp to min/max) |
| `.checkedAdd()` etc. | Return `Result` |

---

## Integer Division

### Truncation Toward Zero

Integer division truncates toward zero.

```go
 7 / 2    //  3   (not 4)
-7 / 2    // -3   (not -4)
 7 / -2   // -3
-7 / -2   //  3
```

This is consistent with the behavior of all common hardware integer division instructions (x86 `idiv`, ARM `sdiv`) and the expectation of programmers coming from C, Go, Java, or Python 2.

### Modulo

The `%` operator returns the remainder after truncated division. The sign of the result follows the sign of the dividend.

```go
 7 % 3    //  1
-7 % 3    // -1   (sign follows dividend, not divisor)
 7 % -3   //  1
-7 % -3   // -1
```

### Division and Modulo by Zero

Division by zero and modulo by zero are **runtime traps** (not undefined behavior):

```go
x := 5 / 0    // runtime trap: division by zero
y := 5 % 0    // runtime trap: division by zero
```

---

## No Implicit Numeric Conversions

Aria has no implicit widening or narrowing conversions between numeric types. Every conversion is explicit.

### What Is a Compile Error

```go
a: i32 = 42
b: i64 = a    // COMPILE ERROR: cannot implicitly convert i32 to i64
```

Even widening (from smaller to larger type) is explicit. This prevents silent precision loss or unexpected behavior from mixed-width arithmetic.

### Explicit Widening

```go
a: i32 = 42
b: i64 = i64(a)      // explicit widening — always safe
c: i64 = a.to[i64]() // method syntax, equivalent
```

Both syntaxes are equivalent. The cast syntax `i64(a)` is preferred for brevity; the method syntax `.to[i64]()` is available for contexts where the target type needs to be generic.

### Narrowing with Safety Check

```go
c: i64 = 100_000_000_000
d: i32 = c.to[i32]()    // traps at runtime if c > i32.max or c < i32.min
```

`.to[T]()` narrows safely — it checks that the value fits and traps if it doesn't. This makes narrowing explicit and auditable without silently discarding high bits.

### Explicit Truncation (Unsafe Narrowing)

```go
c: i64 = 0x1_0000_0042
e: i32 = c.trunc[i32]()  // truncates to low 32 bits: 0x42 = 66
```

`.trunc[T]()` performs a bitwise truncation with no overflow check. Use this only when you explicitly want the truncation behavior (e.g., extracting bytes from a packed integer, low bits from a hash value).

### Signed/Unsigned Reinterpretation

```go
x: i8 = -1
y: u8 = x.reinterpret[u8]()   // 255 — same bits, different interpretation
```

`.reinterpret[T]()` is a bitwise reinterpretation with no value conversion. It is only valid between types of the same width.

### Conversion Summary

| Operation | Method | Traps? | Use Case |
|---|---|---|---|
| Widening | `i64(a)` or `a.to[i64]()` | Never | Promote to larger type |
| Safe narrowing | `a.to[i32]()` | If out of range | Narrowing with runtime safety |
| Truncating narrowing | `a.trunc[i32]()` | Never | Explicit low-bits extraction |
| Bit reinterpret | `a.reinterpret[u8]()` | Never (same width only) | Packed data, FFI |

---

## Floating-Point Types

### Available Float Types

| Type | Width | Precision |
|---|---|---|
| `f32` | 32-bit | ~7 decimal digits |
| `f64` | 64-bit | ~15 decimal digits |

### Default Float Type

Untyped float literals default to `f64`.

```go
x := 3.14       // f64 — the default
y: f32 = 3.14   // explicit f32 annotation
z := 3.14_f32   // suffix annotation
```

**Why `f64`?** Precision surprises from `f32` (values that look equal in source but differ after rounding) are a common source of bugs. `f64` eliminates most such surprises at minimal performance cost on modern hardware.

### IEEE 754 Semantics

Aria floats follow IEEE 754 semantics exactly. `NaN` and `Inf` are valid values with well-defined behavior.

#### Special Values

```go
// NaN
nan := 0.0 / 0.0              // NaN (not a trap — floats don't trap)
also_nan := f64.nan            // named constant

// Infinity
inf := 1.0 / 0.0              // +Inf
neg_inf := -1.0 / 0.0         // -Inf
also_inf := f64.inf            // named constant

// Named constants
f64.nan     // not-a-number
f64.inf     // positive infinity
f64.negInf  // negative infinity
f64.max     // largest finite f64 (~1.8 × 10³⁰⁸)
f64.min     // smallest positive normal f64
f64.epsilon // machine epsilon
```

#### NaN Propagation

```go
nan + 1.0      // NaN
nan * 0.0      // NaN
nan == nan     // false — NaN is not equal to itself (IEEE 754)
nan != nan     // true

// To check for NaN, use the method:
x.isNaN()      // true if x is NaN
x.isInf()      // true if x is +Inf or -Inf
x.isFinite()   // true if x is not NaN, not Inf
```

#### Float Comparison Pitfall

Because `NaN != NaN`, comparisons involving potential NaN values should use the check methods:

```go
// Fragile — returns false if result is NaN
if result == expected { ... }

// Robust — explicit NaN handling
if result.isNaN() {
    handleNaN()
} else if result == expected {
    handleMatch()
}
```

### Float Division

Unlike integers, floating-point division by zero produces `Inf` (or `NaN` for `0.0 / 0.0`), not a trap:

```go
1.0 / 0.0    //  Inf
-1.0 / 0.0   // -Inf
0.0 / 0.0    //  NaN
```

For code that wants explicit error handling on problematic float operations, use the checked methods:

| Method | Return Type | Err Case |
|---|---|---|
| `.checkedDiv(b)` | `Result[f64, MathError]` | `b == 0.0` |
| `.checkedSqrt()` | `Result[f64, MathError]` | Negative input |
| `.checkedLog()` | `Result[f64, MathError]` | Non-positive input |
| `.checkedLog2()` | `Result[f64, MathError]` | Non-positive input |

```go
result := x.checkedDiv(y)?   // propagates MathError if y is 0.0
```

### Float-Integer Conversion

Float-to-integer conversion is always explicit:

```go
f := 3.7
i := i64(f)            // 3     — truncates toward zero
j := f.round[i64]()    // 4     — rounds to nearest
k := f.floor[i64]()    // 3     — rounds toward negative infinity
l := f.ceil[i64]()     // 4     — rounds toward positive infinity

// Converting NaN or Inf to integer: runtime trap
nan := 0.0 / 0.0
m := i64(nan)          // trap: cannot convert NaN to integer
```

Integer-to-float conversion is also explicit:

```go
n: i64 = 9_007_199_254_740_993   // 2^53 + 1
f2 := f64(n)                      // precision loss — i64 values > 2^53 may not round-trip
```

---

## Numeric Constants and Type Properties

Each numeric type exposes its bounds and properties as named constants:

```go
i64.min      // -9,223,372,036,854,775,808
i64.max      //  9,223,372,036,854,775,807
u8.min       // 0
u8.max       // 255
f64.inf      // +∞
f64.nan      // NaN
f64.epsilon  // 2.22 × 10⁻¹⁶
```

---

## Design Rationale Summary

| Decision | Rationale |
|---|---|
| Overflow traps by default | Bugs caught at the exact line; zero extra tokens for safe code |
| `+%` / `+|` for wrapping/saturating | Intent is explicit in the source; 1 character cost |
| No debug/release behavior difference | Reproducible behavior regardless of build mode |
| `i64` default integer | Eliminates width decisions for 95% of code |
| No implicit conversions | Types are what they appear to be; no hidden precision loss |
| `f64` default float | Avoids `f32` precision surprises in the common case |
| IEEE 754 floats | Standard behavior; no surprises for numerical code |
| Integer division truncates toward zero | Consistent with hardware and most languages |
| Division by zero traps for integers | Defined, locatable error instead of undefined behavior |
| Division by zero yields Inf/NaN for floats | IEEE 754 standard; use `.checkedDiv()` when error handling needed |

---

## Interaction with Other Features

### Numeric Types and Pattern Matching

```go
match status_code {
    200 => handleOk()
    404 => handleNotFound()
    500..=599 => handleServerError()
    other => handleUnknown(other)
}
```

Ranges in match arms (`500..=599`) work for all integer types and include both endpoints when using `..=`.

### Numeric Types and the FFI

When calling C functions, Aria integer types map directly to C types:

| Aria | C |
|---|---|
| `i8` | `int8_t` |
| `i32` | `int32_t` |
| `i64` | `int64_t` |
| `u8` | `uint8_t` / `unsigned char` |
| `usize` | `size_t` |
| `f32` | `float` |
| `f64` | `double` |

See [ffi-design.md](ffi-design.md) for C interop details.

### Numeric Types and Overflow in Loops

```go
// Common loop: no overflow risk, i64 is plenty
for i in 0..items.len() {
    process(items[i])
}

// If working near type boundaries, use checked arithmetic
fn safeIncrement(counters: mut [u32], idx: i64) -> Result[void, OverflowError] {
    counters[idx] = counters[idx].checkedAdd(1)?
    Ok(void)
}
```
