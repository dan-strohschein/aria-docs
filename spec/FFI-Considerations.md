# FFI Design Considerations — Discussion Notes

This document captures the design discussion and reasoning behind Aria's FFI (Foreign Function Interface) decisions. For the final design specification, see [ffi-design.md](ffi-design.md).

---

## 1. How to Declare External C Functions

Both inline and separate binding files are supported. However, **separate binding files are almost always cheaper** for AI generation. Here's why:

Inline C means switching between two languages mid-file. Every context switch costs accuracy. The AI has to remember "I'm in C now" — different conventions, different types, different memory semantics. And the Aria parser has to handle embedded C, which adds grammar complexity (compilation speed cost).

**But** for a single function — say you need `memcpy` or a specific syscall — spinning up a whole binding file is overhead that's not worth it.

The preferred approach:

```c
// Inline — for one-off C calls. Minimal ceremony.
extern fn puts(s: *c.char) -> c.int

// That's it. One line. No C code embedded.
// This is a DECLARATION, not a definition. The linker resolves it.

// Binding file — for libraries with many functions
// bindings/sqlite.aria
mod bindings.sqlite

extern "sqlite3" {
    fn sqlite3_open(filename: *c.char, db: **c.sqlite3) -> c.int
    fn sqlite3_close(db: *c.sqlite3) -> c.int
    fn sqlite3_exec(db: *c.sqlite3, sql: *c.char, callback: c.ptr, arg: c.ptr, err: **c.char) -> c.int
    fn sqlite3_errmsg(db: *c.sqlite3) -> *c.char
}
```

The `extern "sqlite3"` block tells the linker which library to link against. All declarations are Aria syntax with C-compatible types. No actual C code is ever written — Aria declarations describe the C interface. This is **dramatically** cheaper for AI generation:

- No language switching
- No embedded C parsing complexity
- Same syntax used everywhere
- The `c.` prefix namespace makes C types unambiguous

---

## 2. How Types Map

The mapping needs to be **mechanical and unambiguous**. Every time the AI has to think "is this a `char*` or a `const char*` or a `unsigned char*`?" it's burning reasoning tokens and introducing bug risk.

```c
// C type namespace — always prefixed with c.
c.char       // char
c.uchar      // unsigned char
c.short      // short
c.int        // int
c.uint       // unsigned int
c.long       // long
c.ulong      // unsigned long
c.longlong   // long long
c.float      // float
c.double     // double
c.size       // size_t
c.ptr        // void*
c.bool       // _Bool

// Pointers
*c.char      // char*
**c.char     // char**
*c.int       // int*

// Const (read-only pointer)
*const c.char  // const char*
```

