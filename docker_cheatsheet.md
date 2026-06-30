# Docker Cheatsheet

> **Legend:** Lines marked `# [compose]` apply to Docker Compose only. Everything else applies to plain Docker CLI.

---

## 1. Core Concepts

| Concept | What it is | Analogy |
|---|---|---|
| **Image** | Read-only blueprint (layers) | House blueprint |
| **Container** | Running instance of an image | Actual house |
| **Volume** | Persistent storage outside the container | USB stick / external drive |
| **Network** | Communication channel between containers | Telephone system |
| **Registry** | Repository to store and share images | App store (e.g. DockerHub) |

---

## 2. Images

```bash
docker pull python:3.11-slim          # download image from DockerHub
docker images                          # list local images
docker image ls                        # same as above
docker image rm python:3.11-slim       # delete local image
docker rmi python:3.11-slim            # shorthand
docker image prune                     # remove dangling (untagged) images
docker image inspect python:3.11-slim  # full metadata as JSON
docker history python:3.11-slim        # show layer history and sizes
```

---

## 3. Containers

```bash
# Run
docker run python:3.11-slim                          # run and exit
docker run -it python:3.11-slim bash                 # interactive shell (-i keep stdin, -t pseudo-TTY)
docker run -d python:3.11-slim                       # detached (background)
docker run --name my-app python:3.11-slim            # custom container name
docker run -p 8080:80 nginx                          # map host port 8080 → container port 80
docker run -e MY_VAR=hello python:3.11-slim          # set environment variable
docker run --rm python:3.11-slim python -c "print(1)" # auto-remove on exit

# Manage
docker ps                    # list running containers
docker ps -a                 # include stopped containers
docker stop my-app           # graceful stop (SIGTERM → SIGKILL after timeout)
docker start my-app          # restart stopped container
docker restart my-app        # stop + start
docker rm my-app             # remove stopped container
docker rm -f my-app          # force-remove running container

# Inspect / Debug
docker logs my-app           # stdout / stderr
docker logs -f my-app        # follow (tail -f equivalent)
docker exec -it my-app bash  # open shell inside running container
docker exec my-app ls /app   # run one-off command inside container
docker inspect my-app        # full metadata as JSON
docker stats                 # live CPU / memory usage
docker top my-app            # running processes inside container
docker cp my-app:/app/out.csv ./out.csv  # copy file out of container
```

---

## 4. Dockerfile

```dockerfile
FROM python:3.11-slim          # base image (always first)

LABEL maintainer="you@example.com"  # optional metadata

ENV PYTHONDONTWRITEBYTECODE=1  # set env var at build time and runtime
ARG APP_VERSION=1.0            # build-time only variable

WORKDIR /app                   # set working directory (creates if missing)

COPY requirements.txt .        # copy from build context into image
COPY . .                       # copy everything (after requirements for cache)
ADD archive.tar.gz /app/       # like COPY but also unpacks archives

RUN pip install --no-cache-dir -r requirements.txt  # run command in new layer

EXPOSE 5000                    # document the port (does NOT publish it)
VOLUME ["/app/data"]           # declare a mount point

USER appuser                   # switch to non-root user

ENTRYPOINT ["python"]          # fixed executable (not overridden by args)
CMD ["app.py"]                 # default args to ENTRYPOINT (or default command)
```

### Build & Tag

```bash
docker build -t my-app:1.0 .          # build from Dockerfile in current dir
docker build -t my-app:1.0 -f Dockerfile.prod .  # specify Dockerfile
docker build --no-cache -t my-app .   # ignore layer cache
docker tag my-app:1.0 myuser/my-app:latest  # add registry tag
docker push myuser/my-app:latest      # push to DockerHub
```

### .dockerignore

```
__pycache__/
*.pyc
.env
.git
.venv
*.csv          # exclude large data files from build context
```

---

## 5. Image Optimization

```dockerfile
# 1. Use slim / alpine base images
FROM python:3.11-slim        # ~50 MB vs ~900 MB for python:3.11

# 2. Combine RUN steps to reduce layers
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# 3. Install dependencies BEFORE copying source (cache invalidation)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .                     # source changes don't bust the pip layer

# 4. Multi-stage build — keep build tools out of final image
FROM python:3.11-slim AS builder
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen

FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY app.py .
CMD ["python", "app.py"]
```

---

## 6. Volumes

```bash
# Named volumes (Docker-managed, survives container removal)
docker volume create pg-data
docker run -v pg-data:/var/lib/postgresql/data postgres:16
docker volume ls
docker volume inspect pg-data
docker volume rm pg-data
docker volume prune           # remove all unused volumes

# Bind mounts (maps host directory into container)
docker run -v $(pwd)/data:/app/data python:3.11-slim  # host path : container path
docker run -v /absolute/path:/app/data python:3.11-slim

# Read-only mount
docker run -v $(pwd)/config:/app/config:ro python:3.11-slim
```

---

