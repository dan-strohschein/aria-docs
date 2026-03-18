# Aria Paradigm Design

## Core Identity

> **Expression-oriented procedural with functional composition and trait-based polymorphism.**
> 
> Or more simply: **"Procedural bones, functional blood, no inheritance."**

---

## Why This Paradigm

Programming paradigms are distinctly human constructs — Functional, Procedural, Object-Oriented, Prototypical. Each was designed around how humans think about problems. Aria is designed around how AI thinks about code generation. The question isn't "which paradigm is elegant?" but "which paradigm produces the most correct code in the fewest tokens?"

### Paradigm Analysis from the AI Perspective

#### Procedural (C, early Go)

```
fn processOrder(order: Order) -> Receipt ! Error {
    validated := validate(order)?
    priced := calculateTotal(validated)?
    charged := chargeCard(priced)?
    receipt := generateReceipt(charged)?
    sendEmail(receipt)?
    receipt
}
```

**Strengths for AI:**
- Linear flow maps well to sequential token generation
- Easy to reason about — step 1, step 2, step 3
- Low token cost per operation
- Easy to verify correctness (each line is independent)

**Weaknesses for AI:**
- No abstraction tools beyond "put it in a function"
- Data and behavior are separated — must keep them in sync mentally
- Complex logic becomes a wall of sequential statements

#### Object-Oriented (Java, C#)

```java
public class OrderProcessor {
    private final PaymentGateway gateway;
    private final EmailService emailService;
    private final OrderValidator validator;

    public OrderProcessor(PaymentGateway gateway, EmailService emailService,
                          OrderValidator validator) {
        this.gateway = gateway;
        this.emailService = emailService;
        this.validator = validator;
    }

    public Receipt process(Order order) throws ProcessingException {
        // ...
    }
}
```

**Strengths for AI:**
- Encapsulation groups related behavior (sometimes useful)
- Well-understood patterns with clear templates

**Weaknesses for AI:**
- **Worst paradigm for token efficiency.** Class declaration, private fields, constructor that just assigns fields, getter/setter patterns, inheritance hierarchies, interface-for-everything — enormous structural boilerplate before a single line of logic.
- Deep inheritance hierarchies cause reasoning errors: "Does this method come from the parent? The grandparent? Is it overridden? Which override am I calling?"
- The deeper the hierarchy, the more reasoning required, and the more likely bugs are produced.

#### Pure Functional (Haskell, Elm)

```haskell
processOrder :: Order -> Either Error Receipt
processOrder = generateReceipt <=< chargeCard <=< calculateTotal <=< validate
```

**Strengths for AI:**
- Incredibly information-dense
- Composition is powerful and mechanical
- Pure functions enable independent reasoning about each piece

**Weaknesses for AI:**
- Real-world side effects (IO, state, errors) require monads/monad transformers — expensive in tokens, error-prone in deeply nested type stacks
- Avoidance of mutation generates unnecessary data copying
- Complex type-level programming is a source of AI bugs

#### The Sweet Spot: Expression-Oriented Procedural + Functional Composition

```
fn processOrder(order: Order) -> Receipt ! ProcessError {
    // Procedural: step by step, clear flow
    validated := validate(order)?

    // Functional: data transformation with closures
    lineItems := validated.items
        .filter(fn(i) => i.quantity > 0)
        .map(fn(i) => i.{total: i.price * i.quantity})

    // Expression-oriented: match returns a value
    discount := match validated.customer.tier {
        .Gold => 0.15
        .Silver => 0.10
        .Bronze => 0.05
        .Standard => 0.0
    }

    total := lineItems.map(.total).sum() * (1.0 - discount)

    // Procedural: side effects are explicit and sequential
    charge := chargeCard(validated.customer.card, total)?
    receipt := Receipt{items: lineItems, total: total, chargeId: charge.id}
    sendEmail(validated.customer.email, receipt)?
    receipt  // last expression is return value
}
```

**Why this is the sweet spot:**

1. **Procedural skeleton** = Top-to-bottom generation, one step at a time, minimal backtracking
2. **Functional expressions** = Data transformation chains (`map`, `filter`, `fold`) are dense and rarely incorrect
3. **No classes, no inheritance** = Zero boilerplate; traits provide polymorphism without hierarchies
4. **Everything is an expression** = No `var result; if (...) { result = a } else { result = b }`; just `result := if cond { a } else { b }`
5. **No monads** = Benefits of functional composition without the type-level complexity

---

## Token Cost Comparison

For a typical 100-line function:

| Paradigm | Tokens | Why |
|---|---|---|
| Java-style OOP | ~400-500 | Class wrapper, constructor, getters, inheritance boilerplate |
| Pure Functional (Haskell) | ~150-200 | Dense but complex type signatures, monad handling |
| Go Procedural | ~250-350 | Clean but verbose error handling, no expressions |
| **Aria** | **~120-180** | Minimal boilerplate, expressions everywhere, pipeline chains |

---

## What Aria Already Has

| Feature | Paradigm Source | Status |
|---|---|---|
| `fn` functions, step-by-step logic | Procedural | ✅ In spec |
| `.map`, `.filter`, `.fold` on collections | Functional | ✅ In spec |
| `match` as expression | Functional/Expression | ✅ In spec |
| Closures / lambdas | Functional | ✅ In spec |
| Traits + `impl` blocks | Composition (Rust-like) | ✅ In spec |
| No class keyword, no inheritance | Anti-OOP | ✅ In spec |
| `if`/`match`/blocks as expressions | Expression-oriented | ✅ Added below |
| Pipeline operator (`\|>`) | Functional | ✅ Added below |
| Destructuring in bindings | Functional/Pattern matching | ✅ Added below |

