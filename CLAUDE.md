# Aria Language — AI Assistant Guide (CLAUDE.md)

## Project Overview

Aria is a programming language designed for AI code generation. This repository contains the language specification — no compiler exists yet. The spec is the product.

### Repository Structure

```
aria-docs/
├── CLAUDE.md                          # This file — AI assistant guide
├── README.md                          # Project overview and documentation index
├── high-level-design.md               # Language spec v0.1 — core syntax and design pillars
├── ARIA_AI_GUIDE.md                   # Quick reference for AI code generators
├── examples/                          # 16 example programs (.aria files)
│   ├── 01-hello-world.aria
│   ├── ...
│   └── 16-real-world-cli-tool.aria
└── spec/                              # Formal specifications (33 files)
    ├── formal-grammar.md              # EBNF grammar (authoritative for syntax)
    ├── trait-system.md                # Trait declarations, bounds, derives, built-in traits
    ├── generics-type-parameters.md    # Generic functions, structs, type inference
    ├── memory-management.md           # GC, @stack, @arena, @inline, Drop, Clone
    ├── pattern-matching.md            # Match expressions, exhaustiveness, pattern types
    ├── effect-system.md               # Effect declarations, purity, propagation
    ├── error-handling.md              # Result types, ?, catch, error traces
    ├── equality-comparison.md         # Eq, Ord, Hash traits, float equality
    ├── newtype-aliases.md             # Newtypes (distinct) vs aliases (interchangeable)
    ├── const-evaluation.md            # Compile-time constants
    ├── annotations.md                 # Closed annotation set, @deprecated, @cold
    ├── testing-framework.md           # Tests, assertions, mocking, dbg, property testing
    ├── compiler-diagnostics.md        # Structured errors, error codes, fix suggestions
    ├── concurrency-design.md          # Tasks, scope, channels, select, Send/Share
    ├── closures-capture-semantics.md  # Capture by value, ref, once closures
    ├── iteration-protocol.md          # Iterable/Iterator traits, lazy chains
    ├── type-conversions.md            # Convert, TryConvert, trunc, no implicit conversions
    ├── paradigm-design.md             # Expression-oriented procedural + functional
    ├── stdlib-design.md               # Standard library modules and APIs
    ├── package-module-system.md       # Modules, imports, visibility
    ├── compiler-architecture.md       # Compilation pipeline, two-tier backend
    ├── scoping-rules.md               # Name resolution, shadowing
    ├── operator-precedence.md         # Precedence table
    ├── string-handling.md             # str type, interpolation
    ├── numeric-overflow.md            # Integer/float overflow behavior
    ├── initialization-zero-values.md  # Zero values and initialization
    ├── datetime-design.md             # Date/time types
    ├── ffi-design.md                  # C interop specification
    ├── FFI-Considerations.md          # FFI design rationale
    └── language-spec-addendum.md      # Pipeline operator, destructuring
```

### Purpose

Every file in this repo is a specification document or example program. There is no source code to compile. The goal is to produce a complete, consistent, implementable language design that an AI or human can use to build the Aria compiler.

---

## Language Design Principles (Non-Negotiable)

These five pillars are the foundation of every design decision. Never propose changes that violate them.

### 1. Every Token Carries Meaning

No boilerplate. No ceremony. If a pattern is repeated more than twice across typical programs, it should be a language primitive. Measure design quality in tokens-per-operation — fewer is better.

### 2. The Type System Is the AI's Pair Programmer

Sum types with exhaustive matching, trait bounds, effect tracking, and typed errors give the compiler enough information to catch bugs before runtime. The richer the type system, the fewer bugs the AI generates.

### 3. Compilation Is Instantaneous

The grammar must be unambiguous with minimal lookahead. No context-dependent parsing. Generics use `[T]` not `<T>` to avoid the turbofish problem. Two-tier backend: fast debug builds, LLVM-optimized release builds.

### 4. Performance Is Opt-In Granular

GC by default — safe and easy. Drop into manual control per-block with `@stack`, `@arena`, `@inline`. No language boundary crossing required. 95% of code uses the GC; 5% gets manual control for hot paths.

### 5. No Implicit Behavior Ever

No implicit conversions. No hidden exceptions. No null. No unchecked throws. No implicit truthiness (`0` is not `false`). If it's not visible in the source code, it doesn't happen.

