# Performance & Memory Strategies

Performance optimization begins in the design phase. By systematically controlling memory allocation and understanding execution paths, we can build predictably fast applications with minimal latency spikes.

### 1. Statically Allocated Memory (Mitigating GC)
*   **No Dynamic Allocations After Init**: Pre-allocate all significant memory at startup. Do not rely on dynamic object allocation (`new Object()`, `[]`, `{}`) in hot paths, as this triggers unpredictable Garbage Collection pauses.
*   **Object Pooling**: For objects that must be created repeatedly during execution, implement custom Object Pools (`Array<T>` pre-filled) to reuse memory.
*   **In-Place Mutation**: Pass an `out` target object to functions instead of returning newly instantiated objects.
*   **TypedArrays**: Where appropriate, use flat memory representations like `Uint8Array`, `Float64Array`, or `SharedArrayBuffer` for deterministic memory layouts and blazing-fast accesses.

```typescript
// ❌ DON'T: Allocating memory in hot paths
function processStream(chunk: number[]) {
    const buffer = new Uint8Array(1024); // Allocates on every call, triggers GC
    // ...
    return buffer;
}

// ✅ DO: In-place mutation with pre-allocated arrays
function processStream(chunk: number[], outBuffer: Uint8Array): void {
    outBuffer.fill(0); // Reset pre-allocated memory
    // ... mutate outBuffer directly
}
```

### 2. Predictable Execution (Mechanical Sympathy)
*   **Batching over Events**: Do not react blindly to individual external events. Batch network requests, disk reads, and CPU work. Run loops at their own pace over queues.
*   **Extract Hot Loops**: Move hot loops into standalone functions taking primitive arguments. This helps the JIT compiler optimize and prevents closure overhead.
*   **Avoid Closures in Hot Paths**: Closures allocate memory for their environment. Use explicit function parameters or class methods instead.
*   **Back-of-the-Envelope Sketches**: Estimate bounds for memory usage, CPU operations, and network payloads before writing the code.

```typescript
// ❌ DON'T: Closures inside hot loops
function processItems(items: Item[], multiplier: number) {
    return items.map(item => {
        // New function allocated per item, captures `multiplier` environment
        return item.value * multiplier; 
    });
}

// ✅ DO: Extracted pure functions
function multiply(value: number, multiplier: number): number {
    return value * multiplier;
}

function processItems(items: Item[], multiplier: number): void {
    const len = items.length;
    for (let i = 0; i < len; i++) {
        const item = items[i];
        if (item === undefined) break; // noUncheckedIndexedAccess guard
        item.value = multiply(item.value, multiplier);
    }
}
```

### 3. V8 Engine Optimizations
*   **Monomorphic Shapes**: Initialize all properties of an object in the constructor (or factory) in the same order. Do not dynamically add or remove properties later (`delete obj.prop`), as this forces V8 to de-optimize the object shape (Hidden Classes).
*   **Avoid Megamorphic Arrays**: Do not mix types within arrays (e.g., `[1, "string", {}]`). Keep arrays strictly typed (`number[]`).

```typescript
// ❌ DON'T: Dynamic property additions and deletions
function createUser(name: string, age?: number) {
    const user: any = { name };
    if (age) user.age = age; // Changes object shape dynamically
    delete user.name; // Destroys V8 hidden class optimizations
    return user;
}

// ✅ DO: Monomorphic initialization
class User {
    name: string | null;
    age: number | null;
    
    constructor(name: string | null, age: number | null) {
        this.name = name;
        this.age = age; // Shape is fixed strictly on creation
    }
}
```