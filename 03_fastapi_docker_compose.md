# FastAPI DevOps – Part 3: Dockerisation with Docker Compose

> Containerise a FastAPI + PostgreSQL + Redis + Celery application. This guide explains images, containers, Dockerfiles, Compose, networks, volumes, environment variables, and development versus production configuration.

## 1. What does Docker solve?

Without containers, deployment instructions may depend on the machine:

- Which Python version?
- Which system libraries?
- Which PostgreSQL version?
- Is Redis installed?
- Which environment variables are required?
- How do you start the API and worker?

Docker packages an application and its runtime dependencies into an **image**. A running instance of an image is a **container**.

An image is a build artifact/template. A container is a running instance.

Docker does not replace FastAPI, PostgreSQL, Redis, or Celery. It runs them in isolated containers.

## 2. Understand the technologies

| Technology | Responsibility |
|---|---|
| FastAPI | HTTP API |
| Uvicorn | ASGI server that runs the FastAPI application |
| SQLModel/SQLAlchemy | Python database layer |
| PostgreSQL | Persistent relational database |
| Redis | Cache and/or Celery broker |
| Celery worker | Executes background jobs |
| Docker | Builds and runs containers |
| Docker Compose | Defines and starts a multi-container application |

A typical local stack:

```text
                 Browser / API client
                         |
                         v
                  FastAPI container
                    /          \
                   v            v
            PostgreSQL       Redis
                                 ^
                                 |
                           Celery worker
```

The API and Celery worker can use the same application image but run different commands.

## 3. Install Docker

Install Docker Desktop on Windows and ensure its Linux container engine is running. Docker Desktop commonly uses WSL 2 as its backend.

Verify:

```bash
docker --version
docker compose version
docker info
```

If `docker info` cannot connect to the daemon, start Docker Desktop and wait for the engine to become ready.

## 4. Recommended project structure

```text
fastapi-project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   ├── tasks.py
│   └── ...
├── celery_app.py
├── migrations/
├── alembic.ini
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── compose.yaml
├── .dockerignore
├── .env
└── .env.example
```

Adapt paths and module names to your actual project.

## 5. Why use `uv.lock`?

A `pyproject.toml` declares dependencies. A lock file records the resolved dependency versions.

For repeatable builds, commit both `pyproject.toml` and `uv.lock` to version control. Do not create a different dependency set every time the image builds.

The example below assumes your project uses `uv` and has a `uv.lock` file.

## 6. Create a Dockerfile

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /code

# Install uv.
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Copy dependency metadata first to improve build caching.
COPY pyproject.toml uv.lock ./

# Install dependencies into the project virtual environment.
RUN uv sync --frozen --no-dev

# Copy application source.
COPY . .

EXPOSE 8000

