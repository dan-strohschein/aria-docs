# Aria v0.1 — Formal Grammar

> This document defines the complete formal grammar of Aria v0.1 using Extended Backus–Naur Form (EBNF).
> It is the authoritative reference for parser implementors. Where narrative spec documents conflict with this grammar, this grammar takes precedence.

---

## 1. Notation

- `=` defines a production rule
- `|` separates alternatives
- `( )` groups sub-expressions
- `[ ]` denotes optional elements (zero or one)
- `{ }` denotes repetition (zero or more)
- `" "` denotes terminal strings (keywords, punctuation)
- `'...'` denotes terminal characters or character ranges
- `(* comment *)` are grammar comments
- `UPPER_CASE` names are token-level (lexer) productions
- `lower_case` names are parser-level productions

---

## 2. Lexical Grammar

### 2.1 Keywords

All reserved words — they CANNOT be used as identifiers:

```
mod      use      pub      fn       type     enum     trait    impl     struct
entry    if       else     match    for      in       while    loop     break
continue return   mut      const    spawn    scope    select   from     after
defer    with     catch    yield    test     assert   derives  where    as
is       self     Self     true     false
```

> **Design Decision:** `Self` (capital S) refers to the type of `self` in trait and impl bodies, while `self` (lowercase) is the instance receiver. Both are reserved to prevent confusion. `yield` is reserved for future generator support. `is` is reserved for potential type-test syntax.

### 2.2 Identifiers

```
IDENT   = LETTER { LETTER | DIGIT | '_' } ;
LETTER  = 'a'..'z' | 'A'..'Z' | '_' ;
DIGIT   = '0'..'9' ;
```

- Identifiers beginning with `_` are permitted but conventionally mean "intentionally unused."
- Keywords listed in §2.1 are not valid identifiers.

### 2.3 Literals

**Integer literals** — decimal, hexadecimal, octal, binary, with optional underscore separators:

```
INT_LIT      = DECIMAL_LIT | HEX_LIT | OCT_LIT | BIN_LIT ;
DECIMAL_LIT  = DIGIT { DIGIT | '_' } ;
HEX_LIT      = "0x" HEX_DIGIT { HEX_DIGIT | '_' } ;
OCT_LIT      = "0o" OCT_DIGIT { OCT_DIGIT | '_' } ;
BIN_LIT      = "0b" BIN_DIGIT { BIN_DIGIT | '_' } ;

HEX_DIGIT    = DIGIT | 'a'..'f' | 'A'..'F' ;
OCT_DIGIT    = '0'..'7' ;
BIN_DIGIT    = '0' | '1' ;
```

**Float literals:**

```
FLOAT_LIT    = DIGIT { DIGIT | '_' } '.' DIGIT { DIGIT | '_' } [ EXPONENT ] ;
EXPONENT     = ( 'e' | 'E' ) [ '+' | '-' ] DIGIT { DIGIT } ;
```

> **Design Decision:** A float literal must have at least one digit on each side of the `.` (so `.5` and `5.` are not valid). This simplifies the lexer: when scanning `1.`, the lexer can immediately determine whether a digit follows (float literal) or not (integer literal followed by a `.` field-access operator), without ambiguity.

**String literals** — UTF-8, with escape sequences and interpolation:

```
STRING_LIT    = '"' { STRING_CHAR | INTERPOLATION } '"' ;
STRING_CHAR   = (* any Unicode scalar value except '"', '\', '{', '}' *)
              | ESCAPE_SEQ ;
ESCAPE_SEQ    = '\' ( 'n' | 'r' | 't' | '\' | '"' | '{' | '}' | '0' ) ;
INTERPOLATION = '{' expression '}' ;
```

- `\{` and `\}` are the escapes for literal braces inside strings.
- Interpolated expressions `{expr}` are full Aria expressions; they may not contain unescaped `"` or `{`/`}` unless properly nested.
- Multi-line strings are permitted; newlines inside string literals are literal newline characters.

**Bool literals:**

```
BOOL_LIT = "true" | "false" ;
```

**Duration literals:**

```
DURATION_LIT    = DECIMAL_LIT DURATION_SUFFIX ;
DURATION_SUFFIX = "ns" | "us" | "ms" | "s" | "m" | "h" ;
```

> **Design Decision:** Duration suffixes are lexed as part of the literal token, not as a separate identifier, so `30s` is a single `DURATION_LIT` token of type `dur`. The type `dur` is a built-in alias for a nanosecond-precision integer.

