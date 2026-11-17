# SQLModel Master Guide: Models, Parent–Child Relationships, Validation, and FastAPI Endpoints

## Table of Contents

1. Understanding what a model is
2. Understanding the different types of models
3. Understanding parent–child relationships
4. Understanding foreign keys
5. Understanding `Relationship()` and `back_populates`
6. Understanding how SQLModel models work together
7. Understanding Pydantic validation
8. Understanding `model_validate()` and `model_dump()`
9. Creating a parent with children
10. Creating a child belonging to an existing parent
11. Reading parent and child records
12. Updating parent and child records
13. Deleting parent and child records
14. Understanding database sessions
15. Common mistakes
16. Recommended project structure
17. Best practices and learning exercises

---

# 1. Understanding What a Model Is

Before learning relationships, you need to understand what a model represents.

A model is a Python class that describes the structure of some data.

For example, imagine we are building a music application.

We have bands and albums.

A band has information such as:

- ID
- Name
- Genre

An album has information such as:

- ID
- Title
- Release year
- Band ID

We can represent a band using a Python class:

```python
class Band:
    def __init__(self, name, genre):
        self.name = name
        self.genre = genre
```

Creating an object:

```python
band = Band(
    name="Coldplay",
    genre="Alternative"
)
```

Here:

- `Band` is the class.
- `band` is an instance of the class.
- `name` and `genre` are attributes.

SQLModel builds on this Python class concept.

It allows us to define models that can participate in database operations.

For example:

```python
from sqlmodel import SQLModel, Field


class Band(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    genre: str
```

The important part is:

```python
class Band(SQLModel, table=True):
```

`table=True` tells SQLModel that this class represents a database table.

Without `table=True`, the model is not a database table. It can still be used for validation and data transfer.

That distinction is extremely important.

# 2. Understanding the Different Types of Models

One of the biggest mistakes beginners make is trying to use one model for everything.

For example, you might use the same model to:

- Receive data from a client.
- Validate the incoming data.
- Insert data into a database.
- Return data to the client.
- Update existing records.

This can work for very small applications, but it becomes difficult to manage as your application grows.

A better approach is to separate responsibilities.

We will use five model types.

## 2.1 Base model

A Base model contains fields shared by other models.

Example:

```python
class BandBase(SQLModel):
    name: str
    genre: str
```

Notice that there is no `table=True`.

It is not a database table.

It simply defines common fields.

## 2.2 Create model

A Create model describes the data the client must provide when creating a resource.

```python
class BandCreate(BandBase):
    pass
```

This inherits:

```python
name: str
genre: str
```

Why do we need this?

Because the client should not normally provide the database-generated ID.

For example, the client sends:

```json
{
    "name": "Coldplay",
    "genre": "Alternative"
}
```

The database generates the ID.

The Create model describes what the client is allowed or required to send.

## 2.3 Table model

The Table model represents the actual database table.

```python
class Band(BandBase, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )
```

Here:

```python
table=True
```

makes it a database table.

And:

```python
id: int | None = Field(
    default=None,
    primary_key=True
)
```

defines the primary key.

The ID is optional when creating a new Python object because the database has not generated it yet.

After insertion, the database-generated ID is populated.

## 2.4 Read model

A Read model describes what the API returns to the client.

```python
class BandRead(BandBase):
    id: int
```

Why separate it from the Table model?

Because the database may contain fields that you do not want to expose through your API.

For example:

- Internal database fields.
- Private information.
- Administrative fields.
- Sensitive information.

A Read model controls the response structure.

## 2.5 Update model

An Update model describes which fields a client can change.

```python
class BandUpdate(SQLModel):
    name: str | None = None
    genre: str | None = None
```

Why make the fields optional?

Because an update may change only one field.

For example:

```json
{
    "name": "Coldplay New Name"
}
```

The client does not need to send the genre again.

### Summary of model responsibilities

| Model | Responsibility |
|---|---|
| `BandBase` | Shared fields |
| `BandCreate` | Incoming creation data |
| `Band` | Database table |
| `BandRead` | Outgoing response data |
| `BandUpdate` | Incoming update data |

