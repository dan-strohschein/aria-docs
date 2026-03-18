# Aria Compiler Architecture

**Status: Draft**

This document describes the architecture of the Aria compiler — how source code moves from `.aria` files to native executables. It is consistent with the design philosophy in [high-level-design.md](../high-level-design.md): every token carries meaning, type system as guardrails, instantaneous compilation, opt-in granular performance.

---

## 1. Overview

Aria compiles directly to native machine code. There is no interpreter, no virtual machine, and no JIT compilation step at runtime.

**Deployment model** — identical to Go:

1. Build the program on any supported machine: `aria build --release`
2. Copy the resulting binary to the target machine
3. Run it — no runtime installation, no dependencies, no dynamic linking required

Every Aria binary is a single standalone executable that contains:

- All compiled user code, statically linked
- All imported Aria packages, statically linked
- The entire Aria runtime (garbage collector, task scheduler, channels, stack management)
- C FFI shims (if any), linked at compile time

The binary has no external dependencies beyond the OS kernel. This is the same model used by Go and Zig — the resulting file is truly self-contained.

---

## 2. Compilation Pipeline

Source code passes through eight sequential stages. Each stage has a single well-defined input and output.

```
Source files (.aria)
       │
       ▼
  1. Lexing ──────────────── Token stream
       │
       ▼
  2. Parsing ─────────────── AST (Abstract Syntax Tree)
       │
       ▼
  3. Name & Import           Resolved AST with symbol table
     Resolution ─────────── and dependency graph
       │
       ▼
  4. Type Checking &         Typed AST, effect annotations
     Inference ──────────── exhaustive match verification
       │
       ▼
  5. IR Generation ────────── Aria IR (typed, SSA-form)
       │
       ├─── debug ──────────── Tier 1: Fast Backend
       │                        │
       └─── release ─────────── Tier 2: LLVM Backend
                                │
                          6. Optimization (release only)
                                │
                          7. Code Generation
                                │
                          8. Linking
                                │
                         Native Executable
```

### Stage 1: Lexing

The lexer converts raw source text into a flat stream of tokens. Aria's grammar is designed to be **lexically unambiguous** — the lexer requires minimal lookahead and never needs to backtrack.

Properties:
- Single-pass tokenization with one character of lookahead in nearly all cases
- No context-dependent tokenization (unlike C++ where `>>` can be either right-shift or closing nested template brackets)
- Keywords are reserved; no contextual keywords that mean different things in different positions
- String interpolation (`"Hello, {name}"`) is handled at the lex stage by emitting an interpolation token sequence

### Stage 2: Parsing

The parser builds an **Abstract Syntax Tree (AST)** from the token stream.

Properties:
- **No context-dependent parsing.** Unlike C++ (which requires a symbol table during parsing to disambiguate `A * B`) or Rust (which has complex grammar interactions requiring lookahead), Aria's grammar is always unambiguous at the syntactic level.
- **Top-down recursive descent** — straightforward to implement, easy to produce precise error messages
- The AST preserves full source location information on every node — critical for error reporting and IDE integration
- Each module is parsed independently; cross-module references are not resolved at this stage

### Stage 3: Name Resolution & Import Resolution

This stage builds the complete **symbol table** and **module dependency graph**.

Name resolution:
- Resolves every identifier to its definition site (local binding, module-level declaration, or imported symbol)
- Detects unresolved names, shadowing, and cycles in declarations
- Builds a deterministic definition order for module-level values

Import resolution:
- Resolves `use` declarations to specific package versions from the lock file
- Downloads missing dependencies if required (or errors in offline mode)
- Constructs the full dependency graph for the current build
- Detects import cycles (which are an error in Aria)

Output: a resolved AST where every name is annotated with the canonical identifier of its definition.

### Stage 4: Type Checking & Inference

Aria uses **bidirectional type inference** — types flow both inward (from context) and outward (from expressions). This eliminates most type annotations while maintaining full static typing.

