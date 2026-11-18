# SQLModel Relationships in FastAPI – Complete Guide

## Table of Contents

1. Introduction to Database Relationships
2. Understanding Foreign Keys
3. Understanding SQLModel `Relationship()`
4. One-to-Many Relationships
5. Many-to-One Relationships
6. One-to-One Relationships
7. Many-to-Many Relationships
8. Many-to-Many Relationships with Extra Fields
9. Self-Referential Relationships
10. Understanding `back_populates`
11. Understanding `link_model`
12. Working with Relationships in FastAPI
13. Creating and Reading Related Records
14. Updating and Deleting Related Records
15. Loading Relationships: Lazy Loading and Eager Loading
16. `selectinload()` and `joinedload()`
17. Cascade Operations
18. Relationship Response Schemas
19. Foreign Key Constraints and Database Behaviour
20. Alembic Migrations and Relationships
21. SQL Queries Behind SQLModel Relationships
22. Common Errors and How to Fix Them
23. Best Practices
24. Complete Practice Project

---

# SECTION 1: Introduction to Database Relationships

A database relationship describes how records in one database table are associated with records in another table.

For example, imagine we are developing a music application.

We have two tables:

- `Band`: stores information about music bands.
- `Album`: stores information about albums released by bands.

A band can release multiple albums.

For example:

**Band**

| id | name | genre |
|---|---|---|
| 1 | Coldplay | Alternative |
| 2 | Metallica | Metal |

**Album**

| id | title | band_id |
|---|---|---|
| 1 | Parachutes | 1 |
| 2 | A Rush of Blood to the Head | 1 |
| 3 | Master of Puppets | 2 |

Notice something important:

The `band_id` column in the Album table identifies the band that owns an album.

This is how relational databases connect records.

There are four major relationship types:

1. One-to-One
2. One-to-Many
3. Many-to-One
4. Many-to-Many

Many-to-One is the reverse perspective of One-to-Many.

# SECTION 2: Understanding Foreign Keys

Before understanding SQLModel relationships, you must understand foreign keys.

A foreign key is a column, or group of columns, that references a primary key or another candidate key in a different table.

Consider:

```python
from sqlmodel import SQLModel, Field


class Band(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str


class Album(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id"
    )
```

Let's understand:

```python
band_id: int = Field(foreign_key="band.id")
```

The expression:

```python
foreign_key="band.id"
```

tells SQLModel that `band_id` references the `id` column in the `band` table.

The format is:

```text
foreign_key="table_name.column_name"
```

The generated SQL is conceptually:

```sql
CREATE TABLE band (
    id INTEGER PRIMARY KEY,
    name VARCHAR NOT NULL
);

CREATE TABLE album (
    id INTEGER PRIMARY KEY,
    title VARCHAR NOT NULL,
    band_id INTEGER NOT NULL,

    FOREIGN KEY (band_id)
        REFERENCES band(id)
);
```

**Important:** A foreign key is a database-level constraint. It protects referential integrity.

For example, if Band 1 does not exist, the database should reject an album referencing `band_id = 1`, provided foreign key enforcement is enabled.

## Nullable and Non-Nullable Foreign Keys

Nullable foreign key:

```python
band_id: int | None = Field(
    default=None,
    foreign_key="band.id"
)
```

Non-nullable foreign key:

```python
band_id: int = Field(
    foreign_key="band.id"
)
```

The difference:

| Definition | Database behaviour |
|---|---|
| `int` | Foreign key must have a value |
| `int \| None` | Foreign key may be NULL |

A nullable foreign key allows an album to exist without an associated band.

A non-nullable foreign key requires every album to belong to a band.

Choose based on your application's requirements.

# SECTION 3: Understanding SQLModel Relationship()

SQLModel provides the `Relationship()` function to define ORM relationships between model classes.

Example:

```python
from sqlmodel import SQLModel, Field, Relationship


class Band(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: "Band" = Relationship(
        back_populates="albums"
    )
```

There are two important things here.

First:

```python
band_id: int = Field(foreign_key="band.id")
```

This creates the actual database foreign key.

Second:

```python
albums: list["Album"] = Relationship(
    back_populates="band"
)
```

This defines an ORM relationship attribute.

It allows you to access albums through a Band object:

```python
band.albums
```

Similarly:

```python
band: "Band" = Relationship(
    back_populates="albums"
)
```

Allows you to access the associated band through an Album object:

```python
album.band
```

**Remember this distinction:**