**Size literals** — for arena/pool/buffer allocation sizes:

```
SIZE_LIT    = DECIMAL_LIT SIZE_SUFFIX ;
SIZE_SUFFIX = "b" | "kb" | "mb" | "gb" ;
```

> **Design Decision:** Size suffixes are similarly lexed as part of the token. `64kb` is a single `SIZE_LIT` token. Size literals have type `usize`.

### 2.4 Operators and Punctuation

**Arithmetic:**

```
+   -   *   /   %
```

**Comparison:**

```
==   !=   <   >   <=   >=
```

**Logical:**

```
&&   ||   !
(* Note: '!' is also used as a postfix assert-success operator *)
```

**Bitwise:**

```
&   |   ^   ~   <<   >>
(* Note: '|' is also used in sum-type declarations and match arms — see §4 for disambiguation *)
```

**Pipeline:**

```
|>
```

**Error propagation / optional chaining:**

```
?    (* postfix: propagate error/None *)
?.   (* optional chaining: propagate None through field access *)
??   (* null coalesce: provide default for None *)
```

**Range:**

```
..    (* half-open range [a, b) *)
..=   (* closed range [a, b] *)
```

**Binding and assignment:**

```
:=   (* immutable or mutable variable binding *)
=    (* assignment to mutable variable; also single-expression fn body *)
=>   (* match arm body; closure body *)
->   (* return type annotation *)
```

**Access and update:**

```
.    (* field access, method call, path separator within an expression *)
.{   (* record update syntax: expr.{ field: newValue } *)
```

**Annotations:**

```
@    (* precedes annotation names: @stack, @arena, @inline *)
```

**Module path separator** (import paths only):

```
.    (* reused — in import_path context, '.' separates path segments *)
```

**Delimiters:**

```
(  )   (* grouping, tuple literals, function parameters *)
[  ]   (* generics, array types, array literals, list comprehension *)
{  }   (* blocks, struct bodies, map literals, string interpolation *)
,      (* separator in parameters, fields, arguments, lists *)
:      (* type annotation separator; also map key/value separator *)
;      (* NOT used — Aria has no semicolons *)
```

### 2.5 Comments

```
LINE_COMMENT = "//" { (* any Unicode scalar except newline *) } NEWLINE ;
(* Block comments are NOT supported in Aria v0.1 *)
```

### 2.6 Whitespace and Newlines

```
WHITESPACE = ' ' | '\t' ;
NEWLINE    = '\n' | '\r' '\n' ;
```

- Whitespace (space, tab) is insignificant between tokens except as a token separator.
- Newlines are significant as implicit statement terminators — see §5 for the complete rules.
- The lexer skips whitespace and comments; they produce no tokens.

---

## 3. Syntactic Grammar

### 3.1 Program Structure

```
program     = module_decl { import_decl } { top_level_decl } EOF ;
module_decl = "mod" IDENT NEWLINE ;
```

> **Design Decision:** Every source file must begin with a `mod` declaration. This is required (not optional), so the module name is always explicit. There is no implicit "main" module.

### 3.2 Import Declarations

```
import_decl = "use" import_path NEWLINE ;
import_path = IDENT { "." IDENT }
            | IDENT { "." IDENT } "." "{" IDENT { "," IDENT } "}" ;
```

Examples:
- `use std.fs` — imports the `fs` module from `std`
- `use std.{json, http}` — imports `json` and `http` from `std`
- `use crypto.sha256 as sha` — alias import (see `as` keyword support below)

```
import_decl = "use" import_path [ "as" IDENT ] NEWLINE ;
```

### 3.3 Top-Level Declarations

```
top_level_decl = fn_decl
               | type_decl
               | enum_decl
               | trait_decl
               | impl_decl
               | const_decl
               | entry_block
               | test_block ;
```

All top-level declarations may be optionally preceded by a visibility modifier:

```
visibility = "pub" | "pub" "(" "pkg" ")" ;
```

- `pub` — visible to any importer of this module
- `pub(pkg)` — visible only within the same package (directory)
- No modifier — module-private (default)

### 3.4 Function Declarations