Checks performed:
- **Type unification** — every expression is assigned a concrete type; mismatches are errors
- **Effect tracking verification** — if a function is annotated `pure`, the checker verifies it calls no effectful functions; if it is annotated with specific effects, all callees must be compatible
- **Exhaustive match checking** — every `match` on a sum type must cover all variants; the compiler errors if any case is missing
- **Ownership and borrow checking** (for `@managed` blocks) — verifies that manual memory management regions have no use-after-free or double-free
- **Error propagation correctness** — `?` can only be used in functions that return a `! Error` type; the checker verifies this

Output: a fully typed AST with inferred types filled in, effect annotations on every function, and exhaustiveness proofs for all match expressions.

### Stage 5: IR Generation

The typed AST is lowered to **Aria IR** — an internal representation that is:

- **Typed** — every IR value carries its type; no type erasure at this stage
- **SSA-form** (Static Single Assignment) — each variable is assigned exactly once; this simplifies optimization analysis significantly
- **Explicit** about control flow — no implicit fall-through; every branch target is named
- **Explicit** about allocations — GC-managed allocations, stack allocations, and `@manual` allocations are distinguished at the IR level
- **Platform-independent** — the IR does not contain any target-specific instructions

Aria IR is the boundary between the frontend and the two backend tiers. Both Tier 1 and Tier 2 consume identical IR. This means the same frontend can target any backend, and adding a new target requires only a new backend, not changes to the frontend.

### Stage 6: Optimization (Release Builds Only)

In release mode, Aria IR is translated to **LLVM IR** and passed through LLVM's full optimization pipeline, including:

- Function inlining
- Dead code elimination
- Scalar replacement of aggregates (SROA)
- Loop vectorization and unrolling
- Constant folding and propagation
- Alias analysis
- Profile-guided optimization (PGO) — planned

In debug mode, this stage is skipped entirely.

### Stage 7: Code Generation

Code generation produces machine code for the target architecture.

- **Tier 1 (debug)** — Aria's own fast code generator emits machine code directly from Aria IR, optimizing for compilation speed rather than output quality. This is comparable to Go's SSA backend: correct and fast to compile, but not heavily optimized.
- **Tier 2 (release)** — LLVM's code generation backend produces highly optimized machine code for the target. LLVM handles instruction selection, register allocation, and instruction scheduling.

Both tiers produce object files (or directly emit machine code for linking).

### Stage 8: Linking

The linker combines:

- All compiled user modules (object files)
- All compiled dependency modules (from cache)
- The Aria runtime (pre-compiled for the target architecture)
- C FFI stubs (if any)

The result is a single statically linked native executable. There are no shared libraries at runtime. The binary does not require any Aria installation on the target machine.

---

## 3. Two-Tier Backend Architecture

Aria has two completely distinct code generation backends. They both consume the same Aria IR, so the frontend (stages 1–5) is shared and there is no duplication of type checking or optimization analysis.

### Tier 1: Fast Backend (Debug)

**Used for**: `aria build`, `aria run`, `aria test`

**Goal**: Millisecond compilation for tight iteration loops

The fast backend is Aria's own custom code generator. It does not use LLVM. It translates Aria IR to machine code using simple, predictable patterns — register allocation is fast (linear scan or similar), no complex optimization passes are run, and the output is correct but not performance-tuned.

Properties:
- **Compilation speed** — the primary design goal is to make `aria run` feel instantaneous
- **Debug symbols** — DWARF debug information is emitted for debugger support
- **No optimization** — function calls are not inlined, dead code is not eliminated; this intentionally preserves source structure for debugging
- **Predictable output** — generated code closely corresponds to what the programmer wrote, making debugger behavior intuitive

Expected compilation speed: comparable to Go's compiler — a large project compiles in under a second.

### Tier 2: LLVM Backend (Release)

**Used for**: `aria build --release`

**Goal**: Maximum runtime performance, approaching C (90–95%)

The LLVM backend translates Aria IR to LLVM IR and invokes the full LLVM compilation and optimization pipeline. Compilation is significantly slower than Tier 1, but the generated code benefits from decades of compiler optimization research.

Optimization passes enabled:
- Full inlining (including cross-module inlining via LTO)
- Vectorization (auto-vectorization for SIMD)
- Dead code and dead allocation elimination
- Loop optimizations (unrolling, fusion, interchange)
- Tail call optimization
- Link-time optimization (LTO) — planned

