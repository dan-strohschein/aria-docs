# Aria Annotations Specification

**A specification for the closed annotation set in Aria v0.1, including memory annotations, FFI annotations, and the decisions to exclude variadics and user-defined annotations.**

Annotations in Aria are compiler directives that modify the behavior or semantics of declarations. In v0.1, annotations are a **closed set** — the compiler knows every valid annotation. Users cannot define custom annotations. This keeps the parser simple, avoids annotation-driven metaprogramming, and ensures every annotation has well-defined semantics.

Cross-references:
- [spec/memory-management.md](memory-management.md) — `@stack`, `@arena`, `@inline` annotations
- [spec/ffi-design.md](ffi-design.md) — `@opaque`, `@cstruct`, `@cffi`, `@owned`, `@borrowed`
- [spec/trait-system.md](trait-system.md) — `derives [...]` mechanism
- [spec/formal-grammar.md](formal-grammar.md) — annotation syntax in the grammar

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Annotation Syntax](#2-annotation-syntax)
3. [Memory Annotations](#3-memory-annotations)
4. [FFI Annotations](#4-ffi-annotations)
5. [Function Annotations](#5-function-annotations)
6. [Type Annotations](#6-type-annotations)
7. [The `derives` Mechanism](#7-the-derives-mechanism)
8. [No User-Defined Annotations (v0.1)](#8-no-user-defined-annotations)
9. [No Variadics (Design Decision)](#9-no-variadics)
10. [Design Rationale Summary](#10-design-rationale-summary)

---

## 1. Design Philosophy

Java has thousands of annotations. They form a shadow type system. Frameworks depend on them. Code becomes unreadable without understanding what each annotation does. This is exactly what Aria avoids.

In Aria v0.1, annotations are a small, fixed set of compiler directives. Every annotation has exactly one meaning, documented in this spec. The compiler rejects unrecognized annotations with a clear error.

**AI rationale:** A closed annotation set means I never have to wonder "what does this annotation do?" Every annotation I encounter is defined in this document. I can generate annotations mechanically from the spec — no framework knowledge required, no guessing.

---

## 2. Annotation Syntax

Annotations use the `@` prefix and appear before the declaration they modify:

```
// On allocations
x := @stack Buffer.new()
x := @arena(arena) Thing{...}

// On struct fields
type Response {
    headers: @inline Map[str, str]
    body: [byte]
}

// On functions
@deprecated("use processV2 instead")
fn process(data: [byte]) -> Result { ... }

// On type declarations (derives is special syntax, not @-prefixed)
type User { name: str, email: str } derives [Eq, Hash, Debug]
```

### Grammar

```
annotation = "@" IDENT [ "(" annotation_args ")" ] ;
annotation_args = expression { "," expression } ;
```

---

## 3. Memory Annotations

These control how values are allocated. See [spec/memory-management.md](memory-management.md) for full details.

| Annotation | Applies to | Meaning |
|---|---|---|
| `@stack` | Allocations | Stack-allocate; compiler verifies value doesn't escape scope |
| `@arena(arena)` | Allocations | Allocate in the specified arena |
| `@inline` | Struct fields | Embed the value directly in the parent struct (no pointer indirection) |

```
buffer := @stack Buffer.new(4096)
parsed := @arena(arena) parseRequest(req)?

type Response {
    headers: @inline Map[str, str]
    body: [byte]
}
```

---

## 4. FFI Annotations

These control how types and memory interact with C code. See [spec/ffi-design.md](ffi-design.md) for full details.

| Annotation | Applies to | Meaning |
|---|---|---|
| `@opaque` | Type declarations | Opaque C type — can only be used as a pointer |
| `@cstruct` | Type declarations | C-compatible struct layout (no reordering, C alignment) |
| `@cffi` | Allocations | Allocate in non-moving memory region (safe for C pointers) |
| `@owned` | FFI return types | Aria takes ownership — will free when dropped |
| `@borrowed` | FFI return types | C retains ownership — Aria must not free |

```
type c.sqlite3 = @opaque
type c.Point = @cstruct { x: c.double, y: c.double }

data := @cffi Buffer.new(1024)

extern "C" fn create_thing() -> *c.Thing @owned
extern "C" fn get_name(ctx: *c.Context) -> *const c.char @borrowed
```

---

## 5. Function Annotations

| Annotation | Applies to | Meaning |
|---|---|---|
| `@deprecated("message")` | Functions, types | Mark as deprecated; compiler warns on usage |
| `@cold` | Functions | Hint that this function is rarely called — optimize for size, not speed |

### `@deprecated`

```
@deprecated("use fetchV2 instead — this version doesn't handle timeouts")
pub fn fetch(url: str) -> str ! HttpError with [Io, Net] {
    // ...
}

// Calling a deprecated function produces a compiler warning:
// warning: `fetch` is deprecated: use fetchV2 instead — this version doesn't handle timeouts
//   --> src/client.aria:15:10
```

The message parameter is required — a deprecation without guidance is not helpful.

### `@cold`

```
@cold
fn handleFatalError(err: Error) {
    log.error("Fatal: {err}")
    dumpStackTrace()
    exit(1)
}
```

`@cold` tells the compiler this function is rarely called. The compiler may:
- Move it to a separate code section to improve instruction cache locality for hot code
- Optimize for code size rather than speed
- Avoid inlining it into callers

**AI rationale:** `@deprecated` is critical for me — I need to know which APIs are obsolete so I generate calls to the current versions. Without it, I'll generate code using old APIs that someone forgot to delete. `@cold` is a simple performance hint that requires zero restructuring.

---

## 6. Type Annotations

| Annotation | Applies to | Meaning |
|---|---|---|
| `@opaque` | Type declarations | Opaque C type (see FFI) |
| `@cstruct` | Type declarations | C-compatible layout (see FFI) |

No additional type annotations are defined in v0.1 beyond the FFI-related ones above.

---

## 7. The `derives` Mechanism

`derives` is syntactically distinct from `@` annotations. It appears after a type body and generates trait implementations:

```
type User {
    name: str
    email: str
    age: u8
} derives [Eq, Hash, Debug, Clone, Default]
```

### `derives` is the canonical syntax

The `derives [...]` postfix form is the only supported syntax. The `@[derive(...)]` prefix form that appears in some early documents is **not valid Aria** — it was an early design exploration that was superseded.

```
// ✅ Canonical — this is the syntax
type User { name: str } derives [Eq, Debug]

// ❌ Not valid Aria — do not use
@[derive(Eq, Debug)]
type User { name: str }
```

See [spec/trait-system.md](trait-system.md) section 10 for the complete derives specification.

---

## 8. No User-Defined Annotations (v0.1)

Aria v0.1 does not support user-defined annotations. The set of valid annotations is fixed and known to the compiler.

### Why

User-defined annotations create a framework ecosystem where code is driven by annotations rather than explicit logic. This has several problems for AI code generation:

1. **The AI must know every framework's annotations** — Java's `@Autowired`, `@Inject`, `@Transactional`, `@RequestMapping` all mean different things in different frameworks
2. **Annotations can change semantics invisibly** — adding `@Transactional` changes whether a function commits on return
3. **Annotation interactions are complex** — `@Async @Transactional` doesn't work the way you'd expect in Spring

By keeping annotations as a closed compiler set, every annotation has well-defined, documented behavior that the AI can rely on.

### Future

A future version may introduce a controlled annotation extension mechanism — likely tied to a macro system rather than a runtime reflection system. This is deferred until the ecosystem needs it.

---

## 9. No Variadics (Design Decision)

Aria does not support variadic functions (functions with a variable number of arguments). This is a deliberate design decision, not an omission.

### Why no variadics

Variadic functions create ambiguity and complexity:
- **Type safety** — what is the type of the extra arguments? In C, it's untyped. In Go, it's `...interface{}`. Both are sources of runtime errors.
- **Parser complexity** — variadic syntax interacts with optional parameters, default values, and generic inference
- **Limited use cases** — the main use cases (print, max, logging) have better solutions

### What to use instead

| Use case | Solution | Example |
|---|---|---|
| Formatted output | Compiler intrinsic `println` with string interpolation | `println("x={x}, y={y}")` |
| Variable number of same-type items | Slice parameter | `fn max(values: [i64]) -> i64` |
| Heterogeneous items | Trait object slice | `fn log(items: [dyn Display])` |
| Optional configuration | Named parameters with defaults | `fn serve(host: str = "localhost", port: u16 = 8080)` |

### `println` is a compiler intrinsic

`println` handles string interpolation at compile time. It is not a variadic function — it is a compiler built-in that understands `{expression}` syntax inside string literals:

```
println("Hello, {name}! You are {age} years old.")
println("Result: {compute(x, y)}")
println("{user.name}: {user.email}")
```

The compiler expands interpolation into a series of string concatenations and `Display` calls at compile time. No runtime format string parsing, no variadic argument handling.

**AI rationale:** Variadic functions are a trap. In Go, `fmt.Sprintf("%s has %d items", name, count)` requires me to match format specifiers to argument types — a constant source of bugs. Aria's string interpolation is type-safe, position-independent, and impossible to get wrong. I never generate a mismatched format string because format strings don't exist.

---

## 10. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Closed annotation set | Every annotation has known semantics — no framework guessing |
| No user-defined annotations (v0.1) | Avoids annotation-driven metaprogramming complexity |
| `@deprecated("message")` | AI needs to know which APIs are current |
| `@cold` for rarely-called functions | Simple performance hint, no restructuring needed |
| `derives [...]` is canonical | One syntax, postfix, clean — supersedes `@[derive(...)]` |
| No variadics | Slices + named defaults + compiler intrinsic print cover all use cases |
| `println` as compiler intrinsic | Type-safe interpolation eliminates format-string bugs |
| No `@unsafe` blocks | FFI boundary is the safety boundary — not individual expressions |

---

## Complete Annotation Reference

| Annotation | Context | Spec |
|---|---|---|
| `@stack` | Allocations | [memory-management.md](memory-management.md) |
| `@arena(arena)` | Allocations | [memory-management.md](memory-management.md) |
| `@inline` | Struct fields | [memory-management.md](memory-management.md) |
| `@opaque` | FFI types | [ffi-design.md](ffi-design.md) |
| `@cstruct` | FFI types | [ffi-design.md](ffi-design.md) |
| `@cffi` | Allocations | [ffi-design.md](ffi-design.md) |
| `@owned` | FFI returns | [ffi-design.md](ffi-design.md) |
| `@borrowed` | FFI returns | [ffi-design.md](ffi-design.md) |
| `@deprecated("msg")` | Functions, types | This document |
| `@cold` | Functions | This document |

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/memory-management.md](memory-management.md), [spec/ffi-design.md](ffi-design.md), and [spec/trait-system.md](trait-system.md).*
