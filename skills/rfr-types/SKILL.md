---
name: rfr-types
description: Use when reasoning about Rust type layout and alignment, choosing between trait objects and generics, understanding Sized bounds, using PhantomData, or implementing marker traits. Use when encountering fat pointer or DST issues.
---

# Rust for Rustaceans — Ch 2: Types

## Types in Memory

### Size and Alignment
- Every type has a **size** (bytes occupied) and **alignment** (address must be divisible by)
- Alignment is always a power of 2
- Struct size is padded to a multiple of its largest field's alignment
- Compiler may reorder fields to minimize padding (unless `repr(C)`)

### Representations

| Repr | Behavior | Use case |
|------|----------|----------|
| Default | Compiler optimizes layout, may reorder fields | Normal Rust types |
| `repr(C)` | C-compatible layout, fields in declaration order | FFI, stable ABI |
| `repr(transparent)` | Same layout as the single field | Newtypes for FFI |
| `repr(packed)` | No padding, alignment = 1 | Wire protocols (creates unaligned access) |
| `repr(align(N))` | Minimum alignment of N | Cache line alignment |

```rust
#[repr(C)]
struct Header {
    version: u8,    // offset 0
    _pad: [u8; 3],  // offset 1 (explicit padding)
    length: u32,    // offset 4
}

#[repr(transparent)]
struct UserId(u64); // Same ABI as u64
```

### Sized vs Dynamically Sized Types (DSTs)

- Most types are `Sized` — their size is known at compile time
- DSTs have unknown size: `str`, `[T]`, `dyn Trait`
- DSTs can only exist behind a pointer: `&str`, `&[T]`, `Box<dyn Trait>`
- **Wide/fat pointers**: pointer to DST = data pointer + metadata
  - `&[T]`: pointer + length
  - `&dyn Trait`: pointer + vtable pointer

### `?Sized` Bound
- `T: ?Sized` removes the implicit `Sized` bound, allowing DSTs
- Use in generic functions that should accept both sized and unsized types
- Required for `&T` where `T` might be a trait object or slice

```rust
// Only accepts Sized types (implicit bound)
fn foo<T>(t: &T) {}

// Accepts both Sized and unsized (str, [u8], dyn Trait)
fn bar<T: ?Sized>(t: &T) {}

// This is why str::len works:
// fn len(&self) -> usize  where Self: ?Sized
```

### Zero-Sized Types (ZSTs)
- Size = 0, alignment = 1: `()`, `PhantomData<T>`, empty structs
- No runtime cost, no memory, but participate in type system
- `Vec<()>` only tracks length — no actual data allocation
- Useful as markers, type-level tags, and in generic code

## Traits

### Static Dispatch (Generics)
- `fn foo<T: Trait>(t: T)` — monomorphized at compile time
- Each concrete type gets its own copy of the function
- Zero overhead at runtime, but increases binary size
- Can inline, optimize per-type

### Dynamic Dispatch (Trait Objects)
- `fn foo(t: &dyn Trait)` — single function, vtable lookup at runtime
- Small runtime overhead (indirect call, no inlining)
- Smaller binary, enables heterogeneous collections
- Object must be behind a pointer (`&dyn`, `Box<dyn>`, `Arc<dyn>`)

### Object Safety
A trait is object-safe (can be used as `dyn Trait`) if:
1. No `Self: Sized` bound on the trait itself
2. No associated functions that use `Self` in argument/return position (except `self`)
3. No generic methods (type params on methods)
4. No associated constants or types with `Self` bounds

```rust
// ✅ Object-safe
trait Draw {
    fn draw(&self);
}

// ❌ Not object-safe: returns Self
trait Clonable {
    fn clone_self(&self) -> Self;
}

// ❌ Not object-safe: generic method
trait Converter {
    fn convert<T>(&self) -> T;
}
```

### Associated Types vs Generic Parameters

| Feature | Associated Type | Generic Parameter |
|---------|----------------|-------------------|
| Syntax | `type Output;` | `trait Foo<T>` |
| Impls per type | Exactly one | Multiple (one per T) |
| Inference | Compiler knows the type | Must specify or infer |
| Use when | One natural output | Multiple valid types |

