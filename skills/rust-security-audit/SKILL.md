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
| Input injection | String interpolation in queries | Use parameterized queries, validate input |
| Secret exposure | Hardcoded credentials | Use env vars, `secrecy` crate |
| Path traversal | User input in file paths | Canonicalize, validate against allowlist |
| Integer overflow | Unchecked arithmetic on input | `checked_add`, `saturating_mul` |
| Timing attacks | Early return on auth failure | Constant-time comparison |
| Insecure random | `rand::thread_rng()` for crypto | `rand::rngs::OsRng` |
| Weak password hash | SHA-256/MD5 for passwords | Use `argon2` or `bcrypt` with salt |
| User enumeration | Different errors for "not found" vs "wrong password" | Same error message for both |

## Security Patterns

### Input Validation

```rust
// BAD: Trust user input
fn process(input: &str) -> Result<(), Error> {
    let path = PathBuf::from(input);
    std::fs::read(path)?;
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

### Secrets Handling

```rust
// BAD: Secrets in memory as plain String
let api_key = std::env::var("API_KEY")?;

// GOOD: Use secrecy crate
use secrecy::{Secret, ExposeSecret};
let api_key: Secret<String> = Secret::new(std::env::var("API_KEY")?);
// Only expose when needed, zeroized on drop
```

### Constant-Time Comparison

```rust
// BAD: Early return leaks timing info
fn verify_token(provided: &str, expected: &str) -> bool {
    provided == expected  // Short-circuits
}

// GOOD: Constant-time comparison
use subtle::ConstantTimeEq;
fn verify_token(provided: &[u8], expected: &[u8]) -> bool {
    provided.ct_eq(expected).into()
}
```

### Password Hashing

```rust
// BAD: Fast hash without salt
fn hash_password(password: &str) -> String {
    sha256(password)  // Too fast, no salt, rainbow table vulnerable
}

// GOOD: Use argon2 with salt
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
    Ok(Argon2::default().verify_password(password.as_bytes(), &parsed_hash).is_ok())
}
```

### Authentication Errors

```rust
// BAD: Leaks whether user exists
fn login(username: &str, password: &str) -> Result<User, Error> {
    let user = db::find_user(username)
        .ok_or(Error::UserNotFound)?;  // Reveals user existence
    verify_password(password, &user.hash)
        .map_err(|_| Error::WrongPassword)?;  // Different error
    Ok(user)
}

// GOOD: Same error for both cases
fn login(username: &str, password: &str) -> Result<User, Error> {
    let user = db::find_user(username);
    let valid = user.as_ref()
        .map(|u| verify_password(password, &u.hash).unwrap_or(false))
        .unwrap_or(false);

    if valid {
        Ok(user.unwrap())
    } else {
        Err(Error::InvalidCredentials)  // Same error for both
    }
}
```

## Audit Checklist

1. **Inputs**: All external data validated before use
2. **Secrets**: No hardcoded credentials, proper zeroization
3. **Crypto**: Using audited crates (ring, rustcrypto), proper RNG
4. **Errors**: No sensitive data in error messages
5. **Logging**: No secrets logged, PII redacted
6. **Dependencies**: Security advisories checked (cargo-audit)

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|--------------|-----|
| `unwrap()` on user input | DoS via panic | Return error, validate first |
| SQL string formatting | Injection | Use query builder, parameterize |
| Logging request bodies | Credential leak | Redact sensitive fields |
| `Default` for secrets | Predictable values | Require explicit initialization |
| Ignoring `#[must_use]` | Security check bypassed | Handle all Results |

## Crates for Security

- `secrecy` - Secret wrapper with zeroization
- `subtle` - Constant-time operations
- `zeroize` - Secure memory clearing
- `ring` / `rustcrypto` - Audited cryptography
- `validator` - Input validation derive macros
- `argon2` / `bcrypt` - Password hashing