Remember:

**Create, Read and Update models are API/data models. The Table model represents your database entity.**

---

# 3. Understanding Parent–Child Relationships

Now let's discuss the most important part.

Imagine we have:

- One band.
- Several albums.

For example:

```text
Coldplay
    |
    |---- Parachutes
    |
    |---- X&Y
    |
    |---- Viva la Vida
```

Here:

- Band is the parent.
- Albums are the children.

Why is Band the parent?

Because the albums belong to a band.

One band can have many albums.

This is called a **one-to-many relationship**.

Other examples:

| Parent | Children |
|---|---|
| Customer | Orders |
| Author | Books |
| Department | Employees |
| Blog | Posts |
| Band | Albums |
| School | Students |

The relationship can be represented as:

```text
One Band  --------  Many Albums
```

A band can have many albums, but each album belongs to one band.

This is the relationship we will implement.

# 4. Understanding Foreign Keys

A foreign key connects two database tables.

Let's imagine two tables.

## Band table

| id | name | genre |
|---:|---|---|
| 1 | Coldplay | Alternative |
| 2 | Metallica | Heavy |

## Album table

| id | title | band_id |
|---:|---|---:|
| 1 | Parachutes | 1 |
| 2 | X&Y | 1 |
| 3 | Black Album | 2 |

Look at the `band_id` column.

The first two albums have:

```text
band_id = 1
```

That means they belong to Coldplay.

The third album has:

```text
band_id = 2
```

That means it belongs to Metallica.

The foreign key is:

```python
band_id: int = Field(foreign_key="band.id")
```

Let's break it down.

### `band_id`

This is the column in the Album table.

### `foreign_key`

This tells SQLModel that this column references another table.

### `"band.id"`

This means:

```text
Table name: band
Column name: id
```

The foreign key is the database-level connection between the tables.

Important distinction:

**A foreign key and an ORM relationship are not the same thing.**

A foreign key connects database records.

An ORM relationship helps you navigate between Python objects.

# 5. Understanding Relationship() and back_populates

Now let's introduce:

```python
Relationship()
```

SQLModel provides `Relationship()` to describe relationships between Python model objects.

Consider these two models:

```python
class Album(AlbumBase, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    band_id: int = Field(foreign_key="band.id")

    band: "Band" = Relationship(
        back_populates="albums"
    )
```

And:

```python
class Band(BandBase, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    albums: list["Album"] = Relationship(
        back_populates="band"
    )
```

Let's understand both sides.

## 5.1 The child side: Album

```python
band: "Band" = Relationship(
    back_populates="albums"
)
```

This means an Album object can access its parent Band object.

For example:

```python
album.band
```

You can access:

```python
album.band.name
album.band.genre
```

The `"Band"` is a forward reference. It allows you to refer to a class that is defined elsewhere or later.

## 5.2 The parent side: Band

```python
albums: list["Album"] = Relationship(
    back_populates="band"
)
```

This means a Band object can access its child Album objects.

For example:

```python
band.albums
```

You can loop through them:

```python
for album in band.albums:
    print(album.title)
```

## 5.3 Why back_populates?

Look carefully:

On Album:

```python
Relationship(back_populates="albums")
```

On Band:

```python
Relationship(back_populates="band")
```

The names refer to the opposite relationship attributes.

They connect the two sides.

```text
Band.albums  <---------->  Album.band
```

They must match the attribute names on the opposite model.

Do not confuse these names with database column names.

`back_populates` is about the Python relationship attributes.

# 6. Complete Parent–Child Models

Let's put everything together.

For this example, we will create a music application.

