# PYDANTIC — Complete Practical Reference for FastAPI

## 1. What is Pydantic?

Pydantic is a Python library used for:

- Data validation.
- Data parsing and conversion.
- Defining the structure of data.
- Serializing Python objects into JSON-compatible data.
- Validating data received from clients.
- Managing application configuration.

In FastAPI, Pydantic is commonly used to define request bodies and response structures.

### Example

Imagine a client sends this JSON to your API:

```json
{
    "name": "Nzegge",
    "age": 25,
    "email": "nzegge@example.com"
}
```

You want to ensure that:

- `name` is a string.
- `age` is an integer.
- `email` is a valid email address.

Pydantic helps you enforce these requirements.

---

## 2. Installation

Using `uv`:

```bash
uv add pydantic
```

For email validation:

```bash
uv add "pydantic[email]"
```

Pydantic is already installed as a dependency of FastAPI, but installing it explicitly is useful when your project uses it directly.

---

## 3. Your First Pydantic Model

Pydantic models are Python classes that inherit from `BaseModel`.

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str
```

Create an instance:

```python
user = User(
    name="Nzegge",
    age=25,
    email="nzegge@example.com"
)

print(user.name)
print(user.age)
print(user.email)
```

Output:

```text
Nzegge
25
nzegge@example.com
```

### Understanding what happens

1. You define the expected fields inside the class.
2. Each field has a type annotation.
3. You create an instance using the supplied data.
4. Pydantic validates the supplied values.
5. If validation succeeds, you get a model instance.
6. If validation fails, Pydantic raises a `ValidationError`.

### Invalid data example

```python
from pydantic import BaseModel, ValidationError

class User(BaseModel):
    name: str
    age: int

try:
    user = User(name="Nzegge", age="twenty")
except ValidationError as error:
    print(error)
```

Pydantic rejects `"twenty"` because it cannot convert that value into an integer.

---

# 4. Important Pydantic Concepts

## 4.1 BaseModel

`BaseModel` is the foundation of ordinary Pydantic models.

```python
from pydantic import BaseModel

class Product(BaseModel):
    name: str
    price: float
    quantity: int
```

Important things you get from `BaseModel` include:

| Method / property | Purpose |
|---|---|
| `model_validate()` | Validate data and create a model instance |
| `model_validate_json()` | Validate JSON data |
| `model_dump()` | Convert model data into a Python dictionary |
| `model_dump_json()` | Convert model data into a JSON string |
| `model_json_schema()` | Generate JSON Schema |
| `model_copy()` | Create a copy of a model |
| `model_fields` | Access information about model fields |
| `model_fields_set` | See which fields were explicitly supplied |

### model_validate()

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

data = {
    "name": "Nzegge",
    "age": 25
}

user = User.model_validate(data)

print(user)
```

You can use this when you already have a Python dictionary and want to validate it.

### model_validate_json()

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

json_data = '{"name": "Nzegge", "age": 25}'

user = User.model_validate_json(json_data)

print(user)
```

This validates JSON text directly.

### model_dump()

```python
user_data = user.model_dump()

print(user_data)
```

Output:

```python
{
    "name": "Nzegge",
    "age": 25
}
```

Useful parameters:

```python
user.model_dump(
    include={"name"},
    exclude={"age"},
    exclude_unset=True,
    exclude_defaults=True,
    exclude_none=True,
    mode="python"
)
```

| Parameter | Meaning |
|---|---|
| `include` | Select fields to include |
| `exclude` | Select fields to exclude |
| `exclude_unset` | Exclude fields not explicitly supplied |
| `exclude_defaults` | Exclude fields equal to their defaults |
| `exclude_none` | Exclude fields whose values are `None` |
| `mode` | Choose Python or JSON-compatible output |

### model_dump_json()

```python
json_data = user.model_dump_json()

print(json_data)
```

Output:

```json
{"name":"Nzegge","age":25}
```

Useful parameters include `indent`, `include`, `exclude`, `exclude_unset`, and `exclude_none`.

### model_copy()

```python
new_user = user.model_copy()

