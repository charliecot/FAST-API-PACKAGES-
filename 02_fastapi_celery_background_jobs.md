# FastAPI DevOps – Part 2: Celery and Background Jobs

> A beginner-friendly explanation of Celery, why it exists, how it differs from FastAPI background tasks, and how to run a worker with Redis and Docker.

## 1. The problem: work that should not hold an HTTP request open

Suppose a customer places an order. Your API must:

1. Validate the request.
2. Save the order in PostgreSQL.
3. Send a confirmation email.
4. Generate an invoice PDF.
5. Notify an external service.

Saving the order is central to the HTTP operation. Sending an email or generating a PDF may take several seconds and may fail temporarily.

If the API waits for every task, the client waits too. A better design is often:

```text
Client
  |
  v
FastAPI endpoint
  |-- validate and save order in PostgreSQL
  |
  +-- publish a job message
          |
          v
      Message broker (Redis or RabbitMQ)
          |
          v
      Celery worker
          |-- send email
          |-- generate PDF
          +-- call external service
```

The API can return a response while a separate worker processes the job.

## 2. What is Celery?

Celery is a distributed task queue for Python.

It allows one process to submit work and another process to execute it. The processes do not have to run on the same machine, provided they can communicate with the broker and access required resources.

Important concepts:

| Term | Meaning |
|---|---|
| Task | A Python function registered with Celery |
| Producer | The application that sends a task message |
| Broker | Holds/routes messages until workers receive them |
| Worker | Separate process that executes tasks |
| Result backend | Optional storage for task states/results |
| Beat | Optional scheduler that sends periodic tasks |

Celery is not a FastAPI feature. It is a separate system that can be used with FastAPI, Django, Flask, or other Python applications.

## 3. Celery versus FastAPI BackgroundTasks

FastAPI provides `BackgroundTasks`:

```python
from fastapi import BackgroundTasks, FastAPI

app = FastAPI()

def send_email(email: str):
    print(f"Sending email to {email}")

@app.post("/signup")
def signup(email: str, background_tasks: BackgroundTasks):
    # Save user first...
    background_tasks.add_task(send_email, email)
    return {"message": "Account created"}
```

This is useful for small, short tasks attached to a response. But the work runs in the application process. It is not a durable distributed queue.

Celery is more suitable when you need:

- Independent worker processes.
- Multiple worker machines.
- Retries and retry policies.
- Scheduled/periodic jobs.
- Long-running tasks.
- Task routing and concurrency controls.
- A queue that remains available while API processes restart (subject to broker durability/configuration).

| FastAPI `BackgroundTasks` | Celery |
|---|---|
| Built into FastAPI/Starlette | Separate task-queue system |
| Simple to set up | More infrastructure |
| Runs in the API process | Runs in worker processes |
| Good for small post-response work | Good for distributed/long-running work |
| No built-in durable distributed queue | Broker-backed task delivery |
| No full Celery retry/scheduling system | Retries, routing, Beat scheduling |

Do not add Celery to every small project. Add it when the work and reliability requirements justify the extra infrastructure.

## 4. Broker: Redis or RabbitMQ?

Celery needs a broker to transport task messages.

### Redis

Redis is easy to start and useful when you already use Redis for caching. It can act as Celery broker and optionally as result backend.

### RabbitMQ

RabbitMQ is a dedicated message broker with mature messaging and routing features. It is a strong choice for message-heavy systems and more complex routing.

| Redis | RabbitMQ |
|---|---|
| In-memory data store with messaging capabilities | Dedicated message broker |
| Often simpler for small deployments | Rich exchange/queue routing model |
| Can also cache application data | Primarily messaging |
| Persistence/durability must be configured and understood | Designed around broker semantics and durable queues |

Do not assume that using Redis for caching automatically makes it configured safely for production task delivery. Configure persistence, memory policy, authentication, network access, and monitoring appropriately.

## 5. Install Celery

For a Redis broker:

```bash
uv add "celery[redis]"
```

This installs Celery with its Redis transport support.

## 6. Suggested project structure

```text
my_fastapi_project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   └── tasks.py
├── celery_app.py
├── .env
└── pyproject.toml
```

## 7. Configure Celery

Create `celery_app.py`:

```python
import os
from celery import Celery

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")

celery_app = Celery(
    "fastapi_tasks",
    broker=REDIS_URL,
    backend=REDIS_URL,
    include=["app.tasks"],
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
)
```

- `broker` transports tasks to workers.
- `backend` stores task states/results if you need to inspect them.
- `include` tells Celery where task definitions live.
- JSON serialization is a sensible interoperable default. Do not enable pickle for untrusted message sources.

You can use a separate Redis database number or separate Redis service for broker and result backend. In larger systems, separate services can isolate workloads.

## 8. Define a task

Create `app/tasks.py`:

```python
import time
from celery_app import celery_app

@celery_app.task
def generate_report(report_id: int) -> dict:
    # Simulate expensive work.
    time.sleep(5)

    # Real task: load report data, create a file, store it, etc.
    return {
        "report_id": report_id,
        "status": "generated",
    }
```

The decorator registers the function as a Celery task.

