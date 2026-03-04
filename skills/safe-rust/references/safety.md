# Memory & Safety

Rust's core value proposition is memory safety without garbage collection. This is achieved through ownership, borrowing, lifetimes, and strict compile-time checks.

## 1. Ownership and Borrowing

- **References First**: Always prefer accepting references (`&T` or `&mut T`) over taking ownership (`T`) unless the function absolutely needs to consume the value or transfer ownership.
  - **Good**: `fn process(data: &Data)`
  - **Bad**: `fn process(data: Data)` (if `data` isn't consumed)
- **Slice Types**: When accepting collections, use slice types instead of specific owned types.
  - **Good**: `fn print_words(words: &[String])` or better `fn print_words(words: &[&str])`
  - **Bad**: `fn print_words(words: &Vec<String>)`
  - **Good**: `fn handle_string(s: &str)`
  - **Bad**: `fn handle_string(s: &String)`
- **Borrow Checker Respect**: Do not fight the borrow checker. If you find yourself adding complex lifetimes or using `Rc<RefCell<T>>` extensively, your architecture might need rethinking. Consider restructuring data to form a clear tree hierarchy.

## 2. Error Handling

- **No Panics**: Production code should almost never panic. Avoid `unwrap()`, `expect()`, `unreachable!()`, and direct array indexing (`arr[i]`).
  - Use `.get(i)` for arrays/slices which returns an `Option`.
  - Use `?` operator to propagate errors.
- **Library vs. Application Errors**:
  - **Libraries**: Define explicit error `enum`s. Use the `thiserror` crate to avoid boilerplate. This allows consumers to match on specific error cases.
  - **Applications**: Use the `anyhow` crate for easy, context-rich error propagation where the specific error type is less important than the trace.
- **Context is Key**: When propagating errors in applications, use `.context("Failed to perform action X")` to provide actionable logs.

## 3. Unsafe Boundaries

- **Avoid Unless Necessary**: Only use `unsafe` for FFI (Foreign Function Interface) calls, interacting with the OS, or proven, critical performance bottlenecks where safe abstractions fail.
- **Document Invariants**: EVERY `unsafe` block MUST have a `// SAFETY:` comment directly preceding it explaining why the operation is sound and what invariants the programmer guarantees that the compiler cannot check.
- **Encapsulate**: Keep `unsafe` blocks as small as possible and wrap them in safe API boundaries so consumers do not need to use `unsafe`.

## 4. Initialization and State

- **No Uninitialized Variables**: Rust enforces initialization, but avoid logical states representing "uninitialized" (e.g., using `Option<T>` solely because a struct is partially built). Use the Builder pattern to ensure a struct is fully valid upon creation.
- **Typestates**: Encode the state machine of your objects into the type system to make invalid states unrepresentable and uncompilable.