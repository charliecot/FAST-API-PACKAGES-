# ALEMBIC COMPLETE GUIDE
## Terminal Commands, SQL, SQLite, PostgreSQL, MySQL, and Configuration Files

---

# TABLE OF CONTENTS

1. Introduction to Alembic
2. Understanding Database Migrations
3. Installing Alembic
4. Creating an Alembic Project
5. Understanding the Alembic Project Structure
6. Understanding `alembic.ini`
7. Understanding `env.py`
8. Understanding `script.py.mako`
9. Alembic Terminal Commands
10. Understanding Migration Files
11. Working with SQLite
12. Working with PostgreSQL
13. Working with MySQL
14. SQL Commands for Database Schema Management
15. Working with SQLModel Models
16. Autogenerate and Metadata
17. Migration Operations
18. Editing and Managing Existing Migrations
19. Database Version Commands
20. Downgrading and Rolling Back Migrations
21. Using Alembic with FastAPI
22. Common Errors and Solutions
23. Recommended Project Structure
24. Complete Migration Workflow
25. Important Best Practices

---

# SECTION 1: INTRODUCTION TO ALEMBIC

## 1.1 What is Alembic?

Alembic is a database migration tool built on top of SQLAlchemy.

It helps developers manage changes to database structures over time.

For example, imagine that you initially have this SQLModel:

```python
from sqlmodel import SQLModel, Field


class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    username: str
```

Alembic can generate a migration that creates the corresponding database table.

Later, you decide to add an email field:

```python
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    username: str
    email: str
```

Alembic can detect that the model has changed and generate a migration to add the `email` column.

You then apply that migration to your database.

## 1.2 Why do we need Alembic?

Without a migration tool, you might manually execute SQL every time you change your database structure.

For example:

```sql
ALTER TABLE user ADD COLUMN email VARCHAR(255);
```

This works, but becomes difficult to manage as your application grows.

Alembic provides a controlled way to:

- Create database tables.
- Add new columns.
- Remove columns.
- Modify column types.
- Create indexes.
- Create foreign keys.
- Remove foreign keys.
- Rename or replace database structures.
- Roll back previous migrations.
- Track which migrations have been applied.

## 1.3 Alembic and SQLAlchemy

Understand the relationship:

```text
FastAPI
   |
   v
SQLModel / SQLAlchemy Models
   |
   v
SQLAlchemy Metadata
   |
   v
Alembic
   |
   v
Database Migration Scripts
   |
   v
SQLite / PostgreSQL / MySQL
```

SQLModel defines your models.

SQLAlchemy provides the database interaction layer and metadata.

Alembic compares your model metadata with the existing database schema and generates migration operations.

The database executes the resulting SQL statements.

**Important:** Alembic does not automatically modify your database just because you modify a Python model. You must generate and apply a migration.

---

# SECTION 2: UNDERSTANDING DATABASE MIGRATIONS

## 2.1 What is a migration?

A migration is a Python file that describes a database schema change.

For example:

```python
def upgrade():
    op.add_column(
        "user",
        sa.Column("email", sa.String(length=255))
    )


def downgrade():
    op.drop_column("user", "email")
```

The `upgrade()` function applies the change.

The `downgrade()` function reverses it.

## 2.2 Upgrade and downgrade

Think of migration history as a sequence:

```text
Migration 1: Create user table
       |
       v
Migration 2: Add email column
       |
       v
Migration 3: Add age column
       |
       v
Migration 4: Create product table
```

Upgrade moves forward:

```text
Migration 1 -> Migration 2 -> Migration 3 -> Migration 4
```

Downgrade moves backward:

```text
Migration 4 -> Migration 3 -> Migration 2 -> Migration 1
```

## 2.3 Alembic version tracking

Alembic creates a table called:

```text
alembic_version
```

This table records the current migration revision applied to the database.

For example:

```sql
SELECT * FROM alembic_version;
```

Possible result:

```text
version_num
----------------
a123456789ab
```

This tells Alembic which migration revision the database is currently using.

It does not store your application's data or every migration's SQL. The migration history is stored in your project files.

---

# SECTION 3: INSTALLING ALEMBIC

## 3.1 Install Alembic

Using pip:

```bash
pip install alembic
```

Using uv:

```bash
uv add alembic
```

Check the installed version:

```bash
alembic --version
```

You can also run:

```bash
python -m alembic --version
```

## 3.2 Database drivers

Alembic uses SQLAlchemy database drivers to connect to databases.

Install the driver for the database you intend to use.

### SQLite

SQLite is supported by Python's built-in `sqlite3` module.

You generally do not need an additional driver.

### PostgreSQL

Install psycopg:

```bash
uv add "psycopg[binary]"
```

Or:

```bash
pip install "psycopg[binary]"
```

An alternative is psycopg2:

```bash
pip install psycopg2-binary
```

### MySQL

Install PyMySQL:

```bash
uv add pymysql
```

Or:

```bash
pip install pymysql
```

Another option is mysqlclient:

```bash
pip install mysqlclient
```

**Note:** The SQLAlchemy URL must match the driver you installed.

---

# SECTION 4: CREATING AN ALEMBIC PROJECT

## 4.1 Initialize Alembic

Inside your FastAPI project directory:

```bash
alembic init migrations
```

This creates a migration directory.

Example:

```text
my-fastapi-project/
│
├── migrations/
│   ├── versions/
│   ├── env.py
│   ├── README
│   └── script.py.mako
│
├── alembic.ini
│
├── src/
│   └── main.py
│
└── pyproject.toml
```

## 4.2 What does the command create?

```bash
alembic init migrations
```

- `alembic`: Runs the Alembic command-line application.
- `init`: Initializes a new Alembic environment.
- `migrations`: The name of the directory that will contain the migration environment.

You can choose another directory name:

```bash
alembic init alembic
```

However, use one consistent directory name throughout your project.

## 4.3 Important files

| File or directory | Purpose |
|---|---|
| `alembic.ini` | Main Alembic configuration |
| `env.py` | Database connection and migration environment |
| `script.py.mako` | Template used to generate migration files |
| `versions/` | Stores individual migration files |
| `README` | Information about the migration environment |

---

