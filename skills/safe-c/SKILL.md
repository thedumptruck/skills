---
name: safe-c
description: Enforce "safe-c" coding principles in C. Based on TigerBeetle's Tiger Style. Use when writing, reading, reviewing, or refactoring C code to ensure maximum safety, predictable execution, zero technical debt, and extreme performance.
---

# Safe C

Build highly predictable, robust, and performant C applications with a "zero technical debt" policy. This style guide is heavily inspired by TigerBeetle's Tiger Style.

## Retrieval-First Development

**Always verify standards against the reference documentation before implementing.**

| Resource | URL / Path |
|----------|------------|
| Safety & Control Flow | `./references/safety.md` |
| Performance Patterns | `./references/performance.md` |
| Developer Experience | `./references/dx.md` |

Review the relevant documentation when writing new logic or performing code reviews.

## When to Use

- Writing new C logic from scratch
- Refactoring existing C code to improve safety or performance
- Reviewing C PRs for code quality and standard adherence
- Optimizing memory allocations or hot paths
- Implementing strict error handling or boundary validation

## Core Principles

### Apply Safe C For

| Need | Example |
|------|---------|
| Predictable Execution | Bounded queues, bounded loops, explicit state machines |
| Memory Stability | Pre-allocating all memory at startup, static allocations, in-place initialization via out pointers |
| Operational Reliability | Strict assertion density (2+ per function), pair assertions, compound assertion splitting |
| Maintainability | Maximum 70 lines per function, max 100 columns per line, options structs |

### Do NOT Use

- Unbounded loops or recursion
- Dynamic memory allocations (`malloc()`, `calloc()`, `free()`) after startup phase
- Architecture-specific types like `long` or `unsigned int` where explicit sizes (`uint32_t`, `int64_t`) are required
- Third-party dependencies (unless explicitly approved)

## Quick Reference

### Bounded Control Flow & Strict Error Handling

```c
// Always bound loops and avoid recursion.
// Use explicit control flow and split compound conditions.
int process_items(const item_t* items, uint32_t items_count) {
    assert(items != NULL);
    assert(items_count <= MAX_ITEMS);

    for (uint32_t i = 0; i < items_count; i++) {
        if (items[i].is_active) {
            if (items[i].value > THRESHOLD) {
                // Handle specific positive case
            } else {
                // Explicitly handle the negative space
            }
        }
    }
    return 0;
}
```

### Allocation-Free Hot Path & Out Pointers

```c
// Construct large structs in-place by passing an out pointer
// Avoids copies and implicit stack allocations
void buffer_state_init(buffer_state_t* out_state, const config_options_t* options) {
    assert(out_state != NULL);
    assert(options != NULL);
    
    // In-place initialization
    *out_state = (buffer_state_t){
        .is_ready = true,
        .capacity = options->capacity_bytes,
        .cursor = 0,
    };
}
```

## Critical Rules

1. **No Recursion** - Keep control flow simple and execution bounds completely static.
2. **Fixed Upper Bounds** - All loops and queues must have a fixed upper bound to prevent infinite loops or tail latency spikes.
3. **No Dynamic Memory After Init** - All memory must be statically allocated at startup.
4. **Short Functions** - Hard limit of 70 lines per function. Push `if`s up, push `for`s down.
5. **High Assertion Density** - Assert function arguments, returns, invariants. Average minimum two assertions per function. Use `static_assert` for compile-time constants.
6. **Explicit Types** - Use explicitly-sized types (`uint32_t`, `int8_t`). Avoid `size_t` or `int` except for indexing small loops or interfacing with standard library.
7. **Handle All Errors** - Never ignore returns. Ensure all edge cases and negative spaces are checked.
8. **Options Structs** - Pass options to functions using an explicit options struct instead of relying on defaults or many boolean flags.
9. **Zero Dependencies** - Strictly avoid third-party libraries; stick to the standard library or built-in OS primitives where possible.
10. **Strict Naming** - `snake_case`, no abbreviations, add units/qualifiers at the end (`latency_ms_max`), sort by descending significance.

## Anti-Patterns (NEVER)

- Using `malloc()` during steady-state execution.
- Writing compound conditions like `if (a && b)`. Use nested ifs instead to handle each branch explicitly.
- Deeply nested logic in a single function (>70 lines).
- Silent error suppression.
- Leaving variables uninitialized or padding bytes unzeroed (buffer bleeds).

## Credits
- [Tiger Style](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md)