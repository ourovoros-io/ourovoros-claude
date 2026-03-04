---
name: rfr-project-structure
description: Use when organizing Rust crates, modules, and workspaces. Use when deciding on feature flags, conditional compilation, module visibility, crate boundaries, or versioning strategy. Use when structuring a multi-crate project.
---

# Rust for Rustaceans — Ch 5: Project Structure

## Crate Types

| Type | Cargo.toml | Use case |
|------|-----------|----------|
| `lib` | `[lib]` | Reusable library |
| `bin` | `[[bin]]` | Executable |
| `proc-macro` | `proc-macro = true` in `[lib]` | Procedural macros |
| `cdylib` | `crate-type = ["cdylib"]` | C-compatible shared library |
| `staticlib` | `crate-type = ["staticlib"]` | C-compatible static library |

- A crate can have both a `lib` and multiple `bin` targets
- Proc-macro crates can ONLY export procedural macros — no other items
- Split proc-macros into separate crates; they can't share items with the main library

## Module System

### Module Tree
```
src/
  lib.rs          // crate root
  config.rs       // mod config
  server/
    mod.rs        // mod server (or server.rs at parent level)
    handler.rs    // mod server::handler
    state.rs      // mod server::state
```

### Visibility

| Keyword | Scope |
|---------|-------|
| (none) | Private to current module and its children |
| `pub(self)` | Same as private (explicit) |
| `pub(super)` | Visible to parent module |
| `pub(crate)` | Visible anywhere in the crate |
| `pub(in path::to::module)` | Visible in a specific ancestor |
| `pub` | Visible to all dependents |

### Module Design Rules
1. **Start private**, promote to `pub(crate)`, then `pub` only when needed
2. **Re-export** important types at the crate root for convenience
3. **One module per concept** — don't dump everything in `lib.rs`
4. **`mod.rs` vs file**: prefer `server.rs` + `server/` directory over `server/mod.rs` (modern style)
5. **Tests in the same file** as the code they test (`mod tests`)

### Re-exports
```rust
// lib.rs — re-export key types for convenience
pub use config::Config;
pub use error::Error;

mod config;
mod error;
mod internal; // not re-exported, stays private
```

## Workspaces

### When to Use
- Multiple related crates in one repo
- Shared dependencies (one `Cargo.lock`)
- Unified CI/CD pipeline

### Structure
```toml
# Root Cargo.toml
[workspace]
members = [
    "crates/core",
    "crates/server",
    "crates/proto",
]

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }

[workspace.lints.clippy]
pedantic = { level = "warn", priority = -1 }
```

```toml
# crates/core/Cargo.toml
[dependencies]
tokio = { workspace = true }
serde = { workspace = true }

[lints]
workspace = true
```

### Workspace Best Practices
- **Share dependency versions** via `[workspace.dependencies]`
- **Share lint config** via `[workspace.lints]`
- **Share Rust edition** via `[workspace.package]`
- Keep the root `Cargo.toml` as workspace-only (no `[package]`)
- Each member crate has its own version for independent publishing

## Feature Flags

### Defining Features
```toml
[features]
default = ["std"]
std = []
serde = ["dep:serde"]     # Optional dependency
full = ["std", "serde"]    # Composite feature

[dependencies]
serde = { version = "1", optional = true }
```

### Feature Design Rules
1. **Features must be additive** — enabling a feature should never remove functionality
2. **No feature should break another** — `cargo test --all-features` must pass
3. **Use `dep:` syntax** for optional deps — prevents implicit feature names
4. **Default features** should include the common case (usually `std`)
5. **Feature-gate heavy deps** — `serde`, `async`, logging frameworks

### Using Features in Code
```rust
// Conditional compilation on feature
#[cfg(feature = "serde")]
use serde::{Serialize, Deserialize};

#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub struct Config {
    pub port: u16,
}

// Conditional module
#[cfg(feature = "async")]
pub mod async_client;
```

## Conditional Compilation

### `cfg` Attributes

| Attribute | Checks |
|-----------|--------|
| `#[cfg(feature = "x")]` | Feature flag enabled |
| `#[cfg(target_os = "linux")]` | Target OS |
| `#[cfg(target_arch = "x86_64")]` | Target architecture |
| `#[cfg(test)]` | Running tests |
| `#[cfg(debug_assertions)]` | Debug build |
| `#[cfg(unix)]` / `#[cfg(windows)]` | OS family |
| `#[cfg(any(A, B))]` | A or B |
| `#[cfg(all(A, B))]` | A and B |
| `#[cfg(not(A))]` | Not A |

### `cfg` vs `cfg_attr`
```rust
// cfg: include/exclude the item entirely
#[cfg(target_os = "linux")]
fn linux_only() { ... }

// cfg_attr: conditionally add an attribute
#[cfg_attr(test, derive(PartialEq))]
struct Packet { ... }
```

### Checking cfg at Build Time
```rust
// build.rs
fn main() {
    println!("cargo::rustc-check-cfg=cfg(custom_flag)");
    if some_condition() {
        println!("cargo::rustc-cfg=custom_flag");
    }
}
```

## Versioning

### Semantic Versioning for Rust
- **Major** (1.0 → 2.0): breaking changes
- **Minor** (1.0 → 1.1): new features, backward compatible
- **Patch** (1.0.0 → 1.0.1): bug fixes only

### What Counts as Breaking in Rust

| Change | Breaking? |
|--------|-----------|
| Adding a public item | No |
| Removing a public item | **Yes** |
| Adding a required method to a trait | **Yes** |
| Adding a default method to a trait | No (usually) |
| Adding a variant to a `#[non_exhaustive]` enum | No |
| Adding a variant to an exhaustive enum | **Yes** |
| Adding a field to a `#[non_exhaustive]` struct | No |
| Adding a required field to a struct | **Yes** |
| Changing a function signature | **Yes** |
| Tightening a generic bound | **Yes** |
| Loosening a generic bound | No (usually) |
| Changing `pub` to `pub(crate)` | **Yes** |
| Adding a trait impl | No (usually, but can conflict) |
| Bumping MSRV | Minor (by convention) |

### Pre-1.0 Versioning
- `0.x.y`: Minor bumps (0.1 → 0.2) are treated as major
- `0.x.y` → `0.x.z`: patch, backward compatible
- Stay at 0.x until API stabilizes

## MSRV (Minimum Supported Rust Version)

- Declare in `Cargo.toml`: `rust-version = "1.70"`
- Test in CI with the declared MSRV
- Bumping MSRV is a minor version change (by convention)
- Only bump when you need a feature from a newer compiler
- Consider your users: embedded/distro Rust may lag behind

## Common Mistakes

1. **Enormous `lib.rs`** — split into focused modules
2. **`pub` on internal helpers** — use `pub(crate)` until external use is proven
3. **Non-additive features** — features that disable functionality break `--all-features`
4. **Missing `#[non_exhaustive]`** — every public enum should have it for future-proofing
5. **Workspace members with wildcard deps** — use exact versions or workspace-level pinning
6. **Circular dependencies** — design crate boundaries to form a DAG
7. **Proc-macro in the same crate as library code** — must be a separate crate