```python
from datetime import date
from enum import Enum

from pydantic import field_validator
from sqlmodel import SQLModel, Field, Relationship


class GenreURLChoices(str, Enum):
    POP_ROCK = "Pop Rock"
    METAL = "Metal"
    ALTERNATIVE = "Alternative"
    HEAVY = "Heavy"


# =================================
# ALBUM MODELS
# =================================

class AlbumBase(SQLModel):
    title: str
    release_year: date


class AlbumCreate(AlbumBase):
    pass


class Album(AlbumBase, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    band_id: int | None = Field(
        default=None,
        foreign_key="band.id"
    )

    band: "Band" = Relationship(
        back_populates="albums"
    )


class AlbumRead(AlbumBase):
    id: int
    band_id: int


# =================================
# BAND MODELS
# =================================

class BandBase(SQLModel):
    name: str
    genre: GenreURLChoices


class BandCreate(BandBase):
    albums: list[AlbumCreate] = Field(
        default_factory=list
    )

    @field_validator("genre", mode="before")
    @classmethod
    def title_case_genre(cls, value):
        return value.title() if isinstance(value, str) else value


class BandUpdate(SQLModel):
    name: str | None = None
    genre: GenreURLChoices | None = None


class Band(BandBase, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    albums: list[Album] = Relationship(
        back_populates="band"
    )


class BandRead(BandBase):
    id: int
    albums: list[AlbumRead] = Field(
        default_factory=list
    )
```

### Important note about band_id

In this example:

```python
band_id: int | None = Field(
    default=None,
    foreign_key="band.id"
)
```

The field is optional during Python object construction.

The database column is nullable by default in this example. For a strict production design where every album must belong to a band, enforce a NOT NULL database constraint and ensure every album is associated with a band before committing.

The relationship attribute itself does not create the foreign key column. The `Field(foreign_key=...)` declaration does that.

---

# 7. Understanding Pydantic Validation

This is another important topic.

SQLModel uses Pydantic for data validation.

Suppose we have:

```python
class BandCreate(SQLModel):
    name: str
    genre: GenreURLChoices
```

A client sends:

```json
{
    "name": "Coldplay",
    "genre": "Alternative"
}
```

Pydantic checks the incoming data.

It checks:

1. Does `name` exist?
2. Is `name` a string?
3. Does `genre` exist?
4. Is `genre` an allowed enum value?

If the client sends:

```json
{
    "name": 123,
    "genre": "Alternative"
}
```

Pydantic will reject the invalid name under its normal strictness/coercion rules.

If the client sends:

```json
{
    "name": "Coldplay",
    "genre": "Jazz"
}
```

Pydantic rejects the genre because Jazz is not in the Enum.

FastAPI automatically performs this validation when you use a Pydantic or SQLModel model as an endpoint parameter.

Example:

```python
@app.post("/bands")
def create_band(band_data: BandCreate):
    ...
```

Here:

```python
band_data: BandCreate
```

tells FastAPI to validate the request body using `BandCreate`.

You do not need to manually call `BandCreate.model_validate()` inside this endpoint.

FastAPI has already done that.

# 8. Understanding Validators

A validator allows you to add your own validation or transformation rules.

For example:

```python
from pydantic import field_validator


class BandCreate(BandBase):

    @field_validator("genre", mode="before")
    @classmethod
    def title_case_genre(cls, value):
        return value.title() if isinstance(value, str) else value
```

Let's understand the parts.

## `@field_validator("genre")`

This tells Pydantic:

I want to validate the `genre` field.

## `mode="before"`

This tells Pydantic to execute the validator before normal field validation.

For example, the incoming value might be:

```text
ALTERNATIVE
```

Your validator transforms it:

```text
Alternative
```

Then Pydantic validates it against the Enum.

## `@classmethod`

The validator is declared as a class method.

Pydantic expects the class and the incoming value as arguments for this form.

## `value`

This is the value supplied for the field.

Example:

```python
value = "ALTERNATIVE"
```

## Return value

A validator should return the value that Pydantic will validate or use.

```python
return value.title()
```

### Another validator example

Suppose you want to prevent an empty band name.

```python
from pydantic import field_validator


class BandCreate(BandBase):

    @field_validator("name")
    @classmethod
    def validate_name(cls, value: str):
        if not value.strip():
            raise ValueError("Band name cannot be empty")

        return value.strip()
```

This rejects whitespace-only names and removes surrounding whitespace.

### Before versus after validators

| Mode | When it runs |
|---|---|
| `before` | Before Pydantic performs normal validation |
| `after` | After Pydantic has validated and converted the value |

