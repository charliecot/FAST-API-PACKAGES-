# FastAPI Ecosystem – Complete Python Package Reference

A practical reference for developers transitioning from Django / Django REST Framework to FastAPI.

---

# 1. FastAPI Core and ASGI Server

| Package | Purpose |
|---|---|
| `fastapi` | Main framework for building REST APIs |
| `uvicorn` | ASGI server for running FastAPI |
| `starlette` | ASGI toolkit underneath FastAPI |
| `pydantic` | Data validation, serialization and schemas |
| `pydantic-settings` | Application configuration and environment variables |
| `python-multipart` | Multipart form data and file uploads |
| `email-validator` | Validate email addresses |
| `httpx` | HTTP client for communicating with external APIs |
| `requests` | Synchronous HTTP requests |
| `python-dotenv` | Load environment variables from `.env` files |

### Installation

```bash
uv add "fastapi[standard]" pydantic-settings httpx requests python-dotenv email-validator
```

Note: FastAPI's standard extra already includes several of these dependencies.

# 2. Database Packages and ORMs

FastAPI does not provide its own ORM. You choose the database toolkit and driver.

## A. SQL databases

| Package | Database / Purpose |
|---|---|
| `sqlalchemy` | General-purpose Python ORM and SQL toolkit |
| `alembic` | SQLAlchemy database migrations |
| `psycopg[binary]` | PostgreSQL driver |
| `asyncpg` | Asynchronous PostgreSQL driver |
| `aiosqlite` | Asynchronous SQLite driver |
| `pymysql` | MySQL driver |
| `mysql-connector-python` | MySQL database connector |
| `oracledb` | Oracle database driver |
| `pyodbc` | SQL Server and other ODBC databases |
| `sqlmodel` | ORM and Pydantic integration built on SQLAlchemy |
| `databases` | Async database query library |

### Recommended SQL stack

```bash
uv add sqlalchemy alembic "psycopg[binary]"
```

For asynchronous PostgreSQL:

```bash
uv add asyncpg
```

For SQLModel:

```bash
uv add sqlmodel
```

**Important:** SQLAlchemy and SQLModel overlap. Learn SQLAlchemy first before introducing SQLModel.

## B. NoSQL databases

| Package | Database / Purpose |
|---|---|
| `pymongo` | MongoDB Python driver |
| `motor` | Older asynchronous MongoDB driver; deprecated in favor of PyMongo's async API |
| `redis` | Redis client |
| `beanie` | Asynchronous MongoDB ODM built on PyMongo |
| `odmantic` | MongoDB ODM using Pydantic |
| `cassandra-driver` | Apache Cassandra |
| `elasticsearch` | Elasticsearch Python client |

### Installation

```bash
uv add pymongo beanie redis elasticsearch
```

For MongoDB, choose either direct PyMongo or an ODM such as Beanie depending on your requirements.

## C. Database comparison

| Database | Package | Type |
|---|---|---|
| PostgreSQL | `psycopg` | Relational |
| PostgreSQL | `asyncpg` | Relational |
| MySQL | `pymysql` | Relational |
| SQLite | `aiosqlite` | Relational |
| MongoDB | `pymongo` | Document database |
| Redis | `redis` | Key-value / in-memory data store |
| Cassandra | `cassandra-driver` | Wide-column |
| Elasticsearch | `elasticsearch` | Search engine |

---

# 3. Authentication and Authorization Packages

Authentication establishes who a user is. Authorization determines what that user is permitted to do.

## A. JWT Authentication

| Package | Purpose |
|---|---|
| `PyJWT` | Generate, decode and verify JWT tokens |
| `python-jose` | JWT and JOSE operations; older projects commonly use it |
| `authlib` | OAuth, OpenID Connect and JOSE integrations |
| `joserfc` | Modern JOSE implementation associated with Authlib |
| `cryptography` | Cryptographic primitives and key handling |

Recommended:

```bash
uv add "PyJWT[crypto]" cryptography
```

## B. Password Hashing

| Package | Purpose |
|---|---|
| `pwdlib` | Modern password hashing interface |
| `argon2-cffi` | Argon2 password hashing |
| `bcrypt` | Bcrypt password hashing |
| `passlib` | Password hashing utilities; widely used in older applications |

Recommended:

```bash
uv add "pwdlib[argon2]"
```

## C. OAuth2 and OpenID Connect

