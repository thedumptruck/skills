# AI Agent Skills

A collection of specialized skills and instructions for AI agents to enhance their capabilities, enforce coding standards, and perform complex tasks effectively.

## Available Skills

| Skill | Description |
|---|---|
| [**safe-golang**](./skills/safe-golang) | Enforces rigorous safety, performance, and developer experience standards for Go programming, inspired by "The Power of 10" and "TigerStyle". Use to build highly predictable, robust, and performant Go applications with a "zero technical debt" policy. |

## Structure

Each skill is located in the `skills/` directory and typically contains:

*   **`README.md`**: Overview of the skill, including use cases and example prompts.
*   **`SKILL.md`**: The core instructions, constraints, and operational guidelines loaded by the agent.
*   **`references/`**: (Optional) Supplemental documentation, rules, or code patterns referenced by the skill.

## How to Use

When interacting with an AI agent equipped with these skills, you can explicitly ask the agent to apply a skill to your current context. For example:

*   *"Write a new HTTP handler for user login using safe-golang."*
*   *"Review this PR for safe-golang violations."*
*   *"Refactor this Go file to follow safe-golang principles."*

## Adding a New Skill

To add a new skill to this repository:
1.  Create a new directory under `skills/` (e.g., `skills/my-new-skill`).
2.  Add a `README.md` describing its purpose and use cases.
3.  Add a `SKILL.md` containing the detailed system instructions for the agent.
4.  Include any necessary supplementary files or reference material in a `references/` subdirectory.