Use `before` when you want to normalize raw incoming data.

Use `after` when you want to check an already validated value.

For example:

```python
@field_validator("name", mode="before")
@classmethod
def normalize_name(cls, value):
    if isinstance(value, str):
        return value.strip()

    return value
```

And:

```python
@field_validator("name", mode="after")
@classmethod
def check_name(cls, value: str):
    if len(value) < 2:
        raise ValueError("Name must contain at least two characters")

    return value
```

### Model validators

Sometimes validation involves multiple fields.

For example, you may want to ensure that an album's release date is not in the future.

Or that two related fields follow a particular rule.

Pydantic provides:

```python
from pydantic import model_validator
```

Example:

```python
from datetime import date
from pydantic import model_validator


class AlbumCreate(SQLModel):
    title: str
    release_year: date

    @model_validator(mode="after")
    def validate_release_date(self):
        if self.release_year > date.today():
            raise ValueError("Release date cannot be in the future")

        return self
```

A field validator focuses on one field.

A model validator can inspect multiple fields together.

---

# 9. Understanding model_validate() and model_dump()

These two methods are frequently used in SQLModel and Pydantic applications.

They do different things.

## 9.1 model_validate()

This creates a model instance from input data and validates it.

Example:

```python
band_data = {
    "name": "Coldplay",
    "genre": "Alternative"
}

band = BandCreate.model_validate(band_data)
```

Now:

```python
band.name
band.genre
```

You have a validated model instance.

You can also validate another Pydantic model:

```python
band = Band.model_validate(band_data)
```

However, be careful when the source is a Create model containing fields that are not part of the Table model.

For example, `BandCreate` contains `albums`, while your Band table model uses a relationship named `albums`.

A model conversion does not automatically guarantee that related database objects are created and persisted correctly. Explicitly construct and attach child objects when saving nested data.

## 9.2 model_dump()

This converts a Pydantic or SQLModel instance into a Python dictionary.

Example:

```python
band = BandCreate(
    name="Coldplay",
    genre="Alternative"
)

band_dict = band.model_dump()
```

Conceptually:

```python
{
    "name": "Coldplay",
    "genre": GenreURLChoices.ALTERNATIVE
}
```

You can use:

```python
band_dict["name"]
```

or:

```python
band_dict["genre"]
```

The default `model_dump()` mode is Python mode. Enum values may remain Enum members. Use:

```python
band.model_dump(mode="json")
```

when you need JSON-compatible values.

### Remember the difference

| Method | What it does |
|---|---|
| `model_validate()` | Input data → validated model |
| `model_dump()` | Model → Python dictionary |
| `model_dump(mode="json")` | Model → JSON-compatible Python data |

---

# 10. Creating a Parent With Children

Now we reach the most important practical section.

Suppose a client wants to create a band and its albums in one request.

The client sends:

```json
{
    "name": "Coldplay",
    "genre": "Alternative",
    "albums": [
        {
            "title": "Parachutes",
            "release_year": "2000-07-10"
        },
        {
            "title": "X&Y",
            "release_year": "2005-06-06"
        }
    ]
}
```

Notice something important.

The client does not send:

```json
"band_id": 1
```

Why?

Because the band does not exist yet.

The server must create the band first.

## 10.1 The recommended relationship approach

```python
from fastapi import Depends, HTTPException
from sqlmodel import Session


@app.post("/bands", response_model=BandRead)
def create_band(
    band_data: BandCreate,
    session: Session = Depends(get_session)
) -> Band:

    try:
        band = Band(
            name=band_data.name,
            genre=band_data.genre,
            albums=[
                Album(
                    title=album.title,
                    release_year=album.release_year
                )
                for album in band_data.albums
            ]
        )

        session.add(band)

        session.commit()

        session.refresh(band)

        return band

    except Exception:
        session.rollback()
        raise
```

Let's understand this carefully.

### Step 1: Receive the request

```python
band_data: BandCreate
```

FastAPI validates the incoming JSON.

You can now access:

```python
band_data.name
band_data.genre
band_data.albums
```

