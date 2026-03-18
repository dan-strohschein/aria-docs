# Aria Language

**A programming language designed for AI-assisted development.**

Aria maximizes AI code generation capabilities: minimal token waste, maximum correctness through its type system, fast compilation, high runtime performance, and cross-platform portability.

## Design Principles

1. **Every token carries meaning** — no boilerplate, no ceremony
2. **The type system is the AI's pair programmer** — rich types, exhaustive matching, effect tracking
3. **Compilation is instantaneous** — unambiguous grammar, incremental builds, two-tier backend
4. **Performance is opt-in granular** — GC by default, manual control when needed
5. **Simplicity for humans** — no dependency hell, no security nightmares, intuitive project structure

## Documentation

| Document | Description | Status |
|---|---|---|
| [Language Spec v0.1](high-level-design.md) | Core language design, syntax, and design pillars | ✅ Complete |
| [Formal Grammar](spec/formal-grammar.md) | Complete EBNF grammar for parser implementors | ✅ Draft |
| [Operator Precedence](spec/operator-precedence.md) | Complete precedence table and parsing rules | ✅ Draft |
| [Scoping Rules](spec/scoping-rules.md) | Name resolution, shadowing, and variable lifecycle | ✅ Draft |
| [Standard Library Design](spec/stdlib-design.md) | Built-in modules, APIs, and tier system | ✅ Complete |
| [Package & Module System](spec/package-module-system.md) | Module declarations, imports, dependency management | ✅ Complete |
| [FFI Considerations](spec/FFI-Considerations.md) | Design discussion and rationale for FFI decisions | ✅ Complete |
| [FFI Design Specification](spec/ffi-design.md) | Foreign function interface for C interop | ✅ Complete |
| [String Handling](spec/string-handling.md) | String types, interpolation, and text processing | ✅ Draft |
| [Numeric Types & Overflow](spec/numeric-overflow.md) | Integer/float behavior, overflow semantics | ✅ Draft |
| [Paradigm Design](spec/paradigm-design.md) | Core programming paradigm design | ✅ Draft |
| [Language Spec Addendum](spec/language-spec-addendum.md) | Pipeline operator & destructuring | ✅ Draft |
| [Compiler Architecture](spec/compiler-architecture.md) | Compilation pipeline, two-tier backend, cross-compilation | ✅ Draft |

## Roadmap

- **Phase 1**: Language spec, standard library, module system, FFI design (documentation only) — ✅ **Complete**
- **Phase 1.5**: Paradigm design, language philosophy, and remaining spec refinements — 🔄 **In Progress**
- **Phase 2**: Compiler implementation (bootstrap)
- **Phase 3**: Write a real program in Aria to stress-test the design

## Repository Structure

```
aria-docs/
├── README.md                           # This file
├── high-level-design.md                # Language spec v0.1 — core syntax and design pillars
└── spec/
    ├── formal-grammar.md               # Complete EBNF grammar for parser implementors
    ├── operator-precedence.md          # Complete precedence table and parsing rules
    ├── scoping-rules.md                # Name resolution, shadowing, and variable lifecycle
    ├── stdlib-design.md                # Standard library module design
    ├── package-module-system.md        # Package, module, and dependency management
    ├── paradigm-design.md              # Paradigm philosophy and language design rationale
    ├── language-spec-addendum.md       # Pipeline operator and destructuring addendum
    ├── FFI-Considerations.md           # FFI design discussion and rationale
    ├── ffi-design.md                   # Formal FFI specification
    ├── string-handling.md              # String types, interpolation, and text processing
    └── numeric-overflow.md             # Integer/float behavior, overflow semantics
```

## License

TBD
