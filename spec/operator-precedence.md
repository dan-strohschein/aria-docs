# Aria v0.1 — Operator Precedence & Associativity

> This document defines the complete operator precedence table for Aria v0.1.
> Operators at **higher levels** bind tighter (are evaluated first).
> All operators at the same level have the same precedence and follow the stated associativity rule.
> This document is authoritative; where narrative spec documents are ambiguous, this table takes precedence.

---

## 1. Precedence Table

| Level | Operators | Description | Associativity |
|------:|-----------|-------------|---------------|
| 16 | `.` &nbsp; `?.` &nbsp; `[idx]` &nbsp; `(args)` &nbsp; `.{...}` | Field access, optional chaining, index, call, record update | Left |
| 15 | `?` &nbsp; `!` (postfix) | Error propagation, assert-success | Postfix (left) |
| 14 | `-` (prefix) &nbsp; `!` (prefix) &nbsp; `~` | Unary negation, logical NOT, bitwise NOT | Right (prefix) |
| 13 | `*` &nbsp; `/` &nbsp; `%` | Multiplication, division, modulo | Left |
| 12 | `+` &nbsp; `-` | Addition, subtraction | Left |
| 11 | `<<` &nbsp; `>>` | Bitwise shift left/right | Left |
| 10 | `&` | Bitwise AND | Left |
| 9 | `^` | Bitwise XOR | Left |
| 8 | `\|` (bitwise) | Bitwise OR | Left |
| 7 | `..` &nbsp; `..=` | Half-open range, closed range | Non-associative |
| 6 | `==` &nbsp; `!=` &nbsp; `<` &nbsp; `>` &nbsp; `<=` &nbsp; `>=` | Comparison | Non-associative |
| 5 | `&&` | Logical AND (short-circuit) | Left |
| 4 | `\|\|` | Logical OR (short-circuit) | Left |
| 3 | `??` | Null coalesce (Option unwrap with default) | Left |
| 2 | `\|>` | Pipeline | Left |
| 1 | `:=` &nbsp; `=` | Variable binding, assignment | Non-associative |

> **Reading the table:** Level 16 is highest precedence (evaluated first). Level 1 is lowest (evaluated last). `a + b * c` is `a + (b * c)` because `*` is level 13 and `+` is level 12.

---

## 2. Detailed Rules

### Level 16 — Postfix Access: `.`, `?.`, `[idx]`, `(args)`, `.{...}`

These operators bind the tightest because they transform a value by accessing its parts or calling it.

**Field access and method call (`.`):**
- `a.b` — accesses field `b` on `a`
- `a.b()` — calls method `b` on `a` with no arguments
- `a.b(x, y)` — calls method `b` with arguments `x` and `y`
- Left-associative: `a.b.c` means `(a.b).c`

**Optional chaining (`?.`):**
- `a?.b` — if `a` is `Some(v)`, evaluates to `Some(v.b)`; if `a` is `None`, evaluates to `None`
- Type: `Option[T]` where `T` is the type of `b`
- Chains: `user?.address?.city` — evaluates to `Option[str]`

**Index access (`[idx]`):**
- `a[i]` — accesses element at index `i` in array or map `a`
- For arrays: `i` must be `usize`; out-of-bounds is a runtime panic in debug, undefined behavior in release unless bounds checking is enabled
- For maps: returns `Option[V]`; use `!` to assert presence
- Left-associative: `a[i][j]` means `(a[i])[j]`

**Function/method call (`(args)`):**
- `f(x, y)` — calls `f` with arguments `x` and `y`
- Named arguments: `f(timeout: 30s)` — matched by name
- Trailing comma is allowed: `f(a, b,)`

**Record update (`.{...}`):**
- `a.{field: newValue}` — creates a copy of `a` with the specified fields replaced
- All other fields retain their original values
- Returns a new value of the same type as `a`; `a` is unchanged

