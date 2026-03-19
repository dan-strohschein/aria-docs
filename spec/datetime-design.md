# Aria Date, Time, and Duration Design

## Design Philosophy

Aria's datetime system is designed to minimize errors in generated code by making distinctions between temporal concepts explicit at the type level. The compiler prevents mixing incompatible temporal types, forces disambiguation of ambiguous calendar arithmetic, and eliminates stringly-typed formatting.

Key principles:

- **Separate types for separate concepts** — wall clock vs monotonic time, absolute vs calendar durations
- **The compiler catches temporal bugs** — no mixing `Instant` and `DateTime` arithmetic, no calendar math without choosing overflow strategy
- **Token efficiency** — common operations (get current time, format ISO 8601, compute deadline) should be 1-3 lines

---

## Core Type Hierarchy

Four distinct types cover all temporal use cases:

| Type | What it is | Timezone? | Use case |
|---|---|---|---|
| `Instant` | Point on monotonic timeline (nanoseconds since epoch) | No — it's absolute | Timestamps, measuring elapsed time, scheduling, database storage |
| `DateTime` | Wall clock with timezone | Always (required) | Display to users, calendar operations, scheduling in human terms |
| `Date` | Calendar date (year-month-day) | Never | Birthdays, deadlines, date-only records |
| `Time` | Wall clock time (hour-minute-second-nanosecond) | Never | Alarm times, opening hours, time-only records |

---

### `Instant`

`Instant` represents a point on the monotonic timeline — an absolute moment with no calendar interpretation. It is the primary output of `time.now()`.

- Internal representation: nanoseconds since Unix epoch (`i64` — covers ±292 years from 1970)
- `time.now()` returns `Instant` — this is the primary way to get "current time"
- Subtraction: `instant2 - instant1` gives a `dur` (absolute duration)
- Addition: `instant + dur` gives a new `Instant`
- Comparison: fully ordered — `<`, `>`, `==` all work
- Cannot do calendar arithmetic on an `Instant` — must convert to `DateTime` first
- `instant.inZone(.UTC)` or `instant.inZone(tz"America/New_York")` converts to `DateTime`
- Epoch accessors: `instant.epochSecs() -> i64`, `instant.epochMillis() -> i64`, `instant.epochNanos() -> i64`

```
now := time.now()           // Instant
later := time.now()
elapsed := later - now      // dur
deadline := now + 30s       // Instant
```

---

### `DateTime`

`DateTime` represents a wall-clock moment tied to a specific timezone. It always carries a timezone — there is no "naive" datetime in Aria.

