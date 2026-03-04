---
name: rust-unsafe-audit
description: Use when reviewing unsafe Rust code, FFI boundaries, raw pointers, or code using transmute. Use when writing or auditing code that bypasses the borrow checker.
---

# Rust Unsafe Audit

## Overview

Review unsafe Rust code for soundness. Unsafe blocks must uphold invariants the compiler can't verify. Every unsafe block needs justification and careful analysis.

## When to Use

- Code contains `unsafe` blocks
- FFI with C libraries
- Raw pointer manipulation
- `std::mem::transmute` usage
- Implementing unsafe traits (`Send`, `Sync`)
- Performance-critical code bypassing checks

**Not for:** Application security (use `rust-security-audit`), general code review

## Quick Reference

| Unsafe Operation | Risk | Required Invariant |
|------------------|------|-------------------|
| Raw pointer deref | UB, crash | Non-null, aligned, valid, no aliasing violations |
| `transmute` | UB, memory corruption | Same size, valid repr, all bit patterns valid |
| FFI call | UB, crash | Correct signature, valid args, exception safety |
| Mutable aliasing | UB | No other refs exist |
| `Send`/`Sync` impl | Data races | Type is actually thread-safe |
| `static mut` | Data races | Use `OnceLock`, atomics, or `Mutex` instead |
| Union field access | UB | Active field matches access |

## Audit Checklist

For every `unsafe` block:

1. **Identify the unsafe operation** -- What specifically requires unsafe?
2. **Document invariants** -- What must be true for this to be sound?
3. **Verify invariants** -- How do you know they hold?
4. **Minimize scope** -- Is all code inside necessary?
5. **Exception safety** -- What if code panics mid-operation?
6. **Test edge cases** -- What happens at boundaries?

## Safety Documentation Format

```rust
// SAFETY: [Why this is sound]
// - Invariant 1: [how it's upheld]
// - Invariant 2: [how it's upheld]
unsafe {
    // minimal unsafe code
}
```

For `pub unsafe fn`, document caller requirements in a `# Safety` doc section:

```rust
/// # Safety
///
/// - `ptr` must be non-null and properly aligned for `T`
/// - `ptr` must point to a valid, initialized value of `T`
/// - No mutable references to this memory may exist
pub unsafe fn deref_raw<T>(ptr: *const T) -> &T {
    unsafe { &*ptr }
}
```

## Edition 2024: `unsafe_op_in_unsafe_fn`

In Edition 2024, the body of an `unsafe fn` no longer implicitly allows unsafe operations:

```rust
// Edition 2024: must use unsafe blocks inside unsafe fn
unsafe fn read_raw(ptr: *const u8, len: usize) -> &[u8] {
    unsafe { std::slice::from_raw_parts(ptr, len) }
}
```

## Variance and PhantomData

Variance determines how subtyping of generics affects the container. Getting this wrong causes UB.

| Type | Over `T` |
|------|----------|
| `&T`, `Box<T>`, `Vec<T>`, `*const T` | Covariant |
| `&mut T`, `*mut T`, `UnsafeCell<T>` | Invariant |
| `fn(T) -> U` | Contravariant in T, covariant in U |

**PhantomData tells the compiler about ownership and variance:**

| Pattern | Variance | Use when |
|---------|----------|----------|
| `PhantomData<T>` | Covariant, owns T | Custom allocator holding `*mut T` |
| `PhantomData<&'a T>` | Covariant | Borrowing T for lifetime 'a |
| `PhantomData<&'a mut T>` | Invariant | Mutable borrow for 'a |
| `PhantomData<*const T>` | Covariant, !Send !Sync | Raw pointer semantics |
| `PhantomData<*mut T>` | Invariant, !Send !Sync | Mutable raw pointer |
| `PhantomData<fn(T)>` | Contravariant, Send+Sync | Type parameter without ownership |

```rust
use std::marker::PhantomData;

struct Slice<'a, T> {
    ptr: *const T,
    len: usize,
    _marker: PhantomData<&'a T>,  // borrows T for 'a, covariant
}
```

## Exception Safety (Panic Safety)

Unsafe code must handle panics. If code panics between setting up invariants and completing the operation, the type may be left in an invalid state.

```rust
// BAD: panic in clone() leaves Vec with uninitialized elements
impl<T: Clone> Vec<T> {
    fn push_all(&mut self, to_push: &[T]) {
        self.reserve(to_push.len());
        unsafe {
            self.set_len(self.len() + to_push.len()); // len set BEFORE init
            for (i, x) in to_push.iter().enumerate() {
                self.ptr().add(i).write(x.clone()); // panic here = UB
            }
        }
    }
}
```

**Fix: destructor guard pattern** -- a struct whose Drop restores invariants on panic:

```rust
struct SetLenOnDrop<'a, T> {
    vec: &'a mut Vec<T>,
    len: usize,
}
impl<T> Drop for SetLenOnDrop<'_, T> {
    fn drop(&mut self) {
        // Only set len to actually-initialized count
        unsafe { self.vec.set_len(self.len); }
    }
}
```

## Raw Pointer Handling

```rust
// BAD: No validation
unsafe fn get_value(ptr: *const i32) -> i32 {
    *ptr
}

// GOOD: Validate before deref
fn get_value(ptr: *const i32) -> Option<i32> {
    if ptr.is_null() {
        return None;
    }
    // SAFETY: Pointer is non-null. Caller must ensure:
    // - Properly aligned for i32
    // - Points to initialized i32
    // - No mutable references to this memory exist
    Some(unsafe { *ptr })
}
```

## FFI Boundaries

```rust
unsafe extern "C" {
    fn process_data(
        input: *const c_char,
        len: size_t,
        output: *mut c_char,
        out_len: *mut size_t,
    ) -> c_int;
}

pub fn process(input: &str) -> Result<String, Error> {
    let c_input = CString::new(input)?;
    let mut output_buf = vec![0u8; 1024];
    let mut out_len: size_t = output_buf.len();

    // SAFETY:
    // - c_input.as_ptr() is valid null-terminated string
    // - input.len() matches actual string length
    // - output_buf is valid writable memory of out_len size
    // - out_len points to valid size_t
    let result = unsafe {
        process_data(
            c_input.as_ptr(),
            input.len(),
            output_buf.as_mut_ptr() as *mut c_char,
            &mut out_len,
        )
    };

    if result != 0 {
        return Err(Error::ProcessFailed(result));
    }

    output_buf.truncate(out_len);
    String::from_utf8(output_buf).map_err(Error::Utf8)
}
```

## Implementing Unsafe Traits

```rust
struct MyWrapper {
    data: *mut u8,
    len: usize,
}

// SAFETY: MyWrapper can be sent between threads because:
// - The pointer is exclusively owned (not shared)
// - No thread-local data is referenced
// - The pointed-to data has no thread affinity
unsafe impl Send for MyWrapper {}

// SAFETY: MyWrapper can be shared between threads because:
// - All methods that access data use proper synchronization
// - No interior mutability without synchronization
unsafe impl Sync for MyWrapper {}
```

## Transmute Alternatives

```rust
// BAD: transmute is a code smell
let bytes: [u8; 4] = unsafe { std::mem::transmute(value) };

// GOOD: Use safe alternatives
let bytes = value.to_ne_bytes();
let bytes = value.to_le_bytes();
let byte_slice: &[u8] = bytemuck::bytes_of(&value);

// If transmute truly needed:
// SAFETY: Both types have identical memory layout because:
// - Same size (verified by static assert)
// - Same alignment (repr(C) on both)
// - All bit patterns are valid for target type
const _: () = assert!(size_of::<Source>() == size_of::<Target>());
let target: Target = unsafe { std::mem::transmute(source) };
```

## Red Flags

| Pattern | Risk | Fix |
|---------|------|-----|
| `unsafe` without `// SAFETY:` | Unknown invariants | Document safety |
| Large unsafe blocks | Hard to audit | Minimize scope |
| `transmute` | Usually wrong | Use safe conversions |
| `as *mut` casts | Aliasing violations | Review carefully |
| `static mut` | Data races | Use `OnceLock`, atomics |
| `MaybeUninit` | Uninitialized reads | Ensure init before use |
| No panic consideration | Invariant violation on unwind | Destructor guard pattern |
| Wrong `PhantomData` | Unsound variance/lifetime | Match ownership semantics |

## Tools for Unsafe Auditing

```bash
# UB detection at runtime (best tool for unsafe audit)
cargo +nightly miri test
# Explore thread interleavings:
MIRIFLAGS="-Zmiri-many-seeds=100" cargo +nightly miri test
# Use Tree Borrows model:
MIRIFLAGS="-Zmiri-tree-borrows" cargo +nightly miri test

# Extra runtime checks (faster than miri, less thorough)
cargo +nightly careful test

# Sanitizers (nightly, catches issues miri misses in FFI)
RUSTFLAGS="-Zsanitizer=address" cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu
RUSTFLAGS="-Zsanitizer=thread" cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu

# Count unsafe in dependencies
cargo geiger
```

**Miri catches:** aliasing violations (Stacked/Tree Borrows), use-after-free, uninitialized reads, misaligned access, invalid values, data races.

**Miri does NOT catch:** FFI/C code bugs, layout-dependent code, exhaustive exploration (only one execution path per run).

Enable the clippy lint: `clippy::undocumented_unsafe_blocks`