```
fn_decl = [ visibility ] "fn" IDENT [ generic_params ]
          "(" [ param_list ] ")" [ "->" type ] [ error_clause ] [ effect_clause ]
          fn_body ;

fn_body  = "=" expression NEWLINE    (* single-expression form *)
         | block ;                   (* block body form *)

param_list = param { "," param } [ "," ] ;
param      = [ "mut" ] pattern ":" type [ "=" expression ] ;
           | IDENT ":" type [ "=" expression ] ;

error_clause  = "!" type { "|" type } ;
effect_clause = "with" "[" IDENT { "," IDENT } "]" ;
```

Examples:
```aria
fn greet(name: str) = print("Hello, {name}")

fn readFile(path: str) -> str ! IoError with [Io, Fs] {
    fs.read(path)?
}

pub fn map[T, U](list: [T], f: T -> U) -> [U] {
    [f(x) for x in list]
}
```

### 3.5 Type Declarations

```
type_decl = [ visibility ] "type" IDENT [ generic_params ] "="
            type_body [ derives_clause ] ;

type_body   = sum_variants   (* sum / tagged union type *)
            | struct_body ;  (* struct type *)

sum_variants = "|" variant { "|" variant } ;
variant      = IDENT [ struct_body | "(" type_list ")" ] ;

struct_body  = "{" [ field_list ] "}" ;
field_list   = field_decl { "," field_decl } [ "," ] ;
field_decl   = [ "pub" ] IDENT ":" type [ "=" expression ] ;

derives_clause = "derives" "[" IDENT { "," IDENT } "]" ;
```

> **Design Decision:** `struct` keyword is supported as syntactic sugar: `struct Name { ... }` desugars to `type Name = Name { ... }`. Both forms are valid and produce identical types.

Examples:
```aria
type Shape =
    | Circle { radius: f64 }
    | Rect   { w: f64, h: f64 }
    | Point

type User {
    name:  str
    email: str
    age:   u8 = 0
} derives [Eq, Hash, Json, Debug]
```

### 3.6 Enum Declarations

```
enum_decl = [ visibility ] "enum" IDENT "{" enum_body "}" ;
enum_body = IDENT { "," IDENT } [ "," ] ;
```

> **Design Decision:** `enum` is for simple C-style enumerations (no payload). For variants with data, use `type` with sum type syntax. This distinction keeps the grammar unambiguous and the syntax transparent.

Example:
```aria
enum Color { Red, Green, Blue }
```

### 3.7 Trait Declarations

```
trait_decl = [ "pub" ] "trait" IDENT [ generic_params ] [ trait_bounds ]
             "{" { trait_item } "}" ;

trait_item = fn_signature ;
fn_signature = "fn" IDENT [ generic_params ]
               "(" [ param_list ] ")" [ "->" type ] [ error_clause ] [ effect_clause ] ;

trait_bounds = ":" trait_bound ;
trait_bound  = IDENT { "+" IDENT } ;
```

> **Design Decision:** Traits cannot have default method implementations in Aria v0.1. This avoids the diamond problem and keeps the semantics predictable. Every `impl` block must implement all methods. (A future version may allow `default fn` for non-overlapping methods.)

Example:
```aria
trait Serializable {
    fn serialize(self) -> [byte]
    fn deserialize(data: [byte]) -> Self ! DecodeError
}
```

### 3.8 Impl Blocks

```
impl_decl = "impl" [ generic_params ] IDENT "for" type [ where_clause ]
            "{" { fn_decl } "}" ;
```

Example:
```aria
impl Serializable for User {
    fn serialize(self) -> [byte] = encode(self)
    fn deserialize(data: [byte]) -> User ! DecodeError = decode(data)
}
```

### 3.9 Entry Block

```
entry_block = "entry" block ;
```

- There must be exactly one `entry` block per program (across all modules).
- The `entry` block is the program entry point.
- It may not appear inside any other declaration.

Example:
```aria
entry {
    greet("Aria")
}
```

### 3.10 Test Blocks

```
test_block = "test" ( IDENT | STRING_LIT ) block ;
```

Example:
```aria
test fibonacci {
    assert fibonacci(10) == 55
}

test "handles empty input" {
    assert parse("") == None
}
```

### 3.11 Const Declarations

```
const_decl = [ visibility ] "const" IDENT [ ":" type ] "=" expression ;
```

- Constants must be computable at compile time (literal values, arithmetic on other constants, compile-time functions).
- The type annotation is optional when it can be inferred from the expression.

Example:
```aria
const MAX_RETRIES = 3
pub const DEFAULT_TIMEOUT: dur = 30s
```

