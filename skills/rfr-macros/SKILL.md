---
name: rfr-macros
description: Use when writing or debugging Rust macros, both declarative (macro_rules!) and procedural (derive, attribute, function-like). Use when choosing between macro types, understanding fragment specifiers, debugging macro expansion, or working with syn/quote.
---

# Rust for Rustaceans — Ch 7: Macros

## When to Use Macros

### Use Macros When
- You need code generation that generics/traits can't express
- Implementing repetitive trait impls for many types
- Creating DSLs (domain-specific languages)
- Reducing boilerplate that would otherwise require copy-paste

### Don't Use Macros When
- Generics or traits solve the problem — prefer them
- A function would work — macros are harder to debug
- The "boilerplate" is only 2-3 repetitions — just write it out

**Rule: Macros are a last resort, not a first choice.**

## Declarative Macros (`macro_rules!`)

### Basic Structure
```rust
macro_rules! my_vec {
    // Pattern => expansion
    ( $( $element:expr ),* $(,)? ) => {
        {
            let mut v = Vec::new();
            $( v.push($element); )*
            v
        }
    };
}

let v = my_vec![1, 2, 3];
```

### Fragment Specifiers

| Specifier | Matches | Example |
|-----------|---------|---------|
| `$e:expr` | Any expression | `42`, `x + 1`, `foo()` |
| `$t:ty` | A type | `i32`, `Vec<u8>`, `&str` |
| `$i:ident` | An identifier | `foo`, `my_var` |
| `$p:pat` | A pattern | `Some(x)`, `_`, `1..=5` |
| `$s:stmt` | A statement | `let x = 1;` |
| `$b:block` | A block | `{ foo(); bar() }` |
| `$l:lifetime` | A lifetime | `'a`, `'static` |
| `$m:meta` | Attribute content | `derive(Debug)`, `cfg(test)` |
| `$tt:tt` | A single token tree | Any token or `(...)`, `[...]`, `{...}` |
| `$item:item` | An item (fn, struct, impl, etc.) | `fn foo() {}` |
| `$vis:vis` | Visibility | `pub`, `pub(crate)`, `` (empty) |
| `$lit:literal` | A literal | `42`, `"hello"`, `true` |

### Repetitions
```rust
macro_rules! impl_display {
    // Match multiple type-format pairs
    ( $( $Type:ty => $fmt:expr ),* $(,)? ) => {
        $(
            impl std::fmt::Display for $Type {
                fn fmt(
                    &self,
                    f: &mut std::fmt::Formatter<'_>,
                ) -> std::fmt::Result {
                    write!(f, $fmt, self.0)
                }
            }
        )*
    };
}

impl_display! {
    UserId => "user:{}",
    GroupId => "group:{}",
}
```

### Repetition Operators

| Operator | Meaning |
|----------|---------|
| `$( ... )*` | Zero or more |
| `$( ... )+` | One or more |
| `$( ... )?` | Zero or one |
| `$( ... ),*` | Zero or more, comma-separated |
| `$( ... ),+` | One or more, comma-separated |

### Multiple Arms
```rust
macro_rules! log {
    // No args
    ($msg:expr) => {
        eprintln!("[LOG] {}", $msg);
    };
    // With key-value pairs
    ($msg:expr, $( $key:ident = $val:expr ),+ ) => {
        eprint!("[LOG] {}", $msg);
        $( eprint!(" {}={}", stringify!($key), $val); )+
        eprintln!();
    };
}
```

### Declarative Macro Rules
1. **Arms are tried top-to-bottom** — put specific patterns before general ones
2. **Macros are hygienic** — variables inside don't leak, variables outside aren't captured
3. **`$crate`** — refers to the crate defining the macro (use for re-exports)
4. **Trailing comma** — always support optional trailing comma with `$(,)?`

## Procedural Macros

**Must live in a separate crate with `proc-macro = true`.**

### Three Types

| Type | Syntax | Input | Output |
|------|--------|-------|--------|
| Derive | `#[derive(MyMacro)]` | Struct/enum definition | Additional impls |
| Attribute | `#[my_attr]` | Any item | Transformed item |
| Function-like | `my_macro!(...)` | Any tokens | Any tokens |

