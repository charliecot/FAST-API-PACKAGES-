# Essential Modules and Patterns for FastAPI Endpoints

## 1. The Django REST Framework comparison

In Django REST Framework, you may commonly use:
- `Response`
- `status.HTTP_200_OK`
- `APIView` / ViewSets
- serializers
- permissions
- pagination
- filters
- exception handling

FastAPI has equivalents, but it does not require every endpoint to use one response class. It uses Python return values, Pydantic response models, Starlette responses, dependencies, and exception handlers.

## 2. HTTP status codes

FastAPI provides `status`:

```python
from fastapi import status

@app.post("/bands/", status_code=status.HTTP_201_CREATED)
def create_band(...):
    ...
```

Common status codes:

| Code | Meaning | Common use |
|---:|---|---|
| 200 | OK | Successful read/update |
| 201 | Created | Resource created |
| 202 | Accepted | Work accepted for later processing |
| 204 | No Content | Successful operation with no response body |
| 400 | Bad Request | Invalid application-level request |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Authenticated but not permitted |
| 404 | Not Found | Resource does not exist or is concealed |
| 409 | Conflict | Duplicate or conflicting state |
| 422 | Unprocessable Content | Request validation failure (FastAPI default validation response) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server failure |
| 503 | Service Unavailable | Service temporarily unavailable |

Prefer symbolic names from `fastapi.status` over unexplained numeric literals.

## 3. Returning JSON — the ordinary case

```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

FastAPI converts ordinary dictionaries, lists, and supported data structures into JSON responses.

For most endpoints, this is the simplest and best approach.

## 4. Response models

A response model defines and validates the structure returned by an endpoint:

```python
from sqlmodel import SQLModel

class BandRead(SQLModel):
    id: int
    name: str
    genre: str

@app.get("/bands/{band_id}", response_model=BandRead)
def get_band(band_id: int):
    ...
```

Benefits:
- documents the API in OpenAPI;
- validates output;
- filters fields to the declared response schema;
- provides consistent client-facing structure.

Never expose `hashed_password`, internal secrets, or private fields in a public response schema.

For nested data:

```python
class AlbumRead(SQLModel):
    id: int
    title: str

class BandRead(SQLModel):
    id: int
    name: str
    albums: list[AlbumRead]
```

## 5. Returning an explicit Response

Use `JSONResponse` when you need to control response details directly:

```python
from fastapi.responses import JSONResponse

@app.get("/custom")
def custom():
    return JSONResponse(
        status_code=200,
        content={"message": "Custom JSON response"},
        headers={"X-API-Version": "1"},
    )
```

Other useful response classes include:
- `HTMLResponse`
- `PlainTextResponse`
- `RedirectResponse`
- `StreamingResponse`
- `FileResponse`
- `Response` (base response)

If you return a `Response` directly, FastAPI does not apply response-model filtering to that response in the same way as an ordinary return value. Validate/serialize its content yourself when needed.

## 6. HTTPException — endpoint errors

```python
from fastapi import HTTPException, status

@app.get("/bands/{band_id}", response_model=BandRead)
def get_band(band_id: int, session: Session = Depends(get_session)):
    band = session.get(Band, band_id)

    if band is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Band not found",
        )

    return band
```

`raise HTTPException(...)` stops the current endpoint flow and returns an HTTP error response. Do not `return HTTPException(...)`; raise it.

You can attach headers:

```python
raise HTTPException(
    status_code=401,
    detail="Not authenticated",
    headers={"WWW-Authenticate": "Bearer"},
)
```

## 7. Custom exception handlers

Use handlers when you want consistent error formatting.

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class ResourceConflict(Exception):
    def __init__(self, message: str):
        self.message = message

@app.exception_handler(ResourceConflict)
async def conflict_handler(request: Request, exc: ResourceConflict):
    return JSONResponse(
        status_code=409,
        content={"error": "conflict", "detail": exc.message},
    )
```

Raise it:

```python
raise ResourceConflict("A band with this name already exists")
```

You can also override handlers for `HTTPException` or request validation errors. Be careful not to leak stack traces, SQL statements, passwords, tokens, or internal infrastructure details to clients.

## 8. Consistent error response design

A simple API error format might be:

