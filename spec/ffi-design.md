# Aria FFI Design Specification

## Overview

Aria's Foreign Function Interface (FFI) enables calling C libraries from Aria code. The design prioritizes:

1. **Safety by default** — Unsafe C interactions are isolated behind safe wrappers
2. **Token efficiency** — Declarations use Aria syntax, never embedded C
3. **Memory correctness** — Ownership annotations (`@owned`, `@borrowed`, `@cffi`) eliminate use-after-free and leak bugs
4. **Performance** — No GC scanning of C memory, no pinning overhead, amortized thread pool for C calls
5. **Effect tracking** — The compiler knows which functions touch C, enabling safe reasoning about pure code
6. **Automation** — `aria bind` auto-generates bindings from C headers

For design discussion and rationale, see [FFI-Considerations.md](FFI-Considerations.md).

---

## C Type System

### The `c` Namespace

All C-compatible types live under the `c.` namespace. This prevents any confusion between Aria types and C types.

```c
// Integer types
c.char          // char (signed)
c.uchar         // unsigned char
c.short         // short
c.ushort        // unsigned short
c.int           // int
c.uint          // unsigned int
c.long          // long
c.ulong         // unsigned long
c.longlong      // long long
c.ulonglong     // unsigned long long

// Floating point
c.float         // float
c.double        // double

// Special types
c.size          // size_t
c.ssize         // ssize_t
c.ptr           // void*
c.bool          // _Bool
c.null          // NULL (typed as c.ptr)

// Fixed-width (mirrors C99 stdint.h)
c.i8            // int8_t
c.i16           // int16_t
c.i32           // int32_t
c.i64           // int64_t
c.u8            // uint8_t
c.u16           // uint16_t
c.u32           // uint32_t
c.u64           // uint64_t
```

### Pointer Types

```c
*c.char         // char*
**c.char        // char**
*c.int          // int*
*const c.char   // const char*
*c.void         // void* (alias for c.ptr)
```

### Opaque Types

For C types whose internal structure is hidden (e.g., `FILE*`, `sqlite3*`):

```c
type c.sqlite3 = @opaque
type c.FILE = @opaque

// These can only be used as pointers:
handle: *c.sqlite3    // ok
handle: c.sqlite3     // compile error: opaque types cannot be instantiated
```

### C Struct Mapping

```c
// C struct:
// struct Point {
//     double x;
//     double y;
// };

type c.Point = @cstruct {
    x: c.double
    y: c.double
}

// @cstruct guarantees C-compatible memory layout:
// - No reordering of fields
// - C-standard alignment and padding
// - No GC metadata embedded
```

### C Enum Mapping

```c
// C enum:
// enum LogLevel { DEBUG = 0, INFO = 1, WARN = 2, ERROR = 3 };

type c.LogLevel = @cenum(c.int) {
    DEBUG = 0
    INFO = 1
    WARN = 2
    ERROR = 3
}
```

### C Function Pointer Types

```c
// C: typedef int (*comparator)(const void*, const void*);
type c.Comparator = c.fn(*const c.void, *const c.void) -> c.int

// Usage in an extern declaration:
extern fn qsort(base: c.ptr, count: c.size, size: c.size, cmp: c.Comparator)
```

---

## Type Conversion

### Aria to C

| Aria Type | C Type | Conversion | Cost |
|---|---|---|---|
| `i8, i16, i32, i64` | `c.i8, c.i16, c.i32, c.i64` | `c.i32(val)` | Zero (reinterpret) |
| `u8, u16, u32, u64` | `c.u8, c.u16, c.u32, c.u64` | `c.u32(val)` | Zero (reinterpret) |
| `f32, f64` | `c.float, c.double` | `c.float(val)` | Zero (reinterpret) |
| `bool` | `c.bool` | `c.bool(val)` | Zero (reinterpret) |
| `str` | `*c.char` | `val.toCStr()` | Allocates (null-terminated copy) |
| `[byte]` | `c.ptr + c.size` | `val.ptr(), val.len()` | Zero (pointer to existing data) |
| `[T]` | `*c.T + c.size` | `val.ptr(), val.len()` | Zero (pointer to existing data) |

### C to Aria

