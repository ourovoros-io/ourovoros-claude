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
| `#[allow(lint)]` | `#[expect(lint, reason = "...")]` |

## API Naming Conventions

Follow the Rust API Guidelines for method prefixes:

| Prefix | Cost | Ownership | Example |
|--------|------|-----------|---------|
| `as_` | Free | Borrowed -> Borrowed | `as_str()`, `as_bytes()` |
| `to_` | Expensive | Borrowed -> Owned | `to_string()`, `to_vec()` |
| `into_` | Variable | Owned -> Owned (consumes) | `into_inner()`, `into_vec()` |

**Getters:** no `get_` prefix for simple field access. Reserve `get_` for lookups with parameters (like `HashMap::get`).

**Iterators:** `iter()` -> `&T`, `iter_mut()` -> `&mut T`, `into_iter()` -> `T`

## Core Patterns

### For Loops vs Iterator Chains

Short, simple transforms (1-2 combinators) are fine as iterators. **Prefer `for` loops when the chain exceeds two combinators or involves side effects:**

```rust
// Fine as iterator: simple, two combinators
let valid_names: Vec<_> = users.iter()
    .filter(|u| u.is_active())
    .map(|u| &u.name)
    .collect();

// Better as for loop: complex logic, side effects, or 3+ combinators
let mut results = Vec::new();
for item in &items {
    if !item.is_valid() {
        continue;
    }
    let transformed = item.transform()?;
    if transformed.meets_criteria() {
        results.push(transformed);
    }
}
```

### let...else for Early Returns

```rust
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
struct UserId(u64);
struct AccountId(u64);
struct Amount(u64);

fn transfer(from: AccountId, to: AccountId, amount: Amount) { }
// transfer(amount, user_id, account_id);  // Won't compile
```

### Shadow Variables Through Transformations

```rust
let input = get_input();
let input = input.trim();
let input: Config = toml::from_str(input)?;
```

### Explicit Destructuring Over Wildcards

```rust
enum Command { Start, Stop, Pause, Resume }

// Name every variant -- adding Resume will produce a compile error
match cmd {
    Command::Start => start(),
    Command::Stop | Command::Pause | Command::Resume => {}
}
```

### Enums for State Machines

```rust
// NOT: bool flags
// struct Connection { is_connected: bool, is_authenticated: bool }

enum ConnectionState {
    Disconnected,
    Connected,
    Authenticated { user: User },
    Error { message: String },
}
```

### `#[non_exhaustive]` on Public Enums/Structs

```rust
// Library code: prevents downstream from exhaustive matching
// Allows adding variants without breaking change
#[non_exhaustive]
pub enum Error {
    NotFound,
    PermissionDenied,
}
```

### Function Parameter Types

```rust
fn process(items: &[String]) { }   // Not &Vec<String>
fn greet(name: &str) { }           // Not &String

// For ownership transfer
fn store(name: impl Into<String>) {
    let name: String = name.into();
}
```

### Cow for Maybe-Clone

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<str> {
    if input.contains('\t') {
        Cow::Owned(input.replace('\t', "    "))
    } else {
        Cow::Borrowed(input)  // no allocation
    }
}
```

### `#[expect]` Over `#[allow]` (Rust 1.81+)

```rust
// BAD: #[allow] silently persists even when the warning is fixed
#[allow(clippy::cast_possible_truncation)]
let x = big_value as u16;

// GOOD: #[expect] warns when the suppressed lint no longer fires
#[expect(
    clippy::cast_possible_truncation,
    reason = "value validated to fit in u16 above"
)]
let x = big_value as u16;
```

### Ownership and Borrowing

```rust
// NOT: unnecessary cloning
fn process(data: &Data) -> String {
    let owned = data.clone();
    owned.name.clone()
}

// Borrow what you need
fn process(data: &Data) -> &str {
    &data.name
}
```

### Error Propagation

```rust
fn read_config() -> Result<Config, Error> {
    let content = std::fs::read_to_string("config.toml")?;
    let config = toml::from_str(&content)?;
    Ok(config)
}
```

### Logging with tracing

```rust
tracing::info!(user_id, "processing user");
tracing::error!(?err, "operation failed");
tracing::debug!(shard_count = shards.len(), "distribution complete");
```

## Type System Patterns

### Typestate Pattern

Encode state machines in the type system for compile-time validation:

```rust
pub struct Draft;
pub struct Published;

pub struct Article<State> {
    title: String,
    _state: PhantomData<State>,
}

impl Article<Draft> {
    pub fn publish(self) -> Article<Published> {
        Article { title: self.title, _state: PhantomData }
    }
}

impl Article<Published> {
    pub fn url(&self) -> String { format!("/articles/{}", self.title) }
}
// article.url() only compiles on Published articles
```

### Sealed Traits

Prevent downstream implementations, allowing you to add methods without breaking changes:

```rust
mod sealed { pub trait Sealed {} }

pub trait Driver: sealed::Sealed {
    fn connect(&self) -> Connection;
}
```

### Extension Traits

Add methods to foreign types:

```rust
pub trait IteratorExt: Iterator {
    fn find_with_index<P>(&mut self, pred: P) -> Option<(usize, Self::Item)>
    where P: FnMut(&Self::Item) -> bool, Self: Sized;
}

impl<I: Iterator> IteratorExt for I { /* ... */ }
```

### Visibility Design

```rust
pub struct Config { /* ... */ }        // External API
pub(crate) fn validate() { /* ... */ } // Crate-internal
// Default: private. Only pub(crate) when another module needs it.
// Only pub when external consumers need it.
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `to_string()` on `&str` param | Unnecessary allocation | Accept `&str` directly |
| `Box<dyn Trait>` everywhere | Runtime cost | Use generics `impl Trait` |
| `Arc<Mutex<_>>` by default | Overhead | `Rc<RefCell<_>>` if single-threaded |
| Manual `Drop` impl | Usually wrong | Let compiler handle it |
| `pub` on everything | Leaky API | Start private, expose as needed |
| `matches!` macro | Misses new fields/variants | Explicit destructuring |
| Wildcard `_` in match | Misses new variants | Name all variants |
| `#[allow]` without reason | Stale suppressions | Use `#[expect(lint, reason = "...")]` |
| `Deref` as inheritance | Confusing method resolution | Explicit delegation |

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
clippy::allow_attributes          (forces #[expect] over #[allow])
clippy::uninlined_format_args
clippy::needless_pass_by_value
```
