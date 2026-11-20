# FastAPI DevOps – Part 4: Deploying FastAPI on Linux

> Yes: Linux is one of the most common and suitable operating systems for deploying FastAPI. This guide walks from a Linux server to a production-style API using Uvicorn, systemd, Nginx, PostgreSQL, HTTPS, and optional Docker.

## 1. Can Linux run FastAPI?

Absolutely. FastAPI is a Python framework and runs on Linux, Windows, and macOS. Linux is widely used for servers because it is stable, scriptable, well supported by cloud providers, and integrates naturally with tools such as Nginx, systemd, PostgreSQL, Docker, and monitoring systems.

You can deploy on:
- Ubuntu Server.
- Debian.
- Rocky Linux / AlmaLinux.
- Other supported Linux distributions.

For a beginner deployment, Ubuntu Server LTS or Debian stable is a practical choice.

Your Windows development computer can remain your development machine. You can deploy the application to a remote Linux VPS or cloud VM.

## 2. Understand the production request path

A common non-container deployment:

```text
Browser / React frontend
          |
        HTTPS
          |
          v
       Nginx
          |
     reverse proxy
          |
          v
   Uvicorn / FastAPI
      |        |
      v        v
 PostgreSQL   Redis
                 ^
                 |
           Celery worker
```

Roles:

- **Nginx:** accepts public HTTP/HTTPS traffic, handles TLS, serves static files if needed, and proxies API requests.
- **Uvicorn:** ASGI server that runs FastAPI.
- **FastAPI:** application routes, validation, business logic.
- **PostgreSQL:** persistent relational data.
- **Redis:** cache and/or Celery broker.
- **Celery worker:** background task execution.
- **systemd:** starts and supervises Linux services.

Nginx does not replace FastAPI. It sits in front of the application.

## 3. Prepare a Linux server

You need:
- A Linux VM/VPS with a public IP.
- SSH access.
- A domain name (optional initially).
- A non-root deployment user.
- Firewall rules.
- A supported Python version.

Connect from your development machine:

```bash
ssh username@SERVER_PUBLIC_IP
```

Update packages on Ubuntu/Debian:

```bash
sudo apt update
sudo apt upgrade -y
```

Install basic tools:

```bash
sudo apt install -y python3 python3-venv python3-pip \
  nginx git curl ufw
```

Package names and Python versions vary by distribution. Use a supported Python version compatible with your project's dependencies.

## 4. Create a deployment user

Do not run your application as root.

```bash
sudo adduser deploy
sudo usermod -aG www-data deploy
```

The `www-data` group is commonly used by web services on Debian/Ubuntu. Group membership and directory permissions should be designed deliberately; do not grant broad write permissions to your entire application directory.

Switch user:

```bash
su - deploy
```

## 5. Upload or clone your project

Example:

```bash
mkdir -p ~/apps
cd ~/apps
git clone https://github.com/your-account/your-project.git
cd your-project
```

Use your real repository URL. Do not place credentials directly in the Git URL.

For private repositories, configure a secure deploy key or another approved authentication method.

## 6. Create a Python environment

Install `uv` using its official installation instructions, or use Python's built-in `venv`.

Example with venv:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

If the project uses `uv` and a committed lock file:

```bash
uv sync --frozen --no-dev
```

Do not mix package managers casually. Follow the dependency management approach used by the project.

## 7. Configure production environment variables

Create a protected environment file outside the publicly served static directory, for example:

```bash
sudo mkdir -p /etc/fastapi
sudo nano /etc/fastapi/myapi.env
```

Example values:

```dotenv
APP_ENV=production
DATABASE_URL=postgresql+psycopg://appuser:STRONG_PASSWORD@127.0.0.1:5432/appdb
REDIS_URL=redis://127.0.0.1:6379/0
SECRET_KEY=GENERATE_A_LONG_RANDOM_SECRET
```

Restrict access:

