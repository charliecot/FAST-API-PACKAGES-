# SQLModel Session: Complete Guide to Fetching, Filtering, and Querying Database Records

## Table of Contents

1. Introduction to SQLModel Sessions
2. Understanding the Session Object
3. Understanding `select()`
4. Fetching All Records
5. Fetching a Single Record
6. Fetching by Primary Key with `session.get()`
7. Filtering Records with `where()`
8. Comparison Operators
9. Multiple Conditions with `and_()` and `or_()`
10. Negating Conditions with `not_()`
11. Filtering with `in_()` and `notin_()`
12. Filtering with `is_()` and `is_not()`
13. Pattern Matching with `like()` and `ilike()`
14. Filtering with `between()`
15. Ordering Records with `order_by()`
16. Limiting Records with `limit()`
17. Pagination with `offset()` and `limit()`
18. Removing Duplicate Results with `distinct()`
19. Selecting Specific Columns
20. Counting Records with `func.count()`
21. Aggregate Functions
22. Grouping with `group_by()`
23. Filtering Groups with `having()`
24. Combining Different SQL Clauses
25. Joining Related Tables
26. Working with Relationships
27. Loading Related Objects with `selectinload()` and `joinedload()`
28. Existence Checks
29. Subqueries
30. Working with Dates
31. Working with Enums
32. Executing Raw SQL
33. Understanding `.all()`, `.first()`, `.one()`, and `.one_or_none()`
34. Session Execution Methods
35. Handling Database Errors
36. Building Reusable Query Functions
37. Using Queries in FastAPI Endpoints
38. Complete CRUD Query Reference
39. Common Mistakes
40. Practice Exercises

---

# SECTION 1: INTRODUCTION TO SQLMODEL SESSIONS

## 1.1 What is a Session?

In SQLModel, a `Session` is an object that manages your interaction with a database.

Think of it as a working environment through which your Python application communicates with the database.

You use a session to:

- Retrieve database records.
- Filter records.
- Insert new records.
- Update existing records.
- Delete records.
- Execute SQL statements.
- Manage database transactions.

For example:

```python
from sqlmodel import Session, create_engine, select

engine = create_engine(
    "sqlite:///database.db",
    echo=True
)

with Session(engine) as session:
    statement = select(Band)

    bands = session.exec(statement).all()
```

Let's understand the components.

```python
with Session(engine) as session:
```

Creates a session connected to the database engine.

```python
statement = select(Band)
```

Constructs a database query.

```python
bands = session.exec(statement).all()
```

Executes the query and retrieves all matching records.

## 1.2 Creating a Database Session Dependency in FastAPI

A common approach is to create a reusable database dependency.

```python
from collections.abc import Generator
from sqlmodel import Session

def get_session() -> Generator[Session, None, None]:

    with Session(engine) as session:
        yield session
```

Then inject the session into your endpoint:

```python
from fastapi import Depends

@app.get("/bands")
def get_bands(
    session: Session = Depends(get_session)
):
    statement = select(Band)

    bands = session.exec(statement).all()

    return bands
```

FastAPI obtains the session through the dependency and closes it when the request finishes.

**Important:** Closing a session is not the same as committing a transaction.

---

# SECTION 2: UNDERSTANDING `select()`

The `select()` function constructs a SQL query.

Import it using:

```python
from sqlmodel import select
```

Suppose you have this model:

```python
class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str
    genre: str
```

You can construct different queries using `select()`.

### Select every band

```python
statement = select(Band)
```

Equivalent SQL:

```sql
SELECT *
FROM band;
```

### Select only the band name

```python
statement = select(Band.name)
```

Equivalent SQL:

```sql
SELECT name
FROM band;
```

### Select the band ID and name

```python
statement = select(
    Band.id,
    Band.name
)
```

Equivalent SQL:

```sql
SELECT id, name
FROM band;
```

### Select multiple columns

```python
statement = select(
    Band.id,
    Band.name,
    Band.genre
)
```

The important distinction:

```python
select(Band)
```

retrieves complete Band ORM objects.

Whereas:

```python
select(Band.name)
```

retrieves only the selected column values.

---

# SECTION 3: FETCHING ALL RECORDS

## 3.1 Using `.all()`

The `.all()` method retrieves all records returned by a query.

```python
statement = select(Band)

bands = session.exec(statement).all()
```

Equivalent SQL:

```sql
SELECT *
FROM band;
```

Suppose your database contains:

| id | name | genre |
|---|---|---|
| 1 | Metallica | Metal |
| 2 | Coldplay | Pop Rock |
| 3 | Linkin Park | Alternative |
| 4 | Iron Maiden | Heavy |

The result is a list of Band objects.

```python
[
    Band(id=1, name="Metallica", genre="Metal"),
    Band(id=2, name="Coldplay", genre="Pop Rock"),
    Band(id=3, name="Linkin Park", genre="Alternative"),
    Band(id=4, name="Iron Maiden", genre="Heavy")
]
```

Use `.all()` when you expect multiple records.

## 3.2 Using `.first()`

Retrieves the first result.

```python
statement = select(Band)

band = session.exec(statement).first()
```

Equivalent SQL conceptually:

```sql
SELECT *
FROM band
LIMIT 1;
```

However, `.first()` does not guarantee which record is returned unless you specify an ordering.

Better:

```python
statement = (
    select(Band)
    .order_by(Band.id)
)

band = session.exec(statement).first()
```

If no records exist, `.first()` returns `None`.

## 3.3 Using `.one()`

Returns exactly one result.

```python
statement = select(Band).where(
    Band.id == 1
)

band = session.exec(statement).one()
```

If there is no matching record, it raises an exception.

If multiple results are returned, it also raises an exception.

Use `.one()` when your query must return exactly one result.

## 3.4 Using `.one_or_none()`

Returns one result or `None`.

```python
statement = select(Band).where(
    Band.id == 100
)

band = session.exec(statement).one_or_none()
```

Possible results:

- One matching record: returns the Band object.
- No matching record: returns `None`.
- Multiple matching records: raises an exception.

This is useful when a record may or may not exist.

---

# SECTION 4: FETCHING A RECORD USING `session.get()`

One of the simplest ways to retrieve a record is `session.get()`.

Syntax:

```python
session.get(Model, primary_key)
```

Example:

```python
band = session.get(Band, 1)
```

This retrieves the Band whose primary key is `1`.

Equivalent SQL conceptually:

```sql
SELECT *
FROM band
WHERE id = 1;
```

Another example:

```python
album = session.get(Album, 5)
```

Retrieves the album with primary key `5`.

## Important: `session.get()` only retrieves by primary key