### Step 2: Create the parent

```python
band = Band(
    name=band_data.name,
    genre=band_data.genre
)
```

This creates the Band Python object.

It is not yet saved in the database.

### Step 3: Create the child objects

```python
albums=[
    Album(
        title=album.title,
        release_year=album.release_year
    )
    for album in band_data.albums
]
```

This is a list comprehension.

It loops over the incoming albums and creates Album objects.

For example, the first iteration creates:

```python
Album(
    title="Parachutes",
    release_year=date(2000, 7, 10)
)
```

The resulting list is assigned to the parent's `albums` relationship.

### Step 4: Add the parent

```python
session.add(band)
```

SQLAlchemy tracks the band and its related albums through the relationship.

### Step 5: Commit

```python
session.commit()
```

SQLAlchemy inserts the band and its related albums in the correct order, managing the foreign key values.

You do not need to manually generate the band's ID.

You do not need to manually assign each album's `band_id`.

The relationship handles that.

**Important:** This relies on the default save-update cascade for the relationship, which is the normal SQLAlchemy behavior.

---

# 11. Alternative: Create Parent First, Then Children

You can also create the parent and children in separate steps.

This approach is useful for understanding primary keys and foreign keys.

```python
@app.post("/bands", response_model=BandRead)
def create_band(
    band_data: BandCreate,
    session: Session = Depends(get_session)
) -> Band:

    try:
        band = Band(
            name=band_data.name,
            genre=band_data.genre
        )

        session.add(band)

        # Send the pending INSERT to the database
        session.flush()

        # The database-generated ID is now available
        for album in band_data.albums:

            album_obj = Album(
                title=album.title,
                release_year=album.release_year,
                band_id=band.id
            )

            session.add(album_obj)

        session.commit()
        session.refresh(band)

        return band

    except Exception:
        session.rollback()
        raise
```

### Understanding flush()

This is important:

```python
session.flush()
```

It sends pending database operations to the database without committing the transaction.

Before flush:

```python
band.id  # None
```

After flush:

```python
band.id  # Database-generated ID
```

This allows you to assign the ID to the albums.

### Understanding commit()

```python
session.commit()
```

This commits the transaction.

The band and albums become part of the committed database state together.

### Understanding rollback()

```python
session.rollback()
```

If an operation fails before the transaction commits, rollback cancels the pending transaction changes.

Do not confuse:

| Operation | Responsibility |
|---|---|
| `session.add()` | Register an object with the session |
| `session.flush()` | Send pending changes to the database |
| `session.commit()` | Commit the transaction |
| `session.rollback()` | Roll back the current transaction |
| `session.refresh()` | Reload an object's database-backed state |

---

# 12. Creating a Child Belonging to an Existing Parent

Now let's reverse the situation.

Suppose Coldplay already exists.

The database contains:

```text
Band
id = 1
name = Coldplay
```

Now the client wants to add a new album to that band.

The client sends:

```json
{
    "title": "Ghost Stories",
    "release_year": "2014-05-16"
}
```

Notice that the client does not need to send the band information again.

The band ID is provided in the URL.

Example:

```text
POST /bands/1/albums
```

Create the endpoint:

```python
@app.post("/bands/{band_id}/albums", response_model=AlbumRead)
def create_album(
    band_id: int,
    album_data: AlbumCreate,
    session: Session = Depends(get_session)
) -> Album:

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found"
        )

    album = Album(
        title=album_data.title,
        release_year=album_data.release_year,
        band_id=band.id
    )

    session.add(album)
    session.commit()
    session.refresh(album)

    return album
```

### Understand the endpoint

```python
band_id: int
```

FastAPI extracts the ID from the URL.

```python
album_data: AlbumCreate
```

FastAPI validates the request body.

```python
band = session.get(Band, band_id)
```

This retrieves the existing band from the database.

If it doesn't exist, return HTTP 404.

Then:

```python
album = Album(
    title=album_data.title,
    release_year=album_data.release_year,
    band_id=band.id
)
```

You create the album and associate it with the existing band.

This is the child-creation direction.

---

# 13. Reading Parent and Child Records

