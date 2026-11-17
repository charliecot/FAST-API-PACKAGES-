# Python Typing, Ty, FastAPI, Pydantic and Databases

## A practical zero-to-hero reference guide

**Updated:** 17 November 2026\
**Focus:** Modern Python typing, static type checking with Astral Ty,
Pydantic V2, FastAPI, SQLAlchemy 2.x, database boundaries, and reliable
application design.

> This guide teaches the concepts and the reasoning behind them. It uses
> modern Python syntax where appropriate. Check `requires-python` in
> your own project before adopting syntax that requires a newer Python
> release.

------------------------------------------------------------------------

# Table of contents

1.  The big picture: runtime, validation and static typing
2.  Python annotations from zero
3.  Built-in types and collection annotations
4.  Optional values, unions and `None`
5.  Functions, parameters and return types
6.  Variables, inference and narrowing
7.  `Any`, `object`, `Never`, `NoReturn` and `Unknown`
8.  Dictionaries, `TypedDict` and structured data
9.  Type aliases, `NewType` and domain-specific identifiers
10. `Literal`, `Enum` and constrained values
11. `Annotated` and metadata
12. `TypeVar`, generics and reusable containers
13. `Protocol`, structural typing and dependency design
14. `Callable`, `ParamSpec` and callback types
15. `TypeGuard`, `TypeIs` and safe narrowing
16. `cast`: what it does and does not do
17. Dataclasses, Pydantic models and ORM models
18. Pydantic V2 models from beginner to advanced
19. Pydantic validators and custom types
20. FastAPI: how annotations control requests and responses
21. FastAPI query, path, body, headers, cookies and dependencies
22. Database typing with SQLAlchemy 2.x
23. The correct separation: API schema, domain object and database row
24. Full FastAPI + SQLAlchemy example
25. Ty: setup, configuration and daily workflow
26. Understanding common Ty errors
27. Common mistakes and better alternatives
28. Testing types, validation and database behavior
29. A practical project structure
30. A professional development workflow
31. Exercises and a learning roadmap
32. Quick-reference tables
33. Official documentation

------------------------------------------------------------------------

# 1. The big picture: three different systems

One of the most important things to understand is that Python
applications can involve three different kinds of checking.

## 1.1 Python runtime

The Python interpreter executes your program.

``` python
age = 25
age = "twenty-five"  # Python permits this reassignment
```

Python is dynamically typed. A variable name is not permanently locked
to one type.

Python generally does not enforce ordinary annotations when the program
runs.

``` python
def add(a: int, b: int) -> int:
    return a + b

print(add("hello", "world"))  # Runtime result: "helloworld"
```

The annotation says what the function is intended to accept and return.
It does not, by itself, reject the strings.

## 1.2 Static type checker

A static type checker analyzes your source code without needing to
execute every possible code path.

Examples include:

-   Ty
-   mypy
-   Pyright

For example:

``` python
def add(a: int, b: int) -> int:
    return a + b

add("hello", "world")  # Type checker should report an error
```

The program may execute, but the call violates the declared contract.

A type checker helps detect mistakes such as:

-   Passing a string where an integer is expected.
-   Returning the wrong type.
-   Accessing a dictionary key that is not declared.
-   Appending an incompatible value to a typed list.
-   Calling a method that does not exist on a value.
-   Forgetting that a value may be `None`.

## 1.3 Runtime validation

A validation library checks actual data while the application runs.

Pydantic is a common example:

``` python
from pydantic import BaseModel

class UserCreate(BaseModel):
    username: str
    age: int

user = UserCreate(username="nzegge", age=25)
```

Pydantic validates the supplied data and creates a model instance.

``` python
UserCreate(username="nzegge", age="not-a-number")
```

This raises a validation error because the input cannot be converted to
the required integer.

## 1.4 Keep the three systems separate

  -----------------------------------------------------------------------
  System                  Main question           Example
  ----------------------- ----------------------- -----------------------
  Python runtime          Can this code execute?  `list.append(value)`

  Static checker          Does the code agree     `list[BandData]`
                          with its declared       
                          types?                  

  Runtime validator       Does incoming data      Pydantic `BaseModel`
                          satisfy the model?      
  -----------------------------------------------------------------------

**Important:** A type annotation is not automatically runtime
validation. A Pydantic model is not automatically a static type checker.
A static type checker does not execute your application to prove that
every external input is valid.

A robust application uses all three appropriately.

------------------------------------------------------------------------

# 2. Python annotations from zero

## 2.1 Variable annotations

``` python
name: str = "Nzegge"
age: int = 25
price: float = 19.99
is_active: bool = True
```

The syntax is:

``` python
variable_name: expected_type = value
```

You can also annotate without assigning immediately:

``` python
username: str
```

This declares the intended type to static analysis tools. It does not
assign a value.

Do not read the variable before assigning it.

## 2.2 Function parameter annotations

``` python
def greet(name: str):
    print(f"Hello, {name}")
```

The parameter `name` is expected to be a string.

## 2.3 Return annotations

``` python
def get_age() -> int:
    return 25
```

The arrow means the function is expected to return an integer.

A function that intentionally returns nothing should normally use
`None`:

``` python
def save_log(message: str) -> None:
    print(message)
```

Do not confuse `None` with the absence of a return annotation.

``` python
def first():
    return 10  # No return annotation

def second() -> None:
    print("No meaningful return value")
```

## 2.4 Annotate public functions

A useful habit is to annotate:

-   Functions used by other modules.
-   Service-layer functions.
-   Database repository functions.
-   FastAPI route functions.
-   Validators.
-   Utility functions with non-obvious inputs or outputs.

You do not have to annotate every temporary local variable when the type
is obvious.

------------------------------------------------------------------------

# 3. Built-in types and collection annotations

Modern built-in generic syntax is usually preferred.

## 3.1 Scalar types

``` python
name: str = "Computer"
count: int = 10
rating: float = 4.5
enabled: bool = True
nothing: None = None
```

Remember that `bool` is a subclass of `int` in Python. Avoid relying on
that relationship for business rules.

## 3.2 Lists

``` python
names: list[str] = ["Ada", "Grace"]
scores: list[int] = [10, 20, 30]
```

A list annotation describes the type of its elements.

``` python
scores: list[int] = [10, 20]
scores.append(30)       # Correct
scores.append("thirty") # Type error
```

The list remains an ordinary Python list at runtime. The annotation
guides static analysis.

## 3.3 Dictionaries

``` python
prices: dict[str, float] = {
    "keyboard": 25.50,
    "mouse": 12.00,
}
```

The two generic parameters are:

``` python
dict[KEY_TYPE, VALUE_TYPE]
```

For example:

``` python
user_ids: dict[str, int] = {
    "nzegge": 1,
    "ada": 2,
}
```

## 3.4 Tuples

A fixed-size tuple can describe each position:

``` python
coordinate: tuple[float, float] = (4.5, 8.2)
```

A variable-length tuple uses an ellipsis:

``` python
scores: tuple[int, ...] = (10, 20, 30, 40)
```

A tuple with different element types:

``` python
record: tuple[int, str, bool] = (1, "Nzegge", True)
```

## 3.5 Sets

``` python
unique_names: set[str] = {"Ada", "Grace"}
```

An empty set is created with `set()`, not `{}`:

``` python
names: set[str] = set()
```

`{}` creates an empty dictionary.

## 3.6 Iterable, Sequence and Collection

Sometimes a function should accept more than a list.

``` python
from collections.abc import Iterable, Sequence

def print_names(names: Iterable[str]) -> None:
    for name in names:
        print(name)

def first_name(names: Sequence[str]) -> str:
    return names[0]
```

Use:

-   `list[T]` when the function specifically needs a mutable list.
-   `Sequence[T]` when it needs ordered access and indexing.
-   `Iterable[T]` when it only needs to loop.
-   `Collection[T]` when it needs iteration, length and membership.
-   `Mapping[K, V]` when it needs dictionary-like read access.

Prefer the least restrictive interface that satisfies the function.

------------------------------------------------------------------------

# 4. Optional values, unions and None

## 4.1 A value that may be absent

``` python
description: str | None = None
```

This means the value can be either a string or `None`.

Equivalent older syntax:

``` python
from typing import Optional

description: Optional[str] = None
```

`str | None` is generally clearer in modern Python.

