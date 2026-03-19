# Implementation Decisions — Deferred to Compiler Phase

These are implementation-level decisions that don't affect the language specification but must be resolved before or during compiler implementation. Captured here from the pre-implementation design review (2026-03-18).

---

## ~~10. GC Algorithm Specifics~~ → RESOLVED

**Resolved:** Formalized in [spec/garbage-collector.md](spec/garbage-collector.md). Design: generational GC with per-task nurseries (bump-pointer allocation, ~2-3ns/alloc, zero contention), concurrent mark-compact old generation (<1ms pauses), card-marking write barriers only for mutable reference stores. Bootstrap uses Go's built-in GC.

---

## ~~11. Task Scheduler Design~~ → RESOLVED

**Resolved:** Formalized in [spec/task-scheduler.md](spec/task-scheduler.md). Design: work-stealing with per-worker local queues, platform-adaptive I/O (io_uring/epoll/kqueue/IOCP), contiguous stacks with copy-on-growth (8KB initial), cooperative preemption with 10ms signal-based fallback, dedicated FFI thread pool, equal task priority for v0.1.

---

## ~~12. Integer Literal Type Suffixes~~ → RESOLVED

**Resolved:** No integer literal suffixes in Aria. Use type annotations on bindings (`x: u8 = 42`, `items: [u8] = [1, 2, 3]`) or lossless conversion syntax (`u32(4096)`). Rationale: suffixes add parser complexity for negligible token savings, create a second way to express types (violates Pillar 1), and the annotation form is already shorter for the most common case (typed arrays). Documented in [spec/design-decisions-v01.md](spec/design-decisions-v01.md).

---

## ~~13. String Representation Details~~ → RESOLVED

**Resolved:** Formalized in the rewritten [spec/string-handling.md](spec/string-handling.md). Design: 24-byte value type with small string optimization (≤23 bytes inline, no heap). Byte indexing (`s[i]` returns `u8`, O(1)). `s.len()` = byte length. `.chars()` for codepoint iteration, `.charCount()` for O(n) codepoint count, `.graphemes()` for visual characters. Character boundary validation on slicing (panic, not silent corruption). Compile-time literal interning. Zero-copy substrings via shared backing memory. `StringBuilder` for loop construction.

---

## 14. Collection Growth Factors

`[T]` is a growable list. Implementation decisions:
- **Growth factor:** 2x (classic doubling) or 1.5x (less memory waste)?
- **Initial capacity:** 0 (allocate on first push) or small default (e.g., 4)?
- **Shrinking:** does the list ever shrink its backing array?
- **Map load factor:** what threshold triggers rehashing? (typically 0.75)

---

## 15. Debug Info Format

For debugger integration:
- **DWARF** — standard, works with GDB/LLDB
- **Custom format** — optimized for Aria's needs but requires custom tooling
- **Source maps** — for mapping optimized code back to source (especially for the LLVM tier)

**Consideration:** DWARF is the pragmatic choice. Custom format can come later for Aria-specific debugger features.

---

## 16. Contiguous Memory Allocation / Buffer Management

For high-performance I/O, networking, and serialization, Aria needs a story for contiguous byte buffers that the programmer controls directly.

**What's needed:**
- A `Buffer` type (or similar) backed by a contiguous byte array with read/write cursors
- Ability to pre-allocate a fixed-size region: `Buffer.withCapacity(64kb)`
- Zero-copy slicing — take a view into a buffer without copying
- Integration with `@arena` for request-scoped buffer pools
- Growth semantics — does a buffer grow automatically, or is it fixed-size with an error on overflow?

**Considerations:**
- The stdlib already references `Buffer` in examples but doesn't formally spec it
- Buffers are the foundation of I/O, serialization, and protocol implementations
- `@stack Buffer.new(4096)` should work for stack-allocated scratch buffers
- Needs clear ownership: who owns the backing memory? The buffer? The arena? The GC?

**Likely home:** `std.io.Buffer` or a Tier 0 built-in, depending on how fundamental it is.

---

## 17. Memory-Mapped I/O (mmap)

Memory-mapped files are essential for high-performance file processing (databases, search indexes, large file processing). Aria needs an API, likely in the stdlib.

