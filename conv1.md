

Let me first get the full contents of the spec and high-level design to make sure the PR is comprehensive and accurate.Now I have a full picture of the repo. Let me create the PR with the AI guide and examples, and then address your deeper question about what you're not covering.The PR is being created now:

````list type="task"
data:
    - title: 'Add AI code generation guide, Copilot instructions, and example programs'
      repository: dan-strohschein/aria-docs
      url: https://github.com/copilot/tasks/pull/PR_kwDORqoNZc7LqnPB?session_id=29bc0695-3bce-42ca-b083-701521801958
      state: queued

````

Now, to your bigger question — **what are you not covering?** After reading through your entire spec corpus (high-level design, stdlib, package system, FFI, paradigm design, language spec addendum), here's what I see missing. I'm organizing this from "will bite you soonest" to "matters later":

---

## 🔴 Critical Gaps (Will Block the Bootstrap Compiler)

### 1. **Operator Precedence & Associativity Table**

You haven't formally specified this anywhere. The high-level design shows operators being used, but never defines:
- What's the precedence of `|>` vs `?` vs `??` vs `+` vs `==` vs `&&`?
- Is `a + b * c` parsed as `a + (b * c)` or `(a + b) * c`?
- You mention `|>` is lower than function call but higher than `:=` in the addendum — but the *full* table doesn't exist

**Why this matters now:** The first thing your Go bootstrap compiler needs to build is a parser. Without a formal precedence table, you'll be making it up as you go — and then every decision becomes a compatibility commitment.

### 2. **Formal Grammar (EBNF/PEG)**

The spec is narrative. That's great for understanding *intent*, but a compiler needs a *formal grammar*. You need something like:

```ebnf
program      := module_decl import* declaration*
module_decl  := 'mod' IDENT
declaration  := fn_decl | type_decl | trait_decl | impl_decl | const_decl
fn_decl      := 'fn' IDENT generic_params? '(' params ')' return_type? fn_body
fn_body      := '=' expr | block
...
```

Without this, you'll be reverse-engineering the grammar from prose and examples every time you hit an ambiguity. And you *will* hit ambiguities — every language does.

### 3. **Scoping Rules**

Where do variable bindings live? Specifically:
- Does `if` introduce a new scope? (Likely yes, but not stated)
- Can you shadow variables? `x := 5; x := "hello"` — is this legal?
- What's the scope of a `for` loop variable?
- Do `match` arms have their own scope?
- Can closures capture mutable variables? If so, what happens with concurrency?
- What are the rules for name resolution when modules have the same function names?

### 4. **Memory Model for Concurrency**

You have `spawn`, `scope`, channels, and `select`. But:
- What happens if two tasks access the same mutable variable? Is it a compile error?
- Are there `Mutex` / `RwLock` primitives? They're not in the stdlib design.
- Is data automatically "moved" into spawned tasks (like Rust), or copied, or shared?
- What about `@stack` allocated data — can it be shared across tasks?

This is where Go has data races, Rust has `Send + Sync`, and you haven't said what Aria does. The effect tracking system *could* solve this ("this function mutates shared state"), but the rules aren't specified.

---

## 🟡 Important Gaps (Will Block Real-World Programs)

### 5. **String Handling Depth**

You say `str` is "UTF-8, immutable." But:
- How do you index into a string? By byte? By codepoint? By grapheme cluster?  
  (`"é"` is 1 grapheme, 1 or 2 codepoints, 2 or 3 bytes depending on normalization)
- Is `str[0]` legal? What does it return?
- What about string slicing? `str[0..5]` — byte offsets or character offsets?
- How does `str.len()` work? Bytes or characters?
- Is there a separate `Char` type?

Every language gets this wrong in a way that's painful later. Go chose bytes (fast, confusing). Rust chose bytes with a `.chars()` iterator (correct, verbose). Swift chose grapheme clusters (correct, slow). You need to decide.

### 6. **Numeric Overflow Behavior**

What happens when `i64` overflows?
- Panic (like Rust debug mode)?
- Wrap (like C, Go, Rust release mode)?
- Undefined behavior (please no)?
- Checked by default with `@unchecked` opt-out for hot paths?

This matters for correctness — AI generating math code needs to know whether `i64.max + 1` panics or wraps silently.

### 7. **Closures and Capture Semantics**

You show closures (`fn(u) => u.active`) but don't specify:
- Capture by value or reference?
- Can closures mutate captured variables?
- What's the type of a closure? Is `fn(i64) -> i64` the same type for a closure and a function pointer?
- Are closures heap-allocated? Stack-allocated? Depends?

