# Performance Patterns

This document covers techniques for predictable performance, memory management, and efficiency in C code. It emphasizes solving performance during the design phase rather than relying on post-hoc profiling.

## 1. Design Phase Performance
- Think about performance from the outset, from the beginning.
- Solve performance and get 1000x wins in the design phase, which is precisely when we can't measure or profile.
- Have mechanical sympathy. Like a carpenter, work with the grain of the hardware.

**Don't:**
```c
struct Node {
    int value;
    struct Node* next;
};
// Linked lists (pointer chasing) cause CPU cache misses.
```

**Do:**
```c
// Pre-allocated contiguous memory
static int global_values[MAX_NODES];

struct Nodes {
    int* values;
    uint32_t count;
};
// Slices into contiguous arrays allow for efficient CPU prefetching.
```

## 2. Sketches & The Four Resources
- Perform back-of-the-envelope sketches with respect to the four resources (network, disk, memory, CPU) and their two main characteristics (bandwidth, latency).
- Use sketches to be “roughly right” and land within 90% of the global maximum.
- Optimize for the slowest resources first (network, disk, memory, CPU) in that order, after compensating for the frequency of usage.

**Don't:**
```c
void read_file_byte_by_byte(FILE* f) {
    char c;
    // Millions of system calls (I/O latency) for 1 byte each.
    while (fread(&c, 1, 1, f) == 1) { /* ... */ }
}
```

**Do:**
```c
void read_file_buffered(FILE* f) {
    char buffer[4096];
    // Amortize I/O latency by reading in large batches.
    while (fread(buffer, 1, sizeof(buffer), f) > 0) { /* ... */ }
}
```

## 3. Batching & Pacing
- Distinguish between the control plane and data plane.
- Amortize network, disk, memory and CPU costs by batching accesses.
- Let the CPU be a sprinter doing the 100m. Be predictable. Don't force the CPU to zig zag and change lanes. Give the CPU large enough chunks of work.

**Don't:**
```c
void on_network_packet(packet_t* p) {
    // Immediate reaction context-switches and prevents batching
    process_packet(p);
}
```

**Do:**
```c
void on_network_packet(packet_t* p) {
    // Queue packet for batch processing at system's own pace
    enqueue_packet(p);
}

void main_loop(void) {
    while (true) {
        packet_batch_t batch = dequeue_batch();
        process_batch(&batch); // Process multiple items at once
    }
}
```

## 4. Be Explicit (No Compiler Magic)
- Minimize dependence on the compiler to do the right thing for you.
- Extract hot loops into stand-alone functions with primitive arguments without `self` or large struct pointers.
- That way, the compiler doesn't need to prove that it can cache struct's fields in registers, and a human reader can spot redundant computations easier.

**Don't:**
```c
void process_state(struct State* s) {
    for (uint32_t i = 0; i < s->count; i++) {
        // Compiler may struggle to cache s->multiplier due to potential aliasing
        s->data[i] *= s->multiplier;
    }
}
```

**Do:**
```c
// Extracted hot loop with primitives: easy for compiler to optimize
void multiply_data(uint32_t* data, uint32_t count, uint32_t multiplier) {
    for (uint32_t i = 0; i < count; i++) {
        data[i] *= multiplier;
    }
}

void process_state(struct State* s) {
    multiply_data(s->data, s->count, s->multiplier);
}
```

## 5. In-Place Initialization (Out Pointers)
- Construct larger structs in-place by passing an _out pointer_ during initialization.
- In-place initializations can assume **pointer stability** and **immovable types** while eliminating intermediate copy-move allocations.
- Keep in mind that in-place initializations are viral — if any field is initialized in-place, the entire container struct should be initialized in-place as well.

**Don't:**
```c
struct LargeStruct create_struct(void) {
    struct LargeStruct s;
    // ... initialize s ...
    return s; // Potential large copy on the stack
}
```

**Do:**
```c
void init_struct(struct LargeStruct* out) {
    assert(out != NULL);
    // Initialize exactly where it lives
    *out = (struct LargeStruct){ .field = 1 };
}
```

## 6. Buffer Bleeds
- Be on your guard for buffer bleeds (buffer underflow).
- Ensure that buffers are fully utilized or that all unused padding bytes are strictly zeroed out.
- This prevents leaking sensitive information and ensures deterministic state guarantees.

**Don't:**
```c
struct Packet {
    uint16_t id;
    // 2 bytes of implicit padding here
    uint32_t value;
};
// Padding here may contain garbage (secret leaks/non-determinism)
```

**Do:**
```c
struct Packet p;
memset(&p, 0, sizeof(p)); // Explicitly zero all bytes including padding
p.id = 1;
```