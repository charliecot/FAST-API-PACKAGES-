# Testing FastAPI Applications with pytest — Detailed Guide

## 1. Why testing matters

A test checks that a small unit or a complete application behaves as expected. Tests help detect regressions when you change code, add endpoints, change database models, or introduce authentication.

A useful testing strategy has several levels:

- **Unit tests**: test one function or small unit in isolation.
- **Integration tests**: test components working together, such as an endpoint and a test database.
- **API tests**: send HTTP requests to the FastAPI application and inspect responses.
- **End-to-end tests**: test a larger workflow from the client's perspective, often against a deployed-like environment.

pytest is a popular Python test runner. FastAPI provides `TestClient` for synchronous HTTP-style tests.

## 2. Install the tools

```bash
uv add --dev pytest httpx
```

If you use `uv`, run tests with:

```bash
uv run pytest
```

Common project structure:

```text
project/
├── app/
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   └── routers/
├── tests/
│   ├── conftest.py
│   ├── test_health.py
│   └── test_bands.py
└── pyproject.toml
```

pytest normally discovers files named `test_*.py` or `*_test.py`, and test functions named `test_*`.

## 3. Your first test

Application:

```python
# app/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def home():
    return {"message": "API is running"}
```

Test:

```python
# tests/test_health.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_home_returns_message():
    response = client.get("/")

    assert response.status_code == 200
    assert response.json() == {"message": "API is running"}
```

Run:

```bash
uv run pytest
```

Understand the assertions:

- `client.get("/")` makes a test HTTP request.
- `response.status_code` is the HTTP status.
- `response.json()` parses the JSON response body.
- `assert` checks an expected condition. A failed assertion makes the test fail.

`TestClient` runs requests against the ASGI application without requiring you to start Uvicorn in another terminal.

## 4. Test different HTTP methods

```python
def test_create_band():
    response = client.post(
        "/bands/",
        json={"name": "Coldplay", "genre": "Rock"},
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Coldplay"

def test_get_bands():
    response = client.get("/bands/")
    assert response.status_code == 200

def test_missing_band():
    response = client.get("/bands/99999")
    assert response.status_code == 404
```

Use `json=` to send JSON. Use `params=` for query parameters:

```python
response = client.get("/bands/", params={"genre": "Rock"})
```

Use `headers=` to provide headers:

```python
response = client.get(
    "/private/",
    headers={"Authorization": "Bearer test-token"},
)
```

## 5. Fixtures — reusable setup

A fixture prepares something that tests need. Examples include a test client, database, user, or authenticated token.

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def client():
    return TestClient(app)
```

Use it:

```python
def test_home(client):
    response = client.get("/")
    assert response.status_code == 200
```

pytest creates the fixture when requested by a test function. The fixture name becomes a function argument.

### Fixture scopes

```python
@pytest.fixture(scope="function")
def resource():
    ...
```

Common scopes:

- `function` (default): once per test function.
- `class`: once per test class.
- `module`: once per module.
- `package`: once per package.
- `session`: once per test session.

Use broader scopes only when sharing is safe. A database fixture often needs per-test isolation.

### Setup and teardown with `yield`

```python
@pytest.fixture
def temporary_resource():
    resource = create_resource()  # setup
    yield resource                # test uses it
    resource.close()              # teardown
```

Code before `yield` is setup; code after it is cleanup.

## 6. Test a SQLModel application with a separate SQLite database

Do not let tests accidentally modify your development or production database. Configure a separate test database.

Example database module:

```python
# app/database.py
from sqlmodel import SQLModel, create_engine, Session

DATABASE_URL = "sqlite:///./app.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False},
)

def get_session():
    with Session(engine) as session:
        yield session
```

Example application:

```python
# app/main.py
from fastapi import FastAPI, Depends
from sqlmodel import Session, select
from app.database import get_session
from app.models import Band

app = FastAPI()

@app.get("/bands/")
def list_bands(session: Session = Depends(get_session)):
    return session.exec(select(Band)).all()