updated_user = user.model_copy(
    update={"age": 30}
)

print(updated_user)
```

`update` supplies replacement values in the copied model. It does not independently validate those update values.

Use `deep=True` when you need a deep copy of nested objects.

### model_json_schema()

```python
schema = User.model_json_schema()

print(schema)
```

This generates a JSON Schema describing the model.

---

## 4.2 Field()

`Field()` allows you to configure individual model fields.

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    username: str = Field(
        min_length=3,
        max_length=30
    )

    age: int = Field(
        gt=0,
        lt=120
    )
```

### Important Field parameters

| Parameter | Purpose |
|---|---|
| `default` | Default value |
| `default_factory` | Function used to generate a default |
| `alias` | Alternative input/output field name |
| `title` | Field title |
| `description` | Field description |
| `examples` | Example values |
| `gt` | Greater than |
| `ge` | Greater than or equal to |
| `lt` | Less than |
| `le` | Less than or equal to |
| `multiple_of` | Value must be a multiple of a number |
| `min_length` | Minimum string or collection length |
| `max_length` | Maximum string or collection length |
| `pattern` | Regular expression constraint |
| `strict` | Require strict type validation |
| `frozen` | Prevent reassignment of the field |
| `exclude` | Exclude field during serialization |
| `repr` | Control whether field appears in model representation |
| `validate_default` | Validate the default value |

### Example: string validation

```python
class User(BaseModel):
    username: str = Field(
        min_length=3,
        max_length=20,
        description="User's unique username"
    )
```

### Example: numeric validation

```python
class Product(BaseModel):
    price: float = Field(gt=0)
    quantity: int = Field(ge=0)
```

This means:

- Price must be greater than zero.
- Quantity must be greater than or equal to zero.

### default and default_factory

```python
from datetime import datetime, timezone
from pydantic import BaseModel, Field

class User(BaseModel):
    name: str

    active: bool = True

    tags: list[str] = Field(default_factory=list)

    created_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
```

`default_factory` calls a function to generate the value when a value is not supplied.

This is especially useful for mutable defaults and dynamically generated values.

---

## 4.3 Optional Fields and Default Values

Consider:

```python
class User(BaseModel):
    name: str
    age: int | None = None
```

Here:

- `name` is required.
- `age` is optional.
- If `age` is not provided, its value becomes `None`.

Important distinction:

```python
age: int | None
```

This field accepts an integer or `None`, but it is still required.

However:

```python
age: int | None = None
```

This field accepts an integer or `None` and has a default value, so it is not required.

Other examples:

```python
class Product(BaseModel):
    name: str
    price: float = 0.0
    description: str | None = None
    available: bool = True
```

---

# 5. Important Pydantic Field Types

Pydantic supports normal Python types and additional specialized types.

## 5.1 Basic Python types

```python
class Example(BaseModel):
    name: str
    age: int
    price: float
    active: bool
    value: None
```

Common types:

| Type | Meaning |
|---|---|
| `str` | String |
| `int` | Integer |
| `float` | Floating-point number |
| `bool` | Boolean |
| `bytes` | Bytes |
| `None` | None value |

## 5.2 Collection types

```python
class Example(BaseModel):
    tags: list[str]
    scores: tuple[int, ...]
    unique_ids: set[int]
    metadata: dict[str, str]
```

You can also use nested types:

```python
class Example(BaseModel):
    users: list[dict[str, int]]
```

## 5.3 EmailStr

Install email support:

```bash
uv add "pydantic[email]"
```

Example:

```python
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    name: str
    email: EmailStr
```

Pydantic checks whether the supplied value has a valid email format.

## 5.4 HttpUrl

```python
from pydantic import BaseModel, HttpUrl

class Website(BaseModel):
    url: HttpUrl
```

This validates HTTP and HTTPS URLs.

## 5.5 UUID