These packages help integrate external identity providers.

| Package | Purpose |
|---|---|
| `authlib` | OAuth2 client, server and OpenID Connect |
| `httpx` | Communicate with external identity providers |
| `PyJWT` | Validate JWTs issued by supported providers |
| `jwcrypto` | JSON Web Encryption and JSON Web Signature operations |
| `python-keycloak` | Integrate with Keycloak |
| `msal` | Microsoft Authentication Library |
| `google-auth` | Google authentication and token verification |
| `google-auth-oauthlib` | Google OAuth2 flows |
| `firebase-admin` | Firebase Authentication and administration |
| `python-social-auth` | Social authentication framework |

Recommended:

```bash
uv add authlib httpx
```

Additional providers:

```bash
uv add msal google-auth google-auth-oauthlib
```

For Keycloak:

```bash
uv add python-keycloak
```

For Firebase:

```bash
uv add firebase-admin
```

## D. External Authentication Providers

| Provider | Package / Integration |
|---|---|
| Google | `authlib`, `google-auth` |
| GitHub | `authlib` |
| Facebook | `authlib` |
| Microsoft | `msal`, `authlib` |
| Apple | `authlib` |
| Keycloak | `python-keycloak` |
| Firebase | `firebase-admin` |
| Auth0 | `authlib` or provider SDK |
| Okta | OIDC through `authlib` or provider SDK |
| Amazon Cognito | OIDC/OAuth2 through `authlib` |
| Supabase Auth | Supabase client / JWT verification |
| Clerk | Clerk SDK or JWT verification |
| WorkOS | WorkOS SDK |

**Important:** External authentication is not the same as local username/password authentication.

For example, Google OAuth2 allows a user to authenticate with Google. Your application still needs to establish its own user account, map the external identity to that account and enforce its own authorization rules.

Never trust an unverified token or merely decode a JWT without validating its signature, issuer, audience, expiry and other relevant claims.

---

# 4. Authorization and Access Control

| Package / Tool | Purpose |
|---|---|
| FastAPI `Depends()` | Dependency injection for authentication and authorization |
| FastAPI `Security()` | Security dependency integration |
| `casbin` | Access-control policies |
| `oso` | Authorization policy engine |
| `py-abac` | Attribute-based access control |
| `python-keycloak` | Keycloak roles and permission integration |

Authorization approaches:

- RBAC: Role-Based Access Control.
- ABAC: Attribute-Based Access Control.
- PBAC: Policy-Based Access Control.
- Ownership-based permissions.

Example roles:

```text
ADMIN
MANAGER
CUSTOMER
STAFF
```

---

# 5. Testing Packages

| Package | Purpose |
|---|---|
| `pytest` | Main testing framework |
| `httpx` | HTTP client |
| `pytest-asyncio` | Asynchronous testing |
| `coverage` | Measure code coverage |
| `factory-boy` | Generate test data |
| `faker` | Generate fake data |
| `respx` | Mock HTTPX requests |
| `freezegun` | Mock dates and times |
| `hypothesis` | Property-based testing |

Install:

```bash
uv add --dev pytest pytest-asyncio coverage factory-boy faker respx freezegun hypothesis
```

FastAPI testing:

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_home():
    response = client.get("/")

    assert response.status_code == 200