### 8. **Initialization and Zero Values**

- Do types have default/zero values? In Go, `int` defaults to `0`, `string` to `""`. In Rust, there are no defaults unless you derive `Default`.
- Can you declare a struct field without a default? If so, what happens if you construct the struct without providing it?
- What about uninitialized memory? Is it possible? The `@stack` annotation suggests manual memory control — can you access uninitialized stack memory?

### 9. **Iteration Protocol**

You show `for item in collection` but haven't defined:
- What trait/interface makes something iterable? Is there an `Iterable` trait?
- How do custom types opt into `for` loops?
- Are iterators lazy or eager?
- What about `for (i, item) in collection.enumerate()`?

---

## 🟠 Design Decisions You'll Regret Deferring

### 10. **Equality and Comparison**

- Is `==` structural equality or reference equality?
- Can you compare structs with `==` by default, or do they need to implement `Eq`?
- What about ordering (`<`, `>`)? Is there an `Ord` trait?
- What about hashing? If you put something in a `Map` or `Set`, it needs to be hashable — is that automatic or opt-in?

### 11. **Const Evaluation / Compile-Time Computation**

- Can you run code at compile time? (Zig's `comptime`, Rust's `const fn`)
- This is huge for performance — things like lookup tables, string processing, config validation at compile time
- If you don't design it now, you can't retrofit it easily

### 12. **Type Aliases vs. Newtypes**

You show `type` for both structs and sum types. But:
- Can you do `type UserId = i64`? Is that a newtype (distinct type) or an alias (interchangeable)?
- If `UserId` and `OrderId` are both `i64` aliases, can you accidentally pass one where the other is expected?
- Newtypes are one of the most valuable tools for AI correctness — preventing semantic type confusion

### 13. **Variadics / Rest Parameters**

- Can functions accept variable numbers of arguments?
- How does `print` work? Is it a special compiler intrinsic, or can user code do the same thing?

### 14. **Annotations / Attributes Beyond Memory**

You have `@stack`, `@arena`, `@inline`, `@opaque`, `@cstruct`. But:
- Is there a general-purpose annotation system?
- Can users define custom annotations?
- What about `@deprecated`, `@unsafe`, `@todo`?
- How do annotations compose with the effect system?

### 15. **Tooling Specification**

Beyond `aria build/run/test/check`, what about:
- `aria fmt` — code formatter (auto-formatting is essential for AI-generated code)
- `aria lint` — you mention this in the package-module doc but don't spec it
- `aria doc` — documentation generation
- `aria repl` — interactive mode
- Debug mode output format
- How does `aria test` discover and run tests? What's the output format?

---

## 🔵 Things You Haven't Thought About Because They're Invisible

### 16. **Error Message Design**

This sounds trivial. It's not. Elm, Rust, and Zig became beloved *specifically because of their error messages*. For an AI-first language, error messages need to be:
- Machine-parseable (structured output for AI tools to read back)
- Actionable (suggest fixes, not just describe problems)
- Context-rich (show the full chain of *why* this is wrong)

### 17. **ABI Stability**

If two Aria packages are compiled separately and linked, do they need to be compiled with the same compiler version? This affects:
- Package distribution (can you distribute precompiled packages?)
- The self-hosting transition (old compiler output + new compiler output = compatible?)
- FFI in the other direction (can C call Aria functions?)

### 18. **Unicode Identifiers**

Can variable names contain Unicode? `let café = "hello"`? `fn 计算(x: i64) -> i64`?  
This matters for global adoption. Most modern languages allow it. But it affects the lexer, the EBNF grammar, and the tooling.

### 19. **Ecosystem Bootstrapping (The Chicken-and-Egg)**

You need all of these *before* anyone can write serious Aria code:
- A package registry (or at least git-based dependencies)
- A standard project template (`aria init`)
- A CI/CD integration story (GitHub Actions, etc.)
- Documentation hosting

### 20. **Versioning the Language Itself**

How do you evolve Aria without breaking existing code?
- Editions (like Rust)?
- Stability guarantees per version?
- Deprecation workflow?

---

The PR will give AI models the ability to generate Aria code *today*. But the items above — especially the first four — are what you need to nail down before writing the first line of Go for the bootstrap compiler.

Want me to create design docs for any of these gaps? I'd recommend tackling **formal grammar**, **operator precedence**, and **scoping rules** next — those are the literal inputs to your parser.