```python
from uuid import UUID
from pydantic import BaseModel

class User(BaseModel):
    id: UUID
```

Useful when working with UUID-based identifiers.

## 5.6 Date and time types

```python
from datetime import date, datetime, time, timedelta
from pydantic import BaseModel

class Event(BaseModel):
    event_date: date
    event_time: time
    created_at: datetime
    duration: timedelta
```

Other useful types include:

- `Decimal`
- `Path`
- `IPvAnyAddress`
- `IPvAnyNetwork`
- `SecretStr`
- `SecretBytes`

---

# 6. Nested Pydantic Models

Models can contain other models.

This is very useful when an API receives complex JSON data.

```python
from pydantic import BaseModel

class Address(BaseModel):
    city: str
    country: str

class User(BaseModel):
    name: str
    age: int
    address: Address
```

Create an instance:

```python
user = User(
    name="Nzegge",
    age=25,
    address={
        "city": "Buea",
        "country": "Cameroon"
    }
)

print(user.address.city)
```

Pydantic automatically validates the nested address data.

### Nested lists

```python
class Product(BaseModel):
    name: str
    price: float

class Order(BaseModel):
    order_id: int
    products: list[Product]
```

Example:

```python
order = Order(
    order_id=1,
    products=[
        {"name": "Laptop", "price": 500},
        {"name": "Mouse", "price": 20}
    ]
)
```

---

# 7. Field and Model Validators

Validators allow you to implement custom validation logic.

Pydantic V2 provides:

- `field_validator`
- `model_validator`

## 7.1 field_validator

Used to validate or transform individual fields.

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    username: str

    @field_validator("username")
    @classmethod
    def validate_username(cls, value: str) -> str:
        if len(value) < 3:
            raise ValueError("Username must contain at least 3 characters")

        return value.lower()
```

Example:

```python
user = User(username="NZEGGE")

print(user.username)
```

Output:

```text
nzegge
```

The validator runs when the model is created or validated.

### Important field_validator modes

```python
@field_validator("username", mode="before")
```

Receives the raw input before normal validation.

```python
@field_validator("username", mode="after")
```

Receives the value after normal validation. This is the default.

```python
@field_validator("username", mode="plain")
```

Replaces normal validation for that field.

```python
@field_validator("username", mode="wrap")
```

Wraps normal validation and can control when it runs.

## 7.2 model_validator

Used when validation depends on multiple fields.

Example: password confirmation.

```python
from pydantic import BaseModel, model_validator

class User(BaseModel):
    password: str
    confirm_password: str

    @model_validator(mode="after")
    def check_passwords(self):
        if self.password != self.confirm_password:
            raise ValueError("Passwords do not match")

        return self
```

The validator checks the relationship between both fields.

### model_validator modes

| Mode | Purpose |
|---|---|
| `before` | Validate or modify raw input before model validation |
| `after` | Validate the completed model instance |
| `wrap` | Wrap the normal model validation process |

---

# 8. ConfigDict — Model Configuration

`ConfigDict` allows you to configure how a Pydantic model behaves.

```python
from pydantic import BaseModel, ConfigDict

class User(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        str_strip_whitespace=True,
        validate_assignment=True
    )

    name: str
    age: int
```

### Important ConfigDict options

| Option | Purpose |
|---|---|
| `extra` | Controls additional unexpected fields |
| `strict` | Enables strict validation |
| `str_strip_whitespace` | Removes surrounding whitespace from strings |
| `str_to_lower` | Converts strings to lowercase |
| `str_to_upper` | Converts strings to uppercase |
| `validate_assignment` | Validates field assignment |
| `validate_default` | Validates default values |
| `frozen` | Prevents model field reassignment |
| `populate_by_name` | Allows population using field names alongside aliases |
| `from_attributes` | Allows validation from object attributes |
| `arbitrary_types_allowed` | Allows arbitrary Python types |
| `use_enum_values` | Uses Enum values during validation |
| `json_schema_extra` | Adds extra JSON Schema information |
| `alias_generator` | Generates aliases automatically |

### Understanding extra

```python
model_config = ConfigDict(extra="forbid")
```

Rejects unexpected fields.

```python
model_config = ConfigDict(extra="ignore")
```

Ignores unexpected fields.

```python
model_config = ConfigDict(extra="allow")
```

Allows unexpected fields and stores them.

### Understanding from_attributes

This is particularly useful when working with SQLAlchemy ORM objects.

```python
from pydantic import BaseModel, ConfigDict

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    username: str
```

Pydantic can read `id` and `username` from an object's attributes instead of requiring a dictionary.

---

# 9. Field Aliases

Aliases allow a model field to have a different external name.

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    username: str = Field(alias="userName")
```