## 4.2 Optional does not mean a parameter has a default

These are different:

``` python
def create_user(nickname: str | None) -> None:
    ...
```

The caller must provide an argument, but it may be `None`.

``` python
def create_user(nickname: str | None = None) -> None:
    ...
```

The caller may omit the argument.

## 4.3 Narrow before using a possibly absent value

Incorrect:

``` python
def show_name(name: str | None) -> None:
    print(name.upper())
```

The name may be `None`.

Correct:

``` python
def show_name(name: str | None) -> None:
    if name is None:
        print("No name supplied")
        return

    print(name.upper())
```

After the `None` check, a type checker can narrow `name` to `str`.

## 4.4 Union types

``` python
identifier: int | str
```

The value can be an integer or string.

Use a type check before performing type-specific operations:

``` python
def normalize_identifier(value: int | str) -> str:
    if isinstance(value, int):
        return str(value)

    return value.strip()
```

Do not assume that every member of a union supports the same methods.

------------------------------------------------------------------------

# 5. Functions, parameters and return types

## 5.1 Basic function contract

``` python
def calculate_total(price: float, quantity: int) -> float:
    return price * quantity
```

A function annotation documents the contract between the caller and the
function.

## 5.2 Default arguments

``` python
def list_bands(limit: int = 20) -> list[str]:
    return []
```

Avoid mutable defaults:

``` python
# Avoid
def add_item(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items
```

The same list can be reused across calls.

Use `None`:

``` python
def add_item(
    item: str,
    items: list[str] | None = None,
) -> list[str]:
    if items is None:
        items = []

    items.append(item)
    return items
```

## 5.3 Positional-only and keyword-only parameters

``` python
def connect(host: str, /, *, port: int = 5432) -> None:
    ...
```

-   Parameters before `/` are positional-only.
-   Parameters after `*` are keyword-only.

This can make APIs easier to use correctly.

## 5.4 `*args` and `**kwargs`

``` python
def total(*values: int) -> int:
    return sum(values)
```

`values` is treated as a tuple of integers.

``` python
def show_settings(**settings: str) -> None:
    for key, value in settings.items():
        print(key, value)
```

`settings` is treated as a dictionary of string keys and string values.

When keyword arguments have a known structure, use `TypedDict` and
`Unpack` rather than allowing arbitrary dictionaries.

## 5.5 Async functions

``` python
async def fetch_name(user_id: int) -> str:
    return "Nzegge"
```

The function is an asynchronous function. Calling it produces a
coroutine that must normally be awaited.

``` python
name = await fetch_name(1)
```

For an async function, annotate the value it returns when awaited, not
`Coroutine[...]` in the ordinary FastAPI route case.

------------------------------------------------------------------------

# 6. Variables, inference and narrowing

## 6.1 Type inference

A type checker can infer obvious types:

``` python
name = "Nzegge"  # inferred as str
age = 25         # inferred as int
```

You do not need to repeat every obvious annotation.

## 6.2 Inference can become too broad

``` python
items = []
```

The checker may not know what should be stored in this list.

Prefer:

``` python
items: list[str] = []
```

This is particularly useful for empty lists, empty dictionaries and
variables initialized to `None`.

``` python
selected_user: User | None = None
```

## 6.3 Type narrowing

Type narrowing means making a broad type more specific after checking
it.

``` python
def describe(value: int | str) -> str:
    if isinstance(value, str):
        return value.upper()

    return str(value)
```

Inside the first branch, `value` is a string. In the second branch, it
is an integer.

Other narrowing techniques include:

``` python
if value is None:
    ...

if isinstance(value, SomeClass):
    ...

if isinstance(value, str):
    ...

if "name" in data:
    ...
```

A good type checker follows these conditions through the code.

------------------------------------------------------------------------

# 7. Any, object, Never, NoReturn and Unknown

These types are important. They are not interchangeable.

## 7.1 `Any`

``` python
from typing import Any

payload: Any = {"name": "Computer"}
```

`Any` effectively opts out of many static checks.

``` python
payload.some_method()
payload["missing"]
payload + 10
```

A checker may allow these operations because the value is `Any`, even
though they may fail at runtime.

Use `Any` at genuinely dynamic boundaries, such as poorly typed
third-party libraries or raw deserialized data that you have not yet
validated.

Do not use `Any` everywhere just to silence errors.

## 7.2 `object`

``` python
value: object = "hello"
```

`object` means any Python object, but it does not permit arbitrary
operations.

``` python
def display(value: object) -> str:
    if isinstance(value, str):
        return value.upper()

    return str(value)
```

The checker requires you to prove that the operation is valid.

**Rule of thumb:** Prefer `object` when you do not know the type and
want to retain type safety. Use `Any` only when you intentionally accept
unchecked dynamic behavior.

## 7.3 `Never`

`Never` represents a type that has no possible values. It is useful for
functions that cannot return normally.

``` python
from typing import Never

def stop_program(message: str) -> Never:
    raise RuntimeError(message)
```

## 7.4 `NoReturn`

`NoReturn` is the older spelling commonly used for functions that never
return normally.

``` python
from typing import NoReturn

def fail() -> NoReturn:
    raise RuntimeError("Failed")
```

Use `Never` when expressing an impossible type or unreachable branch in
modern typing. `NoReturn` remains widely recognized for non-returning
functions.

## 7.5 `Unknown` in Ty diagnostics

Ty can display `Unknown` when it cannot determine a useful type.

This is not the same as a declared `Any`:

-   `Any` is an explicit dynamic escape hatch.
-   `Unknown` means the checker lacks enough information.

Improve the source of the unknown type: annotate a function, type an
empty collection, improve a dependency stub, or validate an external
value.

------------------------------------------------------------------------

# 8. Dictionaries, TypedDict and structured data

This topic is especially important for FastAPI applications that store
dictionaries or receive JSON.

## 8.1 A normal dictionary

``` python
band = {
    "id": 1,
    "name": "Computer",
    "genre": "Alternative",
}
```

A normal dictionary does not declare a fixed schema.

A type checker may infer a broad type such as `dict[str, object]` or
another inferred dictionary type depending on context.

## 8.2 TypedDict

Use `TypedDict` when a dictionary has a known shape.

``` python
from typing import TypedDict

class BandData(TypedDict):
    id: int
    name: str
    genre: str
    album: list[str]
```

Now declare:

``` python
BANDS: list[BandData] = []
```

This means every item in `BANDS` must be a dictionary matching the
`BandData` structure.

Correct:

``` python
band: BandData = {
    "id": 1,
    "name": "Computer",
    "genre": "Alternative",
    "album": [],
}

BANDS.append(band)
```

Incorrect:

``` python
BANDS.append({
    "id": "one",
    "name": "Computer",
    "genre": "Alternative",
    "album": [],
})
```

The `id` value is a string, but `BandData` requires an integer.

## 8.3 TypedDict is not a Pydantic model

This distinction is essential.

``` python
class BandData(TypedDict):
    id: int
    name: str
```

At runtime, a `BandData` value is an ordinary dictionary.

It does not provide Pydantic-style runtime validation.

``` python
band: BandData = {
    "id": 1,
    "name": "Computer",
}
```

The annotation helps static analysis. It does not create a special
object.

## 8.4 Optional keys with NotRequired

A key that may be absent is different from a key whose value may be
`None`.

``` python
from typing import NotRequired, TypedDict

class BandData(TypedDict):
    id: int
    name: str
    description: NotRequired[str]
```

The `description` key may be omitted.

A key that must exist but may contain `None`:

``` python
class BandData(TypedDict):
    id: int
    description: str | None
```

These are different contracts.

## 8.5 Required and NotRequired

``` python
from typing import NotRequired, Required, TypedDict

class UserData(TypedDict, total=False):
    username: Required[str]
    email: Required[str]
    biography: NotRequired[str]
```

`total=False` makes keys non-required by default. `Required` marks keys
that must still exist.

## 8.6 Safe dictionary access

For a normal dictionary:

``` python
value = data.get("name")
```

The result may be absent. Type it accordingly.

For a `TypedDict`, use declared keys:

``` python
name = band["name"]
```

A required key is known to exist according to the TypedDict contract.

Do not use `# type: ignore` to hide a mismatch that should be fixed in
the data model.

## 8.7 When to use TypedDict

Good uses:

