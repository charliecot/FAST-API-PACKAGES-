# SQLModel and FastAPI: Updating Records (PUT, PATCH, and SQL UPDATE)

## 1. The big picture

Yes, SQLAlchemy/SQLModel has an `update()` statement constructor. `select()` and `update()` are not methods that you call on the same object; they are SQL statement constructors.

| Operation | SQLModel / SQLAlchemy tool | Purpose |
|---|---|---|
| Read | `select(Model)` | Build a SELECT statement |
| Insert | `session.add(obj)` | Add an ORM object to the session |
| Update an ORM object | Change its attributes | SQLAlchemy tracks the changes |
| Direct update | `update(Model).where(...).values(...)` | Build a SQL UPDATE statement |
| Delete | `session.delete(obj)` | Mark an ORM object for deletion |
| Save transaction | `session.commit()` | Commit pending changes |

`session.commit()` is not the same thing as `update()`. `update()` describes what SQL should happen; `commit()` makes the transaction durable.

## 2. A sample table model

```python
from sqlmodel import SQLModel, Field

class Band(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    genre: str
    country: str
```

A database row might be:

```text
id | name    | genre | country
1  | Coldplay| Rock  | UK
```

## 3. Separate input schemas from database models

Do not use the table model as the only schema for every operation. A create schema, update schema, and read schema have different jobs.

```python
from sqlmodel import SQLModel

class BandCreate(SQLModel):
    name: str
    genre: str
    country: str

class BandRead(SQLModel):
    id: int
    name: str
    genre: str
    country: str

class BandUpdate(SQLModel):
    # Optional fields allow a client to send only the fields it wants to change.
    name: str | None = None
    genre: str | None = None
    country: str | None = None
```

`BandUpdate` is designed for PATCH. Because its fields are optional, it can also be used for PUT only if your endpoint explicitly checks that all required fields were supplied. For a strict PUT, define a separate schema with required fields.

## 4. Method A — update an ORM object (recommended for many CRUD operations)

```python
from sqlmodel import Session, select

def rename_band(session: Session, band_id: int, new_name: str):
    band = session.get(Band, band_id)

    if band is None:
        return None

    band.name = new_name       # SQLAlchemy tracks this attribute change.
    session.add(band)          # Usually unnecessary for an already-loaded object,
                               # but harmless and sometimes clarifies intent.
    session.commit()           # Flushes and commits the UPDATE.
    session.refresh(band)      # Reload generated/updated database values.
    return band
```

Conceptually, SQLAlchemy issues SQL similar to:

```sql
UPDATE band
SET name = 'New Name'
WHERE id = 1;
```

You normally do not need to write that SQL yourself. SQLAlchemy detects that `band.name` changed.

### Why `session.get()`?

`session.get(Band, 1)` retrieves a row by its primary key. It returns the ORM object or `None`.

You can also use:

```python
statement = select(Band).where(Band.id == band_id)
band = session.exec(statement).first()
```

Use `.one()` only when exactly one result is expected and you want an error if that assumption is false. `.first()` returns the first result or `None`.

## 5. Method B — direct SQL UPDATE with `update()`

```python
from sqlmodel import Session, update

def rename_band_directly(session: Session, band_id: int, new_name: str):
    statement = (
        update(Band)
        .where(Band.id == band_id)
        .values(name=new_name)
    )

    result = session.exec(statement)
    session.commit()

    return result.rowcount
```

Breakdown:

- `update(Band)` says which table is being updated.
- `.where(Band.id == band_id)` identifies the rows.
- `.values(name=new_name)` supplies the new value.
- `session.exec(statement)` executes the statement.
- `session.commit()` commits the transaction.
- `rowcount` commonly reports how many rows were matched/affected, but exact semantics can vary by database and driver.

Equivalent SQL:

```sql
UPDATE band
SET name = 'New Name'
WHERE id = 1;
```

### ORM-object update vs direct UPDATE

| ORM object approach | Direct `update()` |
|---|---|
| Loads the object first | Can update without loading the object |
| Convenient for validation and business logic involving the object | Efficient for bulk/simple changes |
| Attribute changes are tracked | You explicitly specify values |
| Relationships and object state are easier to work with | Does not automatically synchronize every already-loaded ORM object in all execution modes |
| Easy to return the updated object | Often returns row count, not a populated object |

For ordinary API CRUD, loading the object, changing fields, committing, and returning it is a very clear approach. For bulk updates, direct `update()` can be more efficient.

## 6. PATCH — partial update

PATCH means the client sends only the fields it wants to change.

Example request:

```http
PATCH /bands/1
Content-Type: application/json

{
  "genre": "Alternative Rock"
}
```

The name and country remain unchanged.

### Important: `model_dump(exclude_unset=True)`

```python
changes = payload.model_dump(exclude_unset=True)
```

This includes only fields the client actually supplied.

Why is that important? Suppose the schema is:

```python
class BandUpdate(SQLModel):
    name: str | None = None
    genre: str | None = None
    country: str | None = None
```