```

---

# 6. Background Tasks and Job Queues

| Package | Purpose |
|---|---|
| `celery` | Distributed task processing |
| `redis` | Message broker and result backend |
| `rq` | Redis-backed job queue |
| `arq` | Async Redis job queue |
| `dramatiq` | Background task processing |
| `apscheduler` | Schedule periodic jobs |
| `huey` | Lightweight task queue |

Install:

```bash
uv add celery redis
```

Use cases:

- Sending emails.
- Generating invoices.
- Processing uploaded files.
- Sending notifications.
- Processing payment events.
- Running scheduled jobs.

FastAPI also provides `BackgroundTasks` for small tasks that do not require a distributed worker.

---

# 7. Caching and Rate Limiting

| Package | Purpose |
|---|---|
| `redis` | Cache storage |
| `fastapi-cache2` | FastAPI response caching |
| `slowapi` | API rate limiting |
| `limits` | Rate-limiting primitives |
| `cachetools` | In-memory caching |
| `diskcache` | Persistent disk-based caching |

Install:

```bash
uv add redis fastapi-cache2 slowapi cachetools diskcache
```

Example use cases:

- Cache product listings.
- Cache frequently requested user information.
- Limit login attempts.
- Limit API requests per user.
- Reduce database load.

---

# 8. File Uploads and Storage

| Package | Purpose |
|---|---|
| `python-multipart` | Multipart uploads |
| `Pillow` | Image processing |
| `aiofiles` | Asynchronous file operations |
| `boto3` | AWS S3 |
| `minio` | MinIO object storage |
| `cloudinary` | Cloudinary media management |
| `python-magic` | Detect file types |
| `pypdf` | Read and manipulate PDF files |

Install:

```bash
uv add pillow aiofiles boto3 minio cloudinary python-magic pypdf
```

Security considerations:

- Validate file extensions.
- Validate actual file content and MIME types.
- Limit file size.
- Generate safe filenames.
- Avoid executing uploaded files.
- Store sensitive files outside publicly accessible directories.

---

# 9. Email and Notifications

| Package | Purpose |
|---|---|
| `fastapi-mail` | Email integration |
| `aiosmtplib` | Asynchronous SMTP |
| `email-validator` | Validate email addresses |
| `sendgrid` | SendGrid email API |
| `boto3` | AWS SES integration |
| `twilio` | SMS and messaging |
| `firebase-admin` | Firebase push notifications |

Install:

```bash
uv add fastapi-mail aiosmtplib email-validator twilio
```

---

# 10. HTTP Requests and External APIs

| Package | Purpose |
|---|---|
| `httpx` | Synchronous and asynchronous HTTP |
| `requests` | Synchronous HTTP |
| `aiohttp` | Asynchronous HTTP client/server |
| `websockets` | WebSocket communication |
| `tenacity` | Retry failed operations |
| `backoff` | Retry with backoff strategies |

Install:

```bash
uv add httpx aiohttp websockets tenacity backoff
```

Use these when integrating:

- Payment gateways.
- External authentication.
- Shipping services.
- Third-party APIs.
- Other microservices.

---

# 11. Logging, Monitoring and Error Tracking

| Package | Purpose |
|---|---|
| `logging` | Python built-in logging |
| `loguru` | Simplified logging |
| `structlog` | Structured logging |
| `sentry-sdk` | Error monitoring |
| `prometheus-client` | Metrics collection |
| `opentelemetry-api` | Observability API |
| `opentelemetry-sdk` | Observability implementation |
| `opentelemetry-instrumentation-fastapi` | FastAPI tracing instrumentation |

Install:

```bash
uv add loguru sentry-sdk prometheus-client
```

For OpenTelemetry:

```bash
uv add opentelemetry-api opentelemetry-sdk opentelemetry-instrumentation-fastapi
```

---

# 12. API Documentation and Serialization

| Package | Purpose |
|---|---|
| `pydantic` | Request and response validation |
| `orjson` | Fast JSON serialization |
| `msgspec` | Fast serialization and validation |
| `openapi-spec-validator` | Validate OpenAPI specifications |

Install:

```bash
uv add orjson msgspec openapi-spec-validator
```

FastAPI automatically generates OpenAPI documentation and provides Swagger UI and ReDoc.

---

# 13. Middleware and Security

| Package / Tool | Purpose |
|---|---|
| `starlette` | ASGI middleware and application utilities |
| `fastapi` | Dependency and security integration |
| `slowapi` | Rate limiting |
| `secure` | Security headers |
| `python-multipart` | Multipart form parsing |
| `cryptography` | Cryptographic operations |
| `bleach` | HTML sanitization; evaluate its maintenance and suitability before adopting |
| `defusedxml` | Safer XML parsing |

Useful security controls:

- CORS configuration.
- Trusted host validation.
- HTTPS enforcement.
- Secure cookie settings.
- CSRF protection where cookie-based authentication requires it.
- Rate limiting.
- Input validation.
- SQL injection prevention through parameterized queries and ORM usage.
- Password hashing.
- Secure token validation.

---

# 14. Environment and Configuration

| Package | Purpose |
|---|---|
| `pydantic-settings` | Typed application settings |
| `python-dotenv` | Load `.env` files |
| `environs` | Environment configuration |
| `dynaconf` | Configuration management |
| `python-decouple` | Separate settings from source code |

Recommended:

```bash
uv add pydantic-settings python-dotenv
```

Example `.env`:

```env
APP_NAME=FastAPI Application
DEBUG=True

