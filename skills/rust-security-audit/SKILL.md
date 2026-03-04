---
name: rust-security-audit
description: Use when reviewing Rust code that handles user input, authentication, secrets, cryptography, or network boundaries. Use when code processes untrusted data or when security vulnerabilities could lead to data breaches.
---

# Rust Security Audit

## Overview

Review Rust code for application-level security vulnerabilities. Focus on data validation, secrets handling, and secure coding patterns that prevent common attack vectors.

## When to Use

- Code handles user input or external data
- Authentication/authorization logic
- Secrets, API keys, or credentials in code
- Cryptographic operations
- Network request/response handling
- Database queries with dynamic input
- File operations with user-provided paths
- Adding or reviewing dependencies

**Not for:** Low-level unsafe/FFI review (use `rust-unsafe-audit`), general code quality (use `rust-idiomatic-patterns`)

## Quick Reference

| Risk | Pattern | Fix |
|------|---------|-----|
| Input injection | String interpolation in queries | Parameterized queries, validate input |
| Secret exposure | Hardcoded credentials | Env vars, `secrecy` crate |
| Path traversal | User input in file paths | Canonicalize, validate against allowlist |
| Integer overflow | Unchecked arithmetic on input | `checked_add`, `saturating_mul` |
| Timing attacks | Early return on auth failure | Constant-time comparison (`subtle`) |
| Insecure random | `rand::thread_rng()` for crypto | `rand::rngs::OsRng` |
| Weak password hash | SHA-256/MD5 for passwords | `argon2` with salt |
| User enumeration | Different errors for not-found vs wrong-password | Same error message for both |
| TOCTOU race | Check then use on file/resource | Atomic operations, lock-and-use |
| Unbounded reads | Reading untrusted data without limit | `AsyncReadExt::take()`, size limits |
| DoS via exhaustion | No rate limiting on endpoints | Semaphore, token bucket, `governor` |
| `build.rs` code execution | Malicious dependency build script | Audit `build.rs`, use `cargo vet` |

## Integer Overflow in Release Mode

In debug mode, Rust panics on integer overflow. **In release mode, it wraps silently.** This is not UB but causes logic bugs:

```rust
// VULNERABLE: wraps to small number in release
fn allocate_buffer(count: usize, item_size: usize) -> Vec<u8> {
    let size = count * item_size;  // can wrap to small value
    vec![0u8; size]  // too-small buffer
}

// SAFE: checked arithmetic
fn allocate_buffer(count: usize, item_size: usize) -> Result<Vec<u8>, Error> {
    let size = count.checked_mul(item_size)
        .ok_or(Error::IntegerOverflow)?;
    Ok(vec![0u8; size])
}
```

For security-critical code, enable overflow checks in release: `[profile.release] overflow-checks = true`

## Input Validation

```rust
// BAD: Trust user input
fn process(input: &str) -> Result<(), Error> {
    let path = PathBuf::from(input);
    std::fs::read(path)?;
    Ok(())
}

// GOOD: Validate and sanitize
fn process(input: &str) -> Result<(), Error> {
    let path = PathBuf::from(input).canonicalize()?;
    if !path.starts_with("/allowed/directory") {
        return Err(Error::InvalidPath);
    }
    std::fs::read(path)?;
    Ok(())
}
```

## TOCTOU (Time-of-Check-Time-of-Use)

```rust
// BAD: Race between check and use
fn write_if_missing(path: &Path, data: &[u8]) -> Result<(), Error> {
    if !path.exists() {
        std::fs::write(path, data)?;  // File may now exist!
    }
    Ok(())
}

// GOOD: Atomic operation
use std::fs::OpenOptions;
fn write_if_missing(path: &Path, data: &[u8]) -> Result<(), Error> {
    let mut file = OpenOptions::new()
        .write(true)
        .create_new(true)  // Atomic: fails if exists
        .open(path)?;
    file.write_all(data)?;
    Ok(())
}
```

## Limiting Untrusted Reads

```rust
const MAX_REQUEST_SIZE: u64 = 10 * 1024 * 1024; // 10 MiB

async fn read_request(stream: &mut TcpStream) -> Result<Vec<u8>, Error> {
    let mut buf = Vec::new();
    let bytes_read = stream
        .take(MAX_REQUEST_SIZE)
        .read_to_end(&mut buf)
        .await?;

    if bytes_read as u64 >= MAX_REQUEST_SIZE {
        return Err(Error::RequestTooLarge);
    }
    Ok(buf)
}
```