-   In-memory dictionaries with a fixed shape.
-   Structured JSON payloads after you have validated them.
-   Internal records that should remain dictionaries.
-   Function keyword argument specifications.

Avoid using it as a replacement for a Pydantic model when untrusted
incoming data needs runtime validation.

------------------------------------------------------------------------

# 9. Type aliases, NewType and domain-specific identifiers

## 9.1 Type alias

A type alias gives a name to an existing type.

Modern syntax:

``` python
type UserName = str
type BandCollection = list[BandData]
```

Traditional syntax:

``` python
UserName = str
BandCollection = list[BandData]
```

A type alias does not create a new distinct type. It is another name for
the same type.

Use aliases to simplify long annotations and communicate meaning.

## 9.2 NewType

Sometimes two values have the same underlying type but must not be
confused.

``` python
from typing import NewType

UserId = NewType("UserId", int)
BandId = NewType("BandId", int)
```

Then:

``` python
def get_user(user_id: UserId) -> None:
    ...

def get_band(band_id: BandId) -> None:
    ...
```

A type checker can distinguish `UserId` from `BandId`.

``` python
user_id = UserId(10)
band_id = BandId(20)

get_user(user_id)  # Correct
```

A `NewType` does not create a runtime wrapper or validate the value.
Calling it returns the underlying value.

Use `NewType` for logical distinctions, not for runtime validation.

------------------------------------------------------------------------

# 10. Literal, Enum and constrained values

## 10.1 Literal

Use `Literal` when a value must be one of a small, known set.

``` python
from typing import Literal

Status = Literal["pending", "paid", "cancelled"]

def update_status(status: Status) -> None:
    ...
```

A checker can reject an unsupported literal.

## 10.2 Enum

Use an enum when you want named constants and a reusable runtime type.

``` python
from enum import StrEnum

class Genre(StrEnum):
    ROCK = "Rock"
    JAZZ = "Jazz"
    ALTERNATIVE = "Alternative"
```

Then:

``` python
def filter_genre(genre: Genre) -> None:
    ...
```

`StrEnum` requires Python 3.11+. For older versions, use
`class Genre(str, Enum)`.

## 10.3 Literal or Enum?

Use `Literal` for a small set of values in a narrow function contract.

Use `Enum` when the choices are a domain concept used throughout the
application, especially when they need runtime behavior, methods or
serialization.

## 10.4 Do not confuse static constraints with runtime constraints

This annotation:

``` python
age: int
```

does not express that age must be positive.

Use Pydantic field constraints for runtime validation:

``` python
from pydantic import BaseModel, Field

class UserCreate(BaseModel):
    age: int = Field(gt=0, le=120)
```

------------------------------------------------------------------------

# 11. Annotated and metadata

`Annotated` attaches metadata to a type.

``` python
from typing import Annotated
from pydantic import Field

PositiveAge = Annotated[int, Field(gt=0, le=120)]
```

Then:

``` python
class UserCreate(BaseModel):
    age: PositiveAge
```

FastAPI also uses `Annotated` to attach request metadata:

``` python
from typing import Annotated
from fastapi import Query

async def list_bands(
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> list[BandData]:
    ...
```

The underlying type is still `int`; the metadata adds additional meaning
for frameworks that understand it.

Do not assume every Python tool interprets every `Annotated` metadata
item. The consuming library determines what the metadata means.

------------------------------------------------------------------------

# 12. TypeVar, generics and reusable containers

Generics allow a class or function to work with different types while
preserving the relationship between input and output.

## 12.1 The problem without generics

``` python
def first_string(values: list[str]) -> str:
    return values[0]

def first_integer(values: list[int]) -> int:
    return values[0]
```

The logic is duplicated.

## 12.2 Generic function with modern syntax

For Python 3.12 and later:

``` python
def first_item[T](values: list[T]) -> T:
    return values[0]
```

The same type `T` connects the list element type to the return type.

``` python
name = first_item(["Ada", "Grace"])  # str
number = first_item([10, 20, 30])    # int
```

## 12.3 TypeVar syntax for older supported versions

``` python
from typing import TypeVar

T = TypeVar("T")

def first_item(values: list[T]) -> T:
    return values[0]
```

## 12.4 Generic class

Python 3.12+:

``` python
class Box[T]:
    def __init__(self, value: T) -> None:
        self.value = value

    def get(self) -> T:
        return self.value
```

Older syntax:

``` python
from typing import Generic, TypeVar

T = TypeVar("T")

class Box(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value

    def get(self) -> T:
        return self.value
```

## 12.5 Bounded TypeVar

A bound restricts the type to a base type or its subclasses.

``` python
from typing import TypeVar

Number = TypeVar("Number", bound=int)

def double(value: Number) -> Number:
    return value * 2
```

Use bounds when an operation requires capabilities supplied by a
particular base type.

## 12.6 Constrained TypeVar

A constrained type variable limits a type to specific alternatives:

``` python
TextOrBytes = TypeVar("TextOrBytes", str, bytes)

def duplicate(value: TextOrBytes) -> TextOrBytes:
    return value + value
```

A bound and constraints are not the same:

-   A bound allows subclasses of the bound.
-   Constraints choose from the listed alternatives.

## 12.7 Generic API response

A reusable response envelope can be typed:

``` python
class ApiResult[T](BaseModel):
    data: T
    message: str
```

For example:

``` python
band_result: ApiResult[BandWithID]
```

This illustrates generic typing. Confirm the exact generic model
behavior against the Pydantic version used by your project.

------------------------------------------------------------------------

# 13. Protocol, structural typing and dependency design

A `Protocol` describes the operations an object must support.

It is useful when you care about behavior rather than a particular
concrete class.

``` python
from typing import Protocol

class HasId(Protocol):
    id: int

def print_id(value: HasId) -> None:
    print(value.id)
```

Any compatible object with an integer `id` attribute can satisfy this
protocol for static typing.

## 13.1 Protocol with methods

``` python
class Repository[T](Protocol):
    def get_by_id(self, item_id: int) -> T | None:
        ...

    def list_all(self) -> list[T]:
        ...
```

A concrete repository does not need to inherit from `Repository` if it
structurally provides the required methods.

## 13.2 Why this is useful

You can write business logic against an interface instead of a specific
database implementation.

``` python
class BandService:
    def __init__(self, repository: Repository[BandData]) -> None:
        self.repository = repository
```

This makes it easier to replace a real database repository with an
in-memory fake during tests.

## 13.3 runtime_checkable

A protocol can be marked:

``` python
from typing import Protocol, runtime_checkable

@runtime_checkable
class HasName(Protocol):
    name: str
```

This permits limited runtime `isinstance` checks.

Important: runtime-checkable protocol checks primarily test attribute
presence, not full method signatures or value types. Do not treat them
as comprehensive runtime validation.

------------------------------------------------------------------------

# 14. Callable, ParamSpec and callback types

## 14.1 Callable

``` python
from collections.abc import Callable

def execute(operation: Callable[[int, int], int]) -> int:
    return operation(10, 20)
```

The callback accepts two integers and returns an integer.

## 14.2 Callable with no arguments

``` python
Callable[[], str]
```

## 14.3 Callable with arbitrary arguments

``` python
Callable[..., str]
```

The ellipsis means the argument signature is not specified. Avoid it
when you know the real signature.

## 14.4 ParamSpec

`ParamSpec` preserves a function's parameter signature, especially in
decorators.

``` python
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def log_call(func: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print("Calling function")
        return func(*args, **kwargs)

    return wrapper
```

This preserves the original function's parameters and return type for
static analysis.

------------------------------------------------------------------------

# 15. TypeGuard, TypeIs and safe narrowing

Sometimes you write a function that checks whether an object has a
particular type or structure.

## 15.1 TypeGuard

``` python
from typing import TypeGuard

def is_string_list(value: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(item, str) for item in value)
```

Then:

``` python
items: list[object] = ["a", "b"]

if is_string_list(items):
    # Type checker can treat items as list[str]
    print([item.upper() for item in items])
```

The function must genuinely check the condition. A false assertion can
make the program unsafe.

## 15.2 TypeIs

On supported Python versions, `TypeIs` can express a type predicate that
narrows both the true and false branches more precisely.

``` python
from typing import TypeIs

def is_integer(value: object) -> TypeIs[int]:
    return isinstance(value, int)
```

