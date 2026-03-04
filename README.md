# Rust Skills for Claude Code

A collection of Rust-focused skills (plugins) for Claude Code that enhance AI-assisted Rust development with idiomatic patterns, security practices, and specialized workflows.

## Architecture: Hub and Spoke

This collection uses a **hub-and-spoke model** for efficient context usage:

```
rust-development (hub)          ← Loads for all Rust work
    │
    ├── rust-security-audit     ← Auth, secrets, crypto
    ├── rust-unsafe-audit       ← unsafe, FFI, raw pointers
    ├── rust-testing            ← TDD, proptest, mocking
    ├── rust-async-patterns     ← Tokio, channels, select!
    ├── rust-error-handling     ← thiserror, anyhow
    ├── rust-performance        ← Benchmarks, profiling
    ├── rust-dependency-audit   ← cargo audit, licenses
    ├── rust-code-clarity       ← Refactoring, naming
    ├── rust-idiomatic-patterns ← Full idioms reference
    └── rust-nomicon            ← Unsafe Rust mastery

rfr-* (Rust for Rustaceans)     ← Deep-dive reference skills
    │
    ├── rfr-foundations         ← Ownership, lifetimes, drop order
    ├── rfr-types               ← Layout, alignment, traits, Sized
    ├── rfr-designing-interfaces ← API design, signatures, builders
    ├── rfr-error-handling      ← Error types, thiserror/anyhow
    ├── rfr-project-structure   ← Crates, modules, workspaces
    ├── rfr-testing             ← Tests, proptest, fuzzing, Miri
    ├── rfr-macros              ← macro_rules!, proc macros, syn/quote
    ├── rfr-async-programming   ← Futures, Pin, Waker, async/await
    ├── rfr-unsafe-code         ← Raw pointers, MaybeUninit, soundness
    ├── rfr-concurrency         ← Send/Sync, atomics, deadlocks
    ├── rfr-ffi                 ← extern "C", bindgen, repr(C)
    └── rfr-ecosystem           ← Type state, newtype, RAII patterns
```

### How It Works

1. **`rust-development`** (the hub) loads automatically for any Rust work
2. It covers **80% of common patterns** in ~100 lines
3. It **routes to specialized skills** when you need depth
4. Specialized skills load **on-demand** when triggered

### Benefits

- **Fast**: Common patterns available immediately
- **Efficient**: Only loads what you need
- **Deep**: Full reference available when required
- **Maintainable**: Each skill focuses on one area

## Installation

Copy the skills to your Claude Code skills directory:

```bash
cp -r skills/* ~/.claude/skills/
```

Or symlink for easier updates:

```bash
ln -s $(pwd)/skills/* ~/.claude/skills/
```

## Skills Overview

### Hub Skill (Always Loads for Rust)

| Skill | Purpose |
|-------|---------|
| `rust-development` | Essential patterns, routes to specialists |

### Specialized Skills (Load On-Demand)

#### Code Quality
| Skill | When to Use |
|-------|-------------|
| `rust-idiomatic-patterns` | Full reference for Rust idioms, ownership, iterators |
| `rust-code-clarity` | Refactoring, reducing nesting, naming conventions |
| `rust-error-handling` | Designing error types, thiserror vs anyhow |

#### Security & Safety
| Skill | When to Use |
|-------|-------------|
| `rust-security-audit` | User input, auth, secrets, crypto, injection |
| `rust-unsafe-audit` | Reviewing unsafe blocks, FFI, raw pointers |
| `rust-nomicon` | Unsafe Rust mastery, variance, drop check, UB prevention |
| `rust-dependency-audit` | cargo audit, licenses, supply chain |

#### Development Workflows
| Skill | When to Use |
|-------|-------------|
| `rust-testing` | TDD, proptest, mocking, async tests |
| `rust-async-patterns` | Tokio, spawn, channels, deadlocks |
| `rust-performance` | Profiling, benchmarks, optimization |

### Rust for Rustaceans Skills (Deep-Dive Reference)

Based on the book *Rust for Rustaceans*. Load these for in-depth coverage of specific Rust topics.

| Skill | When to Use |
|-------|-------------|
| `rfr-foundations` | Ownership, borrowing, lifetimes, drop order, interior mutability |
| `rfr-types` | Type layout, alignment, `Sized`, traits, `PhantomData`, marker traits |
| `rfr-designing-interfaces` | Public API design, function signatures, standard traits, builders |
| `rfr-error-handling` | Error types, `thiserror`/`anyhow`, `Display`, `?` operator, when to panic |
| `rfr-project-structure` | Crates, modules, workspaces, features, conditional compilation |
| `rfr-testing` | Unit/integration/doc tests, proptest, fuzzing, Miri, benchmarks |
| `rfr-macros` | `macro_rules!`, procedural macros, `syn`/`quote`, hygiene |
| `rfr-async-programming` | Futures, `Pin`, `Waker`, async/await, spawn, `select!`, cancellation |
| `rfr-unsafe-code` | unsafe blocks, raw pointers, `MaybeUninit`, `UnsafeCell`, soundness |
| `rfr-concurrency` | `Send`/`Sync`, `Mutex`, channels, atomics, ordering, deadlock prevention |
| `rfr-ffi` | `extern "C"`, `CStr`/`CString`, `repr(C)`, bindgen/cbindgen |
| `rfr-ecosystem` | Type state, newtype, extension traits, RAII, iterator patterns |

## Usage Examples

### Automatic (Hub)
```
User: "Review this Rust function"
Claude: [Loads rust-development, applies common patterns]
```

### Explicit Routing
```
User: "I need to audit the authentication code"
Claude: [Loads rust-security-audit for deep security patterns]
```

