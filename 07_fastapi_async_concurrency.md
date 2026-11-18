# Async Python and FastAPI — A Detailed Guide to async, await, I/O, and Concurrency

## 1. Why async exists

An API server often spends time waiting:
- waiting for a database;
- waiting for another HTTP API;
- waiting for a file or network operation;
- waiting for a message broker.

During a wait, a synchronous worker may be unable to do other useful work. Asynchronous programming allows a task to pause at an await point so the event loop can run other ready tasks.

Async is especially useful for **I/O-bound** workloads. It does not automatically make CPU-heavy Python code faster.

## 2. Synchronous code

```python
import time

def fetch_one():
    time.sleep(2)
    return "done"

def endpoint_logic():
    first = fetch_one()
    second = fetch_one()
    return [first, second]
```

The two waits happen one after another. Total waiting time is roughly four seconds.

`time.sleep()` blocks the thread.

## 3. async and await

```python
import asyncio

async def fetch_one():
    await asyncio.sleep(2)
    return "done"

async def endpoint_logic():
    first = await fetch_one()
    second = await fetch_one()
    return [first, second]
```

This is still sequential because the second call is awaited only after the first finishes. `async def` alone does not create concurrency.

`await` says: suspend this coroutine until the awaitable operation completes, allowing the event loop to run other tasks.

## 4. Run independent I/O concurrently

```python
async def endpoint_logic():
    first, second = await asyncio.gather(
        fetch_one(),
        fetch_one(),
    )
    return [first, second]
```

The two independent waits can overlap. If each takes around two seconds, total time may be near two seconds rather than four, excluding overhead.

Use concurrency only when operations are genuinely independent and the downstream services can handle it.

## 5. Coroutine, task, and event loop

- **Coroutine function**: declared with `async def`.
- **Coroutine object**: created when you call an async function.
- **Task**: schedules a coroutine to run on the event loop.
- **Event loop**: coordinates ready async tasks and I/O events.

```python
async def work():
    await asyncio.sleep(1)
    return "finished"

async def main():
    task = asyncio.create_task(work())
    result = await task
    print(result)

asyncio.run(main())
```

`asyncio.run()` is normally used at a program's top-level entry point. Do not call it inside a FastAPI endpoint; the server already manages the event loop.

## 6. FastAPI: def vs async def

FastAPI supports both:

```python
@app.get("/sync")
def sync_endpoint():
    return {"mode": "sync"}

@app.get("/async")
async def async_endpoint():
    return {"mode": "async"}
```

Important:
- A normal `def` path operation is run in an external thread pool by FastAPI/Starlette.
- An `async def` path operation runs on the event loop.
- Use `async def` when the work uses awaitable asynchronous libraries.
- Use `def` when calling blocking synchronous libraries, unless you deliberately offload that work.

Do not write `async def` and then call blocking functions directly inside it.

## 7. The blocking-call mistake

Bad:

```python
import requests

@app.get("/external")
async def external():
    response = requests.get("https://example.com")
    return {"status": response.status_code}
```

`requests.get()` is synchronous and blocks the event-loop thread.

Better: use an async HTTP client:

```bash
uv add httpx
```

```python
import httpx

@app.get("/external")
async def external():
    async with httpx.AsyncClient(timeout=10) as client:
        response = await client.get("https://example.com")
        response.raise_for_status()
        return {"status": response.status_code}
```

Or make the endpoint a normal `def` if using a synchronous library:

```python
@app.get("/external-sync")
def external_sync():
    response = requests.get("https://example.com", timeout=10)
    response.raise_for_status()
    return {"status": response.status_code}
```

## 8. SQLModel synchronous Session vs AsyncSession

Your current SQLModel examples use:

```python
from sqlmodel import Session

def endpoint(session: Session = Depends(get_session)):
    ...
```

This is synchronous database access. Keep the endpoint and dependency synchronous unless you have deliberately moved to an async database stack.

An async SQLAlchemy/SQLModel-style stack uses:
- an async-capable database driver, such as `asyncpg` for PostgreSQL;
- `create_async_engine`;
- `AsyncSession`;
- `async def` endpoints;
- `await session.execute(...)` and related async methods.

Conceptual SQLAlchemy example:

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from sqlalchemy.ext.asyncio import AsyncSession

DATABASE_URL = "postgresql+asyncpg://user:password@localhost:5432/appdb"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(
    engine,
    expire_on_commit=False,
    class_=AsyncSession,
)

async def get_session():
    async with AsyncSessionLocal() as session:
        yield session
```

Endpoint using SQLAlchemy's async session:

```python
from sqlalchemy import select

@app.get("/bands")
async def list_bands(session: AsyncSession = Depends(get_session)):
    result = await session.execute(select(Band))
    return result.scalars().all()