You need to understand how to retrieve both sides of a relationship.

## 13.1 Read one band

```python
@app.get("/bands/{band_id}", response_model=BandRead)
def get_band(
    band_id: int,
    session: Session = Depends(get_session)
) -> Band:

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found"
        )

    return band
```

Your response model includes albums, so the relationship must be loaded before response serialization.

Depending on your session and loading configuration, accessing `band.albums` may trigger a database query.

For predictable response loading, especially when using asynchronous database sessions or avoiding lazy-loading problems, configure or request eager loading.

With SQLModel and SQLAlchemy:

```python
from sqlalchemy.orm import selectinload
from sqlmodel import select


statement = (
    select(Band)
    .where(Band.id == band_id)
    .options(selectinload(Band.albums))
)

band = session.exec(statement).first()
```

This loads the band and its albums using a separate relationship query.

## 13.2 Read all bands with albums

```python
@app.get("/bands", response_model=list[BandRead])
def get_bands(
    session: Session = Depends(get_session)
) -> list[Band]:

    statement = select(Band).options(
        selectinload(Band.albums)
    )

    return session.exec(statement).all()
```

Remember to import:

```python
from sqlalchemy.orm import selectinload
from sqlmodel import select
```

## 13.3 Read all albums belonging to a band

```python
@app.get("/bands/{band_id}/albums", response_model=list[AlbumRead])
def get_band_albums(
    band_id: int,
    session: Session = Depends(get_session)
) -> list[Album]:

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found"
        )

    return band.albums
```

You are accessing the children through the parent relationship.

---

# 14. Updating Parent and Child Records

Let's understand updates.

There are two separate operations:

1. Updating the parent.
2. Updating a child.

## 14.1 Update a band

The client sends:

```json
{
    "name": "Coldplay Updated"
}
```

Endpoint:

```python
@app.patch("/bands/{band_id}", response_model=BandRead)
def update_band(
    band_id: int,
    band_data: BandUpdate,
    session: Session = Depends(get_session)
) -> Band:

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found"
        )

    update_data = band_data.model_dump(exclude_unset=True)

    for key, value in update_data.items():
        setattr(band, key, value)

    session.add(band)
    session.commit()
    session.refresh(band)

    return band
```

Understand:

```python
band_data.model_dump(exclude_unset=True)
```

This is very important for PATCH requests.

Suppose the client sends only:

```json
{
    "name": "Coldplay Updated"
}
```

The resulting dictionary contains only the field supplied by the client.

It does not contain:

```python
"genre": None
```

That prevents an omitted field from accidentally overwriting an existing value.

Then:

```python
setattr(band, key, value)
```

dynamically updates the object's attribute.

It is equivalent to:

```python
band.name = "Coldplay Updated"
```

## 14.2 Update an album

```python
class AlbumUpdate(SQLModel):
    title: str | None = None
    release_year: date | None = None
```

Endpoint:

```python
@app.patch("/albums/{album_id}", response_model=AlbumRead)
def update_album(
    album_id: int,
    album_data: AlbumUpdate,
    session: Session = Depends(get_session)
) -> Album:

    album = session.get(Album, album_id)

    if album is None:
        raise HTTPException(
            status_code=404,
            detail="Album not found"
        )

    update_data = album_data.model_dump(exclude_unset=True)

    for key, value in update_data.items():
        setattr(album, key, value)

    session.add(album)
    session.commit()
    session.refresh(album)

    return album
```

You can also move an album to another band by changing its foreign key, provided the target band exists.

For example:

```python
album.band_id = another_band.id
```

That changes the database relationship.

---

# 15. Deleting Parent and Child Records

Deleting a parent raises an important question:

What should happen to its children?

Suppose we delete Coldplay.

Should its albums also be deleted?

There are different possible rules:

1. Delete the band and all its albums.
2. Prevent deletion while albums exist.
3. Keep albums but detach them from the band, if the database permits it.

You must decide based on your application's requirements.

## 15.1 Delete one album