---

## Core Language Identity

> **"Procedural bones, functional blood, no inheritance."**

- **Structs + Traits** — no classes, no inheritance hierarchies
- **Sum types** — tagged unions with exhaustive pattern matching
- **Expression-oriented** — `if`, `match`, blocks all return values
- **Pipeline operator** — `|>` for left-to-right data transformation
- **Effect tracking** — `with [Io, Fs]` declares side effects
- **Typed errors** — `! ErrorType` in signatures, `?` propagation

---

## Spec File Conventions

When writing or modifying a spec file, follow these conventions:

### File Naming

`spec/topic-subtopic.md` — lowercase, hyphenated. Examples: `trait-system.md`, `memory-management.md`.

### Document Structure

1. **Title** — `# Aria [Topic] Specification`
2. **Bold one-line summary** — what this spec covers
3. **Provenance** — which section of `high-level-design.md` this formalizes
4. **Cross-references** — links to related specs
5. `---` horizontal rule
6. **Table of Contents** — for specs with 5+ sections
7. **Design Philosophy** — why this feature exists, through the AI code generation lens
8. **Numbered content sections** — the actual specification
9. **Design Rationale Summary** — table of decisions and rationale
10. **Comparison with Other Languages** — where relevant

### Writing Rules

- Every non-trivial design decision needs an `**AI rationale:**` paragraph explaining why this choice benefits AI code generation
- Include token cost comparison tables vs Go/Rust/Java where relevant
- Code examples in every major section — use bare ``` fences (no language tag), with inline `//` comments
- Cross-references use relative paths: `[spec/foo.md](foo.md)` within `spec/`, `[high-level-design.md](../high-level-design.md)` from `spec/`
- Tone: first-person AI perspective ("I", "my"), authoritative, no hedging ("perhaps", "maybe", "could")
- Grammar productions should match `formal-grammar.md` exactly

---

## What NOT to Propose

These are anti-patterns that violate Aria's core values. Never suggest them:

- **Implicit conversions** — `i32 + i64` must be a compile error
- **Inheritance** — no `class`, no `extends`, no `super`
- **Null** — use `Option[T]` (`T?` sugar)
- **Unchecked exceptions** — use `Result[T, E]` (`T ! E` sugar) and `?`
- **Verbose ceremony** — if it takes more tokens than Go for the same operation, rethink
- **Ambiguous grammar** — every construct must parse with minimal lookahead
- **Semicolons** — Aria uses newline-based statement termination
- **Parentheses around conditions** — `if x > 0 { }` not `if (x > 0) { }`
- **`async`/`await` coloring** — Aria's concurrency is colorless
- **`let`/`var`/`val`** — use `:=` and `mut`
- **Variadic functions** — use slices, named defaults, or compiler-intrinsic `println`
- **User-defined annotations** — annotations are a closed compiler set in v0.1
- **`@[derive(...)]` syntax** — use `derives [...]` postfix syntax (canonical form)
- **Reference equality** — `==` is always structural; there is no "same object" test
- **`PartialEq`** — floats don't implement `Eq`; use `approxEq` instead
- **`as` for type casts** — `as` is only for import aliases; use `.to[T]()`, `.trunc[T]()`, or `T(x)` for conversions
- **Per-field `mut`** — mutability is on bindings, not fields; `mut x := Foo{...}` makes all fields mutable
- **`Box` or pointer annotations for recursive types** — the compiler auto-boxes recursive fields
- **Rust-style `Fn`/`FnMut`/`FnOnce`** — Aria has one closure type: `fn(Args) -> Return` (GC-boxed)

---

## Key Syntax Quick Reference