### Direct Invocation
```
User: "Use rust-testing to help me write tests"
Claude: [Loads rust-testing directly]
```

## Skill Details

### rust-development (Hub)

The primary skill for all Rust work. Covers:
- Iterator patterns
- Error handling with `?`
- Option combinators
- Borrowing best practices
- Pattern matching
- Common fixes table
- Routing to specialized skills

### rust-idiomatic-patterns

Full reference for Rust idioms:
- Iterators over loops
- Ownership and borrowing
- Error propagation
- Enums for state machines
- Option/Result combinators
- Function parameter types

### rust-code-clarity

Simplify code for maintainability:
- Early returns and guard clauses
- Named conditions
- Function extraction
- Meaningful naming
- When to stop simplifying

### rust-error-handling

Design clear error types:
- `thiserror` for libraries
- `anyhow` for applications
- Structured error variants
- Error conversion patterns

### rust-security-audit

Application-level security:
- Input validation
- Password hashing (argon2)
- Secrets handling
- Constant-time comparison
- User enumeration prevention
- SQL injection prevention

### rust-unsafe-audit

Low-level safety review:
- Safety documentation format
- Raw pointer handling
- FFI boundary patterns
- Transmute alternatives
- Miri and cargo-geiger

### rust-testing

Effective testing patterns:
- Test organization
- Async testing
- Property-based testing
- Mocking with mockall
- Floating-point comparisons
- Panic testing

### rust-async-patterns

Correct async Rust:
- spawn vs spawn_blocking
- select! for racing
- Channel patterns
- Shared state
- Cancellation safety

### rust-performance

Systematic optimization:
- Criterion benchmarks
- Flamegraph profiling
- Allocation reduction
- Data structure selection
- Release profile tuning

### rust-nomicon

Unsafe Rust mastery (Rustonomicon reference):
- Safe/unsafe contract and all causes of UB
- Data layout (`repr(C)`, `repr(transparent)`, ZSTs, DSTs)
- Subtyping, variance, and `PhantomData`
- Drop check and `#[may_dangle]`
- Type conversions (`as`, `transmute`, coercions)
- Uninitialized memory and `MaybeUninit`
- OBRM (constructors, destructors, leaking)
- Exception safety (guard pattern, poisoning)
- Concurrency (`Send`/`Sync`, atomics, orderings)
- FFI (extern functions, opaque types, callbacks)
- `#![no_std]` essentials

### rust-dependency-audit

Supply chain security:
- cargo audit
- cargo deny
- cargo vet
- License compliance
- Dependency evaluation

### rfr-foundations

Ownership, borrowing, and lifetime mechanics:
- Memory model and ownership semantics
- Borrowing rules and lifetime annotations
- Drop order and `ManuallyDrop`
- Interior mutability (`Cell`, `RefCell`, `UnsafeCell`)

### rfr-types

Type system deep dive:
- Type layout and alignment
- `Sized` and dynamically sized types
- Trait objects and vtables
- `PhantomData` and marker traits

### rfr-designing-interfaces

Public API design patterns:
- Function signatures and parameter types
- Standard trait implementations
- Builder pattern
- Sealed traits and extension traits

### rfr-error-handling (rfr)

Error design in depth:
- `thiserror` for libraries, `anyhow` for applications
- `Display` and `Error` trait implementations
- `?` operator and error conversion
- When to panic vs return errors

### rfr-project-structure

Crate and module organization:
- Workspace layout and dependencies
- Feature flags and conditional compilation
- Module visibility and re-exports
- Documentation and `cfg` attributes

### rfr-testing (rfr)

Testing strategies:
- Unit, integration, and doc tests
- Property-based testing with proptest
- Fuzzing with cargo-fuzz
- Miri for UB detection
- Benchmarking with criterion

### rfr-macros

Macro development:
- `macro_rules!` patterns and pitfalls
- Procedural macros (derive, attribute, function-like)
- `syn` and `quote` for proc macros
- Hygiene and `cargo expand`

### rfr-async-programming

Async internals:
- `Future` trait and `Pin`/`Unpin`
- `Waker` mechanics
- async/await desugaring
- `spawn`, `select!`, `join!`
- Cancellation safety

### rfr-unsafe-code

Unsafe Rust correctness:
- Raw pointer safety invariants
- `MaybeUninit` and uninitialized memory
- `UnsafeCell` and interior mutability
- Soundness proofs and documentation

### rfr-concurrency

Concurrent programming:
- `Send` and `Sync` traits
- `Mutex`, `RwLock`, and poisoning
- Channel patterns (mpsc, crossbeam)
- Atomic operations and memory ordering
- Deadlock prevention strategies

### rfr-ffi

Foreign function interface:
- `extern "C"` declarations and ABI
- `CStr`/`CString` string handling
- `repr(C)` layout guarantees
- bindgen and cbindgen tooling
- Panic safety across FFI boundaries

### rfr-ecosystem

Design patterns and crate ecosystem:
- Type state pattern
- Newtype pattern
- Extension traits
- RAII and resource management
- Iterator protocol and custom iterators

## Contributing

### Adding a New Skill

1. Create `skills/skill-name/SKILL.md`
2. Use proper YAML frontmatter:
   ```yaml
   ---
   name: skill-name
   description: Use when [specific triggers]
   ---
   ```
3. Include Quick Reference table
4. Add Common Mistakes section
5. Provide practical code examples
6. Update the hub skill's routing table

### Guidelines

- Descriptions start with "Use when..." (triggers only, not workflow)
- Keep specialized skills focused on one area
- Cross-reference related skills
- Test with realistic scenarios before publishing

## License

MIT