| `Field(foreign_key=...)` | `Relationship(...)` |
|---|---|
| Defines database foreign key | Defines ORM relationship |
| Creates a database column | Creates an ORM attribute |
| Enforces referential integrity | Allows navigation between related objects |
| Used by database constraints | Used by SQLAlchemy ORM |

`Relationship()` does not create another database column.

# SECTION 4: One-to-Many Relationships

A One-to-Many relationship means one record in a table can be associated with multiple records in another table.

Example:

- One band can have many albums.
- Each album belongs to one band.

Diagram:

```text
BAND
----
id
name
  |
  | 1
  |
  |---------------------
  |         |          |
  |         |          |
  | *       | *        | *
ALBUM     ALBUM      ALBUM
```

## 4.1 Implementing One-to-Many

```python
from sqlmodel import SQLModel, Field, Relationship


class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: Band = Relationship(
        back_populates="albums"
    )
```

Let's break down the relationship.

### Band model

```python
albums: list["Album"] = Relationship(
    back_populates="band"
)
```

Why `list["Album"]`?

Because one band can have multiple albums.

The ORM relationship returns a collection of Album objects.

Example:

```python
band.albums
```

Could return:

```python
[
    Album(id=1, title="Parachutes", band_id=1),
    Album(id=2, title="Ghost Stories", band_id=1),
]
```

### Album model

```python
band: Band = Relationship(
    back_populates="albums"
)
```

Why `Band` instead of `list[Band]`?

Because one album belongs to one band in this design.

Example:

```python
album.band
```

Returns a Band object.

## 4.2 Creating Related Records

There are different ways to create related records.

### Method 1: Create the parent first

```python
from sqlmodel import Session


with Session(engine) as session:

    band = Band(name="Coldplay")

    session.add(band)

    session.commit()

    session.refresh(band)

    album = Album(
        title="Parachutes",
        band_id=band.id
    )

    session.add(album)

    session.commit()
```

What happens?

1. Create the Band object.
2. Add it to the session.
3. Commit it to the database.
4. Refresh it to obtain its generated ID.
5. Create an Album using that ID.
6. Commit the Album.

The important part:

```python
band_id=band.id
```

The Album references the Band through its foreign key.

### Method 2: Create through the relationship

```python
with Session(engine) as session:

    band = Band(
        name="Coldplay",
        albums=[
            Album(title="Parachutes"),
            Album(title="Ghost Stories"),
        ]
    )

    session.add(band)

    session.commit()

    session.refresh(band)
```

Here, we create the Band and its albums together.

SQLAlchemy manages the relationship and persists the associated objects through the session.

You do not have to manually assign `band_id` to every Album when you use the ORM relationship correctly.

## 4.3 Reading Related Records

Retrieve a band:

```python
band = session.get(Band, 1)
```

Access its albums:

```python
print(band.albums)
```

Retrieve an album:

```python
album = session.get(Album, 1)
```

Access its band:

```python
print(album.band.name)
```

The ORM allows you to navigate between related objects.

# SECTION 5: Many-to-One Relationships

Many-to-One is simply the reverse perspective of One-to-Many.

Consider:

- Many albums can belong to one band.
- Each album belongs to one band.

From the Album perspective, this is Many-to-One.

The foreign key belongs on the Album table because many album records need to reference one band.

Example:

```python
class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: "Band" = Relationship(
        back_populates="albums"
    )
```

To retrieve the band:

```python
album = session.get(Album, 1)

print(album.band.name)
```

You do not create a separate Many-to-One database structure.

It is the same One-to-Many relationship viewed from the other side.

# SECTION 6: One-to-One Relationships

A One-to-One relationship means one record in a table is associated with at most one record in another table.

Example:

- One user has one profile.
- One profile belongs to one user.

Diagram:

```text
USER                 PROFILE
----                 -------
id  ---------------- user_id
name                 biography
```

## 6.1 Implementing One-to-One

```python
from sqlmodel import SQLModel, Field, Relationship


class User(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    username: str

    profile: "Profile" | None = Relationship(
        back_populates="user"
    )


class Profile(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    biography: str

    user_id: int = Field(
        foreign_key="user.id",
        unique=True
    )

    user: User = Relationship(
        back_populates="profile"
    )
```

Important:

```python
user_id: int = Field(
    foreign_key="user.id",
    unique=True
)
```

The `unique=True` constraint prevents multiple profiles from referencing the same user.

Without a uniqueness constraint, the database would permit several Profile records to reference one User.

That would be One-to-Many rather than One-to-One.

## 6.2 Creating a User with a Profile

```python
user = User(
    username="nzegge",
    profile=Profile(
        biography="Python and backend developer"
    )
)

session.add(user)

session.commit()
```