| C Type | Aria Type | Conversion | Cost |
|---|---|---|---|
| `c.i32` etc. | `i32` etc. | `i32(cVal)` | Zero (reinterpret) |
| `c.float, c.double` | `f32, f64` | `f64(cVal)` | Zero (reinterpret) |
| `c.bool` | `bool` | `bool(cVal)` | Zero (reinterpret) |
| `*c.char` | `str` | `str.fromCStr(cVal)` | Allocates (copies into Aria-managed str) |
| `*c.char + c.size` | `str` | `str.fromCStrLen(cVal, len)` | Allocates (copies, no null scan) |
| `*c.T + c.size` | `[T]` | `[T].fromPtr(cVal, len)` | Allocates (copies into Aria slice) |
| `*c.T + c.size` | `[T]` | `[T].wrapPtr(cVal, len)` | Zero (borrows, must manage lifetime) |

### Conversion Rules

1. **Numeric conversions are always zero-cost.** They are compile-time casts with identical bit representations.
2. **String conversions always allocate.** C strings are null-terminated mutable `char*`; Aria strings are length-prefixed immutable UTF-8. The boundary always involves a copy.
3. **Slice-to-pointer is zero-cost.** `val.ptr()` returns a raw pointer to the slice's backing array. The data is not copied.
4. **Pointer-to-slice can be zero-cost or allocating.** `wrapPtr` borrows (zero-cost but lifetime-sensitive). `fromPtr` copies (safe but allocates).
5. **No implicit conversions.** Every boundary crossing is explicit.

---

## Declaring External Functions

### Inline Declaration (Single Functions)

```c
// Single extern function — one line
extern fn puts(s: *c.char) -> c.int

// With specific library linkage
extern "m" fn sqrt(x: c.double) -> c.double   // links against libm

// Void return
extern fn free(ptr: c.ptr)

// Variadic (rare, but needed for printf-family)
extern fn printf(fmt: *const c.char, ...) -> c.int
```

### Block Declaration (Libraries)

````c
mod bindings.sqlite

extern "sqlite3" {
    fn sqlite3_open(filename: *c.char, db: **c.sqlite3) -> c.int
    fn sqlite3_close(db: *c.sqlite3) -> c.int
    fn sqlite3_exec(
        db: *c.sqlite3,
        sql: *c.char,
        callback: c.fn(*c.void, c.int, **c.char, **c.char) -> c.int,
        arg: c.ptr,
        errmsg: **c.char,
    ) -> c.int
    fn sqlite3_errmsg(db: *c.sqlite3) -> *const c.char
    fn sqlite3_changes(db: *c.sqlite3) -> c.int
    fn sqlite3_last_insert_rowid(db: *c.sqlite3) -> c.longlong

    fn sqlite3_prepare_v2(
        db: *c.sqlite3,
        sql: *c.char,
        nByte: c.int,
        stmt: **c.sqlite3_stmt,
        tail: **c.char,
    ) -> c.int
    fn sqlite3_step(stmt: *c.sqlite3_stmt) -> c.int
    fn sqlite3_finalize(stmt: *c.sqlite3_stmt) -> c.int
    fn sqlite3_column_text(stmt: *c.sqlite3_stmt, col: c.int) -> *const c.char
    fn sqlite3_column_int(stmt: *c.sqlite3_stmt, col: c.int) -> c.int
    fn sqlite3_column_double(stmt: *c.sqlite3_stmt, col: c.int) -> c.double
}

// Opaque types used above
type c.sqlite3 = @opaque
type c.sqlite3_stmt = @opaque

// Constants
const SQLITE_OK: c.int = 0
const SQLITE_ROW: c.int = 100
const SQLITE_DONE: c.int = 101
````

### Linkage Specification

```c
// Dynamic linking (default)
extern "sqlite3" { ... }             // links against libsqlite3.so / .dylib / .dll

// Static linking
extern "sqlite3" @static { ... }    // links against libsqlite3.a

// System library (platform-specific search paths)
extern "pthread" @system { ... }    // links against system libpthread

// Framework (macOS)
extern "CoreFoundation" @framework { ... }
```

Build configuration in `aria.toml`:

```toml
[build.ffi]
lib_paths = ["/usr/local/lib", "./third_party/lib"]
include_paths = ["/usr/local/include"]
```

---

## Memory Ownership

### Ownership Annotations

Every C pointer in an Aria wrapper must declare its ownership:

```c
// @owned — Aria is responsible for cleanup.
// The Drop impl will be called when this value is collected or goes out of scope.
handle: @owned *c.sqlite3

// @borrowed — Someone else owns this. Do NOT free it.
// Used for pointers received in callbacks, temporary views, shared references.
ref: @borrowed *c.char

// @cffi — Allocated specifically for C interop.
// Lives in a non-moving memory region. GC-visible but never relocated.
// Freed when it goes out of scope or GC collects its wrapper.
buffer: @cffi [byte]
```