### 3.12 Generic Parameters and Constraints

```
generic_params = "[" generic_param { "," generic_param } "]" ;
generic_param  = IDENT [ ":" trait_bound ] ;
trait_bound    = IDENT { "+" IDENT } ;

where_clause   = "where" where_item { "," where_item } ;
where_item     = IDENT ":" trait_bound ;
```

Examples:
```aria
fn sum[T: Numeric](list: [T]) -> T { ... }
fn zip[T, U](a: [T], b: [U]) -> [(T, U)] { ... }
impl[T: Eq + Hash] MySet[T] for Collection { ... }
```

### 3.13 Types

```
type = named_type
     | function_type
     | tuple_type
     | array_type
     | map_type
     | set_type
     | optional_type
     | result_type
     | "(" type ")" ;      (* parenthesized for disambiguation *)

named_type    = IDENT { "." IDENT } [ "[" type_list "]" ] ;
function_type = type "->" type
              | "(" type_list ")" "->" type ;
tuple_type    = "(" type "," type { "," type } ")" ;
array_type    = "[" type "]" ;
map_type      = "{" type ":" type "}" ;
set_type      = "{" type "}" ;
optional_type = type "?" ;
result_type   = type "!" type ;
type_list     = type { "," type } ;
```

> **Design Decision:** `type "!" type` is the explicit result type syntax (`str ! IoError`). `type "?"` is the optional type syntax (`User?` = `Option[User]`). These are lexer-level sugar; both `Option[T]` and `Result[T, E]` are also valid named types. This mirrors how `Result` and `Option` appear in function signatures.

### 3.14 Expressions

Expressions are defined using a stratified grammar that encodes precedence. Higher-numbered levels bind tighter (evaluated first). See [`operator-precedence.md`](operator-precedence.md) for the complete precedence table.

```
(* Level 1: Binding / assignment — lowest precedence *)
expression = pipeline_expr
           | mut_binding
           | immut_binding
           | assignment ;

mut_binding   = "mut" pattern ":=" expression
              | "mut" IDENT ":" type "=" expression ;
immut_binding = pattern ":=" expression
              | IDENT ":" type "=" expression ;
assignment    = postfix_expr "=" expression ;

(* Level 2: Pipeline *)
pipeline_expr = coalesce_expr { "|>" pipe_target } ;
pipe_target   = postfix_expr ;  (* arbitrary expression used as function to pipe into *)

(* Level 3: Null coalesce *)
coalesce_expr = or_expr { "??" or_expr } ;

(* Level 4: Logical OR *)
or_expr  = and_expr { "||" and_expr } ;

(* Level 5: Logical AND *)
and_expr = bitwise_or_expr { "&&" bitwise_or_expr } ;

(* Level 6: Comparison — non-associative *)
(* A single comparison_expr may contain at most one comparison operator *)
comparison_expr = range_expr [ ( "==" | "!=" | "<" | ">" | "<=" | ">=" ) range_expr ] ;

(* Level 7: Range — non-associative *)
range_expr = bitwise_or_expr [ ( ".." | "..=" ) bitwise_or_expr ] ;

(* Level 8: Bitwise OR *)
bitwise_or_expr = bitwise_xor_expr { "|" bitwise_xor_expr } ;

(* Level 9: Bitwise XOR *)
bitwise_xor_expr = bitwise_and_expr { "^" bitwise_and_expr } ;

(* Level 10: Bitwise AND *)
bitwise_and_expr = shift_expr { "&" shift_expr } ;

(* Level 11: Bitwise shift *)
shift_expr = additive_expr { ( "<<" | ">>" ) additive_expr } ;

(* Level 12: Addition and subtraction *)
additive_expr = multiplicative_expr { ( "+" | "-" ) multiplicative_expr } ;

(* Level 13: Multiplication, division, modulo *)
multiplicative_expr = unary_expr { ( "*" | "/" | "%" ) unary_expr } ;

(* Level 14: Prefix unary operators *)
unary_expr = postfix_expr
           | "-" unary_expr
           | "!" unary_expr
           | "~" unary_expr ;

(* Level 15: Postfix operators *)
postfix_expr = primary_expr { postfix_op } ;
postfix_op   = "?"                           (* error propagation / None propagation *)
             | "!"                           (* assert-success / panic on error *)
             | "." IDENT                     (* field access or method call without args *)
             | "." IDENT "(" [ arg_list ] ")" (* method call with args *)
             | "?." IDENT                    (* optional chaining *)
             | "[" expression "]"            (* index access *)
             | "(" [ arg_list ] ")"          (* call *)
             | ".{" field_init_list "}"      (* record update *)
             ;

(* Level 16: Primary expressions — highest precedence *)
primary_expr = literal
             | IDENT
             | path_expr
             | block_expr
             | if_expr
             | match_expr
             | closure_expr
             | struct_expr
             | array_expr
             | map_expr
             | tuple_expr
             | select_expr
             | scope_expr
             | spawn_expr
             | list_comprehension
             | "(" expression ")" ;
```

