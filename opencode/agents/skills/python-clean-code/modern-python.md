# Modern Python Patterns (3.10+)

## Type Hints

### Basic Types
```python
# Built-in generics (Python 3.9+)
def process(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

# Union types (Python 3.10+)
def parse(value: str | int) -> float:
    return float(value)

# Optional (None union)
def find_user(user_id: int) -> User | None:
    return users.get(user_id)
```

### Abstract Types for Function Parameters
```python
from collections.abc import Iterable, Sequence, Mapping, Callable

# Accept any iterable (most flexible)
def process_items(items: Iterable[str]) -> list[str]:
    return [item.upper() for item in items]

# Accept sequences (indexable)
def get_first(items: Sequence[str]) -> str:
    return items[0]

# Accept mappings (dict-like)
def get_value(data: Mapping[str, int], key: str) -> int:
    return data[key]

# Accept callables
def apply(func: Callable[[int], int], value: int) -> int:
    return func(value)
```

### Type Aliases
```python
from typing import TypeAlias

UserId: TypeAlias = int
UserDict: TypeAlias = dict[str, str | int]
Handler: TypeAlias = Callable[[str], None]

def get_user(user_id: UserId) -> UserDict:
    ...
```

### TypeVar for Generics
```python
from typing import TypeVar

T = TypeVar('T')

def first(items: list[T]) -> T | None:
    return items[0] if items else None

# Bounded TypeVar
from typing import TypeVar
Numeric = TypeVar('Numeric', int, float)

def add(a: Numeric, b: Numeric) -> Numeric:
    return a + b
```

## Data Classes vs Others

### dataclass (Default Choice)
```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    email: str
    age: int = 0
    tags: list[str] = field(default_factory=list)

# Immutable
@dataclass(frozen=True)
class Point:
    x: float
    y: float

# With slots (memory efficient, Python 3.10+)
@dataclass(slots=True)
class Record:
    id: int
    value: str
```

### NamedTuple (Immutable, Hashable)
```python
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

p = Point(1.0, 2.0)
# p.x = 3.0  # Error: immutable
# Can use as dict key, in sets

# Use when: cache keys, event logs, simple records
```

### Pydantic (External Validation)
```python
from pydantic import BaseModel, Field, field_validator

class UserInput(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str
    age: int = Field(ge=0, le=150)

    @field_validator('email')
    @classmethod
    def validate_email(cls, v: str) -> str:
        if '@' not in v:
            raise ValueError('Invalid email')
        return v.lower()

# Auto-validates on creation
user = UserInput(name="John", email="JOHN@example.com", age=25)
# user.email == "john@example.com"

# Use when: API input, form data, config files
```

### Decision Table
| Need | Use |
|------|-----|
| Simple data container | `@dataclass` |
| Immutable record | `@dataclass(frozen=True)` or `NamedTuple` |
| Hashable (dict key/set) | `NamedTuple` or `@dataclass(frozen=True)` |
| External data validation | `Pydantic BaseModel` |
| Memory efficiency | `@dataclass(slots=True)` |

## Pathlib (Not os.path)

```python
from pathlib import Path

# Creating paths
data_dir = Path("data")
file_path = data_dir / "subdir" / "file.txt"  # Use / operator

# Reading/writing
content = Path("file.txt").read_text(encoding="utf-8")
Path("output.txt").write_text("content", encoding="utf-8")
data = Path("data.json").read_bytes()

# Path operations
path = Path("data/file.txt")
path.exists()           # True/False
path.is_file()          # True/False
path.is_dir()           # True/False
path.stem               # "file" (without extension)
path.suffix             # ".txt"
path.parent             # Path("data")
path.name               # "file.txt"
path.resolve()          # Absolute path

# Directory operations
Path("new_dir").mkdir(parents=True, exist_ok=True)

# Glob patterns
for py_file in Path(".").glob("**/*.py"):
    print(py_file)

# Iterating directory
for item in Path(".").iterdir():
    if item.is_file():
        print(item)
```

## Context Managers

```python
# Custom context manager (class)
class DatabaseConnection:
    def __enter__(self):
        self.conn = connect_to_db()
        return self.conn
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()
        return False  # Don't suppress exceptions

# Custom context manager (decorator)
from contextlib import contextmanager

@contextmanager
def temp_file(content: str):
    import tempfile
    import os
    
    path = tempfile.mktemp()
    try:
        with open(path, 'w') as f:
            f.write(content)
        yield path
    finally:
        os.unlink(path)

with temp_file("test content") as path:
    process_file(path)
```

## F-Strings

```python
name = "Alice"
age = 30

# Basic
message = f"Hello {name}, you are {age} years old"

# Expressions
total = f"Total: ${price * quantity:.2f}"

# Debugging (Python 3.8+)
print(f"{value=}")  # Output: value=42

# Multi-line
message = (
    f"User: {name}\n"
    f"Age: {age}\n"
    f"Status: {'active' if age >= 18 else 'inactive'}"
)

# ⚠️ Don't use f-strings for:
# - Logging (use % for lazy evaluation)
logger.info("User %s logged in", username)

# - SQL (use parameterized queries)
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

## Walrus Operator (:=)

```python
# Assignment expression
if (n := len(data)) > 10:
    print(f"Processing {n} items")

# In list comprehensions
results = [y for x in data if (y := expensive_func(x)) is not None]

# In while loops
while (line := file.readline()):
    process(line)
```

## Match Statement (Python 3.10+)

```python
def handle_command(command: dict):
    match command:
        case {"action": "quit"}:
            return "Goodbye"
        case {"action": "greet", "name": name}:
            return f"Hello, {name}"
        case {"action": "add", "x": x, "y": y}:
            return x + y
        case _:
            return "Unknown command"

# With guards
match value:
    case int(n) if n > 0:
        return "positive"
    case int(n) if n < 0:
        return "negative"
    case int():
        return "zero"
```
