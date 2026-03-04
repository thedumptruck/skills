---
name: safe-rust
description: Enforce "safe-rust" coding principles. Use when writing, reading, reviewing, or refactoring Rust code to ensure maximum memory safety, predictable execution, zero-cost abstractions, and idiomatic Rust patterns.
---

# Safe Rust

Build highly predictable, robust, and performant Rust applications with strict adherence to ownership, lifetimes, and type-driven safety.

## Retrieval-First Development

**Always verify standards against the reference documentation before implementing.**

| Resource | URL / Path |
|----------|------------|
| Memory & Safety | `./references/safety.md` |
| Performance Patterns | `./references/performance.md` |
| Developer Experience | `./references/dx.md` |

Review the relevant documentation when writing new logic or performing code reviews.

## When to Use

- Writing new Rust logic from scratch
- Refactoring existing Rust code to improve safety, performance, or idiomaticness
- Reviewing Rust PRs for code quality, borrow checker compliance, and standard adherence
- Optimizing memory allocations or hot paths
- Implementing strict error handling, type-state patterns, or zero-cost abstractions

## Reference Documentation

- `./references/safety.md` - Ownership, lifetimes, unsafe boundaries, error handling, unwrap/expect policies.
- `./references/performance.md` - Allocation minimization, `Cow`, references vs. values, concurrency primitives.
- `./references/dx.md` - Typestates, Newtype pattern, standard library traits (`From`, `AsRef`), Clippy lints.

Search: `unwrap`, `unsafe`, `clippy::pedantic`, `Cow`, `Result`, `Rc`, `Arc`, `Mutex`

## Core Principles

### Apply Safe Rust For

| Need | Example |
|------|---------|
| Compile-Time Safety | Typestate pattern to prevent invalid state transitions |
| Memory Stability | Passing by reference (`&T`), reusing allocations, `Cow<T>` |
| Operational Reliability | Exhaustive pattern matching, `Result` over `panic!`, `thiserror`/`anyhow` |
| Maintainability | Implement `From`, `TryFrom`, `AsRef`, `Display`; document `unsafe` |

### Do NOT Use

- `unwrap()` or `expect()` in production code (unless proving mathematically impossible to fail)
- `unsafe` without a heavily documented `// SAFETY:` comment explaining invariants
- Unnecessary `.clone()` or allocations (`String`, `Vec`) in hot paths when `&str` or `&[T]` works
- `Rc<RefCell<T>>` or `Arc<Mutex<T>>` as the first tool for state sharing (prefer structural borrowing or message passing)

## Quick Reference

### Typestate Pattern

```rust
// Use the type system to enforce valid state transitions at compile time
struct Unverified;
struct Verified;

struct Email<State> {
    address: String,
    _state: std::marker::PhantomData<State>,
}

impl Email<Unverified> {
    fn new(address: String) -> Self {
        Self { address, _state: std::marker::PhantomData }
    }

    fn verify(self) -> Result<Email<Verified>, &'static str> {
        if self.address.contains('@') {
            Ok(Email { address: self.address, _state: std::marker::PhantomData })
        } else {
            Err("Invalid email format")
        }
    }
}

// This function only accepts verified emails
fn send_welcome(email: &Email<Verified>) {
    // ...
}
```

### Allocation-Free Hot Path (Cow)

```rust
use std::borrow::Cow;

// Returns a reference if no changes needed, or an allocated String if modified
fn sanitize_input<'a>(input: &'a str) -> Cow<'a, str> {
    if input.contains('\0') {
        let mut cleaned = input.replace('\0', "");
        Cow::Owned(cleaned)
    } else {
        Cow::Borrowed(input)
    }
}
```

## Critical Rules

1. **No Panics in Production** - Avoid `unwrap()`, `expect()`, array indexing `arr[i]` (prefer `arr.get(i)`). Handle all errors gracefully via `Result` and `?`.
2. **Minimal and Documented `unsafe`** - If `unsafe` is absolutely necessary for FFI or extreme performance, every `unsafe` block must be preceded by a `// SAFETY: ...` comment proving the invariants.
3. **Minimize Allocations** - Prefer passing `&T`, `&[T]`, and `&str` over `T`, `Vec<T>`, and `String` for function arguments unless ownership is required.
4. **Leverage the Type System** - Use the Newtype pattern (e.g., `struct UserId(u64)`) to prevent unit confusion. Use typestates for state machines.
5. **Strict Clippy Compliance** - Treat `#![warn(clippy::pedantic)]` as the baseline. Explicitly `#[allow(...)]` with comments if deviating.
6. **Thread Safety via Message Passing or Ownership** - Share state via channels (`mpsc` or `crossbeam`) or `Arc<T>` (read-only) rather than defaulting to `Arc<Mutex<T>>`.
7. **Idiomatic Traits** - Implement standard traits like `Default`, `From`, `TryFrom`, `AsRef`, and `Display` instead of custom conversion/instantiation methods.
8. **Error Types** - Use `thiserror` for library error types (to provide explicit `enum` variants) and `anyhow` for application entry points/handlers.
9. **Zero-Cost Abstractions** - Use generics and inline functions `#[inline]` for hot paths to allow the compiler to optimize without runtime overhead.
10. **Exhaustive Matches** - Never use the wildcard `_` in a `match` statement on an `enum` unless explicitly intending to ignore all future variants. Force the compiler to check exhaustiveness.

## Anti-Patterns (NEVER)

- Sprinkling `.clone()` to appease the borrow checker instead of fixing lifetimes/architecture.
- Using `unsafe` just to bypass borrow checker limitations without sound reasoning.
- Catching panics (`catch_unwind`) as a general error handling mechanism.
- Storing references with complex lifetimes in structs when ownership makes more sense for API usability.
- Ignoring Clippy warnings without justification.
- Returning `String` or `Vec` from parsing functions that could just return slices (`&str`, `&[T]`) borrowed from the input.