---
name: rust-nomicon
description: Use when writing unsafe Rust, implementing custom allocators, building safe abstractions over unsafe primitives, working with raw pointers, variance, PhantomData, drop check, uninitialized memory, FFI, concurrency primitives, exception safety, or when lifetime/subtyping errors are confusing. Use when reviewing code for soundness or undefined behavior.
---

# The Rustonomicon - Unsafe Rust Mastery

## Overview

Complete reference for writing correct unsafe Rust. Covers the contract between safe and unsafe code, data layout, ownership/lifetime mechanics, type conversions, uninitialized memory, OBRM, exception safety, concurrency, and FFI. The core principle: **Safe Rust must never cause Undefined Behavior, no matter what.** Unsafe code must uphold this guarantee.

## When to Use

- Writing `unsafe` blocks or functions
- Implementing `unsafe` traits (`Send`, `Sync`, `GlobalAlloc`)
- Working with raw pointers, `MaybeUninit`, `transmute`
- Debugging lifetime, variance, or drop-check errors
- Building safe abstractions over unsafe internals
- FFI with C/C++ libraries
- Implementing custom collections, allocators, iterators
- Exception safety analysis (panic during unsafe state transitions)
- Understanding why the borrow checker rejects your code

**Not for:** General Rust programming, application-level security, async patterns

## The Safe/Unsafe Contract

### What Unsafe Unlocks

Only 5 extra capabilities:
1. Dereference raw pointers
2. Call `unsafe` functions (C functions, intrinsics, raw allocator)
3. Implement `unsafe` traits
4. Access/modify mutable statics
5. Access `union` fields

### All Causes of Undefined Behavior

| Category | Specific UB |
|----------|------------|
| Pointers | Deref dangling/unaligned pointer; violate aliasing rules |
| Functions | Wrong call ABI; unwinding with wrong unwind ABI |
| Concurrency | Data races |
| Target features | Execute code compiled for unsupported features |
| Invalid values | `bool` not 0/1; invalid `enum` discriminant; null `fn` ptr; `char` outside valid ranges; `!` type (all values invalid); integer/float from uninit memory; dangling/unaligned ref/`Box`; invalid wide-pointer metadata; `NonNull` that is null |

**NOT UB (but still bugs):** Deadlocks, race conditions, memory leaks, integer overflow, aborting, deleting production database.

### Soundness and Trust

- A correctly implemented unsafe function is **sound** — safe code cannot cause UB through it
- Safety is **non-local**: soundness of unsafe ops depends on state established by safe code
- **Privacy is essential**: module boundaries limit what safe code can break
- Unsafe code must trust some safe code but should **never trust generic safe code blindly**

```
Safe Rust ──trusts──▶ Unsafe Rust (must be written correctly)
Unsafe Rust ──cannot blindly trust──▶ Safe Rust (especially generic impls)
```

## Data Layout

### Alignment and Size

- Alignment is always a power of 2, at least 1
- Size is always a multiple of alignment
- `repr(Rust)`: Default. No guaranteed field ordering between types. Compiler may reorder for efficiency.
- `repr(C)`: C-compatible layout. Fields in declaration order. Required for FFI.
- `repr(transparent)`: Single non-ZST field. Same ABI as inner type. Safe to transmute between.
- `repr(packed)`: Alignment ≤ 1. Creates unaligned fields — taking references is UB on some platforms.
- `repr(align(N))`: Minimum alignment N. Useful for cache-line separation.

### Null Pointer Optimization

`Option<&T>`, `Option<Box<T>>`, `Option<NonNull<T>>` are same size as the inner pointer. `None` = null.

### Exotic Types

| Type | Properties |
|------|-----------|
| DSTs (`[T]`, `dyn Trait`) | No static size. Behind wide pointers (ptr + length or vtable). |
| ZSTs (`()`, `[u8; 0]`) | Size 0. Operations are no-ops. `Map<K, ()>` = `Set<K>` with zero overhead. |
| Empty types (`enum Void {}`) | Cannot be instantiated. `Result<T, Void>` optimized to just `T`. Use `*const ()` for C's `void*`, NOT `*const Void`. |

## Ownership, Lifetimes, and Borrowing

### Lifetime Mechanics

- Lifetimes are named regions of code where a reference must be valid
- Borrow is **alive** from creation to **last use** (not end of scope, unless type has `Drop`)
- Types with `Drop` keep borrows alive until scope end (destructor = last use)

### Lifetime Elision Rules

1. Each elided input lifetime becomes a distinct parameter
2. If exactly one input lifetime, it's assigned to all elided outputs
3. If multiple inputs but one is `&self`/`&mut self`, self's lifetime assigned to outputs
4. Otherwise, output lifetimes must be explicit

### Unbounded Lifetimes

Created by dereferencing raw pointers or `transmute`. Become as big as context demands (stronger than `'static`). **Almost always wrong.** Bound them immediately via function signatures with elision.

### Higher-Rank Trait Bounds (HRTBs)

`for<'a>` means "for all choices of `'a`":

