# Performance

This document covers techniques for predictable performance, memory management, and efficiency in Go code.

## 1. Design Phase & Mechanical Sympathy

Think about performance from the outset, from the beginning. The best time to solve performance—to get the huge 1000x wins—is in the design phase, which is precisely when you cannot measure or profile. Fixing a system after implementation and profiling is harder and yields smaller gains. Have mechanical sympathy: work with the grain of the hardware.

**Don't:**
```go
type Node struct {
    Value int
    Next  *Node
}
// Pointer chasing across dynamically allocated nodes is hostile to CPU caches.
```

**Do:**
```go
type NodeList struct {
    Values []int
}
// Contiguous memory layout (slices/arrays) works with the CPU prefetcher.
```

## 2. Back-of-the-Envelope Sketches

Perform back-of-the-envelope sketches with respect to the four resources (network, disk, memory, CPU) and their two main characteristics (bandwidth, latency).
* Sketches are cheap and help you land within 90% of the global maximum.
* Optimize for the slowest resources first (network, disk, memory, CPU) in that order, after compensating for the frequency of usage. A memory cache miss may be as expensive as a disk fsync if it happens many times more.

**Don't:**
```go
func readByteByByte(file *os.File) {
    b := make([]byte, 1)
    // Millions of system calls; latency dominates.
    for {
        if _, err := file.Read(b); err != nil { break }
    }
}
```

**Do:**
```go
func readInChunks(file *os.File) {
    buf := make([]byte, 4*1024*1024) // 4MB chunks
    // Amortize system call latency; maximize bandwidth.
    for {
        if _, err := file.Read(buf); err != nil { break }
    }
}
```

## 3. Static Memory Allocation

All memory should be statically allocated at startup. To the extent possible, no memory should be dynamically allocated (or freed and reallocated) after initialization. In Go, this means pre-allocating slices and avoiding `make` or `new` in the hot path. This avoids unpredictable behavior (like Garbage Collection pauses or allocation latency) that can significantly affect performance. It also encourages more efficient, simpler designs that consider all possible memory usage patterns upfront.

**Don't:**
```go
func processItems(items []Item) {
    // Allocating dynamically in the hot path triggers GC pressure.
    results := make([]Result, 0, len(items))
    // ...
}
```

**Do:**
```go
type Processor struct {
    results []Result // Pre-allocated once at startup
}

func (p *Processor) processItems(items []Item) {
    // Reuse the pre-allocated buffer without allocating.
    p.results = p.results[:0]
    // ...
}
```

## 4. In-Place Initialization

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

## 5. Event Handling & Pacing

Whenever your program has to interact with external entities, don't do things directly in reaction to external events. Instead, your program should run at its own pace. Not only does this make your program safer by keeping the control flow under your control, it also improves performance because you get to batch instead of context-switching on every event.

**Don't:**
```go
func onNetworkEvent(event Event) {
    // Reacting immediately causes context switching and unpredictable pacing.
    go process(event)
}
```

**Do:**
```go
func onNetworkEvent(event Event) {
    // Enqueue the event. Let the system run at its own controlled pace.
    eventQueue.Push(event)
}

func eventLoop() {
    for {
        // Process in fixed-size batches on a dedicated goroutine.
        batch := eventQueue.PopBatch()
        processBatch(batch)
    }
}
```

## 6. Batching & Control/Data Plane

Amortize network, disk, memory, and CPU costs by batching accesses.
* Distinguish between the control plane and data plane. A clear delineation between the two through the use of batching enables a high level of assertion safety without losing performance.
* Group operations where possible to reduce system call overhead and lock contention.

**Don't:**
```go
func updateRecords(records []Record) {
    for _, r := range records {
        // One network round-trip per record.
        db.Exec("UPDATE table SET val = ? WHERE id = ?", r.Value, r.ID)
    }
}
```

**Do:**
```go
func updateRecords(records []Record) {
    // Group operations to amortize network/disk latency.
    db.Exec("UPDATE table SET val = ...", batchValues(records))
}
```

## 7. CPU Predictability

Let the CPU be a sprinter doing the 100m. Be predictable. Don't force the CPU to zig-zag and change lanes. Give the CPU large enough chunks of work (which comes back to batching).

**Don't:**
```go
func processMixed(items []Item) {
    for _, item := range items {
        // Random branches cause CPU branch prediction failures and pipeline stalls.
        if item.Type == TypeA {
            processA(item)
        } else {
            processB(item)
        }
    }
}
```

**Do:**
```go
func processSorted(items []Item) {
    // If items are grouped/sorted by type first, the branch predictor succeeds.
    // Alternatively, split into two separate homogenous loops.
    for _, item := range items {
        if item.Type == TypeA {
            processA(item)
        } else {
            processB(item)
        }
    }
}
```

## 8. Be Explicit (Minimize Compiler Dependence)

Minimize dependence on the compiler to do the right thing for you.
* Extract hot loops into stand-alone functions with primitive arguments, rather than calling methods on structs (`self`/receiver).
* By passing primitives to a pure function, the compiler doesn't need to prove that it can cache struct fields in registers, and a human reader can spot redundant computations more easily.

**Don't:**
```go
func (s *State) ProcessHotLoop() {
    for i := 0; i < len(s.Data); i++ {
        s.Data[i] = s.Data[i] * s.Multiplier
    }
}
```

**Do:**
```go
// Extracted hot loop with primitive/slice arguments
func processHotLoop(data []int, multiplier int) {
    for i := 0; i < len(data); i++ {
        data[i] = data[i] * multiplier
    }
}

func (s *State) ProcessHotLoop() {
    processHotLoop(s.Data, s.Multiplier)
}
```

## 9. Cache Invalidation & Locality

* **Shrink the scope** to minimize the number of variables at play.
* Calculate or check variables close to where and when they are used. Don't introduce variables before they are needed.
* Use simpler function signatures and return types to reduce dimensionality at the call site and the number of branches that need to be handled.

**Don't:**
```go
func calculate() {
    var a, b, c int // Declared far from use
    
    // ... many lines of code ...
    
    a = 1
    b = 2
    c = a + b
}
```

**Do:**
```go
func calculate() {
    // ... many lines of code ...
    
    a := 1 // Declared exactly where needed, reducing scope.
    b := 2
    c := a + b
}
```
