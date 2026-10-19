# Domain Documentation Layout

This project uses a **single-context** domain model.

## Structure

### Glossary
- **Location**: `GLOSSARY.md` at the repository root
- **Purpose**: Canonical definitions of domain terms, entities, and concepts
- **Format**: Markdown with term headings and definitions

### Architecture Decision Records (ADRs)
- **Location**: `docs/adr/`
- **Purpose**: Record significant architectural and design decisions
- **Format**: Individual markdown files, typically numbered (e.g., `0001-use-ethers-v6.md`)

## Consumer Rules

### When to read the glossary
- Before proposing new terminology
- When encountering unfamiliar domain terms in existing code
- Before naming new classes, modules, or significant functions
- When domain boundaries or responsibilities are unclear

### When to read ADRs
- Before making architectural changes
- When understanding why certain patterns or libraries were chosen
- Before proposing alternatives to existing technical decisions

### When to create ADRs
- When making decisions that affect:
  - System structure or component boundaries
  - Technology choices (libraries, frameworks, tools)
  - Cross-cutting patterns (error handling, logging, state management)
  - API contracts or integration approaches
- When reversing or superseding a previous ADR

## Single-Context Semantics

In a single-context layout:
- All code shares one unified glossary
- One set of ADRs covers the entire repository
- Terms have consistent meanings across all modules
- No namespace prefixes or context qualifiers needed
