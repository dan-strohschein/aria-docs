# Aria String Handling Specification

This document formally specifies string types, character handling, string interpolation, and text processing for the Aria v0.1 language. These design decisions should be considered part of the core language spec alongside [high-level-design.md](../high-level-design.md).

---

## `str` — The String Type

### Core Properties

- `str` is **always UTF-8 internally** and **codepoint-indexed externally**
- `str` is **immutable** — there is no `mut str`
- `s[i]` gives the *i*th Unicode codepoint as a `char`, never a byte
- `s.len()` returns the codepoint count, not the byte count
- `s[0..5]` slices by codepoint index, not byte offset

### Grammar

```
str_literal   := '"' ( char | escape | interpolation )* '"'
                | '"""' multiline_body '"""'
                | 'r"' raw_body '"'
interpolation := '{' expr ( ':' format_spec )? '}'
escape        := '\n' | '\t' | '\r' | '\\' | '\"' | '\{' | '\u{' hex_digit+ '}'
```

### Indexing and Slicing

```go
s := "Hello, 世界"

// Codepoint indexing — always a char
first := s[0]    // 'H' : char
kanji := s[7]    // '世' : char

// Range slicing — codepoint-based
greeting := s[0..5]   // "Hello"
world    := s[7..9]   // "世界"

// Slicing to end
rest := s[7..]    // "世界"
```

`s[i]` is `O(n)` — the string is stored as UTF-8 bytes, so reaching the *i*th codepoint requires a linear scan. This is an intentional tradeoff: sequential access is the common case, and the compiler can optimize sequential scans into a single pass. For hot paths requiring repeated random access, use `s.toChars()` to get a `[char]` array first.

### String Methods

| Method | Return Type | Description |
|---|---|---|
| `s.len()` | `i64` | Number of Unicode codepoints |
| `s.byteLen()` | `i64` | Number of UTF-8 bytes |
| `s.bytes()` | `[u8]` | Raw UTF-8 byte slice (opt-in byte layer) |
| `s.toChars()` | `[char]` | All codepoints as a list (enables O(1) random access) |
| `s.contains(sub)` | `bool` | Whether substring is present |
| `s.startsWith(prefix)` | `bool` | Prefix check |
| `s.endsWith(suffix)` | `bool` | Suffix check |
| `s.indexOf(sub)` | `Option[i64]` | Codepoint index of first match, or None |
| `s.split(sep)` | `[str]` | Split by separator |
| `s.trim()` | `str` | Remove leading/trailing whitespace |
| `s.trimStart()` | `str` | Remove leading whitespace |
| `s.trimEnd()` | `str` | Remove trailing whitespace |
| `s.toUpper()` | `str` | Unicode-aware uppercase |
| `s.toLower()` | `str` | Unicode-aware lowercase |
| `s.replace(old, new)` | `str` | Replace all occurrences |
| `s.lines()` | `[str]` | Split into lines (handles `\n`, `\r\n`, `\r`) |
| `s.repeat(n)` | `str` | Repeat string n times |
| `s.isEmpty()` | `bool` | True if `s.len() == 0` |

### Byte Layer (Opt-In)

Byte-level access is available but never the default. Use it for FFI, network protocols, or binary formats where bytes matter.

```go
s := "café"

// Codepoint layer (default)
s.len()      // 4
s[3]         // 'é' : char

// Byte layer (opt-in)
s.byteLen()  // 5 (é is 2 UTF-8 bytes)
s.bytes()    // [99, 97, 102, 195, 169] : [u8]
```

---

## `char` — The Character Type

### Core Properties

- `char` is a single Unicode codepoint (U+0000 to U+10FFFF)
- `char` is **distinct from `u8`** — no implicit conversion between them
- Literal syntax: `'A'`, `'é'`, `'世'`, `'\n'`, `'\u{1F600}'`

### Grammar

```
char_literal := "'" ( codepoint | char_escape ) "'"
char_escape  := '\n' | '\t' | '\r' | '\\' | "\'" | '\u{' hex_digit+ '}'
```

### `char` Methods

| Method | Return Type | Description |
|---|---|---|
| `c.isAlpha()` | `bool` | Unicode alphabetic |
| `c.isDigit()` | `bool` | Unicode decimal digit |
| `c.isAlphaNum()` | `bool` | Alphabetic or digit |
| `c.isWhitespace()` | `bool` | Unicode whitespace |
| `c.isUpper()` | `bool` | Uppercase letter |
| `c.isLower()` | `bool` | Lowercase letter |
| `c.toUpper()` | `char` | Uppercase codepoint |
| `c.toLower()` | `char` | Lowercase codepoint |
| `c.codepoint()` | `u32` | Numeric codepoint value |
| `c.toStr()` | `str` | Single-character string |

### `char` vs `u8`

