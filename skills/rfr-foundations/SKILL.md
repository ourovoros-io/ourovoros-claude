---
name: rfr-foundations
description: Use when reasoning about Rust memory layout, ownership semantics, borrowing rules, lifetime annotations, drop ordering, or interior mutability. Use when code has borrow checker errors, lifetime confusion, or unexpected drop behavior.
---

# Rust for Rustaceans — Ch 1: Foundations

## Memory Model

### Stack vs Heap
- **Stack**: fixed-size, LIFO, per-thread, automatic cleanup. Local variables, function args.
- **Heap**: dynamic-size, shared across threads (with synchronization), manual lifetime via ownership.
- Every variable has an **owner**. When the owner goes out of scope, the value is dropped.
- Pointers on the stack can reference heap data (`Box`, `Vec`, `String`), or other stack data (references).

### Value Semantics
- Assignment moves by default. After `let y = x;`, `x` is invalid (unless `Copy`).
- `Copy` types: bitwise copy, original remains valid. Only for types where bit-copy is semantically correct (integers, `bool`, `char`, `f64`, tuples/arrays of `Copy` types).
- `Clone` is explicit deep copy. Never derive `Clone` on types holding resources (file handles, connections) without careful thought.

```rust
// Move semantics
let s1 = String::from("hello");
let s2 = s1; // s1 is MOVED, no longer valid

// Copy semantics
let n1: i32 = 42;
let n2 = n1; // n1 is still valid (bitwise copy)
```

## Ownership Rules

1. Each value has exactly one owner
2. When the owner goes out of scope, the value is dropped
3. Ownership can be transferred (moved) but not duplicated (unless `Copy`/`Clone`)

### When to transfer ownership vs borrow
- **Take ownership** when you need to store the value or its lifetime must extend beyond the call
- **Borrow** when you only need temporary access
- **Return owned** when the caller needs to own the result

## Borrowing

### Shared references (`&T`)
- Multiple simultaneous readers allowed
- No mutation through `&T` (unless interior mutability)
- The referent must outlive all borrows

### Exclusive references (`&mut T`)
- Exactly one mutable reference at a time
- No other references (shared or exclusive) may coexist
- Enables mutation without ownership transfer

### The Borrow Checker Enforces
- No dangling references (lifetime checks)
- No aliased mutation (exclusivity checks)
- References never outlive their referent

```rust
// ✅ Multiple shared borrows
let v = vec![1, 2, 3];
let r1 = &v;
let r2 = &v;
println!("{r1:?} {r2:?}");

// ❌ Shared + exclusive borrow conflict
let mut v = vec![1, 2, 3];
let r = &v;
v.push(4); // ERROR: cannot borrow `v` as mutable
println!("{r:?}");
```

## Lifetimes

### What lifetimes represent
- A lifetime is the span during which a reference is valid
- Named lifetimes (`'a`) are constraints, not durations
- `'a` means "some region of code where this reference is valid"

### Lifetime elision rules
1. Each input reference gets its own lifetime
2. If exactly one input lifetime, it's assigned to all outputs
3. If `&self` or `&mut self`, its lifetime is assigned to all outputs

### Named lifetimes — when required
- Multiple input references where compiler can't infer which output borrows from
- Struct fields that are references (struct must be parameterized by lifetime)
- Trait definitions with reference returns

```rust
// Elision handles this (rule 2)
fn first(s: &str) -> &str { &s[..1] }

// Must annotate: two inputs, compiler can't choose
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// Struct holding a reference
struct Wrapper<'a> {
    data: &'a [u8],
}
```

### `'static` lifetime
- Lives for the entire program (string literals, leaked allocations, owned data)
- `T: 'static` means T contains no non-static references (it can be owned data)
- Common misconception: `'static` doesn't mean "lives forever" — it means "CAN live forever if needed"

### Lifetime Variance

**Variance determines how lifetimes relate through type constructors.**

| Type | Variance over `'a` | Meaning |
|------|-------------------|---------|
| `&'a T` | Covariant | Can shorten `'a` (substitute longer lifetime) |
| `&'a mut T` | Invariant | Cannot change `'a` at all |
| `fn(&'a T)` | Contravariant | Can lengthen `'a` (substitute shorter lifetime) |
| `Cell<&'a T>` | Invariant | Interior mutability forces invariance |