```rust
where for<'a> F: Fn(&'a T) -> &'a U
```

Essential for closures that accept references with arbitrary lifetimes.

### Lifetime Limitations

The borrow checker is sometimes too conservative:
- `mutate_and_share(&mut self) -> &Self` — returned shared ref extends mutable borrow
- `get_default` pattern — can't reborrow `&mut` in different match arms
- These are correct programs the lifetime system rejects

## Subtyping and Variance

`'long <: 'short` — a longer lifetime is a subtype of a shorter one.

### Variance Table (Critical Reference)

| Type | Over `'a` | Over `T` |
|------|-----------|----------|
| `&'a T` | covariant | covariant |
| `&'a mut T` | covariant | **invariant** |
| `Box<T>`, `Vec<T>` | — | covariant |
| `UnsafeCell<T>`, `Cell<T>` | — | **invariant** |
| `*const T` | — | covariant |
| `*mut T` | — | **invariant** |
| `fn(T) -> U` | — | **contra**variant / covariant |
| `PhantomData<T>` | — | covariant |

**Why `&mut T` is invariant:** Prevents writing a short-lived value into a location expecting a long-lived one (use-after-free).

**Struct variance:** Inherited from fields. If any field is invariant over `T`, the struct is invariant over `T`. Invariance wins all conflicts.

## PhantomData

| Phantom type | `'a` variance | `T` variance | Send/Sync | Drop check |
|---|---|---|---|---|
| `PhantomData<T>` | — | covariant | inherited | owns `T` (disallows dangling) |
| `PhantomData<&'a T>` | covariant | covariant | needs `T: Sync` | allows dangling |
| `PhantomData<&'a mut T>` | covariant | **invariant** | inherited | allows dangling |
| `PhantomData<*const T>` | — | covariant | `!Send + !Sync` | allows dangling |
| `PhantomData<*mut T>` | — | **invariant** | `!Send + !Sync` | allows dangling |
| `PhantomData<fn(T)>` | — | **contra**variant | `Send + Sync` | allows dangling |
| `PhantomData<fn() -> T>` | — | covariant | `Send + Sync` | allows dangling |

Use `PhantomData` when raw pointers logically own data or carry lifetime information the compiler can't see.

## Drop Check

**Rule:** For a generic type to soundly implement `Drop`, its generic arguments must **strictly outlive** it.

- Adding `Drop` to a type makes the borrow checker more conservative
- `#[may_dangle]` (unstable): Assert that `Drop` impl doesn't access the marked generic parameter
- Use `ManuallyDrop` when drop order matters — don't rely on field order

### Drop Flags

- Rust tracks initialization state at runtime via hidden booleans when needed
- Static drop semantics (no flags) when initialization state is known at compile time
- Dynamic drop semantics (flags on stack) when conditional initialization occurs

## Type Conversions

### Coercions (Implicit, Safe)

Automatic type weakening. Does NOT apply in trait matching (except receivers via dot operator).

### The Dot Operator

Method resolution: try direct call → autoref (`&T`, `&mut T`) → deref chain → unsizing.

### Casts (`as`)

Superset of coercions. Infallible at runtime but can silently truncate/reinterpret. Raw slice casts don't adjust length. Not transitive.

### Transmute (Most Dangerous)

`mem::transmute<T, U>`: Reinterprets bits. Same size required.

**Iron rules:**
- `&` to `&mut` is **ALWAYS UB**. No exceptions.
- Creates unbounded lifetimes when producing references
- `repr(Rust)` layout is not guaranteed — even `Vec<i32>` vs `Vec<u32>` may differ
- `transmute_copy` is even more dangerous (no size check)

**Prefer:** `to_ne_bytes()`, `bytemuck`, `zerocopy`, raw pointer casts, `repr(C)` + `union`

## Uninitialized Memory

- All runtime-allocated memory starts uninitialized
- Reading uninit memory of **any** type = UB
- Use `MaybeUninit<T>` for safe handling

### MaybeUninit Pattern

```rust
let mut x = [const { MaybeUninit::uninit() }; SIZE];
for i in 0..SIZE {
    x[i] = MaybeUninit::new(compute(i));
}
let x = unsafe { mem::transmute::<_, [T; SIZE]>(x) };
```

### ptr Module

- `ptr::write(ptr, val)` — writes without dropping old value
- `ptr::copy(src, dst, count)` — memmove (note: reversed arg order from C)
- `ptr::copy_nonoverlapping(src, dst, count)` — memcpy (reversed from C)

**Never construct a reference to uninitialized data.** Use `&raw mut (*ptr).field` syntax instead.

## OBRM (Constructors, Destructors, Leaking)

### Constructors

Rust has exactly **one** constructor: name the type, initialize all fields. No Copy/Move/Default constructors like C++. Types must be ready to be `memcpy`'d anywhere — no move constructors.

### Destructors

- `Drop::drop(&mut self)` runs, then Rust recursively drops all fields
- No stable way to prevent recursive field dropping
- Use `Option::take()` to suppress recursive drop for specific fields
- `mem::forget` is **safe** — leaking is not UB