Expected performance: 90–95% of equivalent C code, consistent with Go's release builds.

### Backend Swappability

Because both tiers consume identical Aria IR, they are fully interchangeable. The compiler selects the backend based on build flags, and the user never has to think about it:

```
aria build              # Tier 1 — fast, for iteration
aria build --release    # Tier 2 — slow compile, fast binary
```

This architecture also means that future backends (e.g., a WebAssembly backend, a GPU backend, or a future self-hosted backend) require no changes to the frontend.

---

## 4. Incremental & Parallel Compilation

### Module-Level Compilation Units

The unit of compilation in Aria is the **module** (a single `.aria` file). Modules are compiled independently. If a module has not changed since the last build and none of its dependencies have changed, it is not recompiled.

### Content-Addressed Cache

Compilation artifacts are stored in a content-addressed cache keyed by a hash of:

- The module's source content
- The hashes of all its dependencies
- The compiler version
- The build flags (debug vs. release, target architecture)

If the cache key matches, the cached object file is reused directly. This is the same approach used by Zig's build system. The effect is that builds are incremental by default, with no build system configuration required.

The cache is located at `~/.aria/cache/` and is shared across projects on the same machine.

### Parallel Compilation

Stages that can be parallelized are:

- **Lexing and parsing** — each module is parsed independently; all modules can be parsed in parallel
- **Type checking** — modules whose dependencies have already been type-checked can be checked in parallel (respects the dependency ordering)
- **Code generation** — each module is compiled to object code independently and in parallel

The compiler uses a work-stealing thread pool sized to the number of available CPU cores. Large projects see near-linear speedup from parallelism.

### Dependency-Aware Invalidation

When a module changes, only the modules that **transitively depend** on it are invalidated. The compiler computes the reverse dependency graph at the start of each build and queues only the affected modules for recompilation.

Interface changes (exported function signatures, type definitions) trigger downstream recompilation. Pure implementation changes (function bodies that do not affect the public interface) do not.

### `aria check` — Type-Check Only

```
aria check
```

Runs only stages 1–4 (lex, parse, name resolution, type checking). Code generation and linking are skipped. This is the fastest possible feedback loop for catching errors during development.

`aria check` is the command that a language server or editor should invoke on file save.

---

## 5. Cross-Compilation

Aria supports cross-compilation as a first-class feature, using the same model as Go.

### Built-In Target Triples

| Target | Architecture | OS |
|---|---|---|
| `linux-amd64` | x86-64 | Linux |
| `linux-arm64` | AArch64 | Linux |
| `darwin-amd64` | x86-64 | macOS |
| `darwin-arm64` | AArch64 (Apple Silicon) | macOS |
| `windows-amd64` | x86-64 | Windows |
| `wasm32` | WebAssembly (32-bit) | Browser / WASI |

### No External Toolchain Required

For all supported targets, Aria ships with its own linker and target-specific runtime stubs. No external toolchain (no GCC, no binutils, no MSVC) is needed for cross-compilation. This is the same approach taken by Zig.

To cross-compile:

```
aria build --target linux-arm64
aria build --target windows-amd64
aria build --target wasm32
```

The resulting binary can be copied to the target machine and run directly.

### Platform-Specific Code

Code that must vary by platform uses conditional compilation blocks:

```
fn getConfigDir() -> str {
    @os(linux) {
        "/etc/aria"
    }
    @os(darwin) {
        "/Library/Application Support/aria"
    }
    @os(windows) {
        "C:/ProgramData/aria"
    }
}
```

The compiler type-checks all branches for all platforms, even when cross-compiling. Platform-specific code that contains type errors on non-targeted platforms is still an error. This prevents the common problem of platform-specific bugs only being caught on the target platform.

### Target-Specific Runtime Variants

The Aria runtime has platform-specific variants that account for differences in:

- **Memory page sizes** (4 KB on x86-64 Linux, 16 KB on Apple Silicon — the M-series chips have a 16 KB hardware page size)
- **OS-level thread APIs** (pthreads vs. Win32 threads)
- **Signal handling** (POSIX signals vs. Windows structured exception handling)
- **GC tuning parameters** (adjusted for typical memory characteristics per platform)

