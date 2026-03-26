# Aria Compiler Diagnostics Specification

**A specification for compiler error messages, warnings, structured output, and fix suggestions — designed as a machine-readable API for AI-driven development.**

This document defines how the Aria compiler communicates problems to its users — both human and AI. In an AI-first language, compiler diagnostics are not just messages to read; they are a structured API that AI tools consume, parse, and act on. Every design decision here optimizes the cycle: **write code → compile → read diagnostic → fix code.**

Cross-references:
- [spec/compiler-architecture.md](compiler-architecture.md) — compilation pipeline stages where diagnostics originate
- [spec/error-handling.md](error-handling.md) — runtime error traces (distinct from compile-time diagnostics)
- [spec/formal-grammar.md](formal-grammar.md) — syntax errors and parsing diagnostics
- [spec/effect-system.md](effect-system.md) — effect violation diagnostics
- [spec/pattern-matching.md](pattern-matching.md) — exhaustiveness diagnostics
- [spec/trait-system.md](trait-system.md) — trait bound violation diagnostics

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Diagnostic Structure](#2-diagnostic-structure)
3. [Severity Levels](#3-severity-levels)
4. [Error Codes](#4-error-codes)
5. [Human-Readable Output](#5-human-readable-output)
6. [Structured JSON Output](#6-structured-json-output)
7. [Fix Suggestions](#7-fix-suggestions)
8. [Diagnostic Categories](#8-diagnostic-categories)
9. [The `aria fix` Command](#9-the-aria-fix-command)
10. [Design Rationale Summary](#10-design-rationale-summary)

---

## 1. Design Philosophy

Compiler error messages are the most important interface in a programming language. More important than the syntax. More important than the standard library. Because when code doesn't compile, the error message is the **only thing** between the developer and a fix.

For AI code generation, this is even more true. When I generate Aria code that doesn't compile, the diagnostic is my sole input for the next iteration. The diagnostic must tell me:

1. **What** is wrong (the error)
2. **Where** it is (file, line, column)
3. **Why** it's wrong (the rule that was violated)
4. **How** to fix it (a concrete suggestion)

If any of these are missing, I waste a round guessing.

**AI rationale:** Every compiler diagnostic is a structured data payload that I parse and act on. Human-readable prose is a rendering of that payload. The structured form is the primary output; the human-readable form is derived from it. This is the opposite of how most compilers work — they generate prose and then try to extract structure. Aria generates structure first.

---

## 2. Diagnostic Structure

Every diagnostic has the following fields:

| Field | Type | Description |
|---|---|---|
| `code` | `str` | Unique diagnostic code (e.g., `E0042`, `W0015`) |
| `severity` | `Severity` | `error`, `warning`, `info`, or `hint` |
| `message` | `str` | One-line summary of the problem |
| `file` | `str` | Source file path |
| `line` | `u64` | Line number (1-based) |
| `column` | `u64` | Column number (1-based) |
| `span` | `(u64, u64)` | Start and end byte offsets in the source |
| `source_line` | `str` | The source code line containing the error |
| `labels` | `[Label]` | Annotated source spans (primary and secondary) |
| `notes` | `[str]` | Additional context or explanation |
| `suggestions` | `[Suggestion]` | Concrete fix suggestions |

### Label

```
type Label {
    file: str
    line: u64
    column: u64
    span: (u64, u64)
    message: str
    style: LabelStyle     // Primary | Secondary
}
```

### Suggestion

```
type Suggestion {
    message: str           // human-readable description of the fix
    replacement: str       // the text to insert/replace
    span: (u64, u64)      // the span to replace
    applicability: Applicability   // MachineApplicable | MaybeIncorrect | HasPlaceholders
}
```

---

## 3. Severity Levels

| Level | Meaning | Blocks compilation? |
|---|---|---|
| `error` | Code is invalid — cannot compile | Yes |
| `warning` | Code is valid but likely wrong or suboptimal | No |
| `info` | Informational — context for another diagnostic | No |
| `hint` | Suggestion for improvement | No |

### Warning control

```
aria build --deny-warnings        # treat warnings as errors
aria build --allow=W0015          # suppress specific warning
```

---

## 4. Error Codes

Every diagnostic has a unique code. Codes are stable across compiler versions — they can be referenced in documentation, issue trackers, and AI training data.

### Code format

- `E` prefix: errors (e.g., `E0001`, `E0042`)
- `W` prefix: warnings (e.g., `W0001`, `W0015`)

### Error code categories

| Range | Category | Examples |
|---|---|---|
| E0001–E0099 | Syntax errors | Missing `}`, unexpected token, invalid literal |
| E0100–E0199 | Type errors | Type mismatch, cannot infer type, invalid conversion |
| E0200–E0299 | Trait errors | Missing trait impl, trait bound not satisfied, orphan rule |
| E0300–E0399 | Effect errors | Missing effect declaration, pure function violation |
| E0400–E0499 | Pattern matching | Non-exhaustive match, unreachable pattern |
| E0500–E0599 | Ownership/move | Use after move, cannot move across task boundary |
| E0600–E0699 | Concurrency | Send/Share violations, ref capture across spawn |
| E0700–E0799 | Module/import | Unresolved import, visibility violation, circular dependency |
| E0800–E0899 | Const evaluation | Invalid const expression, const overflow |
| E0900–E0999 | FFI | Invalid C type, missing annotation, pointer safety |
| W0001–W0099 | Style warnings | Unused variable, non-conventional naming |
| W0100–W0199 | Logic warnings | Unreachable code, redundant condition |
| W0200–W0299 | Deprecation | Using deprecated function/type |

### Looking up error codes

```
aria explain E0042
```

Prints a detailed explanation of the error, with examples of code that triggers it and how to fix it.

---

## 5. Human-Readable Output

The default output format is designed for terminal display with color highlighting:

```
error[E0105]: type mismatch
  --> src/process.aria:12:18
   |
12 |     result := add(userId, orderId)
   |                   ^^^^^^  ^^^^^^^ expected UserId, found OrderId
   |                   |
   |                   this is UserId
   |
   = note: UserId and OrderId are distinct newtypes wrapping i64
   = help: if you need the raw i64 values, use .value:
           add(userId.value, orderId.value)
```

### Multi-span diagnostics

When an error involves multiple source locations:

```
error[E0201]: trait bound not satisfied: `FileHandle: Eq`
  --> src/main.aria:5:5
   |
5  |     a == b
   |     ^^^^^^ `FileHandle` does not implement `Eq`
   |
  --> src/types.aria:3:1
   |
3  | type FileHandle { fd: i64 }
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^ `Eq` is not derived here
   |
   = help: add `derives [Eq]` to the type declaration:
           type FileHandle { fd: i64 } derives [Eq]
```

### Effect violation

```
error[E0301]: effect violation: `Io` effect required
  --> src/calc.aria:3:5
   |
2  | fn calculate(x: i64) -> i64 {
   |    --------- this function is pure (no `with` clause)
3  |     println("calculating {x}")
   |     ^^^^^^^ `println` requires effect [Io]
   |
   = help: add an effect clause to the function:
           fn calculate(x: i64) -> i64 with [Io] {
```

### Exhaustiveness

```
error[E0401]: non-exhaustive match
  --> src/shapes.aria:15:5
   |
15 | match shape {
   |       ^^^^^ missing variant: `Point`
16 |     Circle(r) => pi * r * r
17 |     Rect(w, h) => w * h
   |
   = help: add the missing arm:
           Point => /* expression */
   = note: or add a wildcard arm:
           _ => /* default expression */
```

---

## 6. Structured JSON Output

```
aria check --format=json
```

```json
{
  "diagnostics": [
    {
      "code": "E0105",
      "severity": "error",
      "message": "type mismatch",
      "file": "src/process.aria",
      "line": 12,
      "column": 18,
      "span": [245, 251],
      "source_line": "    result := add(userId, orderId)",
      "labels": [
        {
          "file": "src/process.aria",
          "line": 12,
          "column": 18,
          "span": [245, 251],
          "message": "this is UserId",
          "style": "primary"
        },
        {
          "file": "src/process.aria",
          "line": 12,
          "column": 26,
          "span": [253, 260],
          "message": "expected UserId, found OrderId",
          "style": "secondary"
        }
      ],
      "notes": [
        "UserId and OrderId are distinct newtypes wrapping i64"
      ],
      "suggestions": [
        {
          "message": "use .value to access the raw i64 values",
          "replacement": "add(userId.value, orderId.value)",
          "span": [234, 262],
          "applicability": "MaybeIncorrect"
        }
      ]
    }
  ],
  "summary": {
    "errors": 1,
    "warnings": 0
  }
}
```

**AI rationale:** JSON output is how I consume compiler diagnostics. I parse the `code`, `message`, `labels`, and `suggestions` fields to understand the error and generate a fix. I can apply `MachineApplicable` suggestions directly. I can use the `span` to locate the exact bytes to change. This is orders of magnitude more efficient than parsing the human-readable format.

---

## 7. Fix Suggestions

Suggestions have three applicability levels:

| Level | Meaning | Safe to auto-apply? |
|---|---|---|
| `MachineApplicable` | The fix is guaranteed correct | Yes |
| `MaybeIncorrect` | The fix is likely correct but may change semantics | With review |
| `HasPlaceholders` | The fix contains placeholders the user must fill in | No |

### Machine-applicable suggestions

```
warning[W0001]: unused variable `x`
  --> src/main.aria:5:5
   |
5  |     x := computeValue()
   |     ^ unused variable
   |
   = suggestion (MachineApplicable): prefix with `_` to indicate intentional non-use
     replacement: _x := computeValue()
```

### Suggestions with placeholders

```
error[E0401]: non-exhaustive match — missing variant `Point`
  --> src/shapes.aria:15:5
   |
   = suggestion (HasPlaceholders): add the missing arm
     replacement:
       Point => /* TODO: handle Point */
```

---

## 8. Diagnostic Categories

### Type mismatch (E0100–E0199)

The most common error category. Includes:
- Argument type doesn't match parameter type
- Return type doesn't match function signature
- Assignment to incompatible type
- Newtype/underlying type confusion
- Generic type inference failure

### "Did you mean?" suggestions

For typos and near-misses, the compiler suggests corrections:

```
error[E0701]: unresolved name `prnitln`
  --> src/main.aria:5:5
   |
5  |     prnitln("hello")
   |     ^^^^^^^ not found in this scope
   |
   = help: did you mean `println`?
```

```
error[E0702]: no field `naem` on type `User`
  --> src/main.aria:8:15
   |
8  |     user.naem
   |          ^^^^ unknown field
   |
   = help: did you mean `name`?
   = note: available fields: name, email, age
```

### Import suggestions

```
error[E0703]: unresolved name `fs`
  --> src/main.aria:5:16
   |
5  |     content := fs.read("config.json")?
   |                ^^ not found in this scope
   |
   = help: add `use std.fs` to import the filesystem module
```

---

## 9. The `aria fix` Command

`aria fix` automatically applies all `MachineApplicable` suggestions:

```
aria fix                           # apply all auto-fixable diagnostics
aria fix --dry-run                 # show what would be changed
aria fix --code=W0001              # only fix specific diagnostic codes
```

### What `aria fix` can fix

- Unused variable warnings → prefix with `_`
- Missing imports → add `use` statement
- Deprecated API calls → replace with suggested alternative
- Formatting issues → normalize whitespace and indentation
- Missing `derives` → add derived traits when the fix is unambiguous

### What `aria fix` cannot fix

- Type mismatches (require understanding intent)
- Missing match arms with placeholders (require business logic)
- Effect violations (require architectural decisions)

**AI rationale:** `aria fix` is my auto-repair tool. After generating code, I run `aria check --format=json`, parse the diagnostics, and for `MachineApplicable` suggestions, I can either apply them myself or run `aria fix`. This reduces the generate-fix cycle from multiple rounds to potentially one.

---

## 10. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Structured JSON as primary output | AI tools consume structure, not prose — parse once, act immediately |
| Unique error codes | Stable references for documentation, lookup, and pattern matching |
| Fix suggestions with applicability | AI knows which fixes are safe to apply automatically |
| Multi-span labels | Complex errors (type mismatches, trait bounds) need to show multiple locations |
| "Did you mean?" for typos | Reduces one of the most common error-fix cycles to one round |
| Import suggestions | Missing imports are the #1 fixable error — suggest the import, don't just complain |
| `aria fix` command | Batch-apply safe fixes — reduces manual iteration |
| `aria explain E0042` | Deep documentation per error code — AI or human can learn why |
| Source line in diagnostics | No need to open the file — the relevant code is in the diagnostic |
| Notes for context | Explain *why* the rule exists, not just *what* was violated |

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/compiler-architecture.md](compiler-architecture.md), [spec/error-handling.md](error-handling.md), and [spec/formal-grammar.md](formal-grammar.md).*