# SECTION 5: UNDERSTANDING ALEMBIC.INI

## 5.1 What is `alembic.ini`?

`alembic.ini` is the main configuration file for Alembic.

It contains settings such as:

- The migration script directory.
- The database connection URL.
- Logging configuration.
- Other Alembic configuration options.

A simplified configuration:

```ini
[alembic]
script_location = %(here)s/migrations

prepend_sys_path = .

sqlalchemy.url = sqlite:///./database.db


[loggers]
keys = root,sqlalchemy,alembic


[handlers]
keys = console


[formatters]
keys = generic


[logger_root]
level = WARN
handlers = console


[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine


[logger_alembic]
level = INFO
handlers = console
qualname = alembic


[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic


[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

## 5.2 Understanding the important settings

### `script_location`

```ini
script_location = %(here)s/migrations
```

This tells Alembic where your migration environment is located.

`%(here)s` refers to the directory containing the `alembic.ini` file.

For example:

```text
project/
│
├── alembic.ini
└── migrations/
```

### `prepend_sys_path`

```ini
prepend_sys_path = .
```

This adds the configured path to Python's import search path.

It helps Alembic locate your project modules.

For example, your `env.py` may need to import:

```python
from src.models import User
```

If your project uses a `src` layout, make sure the appropriate project directory is importable.

### `sqlalchemy.url`

```ini
sqlalchemy.url = sqlite:///./database.db
```

This defines the database connection URL.

The value depends on the database you are using.

Examples:

```ini
# SQLite
sqlalchemy.url = sqlite:///./database.db

# PostgreSQL
sqlalchemy.url = postgresql+psycopg://username:password@localhost:5432/mydatabase

# MySQL
sqlalchemy.url = mysql+pymysql://username:password@localhost:3306/mydatabase
```

You will see these configurations in the individual database sections below.

## 5.3 Should I put my database password in alembic.ini?

You can, but it is not recommended for production.

For example:

```ini
sqlalchemy.url = postgresql+psycopg://admin:secret123@localhost:5432/shop
```

This exposes your database credentials if you commit the file to Git.

A better approach is to load the URL from an environment variable inside `env.py`.

Example:

```python
import os

DATABASE_URL = os.getenv("DATABASE_URL")
```

You can use `python-dotenv`:

```bash
uv add python-dotenv
```

Your `.env` file:

```env
DATABASE_URL=postgresql+psycopg://admin:password@localhost:5432/shop
```

Do not commit `.env` to Git.

Add it to `.gitignore`:

```gitignore
.env
```

---

# SECTION 6: UNDERSTANDING ENV.PY

## 6.1 What is `env.py`?

`env.py` is one of the most important Alembic files.

It defines how Alembic connects to your database and obtains your application's SQLAlchemy metadata.

It is also responsible for running migrations.

The generated file normally contains two important functions:

```python
run_migrations_offline()
run_migrations_online()
```

## 6.2 Understanding target_metadata

Consider this SQLModel:

```python
from sqlmodel import SQLModel, Field


class User(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    username: str
```

SQLModel creates table metadata from the model.

You can access this metadata using:

```python
SQLModel.metadata
```

In Alembic:

```python
target_metadata = SQLModel.metadata
```

This tells Alembic which model definitions it should compare against the database schema.

Without the correct metadata, autogenerate may not detect your tables.

## 6.3 Import your models

For Alembic to detect your models, their modules must be imported.

For example:

```text
src/
│
├── database.py
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── product.py
│
└── main.py
```

Your `env.py` might contain:

```python
from sqlmodel import SQLModel

from src.models.user import User
from src.models.product import Product

target_metadata = SQLModel.metadata
```

The imports are important because Python must execute the model declarations before their tables are registered in `SQLModel.metadata`.

If you have many models, you can import them through a package initializer.

For example, `src/models/__init__.py`:

```python
from .user import User
from .product import Product
from .order import Order
```

Then:

```python
import src.models

target_metadata = SQLModel.metadata
```

## 6.4 Offline migrations

The generated Alembic environment contains:

```python
def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")

    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()
```

Offline mode generates SQL without establishing a live database connection.

For example:

```bash
alembic upgrade head --sql
```

This generates SQL for the migration operations instead of executing them against the database.

Important:

- Offline mode requires a usable database URL or dialect configuration.
- It does not inspect the live database.
- It is useful for reviewing or generating SQL scripts.

## 6.5 Online migrations

The generated environment also contains:

```python
def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
        )

        with context.begin_transaction():
            context.run_migrations()
```

Online mode connects to the actual database and executes migration operations.

This is the normal mode when you run:

```bash
alembic upgrade head
```

## 6.6 Complete SQLModel env.py example

This example uses a synchronous SQLAlchemy engine.

```python
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

from sqlmodel import SQLModel

# Import your models.
# These imports register their tables in SQLModel.metadata.
from src.models import User, Product, Order

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = SQLModel.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")

    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Understanding `compare_type=True`

```python
compare_type=True
```

This enables Alembic to compare column types in your models against the database.

For example, suppose your model changes from:

```python
username: str = Field(max_length=50)
```

to:

```python
username: str = Field(max_length=150)
```

Alembic can detect the type or length difference when comparing metadata with the database.

You should still inspect the generated migration.

### Understanding `NullPool`

```python
poolclass=pool.NullPool
```

This tells SQLAlchemy not to maintain a reusable connection pool for the migration engine.

It is commonly used in the generated Alembic environment because migrations are short-lived operations.

---

# SECTION 7: UNDERSTANDING SCRIPT.PY.MAKO

## 7.1 What is `script.py.mako`?

`script.py.mako` is a template used by Alembic when generating new migration files.

When you execute:

```bash
alembic revision -m "create user table"
```

Alembic uses this template to create a new Python migration file inside:

```text
migrations/versions/
```

The generated migration file contains:

- Revision identifier.
- Previous revision identifier.
- Optional branch information.
- `upgrade()` function.
- `downgrade()` function.

## 7.2 Example of script.py.mako

A typical template looks similar to this:

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""

from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa

${imports if imports else ""}

revision: str = ${repr(up_revision)}

down_revision: Union[str, Sequence[str], None] = ${repr(down_revision)}

