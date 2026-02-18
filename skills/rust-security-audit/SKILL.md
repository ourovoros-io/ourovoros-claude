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

**Not for:** Low-level unsafe/FFI review (use `rust-unsafe-audit`), general code quality (use `rust-idiomatic-patterns`)

## Quick Reference

| Risk | Pattern | Fix |
|------|---------|-----|
| Input injection | String interpolation in queries | Parameterized queries, validate input |
| Secret exposure | Hardcoded credentials | Env vars, `secrecy` crate |
| Path traversal | User input in file paths | Canonicalize, validate against allowlist |
| Integer overflow | Unchecked arithmetic on input | `checked_add`, `saturating_mul` |
| Timing attacks | Early return on auth failure | Constant-time comparison |
| Insecure random | `rand::thread_rng()` for crypto | `rand::rngs::OsRng` |
| Weak password hash | SHA-256/MD5 for passwords | `argon2` or `bcrypt` with salt |
| User enumeration | Different errors for not-found vs wrong-password | Same error message for both |
| TOCTOU race | Check then use on file/resource | Atomic operations, lock-and-use |
| Unbounded reads | Reading untrusted data without limit | `AsyncReadExt::take()`, size limits |
| DoS via resource exhaustion | No rate limiting on endpoints | Semaphore, token bucket, governor |

## Security Patterns

### Input Validation

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

### TOCTOU (Time-of-Check-Time-of-Use)

```rust
// BAD: Race between check and use
fn write_if_missing(path: &Path, data: &[u8]) -> Result<(), Error> {
    if !path.exists() {        // Check
        std::fs::write(path, data)?;  // Use — file may now exist!
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

// GOOD: For async resources, lock then check
async fn claim_slot(slots: &DashMap<u64, Claim>) -> Result<u64, Error> {
    let slot_id = find_free_slot();
    match slots.entry(slot_id) {
        dashmap::mapref::entry::Entry::Vacant(e) => {
            e.insert(Claim::new());
            Ok(slot_id)
        }
        dashmap::mapref::entry::Entry::Occupied(_) => {
            Err(Error::SlotTaken)
        }
    }
}
```

### Limiting Untrusted Reads

```rust
use tokio::io::AsyncReadExt;

// BAD: Read unlimited data from untrusted source
async fn read_request(stream: &mut TcpStream) -> Result<Vec<u8>, Error> {
    let mut buf = Vec::new();
    stream.read_to_end(&mut buf).await?;  // OOM if attacker sends GBs
    Ok(buf)
}

// GOOD: Limit read size
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

### Rate Limiting / DoS Prevention

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

struct Server {
    request_semaphore: Arc<Semaphore>,  // Max concurrent requests
}

impl Server {
    async fn handle(&self, req: Request) -> Result<Response, Error> {
        let _permit = self.request_semaphore
            .try_acquire()
            .map_err(|_| Error::TooManyRequests)?;  // 429 immediately

        process(req).await
    }
}

// For per-client rate limiting, use governor crate:
// use governor::{Quota, RateLimiter};
// let limiter = RateLimiter::direct(Quota::per_second(nonzero!(50u32)));
```

### Secrets Handling

```rust
// BAD: Secrets in memory as plain String
let api_key = std::env::var("API_KEY")?;

// GOOD: Use secrecy crate — zeroized on drop, won't appear in Debug/Display
use secrecy::{Secret, ExposeSecret};
let api_key: Secret<String> = Secret::new(std::env::var("API_KEY")?);
// Only expose when needed:
client.set_header("Authorization", api_key.expose_secret());
```

### Constant-Time Comparison

```rust
// BAD: Early return leaks timing info
fn verify_token(provided: &str, expected: &str) -> bool {
    provided == expected  // Short-circuits on first mismatch
}

// GOOD: Constant-time comparison
use subtle::ConstantTimeEq;
fn verify_token(provided: &[u8], expected: &[u8]) -> bool {
    provided.ct_eq(expected).into()
}
```

### Password Hashing

```rust
use argon2::{Argon2, PasswordHasher, PasswordVerifier};
use argon2::password_hash::SaltString;
use rand::rngs::OsRng;

fn hash_password(password: &str) -> Result<String, Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    Ok(argon2.hash_password(password.as_bytes(), &salt)?.to_string())
}

fn verify_password(password: &str, hash: &str) -> Result<bool, Error> {
    let parsed_hash = PasswordHash::new(hash)?;
    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}
```

### Authentication Errors

```rust
// BAD: Leaks whether user exists
fn login(username: &str, password: &str) -> Result<User, Error> {
    let user = db::find_user(username)
        .ok_or(Error::UserNotFound)?;        // Reveals existence
    verify_password(password, &user.hash)
        .map_err(|_| Error::WrongPassword)?;  // Different error
    Ok(user)
}

// GOOD: Same error for both cases
fn login(username: &str, password: &str) -> Result<User, Error> {
    let user = db::find_user(username);
    let valid = user
        .as_ref()
        .map(|u| verify_password(password, &u.hash).unwrap_or(false))
        .unwrap_or(false);

    if valid {
        Ok(user.expect("verified above"))
    } else {
        Err(Error::InvalidCredentials)  // Same error for both
    }
}
```

## Audit Checklist

1. **Inputs**: All external data validated before use
2. **Size limits**: Untrusted reads bounded (`take()`, max sizes)
3. **TOCTOU**: No check-then-use races on shared resources
4. **Secrets**: No hardcoded credentials, proper zeroization
5. **Crypto**: Using audited crates (ring, rustcrypto), proper RNG
6. **Errors**: No sensitive data in error messages
7. **Logging**: No secrets logged, PII redacted
8. **Rate limits**: Endpoints protected against resource exhaustion
9. **Dependencies**: Security advisories checked (`cargo audit`)

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|--------------|-----|
| `unwrap()` on user input | DoS via panic | Return error, validate first |
| SQL string formatting | Injection | Query builder, parameterize |
| Logging request bodies | Credential leak | Redact sensitive fields |
| `Default` for secrets | Predictable values | Require explicit initialization |
| Ignoring `#[must_use]` | Security check bypassed | Handle all Results |
| Unbounded `read_to_end` | OOM from malicious input | `take()` with size limit |
| Check-then-act on files | TOCTOU race | Atomic operations |

## Crates for Security

- `secrecy` - Secret wrapper with zeroization
- `subtle` - Constant-time operations
- `zeroize` - Secure memory clearing
- `ring` / `rustcrypto` - Audited cryptography
- `validator` - Input validation derive macros
- `argon2` / `bcrypt` - Password hashing
- `governor` - Rate limiting
