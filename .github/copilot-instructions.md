# Aria Language — Copilot Instructions

This repository contains the design documentation for Aria, a programming language designed for AI-assisted development.

When generating Aria code examples or working with `.aria` files:

1. Refer to `ARIA_AI_GUIDE.md` in the repository root for the complete syntax reference
2. Use `fn` for functions (not `func`, `function`, or `def`)
3. Use `:=` for variable binding (not `let`, `var`, or `val`)
4. Use `mut x :=` for mutable variables
5. No semicolons, no parentheses around conditions
6. Use `?` for error propagation, not try/catch
7. Use `Option[T]` instead of null
8. Use `match` with exhaustive patterns
9. Use `entry { }` as the program entry point
10. String interpolation uses `{expr}` inside double quotes
