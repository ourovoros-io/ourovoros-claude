---
name: rust-dependency-audit
description: Use when auditing Rust dependencies with cargo audit, cargo deny, or cargo vet. Use when checking for RUSTSEC advisories, license compliance, supply chain risks, or evaluating new dependencies in Cargo.toml.
---

# Rust Dependency Audit

## Overview

Audit Rust dependencies for security, licensing, and supply chain risks. Every dependency is code you trust -- verify that trust is warranted.

## When to Use

- Adding new dependencies
- Security audit of existing project
- CI/CD security gates
- License compliance review
- Investigating transitive dependencies
- After `cargo update`

**Not for:** Code-level security review (use `rust-security-audit`), performance evaluation

## Quick Reference

| Check | Tool |
|-------|------|
| Known vulnerabilities | `cargo audit` |
| Unsafe code usage | `cargo geiger` |
| License compliance | `cargo deny check licenses` |
| Dependency tree | `cargo tree` |
| Outdated deps | `cargo outdated` |
| Supply chain verification | `cargo vet` |
| `build.rs` review | Manual audit (highest priority) |

## build.rs: The Largest Attack Surface

Build scripts execute with **full privileges** during `cargo build`. They can read/write files, execute programs, make network requests, and access all environment variables. **There is no sandboxing.**

**Mitigation priorities:**
1. Audit `build.rs` files in every new dependency
2. Use `cargo vet` to track human audits
3. Prefer crates without `build.rs` when alternatives exist
4. Build in sandboxed environments (containers, CI with restricted network)
5. Use `cargo vendor` to download and inspect all dependencies offline

## Security Audit Workflow

```bash
# 1. Check for known vulnerabilities
cargo audit

# 2. Update advisory database
cargo audit fetch

# 3. Fix or acknowledge
cargo audit fix  # Auto-update if possible

# If can't update, document exception:
# .cargo/audit.toml
[advisories]
ignore = ["RUSTSEC-2023-0001"]  # Document why!
```

## Dependency Analysis

```bash
cargo tree                   # Full tree
cargo tree -i regex          # Why is X included?
cargo tree -d                # Duplicated versions
cargo tree -f "{p} {f}"     # Features enabled
cargo tree --depth 1         # Direct deps only
```

## License Compliance with cargo-deny

```toml
# deny.toml
[licenses]
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Zlib",
]
copyleft = "deny"

[bans]
multiple-versions = "warn"
deny = [
    { name = "openssl" },  # Prefer rustls
]

[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

```bash
cargo deny init  # Creates deny.toml
cargo deny check # Run all checks
```

## Supply Chain Verification with cargo-vet

```bash
cargo vet init
cargo vet                        # Check all deps are audited
cargo vet suggest                # Show unaudited deps by effort
cargo vet inspect CRATE VERSION  # Open source for review
cargo vet certify CRATE VERSION  # Record your audit
cargo vet import mozilla         # Trust another org's audits
```

**Delta audits** dramatically reduce review burden -- only review the diff between versions.

## Adding Dependencies Checklist

Before adding a new dependency:

1. **Necessity**: Can stdlib or existing deps solve this?
2. **`build.rs`**: Does it have a build script? Audit it.
3. **Maintenance**: Active maintainer? Recent commits?
4. **Quality**: Tests? CI? Documentation?
5. **Security**: `cargo audit` clean? Minimal unsafe? (`cargo geiger`)
6. **License**: Compatible with your project? (`cargo deny`)
7. **Size**: Minimal features needed?
8. **Feature flags**: Are they additive? (enabling a feature should never break existing code)

```toml
# Prefer minimal features
[dependencies]
tokio = { version = "1", features = ["rt", "net"] }  # Not "full"
serde = { version = "1", features = ["derive"] }
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }
```

## Offline Inspection

```bash
# Download all dependencies for offline review
cargo vendor

# Inspect a specific crate's source
ls vendor/suspicious-crate/
cat vendor/suspicious-crate/build.rs  # Priority review target
```

## CI Integration

```yaml
# .github/workflows/security.yml
name: Security Audit
on:
  push:
  schedule:
    - cron: '0 0 * * *'  # Daily

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          persist-credentials: false
      - uses: rustsec/audit-check@69366f33c96575abad1ee0dba8212993eecbe998  # v2.0.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

  deny:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          persist-credentials: false
      - uses: EmbarkStudios/cargo-deny-action@34899fc7ba81ca6268d5947a7a16b4649013fea1  # v2.0.11
```

## Red Flags

| Signal | Risk | Action |
|--------|------|--------|
| Has `build.rs` | Code execution at build time | Audit the build script |
| No recent commits (2+ years) | Unmaintained | Find alternative |
| Single maintainer | Bus factor | Evaluate criticality |
| No tests/CI | Quality concerns | Review carefully |
| Many open security issues | Unpatched vulns | Avoid or fork |
| Yanked versions | Possible compromise | Investigate |
| `features = ["full"]` | Bloat, attack surface | Enable only needed features |

## Update Strategy

```bash
cargo outdated                    # See what's outdated
cargo update -p specific-crate   # Update conservatively
cargo update                     # Update all (test thoroughly)
cargo upgrade --dry-run           # Check for breaking changes
```

## Essential Tools

- `cargo-audit` -- Vulnerability database check
- `cargo-deny` -- License, advisory, ban, source enforcement
- `cargo-vet` -- Supply chain verification with delta audits
- `cargo-geiger` -- Unsafe code counter
- `cargo-outdated` -- Version freshness
- `cargo-tree` -- Dependency visualization
- `cargo vendor` -- Offline dependency inspection
