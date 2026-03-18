# Aria Standard Library Design

## Design Philosophy: Batteries-Included, But Decomposable

The standard library is where the most tokens are spent in any language. The stdlib design directly controls:

1. **How many tokens are burned on imports** (Go: `import "net/http"` × 8 packages; Aria should be less)
2. **How many tokens are burned on API calls** (Go: `json.NewDecoder(r.Body).Decode(&v)` — verbose)
3. **How many bugs are created** (APIs with footguns vs. APIs with pit-of-success design)
4. **How much boilerplate wraps every operation** (error wrapping, resource cleanup, type conversion)

### The Core Principle: One Canonical Way, Zero Ceremony

Every stdlib module follows this rule: **the most common operation should be the shortest expression.** If 90% of the time you read a whole file into a string, that should be 1 function call, not 4 (open, read, close, convert).

---

## Module Tiers

### Tier 0: Built Into the Language (No Import Needed)

These are so fundamental they don't require an import statement. Every token spent on `import` for basic operations is waste.

| Symbol | What It Does |
|---|---|
| `print`, `println` | Output to stdout |
| `[T]` | List/slice type + methods (map, filter, fold, sort, etc.) |
| `Map[K, V]` | Hash map |
| `Set[T]` | Hash set |
| `Option[T]` | `Some(T) \| None`, with `?` sugar |
| `Result[T, E]` | `Ok(T) \| Err(E)`, with `!` sugar |
| `str` | String type + methods (split, trim, contains, replace, etc.) |
| `Range` | `0..10`, `0..=10`, iterable |
| `Tuple` | `(A, B, C)` types |
| `chan[T]` | Channels |
| `spawn`, `scope` | Concurrency primitives |

**Why no import**: `map`, `filter`, `println`, string operations, and `Option/Result` are used in virtually every single function. Requiring imports for these is pure token waste. Go gets this partially right (`append`, `len`, `make` are builtins) but not far enough.

---

### Tier 1: Core Modules (One-Word Imports)

These cover the things needed in 80%+ of real-world programs. Single-word imports: `use io`, not `import "io/ioutil"`.

---

#### `io` — I/O Primitives

```
use io

// Read a whole file — the 90% case, 1 call
content := io.readFile("config.json")?

// Write a whole file — the 90% case, 1 call
io.writeFile("output.txt", data)?

// Streaming — when you need it
reader := io.open("huge.csv")?
defer reader.close()
for line in reader.lines() {
    process(line)
}

// Stdin/stdout/stderr are globals
line := io.stdin.readLine()?
io.stderr.write("warning: something happened")
```

**Design rationale**: In Go, reading a file is `os.Open` → `defer f.Close()` → `io.ReadAll` → `string(bytes)`. That's 4 lines and 3 imports. In Aria, it's `io.readFile(path)?` — 1 line, 1 import. The streaming API exists for the 10% case, but the common path is trivially short.

---

#### `net` — Networking

```
use net

// HTTP client — the common case
resp := net.get("https://api.example.com/users")?
users := resp.json[[]User]()?  // generic deserialization

// With options
resp := net.fetch("https://api.example.com/users", {
    method: .POST
    headers: {"Authorization": "Bearer {token}"}
    body: payload.toJson()
    timeout: 10s
})?

// HTTP server
server := net.serve(":8080")
server.get("/users", fn(req) -> net.Response {
    users := db.getUsers()?
    net.Response.json(users)
})
server.run()?

// TCP/UDP — lower level, same module
conn := net.dial("tcp", "localhost:5432")?
conn.write(data)?
```

**Design rationale**: HTTP is the most commonly generated networking code. `net.get(url)?` should just work and return a response that can be deserialized in one chained call. No `http.NewRequest` → `client.Do` → `defer resp.Body.Close()` → `io.ReadAll` → `json.Unmarshal` dance. That Go pattern is 6 lines; Aria does it in 1-2.

The `{method: .POST}` syntax uses Aria's struct literals with enum shorthand (`.POST` infers `Method.POST` from context). Zero extra tokens for the type name.

---

#### `json` — Serialization

```
use json

// Serialize — if the type derives Json, this just works
output := json.encode(user)?

// Deserialize — generic, type-inferred
user := json.decode[User](data)?

// Pretty print
output := json.encodePretty(user)?

// Stream parsing (for large documents)
parser := json.stream(reader)
for token in parser {
    match token {
        Object{key, value} => ...
        Array{items} => ...
    }
}
```

