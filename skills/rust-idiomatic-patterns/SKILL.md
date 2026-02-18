---
name: rust-idiomatic-patterns
description: Use when writing or reviewing Rust code that could benefit from idiomatic patterns. Use when code uses C-style loops, excessive cloning, manual error handling, or ignores Rust's ownership system.
---

# Rust Idiomatic Patterns

## Overview

Write Rust that leverages the language's strengths: ownership, pattern matching, enums, and zero-cost abstractions. Aligned with project CLAUDE.md conventions.

## When to Use

- Excessive `.clone()` calls
- Manual `if x.is_some() { x.unwrap() }` patterns
- Boolean flags instead of enums
- Ignoring `Result` with `let _ =`
- `matches!` macro instead of explicit destructuring
- Wildcard `_` catch-all in match arms
- `raw_x` / `parsed_x` variable naming instead of shadowing
- `println!` instead of `tracing`

**Not for:** Security review (use `rust-security-audit`), readability cleanup (use `rust-code-clarity`)

## Quick Reference

| Non-Idiomatic | Idiomatic |
|---------------|-----------|
| `if x.is_some() { x.unwrap() }` | `let...else` or `if let` |
| `matches!(x, Variant::A)` | Explicit match or `if let` |
| `_ => {}` catch-all | Name every variant |
| `.clone()` everywhere | Use references `&T` |
| `fn foo(v: &Vec<T>)` | `fn foo(v: &[T])` |
| `fn foo(s: &String)` | `fn foo(s: &str)` |
| `match x { Ok(v) => v, Err(e) => return Err(e) }` | `x?` |
| `bool` flags for state | `enum` with variants |
| `raw_input` / `parsed_input` | Shadow: `let input = parse(input);` |
| `u64` for domain IDs | Newtype: `struct UserId(u64)` |
| `println!("debug: {x}")` | `tracing::debug!("{x}")` |
| Iterator chain with `.filter().map().collect()` | `for` loop with mutable accumulator |

## Core Patterns

### For Loops Over Iterator Chains

```rust
// NON-IDIOMATIC (per project convention): long iterator chain
let result: Vec<_> = items
    .iter()
    .filter(|item| item.is_valid())
    .map(|item| item.transform())
    .collect();

// IDIOMATIC: for loop with mutable accumulator
let mut result = Vec::new();
for item in &items {
    if item.is_valid() {
        result.push(item.transform());
    }
}
```

Short, simple transforms (single `.map()` or `.filter()`) are fine as iterators. Prefer `for` loops when the chain exceeds two combinators or involves side effects.

### let...else for Early Returns

```rust
// NON-IDIOMATIC: nested if-let
fn process(input: Option<&str>) -> Result<Output, Error> {
    if let Some(value) = input {
        if let Ok(parsed) = value.parse::<u64>() {
            do_work(parsed)
        } else {
            Err(Error::InvalidNumber)
        }
    } else {
        Err(Error::MissingInput)
    }
}

// IDIOMATIC: let...else keeps happy path unindented
fn process(input: Option<&str>) -> Result<Output, Error> {
    let Some(value) = input else {
        return Err(Error::MissingInput);
    };
    let Ok(parsed) = value.parse::<u64>() else {
        return Err(Error::InvalidNumber);
    };
    do_work(parsed)
}
```

### Newtypes Over Primitives

```rust
// NON-IDIOMATIC: bare primitives lose meaning
fn transfer(from: u64, to: u64, amount: u64) { }
transfer(amount, user_id, account_id);  // Compiles! Args swapped.

// IDIOMATIC: newtypes prevent mixups at compile time
struct UserId(u64);
struct AccountId(u64);
struct Amount(u64);

fn transfer(from: AccountId, to: AccountId, amount: Amount) { }
// transfer(amount, user_id, account_id);  // Won't compile
```

### Shadow Variables Through Transformations

```rust
// NON-IDIOMATIC: prefixed names
let raw_input = get_input();
let trimmed_input = raw_input.trim();
let parsed_input: Config = toml::from_str(trimmed_input)?;

// IDIOMATIC: shadow through transformations
let input = get_input();
let input = input.trim();
let input: Config = toml::from_str(input)?;
```

