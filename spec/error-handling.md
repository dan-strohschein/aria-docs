# Aria Error Handling Specification

**A complete, implementable specification for error declaration, propagation, recovery, and debugging in Aria.**

This document formalizes and extends the error-handling sketches in [high-level-design.md](../high-level-design.md) into an exhaustive specification. Every design decision is justified through the lens of AI code generation — the primary consumer of this language.

Cross-references:
- [high-level-design.md](../high-level-design.md) — `!`, `?`, `catch`, `match ok/err`
- [spec/stdlib-design.md](stdlib-design.md) — `Result[T, E]`, `Option[T]`
- [spec/ffi-design.md](ffi-design.md) — C error mapping
- [spec/package-module-system.md](package-module-system.md) — module boundaries

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Error Type Declaration](#2-error-type-declaration)
3. [Function Error Signatures — The `!` Operator](#3-function-error-signatures--the--operator)
4. [Error Propagation — The `?` Operator](#4-error-propagation--the--operator)
5. [Error Handling Patterns](#5-error-handling-patterns)
6. [Error Categories — Semantic Traits](#6-error-categories--semantic-traits)
7. [Error Traces — Structured Debugging](#7-error-traces--structured-debugging)
8. [Error Composition Across Module Boundaries](#8-error-composition-across-module-boundaries)
9. [Errors and Concurrency](#9-errors-and-concurrency)
10. [Errors and the FFI Boundary](#10-errors-and-the-ffi-boundary)
11. [Errors and Iteration](#11-errors-and-iteration)
12. [The `panic` Mechanism](#12-the-panic-mechanism)
13. [Testing Error Conditions](#13-testing-error-conditions)
14. [Design Rationale Summary](#14-design-rationale-summary)
15. [Comparison with Other Languages](#15-comparison-with-other-languages)

---

## 1. Design Philosophy

Error handling is the single highest-impact design area for AI code generation quality. In every other language, errors are a continuous source of generated bugs: forgotten error checks, dropped context, incorrect retry logic, mismatched exception types. Aria eliminates these failure modes at the design level.

### Core Principles

#### Errors are data, not strings

Every error in Aria is a typed value in a sum type. There is no `error` interface with a single `.Error() string` method, no untyped exception hierarchy, no stringly-typed error messages. The error *is* the data — a specific variant with specific fields, fully pattern-matchable.

```
// The error is the data. You can access path, user, and after directly.
type IoError =
    | NotFound { path: str }
    | PermissionDenied { path: str, user: str }
    | Timeout { after: dur }
    | ConnectionRefused { addr: str, port: u16 }

// In handling code — exhaustive matching, no string parsing
match err {
    IoError.NotFound{path} => log.warn("missing: {path}")
    IoError.PermissionDenied{path, user} => deny(user, path)
    IoError.Timeout{after} => scheduleRetry(after * 2)
    IoError.ConnectionRefused{addr, port} => alertOps(addr, port)
}
```

**AI rationale:** When errors are typed sum types, the AI can generate exhaustive match arms mechanically from the type definition. When errors are strings, the AI must guess at values, parse documentation, and risk missing cases.

#### The signature tells the full story

If a function can fail, its signature declares exactly which error types it can produce. There are no hidden exceptions, no unchecked throws, no error-by-convention. The AI reads the signature and knows completely what can go wrong.

```
fn readFile(path: str) -> str ! IoError
fn initialize() -> App ! IoError | ParseError | DbError
fn add(a: i64, b: i64) -> i64   // infallible — no ! in sight
```

**AI rationale:** The AI doesn't have to read the function body to understand its error contract. The signature is a complete, machine-readable specification. This eliminates an entire class of "I read the code but missed that it could throw X" errors.

#### Propagation is cheap, context is free

The `?` operator costs one token and the compiler automatically injects call-site context (function name, file, line, column). There is no manual `.context()`, no `fmt.Errorf("...: %w", err)`, no risk of forgotten wrapping.

```
// Every ? automatically captures call-site info. Zero tokens spent on context.
fn processConfig() -> Config ! IoError | ParseError {
    content := readFile("config.json")?
    parsed := parseJson(content)?
    validateConfig(parsed)?
}
```

**AI rationale:** In Go and Rust, adding context to errors requires explicit wrapping at every propagation site. The AI often forgets this or produces inconsistent messages. In Aria, context is structural and automatic — the AI never has to think about it.

#### Recovery intent is typed

Error categories (transient vs. permanent, user-fault vs. system-fault) are encoded in the type system via traits. This enables mechanical generation of correct retry and recovery logic — the AI doesn't guess whether to retry a `NotFound` (usually not) or a `Timeout` (usually yes).

```
// Timeout is Transient + SystemFault — retry is automatic
type IoError =
    | NotFound { path: str }             derives [Permanent, UserFault]
    | Timeout { after: dur }             derives [Transient, SystemFault, Retryable]
    | PermissionDenied { path: str, user: str } derives [Permanent, UserFault]
```

**AI rationale:** When the AI needs to add retry logic, it can generate it mechanically from the `Retryable` trait. No documentation reading, no guessing.

#### Errors compose across boundaries

Error types compose cleanly across module boundaries, function boundaries, and concurrency boundaries without information loss. The `From` trait handles widening automatically; named aggregate error types handle module boundaries.

#### Token cost comparison

The following table shows the token cost for a function that makes 5 fallible calls and propagates all errors with context, in each language:

| Language | Approach | Tokens per propagation | Context added? | Example for 5 calls |
|---|---|---|---|---|
| Go | `if err != nil { return fmt.Errorf("...: %w", err) }` | ~15 | Manual | ~75 tokens |
| Rust (anyhow) | `.context("...")` or `?` | 3–8 | Manual | ~40 tokens |
| Rust (thiserror) | `?` + `#[from]` | 2 | None | ~10 tokens |
| Java | `try { ... } catch (E e) { throw new WrapperEx(e) }` | ~20 | Manual | ~100 tokens |
| Aria | `?` | 1 | **Automatic** | **5 tokens** |

---

## 2. Error Type Declaration

Error types in Aria are ordinary sum types. They use exactly the same `type ... = | Variant` syntax as any other sum type. There is no special `error` keyword, no special declaration syntax, no required interface implementation.

### Simple error enums

```
type IoError =
    | NotFound { path: str }
    | PermissionDenied { path: str, user: str }
    | Timeout { after: dur }
    | ConnectionRefused { addr: str, port: u16 }
```

Each variant can carry different fields. Fields are named (not positional). Any field type is valid as long as it implements `Debug`.

### Empty variants

Variants without fields are permitted. They convey the error category without additional data:

```
type ParseError =
    | UnexpectedEof
    | InvalidUtf8
    | InvalidJson { offset: u64, msg: str }
    | UnknownField { field: str }
```

### Error types with associated data

Every variant in an error type is independent. The fields of one variant do not affect the fields of another. This is identical to any other Aria sum type:

```
type DbError =
    | ConnectionLost { addr: str, after: dur }
    | QueryFailed { sql: str, reason: str }
    | Deadlock { tables: [str] }
    | PoolExhausted { limit: u64, waited: dur }
    | NotFound                              // empty variant — no data needed
    | Constraint { table: str, constraint: str, value: str }
```

### Derives on error types

Error types support the same derive system as all other types. The most common derives for error types are `Debug`, `Eq`, and `Display`:

```
type ConfigError =
    | MissingKey { key: str }              derives [Debug, Eq]
    | InvalidValue { key: str, value: str, expected: str }  derives [Debug, Eq]
    | ParseFailed { path: str, reason: str } derives [Debug, Eq]

derives [Debug, Eq, Display] for ConfigError
```

Derives can be applied per-variant or on the whole type. When applied to the whole type, all variants inherit the derive.

### Display formatting for error types

`Display` can be derived with a format string per variant:

```
type IoError =
    | NotFound { path: str }
        display "file not found: {path}"
    | PermissionDenied { path: str, user: str }
        display "permission denied: {user} cannot access {path}"
    | Timeout { after: dur }
        display "operation timed out after {after}"
    | ConnectionRefused { addr: str, port: u16 }
        display "connection refused: {addr}:{port}"
```

The format string uses the same interpolation syntax as Aria string templates. Field names are in scope. If no `display` is specified, the derived `Display` produces a debug-style representation.

### Error types as first-class sum types

Error types have no special runtime representation. They are ordinary sum types — tagged unions with variant data. This means:

- They can be stored in collections: `errors: [IoError]`
- They can be fields of structs: `type Report { failures: [AppError] }`
- They can be passed to functions: `fn logError(e: IoError)`
- They can be composed into unions: `IoError | ParseError`
- They can be wrapped: `type AppError = | Io { source: IoError } | Parse { source: ParseError }`

The special behavior of error types comes entirely from:
1. Their use in `!` signatures (section 3)
2. Trait implementations like `Transient`, `Retryable` (section 6)
3. Their participation in the `From` trait for automatic widening (section 8)

### Grammar

```
error-type-declaration ::=
    "type" identifier "="
    ("|" variant-decl)+

variant-decl ::=
    identifier
    ("{" field-list "}")?
    ("display" string-literal)?
    ("derives" "[" trait-list "]")?

field-list ::=
    field ("," field)*

field ::=
    identifier ":" type
```

---

## 3. Function Error Signatures — The `!` Operator

The `!` operator in a function signature declares that the function can fail and specifies which error types it can produce. This is part of the function's type — not an attribute, annotation, or convention.

### Single error type

```
fn readFile(path: str) -> str ! IoError
fn writeFile(path: str, content: str) ! IoError
fn parseInt(s: str) -> i64 ! ParseError
```

### Void return with error

When the return type is void (nothing useful on success), omit the return type entirely and keep only the error:

```
fn sendEmail(to: str, body: str) ! SmtpError
fn deleteFile(path: str) ! IoError
fn logEvent(event: Event) ! LogError
```

### Multiple error types — error union

```
fn initialize() -> App ! IoError | ParseError | DbError
fn processRequest(req: Request) -> Response ! AuthError | ValidationError | DbError | IoError
```

Error unions are anonymous sum types. `IoError | ParseError` is itself a type — the union of all variants from both types. This is not a base class or an interface — it is a structural union.

### Infallible functions

Functions with no `!` in their signature are guaranteed not to fail. The compiler enforces this — any fallible call inside an infallible function without handling is a compile error.

```
fn add(a: i64, b: i64) -> i64           // pure, infallible
fn formatDate(d: Date) -> str           // infallible — always produces a string
fn max(a: i64, b: i64) -> i64           // infallible
```

### Error union semantics

Error unions are anonymous and flat. The compiler automatically flattens nested unions:

```
// These are all equivalent:
fn f() -> T ! (A | B) | (B | C)
fn f() -> T ! A | B | C               // flattened: B appears once
```

Union flattening rules:
- Duplicate types in a union are collapsed to one
- Union order is normalized alphabetically for canonical comparison
- A named type that is itself a union is expanded: `type AppError = IoError | ParseError` expands to `IoError | ParseError`

Named aggregate types can wrap unions:

```
type AppError = IoError | ParseError | DbError

fn initialize() -> App ! AppError     // equivalent to ! IoError | ParseError | DbError
```

This is useful for module-level error types (see section 8).

### Interaction with generics

Error types participate fully in Aria's generic system:

```
fn map[T, U, E](result: Result[T, E], f: fn(T) -> U) -> Result[U, E]
fn mapErr[T, E1, E2](result: Result[T, E1], f: fn(E1) -> E2) -> Result[T, E2]
fn flatMap[T, U, E](result: Result[T, E], f: fn(T) -> Result[U, E]) -> Result[U, E]
fn sequence[T, E](results: [Result[T, E]]) -> Result[[T], E]

// A fallible function type — the closure itself can fail
type FallibleOp[T, E] = fn() -> T ! E

fn retry[T, E: Transient](attempts: u64, op: FallibleOp[T, E]) -> T ! E
```

### Grammar specification

```
function-signature ::=
    "fn" identifier
    generic-params?
    "(" param-list? ")"
    ("->" type)?
    ("!" error-type)?

error-type ::=
    named-type
    | error-union

error-union ::=
    named-type ("|" named-type)+

generic-params ::=
    "[" generic-param ("," generic-param)* "]"

generic-param ::=
    identifier (":" trait-bound)?
```

---

## 4. Error Propagation — The `?` Operator

The `?` operator is the primary tool for error propagation. It is postfix, it costs one token, and it is the recommended approach for the vast majority of error handling.

### Basic propagation

```
fn processConfig() -> Config ! IoError {
    content := readFile("config.json")?   // if Err, return the error immediately
    config := parseConfig(content)?        // if Err, return the error immediately
    config                                 // Ok — return the value
}
```

When `?` is applied to an expression of type `Result[T, E]`:
- If the value is `Ok(v)`, the expression evaluates to `v` (type `T`)
- If the value is `Err(e)`, the enclosing function returns `Err(e)` immediately with a new trace frame added

### Automatic context injection

Every `?` automatically captures and attaches:
- Fully qualified function name (e.g., `myapp.config.processConfig`)
- Source file path (e.g., `src/config.aria`)
- Line number
- Column number
- Module path

This context is appended as a new `TraceFrame` on the error's trace chain. The capture is zero-cost on the happy path — the frame is only allocated when an error actually occurs.

```
fn loadApp() -> App ! AppError {
    config := processConfig()?    // on error: adds frame "loadApp config.aria:2:14"
    db := connectDb(config.db)?   // on error: adds frame "loadApp config.aria:3:10"
    App{config, db}
}
```

The resulting trace, if `processConfig` itself used `?`, would contain:
1. The frame where the original error was created (inside `readFile`)
2. The frame from `processConfig`'s `?`
3. The frame from `loadApp`'s `?`

This gives a complete call chain without any manual wrapping.

### Type compatibility rules

The `?` operator is valid only when the error type is compatible with the enclosing function's error signature:

- If the enclosing function declares `-> T ! E`, then `?` is valid on any `Result[_, E]`
- If the enclosing function declares `-> T ! A | B | C`, then `?` is valid on `Result[_, A]`, `Result[_, B]`, `Result[_, C]`, `Result[_, A | B]`, etc.
- If the error type is not in the enclosing function's signature, it is a **compile error**

Compiler error example:
```
fn process() -> Result[str, ParseError] {
    content := readFile("x")?   // compile error: IoError not in signature
    // fix: add ! IoError to the signature, or handle the error locally
}
```

Suggested fix in the compiler error:
```
error: `?` propagates `IoError` but function only declares `! ParseError`
  --> src/process.aria:2:30
  |
2 |     content := readFile("x")?
  |                              ^ cannot propagate IoError here
  |
  = help: add `IoError` to the error signature:
           fn process() -> str ! ParseError | IoError
```

### Automatic error type widening

When a function declares a named aggregate error type, the `?` operator automatically widens narrower error types:

```
type AppError = IoError | ParseError | DbError

fn initialize() -> App ! AppError {
    content := readFile("config.json")?   // IoError widens to AppError
    config := parseConfig(content)?        // ParseError widens to AppError
    db := connectDb(config.db)?            // DbError widens to AppError
    App{config, db}
}
```

This widening requires an `impl From[IoError] for AppError`, which is **auto-generated** by the compiler for union types. See section 8 for the full `From` system.

### `?` in different contexts

**In functions:** Propagates to the caller, adding a trace frame.

**In `entry {}`:** The program's entry point. A `?` in `entry` converts the error to a formatted message printed to stderr, then exits with a non-zero exit code.

```
entry {
    config := loadConfig("app.json")?   // on error: print trace and exit(1)
    serve(config)?
}
```

**In closures:** Propagates out of the closure. The closure's return type must include the error type:

```
// The closure captures the error and the caller of map handles it
result := items.map(fn(item) -> Result[Item, ParseError] {
    parsed := parse(item)?   // valid — closure returns Result[Item, ParseError]
    parsed
})
```

**In `spawn` blocks:** `?` **cannot** be used in `spawn` blocks directly. Errors in spawned tasks must be communicated via the task result or channels. See section 9.

### `?` with transformation

For cases where automatic widening is insufficient and you need to add domain-specific context:

```
// Basic form: ? |e| transform_expr
content := readFile(configPath) ? |e| AppError.ConfigLoad{source: e, path: configPath}
```

The `? |e| expr` syntax means:
1. Evaluate the left-hand side
2. If `Ok(v)`, evaluate to `v`
3. If `Err(e)`, bind `e` to the error, evaluate `expr`, and return `Err(expr)` immediately

This is useful for:
- Adding domain-specific context that isn't captured by the type alone
- Converting between error hierarchies where `From` isn't sufficient
- Enriching errors at important architectural boundaries

```
fn loadUserProfile(userId: u64) -> UserProfile ! AppError {
    row := db.query("SELECT * FROM users WHERE id = ?", userId)
             ? |e| AppError.Database{source: e, context: "loading user {userId}"}
    parseProfile(row)
             ? |e| AppError.DataCorruption{source: e, userId: userId}
}
```

### Summary

| Form | Meaning | Token cost |
|---|---|---|
| `expr?` | Propagate, add trace frame | 1 |
| `expr ? \|e\| transform` | Propagate with transformation | 5–10 |

---

## 5. Error Handling Patterns

Aria provides a complete vocabulary for error handling. Each pattern has a specific use case and optimal context.

### Pattern 1: Propagate (`?`)

The default. When the caller should handle the error, propagate it up with `?`.

```
fn loadConfig(path: str) -> Config ! IoError | ParseError {
    content := readFile(path)?
    parseConfig(content)?
}
```

**When to use:** Any time the current function isn't the right place to handle the error. In well-structured code, most functions use only `?`.

### Pattern 2: Inline handling (`catch`)

Handle an error inline and provide a fallback value. The `yield` keyword provides the fallback:

```
content := readFile("config.json") catch |err| {
    log.warn("config not found, using defaults: {err}")
    yield "{}"
}
```

The `yield` keyword is required in `catch` blocks — it provides the value that the `catch` expression evaluates to on the error path. The type of `yield expr` must be identical to the success type of the left-hand expression.

`catch` blocks are expressions. Their type is the success type `T` of the original `Result[T, E]` expression.

```
// catch is an expression — can be used anywhere
port := parsePort(input) catch |_| { yield 8080 }
```

**When to use:** When there is a meaningful fallback value and logging or other side effects are appropriate.

### Pattern 3: Typed catch arms

Handle different error variants differently in one expression:

```
result := fetchUser(id) catch {
    UserError.NotFound{..} => defaultUser
    UserError.Timeout{..} => {
        log.warn("timeout fetching user {id}, retrying")
        retry(3, fn() => fetchUser(id)) catch {
            _ => defaultUser
        }
    }
    e => return Err(AppError.User{source: e})
}
```

Typed catch arms follow the same syntax as `match` arms. The patterns must cover all variants of the error type (exhaustiveness is checked). A catch arm without `yield` must either return from the enclosing function, panic, or produce a value of the correct type directly.

**When to use:** When different error variants require different recovery strategies.

### Pattern 4: Assert (`!`)

Panics with a full error trace if the result is `Err`. Use only when the error is genuinely impossible or represents a logic error:

```
config := loadConfig("app.json")!   // panics if Err, with full trace
```

This is intentionally terse — it should be used sparingly, only at program startup or in contexts where failure is truly unrecoverable. The `!` produces a panic with the error's `Display` value and the full error trace.

**When to use:** Boot-time assertions, test setup, cases where the error contract has been broken and continuation is meaningless.

### Pattern 5: Assert with message (`must`)

Like `!` but requires a message string, producing richer panic output:

```
config := loadConfig("app.json") must "app config required for startup"
db := Database.connect(config.dbUrl) must "database required for startup"
```

Panic output format for `must`:
```
PANIC: required value was an error
  message: "app config required for startup"
  error:   IoError.NotFound{path: "app.json"}
  at:      myapp.main (main.aria:5:12)

Error trace:
  myapp.main          main.aria:5
  myapp.loadConfig    config.aria:12
  io.readFile         io.aria:87
```

**When to use:** `must` is preferred over `!` in `entry {}` and startup code — the message documents why the value is required.

### Pattern 6: Full match

Use `match` when you need the full power of pattern matching, including access to the success value and the error value in the same expression:

```
match readFile("config.json") {
    Ok(content) => {
        config := parseConfig(content)?
        startApp(config)
    }
    Err(IoError.NotFound{path}) => {
        log.info("no config at {path}, using defaults")
        startApp(defaultConfig())
    }
    Err(e) => return Err(e)
}
```

**When to use:** When the success and error paths both require non-trivial logic, or when you need to destructure the success value alongside error handling.

### Pattern 7: `or` for default values

The `or` operator provides a simple inline default for the error case without a callback:

```
config := readFile("config.json") or "{}"
port := parsePort(input) or 8080
name := getEnvVar("APP_NAME") or "aria-app"
```

`or` is syntactic sugar for `catch { _ => yield value }`. The value must be the same type as the success type. `or` does not provide access to the error value — use `catch` if you need the error.

**When to use:** The single most common recovery pattern — a simple default value when the operation fails.

### Pattern 8: `retry` built-in

For operations that may fail transiently, `retry` handles the retry loop:

```
result := retry(attempts: 3, delay: 1s, backoff: .Exponential) {
    connectToDatabase(url)?
}
```

Full signature:
```
fn retry[T, E: Transient](
    attempts: u64,
    delay: dur = 0s,
    backoff: BackoffStrategy = .None,
    op: fn() -> T ! E
) -> T ! E
```

`BackoffStrategy` variants:
- `.None` — no delay between retries
- `.Fixed` — constant delay
- `.Linear` — delay increases linearly: `delay * attempt`
- `.Exponential` — delay doubles each attempt: `delay * 2^attempt`
- `.Custom(fn(attempt: u64, last_delay: dur) -> dur)` — custom strategy

**Key behavior:** `retry` only retries if the error implements the `Transient` trait. If the error is `Permanent`, `retry` fails immediately on the first error without further attempts. This is enforced by the generic bound `E: Transient` — the compiler rejects `retry` if the error type cannot be `Transient`.

**When to use:** Network operations, database connections, any I/O with transient failure modes.

---

## 6. Error Categories — Semantic Traits

Error category traits encode the semantic meaning of an error into the type system. This is the mechanism by which Aria enables mechanical generation of correct retry, recovery, and reporting logic.

### Trait definitions

```
// A transient error — the same operation might succeed if tried again
trait Transient {}

// A permanent error — retrying will not help
trait Permanent {}

// A user-fault error — the caller's input or action caused the failure
trait UserFault {}

// A system-fault error — the environment caused the failure (disk, network, OOM)
trait SystemFault {}

// A retryable error with scheduling hints
trait Retryable: Transient {
    // How long to wait before retrying. None means retry immediately.
    fn retryAfter(self) -> dur?

    // Maximum number of retries recommended. Default 3.
    fn maxRetries(self) -> u64 = 3
}
```

### Applying category traits

Category traits are implemented like any other trait:

```
impl Transient for IoError.Timeout {}
impl SystemFault for IoError.Timeout {}
impl Retryable for IoError.Timeout {
    fn retryAfter(self) -> dur? = Some(self.after * 2)
    fn maxRetries(self) -> u64 = 5
}

impl Permanent for IoError.NotFound {}
impl UserFault for IoError.NotFound {}

impl Permanent for IoError.PermissionDenied {}
impl UserFault for IoError.PermissionDenied {}
```

Or via `derives` on variants for the common cases:

```
type IoError =
    | NotFound { path: str }
        derives [Permanent, UserFault]
    | PermissionDenied { path: str, user: str }
        derives [Permanent, UserFault]
    | Timeout { after: dur }
        derives [Transient, SystemFault]
    | ConnectionRefused { addr: str, port: u16 }
        derives [Transient, SystemFault]
```

### Mutual exclusivity rules

The compiler enforces:
- `Transient` and `Permanent` are **mutually exclusive** — implementing both on the same type is a compile error
- `UserFault` and `SystemFault` are **mutually exclusive** — implementing both on the same type is a compile error
- `Retryable` implies `Transient` — implementing `Retryable` without `Transient` is a compile error

```
// compile error: Timeout cannot be both Transient and Permanent
impl Transient for IoError.Timeout {}
impl Permanent for IoError.Timeout {}
// error: conflicting trait implementations: Transient and Permanent are mutually exclusive
```

### Pattern matching on categories

Error categories can be used in `match` guards:

```
match err {
    e if e is Retryable => {
        after := e.retryAfter() or 1s
        scheduleRetry(op, after: after, max: e.maxRetries())
    }
    e if e is Transient => retry(3, op)
    e if e is UserFault => return Err(ValidationError.fromCause(e))
    e if e is SystemFault => {
        alertOps(e)
        return Err(InternalError.fromCause(e))
    }
}
```

This pattern is especially powerful in middleware and service frameworks — a single error handler can correctly dispatch all error types by category without knowing the specific error variants.

### The `retry` built-in and category traits

The `retry` built-in (section 5, Pattern 8) uses `Transient` and `Retryable` automatically:
- If the error implements `Retryable`, `retry` uses `retryAfter()` and `maxRetries()` as hints
- If the error implements `Transient` but not `Retryable`, `retry` uses the provided `delay` and `attempts`
- If the error implements `Permanent`, `retry` returns the error immediately without retrying

### AI rationale

When generating code that calls a fallible function, the AI can inspect the error type's category traits to determine the correct recovery strategy without any documentation reading:

- `Retryable` → generate `retry()` call with the error's hints
- `Transient` → generate `retry()` call or `catch` with a retry comment
- `Permanent + UserFault` → generate input validation or 400-style response
- `Permanent + SystemFault` → generate alerting and 500-style response

---

## 7. Error Traces — Structured Debugging

Aria error traces are distinct from stack traces. A stack trace captures the full call stack at the moment of an exception — it is collected eagerly and is expensive. An error trace captures only the propagation chain — the sequence of `?` operations — and is built lazily on the error path only.

### Core types

```
type ErrorTrace[E] {
    error: E                    // the original typed error value
    frames: [TraceFrame]        // propagation chain, most recent first
    timestamp: time.Instant     // when the error was first created
}

type TraceFrame {
    function: str               // fully qualified function name
    file: str                   // source file path
    line: u64                   // line number
    column: u64                 // column number
    module: str                 // module path
}
```

`ErrorTrace[E]` is the actual type returned by fallible functions. When you write `fn f() -> str ! IoError`, the actual return type at the call boundary is `Result[str, ErrorTrace[IoError]]`. The `!` syntax is sugar — the compiler inserts `ErrorTrace` wrapping automatically.

### How traces are built

1. When a fallible function creates and returns `Err(e)`, an `ErrorTrace` is created:
   - `error` is set to `e`
   - `frames` is initialized with the first frame (the site of `Err(e)`)
   - `timestamp` is set to the current monotonic clock value

2. Each subsequent `?` propagation prepends a new `TraceFrame` to `frames`

3. The trace is only allocated when an error actually occurs — on the happy path, there is zero overhead

### Accessing traces in handling code

```
match doOperation() {
    Ok(v) => use(v)
    Err(traced) => {
        println("Error: {traced.error}")
        println("When: {traced.timestamp}")
        println("Origin: {traced.frames.last()?.function}")

        for frame in traced.frames {
            println("  at {frame.function} ({frame.file}:{frame.line}:{frame.column})")
        }
    }
}
```

When using `catch` or other handling patterns, the `err` binding is an `ErrorTrace[E]`, so you have access to both the typed error and the trace:

```
content := readFile(path) catch |traced| {
    log.error("read failed: {traced.error} (origin: {traced.frames.last()?.file})")
    yield ""
}
```

### Trace formatting

`ErrorTrace[E]` derives `Display` with a standard format:

```
Error: IoError.NotFound{path: "/etc/app/config.json"}
Trace (most recent first):
  at myapp.loadConfig       src/config.aria:34:18
  at myapp.initialize       src/main.aria:12:10
  at myapp.main             src/main.aria:5:5
```

For structured logging and error reporting, traces serialize to JSON:

```
// ErrorTrace is JSON-serializable when E: Json
logged := json.encode(traced)?
log.error(logged)
```

JSON output format:
```json
{
  "error": { "variant": "NotFound", "path": "/etc/app/config.json" },
  "timestamp": "2026-03-18T22:30:29Z",
  "frames": [
    { "function": "myapp.loadConfig", "file": "src/config.aria", "line": 34, "column": 18 },
    { "function": "myapp.initialize", "file": "src/main.aria", "line": 12, "column": 10 }
  ]
}
```

### Accessing the raw error value

Convenience accessors on `ErrorTrace[E]`:

```
traced.error           // the typed error value E
traced.frames          // [TraceFrame]
traced.timestamp       // time.Instant
traced.origin()        // TraceFrame? — the first (innermost) frame
traced.site()          // TraceFrame? — the last (outermost) frame where ? was used
```

### Compile-time trace control

```
aria build                          // full traces: file, line, column, function, module
aria build --release                // function names only; no file/line (smaller binary)
aria build --release --no-traces    // no trace collection at all; ? still propagates,
                                    // but frames list is always empty
```

`--no-traces` is for embedded or extremely size-constrained environments. In all other cases, full traces are available and their cost is proportional only to error frequency (zero on the happy path).

### AI rationale

Stack traces are unstructured strings that the AI must parse. Error traces in Aria are typed, structured data with machine-readable fields. When debugging generated code, the AI can programmatically inspect `traced.frames` to identify where an error originated — no string parsing required.

---

## 8. Error Composition Across Module Boundaries

Well-structured programs have clear module boundaries. Each module owns its error types and converts at the boundary — it does not leak implementation errors into its public API. Aria's `From` trait and auto-generation system make this seamless.

### The `From` trait

```
trait From[T] {
    fn from(value: T) -> Self
}
```

`From[T]` converts from `T` into `Self`. When `impl From[IoError] for AppError` exists, any `IoError` can be automatically converted to an `AppError`. The `?` operator uses `From` automatically for widening.

### Auto-generated `From` for error unions

When a function declares `! AppError` where `AppError = IoError | ParseError | DbError`, the compiler auto-generates:

```
// Auto-generated by compiler for: type AppError = IoError | ParseError | DbError
impl From[IoError] for AppError {
    fn from(e: IoError) -> AppError = AppError from e
}
impl From[ParseError] for AppError {
    fn from(e: ParseError) -> AppError = AppError from e
}
impl From[DbError] for AppError {
    fn from(e: DbError) -> AppError = AppError from e
}
```

These implementations allow `?` to automatically widen any of the component types into `AppError` at each call site. The auto-generation is transparent — you never write these implementations manually unless you need custom behavior.

### Manual `From` for rich conversions

When automatic widening is insufficient (e.g., you need to map internal errors to API-stable errors), implement `From` manually:

```
impl From[SqlError] for AppError {
    fn from(e: SqlError) -> AppError = match e {
        SqlError.ConnectionLost{..} =>
            AppError.Database{source: e, message: "database unavailable"}
        SqlError.SyntaxError{query, ..} =>
            AppError.Internal{msg: "bad generated SQL: {query}", source: e}
        SqlError.Deadlock{..} =>
            AppError.Database{source: e, message: "deadlock detected, retry later"}
        _ =>
            AppError.Database{source: e, message: e.to_str()}
    }
}
```

Once this `impl` exists, `?` on a `Result[T, SqlError]` inside a function returning `! AppError` uses this conversion automatically.

### Module-level error aggregation pattern

The recommended pattern for application-level error handling is a single `AppError` type in a dedicated errors module:

```
// module: myapp.errors

type AppError =
    | Config { source: config.ConfigError, path: str }
    | Database { source: db.DbError, message: str }
    | Network { source: net.NetError, url: str }
    | Validation { field: str, message: str }
    | Internal { msg: str, source: dyn Debug? }
    | NotFound { resource: str, id: str }
    | Unauthorized { reason: str }
    | RateLimit { retry_after: dur }
```

All other modules convert their errors into `AppError` at the boundary. This pattern:
- Keeps the public API stable (internal error types can change without changing callers)
- Produces a single, exhaustive error type for the whole application
- Allows the top-level handler to write one comprehensive `match` statement

### Error boundary pattern

Every public-facing module function should convert implementation errors to module-owned error types at the boundary:

```
// Internal function — may expose SqlError
fn queryUserInternal(id: u64) -> User ! SqlError { ... }

// Public function — converts to module error type
fn getUser(id: u64) -> User ! UserError {
    queryUserInternal(id) ? |e| match e {
        SqlError.NotFound => UserError.NotFound{id: id}
        _ => UserError.DatabaseFailure{source: e}
    }
}
```

This ensures that callers of the `user` module never see `SqlError` — they see only `UserError`. If the database implementation changes, only `queryUserInternal` and the conversion in `getUser` need updating.

### Composing errors from multiple modules

When a function calls into multiple modules:

```
fn processOrder(orderId: u64) -> Order ! OrderError {
    user := users.getUser(orderId.userId)?     // UserError widens to OrderError via From
    product := inventory.getProduct(orderId.productId)?  // InventoryError widens to OrderError via From
    payment := payments.charge(user, orderId.amount)?    // PaymentError widens to OrderError via From
    Order{user, product, payment}
}
```

As long as `impl From[UserError] for OrderError`, `impl From[InventoryError] for OrderError`, and `impl From[PaymentError] for OrderError` exist (auto-generated or manual), the `?` operator handles all the widening transparently.

---

## 9. Errors and Concurrency

Aria's concurrency model is structured: `spawn` creates tasks, `scope` groups them with a lifetime, and channels carry typed values. Errors participate naturally in all three.

### Errors in spawned tasks

A spawned task has a typed result accessible via `.result`:

```
task := spawn fetchData(url)
// task: Task[Result[Data, FetchError]]
// task.result blocks until the task completes

match task.result {
    Ok(data) => use(data)
    Err(e) => handleFetchError(e)
}
```

Tasks cannot use `?` to propagate errors out of the spawn boundary — the error type system enforces this. Instead, the error is captured in the `Task`'s result type and accessed by the spawner.

### Errors in structured concurrency (`scope`)

`scope` groups multiple spawned tasks with a shared lifetime. All tasks are guaranteed to complete before the scope exits:

```
scope {
    a := spawn fetchUsers()     // Task[Result[Users, ApiError]]
    b := spawn fetchOrders()    // Task[Result[Orders, DbError]]
}
// After scope: both a and b are complete

users := a.result?     // propagate ApiError if failed
orders := b.result?    // propagate DbError if failed
```

**On first failure — structured cancellation (default):**

When one task in a `scope` fails (returns `Err`), Aria's default behavior is:
1. Signal all sibling tasks to cancel (cooperative cancellation via cancellation tokens)
2. Wait for all sibling tasks to complete or acknowledge cancellation
3. Propagate the first error to the scope owner

This prevents resource leaks from abandoned background tasks and ensures deterministic cleanup.

```
scope {
    a := spawn longNetworkCall()   // starts
    b := spawn anotherCall()       // starts
    // if a fails with Err:
    //   b receives cancellation signal
    //   b completes or acknowledges cancellation
    //   scope exits with a's error
}
```

**Collect-all variant (`scope.collectAll`):**

When you need all results regardless of individual failures:

```
results := scope.collectAll {
    spawn fetchUser(1)
    spawn fetchUser(2)
    spawn fetchUser(3)
}
// results: [Result[User, ApiError]] — all three complete, no cancellation

(successes, failures) := results.partitionResults()
if failures.len() > 0 {
    log.warn("{failures.len()} user fetches failed")
}
```

### Errors over channels

Channels carry typed values. To communicate errors, send `Result` values:

```
ch := chan[Result[Data, ProcessError]](buffer: 10)

spawn {
    for item in source {
        result := process(item)
        ch.send(result)
    }
    ch.close()
}

for result in ch {
    match result {
        Ok(data) => accumulate(data)
        Err(e) => log.warn("processing failed: {e}")
    }
}
```

`?` works naturally on `Result` values received from channels:

```
data := ch.recv()?    // propagates ProcessError if the channel sends an Err
```

### Error aggregation from parallel operations

For collecting all errors from a parallel operation:

```
type AggregateError {
    errors: [ErrorTrace[dyn Debug]]
    total_tasks: u64
    failed_tasks: u64
}

fn processAllItems(items: [Item]) -> [ProcessedItem] ! AggregateError {
    results := scope.collectAll {
        for item in items {
            spawn processItem(item)
        }
    }

    (successes, failures) := results.partitionResults()

    if failures.len() > 0 {
        return Err(AggregateError{
            errors: failures,
            total_tasks: items.len(),
            failed_tasks: failures.len(),
        })
    }

    successes
}
```

### `select` and errors

`select` waits on multiple channels or task completions and handles errors per arm:

```
select {
    result from taskCh => match result {
        Ok(v) => process(v)
        Err(e) => return Err(AppError.fromTaskError(e))
    }
    result from fallbackCh => result?
    after 5s => return Err(AppError.Timeout{after: 5s})
}
```

### Task panic isolation

If a spawned task panics (not an error — a panic, see section 12), the panic is:
- Isolated to that task — it does not propagate to the spawner immediately
- Captured as a special `PanicError` in the task's result
- Propagated to the scope owner when the scope exits

```
scope {
    a := spawn mightPanic()
}
// If a panicked: scope exits with PanicError
// The spawner can inspect: a.panicked() -> bool
```

---

## 10. Errors and the FFI Boundary

When calling C functions, error conventions differ from Aria's typed errors. The FFI layer maps C error conventions to typed Aria errors automatically. See [spec/ffi-design.md](ffi-design.md) for the full FFI specification.

### C return code mapping

C functions that return `-1` on error with `errno` set can declare the mapping:

```
extern "C" fn open(path: *const c_char, flags: c_int) -> c_int
    maps_error { -1 => errno_to_io_error() }
```

With this declaration, calling `open` from Aria returns `Result[c_int, IoError]` instead of a raw `c_int`. The mapping function is called only when the return value matches the error sentinel.

### Null pointer mapping

C functions that return null pointers on failure:

```
extern "C" fn fopen(path: *const c_char, mode: *const c_char) -> *mut FILE
    maps_null => IoError.NotFound{path: ""}
```

The Aria-side signature becomes `fn fopen(...) -> *mut FILE ! IoError`. The null check is automatic.

### `errno` capture

The `errno_to_io_error()` helper function:

```
fn errno_to_io_error() -> IoError {
    match c.errno() {
        c.ENOENT  => IoError.NotFound{path: ""}
        c.EACCES  => IoError.PermissionDenied{path: "", user: ""}
        c.ETIMEDOUT => IoError.Timeout{after: 0s}
        c.ECONNREFUSED => IoError.ConnectionRefused{addr: "", port: 0}
        c.EEXIST  => IoError.AlreadyExists{path: ""}
        c.EISDIR  => IoError.IsDirectory{path: ""}
        c.ENOSPC  => IoError.StorageFull{}
        code      => IoError.Unknown{code: code as i64}
    }
}
```

Custom mappings can be defined for library-specific error codes.

### Memory allocation failure

```
extern "C" fn malloc(size: usize) -> *mut void
    maps_null => AllocationError.OutOfMemory{requested: size}
```

### Safety invariant: C errors never leak as untyped values

The FFI layer guarantees that no C error convention reaches Aria code as an untyped value. Every `maps_error` and `maps_null` declaration transforms the C error into a typed Aria error before the call returns to Aria code. The AI never writes `if result == -1 { ... }` in Aria — that pattern doesn't exist.

### Error type wrapping at FFI boundaries

Well-designed FFI wrappers convert C errors to library-specific Aria error types:

```
// Low-level C binding
extern "C" fn sqlite3_step(stmt: *mut sqlite3_stmt) -> c_int
    maps_error { code if code != SQLITE_ROW && code != SQLITE_DONE => sqlite_code_to_error(code) }

// High-level Aria wrapper  
fn step(stmt: Statement) -> Row? ! DbError {
    code := sqlite3_step(stmt.handle)?    // SqliteError widens to DbError via From
    if code == SQLITE_DONE { None } else { Some(Row{stmt}) }
}
```

---

## 11. Errors and Iteration

Fallible operations appear frequently in iterator chains. Aria provides first-class support for errors in iterators.

### Fallible iterators

An iterator whose `next()` operation can fail:

```
trait FallibleIterator[T, E] {
    fn next(mut self) -> Result[T?, E]
    // Ok(Some(v)) — next value
    // Ok(None)    — iteration complete
    // Err(e)      — iteration failed
}
```

File lines, network streams, and database cursors are common examples of fallible iterators.

### Collecting Results

**Fail on first error** — `sequence()`:

```
// [Result[T, E]] -> Result[[T], E]
// Collects all Ok values; returns on first Err
processed := items
    .map(fn(x) => process(x))    // Iterator[Result[ProcessedItem, ProcessError]]
    .sequence()?                  // Result[[ProcessedItem], ProcessError]
```

**Collect both successes and failures** — `partitionResults()`:

```
// [Result[T, E]] -> ([T], [E])
(successes, failures) := items
    .map(fn(x) => process(x))
    .partitionResults()

log.info("processed {successes.len()} items, {failures.len()} failures")
```

### Skip-on-error pattern

When failures should be silently skipped:

```
valid_items := items
    .iter()
    .filterMap(fn(x) => process(x).ok())   // drops Err, keeps Ok values
    .collect()
```

`.ok()` on a `Result[T, E]` returns `T?` — `Some(v)` if `Ok(v)`, `None` if `Err(_)`. The error is discarded.

### Collect-errors pattern

Process all items, accumulate errors, continue processing:

```
(results, errors) := items
    .iter()
    .map(fn(x) => process(x))
    .partitionResults()

if errors.len() > 0 {
    log.warn("had {errors.len()} failures out of {items.len()} items:")
    for e in errors { log.warn("  - {e}") }
}
use(results)
```

### Fallible `for` loops

In a `for` loop over a fallible iterator, `?` propagates the iteration error:

```
fn processLines(path: str) -> [Line] ! IoError | ParseError {
    results := []
    for line in readLines(path)? {    // readLines returns FallibleIterator[str, IoError]
        parsed := parseLine(line)?
        results.push(parsed)
    }
    results
}
```

The `for` loop over a `FallibleIterator` automatically propagates any iteration errors. To continue on error, use `.filterMap()` or explicit result handling inside the loop body.

### `try_collect` for transforming fallible iterators

```
// Equivalent to .sequence() — more explicit syntax
result := items.iter().map(fn(x) => validate(x)).tryCollect()?
```

---

## 12. The `panic` Mechanism

Panics are categorically different from errors. Errors are typed values representing expected failure modes. Panics represent invariant violations — situations that should never occur in correct code.

### When to panic

Panics occur for:
- **Logic errors**: Index out of bounds, arithmetic overflow (debug mode), null dereference (impossible in safe Aria, but possible via FFI)
- **Invariant violations**: `assert` failures, broken invariants that the type system didn't catch
- **`!` on an `Err` value**: Unwrapping a `Result` that turned out to be `Err`
- **`must` on an `Err` value**: Same as `!` but with a user-provided message
- **Explicit `panic(msg)`**: For cases where the developer knows an invariant has been broken

### Panic is NOT for recoverable errors

This distinction is fundamental:

| | Error | Panic |
|---|---|---|
| Represents | Expected failure mode | Invariant violation / logic error |
| Declared in | Function signature (`!`) | Not declared |
| Caught by | `catch`, `match`, `?` | Cannot be caught in normal code |
| Propagates | Via `?` through the call chain | Terminates the current task |
| Has | Typed value + trace | Stack trace only |
| In concurrent code | Captured in task result | Isolated to the panicking task |

### `assert` and `debug_assert`

```
assert condition                           // always checked; panics if false
assert condition, "message"               // with message
debug_assert condition                    // checked in debug builds only
debug_assert condition, "x must be positive"
```

`debug_assert` is for expensive invariant checks that are too costly for production:

```
fn binarySearch(arr: [i64], target: i64) -> i64? {
    debug_assert arr.isSorted(), "binarySearch requires sorted input"
    // ... O(n) sort check only in debug builds
}
```

### Panic output format

```
PANIC: assertion failed: x > 0
  message: "x must be positive"
  at: mymodule.processData (process.aria:47:5)

Stack trace:
  mymodule.processData      process.aria:47
  mymodule.handleRequest    handler.aria:23
  net.serve.handler         net.aria:156
  runtime.taskEntry         runtime.aria:12
```

For `must`:
```
PANIC: required value was an error
  message: "database required for startup"
  error:   DbError.ConnectionRefused{addr: "localhost", port: 5432}
  at: myapp.main (main.aria:8:12)

Error trace:
  myapp.main            main.aria:8
  db.connect            db.aria:34
  net.tcpConnect        net.aria:201
```

### Panic isolation in concurrent code

In a `scope`, a panicking task:
1. Has its panic captured internally
2. Signals sibling tasks to cancel
3. Once all siblings complete, the panic is re-raised in the scope owner's context

This prevents panics from silently orphaning background tasks while still providing isolation between tasks.

### No panic recovery in normal code

Unlike Rust's `std::panic::catch_unwind` or Java's `catch (Throwable)`, Aria provides no mechanism to catch panics in normal application code. Panics are terminal for the task that experiences them. This design choice makes panics a reliable signal that a genuine programming error occurred.

The only exception is at the runtime level: the runtime may catch panics from spawned tasks to support the isolation model above and provide structured shutdown.

---

## 13. Testing Error Conditions

Aria's test framework provides first-class support for testing error paths.

### `test` blocks with error assertions

```
test readFile_notFound {
    result := readFile("/nonexistent/path")
    assert result.isErr()

    err := result.unwrapErr()
    assert err.error is IoError.NotFound
    assert err.error.path == "/nonexistent/path"
}

test processConfig_propagatesIoError {
    result := processConfig("/bad/path")
    assert result.isErr()

    trace := result.unwrapErr()
    assert trace.error is IoError.NotFound
    assert trace.frames.len() >= 2
    assert trace.frames[0].function.contains("processConfig")
}
```

`unwrapErr()` returns the `ErrorTrace[E]` inside an `Err`, or panics if the result is `Ok`. This is the test-code equivalent of `!` for the error path.

### `assertOk` and `assertErr`

```
test database_connect_success {
    assertOk(db.connect(validConnectionString))
}

test database_connect_badUrl {
    assertErr[DbError.InvalidUrl](db.connect("not-a-url"))
}

test database_connect_refused {
    assertErr[DbError.ConnectionRefused](db.connect(unreachableUrl))
}
```

`assertOk(expr)` — asserts that `expr` is `Ok`; panics with the error value if `Err`.
`assertErr[E](expr)` — asserts that `expr` is `Err` with an error matching type `E`; panics with the success value if `Ok`.

### Testing error traces

```
test error_trace_has_correct_frames {
    trace := processConfig("/bad")
        .unwrapErr()

    // Check the trace chain
    assert trace.frames.len() >= 1
    assert trace.frames[0].function == "aria.io.readFile"
    assert trace.frames[1].function contains "processConfig"
}
```

### Testing error categories

```
test ioTimeout_isTransient {
    err := IoError.Timeout{after: 5s}
    assert err is Transient
    assert err is SystemFault
    assert !(err is Permanent)
    assert !(err is UserFault)
}

test ioNotFound_isPermanent {
    err := IoError.NotFound{path: "/missing"}
    assert err is Permanent
    assert err is UserFault
    assert !(err is Transient)
}

test retryable_hints {
    err := IoError.Timeout{after: 2s}
    assert err is Retryable
    delay := (err as Retryable).retryAfter() or 0s
    assert delay == 4s    // retryAfter returns after * 2
}
```

### Testing error composition

```
test from_conversion {
    io_err := IoError.NotFound{path: "/config"}
    app_err := AppError from io_err    // explicit From conversion
    assert app_err is AppError.Config
}

test propagation_widens_error {
    result := processRequest(badRequest)
    trace := result.unwrapErr()
    // Even though the inner function returned IoError,
    // processRequest returns AppError
    assert trace.error is AppError
}
```

### Mocking for error injection

Error injection in tests uses the same mock/stub mechanism as the rest of the test framework:

```
test retry_on_transient_error {
    call_count := 0
    mock io.readFile with fn(path: str) -> str ! IoError {
        call_count += 1
        if call_count < 3 {
            return Err(IoError.Timeout{after: 100ms})
        }
        Ok("file content")
    }

    result := retry(attempts: 3, delay: 10ms) { readFile("test.txt")? }
    assertOk(result)
    assert call_count == 3
}
```

---

## 14. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Errors are typed sum types | Pattern matching, exhaustive handling, no string parsing, full compiler support |
| `!` in function signatures | Full error contract visible in the signature — AI reads no implementation |
| `?` with automatic context | 1 token per propagation, context is structural and free, no forgotten wrapping |
| Error traces, not stack traces | Lazy allocation (zero cost on happy path), structured data (AI-parseable), proportional overhead |
| `catch` with typed arms | Different errors require different recovery — one expression handles all variants |
| `or` for simple defaults | Minimal tokens for the most common recovery pattern |
| `must` with message | Explicit startup assertions with documentation built in |
| `retry` built-in | Transient error recovery as a single expression, no boilerplate, uses category traits |
| Category traits (`Transient`, `Permanent`, etc.) | Mechanical generation of correct retry/recovery logic — no guessing, no docs needed |
| `From` auto-generation | Error widening is transparent and correct — no manual conversion boilerplate |
| Error unions are flat | `(A \| B) \| (B \| C)` = `A \| B \| C` — no nesting, no surprises |
| Panic ≠ Error | Clear separation of recoverable vs. unrecoverable — panic is always a logic error |
| Scope cancellation on failure | No leaked tasks on error, deterministic cleanup, correct by default |
| `sequence()` for Result collections | Common pattern has a name — one token instead of a fold |
| No implicit conversions | Every type boundary is explicit — the AI never generates a wrong widening |
| FFI errors are typed at the boundary | C error conventions never leak into Aria — no `if result == -1` pattern |
| `partitionResults()` | Collect-both pattern has a name — enables partial success handling |

---

## 15. Comparison with Other Languages

### Error handling feature matrix

| Feature | Go | Rust (thiserror) | Rust (anyhow) | Java | Swift | Kotlin | Aria |
|---|---|---|---|---|---|---|---|
| **Error declaration** | Interface (`error`) | Enum + `#[derive(Error)]` | Opaque (`anyhow::Error`) | Class hierarchy | Protocol (`Error`) | Class/sealed class | Sum type |
| **Typed errors** | ❌ (string interface) | ✅ | ❌ (erased) | ⚠️ (class hierarchy) | ✅ | ⚠️ | ✅ |
| **Propagation syntax** | `if err != nil { return ..., err }` | `?` | `?` | `throw` / `throws` | `try` / `throws` | (unchecked) | `?` |
| **Propagation tokens** | ~10–15 | 1 | 1 | ~5–10 | 3–5 | N/A | **1** |
| **Automatic context** | ❌ | ❌ | ✅ (`.context()` manual) | ❌ | ❌ | ❌ | **✅ (free)** |
| **Exhaustive handling** | ❌ | ✅ | ❌ | ⚠️ (checked only) | ✅ | ❌ | **✅** |
| **Error categories in types** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| **Built-in retry** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| **Structured error traces** | ❌ | ❌ | ❌ | ⚠️ (stack trace strings) | ⚠️ | ⚠️ | **✅ (structured)** |
| **Concurrency error handling** | Manual (channels) | Manual | Manual | Exceptions in tasks | Manual | Coroutine exceptions | **Structured (scope)** |
| **FFI error mapping** | Manual | Manual | Manual | JNI conventions | Manual | Manual | **Declarative (`maps_error`)** |
| **Error composition** | Manual wrapping | `#[from]` + `?` | Automatic (erased) | Inheritance | Manual | Manual | **Auto (`From` generation)** |

### Detailed comparison: propagation verbosity

**Go** — A function calling 3 fallible operations with context:
```go
func processConfig(path string) (*Config, error) {
    content, err := readFile(path)
    if err != nil {
        return nil, fmt.Errorf("processConfig: reading file: %w", err)
    }
    parsed, err := parseJSON(content)
    if err != nil {
        return nil, fmt.Errorf("processConfig: parsing JSON: %w", err)
    }
    config, err := validateConfig(parsed)
    if err != nil {
        return nil, fmt.Errorf("processConfig: validating: %w", err)
    }
    return config, nil
}
```
~12 lines, ~30 tokens for error handling, context is manual strings.

**Rust (thiserror)** — Same function:
```rust
fn process_config(path: &str) -> Result<Config, AppError> {
    let content = read_file(path)?;
    let parsed = parse_json(&content)?;
    let config = validate_config(parsed)?;
    Ok(config)
}
```
~5 lines, 3 tokens for error handling, no automatic context.

**Java** — Same function:
```java
Config processConfig(String path) throws IOException, ParseException, ValidationException {
    String content = readFile(path);     // throws IOException
    Object parsed = parseJSON(content);  // throws ParseException
    return validateConfig(parsed);       // throws ValidationException
}
```
~5 lines, untyped union in `throws`, no context, non-exhaustive callers.

**Aria** — Same function:
```
fn processConfig(path: str) -> Config ! IoError | ParseError | ValidationError {
    content := readFile(path)?
    parsed := parseJson(content)?
    validateConfig(parsed)?
}
```
~4 lines, 3 tokens for error handling, **automatic context at each `?`**.

### Error declaration verbosity

| Language | Declaring a 4-variant error type with messages |
|---|---|
| Go | 4 constant values + 4 struct types + `Error() string` method each |
| Rust (thiserror) | 1 enum + `#[derive(Error)]` + `#[error("...")]` per variant |
| Java | 1 class per variant + constructors + `getMessage()` per variant |
| Swift | 1 enum conforming to `LocalizedError` + `errorDescription` per variant |
| Aria | `type E = \| V1{} display "..." \| V2{} display "..."` |

### Context preservation

| Language | How context is added | Who adds it | Risk |
|---|---|---|---|
| Go | `fmt.Errorf("ctx: %w", err)` | Developer manually | Forgotten at every call site |
| Rust (thiserror) | Not provided | Developer manually writes `context()` | Forgotten, or not done at all |
| Rust (anyhow) | `.context("message")` | Developer manually | Forgotten |
| Java | `new WrapperException("ctx", cause)` | Developer manually | Often lost |
| Aria | Automatic at every `?` | **Compiler** | **Cannot be forgotten** |

### Concurrency interaction

| Language | Error in background task | Behavior |
|---|---|---|
| Go | `goroutine` panics or returns | Unchecked unless channels are used; goroutine leaks possible |
| Rust | `thread::spawn` result is `JoinHandle<Result<T, E>>` | Manual join required; no structured lifetimes |
| Java | `Future.get()` throws `ExecutionException` | Manual, untyped wrapping |
| Kotlin | Coroutine exception propagation | Unstructured by default; `supervisorScope` for isolation |
| Aria | `scope` with typed task results | **Structured**: sibling tasks cancelled on first failure; typed results |

---

*This specification is part of the Aria language design documentation. For related specifications, see [high-level-design.md](../high-level-design.md), [spec/stdlib-design.md](stdlib-design.md), [spec/ffi-design.md](ffi-design.md), and [spec/package-module-system.md](package-module-system.md).*