```bash
sudo chown root:root /etc/fastapi/myapi.env
sudo chmod 600 /etc/fastapi/myapi.env
```

Use a secret manager or deployment platform secrets where available. Never commit `.env` files or print secret values in logs.

If PostgreSQL/Redis are on separate hosts or containers, use their correct private hostnames and connection security.

## 8. Set up PostgreSQL

Install PostgreSQL if you are running it directly on the same server:

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
```

Create a database and user:

```bash
sudo -u postgres psql
```

Inside `psql`:

```sql
CREATE USER appuser WITH PASSWORD 'REPLACE_WITH_A_STRONG_PASSWORD';
CREATE DATABASE appdb OWNER appuser;
\q
```

Use a strong secret and a restricted database role. The application role should not be a PostgreSQL superuser.

For production, configure PostgreSQL authentication, network binding, backups, updates, disk monitoring, and recovery testing. A managed database can reduce operational work.

## 9. Configure the application database URL

If PostgreSQL is on the same Linux server and listening locally:

```dotenv
DATABASE_URL=postgresql+psycopg://appuser:PASSWORD@127.0.0.1:5432/appdb
```

If using Docker Compose, the host is usually the database service name, such as `db`, not `127.0.0.1`.

URL-encode special characters in passwords or use a configuration method that handles them correctly.

## 10. Run Alembic migrations

From the project directory with the environment loaded:

```bash
alembic upgrade head
```

If your app uses `uv`:

```bash
uv run alembic upgrade head
```

Run migrations as a controlled deployment step. Back up important data first and review migrations before applying them.

## 11. Test Uvicorn manually

From the project root:

```bash
uv run uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Replace `app.main:app` with your actual import path.

Test locally on the server:

```bash
curl http://127.0.0.1:8000/
```

Binding to `127.0.0.1` is intentional in this design because Nginx on the same server will proxy to it. If the API is in a separate container/host, networking must be configured differently.

Do not leave the process running in a terminal for production. Use systemd or a container orchestrator.

## 12. Create a systemd service for FastAPI

Create:

```bash
sudo nano /etc/systemd/system/fastapi.service
```

Example:

```ini
[Unit]
Description=FastAPI application
After=network.target postgresql.service
Wants=postgresql.service

[Service]
User=deploy
Group=www-data
WorkingDirectory=/home/deploy/apps/your-project
EnvironmentFile=/etc/fastapi/myapi.env
ExecStart=/home/deploy/apps/your-project/.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Adjust:
- `WorkingDirectory`
- virtual environment path
- `app.main:app`
- service dependencies

If using `uv`, you can point `ExecStart` to the virtual environment's Uvicorn executable after `uv sync`, or use a carefully configured absolute `uv` path.

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now fastapi
sudo systemctl status fastapi
```

Logs:

```bash
sudo journalctl -u fastapi -f
```

`systemd` keeps the service running and starts it during boot. It is not a replacement for application error handling or monitoring.

## 13. Configure Nginx as a reverse proxy

Create:

```bash
sudo nano /etc/nginx/sites-available/myapi
```

Example:

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/myapi /etc/nginx/sites-enabled/myapi
sudo nginx -t
sudo systemctl reload nginx
```

Nginx receives traffic on port 80 and forwards it to Uvicorn on local port 8000.

Configure FastAPI/Uvicorn trusted proxy handling carefully. Do not blindly trust forwarded headers from arbitrary clients. Only trust the proxy addresses you control.

## 14. Point your domain to the server

At your DNS provider, create records such as:

```text
A     api.example.com     SERVER_PUBLIC_IPV4
```

If you use IPv6, configure an AAAA record only when the server and firewall correctly support IPv6.

DNS changes may take time to propagate. Confirm the domain resolves to the intended server before troubleshooting Nginx.

For a React frontend, you might use:
- `example.com` for the frontend.
- `api.example.com` for FastAPI.

Configure CORS to allow the exact frontend origin, including scheme and port where applicable.

## 15. Enable HTTPS

Use a trusted TLS certificate. On Ubuntu/Debian, Certbot with the Nginx plugin is a common option:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d api.example.com
```