### Explicit Destructuring Over Wildcards

```rust
enum Command { Start, Stop, Pause, Resume }

// NON-IDIOMATIC: wildcard hides new variants
match cmd {
    Command::Start => start(),
    _ => {}  // Adding Command::Resume won't warn
}

// IDIOMATIC: name every variant
match cmd {
    Command::Start => start(),
    Command::Stop | Command::Pause | Command::Resume => {}
}

// NON-IDIOMATIC: matches! macro hides variant changes
if matches!(cmd, Command::Start | Command::Stop) { handle(); }

// IDIOMATIC: explicit match
match cmd {
    Command::Start | Command::Stop => handle(),
    Command::Pause | Command::Resume => {}
}
```

### Enums for State Machines

```rust
// NON-IDIOMATIC: boolean flags
struct Connection {
    is_connected: bool,
    is_authenticated: bool,
    error_message: Option<String>,
}

// IDIOMATIC: enum with data in variants
enum ConnectionState {
    Disconnected,
    Connected,
    Authenticated { user: User },
    Error { message: String },
}
```

### Ownership and Borrowing

```rust
// NON-IDIOMATIC: unnecessary cloning
fn process(data: &Data) -> String {
    let owned = data.clone();
    owned.name.clone()
}

// IDIOMATIC: borrow what you need
fn process(data: &Data) -> &str {
    &data.name
}
```

### Error Propagation

```rust
// NON-IDIOMATIC: manual match
fn read_config() -> Result<Config, Error> {
    let content = match std::fs::read_to_string("config.toml") {
        Ok(c) => c,
        Err(e) => return Err(e.into()),
    };
    Ok(toml::from_str(&content)?)
}

// IDIOMATIC: ? operator
fn read_config() -> Result<Config, Error> {
    let content = std::fs::read_to_string("config.toml")?;
    let config = toml::from_str(&content)?;
    Ok(config)
}
```

### Option/Result Combinators

```rust
let value = opt.unwrap_or(0);
let value = opt.unwrap_or_else(|| expensive_default());
let value = opt.unwrap_or_default();
let value = opt.ok_or(Error::Missing)?;
let doubled = opt.map(|x| x * 2);
let result = opt.and_then(|x| validate(x));
```

### Function Parameter Types

```rust
// NON-IDIOMATIC: overly specific
fn process(items: &Vec<String>) { }
fn greet(name: &String) { }

// IDIOMATIC: accept slices
fn process(items: &[String]) { }   // Accepts Vec, array, slice
fn greet(name: &str) { }           // Accepts String, &str, Cow<str>

// For ownership transfer
fn store(name: impl Into<String>) {
    let name: String = name.into();
}
```

### Logging with tracing

```rust
// NON-IDIOMATIC
println!("Processing user {user_id}");
eprintln!("Error: {err}");

// IDIOMATIC
tracing::info!(user_id, "processing user");
tracing::error!(?err, "operation failed");
tracing::debug!(shard_count = shards.len(), "distribution complete");
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `to_string()` on `&str` param | Unnecessary allocation | Accept `impl AsRef<str>` |
| `Box<dyn Trait>` everywhere | Runtime cost | Use generics `impl Trait` |
| `Arc<Mutex<_>>` by default | Overhead | `Rc<RefCell<_>>` if single-threaded |
| Manual `Drop` impl | Usually wrong | Let compiler handle it |
| `pub` on everything | Leaky API | Start private, expose as needed |
| `matches!` macro | Misses new fields/variants | Explicit destructuring |
| Wildcard `_` in match | Misses new variants | Name all variants |
| Long iterator chains | Hard to read/debug | `for` loop with accumulator |

## Clippy Alignment

These lints enforce the patterns above:

```
clippy::wildcard_enum_match_arm
clippy::match_wildcard_for_single_variants
clippy::needless_clone
clippy::manual_unwrap_or
clippy::ptr_arg
clippy::print_stdout
clippy::print_stderr
clippy::dbg_macro
```