```python
@app.delete("/albums/{album_id}")
def delete_album(
    album_id: int,
    session: Session = Depends(get_session)
):

    album = session.get(Album, album_id)

    if album is None:
        raise HTTPException(
            status_code=404,
            detail="Album not found"
        )

    session.delete(album)
    session.commit()

    return {"message": "Album deleted successfully"}
```

## 15.2 Delete a band

For a band whose albums should also be deleted, configure the relationship with an appropriate ORM cascade:

```python
albums: list["Album"] = Relationship(
    back_populates="band",
    sa_relationship_kwargs={
        "cascade": "all, delete-orphan"
    }
)
```

Then:

```python
@app.delete("/bands/{band_id}")
def delete_band(
    band_id: int,
    session: Session = Depends(get_session)
):

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found"
        )

    session.delete(band)
    session.commit()

    return {"message": "Band and its albums deleted"}
```

Important:

- ORM cascade controls what SQLAlchemy does with related objects.
- Database `ON DELETE CASCADE` controls what the database does when a referenced parent row is deleted.
- They are related concepts but are not identical.

For a production application, choose a consistent deletion strategy and configure both ORM and database constraints appropriately.

---

# 16. Understanding Database Sessions

You will repeatedly see:

```python
session: Session = Depends(get_session)
```

Let's understand it.

A database session is the object SQLAlchemy uses to communicate with the database and manage a transaction.

A typical dependency looks like:

```python
from sqlmodel import Session


def get_session():
    with Session(engine) as session:
        yield session
```

FastAPI calls this dependency and provides the session to your endpoint.

For example:

```python
@app.get("/bands")
def get_bands(
    session: Session = Depends(get_session)
):
    ...
```

You do not manually create a new session inside every endpoint.

FastAPI provides one through the dependency.

The `with` statement ensures that the session is closed when its dependency lifecycle ends.

For synchronous SQLModel operations, use normal `def` endpoints.

If you use asynchronous database drivers and an `AsyncSession`, then use the corresponding async methods and `async def` endpoints.

Do not mix synchronous `Session` operations with the assumption that they become asynchronous just because your endpoint is declared `async def`.

---

# 17. Understanding the Most Important Model Differences

Let's summarize the concepts that are easy to confuse.

| Concept | Meaning |
|---|---|
| `SQLModel` | Base class for SQLModel models |
| `table=True` | Makes the class a database table |
| `Field()` | Defines field metadata, constraints and database configuration |
| `primary_key=True` | Declares the primary key |
| `foreign_key="band.id"` | Connects a database column to another table's primary key |
| `Relationship()` | Defines an ORM relationship between Python objects |
| `back_populates` | Connects the two sides of a relationship |
| `BandCreate` | Describes incoming band creation data |
| `BandRead` | Describes outgoing band response data |
| `BandUpdate` | Describes incoming update data |
| `model_validate()` | Validates data and constructs a model |
| `model_dump()` | Converts a model into a dictionary |
| `field_validator()` | Adds custom validation for a field |
| `model_validator()` | Adds custom validation involving a model's data |
| `session.add()` | Adds an object to session tracking |
| `session.flush()` | Sends pending changes to the database |
| `session.commit()` | Commits the transaction |
| `session.rollback()` | Rolls back the current transaction |
| `session.refresh()` | Reloads database-backed object state |

---

# 18. Common Mistakes to Avoid

## Mistake 1: Confusing a foreign key with a relationship

Incorrect understanding:

```python
band_id = Field(foreign_key="band.id")
```

means I can write:

```python
album.band
```

Not by itself.

You also need:

```python
band: "Band" = Relationship(back_populates="albums")
```

The foreign key and relationship have different responsibilities.

## Mistake 2: Using the table model for every operation

You should not assume the same model is always appropriate for creation, reading and updating.

Separate them when their responsibilities differ.

## Mistake 3: Expecting model_validate() to save data

This:

```python
band = Band.model_validate(band_data)
```

creates a Python model object.

It does not insert it into the database.

You still need:

```python
session.add(band)
session.commit()
```

## Mistake 4: Confusing flush and commit

`flush()` sends pending SQL operations to the database.

`commit()` commits the transaction.