Access the profile:

```python
print(user.profile.biography)
```

Access the user from the profile:

```python
print(user.profile.user.username)
```

**Important:** `Relationship()` alone does not enforce One-to-One at the database level. The unique constraint on the foreign key is what enforces that rule.

# SECTION 7: Many-to-Many Relationships

A Many-to-Many relationship means multiple records in one table can be associated with multiple records in another table.

Example:

- One student can enrol in many courses.
- One course can have many students.

Diagram:

```text
STUDENT              COURSE
-------              ------
id                   id
name                 title
   \                  /
    \                /
     \              /
      STUDENT_COURSE
      ---------------
      student_id
      course_id
```

A relational database cannot normally represent this relationship using just one foreign key column.

We use an intermediary table.

This is commonly called a junction table, association table or link table.

## 7.1 Implementing Many-to-Many with link_model

SQLModel provides `link_model` for this purpose.

First, define the intermediary table:

```python
from sqlmodel import SQLModel, Field, Relationship


class StudentCourseLink(SQLModel, table=True):

    student_id: int = Field(
        foreign_key="student.id",
        primary_key=True
    )

    course_id: int = Field(
        foreign_key="course.id",
        primary_key=True
    )
```

Notice:

```python
student_id: int = Field(
    foreign_key="student.id",
    primary_key=True
)
```

And:

```python
course_id: int = Field(
    foreign_key="course.id",
    primary_key=True
)
```

Together, they form a composite primary key.

This prevents the same student-course pair from being inserted twice.

Now define Student:

```python
class Student(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    courses: list["Course"] = Relationship(
        back_populates="students",
        link_model=StudentCourseLink
    )
```

Define Course:

```python
class Course(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    students: list[Student] = Relationship(
        back_populates="courses",
        link_model=StudentCourseLink
    )
```

The `link_model` argument tells SQLModel which table connects the two models.

## 7.2 Creating Many-to-Many Records

```python
student = Student(
    name="Charles",
    courses=[
        Course(title="Python"),
        Course(title="Database Systems"),
    ]
)

session.add(student)

session.commit()
```

SQLModel and SQLAlchemy manage the association records.

## 7.3 Accessing Related Records

Get the student's courses:

```python
student = session.get(Student, 1)

for course in student.courses:
    print(course.title)
```

Get the students enrolled in a course:

```python
course = session.get(Course, 1)

for student in course.students:
    print(student.name)
```

# SECTION 8: Many-to-Many Relationships with Extra Fields

Sometimes an intermediary table needs additional information.

For example, imagine a student enrols in a course.

We might want to store:

- Enrollment date
- Grade
- Enrollment status

A simple link model with only two foreign keys is no longer sufficient.

We create an association object.

```python
from datetime import date
from sqlmodel import SQLModel, Field, Relationship


class Enrollment(SQLModel, table=True):

    student_id: int = Field(
        foreign_key="student.id",
        primary_key=True
    )

    course_id: int = Field(
        foreign_key="course.id",
        primary_key=True
    )

    enrollment_date: date

    grade: float | None = None

    status: str = "active"

    student: "Student" = Relationship(
        back_populates="enrollments"
    )

    course: "Course" = Relationship(
        back_populates="enrollments"
    )


class Student(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    enrollments: list[Enrollment] = Relationship(
        back_populates="student"
    )


class Course(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    enrollments: list[Enrollment] = Relationship(
        back_populates="course"
    )
```

Here, the relationship is no longer directly between Student and Course.

Instead:

```text
Student
   |
   | One-to-Many
   |
Enrollment
   |
   | Many-to-One
   |
Course
```

To access courses:

```python
for enrollment in student.enrollments:
    print(enrollment.course.title)
    print(enrollment.grade)
    print(enrollment.status)
```

This pattern is useful when the relationship itself has properties.

Examples:

- User and Role with assigned date.
- Order and Product with quantity and price.
- Student and Course with grade.
- Employee and Department with joining date.

**Important:** When using an association object with extra fields, model the association table as a normal SQLModel table. Do not treat it as a simple `link_model` relationship if you need to manage those additional fields as first-class ORM objects.

# SECTION 9: Self-Referential Relationships

A self-referential relationship occurs when a table references itself.

Example: Employees and managers.

An employee may have another employee as their manager.

```text
EMPLOYEE
--------
id
name
manager_id
```

Example records:

| id | name | manager_id |
|---|---|---|
| 1 | Charles | NULL |
| 2 | John | 1 |
| 3 | Mary | 1 |