For projects supporting Python versions before `TypeIs` is available in
the standard library, use `typing_extensions`.

## 15.3 Type guards are not validators by themselves

A type guard is a promise to the static checker based on the boolean
result. It should contain real checking logic. For untrusted API data,
use Pydantic or another runtime validation mechanism.

------------------------------------------------------------------------

# 16. cast: what it does and does not do

This is one of the most important topics for understanding Ty errors.

## 16.1 Basic cast

``` python
from typing import cast

value: object = "hello"

text_value = cast(str, value)
```

`cast(str, value)` tells the type checker:

> Treat this expression as a string from this point onward.

## 16.2 Cast does not convert

``` python
number = cast(int, "123")

print(number)       # "123"
print(type(number)) # <class 'str'>
```

It remains a string.

To convert:

``` python
number = int("123")
```

## 16.3 Cast does not validate

``` python
value = cast(BandData, unknown_dictionary)
```

This does not verify that the dictionary contains the required keys or
correct values.

If the dictionary is malformed, the problem remains.

## 16.4 Your earlier FastAPI example

You had a situation similar to:

``` python
BANDS: list[BandData] = []

band = BandWithID(
    id=new_id,
    **band_data.model_dump(),
).model_dump()

BANDS.append(band)
```

Why can Ty complain?

-   `BANDS` expects `BandData`.
-   `model_dump()` is typed broadly as a dictionary.
-   The checker cannot necessarily prove that the returned dictionary
    has the exact `BandData` shape.

A cast can silence that particular static mismatch:

``` python
from typing import cast

band_data_dict = cast(
    BandData,
    BandWithID(
        id=new_id,
        **band_data.model_dump(),
    ).model_dump(),
)

BANDS.append(band_data_dict)
```

But this is safe only if the model and TypedDict truly have matching
fields and types.

## 16.5 Better option: validate and convert deliberately

If the dictionary is a runtime boundary, validate it:

``` python
validated_band = BandWithID.model_validate(raw_data)
```

If you need a dictionary for storage:

``` python
stored_data = validated_band.model_dump()
```

For a statically typed dictionary, you may still need a carefully
justified conversion or a helper function whose return type is
explicitly declared. Do not mistake `model_validate()` for a static type
checker.

## 16.6 When cast is appropriate

Use `cast` when:

1.  You have independently established the value's type.
2.  The checker cannot infer a fact that your program logic guarantees.
3.  The value comes from a trusted API whose runtime behavior is known.
4.  You are bridging a library whose type stubs are incomplete.

Avoid `cast` when:

-   You are guessing.
-   You are trying to convert a value.
-   You are trying to validate untrusted JSON.
-   You want to hide a genuine data-model mismatch.

**Professional rule:** Every cast should have a reason that you could
explain during a code review.

------------------------------------------------------------------------

# 17. Dataclasses, Pydantic models and ORM models

These three concepts solve different problems.

## 17.1 Dataclasses

Dataclasses are useful for ordinary Python data containers.

``` python
from dataclasses import dataclass

@dataclass
class Band:
    id: int
    name: str
    genre: str
```

Creating an instance:

``` python
band = Band(
    id=1,
    name="Computer",
    genre="Alternative",
)
```

Dataclasses generate methods such as `__init__` and `__repr__`.

They do not automatically perform Pydantic-style validation of the
values passed to the constructor.

## 17.2 Pydantic models

``` python
from pydantic import BaseModel

class BandCreate(BaseModel):
    name: str
    genre: str
```

Pydantic models validate data at runtime and support serialization and
JSON Schema generation.

## 17.3 ORM models

An ORM model represents database structure and database records.

With SQLAlchemy 2.x:

``` python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class BandORM(Base):
    __tablename__ = "bands"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    genre: Mapped[str]
```

This class maps to a database table.

## 17.4 Do not mix the responsibilities

  Model type             Main purpose
  ---------------------- --------------------------------------------
  Dataclass              Ordinary Python data structure
  Pydantic model         Runtime validation and serialization
  SQLAlchemy ORM model   Database mapping and persistence
  TypedDict              Static description of dictionary structure

A professional API often uses separate request schemas, response schemas
and ORM models.

------------------------------------------------------------------------

# 18. Pydantic V2 models from beginner to advanced

## 18.1 Basic model

``` python
from pydantic import BaseModel

class BandCreate(BaseModel):
    name: str
    genre: str
```

Create:

``` python
band = BandCreate(
    name="Computer",
    genre="Alternative",
)
```

Read values using attributes:

``` python
print(band.name)
print(band.genre)
```

A Pydantic model instance is not a dictionary.

Incorrect:

``` python
band["name"]
```

Correct:

``` python
band.name
```

Convert to a dictionary when required:

``` python
data = band.model_dump()
```

## 18.2 Required and optional fields

``` python
class BandCreate(BaseModel):
    name: str
    description: str | None = None
```

`name` is required. `description` may be omitted.

A nullable but required field:

``` python
class Example(BaseModel):
    description: str | None
```

This field must be supplied, but its value may be `None`.

## 18.3 Default factories

For mutable defaults, use `Field(default_factory=...)`.

``` python
from pydantic import BaseModel, Field

class BandCreate(BaseModel):
    name: str
    albums: list[str] = Field(default_factory=list)
```

Each model gets its own list.

## 18.4 Field constraints

``` python
from pydantic import BaseModel, Field

class BandCreate(BaseModel):
    name: str = Field(min_length=2, max_length=80)
    year_formed: int = Field(ge=1900, le=2100)
    rating: float = Field(ge=0, le=5)
```

Common constraints:

  Field option        Meaning
  ------------------- ---------------------------------------
  `gt`                Greater than
  `ge`                Greater than or equal
  `lt`                Less than
  `le`                Less than or equal
  `min_length`        Minimum string or collection length
  `max_length`        Maximum string or collection length
  `pattern`           Regular-expression pattern
  `default`           Default value
  `default_factory`   Callable that creates a default value

## 18.5 Field aliases

``` python
from pydantic import BaseModel, Field

class UserCreate(BaseModel):
    username: str = Field(alias="userName")
```

Aliases help when external JSON naming differs from Python naming.

Pydantic V2 also supports validation and serialization alias
configuration. Read the exact behavior for the installed Pydantic
version before configuring complex alias rules.

## 18.6 Extra fields

You can configure how unknown fields are handled:

``` python
from pydantic import BaseModel, ConfigDict

class BandCreate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    name: str
    genre: str
```

Common choices:

-   `ignore`: Ignore unknown fields (default behavior in common
    configurations).
-   `forbid`: Reject unknown fields.
-   `allow`: Preserve extra fields.

For many public write APIs, `extra="forbid"` is useful because it
catches misspelled or unexpected client fields.

## 18.7 model_dump

``` python
data = band.model_dump()
```

Important options include:

``` python
band.model_dump(
    exclude_unset=True,
    exclude_defaults=True,
    exclude_none=True,
)
```

-   `exclude_unset=True`: Exclude fields not explicitly set.
-   `exclude_defaults=True`: Exclude values equal to their defaults.
-   `exclude_none=True`: Exclude values that are `None`.

These options are not interchangeable.

## 18.8 model_dump_json

``` python
json_text = band.model_dump_json()
```

This returns JSON text, not a Python dictionary.

## 18.9 model_validate

``` python
band = BandCreate.model_validate(raw_data)
```

Validates a Python object against the model.

## 18.10 model_validate_json

``` python
band = BandCreate.model_validate_json(
    '{"name": "Computer", "genre": "Alternative"}'
)
```

Validates JSON input.

## 18.11 model_copy

``` python
updated = band.model_copy(update={"genre": "Rock"})
```

Important: `model_copy(update=...)` does not validate the update data in
the same way as constructing a new validated model. Do not use it as a
substitute for validating untrusted updates.

## 18.12 from_attributes

When validating objects with attributes, configure:

``` python
from pydantic import BaseModel, ConfigDict

class BandRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str
    genre: str
```

Then Pydantic can read attributes from an ORM instance:

``` python
response = BandRead.model_validate(orm_band)
```

This is useful when returning SQLAlchemy objects.

------------------------------------------------------------------------

# 19. Pydantic validators and custom types

## 19.1 Pydantic V2 field_validator

Use `@field_validator` instead of the deprecated V1 `@validator`.