## 7. Networks

```bash
docker network ls                            # list networks
docker network create my-net                 # custom bridge network
docker network inspect my-net
docker network rm my-net

# Connect containers to same network → reach each other by container name
docker run -d --name postgres-db --network my-net postgres:16
docker run -d --name app --network my-net my-app:1.0

docker network connect my-net existing-container   # add existing container
docker network disconnect my-net container-name
```

| Network driver | Use case |
|---|---|
| `bridge` (default) | Single-host container communication |
| `host` | Container shares host network (Linux only) |
| `none` | Fully isolated, no networking |

---

## 8. Docker Compose

### docker-compose.yml structure

```yaml
services:

  db:
    image: postgres:16
    container_name: postgres-db-demo
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}   # read from .env file
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend

  app:
    build: .                              # build Dockerfile in current dir
    container_name: app-demo
    depends_on:
      - db                                # start db first (no readiness check)
    environment:
      - DATABASE_URL=postgresql://postgres:${DB_PASSWORD}@postgres-db-demo:5432/postgres
    ports:
      - "8080:80"
    volumes:
      - ./data:/app/data                  # bind mount for local dev
    networks:
      - backend
    restart: unless-stopped

volumes:
  postgres_data:                          # declare named volume

networks:
  backend:                                # declare custom network
```

### Compose lifecycle commands

```bash
# [compose]
docker compose up               # build (if needed), create, and start all services
docker compose up -d            # detached
docker compose up --build       # force rebuild images
docker compose up app           # start only the 'app' service

docker compose down             # stop and remove containers + networks
docker compose down -v          # also remove named volumes (deletes data!)

docker compose ps               # list services and their status
docker compose logs             # all service logs
docker compose logs -f app      # follow logs for 'app' service

docker compose start            # start stopped services (no recreate)
docker compose stop             # stop without removing
docker compose restart app      # restart one service

docker compose exec app bash    # shell into running service
docker compose run app python script.py  # one-off command in new container
docker compose build            # build images without starting
docker compose pull             # pull latest images
docker compose config           # validate and print resolved compose file
```

### .env file (auto-loaded by Compose)

```bash
# .env  ← always add to .gitignore, commit a .env.example instead
DB_PASSWORD=admin
POSTGRES_USER=postgres
APP_PORT=8080
```

Compose has **two separate uses** for `.env` — easy to confuse:

| | What it does | How to enable |
|---|---|---|
| **Variable substitution** | Fills `${VAR}` placeholders in the YAML itself | Auto — just have a `.env` beside `docker-compose.yml` |
| **Container injection** | Makes vars available inside the running container | Requires `env_file:` directive on the service |

```yaml
services:
  app:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}   # substitution: Compose reads .env to build the YAML
    env_file:
      - .env                              # injection: every key in .env lands inside the container
    ports:
      - "${APP_PORT}:80"                  # substitution again
```

Without `env_file:`, the container won't see `DB_PASSWORD` even though Compose used it for substitution.

**When to use `.env`:**
- Values that differ per environment (dev / staging / prod) — passwords, ports, hostnames
- Keeping secrets out of `docker-compose.yml` (which gets committed)

**When not to bother:**
- Value is the same everywhere and non-sensitive — just hardcode it in the YAML
- Need multiple env files — use `--env-file` flag or per-service `env_file:` instead

```bash
docker compose --env-file .env.prod up -d   # override the default .env
```

---

## 9. Common Patterns

### Connect Python app to PostgreSQL (SQLAlchemy)

```python
# Use container_name (or service name) as host, NOT localhost
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:admin@postgres-db-demo:5432/postgres")
```

### Load CSV into PostgreSQL inside Docker

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:admin@postgres-db-demo:5432/postgres")
df = pd.read_csv("data/records.csv")
df.to_sql("records", engine, if_exists="replace", index=False)
```

### Minimal Python Dockerfile with uv

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen
COPY app.py .
CMD ["uv", "run", "app.py"]
```

---

## 10. Cleanup

```bash
docker system prune            # remove stopped containers, dangling images, unused networks
docker system prune -a         # also remove unused images (not just dangling)
docker system prune -a --volumes  # also removes anonymous (unnamed) volumes — named volumes are NOT removed
docker volume prune                # remove all unused named volumes
docker system df               # show disk usage breakdown
```

---

## 11. Quick Reference

| Task | Command |
|---|---|
| Run PostgreSQL locally | `docker run -d --name pg -e POSTGRES_PASSWORD=admin -p 5432:5432 postgres:16` |
| Open psql shell | `docker exec -it pg psql -U postgres` |
| Build image | `docker build -t app:1.0 .` |
| Run container with env file | `docker run --env-file .env app:1.0` |
| Start all services | `docker compose up -d` |
| Tear down + wipe data | `docker compose down -v` |
| Follow logs | `docker compose logs -f` |
| Force rebuild | `docker compose up --build` |
| Shell into service | `docker compose exec app bash` |