```

### Override the database dependency

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlmodel import SQLModel, Session, create_engine
from sqlalchemy.pool import StaticPool

from app.main import app
from app.database import get_session

@pytest.fixture
def test_engine():
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    SQLModel.metadata.create_all(engine)
    yield engine
    SQLModel.metadata.drop_all(engine)
    engine.dispose()

@pytest.fixture
def client(test_engine):
    def override_get_session():
        with Session(test_engine) as session:
            yield session

    app.dependency_overrides[get_session] = override_get_session

    with TestClient(app) as test_client:
        yield test_client

    app.dependency_overrides.clear()
```

Why `sqlite://` and `StaticPool`? This creates an in-memory SQLite database whose connection is shared across the test client's threads. SQLite's `check_same_thread=False` allows the connection to be used across threads. This is a common lightweight testing setup; production PostgreSQL behaviour should also be tested in an integration environment.

Import all table models before `SQLModel.metadata.create_all(engine)` so their tables are registered in metadata.

## 7. Test create, read, update, and delete

```python
def test_band_crud(client):
    # CREATE
    created = client.post(
        "/bands/",
        json={"name": "Coldplay", "genre": "Rock"},
    )
    assert created.status_code == 201
    band_id = created.json()["id"]

    # READ
    fetched = client.get(f"/bands/{band_id}")
    assert fetched.status_code == 200
    assert fetched.json()["name"] == "Coldplay"

    # PATCH
    updated = client.patch(
        f"/bands/{band_id}",
        json={"genre": "Alternative Rock"},
    )
    assert updated.status_code == 200
    assert updated.json()["genre"] == "Alternative Rock"
    assert updated.json()["name"] == "Coldplay"

    # DELETE
    deleted = client.delete(f"/bands/{band_id}")
    assert deleted.status_code in (200, 204)

    # READ after deletion
    missing = client.get(f"/bands/{band_id}")
    assert missing.status_code == 404
```

The exact expected status code depends on your API contract.

## 8. Parametrization — run one test with multiple inputs

```python
import pytest

@pytest.mark.parametrize(
    "value, expected",
    [
        ("rock", "ROCK"),
        ("jazz", "JAZZ"),
        ("pop", "POP"),
    ],
)
def test_uppercase(value, expected):
    assert value.upper() == expected
```

This avoids copying nearly identical tests.

You can also parametrize API status cases:

```python
@pytest.mark.parametrize(
    "payload",
    [
        {},
        {"name": ""},
        {"name": "Coldplay", "unexpected": "value"},
    ],
)
def test_invalid_band_payload(client, payload):
    response = client.post("/bands/", json=payload)
    assert response.status_code in (422, 400)
```

Choose exact expected results based on your validation rules rather than accepting broad ranges in a mature test suite.

## 9. Test validation and errors

```python
def test_invalid_id_type(client):
    response = client.get("/bands/not-an-integer")
    assert response.status_code == 422

def test_unknown_band_returns_404(client):
    response = client.get("/bands/987654")
    assert response.status_code == 404
    assert response.json()["detail"] == "Band not found"
```

Test both the status code and useful response details. Avoid asserting unstable internal traceback text.

## 10. Dependency overrides

FastAPI dependencies are especially easy to replace in tests.

```python
from fastapi import Depends

def get_current_user():
    ...

@app.get("/me")
def read_me(user=Depends(get_current_user)):
    return {"username": user.username}
```

Test override:

```python
def fake_current_user():
    return {"username": "test-user"}

app.dependency_overrides[get_current_user] = fake_current_user
```

If your endpoint expects an object with attributes, return an appropriate test object rather than a dictionary. Always clear overrides after tests, preferably through a fixture's teardown.

## 11. Mock external services

Do not send real emails or charge real payment services during ordinary unit tests.

Python's `unittest.mock` supplies `Mock`, `MagicMock`, and `patch`.

```python
from unittest.mock import patch

def send_welcome_email(email: str):
    ...

def register_user(email: str):
    # save user...
    send_welcome_email(email)

@patch("app.users.send_welcome_email")
def test_register_sends_email(mock_send):
    register_user("person@example.com")
    mock_send.assert_called_once_with("person@example.com")
```

