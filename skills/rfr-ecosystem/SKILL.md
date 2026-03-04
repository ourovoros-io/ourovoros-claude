---
name: rfr-ecosystem
description: Use when applying Rust design patterns like type state, builder, newtype, extension traits, or RAII. Use when choosing crate architecture patterns, implementing iterators, or deciding when to use Deref polymorphism vs composition.
---

# Rust for Rustaceans — Ch 12: Rust Ecosystem Patterns

## Type State Pattern

**Encode state transitions in the type system so invalid transitions don't compile.**

```rust
use std::marker::PhantomData;

// States
struct Draft;
struct Review;
struct Published;

struct Document<State> {
    content: String,
    _state: PhantomData<State>,
}

impl Document<Draft> {
    fn new(content: String) -> Self {
        Self { content, _state: PhantomData }
    }

    fn submit_for_review(self) -> Document<Review> {
        Document { content: self.content, _state: PhantomData }
    }
}

impl Document<Review> {
    fn approve(self) -> Document<Published> {
        Document { content: self.content, _state: PhantomData }
    }

    fn reject(self) -> Document<Draft> {
        Document { content: self.content, _state: PhantomData }
    }
}

impl Document<Published> {
    fn content(&self) -> &str { &self.content }
    // Can't call submit_for_review or approve — wrong state
}
```

### When to Use
- State machines with clear transitions (connections, protocols, workflows)
- When calling methods in the wrong order is a logic bug
- When you want compile-time, not runtime, state checking

### When NOT to Use
- Too many states (>5-6) — combinatorial explosion
- Dynamic state determined at runtime — use enums instead
- States with shared behavior — lots of duplicated impls

## Builder Pattern

**Construct complex objects step-by-step with optional configuration.**

```rust
pub struct Server {
    host: String,
    port: u16,
    max_connections: usize,
    timeout: Duration,
}

pub struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: usize,
    timeout: Duration,
}

impl ServerBuilder {
    #[must_use]
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        Self {
            host: host.into(),
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

    pub fn build(self) -> Result<Server, BuildError> {
        if self.port == 0 {
            return Err(BuildError::InvalidPort);
        }
        Ok(Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
            timeout: self.timeout,
        })
    }
}

// Usage
let server = ServerBuilder::new("localhost", 8080)
    .max_connections(500)
    .timeout(Duration::from_secs(60))
    .build()?;
```

### Builder Guidelines
- Required params in `new()`, optional params as chainable methods
- `#[must_use]` on the builder and all setter methods
- `build()` validates and returns `Result` if validation can fail
- Take `mut self` (consuming) for builders, not `&mut self`
- Use `impl Into<String>` for string parameters

## Newtype Pattern

**Wrap a primitive type to add type safety and custom behavior.**

```rust
// Type safety: prevent mixing up IDs
struct UserId(u64);
struct OrderId(u64);

// Can't accidentally pass OrderId where UserId is expected
fn get_user(id: UserId) -> User { ... }

// Custom Display without orphan rule issues
struct Hex(Vec<u8>);
impl std::fmt::Display for Hex {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        for byte in &self.0 {
            write!(f, "{byte:02x}")?;
        }
        Ok(())
    }
}

// Enforce invariants
struct NonEmpty<T>(Vec<T>);
impl<T> NonEmpty<T> {
    fn new(first: T) -> Self {
        Self(vec![first])
    }

    fn push(&mut self, item: T) {
        self.0.push(item);
    }

    fn first(&self) -> &T {
        &self.0[0] // Always safe — guaranteed non-empty
    }
}
```

### When to Use Newtypes
- **Type safety**: `UserId` vs `OrderId` vs `u64`
- **Orphan rule**: impl foreign traits on foreign types
- **Invariant enforcement**: `NonEmpty`, `Positive`, `Validated`
- **API clarity**: self-documenting parameter types
- **Units**: `Meters(f64)`, `Seconds(f64)` — prevent unit confusion

### `repr(transparent)`
```rust
#[repr(transparent)]
struct Wrapper(u64);
// Wrapper has identical ABI to u64 — safe for FFI
```

## Extension Traits

**Add methods to types you don't own without newtypes.**

```rust
trait StringExt {
    fn truncate_to(&self, max_len: usize) -> &str;
}

impl StringExt for str {
    fn truncate_to(&self, max_len: usize) -> &str {
        if self.len() <= max_len {
            self
        } else {
            let mut end = max_len;
            while !self.is_char_boundary(end) {
                end -= 1;
            }
            &self[..end]
        }
    }
}

// Usage: "hello world".truncate_to(5)
```

### Extension Trait Rules
- Name with `Ext` suffix by convention
- Only add methods that genuinely belong on the type
- Consider if a free function would be clearer
- Users must `use YourCrate::StringExt;` to get the methods

## Iterator Pattern

