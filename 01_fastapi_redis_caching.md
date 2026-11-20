# FastAPI DevOps – Part 1: Redis and Caching

> A practical guide to adding caching to a FastAPI + SQLModel application. Examples use synchronous SQLModel sessions first, then explain async alternatives.

## 1. What problem does caching solve?

Imagine an endpoint:

```python
@app.get("/bands")
def list_bands(session: Session = Depends(get_session)):
    return session.exec(select(Band)).all()
```

Every request executes a database query. If the same data is requested hundreds of times, repeated database work can waste time and database resources.

A **cache** stores a copy of data so that a later request can retrieve it more quickly.

Typical flow without cache:

```text
Client -> FastAPI -> PostgreSQL -> FastAPI -> Client
```

With a cache:

```text
Client -> FastAPI -> Redis
                    | cache hit
                    +-----------> Client

                    | cache miss
                    v
                 PostgreSQL -> Redis (store result) -> Client
```

A cache is normally not the permanent source of truth. Your database remains authoritative; cached values can expire or be invalidated.

## 2. What is Redis?

Redis is an in-memory data store. It keeps data in memory for very fast access and supports data structures such as strings, hashes, lists, sets, sorted sets, and streams.

Common Redis uses:

- API response/data caching.
- Celery broker and result backend.
- Rate-limit counters.
- Short-lived verification codes and temporary tokens.
- Session-like temporary state.
- Pub/Sub and streams for messaging use cases.

Redis is a server, not a Python library. Your FastAPI application connects to a Redis server using a Python client.

### Redis deployment choices

You can run Redis:

1. Locally on your development machine.
2. In a Docker container.
3. On a separate Linux server.
4. As a managed cloud service.

For development, Docker is convenient. For production, use a managed Redis service or a properly secured Redis server on private infrastructure.

## 3. Install and start Redis

Redis is commonly used on Linux. On Windows development machines, Docker is often the easiest way to run a Redis server.

```bash
docker run --name fastapi-redis \
  -p 6379:6379 \
  -d redis:7-alpine
```

Check it:

```bash
docker ps
docker exec -it fastapi-redis redis-cli ping
```

Expected response:

```text
PONG
```

Do not expose Redis publicly without authentication, network restrictions, and TLS where appropriate. In production, Redis should normally be reachable only from your application network.

## 4. Install the Python Redis client

For a synchronous FastAPI endpoint:

```bash
uv add redis
```

For async Redis operations, the same `redis` package provides `redis.asyncio`.

The Redis server and Python client are two different things:

| Item | Purpose |
|---|---|
| Redis server | Stores cache data |
| `redis` Python package | Lets Python communicate with Redis |
| FastAPI | Receives HTTP requests |
| SQLModel/SQLAlchemy | Communicates with the relational database |

## 5. Connect FastAPI to Redis

Create `app/redis_client.py`:

```python
import os
import redis

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")

redis_client = redis.Redis.from_url(
    REDIS_URL,
    decode_responses=True,
)
```

`decode_responses=True` makes Redis return strings rather than raw bytes for string values.

Test the connection:

```python
# test_redis.py
from app.redis_client import redis_client

redis_client.set("message", "Hello from Redis", ex=30)
print(redis_client.get("message"))
```

`ex=30` means the key expires after 30 seconds.

## 6. Understand cache keys and expiration

A Redis key identifies a cached value:

```python
redis_client.set("bands:all", '[{"id": 1, "name": "Coldplay"}]', ex=60)
```

- `bands:all` is the key.
- The JSON string is the value.
- `ex=60` is the time-to-live (TTL), in seconds.

Use predictable names. Include relevant query parameters in keys:

```text
bands:page:1:limit:20
bands:page:2:limit:20
bands:genre:rock
band:42
```

If a response depends on the logged-in user, tenant, language, or permissions, those dimensions may need to be part of the cache key. Never accidentally share one user's private response with another user.

## 7. Cache-aside pattern (recommended starting point)

The application checks the cache first. If the value is missing, it reads from the database and places the result in Redis.

```python
import json
from fastapi import APIRouter, Depends
from sqlmodel import Session, select

from app.database import get_session
from app.models import Band
from app.redis_client import redis_client

router = APIRouter()

@router.get("/bands")
def list_bands(session: Session = Depends(get_session)):
    cache_key = "bands:all"

    cached = redis_client.get(cache_key)
    if cached is not None:
        return json.loads(cached)

    bands = session.exec(select(Band)).all()

    # Convert ORM objects to JSON-compatible dictionaries.
    data = [
        {"id": band.id, "name": band.name}
        for band in bands
    ]

    redis_client.set(cache_key, json.dumps(data), ex=60)
    return data
```