```rust
// Associated type: Iterator has exactly one Item per impl
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// Generic: a type can implement From<T> for many T
trait From<T> {
    fn from(t: T) -> Self;
}
```

### Orphan Rule
- You can only implement a trait for a type if you own the trait OR the type
- **Newtype pattern** bypasses this: wrap the foreign type in your own struct

```rust
// Can't impl Display for Vec<T> (both foreign)
// Solution: newtype wrapper
struct Wrapper(Vec<String>);

impl std::fmt::Display for Wrapper {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0.join(", "))
    }
}
```

### Blanket Implementations
- `impl<T: TraitA> TraitB for T` — implements TraitB for ALL types that have TraitA
- Powerful but constraining: prevents downstream impls of TraitB
- Used extensively in std: `impl<T: Display> ToString for T`

### Supertraits
- `trait B: A` means "implementing B requires implementing A"
- Not inheritance — it's a bound. B's methods can call A's methods on `&self`

```rust
trait Printable: std::fmt::Display {
    fn print(&self) {
        println!("{self}"); // Can use Display because of supertrait
    }
}
```

## PhantomData

**Tells the compiler your type logically contains a `T` even though it doesn't physically store one.**

### Use Cases

| `PhantomData<X>` | Effect | Use case |
|-------------------|--------|----------|
| `PhantomData<T>` | Owns a `T` (drop check, variance) | Containers that own `T` through raw pointers |
| `PhantomData<&'a T>` | Borrows `T` for `'a` | Iterators over raw pointers |
| `PhantomData<*const T>` | Covariant, no ownership | Type tags without drop implications |
| `PhantomData<fn(T)>` | Contravariant over T | Rarely needed |
| `PhantomData<fn() -> T>` | Covariant, no ownership/drop | Phantom type parameters |

```rust
use std::marker::PhantomData;

// Type-state: compile-time state machine
struct Locked;
struct Unlocked;

struct Door<State> {
    _state: PhantomData<State>,
}

impl Door<Locked> {
    fn unlock(self) -> Door<Unlocked> {
        Door { _state: PhantomData }
    }
}

impl Door<Unlocked> {
    fn open(&self) { /* ... */ }
}
```

## Marker Traits

| Trait | Meaning | Auto-derived? |
|-------|---------|---------------|
| `Send` | Safe to transfer between threads | Yes, if all fields are Send |
| `Sync` | Safe to share references between threads (`&T` is Send) | Yes, if all fields are Sync |
| `Copy` | Bitwise copy, no drop | Manual derive only |
| `Unpin` | Safe to move after pinning | Yes, by default |
| `Sized` | Size known at compile time | Implicit bound on all type params |

### Send and Sync Relationships
- `T: Send` → `T` can be moved to another thread
- `T: Sync` → `&T` can be sent to another thread (i.e., `&T: Send`)
- `T: Sync` ⟺ `&T: Send`
- `Rc<T>`: not Send, not Sync (reference count not atomic)
- `Arc<T>`: Send + Sync if `T: Send + Sync`
- `Cell<T>`: Send (if T: Send), NOT Sync
- `Mutex<T>`: Send + Sync (if T: Send)
- Raw pointers: neither Send nor Sync by default

```rust
// Opting out of auto-traits
struct NotSend {
    _marker: PhantomData<*const ()>, // *const is !Send + !Sync
}

// Opting in (unsafe — you must ensure soundness)
struct MyPtr(*mut u8);
unsafe impl Send for MyPtr {}
unsafe impl Sync for MyPtr {}
```

## Common Mistakes

1. **Using `repr(packed)` without understanding unaligned access** — creates references to unaligned memory (UB). Use `read_unaligned`/`write_unaligned` or `addr_of!`
2. **Missing `PhantomData` on types with raw pointers** — compiler won't know about ownership/lifetime, leading to incorrect drop check or variance
3. **Making a trait generic when associated type suffices** — forces callers to specify the type parameter everywhere
4. **Forgetting orphan rule** — can't impl foreign trait for foreign type; use newtype
5. **Assuming `dyn Trait` has zero cost** — vtable lookup prevents inlining; use generics on hot paths
6. **Deriving `Send`/`Sync` manually without upholding invariants** — must guarantee thread safety yourself