Charles manages John and Mary.

## 9.1 SQLModel Implementation

```python
from sqlmodel import SQLModel, Field, Relationship


class Employee(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    manager_id: int | None = Field(
        default=None,
        foreign_key="employee.id"
    )

    manager: "Employee" | None = Relationship(
        back_populates="subordinates",
        sa_relationship_kwargs={
            "remote_side": "Employee.id"
        }
    )

    subordinates: list["Employee"] = Relationship(
        back_populates="manager"
    )
```

Understand the two directions:

```python
manager: Employee | None
```

An employee has one manager, or no manager.

```python
subordinates: list[Employee]
```

An employee can manage multiple employees.

The `remote_side` argument tells SQLAlchemy which side of the self-referential relationship represents the parent row.

Example:

```python
manager = Employee(
    name="Charles"
)

employee1 = Employee(
    name="John",
    manager=manager
)

employee2 = Employee(
    name="Mary",
    manager=manager
)

session.add(manager)
session.add(employee1)
session.add(employee2)

session.commit()
```

Retrieve an employee's manager:

```python
employee = session.get(Employee, 2)

print(employee.manager.name)
```

Retrieve subordinates:

```python
manager = session.get(Employee, 1)

for employee in manager.subordinates:
    print(employee.name)
```

# SECTION 10: Understanding back_populates

This is one of the most important concepts in SQLModel relationships.

Consider:

```python
class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: Band = Relationship(
        back_populates="albums"
    )
```

What does `back_populates` mean?

In the Band model:

```python
back_populates="band"
```

It references the relationship attribute named `band` in the Album model.

In the Album model:

```python
back_populates="albums"
```

It references the relationship attribute named `albums` in the Band model.

They must reference each other correctly.

Think of it as two sides of the same relationship.

```text
Band.albums  <---------------->  Album.band
```

When you assign:

```python
album.band = band
```

SQLAlchemy synchronizes the inverse relationship in memory.

Likewise, assigning an album through the collection:

```python
band.albums.append(album)
```

synchronizes the corresponding relationship.

**Important:** `back_populates` does not create a foreign key. The foreign key is defined separately using `Field(foreign_key=...)`.

# SECTION 11: Understanding link_model

The `link_model` parameter is used for Many-to-Many relationships through a link table.

Example:

```python
courses: list["Course"] = Relationship(
    back_populates="students",
    link_model=StudentCourseLink
)
```

Here:

```python
link_model=StudentCourseLink
```

tells SQLModel to use `StudentCourseLink` as the intermediary table.

Compare:

| Relationship | How it is implemented |
|---|---|
| One-to-Many | Foreign key on the many side |
| Many-to-One | Same foreign key, viewed from the other side |
| One-to-One | Foreign key plus uniqueness constraint |
| Many-to-Many | Link table with two foreign keys |
| Many-to-Many with extra fields | Association object |

# SECTION 12: Working with Relationships in FastAPI

Now let's connect relationships to FastAPI endpoints.

We'll use the Band and Album models.

## 12.1 Separate Database Models from API Schemas

A common mistake is to use the same model for everything.

Instead, separate:

- Table models: define database tables.
- Create schemas: validate incoming requests.
- Read schemas: define API responses.
- Update schemas: define fields that can be changed.

Example:

```python
from sqlmodel import SQLModel, Field, Relationship


class AlbumBase(SQLModel):

    title: str


class AlbumCreate(AlbumBase):
    pass


class AlbumRead(AlbumBase):

    id: int


class BandBase(SQLModel):

    name: str


class BandCreate(BandBase):

    albums: list[AlbumCreate] = []


class BandRead(BandBase):

    id: int

    albums: list[AlbumRead] = []


class Band(BandBase, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(AlbumBase, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: Band = Relationship(
        back_populates="albums"
    )
```

For more complex projects, you may organize these classes into separate files.

One important improvement: use `Field(default_factory=list)` instead of `[]` for list defaults.

For example:

```python
class BandCreate(BandBase):

    albums: list[AlbumCreate] = Field(
        default_factory=list
    )
```

## 12.2 Creating a Band with Albums

```python
from fastapi import FastAPI, Depends
from sqlmodel import Session

app = FastAPI()


def get_session():
    with Session(engine) as session:
        yield session


@app.post("/bands", response_model=BandRead)
def create_band(
    band_data: BandCreate,
    session: Session = Depends(get_session)
):

    band = Band(
        name=band_data.name,
        albums=[
            Album(title=album.title)
            for album in band_data.albums
        ]
    )

    session.add(band)

    session.commit()

    session.refresh(band)

    return band
```