DATABASE_URL=postgresql+psycopg://postgres:password@localhost:5432/shop

JWT_SECRET_KEY=replace_with_a_secure_secret
JWT_ALGORITHM=HS256

REDIS_URL=redis://localhost:6379/0
```

Never commit real secrets or production credentials to Git.

---

# 15. WebSockets and Real-Time Communication

| Package | Purpose |
|---|---|
| `websockets` | WebSocket protocol |
| `python-socketio` | Socket.IO server/client |
| `redis` | Pub/Sub and message coordination |

Use cases:

- Chat applications.
- Live notifications.
- Real-time dashboards.
- Live order tracking.
- Collaborative applications.

---

# 16. GraphQL

| Package | Purpose |
|---|---|
| `strawberry-graphql` | GraphQL integration with FastAPI |
| `ariadne` | GraphQL server library |
| `graphql-core` | GraphQL implementation |

Install:

```bash
uv add strawberry-graphql
```

GraphQL is an alternative API approach, not a replacement requirement for REST.

---

# 17. Deployment and Production

| Package / Tool | Purpose |
|---|---|
| `uvicorn` | ASGI server |
| `gunicorn` | Process manager with Uvicorn workers |
| `granian` | ASGI/WSGI server |
| `docker` | Application containerization |
| `nginx` | Reverse proxy |
| `apache` | Web server and reverse proxy |
| `sentry-sdk` | Production error tracking |

For a typical deployment:

```text
Internet
   |
   v
Domain / HTTPS
   |
   v
Nginx
   |
   v
Uvicorn
   |
   v
FastAPI
   |
   +------ PostgreSQL
   |
   +------ Redis
   |
   +------ Celery Workers
```

Install server packages:

```bash
uv add "fastapi[standard]"
```

---

# 18. Recommended Installation for Your First FastAPI Project

You do not need every package in this document.

Start with this practical stack:

```bash
# Framework
uv add "fastapi[standard]"

# Configuration
uv add pydantic-settings python-dotenv

# Database
uv add sqlalchemy alembic "psycopg[binary]"

# Authentication
uv add "PyJWT[crypto]" "pwdlib[argon2]" authlib

# HTTP
uv add httpx

# Caching and background processing
uv add redis celery

# File handling
uv add pillow aiofiles

# Testing
uv add --dev pytest pytest-asyncio coverage

# Code quality
uv add --dev ruff
```

---

# 19. Django REST Framework vs FastAPI Package Mapping

| Django / DRF | FastAPI ecosystem |
|---|---|
| Django | FastAPI |
| Django REST Framework | FastAPI |
| Django ORM | SQLAlchemy |
| DRF Serializers | Pydantic |
| Django Migrations | Alembic |
| SimpleJWT | PyJWT + FastAPI security |
| Django Allauth | Authlib / provider integrations |
| Djoser | No direct equivalent; build authentication endpoints or use an appropriate auth solution |
| django-filter | FastAPI query parameters / SQLAlchemy filtering |
| Django Cache Framework | Redis + fastapi-cache2 |
| Celery | Celery |
| django-cors-headers | Starlette CORSMiddleware |
| Django Signals | Explicit event handlers / application events |
| Django Admin | No built-in equivalent; add an admin package or build an admin interface |
| Django TestCase | Pytest |
| django-storages | boto3 / storage SDKs |

---

# 20. Final Learning Advice

Learn these technologies in this order:

1. FastAPI routing and HTTP methods.
2. Pydantic validation and serialization.
3. Dependency injection using `Depends()`.
4. SQLAlchemy and PostgreSQL.
5. Alembic migrations.
6. Authentication using JWT.
7. External authentication using OAuth2 and OpenID Connect.
8. Authorization using roles and permissions.
9. Redis and caching.
10. Celery and background processing.
11. File uploads and email.
12. Pytest and API testing.
13. Docker and Linux deployment.

**Recommended first project: E-Commerce REST API**

Build:

- User registration and login.
- Google OAuth login.
- JWT access and refresh tokens.
- User roles and permissions.
- Products and categories.
- Shopping cart.
- Orders and order items.
- PostgreSQL database.
- Redis caching.
- Celery email notifications.
- File uploads.
- API testing.
- Docker deployment.

The goal is not to install as many packages as possible. The goal is to understand what problem each package solves, how it integrates with FastAPI, and when you actually need it.