**Arguments:**

```
arg_list   = arg { "," arg } [ "," ] ;
arg        = [ IDENT ":" ] expression ;    (* named or positional argument *)
```

**Path expressions** (qualified identifiers like `std.fs.read`):

```
path_expr = IDENT "." IDENT { "." IDENT } ;
```

**Block expressions:**

```
block_expr = "{" { statement } [ expression ] "}" ;
```

> **Design Decision:** The last item in a block is its value. If the last item is a statement (e.g. a `for` loop or a `:=` binding), the block evaluates to `()` (unit). If it is an expression, the block evaluates to that expression's value. There is no explicit `return` needed at the end of a block-valued expression.

**If expressions:**

```
if_expr = "if" expression block_expr [ "else" ( block_expr | if_expr ) ] ;
```

- No parentheses around the condition (unlike C/Java).
- `if` without `else` has type `()` when used in expression position — this is a compile error if the `if` result is used. Always provide `else` when using `if` as an expression.

**Match expressions:**

```
match_expr = "match" expression "{" { match_arm } "}" ;
match_arm  = pattern [ "if" expression ] "=>" expression [ "," ] ;
```

- Match is exhaustive. Unmatched cases are a compile error.
- Arms are separated by newlines or commas (either is valid).

**Closure expressions:**

```
closure_expr = "fn" "(" [ param_list ] ")"              "=>" expression
             | "fn" "(" [ param_list ] ")" "->" type    "=>" expression
             | "move" "fn" "(" [ param_list ] ")"        "=>" expression
             | "move" "fn" "(" [ param_list ] ")" "->" type "=>" expression ;
```

> **Design Decision:** Closures use the same `fn` keyword as function declarations, distinguished by the `=>` body form and the absence of a name. `move` before `fn` forces the closure to take ownership of all captured variables, enabling use with `spawn`.

**Struct construction expressions:**

```
struct_expr     = IDENT "{" [ field_init_list ] "}" ;
field_init_list = field_init { "," field_init } [ "," ] ;
field_init      = IDENT ":" expression
                | IDENT ;              (* shorthand: field name == variable name *)
```

**Array literal:**

```
array_expr = "[" [ expression { "," expression } [ "," ] ] "]" ;
```

**Map literal:**

```
map_expr = "{" map_entry { "," map_entry } [ "," ] "}" ;
map_entry = expression ":" expression ;
```

**Tuple literal:**

```
tuple_expr = "(" expression "," expression { "," expression } [ "," ] ")" ;
(* Note: a single-element tuple requires a trailing comma: (x,) *)
```

**List comprehension:**

```
list_comprehension = "[" expression "for" IDENT "in" expression [ "where" expression ] "]" ;
```

Example: `[x * x for x in 1..=10 where x % 2 == 0]`

**Spawn expression:**

```
spawn_expr = "spawn" expression ;
```

**Scope expression:**

```
scope_expr = "scope" block_expr ;
```

**Select expression:**

```
select_expr = "select" "{" { select_arm } "}" ;
select_arm  = IDENT "from" expression "=>" expression
            | "after" expression        "=>" expression ;
```

**Literal:**

```
literal = INT_LIT
        | FLOAT_LIT
        | STRING_LIT
        | BOOL_LIT
        | DURATION_LIT
        | SIZE_LIT ;
```

### 3.15 Statements