They are not interchangeable.

## Mistake 5: Forgetting that a child needs a parent

If every album must belong to a band, ensure that the relationship is established before committing the album.

## Mistake 6: Forgetting response serialization and relationship loading

If your response model contains:

```python
albums: list[AlbumRead]
```

you must ensure that the albums are loaded before FastAPI serializes the response.

Use eager loading where appropriate.

## Mistake 7: Assuming a validator automatically changes every model

A validator belongs to the model in which you declare it.

If you have:

```python
class BandCreate(BandBase):
    ...
```

and the validator is defined there, do not assume it also runs when someone directly constructs a different model such as `Band`.

Keep validation rules in shared models where appropriate, or explicitly reuse the relevant validation logic.

---

# 19. Recommended Project Structure

As your application grows, organize your files.

```text
introduction_into_fastapi/
│
├── main.py
│
├── db.py
│
├── models/
│   ├── __init__.py
│   ├── band.py
│   └── album.py
│
├── schemas/
│   ├── __init__.py
│   ├── band.py
│   └── album.py
│
├── routers/
│   ├── __init__.py
│   ├── bands.py
│   └── albums.py
│
└── services/
    ├── __init__.py
    └── band_service.py
```

One possible responsibility arrangement:

- `models/`: SQLModel table classes and database relationships.
- `schemas/`: Create, Read and Update models.
- `routers/`: FastAPI endpoints.
- `services/`: Business logic when your application grows.
- `db.py`: Database engine and session dependency.
- `main.py`: FastAPI application setup.

For a small learning project, keeping the models and schemas together is acceptable. Do not split files unnecessarily before you understand the concepts.

---

# 20. The Best Learning Approach

Do not try to learn all the operations at once.

Build the music application in this order.

## Stage 1: Understand individual models

Create:

- BandBase
- BandCreate
- Band
- BandRead
- BandUpdate

Do the same for Album.

Make sure you understand why each model exists.

## Stage 2: Understand database relationships

Learn:

- Primary keys.
- Foreign keys.
- One-to-many relationships.
- `Relationship()`.
- `back_populates`.

Create a Band and an Album manually.

## Stage 3: Create a parent with children

Implement:

```text
POST /bands
```

Allow the client to create a band and several albums in one request.

## Stage 4: Create a child with an existing parent

Implement:

```text
POST /bands/{band_id}/albums
```

## Stage 5: Read relationships

Implement:

```text
GET /bands
GET /bands/{band_id}
GET /bands/{band_id}/albums
```

Learn eager loading and response models.

## Stage 6: Update relationships

Implement:

```text
PATCH /bands/{band_id}
PATCH /albums/{album_id}
```

Then learn how to move an album between bands.

## Stage 7: Delete relationships

Implement:

```text
DELETE /albums/{album_id}
DELETE /bands/{band_id}
```

Understand cascade deletion.

---

# 21. Final Rules to Remember

1. A Base model holds common fields.
2. A Create model describes incoming creation data.
3. A Table model represents the database table.
4. A Read model describes the API response.
5. An Update model describes fields that can be changed.
6. A foreign key connects database records.
7. A Relationship connects Python ORM objects.
8. `back_populates` connects both sides of a relationship.
9. Pydantic validates incoming data.
10. `model_validate()` creates a validated model instance.
11. `model_dump()` converts a model to a dictionary.
12. `session.add()` tracks objects.
13. `session.flush()` sends pending changes to the database.
14. `session.commit()` commits the transaction.
15. `session.rollback()` cancels the current transaction.
16. `session.refresh()` reloads an object's database state.

**The most important idea:**

Do not think of SQLModel as just a way to create database tables.

Think of it as a way to connect three things:

```text
CLIENT REQUEST
      |
      v
PYDANTIC VALIDATION
      |
      v
PYTHON MODEL OBJECTS
      |
      v
SQLMODEL / SQLALCHEMY SESSION
      |
      v
DATABASE TABLES
      |
      v
RESPONSE MODEL
      |
      v
CLIENT
```

Once you understand how data travels through these stages, using SQLModel in FastAPI becomes much easier.
