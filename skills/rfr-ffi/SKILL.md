---
name: rfr-ffi
description: Use when writing FFI code between Rust and C/C++, using extern functions, handling C types (CStr, CString), or working with bindgen/cbindgen. Use when crossing language boundaries, managing callbacks, or ensuring panic safety at FFI boundaries.
---

# Rust for Rustaceans — Ch 11: Foreign Function Interfaces

## Calling C from Rust

### Declaring External Functions
```rust
extern "C" {
    fn strlen(s: *const std::ffi::c_char) -> usize;
    fn malloc(size: usize) -> *mut std::ffi::c_void;
    fn free(ptr: *mut std::ffi::c_void);
}

// Usage (always unsafe)
let len = unsafe { strlen(c_str.as_ptr()) };
```

### Linking
```rust
// Link to a system library
#[link(name = "z")]      // libz (zlib)
extern "C" {
    fn compress(/* ... */);
}

// In build.rs for custom libraries
println!("cargo::rustc-link-lib=mylib");
println!("cargo::rustc-link-search=/path/to/lib");
```

## Exposing Rust to C

### `extern "C"` Functions
```rust
// Callable from C
#[no_mangle]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}

// Generate C header with cbindgen
```

### `#[no_mangle]`
- Prevents Rust from mangling the symbol name
- Required for C to find the function by name
- Puts the symbol in global namespace — ensure unique names

## Type Mapping

### Primitive Types

| C Type | Rust Type | Notes |
|--------|-----------|-------|
| `int` | `c_int` | Usually `i32` but platform-dependent |
| `unsigned int` | `c_uint` | |
| `long` | `c_long` | 32-bit on Windows, 64-bit on Unix |
| `size_t` | `usize` | |
| `char` | `c_char` | `i8` or `u8` depending on platform |
| `void*` | `*mut c_void` | |
| `const void*` | `*const c_void` | |
| `bool` (C99) | `bool` | |
| `float` | `f32` | |
| `double` | `f64` | |

**Always use `std::ffi::c_*` types, not raw `i32`/`u64`.** The sizes may differ across platforms.

### Strings

| Direction | Type | Notes |
|-----------|------|-------|
| Rust → C | `CString` → `.as_ptr()` | Null-terminated, owned |
| C → Rust (borrowed) | `CStr::from_ptr()` → `.to_str()` | Borrowed, no allocation |
| C → Rust (owned) | `CString::from_raw()` | Takes ownership of C allocation |

```rust
use std::ffi::{CStr, CString};

// Rust string → C
let c_string = CString::new("hello").unwrap();
unsafe { c_function(c_string.as_ptr()); }
// c_string is still alive — pointer remains valid

// C string → Rust
unsafe {
    let c_str = CStr::from_ptr(c_ptr);
    let rust_str: &str = c_str.to_str().unwrap();
}
```

### Structs
```rust
#[repr(C)]
struct Point {
    x: f64,
    y: f64,
}

// repr(C) ensures:
// - Fields in declaration order
// - C-compatible alignment and padding
// - Deterministic layout
```

### Enums
```rust
#[repr(C)]
enum Color {
    Red = 0,
    Green = 1,
    Blue = 2,
}

// For enums with data, C doesn't have a direct equivalent
// Use a tagged union pattern:
#[repr(C)]
struct TaggedValue {
    tag: u32,
    payload: ValuePayload,
}

#[repr(C)]
union ValuePayload {
    integer: i64,
    float: f64,
    string: *const c_char,
}
```

### Opaque Types
When C code shouldn't know the layout:

```rust
// Expose as opaque pointer
pub struct MyContext { /* internal fields */ }

#[no_mangle]
pub extern "C" fn context_new() -> *mut MyContext {
    Box::into_raw(Box::new(MyContext::new()))
}

#[no_mangle]
pub unsafe extern "C" fn context_free(ctx: *mut MyContext) {
    if !ctx.is_null() {
        drop(Box::from_raw(ctx));
    }
}

#[no_mangle]
pub unsafe extern "C" fn context_process(
    ctx: *mut MyContext,
    data: *const u8,
    len: usize,
) -> i32 {
    let ctx = &mut *ctx;
    let data = std::slice::from_raw_parts(data, len);
    match ctx.process(data) {
        Ok(()) => 0,
        Err(_) => -1,
    }
}
```

## Callbacks

