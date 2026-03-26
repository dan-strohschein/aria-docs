# Aria Language

**A programming language designed for AI-assisted development.**

Aria maximizes AI code generation capabilities: minimal token waste, maximum correctness through its type system, fast compilation, high runtime performance, and cross-platform portability.

## Design Principles

1. **Every token carries meaning** — no boilerplate, no ceremony
2. **The type system is the AI's pair programmer** — rich types, exhaustive matching, effect tracking
3. **Compilation is instantaneous** — unambiguous grammar, incremental builds, two-tier backend
4. **Performance is opt-in granular** — GC by default, manual control when needed
5. **Simplicity for humans** — no dependency hell, no security nightmares, intuitive project structure

## AI Code Generation

Aria is designed to be generated correctly by AI models *without specific training*. The language's unambiguous grammar, minimal syntax, and rich type system make it inherently easy for AI to produce correct code from specification alone.

- **[AI Code Generation Guide](ARIA_AI_GUIDE.md)** — inject this into any AI assistant's context for instant Aria fluency
- **[Example Programs](examples/)** — 16 reference programs covering every language feature
- **[Copilot Instructions](.github/copilot-instructions.md)** — automatic context for GitHub Copilot

## Documentation

| Document | Description | Status |
|---|---|---|
| [Language Spec v0.1](high-level-design.md) | Core language design, syntax, and design pillars | ✅ Complete |
| [AI Code Generation Guide](ARIA_AI_GUIDE.md) | Quick reference for AI coding assistants | ✅ Complete |
| [AI Assistant Guide](CLAUDE.md) | Conventions and rules for AI assistants working on this repo | ✅ Complete |
| [Formal Grammar](spec/formal-grammar.md) | Complete EBNF grammar for parser implementors | ✅ Draft |
| [Operator Precedence](spec/operator-precedence.md) | Complete precedence table and parsing rules | ✅ Draft |
| [Scoping Rules](spec/scoping-rules.md) | Name resolution, shadowing, and variable lifecycle | ✅ Draft |
| [Standard Library Design](spec/stdlib-design.md) | Built-in modules, APIs, and tier system | ✅ Complete |
| [Package & Module System](spec/package-module-system.md) | Module declarations, imports, dependency management | ✅ Complete |
| [Paradigm Design](spec/paradigm-design.md) | Expression-oriented procedural paradigm design | ✅ Draft |
| [Language Spec Addendum](spec/language-spec-addendum.md) | Pipeline operator & destructuring | ✅ Draft |
| [Trait System](spec/trait-system.md) | Trait declarations, bounds, derives, built-in traits | ✅ Draft |
| [Generics & Type Parameters](spec/generics-type-parameters.md) | Generic functions, structs, monomorphization | ✅ Draft |
| [Pattern Matching](spec/pattern-matching.md) | Match expressions, exhaustiveness, pattern types | ✅ Draft |
| [Error Handling](spec/error-handling.md) | Error declaration, propagation, recovery, traces | ✅ Complete |
| [Effect System](spec/effect-system.md) | Effect declarations, purity, propagation | ✅ Draft |
| [Memory Management](spec/memory-management.md) | GC, @stack, @arena, @inline, Drop, ownership | ✅ Draft |
| [Concurrency Design](spec/concurrency-design.md) | Tasks, scope, channels, select, cancellation | ✅ Complete |
| [Closures & Capture Semantics](spec/closures-capture-semantics.md) | Closure syntax, capture by value/ref, once | ✅ Draft |
| [Iteration Protocol](spec/iteration-protocol.md) | Iterable/Iterator traits, lazy chains | ✅ Draft |
| [Type Conversions](spec/type-conversions.md) | Convert/TryConvert traits, three conversion mechanisms | ✅ Draft |
| [Initialization & Zero Values](spec/initialization-zero-values.md) | Zero values and struct initialization | ✅ Draft |
| [String Handling](spec/string-handling.md) | String types, interpolation, and text processing | ✅ Draft |
| [Numeric Types & Overflow](spec/numeric-overflow.md) | Integer/float behavior, overflow semantics | ✅ Draft |
| [DateTime Design](spec/datetime-design.md) | Date/time types and operations | ✅ Draft |
| [Equality & Comparison](spec/equality-comparison.md) | Eq, Ord, Hash traits, structural equality, float handling | ✅ Draft |
| [Newtypes & Aliases](spec/newtype-aliases.md) | Newtypes (distinct types) vs aliases (interchangeable) | ✅ Draft |
| [Const Evaluation](spec/const-evaluation.md) | Compile-time constants and constant expressions | ✅ Draft |
| [Annotations](spec/annotations.md) | Closed annotation set, @deprecated, @cold, no variadics | ✅ Draft |
| [Testing Framework](spec/testing-framework.md) | Test blocks, assertions, mocking, dbg, property testing | ✅ Draft |
| [Compiler Diagnostics](spec/compiler-diagnostics.md) | Structured errors, error codes, JSON output, fix suggestions | ✅ Draft |
| [Design Decisions v0.1](spec/design-decisions-v01.md) | Resolved design questions: struct/type, operators, `as`, recursion, mutability, closures, `with`, bootstrap | ✅ Complete |
| [Garbage Collector](spec/garbage-collector.md) | Per-task nurseries, concurrent old-gen, write barriers, diagnostics | ✅ Complete |
| [Task Scheduler](spec/task-scheduler.md) | Work-stealing, I/O suspension, stack management, preemption | ✅ Complete |
| [Compiler Architecture](spec/compiler-architecture.md) | Compilation pipeline, two-tier backend, cross-compilation | ✅ Draft |
| [FFI Design Specification](spec/ffi-design.md) | Foreign function interface for C interop | ✅ Complete |
| [FFI Considerations](spec/FFI-Considerations.md) | Design discussion and rationale for FFI decisions | ✅ Complete |

