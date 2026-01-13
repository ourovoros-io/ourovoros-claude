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
    └── rust-idiomatic-patterns ← Full idioms reference
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
| `rust-dependency-audit` | cargo audit, licenses, supply chain |

#### Development Workflows
| Skill | When to Use |
|-------|-------------|
| `rust-testing` | TDD, proptest, mocking, async tests |
| `rust-async-patterns` | Tokio, spawn, channels, deadlocks |
| `rust-performance` | Profiling, benchmarks, optimization |

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

### rust-dependency-audit

Supply chain security:
- cargo audit
- cargo deny
- cargo vet
- License compliance
- Dependency evaluation

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