**Why it matters:**
- `&'a mut T` is invariant because if you could shorten the lifetime, you could write a shorter-lived reference through it, creating a dangling pointer
- `Cell<&'a T>` is invariant because you can write through it (interior mutability), same danger as `&mut`
- Covariance is the "safe default" — most types are covariant over their lifetime parameters

```rust
// Covariance: &'static str can be used where &'a str expected
fn print_str(s: &str) { println!("{s}"); }
let s: &'static str = "hello";
print_str(s); // 'static → shorter 'a: OK

// Invariance: &mut prevents lifetime substitution
fn extend_lifetime<'a>(
    short: &mut &'a str,
    long: &'static str,
) {
    // *short = long; // Would be unsound if 'a != 'static
}
```

## Drop

### Drop order
1. **Variables**: dropped in reverse declaration order (LIFO)
2. **Struct fields**: dropped in declaration order (first field first)
3. **Tuple elements**: dropped in order (index 0 first)
4. **Enum**: the active variant's fields are dropped
5. `Vec`, `HashMap` etc.: elements dropped in iteration order

### `Drop` trait
- Implement `Drop` for cleanup (close files, free resources, flush buffers)
- `Drop::drop` takes `&mut self`, not ownership — you can't move out of fields
- You cannot call `drop()` method directly; use `std::mem::drop(value)` or let scope end
- Implementing `Drop` prevents `Copy`

### `ManuallyDrop`
- Wraps a value to prevent automatic dropping
- Use for FFI resources, union fields, or precise drop timing
- Must manually call `ManuallyDrop::drop()` or `ManuallyDrop::into_inner()`
- Forgetting to drop causes a resource leak (not UB, but still a bug)

```rust
struct Connection { handle: RawHandle }
impl Drop for Connection {
    fn drop(&mut self) {
        unsafe { close_handle(self.handle); }
    }
}

// Variables drop in reverse order: b first, then a
let a = Connection { handle: open("a") };
let b = Connection { handle: open("b") };
// drop(b), then drop(a)
```

## Interior Mutability

**Allows mutation through `&T` (shared reference) by enforcing borrowing rules at runtime or through atomic operations.**

| Type | Thread-safe? | Overhead | Use case |
|------|-------------|----------|----------|
| `Cell<T>` | No | Zero (copy in/out) | Small `Copy` types in single-threaded code |
| `RefCell<T>` | No | Runtime borrow check | Complex borrows in single-threaded code |
| `Mutex<T>` | Yes | Lock acquisition | Shared mutable state across threads |
| `RwLock<T>` | Yes | Lock acquisition | Many readers, few writers across threads |
| `Atomic*` | Yes | CPU atomic ops | Counters, flags, lock-free data |
| `UnsafeCell<T>` | No | Zero | Building custom interior-mutable types (unsafe) |

### Key rules
- `Cell`: get/set/replace — no references into the cell, only copies
- `RefCell`: borrow()/borrow_mut() — panics on violation at runtime
- `UnsafeCell`: the primitive — ALL interior mutability is built on this
- If your type has `UnsafeCell`, it becomes invariant over the contained type's lifetime

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);
// Shared reference, but can mutate:
data.borrow_mut().push(4);
assert_eq!(data.borrow().len(), 4);

// Runtime panic: double mutable borrow
// let r1 = data.borrow_mut();
// let r2 = data.borrow_mut(); // PANIC
```

## Quick Reference

| Concept | Rule |
|---------|------|
| Move | Default for non-Copy types; original invalidated |
| Copy | Opt-in via derive; only for bitwise-copyable types |
| `&T` | Many simultaneous; no mutation |
| `&mut T` | Exactly one; exclusive access |
| Lifetime `'a` | Constraint on reference validity region |
| `'static` | No non-static references; can live arbitrarily long |
| Drop order | Variables: reverse. Fields: declaration order |
| Interior mutability | Runtime-checked or atomic borrowing rules |

## Common Mistakes

1. **Assuming `'static` means immortal** — it means "no borrowed data with limited lifetime"
2. **Fighting the borrow checker with `clone()`** — usually indicates a design issue; restructure ownership instead
3. **Ignoring drop order** — matters for types that reference each other (e.g., a struct holding a `JoinHandle` and the data it reads)
4. **Using `RefCell` across threads** — it's not `Sync`; use `Mutex` or `RwLock`
5. **Storing references when you should store owned data** — if the lifetime is unclear, own it
6. **Deriving `Copy` on types with heap allocations** — won't compile, but attempting it shows a design confusion