Let's understand the process.

The incoming request might be:

```json
{
    "name": "Coldplay",
    "albums": [
        {
            "title": "Parachutes"
        },
        {
            "title": "Ghost Stories"
        }
    ]
}
```

FastAPI validates the JSON using `BandCreate`.

Next:

```python
band = Band(
    name=band_data.name,
    albums=[
        Album(title=album.title)
        for album in band_data.albums
    ]
)
```

Creates the parent and child ORM objects.

Then:

```python
session.add(band)
```

Adds the object graph to the session.

Then:

```python
session.commit()
```

Persists the records.

SQLAlchemy handles the insertion order and foreign key assignments.

Finally:

```python
session.refresh(band)
```

Refreshes the Band instance from the database.

## 12.3 Getting a Band with Its Albums

```python
@app.get("/bands/{band_id}", response_model=BandRead)
def get_band(
    band_id: int,
    session: Session = Depends(get_session)
):

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found"
        )

    return band
```

Because `BandRead` contains:

```python
albums: list[AlbumRead]
```

FastAPI can return the associated albums as nested JSON, provided they have been loaded before serialization.

# SECTION 13: Updating Related Records

Suppose we want to add an album to an existing band.

```python
@app.post("/bands/{band_id}/albums")
def create_album(
    band_id: int,
    album_data: AlbumCreate,
    session: Session = Depends(get_session)
):

    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=404,
            detail="Band not found"
        )

    album = Album(
        title=album_data.title,
        band=band
    )

    session.add(album)

    session.commit()

    session.refresh(album)

    return album
```

Notice:

```python
band=band
```

We assign the relationship object instead of manually assigning its ID.

SQLAlchemy synchronizes the foreign key.

Another approach:

```python
album = Album(
    title=album_data.title,
    band_id=band.id
)
```

Both approaches can work.

The first uses the ORM relationship; the second explicitly assigns the foreign key.

## Updating an Album's Band

If an album is allowed to change ownership:

```python
album = session.get(Album, album_id)

new_band = session.get(Band, new_band_id)

if album is None or new_band is None:
    raise HTTPException(
        status_code=404,
        detail="Album or band not found"
    )

album.band = new_band

session.commit()

session.refresh(album)
```

The ORM updates the `band_id` foreign key.

# SECTION 14: Deleting Related Records

Suppose we delete a Band.

What should happen to its Albums?

There are different possible behaviours:

1. Prevent deletion if albums exist.
2. Delete the albums automatically.
3. Set their `band_id` to NULL.

This is a database design decision.

## 14.1 Deleting a Parent Record

```python
band = session.get(Band, band_id)

if band is None:
    raise HTTPException(
        status_code=404,
        detail="Band not found"
    )

session.delete(band)

session.commit()
```

The actual outcome depends on the database constraints and ORM configuration.

## 14.2 Database-Level ON DELETE CASCADE

You can configure the foreign key:

```python
class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id",
        ondelete="CASCADE"
    )
```

This instructs the database to delete associated albums when their parent band is deleted.

However, database cascade behaviour must be supported and enabled by the database.

For SQLite, foreign key enforcement must be enabled on each connection.

## 14.3 ORM Cascade

SQLAlchemy also supports ORM cascade settings.

For example:

```python
class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    albums: list["Album"] = Relationship(
        back_populates="band",
        sa_relationship_kwargs={
            "cascade": "all, delete-orphan"
        }
    )
```

`delete-orphan` means an Album removed from the Band's relationship collection can be deleted when the ORM processes the change.

Be careful with cascade settings. They can cause related records to be deleted automatically.

Do not use cascade deletion unless it matches your business rules.

# SECTION 15: Loading Relationships

This is an important SQLAlchemy topic.

When you execute:

```python
band = session.get(Band, 1)
```

Does SQLAlchemy automatically retrieve all albums?

Not necessarily.

SQLAlchemy relationships use a loading strategy.

The most common strategies are:

- Lazy loading
- Eager loading

## 15.1 Lazy Loading

Lazy loading means related records are loaded when you access the relationship attribute.

Example:

```python
band = session.get(Band, 1)

print(band.name)

print(band.albums)
```

The initial query retrieves the Band.

When you access:

```python
band.albums
```

SQLAlchemy may execute another SQL query to retrieve the albums.

Conceptually:

```sql
SELECT * FROM band
WHERE id = 1;
```

Then:

```sql
SELECT * FROM album
WHERE band_id = 1;
```

