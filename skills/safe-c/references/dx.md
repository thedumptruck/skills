# Developer Experience (DX) & Style

This document outlines conventions for naming, formatting, and code structure in C. These rules aim to reduce dimensionality, prevent common logic errors, and ensure the codebase remains maintainable as the team grows.

## 1. Function Boundaries
- **70 Lines Max**: Hard limit of 70 lines per function.
- **Good shape**: Inverse hourglass: few parameters, simple return type, meaty logic in the middle.
- Keep leaf functions pure. Parent functions manage state and control flow; helpers compute what needs to change.

**Don't:**
```c
void complex_task(struct State* s) {
    // 200 lines of mixed logic, branching, and state changes
    if (s->ready) { /* 50 lines */ }
    for (int i=0; i<100; i++) { /* 80 lines */ }
}
```

**Do:**
```c
void process_item(struct Item* item) { /* Leaf: pure logic */ }

void complex_task(struct State* s) {
    // Parent: simple control flow and helper calls
    if (!s->ready) return;
    for (int i = 0; i < s->count; i++) {
        process_item(&s->items[i]);
    }
}
```

## 2. Naming Things
- **Get nouns and verbs just right**: Build crisp mental models.
- **Format**: `snake_case` for function, variable, and file names.
- **No abbreviations**: Unless it's a primitive integer type used in a sort/matrix calculation (like `i`, `j`).
- **Acronyms**: Proper capitalization.
- **Units last**: Append units/qualifiers at the end, sorted by descending significance (e.g. `latency_ms_max` not `max_latency_ms`). Lines up related variables nicely.
- **Meaningful names**: E.g., `arena_allocator` tells the reader if explicit frees are needed, unlike a generic `alloc`.
- **Symmetry**: Try to match lengths for related names (e.g., `source` and `target` instead of `src` and `dest`) so they align vertically.
- **Helper Prefixes**: Prefix helper/callback names with the calling function's name (e.g., `read_sector()` and `read_sector_callback()`).
- **Callbacks Last**: Put callbacks at the end of parameter lists (mirroring control flow).
- **Options Structs**: If a function takes multiple generic arguments (like two `uint64_t` or multiple booleans), use an explicit named options struct.

**Don't:**
```c
uint32_t max_lat_ms = 100;
void do_it(int a, bool b, bool c); // Generic names and flags
```

**Do:**
```c
uint32_t latency_ms_max = 100; // Qualifiers sorted by significance
void sector_read(const struct ReadOptions* options); // Passed by const pointer
```

## 3. Code Layout & Formatting
- **Line Length**: Hard limit of 100 columns. Wrap structures, arrays, and signatures elegantly.
- **Indentation**: 4 spaces, not 2.
- **Braces**: Always use braces for `if` statements, even one-liners (unless the entire if-statement fits on a single line safely), as defense against `goto fail;` bugs.
- **Ordering**: Order matters for readability. Main functions first. Top-down reading. Order structs: fields, then types, then methods.
- **Visual Grouping**: Use newlines to group resource allocation and its corresponding deallocation/cleanup or error checking.

**Don't:**
```c
if (is_ready) update(); // Missing braces

struct Data {
    void (*process)(struct Data*); // Methods mixed with fields
    int value;
};
```

**Do:**
```c
if (is_ready) {
    update(); // Explicit braces for safety
}

struct Data {
    int value; // Fields first
};
```

## 4. Cache Invalidation & Variable State
- **Don't duplicate variables** or take aliases to them.
- **Pass by value vs const pointer**: If an argument is larger than 16 bytes, pass it as `const type_t*` to avoid accidental stack copies.
- **Shrink the scope**: Calculate or check variables close to where/when they are used to avoid POCPOU (place-of-check to place-of-use) bugs. Don't introduce variables before they are needed.
- **Simplify signatures**: Reduce dimensionality. e.g., returning `void` > `bool` > `uint64_t` > compound struct.
- **Don't overload names**: Ensure context-dependent meanings are separate.

**Don't:**
```c
void process_large(struct BigStruct s); // Large copy on stack
```

**Do:**
```c
void process_large(const struct BigStruct* s); // Efficient const pointer
```

## 5. Off-By-One Errors
- Distinguish between `index`, `count`, and `size`. They are distinct types functionally.
- `index` -> `count`: add one (0-based vs 1-based).
- `count` -> `size`: multiply by unit size.
- Include units/qualifiers in names to clarify intent.
- Show intent with division (floor, ceil, exact).

**Don't:**
```c
uint32_t len = 10;
// Ambiguous name: is 'len' the count of items or the size in bytes?
update_buffer(data, len); 
```

**Do:**
```c
uint32_t items_count = 10;
uint32_t buffer_size_bytes = items_count * sizeof(struct Item);
// Static allocation with clear units
static uint8_t buffer[MAX_ITEMS * sizeof(struct Item)];
```

## 6. General DX Guidelines
- **Zero Dependencies**: Avoid third-party dependencies outside the standard library to minimize supply chain risks and compile times.
- **Strict Compiler Warnings**: Use the strictest compiler settings possible (`-Wall -Wextra -Werror -pedantic`, etc.).
- **Always Say Why**: Code is not documentation. Explain the rationale for decisions. Provide high-level goals at the top of tests.
- **Descriptive Commits**: PR descriptions are not preserved in `git blame`. Make commit messages inform and delight.
- **Grammar in Comments**: Comments are sentences. Start with a capital letter, end with a period. Use a space after `//`.

**Don't:**
```c
// loop over data
for (int i=0; i<10; i++) { /* ... */ }
```

**Do:**
```c
// Ensure that we only process up to the capacity to avoid overflow.
for (int i = 0; i < capacity; i++) { /* ... */ }
```