**What's needed:**
- `mmap` a file (or anonymous region) into the address space
- Read-only, read-write, and copy-on-write modes
- Explicit unmap with deterministic cleanup (via `with` or `Drop`)
- Integration with the memory model: mmap'd regions are NOT GC-managed
- Safety: accessing an mmap'd region after unmap must be prevented (compile-time or runtime)

**Design sketch:**
```
use std.io.mmap

with region := mmap.open("data.bin", mode: .ReadOnly)? {
    // region is a [byte] view into the file — no copy
    header := region[0..64]
    process(header)
}
// region unmapped here
```

**Considerations:**
- mmap interacts with `@cffi` memory (non-moving, not GC-managed)
- Needs to work with the `Ffi` effect (or a new `Mmap` effect?)
- Platform differences: mmap behavior varies between Linux/macOS/Windows
- Should be in the stdlib, not a language primitive

---

## 18. Direct I/O / Positioned I/O (O_DIRECT, pread, pwrite)

For databases and storage engines, bypassing the OS page cache and doing positioned reads/writes is critical.

**What's needed:**
- `O_DIRECT` flag on file open — bypass page cache for aligned I/O
- `pread(fd, buf, offset)` / `pwrite(fd, buf, offset)` — read/write at a specific file offset without seeking
- Alignment requirements: O_DIRECT typically requires page-aligned buffers (4KB)
- Integration with buffer management (item 16) — aligned allocation

**Design sketch:**
```
use std.io

file := io.open("data.db", flags: [.ReadWrite, .Direct])?
defer file.close()

// Positioned read — no seek, no shared cursor, safe for concurrent access
buf := @aligned(4096) Buffer.withCapacity(4096)
file.pread(buf, offset: page_num * 4096)?

// Positioned write
file.pwrite(data, offset: page_num * 4096)?
file.sync()?    // fsync
```

**Considerations:**
- `@aligned(N)` annotation may be needed for O_DIRECT buffer alignment
- pread/pwrite are inherently thread-safe (no shared cursor) — good fit for concurrent Aria tasks
- This is stdlib/FFI level, not language level — wraps POSIX syscalls

---

## 19. Memory Ordering on Atomics (Acquire/Release/Relaxed)

The concurrency spec defines `sync.Atomic[T]` with basic operations (`load`, `store`, `add`, `cas`) but doesn't specify memory ordering semantics.

**What's needed:**
- Memory ordering options: `Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst`
- Default ordering (when not specified) — `SeqCst` is safest, `AcqRel` is typical
- Explicit ordering parameter on atomic operations

**Design sketch:**
```
counter := sync.Atomic[i64].new(0)

// Default — sequentially consistent (safest, most expensive)
counter.store(42)
val := counter.load()

// Explicit ordering for performance-critical code
counter.store(42, order: .Release)
val := counter.load(order: .Acquire)

// Relaxed — no ordering guarantees (fastest, for counters where order doesn't matter)
counter.add(1, order: .Relaxed)
```

**Considerations:**
- Default should be `SeqCst` — safest choice, matches Go's atomic semantics
- Relaxed/Acquire/Release are expert-level — most Aria code should use defaults
- The effect system should require `Async` for all atomic operations (already the case)
- This is API design, not language design — lives in `sync.Atomic[T]` method signatures

---

## 20. System Call Access for File Sync Primitives

Databases and storage engines need fine-grained control over when data hits disk.

**What's needed:**
- `file.sync()` — fsync (flush file data + metadata to disk)
- `file.dataSync()` — fdatasync (flush file data only, skip metadata — faster)
- `dir.sync()` — fsync on directory (ensures directory entries are durable)
- `file.syncRange(offset, len)` — sync_file_range (Linux-specific, partial sync)
- Error handling: sync failures MUST be surfaced (silent fsync failure = data loss)

**Design sketch:**
```
file := io.open("wal.log", flags: [.ReadWrite, .Append, .Create])?
defer file.close()

file.write(entry.serialize())?
file.dataSync()?    // ensure this entry is durable before acknowledging

// Directory sync after creating a new file (ensures the filename is durable)
dir := io.openDir("data/")?
dir.sync()?
```

**Considerations:**
- These are thin wrappers over POSIX syscalls — stdlib level
- All require `Io` + `Fs` effects
- Error handling is critical — a failed fsync means the data may not be on disk
- Platform differences: `fdatasync` doesn't exist on all platforms (macOS has `F_FULLFSYNC`)

---

