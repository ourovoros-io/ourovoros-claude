---
name: rust-development
description: Use when writing, reviewing, or debugging Rust code. This is the primary skill for Rust development - covers common patterns and routes to specialized skills for security, testing, async, performance, and more.
---

# Rust Development

## Overview

Your primary guide for Rust development. Covers the essential patterns for writing idiomatic, safe, and maintainable Rust code. For deep dives, use the specialized skills referenced below.

## Quick Patterns

### Iterators Over Loops
```rust
// Instead of: for i in 0..v.len() { v[i] }
let result: Vec<_> = items.iter().filter(|x| x.valid).map(|x| x.name).collect();
```

### Error Handling
```rust
// Use ? for propagation
let content = std::fs::read_to_string(path)?;
let config: Config = toml::from_str(&content)?;
Ok(config)
```

### Option Handling
```rust
let value = opt.unwrap_or(default);
let value = opt.unwrap_or_else(|| expensive());
let mapped = opt.map(|x| transform(x));
let result = opt.ok_or(Error::Missing)?;
```

### Borrowing
```rust
// Accept slices, not Vec/String
fn process(items: &[Item]) { }   // not &Vec<Item>
fn greet(name: &str) { }         // not &String
```

### Pattern Matching
```rust
match result {
    Ok(v) if v > 0 => process(v),
    Ok(_) => {},
    Err(e) => log::warn!("{e}"),
}
```

### Enums for State
```rust
enum State { Idle, Running { task: Task }, Error { msg: String } }
// Instead of: is_running: bool, error: Option<String>
```

## Common Fixes

| Problem | Fix |
|---------|-----|
| `for i in 0..len` | `.iter()` / `.iter_mut()` |
| `.clone()` everywhere | Use `&T` references |
| `if x.is_some() { x.unwrap() }` | `if let Some(x) = x` |
| `.unwrap()` in library | Return `Result`, use `?` |
| `&Vec<T>` parameter | `&[T]` slice |
| Manual error match | `?` operator |
| Bool flags for state | `enum` variants |
| `String + &str` | `format!()` |

## When to Use Specialized Skills

| Situation | Use Skill |
|-----------|-----------|
| Handling user input, auth, secrets, crypto | `rust-security-audit` |
| Reviewing `unsafe`, FFI, raw pointers | `rust-unsafe-audit` |
| Writing tests, TDD, mocking, proptest | `rust-testing` |
| Async/Tokio, deadlocks, channels, select! | `rust-async-patterns` |
| Designing error types, thiserror/anyhow | `rust-error-handling` |
| Profiling, benchmarks, optimization | `rust-performance` |
| cargo audit, licenses, dependencies | `rust-dependency-audit` |
| Deep nesting, unclear naming, refactoring | `rust-code-clarity` |
| Full idiomatic patterns reference | `rust-idiomatic-patterns` |

## Red Flags to Watch

| Code Smell | Risk | Action |
|------------|------|--------|
| `.unwrap()` on external input | Panic/DoS | Validate, return Result |
| `unsafe` block | Memory unsafety | Use `rust-unsafe-audit` |
| String interpolation in SQL | Injection | Parameterize queries |
| `sleep()` in async | Blocks runtime | Use `tokio::time::sleep` |
| Hardcoded secrets | Credential leak | Use env vars, `secrecy` |
| `Arc<Mutex<_>>` default | Overhead | `Rc<RefCell<_>>` if single-threaded |

## Essential Commands

```bash
cargo clippy              # Lint for idioms
cargo fmt                 # Format code
cargo test                # Run tests
cargo audit               # Security check
cargo doc --open          # Generate docs
```

## Clippy Lints to Enable

```toml
# Cargo.toml or .clippy.toml
[lints.clippy]
needless_clone = "warn"
manual_unwrap_or = "warn"
ptr_arg = "warn"           # &Vec -> &[]
single_match = "warn"
```
