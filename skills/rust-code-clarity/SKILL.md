---
name: rust-code-clarity
description: Use when Rust code is hard to read, has deep nesting, unclear naming, or overly complex logic. Use when refactoring for maintainability without changing behavior.
---

# Rust Code Clarity

## Overview

Simplify Rust code for readability and maintainability. Focus on reducing cognitive load through clear structure, meaningful names, and explicit logic flow.

## When to Use

- Functions approaching 100 lines (hard limit from CLAUDE.md)
- Nesting deeper than 3 levels
- Unclear variable or function names
- Complex match expressions
- Duplicated logic patterns
- Hard-to-follow control flow

**Not for:** Security issues (use `rust-security-audit`), Rust idiom adoption (use `rust-idiomatic-patterns`)

## Quick Reference

| Problem | Solution |
|---------|----------|
| Deep nesting | `let...else`, guard clauses, early returns |
| Long functions | Extract helper functions (aim for <=100 lines) |
| Complex conditions | Named booleans, extract predicates |
| Unclear names | `process_user_input` not `proc` |
| Magic numbers | Named constants |
| Repeated patterns | Extract to functions/traits |
| Variable name chains | Shadow through transformations |

## Core Techniques

### let...else for Flat Control Flow

```rust
// UNCLEAR: Deep nesting
fn process(user: Option<User>) -> Result<Output, Error> {
    if let Some(user) = user {
        if user.is_active {
            if user.has_permission("write") {
                Ok(do_work(&user))
            } else {
                Err(Error::NoPermission)
            }
        } else {
            Err(Error::Inactive)
        }
    } else {
        Err(Error::NoUser)
    }
}

// CLEAR: let...else + guard clauses
fn process(user: Option<User>) -> Result<Output, Error> {
    let Some(user) = user else {
        return Err(Error::NoUser);
    };
    if !user.is_active {
        return Err(Error::Inactive);
    }
    if !user.has_permission("write") {
        return Err(Error::NoPermission);
    }
    Ok(do_work(&user))
}
```

### if let Chains (Edition 2024)

```rust
// Before: nested if-let
if let Some(result) = val {
    if let Ok(data) = result {
        if data.is_valid() {
            handle(data);
        }
    }
}

// Edition 2024: chained conditions
if let Some(Ok(data)) = val
    && data.is_valid()
{
    handle(data);
}
```

### Shadow Variables for Transformations

```rust
// UNCLEAR: name proliferation
let raw_config = read_file(path)?;
let parsed_config = toml::from_str(&raw_config)?;
let validated_config = validate(parsed_config)?;

// CLEAR: shadow through transformations
let config = read_file(path)?;
let config = toml::from_str(&config)?;
let config = validate(config)?;
```

### Named Conditions

```rust
// UNCLEAR
if user.role == Role::Admin || (user.role == Role::Mod && post.author_id == user.id) {
    delete_post(post);
}

// CLEAR
let is_admin = user.role == Role::Admin;
let is_mod_deleting_own = user.role == Role::Mod && post.author_id == user.id;
let can_delete = is_admin || is_mod_deleting_own;

if can_delete {
    delete_post(post);
}
```

### Extract Functions

```rust
// UNCLEAR: Everything in one function
fn handle_request(req: Request) -> Response {
    // 20 lines of validation
    // 30 lines of processing
    // 15 lines of formatting
}

// CLEAR: Single responsibility
fn handle_request(req: Request) -> Result<Response, Error> {
    let validated = validate_request(&req)?;
    let result = process_data(&validated)?;
    Ok(format_response(result))
}
```

### Meaningful Names

```rust
// UNCLEAR
fn proc(d: &[u8], f: bool) -> Vec<u8> { /* ... */ }

// CLEAR
fn transform_bytes(input: &[u8], uppercase: bool) -> Vec<u8> { /* ... */ }
```

### Named Constants

```rust
const CLIENT_ERROR_MIN: u16 = 400;
const CLIENT_ERROR_MAX: u16 = 500;
const RETRY_DELAY: Duration = Duration::from_secs(1);

if response.status >= CLIENT_ERROR_MIN && response.status < CLIENT_ERROR_MAX {
    retry_with_backoff(RETRY_DELAY);
}
```

### Visibility as Documentation

```rust
pub struct Server { /* ... */ }          // Part of public API
pub(crate) fn validate() { /* ... */ }   // Shared between modules
fn helper() { /* ... */ }                // Private implementation detail
```

Starting private and promoting to `pub(crate)` or `pub` as needed communicates intent clearly.

## Simplification Checklist

1. **Names**: Can someone understand intent without reading implementation?
2. **Length**: Functions under 100 lines? Aim for 40-60 as default.
3. **Nesting**: Max 3 levels of indentation? Use `let...else` to flatten.
4. **Single responsibility**: Each function does one thing?
5. **Magic values**: All literals named or obvious?
6. **Shadowing**: Using `let x = transform(x)` instead of `raw_x`/`new_x`?
7. **DRY**: No copy-pasted logic blocks?
8. **Visibility**: Is everything as private as possible?

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| Over-abstracting | Harder to follow than original | Only abstract repeated patterns |
| Tiny functions | 10 one-line functions obscure flow | Balance granularity |
| Generic names | `data`, `result`, `temp` | Name by purpose |
| Comment-heavy code | Usually means unclear code | Refactor to be self-documenting |
| Premature optimization | Clever but unreadable | Optimize only proven bottlenecks |
| Name chains | `raw_x`, `parsed_x`, `final_x` | Shadow: `let x = parse(x)` |
| Deep nesting | Hard to follow happy path | `let...else`, early return |

## When to Stop

Don't simplify if:
- Code is already clear to a Rust newcomer
- Simplification adds more lines than it removes
- It would break existing tests
- It's a well-known Rust pattern (even if verbose)