This is incorrect:

```python
band = session.get(Band, "Metal")
```

If you want to retrieve bands by genre, use `select()` and `where()`:

```python
statement = select(Band).where(
    Band.genre == "Metal"
)

bands = session.exec(statement).all()
```

### Difference between `get()` and `select()`

| Feature | `session.get()` | `select()` |
|---|---|---|
| Retrieves by primary key | Yes | Yes |
| Filters by name | No | Yes |
| Filters by genre | No | Yes |
| Supports multiple conditions | No | Yes |
| Supports ordering | No | Yes |
| Supports joins | No | Yes |
| Supports pagination | No | Yes |

Remember:

- Use `session.get()` for primary-key lookups.
- Use `select()` for general database queries.

---

# SECTION 5: FILTERING RECORDS WITH `where()`

The `where()` clause filters records according to a condition.

Syntax:

```python
statement = select(Model).where(condition)
```

For example:

```python
statement = select(Band).where(
    Band.genre == "Metal"
)

bands = session.exec(statement).all()
```

Equivalent SQL:

```sql
SELECT *
FROM band
WHERE genre = 'Metal';
```

The result contains only bands whose genre is Metal.

## 5.1 Filter by ID

```python
statement = select(Band).where(
    Band.id == 1
)

band = session.exec(statement).first()
```

SQL:

```sql
SELECT *
FROM band
WHERE id = 1;
```

## 5.2 Filter by name

```python
statement = select(Band).where(
    Band.name == "Metallica"
)
```

SQL:

```sql
SELECT *
FROM band
WHERE name = 'Metallica';
```

## 5.3 Filter by genre

```python
statement = select(Band).where(
    Band.genre == "Metal"
)
```

SQL:

```sql
SELECT *
FROM band
WHERE genre = 'Metal';
```

## 5.4 Filter by album title

```python
statement = select(Album).where(
    Album.title == "Master of Puppets"
)
```

SQL:

```sql
SELECT *
FROM album
WHERE title = 'Master of Puppets';
```

---

# SECTION 6: COMPARISON OPERATORS

SQLModel allows you to use Python comparison operators when constructing SQL expressions.

| Python operator | Meaning | SQL equivalent |
|---|---|---|
| `==` | Equal | `=` |
| `!=` | Not equal | `<>` |
| `>` | Greater than | `>` |
| `<` | Less than | `<` |
| `>=` | Greater than or equal | `>=` |
| `<=` | Less than or equal | `<=` |

## 6.1 Equal: `==`

```python
statement = select(Band).where(
    Band.genre == "Metal"
)
```

SQL:

```sql
SELECT *
FROM band
WHERE genre = 'Metal';
```

## 6.2 Not equal: `!=`

```python
statement = select(Band).where(
    Band.genre != "Metal"
)
```

SQL:

```sql
SELECT *
FROM band
WHERE genre <> 'Metal';
```

## 6.3 Greater than: `>`

Suppose albums have a release year represented by a date.

```python
statement = select(Album).where(
    Album.release_year > date(2000, 1, 1)
)
```

SQL conceptually:

```sql
SELECT *
FROM album
WHERE release_year > '2000-01-01';
```

## 6.4 Less than: `<`

```python
statement = select(Album).where(
    Album.release_year < date(2000, 1, 1)
)
```

## 6.5 Greater than or equal: `>=`

```python
statement = select(Album).where(
    Album.release_year >= date(2000, 1, 1)
)
```

## 6.6 Less than or equal: `<=`

```python
statement = select(Album).where(
    Album.release_year <= date(2000, 1, 1)
)
```

---

# SECTION 7: MULTIPLE CONDITIONS WITH `and_()`

Sometimes you need to filter records using multiple conditions that must all be true.

Use `and_()`.

Import:

```python
from sqlalchemy import and_
```

Example:

```python
statement = select(Band).where(
    and_(
        Band.genre == "Metal",
        Band.name == "Metallica"
    )
)
```

Equivalent SQL:

```sql
SELECT *
FROM band
WHERE genre = 'Metal'
AND name = 'Metallica';
```

Both conditions must be true.

## 7.1 Multiple conditions using comma-separated expressions

You do not always need `and_()`.

This is also valid:

```python
statement = select(Band).where(
    Band.genre == "Metal",
    Band.name == "Metallica"
)
```

SQLAlchemy combines these conditions using `AND`.

## 7.2 Multiple conditions using `&`

You can also use the bitwise AND operator:

```python
statement = select(Band).where(
    (Band.genre == "Metal") &
    (Band.name == "Metallica")
)
```

Important: Always put individual conditions inside parentheses when using `&`.

Do not confuse this with Python's logical `and`.

Avoid:

```python
statement = select(Band).where(
    Band.genre == "Metal" and
    Band.name == "Metallica"
)
```

Use `and_()` or `&` instead.

---

# SECTION 8: MULTIPLE CONDITIONS WITH `or_()`

Use `or_()` when at least one condition must be true.

Import:

```python
from sqlalchemy import or_
```

Example:

```python
statement = select(Band).where(
    or_(
        Band.genre == "Metal",
        Band.genre == "Pop Rock"
    )
)
```

Equivalent SQL:

```sql
SELECT *
FROM band
WHERE genre = 'Metal'
OR genre = 'Pop Rock';
```

A band is returned if either condition is true.

## 8.1 Using `|`

You can also use the bitwise OR operator:

```python
statement = select(Band).where(
    (Band.genre == "Metal") |
    (Band.genre == "Pop Rock")
)
```

Again, use parentheses around individual conditions.

---

# SECTION 9: NEGATING CONDITIONS WITH `not_()`

Use `not_()` to negate a SQL condition.

Import:

```python
from sqlalchemy import not_
```

Example:

```python
statement = select(Band).where(
    not_(Band.genre == "Metal")
)
```

SQL:

```sql
SELECT *
FROM band
WHERE NOT (genre = 'Metal');
```

You can also use:

```python
statement = select(Band).where(
    ~(Band.genre == "Metal")
)
```

This excludes Metal bands.

---

# SECTION 10: FILTERING WITH `in_()` AND `notin_()`

The `in_()` method checks whether a column's value exists in a supplied collection.

This is particularly useful when filtering against multiple possible values.

## 10.1 Using `in_()`

```python
genres = [
    "Metal",
    "Pop Rock",
    "Alternative"
]

statement = select(Band).where(
    Band.genre.in_(genres)
)

bands = session.exec(statement).all()
```

SQL:

```sql
SELECT *
FROM band
WHERE genre IN (
    'Metal',
    'Pop Rock',
    'Alternative'
);
```

