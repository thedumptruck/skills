# Performance (TigerStyle)

This document covers techniques for predictable performance, memory management, and efficiency in Go code.

## 1. In-Place Initialization

To minimize intermediate copy-move allocations and stack growth, construct larger structs in-place by passing a pointer during initialization.

**Don't:**
```go
func NewLargeBuffer() [1024 * 1024]byte {
    // Returns 1MB on the stack, causing a large copy
    var buf [1024 * 1024]byte
    return buf
}
```

**Do:**
```go
func InitLargeBuffer(target *[1024 * 1024]byte) {
    // Target is initialized in place
    for i := range target {
        target[i] = 0
    }
}
```

## 2. Batching

Amortize network, disk, memory, and CPU costs by batching accesses. Distinguish between the control plane and data plane. Group operations where possible to reduce system call overhead and lock contention.