Let me review what's currently on `main` and what's in the open PRs to build a complete picture.Here's the full inventory of what's on `main` vs. what the README currently lists, plus what's been completed through PRs but isn't reflected yet.

---

## ✅ Already on `main` (19 spec files + 2 root docs)

### Root files
| File | Description |
|---|---|
| `high-level-design.md` | Core language spec v0.1 |
| `ARIA_AI_GUIDE.md` | AI coding assistant guide |
| `README.md` | (outdated — see below) |

### `spec/` directory (19 files on `main`)
| File | Topic | Listed in README? |
|---|---|---|
| `formal-grammar.md` | EBNF grammar | ✅ Yes |
| `operator-precedence.md` | Precedence table | ✅ Yes |
| `scoping-rules.md` | Name resolution, shadowing | ✅ Yes |
| `stdlib-design.md` | Standard library design | ✅ Yes |
| `package-module-system.md` | Modules, imports, deps | ✅ Yes |
| `FFI-Considerations.md` | FFI rationale | ✅ Yes |
| `ffi-design.md` | FFI specification | ✅ Yes |
| `string-handling.md` | String types, interpolation | ✅ Yes |
| `numeric-overflow.md` | Integer/float overflow | ✅ Yes |
| `paradigm-design.md` | Programming paradigm | ✅ Yes |
| `language-spec-addendum.md` | Pipeline operator, destructuring | ✅ Yes |
| `compiler-architecture.md` | Compiler pipeline | ✅ Yes |
| `datetime-design.md` | Temporal types, durations | ❌ **Missing from README** |
| `type-conversions.md` | Conversion mechanisms | ❌ **Missing from README** |
| `closures-capture-semantics.md` | Capture semantics | ❌ **Missing from README** |
| `initialization-zero-values.md` | Init rules, `late`, `Default` | ❌ **Missing from README** |
| `iteration-protocol.md` | Iterator traits, `for` loops | ❌ **Missing from README** |
| `error-handling.md` | Error types, `?`, `!`, traces | ❌ **Missing from README** |
| `concurrency-design.md` | Tasks, channels, `scope`, sync | ❌ **Missing from README** |

---

## 🔴 README is stale — 7 specs exist but aren't listed

The README only lists 13 documents. **7 specs** that are already merged to `main` are completely missing from the README:

1. `datetime-design.md`
2. `type-conversions.md`
3. `closures-capture-semantics.md`
4. `initialization-zero-values.md`
5. `iteration-protocol.md`
6. `error-handling.md`
7. `concurrency-design.md`

The README's "Repository Structure" tree is also outdated (only shows 11 spec files vs. the 19 that actually exist).

The Roadmap still says Phase 1.5 is "🔄 **In Progress**" — at this point it looks essentially **Complete**.

---

## 📋 What's left to spec?

Looking at what a complete language spec typically needs and cross-referencing what the existing docs mention but don't have dedicated specs for:

| Topic | Status | Notes |
|---|---|---|
| **Generics / Type Parameters** | 🔜 **Not started** | `high-level-design.md` uses generics syntax (`[T]`, `Map[K,V]`) but no formal spec exists |
| **Trait System** | 🔜 **Not started** | Traits are used everywhere (`Convert`, `TryConvert`, `Display`, `Iterator`, `Send`, `Share`) but no dedicated trait spec |
| **Memory Management / GC** | 🔜 **Not started** | Pillar 4 promises "GC by default, manual control when needed" — no spec |
| **Ownership / Move Semantics** | 🔜 **Not started** | Concurrency spec references `Send`/`Share` and move semantics, but no formal ownership spec |
| **Pattern Matching (deep dive)** | 🔜 **Not started** | Used throughout but only partially specified in `high-level-design.md` |
| **Effect System (deep dive)** | 🔜 **Not started** | `Async`, `IO`, `pure` are used in multiple specs but no dedicated effect system spec |
| **Testing Framework** | 🔜 **Not started** | Error handling spec mentions `assertOk`/`assertErr` but no testing spec |
| **Build System / Tooling** | 🔜 **Not started** | Compiler architecture spec exists but no build tool / `aria.toml` spec |
| **README update** | 🔜 **Needed** | Must add all 7 missing docs, update structure tree, update roadmap |

---

**Summary**: The spec library is in great shape — 19 spec files covering most of the language. The immediate action items are:
1. **Update README.md** to reflect all 19 specs (this should be a single PR)
2. **Generics + Traits** are the biggest missing foundational specs
3. **Memory management / ownership** is the next critical gap
4. **Effect system** and **pattern matching** deep-dives round out the core language