branch_labels: Union[str, Sequence[str], None] = ${repr(branch_labels)}

depends_on: Union[str, Sequence[str], None] = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

## 7.3 Understanding the template variables

| Variable | Meaning |
|---|---|
| `${message}` | Description supplied with `-m` |
| `${up_revision}` | New migration identifier |
| `${down_revision}` | Previous migration identifier |
| `${create_date}` | Migration creation date |
| `${imports}` | Additional imports required by generated operations |
| `${upgrades}` | Generated upgrade operations |
| `${downgrades}` | Generated downgrade operations |
| `${branch_labels}` | Optional branch labels |
| `${depends_on}` | Optional migration dependencies |

## 7.4 Should I modify script.py.mako?

Normally, no.

Alembic provides a default template that works for most projects.

You may customize it if you want generated migrations to contain additional imports, formatting, comments, or project-specific conventions.

Do not confuse the template with an actual migration file.

- `script.py.mako` generates migration files.
- Files inside `versions/` are the actual migration files executed by Alembic.

---

# SECTION 8: ALEMBIC TERMINAL COMMANDS

This section covers the commands you will use regularly.

## 8.1 Initialize a migration environment

```bash
alembic init migrations
```

Creates the Alembic environment.

## 8.2 Create an empty migration

```bash
alembic revision -m "initial migration"
```

Creates a migration file with empty upgrade and downgrade functions.

Use this when you want to write the migration operations yourself.

## 8.3 Autogenerate a migration

```bash
alembic revision --autogenerate -m "create user table"
```

This compares the current database schema with your model metadata and generates migration operations for detected differences.

Important:

Autogenerate does not apply the migration. It only creates the migration file.

## 8.4 Apply all migrations

```bash
alembic upgrade head
```

Applies migrations until the latest revision.

## 8.5 Apply one migration step

```bash
alembic upgrade +1
```

Moves forward by one revision from the current database revision.

## 8.6 Upgrade to a specific revision

```bash
alembic upgrade a123456789ab
```

Applies migrations until the specified revision.

## 8.7 Downgrade one revision

```bash
alembic downgrade -1
```

Moves backward by one revision.

## 8.8 Downgrade to a specific revision

```bash
alembic downgrade a123456789ab
```

Moves backward until the specified revision.

## 8.9 Downgrade to the beginning

```bash
alembic downgrade base
```

Reverses all applied migrations.

**Warning:** This may drop tables or remove columns, depending on your downgrade functions. Back up important data first.

## 8.10 Display migration history

```bash
alembic history
```

Shows migration revisions and their relationships.

More detailed:

```bash
alembic history --verbose
```

## 8.11 Display the current database revision

```bash
alembic current
```

Shows the revision currently applied to the connected database.

More detailed:

```bash
alembic current --verbose
```

## 8.12 Display the latest revision

```bash
alembic heads
```

Displays the current migration heads.

## 8.13 Display the base revisions

```bash
alembic branches
```

Displays branch points in migration history.

## 8.14 Display migration information

```bash
alembic show head
```

Displays information about the latest revision.

You can also use a specific revision:

```bash
alembic show a123456789ab
```

## 8.15 Generate SQL without executing migrations

```bash
alembic upgrade head --sql
```

Generates the SQL statements for upgrading to the latest revision.

For a specific range:

```bash
alembic upgrade base:head --sql
```

For downgrading:

```bash
alembic downgrade head:base --sql
```

## 8.16 Stamp the database

```bash
alembic stamp head
```

Marks the database as being at the latest migration revision without executing the migration operations.

Use this carefully.

For example, you might use `stamp` when you have an existing database whose schema already matches the migration history you want to establish.

It does not create missing tables or columns.

## 8.17 Create a new revision with a specific parent

```bash
alembic revision --rev-id my_revision -m "add profile table"
```

Creates a revision with a custom revision ID.

Usually, you should allow Alembic to generate revision IDs automatically.

---

# SECTION 9: UNDERSTANDING MIGRATION FILES

## 9.1 Example migration file

```python
"""create user table

Revision ID: a123456789ab
Revises:
"""

from alembic import op
import sqlalchemy as sa


revision = "a123456789ab"

down_revision = None

branch_labels = None

depends_on = None


def upgrade() -> None:
    op.create_table(
        "user",
        sa.Column(
            "id",
            sa.Integer(),
            primary_key=True
        ),
        sa.Column(
            "username",
            sa.String(length=100),
            nullable=False
        ),
    )


def downgrade() -> None:
    op.drop_table("user")
```

## 9.2 Understanding revision

```python
revision = "a123456789ab"
```

This is the unique identifier of the migration.

## 9.3 Understanding down_revision

```python
down_revision = None
```

This tells Alembic which migration came before the current one.

For the first migration:

```python
down_revision = None
```

For the second migration:

```python
down_revision = "a123456789ab"
```

The migration history forms a dependency chain.

## 9.4 Understanding upgrade()

```python
def upgrade():
    op.create_table(...)
```

This function applies the migration.

For example:

```python
op.add_column(
    "user",
    sa.Column("email", sa.String(255))
)
```

Equivalent SQL:

```sql
ALTER TABLE user
ADD COLUMN email VARCHAR(255);
```

## 9.5 Understanding downgrade()

```python
def downgrade():
    op.drop_column("user", "email")
```

This reverses the change.

Equivalent SQL:

```sql
ALTER TABLE user
DROP COLUMN email;
```

The exact SQL syntax can differ between database systems.

---

# SECTION 10: SQLITE CONFIGURATION

## 10.1 What is SQLite?

SQLite is a lightweight relational database.

Unlike PostgreSQL and MySQL, it does not require a separate database server process.

The database is usually stored in a file.

Example:

```text
project/
│
├── database.db
└── migrations/
```

## 10.2 SQLite database URL

Relative path:

```ini
sqlalchemy.url = sqlite:///./database.db
```

Absolute Windows path:

```ini
sqlalchemy.url = sqlite:///C:/Projects/myapp/database.db
```

Use forward slashes in SQLAlchemy URLs on Windows.

For an in-memory database:

```ini
sqlalchemy.url = sqlite://
```

An in-memory database is temporary and generally not suitable for persistent production migrations.

