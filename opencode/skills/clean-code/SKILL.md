---
name: clean-code
description: Clean code guidelines covering naming, functions, SOLID principles, error handling, and Python tooling (ruff, pyright, uv). Use when writing, reviewing, or refactoring code.
---

# Clean Code Guidelines

Focus on Python but applicable to all languages.

## When to use me
- Writing new code or reviewing existing code
- Refactoring or fixing code smells
- Applying SOLID principles
- Ensuring Python/general programming standards

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Variables | Descriptive nouns, snake_case | `user_count`, `total_price` |
| Functions | Verb + noun, snake_case | `calculate_total()`, `validate_email()` |
| Classes | PascalCase, nouns | `UserAccount`, `PaymentProcessor` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `API_TIMEOUT` |
| Booleans | Question form | `is_active`, `has_permission` |

- Names reveal intent without comments
- Pronounceable names (`customer_address` not `cstmr_addr`)
- Avoid abbreviations unless universal (`id`, `url`, `http`)
- Consistent vocabulary (don't mix `fetch`, `get`, `retrieve`)

## Function Design

| Metric | Limit |
|--------|-------|
| Length | 20-30 lines max |
| Parameters | 3-4 max (use dataclass/TypedDict for more) |
| Nesting depth | 2-3 levels max |
| Cyclomatic complexity | <=10 |

- One function = one task
- Use `*` to force keyword-only args for optional params

## Comments & Docs

- Comments explain **WHY**, never WHAT — code explains WHAT
- When to comment: business rules, non-obvious algorithms, edge cases, TODO/FIXME with ticket refs
- Docstrings: Google style — Args, Returns, Raises

## Core Principles

| Principle | Rule |
|-----------|------|
| **DRY** | Extract duplicated logic |
| **KISS** | Prefer obvious over clever |
| **YAGNI** | Build only what's needed now |

## SOLID (Quick)

- **S** — Single Responsibility: one class/function = one reason to change
- **O** — Open/Closed: extend via subclasses/protocols, don't modify existing
- **L** — Liskov: subtypes must be substitutable for base types
- **I** — Interface Segregation: many small `Protocol`s > one fat interface
- **D** — Dependency Inversion: inject abstractions, never instantiate concretions inside

## Python Specifics

- 4 spaces, 88 char lines (Black/Ruff default)
- Imports: one per line, grouped (stdlib → third-party → local)
- Type hints everywhere (Python 3.10+ union syntax: `str | None`)
- Use list comprehensions, `enumerate()`, context managers, f-strings, truthiness checks
- Avoid: mutable default args, bare `except`, `type()` instead of `isinstance()`

## Error Handling

- Custom exception hierarchy inheriting from a base `AppError`
- Catch specific exceptions, log with `exc_info=True`, then re-raise
- Logging levels: DEBUG (diagnostic) → INFO (milestones) → WARNING (recoverable) → ERROR (failed) → CRITICAL (system down)

## Code Smells → Fixes

| Smell | Fix |
|-------|-----|
| Long Method (>30 lines) | Extract methods |
| Large Class (>500 lines) | Split classes |
| Data Clumps | Create dataclass |
| Primitive Obsession | Value objects |
| Feature Envy | Move method to correct class |
| Dead Code | Delete it |

## Tooling (2025-2026)

| Tool | Purpose |
|------|---------|
| **Ruff** | Lint + format (replaces flake8, black, isort) |
| **Pyright** | Type checking |
| **uv** | Package management |
| **pytest** | Testing |
| **pre-commit** | Git hooks |

```toml
# pyproject.toml — Ruff essentials
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E","W","F","I","B","C4","UP","SIM","S","N"]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]
```

## Review Checklist

- [ ] Does code do what it's supposed to? Edge cases handled?
- [ ] Names meaningful? Code self-documenting?
- [ ] SOLID followed? No duplication?
- [ ] Type hints present? Pythonic idioms used?
- [ ] Tests sufficient? Secrets handled properly?