### Leaking is Safe

Code must not rely on destructors running for **safety** (only for correctness):
- `Drain`: Sets Vec len to 0 upfront — leak amplification, not UB
- `Rc`: Checks for ref count overflow — aborts rather than UB
- `thread::scoped` (removed): Was unsound because `mem::forget` could skip join

## Exception Safety (Unwinding)

### Two Levels

1. **Minimal** (required for unsafe code): No UB if panic occurs during unsafe state transition
2. **Maximal** (desirable for safe code): Program state remains consistent after panic

### Patterns

- **Guard pattern**: Store transient state in a struct with `Drop` that restores consistency
- **Two-phase approach**: Run untrusted code first, then perform unsafe mutations with only trusted code
- **Poisoning**: Mark shared state as potentially inconsistent after panic (e.g., `Mutex`)

### FFI Boundary

**Unwinding across FFI = UB.** Always use `catch_unwind` at FFI boundaries or use `-unwind` ABI variants (`"C-unwind"`).

## Concurrency

### Send and Sync

| Trait | Meaning | Auto-derived? |
|-------|---------|--------------|
| `Send` | Safe to move to another thread | Yes, if all fields `Send` |
| `Sync` | Safe to share via `&T` between threads (`&T` is `Send`) | Yes, if all fields `Sync` |

**Not Send/Sync:** Raw pointers (lint), `Rc` (unsynchronized refcount), `UnsafeCell` (not Sync), `MutexGuard` (not Send — must unlock on same thread).

### Data Races vs Race Conditions

- **Data race** = UB. Prevented by ownership system + `Send`/`Sync`.
- **Race condition** = logic bug. Cannot be prevented by type system.

### Atomics (C++20 Model)

| Ordering | Guarantees | Use Case |
|----------|-----------|----------|
| `Relaxed` | Atomic only, no ordering | Counters, statistics |
| `Acquire` | All accesses after stay after | Lock acquisition |
| `Release` | All accesses before stay before | Lock release |
| `AcqRel` | Both acquire and release | Read-modify-write |
| `SeqCst` | Total global order | When unsure; downgrade later |

Acquire-Release establishes **happens-before** between threads on the **same** memory location.

## FFI Quick Reference

### Declaring Foreign Functions

```rust
#[link(name = "mylib")]
unsafe extern "C" {
    fn c_function(input: *const u8, len: libc::size_t) -> libc::c_int;
}
```

### Exposing Rust to C

```rust
#[unsafe(no_mangle)]
pub extern "C" fn rust_function(x: i32) -> i32 { x + 1 }
```

### Opaque Types for FFI

```rust
#[repr(C)]
pub struct OpaqueHandle {
    _data: (),
    _marker: core::marker::PhantomData<(*mut u8, core::marker::PhantomPinned)>,
}
```

### Nullable Pointer Optimization

`Option<extern "C" fn(c_int) -> c_int>` has same layout as a C function pointer. `None` = null.

### Callbacks

```rust
// Rust callback callable from C
extern "C" fn callback(a: i32) { println!("Called with {}", a); }
// Pass via: register_callback(callback)
```

For object-targeted callbacks, pass `*mut RustObject` as context parameter.

### Variadic Functions

```rust
unsafe extern "C" {
    fn printf(fmt: *const c_char, ...);
}
```

### Link Types

- `#[link(name = "foo")]` — dynamic (default)
- `#[link(name = "foo", kind = "static")]` — static
- `#[link(name = "Foo", kind = "framework")]` — macOS framework

## `#![no_std]` Essentials

- Requires `#[panic_handler]` with signature `fn(&PanicInfo) -> !`
- Entry point via `#[unsafe(no_mangle)] extern "C" fn main(...)` + `#![no_main]`
- Use `libc` crate with `default-features = false`
- May need `compiler_builtins` for intrinsics
- `eh_personality` lang item needed on some platforms (nightly only)

## Splitting Borrows

The borrow checker understands disjoint struct field borrowing but NOT array/slice indexing.

**Solutions:**
- `split_at_mut()` for slices
- `Option::take()` for consuming iterator patterns
- Mutable iterators work safely for many containers without unsafe

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| `&` to `&mut` transmute | Always UB, optimizer assumes shared refs don't change | Use `UnsafeCell` |
| `repr(Rust)` transmute | Layout not guaranteed | Use `repr(C)` or `repr(transparent)` |
| Relying on destructor for safety | `mem::forget` is safe | Design for leak amplification |
| Unbounded lifetime from raw ptr | Outlives actual data | Bind via function signature elision |
| `*const Void` for C's `void*` | Deref/ref creation is UB | Use `*const ()` or opaque struct |
| Forgetting FFI unwind safety | Unwinding across FFI = UB | Use `catch_unwind` or `-unwind` ABI |
| Large unsafe blocks | Hard to audit, easy to miss panics | Minimize scope, isolate unsafe ops |
| Trusting generic `Ord`/`Eq` impls | Could be incorrect | Defend against bad impls in unsafe code |