## 10.3 SQLite env.py configuration

SQLite may require special connection arguments.

For example:

```python
from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context
from sqlmodel import SQLModel

from src.models import User, Product

config = context.config

target_metadata = SQLModel.metadata


def run_migrations_offline():
    url = config.get_main_option("sqlalchemy.url")

    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
        connect_args={"check_same_thread": False},
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Why `check_same_thread=False`?

SQLite's Python driver normally restricts a connection to the thread that created it.

```python
connect_args={"check_same_thread": False}
```

Disables that restriction.

This option is commonly used in FastAPI SQLite examples.

It does not make SQLite automatically safe for unlimited concurrent writes.

## 10.4 SQLite migration workflow

Initialize:

```bash
alembic init migrations
```

Generate migration:

```bash
alembic revision --autogenerate -m "create initial tables"
```

Inspect the generated file.

Apply:

```bash
alembic upgrade head
```

Check current revision:

```bash
alembic current
```

## 10.5 SQLite SQL commands

SQLite command-line example:

```bash
sqlite3 database.db
```

List tables:

```sql
.tables
```

Display schema:

```sql
.schema
```

Display a particular table:

```sql
.schema user
```

Query records:

```sql
SELECT * FROM user;
```

Display migration version:

```sql
SELECT * FROM alembic_version;
```

Exit:

```sql
.quit
```

## 10.6 SQLite limitations

SQLite is excellent for:

- Learning.
- Local development.
- Small applications.
- Testing.
- Prototypes.

However, SQLite has limitations around concurrent writes and certain schema alteration operations.

Alembic may use a batch migration strategy to handle some SQLite schema changes.

Example:

```python
def upgrade():
    with op.batch_alter_table("user") as batch_op:
        batch_op.add_column(
            sa.Column("email", sa.String(255))
        )
```

A batch operation can recreate a table when necessary instead of relying on a direct `ALTER TABLE`.

---

# SECTION 11: POSTGRESQL CONFIGURATION

## 11.1 What is PostgreSQL?

PostgreSQL is a powerful relational database management system.

Unlike SQLite, PostgreSQL normally runs as a database server.

Your FastAPI application connects to it using:

- Host.
- Port.
- Database name.
- Username.
- Password.
- Database driver.

Default PostgreSQL port:

```text
5432
```

## 11.2 Install PostgreSQL driver

Recommended modern Psycopg driver:

```bash
uv add "psycopg[binary]"
```

Or:

```bash
pip install "psycopg[binary]"
```

## 11.3 PostgreSQL connection URL

General format:

```text
postgresql+psycopg://username:password@host:port/database_name
```

Example:

```ini
sqlalchemy.url = postgresql+psycopg://postgres:password@localhost:5432/shop
```

Explanation:

| Component | Meaning |
|---|---|
| `postgresql` | Database dialect |
| `psycopg` | Driver |
| `postgres` | Database username |
| `password` | Database password |
| `localhost` | Database host |
| `5432` | PostgreSQL port |
| `shop` | Database name |

## 11.4 Create a PostgreSQL database

Open PostgreSQL using `psql`:

```bash
psql -U postgres
```

Create a database:

```sql
CREATE DATABASE shop;
```

Create a user:

```sql
CREATE USER shop_user WITH PASSWORD 'strong_password';
```

Grant privileges:

```sql
GRANT ALL PRIVILEGES ON DATABASE shop TO shop_user;
```

Connect to the database:

```sql
\c shop
```

For modern PostgreSQL installations, also ensure the application role has the necessary privileges on the schema:

```sql
GRANT USAGE, CREATE ON SCHEMA public TO shop_user;
```

If you use an existing schema with existing tables, the necessary table and sequence privileges may also need to be granted.

## 11.5 PostgreSQL alembic.ini

You can use:

```ini
[alembic]

script_location = %(here)s/migrations

prepend_sys_path = .

sqlalchemy.url = postgresql+psycopg://shop_user:strong_password@localhost:5432/shop
```

For production, load the URL from an environment variable instead of committing credentials.

## 11.6 PostgreSQL env.py

Use the same SQLAlchemy migration environment shown earlier.

The important difference is the database URL and installed driver.

A simplified example:

```python
from sqlalchemy import engine_from_config, pool

from alembic import context
from sqlmodel import SQLModel

from src.models import User, Product, Order

config = context.config

target_metadata = SQLModel.metadata


def run_migrations_offline():
    url = config.get_main_option("sqlalchemy.url")

    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

## 11.7 PostgreSQL SQL commands

List databases:

```sql
\l
```

Connect to a database:

```sql
\c shop
```

List tables:

```sql
\dt
```

Describe a table:

```sql
\d user
```

Display all columns and details:

```sql
\d+ user
```

List users:

```sql
\du
```

Query users:

```sql
SELECT * FROM "user";
```

Check Alembic version:

```sql
SELECT * FROM alembic_version;
```

Exit:

```sql
\q
```

**Note:** `user` can be a reserved or special identifier in some database contexts. Quoting it as `"user"` is a safe choice in PostgreSQL.

## 11.8 PostgreSQL migration workflow

```bash
alembic revision --autogenerate -m "initial postgres migration"
```

Inspect the generated migration.

Apply it:

```bash
alembic upgrade head
```

Check current revision:

```bash
alembic current
```

Review migration history:

```bash
alembic history
```

---

# SECTION 12: MYSQL CONFIGURATION

## 12.1 What is MySQL?

MySQL is a relational database management system.

Like PostgreSQL, MySQL normally runs as a database server.

Default MySQL port:

```text
3306
```

Your application connects using a MySQL driver.

## 12.2 Install PyMySQL

Recommended simple installation:

```bash
uv add pymysql
```

Or:

```bash
pip install pymysql
```

## 12.3 MySQL connection URL

General format:

```text
mysql+pymysql://username:password@host:port/database_name
```

Example:

```ini
sqlalchemy.url = mysql+pymysql://root:password@localhost:3306/shop
```

Explanation:

| Component | Meaning |
|---|---|
| `mysql` | Database dialect |
| `pymysql` | Python driver |
| `root` | Database username |
| `password` | Database password |
| `localhost` | Database host |
| `3306` | MySQL port |
| `shop` | Database name |