## 21. Unsafe/Trusted Blocks for Zero-Copy Byte Casting

**Requires a small language addition.**

For serialization, protocol parsing, and database page management, casting between `[byte]` and a struct without copying is essential for performance. This is inherently unsafe — the bytes may not represent a valid struct.

**What's needed:**
- A way to reinterpret a byte slice as a typed value (and vice versa)
- Must be explicitly marked as unsafe — the compiler can't verify correctness
- The cast itself is zero-copy — just reinterpret the pointer

**Design options:**

**Option A: `@trusted` block**
```
@trusted {
    header := bytes.cast[PageHeader]()     // zero-copy reinterpret
    data := header.cast[[byte]]()          // cast back to bytes
}
```

**Option B: `unsafe` module functions**
```
use std.unsafe

header := unsafe.cast[PageHeader](bytes)?   // returns Result — checks alignment/size
header := unsafe.castUnchecked[PageHeader](bytes)  // no checks — caller responsible
```

**Considerations:**
- This is the ONE place where Aria needs something like `unsafe`
- Option B (module functions) is cleaner — doesn't require a new block syntax
- Should require the `Ffi` effect (since it's bypassing the type system, like calling C)
- Alignment must be checked or guaranteed by the caller
- The `@cstruct` annotation on the target type could enable safe-ish casting (compiler verifies layout)

---

## 22. SIMD Intrinsics or Auto-Vectorization Hints

**Nice-to-have for closing the last performance gap.**

SIMD (Single Instruction, Multiple Data) operations process multiple values in parallel using wide CPU registers (128-bit SSE, 256-bit AVX, etc.). This is critical for:
- String searching (memchr, SIMD-accelerated parsing)
- Hashing (CRC32, SIMD-accelerated hash functions)
- Numerical computation (vector/matrix math)
- Compression/decompression

**Options:**

**Option A: Auto-vectorization only**
- Rely on LLVM's auto-vectorizer (Tier 2 backend) to detect and optimize vectorizable loops
- Add `@vectorize` hint annotation for loops the compiler should try harder to vectorize
- No explicit SIMD types or intrinsics

**Option B: Explicit SIMD types**
```
use std.simd

a := simd.f64x4.load(data, offset: 0)     // load 4 f64s from memory
b := simd.f64x4.load(data, offset: 32)
c := a + b                                   // SIMD addition — 4 adds in one instruction
c.store(result, offset: 0)
```

**Option C: Platform-specific intrinsics (escape hatch)**
```
use std.simd.x86

// Explicit SSE/AVX intrinsics — maximum control, minimum portability
result := x86.mm256_add_pd(a, b)
```

**Considerations:**
- Option A is sufficient for v0.1 — LLVM is good at auto-vectorization
- Option B is the right long-term design — portable SIMD types
- Option C is for experts only — should be behind `Ffi` effect
- The `@vectorize` hint annotation fits cleanly into the closed annotation set (add it when needed)
- SIMD interacts with memory alignment — may need `@aligned(32)` annotation

---

## 23. Cache-Line Alignment Annotations

**Nice-to-have for closing the last performance gap.**

False sharing occurs when two independently-accessed values share a cache line (typically 64 bytes), causing cache invalidation across CPU cores. This is a performance killer in concurrent code.

**What's needed:**
- `@aligned(N)` annotation on struct fields or types to guarantee alignment
- `@cacheline` shorthand for `@aligned(64)` (typical cache line size)
- Padding to prevent false sharing between concurrently-accessed fields

**Design sketch:**
```
type WorkerState {
    @cacheline counter: sync.Atomic[i64]    // aligned to cache line boundary
    @cacheline status: sync.Atomic[bool]     // separate cache line from counter
}

// Or on types:
@aligned(64)
type CacheAlignedCounter {
    value: sync.Atomic[i64]
}
```

**Considerations:**
- This is a pure performance annotation — no semantic change
- Fits into the existing annotation system (add `@aligned(N)` and `@cacheline`)
- Only matters for concurrent, high-throughput code — the 5% case
- Cache line size varies by platform (64 bytes on x86/ARM, 128 bytes on some ARM servers)
- Could detect at compile time for the target platform, or use a conservative default

---

*These items are captured for the implementation phase. They do not block the language specification or the bootstrap compiler design. Items 21-23 may require small spec additions when they are implemented.*
