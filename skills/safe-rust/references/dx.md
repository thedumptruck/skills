# Developer Experience & Architecture

Writing idiomatic Rust means leveraging the standard library traits, standardizing project structures, and utilizing tooling.

## 1. Type-Driven Design

- **Newtype Pattern**: Wrap primitive types in single-element tuple structs to enforce type safety at compile time.
  ```rust
  struct UserId(u64);
  struct ProductId(u64);
  // Prevents accidentally passing a ProductId to a function expecting a UserId
  ```
- **PhantomData**: Use `std::marker::PhantomData` to tie lifetimes or types to structs that don't directly store them, crucial for typestate patterns and unsafe abstractions.

## 2. Standard Traits Implementations

Implement standard traits rather than creating custom named methods to ensure interoperability.

- **`Default`**: Implement `Default` instead of a custom `new_empty()` method if there is a logical zero-state.
- **`From` and `Into`**: Implement `From<T>` for fallible conversions. Implementing `From` automatically provides `Into`. This cleans up type conversions significantly.
- **`TryFrom` and `TryInto`**: For conversions that can fail.
- **`AsRef` and `AsMut`**: For cheap reference-to-reference conversions (e.g., `impl AsRef<Path> for Config`).
- **`Display`**: Implement `std::fmt::Display` for user-facing string representations instead of an `.as_string()` method.
- **`Debug`**: Always `#[derive(Debug)]` on public structs and enums.

## 3. Tooling and Linting

- **Clippy is Mandatory**: Treat `clippy` as a co-author. 
  - Add `#![warn(clippy::pedantic)]` to the top of `lib.rs` or `main.rs`.
  - Fix warnings rather than ignoring them. If you must ignore, use `#[allow(clippy::rule_name)]` and add a comment explaining *why* it is necessary.
- **Rustfmt**: Use standard `rustfmt`. Do not bike-shed formatting rules.
- **Cargo Make/Just**: Use a task runner (like `just` or `cargo-make`) to standardize common commands (linting, testing, building) across the team.

## 4. API Design

- **Public Interfaces**: Keep public APIs small. Hide internal complexity behind well-defined structs and traits.
- **Builder Pattern**: For structs with many optional fields, use the Builder pattern rather than passing a dozen arguments or forcing users to provide `None`.
- **Visibility**: Be explicit about visibility (`pub`, `pub(crate)`, `pub(super)`). Default to private.