## 12.4 Create a MySQL database

Connect to MySQL:

```bash
mysql -u root -p
```

Create a database:

```sql
CREATE DATABASE shop;
```

Create a user:

```sql
CREATE USER 'shop_user'@'localhost'
IDENTIFIED BY 'strong_password';
```

Grant permissions:

```sql
GRANT ALL PRIVILEGES ON shop.* TO 'shop_user'@'localhost';
```

Apply privilege changes:

```sql
FLUSH PRIVILEGES;
```

Select the database:

```sql
USE shop;
```

## 12.5 MySQL alembic.ini

```ini
[alembic]

script_location = %(here)s/migrations

prepend_sys_path = .

sqlalchemy.url = mysql+pymysql://shop_user:strong_password@localhost:3306/shop
```

## 12.6 MySQL env.py

Use the same general online/offline environment as PostgreSQL.

For example:

```python
from sqlalchemy import engine_from_config, pool

from alembic import context
from sqlmodel import SQLModel

from src.models import User, Product, Order

config = context.config

target_metadata = SQLModel.metadata


def run_migrations_offline():
    url = config.get_main_option("sqlalchemy.url")

    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

## 12.7 MySQL SQL commands

Show databases:

```sql
SHOW DATABASES;
```

Select database:

```sql
USE shop;
```

Show tables:

```sql
SHOW TABLES;
```

Describe a table:

```sql
DESCRIBE user;
```

Alternative:

```sql
SHOW CREATE TABLE user;
```

Query records:

```sql
SELECT * FROM user;
```

Display migration revision:

```sql
SELECT * FROM alembic_version;
```

Show current database:

```sql
SELECT DATABASE();
```

Exit:

```sql
EXIT;
```

## 12.8 MySQL migration workflow

```bash
alembic revision --autogenerate -m "initial mysql migration"
```

Inspect the migration.

Apply it:

```bash
alembic upgrade head
```

Check:

```bash
alembic current
```

---

# SECTION 13: DATABASE CONFIGURATION COMPARISON

## 13.1 Connection URL comparison

| Database | SQLAlchemy URL |
|---|---|
| SQLite | `sqlite:///./database.db` |
| PostgreSQL | `postgresql+psycopg://user:password@localhost:5432/shop` |
| MySQL | `mysql+pymysql://user:password@localhost:3306/shop` |

## 13.2 Default ports

| Database | Default port |
|---|---:|
| SQLite | No network port |
| PostgreSQL | 5432 |
| MySQL | 3306 |

## 13.3 Database connection differences

| Feature | SQLite | PostgreSQL | MySQL |
|---|---|---|---|
| Server required | No | Yes | Yes |
| Storage | Database file | Database server | Database server |
| Python driver | Built-in `sqlite3` | Psycopg | PyMySQL |
| Default port | None | 5432 | 3306 |
| Alembic supported | Yes | Yes | Yes |

## 13.4 Important configuration rule

You normally do not need a completely different `env.py` for each database.

The same general Alembic environment can support all three.

You change:

1. The database URL.
2. The installed database driver.
3. Any database-specific connection arguments.
4. Migration operations when the database dialect requires special handling.

Your models and metadata can remain the same, although database-specific type and constraint behavior may differ.

---

# SECTION 14: SQL COMMANDS FOR DATABASE SCHEMA MANAGEMENT

Alembic generates migration operations that eventually translate into SQL statements.

Understanding SQL makes it easier to understand what Alembic is doing.

## 14.1 CREATE DATABASE

PostgreSQL:

```sql
CREATE DATABASE shop;
```

MySQL:

```sql
CREATE DATABASE shop;
```

SQLite:

```bash
sqlite3 shop.db
```

SQLite creates the database file when it is opened for use.

## 14.2 CREATE TABLE

```sql
CREATE TABLE user (
    id INTEGER PRIMARY KEY,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE
);
```

## 14.3 ALTER TABLE

Add a column:

```sql
ALTER TABLE user
ADD COLUMN age INTEGER;
```

Modify a column:

PostgreSQL:

```sql
ALTER TABLE user
ALTER COLUMN username TYPE VARCHAR(150);
```

MySQL:

```sql
ALTER TABLE user
MODIFY COLUMN username VARCHAR(150) NOT NULL;
```

SQLite has more limited direct column alteration support; some changes require table recreation.

## 14.4 DROP COLUMN

PostgreSQL:

```sql
ALTER TABLE user
DROP COLUMN age;
```

MySQL:

```sql
ALTER TABLE user
DROP COLUMN age;
```

SQLite support depends on the SQLite version and operation.

## 14.5 DROP TABLE

```sql
DROP TABLE user;
```

## 14.6 CREATE INDEX

```sql
CREATE INDEX ix_user_username
ON user(username);
```

## 14.7 CREATE FOREIGN KEY

Example:

```sql
CREATE TABLE album (
    id INTEGER PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    band_id INTEGER NOT NULL,
    FOREIGN KEY (band_id) REFERENCES band(id)
);
```

The exact type and generated syntax may vary between database dialects.

---

# SECTION 15: WORKING WITH SQLMODEL MODELS

## 15.1 Example models

Suppose you have:

```python
from sqlmodel import SQLModel, Field, Relationship


class User(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    username: str = Field(
        max_length=100,
        unique=True,
        index=True
    )

    email: str = Field(
        max_length=255,
        unique=True
    )

    orders: list["Order"] = Relationship(
        back_populates="user"
    )


class Order(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    total: float

    user_id: int = Field(
        foreign_key="user.id"
    )

    user: User = Relationship(
        back_populates="orders"
    )
```

The models define:

- Two tables.
- Primary keys.
- Unique constraints.
- An index.
- A foreign key.
- ORM relationships.

## 15.2 Import models in env.py

```python
from sqlmodel import SQLModel

from src.models.user import User
from src.models.order import Order

target_metadata = SQLModel.metadata
```

Without importing the model modules, their table definitions may not be registered.

## 15.3 Generate a migration

```bash
alembic revision --autogenerate -m "create user and order tables"
```

Alembic may generate something similar to:

```python
def upgrade():
    op.create_table(
        "user",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("username", sa.String(length=100), nullable=False),
        sa.Column("email", sa.String(length=255), nullable=False),
        sa.PrimaryKeyConstraint("id"),
        sa.UniqueConstraint("username"),
        sa.UniqueConstraint("email"),
    )

    op.create_table(
        "order",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("total", sa.Float(), nullable=False),
        sa.Column("user_id", sa.Integer(), nullable=False),
        sa.ForeignKeyConstraint(["user_id"], ["user.id"]),
        sa.PrimaryKeyConstraint("id"),
    )
```

The generated output can differ depending on the SQLModel, SQLAlchemy, Alembic versions and database dialect.

## 15.4 Apply the migration

```bash
alembic upgrade head
```

Your database tables are now created.

---

# SECTION 16: AUTOGENERATE AND METADATA

## 16.1 What does autogenerate do?

Command:

```bash
alembic revision --autogenerate -m "add email"
```

Autogenerate compares:

```text
Current SQLAlchemy Metadata
             |
             v
      Alembic Comparison
             |
             v
     Existing DB Schema
             |
             v
     Generated Migration
```

For example:

Existing database:

```sql
CREATE TABLE user (
    id INTEGER PRIMARY KEY,
    username VARCHAR(100)
);
```

Updated model:

```python
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)

    username: str = Field(max_length=100)

    email: str = Field(max_length=255)
```

Alembic may detect the missing email column.

Generated migration:

```python
def upgrade():
    op.add_column(
        "user",
        sa.Column("email", sa.String(length=255), nullable=False)
    )


def downgrade():
    op.drop_column("user", "email")
```

**Important:** Adding a non-nullable column to a populated table can fail if existing rows have no value for that column.

You may need a temporary server default, a nullable first migration followed by a data migration, or another controlled strategy.

## 16.2 Autogenerate is not perfect

Alembic cannot reliably infer every kind of change.

Always inspect generated migrations.

Examples requiring attention:

- Renaming a column.
- Renaming a table.
- Data transformations.
- Complex constraints.
- Database-specific operations.
- Changes to server defaults.
- Some enum changes.

For example, Alembic may interpret a column rename as:

```python
op.add_column(...)
op.drop_column(...)
```

That is not equivalent to a true rename because the old data may be lost.

A proper rename may require:

```python
op.alter_column(
    "user",
    "old_name",
    new_column_name="new_name"
)
```

Review the generated operations before applying them.

---

# SECTION 17: ALEMBIC MIGRATION OPERATIONS

Alembic provides an operation object called `op`.

You import it using:

```python
from alembic import op
```

## 17.1 Create a table

```python
op.create_table(
    "user",
    sa.Column("id", sa.Integer(), primary_key=True),
    sa.Column("username", sa.String(100), nullable=False),
)
```

## 17.2 Drop a table

```python
op.drop_table("user")
```

## 17.3 Add a column

```python
op.add_column(
    "user",
    sa.Column("email", sa.String(255), nullable=True)
)
```

## 17.4 Drop a column

```python
op.drop_column("user", "email")
```

## 17.5 Alter a column

```python
op.alter_column(
    "user",
    "username",
    existing_type=sa.String(100),
    type_=sa.String(150),
    existing_nullable=False,
)
```

## 17.6 Rename a column

```python
op.alter_column(
    "user",
    "username",
    new_column_name="display_name",
    existing_type=sa.String(100),
)
```

## 17.7 Create an index

```python
op.create_index(
    "ix_user_username",
    "user",
    ["username"],
    unique=False,
)
```

## 17.8 Drop an index

```python
op.drop_index(
    "ix_user_username",
    table_name="user",
)
```

## 17.9 Create a foreign key

```python
op.create_foreign_key(
    "fk_order_user",
    "order",
    "user",
    ["user_id"],
    ["id"],
)
```

## 17.10 Drop a foreign key

```python
op.drop_constraint(
    "fk_order_user",
    "order",
    type_="foreignkey",
)
```

## 17.11 Create a unique constraint

```python
op.create_unique_constraint(
    "uq_user_email",
    "user",
    ["email"],
)
```

## 17.12 Drop a unique constraint

```python
op.drop_constraint(
    "uq_user_email",
    "user",
    type_="unique",
)
```

## 17.13 Execute raw SQL

```python
op.execute(
    "UPDATE user SET username = 'unknown' WHERE username IS NULL"
)
```

Use raw SQL when an operation is easier to express directly in SQL.

Remember that raw SQL can be database-specific.

---

# SECTION 18: EDITING AND MANAGING EXISTING MIGRATIONS

## 18.1 Should I edit an existing migration?

It depends on whether it has already been applied.

If a migration has not been shared or applied to important databases, you may be able to edit it safely.

If it has already been applied in production or shared with teammates, normally create a new migration instead.

## 18.2 Example: adding an email column

Generate:

```bash
alembic revision -m "add email to user"
```

Edit the generated file:

```python
def upgrade():
    op.add_column(
        "user",
        sa.Column("email", sa.String(255), nullable=True)
    )


def downgrade():
    op.drop_column("user", "email")
```

Apply:

```bash
alembic upgrade head
```

## 18.3 Data migrations

Sometimes you need to change existing records, not just the table structure.

Example:

```python
def upgrade():
    op.execute(
        "UPDATE user SET username = 'unknown' WHERE username IS NULL"
    )
```

For more complex migrations, you can use SQLAlchemy connections and statements inside the migration.

Always plan how to reverse or safely handle the data transformation.

---

# SECTION 19: DATABASE VERSION COMMANDS

These commands help you understand the state of your migration history.

| Command | Purpose |
|---|---|
| `alembic current` | Current database revision |
| `alembic heads` | Latest revision heads |
| `alembic history` | Migration history |
| `alembic branches` | Migration branches |
| `alembic show head` | Details of latest revision |
| `alembic stamp head` | Mark database as latest without running migrations |
| `alembic upgrade head` | Apply all remaining migrations |
| `alembic downgrade -1` | Reverse one revision |

## 19.1 Current versus head

These are different concepts.

```bash
alembic current
```

Shows what the database has already applied.

```bash
alembic heads
```

Shows the latest revision available in your migration files.