```
statement = variable_decl
          | assignment_stmt
          | expression_stmt
          | for_loop
          | while_loop
          | loop_stmt
          | return_stmt
          | break_stmt
          | continue_stmt
          | defer_stmt
          | with_stmt
          | catch_block ;

variable_decl   = [ "mut" ] pattern ":=" expression NEWLINE
                | [ "mut" ] IDENT ":" type "=" expression NEWLINE ;

assignment_stmt = postfix_expr "=" expression NEWLINE ;
(* Only valid when postfix_expr refers to a `mut`-declared binding or field *)

expression_stmt = expression NEWLINE ;

for_loop   = "for" pattern "in" expression [ "where" expression ] block_expr ;
while_loop = "while" expression block_expr ;
loop_stmt  = "loop" block_expr ;

return_stmt   = "return" [ expression ] NEWLINE ;
break_stmt    = "break" [ expression ] NEWLINE ;
continue_stmt = "continue" NEWLINE ;

defer_stmt = "defer" ( expression | block_expr ) NEWLINE ;
with_stmt  = "with" pattern ":=" expression block_expr ;

catch_block = expression "catch" "|" IDENT "|" block_expr ;
```

> **Design Decision:** `for`, `while`, and `loop` are statements and evaluate to `()`. They may appear in expression position only via list comprehension (for `for`). `if` and `match` are expressions. This matches the spec in `paradigm-design.md`.

> **Design Decision:** `break` may carry a value expression. In a `loop` block used as an expression (e.g. assigned to a variable), the `break value` form provides the block's result. This is consistent with Rust's loop expressions and avoids mutable temporaries.

### 3.16 Patterns

```
pattern = wildcard_pattern
        | binding_pattern
        | literal_pattern
        | struct_pattern
        | variant_pattern
        | tuple_pattern
        | array_pattern
        | or_pattern
        | rest_pattern ;

wildcard_pattern = "_" ;
binding_pattern  = [ "mut" ] IDENT ;
literal_pattern  = literal ;
struct_pattern   = IDENT "{" field_pattern_list "}" ;
variant_pattern  = IDENT "(" pattern_list ")" ;
tuple_pattern    = "(" pattern_list ")" ;
array_pattern    = "[" array_pat_list "]" ;
or_pattern       = pattern "|" pattern ;       (* same binding names + types on both sides *)
rest_pattern     = ".." [ IDENT ] ;            (* matches zero or more remaining elements *)

field_pattern_list = field_pattern { "," field_pattern } [ "," ] ;
field_pattern      = IDENT                   (* bind field name directly *)
                   | IDENT ":" pattern       (* bind field with sub-pattern *)
                   | ".."                    (* ignore remaining fields *)
                   ;

pattern_list   = pattern { "," pattern } [ "," ] ;
array_pat_list = pattern { "," pattern } [ "," ] [ ".." IDENT ] ;
```

### 3.17 `.field` Shorthand in Pipelines

When a closure of the form `fn(x) => x.field` appears immediately after `|>`, the shorthand `.field` may be used:

```
pipe_target = postfix_expr
            | "." IDENT ;    (* shorthand: .field means fn(x) => x.field *)
```

Examples:
- `|> sortBy(.lastLogin)` means `|> sortBy(fn(x) => x.lastLogin)`
- `|> filter(.active)` means `|> filter(fn(x) => x.active)`
- `|> map(.name)` means `|> map(fn(x) => x.name)`

---

## 4. Grammar Ambiguities and Resolution Rules

### 4.1 `<` as Comparison vs. Generic Parameter

In the expression `f < g`, the `<` is always parsed as less-than comparison. Generics use `[` and `]` for type parameters, not `<` and `>`. This eliminates the classic C++/Rust turbofish ambiguity entirely.

```aria
fn map[T, U](list: [T], f: T -> U) -> [U]   // generics use [ ]
a < b                                          // always comparison
```

### 4.2 `{` After an Expression — Block vs. Struct Literal

When `{` immediately follows:
- A type name (an identifier), it is parsed as a **struct literal**: `Point{x: 1, y: 2}`
- Any other expression, it is parsed as a **block expression**: `foo(); { ... }`
- After `if`/`while`/`for`/`loop`/`else`/`match`/`scope`/`entry`/`test`, it is always a **block**

> **Design Decision:** To construct a struct from an expression (not a literal type name), use a temporary binding: `t := myType; t{...}` is NOT valid. Use struct construction only with explicit type names. If the type must be computed, use a constructor function.

### 4.3 `|` in Type Position vs. Bitwise OR

`|` introduces a sum-type variant only at the start of a type body (inside `type Foo =`) or at the beginning of `match` arm patterns (separated by the `or_pattern` form). In expression context, `|` is always bitwise OR. The parser disambiguates by context (in type declarations vs. in expressions).

