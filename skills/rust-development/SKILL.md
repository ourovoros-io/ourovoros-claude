---
name: rust-development
description: Use when writing, reviewing, or debugging Rust code. This is the primary skill for Rust development - covers common patterns and routes to specialized skills for security, testing, async, performance, and more.
---

# Rust Development

## Overview

Primary routing guide for Rust development. Identifies the right specialized skill for each task. When multiple concerns overlap, invoke skills in priority order: security -> correctness -> clarity -> performance.

## Skill Router

| Situation | Skill | Trigger |
|-----------|-------|---------|
| Async code, Tokio, channels, select!, concurrency | `rust-async-patterns` | Any async fn, spawn, channel, runtime config |
| Designing error types, thiserror/anyhow choice | `rust-error-handling` | New module, inconsistent error handling |
| Writing tests, TDD, mocking, property testing | `rust-testing` | New code, bug fix, test infrastructure |
| Profiling, benchmarks, reducing allocations | `rust-performance` | Measurably slow code, data structure choice |
| Ownership, enums, let...else, newtypes | `rust-idiomatic-patterns` | C-style code, excessive cloning, bool flags |
| Readability, naming, nesting, refactoring | `rust-code-clarity` | Hard-to-read code, deep nesting, long functions |
| User input, auth, secrets, crypto, network | `rust-security-audit` | External data handling, credentials |
| unsafe blocks, FFI, raw pointers, transmute | `rust-unsafe-audit` | Any unsafe code |
| cargo audit, licenses, supply chain | `rust-dependency-audit` | Adding deps, security gates, cargo update |

## Red Flags

These warrant immediate attention -- invoke the corresponding skill:

| Code Smell | Risk | Skill |
|------------|------|-------|
| `.unwrap()` on external input | Panic/DoS | `rust-security-audit` |
| `unsafe` block | Memory unsafety | `rust-unsafe-audit` |
| String interpolation in SQL/commands | Injection | `rust-security-audit` |
| `std::thread::sleep()` in async | Blocks runtime | `rust-async-patterns` |
| Holding lock across `.await` | Deadlock | `rust-async-patterns` |
| Hardcoded secrets | Credential leak | `rust-security-audit` |
| `clone()` in hot loop | Performance | `rust-performance` |
| `matches!` macro | Misses field changes | `rust-idiomatic-patterns` |
| Wildcard `_` in match arms | Misses new variants | `rust-idiomatic-patterns` |
| `println!` / `eprintln!` | Not structured logging | `rust-idiomatic-patterns` |
| `#[allow]` without reason | Stale lint suppression | `rust-idiomatic-patterns` |
| Unbounded channel | OOM under load | `rust-async-patterns` |
| `build.rs` in new dependency | Supply chain risk | `rust-dependency-audit` |
| Integer arithmetic on untrusted input | Overflow in release | `rust-security-audit` |

## Project Standards

These apply across all skills -- defined in CLAUDE.md:

- **Loops**: Prefer `for` loops with mutable accumulators over long iterator chains (2+ combinators)
- **Matching**: No wildcard matches; no `matches!` macro -- use explicit destructuring
- **Early returns**: Use `let...else` to keep happy path unindented
- **Types**: Newtypes over primitives (`UserId(u64)` not `u64`)
- **Variables**: Shadow through transformations (no `raw_x`/`parsed_x` prefixes)
- **State**: Enums for state machines, not boolean flags
- **Errors**: `thiserror` for libraries, `anyhow` for applications
- **Logging**: `tracing` (`error!`/`warn!`/`info!`/`debug!`), never `println!`
- **Panics**: No `.unwrap()` -- use `?`, `unwrap_or`, or `let...else`
- **Lint suppression**: `#[expect(lint, reason = "...")]` not `#[allow]`

## Essential Commands

```bash
cargo clippy --all-targets --all-features -- -D warnings   # Lint
cargo fmt --all -- --check                                   # Format check
cargo nextest run --workspace                                # Test (recommended)
cargo test --doc                                             # Doc tests (nextest can't)
cargo deny check                                             # Deps audit
cargo +nightly miri test                                     # UB detection
cargo +nightly careful test                                  # Extra runtime checks
```
