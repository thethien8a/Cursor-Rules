# Clean Code Principles for Python

## SOLID Principles

### Single Responsibility Principle (SRP)

Each class should have one reason to change.

```python
# ❌ Bad: Multiple responsibilities
class UserManager:
    def create_user(self): ...
    def send_email(self): ...
    def generate_report(self): ...
    def validate_data(self): ...

# ✅ Good: Separated concerns
class UserRepository:
    def create_user(self): ...
    def get_user(self): ...

class EmailService:
    def send_email(self): ...

class ReportGenerator:
    def generate_report(self): ...
```

### Open/Closed Principle (OCP)

Open for extension, closed for modification.

```python
# ❌ Bad: Modify existing code for new shapes
class Shape:
    def calculate_area(self):
        if self.type == "rectangle":
            return self.width * self.height
        elif self.type == "circle":
            return pi * self.radius ** 2
        # Need to modify for each new shape

# ✅ Good: Extend via new classes
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def calculate_area(self) -> float:
        pass

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height
    
    def calculate_area(self) -> float:
        return self.width * self.height

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    
    def calculate_area(self) -> float:
        return pi * self.radius ** 2
```

### Liskov Substitution Principle (LSP)

Subtypes must be substitutable for base types.

```python
# ❌ Bad: Square violates Rectangle's contract
class Rectangle:
    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

class Square(Rectangle):
    def set_width(self, w):
        self.width = self.height = w  # Violates contract!

# ✅ Good: Shared abstract base
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: pass

class Rectangle(Shape):
    def __init__(self, w: float, h: float):
        self.width, self.height = w, h
    def area(self) -> float:
        return self.width * self.height

class Square(Shape):
    def __init__(self, side: float):
        self.side = side
    def area(self) -> float:
        return self.side ** 2
```

### Interface Segregation Principle (ISP)

Clients shouldn't depend on methods they don't use.

```python
# ❌ Bad: Fat interface
class Printer(ABC):
    @abstractmethod
    def print(self, doc): pass
    @abstractmethod
    def fax(self, doc): pass
    @abstractmethod
    def scan(self, doc): pass

class OldPrinter(Printer):
    def fax(self, doc):
        raise NotImplementedError  # Forced to implement!

# ✅ Good: Segregated interfaces
class Printer(ABC):
    @abstractmethod
    def print(self, doc): pass

class Fax(ABC):
    @abstractmethod
    def fax(self, doc): pass

class Scanner(ABC):
    @abstractmethod
    def scan(self, doc): pass

class OldPrinter(Printer):
    def print(self, doc): ...  # Only implements what it can

class ModernPrinter(Printer, Fax, Scanner):
    def print(self, doc): ...
    def fax(self, doc): ...
    def scan(self, doc): ...
```

### Dependency Inversion Principle (DIP)

Depend on abstractions, not concrete implementations.

```python
# ❌ Bad: High-level depends on low-level
class FrontEnd:
    def __init__(self):
        self.back_end = MySQLDatabase()  # Hardcoded!
    
    def display_data(self):
        return self.back_end.get_data()

# ✅ Good: Depend on abstraction
class DataSource(ABC):
    @abstractmethod
    def get_data(self) -> dict: pass

class FrontEnd:
    def __init__(self, data_source: DataSource):
        self.data_source = data_source  # Injected!
    
    def display_data(self):
        return self.data_source.get_data()

# Usage - easily swappable
front_end = FrontEnd(MySQLDatabase())
front_end = FrontEnd(PostgresDatabase())
front_end = FrontEnd(MockDatabase())  # For testing
```

## DRY (Don't Repeat Yourself)

```python
# ❌ Bad: Duplicated logic
def check_guest(guest):
    return guest in ["Mia", "John", "Linda"]

def validate_user(user):
    return user in ["Mia", "John", "Linda"]

# ✅ Good: Single source of truth
ALLOWED_USERS = frozenset(["Mia", "John", "Linda"])

def is_allowed_user(name: str) -> bool:
    return name in ALLOWED_USERS
```

**Rule of Three**: Refactor after seeing same pattern 3 times.

## KISS (Keep It Simple)

```python
# ❌ Clever but complex
def get_status(value):
    return {True: "active", False: "inactive", None: "unknown"}.get(
        value, "unknown"
    ) if isinstance(value, bool) or value is None else "invalid"

# ✅ Simple and clear
def get_status(value: bool | None) -> str:
    if value is True:
        return "active"
    elif value is False:
        return "inactive"
    elif value is None:
        return "unknown"
    return "invalid"
```

## YAGNI (You Ain't Gonna Need It)

```python
# ❌ Bad: Building for hypothetical future
class UserManager:
    def __init__(self):
        self.storage_type = "memory"  # Maybe database later?
        self.cache_enabled = False    # Maybe caching later?
        self.plugins = []             # Maybe plugins later?
    
    def save_user(self, user):
        if self.storage_type == "memory":
            self.storage[user.id] = user
        elif self.storage_type == "database":
            pass  # TODO: implement someday
        elif self.storage_type == "file":
            pass  # TODO: implement someday

# ✅ Good: Build what you need now
class UserManager:
    def __init__(self):
        self.storage: dict[int, User] = {}
    
    def save_user(self, user: User) -> None:
        self.storage[user.id] = user
```

Refactor when you actually need the feature.
