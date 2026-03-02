# Control Flow & Safety Constraints

To build highly predictable TypeScript applications with zero technical debt, control flow must be rigidly bounded and explicit. This ensures execution is predictable, errors are handled systematically, and edge cases are eliminated by design.

### 1. Simple, Bounded Control Flow
*   **No Recursion**: Do not use direct or indirect recursion. Use iterative bounds instead. Recursion hides unbounded execution and stack overflows.
*   **Bounded Loops**: All loops must have a fixed upper bound. Use `for` loops with a strict limit rather than unbounded `while(true)` loops. If an event loop cannot terminate, it must be asserted.
*   **Bounded Async**: All `Promise`s and asynchronous operations must be bounded by a timeout. Use `AbortController` and `AbortSignal.timeout()` to guarantee predictable execution bounds. Unbounded promises are memory leaks.

```typescript
// ❌ DON'T: Unbounded async operations
async function fetchUser(id: string) {
    return await fetch(`/api/users/${id}`);
}

// ✅ DO: Strictly bounded async operations
async function fetchUser(id: string, timeoutMs: number) {
    return await fetch(`/api/users/${id}`, {
        signal: AbortSignal.timeout(timeoutMs)
    });
}
```

### 2. Assertions & Validation
*   **Heavy Assertion Density**: Assert preconditions, postconditions, and invariants. A function should average two assertions. Use assertions to document the "positive space" (what you expect) and the "negative space" (what should never happen).
*   **Pair Assertions**: Assert data validity at multiple boundaries (e.g., right before sending to the network, and immediately after receiving it).
*   **Explicit Panics**: Assertions are for programmer errors (bugs). If an assertion fails, crash the program (`process.exit(1)` or throw an uncatchable `Error`). Do not catch assertion failures.
*   **No Implicit Data Trusts**: A function must not blindly operate on data it has not validated. Validate schema boundaries aggressively (using explicit types or libraries like Zod).

```typescript
// ❌ DON'T: Blindly trusting data or failing silently
function processPayload(payload: any) {
    if (!payload.id) return; // Silently failing on invalid data
    db.save(payload.id);
}

// ✅ DO: Validate boundaries and assert invariants
function processPayload(payload: unknown): Result<number, Error> {
    const parsed = schema.safeParse(payload);
    if (!parsed.success) return { ok: false, error: parsed.error };

    const id = parsed.data.id;
    assert(id > 0, "Invariant violated: parsed ID must be strictly positive");

    const saveResult = db.save(id);
    if (!saveResult.ok) return { ok: false, error: saveResult.error };

    return { ok: true, value: id };
}
```

### 3. Error Handling
*   **No `throw` for Control Flow**: Expected operational errors must be returned as values, not thrown. Exceptions break control flow predictability and bypass type signatures.
*   **Result Types**: Use discriminated unions or explicit `Result<Value, Error>` types for functions that can fail operationally.
*   **Check All Returns**: Never ignore the result of a function. The TypeScript compiler should enforce exhaustive checks on discriminated unions.

```typescript
// ❌ DON'T: Throwing operational errors
function parseJSON(str: string) {
    try {
        return JSON.parse(str);
    } catch (err) {
        throw new Error("Invalid JSON");
    }
}

// ✅ DO: Returning explicit Result types
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function parseJSON(str: string): Result<unknown, SyntaxError> {
    try {
        return { ok: true, value: JSON.parse(str) };
    } catch (err) {
        return { ok: false, error: err instanceof SyntaxError ? err : new SyntaxError(String(err)) };
    }
}
```

### 4. Code Structure
*   **Short Functions**: Enforce a hard limit of 70 lines per function. "Push `if`s up, push `for`s down."
*   **Minimize Scope**: Declare variables at the smallest possible scope. Do not reuse variables for different purposes. Calculate or check variables close to where they are used to prevent TOCTOU bugs.
*   **Avoid Compound Conditions**: Split complex `if (a && b)` statements into nested `if` statements so both positive and negative spaces are handled clearly.