```json
{
  "error": "not_found",
  "detail": "Band not found"
}
```

For validation errors, include field-level details where useful. Do not create an elaborate custom format unless it improves client integration. FastAPI's default validation error response is already useful.

## 9. Request data: path, query, body, headers, cookies

### Path parameter

```python
@app.get("/bands/{band_id}")
def get_band(band_id: int):
    ...
```

### Query parameter

```python
@app.get("/bands/")
def list_bands(genre: str | None = None, limit: int = 20):
    ...
```

Request example:

```text
GET /bands/?genre=Rock&limit=10
```

### Request body

```python
class BandCreate(SQLModel):
    name: str
    genre: str

@app.post("/bands/")
def create_band(payload: BandCreate):
    ...
```

### Header

```python
from fastapi import Header

@app.get("/client-info")
def client_info(user_agent: str | None = Header(default=None)):
    return {"user_agent": user_agent}
```

### Cookie

```python
from fastapi import Cookie

@app.get("/cookie")
def read_cookie(session_id: str | None = Cookie(default=None)):
    return {"session_id_present": session_id is not None}
```

Do not echo sensitive cookie values unnecessarily.

## 10. Pagination

Without pagination, a table with millions of rows could produce an enormous response.

Simple limit/offset:

```python
from sqlmodel import select

@app.get("/bands/", response_model=list[BandRead])
def list_bands(
    offset: int = 0,
    limit: int = 20,
    session: Session = Depends(get_session),
):
    statement = (
        select(Band)
        .offset(offset)
        .limit(min(limit, 100))
    )
    return session.exec(statement).all()
```

Validate bounds:

```python
from fastapi import Query

@app.get("/bands/")
def list_bands(
    offset: int = Query(default=0, ge=0),
    limit: int = Query(default=20, ge=1, le=100),
):
    ...
```

For very large or frequently changing datasets, keyset/cursor pagination can be more efficient and stable than large offsets.

A response envelope can include metadata:

```json
{
  "items": [],
  "limit": 20,
  "offset": 0,
  "total": 135
}
```

If you include `total`, calculate it with a separate count query. Avoid expensive counts when they are not needed.

## 11. Filtering and sorting

```python
@app.get("/bands/")
def list_bands(
    genre: str | None = None,
    order_by: str = "name",
    session: Session = Depends(get_session),
):
    statement = select(Band)

    if genre:
        statement = statement.where(Band.genre == genre)

    allowed_sort_fields = {
        "name": Band.name,
        "genre": Band.genre,
    }

    column = allowed_sort_fields.get(order_by)
    if column is None:
        raise HTTPException(400, detail="Unsupported sort field")

    statement = statement.order_by(column)
    return session.exec(statement).all()
```

Never insert an arbitrary client-supplied string into raw SQL as a column name. Map requested sort keys to known model columns.

For larger APIs, consider reusable query dependencies, `fastapi-filter`, or carefully designed repository/service functions. Check library maintenance and compatibility before adopting an extension.

## 12. File upload

FastAPI uses `UploadFile` and `File`:

```python
from fastapi import UploadFile, File

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    contents = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
    }
```

For large files, do not blindly read the entire file into memory. Stream/copy in chunks. Validate file size, content, extension, and actual file signature as appropriate. Store uploads outside publicly executable directories and generate safe server-side names.

Install multipart support:

```bash
uv add python-multipart
```

Never trust `file.filename` or `content_type` as proof that a file is safe.

## 13. Background tasks

```python
from fastapi import BackgroundTasks

def send_notification(email: str):
    ...

@app.post("/notify", status_code=202)
def notify(background_tasks: BackgroundTasks, email: str):
    background_tasks.add_task(send_notification, email)
    return {"message": "Notification queued"}
```

The example returns accepted, but Starlette BackgroundTasks are in-process tasks, not a durable queue. For critical emails, payments, or long-running jobs, use a worker system and durable broker.

## 14. Middleware

Middleware runs around requests and responses. Common uses:
- CORS;
- request IDs;
- timing/logging;
- trusted host handling;
- security headers;
- metrics.

Example timing middleware:

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = str(time.perf_counter() - start)
    return response