**Design rationale**: `derives [Json]` on the type + `json.encode/decode` with generics = done. The type system and derives do the heavy lifting. No struct tags to get wrong, no separate marshal/unmarshal methods to write.

---

#### `db` — Database

```
use db

// Connect
conn := db.connect("postgres://localhost/mydb")?
defer conn.close()

// Query with type-safe results
users := conn.query[User]("SELECT * FROM users WHERE age > {minAge}")?

// Single row
user := conn.queryOne[User]("SELECT * FROM users WHERE id = {id}")?

// Execute (inserts, updates, deletes)
count := conn.exec("DELETE FROM sessions WHERE expired_at < {now}")?

// Transactions
conn.tx(fn(tx) {
    tx.exec("UPDATE accounts SET balance = balance - {amount} WHERE id = {from}")?
    tx.exec("UPDATE accounts SET balance = balance + {amount} WHERE id = {to}")?
})?
```

**Design rationale**: Database access is in 60%+ of generated services. Aria's `db` module is driver-agnostic — the connection string determines the backend (postgres, sqlite, mysql). Query interpolation is **parameterized** (not string concatenation — no SQL injection), and results deserialize via generics + `Json`/`Db` derives.

---

#### `time` — Time & Duration

```
use time

now := time.now()
today := time.today()

// Duration literals are built into the language
timeout := 30s
interval := 5m
ttl := 24h

// Arithmetic
expires := now + 24h
elapsed := time.since(start)

// Formatting
formatted := now.format("2006-01-02")  // or named: now.format(.ISO8601)

// Parsing
t := time.parse("2026-03-18", "2006-01-02")?

// Sleep
time.sleep(100ms)

// Timers & tickers
ticker := time.every(1s)
for tick in ticker {
    heartbeat()
}
```

**Design rationale**: Duration literals (`30s`, `5m`) are in the language spec — no import needed for the type itself. The `time` module adds formatting, parsing, arithmetic, and timers.

---

#### `math` — Mathematics

```
use math

// Constants
pi, e, tau, inf, nan

// Standard functions — all generic over Numeric
abs, min, max, clamp
sqrt, cbrt, pow, log, log2, log10
sin, cos, tan, asin, acos, atan, atan2
floor, ceil, round, trunc

// Random
r := math.random()          // f64 in [0, 1)
n := math.randInt(1, 100)   // i64 in [1, 100]
item := math.choice(list)   // random element
shuffled := math.shuffle(list)
```

---

#### `crypto` — Cryptography

```
use crypto

hash := crypto.sha256(data)
hash := crypto.blake3(data)

// Secure random
token := crypto.randomBytes(32)
uuid := crypto.uuid()

// Hashing passwords
hashed := crypto.hashPassword(password)?
valid := crypto.verifyPassword(password, hashed)

// Symmetric encryption
encrypted := crypto.encrypt(plaintext, key)?
decrypted := crypto.decrypt(ciphertext, key)?

// TLS is handled transparently by `net`
```

---

#### `os` — Operating System Interface

```
use os

// Environment
val := os.env("DATABASE_URL") ?? "localhost"
os.setEnv("MODE", "production")

// Args
args := os.args()  // [str]

// Process
os.exit(1)
pid := os.pid()

// Filesystem (beyond simple read/write, which is in `io`)
entries := os.listDir(".")?
os.mkdir("output")?
os.remove("temp.txt")?
exists := os.exists("config.json")
info := os.stat("file.txt")?

// Exec
result := os.exec("git", ["status"])?
output := result.stdout
```

---

#### `log` — Logging

```
use log

log.info("server started on port {port}")
log.warn("connection slow: {elapsed}")
log.error("failed to connect: {err}")
log.debug("query: {sql}")

// Structured
log.info("request handled", {
    method: req.method
    path: req.path
    status: resp.status
    duration: elapsed
})

// Configuration
log.setLevel(.Debug)
log.setOutput(file)
log.setFormat(.Json)  // structured JSON logging
```

---

#### `sync` — Synchronization

```
use sync

// Mutex
mu := sync.Mutex.new()
mu.lock(fn {
    // critical section — lock auto-released when block exits
    sharedState.update()
})

// Atomic
counter := sync.Atomic[i64].new(0)
counter.add(1)
val := counter.load()

// WaitGroup (rarely needed with structured concurrency, but available)
wg := sync.WaitGroup.new()
wg.add(1)
spawn {
    defer wg.done()
    work()
}
wg.wait()

// Once
init := sync.Once.new()
init.do(fn { expensiveSetup() })
```