Now you can validate:

```python
user = User(userName="Nzegge")

print(user.username)
```

Output:

```text
Nzegge
```

### validation_alias

Controls the name used when reading input.

```python
class User(BaseModel):
    username: str = Field(validation_alias="userName")
```

### serialization_alias

Controls the name used when serializing output.

```python
class User(BaseModel):
    username: str = Field(serialization_alias="userName")
```

Serialize using aliases:

```python
user.model_dump(by_alias=True)
```

### Alias generator

Useful when converting between naming conventions.

```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

class User(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        validate_by_name=True,
        validate_by_alias=True
    )

    first_name: str
    last_name: str
```

This generates aliases such as `firstName` and `lastName`.

---

# 10. Serialization and Deserialization

These two concepts are important when working with APIs.

### Deserialization

Converting incoming data into Python objects.

```python
data = {
    "name": "Nzegge",
    "age": 25
}

user = User.model_validate(data)
```

### Serialization

Converting Python objects into data that can be transmitted or stored.

```python
data = user.model_dump()

json_data = user.model_dump_json()
```

Remember:

- `model_validate()` validates incoming Python data.
- `model_validate_json()` validates incoming JSON.
- `model_dump()` produces Python data.
- `model_dump_json()` produces JSON text.

---

# 11. Pydantic in FastAPI

This is where Pydantic becomes especially important.

FastAPI uses Pydantic models to validate request bodies and document API schemas.

## 11.1 Request Body

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()

class UserCreate(BaseModel):
    username: str = Field(min_length=3)
    email: str
    age: int = Field(gt=0)

@app.post("/users/")
async def create_user(user: UserCreate):
    return {
        "message": "User created successfully",
        "user": user
    }
```

When a client sends:

```json
{
    "username": "Nzegge",
    "email": "nzegge@example.com",
    "age": 25
}
```

FastAPI:

1. Receives the HTTP request.
2. Reads the JSON request body.
3. Uses Pydantic to validate the data.
4. Creates a `UserCreate` instance.
5. Passes that instance to your endpoint.
6. Serializes the returned data into an HTTP response.

If validation fails, FastAPI normally returns HTTP `422 Unprocessable Entity`.

## 11.2 Response Model

You should not always return every field in your database model.

Use a response model to control what clients receive.

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: str

@app.post("/users/", response_model=UserResponse)
async def create_user(user: UserCreate):
    return {
        "id": 1,
        "username": user.username,
        "email": user.email,
        "password": user.password
    }
```

FastAPI validates and serializes the response according to `UserResponse`.

The password is not included in the response.

**Important:** Response models are not a replacement for secure password storage. Passwords should be hashed before storage, never stored as plaintext.

---

# 12. Different Models for Different Operations

A good API commonly uses separate models for creating, updating, and returning resources.

```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    username: str = Field(min_length=3)
    email: EmailStr
    password: str = Field(min_length=8)

class UserUpdate(BaseModel):
    username: str | None = None
    email: EmailStr | None = None

class UserResponse(BaseModel):
    id: int
    username: str
    email: EmailStr
```

### Why separate them?

| Model | Purpose |
|---|---|
| `UserCreate` | Validates data used to create a user |
| `UserUpdate` | Validates data used to update a user |
| `UserResponse` | Defines the information returned to clients |

