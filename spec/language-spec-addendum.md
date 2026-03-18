# Aria Language Spec Addendum: Pipeline Operator & Destructuring

This addendum formally adds two language features to the Aria v0.1 specification. These features were identified during the [paradigm design](paradigm-design.md) process as high-value additions for AI code generation efficiency.

These features should be considered part of the core language spec alongside [high-level-design.md](../high-level-design.md).

---

## Pipeline Operator (`|>`)

### Summary

The pipeline operator passes the result of the left-hand expression as the first argument to the right-hand expression. It enables left-to-right reading and generation of transformation chains, eliminating deeply nested function calls.

### Grammar

```
pipeline_expr := expr ( "|>" pipe_target )*
pipe_target   := identifier ( "(" arg_list ")" )? "?"?
```

### Desugaring Rules

| Expression | Desugars To |
|---|---|
| `x \|> f` | `f(x)` |
| `x \|> f(a, b)` | `f(x, a, b)` |
| `x \|> f?` | `f(x)?` |
| `a \|> b \|> c` | `c(b(a))` |

### Precedence

- Lower than function call and method access
- Higher than assignment (`:=`, `=`)
- Left-associative

### Examples

```go
// Basic pipeline
result := input |> parse |> validate |> transform |> serialize

// With error propagation
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

// Mixed with method calls
users := db.query[User]("SELECT * FROM users")?
    |> filter(fn(u) => u.active)
    |> sortBy(.lastLogin)
    |> take(10)

// Pipeline into a block (for complex transformations)
report := rawData
    |> parseCSV?
    |> filter(fn(row) => row.year >= 2024)
    |> groupBy(.region)
    |> map(fn((region, rows)) => {
        total := rows.map(.revenue).sum()
        RegionReport{region: region, total: total, count: rows.len()}
    })
```

### Interaction with Other Features

- **Error propagation**: `|> f?` applies `?` after the call, so errors propagate through pipelines naturally.
- **Method calls**: Pipelines and method chains can be mixed freely. Use method syntax when the operation is a method on the value, use pipeline syntax when it's a free function.
- **Closures**: Pipeline targets can be closures: `x |> fn(v) => v * 2`

---

## Expression-Oriented Blocks

### Summary

All block constructs in Aria are expressions — they produce a value. The value of a block is its last expression. This eliminates mutable temporary variables and enables denser code.

### `if` as Expression

```go
// Single line
status := if user.active { "active" } else { "inactive" }

// Multi-line — last expression in each branch is the value
message := if count > 100 {
    log.warn("high count: {count}")
    "too many items"
} else if count > 0 {
    "processing {count} items"
} else {
    "no items"
}
```

**Type rule**: All branches must produce values of the same type. An `if` without `else` has type `void` unless used in statement position.

### Block Expressions

```go
// A bare block's value is its last expression
total := {
    subtotal := items.map(.price).sum()
    tax := subtotal * taxRate
    shipping := if subtotal > 50.0 { 0.0 } else { 5.99 }
    subtotal + tax + shipping  // this is the block's value
}
```

### `match` as Expression

Already in the v0.1 spec. Included here for completeness:

```go
label := match status {
    .Active => "🟢 Active"
    .Pending => "🟡 Pending"
    .Disabled => "🔴 Disabled"
}
```

**Type rule**: All match arms must produce values of the same type. Match must be exhaustive.

### `for` with Collection

`for` loops can also be used as expressions to produce collections (list comprehension):

```go
// for-as-expression produces a list
squares := for x in 1..=10 { x * x }
// equivalent to: (1..=10).map(fn(x) => x * x)

// With filter
evens := for x in 1..100 where x % 2 == 0 { x }
```

---

## Destructuring and Pattern Matching in Bindings

### Summary

Destructuring allows binding multiple variables from a single expression in one statement. Pattern matching in let bindings combines binding with control flow. Both reduce token count and eliminate temporary variables.

### Tuple Destructuring

```go
// Basic
(x, y) := getCoordinates()

// Ignore elements
(_, y, _) := getVector3D()

// Nested
((a, b), c) := getNestedTuple()

// In function parameters
fn distance((x1, y1): Point, (x2, y2): Point) -> f64 {
    sqrt((x2 - x1).pow(2) + (y2 - y1).pow(2))
}
```

### Struct Destructuring

```go
// Extract specific fields
{name, email} := getUser()

// Rename during destructure
{name: userName, email: userEmail} := getUser()

// Rest pattern (ignore remaining fields)
{name, email, ..} := getUser()
```

**Scope rule**: Destructured bindings have the same scope as a normal `:=` binding.

**Type rule**: The destructured fields must exist on the type. Missing fields without `..` is a compile error.

### Pattern Matching in Let Bindings

```go
// Bind if pattern matches, otherwise execute else branch
Some(user) := findUser(id) else return None

// With error capture in else
Ok(data) := parseJson(input) else |err| {
    log.warn("parse failed: {err}")
    return defaultConfig
}

// With enum variants
Circle{radius} := shape else {
    log.warn("expected circle, got {shape}")
    return 0.0
}
```

**Semantics**: The `else` branch must diverge — it must `return`, `break`, `continue`, or `panic`. This ensures the binding is always valid after the statement.

### Destructuring in `for` Loops

```go
// Map entries
for (key, value) in config.entries() {
    println("{key} = {value}")
}

// Struct fields
for {name, score, ..} in students {
    if score >= 90 { println("{name}: honors") }
}

// Nested
for (index, {name, email, ..}) in users.enumerate() {
    println("{index}: {name} <{email}>")
}
```

### Destructuring in `match` Arms

Already supported in v0.1 spec. Extended here with nested patterns:

```go
match response {
    Ok({status: 200, body, ..}) => processBody(body)
    Ok({status: 404, ..}) => handleNotFound()
    Ok({status, ..}) => handleOther(status)
    Err(e) => handleError(e)
}
```

---

## Combined Example

Demonstrating all new features working together:

```go
fn generateReport(db: Database, year: i64) -> Report ! DbError {
    // Pipeline + destructuring + expressions
    {customers, orders} := {
        c := db.query[Customer]("SELECT * FROM customers")?
        o := db.query[Order]("SELECT * FROM orders WHERE year = {year}")?
        (customers: c, orders: o)
    }

    // Pipeline with functional transforms
    summaries := orders
        |> groupBy(.customerId)
        |> map(fn((id, orders)) => {
            Some(customer) := customers.find(fn(c) => c.id == id) else continue
            total := orders.map(.amount).sum()
            discount := match customer.tier {
                .Gold => 0.15
                .Silver => 0.10
                _ => 0.0
            }
            CustomerSummary{
                name: customer.name
                total: total * (1.0 - discount)
                orderCount: orders.len()
            }
        })
        |> sortBy(.total)
        |> reverse

    Report{year: year, summaries: summaries, generated: time.now()}
}
```

This function in Go would require approximately 60-80 lines. In Aria with these features: ~25 lines. A **3x reduction** with equivalent (or greater) type safety.
