# Performance Patterns

Rust enables zero-cost abstractions, but achieving maximum performance requires understanding allocations and memory layouts.

## 1. Allocation Minimization

- **Clone Consciously**: Never sprinkle `.clone()` just to get code to compile. Understand why the compiler is complaining and fix the ownership semantics. `Clone` can be expensive.
- **Copy over Clone**: For small, primitive-like data types, derive `Copy`. Passing a `Copy` type is often faster than passing a reference due to pointer indirection.
- **Pre-allocation**: When building collections where the final size is known or estimable, use `Vec::with_capacity(n)` and `String::with_capacity(n)` to avoid reallocations.
- **Cow (Clone on Write)**: Use `std::borrow::Cow` when a function usually returns a borrowed reference but sometimes needs to return an allocated modified version. This avoids allocations in the fast path.

## 2. Concurrency Primitives

- **Prefer Message Passing**: Follow the Go proverb: "Do not communicate by sharing memory; instead, share memory by communicating." Use channels (`std::sync::mpsc` or `crossbeam_channel`) to pass data between threads.
- **Lock Contention**: When using `Mutex`, keep the lock held for the shortest possible duration. Do not perform heavy I/O or long computations while holding a lock.
- **RwLock over Mutex**: If a shared resource has many readers and few writers, prefer `std::sync::RwLock` over `Mutex` to increase concurrency.
- **Atomic Operations**: For simple counters or flags, use `std::sync::atomic` types instead of a `Mutex` to avoid locking overhead.

## 3. Memory Layout

- **Cache Locality**: Vectors (`Vec<T>`) store data contiguously in memory, which is highly cache-friendly. Prefer `Vec` over `LinkedList` or tree-based structures for iteration unless insertion/deletion characteristics strictly demand otherwise.
- **Struct Packing**: The compiler optimizes struct layouts automatically, but grouping fields of similar sizes can minimize padding.
- **Enum Sizes**: The size of an `enum` is determined by its largest variant. Avoid giant enum variants. Box large variants (`Variant(Box<LargeStruct>)`) to keep the overall enum size small.

## 4. Zero-Cost Abstractions

- **Generics vs. Trait Objects**: 
  - Prefer Generics with trait bounds (`fn do_thing<T: Trait>(t: T)`) which utilize static dispatch (monomorphization) - zero runtime overhead but larger binary size.
  - Use Trait Objects (`fn do_thing(t: &dyn Trait)`) only when heterogeneous collections are needed or to drastically reduce compile times/binary size. This uses dynamic dispatch (vtable overhead).
- **Inline Functions**: Use `#[inline]` for very small, frequently called functions to suggest to the compiler that the function body should be expanded at the call site, avoiding function call overhead. Use `#[inline(always)]` sparingly.