This is similar to the idea of using separate serializers for different operations in Django REST Framework.

---

# 13. Partial Updates with PATCH

For a PATCH endpoint, clients may provide only the fields they want to update.

```python
class UserUpdate(BaseModel):
    username: str | None = None
    email: EmailStr | None = None
```

Example:

```python
@app.patch("/users/{user_id}")
async def update_user(user_id: int, user: UserUpdate):
    update_data = user.model_dump(exclude_unset=True)

    return {
        "user_id": user_id,
        "updated_fields": update_data
    }
```

Why use `exclude_unset=True`?

Suppose the client sends:

```json
{
    "username": "NewName"
}
```

Then:

```python
user.model_dump()
```

May return:

```python
{
    "username": "NewName",
    "email": None
}
```

But:

```python
user.model_dump(exclude_unset=True)
```

Returns:

```python
{
    "username": "NewName"
}
```

This prevents an omitted field from accidentally being treated as an instruction to clear its existing value.

---

# 14. Important Pydantic Special Types

Pydantic provides additional types for specific validation requirements.

| Type | Purpose |
|---|---|
| `EmailStr` | Email validation |
| `HttpUrl` | HTTP/HTTPS URL validation |
| `AnyUrl` | General URL validation |
| `IPvAnyAddress` | IPv4 or IPv6 address |
| `IPvAnyNetwork` | IPv4 or IPv6 network |
| `SecretStr` | Secret string representation |
| `SecretBytes` | Secret bytes representation |
| `FilePath` | Existing file path |
| `DirectoryPath` | Existing directory path |
| `PastDate` | Date in the past |
| `FutureDate` | Date in the future |
| `PositiveInt` | Positive integer |
| `NegativeInt` | Negative integer |
| `NonNegativeInt` | Zero or positive integer |
| `PositiveFloat` | Positive float |
| `NonNegativeFloat` | Zero or positive float |
| `StrictInt` | Strict integer |
| `StrictStr` | Strict string |
| `StrictBool` | Strict boolean |

Example:

```python
from pydantic import BaseModel, EmailStr, PositiveInt, SecretStr

class User(BaseModel):
    email: EmailStr
    age: PositiveInt
    password: SecretStr
```

`SecretStr` helps avoid accidentally exposing the secret in ordinary representations. It does not hash or encrypt the password.

---

# 15. TypeAdapter

`TypeAdapter` validates types that are not necessarily represented by a `BaseModel`.

```python
from pydantic import TypeAdapter

adapter = TypeAdapter(list[int])

numbers = adapter.validate_python([1, 2, 3])

print(numbers)
```

Invalid example:

```python
adapter.validate_python([1, "hello", 3])
```

This raises a validation error.

Other examples:

```python
TypeAdapter(list[str])
TypeAdapter(dict[str, int])
TypeAdapter(int | str)
```

Useful methods include:

- `validate_python()`
- `validate_json()`
- `validate_strings()`
- `dump_python()`
- `dump_json()`
- `json_schema()`

Use `TypeAdapter` when you need Pydantic validation without defining a complete model class.

---

# 16. Generic Models

Generic models allow you to reuse a model with different data types.

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class Response(BaseModel, Generic[T]):
    success: bool
    data: T
```

Example:

```python
class User(BaseModel):
    username: str

user_response = Response[User](
    success=True,
    data={"username": "Nzegge"}
)
```

You can also use:

```python
class Product(BaseModel):
    name: str
    price: float

product_response = Response[Product](
    success=True,
    data={
        "name": "Laptop",
        "price": 500
    }
)
```

This is useful when building reusable API response structures.

---

# 17. Common Pydantic Errors

### Error 1: Missing required field

```python
class User(BaseModel):
    username: str
    age: int