A band is returned if its genre matches any value in the list.

## 10.2 Using `notin_()`

You can exclude multiple values.

```python
statement = select(Band).where(
    Band.genre.notin_([
        "Metal",
        "Heavy"
    ])
)
```

SQL:

```sql
SELECT *
FROM band
WHERE genre NOT IN ('Metal', 'Heavy');
```

## 10.3 Using an empty list

Be careful when your list comes from user input.

```python
genres = []
```

An empty `IN` collection matches no records.

Consider checking whether the collection is empty before constructing the query.

---

# SECTION 11: FILTERING NULL VALUES

SQL databases use `NULL` to represent missing or unknown values.

Python uses `None`.

You should not compare SQL NULL values using ordinary equality.

Incorrect:

```python
statement = select(Album).where(
    Album.band_id == None
)
```

Correct:

```python
statement = select(Album).where(
    Album.band_id.is_(None)
)
```

Equivalent SQL:

```sql
SELECT *
FROM album
WHERE band_id IS NULL;
```

## 11.1 `is_not()`

Find albums whose `band_id` is not NULL.

```python
statement = select(Album).where(
    Album.band_id.is_not(None)
)
```

SQL:

```sql
SELECT *
FROM album
WHERE band_id IS NOT NULL;
```

Remember:

| SQLModel expression | SQL |
|---|---|
| `column.is_(None)` | `IS NULL` |
| `column.is_not(None)` | `IS NOT NULL` |

---

# SECTION 12: PATTERN MATCHING WITH `like()` AND `ilike()`

Pattern matching helps you find records when you do not know their complete values.

## 12.1 `like()`

The `like()` method performs pattern matching.

```python
statement = select(Band).where(
    Band.name.like("Metal%")
)
```

SQL:

```sql
SELECT *
FROM band
WHERE name LIKE 'Metal%';
```

The `%` wildcard represents zero or more characters.

For example:

- Metallica
- Metal Church
- Metallica Tribute

All can match the pattern.

### Common SQL wildcards

| Wildcard | Meaning |
|---|---|
| `%` | Zero or more characters |
| `_` | Exactly one character |

Example:

```python
statement = select(Band).where(
    Band.name.like("_etallica")
)
```

The underscore represents one character.

## 12.2 `ilike()`

`ilike()` performs case-insensitive pattern matching on databases that support it natively.

```python
statement = select(Band).where(
    Band.name.ilike("metal%")
)
```

This can match:

- Metallica
- METALLICA
- metallica

PostgreSQL supports `ILIKE` directly.

SQL:

```sql
SELECT *
FROM band
WHERE name ILIKE 'metal%';
```

On databases without native `ILIKE`, SQLAlchemy may emulate case-insensitive matching.

## 12.3 Searching for a word anywhere

```python
statement = select(Band).where(
    Band.name.ilike("%metal%")
)
```

The `%` characters on both sides allow the word to appear anywhere in the name.

## 12.4 Searching by the beginning of a string

```python
statement = select(Band).where(
    Band.name.startswith("Metal")
)
```

## 12.5 Searching by the ending of a string

```python
statement = select(Band).where(
    Band.name.endswith("lica")
)
```

## 12.6 Searching for a substring

```python
statement = select(Band).where(
    Band.name.contains("tal")
)
```

These convenience methods are useful, but remember that user-provided wildcard characters can affect the matching pattern. If literal matching is required, consider escaping wildcard characters.

---

# SECTION 13: FILTERING WITH `between()`

The `between()` method checks whether a value falls within a range.

Example:

```python
from datetime import date

statement = select(Album).where(
    Album.release_year.between(
        date(1980, 1, 1),
        date(1990, 12, 31)
    )
)
```

SQL:

```sql
SELECT *
FROM album
WHERE release_year BETWEEN
'1980-01-01' AND '1990-12-31';
```

`BETWEEN` includes both boundaries.

For example, a release date of `1980-01-01` is included.

You can use it for:

- Date ranges.
- Numeric ranges.
- Prices.
- Ages.
- Scores.

---

# SECTION 14: ORDERING RECORDS WITH `order_by()`

The `order_by()` clause controls the order in which records are returned.

## 14.1 Ascending order

```python
statement = select(Band).order_by(
    Band.name
)
```

SQL:

```sql
SELECT *
FROM band
ORDER BY name ASC;
```

Ascending order is the default.

You can explicitly use:

```python
from sqlalchemy import asc

statement = select(Band).order_by(
    asc(Band.name)
)
```

## 14.2 Descending order

```python
from sqlalchemy import desc

statement = select(Band).order_by(
    desc(Band.name)
)
```

SQL:

```sql
SELECT *
FROM band
ORDER BY name DESC;
```

## 14.3 Ordering by multiple columns

```python
statement = select(Band).order_by(
    Band.genre,
    Band.name
)
```

SQL:

```sql
SELECT *
FROM band
ORDER BY genre ASC, name ASC;
```

## 14.4 Different directions for different columns

```python
statement = select(Band).order_by(
    asc(Band.genre),
    desc(Band.name)
)
```

SQL:

```sql
SELECT *
FROM band
ORDER BY genre ASC, name DESC;
```

---

# SECTION 15: LIMITING RECORDS WITH `limit()`

The `limit()` clause controls the maximum number of records returned.

Example:

```python
statement = select(Band).limit(5)

bands = session.exec(statement).all()
```

SQL:

```sql
SELECT *
FROM band
LIMIT 5;
```

This returns at most five records.

## Combining `limit()` with `order_by()`

```python
statement = (
    select(Band)
    .order_by(Band.id)
    .limit(5)
)
```

This returns the first five bands ordered by ID.

For predictable results, use `order_by()` whenever you use pagination or limits.

---

# SECTION 16: PAGINATION WITH `offset()` AND `limit()`

Pagination allows an application to divide a large collection of records into smaller pages.

Suppose you have 100 bands.

Instead of retrieving all 100 at once, you can retrieve 10 per page.

Two important methods:

- `limit()`: how many records to retrieve.
- `offset()`: how many records to skip.

## 16.1 Understanding `offset()`

```python
statement = select(Band).offset(10)
```

SQL:

```sql
SELECT *
FROM band
OFFSET 10;
```

This skips the first ten records.

## 16.2 Combining `offset()` and `limit()`

```python
statement = (
    select(Band)
    .offset(10)
    .limit(10)
)
```

SQL:

```sql
SELECT *
FROM band
LIMIT 10
OFFSET 10;
```

This skips ten records and retrieves the next ten.

## 16.3 Calculate the offset dynamically