This can be convenient, but it may create performance problems when retrieving many parent records.

## 15.2 The N+1 Query Problem

Imagine retrieving 100 bands:

```python
bands = session.exec(
    select(Band)
).all()
```

Then:

```python
for band in bands:
    print(band.albums)
```

You might execute:

- One query to retrieve all bands.
- One additional query for each band's albums.

That is potentially 101 queries.

This is known as the N+1 query problem.

Eager loading helps solve it.

# SECTION 16: selectinload() and joinedload()

SQLAlchemy provides loading options that retrieve related records more efficiently.

## 16.1 selectinload()

```python
from sqlalchemy.orm import selectinload
from sqlmodel import select


statement = (
    select(Band)
    .options(selectinload(Band.albums))
)

bands = session.exec(statement).all()
```

What happens?

First, SQLAlchemy retrieves the bands:

```sql
SELECT * FROM band;
```

Then it retrieves their albums in a separate query using the IDs of the retrieved bands.

Conceptually:

```sql
SELECT * FROM album
WHERE band_id IN (1, 2, 3, 4);
```

This avoids executing a separate album query for every band.

For collections such as One-to-Many and Many-to-Many, `selectinload()` is often a good default.

## 16.2 joinedload()

```python
from sqlalchemy.orm import joinedload


statement = (
    select(Album)
    .options(joinedload(Album.band))
)

albums = session.exec(statement).all()
```

Conceptually, SQLAlchemy executes a JOIN:

```sql
SELECT *
FROM album
LEFT JOIN band
ON album.band_id = band.id;
```

The associated Band objects are loaded as part of the same query.

For a collection relationship, joined loading can produce repeated parent rows in the SQL result.

When using SQLAlchemy's direct `Result` API with collection joined loading, you must call `unique()` before consuming the results.

For example:

```python
statement = (
    select(Band)
    .options(joinedload(Band.albums))
)

result = session.execute(statement)

bands = result.unique().scalars().all()
```

With SQLModel, `selectinload()` is often simpler for collections.

## 16.3 Comparison

| Feature | selectinload | joinedload |
|---|---|---|
| Query strategy | Separate SELECT query | SQL JOIN |
| Collections | Very useful | Can duplicate parent rows |
| Many parent records | Usually efficient | May create large results |
| Single related object | Works | Often efficient |
| N+1 prevention | Yes | Yes |

# SECTION 17: Relationship Configuration

SQLAlchemy provides several relationship configuration options.

SQLModel exposes many of them through `Relationship()` and `sa_relationship_kwargs`.

Examples include:

- `back_populates`
- `link_model`
- `cascade`
- `lazy`
- `passive_deletes`
- `foreign_keys`
- `primaryjoin`
- `secondaryjoin`
- `remote_side`
- `uselist`
- `order_by`

Example:

```python
albums: list["Album"] = Relationship(
    back_populates="band",
    sa_relationship_kwargs={
        "lazy": "selectin"
    }
)
```

This configures the relationship's loading strategy.

Another example:

```python
sa_relationship_kwargs={
    "order_by": "Album.id"
}
```

This orders the associated albums by their ID when loading the collection.

Use advanced options only when you understand the relationship they configure.

# SECTION 18: Relationship Response Schemas

A common FastAPI mistake is returning database relationship objects without planning the response schema.

Consider:

```python
class AlbumRead(SQLModel):

    id: int

    title: str


class BandRead(SQLModel):

    id: int

    name: str

    albums: list[AlbumRead]
```

This allows a response such as:

```json
{
    "id": 1,
    "name": "Coldplay",
    "albums": [
        {
            "id": 1,
            "title": "Parachutes"
        },
        {
            "id": 2,
            "title": "Ghost Stories"
        }
    ]
}
```

Notice that `AlbumRead` does not contain the full Band object.

This is intentional.

## 18.1 Avoid Circular Response Schemas

Do not blindly create this structure:

```python
class BandRead(SQLModel):

    id: int

    albums: list["AlbumRead"]


class AlbumRead(SQLModel):

    id: int

    band: BandRead
```

This creates a recursive response structure:

```text
Band
  |
  v
Albums
  |
  v
Band
  |
  v
Albums
```

Instead, create different schemas for different API operations.

For example:

```python
class AlbumRead(SQLModel):

    id: int

    title: str


class AlbumWithBandRead(SQLModel):

    id: int

    title: str

    band_id: int


class BandRead(SQLModel):

    id: int

    name: str

    albums: list[AlbumRead]
```

You can return the band with its albums without returning the entire band again inside each album.