All of this is handled automatically — the user selects a target and gets an appropriately tuned binary.

---

## 6. Embedded Runtime

Every Aria binary includes the full Aria runtime, statically linked. There is no separate runtime installation. The runtime is transparent to the user but provides the foundation for all concurrency and memory management features.

### Garbage Collector

Aria uses a **concurrent, generational garbage collector** by default.

Design properties:
- **Generational** — short-lived allocations (the common case) are collected very cheaply in the young generation; long-lived objects are promoted to the old generation and collected less frequently
- **Concurrent** — GC mark phase runs concurrently with user code on a background thread, minimizing stop-the-world pauses
- **Stop-the-world pauses** are bounded and short (target: <1ms for most workloads)
- **Write barriers** are used to track cross-generation pointers for the young-generation collector

GC tuning parameters can be adjusted at startup:

```
ARIA_GC_HEAP_MAX=512m aria run main.aria
```

Functions annotated `@manual` or within `@manual { }` blocks use explicit allocation and are not subject to GC. This allows hot paths to opt out of GC entirely.

### Task Scheduler

Aria's concurrency model is based on lightweight tasks (`spawn`) and structured concurrency (`scope { }`). The task scheduler provides the runtime for these.

Design properties:
- **M:N scheduling** — many lightweight tasks multiplexed across a small number of OS threads (one per CPU core by default)
- **Work stealing** — idle OS threads steal tasks from busy threads' queues for load balancing
- **Cooperative + preemptive** — tasks yield at I/O boundaries and at safe points injected by the compiler; long-running CPU-bound tasks are preempted by the scheduler
- **Structured concurrency** — `scope { }` blocks ensure all spawned tasks complete (or are cancelled) before the scope exits; no task can outlive its enclosing scope

### Channel Implementation

Channels are the primary communication mechanism between tasks.

Properties:
- **Buffered channels** — a fixed-capacity queue; `send` blocks when full, `receive` blocks when empty
- **Unbuffered channels** — a rendezvous point; `send` and `receive` block until both sides are ready
- **Select** — wait on multiple channels simultaneously, proceeding when any one is ready
- Channels are first-class values and can be passed to functions, stored in structs, and sent through other channels

### Stack Management

Lightweight tasks use **growable stacks**.

- Initial stack size is small (a few KB) to allow spawning thousands of tasks without exhausting memory
- When a task's stack is nearly full, the runtime transparently allocates a larger stack segment and copies the stack frame
- Maximum stack depth is configurable; exceeding it is a runtime error (not silent memory corruption)
- The OS thread stack (for the scheduler itself) is a fixed-size stack

### Runtime Overhead

The runtime is designed to impose near-zero cost on code that does not use its features:

- A function that does not allocate, does not spawn tasks, and does not use channels compiles to code identical to C — no runtime overhead
- GC write barriers are only emitted in functions that perform heap allocation
- The scheduler's preemption check points are infrequent (injected at loop back-edges and function entries)

---

## 7. CLI Interface

The `aria` command is both the build tool and the package manager. There is no separate `make`, `cmake`, `cargo`, or `go build` equivalent — `aria` does it all.

```
# Development builds (Tier 1 backend — fast)
aria build              # compile current project, debug mode
aria run                # build + run (most common during development)
aria test               # build + run all inline tests
aria check              # type-check only, no code generation (fastest feedback)

# Release build (Tier 2 / LLVM backend — optimized)
aria build --release    # produce an optimized release binary

# Cross-compilation
aria build --target linux-amd64
aria build --target linux-arm64
aria build --target darwin-amd64
aria build --target darwin-arm64
aria build --target windows-amd64
aria build --target wasm32

# Cross-compile + release can be combined
aria build --release --target linux-arm64

# Package management
aria add <package>      # add a dependency
aria remove <package>   # remove a dependency
aria update             # update dependencies to latest compatible versions
```

### Build Artifacts