Suppose the client requests:

```text
page = 3
page_size = 10
```

The offset is:

```python
offset = (page - 1) * page_size
```

Therefore:

```python
offset = (3 - 1) * 10
offset = 20
```

The query:

```python
statement = (
    select(Band)
    .order_by(Band.id)
    .offset(offset)
    .limit(page_size)
)

bands = session.exec(statement).all()
```

## 16.4 Complete FastAPI pagination endpoint

```python
from fastapi import Query

@app.get("/bands")
def get_bands(
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=10, ge=1, le=100),
    session: Session = Depends(get_session)
):

    offset = (page - 1) * page_size

    statement = (
        select(Band)
        .order_by(Band.id)
        .offset(offset)
        .limit(page_size)
    )

    bands = session.exec(statement).all()

    return bands
```

Example requests:

```http
GET /bands?page=1&page_size=10
GET /bands?page=2&page_size=10
GET /bands?page=3&page_size=10
```

---

# SECTION 17: REMOVING DUPLICATES WITH `distinct()`

`distinct()` removes duplicate values from a query result.

Suppose your database contains:

| id | name | genre |
|---|---|---|
| 1 | Metallica | Metal |
| 2 | Iron Maiden | Metal |
| 3 | Coldplay | Pop Rock |
| 4 | Linkin Park | Alternative |

You want to retrieve unique genres.

```python
statement = select(Band.genre).distinct()

genres = session.exec(statement).all()
```

SQL:

```sql
SELECT DISTINCT genre
FROM band;
```

Possible result:

```python
[
    "Metal",
    "Pop Rock",
    "Alternative"
]
```

You can also select distinct combinations of columns:

```python
statement = select(
    Band.genre,
    Band.name
).distinct()
```

This removes duplicate combinations of the selected values.

---

# SECTION 18: SELECTING SPECIFIC COLUMNS

You do not always need complete ORM objects.

Sometimes you only need a few columns.

## 18.1 Select only names

```python
statement = select(Band.name)

names = session.exec(statement).all()
```

Possible result:

```python
[
    "Metallica",
    "Coldplay",
    "Linkin Park"
]
```

## 18.2 Select IDs and names

```python
statement = select(
    Band.id,
    Band.name
)

results = session.exec(statement).all()
```

With multiple selected columns, the results are row-like objects.

You can access their values:

```python
for row in results:
    print(row.id)
    print(row.name)
```

You can also unpack them:

```python
for band_id, name in results:
    print(band_id, name)
```

## 18.3 Select a single column and filter it

```python
statement = (
    select(Band.name)
    .where(Band.genre == "Metal")
)

names = session.exec(statement).all()
```

This retrieves only the names of Metal bands.

**Important:** Selecting columns instead of complete ORM objects can reduce the amount of data transferred when you do not need the complete models.

---

# SECTION 19: COUNTING RECORDS WITH `func.count()`

SQL has an aggregate function called `COUNT()`.

It counts records or non-NULL values.

In SQLModel, import `func`:

```python
from sqlalchemy import func
```

## 19.1 Count all bands

```python
statement = select(
    func.count()
).select_from(Band)

total = session.exec(statement).one()
```

SQL:

```sql
SELECT COUNT(*)
FROM band;
```

## 19.2 Count bands with a particular genre

```python
statement = (
    select(func.count())
    .select_from(Band)
    .where(Band.genre == "Metal")
)

total = session.exec(statement).one()
```

SQL:

```sql
SELECT COUNT(*)
FROM band
WHERE genre = 'Metal';
```

## 19.3 Count albums

```python
statement = select(
    func.count()
).select_from(Album)

total_albums = session.exec(statement).one()
```

SQL:

```sql
SELECT COUNT(*)
FROM album;
```

## 19.4 Difference between `COUNT(*)` and `COUNT(column)`

```sql
SELECT COUNT(*)
FROM album;
```

Counts rows.

```sql
SELECT COUNT(band_id)
FROM album;
```

Counts rows where `band_id` is not NULL.

This distinction matters when a column allows NULL values.

---

# SECTION 20: AGGREGATE FUNCTIONS

Aggregate functions calculate values from multiple rows.

Common SQL aggregate functions:

| Function | Purpose |
|---|---|
| `COUNT()` | Counts records |
| `SUM()` | Adds values |
| `AVG()` | Calculates the average |
| `MIN()` | Returns the minimum |
| `MAX()` | Returns the maximum |

Import:

```python
from sqlalchemy import func
```

## 20.1 `COUNT()`

```python
statement = select(
    func.count()
).select_from(Album)
```

## 20.2 `MIN()`

Find the earliest album release date:

```python
statement = select(
    func.min(Album.release_year)
)

earliest = session.exec(statement).one()
```

SQL:

```sql
SELECT MIN(release_year)
FROM album;
```

## 20.3 `MAX()`

Find the latest album release date:

```python
statement = select(
    func.max(Album.release_year)
)

latest = session.exec(statement).one()
```

## 20.4 `AVG()`

Suppose you have a numeric column called `rating`.

```python
statement = select(
    func.avg(Album.rating)
)
```

This calculates the average rating.

## 20.5 `SUM()`

Suppose you have a numeric column called `sales`.

```python
statement = select(
    func.sum(Album.sales)
)
```

This calculates total sales.

Note: `rating` and `sales` are illustrative columns; add them to your model before using those examples.

---

# SECTION 21: GROUPING RECORDS WITH `group_by()`

`GROUP BY` groups records that share the same value.

Suppose you want to count how many bands belong to each genre.

```python
statement = (
    select(
        Band.genre,
        func.count(Band.id)
    )
    .group_by(Band.genre)
)

results = session.exec(statement).all()
```

SQL:

```sql
SELECT genre, COUNT(id)
FROM band
GROUP BY genre;
```

Possible result:

```python
[
    ("Metal", 5),
    ("Pop Rock", 3),
    ("Alternative", 4),
    ("Heavy", 2)
]
```

## 21.1 Group by multiple columns

```python
statement = (
    select(
        Band.genre,
        Band.name,
        func.count(Album.id)
    )
    .join(Album)
    .group_by(
        Band.genre,
        Band.name
    )
)
```

This groups the joined records by genre and band name.

When selecting ordinary columns alongside aggregate functions, make sure the non-aggregated columns are included in `GROUP BY` where required.

---

# SECTION 22: FILTERING GROUPS WITH `having()`

`WHERE` and `HAVING` are different.

- `WHERE` filters individual rows before grouping.
- `HAVING` filters groups after grouping.

Example: Find genres that have more than two bands.