## Roadmap

- **Phase 1**: Language spec, standard library, module system, FFI design (documentation only) — ✅ **Complete**
- **Phase 1.5**: Paradigm design, language philosophy, and remaining spec refinements — ✅ **Complete**
- **Phase 1.75**: Core type system specs (traits, generics, effects, memory, pattern matching) — ✅ **Complete**
- **Phase 1.9**: Equality, newtypes, const, annotations, testing, diagnostics — ✅ **Complete**
- **Phase 2**: Compiler implementation (bootstrap)
- **Phase 3**: Write a real program in Aria to stress-test the design

## Repository Structure

```
aria-docs/
├── README.md                           # This file
├── ARIA_AI_GUIDE.md                    # Quick reference for AI code generators
├── high-level-design.md                # Language spec v0.1 — core syntax and design pillars
├── .github/
│   └── copilot-instructions.md         # Automatic context for GitHub Copilot
├── examples/                           # 16 example programs
│   ├── 01-hello-world.aria
│   ├── 02-types-and-variables.aria
│   ├── 03-functions.aria
│   ├── 04-control-flow.aria
│   ├── 05-error-handling.aria
│   ├── 06-structs-and-traits.aria
│   ├── 07-enums-and-pattern-matching.aria
│   ├── 08-collections-and-pipelines.aria
│   ├── 09-concurrency.aria
│   ├── 10-error-handling-advanced.aria
│   ├── 11-modules-and-imports.aria
│   ├── 12-resource-management.aria
│   ├── 13-testing.aria
│   ├── 14-generics.aria
│   ├── 15-real-world-http-server.aria
│   └── 16-real-world-cli-tool.aria
└── spec/                               # 33 formal specifications
    ├── formal-grammar.md               # Complete EBNF grammar
    ├── operator-precedence.md          # Precedence table and parsing rules
    ├── scoping-rules.md                # Name resolution, shadowing, variable lifecycle
    ├── stdlib-design.md                # Standard library modules and APIs
    ├── package-module-system.md        # Modules, imports, dependency management
    ├── paradigm-design.md              # Expression-oriented procedural paradigm
    ├── language-spec-addendum.md       # Pipeline operator and destructuring
    ├── trait-system.md                 # Trait declarations, bounds, derives, built-in traits
    ├── generics-type-parameters.md     # Generic functions, structs, monomorphization
    ├── pattern-matching.md             # Match expressions, exhaustiveness, pattern types
    ├── error-handling.md               # Error declaration, propagation, recovery, traces
    ├── effect-system.md                # Effect declarations, purity, propagation
    ├── memory-management.md            # GC, @stack, @arena, @inline, Drop, ownership
    ├── equality-comparison.md          # Eq, Ord, Hash, structural equality, floats
    ├── newtype-aliases.md              # Newtypes (distinct) vs aliases (interchangeable)
    ├── const-evaluation.md             # Compile-time constants
    ├── annotations.md                  # Closed annotation set, @deprecated, @cold
    ├── testing-framework.md            # Tests, assertions, mocking, dbg, property testing
    ├── compiler-diagnostics.md         # Structured errors, error codes, fix suggestions
    ├── design-decisions-v01.md         # Resolved design questions for v0.1
    ├── garbage-collector.md            # GC algorithm: per-task nurseries, concurrent old-gen
    ├── task-scheduler.md               # Work-stealing, I/O suspension, stack management
    ├── concurrency-design.md           # Tasks, scope, channels, select, cancellation
    ├── closures-capture-semantics.md   # Closure syntax, capture by value/ref
    ├── iteration-protocol.md           # Iterable/Iterator traits, lazy chains
    ├── type-conversions.md             # Convert/TryConvert, three conversion mechanisms
    ├── initialization-zero-values.md   # Zero values and struct initialization
    ├── string-handling.md              # String types, interpolation, text processing
    ├── numeric-overflow.md             # Integer/float overflow behavior
    ├── datetime-design.md              # Date/time types and operations
    ├── compiler-architecture.md        # Compilation pipeline, two-tier backend
    ├── ffi-design.md                   # Foreign function interface for C interop
    └── FFI-Considerations.md           # FFI design discussion and rationale
```

## License

TBD