```
project/
├── aria.toml           # project manifest
├── aria.lock           # dependency lock file (committed to VCS)
├── src/
│   └── main.aria
└── .aria/
    ├── build/          # build artifacts (debug)
    │   └── main        # debug binary
    └── release/        # build artifacts (release)
        └── main        # release binary
```

The `.aria/` directory is a build cache and should be added to `.gitignore`.

---

## 8. Error Reporting

Aria is designed to produce **actionable error messages**. An error message that does not tell the programmer what to do is considered a compiler bug.

### Source Location

Every error references the exact source location — file, line, and column — and prints the relevant source line with a caret pointing to the problematic token or expression:

```
error[E0042]: type mismatch
  --> src/server.aria:47:12
   |
47 |     port := "8080"
   |             ^^^^^^ expected u16, found str
   |
help: try converting: port := "8080".parse[u16]()?
```

### Error Chains with `?`

When the `?` propagation operator is used, Aria automatically adds context to the error chain. Each propagation point adds the current function name and source location to the error:

```
error: failed to start server
  at src/server.aria:12 in Server.start()
  at src/main.aria:8 in main()
caused by: failed to bind to port 8080
  at src/net/tcp.aria:234 in TcpListener.bind()
caused by: address already in use (os error 98)
```

This error chain is constructed automatically — the programmer does not write it manually.

### Suggested Fixes

Where the compiler can determine a correct fix with high confidence, it emits a `help:` suggestion. These suggestions are designed to be copy-pasteable. Examples:

- **Missing `?`**: "this function returns a Result; add `?` to propagate the error"
- **Wrong type**: "try converting: `value.to[TargetType]()`"
- **Exhaustiveness**: "missing match arm for variant `.Cancelled`"
- **Unused import**: "remove unused import: `use net/http`"

### Structured Output for Tooling

In addition to human-readable terminal output, `aria check --json` emits a structured JSON format:

```json
{
  "errors": [
    {
      "code": "E0042",
      "message": "type mismatch",
      "file": "src/server.aria",
      "line": 47,
      "column": 12,
      "span_length": 6,
      "help": "try converting: port := \"8080\".parse[u16]()?"
    }
  ]
}
```

This format is consumed by the LSP server and editor integrations.

---

## 9. Future Considerations

### Self-Hosting

The initial Aria compiler (bootstrap compiler) will be written in a systems language — most likely Go or Rust. Once the language is mature, the compiler itself will be rewritten in Aria. This is a milestone that demonstrates the language is capable of real systems programming.

Self-hosting also provides a significant correctness benefit: the compiler becomes the largest real-world test of the language and standard library.

### Language Server Protocol (LSP)

An LSP server for editor integration (VS Code, Neovim, Helix, etc.) will provide:

- Real-time `aria check` diagnostics as the user types
- Go-to-definition and find-references
- Auto-complete based on type information
- Inline type annotations
- Suggested fixes from the compiler applied as code actions

The LSP server will reuse the compiler's frontend (stages 1–4) directly, running incrementally as files change.

### Debug Info (DWARF)

Both backends will emit DWARF debug information so that standard debuggers (GDB, LLDB) can:

- Set breakpoints by source file and line number
- Inspect local variables with their Aria types
- Print stack traces with source locations
- Step through Aria source code instruction by instruction

### Profile-Guided Optimization (PGO)

A future release will support PGO via a two-pass compilation workflow:

```
aria build --release --instrument   # emit instrumented binary
./main                              # run with realistic workload
aria build --release --pgo          # recompile using collected profile
```

PGO allows the compiler to make better inlining, branch prediction, and code layout decisions based on actual usage patterns, potentially pushing performance above the 95% of C baseline.

### Link-Time Optimization (LTO)

Cross-module LTO via LLVM's ThinLTO will allow function inlining and optimization across module boundaries in release builds. This requires no changes from the programmer — enabling it is a compiler flag.

### Additional Targets

Planned future compilation targets:

- **RISC-V** (`riscv64`) — emerging server and embedded target
- **WASI** (WebAssembly System Interface) — server-side WebAssembly
- **Bare metal** (`linux-arm64-nolibc`) — embedded systems without an OS

---

*This document is part of the Aria language specification. See [high-level-design.md](../high-level-design.md) for the core language design.*