### Custom Iterator
```rust
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Fibonacci {
    fn new() -> Self { Self { a: 0, b: 1 } }
}

impl Iterator for Fibonacci {
    type Item = u64;
    fn next(&mut self) -> Option<u64> {
        let result = self.a;
        self.a = self.b;
        self.b = result.checked_add(self.b)?; // None on overflow
        Some(result)
    }
}

// Usage
let first_10: Vec<u64> = Fibonacci::new().take(10).collect();
```

### `IntoIterator`
Implement for your collection types to enable `for` loops:

```rust
impl<T> IntoIterator for &MyCollection<T> {
    type Item = &T;
    type IntoIter = std::slice::Iter<'_, T>;
    fn into_iter(self) -> Self::IntoIter {
        self.items.iter()
    }
}

// Now: for item in &my_collection { ... }
```

### Iterator Adaptor Pattern
```rust
// Custom adaptor that wraps another iterator
struct Batched<I: Iterator> {
    inner: I,
    size: usize,
}

impl<I: Iterator> Iterator for Batched<I> {
    type Item = Vec<I::Item>;
    fn next(&mut self) -> Option<Self::Item> {
        let mut batch = Vec::with_capacity(self.size);
        for _ in 0..self.size {
            match self.inner.next() {
                Some(item) => batch.push(item),
                None => break,
            }
        }
        if batch.is_empty() { None } else { Some(batch) }
    }
}

// Extension trait to add .batched() to all iterators
trait IteratorExt: Iterator + Sized {
    fn batched(self, size: usize) -> Batched<Self> {
        Batched { inner: self, size }
    }
}
impl<I: Iterator> IteratorExt for I {}
```

## RAII Pattern (Drop-Based Cleanup)

```rust
struct TempFile {
    path: PathBuf,
}

impl TempFile {
    fn new() -> std::io::Result<Self> {
        let path = std::env::temp_dir().join(uuid::Uuid::new_v4().to_string());
        std::fs::File::create(&path)?;
        Ok(Self { path })
    }

    fn path(&self) -> &Path { &self.path }
}

impl Drop for TempFile {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.path);
    }
}

// File is automatically cleaned up when TempFile goes out of scope
```

### RAII Guidelines
- Use for any resource that needs cleanup (files, connections, locks, temp dirs)
- `Drop::drop` receives `&mut self` — can't move out of fields
- Don't panic in `Drop` — can abort during unwinding
- `Drop` isn't guaranteed to run (mem::forget, process abort) — safety must not depend on it

## Deref Polymorphism

### What It Is
`Deref` allows a type to "behave like" its inner type:

```rust
use std::ops::Deref;

struct MyString(String);

impl Deref for MyString {
    type Target = str;
    fn deref(&self) -> &str { &self.0 }
}

// Now MyString can use all &str methods: .len(), .contains(), etc.
```

### When to Use
- Smart pointers (`Box`, `Arc`, `Rc`, `MutexGuard`)
- Types that genuinely wrap and represent their inner value

### When NOT to Use
- General "inheritance" — use composition + explicit delegation
- Newtypes where you want to restrict the API
- Any case where implicit coercion could be surprising

```rust
// ❌ Bad: Deref for "inheritance"
struct Employee(Person);
impl Deref for Employee {
    type Target = Person;
    fn deref(&self) -> &Person { &self.0 }
}
// Problem: Employee implicitly has ALL Person methods

// ✅ Good: Explicit delegation
struct Employee {
    person: Person,
    department: String,
}
impl Employee {
    fn name(&self) -> &str { self.person.name() }
    // Only expose what makes sense
}
```

## Sealed Trait + Non-Exhaustive Enum Combo

For maximum forward compatibility:

```rust
mod private { pub trait Sealed {} }

#[non_exhaustive]
pub enum Format {
    Json,
    Yaml,
}

pub trait Formatter: private::Sealed {
    fn format(&self, data: &Data) -> String;
}

// You control all implementations and all variants
// Can add variants and impls freely
```

## Common Patterns Summary

| Pattern | Use When |
|---------|----------|
| Type State | Compile-time state machine enforcement |
| Builder | Complex object construction with optional config |
| Newtype | Type safety, orphan rule bypass, invariant enforcement |
| Extension Trait | Adding methods to foreign types |
| Iterator | Custom sequences, lazy evaluation |
| RAII | Automatic resource cleanup |
| Deref | Smart pointers (NOT general inheritance) |
| Sealed Trait | Prevent external implementations |

## Common Mistakes

1. **Deref for code reuse** — it's for smart pointers, not inheritance
2. **Builder with `&mut self`** — take `mut self` for ergonomic chaining
3. **Newtype without forwarding traits** — users can't `Debug`, `Display`, etc. unless you derive/impl
4. **Type state with too many states** — >5 states = enum + runtime checks is simpler
5. **Extension traits that shadow existing methods** — confusing behavior
6. **RAII depending on Drop for safety** — `mem::forget` is safe; safety can't rely on Drop running
7. **Missing `#[must_use]` on builder** — users might forget to call `.build()`