Follow the prompts and verify automatic renewal:

```bash
sudo certbot renew --dry-run
```

HTTPS protects credentials and API traffic in transit. Do not send passwords or bearer tokens over plain HTTP in production.

## 16. Deploying Celery workers with systemd

Create `/etc/systemd/system/celery-worker.service`:

```ini
[Unit]
Description=Celery worker
After=network.target redis-server.service
Wants=redis-server.service

[Service]
User=deploy
Group=www-data
WorkingDirectory=/home/deploy/apps/your-project
EnvironmentFile=/etc/fastapi/myapi.env
ExecStart=/home/deploy/apps/your-project/.venv/bin/celery -A celery_app:celery_app worker --loglevel=INFO
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now celery-worker
sudo systemctl status celery-worker
sudo journalctl -u celery-worker -f
```

Celery Beat, if required, should normally be a separate service from the worker. Run only the intended scheduler instance for a schedule.

## 17. Firewall basics

For a server using SSH, HTTP and HTTPS:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

Before enabling a firewall over SSH, ensure the SSH rule is correct so you do not lock yourself out.

Do not publicly open PostgreSQL port 5432 or Redis port 6379 unless there is a deliberate, secured reason. Prefer private networking.

## 18. React frontend and FastAPI API on one domain

Possible layout:

```text
https://example.com       -> React frontend
https://api.example.com   -> FastAPI API
```

Nginx can serve the built React files and proxy `/` or `/api` to FastAPI, depending on your URL design.

If frontend and API use different origins, configure FastAPI CORS with the exact allowed frontend origin. CORS is a browser policy; it is not authentication or a firewall.

For cookie-based authentication, configure `Secure`, `HttpOnly`, and appropriate `SameSite` settings, and implement CSRF protections where needed.

## 19. Docker-based Linux deployment alternative

Instead of installing Python dependencies directly on the host, you can deploy using Docker Compose:

```text
Linux server
   |
 Docker Engine
   |
 Docker Compose
   |-- api container
   |-- celery worker container
   |-- postgres container or managed DB
   |-- redis container or managed Redis
   +-- nginx / external reverse proxy
```

The same Docker image can run the API and worker with different commands. Keep persistent database data in volumes or use a managed database. Keep secrets outside images and source control.

Docker does not remove the need for backups, TLS, firewall rules, monitoring, migrations, and updates.

## 20. Production checklist

- [ ] Linux server updated.
- [ ] Non-root application user.
- [ ] Supported Python version.
- [ ] Dependencies locked and installed.
- [ ] Production environment variables secured.
- [ ] PostgreSQL configured with least privilege.
- [ ] Alembic migrations reviewed and applied.
- [ ] Uvicorn supervised by systemd or a container platform.
- [ ] Nginx reverse proxy configured.
- [ ] Domain DNS points to the server.
- [ ] HTTPS enabled and renewal tested.
- [ ] Firewall allows only required public ports.
- [ ] Redis and PostgreSQL are not publicly exposed.
- [ ] Celery worker supervised if used.
- [ ] Backups and restore tests exist.
- [ ] Logs, metrics, disk, memory, and service health monitored.
- [ ] CORS, trusted hosts, proxy headers, and authentication configured.
- [ ] Debug mode and development reload disabled.

## 21. A sensible learning path

1. Deploy a tiny FastAPI endpoint on a Linux VM.
2. Run Uvicorn manually and test with `curl`.
3. Create a systemd service.
4. Put Nginx in front of Uvicorn.
5. Attach a domain and HTTPS.
6. Add PostgreSQL and Alembic.
7. Add Redis.
8. Add Celery worker.
9. Repeat the deployment using Docker Compose.

This sequence helps you understand each layer instead of hiding everything inside a deployment script.