Important: pass simple serializable identifiers and values, not a live SQLModel `Session`, database connection, HTTP request object, or ORM instance. The worker should create its own database session when it needs the database.

## 9. Submit a task from FastAPI

```python
from fastapi import FastAPI
from app.tasks import generate_report

app = FastAPI()

@app.post("/reports/{report_id}", status_code=202)
def request_report(report_id: int):
    task = generate_report.delay(report_id)

    return {
        "message": "Report generation queued",
        "task_id": task.id,
    }
```

`.delay(...)` is a convenient shortcut for sending a task message. It does not execute the function in the API process.

HTTP `202 Accepted` is appropriate when the request has been accepted for processing but the work is not yet complete.

## 10. Start the worker

In a separate terminal, from the project root:

```bash
uv run celery -A celery_app:celery_app worker --loglevel=INFO
```

You now have two processes:

```text
Terminal 1: uv run uvicorn app.main:app --reload
Terminal 2: uv run celery -A celery_app:celery_app worker --loglevel=INFO
```

The API submits jobs. The worker consumes and executes them.

On Windows, Celery's prefork pool is not generally supported in the same way as on Unix-like systems. For local Windows learning, you may use the solo pool:

```bash
uv run celery -A celery_app:celery_app worker --loglevel=INFO --pool=solo
```

For production, run workers on Linux (including Linux containers).

## 11. Check task status

If you configured a result backend:

```python
from celery.result import AsyncResult
from celery_app import celery_app

@app.get("/tasks/{task_id}")
def get_task_status(task_id: str):
    result = AsyncResult(task_id, app=celery_app)

    response = {"task_id": task_id, "state": result.state}

    if result.ready():
        if result.successful():
            response["result"] = result.result
        else:
            response["error"] = str(result.result)

    return response
```

Be careful about exposing task IDs and results. Verify that the requesting user is allowed to inspect that job. A random task ID must not become an authorization mechanism.

## 12. Retry temporary failures

```python
from celery_app import celery_app

@celery_app.task(
    bind=True,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,
    retry_kwargs={"max_retries": 5},
)
def call_external_service(self, customer_id: int):
    # Make a request to a third-party service.
    ...
```

Retries are useful for temporary network errors, rate limits, and unavailable services. Do not retry every exception blindly. A permanent validation error will not become successful by repeating it.

Tasks should be **idempotent** where possible: executing the same task more than once should not create duplicate payments, emails, or orders. Use unique operation IDs, database constraints, and provider idempotency keys where available.

## 13. Periodic tasks with Celery Beat

Celery Beat is the scheduler. It publishes scheduled tasks; workers execute them.

Example:

```python
from celery.schedules import crontab

celery_app.conf.beat_schedule = {
    "cleanup-every-night": {
        "task": "app.tasks.cleanup_expired_records",
        "schedule": crontab(hour=2, minute=0),
    },
}
```

Define the task in `app/tasks.py`, then run Beat separately:

```bash
uv run celery -A celery_app:celery_app beat --loglevel=INFO
```

Run one scheduler instance for a given schedule unless you have deliberately designed coordination. Multiple uncoordinated Beat instances can enqueue duplicate jobs.

## 14. Database work inside a task

A worker is a separate process. It must create and close its own session:

```python
from app.database import engine
from sqlmodel import Session

@celery_app.task
def process_order(order_id: int):
    with Session(engine) as session:
        order = session.get(Order, order_id)
        if order is None:
            return {"status": "not_found"}

        # Perform safe database work.
        order.status = "processing"
        session.add(order)
        session.commit()

    return {"status": "processed"}
```

Do not pass a request-scoped session from FastAPI into Celery.

## 15. Reliability and production concerns

- Broker availability: decide what happens if Redis/RabbitMQ is unavailable.
- Acknowledgements and redelivery: understand the delivery semantics and possible duplicate execution.
- Idempotency: tasks may execute more than once.
- Time limits: prevent runaway tasks.
- Retries: retry transient errors with backoff and limits.
- Monitoring: observe queue length, worker health, task failures and duration.
- Secrets: use environment variables or a secrets manager.
- Results: do not store large results forever; set result expiry or store output in object storage.
- Security: broker should be on a private network, not exposed publicly.
- Graceful shutdown: allow workers to finish or safely requeue work.

A queue is not a substitute for database transactions. For critical business events, consider the transactional outbox pattern: write the business change and an outbox record in the same database transaction, then publish the outbox event reliably.

## 16. When should you use Celery?

Use it for:
- Sending batches of emails.
- PDF/report generation.
- Image/video processing.
- Data imports/exports.
- Scheduled cleanup.
- External API work that needs retries.
- Long-running calculations.

Use `BackgroundTasks` for small, simple tasks where losing the task during a process crash is acceptable.

## 17. Practice project

Create an endpoint that accepts a report request:

1. Store a `ReportJob` row with status `queued`.
2. Submit `generate_report.delay(report_job.id)`.
3. Worker loads the job with its own SQLModel session.
4. Worker changes status to `running`.
5. Worker generates a report.
6. Worker saves the file and sets status to `completed`.
7. On failure, record a safe error state.
8. Add `GET /reports/{id}` so the owner can inspect progress.

This resembles a real production workflow.