```

This is an async SQLAlchemy example. SQLModel has async support through SQLAlchemy's async infrastructure, but check the current SQLModel documentation for the exact patterns supported by your installed version. Do not mix a synchronous `Session` with `await`.

## 9. Driver URL matters

Synchronous PostgreSQL URL:

```text
postgresql+psycopg://user:password@host:5432/database
```

Asynchronous PostgreSQL URL:

```text
postgresql+asyncpg://user:password@host:5432/database
```

The `+driver` part selects the SQLAlchemy dialect/driver combination. Install the matching driver:

```bash
uv add psycopg
# or
uv add asyncpg
```

Do not copy an async URL into code that creates a synchronous engine.

## 10. CPU-bound work is different

Examples of CPU-heavy work:
- image processing;
- large cryptographic calculations;
- compression;
- complex data transformations;
- machine-learning inference.

Async does not make CPU work run in parallel. A long CPU calculation inside an async endpoint can block the event loop.

Possible solutions:
- process pool for CPU-bound Python work;
- worker queue such as Celery or RQ;
- separate service;
- native libraries that release the GIL where appropriate.

For blocking I/O, a thread pool may be suitable. For CPU-heavy pure Python work, a process pool is often more appropriate.

## 11. asyncio.to_thread

Use `asyncio.to_thread()` to offload a blocking I/O function from the event-loop thread:

```python
import asyncio
import requests

def blocking_request():
    return requests.get("https://example.com", timeout=10).status_code

@app.get("/threaded")
async def threaded():
    status_code = await asyncio.to_thread(blocking_request)
    return {"status": status_code}
```

This does not make the blocking library asynchronous; it runs the blocking function in a worker thread so the event loop can continue.

## 12. Timeouts and cancellation

Never allow external network operations to wait forever.

```python
async with httpx.AsyncClient(timeout=10) as client:
    response = await client.get("https://example.com")
```

For asyncio operations:

```python
result = await asyncio.wait_for(some_coroutine(), timeout=5)
```

A timeout raises `asyncio.TimeoutError` (built-in `TimeoutError` in modern Python versions). Handle it appropriately.

Cancellation is part of async task management. Use `try/finally` to release resources and generally do not swallow `asyncio.CancelledError` without a deliberate reason.

## 13. Background tasks vs durable job queues

FastAPI `BackgroundTasks` can run small tasks after a response is sent:

```python
from fastapi import BackgroundTasks

def write_audit_log(message: str):
    ...

@app.post("/events")
def create_event(background_tasks: BackgroundTasks):
    background_tasks.add_task(write_audit_log, "event created")
    return {"accepted": True}
```

This is useful for small in-process tasks. It is not a durable distributed job system:
- work may be lost if the process stops;
- no built-in durable retry queue;
- not ideal for long-running or resource-heavy jobs.

Use Celery, RQ, Dramatiq, or another worker system for durable background processing, retries, scheduling, and distributed workers.

## 14. Common async mistakes

1. Using `requests` inside `async def`.
2. Calling `time.sleep()` inside `async def`; use `await asyncio.sleep()`.
3. Assuming `async def` automatically makes work concurrent.
4. Forgetting `await` on a coroutine.
5. Calling `asyncio.run()` inside a running event loop.
6. Using a synchronous SQLModel `Session` as if it were async.
7. Using a blocking CPU-heavy function on the event loop.
8. Creating too many concurrent requests and overwhelming a downstream service.
9. Forgetting timeouts and error handling.

## 15. A decision guide

| Work | Recommended approach |
|---|---|
| Sync SQLModel `Session` | Normal `def` endpoint |
| Async database driver/session | `async def` and await DB operations |
| Async HTTPX | `async def` and `await` |
| Requests library | `def`, or `asyncio.to_thread` |
| `time.sleep` | `def` or replace with `await asyncio.sleep` |
| CPU-heavy work | Process pool / worker |
| Small post-response action | FastAPI `BackgroundTasks` |
| Durable long-running task | Celery/RQ/Dramatiq |

## 16. Practice

1. Write two async functions that each wait one second.
2. Run them sequentially, then with `asyncio.gather()`, and compare elapsed time.
3. Call an external endpoint using HTTPX AsyncClient.
4. Demonstrate why `requests.get()` blocks an async endpoint.
5. Create a synchronous SQLModel endpoint and an async SQLAlchemy endpoint; keep the stacks separate.
6. Add timeouts and handle errors.
7. Move a long-running task to a worker queue.

## 17. Official references

- FastAPI async: https://fastapi.tiangolo.com/async/
- Python asyncio: https://docs.python.org/3/library/asyncio.html
- SQLAlchemy asyncio: https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
- HTTPX async: https://www.python-httpx.org/async/
