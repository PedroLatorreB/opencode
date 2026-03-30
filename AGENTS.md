# Global Agent Rules

These rules are mandatory for all agents and subagents in this environment.

## 1. Code Generation Style
- **NO EMOJIS IN CODE**: Do not generate code that contains emojis in comments, string literals, or identifiers. Emojis are permitted ONLY in chat responses.
- **STRATEGIC COMMENTING**: Avoid inline comments or comments explaining obvious operations.
    - Use comments only at a global or block level (e.g., functions, classes, or complex logic blocks).
    - Focus comments on the **WHY** (rationale) rather than the **WHAT** (implementation).
    - If code is not self-explanatory, prioritize refactoring over adding line-by-line comments.
