# Developer Experience

Guidelines for naming, API design, formatting, and dependencies.

## 1. Naming Conventions

Combine Go's standard `camelCase`/`PascalCase` with strict noun/verb rules.
Add units or qualifiers to variable names, putting the units last (descending significance).

**Don't:**
```go
var maxLatency int
var timeout time.Duration
var isProcessing bool
```

**Do:**
```go
var latencyMaxMs int
var timeoutMs time.Duration
var processingActive bool
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

## 3. Formatting and Line Limits

* Hard limit all line lengths to at most **100 columns**. 
* Always use `gofmt` or `goimports` for standard Go formatting.

## 4. Zero Dependencies

We maintain a strict "zero dependencies" policy outside of the Go standard library, barring absolute necessities. Every dependency introduces supply chain risk and increases build times.