```python
statement = (
    select(
        Band.genre,
        func.count(Band.id).label("total_bands")
    )
    .group_by(Band.genre)
    .having(func.count(Band.id) > 2)
)

results = session.exec(statement).all()
```

SQL:

```sql
SELECT genre, COUNT(id) AS total_bands
FROM band
GROUP BY genre
HAVING COUNT(id) > 2;
```

## Combining WHERE and HAVING

```python
statement = (
    select(
        Band.genre,
        func.count(Band.id)
    )
    .where(Band.genre != "Heavy")
    .group_by(Band.genre)
    .having(func.count(Band.id) > 1)
)
```

The query:

1. Excludes Heavy bands.
2. Groups the remaining records by genre.
3. Returns only groups containing more than one band.

---

# SECTION 23: COMBINING DIFFERENT SQL CLAUSES

You can combine several query methods to construct complex queries.

Example:

Find the first five Metal bands, ordered alphabetically.

```python
statement = (
    select(Band)
    .where(Band.genre == "Metal")
    .order_by(Band.name)
    .limit(5)
)

bands = session.exec(statement).all()
```

SQL:

```sql
SELECT *
FROM band
WHERE genre = 'Metal'
ORDER BY name ASC
LIMIT 5;
```

Another example:

Find albums released between 1980 and 2000, ordered from newest to oldest.

```python
statement = (
    select(Album)
    .where(
        Album.release_year.between(
            date(1980, 1, 1),
            date(2000, 12, 31)
        )
    )
    .order_by(Album.release_year.desc())
)

albums = session.exec(statement).all()
```

You can combine:

- `select()`
- `where()`
- `and_()`
- `or_()`
- `order_by()`
- `limit()`
- `offset()`
- `distinct()`
- `group_by()`
- `having()`
- `join()`

The order in which you chain these methods should follow the SQL query structure and the meaning of the query.

---

# SECTION 24: JOINING RELATED TABLES

A JOIN allows you to retrieve information from multiple related tables.

Remember our models:

```python
class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str
    genre: str

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str
    release_year: date

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: Band = Relationship(
        back_populates="albums"
    )
```

The relationship is:

```text
Band
----
id
name
genre

       1
       |
       |
       | One-to-Many
       |
       *
Album
-----
id
title
release_year
band_id
```

The `album.band_id` column references `band.id`.

## 24.1 Inner JOIN

Use `.join()`.

```python
statement = select(
    Band,
    Album
).join(Album)

results = session.exec(statement).all()
```

SQL:

```sql
SELECT band.*, album.*
FROM band
INNER JOIN album
ON band.id = album.band_id;
```

An INNER JOIN returns matching records from both tables.

A band without albums will not appear in this result.

## 24.2 JOIN using an explicit condition

```python
statement = (
    select(Band, Album)
    .join(
        Album,
        Band.id == Album.band_id
    )
)
```

Here, you explicitly specify the joining condition.

## 24.3 LEFT OUTER JOIN

A LEFT OUTER JOIN returns all bands, including bands without albums.

```python
statement = (
    select(Band, Album)
    .outerjoin(Album)
)

results = session.exec(statement).all()
```

SQL:

```sql
SELECT band.*, album.*
FROM band
LEFT OUTER JOIN album
ON band.id = album.band_id;
```

The difference:

| INNER JOIN | LEFT OUTER JOIN |
|---|---|
| Returns matching records | Returns all left-side records |
| Excludes bands without albums | Includes bands without albums |
| Requires a matching right-side record | Right-side values can be NULL |

## 24.4 JOIN and WHERE

Find all albums belonging to Metal bands.

```python
statement = (
    select(Album)
    .join(Band)
    .where(Band.genre == "Metal")
)

albums = session.exec(statement).all()
```

SQL:

```sql
SELECT album.*
FROM album
INNER JOIN band
ON band.id = album.band_id
WHERE band.genre = 'Metal';
```

## 24.5 JOIN and ORDER BY

```python
statement = (
    select(Album)
    .join(Band)
    .where(Band.name == "Metallica")
    .order_by(Album.release_year)
)

albums = session.exec(statement).all()
```

This retrieves Metallica's albums ordered by release date.

## 24.6 JOIN and COUNT

Count the albums belonging to each band.

```python
statement = (
    select(
        Band.name,
        func.count(Album.id).label("total_albums")
    )
    .outerjoin(Album)
    .group_by(Band.id, Band.name)
)

results = session.exec(statement).all()
```

Why use `outerjoin()`?

Because it also includes bands that have zero albums.

Why count `Album.id` rather than `COUNT(*)`?

Because a LEFT JOIN produces a row for a band even when there is no matching album. `COUNT(Album.id)` ignores the NULL album ID, giving zero.

---

# SECTION 25: WORKING WITH RELATIONSHIPS

SQLModel relationships allow you to navigate between related ORM objects.

For example:

```python
band = session.get(Band, 1)
```

If the band exists, you can access:

```python
band.albums
```

This accesses the albums associated with that band.

Similarly:

```python
album = session.get(Album, 1)
```

You can access:

```python
album.band
```

This accesses the parent band.

## 25.1 Retrieve a band and its albums

```python
band = session.get(Band, 1)

if band is not None:
    for album in band.albums:
        print(album.title)
```

Depending on the relationship loading configuration, accessing `band.albums` may trigger another SQL query.

This is called lazy loading.

## 25.2 Retrieve an album and its band

```python
album = session.get(Album, 1)

if album is not None:
    print(album.band.name)
```

This accesses the associated band.

## 25.3 Important distinction

These are not the same:

```python
Album.band_id
```

This is a foreign-key column containing the parent's ID.

```python
Album.band
```

This is an ORM relationship attribute containing the associated Band object.

The foreign key stores the database relationship.

The ORM relationship gives you convenient access to the related Python object.

---

# SECTION 26: EAGER LOADING WITH `selectinload()`

When retrieving related records, SQLAlchemy supports different loading strategies.

One useful strategy is `selectinload()`.

Import:

```python
from sqlalchemy.orm import selectinload
```

Example:

```python
statement = (
    select(Band)
    .options(selectinload(Band.albums))
)

bands = session.exec(statement).all()
```

This retrieves bands and loads their related albums using an additional SELECT query.

Conceptually:

First query:

```sql
SELECT *
FROM band;
```

Second query:

```sql
SELECT *
FROM album
WHERE band_id IN (1, 2, 3, 4);
```

This is useful when retrieving multiple parent records with their child collections.

## 26.1 Filter bands and load their albums

```python
statement = (
    select(Band)
    .where(Band.genre == "Metal")
    .options(selectinload(Band.albums))
)

bands = session.exec(statement).all()
```