CMD ["uv", "run", "--no-sync", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Replace `app.main:app` with your actual import path.

### Understand the important lines

- `FROM`: base image, here a slim Python image.
- `WORKDIR`: working directory inside the image.
- `COPY`: copies files into the image.
- `RUN`: executes commands during image build.
- `EXPOSE`: documents the port the application listens on; it does not publish the port to your host.
- `CMD`: default process started when the container runs.

The API must bind to `0.0.0.0` inside the container. Binding to `127.0.0.1` would make it reachable only from inside that container.

For production, pin the Python base image to an approved version/digest and use a deliberate upgrade process. Avoid relying on an unreviewed floating `latest` tag for critical builds.

## 7. Create `.dockerignore`

```text
.venv
__pycache__
*.py[cod]
.pytest_cache
.mypy_cache
.ruff_cache
.git
.env
*.sqlite
dist
build
```

Why? These files do not need to be copied into the image. Excluding `.env` prevents accidentally baking local secrets into an image.

Do not ignore migrations or source files that the application needs.

## 8. Environment variables

Create `.env.example` and commit it:

```dotenv
APP_ENV=development
DATABASE_URL=postgresql+psycopg://appuser:change-me@db:5432/appdb
REDIS_URL=redis://redis:6379/0
SECRET_KEY=replace-this
```

Create your real `.env` locally and do not commit it.

Important: inside Compose, `db` and `redis` are service hostnames. `localhost` inside the API container means the API container itself, not the PostgreSQL or Redis container.

The host machine can use a published port, for example `localhost:5432`, while containers use `db:5432`.

## 9. Create `compose.yaml`

```yaml
services:
  api:
    build: .
    command: uv run --no-sync uvicorn app.main:app --host 0.0.0.0 --port 8000
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  worker:
    build: .
    command: uv run --no-sync celery -A celery_app:celery_app worker --loglevel=INFO
    env_file:
      - .env
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: change-me
      POSTGRES_DB: appdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

This is an educational starting point. Replace sample passwords with secrets. For production, configure Redis authentication/TLS and a suitable Redis persistence and memory policy. Do not publish database or Redis ports to the public internet.

### Why no `ports` for the database and Redis?

The API and worker can reach them through the private Compose network using service names. You can temporarily publish a database port for local database tools, but remove it or restrict it for production.

### What does `depends_on` do?

The health-conditioned form waits for the dependency health check before starting the dependent service. It does not guarantee that a dependency will never fail later. Applications still need connection retries and resilient error handling.

## 10. Build and start the stack

```bash
docker compose config
docker compose build
docker compose up -d
docker compose ps
```

View logs:

```bash
docker compose logs -f api
docker compose logs -f worker
docker compose logs -f db
docker compose logs -f redis
```

Open:

```text
http://localhost:8000/docs
```

Stop containers:

```bash
docker compose down
```

Stop and delete named volumes (this deletes persisted database/cache data):

```bash
docker compose down -v
```

Do not use `down -v` casually. Your PostgreSQL data lives in the named volume.

## 11. Run database migrations

Do not rely on `SQLModel.metadata.create_all()` as your production schema migration strategy. Use Alembic.

A simple manual command:

```bash
docker compose run --rm api uv run --no-sync alembic upgrade head
```

Run it after the database is ready and before relying on the new API version.

For production deployments, make migrations a controlled one-off release step. Avoid having every API replica independently run migrations at startup.

## 12. Development versus production

### Development

You may mount source code and enable reload:

```yaml
services:
  api:
    build: .
    command: uv run uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    volumes:
      - .:/code
```

This is convenient but not a production configuration.

### Production

- No `--reload`.
- Use a non-root user in the image.
- Keep secrets outside the image.
- Pin and scan base images.
- Use a health endpoint and container health checks.
- Add resource limits appropriate to your host.
- Configure logs and monitoring.
- Use HTTPS at a reverse proxy/load balancer.
- Back up PostgreSQL and test restoration.
- Keep Redis private.
- Use a persistent volume for database data.
- Run API and worker as separate services.
- Set an appropriate number of API workers based on CPU and workload.

## 13. Common Docker networking mistakes

**Mistake:** `DATABASE_URL=...@localhost:5432/...` inside the API container.

**Why wrong:** localhost points to the API container.

**Correct:** `...@db:5432/...` when the Compose service is named `db`.

**Mistake:** Uvicorn uses `--host 127.0.0.1`.

**Why wrong:** other containers/host cannot reach that listener.

**Correct:** `--host 0.0.0.0`.

**Mistake:** using a container's IP address in configuration.

**Why wrong:** container IPs can change.

**Correct:** use Compose service DNS names such as `db` and `redis`.

**Mistake:** deleting a container and assuming its data is safe.

**Why wrong:** data written only to the container's writable layer can disappear when the container is removed.

**Correct:** use persistent volumes and backups for data that matters.

## 14. Useful commands

```bash
docker ps
docker images
docker compose ps
docker compose logs -f
docker compose exec api sh
docker compose exec db psql -U appuser -d appdb
docker compose restart api
docker compose down
docker compose down -v
```

## 15. Practice checklist

- [ ] Build the FastAPI image.
- [ ] Start PostgreSQL and Redis.
- [ ] Connect SQLModel to `db`, not `localhost`.
- [ ] Connect Redis to `redis`, not `localhost`.
- [ ] Run Alembic migrations.
- [ ] Start the Celery worker.
- [ ] Submit a task through an API endpoint.
- [ ] Confirm the worker processes it.
- [ ] Restart the database container and verify named-volume persistence.
- [ ] Confirm secrets are not committed or baked into the image.

## 16. Mental model

Docker packages each service. Compose connects the services into one application environment.

You are not putting PostgreSQL, Redis, Celery, and FastAPI into one giant container. A common design is one service/container for the API, one for the worker, one for PostgreSQL, and one for Redis.