``` python
from pydantic import BaseModel, field_validator

class BandCreate(BaseModel):
    name: str

    @field_validator("name")
    @classmethod
    def validate_name(cls, value: str) -> str:
        value = value.strip()

        if len(value) < 2:
            raise ValueError("Name must contain at least 2 characters")

        return value
```

## 19.2 Before validator

``` python
from typing import Any
from pydantic import BaseModel, field_validator

class BandCreate(BaseModel):
    genre: str

    @field_validator("genre", mode="before")
    @classmethod
    def title_case_genre(cls, value: Any) -> Any:
        return value.title() if isinstance(value, str) else value
```

`mode="before"` receives the raw input before normal field validation.

If the input is an integer:

1.  `isinstance(value, str)` is false.
2.  The validator returns the original integer.
3.  Pydantic performs normal string validation.
4.  Pydantic rejects the integer for a string field under its normal
    strictness rules.

The `else value` does not mean "accept any type". It means "leave the
value unchanged and let the next validation stage decide".

## 19.3 After validator

``` python
@field_validator("name", mode="after")
@classmethod
def normalize_name(cls, value: str) -> str:
    return value.strip().title()
```

An after validator receives a value that has already passed normal field
validation.

## 19.4 Model validator

Use `@model_validator` when validation depends on multiple fields.

``` python
from pydantic import BaseModel, model_validator

class DateRange(BaseModel):
    start: int
    end: int

    @model_validator(mode="after")
    def check_range(self) -> "DateRange":
        if self.end < self.start:
            raise ValueError("end must not be before start")

        return self
```

## 19.5 Before model validator

A before model validator receives the raw input, which may be a
dictionary or another object.

``` python
from typing import Any
from pydantic import BaseModel, model_validator

class UserCreate(BaseModel):
    username: str

    @model_validator(mode="before")
    @classmethod
    def normalize_input(cls, data: Any) -> Any:
        if isinstance(data, dict):
            data = data.copy()
            if isinstance(data.get("username"), str):
                data["username"] = data["username"].strip()

        return data
```

Do not assume a before model validator always receives a dictionary.

## 19.6 Avoid deprecated V1 validators

Old:

``` python
from pydantic import validator

@validator("name")
def validate_name(cls, value):
    return value.strip()
```

Modern:

``` python
from pydantic import field_validator

@field_validator("name")
@classmethod
def validate_name(cls, value: str) -> str:
    return value.strip()
```

For cross-field checks, use `@model_validator`.

## 19.7 Custom reusable types with Annotated

``` python
from typing import Annotated
from pydantic import Field

BandName = Annotated[str, Field(min_length=2, max_length=80)]
```

Then reuse:

``` python
class BandCreate(BaseModel):
    name: BandName
```

For many constraints, this is clearer than repeating validators in every
model.

## 19.8 Custom validation using AfterValidator

``` python
from typing import Annotated
from pydantic import AfterValidator

def normalize_genre(value: str) -> str:
    return value.strip().title()

GenreName = Annotated[str, AfterValidator(normalize_genre)]
```

Use:

``` python
class BandCreate(BaseModel):
    genre: GenreName
```

## 19.9 Custom validation using BeforeValidator

``` python
from typing import Annotated, Any
from pydantic import BeforeValidator

def strip_string(value: Any) -> Any:
    if isinstance(value, str):
        return value.strip()
    return value

CleanString = Annotated[str, BeforeValidator(strip_string)]
```

Pydantic still validates the result as a string.

## 19.10 Custom types beyond simple validators

Pydantic V2 supports custom core schemas through
`__get_pydantic_core_schema__`. This is an advanced extension point.

Do not begin with it. Prefer, in order:

1.  Built-in types and `Field`.
2.  `Annotated` with `AfterValidator` or `BeforeValidator`.
3.  `field_validator`.
4.  `model_validator`.
5.  A custom class and core-schema integration only when the simpler
    options are insufficient.

Custom core schemas are powerful but couple your type to Pydantic's
validation machinery. Keep them small and test them carefully.

------------------------------------------------------------------------

# 20. FastAPI: how annotations control requests and responses

FastAPI uses Python annotations and Pydantic to understand API data.

## 20.1 Path parameters

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/bands/{band_id}")
async def get_band(band_id: int) -> dict[str, int]:
    return {"id": band_id}
```

The path value arrives as text in the URL. FastAPI validates and
converts it to an integer.

A non-integer path value results in request validation failure.

## 20.2 Query parameters

``` python
@app.get("/bands")
async def list_bands(
    genre: str | None = None,
    has_albums: bool = False,
) -> list[BandRead]:
    ...
```

These are query parameters because they are simple parameters not
declared as path parameters or body models.

Example:

``` text
GET /bands?genre=Alternative&has_albums=true
```

## 20.3 Request body

``` python
class BandCreate(BaseModel):
    name: str
    genre: str

@app.post("/bands")
async def create_band(band: BandCreate) -> BandRead:
    ...
```

A Pydantic model parameter is interpreted as request body data.

## 20.4 Response model

``` python
@app.get("/bands", response_model=list[BandRead])
async def list_bands() -> list[BandRead]:
    ...
```

FastAPI uses the response model to validate, serialize, document and
filter output.

This is important for security. If your database model contains a
password hash, do not expose it through a public response schema.

## 20.5 Response model and Python return annotation

The return annotation is useful for static typing.

The decorator's `response_model` tells FastAPI what to validate and
expose.

Usually, make them agree:

``` python
@app.get("/bands", response_model=list[BandRead])
async def list_bands() -> list[BandRead]:
    ...
```

If you intentionally return a different internal object and let FastAPI
serialize it through a response model, type the function according to
what it actually returns. Use `response_model` for the API output
contract.

Do not casually annotate everything as `Any`; it removes useful static
guarantees.

------------------------------------------------------------------------

# 21. FastAPI query, path, body, headers, cookies and dependencies

Use `Annotated` to attach FastAPI metadata to a type.

## 21.1 Query

``` python
from typing import Annotated
from fastapi import Query

