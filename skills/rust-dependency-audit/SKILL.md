---
name: rust-dependency-audit
description: Use when auditing Rust dependencies with cargo audit, cargo deny, or cargo vet. Use when checking for RUSTSEC advisories, license compliance, supply chain risks, or evaluating new dependencies in Cargo.toml.
---

# Rust Dependency Audit

## Overview

Audit Rust dependencies for security, licensing, and supply chain risks. Every dependency is code you trust — verify that trust is warranted.

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
| License compliance | `cargo deny` |
| Dependency tree | `cargo tree` |
| Outdated deps | `cargo outdated` |
| Supply chain risks | `cargo vet` |

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

### View Dependency Tree

```bash
cargo tree                   # Full tree
cargo tree -i regex          # Why is X included?
cargo tree -d                # Duplicated versions
cargo tree -f "{p} {f}"     # Features enabled
cargo tree --depth 1         # Direct deps only
```

### Unsafe Code Analysis

```bash
cargo geiger
# Output shows unsafe usage per crate:
# Functions  Expressions  Impls  Traits  Methods
# 0/0        0/0          0/0    0/0     0/0      crate_name
```

## License Compliance with cargo-deny

```bash
cargo deny init  # Creates deny.toml
cargo deny check licenses
```

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

[licenses.exceptions]
ring = ["ISC", "MIT", "OpenSSL"]

[bans]
multiple-versions = "warn"
deny = [
    { name = "openssl" },  # Prefer rustls
]

[advisories]
vulnerability = "deny"
unmaintained = "warn"
```

## Supply Chain Verification with cargo-vet

```bash
cargo vet init
cargo vet

# Record audit after manual review
cargo vet certify serde 1.0.188

# Import trusted audits from Mozilla, Google, etc.
cargo vet import mozilla
```

## Adding Dependencies Checklist

Before adding a new dependency:

1. **Necessity**: Can stdlib or existing deps solve this?
2. **Maintenance**: Active maintainer? Recent commits?
3. **Popularity**: Downloads, dependents, stars?
4. **Quality**: Tests? CI? Documentation?
5. **Security**: `cargo audit` clean? Unsafe code minimal?
6. **License**: Compatible with your project?
7. **Size**: Minimal features needed?

```toml
# Prefer minimal features
[dependencies]
tokio = { version = "1", features = ["rt", "net"] }  # Not "full"
serde = { version = "1", features = ["derive"] }
reqwest = { version = "0.11", default-features = false, features = ["rustls-tls"] }
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
| No recent commits (2+ years) | Unmaintained | Find alternative |
| Single maintainer | Bus factor | Evaluate criticality |
| No tests/CI | Quality concerns | Review carefully |
| Many open security issues | Unpatched vulns | Avoid or fork |
| Yanked versions | Possible compromise | Investigate |
| Name squatting | Typosquatting | Verify exact name |
| Excessive permissions | Supply chain risk | Review build.rs |

## Common Mistakes

| Mistake | Issue | Fix |
|---------|-------|-----|
| `features = ["full"]` | Bloat, attack surface | Enable only needed |
| Ignoring `cargo audit` | Known vulns in prod | Run in CI, block on failure |
| Outdated lockfile | Missing security patches | Regular `cargo update` |
| No license check | Legal liability | Use `cargo deny` |
| Trusting all deps | Supply chain attack | Use `cargo vet` |
| `*` version | Breaking updates | Pin to semver range |
| Unpinned CI actions | Supply chain risk | Pin to SHA with version comment |

## Update Strategy

```bash
cargo outdated                    # See what's outdated
cargo update -p specific-crate   # Update conservatively
cargo update                     # Update all (test thoroughly)
cargo upgrade --dry-run           # Check for breaking changes (cargo-edit)
```

## Essential Tools

- `cargo-audit` - Vulnerability database check
- `cargo-deny` - License and ban enforcement
- `cargo-vet` - Supply chain verification
- `cargo-geiger` - Unsafe code counter
- `cargo-outdated` - Version freshness
- `cargo-tree` - Dependency visualization
- `cargo-edit` - Add/remove/upgrade deps