Example:

```text
Database current revision:  a111
Latest migration head:      b222
```

This means the database has not yet applied the migration represented by `b222`.

Run:

```bash
alembic upgrade head
```

to apply the remaining migration.

---

# SECTION 20: DOWNGRADING AND ROLLING BACK MIGRATIONS

## 20.1 Why downgrade matters

Suppose you add a column:

```python
def upgrade():
    op.add_column(
        "user",
        sa.Column("phone", sa.String(20))
    )
```

A downgrade can reverse that change:

```python
def downgrade():
    op.drop_column("user", "phone")
```

## 20.2 Downgrade one revision

```bash
alembic downgrade -1
```

## 20.3 Downgrade to base

```bash
alembic downgrade base
```

## 20.4 Important warning

Downgrading can destroy data.

For example:

```python
def downgrade():
    op.drop_column("user", "email")
```

This removes the email column and its stored values.

A downgrade is not the same as restoring a database backup.

---

# SECTION 21: USING ALEMBIC WITH FASTAPI

## 21.1 Suggested project structure

```text
fastapi-project/
│
├── alembic.ini
│
├── migrations/
│   ├── versions/
│   ├── env.py
│   └── script.py.mako
│
├── src/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── database.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── product.py
│   │   └── order.py
│   │
│   ├── schemas/
│   └── routers/
│
├── .env
├── .gitignore
└── pyproject.toml
```

## 21.2 database.py

Example using SQLModel:

```python
from sqlmodel import create_engine, Session

DATABASE_URL = "sqlite:///./database.db"

engine = create_engine(
    DATABASE_URL,
    echo=True,
    connect_args={"check_same_thread": False},
)


def get_session():
    with Session(engine) as session:
        yield session
```

For PostgreSQL or MySQL, use the appropriate URL and driver.

## 21.3 main.py

```python
from fastapi import FastAPI

app = FastAPI(
    title="My API",
    version="1.0.0",
)


@app.get("/")
def home():
    return {
        "message": "FastAPI is running"
    }
```

## 21.4 Important: do not confuse create_all with migrations

You may see:

```python
SQLModel.metadata.create_all(engine)
```

This creates missing tables.

However, it is not a replacement for Alembic.

For example:

- `create_all()` can create a missing table.
- It does not provide a complete migration history.
- It does not automatically alter existing tables to match changed models.

When using Alembic to manage your schema, use migrations as the authoritative schema-change mechanism.

---

# SECTION 22: COMMON ERRORS AND SOLUTIONS

## Error 1: Target database is not up to date

Possible message:

```text
Target database is not up to date.
```

Meaning:

Your database revision does not match the expected migration state for the operation.

Check:

```bash
alembic current
```

Check history:

```bash
alembic history
```

Apply outstanding migrations if appropriate:

```bash
alembic upgrade head
```

## Error 2: No changes detected

Possible message:

```text
No changes detected
```

Possible causes:

1. Your models were not imported into `env.py`.
2. `target_metadata` is incorrect.
3. The database schema already matches your models.
4. You changed something Alembic does not automatically detect.

Check:

```python
target_metadata = SQLModel.metadata
```

Ensure all model modules are imported before this metadata is used.

## Error 3: Can't locate revision identified by

Possible message:

```text
Can't locate revision identified by '...'
```

Possible causes:

- A migration file was deleted.
- Migration history was changed.
- The database references a revision missing from the project.
- You are connecting to the wrong database.

Check:

```bash
alembic current
alembic history
alembic heads
```

Do not randomly delete the `alembic_version` table to fix this.

## Error 4: Could not connect to database

For PostgreSQL, check:

- PostgreSQL server is running.
- Host is correct.
- Port is correct.
- Database exists.
- Username and password are correct.
- Psycopg is installed.

For MySQL, check:

- MySQL server is running.
- Port is correct.
- Database exists.
- User has permission.
- PyMySQL is installed.

For SQLite, check:

- The database path is correct.
- The directory exists.
- The application has permission to create or modify the file.

## Error 5: ModuleNotFoundError

Example:

```text
ModuleNotFoundError: No module named 'src'
```

Possible causes:

- Your project root is not on Python's import path.
- Your package structure is incorrect.
- Your `prepend_sys_path` setting is wrong.
- You are running Alembic from an unexpected working directory.

Check:

```ini
prepend_sys_path = .
```

If you use a `src` layout, configure Python imports so that `src` is discoverable.

Also ensure package directories contain appropriate `__init__.py` files when needed.

## Error 6: NOT NULL constraint failed

Example:

```text
NOT NULL constraint failed: user.email
```

This can happen when you add a non-nullable column to a table that already contains records.

A safer migration may be:

1. Add the column as nullable.
2. Populate existing records.
3. Alter the column to non-nullable.

Example:

```python
def upgrade():
    op.add_column(
        "user",
        sa.Column("email", sa.String(255), nullable=True)
    )

    op.execute(
        "UPDATE user SET email = 'unknown@example.com' WHERE email IS NULL"
    )

    op.alter_column(
        "user",
        "email",
        existing_type=sa.String(255),
        nullable=False,
    )
```

The SQL and alteration behavior must be checked for your database dialect.

---

# SECTION 23: RECOMMENDED PROJECT STRUCTURE

For your FastAPI and SQLModel projects, I recommend separating database models, API schemas, and database configuration.

```text
project/
│
├── alembic.ini
│
├── migrations/
│   ├── versions/
│   ├── env.py
│   └── script.py.mako
│
├── src/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   └── config.py
│   │
│   ├── database/
│   │   ├── __init__.py
│   │   └── connection.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── band.py
│   │   └── album.py
│   │
│   ├── schemas/
│   │   ├── user.py
│   │   └── band.py
│   │
│   └── routers/
│       ├── users.py
│       └── bands.py
│
├── .env
├── .gitignore
└── pyproject.toml
```

The important relationship is:

```text
Models
  |
  v
SQLModel.metadata
  |
  v
Alembic env.py
  |
  v
Migration scripts
  |
  v
Database
```

---

# SECTION 24: COMPLETE MIGRATION WORKFLOW

This is the workflow you should understand and practice.

## Step 1: Install packages