# SECTION 19: Foreign Key Constraints and Database Behaviour

Different database systems support relational constraints, but their behaviour and configuration details can differ.

## PostgreSQL

PostgreSQL supports:

- Foreign keys
- Composite primary keys
- Unique constraints
- Check constraints
- Cascade deletion
- Transactions
- Deferrable constraints

## MySQL

For relational foreign keys, use the InnoDB storage engine.

MySQL supports:

- Foreign keys
- Composite primary keys
- Unique constraints
- Cascade deletion
- Transactions with InnoDB

## SQLite

SQLite supports:

- Foreign keys
- Composite primary keys
- Unique constraints
- Cascade deletion

However, SQLite foreign key enforcement must be enabled.

For SQLAlchemy, a connection event can enable it:

```python
from sqlalchemy import event
from sqlmodel import create_engine


engine = create_engine(
    "sqlite:///./db.sqlite",
    connect_args={
        "check_same_thread": False
    }
)


@event.listens_for(engine, "connect")
def enable_sqlite_foreign_keys(
    dbapi_connection,
    connection_record
):

    cursor = dbapi_connection.cursor()

    cursor.execute("PRAGMA foreign_keys=ON")

    cursor.close()
```

This ensures foreign key enforcement is enabled for new SQLite connections.

# SECTION 20: Alembic Migrations and Relationships

Whenever you modify a database model, the database schema may need to change.

For example, you initially have:

```python
class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str
```

Later, you add:

```python
albums: list["Album"] = Relationship(
    back_populates="band"
)
```

Does this automatically create an `albums` column in the Band table?

No.

`Relationship()` creates an ORM relationship attribute, not a database column.

The actual database change is usually the foreign key in the Album table:

```python
band_id: int = Field(
    foreign_key="band.id"
)
```

Generate a migration:

```bash
uv run alembic revision --autogenerate -m "create album relationship"
```

Inspect the generated migration.

Apply it:

```bash
uv run alembic upgrade head
```

Useful Alembic commands:

```bash
uv run alembic current

uv run alembic history

uv run alembic heads

uv run alembic revision --autogenerate -m "description"

uv run alembic upgrade head

uv run alembic downgrade -1
```

Remember:

- Foreign keys affect database schema.
- Unique constraints affect database schema.
- Adding or removing table columns affects database schema.
- `Relationship()` itself does not normally add a database column.

# SECTION 21: SQL Queries Behind SQLModel Relationships

Understanding the SQL generated by SQLAlchemy will make you a better backend developer.

Consider:

```python
class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: Band = Relationship(
        back_populates="albums"
    )
```

## Retrieve all bands

SQLModel:

```python
bands = session.exec(
    select(Band)
).all()
```

SQL:

```sql
SELECT * FROM band;
```

## Retrieve albums belonging to a band

SQL:

```sql
SELECT *
FROM album
WHERE band_id = 1;
```

## Retrieve the band associated with an album

SQL:

```sql
SELECT band.*
FROM band
JOIN album
ON band.id = album.band_id
WHERE album.id = 1;
```

## Retrieve bands and their albums

SQL:

```sql
SELECT
    band.id,
    band.name,
    album.id,
    album.title
FROM band
LEFT JOIN album
ON band.id = album.band_id;
```

The ORM provides a Python interface for working with these relational structures.

It does not eliminate SQL. It helps you work with SQL through Python objects.

# SECTION 22: Common Errors and How to Fix Them

## Error 1: Missing foreign key

Incorrect:

```python
class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int
```

There is no foreign key constraint.

Correct:

```python
band_id: int = Field(
    foreign_key="band.id"
)
```

## Error 2: Incorrect back_populates

Incorrect:

```python
class Band(SQLModel, table=True):

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(SQLModel, table=True):

    band: "Band" = Relationship(
        back_populates="album"
    )
```

The second relationship refers to `album`, but the Band attribute is named `albums`.

Correct:

```python
class Album(SQLModel, table=True):

    band: "Band" = Relationship(
        back_populates="albums"
    )
```

## Error 3: Incorrect collection type

For One-to-Many:

```python
albums: list["Album"] = Relationship(...)
```

For Many-to-One:

```python
band: "Band" = Relationship(...)
```

For One-to-One:

```python
profile: "Profile" | None = Relationship(...)
```

For Many-to-Many:

```python
courses: list["Course"] = Relationship(...)
```

## Error 4: Forgetting to import models in Alembic

In `env.py`, ensure your models are imported before assigning metadata:

```python
from sqlmodel import SQLModel

from introduction_into_fastapi.models import Band, Album

target_metadata = SQLModel.metadata
```