- Constructed from `Instant` + timezone: `instant.inZone(.UTC)`
- Or directly: `DateTime.of(2026, 3, 18, 14, 30, 0, .UTC)`
- Fields: `.year -> i64`, `.month -> u8`, `.day -> u8`, `.hour -> u8`, `.minute -> u8`, `.second -> u8`, `.nano -> u32`, `.zone -> Timezone`
- Convert to other zones: `dt.inZone(tz"Europe/London")`
- Extract components: `dt.date() -> Date`, `dt.time() -> Time`, `dt.instant() -> Instant`
- Absolute arithmetic with `dur` is allowed: `dt + 2h` (adds exactly 2 hours of absolute time, adjusting the wall clock accordingly — DST-safe)
- Calendar arithmetic requires `Period` with explicit overflow handling (see [Calendar Arithmetic](#calendar-arithmetic-with-overflow-handling))

```
now := time.now().inZone(.UTC)        // DateTime
year := now.year                       // i64
tomorrow := now + 24h                  // DateTime (adds absolute time)
```

---

### `Date`

`Date` represents a calendar date — year, month, and day only. No time of day, no timezone.

- `Date.of(2026, 3, 18)` — construct directly
- `Date.today(.UTC)` — requires a timezone to determine which calendar day "today" is
- Fields: `.year -> i64`, `.month -> u8`, `.day -> u8`, `.dayOfWeek -> DayOfWeek`, `.dayOfYear -> u16`
- Calendar arithmetic with `Period` (with overflow handling)
- Comparison: fully ordered

```
birthday := Date.of(1990, 6, 15)
today := Date.today(.UTC)
```

---

### `Time`

`Time` represents a wall-clock time — hour, minute, second, and nanosecond. No date, no timezone.

- `Time.of(14, 30, 0)` — construct directly
- `Time.now(.UTC)` — requires a timezone to determine the current wall-clock time
- Fields: `.hour -> u8`, `.minute -> u8`, `.second -> u8`, `.nano -> u32`
- Duration arithmetic wraps around midnight: `Time.of(23, 0, 0) + 2h` → `Time.of(1, 0, 0)`
- Comparison: fully ordered

```
alarm := Time.of(7, 30, 0)
```

---

## Two Duration Types

| Type | What it is | Literal syntax | Example |
|---|---|---|---|
| `dur` | Absolute duration (nanosecond precision) | `30s`, `5m`, `2h`, `100ms`, `500us`, `1ns` | Timeouts, elapsed time, sleep |
| `Period` | Calendar duration (variable-length) | Struct construction | Months, years — length depends on which month/year |

---

### `dur` (Absolute Duration)

`dur` is already part of the language spec (see `high-level-design.md`) with built-in literal syntax. It represents an exact number of nanoseconds.

- Internal representation: nanoseconds (`i64`)
- Literal suffixes: `ns`, `us` (microseconds), `ms`, `s`, `m` (minutes), `h`
- Compound durations: not supported as literals; use addition — `1h + 30m`
- Arithmetic: `dur + dur`, `dur - dur`, `dur * i64`, `dur / i64`, `dur / dur -> f64`
- Methods: `.hours() -> f64`, `.minutes() -> f64`, `.seconds() -> f64`, `.millis() -> i64`, `.nanos() -> i64`
- Extension methods on numeric types: `30.seconds()`, `5.minutes()`, `2.hours()`

```
timeout := 30s
halfLife := timeout / 2          // 15s
ratio := elapsed / timeout       // f64
```

---

### `Period` (Calendar Duration)

`Period` represents a human-scale calendar interval such as "1 month" or "2 years and 5 days". Its actual length in seconds varies depending on the reference date — 1 month from January 31 is different from 1 month from March 1.

- Construction: `Period{months: 1}`, `Period{years: 2, days: 5}`, `Period{weeks: 1}`
- Fields: `.years -> i64`, `.months -> i64`, `.weeks -> i64`, `.days -> i64`
- Cannot be converted to `dur` without a reference date (1 month ≠ 30 days)
- Adding to `DateTime` or `Date` requires an explicit overflow strategy (see below)
- `Period` is **not orderable** — "is 1 month > 30 days?" is context-dependent

---

## Calendar Arithmetic with Overflow Handling

"January 31 + 1 month" is ambiguous — February 28 has only 28 days. Rather than silently clamping or wrapping, Aria forces the caller to choose a strategy. This eliminates an entire class of subtle calendar bugs.

```
dt := DateTime.of(2026, 1, 31, 12, 0, 0, .UTC)

// Option 1: Clamp to last valid day in the resulting month
dt.add(Period{months: 1}, overflow: .Clamp)
// → Feb 28, 2026

// Option 2: Return an error if the resulting day doesn't exist
dt.add(Period{months: 1}, overflow: .Error)
// → Err(InvalidDate)

// Option 3: Carry the overflow into the next month
dt.add(Period{months: 1}, overflow: .Carry)
// → Mar 3, 2026
```

**Rule**: All `Period` arithmetic on `DateTime` and `Date` **requires the `overflow` parameter**. There is no default. This forces every callsite to make an explicit choice, eliminating silent date clamping and wrapping bugs.

The same rule applies to `Date`:

```
d := Date.of(2026, 1, 31)
d.add(Period{months: 1}, overflow: .Clamp)    // Date.of(2026, 2, 28)
d.add(Period{months: 1}, overflow: .Error)    // Err(InvalidDate)
d.add(Period{months: 1}, overflow: .Carry)    // Date.of(2026, 3, 3)
```

---

## Timezone Handling

```
// Built-in zones
.UTC
.Local   // system timezone

// IANA timezone literals — compile-time validated
tz"America/New_York"
tz"Europe/London"
tz"Asia/Tokyo"

// Runtime timezone (user input, config files) — fallible
zone := Timezone.parse("America/New_York")?   // Result[Timezone, TimezoneError]
```

- `tz"..."` is a **compile-time validated timezone literal** — if the string is not a valid IANA timezone identifier, it is a compile error, not a runtime panic
- `Timezone.parse()` handles runtime timezone strings and returns `Result`
- DST transitions are handled automatically — adding `1h` of absolute time across a DST boundary correctly adjusts the wall clock

---

## Formatting — Method-Based, Not String-Based

Formatting uses methods and enum variants, not format strings. This eliminates the "wrong format dialect" error class and avoids the stringly-typed confusion of Go's reference-date approach.

```
dt := time.now().inZone(.UTC)

// Standard formats — zero-argument methods
dt.iso8601()           // "2026-03-18T14:30:00Z"
dt.rfc3339()           // "2026-03-18T14:30:00+00:00"
dt.rfc2822()           // "Wed, 18 Mar 2026 14:30:00 +0000"

// Component formats — enum variants
dt.format(.Date)          // "2026-03-18"
dt.format(.Time)          // "14:30:00"
dt.format(.DateCompact)   // "20260318"

// Custom format — uses Aria's interpolation syntax, not a format string dialect
dt.format(.Custom("{year}-{month:02}-{day:02} {hour:02}:{minute:02}"))

// Date and Time types have their own formatting
date.iso8601()            // "2026-03-18"
time.iso8601()            // "14:30:00"
```

---

## Parsing — Explicit and Fallible

All parsing operations are explicit and return `Result`. There are no implicit conversions from strings to temporal types.

```
// Standard format parsing
DateTime.parseIso("2026-03-18T14:30:00Z")?          // Result[DateTime, ParseError]
DateTime.parseRfc3339("2026-03-18T14:30:00+00:00")?
Date.parseIso("2026-03-18")?
Time.parseIso("14:30:00")?

// Custom format parsing
DateTime.parse("18/03/2026 14:30", .Custom("{day:02}/{month:02}/{year} {hour:02}:{minute:02}"))?

// Instant from epoch values
Instant.fromEpochSecs(1742310600)
Instant.fromEpochMillis(1742310600000)
```

---

## Comparison and Ordering

| Type | Orderable? | Notes |
|---|---|---|
| `Instant` | ✅ Yes | Fully ordered — absolute nanosecond comparison |
| `DateTime` | ✅ Yes | Compared by underlying `Instant` — timezone does not affect ordering |
| `Date` | ✅ Yes | Chronological ordering |
| `Time` | ✅ Yes | `00:00:00` < `23:59:59` |
| `dur` | ✅ Yes | Nanosecond comparison |
| `Period` | ❌ No | "Is 1 month > 30 days?" is context-dependent |

---

## Interaction with Other Language Features

**Pattern matching on `DayOfWeek`:**

```
match dt.dayOfWeek {
    .Monday | .Wednesday | .Friday => "MWF schedule"
    .Tuesday | .Thursday => "TTh schedule"
    _ => "weekend"
}
```

**Pipeline operator:**

```
result := time.now()
    |> .inZone(.UTC)
    |> .add(Period{months: 3}, overflow: .Clamp)
    |> .format(.Date)
```

**Error propagation with `?`:**

```
fn parseDeadline(input: str) -> DateTime ! ParseError {
    DateTime.parseIso(input)?
}
```

---

## Design Rationale Summary

| Decision | Rationale |
|---|---|
| Four distinct temporal types | Compiler prevents mixing wall-clock and monotonic time |
| `Instant` from `time.now()` | Monotonic by default — correct for elapsed time measurement |
| `DateTime` always has timezone | Eliminates "naive datetime" ambiguity |
| Two duration types (`dur` vs `Period`) | "1 month" ≠ "30 days" — compiler enforces the distinction |
| Forced `overflow` parameter on `Period` arithmetic | Eliminates silent date clamping/wrapping bugs |
| `tz"..."` compile-time literals | Invalid timezone caught at compile time, not runtime |
| Method-based formatting | No format string dialect confusion (no Go reference-date magic) |
| `Period` is not orderable | Prevents meaningless comparisons between variable-length intervals |

---

## Token Comparison

Common datetime operations in Go vs Aria:

**Get current time, format as ISO 8601:**

| Language | Code |
|---|---|
| Go | `time.Now().UTC().Format(time.RFC3339)` (1 import, must know the constant name) |
| Aria | `time.now().inZone(.UTC).iso8601()` (`time` is Tier 1 — no import needed) |

**Compute a deadline 30 minutes from now:**

| Language | Code |
|---|---|
| Go | `deadline := time.Now().Add(30 * time.Minute)` |
| Aria | `deadline := time.now() + 30m` |

**Parse a date string:**

| Language | Code |
|---|---|
| Go | `t, err := time.Parse("2006-01-02", input)` (magic reference date!) |
| Aria | `d := Date.parseIso(input)?` |

---

## Related Specifications

- `high-level-design.md` — `dur` literal syntax (`30s`, `5m`, `2h`) is defined in the core language spec
- `spec/stdlib-design.md` — the `time` module (Tier 1) provides `time.now()`, `time.sleep()`, timers, and tickers