**Why the `c.` namespace**: The AI never confuses Aria types with C types. `i32` is an Aria integer. `c.int` is a C integer. They might be the same size, but they have different semantics (Aria's is bounds-checked, C's isn't). The prefix is 2 tokens of overhead but eliminates an entire class of type confusion bugs.

**Conversion between Aria and C types is always explicit:**

```c
// str -> *c.char (for passing to C)
ariaStr := "hello"
cStr := ariaStr.toCStr()   // allocates null-terminated copy
defer c.free(cStr)         // must free!

// *c.char -> str (for receiving from C)
cResult := someCFunction()
ariaStr := str.fromCStr(cResult)  // copies into Aria-managed str

// Numeric — zero cost, just a cast
ariaInt: i32 = 42
cInt := c.int(ariaInt)     // no allocation, just reinterpret

// Slices -> pointers
ariaBytes: [byte] = [1, 2, 3]
cPtr := ariaBytes.ptr()    // raw pointer to underlying data
cLen := ariaBytes.len()    // pass length separately (C convention)
```

---

## 3. Memory Ownership — The Hard Problem

The stakeholder requirement is that Aria should take responsibility for memory when possible, while avoiding the performance hit that Go's GC approach incurs.

### The Solution: The Bridge Pattern — Owned Wrappers with Deterministic Cleanup

```c
// Raw C binding (unsafe, low-level)
extern "sqlite3" {
    fn sqlite3_open(filename: *c.char, db: **c.sqlite3) -> c.int
    fn sqlite3_close(db: *c.sqlite3) -> c.int
}

// Safe Aria wrapper (this is what users actually call)
pub type Database {
    handle: @owned *c.sqlite3   // @owned = Aria takes responsibility
} derives [Drop]

impl Drop for Database {
    fn drop(self) {
        sqlite3_close(self.handle)
    }
}

pub fn open(path: str) -> Database ! DbError {
    handle: *c.sqlite3 = c.null
    cPath := path.toCStr()
    defer c.free(cPath)

    result := sqlite3_open(cPath, &handle)
    if result != 0 {
        return err(DbError{code: result})
    }
    Database{handle: handle}
}
```

### `@owned` vs `@borrowed` — Declare Who's Responsible

```c
// @owned means: Aria's runtime tracks this. When the wrapper is GC'd
// or dropped, the Drop impl runs and cleans up the C resource.
handle: @owned *c.sqlite3

// @borrowed means: someone else owns this. Don't free it.
// Used for pointers passed to callbacks, temporary views, etc.
ref: @borrowed *c.char
```

This is how the Go GC problem is avoided. The GC doesn't scan C memory — it only tracks the **Aria wrapper struct**. When the wrapper is collected, `Drop` runs, which calls the C cleanup function. The C memory is freed by C, not by the GC.

**Performance profile:**
- GC only scans Aria objects (small wrapper struct) — not the C heap
- C memory is freed deterministically via `Drop` — not by GC scanning
- No finalizer overhead (Go's `runtime.SetFinalizer` is slow and unreliable)
- The `Drop` trait is compiled to a destructor call, not a GC hook

### Why This Avoids Go's Performance Hit

Go's `cgo` problem is threefold:

1. **Stack switching**: Go goroutines use small stacks; C needs big stacks. Every cgo call switches stacks. Aria solves this by running C calls on OS threads (like goroutines that call into C already do), but with a **thread pool** for C calls so the cost is amortized.
2. **GC scanning C pointers**: Go's GC has to understand C pointers. Aria's doesn't — C pointers live in `@owned` wrappers, and the GC only sees the wrapper.
3. **Pin/unpin overhead**: Go has to pin memory when passing to C so the GC doesn't move it. Aria's approach: C-facing memory is allocated in a **non-moving region** (arena or manual). The GC never moves it, so no pinning needed.

```c
// C-facing allocation: non-moving, GC-visible but never relocated
buffer := @cffi [byte].alloc(4096)
someCFunction(buffer.ptr(), buffer.len())
// buffer is freed when it goes out of scope or GC collects it
// but it was NEVER moved, so C pointers to it were always valid
```

---

## 4. Auto-Generated Bindings

This is one of the highest-value features for AI code generation.

```bash
# Point at a C header, get Aria bindings
aria bind sqlite3.h --lib sqlite3 --out bindings/sqlite.aria
```

This parses the C header (using libclang internally) and generates:

```c
// AUTO-GENERATED by `aria bind` — do not edit
mod bindings.sqlite

extern "sqlite3" {
    fn sqlite3_open(filename: *c.char, db: **c.sqlite3) -> c.int
    fn sqlite3_close(db: *c.sqlite3) -> c.int
    fn sqlite3_exec(db: *c.sqlite3, sql: *c.char, cb: c.ptr, arg: c.ptr, err: **c.char) -> c.int
    // ... every function in the header
}

// C struct mappings
type c.sqlite3 = @opaque  // opaque type, only used as pointer
```

**Why this matters for AI generation:** The AI doesn't have to manually transcribe C headers — which is error-prone and tedious. The binding generator does it once, correctly, and the AI works with the generated Aria types.

For common libraries, the registry could ship **pre-generated bindings:**

```toml
[deps]
sqlite = "3.40"   # includes pre-generated bindings + safe wrappers
```

---

## 5. Effects System Integration

FFI calls should be tracked by the effects system. This is non-negotiable for AI-generated code.

```c
// This function has the Ffi effect
extern fn puts(s: *c.char) -> c.int  // implicitly: with [Ffi]

// Safe wrappers can strip the effect if they fully encapsulate the unsafety
pub fn print(msg: str) {  // no Ffi effect — it's safe
    cStr := msg.toCStr()
    defer c.free(cStr)
    puts(cStr)
}
```

**Why this matters for AI generation:**
- If a function has the `Ffi` effect, the AI knows it touches C and can be more careful.
- If a function does NOT have `Ffi`, it's pure Aria — full confidence reasoning.
- Effects propagate: if `foo` calls an `extern` function, `foo` has `Ffi` unless it wraps it safely.
- This creates a natural "safe boundary" — most of the codebase is effect-free, with FFI isolated to wrappers.

---

## The Full Architecture: Layers of FFI

```
┌─────────────────────────────────────────────┐
│  Application Code (pure Aria, no Ffi)       │
│  use db                                      │
│  users := db.query[User]("SELECT * ...")?   │
├─────────────────────────────────────────────┤
│  Safe Wrapper (Ffi encapsulated)            │
│  mod db                                      │
│  pub fn query[T](...) -> [T] ! DbError      │
│  // wraps unsafe C calls, manages memory     │
├─────────────────────────────────────────────┤
│  Raw Bindings (auto-generated, Ffi effect)  │
│  mod bindings.sqlite                         │
│  extern "sqlite3" { ... }                    │
├─────────────────────────────────────────────┤
│  C Library (linked at compile time)          │
│  libsqlite3.so / .dylib / .dll              │
└─────────────────────────────────────────────┘
```

AI-generated code operates at the top layer 95% of the time. Safe wrappers exist for common libraries (shipped via the registry). Raw bindings are only generated when interfacing with something exotic — and even then, `aria bind` does the heavy lifting.
