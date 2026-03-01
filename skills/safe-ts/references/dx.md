# Developer Experience & Tooling

Good developer experience stems from explicit, predictable, and maintainable code. By establishing strict naming conventions, maximizing compiler safety, and minimizing external dependencies, we reduce cognitive load and prevent entire classes of bugs.

### 1. Naming & Syntax
*   **Precise Naming**: Use accurate nouns and verbs. Append units and qualifiers at the end, sorted by significance (e.g., `latencyMsMax`, `timeoutMs`).
*   **Options Structs**: Functions taking more than 2 arguments, especially booleans or same-type primitives, must use an options object interface instead of positional arguments.
*   **No Abbreviations**: Use explicit, descriptive names, except for primitive loop indices (`i`, `j`).
*   **100 Column Limit**: Hard limit lines to 100 columns. If a line exceeds this, break it using trailing commas and the code formatter.

```typescript
// ❌ DON'T: Positional primitives without units
function connect(host: string, port: number, secure: boolean, timeout: number) {
    // ...
}
connect("localhost", 8080, true, 5000); // What does `true` or `5000` mean?

// ✅ DO: Options interfaces and unit suffixes
interface ConnectOptions {
    host: string;
    port: number;
    secure: boolean;
    timeoutMs: number;
}
function connect(options: ConnectOptions) {
    // ...
}
connect({ host: "localhost", port: 8080, secure: true, timeoutMs: 5000 });
```

### 2. Strict Compiler Tooling
*   **TypeScript Configuration**:
    *   `strict: true` (Enables `strictNullChecks`, `noImplicitAny`, etc.)
    *   `noUncheckedIndexedAccess: true` (Forces checking array indices)
    *   `exactOptionalPropertyTypes: true`
    *   `noImplicitReturns: true`
*   **Warnings as Errors**: Treat all TypeScript compiler warnings and ESLint warnings as errors. They must be resolved, not ignored.

```typescript
// ❌ DON'T: Assuming array indices exist
// tsconfig: noUncheckedIndexedAccess = false
function getFirstOrThrow(items: string[]) {
    const first = items[0]; 
    return first.toUpperCase(); // Runtime error if items is empty
}

// ✅ DO: Checking array bounds
// tsconfig: noUncheckedIndexedAccess = true
function getFirstOrThrow(items: string[]) {
    const first = items[0]; // Type is string | undefined
    if (first === undefined) throw new Error("List is empty");
    return first.toUpperCase(); 
}
```

### 3. Dependencies
*   **Zero Dependency Mindset**: Avoid third-party npm packages unless absolutely necessary. Every dependency introduces supply chain risk, increases bundle size, and adds technical debt. Rely on the Node.js/Deno standard library whenever possible.
*   **No Magic**: Strictly avoid `Proxy`, `Reflect`, and heavy decorator abstractions. They obscure execution paths and make the code difficult to trace.
*   **Avoid `any` and `as`**: Using `any` or `as Type` (type assertions) circumvents the compiler. Force the compiler to prove the type through narrowing (`typeof`, `instanceof`, or custom type guards).

```typescript
// ❌ DON'T: Bypassing the compiler
function parseUser(json: string): User {
    const data = JSON.parse(json);
    return data as User; // Compiler trusts you, but the data could be completely wrong
}

// ✅ DO: Type narrowing and guards
function isUser(data: unknown): data is User {
    return typeof data === 'object' && data !== null && 'id' in data;
}

function parseUser(json: string): Result<User, Error> {
    const data = JSON.parse(json);
    if (!isUser(data)) return { ok: false, error: new Error("Invalid schema") };
    return { ok: true, value: data };
}
```