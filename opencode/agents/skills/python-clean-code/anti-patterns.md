# Python Anti-Patterns & Code Smells

## Critical Anti-Patterns

### 1. Mutable Default Arguments ⚠️
```python
# ❌ BUG: Default list is shared across calls!
def append_to(element, to=[]):
    to.append(element)
    return to

append_to(1)  # [1]
append_to(2)  # [1, 2] - NOT [2]!

# ✅ CORRECT: Use None and create new list
def append_to(element, to=None):
    if to is None:
        to = []
    to.append(element)
    return to
```

### 2. Bare Except Clauses ⚠️
```python
# ❌ Catches KeyboardInterrupt, SystemExit, etc!
try:
    risky_operation()
except:
    handle_error()

# ❌ Still too broad
except Exception:
    pass

# ✅ CORRECT: Catch specific exceptions
try:
    risky_operation()
except ValueError as e:
    handle_value_error(e)
except FileNotFoundError as e:
    handle_file_error(e)
except Exception as e:
    logger.error("Unexpected error: %s", e)
    raise  # Re-raise if you can't handle it
```

### 3. Using `type()` Instead of `isinstance()`
```python
# ❌ Doesn't work with inheritance
if type(obj) == int:
    process_int(obj)

# ✅ CORRECT: Works with subclasses
if isinstance(obj, int):
    process_int(obj)

# ✅ Multiple types
if isinstance(obj, (int, float)):
    process_number(obj)
```

### 4. Comparing to None/True/False Incorrectly
```python
# ❌ Wrong
if x == None:
if x == True:
if x == False:

# ✅ CORRECT
if x is None:
if x is True:  # or just: if x:
if x is False:  # or just: if not x:

# ⚠️ Exception: When you need to distinguish False from None
if x is False:  # Explicitly False, not just falsy
```

### 5. Global Mutable State
```python
# ❌ Hard to test, race conditions
_cache = {}
_connection = None

def get_data(key):
    global _connection
    if _connection is None:
        _connection = create_connection()
    return _cache.get(key)

# ✅ CORRECT: Dependency injection
class DataService:
    def __init__(self, connection, cache=None):
        self.connection = connection
        self.cache = cache or {}
    
    def get_data(self, key):
        return self.cache.get(key)

# Easy to test
service = DataService(mock_connection, {})
```

## Code Smells

### Long Function (>15 lines)
```python
# ❌ Too long, multiple responsibilities
def process_order(order):
    # Validate (5 lines)
    if not order.items: raise ValueError()
    # ... more validation
    
    # Calculate tax (10 lines)
    if order.country == "US": tax = 0.08
    # ... more calculation
    
    # Apply discount (8 lines)
    # ...
    
    # Save to database (5 lines)
    # ...

# ✅ Extract functions
def validate_order(order): ...
def calculate_tax(order): ...
def calculate_discount(order): ...
def save_order(order): ...

def process_order(order):
    validate_order(order)
    tax = calculate_tax(order)
    discount = calculate_discount(order)
    order.total = order.subtotal + tax - discount
    save_order(order)
```

### Too Many Parameters
```python
# ❌ Parameter overload
def create_user(name, email, phone, address, city, 
                state, zip_code, country, role, department):
    ...

# ✅ Use dataclass/NamedTuple
from dataclasses import dataclass

@dataclass
class Address:
    street: str
    city: str
    state: str
    zip_code: str
    country: str

@dataclass
class UserInfo:
    name: str
    email: str
    phone: str
    address: Address

def create_user(user_info: UserInfo, role: str, department: str):
    ...
```

### Magic Numbers/Strings
```python
# ❌ What do these mean?
if user.age >= 18:
    allow_access()

if retry_count > 3:
    raise Error()

if status == "A":
    process()

# ✅ Named constants
LEGAL_AGE = 18
MAX_RETRIES = 3

class Status:
    ACTIVE = "A"
    INACTIVE = "I"

if user.age >= LEGAL_AGE:
    allow_access()

if retry_count > MAX_RETRIES:
    raise Error()

if status == Status.ACTIVE:
    process()
```

