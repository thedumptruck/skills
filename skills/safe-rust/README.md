# safe-rust

This skill instructs the agent to enforce rigorous memory safety, zero-cost abstractions, and idiomatic patterns for Rust programming, prioritizing explicit error handling, compile-time safety (typestates), and minimal allocations.

## Use Cases & Triggers
- **Writing New Code:** "Write a new data processing pipeline in Rust using safe-rust."
- **Refactoring:** "Refactor this Rust code to follow safe-rust principles (remove unwraps, avoid unnecessary clones, implement typestates)."
- **Reviewing:** "Review this PR for safe-rust violations, checking for proper borrow checker compliance and zero-cost abstractions."
- **Memory Allocation:** "Optimize this hot path to avoid String allocations and use `Cow` per safe-rust."

The specific guidelines, including instructions and reference links for detailed use-case examples, are defined in `SKILL.md`.