A request containing only `{"genre": "Jazz"}` creates a model whose other attributes have defaults of `None`. If you dump every field, you could accidentally overwrite the band's name and country with `None`.

`exclude_unset=True` distinguishes fields omitted by the client from fields explicitly supplied.

### Complete PATCH endpoint

```python
from fastapi import FastAPI, HTTPException, status
from sqlmodel import Session

app = FastAPI()

@app.patch("/bands/{band_id}", response_model=BandRead)
def update_band(
    band_id: int,
    payload: BandUpdate,
):
    with Session(engine) as session:
        band = session.get(Band, band_id)

        if band is None:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Band not found",
            )

        changes = payload.model_dump(exclude_unset=True)

        if not changes:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="No fields were supplied",
            )

        for field, value in changes.items():
            setattr(band, field, value)

        session.add(band)
        session.commit()
        session.refresh(band)

        return band
```

### Explicit null versus omitted field

These are not necessarily the same:

```json
{}
```

means no field was supplied.

```json
{"country": null}
```

means the client explicitly supplied `country` with a null value.

If `country` is non-nullable in the database, validate and reject `null`. If it is nullable, decide whether clearing it is allowed. `exclude_unset=True` preserves this distinction; `exclude_none=True` does not.

## 7. PUT — full replacement

PUT conventionally means replacing the resource representation. A strict PUT schema should require the fields that make up the replaceable representation:

```python
class BandReplace(SQLModel):
    name: str
    genre: str
    country: str
```

Endpoint:

```python
@app.put("/bands/{band_id}", response_model=BandRead)
def replace_band(band_id: int, payload: BandReplace):
    with Session(engine) as session:
        band = session.get(Band, band_id)

        if band is None:
            raise HTTPException(status_code=404, detail="Band not found")

        band.name = payload.name
        band.genre = payload.genre
        band.country = payload.country

        session.add(band)
        session.commit()
        session.refresh(band)
        return band
```

A PUT request should provide the complete set of fields required by the API's replacement representation. Do not silently treat an incomplete PUT as PATCH unless that is an intentional API design.

## 8. PUT vs PATCH

| Detail | PUT | PATCH |
|---|---|---|
| Meaning | Replace resource representation | Apply partial changes |
| Typical payload | Complete representation | Only changed fields |
| Missing fields | Usually invalid or handled according to replacement rules | Left unchanged |
| Schema | Required fields | Optional fields |
| Example | Change name, genre, country together | Change genre only |

HTTP semantics are defined by the API contract. Be consistent and document what your endpoint expects.

## 9. Update a related model

Suppose Album has a foreign key to Band:

```python
class Album(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    title: str
    band_id: int = Field(foreign_key="band.id")
```

To move an album to another band, update `band_id` after checking that the target band exists:

```python
album = session.get(Album, album_id)
new_band = session.get(Band, new_band_id)

if album is None:
    raise HTTPException(404, "Album not found")
if new_band is None:
    raise HTTPException(404, "Target band not found")

album.band_id = new_band.id
session.commit()
```

The foreign key is what stores the relationship in the database.

## 10. Transactions, rollback, and errors

A transaction groups database operations into one unit.

```python
try:
    band.name = "Updated"
    session.commit()
except Exception:
    session.rollback()
    raise
```

A context manager for a session does not mean every transaction automatically succeeds. If a flush or commit fails, roll back before reusing that session. In production code, catch expected database exceptions specifically rather than hiding every error.

## 11. Common mistakes

1. **Forgetting `commit()`**: the change may not be persisted.
2. **Using `model_dump()` for PATCH without `exclude_unset=True`**: omitted fields can be overwritten.
3. **Not checking whether the row exists**: return 404 rather than causing an attribute error.
4. **Using a create schema for update**: create and update have different validation requirements.
5. **Allowing clients to update protected fields**: never accept `id`, ownership, role, or permission fields unless explicitly authorised.
6. **Forgetting validation**: optional typing does not mean every `None` is valid for the database.
7. **Updating without a WHERE clause**: a direct UPDATE without a condition may change every row.

## 12. Practice

1. Implement `PATCH /bands/{id}` that changes only `name`.
2. Implement strict `PUT /bands/{id}`.
3. Implement `PATCH /albums/{id}` to change an album title.
4. Implement an endpoint that moves an album to another existing band.
5. Try a nonexistent ID and return 404.
6. Try an empty PATCH body and decide whether to reject it.
7. Compare SQL logs for ORM-object update and direct `update()`.

## 13. Quick reference

```python
# Read
band = session.get(Band, band_id)
bands = session.exec(select(Band)).all()

# Insert
session.add(Band(name="Coldplay", genre="Rock", country="UK"))
session.commit()

# ORM update
band.name = "New Name"
session.commit()
session.refresh(band)

# Direct UPDATE
statement = update(Band).where(Band.id == band_id).values(name="New Name")
session.exec(statement)
session.commit()

# Delete
session.delete(band)
session.commit()
```
