# Aria Iteration Protocol

## Design Philosophy

Iteration is the most common operation in any program. Aria's iteration protocol is designed around two traits — `Iterable` and `Iterator` — that unify `for` loops, functional chains (`.map()`, `.filter()`), pipelines, destructuring, and collection comprehensions into a single, composable system.

Key principles:
- **One protocol for everything** — `for` loops, `.map()`, `.filter()`, `|>` pipelines all use the same trait
- **Lazy by default** — iterator chains don't allocate intermediate collections
- **Materialization is explicit** — `.collect()`, `.toSet()`, `.toMap()` make allocation visible
- **Works with destructuring** — `for (k, v) in map { ... }` uses the same protocol

---

## Core Traits

```
trait Iterable {
    type Item
    fn iter(self) -> Iterator[self.Item]
}

trait Iterator[T] {
    fn next(mut self) -> T?    // returns Option[T] — Some(value) or None
}
```

- `Iterable` is the "I can be iterated" trait. Types that implement it can be used in `for` loops.
- `Iterator[T]` is the "I produce values one at a time" trait. It has one required method: `next()`.
- `next()` returns `T?` (sugar for `Option[T]`). `Some(value)` means "here's the next element," `None` means "iteration is complete."

---

## `for` Loop Desugaring

All `for` loops desugar to the `Iterable`/`Iterator` protocol:

```
// This:
for x in collection {
    process(x)
}

// Desugars to:
{
    _iter := collection.iter()
    loop {
        match _iter.next() {
            Some(x) => process(x)
            None => break
        }
    }
}
```

---

## Built-in Iterable Types

| Type | Item type | Notes |
|---|---|---|
| `[T]` (List) | `T` | Iterates elements in order |
| `Set[T]` | `T` | Iteration order is unspecified |
| `Map[K, V]` | `(K, V)` | Iterates key-value pairs as tuples |
| `Range` (`0..10`) | `i64` (or inferred integer type) | Iterates in order, exclusive end |
| `Range` (`0..=10`) | `i64` (or inferred integer type) | Iterates in order, inclusive end |
| `str` | `char` | Iterates Unicode codepoints |
| `chan[T]` | `T` | Blocks until value available, stops when channel closed |

```
// Lists
for x in [1, 2, 3] { println(x) }

// Ranges
for i in 0..10 { println(i) }         // 0 to 9
for i in 0..=10 { println(i) }        // 0 to 10

// Strings
for ch in "hello" { println(ch) }     // h, e, l, l, o

// Maps (destructuring tuples)
for (k, v) in myMap { println("{k}: {v}") }

// Sets
for item in mySet { process(item) }

// Channels (blocks until closed)
for msg in channel { handle(msg) }
```

---

## Lazy Iterator Chains

Methods on `Iterator` return new iterators — they don't allocate intermediate collections. The chain is evaluated lazily, one element at a time, when a terminal operation (`.collect()`, `.count()`, `.find()`, etc.) is called.

```
// Lazy — no intermediate lists are created
result := items
    .iter()
    .filter(fn(x) => x.active)      // returns Iterator, not [T]
    .map(fn(x) => x.name)           // returns Iterator, not [str]
    .take(10)                         // returns Iterator, still lazy
    .collect()                        // NOW materializes into [str]
```

**Auto-iter on collections**: When calling iterator methods directly on a collection (not through `.iter()` explicitly), the collection auto-converts to an iterator. This saves tokens in the common case:

```
// These are equivalent:
names := users.iter().map(fn(u) => u.name).collect()
names := users.map(fn(u) => u.name)    // auto-iter, auto-collect for [T] -> [U]

// But when chaining multiple operations, explicit .iter() + .collect() is clearer:
result := users
    .iter()
    .filter(fn(u) => u.active)
    .map(fn(u) => u.name)
    .take(10)
    .collect()
```

---

## Iterator Methods (Complete Table)

### Transforming

| Method | Signature | Description |
|---|---|---|
| `.map(f)` | `fn(T) -> U` → `Iterator[U]` | Transform each element |
| `.flatMap(f)` | `fn(T) -> Iterator[U]` → `Iterator[U]` | Transform and flatten |
| `.filter(f)` | `fn(T) -> bool` → `Iterator[T]` | Keep elements where f returns true |
| `.filterMap(f)` | `fn(T) -> U?` → `Iterator[U]` | Transform + filter in one pass |
| `.enumerate()` | → `Iterator[(u64, T)]` | Pair each element with its index |
| `.zip(other)` | `Iterator[U]` → `Iterator[(T, U)]` | Pair elements from two iterators |
| `.chain(other)` | `Iterator[T]` → `Iterator[T]` | Concatenate two iterators |
| `.flatten()` | (where `T: Iterable`) → `Iterator[T.Item]` | Flatten nested iterables |
| `.scan(init, f)` | `fn(State, T) -> State` → `Iterator[State]` | Stateful map (like fold but yields intermediate values) |
| `.inspect(f)` | `fn(T) -> void` → `Iterator[T]` | Side effect on each element without changing it (for debugging) |

