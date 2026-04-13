# Python Style Guide (PEP 8)

## Code Layout

### Indentation
- 4 spaces per level (no tabs)
- Continuation lines: align with opening delimiter or use hanging indent

```python
# Aligned with opening delimiter
foo = long_function_name(var_one, var_two,
                         var_three, var_four)

# Hanging indent (add 4 spaces)
def long_function_name(
        var_one, var_two, var_three,
        var_four):
    print(var_one)
```

### Line Length
- Max 79 characters (strict)
- Max 99 characters (acceptable for teams)
- Max 72 for docstrings/comments

### Blank Lines
- 2 blank lines around top-level functions/classes
- 1 blank line around method definitions
- Use sparingly within functions for logical sections

### Imports
```python
# Order: stdlib → third-party → local
import os
import sys

import requests
import pandas as pd

from myproject import utils
from myproject.models import User

# One import per line
# Use absolute imports (preferred)
# Avoid wildcard imports: from module import *
```

## Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Package | lowercase | `mypackage` |
| Module | snake_case | `my_module.py` |
| Class | PascalCase | `MyClass` |
| Exception | PascalCase + Error | `ValidationError` |
| Function | snake_case | `calculate_total` |
| Variable | snake_case | `user_count` |
| Constant | UPPER_SNAKE | `MAX_SIZE` |
| Private | _leading | `_internal_func` |
| Name mangling | __double | `__private_attr` |
| Magic | __dunder__ | `__init__` |

### Naming Best Practices
```python
# Functions: verbs
def get_user(): ...
def calculate_tax(): ...
def validate_input(): ...
def is_valid(): ...  # Boolean returns

# Classes: nouns
class UserManager: ...
class PaymentProcessor: ...
class HTTPClient: ...  # Acronyms in caps

# Variables: descriptive nouns
user_count = 10
active_users = []
config_path = Path("config.yaml")

# Avoid: single letters (except i,j,k for loops, x,y,z for coords)
# Avoid: abbreviations unless widely known (URL, HTTP, ID)
```

## Whitespace

```python
# Around operators
x = 1 + 2
y = x * 2

# No space before function call
spam(1)
dict['key']

# No space inside brackets
spam(ham[1], {eggs: 2})

# Space after comma
func(a, b, c)

# Space around = in keyword args with type hints
def func(param: str = "default") -> None: ...

# No space around = in keyword args without type hints
func(arg=value)
```

## Comments

```python
# Block comments: full sentences, 2 spaces before inline
x = x + 1  # Compensate for border

# Docstrings: first line is summary
def process(data):
    """Process data and return results.
    
    Args:
        data: Input data to process.
    
    Returns:
        Processed results as dictionary.
    
    Raises:
        ValueError: If data is empty.
    """
```

## String Quotes

- Single `'` or double `"` - be consistent
- Triple quotes `"""` for docstrings
- Use opposite quote to avoid escaping: `"It's a test"`