### 4.4 `-` as Unary Negation vs. Binary Subtraction

Standard precedence climbing handles this: after parsing a primary or postfix expression, if `-` appears, it is binary subtraction. `-` at the beginning of a `unary_expr` production is prefix negation.

### 4.5 `!` as Prefix Logical NOT vs. Postfix Assert-Success

`!` in `unary_expr` (prefix) is logical NOT. `!` in `postfix_op` is assert-success (unwrap or panic). The parser resolves this by position: after a complete `primary_expr { postfix_op* }`, a trailing `!` is postfix; at the start of `unary_expr`, `!` is prefix.

```aria
!flag          // prefix: logical NOT
readFile()?!   // postfix: assert file.read() succeeded (no trailing '!' on '?')
result!        // postfix: assert result is not an error
```

### 4.6 `.field` Shorthand in Pipeline Context

`.field` as a `pipe_target` (the shorthand form) is only valid as the immediate argument to `|>`. In all other contexts, `.field` by itself is a syntax error. The parser recognizes this by checking whether the `.` is preceded by `|>`.

### 4.7 Struct Field Init Shorthand

Inside a struct literal `Point{x, y}`, a bare identifier `x` is shorthand for `x: x` (the field named `x` gets the value of the variable named `x`). This is resolved by the parser at the `field_init` production level.

---

## 5. Automatic Semicolon Insertion (Statement Termination)

Aria has **no semicolons**. Instead, the lexer applies newline-as-statement-terminator rules inspired by Go's semicolon insertion, but extended to handle expression-oriented constructs and pipelines.

### 5.1 The Rule

A newline token is treated as a **statement terminator** (implicit semicolon) when the last token on the line is one of:

- An identifier (`IDENT`)
- A literal (`INT_LIT`, `FLOAT_LIT`, `STRING_LIT`, `BOOL_LIT`, `DURATION_LIT`, `SIZE_LIT`)
- `break`, `continue`, `return`
- `)`, `]`, `}`
- `?`, `!` (postfix operators)

### 5.2 Continuation Lines

A newline is **NOT** treated as a statement terminator (i.e., the next line is a continuation) when the last token on the current line is one of:

- Any binary operator: `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `&&`, `||`, `&`, `|`, `^`, `~`, `<<`, `>>`
- Pipeline operator: `|>`
- Assignment/binding operators: `:=`, `=`, `=>`
- Return type arrow: `->`
- `,` (inside argument lists, parameter lists, field lists)
- `.` (field access chain continued on next line)
- `(`, `[`, `{` (inside open delimiter — newlines are transparent inside balanced delimiters)
- `??`, `..`, `..=`

### 5.3 Balanced-Delimiter Rule

Inside any balanced pair of `( )`, `[ ]`, or `{ }`, newlines are always ignored (treated as whitespace). This means argument lists, array literals, struct bodies, and block expressions can freely span multiple lines.

> **Exception:** Inside a block expression `{ ... }` that appears as a statement body (e.g. the body of `if`, `for`, `fn`, etc.), individual statements inside the block are still terminated by the newline rule. The block itself does not suppress statement termination of its contents.

### 5.4 Examples

```aria
// OK — continuation after operator
result := a
    + b
    + c

// OK — pipeline continuation after |>
result := input
    |> parse?
    |> validate?

// OK — method chain continuation after .
users
    .filter(fn(u) => u.active)
    .sortBy(.name)

// Statement terminator inserted here:
x := 5     // <- terminator after '5'
y := x + 1 // <- terminator after '1'

// OK — multiline struct literal (inside balanced { })
user := User{
    name: "Alice",
    email: "alice@example.com",
}
```

---

## 6. Complete Example

The following demonstrates the grammar applied to a realistic program:

```aria
mod main

use std.fs
use std.{json, io}

type Config {
    host:    str    = "localhost"
    port:    u16    = 8080
    timeout: dur    = 30s
} derives [Eq, Json, Debug]

fn loadConfig(path: str) -> Config ! IoError | ParseError {
    content := fs.read(path)?
    json.parse[Config](content)?
}

entry {
    config := loadConfig("config.json") catch |err| {
        io.println("Using defaults: {err}")
        Config{}
    }
    io.println("Starting on {config.host}:{config.port}")
}

test "loadConfig uses defaults on missing file" {
    result := loadConfig("/nonexistent")
    assert result == Config{}
}
```