### Limiting

| Method | Signature | Description |
|---|---|---|
| `.take(n)` | `u64` → `Iterator[T]` | First n elements |
| `.skip(n)` | `u64` → `Iterator[T]` | Skip first n elements |
| `.takeWhile(f)` | `fn(T) -> bool` → `Iterator[T]` | Take while predicate is true |
| `.skipWhile(f)` | `fn(T) -> bool` → `Iterator[T]` | Skip while predicate is true |
| `.step(n)` | `u64` → `Iterator[T]` | Every nth element |
| `.chunks(n)` | `u64` → `Iterator[[T]]` | Group into chunks of size n |
| `.windows(n)` | `u64` → `Iterator[[T]]` | Sliding windows of size n |
| `.dedup()` | → `Iterator[T]` (where `T: Eq`) | Remove consecutive duplicates |
| `.unique()` | → `Iterator[T]` (where `T: Hash + Eq`) | Remove all duplicates |

### Reducing (Terminal — consume the iterator)

| Method | Signature | Description |
|---|---|---|
| `.fold(init, f)` | `fn(Acc, T) -> Acc` → `Acc` | Reduce to single value |
| `.reduce(f)` | `fn(T, T) -> T` → `T?` | Reduce without initial value (None if empty) |
| `.count()` | → `u64` | Count elements |
| `.sum()` | → `T` (where `T: Numeric`) | Sum all elements |
| `.product()` | → `T` (where `T: Numeric`) | Multiply all elements |
| `.min()` | → `T?` (where `T: Ord`) | Minimum element |
| `.max()` | → `T?` (where `T: Ord`) | Maximum element |
| `.minBy(f)` | `fn(T) -> K` (where `K: Ord`) → `T?` | Element with minimum key |
| `.maxBy(f)` | `fn(T) -> K` (where `K: Ord`) → `T?` | Element with maximum key |

### Searching (Terminal)

| Method | Signature | Description |
|---|---|---|
| `.any(f)` | `fn(T) -> bool` → `bool` | True if any element matches |
| `.all(f)` | `fn(T) -> bool` → `bool` | True if all elements match |
| `.none(f)` | `fn(T) -> bool` → `bool` | True if no elements match |
| `.find(f)` | `fn(T) -> bool` → `T?` | First element matching predicate |
| `.findMap(f)` | `fn(T) -> U?` → `U?` | Find + transform in one pass |
| `.position(f)` | `fn(T) -> bool` → `u64?` | Index of first matching element |
| `.contains(val)` | `T` (where `T: Eq`) → `bool` | True if value is in iterator |

### Materializing (Terminal)

| Method | Signature | Description |
|---|---|---|
| `.collect()` | → `[T]` | Materialize into a list |
| `.toSet()` | → `Set[T]` (where `T: Hash + Eq`) | Materialize into a set |
| `.toMap()` | → `Map[K, V]` (where `T = (K, V)`) | Materialize into a map |
| `.join(sep)` | `str` → `str` (where `T: Display`) | Join into a string with separator |
| `.groupBy(f)` | `fn(T) -> K` → `Map[K, [T]]` | Group elements by key |
| `.partition(f)` | `fn(T) -> bool` → `([T], [T])` | Split into (matching, non-matching) |
| `.unzip()` | (where `T = (A, B)`) → `([A], [B])` | Unzip pairs into two lists |

---

## `for`-as-Expression (List Comprehension)

`for` loops can be used as expressions that produce collections:

```
// Basic comprehension
squares := for x in 1..=10 { x * x }
// Produces: [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
// Desugars to: (1..=10).iter().map(fn(x) => x * x).collect()

// With filter (where clause)
evens := for x in 1..100 where x % 2 == 0 { x }
// Desugars to: (1..100).iter().filter(fn(x) => x % 2 == 0).collect()

// With destructuring
names := for {name, active, ..} in users where active { name }
// Desugars to: users.iter().filter(fn(u) => u.active).map(fn(u) => u.name).collect()

// Nested comprehension
pairs := for x in 1..=3 {
    for y in 1..=3 {
        (x, y)
    }
}
// Produces: [(1,1), (1,2), (1,3), (2,1), (2,2), (2,3), (3,1), (3,2), (3,3)]
```

---

## Custom Iterable Types