```

Do not log credentials, bearer tokens, passwords, or sensitive request bodies.

## 15. Dependency injection

FastAPI's `Depends` is used for shared requirements:

```python
def get_session():
    with Session(engine) as session:
        yield session

@app.get("/bands/")
def list_bands(session: Session = Depends(get_session)):
    ...
```

Other dependency uses:
- current user;
- permission checks;
- pagination parameters;
- shared filters;
- configuration;
- external clients.

Dependencies can depend on other dependencies. Keep them focused and testable.

## 16. Router organisation

A larger application should not place every endpoint in one `main.py`.

```text
app/
├── main.py
├── database.py
├── models/
│   ├── band.py
│   └── album.py
├── schemas/
│   ├── band.py
│   └── album.py
├── routers/
│   ├── bands.py
│   └── albums.py
├── services/
├── dependencies/
└── core/
    └── config.py
```

Router:

```python
from fastapi import APIRouter

router = APIRouter(prefix="/bands", tags=["bands"])

@router.get("/")
def list_bands():
    ...

@router.get("/{band_id}")
def get_band(band_id: int):
    ...
```

Register it:

```python
app.include_router(router)
```

## 17. OpenAPI documentation

FastAPI automatically generates:
- OpenAPI schema;
- Swagger UI at `/docs`;
- ReDoc at `/redoc` by default.

Use:
- `response_model`;
- descriptions and summaries;
- tags;
- request/response schemas;
- status-code documentation;
- security dependencies.

These improve generated API documentation and frontend integration.

## 18. Other useful API concerns

- **Logging**: Python `logging`; structured logs are useful in production.
- **Configuration**: `pydantic-settings` and environment variables.
- **Database migrations**: Alembic.
- **Rate limiting**: a dedicated middleware/library or gateway; do not assume FastAPI provides a complete built-in rate limiter.
- **Caching**: Redis or an HTTP cache strategy where appropriate.
- **Observability**: metrics, traces, health/readiness checks.
- **Request IDs**: correlate client reports with logs.
- **Security headers**: configure through middleware or reverse proxy.
- **Compression**: GZip middleware or reverse proxy.
- **Trusted hosts**: restrict accepted Host headers where appropriate.
- **HTTPS**: usually terminated at a trusted reverse proxy/load balancer in production.

## 19. DRF to FastAPI quick comparison

| Django REST Framework | FastAPI / Starlette |
|---|---|
| `Response(data, status=...)` | Return dict/list, or `JSONResponse` |
| `status.HTTP_201_CREATED` | `fastapi.status.HTTP_201_CREATED` |
| `serializer` | Pydantic/SQLModel schema |
| `serializer.is_valid()` | Pydantic request validation |
| `serializer.save()` | Explicit service/session operations |
| `APIView` | Path operation function |
| `ViewSet` + router | `APIRouter` + path operations |
| Permission classes | Dependencies/policy functions |
| Pagination classes | Query parameters and pagination logic |
| Middleware | Starlette/FastAPI middleware |
| Exception handler | FastAPI exception handler |
| `request.data` | Typed body parameter |
| `request.query_params` | Typed query parameters |
| `request.user` | Current-user dependency |

## 20. A practical endpoint checklist

Before shipping an endpoint, ask:

- [ ] Is the HTTP method correct?
- [ ] Is the request schema appropriate?
- [ ] Is the response model safe and accurate?
- [ ] Is the status code correct?
- [ ] Are missing resources handled?
- [ ] Are authentication and object permissions checked?
- [ ] Are query parameters bounded and validated?
- [ ] Is database access efficient?
- [ ] Are transactions committed/rolled back correctly?
- [ ] Are external operations timed out?
- [ ] Are errors useful but not revealing?
- [ ] Is the endpoint tested?
- [ ] Is it documented in OpenAPI?

## 21. Official references

- FastAPI responses: https://fastapi.tiangolo.com/advanced/custom-response/
- FastAPI status codes: https://fastapi.tiangolo.com/tutorial/response-status-code/
- FastAPI error handling: https://fastapi.tiangolo.com/tutorial/handling-errors/
- FastAPI dependencies: https://fastapi.tiangolo.com/tutorial/dependencies/
- FastAPI request files: https://fastapi.tiangolo.com/tutorial/request-files/
- Starlette responses: https://www.starlette.io/responses/
