# Safety & Control Flow

This document details the core rules for ensuring safety, predictable execution bounds, and robust error handling in C applications. It focuses on eliminating unpredictable behavior like recursion and dynamic memory allocation to guarantee system liveness and correctness.

## 1. Explicit Control Flow
- Use **only very simple, explicit control flow** for clarity.
- **Do not use recursion**. Ensure all bounded executions are truly bounded.
- Limit abstractions. Abstractions are never zero cost and introduce the risk of a leaky abstraction.
- **Push `if`s up and `for`s down**. Centralize control flow. Parent functions should keep all switch/if statements. Move non-branchy logic fragments to helper functions.

**Don't:**
```c
void process_items_recursive(item_t* items, uint32_t count, uint32_t index) {
    if (index >= count) return;
    // Recursion makes execution bounds and stack usage unpredictable
    process_item(&items[index]);
    process_items_recursive(items, count, index + 1);
}
```

**Do:**
```c
void process_items(item_t* items, uint32_t count) {
    for (uint32_t i = 0; i < count; i++) {
        // Sequential iteration with fixed bounds
        process_item(&items[i]);
    }
}
```

## 2. Fixed Upper Bounds
- **Put a limit on everything**. Everything has a limit.
- All loops and all queues must have a fixed upper bound to prevent infinite loops or tail latency spikes.
- Follow the fail-fast principle. Where a loop cannot terminate (e.g. an event loop), this must be explicitly asserted.

**Don't:**
```c
// Unbounded loop waiting for external state can hang forever
while (!is_ready()) {
    // No timeout or bound
}
```

**Do:**
```c
// Bounded loop with explicit limit
uint64_t attempts = 0;
while (!is_ready()) {
    attempts++;
    assert(attempts < MAX_ATTEMPTS); // Fail fast
    sleep_ms(10);
}
```

## 3. Strict Assertions
Assertions downgrade catastrophic correctness bugs into liveness bugs. Assertions are a force multiplier for discovering bugs by fuzzing.
- **Density**: The assertion density of the code must average a minimum of two assertions per function.
- **Assert everything**: Arguments, return values, pre/postconditions, and invariants. A function must not operate blindly on unchecked data.
- **Pair assertions**: For every property, find at least two different code paths where an assertion can be added (e.g., right before writing, and right after reading).
- **Split compound assertions**: Prefer `assert(a); assert(b);` over `assert(a && b);`. This provides precise failure locations.

**Don't:**
```c
void update_value(uint32_t index, int32_t value) {
    // Operating blindly on data
    global_array[index] = value;
}
```

**Do:**
```c
void update_value(uint32_t index, int32_t value) {
    assert(index < ARRAY_SIZE); // Pre-condition
    assert(value > MIN_VALUE);   // Argument validation
    global_array[index] = value;
    assert(global_array[index] == value); // Post-condition
}
```

## 4. Static Memory Allocation
- All memory must be statically allocated at startup.
- **No memory may be dynamically allocated (or freed and reallocated) after initialization.**
- This avoids unpredictable behavior, tail latency spikes, and use-after-free bugs. It forces simpler, more performant designs.

**Don't:**
```c
void handle_request(const uint8_t* data, size_t size) {
    // Dynamic allocation in hot path causes fragmentation and latency spikes
    uint8_t* buffer = malloc(size);
    memcpy(buffer, data, size);
    // ...
    free(buffer);
}
```

**Do:**
```c
// Pre-allocated static buffer or pool at startup
static uint8_t request_buffer[MAX_REQUEST_SIZE];

void handle_request(const uint8_t* data, size_t size) {
    assert(size <= MAX_REQUEST_SIZE);
    memcpy(request_buffer, data, size);
    // ...
}
```

## 5. Scope and Variable declarations
- Declare variables at the **smallest possible scope**.
- Minimize the number of variables in scope to reduce misuse probability.

**Don't:**
```c
void process(void) {
    uint32_t i; // Declared at top level, scope too large
    // ... 50 lines of code ...
    for (i = 0; i < 10; i++) { /* ... */ }
}
```

**Do:**
```c
void process(void) {
    // ... 50 lines of code ...
    for (uint32_t i = 0; i < 10; i++) { // Scoped to the loop
        /* ... */
    }
}
```

## 6. Compound Conditions & Negations
- Prefer early returns (guard clauses) over nested `if`/`else` blocks.
- Simplify complex compound conditions by breaking them into discrete checks.
- State invariants positively.
- **Exhaustive branching**: All `if-else if` chains must terminate with an `else` block to handle unexpected cases. Standalone `if` statements do not require an `else` block if the negative case is a "no-op".

**Don't:**
```c
if (is_valid) {
    if (count > 0) {
        // Nested logic is hard to follow
    } else {
        if (is_forced) {
            // ...
        }
    }
}

// Redundant empty else for simple if
if (items_count > 0) {
    do_work();
} else {
    // Empty else adds clutter
}
```

**Do:**
```c
if (!is_valid) return;

if (count > 0) {
    // Case 1
    return;
}

if (is_forced) {
    // Case 2
    return;
}

// Case 3: Default
```

## 7. Exhaustive Error Handling
- All errors must be handled. Almost all catastrophic system failures result from incorrect handling of non-fatal errors signaled in software.
- Avoid external event reactions. Don't do things directly in reaction to external events. Your program should run at its own pace to stay in control and batch work.

**Don't:**
```c
void log_data(const char* msg) {
    // Ignoring return value: what if disk is full?
    fprintf(logfile, "%s\n", msg);
}
```

**Do:**
```c
int log_data(const char* msg) {
    if (fprintf(logfile, "%s\n", msg) < 0) {
        return ERROR_DISK_FULL; // Propagate error
    }
    return 0;
}
```