```
// Variables
x := value                    // immutable
mut x := value                // mutable
x: Type = value               // explicit type

// Functions
fn name(param: Type) -> ReturnType { body }
fn short(x: i64) -> i64 = x * 2              // single-expression

// Error handling
fn fallible() -> T ! ErrorType { ... }
result?                       // propagate error
result!                       // assert success (panic on error)
result catch |e| { yield fallback }
result or default_value

// Types
type Sum = | Variant1(T) | Variant2 { field: T }
struct Name { field: Type = default }
trait Name { fn method(self) -> Type }
impl Trait for Type { fn method(self) -> Type { ... } }
impl Type { fn inherent(self) -> Type { ... } }        // inherent methods

// Generics
fn name[T: Bound](x: T) -> T
struct Container[T] { items: [T] }

// Effects
fn impure() -> T with [Io, Fs] { ... }
fn pure(x: i64) -> i64 = x * 2               // no effects = pure

// Pattern matching
match value {
    Pattern1(x) => expr
    Pattern2 { field } if guard => expr
    _ => default
}

// Concurrency
spawn task()
scope { spawn a(); spawn b() }
ch := chan[T](buffer: 10)
select { msg from ch => handle(msg); after 5s => timeout() }

// Pipeline
result := data |> transform |> filter(pred) |> collect()

// Memory annotations
x := @stack Thing{...}
x := @arena Thing{...}
```

---

## Existing Spec Index

| File | Description |
|---|---|
| `high-level-design.md` | Core language design — types, syntax, error handling, concurrency, memory |
| `spec/formal-grammar.md` | Complete EBNF grammar — authoritative for parser implementors |
| `spec/operator-precedence.md` | Operator precedence table and parsing rules |
| `spec/scoping-rules.md` | Name resolution, shadowing, variable lifecycle |
| `spec/stdlib-design.md` | Standard library modules, APIs, tier system |
| `spec/package-module-system.md` | Module declarations, imports, dependency management |
| `spec/paradigm-design.md` | Expression-oriented procedural paradigm design |
| `spec/language-spec-addendum.md` | Pipeline operator and destructuring |
| `spec/error-handling.md` | Error declaration, propagation, recovery, traces |
| `spec/concurrency-design.md` | Tasks, scope, channels, select, cancellation |
| `spec/closures-capture-semantics.md` | Closure syntax, capture by value/ref, once |
| `spec/iteration-protocol.md` | Iterable/Iterator traits, lazy chains, comprehensions |
| `spec/type-conversions.md` | Convert/TryConvert traits, three conversion mechanisms |
| `spec/initialization-zero-values.md` | Zero values and struct initialization |
| `spec/trait-system.md` | Trait declarations, bounds, derives, built-in traits |
| `spec/generics-type-parameters.md` | Generic functions, structs, monomorphization |
| `spec/memory-management.md` | GC, stack/arena/inline annotations, Drop, ownership |
| `spec/pattern-matching.md` | Match expressions, exhaustiveness, pattern types |
| `spec/effect-system.md` | Effect declarations, purity, propagation |
| `spec/equality-comparison.md` | Eq, Ord, Hash traits, structural equality, float handling |
| `spec/newtype-aliases.md` | Newtypes (distinct types) vs aliases (interchangeable) |
| `spec/const-evaluation.md` | Compile-time constants and constant expressions |
| `spec/annotations.md` | Closed annotation set, @deprecated, @cold, no variadics |
| `spec/testing-framework.md` | Test blocks, assertions, mocking, dbg, property testing |
| `spec/compiler-diagnostics.md` | Structured errors, error codes, JSON output, fix suggestions |
| `spec/design-decisions-v01.md` | Resolved design questions: operators, `as`, recursion, mutability, closures, `with`, bootstrap |
| `spec/garbage-collector.md` | Per-task nurseries, concurrent old-gen, write barriers, GC diagnostics |
| `spec/task-scheduler.md` | Work-stealing, I/O suspension, stack management, preemption |
| `spec/compiler-architecture.md` | Compilation pipeline, two-tier backend |
| `spec/string-handling.md` | String types, interpolation, text processing |
| `spec/numeric-overflow.md` | Integer/float overflow behavior |
| `spec/datetime-design.md` | Date/time types and operations |
| `spec/ffi-design.md` | Foreign function interface for C interop |
| `spec/FFI-Considerations.md` | FFI design discussion and rationale |

---

## Build & Verify

There is no compiler yet. Verification means cross-referencing consistency across specs:

1. **Syntax check** — every code example must match `formal-grammar.md` productions
2. **Cross-reference check** — no contradictions between specs (especially with `high-level-design.md`)
3. **AI rationale check** — every major design decision has an `**AI rationale:**` paragraph
4. **Code example check** — every section has at least one Aria code example
5. **Link check** — all cross-reference links resolve to existing files
6. **Terminology check** — consistent naming across all specs (e.g., always "sum type" not "algebraic data type", always "trait" not "interface")
