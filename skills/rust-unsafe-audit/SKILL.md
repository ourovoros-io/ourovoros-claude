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
| Raw pointer deref | UB, crash | Non-null, aligned, valid |
| `transmute` | UB, memory corruption | Same size, valid repr |
| FFI call | UB, crash | Correct signature, valid args |
| Mutable aliasing | UB | No other refs exist |
| `Send`/`Sync` impl | Data races | Type is actually thread-safe |

## Audit Checklist

For every `unsafe` block:

1. **Identify the unsafe operation** - What specifically requires unsafe?
2. **Document invariants** - What must be true for this to be sound?
3. **Verify invariants** - How do you know they hold?
4. **Minimize scope** - Is all code inside necessary?
5. **Test edge cases** - What happens at boundaries?

## Common Patterns

### Raw Pointer Handling

```rust
// BAD: No validation
unsafe fn get_value(ptr: *const i32) -> i32 {
    *ptr  // Could be null, misaligned, or dangling
}

// GOOD: Validate before deref
fn get_value(ptr: *const i32) -> Option<i32> {
    if ptr.is_null() {
        return None;
    }
    // SAFETY: Pointer is non-null. Caller must ensure:
    // - Pointer is properly aligned for i32
    // - Pointer points to initialized i32
    // - No mutable references to this memory exist
    Some(unsafe { *ptr })
}
```

### FFI Boundaries

```rust
// C function signature
// int process_data(const char* input, size_t len, char* output, size_t* out_len);

extern "C" {
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

### Implementing Unsafe Traits

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

### Transmute Alternatives

```rust
// BAD: transmute is a code smell
let bytes: [u8; 4] = unsafe { std::mem::transmute(value) };

// GOOD: Use safe alternatives
let bytes = value.to_ne_bytes();  // For integers
let bytes = value.to_le_bytes();  // Explicit endianness

// For slices
let byte_slice: &[u8] = bytemuck::bytes_of(&value);

// If transmute truly needed, document thoroughly:
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
| `unsafe` without comment | Unknown invariants | Document SAFETY |
| Large unsafe blocks | Hard to audit | Minimize scope |
| `transmute` | Usually wrong | Use safe conversions |
| `as *mut` casts | Aliasing violations | Review carefully |
| `static mut` | Data races | Use `OnceLock`, atomics |
| `MaybeUninit` | Uninitialized reads | Ensure init before use |

## Safety Documentation Format

```rust
// SAFETY: [Why this is sound]
// - Invariant 1: [how it's upheld]
// - Invariant 2: [how it's upheld]
// Caller requirements (if pub unsafe fn):
// - Requirement 1
// - Requirement 2
unsafe {
    // minimal unsafe code
}
```

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| No safety comment | Can't audit | Always document |
| Wrong pointer lifetime | Use after free | Tie to borrow |
| Aliasing `&mut` | Instant UB | Use `UnsafeCell` |
| Unaligned access | UB on some platforms | Use `read_unaligned` |
| Forgetting drop | Memory leak | `ManuallyDrop` or explicit |
| `#[repr(Rust)]` transmute | Layout not guaranteed | Use `#[repr(C)]` |

## Tools for Unsafe Auditing

- `cargo miri` - Detects UB at runtime
- `cargo careful` - Extra runtime checks
- `-Z sanitizer=address` - Memory errors
- `clippy::undocumented_unsafe_blocks` - Enforce comments
- `cargo-geiger` - Count unsafe in dependencies
