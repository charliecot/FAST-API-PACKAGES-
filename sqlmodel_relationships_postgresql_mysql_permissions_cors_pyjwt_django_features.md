# SQLModel Relationships, Database Integration, Security, and Django-Inspired Features

> Companion chapters for the FastAPI + Python typing + Pydantic + authentication guide. Examples use synchronous SQLModel sessions, PostgreSQL first, and MySQL second. Check the official documentation when package versions change.

## Contents

1. SQLModel relationships
2. PostgreSQL integration
3. MySQL integration
4. Authentication, permissions, and Django comparisons
5. CORS middleware
6. PyJWT explained
7. Django-inspired features in FastAPI
8. Recommended project structure and learning sequence

---

# 1. SQLModel relationships

## 1.1 Start with the relational database idea

A relational database stores data in tables. A row identifies one record. A primary key uniquely identifies a row. A foreign key stores a reference to a row in another table.

Example:

```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    status VARCHAR(30) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

`orders.user_id` references `users.id`. One user can own many orders, but each order belongs to one user.

```text
users                    orders
-----                    ------
id (PK)  1 ---------- *  user_id (FK)
username                 id (PK)
                         status
```

Important distinction:

- `Field(foreign_key="user.id")` declares a database column and foreign-key constraint.
- `Relationship(...)` defines ORM navigation between Python objects.
- A relationship attribute is not itself a database column.

## 1.2 One-to-many and many-to-one

```python
from sqlmodel import Field, Relationship, SQLModel

class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    username: str = Field(index=True, unique=True)

    orders: list["Order"] = Relationship(back_populates="user")


