# Python Enum – Quick Reference

## 1. What is Enum?

`Enum` (Enumeration) is a Python class used to define a collection of named constant values.

It helps prevent inconsistent values and spelling mistakes.

```python
from enum import Enum

class Genre(Enum):
    METAL = "Metal"
    POP_ROCK = "Pop Rock"
    ALTERNATIVE = "Alternative"
    HEAVY = "Heavy"
```

## 2. Accessing Enum Members

```python
print(Genre.METAL)
# Genre.METAL

print(Genre.METAL.name)
# METAL

print(Genre.METAL.value)
# Metal
```

- `.name` returns the member's name.
- `.value` returns its assigned value.

## 3. Comparing Enum Members

```python
Genre.METAL == Genre.METAL  # True
Genre.METAL == Genre.POP_ROCK  # False

Genre.METAL == "Metal"  # False
Genre.METAL.value == "Metal"  # True
```

Normal Enum members are not ordinary strings.

## 4. String Enum

Use `str, Enum` when you want enumeration members to behave like strings.

```python
from enum import Enum

class Genre(str, Enum):
    METAL = "Metal"
    POP_ROCK = "Pop Rock"

print(Genre.METAL == "Metal")  # True
```

Alternatively, Python 3.11+ provides `StrEnum`:

```python
from enum import StrEnum

class Genre(StrEnum):
    METAL = "Metal"
    POP_ROCK = "Pop Rock"
```

## 5. Integer Enum

Use `IntEnum` for integer-based enumeration.

```python
from enum import IntEnum

class Status(IntEnum):
    PENDING = 1
    PROCESSING = 2
    COMPLETED = 3

print(Status.PENDING.value)  # 1
print(Status.PENDING == 1)  # True
```

## 6. Iterating Through Enum

Enum classes are iterable.

```python
for genre in Genre:
    print(genre.name, genre.value)
```

## 7. Looking Up Enum Members

**Using the member name:**

```python
Genre["METAL"]
```

**Using the member value:**

```python
Genre("Metal")
```

Invalid values raise `ValueError`.

## 8. Automatic Values with auto()

```python
from enum import Enum, auto

class Role(Enum):
    ADMIN = auto()
    CUSTOMER = auto()
    STAFF = auto()
```

`auto()` automatically generates values.

## 9. Prevent Duplicate Values with unique

```python
from enum import Enum, unique

@unique
class Status(Enum):
    PENDING = 1
    COMPLETED = 2
    CANCELLED = 3
```

`@unique` prevents duplicate values.

## 10. Other Enum Classes

| Class | Purpose |
|---|---|
| `Enum` | General enumeration |
| `IntEnum` | Integer enumeration |
| `StrEnum` | String enumeration (Python 3.11+) |
| `Flag` | Combine members using bitwise operations |
| `IntFlag` | Integer-based flags |

## 11. Practical Example

```python
from enum import StrEnum

class Genre(StrEnum):
    METAL = "Metal"
    POP_ROCK = "Pop Rock"
    ALTERNATIVE = "Alternative"

BANDS = [
    {"id": 1, "name": "Coldplay", "genre": Genre.ALTERNATIVE},
    {"id": 2, "name": "Metallica", "genre": Genre.METAL},
    {"id": 3, "name": "Maroon 5", "genre": Genre.POP_ROCK},
]

def get_bands(genre: Genre):
    return [
        band for band in BANDS
        if band["genre"] == genre
    ]

print(get_bands(Genre.METAL))
```

## 12. Where Are Enums Useful?

- User roles: ADMIN, CUSTOMER, STAFF.
- Order statuses: PENDING, SHIPPED, DELIVERED.
- Payment statuses: PAID, UNPAID, FAILED.
- Product categories.
- API request validation.
- Application configuration.
- Network protocol states.

## Important Notes

1. Enum members are accessed using `ClassName.MEMBER`.
2. `.name` retrieves the member name.
3. `.value` retrieves the assigned value.
4. Enum classes are iterable.
5. Invalid value lookups raise `ValueError`.
6. `str, Enum` and `StrEnum` make members string-compatible.
7. `auto()` generates values automatically.

**Official documentation:** https://docs.python.org/3/library/enum.html
