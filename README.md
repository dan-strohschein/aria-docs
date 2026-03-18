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
| [Standard Library Design](spec/stdlib-design.md) | Built-in modules, APIs, and tier system | ✅ Complete |
| [Package & Module System](spec/package-module-system.md) | Module declarations, imports, dependency management | ✅ Complete |
| [FFI Considerations](spec/FFI-Considerations.md) | Design discussion and rationale for FFI decisions | ✅ Complete |
| [FFI Design Specification](spec/ffi-design.md) | Foreign function interface for C interop | ✅ Complete |
| [AI Code Generation Guide](ARIA_AI_GUIDE.md) | AI context reference for code generation | ✅ Draft |
| Paradigm & Language Philosophy | Core programming paradigm design | 🔜 Next |
| Compiler Architecture | Bootstrap compiler design | 🔜 Planned |

## Roadmap

- **Phase 1**: Language spec, standard library, module system, FFI design (documentation only) — ✅ **Complete**
- **Phase 1.5**: Paradigm design, language philosophy, and remaining spec refinements — 🔄 **In Progress**
- **Phase 2**: Compiler implementation (bootstrap)
- **Phase 3**: Write a real program in Aria to stress-test the design

## Repository Structure

```
aria-docs/
├── README.md                           # This file
├── ARIA_AI_GUIDE.md                    # AI context reference for code generation
├── high-level-design.md                # Language spec v0.1 — core syntax and design pillars
├── .github/
│   └── copilot-instructions.md         # Automatic context for GitHub Copilot
├── examples/                           # 16 reference programs covering every language feature
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
└── spec/
    ├── stdlib-design.md                # Standard library module design
    ├── package-module-system.md        # Package, module, and dependency management
    ├── FFI-Considerations.md           # FFI design discussion and rationale
    └── ffi-design.md                   # Formal FFI specification
```

## License

TBD