class Order(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    status: str = "pending"

    user_id: int = Field(foreign_key="user.id", index=True)
    user: User | None = Relationship(back_populates="orders")
```

Understand each attribute:

- `Order.user_id` is the actual foreign-key column.
- `Order.user` navigates from an order to its user.
- `User.orders` navigates from a user to their orders.
- `back_populates` connects the two ORM relationship attributes.

The foreign key belongs on the many side: each order stores the ID of its owner.

## 1.3 One-to-one

Example: one user has one profile. The foreign key needs a uniqueness constraint; otherwise several profiles could reference the same user.

```python
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    username: str

    profile: "Profile" | None = Relationship(back_populates="user")


class Profile(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    biography: str = ""

    user_id: int = Field(
        foreign_key="user.id",
        unique=True,
    )
    user: User | None = Relationship(back_populates="profile")
```

The `unique=True` constraint is what enforces one profile per user at database level. The Python type `Profile | None` says that the relationship may be absent.

## 1.4 Many-to-many and link tables

A user can join many teams, and each team can contain many users. Relational databases represent this using a third table.

```text
users             user_team              teams
-----             ---------              -----
id                user_id (PK, FK)        id
username          team_id (PK, FK)        name
                  joined_at
```

SQLModel's simple many-to-many pattern uses `link_model`:

```python
from datetime import datetime, timezone
from sqlmodel import Field, Relationship, SQLModel

class UserTeam(SQLModel, table=True):
    user_id: int = Field(foreign_key="user.id", primary_key=True)
    team_id: int = Field(foreign_key="team.id", primary_key=True)
    joined_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )


class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    username: str

    teams: list["Team"] = Relationship(
        back_populates="members",
        link_model=UserTeam,
    )


class Team(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str

    members: list[User] = Relationship(
        back_populates="teams",
        link_model=UserTeam,
    )
```

The composite primary key prevents the same user/team pair being inserted twice. If the association needs fields such as membership role, invitation status, or who added the member, model it as an explicit association object and query that model directly.

## 1.5 A store example: Order, Product, OrderItem

An order contains multiple products. A product can occur in multiple orders. This is many-to-many, but the link itself has data: quantity and the price at purchase time.

```python
class Product(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str
    unit_price: float

    order_items: list["OrderItem"] = Relationship(
        back_populates="product"
    )


class Order(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    status: str = "pending"
    user_id: int = Field(foreign_key="user.id", index=True)

    user: User | None = Relationship(back_populates="orders")
    items: list["OrderItem"] = Relationship(back_populates="order")


class OrderItem(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    order_id: int = Field(foreign_key="order.id", index=True)
    product_id: int = Field(foreign_key="product.id", index=True)
    quantity: int = Field(gt=0)
    unit_price_at_purchase: float

    order: Order | None = Relationship(back_populates="items")
    product: Product | None = Relationship(back_populates="order_items")
```

Why save `unit_price_at_purchase`? Product prices change. An old order should preserve the price the customer agreed to pay. For real money, use `Decimal` and an appropriate fixed-precision database column rather than binary floating-point `float`.

## 1.6 Persist related objects

```python
user = User(username="alice")
order = Order(status="pending", user=user)

session.add(order)
session.commit()
session.refresh(order)
```

The ORM tracks the relationship and can persist the associated object. You can also explicitly add both objects. For many-to-many:

```python
team = Team(name="Backend")
user = User(username="alice")
user.teams.append(team)

session.add(user)
session.commit()
```

Relationship collections behave like Python collections, but changes are tracked by the ORM.

## 1.7 Query relationships

```python
from sqlmodel import select

# Filter by the foreign key:
orders = session.exec(
    select(Order).where(Order.user_id == 1)
).all()

# Navigate through the relationship:
user = session.get(User, 1)
if user is not None:
    for order in user.orders:
        print(order.id, order.status)
```

Filtering on a foreign-key column is often simple and efficient. Relationship attributes are useful for navigation and relationship-aware queries.

## 1.8 Lazy loading, eager loading, and N+1

If you fetch 100 orders and then access `order.user` on each one, the ORM may issue additional queries. This can create the N+1 query problem.

Use eager loading when you know the related objects will be needed:

```python
from sqlalchemy.orm import selectinload
from sqlmodel import select

statement = (
    select(Order)
    .options(
        selectinload(Order.items).selectinload(OrderItem.product)
    )
)
orders = session.exec(statement).all()
```

`selectinload` loads related records in additional batched queries instead of one query per parent object. `joinedload` can be useful for many-to-one relationships:

```python
from sqlalchemy.orm import joinedload

statement = select(Order).options(joinedload(Order.user))
orders = session.exec(statement).all()
```

Choose loading strategies based on the relationship and response size. Do not eager-load every relationship automatically. With async SQLAlchemy, avoid accidental implicit lazy loading and use explicit loading strategies.

## 1.9 Delete behavior and cascade

Possible database foreign-key behaviors:

- `RESTRICT` / `NO ACTION`: prevent deleting a parent while dependent rows exist.
- `CASCADE`: delete dependent rows too.
- `SET NULL`: clear the foreign key; the column must allow NULL.

Do not casually cascade-delete important records. Deleting a user should not erase financial order history unless that is explicitly your business rule. Database `ON DELETE` behavior and ORM cascade behavior are related but distinct; configure and test them intentionally.

## 1.10 Relationship mistakes

1. Confusing `Field(foreign_key=...)` with `Relationship(...)`.
2. Forgetting the matching `back_populates`.
3. Using the wrong table/column name in a foreign key.
4. Forgetting uniqueness on a one-to-one foreign key.
5. Trying to model many-to-many without a link table.
6. Not preventing duplicate link rows.
7. Returning ORM objects without considering lazy loading or private fields.
8. Creating recursive response schemas (User -> Orders -> User -> Orders).
9. Not testing deletion and relationship loading against the real database.

---

# 2. PostgreSQL integration with SQLModel

## 2.1 Understand the layers

```text
FastAPI route
    |
SQLModel / SQLAlchemy ORM
    |
SQLAlchemy dialect
    |
Python database driver (psycopg)
    |
PostgreSQL server
```

SQLModel provides models and a convenient session API. SQLAlchemy manages the engine, connection pool, SQL generation, and ORM. The driver communicates with PostgreSQL.

## 2.2 Install packages

For synchronous SQLModel examples:

```bash
uv add sqlmodel "psycopg[binary]"
```

Psycopg 3 is the modern PostgreSQL driver. The binary extra is convenient for development; follow the driver's deployment guidance for production.

## 2.3 Create a database and user

As a PostgreSQL administrator:

```sql
CREATE USER api_user WITH PASSWORD 'replace-with-a-strong-secret';
CREATE DATABASE store_api OWNER api_user;
```

Use a real secret from environment configuration or a secret manager. Never commit database passwords.

## 2.4 Connection URL

```text
postgresql+psycopg://api_user:password@localhost:5432/store_api
```

- `postgresql`: SQLAlchemy dialect.
- `+psycopg`: driver.
- `api_user`: database user.
- `password`: database password.
- `localhost`: host.
- `5432`: default PostgreSQL port.
- `store_api`: database name.

Passwords containing characters such as `@`, `/`, `:` or `#` need URL encoding. Prefer `URL.create()` instead of manually concatenating credentials.

## 2.5 Environment configuration and engine

Install dotenv if you want local `.env` support:

```bash
uv add python-dotenv
```

`.env`:

```dotenv
DATABASE_URL=postgresql+psycopg://api_user:replace-me@localhost:5432/store_api
```

Add `.env` to `.gitignore`.

```python
import os
from dotenv import load_dotenv
from sqlmodel import create_engine

load_dotenv()

database_url = os.getenv("DATABASE_URL")
if not database_url:
    raise RuntimeError("DATABASE_URL is not configured")

engine = create_engine(
    database_url,
    echo=False,
    pool_pre_ping=True,
)
```

`pool_pre_ping=True` checks that a pooled connection is alive before use. It helps with stale connections but is not a substitute for database monitoring.

Safer URL construction when credentials may contain special characters:

```python
import os
from sqlalchemy.engine import URL
from sqlmodel import create_engine

url = URL.create(
    drivername="postgresql+psycopg",
    username=os.environ["DB_USER"],
    password=os.environ["DB_PASSWORD"],
    host=os.getenv("DB_HOST", "localhost"),
    port=int(os.getenv("DB_PORT", "5432")),
    database=os.environ["DB_NAME"],
)
engine = create_engine(url, pool_pre_ping=True)
```

## 2.6 Create tables while learning

Import every table model before calling `create_all`, so SQLModel metadata knows about them:

```python
from sqlmodel import SQLModel
from app.models import user, product, order  # noqa: F401

SQLModel.metadata.create_all(engine)
```

`create_all()` is useful for a first tutorial or disposable development database. It is not a production schema migration system. It does not safely manage a history of schema changes.

## 2.7 Session dependency

```python
from collections.abc import Generator
from sqlmodel import Session

def get_session() -> Generator[Session, None, None]:
    with Session(engine) as session:
        yield session
```

Inject it:

```python
from typing import Annotated
from fastapi import Depends
from sqlmodel import Session

SessionDep = Annotated[Session, Depends(get_session)]

@app.get("/products")
def list_products(session: SessionDep):
    return session.exec(select(Product)).all()
```

The engine is long-lived; sessions should be short-lived and scoped to a request or unit of work. Do not create one global session and share it among concurrent requests.

## 2.8 Commit, rollback, refresh

```python
def create_product(session: Session, product: Product) -> Product:
    session.add(product)
    try:
        session.commit()
        session.refresh(product)
    except Exception:
        session.rollback()
        raise
    return product
```

- `add()` stages an object in the session.
- `commit()` commits the transaction.
- `refresh()` reloads generated/defaulted values.
- `rollback()` abandons a failed transaction.

Group writes that must succeed together in one transaction. Do not leave a partially completed business operation.

## 2.9 Alembic migrations

Install and initialize:

```bash
uv add alembic
uv run alembic init migrations
```

In Alembic `env.py`, import all table models and set:

```python
from sqlmodel import SQLModel
target_metadata = SQLModel.metadata
```

Generate and review a migration:

```bash
uv run alembic revision --autogenerate -m "create initial tables"
uv run alembic upgrade head
```

Autogeneration is a proposal, not a guarantee. Review constraints, indexes, renames, data backfills, and destructive operations before applying a migration.

## 2.10 PostgreSQL features

Learn constraints, transactions, indexes, JSONB, full-text search, window functions, common table expressions, partial indexes, and query plans. Use normal columns and foreign keys for core entities. JSONB is useful for genuinely flexible data, not an excuse to store the whole relational design in one JSON column.

---

# 3. MySQL integration

The ORM concepts remain the same, but the dialect, driver, URL, and some database behaviors differ.

## 3.1 Install PyMySQL

```bash
uv add pymysql
```

PyMySQL is a pure-Python driver and is often straightforward to install. Another option is `mysqlclient`, which uses native components and may need system development libraries.

## 3.2 Create the database and account

As a MySQL administrator:

```sql
CREATE DATABASE store_api
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

CREATE USER 'api_user'@'localhost'
    IDENTIFIED BY 'replace-with-a-strong-password';

GRANT ALL PRIVILEGES ON store_api.* TO 'api_user'@'localhost';
```

Use least privilege in production. The application account should not be a global database administrator.

## 3.3 MySQL URL and engine

```text
mysql+pymysql://api_user:password@localhost:3306/store_api?charset=utf8mb4
```

- `mysql`: dialect.
- `+pymysql`: Python driver.
- `3306`: common MySQL port.
- `utf8mb4`: full Unicode character support.

Use `URL.create()` to safely handle credentials:

```python
import os
from sqlalchemy.engine import URL
from sqlmodel import create_engine

url = URL.create(
    drivername="mysql+pymysql",
    username=os.environ["DB_USER"],
    password=os.environ["DB_PASSWORD"],
    host=os.getenv("DB_HOST", "localhost"),
    port=int(os.getenv("DB_PORT", "3306")),
    database=os.environ["DB_NAME"],
    query={"charset": "utf8mb4"},
)

engine = create_engine(url, pool_pre_ping=True)
```

The same SQLModel `Session`, `select()`, `add()`, `commit()`, `refresh()`, and relationship patterns apply.

## 3.4 PostgreSQL versus MySQL

| Topic | PostgreSQL | MySQL |
|---|---|---|
| Default port | 5432 | 3306 |
| SQLAlchemy URL | `postgresql+psycopg://` | `mysql+pymysql://` |
| JSON | JSON and JSONB | JSON |
| Auto-generated integer | identity/serial patterns | AUTO_INCREMENT |
| Python ORM | SQLModel/SQLAlchemy | SQLModel/SQLAlchemy |

The ORM makes common CRUD portable, but differences remain in JSON querying, collations, enums, time zones, numeric precision, indexes, migrations, locking, and transaction isolation. Run integration tests against the actual database you plan to deploy. SQLite tests alone do not prove PostgreSQL or MySQL compatibility.

**Recommended learning order:** learn PostgreSQL as your primary database, then repeat the integration with MySQL to understand the driver/dialect changes. Do not add both databases to one application without a real architectural reason.

---

# 4. Authentication and permissions

## 4.1 Authentication versus authorization

Authentication asks: **Who are you?**

Authorization asks: **What are you allowed to do?**

A valid JWT does not mean the user may access every order or delete every product.

```text
HTTP request
  -> validate access token
  -> load current user
  -> check active account
  -> check role/permission
  -> check resource ownership
  -> execute endpoint
```

## 4.2 Django and DRF comparison

| Django / DRF | FastAPI approach |
|---|---|
| `request.user` | `current_user` dependency |
| `IsAuthenticated` | dependency requiring a valid user |
| `IsAdminUser` | role/permission dependency |
| custom permission class | reusable dependency |
| object-level permission | ownership/resource check |
| Groups | role/group tables |
| model permissions | permission records or policy checks |

FastAPI does not impose a single permission framework. You compose dependencies to implement your policy.

## 4.3 Define roles

```python
from enum import StrEnum

class UserRole(StrEnum):
    ADMIN = "admin"
    STAFF = "staff"
    CUSTOMER = "customer"
```

If every user has one role, a role field may be sufficient. If users have multiple roles, use a many-to-many relationship. For flexible permission assignment, consider `Role`, `Permission`, `UserRole`, and `RolePermission` tables.

## 4.4 Require an authenticated user

Assuming `get_current_user` validates the bearer token and loads the user:

```python
from typing import Annotated
from fastapi import Depends

CurrentUser = Annotated[User, Depends(get_current_user)]

@app.get("/me")
def read_my_profile(current_user: CurrentUser):
    return current_user
```

Do not trust a user ID sent in the request body as proof of identity.

## 4.5 Reusable role dependency

```python
from collections.abc import Callable
from typing import Annotated
from fastapi import Depends, HTTPException, status

def require_roles(*allowed_roles: UserRole):
    def role_checker(
        current_user: Annotated[User, Depends(get_current_user)],
    ) -> User:
        if current_user.role not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="You do not have permission",
            )
        return current_user
    return role_checker
```

Use it:

```python
AdminUser = Annotated[
    User,
    Depends(require_roles(UserRole.ADMIN)),
]

@app.delete("/admin/products/{product_id}")
def delete_product(product_id: int, admin: AdminUser):
    ...
```

This resembles a reusable DRF permission class. For production, type the dependency factory/callable precisely and verify it with your type checker.

## 4.6 Object-level permissions: protect orders

Unsafe: a logged-in user can request any order ID.

```python
@app.get("/orders/{order_id}")
def get_order(order_id: int, session: SessionDep):
    return session.get(Order, order_id)
```

Safer: scope the database query to the current user:

```python
from fastapi import HTTPException, status
from sqlmodel import select

@app.get("/orders/{order_id}")
def get_my_order(
    order_id: int,
    session: SessionDep,
    current_user: CurrentUser,
):
    statement = select(Order).where(
        Order.id == order_id,
        Order.user_id == current_user.id,
    )
    order = session.exec(statement).first()

    if order is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Order not found",
        )
    return order
```

This prevents a customer from changing the URL ID to read another customer's order. A 404 can avoid revealing whether another user's record exists.

## 4.7 401 versus 403 versus 404

- **401**: authentication is missing or invalid.
- **403**: the user is authenticated but not allowed to perform the action.
- **404**: may be used to avoid revealing the existence of another user's private resource.

Do not return HTTP 200 with an error message for a permission failure.

## 4.8 Roles versus permissions

A role groups permissions. A permission describes an action.

```text
editor:
  product:read
  product:create
  product:update

viewer:
  product:read

administrator:
  product:read
  product:create
  product:update
  product:delete
  user:manage
```

An enum role is a good starting point. Database-backed role and permission tables are more flexible but add joins, management screens, and cache invalidation. Add that complexity when requirements need it.

---

# 5. CORS middleware

## 5.1 What is an origin?

An origin is the combination of scheme, host, and port. These are different origins:

```text
http://localhost:5173
http://127.0.0.1:5173
http://localhost:3000
https://localhost:5173
```

A React/Vite frontend at `http://localhost:5173` and a FastAPI backend at `http://127.0.0.1:8000` are cross-origin.

## 5.2 Configure CORS

FastAPI uses Starlette's CORS middleware:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

origins = [
    "http://localhost:5173",
    "http://127.0.0.1:5173",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type"],
)
```

Options:

- `allow_origins`: exact browser origins allowed.
- `allow_credentials`: allows credentialed browser requests, such as cookies.
- `allow_methods`: HTTP methods allowed.
- `allow_headers`: request headers allowed.
- `expose_headers`: response headers browser JavaScript may read.
- `max_age`: duration the browser can cache preflight results.

When credentials are enabled, do not use wildcard `*` for allowed origins. Use explicit origins.

## 5.3 Preflight requests

For some cross-origin requests, the browser sends an `OPTIONS` preflight asking whether the real request is allowed. JSON requests with an Authorization header commonly trigger preflight. The middleware handles this; you normally do not create an `OPTIONS` route for every endpoint.

## 5.4 CORS is not authentication

CORS does not stop curl, Postman, Python requests, or another server from calling your API. It does not prevent an attacker from making requests outside a browser. It controls browser JavaScript's access to cross-origin responses.

Your backend still needs authentication, permissions, input validation, and ownership checks.

If using cookies, understand `HttpOnly`, `Secure`, `SameSite`, and CSRF protection. Cookies are automatically attached by browsers in some situations, so cookie-based authentication needs a deliberate CSRF strategy. If using bearer tokens in an Authorization header, allow that header in CORS and send it from the frontend.

---

# 6. PyJWT explained

## 6.1 JWT structure

A signed JWT has three Base64URL-encoded parts:

```text
header.payload.signature
```

Header example:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Payload example:

```json
{
  "sub": "42",
  "iss": "store-api",
  "aud": "store-frontend",
  "iat": 1795000000,
  "exp": 1795000900
}
```

Common claims:

- `sub`: subject, often the user's stable ID.
- `iss`: issuer.
- `aud`: intended audience.
- `iat`: issued-at time.
- `nbf`: not-before time.
- `exp`: expiration time.
- `jti`: unique token ID, useful for revocation tracking.

The payload is encoded, not encrypted. Anyone holding a normal signed JWT can decode its claims. Do not place passwords, password hashes, private keys, or confidential information in it.

The signature protects integrity when correctly validated. If someone changes the payload, signature validation should fail.

## 6.2 Install PyJWT and create a secret

```bash
uv add pyjwt
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Store the generated secret in an environment variable or secret manager. Never hardcode a production signing key in source control.

## 6.3 Create an access token

```python
import os
from datetime import datetime, timedelta, timezone
import jwt

SECRET_KEY = os.environ["JWT_SECRET_KEY"]
ALGORITHM = "HS256"
ACCESS_TOKEN_MINUTES = 15

def create_access_token(user_id: int) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": str(user_id),
        "iat": now,
        "exp": now + timedelta(minutes=ACCESS_TOKEN_MINUTES),
        "iss": "store-api",
        "aud": "store-frontend",
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
```

The `sub` claim is represented as a string. Use a stable identifier rather than a username or email that may change.

## 6.4 Decode and validate

```python
import jwt
from jwt import InvalidTokenError

def decode_access_token(token: str) -> dict:
    try:
        payload = jwt.decode(
            token,
            SECRET_KEY,
            algorithms=[ALGORITHM],
            issuer="store-api",
            audience="store-frontend",
            options={"require": ["sub", "iat", "exp"]},
        )
    except InvalidTokenError as exc:
        raise ValueError("Invalid or expired token") from exc

    return payload
```

Validation checklist:

1. Pin the allowed algorithm in server configuration.
2. Verify the signature.
3. Validate expiry.
4. Validate issuer and audience if used.
5. Require claims the application depends on.
6. Validate subject format and load the user.
7. Check that the account still exists and is active.

Never trust an algorithm selected by an untrusted token.

## 6.5 FastAPI bearer-token dependency

This example assumes `SessionDep`, `User`, and `SECRET_KEY` are configured:

```python
from typing import Annotated
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jwt import InvalidTokenError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    session: SessionDep,
) -> User:
    credentials_error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = jwt.decode(
            token,
            SECRET_KEY,
            algorithms=[ALGORITHM],
            issuer="store-api",
            audience="store-frontend",
            options={"require": ["sub", "iat", "exp"]},
        )
        user_id = int(payload["sub"])
    except (InvalidTokenError, KeyError, TypeError, ValueError) as exc:
        raise credentials_error from exc

    user = session.get(User, user_id)
    if user is None or not user.is_active:
        raise credentials_error
    return user
```

Adapt `is_active` to your user model. If your primary key is a UUID, parse a UUID instead of converting to `int`.

## 6.6 Access and refresh tokens

An access token is short-lived and accompanies API requests. A refresh token is longer-lived and is exchanged for a new access token.

A robust refresh design considers rotation, reuse detection, revocation, secure storage, expiry, absolute session lifetime, and replay handling.

A self-contained JWT is not automatically revoked when a user logs out. If immediate revocation is required, maintain server-side state such as a session table, token-ID denylist, or token version. A refresh token is not merely an access token with a longer expiry.

## 6.7 JWT mistakes

- Weak or committed signing secret.
- Accepting arbitrary algorithms.
- Not validating expiration.
- Putting sensitive data in the payload.
- Very long-lived access tokens without a reason.
- Trusting role claims indefinitely when permissions can change.
- Assuming logout invalidates a stateless token.
- Not checking that the user still exists and is active.
- Logging full bearer tokens.
- Sending tokens over plain HTTP in production.

For multi-service systems, asymmetric signing such as RS256 can be useful: the issuer keeps the private key and verification services use the public key. Key rotation must be designed carefully.

---

# 7. Django-inspired features in FastAPI

Django is batteries-included. FastAPI is smaller and lets you select components. Similar features can be built, but they are not all built into FastAPI.

## 7.1 Django Admin

Django includes an admin site. FastAPI does not include the same interface automatically.

Options include SQLAdmin for SQLAlchemy-oriented admin interfaces or a custom React admin dashboard using protected API endpoints. Admin interfaces must have authentication, authorization, and audit logging. An admin UI is not a replacement for API permissions.

## 7.2 Django choices and enums

```python
from enum import StrEnum

class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"
    SHIPPED = "shipped"
    CANCELLED = "cancelled"
```

Use the enum in Pydantic request schemas:

```python
from pydantic import BaseModel

class OrderStatusUpdate(BaseModel):
    status: OrderStatus
```

Decide whether to store a native database enum or a string plus a check constraint. Consider migration and database portability.

## 7.3 DRF serializers and forms

Pydantic models play a similar API-boundary role:

```python
from pydantic import BaseModel, ConfigDict, EmailStr

class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str
    email: EmailStr

class UserUpdate(BaseModel):
    username: str | None = None
    email: EmailStr | None = None
```

Use separate create, read, and update schemas. Never expose password hashes through public response schemas.

## 7.4 Django signals and application events

Django has signals such as `post_save`. SQLAlchemy has ORM events, but explicit service functions are often easier to reason about for important business operations:

```python
def register_user(session: Session, data: UserCreate) -> User:
    user = User(
        username=data.username,
        email=data.email,
        password_hash=hash_password(data.password),
    )
    session.add(user)
    session.commit()
    session.refresh(user)
    return user
```

For reliable email/task dispatch, learn the transactional outbox pattern: save the business change and an outbox event in the same transaction, then let a worker deliver it. This avoids losing an event if the process crashes between committing data and sending an email.

## 7.5 Middleware

Middleware wraps request/response processing. Starlette middleware can provide CORS, request IDs, timing, security headers, trusted-host checks, and carefully designed request logging.

Avoid putting all business authorization in middleware. FastAPI dependencies are generally better for route-specific authentication and permissions because they can use typed parameters and injected sessions.

## 7.6 Pagination

```python
from typing import Annotated
from fastapi import Query
from sqlmodel import select

@app.get("/products")
def list_products(
    session: SessionDep,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
):
    statement = select(Product).offset(offset).limit(limit)
    return session.exec(statement).all()
```

For large datasets, learn cursor/keyset pagination. Large offsets can become expensive and page boundaries can shift while records are inserted or deleted.

## 7.7 Filtering and search

```python
@app.get("/products")
def search_products(
    session: SessionDep,
    q: str | None = None,
):
    statement = select(Product)
    if q:
        statement = statement.where(Product.name.ilike(f"%{q}%"))
    return session.exec(statement).all()
```

Use ORM expressions/parameters; never concatenate untrusted input into raw SQL. Understand wildcard characters in `LIKE`/`ILIKE` if literal matching is required.

## 7.8 File uploads

FastAPI supports `UploadFile` and `File`. Production upload handling should validate size and content, distrust filenames and browser content types, generate server-side storage names, store files outside executable application directories, protect private downloads, and consider malware scanning. Object storage is useful at scale.

## 7.9 Caching

Redis can cache expensive or frequently used data. Cache keys must include relevant user/tenant/permission context for private responses. Invalidate cache entries when source data changes. Avoid serving one user's private data to another through a shared cache.

## 7.10 Management commands and background jobs

Django management commands can be approximated with Typer or Click CLI commands, standalone Python scripts, and Alembic for migrations. Use a task queue/worker for long-running background jobs rather than keeping an HTTP request open.

## 7.11 Testing

Use pytest and FastAPI's TestClient. Test:

- registration, password hashing, and login;
- expired, malformed, and wrong-audience tokens;
- 401 and 403 behavior;
- customer A cannot read customer B's order;
- role changes take effect;
- allowed and disallowed CORS origins;
- database constraints and rollback;
- migrations against the actual target database.

Test both the allowed action and the denied action. For example, admin can delete a product, customer cannot, and customer cannot access another customer's order by changing the URL ID.

---

# 8. Suggested project structure

```text
store_api/
├── app/
│   ├── main.py
│   ├── core/
│   │   ├── config.py
│   │   ├── security.py
│   │   └── permissions.py
│   ├── database/
│   │   ├── engine.py
│   │   └── session.py
│   ├── models/
│   │   ├── user.py
│   │   ├── product.py
│   │   ├── order.py
│   │   └── order_item.py
│   ├── schemas/
│   │   ├── user.py
│   │   ├── product.py
│   │   └── order.py
│   ├── api/routes/
│   │   ├── auth.py
│   │   ├── users.py
│   │   ├── products.py
│   │   └── orders.py
│   ├── services/
│   │   ├── auth.py
│   │   └── orders.py
│   └── dependencies/
│       └── auth.py
├── migrations/
├── tests/
├── .env
├── .env.example
├── .gitignore
└── pyproject.toml
```

Responsibilities:

- `models/`: database tables and relationships.
- `schemas/`: request/response validation.
- `database/`: engine and session lifecycle.
- `core/security.py`: password hashing and token functions.
- `dependencies/auth.py`: current-user dependency.
- `core/permissions.py`: role and permission dependencies.
- `services/`: business operations and transaction boundaries.
- `api/routes/`: HTTP endpoints.
- `migrations/`: Alembic revisions.
- `tests/`: unit and integration tests.

## Recommended learning sequence

1. Connect SQLModel to PostgreSQL.
2. Create User and Product tables.
3. Add Order and OrderItem relationships.
4. Practice CRUD and relationship queries.
5. Create separate Pydantic create/read/update schemas.
6. Add Alembic migrations.
7. Add registration and password hashing.
8. Add PyJWT access-token creation and validation.
9. Add the current-user dependency.
10. Add order ownership checks.
11. Add roles and reusable permission dependencies.
12. Configure CORS for the React development origin.
13. Add pagination and filtering.
14. Test relationships and authorization.
15. Repeat the database integration with MySQL.
16. Add admin UI, background tasks, caching, and file uploads after the core API is stable.

## Final mental model

- **Table model:** describes persisted data.
- **Foreign key:** creates a database-level link.
- **Relationship:** lets Python navigate related ORM objects.
- **Engine:** manages database connectivity and connection pooling.
- **Session:** tracks a unit of work and transaction.
- **Alembic:** versions schema changes.
- **Pydantic schema:** validates API input and controls API output.
- **Authentication:** establishes identity.
- **Authorization:** decides what that identity may do.
- **JWT:** a token format, not a complete authentication system.
- **CORS:** browser cross-origin response policy, not access control.
- **Middleware:** wraps request/response processing.
- **Service layer:** holds business operations and transaction logic.

## Official documentation

- SQLModel: https://sqlmodel.tiangolo.com/
- SQLAlchemy relationships: https://docs.sqlalchemy.org/en/20/orm/relationships.html
- SQLAlchemy engines: https://docs.sqlalchemy.org/en/20/core/engines.html
- PostgreSQL: https://www.postgresql.org/docs/
- MySQL: https://dev.mysql.com/doc/
- Alembic: https://alembic.sqlalchemy.org/
- FastAPI dependencies: https://fastapi.tiangolo.com/tutorial/dependencies/
- FastAPI CORS: https://fastapi.tiangolo.com/tutorial/cors/
- PyJWT: https://pyjwt.readthedocs.io/
- Starlette middleware: https://www.starlette.io/middleware/