Now you can iterate:

```python
for band in bands:
    print(band.name)

    for album in band.albums:
        print(album.title)
```

## 26.2 Why use `selectinload()`?

Without eager loading, accessing each band's albums can trigger an additional query for each band.

For example, retrieving ten bands and then accessing every band's albums may result in many queries.

This is commonly known as the N+1 query problem.

`selectinload()` reduces this problem by loading related collections in batches.

---

# SECTION 27: USING `joinedload()`

Another eager-loading strategy is `joinedload()`.

Import:

```python
from sqlalchemy.orm import joinedload
```

Example:

```python
statement = (
    select(Album)
    .options(joinedload(Album.band))
)

albums = session.exec(statement).all()
```

This loads albums together with their associated bands through a SQL JOIN.

Conceptually:

```sql
SELECT album.*, band.*
FROM album
LEFT OUTER JOIN band
ON band.id = album.band_id;
```

## 27.1 When should you use each?

| Strategy | How it loads | Common use |
|---|---|---|
| Lazy loading | Loads related data when accessed | Simple object navigation |
| `selectinload()` | Uses a separate SELECT for related records | Parent with many children |
| `joinedload()` | Uses a JOIN | Related single objects or suitable collections |

For one-to-many relationships such as Band to Album, `selectinload()` is often a good starting choice.

**Important:** When using `joinedload()` for collection relationships, SQLAlchemy results may contain repeated parent rows. With SQLAlchemy's standard `Session`, use `.unique()` where required before consuming ORM entities. SQLModel's `Session.exec()` uses SQLAlchemy result behavior, so verify the result type and loading pattern for your installed version.

---

# SECTION 28: EXISTENCE CHECKS

Sometimes you do not need the actual records. You only need to know whether a matching record exists.

SQL provides `EXISTS`.

Import:

```python
from sqlalchemy import exists
```

Example:

```python
statement = select(
    exists().where(Band.name == "Metallica")
)

result = session.exec(statement).one()
```

The result is a boolean.

Conceptually, SQL:

```sql
SELECT EXISTS (
    SELECT 1
    FROM band
    WHERE name = 'Metallica'
);
```

Possible results:

```python
True
```

or:

```python
False
```

You can also use this for albums:

```python
statement = select(
    exists().where(Album.title == "Master of Puppets")
)

album_exists = session.exec(statement).one()
```

This is useful for existence checks without loading an entire ORM object.

---

# SECTION 29: SUBQUERIES

A subquery is a query nested inside another query.

It is useful when one query depends on the results of another query.

Example: Find bands that have at least one album.

```python
album_band_ids = (
    select(Album.band_id)
    .where(Album.band_id.is_not(None))
)

statement = select(Band).where(
    Band.id.in_(album_band_ids)
)

bands = session.exec(statement).all()
```

SQL conceptually:

```sql
SELECT *
FROM band
WHERE id IN (
    SELECT band_id
    FROM album
    WHERE band_id IS NOT NULL
);
```

## 29.1 Find bands without albums

```python
album_band_ids = select(Album.band_id)

statement = select(Band).where(
    Band.id.notin_(album_band_ids)
)

bands = session.exec(statement).all()
```

A LEFT OUTER JOIN with `IS NULL` is another common approach:

```python
statement = (
    select(Band)
    .outerjoin(Album)
    .where(Album.id.is_(None))
)
```

The LEFT JOIN approach is particularly useful for finding parents without children.

---

# SECTION 30: WORKING WITH DATES

Your Album model uses:

```python
release_year: date
```

This means the database field represents a date, not just a four-digit year.

Import:

```python
from datetime import date
```

## 30.1 Find albums released after a date

```python
statement = select(Album).where(
    Album.release_year > date(2000, 1, 1)
)
```

## 30.2 Find albums released before a date

```python
statement = select(Album).where(
    Album.release_year < date(2000, 1, 1)
)
```

## 30.3 Find albums released within a date range

```python
statement = select(Album).where(
    Album.release_year.between(
        date(1980, 1, 1),
        date(1990, 12, 31)
    )
)
```

## 30.4 Find albums released in a particular year

A date column is not directly the same as a year column.

You can use a date range:

```python
statement = select(Album).where(
    Album.release_year >= date(1986, 1, 1),
    Album.release_year < date(1987, 1, 1)
)
```

This finds albums released during 1986.

For PostgreSQL-specific queries, you can also use date extraction functions, but a range is often straightforward and can work well with indexes.

---

# SECTION 31: WORKING WITH ENUMS

Suppose you define:

```python
from enum import Enum

class GenreURLChoices(str, Enum):

    POP_ROCK = "Pop Rock"
    METAL = "Metal"
    ALTERNATIVE = "Alternative"
    HEAVY = "Heavy"
```

And your model:

```python
class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str
    genre: GenreURLChoices
```

You can filter using the enum:

```python
statement = select(Band).where(
    Band.genre == GenreURLChoices.METAL
)
```

In your FastAPI endpoint:

```python
@app.get("/bands/genre/{genre}")
def bands_for_genre(
    genre: GenreURLChoices,
    session: Session = Depends(get_session)
):

    statement = select(Band).where(
        Band.genre == genre
    )

    return session.exec(statement).all()
```

Example request:

```http
GET /bands/genre/Metal
```

FastAPI validates the path parameter against the enum.

Invalid values will result in a validation error.

---

# SECTION 32: EXECUTING RAW SQL

Sometimes you need to execute SQL directly.

SQLAlchemy provides `text()` for SQL statements.

Import:

```python
from sqlalchemy import text
```

Example:

```python
statement = text(
    "SELECT * FROM band"
)

result = session.execute(statement).all()
```

SQL with parameters:

```python
statement = text(
    "SELECT * FROM band WHERE genre = :genre"
)

result = session.execute(
    statement,
    {"genre": "Metal"}
).all()
```

Notice:

```python
:genre
```

This is a named parameter.

You provide its value separately:

```python
{"genre": "Metal"}
```

Never build SQL by directly concatenating untrusted user input.

Dangerous:

```python
genre = user_input

statement = text(
    f"SELECT * FROM band WHERE genre = '{genre}'"
)
```

Safe:

```python
statement = text(
    "SELECT * FROM band WHERE genre = :genre"
)

result = session.execute(
    statement,
    {"genre": genre}
)
```

For ordinary ORM queries, prefer `select()` and SQLModel expressions.

---

# SECTION 33: UNDERSTANDING RESULT METHODS

These methods are commonly used to consume query results.