### Deeply Nested Conditionals
```python
# ❌ Pyramid of doom
def process(data):
    if data:
        if data.is_valid():
            if data.user:
                if data.user.is_active:
                    return do_something(data)
    return None

# ✅ Early returns (guard clauses)
def process(data):
    if not data:
        return None
    if not data.is_valid():
        return None
    if not data.user:
        return None
    if not data.user.is_active:
        return None
    return do_something(data)
```

### Feature Envy
```python
# ❌ Method uses other class's data excessively
class Order:
    def print_customer_info(self):
        print(f"Name: {self.customer.get_name()}")
        print(f"Email: {self.customer.get_email()}")
        print(f"Phone: {self.customer.get_phone()}")
        print(f"Address: {self.customer.get_address()}")

# ✅ Move method to where data lives
class Customer:
    def get_contact_info(self) -> str:
        return f"Name: {self.name}\nEmail: {self.email}\n..."

class Order:
    def print_customer_info(self):
        print(self.customer.get_contact_info())
```

### Primitive Obsession
```python
# ❌ Using primitives for domain concepts
def process_money(amount: float, currency: str):
    ...

def validate_email(email: str):
    ...

# ✅ Value objects
from dataclasses import dataclass
import re

@dataclass(frozen=True)
class Money:
    amount: float
    currency: str
    
    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)

@dataclass(frozen=True)
class Email:
    value: str
    
    def __post_init__(self):
        if not re.match(r"[^@]+@[^@]+\.[^@]+", self.value):
            raise ValueError(f"Invalid email: {self.value}")
```

## Refactoring Patterns

### Extract Method
Before → After (see Long Function above)

### Introduce Parameter Object
Before → After (see Too Many Parameters above)

### Replace Conditional with Polymorphism
```python
# ❌ Type checking with conditionals
def get_speed(bird):
    if bird.type == "European":
        return bird.base_speed
    elif bird.type == "African":
        return bird.base_speed - bird.load_factor
    elif bird.type == "Norwegian":
        return 0 if bird.is_nailed else bird.voltage * 2

# ✅ Polymorphism
from abc import ABC, abstractmethod

class Bird(ABC):
    @abstractmethod
    def get_speed(self) -> float: ...

class EuropeanBird(Bird):
    def get_speed(self) -> float:
        return self.base_speed

class AfricanBird(Bird):
    def get_speed(self) -> float:
        return self.base_speed - self.load_factor

class NorwegianBird(Bird):
    def get_speed(self) -> float:
        return 0 if self.is_nailed else self.voltage * 2
```

### Replace Conditional with Dictionary Dispatch
```python
# ❌ Long if-elif chain
def handle_command(cmd):
    if cmd == "start":
        start()
    elif cmd == "stop":
        stop()
    elif cmd == "restart":
        restart()
    elif cmd == "status":
        status()
    else:
        raise ValueError(f"Unknown: {cmd}")

# ✅ Dictionary dispatch
def handle_command(cmd):
    handlers = {
        "start": start,
        "stop": stop,
        "restart": restart,
        "status": status,
    }
    handler = handlers.get(cmd)
    if handler is None:
        raise ValueError(f"Unknown: {cmd}")
    handler()
```

## Quick Detection Checklist

| Smell | Detection | Fix |
|-------|-----------|-----|
| Long function | >15 lines | Extract methods |
| Many params | >4 parameters | Parameter object |
| Magic values | Hardcoded numbers/strings | Named constants |
| Deep nesting | >2 levels | Guard clauses |
| Duplicate code | Similar blocks x3 | Extract shared function |
| Feature envy | Uses other object's data | Move method |
| God class | >10 methods | Split by responsibility |
| Primitive obsession | str/int for domain concepts | Value objects |