@app.get("/bands")
async def list_bands(
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> list[BandRead]:
    ...
```

## 21.2 Path

``` python
from fastapi import Path

@app.get("/bands/{band_id}")
async def get_band(
    band_id: Annotated[int, Path(gt=0)],
) -> BandRead:
    ...
```

## 21.3 Header

``` python
from fastapi import Header

@app.get("/client-info")
async def client_info(
    user_agent: Annotated[str | None, Header()] = None,
) -> dict[str, str | None]:
    return {"user_agent": user_agent}
```

## 21.4 Cookie

``` python
from fastapi import Cookie

@app.get("/session")
async def read_session(
    session_id: Annotated[str | None, Cookie()] = None,
) -> dict[str, str | None]:
    return {"session_id": session_id}
```

## 21.5 Body

``` python
from fastapi import Body

@app.put("/bands/{band_id}")
async def update_band(
    band_id: int,
    band: Annotated[BandCreate, Body()],
) -> dict[str, int]:
    return {"id": band_id}
```

A Pydantic model parameter is normally recognized as a body
automatically. `Body` is useful when you need to make the source
explicit or customize it.

## 21.6 Dependencies

Dependencies are reusable functions or classes that FastAPI resolves for
a request.

``` python
from typing import Annotated
from fastapi import Depends

def get_current_user() -> str:
    return "nzegge"

CurrentUser = Annotated[str, Depends(get_current_user)]

@app.get("/me")
async def read_me(user: CurrentUser) -> dict[str, str]:
    return {"username": user}
```

This creates a reusable dependency annotation.

## 21.7 Database session dependency

A common pattern is:

``` python
def get_session():
    with Session(engine) as session:
        yield session
```

Then:

``` python
SessionDep = Annotated[Session, Depends(get_session)]
```

The dependency lifecycle ensures that the session is managed for the
request. The exact transaction and commit strategy should be designed
deliberately.

------------------------------------------------------------------------

# 22. Database typing with SQLAlchemy 2.x

This section uses SQLAlchemy's modern typed ORM style.

## 22.1 Declarative base

``` python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

## 22.2 Mapped columns

``` python
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

class BandORM(Base):
    __tablename__ = "bands"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80))
    genre: Mapped[str] = mapped_column(String(80))
```

Understand the pieces:

-   `Mapped[int]`: SQLAlchemy-mapped attribute whose Python value is an
    integer.
-   `mapped_column(...)`: Defines database column configuration.
-   `primary_key=True`: Makes the column a primary key.
-   `String(80)`: Defines a database string column type.

## 22.3 Nullable columns

``` python
description: Mapped[str | None] = mapped_column(
    String(500),
    nullable=True,
)
```

The Python annotation and database nullability should agree.

## 22.4 Relationships

``` python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

class AlbumORM(Base):
    __tablename__ = "albums"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]

    band_id: Mapped[int] = mapped_column(
        ForeignKey("bands.id")
    )

    band: Mapped["BandORM"] = relationship(
        back_populates="albums"
    )
```

On the band:

``` python
class BandORM(Base):
    __tablename__ = "bands"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    genre: Mapped[str]

    albums: Mapped[list["AlbumORM"]] = relationship(
        back_populates="band"
    )
```

For optional one-to-one or many-to-one relationships, the annotation and
foreign-key nullability should reflect whether the relationship can be
absent.

## 22.5 Session typing

``` python
from sqlalchemy.orm import Session

def get_band(session: Session, band_id: int) -> BandORM | None:
    return session.get(BandORM, band_id)
```

`Session.get` may return `None`, so the return type must account for
that.

## 22.6 Select statements

``` python
from sqlalchemy import select
from sqlalchemy.orm import Session

def list_bands(session: Session) -> list[BandORM]:
    statement = select(BandORM)
    return list(session.scalars(statement).all())
```

For a single result:

``` python
def get_band_by_name(
    session: Session,
    name: str,
) -> BandORM | None:
    statement = select(BandORM).where(BandORM.name == name)
    return session.scalars(statement).first()
```

## 22.7 Add, commit and refresh

``` python
def create_band(
    session: Session,
    name: str,
    genre: str,
) -> BandORM:
    band = BandORM(name=name, genre=genre)

    session.add(band)
    session.commit()
    session.refresh(band)

    return band
```

-   `add`: Marks the object for persistence.
-   `commit`: Commits the transaction.
-   `refresh`: Reloads database-generated values, such as an
    auto-generated primary key.

In a larger application, decide whether transaction commits belong in
the service layer, repository, or request-level unit-of-work. Avoid
committing unpredictably in many unrelated helper functions.

## 22.8 SQLAlchemy model is not Pydantic model

Do not assume that a SQLAlchemy ORM instance supports:

``` python
orm_band.model_dump()
```

That method belongs to Pydantic models, not ordinary SQLAlchemy ORM
models.

Use a Pydantic response model with `from_attributes=True`:

``` python
class BandRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str
    genre: str
```

Then:

``` python
response = BandRead.model_validate(orm_band)
```

------------------------------------------------------------------------

# 23. The correct separation: API schema, domain object and database row

A reliable architecture keeps external input, application logic and
persistence separate.

## 23.1 Request schema

``` python
class BandCreate(BaseModel):
    name: str
    genre: str
```

This describes what a client may submit.

## 23.2 Response schema

``` python
class BandRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str
    genre: str
```

This describes what a client may receive.

## 23.3 Update schema

``` python
class BandUpdate(BaseModel):
    name: str | None = None
    genre: str | None = None
```

For PATCH semantics, use `model_dump(exclude_unset=True)` so that
omitted fields can be distinguished from explicitly supplied fields.

If `None` is not a valid update value, use a schema design that
distinguishes omission from explicit null, and validate accordingly.

## 23.4 Database model

``` python
class BandORM(Base):
    __tablename__ = "bands"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    genre: Mapped[str]
```

The database model describes persistence.

## 23.5 Why separate them?

Imagine your database contains:

``` text
id
name
genre
internal_notes
created_at
updated_at
```

The public API should not necessarily expose every database column.

A separate `BandRead` schema controls the public response.

This reduces accidental data exposure and makes database changes less
likely to break the API contract.

------------------------------------------------------------------------

# 24. Full FastAPI + SQLAlchemy example

This is a compact learning example. It uses SQLite to keep setup simple.
For production, use a managed database, migrations, proper secrets and
operational configuration.

Install:

``` bash
uv add fastapi "uvicorn[standard]" sqlalchemy pydantic
```

Create `main.py`:

``` python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Query
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import String, create_engine, select
from sqlalchemy.orm import (
    DeclarativeBase,
    Mapped,
    Session,
    mapped_column,
)

DATABASE_URL = "sqlite:///./bands.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},
)


class Base(DeclarativeBase):
    pass


class BandORM(Base):
    __tablename__ = "bands"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80))
    genre: Mapped[str] = mapped_column(String(80))


class BandCreate(BaseModel):
    name: str = Field(min_length=2, max_length=80)
    genre: str = Field(min_length=2, max_length=80)


class BandUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=2, max_length=80)
    genre: str | None = Field(default=None, min_length=2, max_length=80)


class BandRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str
    genre: str


def get_session():
    with Session(engine) as session:
        yield session


SessionDep = Annotated[Session, Depends(get_session)]

app = FastAPI(title="Bands API")


@app.on_event("startup")
def create_tables() -> None:
    Base.metadata.create_all(engine)


@app.post("/bands", response_model=BandRead, status_code=201)
def create_band(
    band_data: BandCreate,
    session: SessionDep,
) -> BandORM:
    band = BandORM(
        name=band_data.name,
        genre=band_data.genre,
    )

    session.add(band)
    session.commit()
    session.refresh(band)

    return band


@app.get("/bands", response_model=list[BandRead])
def list_bands(
    session: SessionDep,
    genre: str | None = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> list[BandORM]:
    statement = select(BandORM)

    if genre is not None:
        statement = statement.where(BandORM.genre == genre)

    statement = statement.limit(limit)

    return list(session.scalars(statement).all())


@app.get("/bands/{band_id}", response_model=BandRead)
def get_band(
    band_id: int,
    session: SessionDep,
) -> BandORM:
    band = session.get(BandORM, band_id)

    if band is None:
        raise HTTPException(status_code=404, detail="Band not found")

    return band


@app.patch("/bands/{band_id}", response_model=BandRead)
def update_band(
    band_id: int,
    band_data: BandUpdate,
    session: SessionDep,
) -> BandORM:
    band = session.get(BandORM, band_id)

    if band is None:
        raise HTTPException(status_code=404, detail="Band not found")

    changes = band_data.model_dump(exclude_unset=True)

    for field, value in changes.items():
        if value is not None:
            setattr(band, field, value)

    session.commit()
    session.refresh(band)

    return band


@app.delete("/bands/{band_id}", status_code=204)
def delete_band(
    band_id: int,
    session: SessionDep,
) -> None:
    band = session.get(BandORM, band_id)

    if band is None:
        raise HTTPException(status_code=404, detail="Band not found")

    session.delete(band)
    session.commit()
```

Run:

``` bash
uv run uvicorn main:app --reload
```

Open:

``` text
http://127.0.0.1:8000/docs
```

**Note:** `@app.on_event("startup")` is a simple teaching pattern and is
deprecated in favor of FastAPI lifespan handlers in modern applications.
For production, use a lifespan context manager and database migrations
such as Alembic rather than relying on `create_all` at startup.

------------------------------------------------------------------------

# 25. Ty: setup, configuration and daily workflow

Ty is a static type checker from Astral, the creators of uv and Ruff.

## 25.1 Install Ty

Using uv:

``` bash
uv add --dev ty
```

Or run without adding it to the project:

``` bash
uvx ty check
```

## 25.2 Run the checker

``` bash
uv run ty check
```

Check a specific file:

``` bash
uv run ty check main.py
```

Watch for changes:

``` bash
uv run ty check --watch
```

## 25.3 Make sure Ty sees the correct environment

A frequent cause of unresolved imports is that the checker is inspecting
a different Python environment from the one where the project
dependencies are installed.

Check:

``` bash
uv run python -c "import sys; print(sys.executable)"
uv run python -m pip show fastapi
uv run ty check
```

If you use a manually created virtual environment, activate it or
explicitly configure the interpreter.

## 25.4 pyproject.toml

A project can declare its Python compatibility:

``` toml
[project]
name = "bands-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "pydantic",
    "sqlalchemy",
]

[dependency-groups]
dev = [
    "ty",
    "pytest",
]
```

The minimum Python version should match the syntax and runtime features
you actually use.

## 25.5 Ty configuration

Ty supports `pyproject.toml` under `[tool.ty]` or a separate `ty.toml`.

Example:

``` toml
[tool.ty.environment]
python-version = "3.12"

[tool.ty.rules]
# Add project-specific rule settings only when you understand their effect.
```

Ty does not currently use a single mypy-style `--strict` switch. It has
its own rules and configuration.

## 25.6 A useful command sequence

``` bash
uv sync
uv run ty check
uv run pytest
uv run uvicorn main:app --reload
```

Run type checking and tests regularly, not only when the editor displays
red underlines.

------------------------------------------------------------------------

# 26. Understanding common Ty errors

## 26.1 Expected `BandData`, found `dict[str, Any]`

Example:

``` python
BANDS: list[BandData] = []

raw = model.model_dump()
BANDS.append(raw)
```

Why it happens:

-   `BANDS` requires a `BandData` TypedDict.
-   The static return type of `model_dump()` is a general dictionary.
-   Ty cannot prove that the dictionary has exactly the required
    structure.

Better solutions:

1.  Design a clear conversion function.
2.  Ensure the Pydantic model and TypedDict fields agree.
3.  Validate external data at runtime.
4.  Use a justified cast only when the shape is guaranteed.

Do not repeatedly cast mismatched data until the warnings disappear.

## 26.2 `list.append` argument mismatch

``` python
numbers: list[int] = []
numbers.append("hello")
```

Fix the value or the list declaration. `append` expects an integer
because the list is declared as `list[int]`.

## 26.3 Not subscriptable

``` python
band = BandRead(...)
print(band["name"])
```

A Pydantic model is accessed by attribute:

``` python
print(band.name)
```

A dictionary is accessed by key:

``` python
print(band_dict["name"])
```

Know which object you have.

## 26.4 Possibly None

``` python
band = session.get(BandORM, band_id)
print(band.name)
```

`session.get` can return `None`.

Fix:

``` python
band = session.get(BandORM, band_id)

if band is None:
    raise HTTPException(status_code=404, detail="Not found")

print(band.name)
```

## 26.5 Missing return

``` python
def find_band(band_id: int) -> BandORM:
    if band_id == 1:
        return ...
```

The function does not return a value on every path.

Either handle every path or declare the real return possibility:

``` python
def find_band(band_id: int) -> BandORM | None:
    ...
```

## 26.6 Incompatible return type

``` python
def get_count() -> int:
    return "five"
```

Return an integer or change the annotation if the function's real
contract is different.

## 26.7 Unknown imported module

Possible causes:

-   Wrong virtual environment.
-   Dependency not installed.
-   Wrong interpreter selected.
-   Missing type stubs for a third-party library.
-   Incorrect import path.

Check the environment before suppressing the import error.

## 26.8 Incorrect `# ty: ignore`

An ignore comment suppresses a diagnostic. It does not repair the code.

Use an ignore only when you have identified a genuine checker limitation
or an intentional incompatibility that cannot reasonably be expressed in
the type system.

Prefer a narrow, rule-specific suppression with an explanation,
following Ty's supported syntax and configuration.

------------------------------------------------------------------------

# 27. Common mistakes and better alternatives

  ---------------------------------------------------------------------------
  Avoid                   Prefer                      Reason
  ----------------------- --------------------------- -----------------------
  `Any` everywhere        Specific types or `object`  Keeps useful checking

  `cast()` to convert     `int()`, `str()`,           Cast does not convert
                          validation                  

  `cast()` to validate    Pydantic `model_validate()` Cast performs no
  JSON                                                runtime check

  `# type: ignore`        Fix the underlying type     Suppression hides
  everywhere                                          problems

  Pydantic V1             V2 `@field_validator`       Modern API
  `@validator`                                        

  Pydantic V1             V2 `@model_validator`       Modern API
  `@root_validator`                                   

  `b["name"]` on a model  `b.name`                    Models use attributes

  `b.name` on a dict      `b["name"]`                 Dictionaries use keys

  `dict[str, Any]` for    `TypedDict` or Pydantic     Documents structure
  every record            model                       

  ORM model as public     Separate response model     Controls exposure
  schema                                              

  `list = []`             `list[SpecificType] = []`   Gives checker element
                                                      information

  `str | None` without    Explicitly choose           Avoids API contract
  considering defaults    required/optional           confusion

  Commit in every tiny    Deliberate transaction      Predictable transaction
  helper                  boundary                    behavior

  Assume annotation       Use a runtime validator     Static typing is not
  validates at runtime                                runtime validation
  ---------------------------------------------------------------------------

------------------------------------------------------------------------

# 28. Testing types, validation and database behavior

Type checking and tests complement each other.

## 28.1 Test Pydantic validation

``` python
import pytest
from pydantic import ValidationError

def test_band_name_must_be_valid() -> None:
    with pytest.raises(ValidationError):
        BandCreate(name="A", genre="Rock")
```

## 28.2 Test successful validation

``` python
def test_band_create() -> None:
    band = BandCreate(
        name="Computer",
        genre="Alternative",
    )

    assert band.name == "Computer"
```

## 28.3 Test a validator

``` python
def test_genre_is_title_cased() -> None:
    band = BandCreate(
        name="Computer",
        genre="alternative rock",
    )

    assert band.genre == "Alternative Rock"
```

## 28.4 Test database behavior

Use a separate test database, often SQLite in-memory for simple tests.
Be aware that SQLite connection and threading settings affect in-memory
database behavior.

Test:

-   Create.
-   Read.
-   Update.
-   Delete.
-   Missing record.
-   Duplicate or unique constraints.
-   Invalid request data.
-   Database rollback behavior.

## 28.5 What tests cannot replace

Tests only exercise the paths you run. Static type checking can find
incompatible operations on paths not covered by tests.

Likewise, type checking cannot prove that a database constraint, network
service or external client behaves correctly.

Use both.

------------------------------------------------------------------------

# 29. A practical project structure

A growing FastAPI project can be organized like this:

``` text
bands_api/
│
├── app/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes/
│   │       ├── __init__.py
│   │       └── bands.py
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   └── band.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   └── band.py
│   │
│   ├── repositories/
│   │   ├── __init__.py
│   │   └── band.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   └── band.py
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   └── session.py
│   │
│   └── core/
│       ├── __init__.py
│       └── config.py
│
├── tests/
│   ├── test_bands_api.py
│   └── test_band_schemas.py
│
├── alembic/
├── alembic.ini
├── pyproject.toml
└── README.md
```

Responsibilities:

-   `schemas`: Pydantic request and response models.
-   `models`: SQLAlchemy ORM models.
-   `repositories`: Database access.
-   `services`: Business rules.
-   `api/routes`: HTTP endpoints.
-   `db`: Engine, sessions and database setup.
-   `core`: Settings and application configuration.
-   `tests`: Automated tests.

Avoid creating many layers for a tiny exercise. Introduce them as the
application grows.

------------------------------------------------------------------------

# 30. A professional development workflow

Use this sequence when creating a feature.

## Step 1: Define the data contract

Ask:

-   What fields are required?
-   What fields may be omitted?
-   Can a field be `None`?
-   What types are allowed?
-   What constraints apply?
-   What should the API return?

## Step 2: Define the Pydantic schemas

Create request and response models.

## Step 3: Define the database model

Use SQLAlchemy mapped annotations and database constraints.

## Step 4: Implement the data access

Write typed repository functions. Account for missing records with
`T | None`.

## Step 5: Implement business logic

Keep validation and business rules clear. Do not put all application
behavior inside route functions.

## Step 6: Implement the route

Use `Annotated`, `Depends`, Pydantic request models and response models.

## Step 7: Run Ty

``` bash
uv run ty check
```

Fix the underlying cause instead of adding casts or ignores immediately.

## Step 8: Run tests

``` bash
uv run pytest
```

## Step 9: Test the HTTP API

Use `/docs`, an HTTP client file, curl or an API client.

## Step 10: Review the boundaries

Check that:

-   External data is validated.
-   Database values are converted to response schemas.
-   Sensitive columns are not exposed.
-   Missing records are handled.
-   Transactions are controlled.
-   Type annotations reflect actual behavior.

------------------------------------------------------------------------

# 31. Exercises and a learning roadmap

Do not try to master every advanced typing feature in one sitting. Learn
the concepts in sequence.

## Level 1: Fundamentals

-   [ ] Annotate variables.
-   [ ] Annotate function parameters.
-   [ ] Annotate return values.
-   [ ] Use `list`, `dict`, `tuple` and `set`.
-   [ ] Understand `str | None`.
-   [ ] Understand type narrowing with `isinstance`.

Exercise: Create a typed function that accepts a list of prices and
returns the total.

## Level 2: Structured data

-   [ ] Create a `TypedDict`.
-   [ ] Create a Pydantic model.
-   [ ] Compare a dictionary with a model instance.
-   [ ] Use `model_dump`.
-   [ ] Use `model_validate`.
-   [ ] Use optional and required fields.

Exercise: Define `BandData`, `BandCreate` and `BandRead`.

## Level 3: Advanced typing

-   [ ] Learn `Any` versus `object`.
-   [ ] Learn `cast`.
-   [ ] Learn `Literal` and `Enum`.
-   [ ] Learn `Annotated`.
-   [ ] Learn `TypeVar`.
-   [ ] Learn generics.
-   [ ] Learn `Protocol`.
-   [ ] Learn `TypeGuard` and `TypeIs`.

Exercise: Create a generic repository interface and an in-memory
implementation.

## Level 4: Pydantic

-   [ ] Use `Field`.
-   [ ] Use `field_validator`.
-   [ ] Use `model_validator`.
-   [ ] Use before and after validation.
-   [ ] Create reusable annotated types.
-   [ ] Configure extra fields.
-   [ ] Validate ORM objects using `from_attributes`.

Exercise: Create a band schema with name normalization, rating limits
and album data.

## Level 5: FastAPI

-   [ ] Create POST endpoints.
-   [ ] Create GET endpoints.
-   [ ] Use path parameters.
-   [ ] Use query parameters.
-   [ ] Use response models.
-   [ ] Use dependencies.
-   [ ] Handle HTTP exceptions.
-   [ ] Test validation errors.

Exercise: Build a complete in-memory Bands API with create, read,
update, delete and filtering.

## Level 6: Database

-   [ ] Learn `Mapped` and `mapped_column`.
-   [ ] Create SQLAlchemy models.
-   [ ] Create a database engine.
-   [ ] Manage sessions.
-   [ ] Use `select`.
-   [ ] Create relationships.
-   [ ] Use migrations.
-   [ ] Test database operations.

Exercise: Replace the in-memory `BANDS` list with SQLite, then
PostgreSQL.

## Level 7: Professional quality

-   [ ] Configure Ty.
-   [ ] Run Ty in your development workflow.
-   [ ] Add automated tests.
-   [ ] Separate schemas and ORM models.
-   [ ] Add database migrations.
-   [ ] Handle transactions deliberately.
-   [ ] Review security and response serialization.

------------------------------------------------------------------------

# 32. Quick-reference tables

## 32.1 Which type should I use?

  -----------------------------------------------------------------------
  Situation                           Recommended type
  ----------------------------------- -----------------------------------
  Text                                `str`

  Whole number                        `int`

  Decimal number                      `float` or `Decimal` for exact
                                      decimal arithmetic

  True/false                          `bool`

  May be absent                       `T | None`

  List of strings                     `list[str]`

  Dictionary with uniform values      `dict[str, T]`

  Fixed-shape dictionary              `TypedDict`

  Validated input object              Pydantic `BaseModel`

  Database row                        SQLAlchemy ORM model

  Several allowed literal values      `Literal[...]`

  Named domain choices                `Enum` / `StrEnum`

  Distinct logical identifier         `NewType`

  Reusable generic function           `TypeVar` or modern type parameter
                                      syntax

  Behavior-based interface            `Protocol`

  Unknown but safely constrained      `object`
  value                               

  Intentionally dynamic value         `Any`

  Impossible return                   `Never`
  -----------------------------------------------------------------------

## 32.2 Which Pydantic validator should I use?

  -----------------------------------------------------------------------
  Need                                Use
  ----------------------------------- -----------------------------------
  Validate one field after normal     `@field_validator`
  parsing                             

  Transform raw field input           `@field_validator(mode="before")`

  Check multiple fields together      `@model_validator(mode="after")`

  Inspect or normalize raw model      `@model_validator(mode="before")`
  input                               

  Reuse a simple field constraint     `Annotated` + `Field`

  Reuse a small transformation        `Annotated` + `BeforeValidator` /
                                      `AfterValidator`

  Build a deeply integrated custom    Custom core schema, only when
  Pydantic type                       needed
  -----------------------------------------------------------------------

## 32.3 Which conversion should I use?

  Goal                                          Correct operation
  --------------------------------------------- -----------------------------------
  Convert text to integer                       `int(value)`
  Convert integer to text                       `str(value)`
  Validate dictionary with Pydantic             `Model.model_validate(data)`
  Validate JSON string                          `Model.model_validate_json(data)`
  Convert Pydantic model to dictionary          `model.model_dump()`
  Convert Pydantic model to JSON text           `model.model_dump_json()`
  Tell checker a type you have already proved   `cast(Type, value)`
  Check runtime type                            `isinstance(value, Type)`

## 32.4 The most important mental model

  Object                    How to access data
  ------------------------- ----------------------
  Python dictionary         `data["name"]`
  Pydantic model            `model.name`
  Dataclass instance        `instance.name`
  SQLAlchemy ORM instance   `orm_object.name`
  TypedDict value           `typed_dict["name"]`

A TypedDict is still a dictionary at runtime.

------------------------------------------------------------------------

# 33. Official documentation

Use official documentation as the final authority when APIs evolve.

## Python typing

-   Python typing module: https://docs.python.org/3/library/typing.html
-   Typing specification: https://typing.python.org/en/latest/
-   TypedDict specification:
    https://typing.python.org/en/latest/spec/typeddict.html
-   Type checker directives and cast:
    https://typing.python.org/en/latest/spec/directives.html

## Ty

-   Ty documentation: https://docs.astral.sh/ty/
-   Type checking: https://docs.astral.sh/ty/type-checking/
-   Configuration: https://docs.astral.sh/ty/reference/configuration/
-   Typing FAQ: https://docs.astral.sh/ty/reference/typing-faq/

## Pydantic V2

-   Models: https://docs.pydantic.dev/latest/concepts/models/
-   Fields: https://docs.pydantic.dev/latest/concepts/fields/
-   Validators: https://docs.pydantic.dev/latest/concepts/validators/
-   Custom types: https://docs.pydantic.dev/latest/concepts/types/
-   Migration guide: https://docs.pydantic.dev/latest/migration/

## FastAPI

-   Request bodies: https://fastapi.tiangolo.com/tutorial/body/
-   Response models:
    https://fastapi.tiangolo.com/tutorial/response-model/
-   Dependencies: https://fastapi.tiangolo.com/tutorial/dependencies/
-   SQL databases: https://fastapi.tiangolo.com/tutorial/sql-databases/

## SQLAlchemy

-   SQLAlchemy 2.x ORM quick start:
    https://docs.sqlalchemy.org/en/20/orm/quickstart.html
-   ORM mapped classes:
    https://docs.sqlalchemy.org/en/20/orm/declarative_mapping.html
-   SELECT statements:
    https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html
-   Session basics:
    https://docs.sqlalchemy.org/en/20/orm/session_basics.html

------------------------------------------------------------------------

# Final advice: how to avoid type errors

1.  **Know what object you are holding.** Is it a dictionary, Pydantic
    model, dataclass or ORM instance?
2.  **Annotate collections.** Especially empty lists and dictionaries.
3.  **Do not confuse static typing with runtime validation.**
4.  **Use Pydantic at external data boundaries.**
5.  **Use TypedDict for known dictionary shapes.**
6.  **Use ORM models for database persistence.**
7.  **Use response schemas to control what your API exposes.**
8.  **Handle `None` explicitly.** Database lookups frequently return
    `T | None`.
9.  **Use `cast` sparingly.** It changes the checker's understanding,
    not the actual value.
10. **Do not suppress an error before understanding it.**
11. **Run Ty regularly.** A type error is often an opportunity to
    improve your design.
12. **Keep your dependencies and Python target consistent.** The code,
    interpreter, virtual environment, editor and checker should agree.

The objective is not to make every warning disappear. The objective is
to design a program whose types accurately describe its real behavior.