```aria
user.{ age: 30, email: "new@example.com" }
```

---

### Level 15 — Postfix `?` and `!`

These operators apply to a complete postfix expression (after all `.`, `[...]`, `(...)` operators have been applied).

**Error propagation (`?`):**
- On `Result[T, E]` — if the value is `Ok(v)`, unwraps to `v`; if `Err(e)`, returns `Err(e)` from the enclosing function
- On `Option[T]` — if the value is `Some(v)`, unwraps to `v`; if `None`, returns `None` from the enclosing function
- The enclosing function's return type must be compatible (must itself return `Result` or `Option`)
- `?` binds tighter than arithmetic: `read()? + 1` means `(read()?) + 1`

```aria
fn process() -> str ! IoError {
    content := fs.read("file.txt")?    // propagates IoError
    content.trim()?                    // propagates if trim returns Result
}
```

**Assert-success (`!`):**
- On `Result[T, E]` — unwraps to `T`, panicking at runtime if the value is `Err(e)`
- On `Option[T]` — unwraps to `T`, panicking at runtime if the value is `None`
- Use only when you are certain the operation cannot fail, or when a panic is acceptable
- Panic message includes the file and line number

```aria
data := fs.read("known-good.txt")!   // panics if file missing
```

> **Design Decision:** `?` and `!` are both postfix and both bind at level 15, which is above all binary operators. This means `read()! + 1` is `(read()!) + 1`, not `read()!(+ 1)`. This is the most natural reading and consistent with Rust's `?` operator.

---

### Level 14 — Prefix Unary: `-`, `!`, `~`

**Unary negation (`-`):**
- `-x` — numeric negation; type of result equals type of operand
- Valid on: `i8`, `i16`, `i32`, `i64`, `f32`, `f64`
- NOT valid on unsigned integers (`u8`, `u16`, `u32`, `u64`) — use explicit casting if needed
- Right-associative: `--x` means `-(-x)` (double negation)

**Logical NOT (`!`):**
- `!b` — logical NOT; operand and result must be `bool`
- No truthy/falsy coercion: `!0` is a compile error
- Right-associative: `!!b` means `!(!b)` (double NOT, which simplifies to `b`)

**Bitwise NOT (`~`):**
- `~n` — bitwise complement; type of result equals type of operand
- Valid on all integer types: `i8`..`i64`, `u8`..`u64`

> **Design Decision:** `!` serves double duty — prefix logical NOT (level 14) and postfix assert-success (level 15). The parser disambiguates purely by position: `!` at the start of a `unary_expr` is prefix; `!` after a complete `postfix_expr` is postfix. There is no ambiguity because postfix operators are left-recursive and prefix operators are right-recursive.

---

### Level 13 — Multiplicative: `*`, `/`, `%`

- Left-associative: `a * b / c` means `(a * b) / c`
- Integer division truncates toward zero (same as C, Go, Rust)
- Integer modulo has the same sign as the dividend: `-7 % 3 == -1`
- No implicit promotion: `i64 * f64` is a compile error; use explicit conversion
- Division by zero: runtime panic in debug mode; undefined behavior in release unless safety checks are enabled

---

### Level 12 — Additive: `+`, `-`