```go
c: char = 'A'
b: u8   = 65

// Implicit conversion: COMPILE ERROR
// b = c  -- error: char is not u8

// Explicit conversion (only valid for ASCII codepoints 0–127)
b2: u8 = u8(c.codepoint())  // explicit narrowing; traps if value > 127

// char from codepoint
c2: char = char(65)    // 'A'
```

---

## String Interpolation

### Basic Interpolation

Wrap any expression in `{...}` inside a string literal. The expression is evaluated and formatted with its default display representation.

```go
name := "Aria"
version := 1

greeting := "Hello, {name}!"              // "Hello, Aria!"
info     := "Version: {version}"           // "Version: 1"
computed := "Sum: {1 + 2 + 3}"            // "Sum: 6"
method   := "Upper: {name.toUpper()}"     // "Upper: ARIA"
```

### Format Specifiers

A colon inside an interpolation introduces a format specifier. Format specifiers give precise control over number formatting, alignment, and presentation — with zero imports required.

#### Grammar

```
interpolation := '{' expr ( ':' format_spec )? '}'
format_spec   := fill? align? sign? '#'? '0'? width? ( '.' precision )? type?
fill          := any_char
align         := '<' | '^' | '>'
sign          := '+' | '-'
width         := integer | '{' expr '}'
precision     := integer | '{' expr '}'
type          := 'b' | 'd' | 'e' | 'E' | 'f' | 'o' | 'x' | 'X' | '%'
```

#### Precision

```go
pi := 3.14159265358979

"{pi:.4}"     // "3.1416"   — 4 decimal places (rounds)
"{pi:.2}"     // "3.14"
"{pi:.0}"     // "3"
```

#### Number Bases

```go
value := 255

"{value:#x}"   // "0xff"   — hex with 0x prefix
"{value:#X}"   // "0XFF"   — hex uppercase
"{value:#b}"   // "0b11111111"  — binary with 0b prefix
"{value:#o}"   // "0o377"  — octal with 0o prefix
"{value:x}"    // "ff"     — hex, no prefix
"{value:d}"    // "255"    — explicit decimal (default)
```

#### Padding and Alignment

```go
n := 42

"{n:>10}"    // "        42"  — right-aligned, width 10
"{n:<10}"    // "42        "  — left-aligned, width 10
"{n:^10}"    // "    42    "  — centered, width 10
"{n:0>5}"    // "00042"       — right-aligned, zero-padded
"{n:*^9}"    // "***42****"   — centered, star-padded

// Dynamic width from variable
w := 8
"{n:>{w}}"   // "      42"
```

#### Sign

```go
x := 42
y := -7

"{x:+}"   // "+42"
"{y:+}"   // "-7"
```

#### Percent

```go
ratio := 0.853

"{ratio:%}"    // "85.3%"    — multiplies by 100, appends %
"{ratio:.1%}"  // "85.3%"
"{ratio:.0%}"  // "85%"
```

#### Scientific Notation

```go
big := 123456789.0

"{big:e}"    // "1.23456789e8"
"{big:.2e}"  // "1.23e8"
"{big:E}"    // "1.23456789E8"
```

### Escaping Braces

```go
// To include a literal { or } in a string, double it
template := "Use {{braces}} like this: {value}"   // "Use {braces} like this: 42"

// To include a literal { in an interpolated expression, use \{
s := "x = \{literal brace}"  // "x = {literal brace}"
```

---

## String Builder

`str` is immutable, so concatenation of many strings avoids creating intermediate copies by using a builder.

### API

```go
// Create a builder
b := str.builder()

// Append values
b.add("Hello")
b.add(", ")
b.add(name)
b.add("!")

// Freeze into an immutable string
result := b.build()   // "Hello, Aria!" : str
```

The builder is mutable; the result of `.build()` is an immutable `str`. After calling `.build()`, the builder is consumed and cannot be used again.

### Additional Builder Methods

| Method | Description |
|---|---|
| `b.add(s: str)` | Append a string |
| `b.addChar(c: char)` | Append a single character |
| `b.addLine(s: str)` | Append a string followed by `\n` |
| `b.addFmt(s: str)` | Append a format string (interpolation evaluated at call site) |
| `b.len()` | Current codepoint count |
| `b.build() -> str` | Freeze into immutable string; consumes the builder |
| `b.clear()` | Reset without building (builder is still usable) |

### Example: Building a CSV Row

```go
fn formatRow(fields: [str]) -> str {
    b := str.builder()
    for (i, field) in fields.enumerate() {
        if i > 0 { b.add(",") }
        if field.contains(",") or field.contains('"') {
            b.add('"')
            b.add(field.replace('"', '""'))
            b.add('"')
        } else {
            b.add(field)
        }
    }
    b.build()
}
```

### Rationale: No `mut str`