## Secrets Handling

```rust
use secrecy::{Secret, ExposeSecret};

// Zeroized on drop, won't appear in Debug/Display
let api_key: Secret<String> = Secret::new(std::env::var("API_KEY")?);
client.set_header("Authorization", api_key.expose_secret());
```

`secrecy` + `zeroize`: prevents accidental logging, clears memory on drop via `write_volatile`.

**Limitation:** Moves, copies, and heap reallocations can leave residual copies. For defense-in-depth, consider `mlock(2)` to prevent swapping.

## Constant-Time Comparison

```rust
use subtle::ConstantTimeEq;

// BAD: Short-circuits on first mismatch, leaks timing info
fn verify_token(provided: &str, expected: &str) -> bool {
    provided == expected
}

// GOOD: Constant-time, no timing side channel
fn verify_token(provided: &[u8], expected: &[u8]) -> bool {
    provided.ct_eq(expected).into()
}
```

**Always verify in release mode** -- debug builds may contain secret-dependent branches from debug assertions.

## Password Hashing

```rust
use argon2::{Argon2, PasswordHasher, PasswordVerifier};
use argon2::password_hash::SaltString;
use rand::rngs::OsRng;

fn hash_password(password: &str) -> Result<String, Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    Ok(argon2.hash_password(password.as_bytes(), &salt)?.to_string())
}
```

## Authentication Errors

```rust
// GOOD: Same error for user-not-found and wrong-password
fn login(username: &str, password: &str) -> Result<User, Error> {
    let user = db::find_user(username);
    let valid = user
        .as_ref()
        .map(|u| verify_password(password, &u.hash).unwrap_or(false))
        .unwrap_or(false);

    if valid {
        Ok(user.expect("verified above"))
    } else {
        Err(Error::InvalidCredentials)
    }
}
```

## `build.rs` as Attack Surface

Build scripts execute with full privileges during `cargo build` and `cargo check`. They can:
- Read/write arbitrary files (SSH keys, credentials)
- Execute programs, make network requests
- Access all environment variables

**Mitigations:**
1. Audit `build.rs` in every dependency (use `cargo vet`)
2. Prefer crates without `build.rs` when alternatives exist
3. Build in sandboxed environments (containers)
4. Use `cargo-deny` sources check to restrict to known registries

## Capability Design

```rust
// BAD: Ambient authority -- function reaches into environment
fn process_request(req: &Request) {
    let db = Database::connect_from_env();
}

// GOOD: Explicit capability passing
fn process_request(req: &Request, db: &Database) {
    // Can only access what was explicitly given
}

// GOOD: Separate traits per capability
trait StorageRead { fn read(&self, key: &str) -> Vec<u8>; }
trait StorageWrite { fn write(&mut self, key: &str, val: &[u8]); }
// Don't give write capability to code that only needs read
```

## Rate Limiting / DoS Prevention

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

struct Server {
    request_semaphore: Arc<Semaphore>,
}

impl Server {
    async fn handle(&self, req: Request) -> Result<Response, Error> {
        let _permit = self.request_semaphore
            .try_acquire()
            .map_err(|_| Error::TooManyRequests)?;
        process(req).await
    }
}
```

## Audit Checklist

1. **Inputs**: All external data validated before use
2. **Size limits**: Untrusted reads bounded (`take()`, max sizes)
3. **Integer arithmetic**: Checked/saturating on untrusted values
4. **TOCTOU**: No check-then-use races on shared resources
5. **Secrets**: No hardcoded credentials, proper zeroization
6. **Crypto**: Audited crates (`ring`, `rustcrypto`), proper RNG (`OsRng`)
7. **Errors**: No sensitive data in error messages
8. **Logging**: No secrets logged, PII redacted
9. **Rate limits**: Endpoints protected against resource exhaustion
10. **Dependencies**: `cargo audit` clean, `build.rs` reviewed
11. **Capabilities**: Minimal authority, explicit not ambient

## Crates for Security

- `secrecy` -- Secret wrapper with zeroization
- `subtle` -- Constant-time operations
- `zeroize` -- Secure memory clearing
- `ring` / `rustcrypto` -- Audited cryptography
- `validator` -- Input validation derive macros
- `argon2` -- Password hashing
- `governor` -- Rate limiting
- `cargo-audit` -- Vulnerability database check
- `cargo-vet` -- Supply chain verification