### C Calling Back into Rust
```rust
// C-compatible function pointer
type Callback = extern "C" fn(i32) -> i32;

#[no_mangle]
pub extern "C" fn register_callback(cb: Callback) {
    let result = cb(42); // Call the C function pointer
}

// With user data (closure-like)
type CallbackWithData = extern "C" fn(
    *mut c_void,  // user data
    i32,          // argument
) -> i32;

#[no_mangle]
pub unsafe extern "C" fn register_callback_with_data(
    cb: CallbackWithData,
    user_data: *mut c_void,
) {
    cb(user_data, 42);
}
```

### Passing Rust Closures to C
```rust
// Convert closure to (fn pointer + data pointer)
extern "C" fn trampoline<F: FnMut(i32)>(
    data: *mut c_void,
    value: i32,
) {
    let callback = unsafe { &mut *(data as *mut F) };
    callback(value);
}

fn register<F: FnMut(i32)>(mut callback: F) {
    let data = &mut callback as *mut F as *mut c_void;
    unsafe {
        c_register(trampoline::<F>, data);
    }
    // callback must outlive the C registration!
}
```

## Panic Safety at FFI Boundaries

**Unwinding across FFI boundaries is undefined behavior.**

```rust
// ❌ Panic can unwind into C code
#[no_mangle]
pub extern "C" fn dangerous() {
    panic!("this is UB if called from C!");
}

// ✅ Catch panics at the boundary
#[no_mangle]
pub extern "C" fn safe_function() -> i32 {
    match std::panic::catch_unwind(|| {
        might_panic();
        0
    }) {
        Ok(result) => result,
        Err(_) => -1, // Return error code instead of unwinding
    }
}

// ✅ Or use extern "C-unwind" (Rust 1.71+) if C code uses exceptions
#[no_mangle]
pub extern "C-unwind" fn allows_unwind() {
    panic!("this can unwind through C frames that support it");
}
```

## Code Generation Tools

### `bindgen` — C Headers → Rust Bindings
```bash
cargo install bindgen-cli
bindgen wrapper.h -o src/bindings.rs
```

```rust
// build.rs
let bindings = bindgen::Builder::default()
    .header("wrapper.h")
    .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
    .generate()
    .expect("unable to generate bindings");

bindings
    .write_to_file(out_path.join("bindings.rs"))
    .expect("couldn't write bindings");
```

### `cbindgen` — Rust → C Headers
```bash
cargo install cbindgen
cbindgen --crate mylib --output mylib.h
```

```toml
# cbindgen.toml
language = "C"
include_guard = "MYLIB_H"
```

## Safety Wrapper Pattern

**Always wrap raw FFI in a safe Rust API:**

```rust
// Raw FFI (private)
mod ffi {
    use std::ffi::c_int;
    extern "C" {
        pub fn raw_open(path: *const i8) -> c_int;
        pub fn raw_close(fd: c_int) -> c_int;
        pub fn raw_read(
            fd: c_int,
            buf: *mut u8,
            len: usize,
        ) -> isize;
    }
}

// Safe wrapper (public)
pub struct FileHandle(std::ffi::c_int);

impl FileHandle {
    pub fn open(path: &str) -> Result<Self, Error> {
        let c_path = CString::new(path)?;
        let fd = unsafe { ffi::raw_open(c_path.as_ptr()) };
        if fd < 0 {
            return Err(Error::Open(path.to_owned()));
        }
        Ok(Self(fd))
    }

    pub fn read(&self, buf: &mut [u8]) -> Result<usize, Error> {
        let n = unsafe {
            ffi::raw_read(self.0, buf.as_mut_ptr(), buf.len())
        };
        if n < 0 {
            return Err(Error::Read);
        }
        Ok(n as usize)
    }
}

impl Drop for FileHandle {
    fn drop(&mut self) {
        unsafe { ffi::raw_close(self.0); }
    }
}
```

## Common Mistakes

1. **Using Rust `String` across FFI** — use `CString`/`CStr`; Rust strings aren't null-terminated
2. **Dropping `CString` while C still uses the pointer** — pointer dangles after drop
3. **Panicking across FFI** — catch panics at the boundary or use `extern "C-unwind"`
4. **Wrong `repr`** — always `#[repr(C)]` for types shared with C
5. **Platform-dependent sizes** — use `c_int`/`c_long` not `i32`/`i64`
6. **Forgetting null checks** — C can pass null; check before `&*ptr`
7. **Missing `#[no_mangle]`** — C can't find the symbol without it
8. **Owning C-allocated memory** — don't `Box::from_raw` a pointer `malloc`'d by C; use C's `free`