- Left-associative: `a + b - c` means `(a + b) - c`
- `+` on strings is NOT valid — use string interpolation or `str.concat`
- Integer overflow: wraps in release mode, panics in debug mode (same as Rust's debug mode behavior)

---

### Level 11 — Bitwise Shift: `<<`, `>>`

- Left-associative
- `<<` — shift left; fills with zeros
- `>>` — shift right; for signed types, arithmetic shift (fills with sign bit); for unsigned types, logical shift (fills with zeros)
- Shift amount must be a non-negative integer; shift by >= bit width is a compile error if statically known, a runtime panic otherwise

---

### Level 10 — Bitwise AND: `&`

- Left-associative
- Valid on all integer types; both operands must have the same type

---

### Level 9 — Bitwise XOR: `^`

- Left-associative
- Valid on all integer types; both operands must have the same type

---

### Level 8 — Bitwise OR: `|`

- Left-associative
- Valid on all integer types; both operands must have the same type
- **Disambiguation from `|` in type declarations:** In a `type` body, `|` introduces a sum-type variant. In expression context, `|` is always bitwise OR. The parser uses context (inside `type` declaration body vs. inside an expression) to disambiguate.

---

### Level 7 — Range: `..`, `..=`

- **Non-associative**: `a..b..c` is a compile error at parse time
- `0..10` — half-open range `[0, 10)`, i.e., includes 0, excludes 10
- `0..=9` — closed range `[0, 9]`, i.e., includes both 0 and 9
- Both operands must have the same integer type
- Produces a `Range[T]` value; usable in `for` loops and slice expressions

```aria
for i in 0..10 { ... }    // i takes values 0, 1, ..., 9
xs[2..=5]                  // slice of xs from index 2 to 5 inclusive
```

---

### Level 6 — Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`

- **Non-associative**: `a < b < c` is a compile error (must write `a < b && b < c`)
- `==` and `!=` require the `Eq` trait on the operand type
- `<`, `>`, `<=`, `>=` require the `Ord` trait on the operand type
- `==` is **structural equality** for value types (compares field-by-field)
- For custom types, derive `Eq` and `Ord` or implement them manually
- No pointer equality (`===`) — there are no pointers in safe Aria

```aria
// Compile error — non-associative:
if a < b < c { ... }

// Correct:
if a < b && b < c { ... }
```

---

### Level 5 — Logical AND: `&&`

- Left-associative
- **Short-circuit**: right side is not evaluated if left side is `false`
- Both operands must be `bool`
- Result type: `bool`

---

### Level 4 — Logical OR: `||`

- Left-associative
- **Short-circuit**: right side is not evaluated if left side is `true`
- Both operands must be `bool`
- Result type: `bool`

---

### Level 3 — Null Coalesce: `??`

- Left-associative: `a ?? b ?? c` means `(a ?? b) ?? c`
- Only valid on `Option[T]`
- `a ?? b` — if `a` is `Some(v)`, evaluates to `v` (type `T`); if `a` is `None`, evaluates to `b` (also type `T`)
- Right operand (`b`) must have type `T` (not `Option[T]`)
- Short-circuit: `b` is not evaluated if `a` is `Some(v)`

```aria
name := findUser(id)?.name ?? "anonymous"
config := loadConfig() ?? Config{}
```

---

### Level 2 — Pipeline: `|>`

- Left-associative: `x |> f |> g` means `g(f(x))`
- `x |> f` desugars to `f(x)` — `x` becomes the first argument
- `x |> f(a, b)` desugars to `f(x, a, b)` — `x` is prepended to the argument list
- `x |> f?` desugars to `f(x)?` — error propagation applies after the call
- Lower precedence than all binary operators: `a + b |> f` means `f(a + b)`
- Higher precedence than assignment: `result := x |> f` means `result := (x |> f)`
- **Pipeline NEVER terminates at a newline**: a trailing `|>` always means continuation on the next line

**`.field` shorthand:**

In pipeline context, `.field` is shorthand for `fn(x) => x.field`:

```aria
users |> filter(.active) |> map(.name)
// equivalent to:
users |> filter(fn(u) => u.active) |> map(fn(u) => u.name)
```

This shorthand is ONLY valid immediately after `|>`. In all other positions, `.field` is a syntax error.

```aria
// Typical pipeline — reads top-to-bottom, left-to-right
result := rawData
    |> parseCSV?
    |> filter(.valid)
    |> map(fn(row) => row.{total: row.price * row.qty})
    |> sortBy(.total)
```

---

### Level 1 — Binding and Assignment: `:=`, `=`

- **Non-associative**: chained binding (`a := b := c`) is a compile error
- `:=` — variable binding (creates a new binding in the current scope)
- `=` — assignment to an existing `mut` variable or mutable field
- These have the lowest precedence: everything on the right of `:=` or `=` is evaluated first

```aria
x := 5           // immutable binding
mut y := 10      // mutable binding
y = y + 1        // assignment to mutable
z := a + b * c   // z := (a + (b * c))
```

---

## 3. Parsing Strategy

### Recommended Approach: Pratt Parsing (Top-Down Operator Precedence)

The recommended implementation approach for Aria's expression parser is **Pratt parsing** (also called "top-down operator precedence parsing" or "precedence climbing"). This approach:

- Handles all the precedence levels in the table above with a simple, recursive algorithm
- Easily supports both prefix (nud) and infix (led) positions for operators
- Handles non-associativity by adjusting the binding power
- Is fast (O(n) in expression length)
- Is easy to extend with new operators

**Binding powers for Pratt parser:**

| Operator | Left BP | Right BP | Notes |
|----------|---------|----------|-------|
| `.`, `?.`, `[`, `(`, `.{` | 160 | 161 | Postfix; left-recursive |
| `?`, `!` (postfix) | 150 | — | Postfix |
| `-`, `!`, `~` (prefix) | — | 140 | Prefix only |
| `*`, `/`, `%` | 130 | 131 | Left-assoc |
| `+`, `-` | 120 | 121 | Left-assoc |
| `<<`, `>>` | 110 | 111 | Left-assoc |
| `&` | 100 | 101 | Left-assoc |
| `^` | 90 | 91 | Left-assoc |
| `\|` (bitwise) | 80 | 81 | Left-assoc |
| `..`, `..=` | 70 | 70 | Non-assoc (same BP left and right → error if chained) |
| `==`, `!=`, `<`, `>`, `<=`, `>=` | 60 | 60 | Non-assoc |
| `&&` | 50 | 51 | Left-assoc |
| `\|\|` | 40 | 41 | Left-assoc |
| `??` | 30 | 31 | Left-assoc |
| `\|>` | 20 | 21 | Left-assoc |
| `:=`, `=` | 10 | 10 | Non-assoc |

> For non-associative operators, use the same value for left and right binding power. When the parser sees the same operator again at the same level, it treats it as an error (cannot apply the operator again without a lower-precedence context).

### Handling the `!` Prefix/Postfix Ambiguity

- In the **nud** (null denotation) handler: `!` is prefix logical NOT (binding power 140)
- In the **led** (left denotation) handler: `!` is postfix assert-success (binding power 150)
- The Pratt parser naturally handles this — nud is called when `!` appears where a value is expected (beginning of expression); led is called when `!` appears after a complete value

### Distinguishing Generic `[T]` from Index `[0]`

Aria uses `[...]` for both generic type parameters and array indexing. The parser resolves this with a lookahead rule:

- `[` immediately after a **type name** (IDENT in type position) → generic parameter list
- `[` immediately after a **complete expression** → index access
- In function calls: `f[T](x)` → `f` with generic type `T` applied, then called with `x`

Since generics use `[` and `]` (not `<` and `>`), there is no scanner-level ambiguity; the parser resolves by context.

### Handling `{` — Block vs. Struct Literal

- `{` immediately after an IDENT at **expression start** → struct literal: `Point{x: 1}`
- `{` after a **keyword** (`if`, `else`, `for`, `while`, `loop`, `match`, `scope`, `entry`, `test`) → block
- `{` after any **operator** → block expression (e.g., `x := { ... }`)
- `{` at the **start of a statement** → block expression

If the parser sees `IDENT {` and the IDENT is a known type name, it parses as a struct literal. If the parser sees `IDENT {` and the IDENT is a variable or function name, it parses the IDENT as an expression statement and `{` as the start of the next block statement.

> **Design Decision:** Struct literal syntax `TypeName{...}` requires the type name to be directly adjacent to `{` with no space between them at the parser level — or rather, whitespace is insignificant but the grammatical rule is that `{` must immediately follow an IDENT in the `struct_expr` production. The parser must use one token of lookahead to make this determination.

---

## 4. Interaction with Newlines

The pipeline operator `|>` specifically interacts with the newline termination rules. See [`formal-grammar.md §5`](formal-grammar.md#5-automatic-semicolon-insertion-statement-termination) for the complete rules. The key rules for operators:

**A newline is NOT a statement terminator when the last token on the current line is:**

| Token | Example |
|-------|---------|
| Any binary operator | `a + ` (next line continues) |
| `\|>` | `x \|>` (pipeline continues) |
| `:=` or `=` | `result :=` (right-hand side on next line) |
| `=>` | `arm =>` (match/closure body on next line) |
| `->` | `fn f() ->` (return type on next line) |
| `,` | `f(a,` (argument list continues) |
| `.` | `a.` (method chain continues) |
| `(`, `[`, `{` | Balanced delimiter — all newlines inside are ignored |
| `??`, `..`, `..=` | Expression continues |

**A newline IS a statement terminator when the last token is:**

| Token | Example |
|-------|---------|
| IDENT | `x` (end of expression referencing a variable) |
| Any literal | `42`, `"hello"`, `true` |
| `)`, `]`, `}` | End of grouping/block/array |
| `?`, `!` (postfix) | End of propagation/assert chain |
| `break`, `continue`, `return` | Control flow (even without a value) |

---

## 5. Worked Examples

### Example 1: Arithmetic Precedence

```aria
result := 2 + 3 * 4 - 1
// Parsed as: 2 + (3 * 4) - 1 = 2 + 12 - 1 = 13
// Step by step:
//   3 * 4 = 12        (level 13)
//   2 + 12 = 14       (level 12, left-assoc)
//   14 - 1 = 13       (level 12, left-assoc)
```

### Example 2: Comparison and Logical

```aria
valid := age >= 18 && score > 50 || vip
// Parsed as: (age >= 18 && score > 50) || vip
// Step by step:
//   age >= 18          (level 6)
//   score > 50         (level 6)
//   (age >= 18) && (score > 50)   (level 5)
//   (...) || vip                   (level 4)
```

### Example 3: Postfix Operators

```aria
result := fs.read("file.txt")?.trim()!
// Parsed as: ((fs.read("file.txt"))?.trim())!
// Step by step:
//   fs.read("file.txt")        (level 16: method call)
//   (...)?                     (level 15: propagate None)
//   (...).trim()               (level 16: method call on unwrapped str?)
//   (...)!                     (level 15: assert not None)
```

### Example 4: Pipeline

```aria
result := input |> parse? |> validate |> transform
// Parsed as: transform(validate(parse(input)?))
// Left-associative:
//   input |> parse?           =>  parse(input)?
//   (parse(input)?) |> validate  =>  validate(parse(input)?)
//   (...) |> transform           =>  transform(validate(parse(input)?))
```

### Example 5: Mixed Pipeline and Arithmetic

```aria
total := prices |> map(fn(p) => p * 1.1) |> sum
// Parsed as: sum(map(prices, fn(p) => p * 1.1))
// Arithmetic p * 1.1 is inside the closure, separate from pipeline precedence
```

### Example 6: Null Coalesce

```aria
name := user?.profile?.displayName ?? user?.name ?? "anonymous"
// Parsed as: ((user?.profile?.displayName) ?? (user?.name)) ?? "anonymous"
// Left-associative, so evaluated left-to-right:
//   user?.profile?.displayName — optional chaining (level 16)
//   (...) ?? (user?.name)      — coalesce (level 3)
//   (...) ?? "anonymous"       — coalesce (level 3)
```