| Method | Purpose | If no result |
|---|---|---|
| `.all()` | Retrieves all results | Empty list |
| `.first()` | Retrieves first result | `None` |
| `.one()` | Requires exactly one result | Raises exception |
| `.one_or_none()` | Allows zero or one result | `None` |
| `.fetchall()` | Fetches all rows from a lower-level result | Empty collection |
| `.fetchone()` | Fetches one row from a lower-level result | `None` |

For SQLModel ORM queries, you will frequently use:

```python
session.exec(statement).all()
```

```python
session.exec(statement).first()
```

```python
session.exec(statement).one()
```

```python
session.exec(statement).one_or_none()
```

Remember that `.one()` and `.one_or_none()` enforce result cardinality. They do not silently return the first record when multiple records match.

---

# SECTION 34: SESSION EXECUTION METHODS

There are several important methods associated with the session.

## 34.1 `session.exec()`

SQLModel's convenient execution method.

```python
statement = select(Band)

bands = session.exec(statement).all()
```

It is especially convenient for ORM queries and scalar selections.

## 34.2 `session.execute()`

The standard SQLAlchemy execution method.

```python
statement = select(Band)

result = session.execute(statement)
```

For a standard SQLAlchemy ORM SELECT, you can retrieve entities with:

```python
bands = result.scalars().all()
```

For SQLModel, `session.exec()` is generally the simpler option for ordinary model queries.

## 34.3 `session.add()`

Adds an object to the session.

```python
band = Band(
    name="Metallica",
    genre=GenreURLChoices.METAL
)

session.add(band)
```

This does not necessarily execute the INSERT immediately.

## 34.4 `session.add_all()`

Adds multiple objects.

```python
bands = [
    Band(name="Metallica", genre=GenreURLChoices.METAL),
    Band(name="Coldplay", genre=GenreURLChoices.POP_ROCK)
]

session.add_all(bands)
```

## 34.5 `session.flush()`

Sends pending changes to the database within the current transaction.

```python
session.add(band)

session.flush()

print(band.id)
```

Useful when you need the generated primary key before committing.

## 34.6 `session.commit()`

Commits the transaction.

```python
session.commit()
```

This makes the transaction's changes permanent, subject to database transaction guarantees.

## 34.7 `session.refresh()`

Reloads attributes of an ORM object from the database.

```python
session.refresh(band)
```

It expects an ORM object, not a list.

Incorrect:

```python
session.refresh(band.albums)
```

For reloading a relationship collection, execute an appropriate query with eager loading.

## 34.8 `session.delete()`

Marks an ORM object for deletion.

```python
band = session.get(Band, 1)

if band is not None:
    session.delete(band)
    session.commit()
```

## 34.9 `session.rollback()`

Rolls back the current transaction.

```python
try:
    session.add(band)
    session.commit()

except Exception:
    session.rollback()
    raise
```

---

# SECTION 35: HANDLING DATABASE ERRORS

Database operations can fail because of:

- Foreign-key violations.
- Unique constraints.
- NOT NULL constraints.
- Invalid SQL.
- Connection failures.
- Transaction errors.

Example:

```python
from sqlalchemy.exc import IntegrityError

try:

    session.add(band)
    session.commit()

except IntegrityError:

    session.rollback()

    raise HTTPException(
        status_code=400,
        detail="Database integrity constraint violated."
    )
```

`IntegrityError` is useful for errors involving database integrity constraints.

For unexpected exceptions, log the error and return a generic response rather than exposing database internals.

Important: After a failed transaction, call `rollback()` before attempting further database operations with that session.

---

# SECTION 36: BUILDING REUSABLE QUERY FUNCTIONS

You do not need to put every query directly inside a FastAPI endpoint.

You can create reusable functions.

Example:

```python
def get_all_bands(session: Session) -> list[Band]:

    statement = select(Band)

    return session.exec(statement).all()
```

Get a band by ID:

```python
def get_band_by_id(
    band_id: int,
    session: Session
) -> Band | None:

    return session.get(Band, band_id)
```

Get bands by genre:

```python
def get_bands_by_genre(
    genre: GenreURLChoices,
    session: Session
) -> list[Band]:

    statement = select(Band).where(
        Band.genre == genre
    )

    return session.exec(statement).all()
```

Search by name:

```python
def search_bands(
    name: str,
    session: Session
) -> list[Band]:

    statement = select(Band).where(
        Band.name.ilike(f"%{name}%")
    )

    return session.exec(statement).all()
```

For user-provided search strings, consider how wildcard characters should be handled if you intend literal substring matching.

---

# SECTION 37: USING QUERIES IN FASTAPI ENDPOINTS

Let's put the concepts together.

## 37.1 Retrieve all bands

```python
@app.get("/bands", response_model=list[BandRead])
def get_bands(
    session: Session = Depends(get_session)
):

    statement = (
        select(Band)
        .options(selectinload(Band.albums))
    )

    return session.exec(statement).all()
```

## 37.2 Retrieve one band by ID

```python
@app.get("/bands/{band_id}")
def get_band(
    band_id: int,
    session: Session = Depends(get_session)
):

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found."
        )

    return band
```

## 37.3 Retrieve bands by genre

```python
@app.get("/bands/genre/{genre}")
def bands_for_genre(
    genre: GenreURLChoices,
    session: Session = Depends(get_session)
):

    statement = (
        select(Band)
        .where(Band.genre == genre)
        .options(selectinload(Band.albums))
    )

    return session.exec(statement).all()
```

## 37.4 Search bands by name

```python
@app.get("/bands/search/{name}")
def search_bands(
    name: str,
    session: Session = Depends(get_session)
):

    statement = select(Band).where(
        Band.name.ilike(f"%{name}%")
    )

    return session.exec(statement).all()
```

## 37.5 Retrieve albums for a particular band

```python
@app.get("/bands/{band_id}/albums")
def get_band_albums(
    band_id: int,
    session: Session = Depends(get_session)
):

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found."
        )

    statement = select(Album).where(
        Album.band_id == band_id
    )

    return session.exec(statement).all()
```

## 37.6 Retrieve albums released within a date range

```python
@app.get("/albums")
def get_albums(
    start_date: date,
    end_date: date,
    session: Session = Depends(get_session)
):

    statement = (
        select(Album)
        .where(
            Album.release_year >= start_date,
            Album.release_year <= end_date
        )
        .order_by(Album.release_year.desc())
    )

    return session.exec(statement).all()
```

---

# SECTION 38: COMPLETE CRUD QUERY REFERENCE