Why convert the models to dictionaries? Redis stores bytes or strings, not live Python ORM objects. JSON is a portable representation.

For production, define explicit response models and serialize them deliberately. Do not blindly serialize ORM objects containing relationships or sensitive fields.

## 8. Invalidate cache after writes

A common bug is returning old cached data after a record has changed.

After creating a band:

```python
@router.post("/bands")
def create_band(
    band: Band,
    session: Session = Depends(get_session),
):
    session.add(band)
    session.commit()
    session.refresh(band)

    redis_client.delete("bands:all")
    return band
```

The next GET request misses the cache and reloads fresh database data.

For a single-record cache, invalidate the corresponding key too:

```python
redis_client.delete(f"band:{band.id}")
```

Cache invalidation becomes more complicated when there are many filters, pages, aggregates, and related objects. Start with a small number of cache keys and document exactly which writes invalidate them.

## 9. Cache individual records

```python
@router.get("/bands/{band_id}")
def get_band(
    band_id: int,
    session: Session = Depends(get_session),
):
    key = f"band:{band_id}"
    cached = redis_client.get(key)

    if cached:
        return json.loads(cached)

    band = session.get(Band, band_id)
    if band is None:
        raise HTTPException(status_code=404, detail="Band not found")

    data = {"id": band.id, "name": band.name}
    redis_client.set(key, json.dumps(data), ex=120)
    return data
```

When updating or deleting that band, delete `band:{band_id}`.

## 10. Async Redis client

If your endpoint is `async def`, avoid calling blocking synchronous Redis methods directly inside it. Use the async client:

```python
import os
import redis.asyncio as redis

redis_client = redis.from_url(
    os.getenv("REDIS_URL", "redis://localhost:6379/0"),
    decode_responses=True,
)

async def example():
    await redis_client.set("greeting", "hello", ex=30)
    value = await redis_client.get("greeting")
    return value
```

Close the client during application shutdown/lifespan:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    await redis_client.aclose()

app = FastAPI(lifespan=lifespan)
```

Use one appropriately managed client/pool for the application rather than creating a new connection for every request.

## 11. Other caching choices

| Technology | What it is | Good fit |
|---|---|---|
| Redis | Networked in-memory data store | Shared cache across multiple API workers/servers |
| Memcached | Distributed in-memory key/value cache | Simple cache values |
| `cachetools` | In-process Python cache utilities | Small single-process applications |
| `functools.lru_cache` | Python function-result cache | Pure, repeatable calculations/config lookups |
| CDN / reverse-proxy cache | HTTP response cache near clients | Public static or cacheable GET responses |

An in-process cache is not shared between multiple Uvicorn workers. If worker A caches a value, worker B may not see it. Redis is useful when multiple processes or servers need a shared cache.

## 12. Important cache design decisions

- **TTL:** How long can the value remain cached?
- **Invalidation:** Which writes remove or refresh it?
- **Cache key:** Does it include filters, page, tenant, user, or language?
- **Serialization:** Can the value be safely converted to/from JSON?
- **Failure policy:** If Redis is down, should the endpoint fall back to the database?
- **Stampede:** Many requests may miss the same key simultaneously and all query the database. Consider locks, stale-while-revalidate, or request coalescing for hot keys.
- **Memory:** Set sensible TTLs and memory policies. Monitor memory use.
- **Security:** Do not cache secrets or private responses under shared keys.

A cache should improve performance without becoming a correctness or security bug.

## 13. Cache versus database

| Database | Cache |
|---|---|
| Permanent/authoritative records | Temporary copy of useful data |
| Designed for durable queries and transactions | Designed for very fast access |
| Usually stores normalized relational data | Often stores precomputed values |
| Updates are part of business logic | Must be invalidated/refreshed |

## 14. Suggested practice

Build these endpoints:

1. `GET /bands` – cache the list for 60 seconds.
2. `GET /bands/{id}` – cache one band.
3. `POST /bands` – invalidate the list cache.
4. `PATCH /bands/{id}` – invalidate list and detail cache.
5. `DELETE /bands/{id}` – invalidate list and detail cache.

Test both cache hit and cache miss. Change a band and verify that the next GET does not return stale data.

## 15. Key takeaways

- Redis is a separate server; `redis` is the Python client.
- Cache-aside is a good first pattern.
- Use TTLs and deliberate invalidation.
- Use async Redis calls inside async endpoints.
- Never treat the cache as the permanent source of truth.
- Protect user-specific cached data from cross-user leakage.