### Derive Macro
```rust
// my_derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(MyDebug)]
pub fn derive_my_debug(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;

    let expanded = quote! {
        impl std::fmt::Debug for #name {
            fn fmt(
                &self,
                f: &mut std::fmt::Formatter<'_>,
            ) -> std::fmt::Result {
                write!(f, stringify!(#name))
            }
        }
    };

    TokenStream::from(expanded)
}
```

### Attribute Macro
```rust
#[proc_macro_attribute]
pub fn route(
    attr: TokenStream,  // The attribute arguments
    item: TokenStream,  // The annotated item
) -> TokenStream {
    // Parse, transform, return
    let input = parse_macro_input!(item as syn::ItemFn);
    let method = attr.to_string();
    // ... generate routing code
    TokenStream::from(expanded)
}

// Usage:
// #[route("GET")]
// fn index() -> Response { ... }
```

### Function-like Macro
```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    let query = input.to_string();
    // Parse SQL at compile time, generate typed query
    // ...
    TokenStream::from(expanded)
}

// Usage: sql!(SELECT * FROM users WHERE id = ?)
```

### Key Crates

| Crate | Purpose |
|-------|---------|
| `proc-macro` | Compiler interface (TokenStream) |
| `syn` | Parse Rust syntax into AST |
| `quote` | Generate Rust code from templates |
| `proc-macro2` | Wrapper for proc-macro that works in unit tests |

### `syn` Parsing
```rust
use syn::{DeriveInput, Data, Fields};

let input = parse_macro_input!(input as DeriveInput);

// Access struct fields
if let Data::Struct(data) = &input.data {
    if let Fields::Named(fields) = &data.fields {
        for field in &fields.named {
            let name = &field.ident;
            let ty = &field.ty;
            // Generate code per field
        }
    }
}
```

### `quote` Code Generation
```rust
use quote::quote;

let name = &input.ident;
let fields = get_field_names(&input);

let expanded = quote! {
    impl #name {
        fn field_names() -> &'static [&'static str] {
            &[ #( stringify!(#fields) ),* ]
        }
    }
};
```

## Macro Hygiene

### Declarative Macros
- Variables defined inside the macro are scoped to the macro
- Use `$crate::` to reference items from your crate
- Callers' variables don't interfere with macro variables

```rust
macro_rules! create_map {
    ( $( $key:expr => $val:expr ),* ) => {
        {
            // This `map` doesn't conflict with caller's `map`
            let mut map = std::collections::HashMap::new();
            $( map.insert($key, $val); )*
            map
        }
    };
}
```

### Procedural Macros
- NOT hygienic by default — generated code shares caller's namespace
- Use full paths: `::std::vec::Vec` not `Vec`
- Use `quote_spanned!` for error reporting at the right source location

## Debugging Macros

### `cargo expand`
Shows the expanded code after all macros are applied:

```bash
cargo install cargo-expand
cargo expand              # Expand entire crate
cargo expand module_name  # Expand specific module
```

### `trace_macros!`
```rust
#![feature(trace_macros)]
trace_macros!(true);
my_macro!(args);
trace_macros!(false);
```

### Compile Error Messages
```rust
// In procedural macros, emit clear errors
compile_error!("expected a struct, got an enum");

// In proc macros with syn:
return syn::Error::new_spanned(
    input,
    "this derive only works on structs",
).to_compile_error().into();
```

## Common Mistakes

1. **Macro when function/generic suffices** — macros are harder to debug, test, and IDE-support
2. **Not supporting trailing commas** — always add `$(,)?`
3. **Missing `$crate::`** — paths break when the macro is used from another crate
4. **Procedural macro in same crate** — must be a separate `proc-macro` crate
5. **No `cargo expand` verification** — always check expanded code
6. **Bare names in proc macros** — use full paths (`::std::option::Option`) to avoid conflicts
7. **Poor error messages** — use `compile_error!` or `syn::Error::new_spanned` with clear text
8. **Overly complex macros** — if the macro is >50 lines, consider a proc macro instead