user = User(username="Nzegge")
```

Problem: `age` is required.

Solution:

```python
user = User(username="Nzegge", age=25)
```

### Error 2: Incorrect type

```python
user = User(username="Nzegge", age="hello")
```

Problem: `"hello"` cannot be validated as an integer.

### Error 3: Incorrect field constraint

```python
class User(BaseModel):
    age: int = Field(gt=18)
```

This rejects values less than or equal to 18.

### Error 4: Unexpected fields

```python
class User(BaseModel):
    model_config = ConfigDict(extra="forbid")

    username: str
```

This rejects additional fields not declared in the model.

### Error 5: Confusing optional with nullable

```python
age: int | None
```

This allows `None`, but the field is still required.

```python
age: int | None = None
```

This makes the field optional.

---

# 18. Recommended Project Structure

For a FastAPI application:

```text
my_project/
│
├── src/
│   └── my_project/
│       ├── __init__.py
│       ├── main.py
│       │
│       ├── schemas/
│       │   ├── __init__.py
│       │   ├── user.py
│       │   ├── product.py
│       │   └── order.py
│       │
│       ├── models/
│       │   ├── __init__.py
│       │   ├── user.py
│       │   ├── product.py
│       │   └── order.py
│       │
│       ├── routers/
│       ├── services/
│       ├── database/
│       └── core/
│
├── tests/
├── pyproject.toml
└── README.md
```

### Important distinction: schemas vs models

In a typical FastAPI + SQLAlchemy project:

- `schemas/` contains Pydantic models used for validation and serialization.
- `models/` contains SQLAlchemy classes representing database tables.

Example:

**schemas/user.py**

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: EmailStr
```

**models/user.py**

```python
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import String
from my_project.database.connection import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)
```

They are not the same thing.

Pydantic validates application data; SQLAlchemy maps Python objects to database tables.

---

# 19. Best Practices

1. Use Pydantic models to validate incoming request data.
2. Use response models to control outgoing data.
3. Separate create, update, and response schemas.
4. Use `Field()` for field-level constraints.
5. Use `field_validator` for custom field validation.
6. Use `model_validator` for validation involving multiple fields.
7. Use `ConfigDict(from_attributes=True)` when validating ORM objects by attributes.
8. Use `EmailStr` for email validation.
9. Use `exclude_unset=True` for partial updates.
10. Never return passwords or sensitive fields in API responses.
11. Do not confuse Pydantic models with SQLAlchemy database models.
12. Prefer Pydantic V2 APIs such as `model_dump()` and `model_validate()` over older V1 methods such as `.dict()` and `.parse_obj()`.

---

# 20. Quick Reference Cheat Sheet

| Requirement | Pydantic API |
|---|---|
| Define a model | `BaseModel` |
| Configure a model | `ConfigDict` |
| Configure a field | `Field()` |
| Validate a dictionary | `model_validate()` |
| Validate JSON | `model_validate_json()` |
| Convert model to dictionary | `model_dump()` |
| Convert model to JSON | `model_dump_json()` |
| Generate JSON Schema | `model_json_schema()` |
| Copy a model | `model_copy()` |
| Validate one field | `field_validator` |
| Validate multiple fields | `model_validator` |
| Validate arbitrary types | `TypeAdapter` |
| Validate email | `EmailStr` |
| Validate URL | `HttpUrl` |
| Validate UUID | `UUID` |
| Create reusable generic models | `Generic` |
| Read object attributes | `from_attributes=True` |

---

# 21. Official Documentation

- Pydantic: https://docs.pydantic.dev/latest/
- Models: https://docs.pydantic.dev/latest/concepts/models/
- Fields: https://docs.pydantic.dev/latest/concepts/fields/
- Validators: https://docs.pydantic.dev/latest/concepts/validators/
- Serialization: https://docs.pydantic.dev/latest/concepts/serialization/
- Configuration: https://docs.pydantic.dev/latest/concepts/config/
- Types: https://docs.pydantic.dev/latest/concepts/types/
- FastAPI integration: https://fastapi.tiangolo.com/tutorial/body/

---

# END OF PYDANTIC REFERENCE