This ensures the model table definitions are registered.

## Error 5: Table does not exist

Error:

```text
sqlite3.OperationalError: no such table: band
```

Possible causes:

- Migration was never generated.
- Migration was generated but not applied.
- Alembic and FastAPI use different database files.
- Alembic did not import the models.
- The migration does not create the expected table.

Check:

```bash
uv run alembic current

uv run alembic history

uv run alembic upgrade head
```

## Error 6: Foreign key constraint failed

Example:

```text
FOREIGN KEY constraint failed
```

This usually means the child record references a parent record that does not exist, or a related deletion violates a foreign key constraint.

Verify:

```python
band = session.get(Band, band_id)

if band is None:
    raise HTTPException(
        status_code=404,
        detail="Band not found"
    )
```

## Error 7: Missing relationship data in API response

Check:

1. Does the response schema include the relationship?
2. Has the relationship been loaded?
3. Is the session still available when serialization occurs?
4. Are you using the correct loading strategy?

For example:

```python
statement = (
    select(Band)
    .options(selectinload(Band.albums))
)

bands = session.exec(statement).all()
```

# SECTION 23: Best Practices

1. Always understand which side owns the foreign key.
2. Define foreign keys using `Field(foreign_key=...)`.
3. Use `Relationship()` for ORM navigation.
4. Use matching `back_populates` attributes on both sides.
5. Use `list[Model]` for collection relationships.
6. Use a single model type for a relationship to one object.
7. Use `unique=True` when implementing a One-to-One foreign key.
8. Use a link table for Many-to-Many relationships.
9. Use an association object when the link has additional attributes.
10. Use `selectinload()` to avoid common N+1 query problems with collections.
11. Design response schemas separately from table models.
12. Avoid recursive nested response schemas.
13. Be deliberate with cascade deletion.
14. Use Alembic to manage database schema changes.
15. Test foreign key behaviour in your chosen database.
16. Understand the SQL behind ORM operations.

# SECTION 24: Complete Practice Project

Let's combine everything into a small music management application.

Our database contains:

- Band
- Album
- Artist
- Genre

For the first exercise, implement Band and Album.

## Models

```python
from sqlmodel import (
    SQLModel,
    Field,
    Relationship
)


class Band(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    name: str

    albums: list["Album"] = Relationship(
        back_populates="band"
    )


class Album(SQLModel, table=True):

    id: int | None = Field(
        default=None,
        primary_key=True
    )

    title: str

    band_id: int = Field(
        foreign_key="band.id"
    )

    band: Band = Relationship(
        back_populates="albums"
    )
```

## Exercises

### Exercise 1: Create a Band

Create a Band called Coldplay.

### Exercise 2: Create Albums

Create three albums belonging to Coldplay.

### Exercise 3: Retrieve a Band

Retrieve Coldplay and display its albums.

### Exercise 4: Retrieve an Album

Retrieve one album and display the name of its band.

### Exercise 5: Update an Album

Change the title of one album.

### Exercise 6: Change an Album's Band

Create another band and transfer an album to it.

### Exercise 7: Delete an Album

Delete one album without deleting its band.

### Exercise 8: Add One-to-One

Create a User and Profile relationship.

### Exercise 9: Add Many-to-Many

Create Student and Course models connected through a link table.

### Exercise 10: Eager Loading

Retrieve all bands and their albums using `selectinload()`.

### Exercise 11: Advanced Association Object

Create an Order and Product relationship where the association stores:

- Quantity
- Unit price
- Created date

### Exercise 12: Database Comparison

Run the same relationship project using:

- SQLite
- PostgreSQL
- MySQL

Compare their foreign key behaviour, migrations and generated SQL.

---

# FINAL SUMMARY

| Relationship | Example | Foreign key location |
|---|---|---|
| One-to-One | User → Profile | Usually on Profile, with UNIQUE |
| One-to-Many | Band → Albums | Album |
| Many-to-One | Albums → Band | Album |
| Many-to-Many | Students ↔ Courses | Intermediate table |
| Self-referential | Employee → Manager | Employee |

The most important distinction to remember:

```python
Field(foreign_key="band.id")
```

Defines the database-level foreign key.

```python
Relationship(back_populates="albums")
```

Defines the ORM-level relationship.

These two concepts work together, but they are not the same thing.

**My advice:** Master One-to-Many first. Once you fully understand why the foreign key belongs on the Album table, how `back_populates` connects the two model attributes, and how SQLAlchemy saves related records, the other relationship types will become much easier to understand.