---

## New Language Features

### Pipeline Operator (`|>`)

The pipeline operator passes the result of the left expression as the first argument to the right expression. It enables left-to-right reading of transformation chains.

#### Syntax

```
expression |> function
expression |> function(additional_args)
```

#### Examples

```go
// Without pipeline (inside-out reading, hard to follow)
result := serialize(transform(validate(parse(input)?)?)?)?

// With pipeline (left-to-right, step-by-step)
result := input
    |> parse?
    |> validate?
    |> transform?
    |> serialize?

// With additional arguments
output := data
    |> encode(.UTF8)
    |> compress(.Gzip, level: 6)
    |> encrypt(key)
    |> io.writeFile("output.bin")?

// Combining with method calls
users := db.query[User]("SELECT * FROM users")?
    |> filter(fn(u) => u.active)
    |> sortBy(.lastLogin)
    |> take(10)
    |> map(fn(u) => u.{passwordHash: "[redacted]"})
```

#### Semantics

- `x |> f` desugars to `f(x)`
- `x |> f(a, b)` desugars to `f(x, a, b)`
- `x |> f?` desugars to `f(x)?` (error propagation applies after the call)
- Pipelines are left-associative: `a |> b |> c` = `c(b(a))`
- The pipeline operator has lower precedence than function calls but higher than assignment

#### Why This Matters for AI

- AI generates code sequentially, left-to-right; pipelines match that generation pattern
- Each pipeline stage is independently verifiable
- Deeply nested function calls are a source of parenthesis-matching bugs; pipelines eliminate nesting
- Composes naturally with `?` error propagation

---

### Expression-Oriented Blocks

Everything in Aria is an expression — blocks, `if`, `match`, and loops all return values.

#### `if` as Expression

```go
// if/else returns a value
status := if user.active { "active" } else { "inactive" }

// Multi-line blocks — last expression is the value
message := if count > 100 {
    log.warn("high count: {count}")
    "too many items"
} else if count > 0 {
    "processing {count} items"
} else {
    "no items"
}
```

#### Block Expressions

```go
// A block's value is its last expression
total := {
    subtotal := items.map(.price).sum()
    tax := subtotal * taxRate
    shipping := if subtotal > 50.0 { 0.0 } else { 5.99 }
    subtotal + tax + shipping
}
```

#### `match` as Expression (Already in Spec)

```
label := match status {
    .Active => "🟢 Active"
    .Pending => "🟡 Pending"
    .Disabled => "🔴 Disabled"
}
```

#### Why This Matters for AI

- Eliminates mutable temporary variables (`var result; if (...) { result = a } else { result = b }`)
- Fewer mutation points = fewer bugs
- Denser code — the same logic in fewer tokens
- Every construct returns a value, so composition is always possible

---

### Destructuring and Pattern Matching in Bindings

#### Tuple Destructuring

```go
// Destructure a tuple
(x, y) := getCoordinates()

// Ignore elements with _
(_, y, _) := getVector3D()

// In function parameters
fn distance((x1, y1): Point, (x2, y2): Point) -> f64 {
    sqrt((x2 - x1).pow(2) + (y2 - y1).pow(2))
}
```

#### Struct Destructuring

```go
// Destructure specific fields
{name, email} := getUser()

// Destructure with rename
{name: userName, email: userEmail} := getUser()

// Destructure with rest (ignore remaining fields)
{name, email, ..} := getUser()
```

#### Pattern Matching in Let Bindings

```go
// Bind if pattern matches, otherwise execute else branch
Some(user) := findUser(id) else return None

// With error handling
Ok(data) := parseJson(input) else |err| {
    log.warn("parse failed: {err}")
    return defaultConfig
}
```

#### Destructuring in `for` Loops

```go
// Destructure map entries
for (key, value) in config.entries() {
    println("{key} = {value}")
}

// Destructure struct fields in iteration
for {name, score, ..} in students {
    if score >= 90 { println("{name}: honors") }
}
```

#### Why This Matters for AI

- Eliminates temporary variables: 3 lines become 1
- Reduces token count across entire programs (every avoided `user.name` accessor saves tokens)
- Pattern matching in let bindings combines binding + control flow — fewer constructs to generate
- Destructuring in loop headers is extremely common and eliminates repeated field access

---

## Paradigm Summary

Aria's paradigm is not purely procedural, not purely functional, and explicitly not object-oriented. It takes the strongest elements from each:

| From Procedural | From Functional | Deliberately Not From OOP |
|---|---|---|
| Sequential step-by-step logic | `map`/`filter`/`fold` chains | No classes |
| Mutable state when needed | Immutable by default | No inheritance |
| Simple control flow | Pattern matching | No constructors |
| Side effects are explicit | Pipeline operator | No getters/setters |
| | Closures as values | No method overriding |
| | Everything is an expression | No `this`/`self` pointer ambiguity |
| | Destructuring | |

Polymorphism comes from **traits** — explicit interface contracts without hierarchy. Data is **structs with defaults** — plain data, no behavior hiding. Behavior is **functions and trait implementations** — clear, findable, no inheritance chain to trace.