| Operation | SQLModel code |
|---|---|
| Get all records | `session.exec(select(Band)).all()` |
| Get by primary key | `session.get(Band, band_id)` |
| Filter | `select(Band).where(Band.genre == genre)` |
| Multiple AND conditions | `where(condition1, condition2)` |
| OR conditions | `where(or_(condition1, condition2))` |
| NOT condition | `where(not_(condition))` |
| IN | `where(Band.genre.in_(genres))` |
| NOT IN | `where(Band.genre.notin_(genres))` |
| IS NULL | `where(Album.band_id.is_(None))` |
| IS NOT NULL | `where(Album.band_id.is_not(None))` |
| LIKE | `where(Band.name.like("Metal%"))` |
| ILIKE | `where(Band.name.ilike("metal%"))` |
| BETWEEN | `where(Album.release_year.between(start, end))` |
| Ascending | `order_by(Band.name)` |
| Descending | `order_by(Band.name.desc())` |
| Limit | `limit(10)` |
| Offset | `offset(20)` |
| Distinct | `select(Band.genre).distinct()` |
| Count | `select(func.count()).select_from(Band)` |
| Group by | `group_by(Band.genre)` |
| Having | `having(func.count(Band.id) > 2)` |
| Inner join | `select(Band).join(Album)` |
| Left join | `select(Band).outerjoin(Album)` |
| Eager loading | `options(selectinload(Band.albums))` |
| Existence | `select(exists().where(...))` |

---

# SECTION 39: COMMON MISTAKES

## Mistake 1: Using `session.get()` to filter by genre

Incorrect:

```python
session.get(Band, "Metal")
```

Correct:

```python
session.exec(
    select(Band).where(Band.genre == "Metal")
).all()
```

## Mistake 2: Forgetting `.all()`

Incorrect:

```python
bands = session.exec(select(Band))
```

This gives you a result object, not the materialized list of bands.

Correct:

```python
bands = session.exec(select(Band)).all()
```

## Mistake 3: Using Python `and` instead of SQL expressions

Incorrect:

```python
where(
    Band.genre == "Metal" and
    Band.name == "Metallica"
)
```

Correct:

```python
where(
    Band.genre == "Metal",
    Band.name == "Metallica"
)
```

## Mistake 4: Using `== None`

Avoid:

```python
where(Album.band_id == None)
```

Prefer:

```python
where(Album.band_id.is_(None))
```

## Mistake 5: Forgetting to order paginated results

Avoid:

```python
statement = select(Band).limit(10).offset(10)
```

Prefer:

```python
statement = (
    select(Band)
    .order_by(Band.id)
    .limit(10)
    .offset(10)
)
```

## Mistake 6: Returning relationships without loading them

If your response contains nested albums, make sure the relationship is available before serialization.

Example:

```python
statement = (
    select(Band)
    .options(selectinload(Band.albums))
)

bands = session.exec(statement).all()
```

Also define an appropriate response schema.

## Mistake 7: Confusing a foreign key with a relationship

```python
Album.band_id
```

is a database column.

```python
Album.band
```

is a relationship to a Band object.

## Mistake 8: Forgetting to rollback

After a database transaction fails:

```python
session.rollback()
```

Do this before continuing to use that session.

## Mistake 9: Using `async def` with blocking database sessions

If you use the regular synchronous SQLModel `Session`, prefer:

```python
def get_bands(
    session: Session = Depends(get_session)
):
    ...
```

For asynchronous database access, use an asynchronous SQLAlchemy engine and `AsyncSession` consistently instead.

## Mistake 10: Confusing `commit()` and `refresh()`

```python
session.commit()
```

Persists the transaction.

```python
session.refresh(band)
```

Reloads an object's database-backed attributes.

They perform different jobs.

---

# SECTION 40: PRACTICE EXERCISES

Use your Band and Album models to practise the following queries.

## Beginner exercises

1. Retrieve all bands.
2. Retrieve all albums.
3. Retrieve a band using its ID.
4. Retrieve an album using its ID.
5. Find all Metal bands.
6. Find all bands whose names equal Metallica.
7. Find albums released after 1990.
8. Find albums released before 1980.
9. Retrieve the first five bands.
10. Retrieve bands ordered alphabetically.

## Intermediate exercises

11. Find bands belonging to Metal or Heavy genres.
12. Find bands that are not Metal.
13. Find albums released between 1980 and 1990.
14. Find bands whose names contain the word `metal`, ignoring case.
15. Retrieve only band names.
16. Retrieve the number of bands.
17. Retrieve the number of albums.
18. Retrieve distinct genres.
19. Retrieve the first five Metal bands.
20. Implement pagination using `offset()` and `limit()`.

## Advanced exercises

21. Retrieve every band together with its albums.
22. Retrieve albums together with their parent bands.
23. Find bands that have no albums.
24. Find bands that have at least one album.
25. Count the albums belonging to each band.
26. Count the bands belonging to each genre.
27. Find genres containing more than two bands.
28. Find all albums belonging to Metal bands.
29. Retrieve the three most recently released albums.
30. Build a search endpoint that supports genre, name, and pagination together.

## Suggested advanced endpoint

Build:

```http
GET /bands?genre=Metal&search=metal&page=1&page_size=10
```

Your endpoint should:

1. Filter by genre when supplied.
2. Search the band name when supplied.
3. Apply pagination.
4. Order records by ID.
5. Return the matching bands.

---

# FINAL SUMMARY: WHAT YOU SHOULD REMEMBER

The most important idea is that a database query has two stages:

**Stage 1: Construct the query**

```python
statement = (
    select(Band)
    .where(Band.genre == "Metal")
    .order_by(Band.name)
    .limit(10)
)
```

At this point, you have constructed a query. You have not necessarily retrieved the records yet.

**Stage 2: Execute and consume the query**

```python
bands = session.exec(statement).all()
```

Now the database query is executed and its results are retrieved.

Remember these fundamental operations:

```python
# Fetch all records
session.exec(select(Band)).all()

# Fetch by primary key
session.get(Band, 1)

# Filter records
session.exec(
    select(Band).where(Band.genre == "Metal")
).all()

# Sort records
session.exec(
    select(Band).order_by(Band.name.desc())
).all()

# Limit records
session.exec(
    select(Band).limit(5)
).all()

# Count records
session.exec(
    select(func.count()).select_from(Band)
).one()

# Load relationships
session.exec(
    select(Band).options(selectinload(Band.albums))
).all()
```

**The most important distinction:**

`select()` constructs a query, `where()` filters it, `order_by()` sorts it, `limit()` restricts it, and `session.exec()` executes it.

Once you understand how to combine these operations, you can build most of the database retrieval functionality needed in a FastAPI application.