**Design note**: With structured concurrency (`scope`), most use cases for `WaitGroup` disappear. And with the lock-via-closure pattern for `Mutex`, you can't forget to unlock. These designs make it **structurally impossible** to generate incorrect synchronization code.

---

#### `test` — Testing Utilities

```go
// `test` and `assert` are language builtins, but `test` module adds:
use test

test "user validation" {
    user := User{name: "", email: "bad"}

    // Table-driven (built into the language)
    cases := [
        (input: "", expected: false)
        (input: "alice", expected: true)
        (input: "a", expected: false)
    ]

    for c in cases {
        assert validate(c.input) == c.expected
    }

    // Snapshot testing
    test.snapshot("user_json", user.toJson())

    // Benchmarking
    bench "serialize user" {
        json.encode(user)
    }
}
```

---

### Tier 2: Extended Modules (Common but Not Universal)

These ship with Aria but cover more specialized use cases.

| Module | Purpose |
|---|---|
| `regex` | Regular expressions |
| `csv` | CSV parsing/writing |
| `xml` | XML parsing/writing |
| `html` | HTML parsing/templating |
| `compress` | gzip, zlib, zstd, snappy |
| `encoding` | base64, hex, url-encoding |
| `flag` | CLI argument parsing |
| `path` | File path manipulation |
| `signal` | OS signal handling |
| `embed` | Embed files into binary at compile time |

---

## Design Rules Across All Modules

### Rule 1: The Common Case Is One Line

```go
// ✅ Good: 90% case is trivial
content := io.readFile("data.txt")?
parsed := json.decode[Config](content)?

// ❌ Bad: forcing ceremony on the common case
file := io.open("data.txt", io.ReadOnly)?
defer file.close()
reader := io.buffered(file)
bytes := reader.readAll()?
content := str.fromBytes(bytes)
```

The verbose path exists for the 10% case. It's never the default.

### Rule 2: Resource Cleanup Is Structural

```go
// Mutex: closure-based, can't forget to unlock
mu.lock(fn { ... })

// Files: defer exists but open-read-close has a one-liner
content := io.readFile(path)?

// Transactions: closure-based, auto-rollback on error
conn.tx(fn(tx) { ... })?

// Scoped concurrency: can't leak goroutines
scope { spawn work() }
```

Closure-based resource management makes forgetting cleanup impossible.

### Rule 3: Zero Conversions

```go
// In Go, constant type conversion boilerplate:
// string(bytes), []byte(str), strconv.Itoa(n), strconv.Atoi(s), fmt.Sprintf("%d", n)

// In Aria: methods on the types
n.toStr()       // i64 -> str
s.toInt()?      // str -> i64 (fallible)
bytes.toStr()   // [byte] -> str
s.toBytes()     // str -> [byte]

// Or just use interpolation
msg := "count: {n}"  // n auto-converts in interpolation
```

### Rule 4: No Import Aliasing Hell

```go
// Go:
import (
    "encoding/json"
    "net/http"
    "database/sql"
    "fmt"
    "os"
    "strconv"
    "time"
    "sync"
)

// Aria:
use io, json, net, db, time, sync
```

One line. Short names. No paths. No aliases.

---

## Why This Design Maximizes AI Code Generation

| Problem in Other Languages | How Aria's Stdlib Solves It |
|---|---|
| Tokens burned on import blocks | `use` is 1 line, short names |
| Multi-line ceremony for simple ops | One-liners for common cases |
| Forgetting resource cleanup | Closure-based APIs make it structural |
| Messing up type conversions | Methods on types, interpolation handles the rest |
| Inconsistent error handling | `?` propagation works uniformly across all modules |
| Producing incorrect concurrent code | Structured concurrency, closure-based locks |
| Writing redundant serialization code | `derives [Json]` + `json.encode/decode` |
| Generating too many test boilerplate lines | Inline `test` + `assert` + `bench` |

### Net Token Savings Estimate
For a typical web service (HTTP API, database, JSON, logging):

| Component | Go Tokens | Aria Tokens | Savings |
|---|---|---|---|
| Imports | ~80 | ~15 | 81% |
| Error handling | ~200 | ~50 | 75% |
| HTTP handler setup | ~120 | ~40 | 67% |
| Database queries | ~150 | ~50 | 67% |
| JSON serialization | ~100 | ~10 | 90% |
| Tests | ~150 | ~50 | 67% |
| **Total** | **~800** | **~215** | **~73%** |

This means for every program built, roughly **1/4 the tokens** are generated for the same functionality. That translates to faster responses, more code per interaction, and more room in the context window for complex logic.