For a PostgreSQL project:

```bash
uv add fastapi sqlmodel sqlalchemy alembic "psycopg[binary]"
```

## Step 2: Initialize Alembic

```bash
alembic init migrations
```

## Step 3: Configure the database URL

In `alembic.ini`, configure your database URL, or load it securely through `env.py`.

## Step 4: Import your models

In `migrations/env.py`:

```python
from sqlmodel import SQLModel
from src.models import User, Product, Order

target_metadata = SQLModel.metadata
```

## Step 5: Create your models

Example:

```python
from sqlmodel import SQLModel, Field


class User(SQLModel, table=True):
    id: int | None = Field(
        default=None,
        primary_key=True
    )

    username: str
```

## Step 6: Generate a migration

```bash
alembic revision --autogenerate -m "create user table"
```

## Step 7: Inspect the migration

Open:

```text
migrations/versions/
```

Review the generated Python file.

Check:

- `upgrade()`
- `downgrade()`
- Table names.
- Column types.
- Primary keys.
- Foreign keys.
- Nullable values.
- Indexes.
- Constraints.

## Step 8: Apply the migration

```bash
alembic upgrade head
```

## Step 9: Verify the database

PostgreSQL:

```sql
\dt
```

MySQL:

```sql
SHOW TABLES;
```

SQLite:

```sql
.tables
```

## Step 10: Modify a model

Add:

```python
email: str
```

## Step 11: Generate another migration

```bash
alembic revision --autogenerate -m "add email to user"
```

## Step 12: Inspect and apply

```bash
alembic upgrade head
```

## Step 13: Check migration history

```bash
alembic current
alembic history
```

---

# SECTION 25: IMPORTANT BEST PRACTICES

1. Always inspect an autogenerated migration before applying it.
2. Do not delete migration files that have already been applied or shared.
3. Use descriptive migration messages.
4. Keep `.env` and database credentials out of Git.
5. Import every model module required by `SQLModel.metadata`.
6. Use `compare_type=True` when you want Alembic to detect column-type changes.
7. Back up important production databases before risky migrations.
8. Do not use `alembic stamp head` as a substitute for applying missing migrations.
9. Understand the difference between `upgrade()` and `downgrade()`.
10. Do not rely on `SQLModel.metadata.create_all()` as your production migration system.
11. Review data migrations carefully; schema changes can affect existing records.
12. Use the correct database driver for your connection URL.
13. Test migrations against the same database system you intend to deploy.
14. Use environment variables for connection credentials.
15. Avoid modifying migration history that has already been applied to shared databases.

---

# SECTION 26: QUICK COMMAND REFERENCE

| Command | Meaning |
|---|---|
| `alembic init migrations` | Initialize Alembic |
| `alembic revision -m "message"` | Create empty migration |
| `alembic revision --autogenerate -m "message"` | Generate migration from metadata differences |
| `alembic upgrade head` | Apply all migrations |
| `alembic upgrade +1` | Upgrade one revision |
| `alembic upgrade <revision>` | Upgrade to a revision |
| `alembic downgrade -1` | Downgrade one revision |
| `alembic downgrade base` | Reverse all migrations |
| `alembic current` | Show current database revision |
| `alembic heads` | Show latest revision heads |
| `alembic history` | Show migration history |
| `alembic branches` | Show migration branches |
| `alembic show head` | Show latest revision details |
| `alembic stamp head` | Mark database as latest without executing migrations |
| `alembic upgrade head --sql` | Generate upgrade SQL |
| `alembic downgrade head:base --sql` | Generate downgrade SQL |

---

# SECTION 27: PRACTICAL EXERCISES

Use these exercises to confirm that you understand Alembic.

## Exercise 1: Initial migration

1. Create a SQLModel `User`.
2. Initialize Alembic.
3. Configure `env.py`.
4. Generate a migration.
5. Inspect the generated migration.
6. Apply it.
7. Verify that the table exists.

## Exercise 2: Add a column

Add:

```python
email: str
```

Then:

```bash
alembic revision --autogenerate -m "add email"
```

Inspect the generated migration and apply it.

## Exercise 3: Add a foreign key

Create two models:

```python
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)


class Post(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)

    user_id: int = Field(foreign_key="user.id")
```

Generate a migration.

Inspect the foreign key operation.

## Exercise 4: Downgrade

Run:

```bash
alembic downgrade -1
```

Inspect the database.

Then upgrade again:

```bash
alembic upgrade head
```

## Exercise 5: PostgreSQL configuration

1. Create a PostgreSQL database.
2. Install Psycopg.
3. Configure your connection URL.
4. Generate a migration.
5. Apply it.
6. Inspect tables using `psql`.

## Exercise 6: MySQL configuration

Repeat the same process using:

- MySQL.
- PyMySQL.
- A MySQL SQLAlchemy URL.
- MySQL command-line tools.

## Exercise 7: Compare SQL and Alembic

Write the SQL equivalent of these operations:

```python
op.create_table(...)
op.add_column(...)
op.drop_column(...)
op.create_index(...)
op.drop_table(...)
```

Then explain what each operation does to your database.

---

# FINAL CONCEPT TO REMEMBER

Alembic is not your database, and it is not your ORM.

Each component has a separate responsibility:

| Component | Responsibility |
|---|---|
| SQL | Language used to communicate with relational databases |
| SQLite | Database system |
| PostgreSQL | Database system |
| MySQL | Database system |
| SQLAlchemy | Python SQL toolkit and ORM |
| SQLModel | Pydantic and SQLAlchemy-based modeling library |
| Alembic | Database migration management |
| `alembic.ini` | Alembic configuration |
| `env.py` | Database connection and migration execution environment |
| `script.py.mako` | Template for generating migration files |
| `versions/` | Actual migration history |

The most important workflow is:

```text
1. Define or modify a model
             |
             v
2. Register the model in SQLModel.metadata
             |
             v
3. Alembic compares metadata with database schema
             |
             v
4. Generate a migration
             |
             v
5. Inspect the migration
             |
             v
6. Execute alembic upgrade head
             |
             v
7. Database schema is updated
```

**Remember:** Changing a Python model does not automatically change an existing database table. Generate and apply a migration to make that schema change.