### The `Drop` Trait

Types that hold `@owned` C resources must implement `Drop`:

````c
pub type Database {
    handle: @owned *c.sqlite3
} derives [Drop]

impl Drop for Database {
    fn drop(self) {
        sqlite3_close(self.handle)
    }
}
````

**Drop guarantees:**
- `drop` is called **exactly once** when the value is no longer reachable
- `drop` is called **deterministically** when the value goes out of scope in `@stack` or `@arena` allocation modes
- `drop` is called **by the GC** when the value is collected in default (GC) allocation mode
- `drop` runs on the **finalizer thread**, not the GC thread — it does not block GC
- `drop` for `@owned` resources is called **before** the Aria wrapper struct is freed

### Compile-Time Ownership Checks

```c
// Error: @owned pointer without Drop implementation
pub type BadWrapper {
    handle: @owned *c.sqlite3
    // compile error: type has @owned field but does not implement Drop
}

// Error: returning @borrowed pointer beyond its scope
fn bad() -> @borrowed *c.char {
    cStr := someString.toCStr()
    return cStr
    // compile error: @borrowed pointer escapes its scope
}

// Error: double free potential
fn alsobad(db: Database) {
    sqlite3_close(db.handle)
    // compile error: manually closing @owned resource; Drop will handle this
}
```

### Non-Moving Memory: `@cffi` Allocations

When passing Aria-managed memory to C functions that may store the pointer:

```c
// Problem: GC might relocate this buffer while C holds a pointer to it
buffer := [byte].alloc(4096)
someCFunction(buffer.ptr())
// warning: passing GC-managed pointer to C; use @cffi

// Solution: @cffi allocation lives in non-moving region
buffer := @cffi [byte].alloc(4096)
someCFunction(buffer.ptr())    // safe: buffer will never be relocated
defer buffer.free()            // explicit free, or let GC collect it
```

**When to use `@cffi`:**
- Passing buffers to C functions that store the pointer for later use
- Shared memory between Aria and C over multiple calls
- Callback data pointers

**When NOT needed:**
- Passing buffers to C functions that only read/write during the call and don't store the pointer
- Numeric values and small structs (passed by value)

---

## Safe Wrappers

### Pattern: Wrapping a C Library

The standard pattern for exposing a C library to Aria code:

```c
// File: src/db/sqlite.aria
mod db

use bindings.sqlite

pub type Database {
    handle: @owned *c.sqlite3
} derives [Drop]

impl Drop for Database {
    fn drop(self) {
        sqlite.sqlite3_close(self.handle)
    }
}

pub fn open(path: str) -> Database ! DbError {
    handle: *c.sqlite3 = c.null
    cPath := path.toCStr()
    defer c.free(cPath)

    code := sqlite.sqlite3_open(cPath, &handle)
    if code != SQLITE_OK {
        msg := str.fromCStr(sqlite.sqlite3_errmsg(handle))
        return err(DbError{code: i32(code), message: msg})
    }
    Database{handle: handle}
}

pub fn (db: Database) exec(sql: str) -> i64 ! DbError {
    cSql := sql.toCStr()
    defer c.free(cSql)
    errMsg: *c.char = c.null

    code := sqlite.sqlite3_exec(db.handle, cSql, c.null, c.null, &errMsg)
    if code != SQLITE_OK {
        msg := str.fromCStr(errMsg)
        sqlite.free(errMsg)
        return err(DbError{code: i32(code), message: msg})
    }
    i64(sqlite.sqlite3_changes(db.handle))
}

pub fn (db: Database) query[T: FromRow](sql: str) -> [T] ! DbError {
    cSql := sql.toCStr()
    defer c.free(cSql)
    stmt: *c.sqlite3_stmt = c.null

    code := sqlite.sqlite3_prepare_v2(db.handle, cSql, c.int(-1), &stmt, c.null)
    if code != SQLITE_OK {
        return err(DbError.fromHandle(db.handle))
    }
    defer sqlite.sqlite3_finalize(stmt)

    results: [T] = []
    while sqlite.sqlite3_step(stmt) == SQLITE_ROW {
        results.push(T.fromRow(stmt)?)
    }
    results
}

pub type DbError {
    code: i32
    message: str
} derives [Eq, Debug]

fn DbError.fromHandle(handle: *c.sqlite3) -> DbError {
    DbError{
        code: i32(sqlite.sqlite3_errcode(handle))
        message: str.fromCStr(sqlite.sqlite3_errmsg(handle))
    }
}
```

