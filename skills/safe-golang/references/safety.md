# Safety & Control Flow

This document details the core rules for ensuring safety, predictable execution bounds, and robust error handling in Go applications.

## 1. Simple Control Flow

Use very simple, explicit control flow. Do not use `goto` statements, and **do not use recursion**. This ensures that all executions are statically bounded.

**Don't:**
```go
func processItems(items []Item, index int) {
    if index >= len(items) {
        return
    }
    // Process item
    processItems(items, index+1) // Recursion makes execution bounds unpredictable
}
```

**Do:**
```go
func processItems(items []Item) {
    for i := 0; i < len(items); i++ {
        // Process item sequentially
    }
}
```

## 2. Fixed Upper Bounds

All loops and channels must have a fixed upper bound. Infinite loops must be strictly controlled (e.g., event loops with context cancellation).

**Don't:**
```go
// Unbounded channel can lead to OOM
ch := make(chan Task) 

// Unbounded loop waiting for external state without context
for {
    if checkStatus() == "done" {
        break
    }
}
```

**Do:**
```go
// Bounded channel
const maxTasks = 1000
ch := make(chan Task, maxTasks)

// Bounded loop with context timeout
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        if checkStatus() == "done" {
            return nil
        }
        time.Sleep(10 * time.Millisecond)
    }
}
```

## 3. No Dynamic Memory Allocation After Initialization

All significant memory must be allocated at startup. Avoid `make()` or `new()` inside hot paths or event loops. This avoids unpredictable behavior, latency spikes from garbage collection, and makes the system easier to reason about. Use `sync.Pool` or pre-allocated arenas if dynamic reuse is absolutely necessary.

**Don't:**
```go
func handleRequest(data []byte) {
    // Allocating in a hot path
    buffer := make([]byte, 1024*1024) 
    copy(buffer, data)
    // ...
}
```

**Do:**
```go
type Server struct {
    bufferPool sync.Pool
}

func NewServer() *Server {
    return &Server{
        bufferPool: sync.Pool{
            New: func() interface{} {
                b := make([]byte, 1024*1024)
                return &b
            },
        },
    }
}

func (s *Server) handleRequest(data []byte) {
    bufferPtr := s.bufferPool.Get().(*[]byte)
    defer s.bufferPool.Put(bufferPtr)
    
    buffer := *bufferPtr
    copy(buffer, data)
    // ...
}
```

## 4. Split Compound Conditions

Compound conditions that evaluate multiple booleans make it difficult to verify that all cases are handled. Split compound conditions into simple conditions using nested `if/else` branches. Check positive and negative spaces thoroughly.

**Don't:**
```go
if isValid && (count > 0 || force) {
    // Complex condition hides behavior and cases
}
```

**Do:**
```go
if !isValid {
    return
}
if count > 0 {
    // ...
} else if force {
    // ...
}
```

## 5. Assertions and Invariants

Go does not have a built-in `assert` keyword, but we enforce the principle: **panic only for programmer errors/broken invariants, and return explicitly wrapped errors for all operational errors.**

Use "pair assertions": assert validity right before writing and immediately after reading. Assert both the *positive space* that you do expect AND the *negative space* that you do not expect.

**Don't:**
```go
func calculateRatio(a, b int) int {
    // If b is 0, this will panic unexpectedly due to division by zero
    return a / b
}
```

**Do:**
```go
func calculateRatio(a, b int) int {
    if b == 0 {
        // Explicitly panic for a programmer error (invariant violation)
        panic("calculateRatio: b cannot be zero")
    }
    return a / b
}
```

## 6. Minimal Variable Scope

Declare variables at the smallest possible scope.

**Don't:**
```go
func processUser(id int) error {
    var user User
    var err error
    
    // ... many lines of code ...
    
    user, err = db.GetUser(id)
    if err != nil {
        return err
    }
    return nil
}
```

**Do:**
```go
func processUser(id int) error {
    // ... many lines of code ...
    
    // Scoped strictly to where it is needed
    user, err := db.GetUser(id)
    if err != nil {
        return err
    }
    _ = user // Use user
    return nil
}
```

## 7. Check All Return Values

Every return value must be checked. Never ignore errors with `_`.

**Don't:**
```go
func saveFile(data []byte) {
    _ = os.WriteFile("data.txt", data, 0644) // Ignored error
}
```

**Do:**
```go
func saveFile(data []byte) error {
    err := os.WriteFile("data.txt", data, 0644)
    if err != nil {
        return fmt.Errorf("failed to save data.txt: %w", err)
    }
    return nil
}
```

## 8. Avoid "Magic" (Macros/Reflection/Init)

The use of `init()` functions, global mutable state, and `reflect` should be strictly minimized or avoided entirely.

**Don't:**
```go
var dbConnection *sql.DB

func init() {
    // Magic initialization hidden from the caller
    dbConnection = connectDB() 
}
```

**Do:**
```go
// Explicit initialization controlled by main()
func ConnectDB(ctx context.Context, config DBConfig) (*sql.DB, error) {
    // ...
}
```

## 9. Restrict Indirection

Limit pointers to one level of dereference. Do not use double pointers (`**`) or pointers to interfaces (`*error`, `*io.Reader`). Prefer value types where possible.

**Don't:**
```go
func updateNode(node **Node) {
    // Double indirection is hard to follow and bug-prone
    **node = Node{Value: 1}
}
```

**Do:**
```go
func updateNode(node *Node) {
    node.Value = 1
}
```

## 10. Strict Static Analysis

All code must compile without warnings and pass strict static analysis. We use `golangci-lint` with rigorous settings. Treat all linter warnings as errors.