Any type can participate in the iteration protocol by implementing `Iterable`:

```
type FileLines {
    path: str
}

impl Iterable for FileLines {
    type Item = str

    fn iter(self) -> Iterator[str] {
        reader := io.open(self.path)!
        LinesIterator{reader: reader}
    }
}

type LinesIterator {
    reader: io.Reader
}

impl Iterator[str] for LinesIterator {
    fn next(mut self) -> str? {
        match self.reader.readLine() {
            Ok(line) => Some(line)
            Err(_) => None
        }
    }
}

// Now works everywhere:
for line in FileLines{path: "data.csv"} {
    process(line)
}

count := FileLines{path: "data.csv"}
    .iter()
    .filter(fn(line) => line.contains("error"))
    .count()
```

---

## Infinite Iterators

Iterators can be infinite — they never return `None` from `next()`. Use `.take()` or `.takeWhile()` to limit them:

```
// Built-in infinite iterators
naturals := Iterator.from(0, fn(n) => n + 1)    // 0, 1, 2, 3, ...
repeated := Iterator.repeat(42)                   // 42, 42, 42, ...
cycled := [1, 2, 3].iter().cycle()                // 1, 2, 3, 1, 2, 3, ...

// Must be limited before materializing
first10 := naturals.take(10).collect()            // [0, 1, 2, ..., 9]
```

---

## Interaction with Other Features

**Destructuring in `for` loops** (see `spec/language-spec-addendum.md`):
```
for (key, value) in config.entries() {
    println("{key} = {value}")
}

for (index, {name, email, ..}) in users.enumerate() {
    println("{index}: {name} <{email}>")
}
```

**Pipeline operator** (see `spec/paradigm-design.md`):
```
result := data
    |> filter(fn(x) => x.active)
    |> map(fn(x) => x.name)
    |> take(10)
    |> collect()
```

**Error propagation in iterators:**
```
// filterMap + ? for fallible operations
results := inputs
    .iter()
    .filterMap(fn(input) => {
        match process(input) {
            Ok(val) => Some(val)
            Err(_) => None           // skip failures
        }
    })
    .collect()

// Or collect Results:
results := inputs.map(fn(x) => process(x)).collect()  // [Result[T, E]]
// Then:
allResults := results.sequence()?  // Result[[T], E] — fails on first error
```

**Closures and field shorthand** (see `spec/closures-capture-semantics.md`):
```
names := users |> map(.name)
actives := users |> filter(.active)
lengths := strings |> map(.len())
```

---

## Performance Characteristics

- **Lazy evaluation**: No intermediate collections are allocated in iterator chains
- **Single-pass**: Each element passes through the entire chain before the next element is processed
- **Fusion**: The compiler can fuse adjacent `.map()` calls into a single pass
- **Bounds checking**: The compiler can elide bounds checks when iterating over known-length collections
- **Vectorization**: Simple `.map()` operations on numeric lists can be auto-vectorized by the LLVM backend

---

## Design Rationale Summary

| Decision | Rationale |
|---|---|
| Two traits (`Iterable` + `Iterator`) | Simple protocol, easy to implement for custom types |
| `next() -> T?` | Natural integration with Option and pattern matching |
| Lazy by default | No intermediate allocations in chains — better for AI-generated multi-stage transformations |
| Explicit `.collect()` | Materialization cost is visible in the code |
| Auto-iter on collections | Saves tokens for simple single-method calls |
| `for`-as-expression | List comprehensions without new syntax — reuses existing constructs |
| Field shorthand (`.name`) | Most common closure pattern reduced to minimum tokens |

---

## Comparison with Other Languages

| Feature | Go | Rust | Python | Aria |
|---|---|---|---|---|
| For loop protocol | `range` keyword | `IntoIterator` trait | `__iter__`/`__next__` | `Iterable` trait |
| Lazy chains | None (manual) | `.iter().map().filter()` | Generators | `.iter().map().filter()` |
| List comprehension | None | `.collect()` | `[x for x in ...]` | `for x in ... { x }` |
| Materialization | Implicit (slices) | Explicit (`.collect()`) | Implicit (list comp) | Explicit (`.collect()`) |
| Field shorthand | None | None | `attrgetter` | `.fieldName` |

---

## Related Specs

- `high-level-design.md` — Core language design overview
- `spec/stdlib-design.md` — Collection types (`[T]`, `Map[K,V]`, `Set[T]`) that implement `Iterable`
- `spec/language-spec-addendum.md` — Destructuring in `for` loops
- `spec/paradigm-design.md` — Functional composition and pipeline operator
- `spec/closures-capture-semantics.md` — Closure syntax and capture rules used in iterator methods
