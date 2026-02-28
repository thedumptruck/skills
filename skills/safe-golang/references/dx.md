# Developer Experience

Guidelines for naming, API design, formatting, and dependencies.

## 1. Naming Conventions

Combine Go's standard `camelCase`/`PascalCase` with strict noun/verb rules.
* **Add units or qualifiers** to variable names, putting the units last (descending significance).
* **Do not abbreviate** variable names unless they are standard primitive integer types used in simple loops/math.
* **Match lengths** of related names (e.g., use `source` and `target` instead of `src` and `dest`) so that related variables line up symmetrically in the source code.

**Don't:**
```go
var maxLatency int
var timeout time.Duration
var isProcessing bool
var src []byte
var dest []byte
```

**Do:**
```go
var latencyMaxMs int
var timeoutMs time.Duration
var processingActive bool
var source []byte
var target []byte
```

## 2. Options Structs

Use explicit options structs instead of relying on default parameters or passing multiple arguments of the same type.

**Don't:**
```go
// Easy to mix up the order of the two booleans
func OpenFile(path string, readOnly bool, createIfMissing bool) { ... }
```

**Do:**
```go
type OpenOptions struct {
    ReadOnly        bool
    CreateIfMissing bool
}

func OpenFile(path string, opts OpenOptions) { ... }
```

## 3. Short Functions

Functions must fit on a single screen to be easily understood without scrolling. We enforce a **hard limit of 70 lines per function**.

Push `if`s up and `for`s down. Keep all switch/if statements in the "parent" function, and move non-branchy logic fragments to helper functions.

**Don't:**
```go
func processComplexLogic(data Data) {
    if data.isValid() {
        // ... 30 lines of logic ...
        for _, item := range data.Items {
            // ... 30 lines of inner logic ...
        }
    }
}
```

**Do:**
```go
func processComplexLogic(data Data) {
    if !data.isValid() {
        return
    }
    processItems(data.Items)
}

func processItems(items []Item) {
    for _, item := range items {
        processItem(item)
    }
}

func processItem(item Item) {
    // ... focused logic ...
}
```

## 4. Formatting and Line Limits

* Hard limit all line lengths to at most **100 columns**. 
* Always use `gofmt` or `goimports` for standard Go formatting.

## 5. Comments and Documentation

Code alone is not documentation.
* **Explain the "Why":** Use comments to explain why you wrote the code the way you did, not just what it does.
* **Proper Punctuation:** Comments are sentences. They should start with a capital letter, have a space after the `//`, and end with a period (or a colon if preceding a block).

**Don't:**
```go
// increment counter
c++
```

**Do:**
```go
// Increment the counter to track the number of active connections.
// This is used downstream to enforce the connection limit.
c++
```

## 6. Zero Dependencies

We maintain a strict "zero dependencies" policy outside of the Go standard library, barring absolute necessities. Every dependency introduces supply chain risk and increases build times.
