# Aria Testing Framework Specification

**A specification for test blocks, assertions, mocking, debug facilities, property-based testing, and structured test output — designed for AI-driven development.**

This document formalizes Aria's testing framework. Every feature in this spec was designed to answer one question: what does an AI need to write correct tests, debug failures, and verify generated code? The answer is: precise assertions, dependency mocking, structured output, and a debug facility that shows exactly what went wrong.

Cross-references:
- [spec/error-handling.md](error-handling.md) — `assertOk`, `assertErr`, test blocks with error handling
- [spec/effect-system.md](effect-system.md) — test blocks implicitly have all effects
- [spec/trait-system.md](trait-system.md) — `Eq`, `Debug`, `Display` traits used in assertions
- [high-level-design.md](../high-level-design.md) — inline test syntax overview

---

## Table of Contents

1. [Design Philosophy](#1-design-philosophy)
2. [Test Blocks](#2-test-blocks)
3. [Assertions](#3-assertions)
4. [The `dbg` Function](#4-the-dbg-function)
5. [Mocking](#5-mocking)
6. [Test Discovery and Execution](#6-test-discovery-and-execution)
7. [Structured Test Output](#7-structured-test-output)
8. [Property-Based Testing](#8-property-based-testing)
9. [Benchmark Blocks](#9-benchmark-blocks)
10. [Test Fixtures](#10-test-fixtures)
11. [Design Rationale Summary](#11-design-rationale-summary)

---

## 1. Design Philosophy

Testing is where AI code generation quality is measured. When I generate a function, I also generate tests. When those tests fail, I read the failure output and fix the code. This cycle — generate, test, read output, fix — is my primary workflow. Every design decision in this spec optimizes that cycle.

### What I need from a test framework

1. **Tests next to the code** — no separate test files, no test discovery configuration
2. **Precise failure messages** — not "assertion failed" but "expected 42, got 41 at fibonacci.aria:15"
3. **Mocking** — I can't test a function that calls a database without mocking the database
4. **Structured output** — I need to parse test results programmatically, not scrape human-readable text
5. **Debug printing** — when a test fails, I need to see intermediate values without adding permanent log statements

**AI rationale:** Every extra step between "test fails" and "I understand why" costs tokens. Precise assertions, structured output, and `dbg()` minimize the number of rounds I need to diagnose and fix a failure. In Go, I often spend 3-4 rounds just understanding what `t.Errorf` is telling me. Aria's test framework should make one round sufficient.

---

## 2. Test Blocks

Tests are declared inline with the code they test, using `test` blocks:

```
fn fibonacci(n: u64) -> u64 = match n {
    0 => 0
    1 => 1
    n => fibonacci(n - 1) + fibonacci(n - 2)
}

test "fibonacci base cases" {
    assert fibonacci(0) == 0
    assert fibonacci(1) == 1
}

test "fibonacci sequence" {
    assert fibonacci(10) == 55
    assert fibonacci(20) == 6765
}
```

### Test block rules

- Test blocks are top-level declarations (same level as `fn`, `type`, `impl`)
- Test names can be identifiers or string literals: `test myTest { }` or `test "my test" { }`
- Test blocks implicitly have **all effects** — they can do I/O, spawn tasks, etc.
- Test blocks are excluded from release builds — they exist only when running `aria test`
- Each test block runs in isolation — one test's failure does not affect others

### Test block grammar

```
test_block = "test" ( IDENT | STRING_LIT ) block ;
```

---

## 3. Assertions

### `assert`

The basic assertion. If the expression is `false`, the test fails with a message showing the expression, its value, and the source location:

```
assert fibonacci(10) == 55
```

Failure output:
```
FAIL: assertion failed
  expression: fibonacci(10) == 55
  left:  55
  right: 55
  at:    src/math.aria:15:5
```

When the assertion involves a comparison (`==`, `!=`, `<`, `>`, `<=`, `>=`), both sides are shown separately with their values.

### `assert` with message

```
assert count > 0, "count must be positive, got {count}"
```

Failure output:
```
FAIL: count must be positive, got -3
  expression: count > 0
  at:    src/validate.aria:8:5
```

### `assertOk`

Asserts that a `Result` is `Ok`. On failure, shows the error:

```
assertOk(db.connect(validUrl))
```

Failure output:
```
FAIL: expected Ok, got Err
  error: DbError.ConnectionRefused{addr: "localhost", port: 5432}
  at:    src/db_test.aria:5:5
```

### `assertErr[E]`

Asserts that a `Result` is `Err` with a specific error variant:

```
assertErr[DbError.InvalidUrl](db.connect("not-a-url"))
```

Failure output (if `Ok`):
```
FAIL: expected Err(DbError.InvalidUrl), got Ok
  value: Connection{...}
  at:    src/db_test.aria:10:5
```

### `assertEqual`

Shorthand for `assert a == b` with better failure messages:

```
assertEqual(fibonacci(10), 55)
```

Failure output:
```
FAIL: values are not equal
  expected: 55
  actual:   54
  at:       src/math.aria:15:5
```

### `assertNear`

For floating-point comparisons with epsilon:

```
assertNear(calculatePi(), 3.14159, epsilon: 1e-5)
```

Failure output:
```
FAIL: values are not approximately equal
  expected: 3.14159 (±0.00001)
  actual:   3.15
  at:       src/math.aria:20:5
```

---

## 4. The `dbg` Function

`dbg` is a debug print function that outputs the expression text, its value, and the source location. It returns the value unchanged, so it can be inserted into any expression without changing behavior.

```
fn process(items: [i64]) -> i64 {
    total := dbg(items.map(fn(x) => x * 2).sum())
    // Prints: [src/process.aria:2] items.map(fn(x) => x * 2).sum() = 42
    total
}
```

### Output format

```
[file:line] expression = value
```

### Usage patterns

```
// Debug a variable
dbg(x)
// [src/main.aria:5] x = 42

// Debug a complex expression (returns the value — can be chained)
result := dbg(items |> filter(fn(x) => x > 0) |> sum())
// [src/main.aria:8] items |> filter(fn(x) => x > 0) |> sum() = 150

// Debug inside a pipeline (non-destructive — returns the value)
result := items
    |> dbg                        // prints the list
    |> filter(fn(x) => x > 0)
    |> dbg                        // prints the filtered list
    |> sum()

// Debug in match arms
match shape {
    Circle(r) => dbg(pi * r * r)
    Rect(w, h) => dbg(w * h)
    Point => 0.0
}
```

### `dbg` has the `Io` effect

`dbg` writes to stderr (not stdout). Since it performs I/O, it has the `Io` effect. However, in `test` blocks (which have all effects implicitly), `dbg` can be used freely.

In non-test code, using `dbg` in a pure function is a compile error — this prevents accidentally leaving debug prints in production code:

```
fn calculate(x: i64) -> i64 {
    dbg(x * 2)    // ❌ compile error: dbg has Io effect, but calculate is pure
                   // help: add `with [Io]` or remove the dbg call
}
```

**AI rationale:** `dbg` is the single most important debugging tool for me. When a test fails, I insert `dbg` at key points, re-run, and read the output. Unlike `println`, `dbg` shows the expression text — I don't need to write `println("items = {items}")`, I just write `dbg(items)`. And because `dbg` is a compile error in pure functions, I can't accidentally leave it in production code.

---

## 5. Mocking

Mocking replaces a function's implementation within a test block. This is essential for testing code that depends on I/O, databases, or external services.

### Basic mock syntax

```
test "readFile returns content" {
    mock io.readFile with fn(path: str) -> str ! IoError {
        match path {
            "config.json" => Ok("{\"port\": 8080}")
            _ => Err(IoError.NotFound{path: path})
        }
    }

    content := io.readFile("config.json")?
    assert content == "{\"port\": 8080}"
}
```

### Mock scope

Mocks are scoped to the test block they appear in. When the test block exits, the original implementation is restored. Mocks do not affect other tests.

### Mocking module functions

```
test "handles database errors" {
    mock db.connect with fn(url: str) -> Connection ! DbError {
        Err(DbError.ConnectionRefused{addr: "localhost", port: 5432})
    }

    result := initializeApp()
    assertErr[AppError.Database](result)
}
```

### Mock with call counting

```
test "retries on transient error" {
    mut call_count := 0
    mock net.get with fn(url: str) -> str ! HttpError {
        call_count += 1
        if call_count < 3 {
            Err(HttpError.Timeout{after: 100ms})
        } else {
            Ok("response body")
        }
    }

    result := retry(attempts: 3, delay: 10ms) { net.get("https://api.example.com")? }
    assertOk(result)
    assert call_count == 3
}
```

### Mock with argument capture

```
test "sends correct request" {
    mut captured_url := ""
    mock net.post with fn(url: str, body: str) -> str ! HttpError {
        captured_url = url
        Ok("{\"status\": \"ok\"}")
    }

    sendNotification(user)?
    assert captured_url == "https://api.example.com/notify"
}
```

### Grammar

```
mock_stmt = "mock" path_expr "with" expression ;
```

**AI rationale:** Mocking is the single biggest testing gap in Go. To test a function that calls `http.Get`, I must restructure the code to accept an interface, create a mock struct, implement the interface methods — 30+ lines of boilerplate before writing the actual test. Aria's `mock` statement replaces an implementation in one line. I can test any function that calls I/O without restructuring the code under test.

---

## 6. Test Discovery and Execution

### Running tests

```
aria test                          # run all tests
aria test "fibonacci"              # run tests matching "fibonacci"
aria test --file src/math.aria     # run tests in a specific file
aria test --verbose                # show all test names and results
aria test --fail-fast              # stop on first failure
aria test --parallel=4             # run tests in parallel (default: CPU count)
aria test --format=json            # structured output (see section 7)
```

### Test discovery

All `test` blocks in the compilation unit are discovered automatically. No registration, no test runner configuration, no naming conventions beyond the `test` keyword.

### Test ordering

Tests run in **undefined order** by default. Tests must not depend on execution order. The `--parallel` flag enables concurrent test execution (default behavior — tests are parallelized unless `--parallel=1`).

### Test isolation

Each test block runs in its own context:
- Mocks are scoped to the test block
- Global state modifications do not leak between tests
- Panics in one test do not affect others

---

## 7. Structured Test Output

### Default output (human-readable)

```
aria test

  ✓ fibonacci base cases (0.1ms)
  ✓ fibonacci sequence (0.2ms)
  ✗ database connection (1.5ms)
    FAIL: expected Ok, got Err
      error: DbError.ConnectionRefused{addr: "localhost", port: 5432}
      at:    src/db.aria:15:5

  3 tests: 2 passed, 1 failed (1.8ms)
```

### JSON output (machine-readable)

```
aria test --format=json
```

```json
{
  "tests": [
    {
      "name": "fibonacci base cases",
      "file": "src/math.aria",
      "line": 10,
      "status": "passed",
      "duration_ms": 0.1
    },
    {
      "name": "fibonacci sequence",
      "file": "src/math.aria",
      "line": 15,
      "status": "passed",
      "duration_ms": 0.2
    },
    {
      "name": "database connection",
      "file": "src/db.aria",
      "line": 5,
      "status": "failed",
      "duration_ms": 1.5,
      "failure": {
        "message": "expected Ok, got Err",
        "error": "DbError.ConnectionRefused{addr: \"localhost\", port: 5432}",
        "file": "src/db.aria",
        "line": 15,
        "column": 5
      }
    }
  ],
  "summary": {
    "total": 3,
    "passed": 2,
    "failed": 1,
    "duration_ms": 1.8
  }
}
```

**AI rationale:** JSON output is how I parse test results. When I run `aria test --format=json`, I can programmatically determine which tests failed, what the failure was, and where it occurred — then generate a targeted fix. Human-readable output requires me to parse prose, which is error-prone and costs tokens.

---

## 8. Property-Based Testing

Property-based testing generates random inputs and verifies that a property holds for all of them.

### `forAll`

```
test "addition is commutative" {
    forAll(fn(a: i64, b: i64) {
        assert a + b == b + a
    })
}

test "sort produces ordered output" {
    forAll(fn(items: [i64]) {
        sorted := items.sort()
        for i in 0..sorted.len() - 1 {
            assert sorted[i] <= sorted[i + 1]
        }
    })
}

test "serialize then deserialize is identity" {
    forAll(fn(user: User) {
        assertEqual(deserialize(serialize(user)), user)
    })
}
```

### How `forAll` works

1. `forAll` inspects the parameter types and generates random values
2. By default, it runs 100 iterations (configurable: `forAll(iterations: 1000, fn(...) { ... })`)
3. On failure, it **shrinks** the input to find the minimal failing case
4. The minimal failing case is reported in the test output

### Shrinking

When a property fails, `forAll` attempts to find a simpler input that still fails:

```
test "list reversal" {
    forAll(fn(items: [i64]) {
        assert items.reverse().reverse() == items
    })
}

// If this fails for [5, -2, 99, 0, 3], shrinking might reduce to [5, -2]
// The failure report shows the minimal case
```

### Supported types for generation

`forAll` can generate random values for:
- All primitive types (`i64`, `f64`, `str`, `bool`, etc.)
- Collections (`[T]`, `Map[K,V]`, `Set[T]`) where element types are generatable
- Sum types (randomly selects a variant)
- Structs where all fields are generatable
- Custom types via the `Arbitrary` trait

### Custom generators

```
trait Arbitrary {
    fn arbitrary(rng: mut ref Random) -> Self
    fn shrink(self) -> [Self] = []    // default: no shrinking
}

impl Arbitrary for Email {
    fn arbitrary(rng: mut ref Random) -> Email {
        user := rng.alphanumeric(1..20)
        domain := rng.alphanumeric(1..10)
        Email("{user}@{domain}.com")
    }
}
```

**AI rationale:** Property-based testing lets me express **what should always be true** rather than listing specific examples. This catches edge cases I wouldn't think to test. When I generate a sorting function, `forAll(fn(items) { assert isSorted(sort(items)) })` tests it on hundreds of random inputs — far more thorough than `assert sort([3,1,2]) == [1,2,3]`.

---

## 9. Benchmark Blocks

Benchmark blocks measure the performance of code:

```
bench "fibonacci 30" {
    fibonacci(30)
}

bench "sort 10000 items" {
    items := randomList(10000)
    items.sort()
}
```

### Running benchmarks

```
aria bench                     # run all benchmarks
aria bench "fibonacci"         # run matching benchmarks
aria bench --format=json       # structured output
```

### Benchmark output

```
aria bench

  fibonacci 30          42ns/op    (1000000 iterations)
  sort 10000 items      1.2ms/op  (1000 iterations)
```

The framework automatically determines the number of iterations to get stable timing.

---

## 10. Test Fixtures

For tests that need shared setup and teardown:

### `before` and `after` blocks

```
mut db: Database? = None

before {
    db = Some(Database.connect("test://localhost")?))
}

after {
    db?.disconnect()
}

test "insert user" {
    db!.exec("INSERT INTO users (name) VALUES ('Alice')")?
    users := db!.query("SELECT * FROM users")?
    assert users.len() == 1
}

test "delete user" {
    db!.exec("INSERT INTO users (name) VALUES ('Bob')")?
    db!.exec("DELETE FROM users WHERE name = 'Bob'")?
    users := db!.query("SELECT * FROM users")?
    assert users.len() == 0
}
```

`before` runs before each test in the same module. `after` runs after each test. Both are scoped to the module — they don't affect tests in other modules.

---

## 11. Design Rationale Summary

| Decision | Rationale |
|---|---|
| Tests inline with code | No separate test files — tests live next to what they test |
| Rich assertion messages | Show expression, both values, and location — one round to diagnose |
| `dbg()` debug print | Shows expression + value + location; compile error in pure fns prevents leaking |
| `mock` statement | One-line dependency replacement — no interface restructuring needed |
| Structured JSON output | AI can parse results programmatically — no prose scraping |
| Property-based testing (`forAll`) | Tests properties over hundreds of random inputs — catches edge cases |
| Test isolation | Mocks scoped to test blocks, no ordering dependency, parallel by default |
| Benchmarks | `bench` blocks with automatic iteration calibration |
| Fixtures (`before`/`after`) | Shared setup without global state leaking |

---

*This specification is part of the Aria language design documentation. For related specifications, see [spec/error-handling.md](error-handling.md), [spec/effect-system.md](effect-system.md), and [high-level-design.md](../high-level-design.md).*