### Pattern: Callback Wrapping

C callbacks require special handling — a C function pointer can't capture Aria closures directly:

```c
// C function that takes a callback:
// void iterate(void* data, int count, void (*callback)(void* ctx, int index))

extern fn iterate(data: c.ptr, count: c.int, cb: c.fn(c.ptr, c.int))

// Safe Aria wrapper
pub fn each(items: [Item], f: Item -> void) {
    ctx := @cffi CallbackContext{items: items, func: f}

    trampoline := fn @cconv (rawCtx: c.ptr, idx: c.int) {
        ctx := CallbackContext.fromPtr(rawCtx)
        ctx.func(ctx.items[u64(idx)])
    }

    iterate(ctx.ptr(), c.int(items.len()), trampoline)
}

type CallbackContext {
    items: [Item]
    func: Item -> void
}
```

**Key points:**
- `@cconv` marks a function as using the C calling convention
- The context struct is `@cffi` allocated (non-moving, so the C pointer remains valid)
- The trampoline function bridges between C's `void*` callback pattern and Aria's closures

---

## Effects System Integration

### Automatic Ffi Effect

All `extern` functions implicitly carry the `Ffi` effect:

```c
// This implicitly has: with [Ffi]
extern fn puts(s: *c.char) -> c.int
```

### Effect Propagation

```c
// This function calls an extern, so it inherits Ffi
fn rawPrint(msg: *c.char) with [Ffi] {
    puts(msg)
}

// This function wraps the unsafe call safely — effect is stripped
pub fn print(msg: str) {
    cStr := msg.toCStr()
    defer c.free(cStr)
    puts(cStr)
}
// The compiler allows stripping Ffi when:
// 1. All C memory is properly managed (@owned/@borrowed/@cffi)
// 2. No raw C pointers escape the function
// 3. All C calls are error-checked
```

### Why Effects Matter for FFI

```c
// The compiler can verify:
fn pureCalculation(x: f64) -> f64 = x * x + 2.0
// ^ No Ffi effect. This function CANNOT call C code.
//   It can be freely reordered, cached, parallelized.

fn loadData(path: str) -> [byte] ! IoError with [Ffi] {
    // ^ Has Ffi. The compiler knows this crosses the C boundary.
    //   Extra scrutiny applies.
}
```

---

## Binding Generator: `aria bind`

### Usage

```bash
# Generate bindings from a C header
aria bind sqlite3.h --lib sqlite3 --out bindings/sqlite.aria

# With include paths
aria bind openssl/ssl.h --lib ssl --include /usr/local/include --out bindings/ssl.aria

# Generate bindings for a whole directory of headers
aria bind include/ --lib mylib --out bindings/mylib.aria

# Preview without writing (dry run)
aria bind sqlite3.h --lib sqlite3 --dry-run
```

### What It Generates

Given a C header:

```c
// example.h
typedef struct {
    int x;
    int y;
} Point;

typedef enum {
    COLOR_RED = 0,
    COLOR_GREEN = 1,
    COLOR_BLUE = 2,
} Color;

Point* create_point(int x, int y);
void free_point(Point* p);
double distance(const Point* a, const Point* b);
void set_color(Point* p, Color c);
```

`aria bind` produces:

```c
// AUTO-GENERATED by aria bind — do not edit
// Source: example.h
// Library: example

mod bindings.example

extern "example" {
    fn create_point(x: c.int, y: c.int) -> *c.Point
    fn free_point(p: *c.Point)
    fn distance(a: *const c.Point, b: *const c.Point) -> c.double
    fn set_color(p: *c.Point, color: c.Color)
}

type c.Point = @cstruct {
    x: c.int
    y: c.int
}

type c.Color = @cenum(c.int) {
    RED = 0
    GREEN = 1
    BLUE = 2
}
```

### Binding Generator Internals

- Uses **libclang** to parse C headers (handles macros, typedefs, preprocessor correctly)
- Resolves `#include` chains and `typedef` aliases
- Maps C types to `c.` namespace types according to the type conversion table
- Generates `@opaque` types for forward-declared structs
- Generates `@cstruct` for fully-defined structs
- Generates `@cenum` for enums
- Strips common prefixes from enum variants (e.g., `COLOR_RED` becomes `RED`)
- Preserves original C names in comments for traceability

