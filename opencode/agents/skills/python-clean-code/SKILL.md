---
name: python-clean-code
description: Apply Python clean code principles, best practices, and modern patterns. Use when writing, reviewing, or refactoring Python code. Covers PEP 8 style, SOLID principles, type hints, code smells detection, and tooling setup (ruff, mypy, uv). Activate for any Python development task.
---

# Python Clean Code & Best Practices

## Quick Reference

### Naming Conventions
```python
# Variables/functions: snake_case
user_name = "John"
def calculate_total_price(): pass

# Classes: PascalCase
class UserAccount: pass

# Constants: UPPER_CASE
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30

# Private: single underscore
def _internal_helper(): pass
```

### Function Design Checklist
- [ ] Single responsibility (does ONE thing)
- [ ] Max 3-4 parameters (use dataclass for more)
- [ ] Clear return type annotation
- [ ] Docstring with Args/Returns/Raises
- [ ] No side effects when possible (pure functions)
- [ ] Under 15 lines (extract if longer)

### Type Hints (Python 3.10+)
```python
# Modern syntax
def process(items: list[str]) -> dict[str, int]: ...
def find_user(id: int) -> User | None: ...

# Use abstract types for inputs
from collections.abc import Iterable, Sequence
def process_items(items: Iterable[str]) -> list[str]: ...
```

### Error Handling
```python
# Specific exceptions, not bare except
try:
    value = data[key]
except KeyError:
    return default_value

# Exception chaining
except FileNotFoundError as e:
    raise ProcessingError(f"Cannot process {filename}") from e

# Context managers for resources
with open('file.txt') as f:
    content = f.read()
```

### Code Quality Commands
```bash
# Ruff (linting + formatting)
ruff check . --fix
ruff format .

# Type checking
mypy src/
pyright src/

# Pre-commit (all checks)
pre-commit run --all-files
```

## Decision Trees

### When to Use What Data Structure
```
Need validation from external data? → Pydantic BaseModel
Need immutable, hashable record?   → NamedTuple
Need mutable domain object?        → @dataclass
Need dict-like access?             → TypedDict
Simple data container?             → @dataclass(frozen=True)
```

### Code Smell → Fix
```
Long function (>15 lines)     → Extract smaller functions
Too many parameters (>4)      → Introduce parameter object
Repeated code blocks          → Extract to shared function
Magic numbers/strings         → Define named constants
Nested conditionals           → Early returns or polymorphism
Global mutable state          → Dependency injection
```

## SOLID Quick Reference

| Principle | Rule | Python Pattern |
|-----------|------|----------------|
| **S**ingle Responsibility | One reason to change | Split classes by concern |
| **O**pen/Closed | Extend, don't modify | ABC + inheritance |
| **L**iskov Substitution | Subtypes are substitutable | Honor base contracts |
| **I**nterface Segregation | Small, focused interfaces | Multiple ABCs/Protocols |
| **D**ependency Inversion | Depend on abstractions | Constructor injection |

## Modern Python Patterns

### Prefer
```python
# Pathlib over os.path
from pathlib import Path
content = Path("file.txt").read_text()

# f-strings over .format()
message = f"Hello {name}, you are {age} years old"

# Walrus operator for assignment in conditions
if (n := len(data)) > 10:
    print(f"Processing {n} items")

# Dictionary merge (3.9+)
merged = dict1 | dict2
```

### Avoid
```python
# Mutable default arguments
def bad(items=[]): ...      # BUG!
def good(items=None): 
    items = items or []     # CORRECT

# Bare except
except:                     # NEVER
except Exception as e:      # OK if needed

# type() for type checking
if type(x) == int:          # BAD
if isinstance(x, int):      # GOOD
```

## Additional Resources

For detailed guidance on specific topics (portable paths):
- `./style-guide.md` - PEP 8 conventions, imports, whitespace
- `./clean-code.md` - SOLID principles with examples
- `./modern-python.md` - Type hints, dataclasses, async
- `./tooling.md` - Ruff, mypy, pre-commit, uv configuration
- `./anti-patterns.md` - Code smells and refactoring patterns

## Checklist Before Committing

```
Code Quality:
- [ ] ruff check passes
- [ ] ruff format applied
- [ ] mypy/pyright passes (or warnings understood)
- [ ] Tests pass with good coverage

Style:
- [ ] Functions have type hints
- [ ] Public functions have docstrings
- [ ] No magic numbers (use constants)
- [ ] Descriptive variable/function names

Architecture:
- [ ] Single responsibility per function/class
- [ ] No circular imports
- [ ] Dependencies injected, not hardcoded
```