Patch the name where the code under test looks it up, not necessarily where the original function was first defined.

Useful mock assertions:

```python
mock.assert_called_once()
mock.assert_called_once_with("expected")
assert mock.call_count == 2
```

Mocks verify interactions; they do not prove that the real external service works. Keep a smaller number of integration tests for real service boundaries where practical.

## 12. Test authentication and permissions

Examples:

```python
def test_private_endpoint_without_token(client):
    response = client.get("/private/")
    assert response.status_code == 401

def test_user_cannot_read_another_users_record(client, user_token):
    response = client.get(
        "/users/another-user/private-data",
        headers={"Authorization": f"Bearer {user_token}"},
    )
    assert response.status_code == 403
```

The distinction matters:

- `401 Unauthorized`: authentication is missing or invalid.
- `403 Forbidden`: identity is known, but the user is not allowed to perform the action.

Some authentication schemes/framework configurations may use 401 or 403 differently; define and test your API's behaviour consistently.

Test owner access, non-owner access, admin access, expired tokens, malformed tokens, and inactive users.

## 13. Async tests

For async endpoints or async database clients, use `httpx.AsyncClient` with `ASGITransport` and pytest-asyncio.

Install:

```bash
uv add --dev pytest-asyncio httpx
```

Example:

```python
import pytest
from httpx import ASGITransport, AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_async_endpoint():
    transport = ASGITransport(app=app)

    async with AsyncClient(
        transport=transport,
        base_url="http://test",
    ) as client:
        response = await client.get("/")

    assert response.status_code == 200
```

Depending on your installed pytest-asyncio version/configuration, configure its asyncio mode in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

Do not use `TestClient` and async test patterns randomly together. Use the synchronous client for ordinary synchronous tests; use an async client when the test itself must await asynchronous operations.

## 14. Useful pytest commands

```bash
uv run pytest
uv run pytest -v
uv run pytest -q
uv run pytest tests/test_bands.py
uv run pytest tests/test_bands.py::test_create_band
uv run pytest -k "band"
uv run pytest -x
uv run pytest --maxfail=2
uv run pytest -s
uv run pytest --tb=short
uv run pytest --lf
uv run pytest --ff
```

Meaning:

- `-v`: verbose test names.
- `-q`: quieter output.
- `-k`: select tests by expression.
- `-x`: stop after first failure.
- `--maxfail=2`: stop after two failures.
- `-s`: show print output.
- `--tb=short`: shorter tracebacks.
- `--lf`: run last-failed tests.
- `--ff`: run last-failed tests first.

Coverage is a separate plugin:

```bash
uv add --dev pytest-cov
uv run pytest --cov=app --cov-report=term-missing
```

Coverage shows which code ran; high coverage does not automatically mean tests are good.

## 15. Arrange, Act, Assert

A simple test structure:

1. Arrange: prepare test data and dependencies.
2. Act: call the function or endpoint.
3. Assert: verify the result.

```python
def test_band_name(client):
    # Arrange
    payload = {"name": "Coldplay", "genre": "Rock"}

    # Act
    response = client.post("/bands/", json=payload)

    # Assert
    assert response.status_code == 201
    assert response.json()["name"] == "Coldplay"
```

## 16. Recommended testing checklist

- [ ] Successful response for each endpoint.
- [ ] Missing and invalid request fields.
- [ ] Unknown resource IDs.
- [ ] Database changes actually persist.
- [ ] PATCH leaves omitted fields unchanged.
- [ ] Authentication missing, invalid, expired.
- [ ] Permission denied for non-owners.
- [ ] Pagination, filtering, sorting.
- [ ] External services mocked in unit tests.
- [ ] Test database isolated from development/production.
- [ ] Important PostgreSQL-specific behaviour tested against PostgreSQL.
- [ ] Tests are repeatable and do not depend on execution order.

## 17. Further reading

- pytest documentation: https://docs.pytest.org/
- FastAPI testing: https://fastapi.tiangolo.com/tutorial/testing/
- HTTPX: https://www.python-httpx.org/
- SQLModel: https://sqlmodel.tiangolo.com/