---

## Build Integration

### `aria.toml` Configuration

```toml
[build.ffi]
# Library search paths
lib_paths = [
    "/usr/local/lib",
    "./third_party/lib",
]

# Header search paths (for aria bind)
include_paths = [
    "/usr/local/include",
    "./third_party/include",
]

# System libraries to always link
system_libs = ["pthread", "dl", "m"]

# Static libraries to bundle
static_libs = ["sqlite3"]
```

### Cross-Compilation

When cross-compiling, FFI libraries must be available for the target platform:

```bash
# Cross-compile with FFI
aria build --target linux-arm64
```

Per-target configuration:

```toml
[build.ffi.target.linux-arm64]
lib_paths = ["./third_party/lib/linux-arm64"]

[build.ffi.target.darwin-arm64]
lib_paths = ["./third_party/lib/darwin-arm64"]
```

---

## Thread Pool for C Calls

### The Problem

Aria uses lightweight tasks (like goroutines). These run on small stacks managed by the Aria runtime scheduler. C functions expect full OS thread stacks. Calling C from a lightweight task requires a stack switch, which is expensive if done per-call (this is Go's `cgo` overhead).

### The Solution

Aria maintains a **thread pool dedicated to C calls**:

```
+----------------------------------------------+
|  Aria Runtime                                  |
|  +----------------------------------------+   |
|  |  Task Scheduler                        |   |
|  |  (lightweight tasks)                   |   |
|  +-------------------+--------------------+   |
|                      | extern call             |
|  +-------------------v--------------------+   |
|  |  C Call Thread Pool                    |   |
|  |  (OS threads, full stacks)             |   |
|  |  +-------+ +-------+ +-------+        |   |
|  |  |  T1   | |  T2   | |  T3   |        |   |
|  |  +-------+ +-------+ +-------+        |   |
|  +----------------------------------------+   |
|                                              |
+----------------------------------------------+
```

- The thread pool is lazily initialized on first `extern` call
- Pool size defaults to `runtime.numCPU()` but is configurable
- Tasks block (yield to scheduler) while waiting for a C call to complete
- Multiple concurrent C calls run on separate pool threads
- No stack switching per call — the pool thread already has a full OS stack

Configuration:

```toml
[runtime]
ffi_threads = 8    # C call thread pool size (default: num CPUs)
```

---

## Pre-Built Registry Packages

For commonly used C libraries, the Aria package registry ships **pre-generated bindings and safe wrappers**:

```toml
[deps]
sqlite = "3.40"      # bindings + safe wrapper + bundled libsqlite3
openssl = "3.0"      # bindings + safe wrapper (links system OpenSSL)
zlib = "1.3"         # bindings + safe wrapper + bundled zlib
curl = "8.0"         # bindings + safe wrapper (links system libcurl)
sdl2 = "2.28"        # bindings + safe wrapper (links system SDL2)
```

These packages include:
1. **Auto-generated raw bindings** (the `extern` declarations)
2. **Hand-written safe wrappers** (idiomatic Aria API, no `Ffi` effect)
3. **Bundled static library** (when licensing permits) or link instructions

For most use cases, developers never interact with FFI directly. They `use sqlite` or `use db` and get a safe, idiomatic Aria API.

---

## Summary

| Aspect | Design Decision | Rationale |
|---|---|---|
| Syntax | Aria syntax with `c.` namespace, never embedded C | One language, no context switching |
| Type mapping | Explicit `c.` types, explicit conversion functions | Zero ambiguity, zero implicit casts |
| Memory ownership | `@owned`, `@borrowed`, `@cffi` annotations | Compiler-verified, deterministic cleanup |
| Resource cleanup | `Drop` trait on wrapper types | Automatic, exactly-once, no finalizer overhead |
| GC interaction | GC tracks wrappers only, not C memory | No GC scanning of C heap |
| Non-moving memory | `@cffi` allocation region | C pointers remain valid, no pinning needed |
| Effect tracking | `Ffi` effect on all extern calls, strippable by safe wrappers | Compiler knows what touches C |
| Binding generation | `aria bind` using libclang | Automated, correct, no manual transcription |
| C call threading | Dedicated OS thread pool | No per-call stack switch overhead |
| Registry packages | Pre-built bindings + safe wrappers for common libraries | Most code never touches FFI directly |