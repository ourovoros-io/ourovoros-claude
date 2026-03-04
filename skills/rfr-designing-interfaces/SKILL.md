---
name: rfr-designing-interfaces
description: Use when designing public Rust APIs, choosing function signatures, deciding between borrowed and owned parameters, implementing standard traits, or making APIs ergonomic and extensible. Use when reviewing API surface for usability.
---

# Rust for Rustaceans — Ch 3: Designing Interfaces

## Unsurprising APIs

**Users should be able to guess how your API works based on Rust conventions.**

### Implement Standard Traits
Every public type should consider implementing:

| Trait | When | Why |
|-------|------|-----|
| `Debug` | Always | Required for `{:?}` formatting, error messages, logging |
| `Display` | User-facing types | `println!("{x}")`, error messages |
| `Clone` | When duplication is meaningful | Let users copy your type |
| `Default` | When a zero/empty value makes sense | Works with `..Default::default()` |
| `PartialEq`, `Eq` | Value types | Enables `==` comparison and `assert_eq!` |
| `Hash` | If `Eq` is implemented | Enables use as `HashMap`/`HashSet` key |
| `PartialOrd`, `Ord` | Ordered types | Enables sorting, `BTreeMap` keys |
| `From`/`Into` | Type conversions | Enables `?` for errors, ergonomic constructors |
| `AsRef`/`AsMut` | Cheap reference conversions | Flexible function parameters |
| `Deref` | Smart pointer types ONLY | Not for general "inheritance" |
| `Send`/`Sync` | Concurrent types | Usually auto-derived; opt out explicitly if not safe |
| `Serialize`/`Deserialize` | Data types (behind feature flag) | Interop with serde ecosystem |

### Naming Conventions

| Pattern | Convention | Example |
|---------|-----------|---------|
| Conversion (owned) | `into_*` | `into_bytes()`, `into_inner()` |
| Conversion (borrowed) | `as_*` | `as_str()`, `as_bytes()` |
| Conversion (new type) | `to_*` | `to_string()`, `to_vec()` |
| Fallible | `try_*` | `try_from()`, `try_lock()` |
| Getter | field name (no `get_`) | `fn len()`, `fn name()` |
| Predicate | `is_*` or `has_*` | `is_empty()`, `has_key()` |
| Iterator | `iter()`, `iter_mut()`, `into_iter()` | Convention from std |
| Builder setter | `fn name(mut self, v: T) -> Self` | Chainable builder |
| Mutable access | `*_mut` | `get_mut()`, `iter_mut()` |

### Follow `std` Patterns
- If `is_empty()` exists, `len()` should too (and vice versa)
- If `new()` takes no args, implement `Default`
- If `Clone`, consider if `Copy` makes sense
- Iterators: implement `IntoIterator` for `&T`, `&mut T`, and `T`

## Flexible APIs

### Accept Broad Input Types

```rust
// ❌ Too restrictive
fn open(path: &str) -> File { ... }

// ✅ Accepts &str, String, PathBuf, &Path
fn open(path: impl AsRef<Path>) -> File { ... }

// ❌ Only takes String
fn greet(name: String) { ... }

// ✅ Accepts &str, String, Cow<str>
fn greet(name: &str) { ... }

// ✅ For ownership transfer: accept Into<String>
fn set_name(name: impl Into<String>) {
    let name = name.into();
}
```

### Borrowed vs Owned Parameters

| Take `&T` when | Take `T` when |
|----------------|---------------|
| Only reading the data | Storing it beyond the call |
| Caller should keep ownership | Transforming/consuming it |
| T is large | T is `Copy` and small |
| Multiple callers need access | Caller is done with it |

### Use `Cow` for Maybe-Owned Data
```rust
use std::borrow::Cow;

// Returns borrowed if no modification needed, owned if modified
fn normalize(s: &str) -> Cow<'_, str> {
    if s.contains(' ') {
        Cow::Owned(s.replace(' ', "_"))
    } else {
        Cow::Borrowed(s)
    }
}
```

### Iterator-Based APIs
```rust
// ❌ Requires a Vec
fn process(items: &[Item]) { ... }

// ✅ Accepts any iterator (Vec, slice, HashSet, generator)
fn process(items: impl IntoIterator<Item = Item>) {
    for item in items { ... }
}
```

### Return Types — Be Specific
- Return concrete types, not `impl Trait`, unless you need to hide the type
- Return `Result<T, E>` not `Option<T>` when the error carries information
- Return iterators as `impl Iterator<Item = T>` (hides implementation)
- Return `Self` from builder methods for chaining