Aria does not support `mut str`. A mutable string would introduce aliasing hazards — two references to the same string could observe intermediate states during modification. The builder pattern provides a clear lifecycle:

1. **Build phase**: mutable, local, not shareable
2. **Freeze**: `build()` produces an immutable, freely shareable `str`
3. **Use phase**: immutable, safe to pass anywhere, cache freely

This mirrors how other immutable-first languages (Swift's `String`/`NSMutableString`, Java's `StringBuilder`) handle the same problem, but without the ceremony of two separate types.

---

## Multi-Line Strings

### Triple-Quoted Strings

Use `"""..."""` for multi-line string literals. Leading whitespace is automatically dedented based on the indentation of the closing `"""`.

```go
// Whitespace from the closing """ indent level is stripped
message := """
    Hello, {name}!
    Welcome to Aria.
    Your score is {score}.
    """

// Equivalent to:
// "Hello, {name}!\nWelcome to Aria.\nYour score is {score}.\n"
```

Interpolation works inside triple-quoted strings. Dedentation removes the uniform leading whitespace; any extra indentation beyond the closing `"""` level is preserved.

```go
sql := """
    SELECT *
    FROM users
    WHERE id = {userId}
      AND active = true
    """
// "SELECT *\nFROM users\nWHERE id = {userId}\n  AND active = true\n"
```

### Raw Strings

Use `r"..."` for raw strings — no escape processing and no interpolation. Useful for regular expressions, Windows file paths, and any literal that would otherwise require heavy escaping.

```go
pattern := r"\d{4}-\d{2}-\d{2}"        // regex: no need to escape backslashes
path    := r"C:\Users\aria\Documents"   // Windows path: backslashes are literal
json    := r#"{"key": "value"}"#        // raw string with # delimiter to allow "

// raw strings cannot contain interpolation
// r"Hello, {name}"  -- this is LITERAL "{name}", not interpolated
```

Raw string delimiter variants:

| Syntax | Allows |
|---|---|
| `r"..."` | No `"` inside |
| `r#"..."#` | Allows `"` inside, but not `"#` |
| `r##"..."##` | Allows `"#` inside, but not `"##` |

---

## Interaction with Other Features

### Strings and the `?` Operator

String parsing functions return `Result` or `Option`, compatible with `?`:

```go
n := "42".parse[i64]()?         // Ok(42) or Err(ParseError)
c := "hello".indexOf("ll")?     // Some(2) or None (unwrap with ?)
```

### Strings and Pattern Matching

```go
match response.contentType {
    "application/json" => parseJson(response.body)?
    "text/plain"       => response.body
    other              => Err(UnsupportedMediaType(other))
}
```

### Strings and the FFI

When calling C functions, convert between `str` and C strings explicitly:

```go
use ffi

c_str := ffi.toCString(s)      // null-terminated [u8]
s2    := ffi.fromCString(ptr)? // parse null-terminated *u8 as str (validates UTF-8)
```

See [ffi-design.md](ffi-design.md) for details.

---

## Design Rationale

### One Mental Model

`s[i]` always means the *i*th character. `s.len()` always means character count. There is no mode, flag, or import needed to get character-level semantics — they are the default. This eliminates an entire class of bugs that appear when character-indexed and byte-indexed operations are mixed accidentally.

### Zero Conversion Tokens

In Go: `[]rune(s)[i]`, `len([]rune(s))`.  
In Rust: `s.chars().nth(i)`, `s.chars().count()`.  
In Aria: `s[i]`, `s.len()`.

The common case requires zero extra tokens. The byte layer is opt-in (`s.bytes()`, `s.byteLen()`), not the mandatory foundation everything else is built on top of.

### O(n) Indexing Tradeoff

Random access by codepoint index is `O(n)` because UTF-8 variable-width encoding requires scanning from the start. This is an accepted tradeoff because:

1. Random access by integer index into a string is rare in correct programs. Most string processing is sequential (iteration, splitting, pattern matching).
2. When random access is genuinely needed, `s.toChars()` converts to a `[char]` array, giving `O(1)` indexing at the cost of one upfront allocation.
3. Sequential iteration (`for c in s`) is `O(n)` total, not `O(n²)`, because the iteration maintains a cursor.

### No Grapheme Cluster Default

Aria indexes by codepoint, not grapheme cluster. Grapheme clusters (user-perceived characters like "é" as a base character + combining accent) are useful for display and text editing, but the rules are complex, locale-dependent, and Unicode-version-dependent. Grapheme cluster support is available via the `text` stdlib module for code that needs it.

| Layer | Access | Use Case |
|---|---|---|
| Bytes | `s.bytes()` | FFI, network, binary formats |
| Codepoints | `s[i]`, `s.len()` | General string processing (default) |
| Grapheme clusters | `text.graphemes(s)` | Display, text editing, user-facing measurement |
