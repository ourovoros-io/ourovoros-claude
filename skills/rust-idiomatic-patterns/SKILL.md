---
name: rust-idiomatic-patterns
description: Use when writing or reviewing Rust code that could benefit from idiomatic patterns. Use when code uses C-style loops, excessive cloning, manual error handling, or ignores Rust's ownership system.
---

# Rust Idiomatic Patterns

## Overview

Transform Rust code to leverage the language's strengths: ownership, iterators, pattern matching, and zero-cost abstractions. Idiomatic Rust is safer, more readable, and often faster.

## When to Use

- C-style `for i in 0..len` loops
- Excessive `.clone()` calls
- Manual `if x.is_some() { x.unwrap() }` patterns
- Ignoring `Result` with `let _ =`
- String concatenation with `+`
- Boolean flags instead of enums

**Not for:** Security review (use `rust-security-audit`), simplification (use `rust-code-clarity`)

## Quick Reference

| Non-Idiomatic | Idiomatic |
|---------------|-----------|
| `for i in 0..v.len()` | `for item in &v` or `v.iter()` |
| `if x.is_some() { x.unwrap() }` | `if let Some(x) = x` |
| `opt.is_some() ? opt.unwrap() : default` | `opt.unwrap_or(default)` |
| `opt.map(\|x\| x).unwrap_or(default)` | `opt.unwrap_or(default)` |
| `.clone()` everywhere | Use references `&T` |
| `fn foo(v: &Vec<T>)` | `fn foo(v: &[T])` (slice) |
| `fn foo(s: &String)` | `fn foo(s: &str)` |
| `match x { Ok(v) => v, Err(e) => return Err(e) }` | `x?` |
| `String + &str + &str` | `format!()` or `push_str` |
| `bool` flags for state | `enum` with variants |
| `Vec::new(); v.push(); v.push()` | `vec![a, b]` |
| Nested `if let` | `match` or `?` chains |

## Core Patterns

### Iterators Over Loops

```rust
// NON-IDIOMATIC
let mut result = Vec::new();
for i in 0..items.len() {
    if items[i].is_valid() {
        result.push(items[i].transform());
    }
}

// IDIOMATIC
let result: Vec<_> = items
    .iter()
    .filter(|item| item.is_valid())
    .map(|item| item.transform())
    .collect();
```

### Ownership and Borrowing

```rust
// NON-IDIOMATIC: Unnecessary cloning
fn process(data: &Data) -> String {
    let owned = data.clone();  // Why clone?
    owned.name.clone()
}

// IDIOMATIC: Borrow what you need
fn process(data: &Data) -> &str {
    &data.name
}
```

### Error Propagation

```rust
// NON-IDIOMATIC
fn read_config() -> Result<Config, Error> {
    let content = match std::fs::read_to_string("config.toml") {
        Ok(c) => c,
        Err(e) => return Err(e.into()),
    };
    let config = match toml::from_str(&content) {
        Ok(c) => c,
        Err(e) => return Err(e.into()),
    };
    Ok(config)
}

// IDIOMATIC
fn read_config() -> Result<Config, Error> {
    let content = std::fs::read_to_string("config.toml")?;
    let config = toml::from_str(&content)?;
    Ok(config)
}
```

### Enums for State

```rust
// NON-IDIOMATIC
struct Connection {
    is_connected: bool,
    is_authenticated: bool,
    error_message: Option<String>,
}

// IDIOMATIC
enum ConnectionState {
    Disconnected,
    Connected,
    Authenticated { user: User },
    Error { message: String },
}
```

### Pattern Matching

```rust
// NON-IDIOMATIC
if result.is_ok() {
    let value = result.unwrap();
    if value > 0 {
        process(value);
    }
}

// IDIOMATIC
if let Ok(value) = result {
    if value > 0 {
        process(value);
    }
}

// EVEN BETTER with guards
match result {
    Ok(value) if value > 0 => process(value),
    Ok(_) => {},  // Zero or negative
    Err(e) => log::warn!("Failed: {e}"),
}
```

### Option/Result Combinators

```rust
// NON-IDIOMATIC: Manual unwrap with default
let value = if opt.is_some() { opt.unwrap() } else { 0 };

// IDIOMATIC: unwrap_or
let value = opt.unwrap_or(0);

// For expensive defaults, use unwrap_or_else
let value = opt.unwrap_or_else(|| compute_default());

// For Default trait
let value = opt.unwrap_or_default();

// Transform with map
let doubled = opt.map(|x| x * 2);  // Option<i32> -> Option<i32>

// Chain operations with and_then (flatMap)
let result = opt
    .and_then(|x| validate(x))     // Option -> Option
    .map(|x| transform(x));

// Convert Option to Result
let result = opt.ok_or(Error::Missing)?;
```

### Function Parameter Types

```rust
// NON-IDIOMATIC: Overly specific
fn process(items: &Vec<String>) { ... }
fn greet(name: &String) { ... }

// IDIOMATIC: Accept slices for flexibility
fn process(items: &[String]) { ... }  // Accepts Vec, array, slice
fn greet(name: &str) { ... }          // Accepts String, &str, Cow<str>

// For ownership transfer, use generics
fn store(name: impl Into<String>) {
    let name: String = name.into();  // Accepts String or &str
}
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `to_string()` on `&str` param | Unnecessary allocation | Accept `impl AsRef<str>` |
| `Box<dyn Trait>` everywhere | Runtime cost | Use generics `impl Trait` |
| `Arc<Mutex<_>>` by default | Overhead | Start with `Rc<RefCell<_>>` if single-threaded |
| Manual `Drop` impl | Usually wrong | Let compiler handle it |
| `pub` on everything | Leaky API | Start private, expose as needed |

## Clippy Alignment

Run `cargo clippy` and address:
- `clippy::needless_clone`
- `clippy::manual_map`
- `clippy::manual_unwrap_or`
- `clippy::iter_nth_zero`
- `clippy::single_match`
