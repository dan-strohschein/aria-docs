# Aria String Handling Specification

**A complete specification for the `str` type, `char` type, in-memory representation, indexing semantics, string interpolation, and text processing.**

This document formally specifies how strings work in Aria — from the in-memory representation (small string optimization, GC-traced backing) to the API surface (byte indexing, explicit character iteration, zero-copy substrings). Every design decision is driven by the principle that string operations should be explicit about their cost and never silently corrupt data.

Cross-references:
- [high-level-design.md](../high-level-design.md) — `str` type, string interpolation
- [spec/formal-grammar.md](formal-grammar.md) — string literal grammar, interpolation syntax
- [spec/type-conversions.md](type-conversions.md) — `.toStr()`, string parsing
- [spec/iteration-protocol.md](iteration-protocol.md) — `str` implements `Iterable` with `Item = char`
- [spec/garbage-collector.md](garbage-collector.md) — GC interaction with string backing memory
- [spec/ffi-design.md](ffi-design.md) — C string interop
- [spec/equality-comparison.md](equality-comparison.md) — string `Eq`, `Ord`, `Hash`

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [In-Memory Representation](#2-in-memory-representation)
3. [Indexing and Slicing](#3-indexing-and-slicing)
4. [The Layered Access Model](#4-the-layered-access-model)
5. [The `char` Type](#5-the-char-type)
6. [String Methods](#6-string-methods)
7. [String Interpolation](#7-string-interpolation)
8. [String Builder](#8-string-builder)
9. [String Concatenation](#9-string-concatenation)
10. [Multi-Line Strings](#10-multi-line-strings)
11. [Raw Strings](#11-raw-strings)
12. [String Interning](#12-string-interning)
13. [Interaction with Other Features](#13-interaction-with-other-features)
14. [Design Rationale Summary](#14-design-rationale-summary)
15. [Comparison with Other Languages](#15-comparison-with-other-languages)

---

## 1. Design Philosophy

Strings are the most common data type in programming. Getting them wrong has cascading consequences — every function that processes text inherits the wrong semantics. Aria's string design is built on three rules:

1. **One string type** — `str`. No `String` vs `&str`. No `string` vs `[]byte`. One type, everywhere.
2. **UTF-8 always** — the invariant is enforced at construction. Every `str` value is valid UTF-8.
3. **Operations are explicit about cost** — `s.len()` is O(1) byte length. `s.charCount()` is O(n) codepoint count. The name tells you the cost.

### Why byte indexing, not codepoint indexing

The most important design decision in this spec: **`s[i]` returns a byte (`u8`), not a codepoint (`char`).**

This is counterintuitive. Most humans expect `"café"[3]` to be `'é'`. But in UTF-8, reaching the *i*th codepoint requires scanning from the start — it's O(n). Making `s[i]` look O(1) (like array indexing) while being O(n) violates Aria's "no implicit behavior" tenet. It hides a cost that compounds into O(n²) bugs in innocent-looking loops.

Instead, byte indexing is O(1), the cost is honest, and character-level access is available through explicit methods (`.chars()`, `.charCount()`) whose names communicate the cost.

**AI rationale:** When I generate `s[i]`, I need to know its cost. If byte indexing, I know it's O(1) — I can use it freely in hot loops. If I need characters, I write `s.chars()` and process the iterator — the O(n) cost is visible in the code. I never accidentally generate O(n²) string processing by indexing in a loop.

---

## 2. In-Memory Representation

### Small String Optimization (SSO)

`str` is a 24-byte value type. Short strings (≤ 23 bytes) are stored inline — no heap allocation, no GC pressure, no pointer indirection.

```
// 24 bytes total on 64-bit
str = discriminated union {
    inline: {
        flag: 1 bit = 0              // distinguishes inline from heap
        len: 7 bits                   // byte length (0..23)
        data: [23]u8                  // inline UTF-8 bytes
    }
    heap: {
        flag: 1 bit = 1              // distinguishes heap from inline
        len: 63 bits                  // byte length
        ptr: *u8                      // pointer to GC-managed byte array
    }
}
```

The high bit of the first byte is the discriminant: 0 = inline, 1 = heap. This is a single bit test — well-predicted by the CPU.

### Why SSO matters

Empirically, 60-80% of strings in typical programs are ≤ 23 bytes: field names, map keys, HTTP headers, short error messages, status codes, format fragments. SSO means the majority of string allocations never touch the GC heap.

```
// All inline — zero heap allocation
"name"                    // 4 bytes — inline
"Content-Type"            // 12 bytes — inline
"application/json"        // 16 bytes — inline
"hello, world!"           // 13 bytes — inline
"error: file not found"   // 21 bytes — inline

// Heap-allocated (> 23 bytes)
"this is a longer string that exceeds the inline threshold"
```

### Heap-backed strings

Strings longer than 23 bytes store their UTF-8 data in a GC-managed byte array. The `str` value holds a pointer and a length. The GC traces the pointer and keeps the backing memory alive.

### Substrings: zero-copy views

Since `str` is immutable, substrings share the backing memory of the parent string:

```
s := "Hello, World!"          // heap-allocated (13 bytes, but could be inline too)
greeting := s[0..5]            // "Hello" — new str value pointing into s's memory
name := s[7..12]               // "World" — same backing memory, different offset

// No data is copied. The GC keeps the backing memory alive as long
// as any str value references it.
```

A substring `str` is just a different pointer+length into the same backing bytes. This is O(1) and zero-copy.

**Note:** For inline strings (≤ 23 bytes), substrings are also inline — the bytes are copied into the new inline `str` value. This is still O(1) since it's at most 23 bytes.

---

## 3. Indexing and Slicing

### Byte indexing: `s[i]`

`s[i]` returns the byte (`u8`) at position `i`. This is O(1).

```
s := "café"

s[0]    // 99  (u8 — byte value of 'c')
s[1]    // 97  (u8 — byte value of 'a')
s[2]    // 102 (u8 — byte value of 'f')
s[3]    // 195 (u8 — first byte of the 2-byte UTF-8 encoding of 'é')
s[4]    // 169 (u8 — second byte of 'é')
```

### Bounds checking

`s[i]` panics if `i >= s.len()`:

```
s := "hello"
s[5]    // PANIC: string index out of bounds — index 5, length 5
```

### Byte slicing: `s[start..end]`

`s[start..end]` returns a `str` containing the bytes from `start` (inclusive) to `end` (exclusive). This is O(1) — it creates a view, not a copy.

```
s := "Hello, World!"
s[0..5]     // "Hello"
s[7..12]    // "World"
s[0..s.len()]  // "Hello, World!" (full string)
```

### Character boundary validation on slicing

Byte slicing panics if the boundary falls in the middle of a multi-byte UTF-8 character. This preserves the invariant that every `str` value is valid UTF-8.

```
s := "café"
// s bytes: [99, 97, 102, 195, 169]
//                        ^^^^^^^^^
//                        'é' = bytes 3..5

s[0..3]     // "caf" — valid boundary ✅
s[0..5]     // "café" — valid boundary ✅
s[0..4]     // PANIC: string slice at byte 4 splits the character 'é' (bytes 3..5)
            // help: use s.chars().take(4).collect() for character-level slicing
```

The check is cheap: one table lookup to verify the byte at the boundary is a valid UTF-8 start byte (high bits are `0xxxxxxx` or `11xxxxxx`, never `10xxxxxx`).

### Inclusive range slicing

```
s := "hello"
s[0..=4]    // "hello" — includes byte at position 4
```

---

## 4. The Layered Access Model

Aria provides three layers for accessing string contents. Each layer is explicit — the method name communicates the abstraction level and cost.

| Layer | Access method | Element type | Cost | Use case |
|---|---|---|---|---|
| **Bytes** | `s[i]`, `s.bytes()`, `s.len()` | `u8` | O(1) index, O(n) iterate | Buffer sizing, protocols, FFI |
| **Codepoints** | `s.chars()`, `s.charCount()` | `char` | O(n) iterate, O(n) count | Character-level text processing |
| **Graphemes** | `s.graphemes()` | `str` | O(n) iterate | Visual characters (emoji, accents) |

### Bytes layer (default)

```
s := "café"

s.len()         // 5 — byte count (O(1))
s[3]            // 195 — byte value (O(1))
s.bytes()       // iterator over [u8]: 99, 97, 102, 195, 169
```

### Codepoints layer (explicit)

```
s := "café"

s.charCount()   // 4 — codepoint count (O(n))
s.chars()       // iterator over [char]: 'c', 'a', 'f', 'é'

// Nth character (O(n) from the start)
s.chars().nth(3)   // Some('é')

// Character iteration in for loop
for ch in s.chars() {
    println(ch)     // c, a, f, é
}
```

### Graphemes layer (explicit)

Grapheme clusters are the visual characters users perceive. A single grapheme may consist of multiple codepoints:

```
face := "👨‍👩‍👧"

face.len()                  // 18 — bytes
face.charCount()            // 5 — codepoints (man + ZWJ + woman + ZWJ + girl)
face.graphemes().count()    // 1 — one visual character

// Another example: 'é' as combining characters
e_accent := "é"             // could be U+00E9 (1 codepoint) or U+0065 U+0301 (2 codepoints)
e_accent.graphemes().count() // 1 — always one visual character regardless of encoding
```

### Iteration default

`for ch in s` iterates over codepoints (yields `char`), because this is the most commonly useful iteration level for text processing:

```
for ch in "hello" {
    println(ch)     // h, e, l, l, o — each is a char
}
```

This is consistent with `str` implementing `Iterable` with `type Item = char`.

**AI rationale:** The layered model means I always choose my abstraction level explicitly. `s.len()` = bytes, fast. `s.charCount()` = codepoints, slow. `s.graphemes().count()` = visual characters, slowest. The name tells me the cost. I never accidentally use the wrong one because they're different methods, not different modes.

---

## 5. The `char` Type

A `char` is a Unicode codepoint (U+0000 to U+10FFFF). It is a 32-bit value, distinct from `u8`, `u16`, or `u32`.

### Literals

```
ch: char = 'A'
ch: char = 'é'
ch: char = '世'
ch: char = '\n'
ch: char = '\u{1F600}'     // 😀
```

### Grammar

```
char_literal = "'" ( codepoint | char_escape ) "'" ;
char_escape  = "\n" | "\t" | "\r" | "\\" | "\'" | "\u{" hex_digit+ "}" ;
```

### `char` methods

| Method | Return type | Description |
|---|---|---|
| `c.isAlpha()` | `bool` | Unicode alphabetic |
| `c.isDigit()` | `bool` | Unicode decimal digit |
| `c.isAlphaNum()` | `bool` | Alphabetic or digit |
| `c.isWhitespace()` | `bool` | Unicode whitespace |
| `c.isUpper()` | `bool` | Uppercase letter |
| `c.isLower()` | `bool` | Lowercase letter |
| `c.isAscii()` | `bool` | ASCII range (0..127) |
| `c.toUpper()` | `char` | Unicode uppercase |
| `c.toLower()` | `char` | Unicode lowercase |
| `c.toU32()` | `u32` | Codepoint as integer |
| `c.toStr()` | `str` | Single-character string |
| `c.utf8Len()` | `u8` | Number of bytes in UTF-8 encoding (1-4) |

### `char` vs `u8`

`char` and `u8` are distinct types with no implicit conversion:

```
c: char = 'A'
b: u8 = 65

b = c        // ❌ compile error: char is not u8
c = b        // ❌ compile error: u8 is not char

// Explicit conversion
b = c.toU32().to[u8]()?      // checked: fails if codepoint > 255
c = char.fromU32(b.to[u32]())? // checked: fails if not valid codepoint
```

### Constructing `char` from integers

```
ch := char.fromU32(0x00E9)?     // Ok('é')
ch := char.fromU32(0x110000)?   // Err — above Unicode range
ch := char.fromU32(0xD800)?     // Err — surrogate codepoint (not valid Unicode scalar)
```

---

## 6. String Methods

### Length and emptiness

| Method | Return type | Cost | Description |
|---|---|---|---|
| `s.len()` | `i64` | O(1) | Byte length |
| `s.charCount()` | `i64` | O(n) | Number of Unicode codepoints |
| `s.isEmpty()` | `bool` | O(1) | True if byte length is 0 |

### Searching

| Method | Return type | Description |
|---|---|---|
| `s.contains(sub)` | `bool` | Whether substring is present |
| `s.startsWith(prefix)` | `bool` | Prefix check |
| `s.endsWith(suffix)` | `bool` | Suffix check |
| `s.find(sub)` | `i64?` | Byte offset of first occurrence, or None |
| `s.findLast(sub)` | `i64?` | Byte offset of last occurrence, or None |

`find` and `findLast` return byte offsets, suitable for use with `s[offset..]` slicing:

```
s := "Hello, World!"
offset := s.find("World")?    // Some(7) — byte offset
greeting := s[0..offset]       // "Hello, " — byte slice up to the match
```

### Transforming

| Method | Return type | Description |
|---|---|---|
| `s.toUpper()` | `str` | Unicode-aware uppercase |
| `s.toLower()` | `str` | Unicode-aware lowercase |
| `s.trim()` | `str` | Remove leading/trailing whitespace |
| `s.trimStart()` | `str` | Remove leading whitespace |
| `s.trimEnd()` | `str` | Remove trailing whitespace |
| `s.replace(old, new)` | `str` | Replace all occurrences |
| `s.replaceFirst(old, new)` | `str` | Replace first occurrence |
| `s.repeat(n)` | `str` | Repeat string n times |
| `s.reverse()` | `str` | Reverse by codepoints (not bytes) |

### Splitting

| Method | Return type | Description |
|---|---|---|
| `s.split(sep)` | `[str]` | Split by separator |
| `s.splitN(sep, n)` | `[str]` | Split into at most n parts |
| `s.lines()` | `[str]` | Split into lines (`\n`, `\r\n`, `\r`) |

### Iterators

| Method | Yields | Description |
|---|---|---|
| `s.bytes()` | `u8` | Iterator over raw UTF-8 bytes |
| `s.chars()` | `char` | Iterator over Unicode codepoints |
| `s.graphemes()` | `str` | Iterator over grapheme clusters |

### Conversion

| Method | Return type | Description |
|---|---|---|
| `s.toBytes()` | `[u8]` | UTF-8 bytes as a byte array (copy) |
| `s.intern()` | `str` | Return an interned copy (deduplication) |

---

## 7. String Interpolation

String interpolation is Aria's primary mechanism for building strings from expressions. It is handled at compile time — no runtime format string parsing.

### Basic interpolation

```
name := "Aria"
version := 1

greeting := "Hello, {name}!"              // "Hello, Aria!"
info := "Version: {version}"               // "Version: 1"
computed := "Sum: {1 + 2 + 3}"            // "Sum: 6"
method := "Upper: {name.toUpper()}"       // "Upper: ARIA"
```

### Format specifiers

A colon inside an interpolation introduces a format specifier:

```
interpolation = "{" expr ( ":" format_spec )? "}"
format_spec   = fill? align? sign? "#"? "0"? width? ( "." precision )? type?
fill          = any_char
align         = "<" | "^" | ">"
sign          = "+" | "-"
width         = integer | "{" expr "}"
precision     = integer | "{" expr "}"
type          = "b" | "d" | "e" | "E" | "f" | "o" | "x" | "X" | "%"
```

### Precision

```
pi := 3.14159265358979

"{pi:.4}"     // "3.1416"   — 4 decimal places (rounds)
"{pi:.2}"     // "3.14"
"{pi:.0}"     // "3"
```

### Number bases

```
value := 255

"{value:#x}"   // "0xff"          — hex with prefix
"{value:#X}"   // "0XFF"          — hex uppercase
"{value:#b}"   // "0b11111111"    — binary with prefix
"{value:#o}"   // "0o377"         — octal with prefix
"{value:x}"    // "ff"            — hex, no prefix
```

### Padding and alignment

```
n := 42

"{n:>10}"    // "        42"  — right-aligned, width 10
"{n:<10}"    // "42        "  — left-aligned, width 10
"{n:^10}"    // "    42    "  — centered, width 10
"{n:0>5}"    // "00042"       — right-aligned, zero-padded

// Dynamic width
w := 8
"{n:>{w}}"   // "      42"
```

### Sign

```
x := 42
y := -7

"{x:+}"   // "+42"
"{y:+}"   // "-7"
```

### Percent

```
ratio := 0.853

"{ratio:%}"     // "85.3%"
"{ratio:.1%}"   // "85.3%"
"{ratio:.0%}"   // "85%"
```

### Scientific notation

```
big := 123456789.0

"{big:e}"      // "1.23456789e8"
"{big:.2e}"    // "1.23e8"
"{big:E}"      // "1.23456789E8"
```

### Escaping braces

```
s := "Use \\{braces\\} for interpolation: {value}"
// Produces: Use {braces} for interpolation: 42
```

`\{` and `\}` produce literal braces inside string literals.

---

## 8. String Builder

For building strings incrementally (especially in loops), use `StringBuilder`:

```
mut builder := StringBuilder.new()

for item in items {
    builder.append("{item.name}: {item.value}\n")
}

result := builder.build()    // produces immutable str
```

### Builder API

| Method | Description |
|---|---|
| `StringBuilder.new()` | Create an empty builder |
| `StringBuilder.withCapacity(n)` | Create with pre-allocated capacity (bytes) |
| `b.append(s: str)` | Append a string |
| `b.appendChar(c: char)` | Append a single character |
| `b.appendLine(s: str)` | Append a string followed by `\n` |
| `b.len()` | Current byte length |
| `b.build() -> str` | Freeze into immutable str; consumes the builder |
| `b.clear()` | Reset without building (reuse the buffer) |

### Zero-copy build

`.build()` consumes the builder and produces an immutable `str`. If the built string is ≤ 23 bytes, it's stored inline (SSO). If longer, the builder's heap buffer becomes the string's backing storage directly — no copy.

After calling `.build()`, the builder is consumed and cannot be used. To build multiple strings, use `.clear()` to reset the builder or create a new one.

### Example: building a CSV row

```
fn formatRow(fields: [str]) -> str {
    mut b := StringBuilder.new()
    for (i, field) in fields.enumerate() {
        if i > 0 { b.append(",") }
        if field.contains(",") || field.contains("\"") {
            b.append("\"")
            b.append(field.replace("\"", "\"\""))
            b.append("\"")
        } else {
            b.append(field)
        }
    }
    b.build()
}
```

---

## 9. String Concatenation

The `+` operator creates a new string from two operands:

```
greeting := "Hello, " + name + "!"
```

### Compiler optimization

The compiler optimizes adjacent concatenations into a single allocation:

```
// The compiler sees three parts:
result := "Hello, " + name + "!"
// Compiles as: allocate(7 + name.len() + 1), copy all three parts

// NOT: allocate("Hello, " + name), then allocate(tmp + "!")
```

### Loop concatenation warning

Using `+` in a loop is O(n²) — each iteration copies the entire accumulated string. The compiler warns:

```
warning[W0150]: string concatenation in loop — consider using StringBuilder
  --> src/build.aria:5:9
  |
5 |     result = result + item.name
  |              ^^^^^^^^^^^^^^^^^^^ O(n²) — each iteration copies the entire string
  |
  = help: use StringBuilder for efficient loop concatenation
```

---

## 10. Multi-Line Strings

### Triple-quoted strings

`"""..."""` for multi-line string literals. Leading whitespace is automatically dedented based on the indentation of the closing `"""`:

```
message := """
    Hello, {name}!
    Welcome to Aria.
    Your score is {score}.
    """

// Equivalent to: "Hello, {name}!\nWelcome to Aria.\nYour score is {score}.\n"
```

Interpolation works inside triple-quoted strings. Extra indentation beyond the closing `"""` level is preserved:

```
sql := """
    SELECT *
    FROM users
    WHERE id = {userId}
      AND active = true
    """
// "SELECT *\nFROM users\nWHERE id = {userId}\n  AND active = true\n"
```

---

## 11. Raw Strings

`r"..."` for raw strings — no escape processing, no interpolation:

```
pattern := r"\d{4}-\d{2}-\d{2}"        // regex: backslashes are literal
path := r"C:\Users\aria\Documents"      // Windows path: literal backslashes

// r"Hello, {name}" — this is LITERAL "{name}", not interpolated
```

Raw string delimiter variants for strings containing quotes:

| Syntax | Allows |
|---|---|
| `r"..."` | No `"` inside |
| `r#"..."#` | Allows `"` inside |
| `r##"..."##` | Allows `"#` inside |

---

## 12. String Interning

### Compile-time interning

All string literals in source code are interned at compile time. Two occurrences of the same literal share the same backing memory:

```
a := "hello"
b := "hello"
// a and b reference the same interned bytes — zero additional allocation
```

Interned strings are placed in the old generation at program startup and are never collected by the GC.

### Runtime interning

Runtime strings (computed, read from files, etc.) are NOT interned by default. Explicit interning is available for deduplication:

```
key := computeKey().intern()    // intern the computed string
```

`s.intern()` checks a global intern table. If an equal string is already interned, the existing interned string is returned. If not, the string is added to the table.

**When to use runtime interning:**
- Symbol tables (compiler internals, interpreter variable names)
- Frequently compared strings (config keys, enum-like string values)
- Long-lived strings that appear many times (log field names)

**When NOT to use:** Request-scoped strings, temporary values, strings that appear once.

---

## 13. Interaction with Other Features

### Strings and error propagation

```
n := "42".parseInt[i64]()?          // Ok(42) or Err(ParseError)
offset := s.find("needle")?         // Some(7) or None
```

### Strings and pattern matching

```
match response.contentType {
    "application/json" => parseJson(response.body)?
    "text/plain" => response.body
    ct if ct.startsWith("text/") => response.body
    other => Err(UnsupportedMediaType(other))
}
```

### Strings and traits

`str` implements:
- `Eq` — byte-level equality (two strings are equal if their UTF-8 bytes are identical)
- `Ord` — lexicographic byte ordering
- `Hash` — hash of the UTF-8 bytes
- `Display` — returns itself
- `Debug` — returns quoted form with escapes
- `Clone` — copies the string value (inline strings copy 24 bytes; heap strings share backing memory via GC)
- `Iterable` — `type Item = char`, iterates over codepoints

### Strings and FFI

```
use ffi

c_str := ffi.toCString(s)       // null-terminated [u8], allocated in @cffi region
s2 := ffi.fromCString(ptr)?     // parse null-terminated *u8 as str (validates UTF-8)
s3 := ffi.fromCStringLossy(ptr) // replace invalid UTF-8 with replacement character
```

See [spec/ffi-design.md](ffi-design.md) for details.

### Strings and the pipeline operator

```
result := raw_input
    |> .trim()
    |> .toLower()
    |> .replace(" ", "-")
    |> .split("-")
    |> filter(fn(s) => !s.isEmpty())
    |> join("-")
```

---

## 14. Design Rationale Summary

| Decision | Rationale |
|---|---|
| One string type (`str`) | No `String` vs `&str` confusion — one type everywhere |
| UTF-8 always | Dominant encoding for web, APIs, files — no conversion overhead |
| Small string optimization (SSO, ≤23 bytes) | 60-80% of strings avoid heap allocation — massive GC pressure reduction |
| Byte indexing (`s[i]` returns `u8`) | O(1), honest about cost — no hidden O(n) behind array-index syntax |
| `s.len()` = byte length | O(1), most common use (buffer sizing, I/O) |
| `s.charCount()` = codepoint count | O(n), name reflects cost — never confused with byte length |
| Character boundary validation on slicing | Preserves UTF-8 invariant — no silent data corruption |
| `.chars()` for codepoint iteration | Explicit opt-in to character-level processing |
| `.graphemes()` for visual characters | Explicit opt-in to grapheme cluster processing |
| `for ch in s` iterates codepoints | Most useful default for text processing |
| Zero-copy substrings | Immutable backing memory shared between parent and substring |
| Compile-time literal interning | Deduplication of string constants — zero runtime cost |
| `StringBuilder` for loop construction | Avoids O(n²) concatenation — explicit mutable-then-freeze lifecycle |
| Compiler optimizes adjacent `+` | `"a" + b + "c"` compiles to single allocation |
| Panic on boundary-splitting slice | Better than silent corruption — error message shows the problem |

---

## 15. Comparison with Other Languages

| Feature | Go | Rust | Java | Python | Aria |
|---|---|---|---|---|---|
| String types | 1 (`string`) | 2 (`String`, `&str`) | 1 (`String`) | 1 (`str`) | **1 (`str`)** |
| Encoding | UTF-8 | UTF-8 | UTF-16 (compact) | UCS-4 / compact | **UTF-8** |
| `s[i]` returns | byte | compile error | UTF-16 code unit | codepoint | **byte (`u8`)** |
| `len()` returns | byte count | byte count | UTF-16 unit count | codepoint count | **byte count** |
| Bad slice boundary | silent corruption | panic | N/A (char-indexed) | N/A (char-indexed) | **panic with helpful message** |
| SSO | No | No (std), Yes (crates) | No (compact strings) | Interning only | **Yes (≤23 bytes)** |
| Literal interning | Partial | `&'static str` | Yes | Yes (small) | **Yes (all literals)** |
| Immutable | Yes | `&str` yes, `String` no | Yes | Yes | **Yes** |
| Builder | `strings.Builder` | `String` itself | `StringBuilder` | `list` + `join` | **`StringBuilder`** |

### Token cost: common string operations

| Operation | Go | Rust | Aria |
|---|---|---|---|
| String length (bytes) | `len(s)` | `s.len()` | `s.len()` |
| Character count | `len([]rune(s))` | `s.chars().count()` | `s.charCount()` |
| Nth character | `[]rune(s)[n]` | `s.chars().nth(n)` | `s.chars().nth(n)` |
| Contains | `strings.Contains(s, sub)` | `s.contains(sub)` | `s.contains(sub)` |
| Build in loop | `var b strings.Builder` + `b.WriteString()` | `let mut s = String::new()` + `s.push_str()` | `mut b := StringBuilder.new()` + `b.append()` |
| Interpolation | `fmt.Sprintf("x=%d", x)` | `format!("x={x}")` | `"x={x}"` |

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/formal-grammar.md](formal-grammar.md), [spec/type-conversions.md](type-conversions.md), and [spec/iteration-protocol.md](iteration-protocol.md).*