## Obvious APIs

### Type-Driven Design
Encode invariants in the type system so illegal states are unrepresentable:

```rust
// ❌ Runtime check: can call send() before connect()
struct Connection { connected: bool }
impl Connection {
    fn send(&self, data: &[u8]) {
        assert!(self.connected); // Runtime panic
    }
}

// ✅ Compile-time enforcement via type state
struct Disconnected;
struct Connected;
struct Connection<S> { _state: PhantomData<S>, /* ... */ }

impl Connection<Disconnected> {
    fn connect(self) -> Connection<Connected> { ... }
}
impl Connection<Connected> {
    fn send(&self, data: &[u8]) { ... } // Can only call when Connected
}
```

### Use Newtypes to Prevent Mixing

```rust
// ❌ Easy to mix up
fn transfer(from: u64, to: u64, amount: u64) { ... }

// ✅ Compiler prevents argument swapping
struct AccountId(u64);
struct Amount(u64);
fn transfer(from: AccountId, to: AccountId, amount: Amount) { ... }
```

### Documentation
- Doc comments on all public items
- `# Examples` section with runnable code
- `# Errors` section listing when `Result::Err` is returned
- `# Panics` section if function can panic
- `# Safety` section on `unsafe fn` explaining invariants
- Link related items with `[`OtherType`]` intra-doc links

## Constrained APIs

### `#[must_use]`
- Add to `Result`-returning functions — prevents ignoring errors
- Add to builder methods — prevents calling without using result
- Add to types where ignoring the value is always a bug (e.g., `MutexGuard`)

```rust
#[must_use = "this Result may contain an error"]
fn save(&self) -> Result<(), Error> { ... }

#[must_use]
struct Guard<'a> { /* ... */ }
```

### `#[non_exhaustive]`
- On enums: callers must have a `_` wildcard arm (you can add variants)
- On structs: callers can't construct directly (must use constructor/builder)
- Use on all public enums and error types for forward compatibility

```rust
#[non_exhaustive]
pub enum Error {
    Io(std::io::Error),
    Parse(String),
    // Can add new variants without breaking downstream
}

#[non_exhaustive]
pub struct Config {
    pub timeout: Duration,
    // Can add fields without breaking downstream
}
```

### Sealed Traits
Prevent downstream implementations while allowing downstream use:

```rust
// In your crate:
mod private {
    pub trait Sealed {}
}

pub trait MyTrait: private::Sealed {
    fn method(&self);
}

// Only your crate can implement MyTrait
// because only your crate can implement Sealed
impl private::Sealed for MyType {}
impl MyTrait for MyType {
    fn method(&self) { ... }
}
```

### Visibility

| Visibility | Meaning |
|-----------|---------|
| `pub` | Visible everywhere |
| `pub(crate)` | Visible within the crate |
| `pub(super)` | Visible in parent module |
| `pub(in path)` | Visible in specific ancestor |
| (none) | Private to current module |

**Default to minimum visibility.** Start private, make `pub(crate)`, promote to `pub` only when needed.

## Builder Pattern

```rust
pub struct ServerConfig {
    port: u16,
    max_connections: usize,
    timeout: Duration,
}

pub struct ServerConfigBuilder {
    port: u16,
    max_connections: usize,
    timeout: Duration,
}

impl ServerConfigBuilder {
    #[must_use]
    pub fn new(port: u16) -> Self {
        Self {
            port,
            max_connections: 100,
            timeout: Duration::from_secs(30),
        }
    }

    #[must_use]
    pub fn max_connections(mut self, n: usize) -> Self {
        self.max_connections = n;
        self
    }

    #[must_use]
    pub fn timeout(mut self, d: Duration) -> Self {
        self.timeout = d;
        self
    }

    pub fn build(self) -> ServerConfig {
        ServerConfig {
            port: self.port,
            max_connections: self.max_connections,
            timeout: self.timeout,
        }
    }
}
```

## Common Mistakes

1. **Using `Deref` for "inheritance"** — `Deref` is for smart pointers only. Use composition + delegation or traits instead
2. **Taking `String` when `&str` suffices** — forces unnecessary cloning
3. **Returning `impl Trait` when concrete type is fine** — hides useful information, prevents naming the type
4. **Exhaustive public enums** — adding a variant is a breaking change without `#[non_exhaustive]`
5. **`get_` prefix on getters** — Rust convention omits it: `fn name(&self)` not `fn get_name(&self)`
6. **Missing `From`/`Into` implementations** — forces callers to convert manually; `?` won't work for error propagation
