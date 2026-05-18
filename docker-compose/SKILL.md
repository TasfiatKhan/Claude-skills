---
name: docker-compose
description: Set up or audit Docker + Compose for multi-service apps. Use when the user says "containerize this", "set up Docker", "add a service to compose", or "audit the Docker setup".
---

You are setting up or auditing a Docker + Compose configuration for a multi-service application.

## Core principle

Each service runs in its own container. Containers communicate over a shared network by service name. The host machine only exposes the ports it needs.

## Reference: Witly stack

```yaml
# docker-compose.yml
version: '3.9'

services:
  backend:
    build: ./backend
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app          # live reload in dev
    ports:
      - "8000:8000"
    env_file:
      - ./backend/.env
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: witly
      POSTGRES_USER: witly
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data  # persist across restarts
    ports:
      - "5432:5432"             # expose only in dev — remove in production

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"             # expose only in dev

volumes:
  postgres_data:
```

## Dockerfile (Django)

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

Rules:
- `COPY requirements.txt` before `COPY .` — Docker layer cache means deps only reinstall when requirements change
- `--no-cache-dir` on pip install — smaller image
- `0.0.0.0` not `localhost` — localhost inside a container is unreachable from outside

## Service networking

Services talk to each other by **service name**, not `localhost`:

```python
# backend/.env
DB_HOST=db          # not localhost, not 127.0.0.1 — the service name
REDIS_URL=redis://redis:6379/0
```

This works because Compose puts all services on the same Docker network automatically.

## Common commands

```bash
# Start all services (build if needed)
docker compose up --build

# Start in background
docker compose up -d

# Stop all
docker compose down

# Stop and delete volumes (wipe DB)
docker compose down -v

# Run a command in a running service
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py createsuperuser
docker compose exec db psql -U witly

# View logs
docker compose logs backend
docker compose logs -f backend    # follow

# Rebuild one service
docker compose up --build backend
```

## .env and secrets

```yaml
# In docker-compose.yml
env_file:
  - ./backend/.env      # reads all vars from file
```

Never hardcode secrets in `docker-compose.yml` — use `env_file` or `${VAR}` substitution from a `.env` file at the project root.

## Volumes

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data  # named volume — persists across restarts
  - ./backend:/app                           # bind mount — syncs local files to container
```

- **Named volumes** (`postgres_data:`) — managed by Docker, persist data, survive `docker compose down`
- **Bind mounts** (`./backend:/app`) — mirrors your local directory into the container for live reload in dev
- `docker compose down -v` deletes named volumes — **this wipes your database**

## Production differences

In production:
- Remove bind mounts (`volumes: - ./backend:/app`) — you want the built image, not live code
- Use a production WSGI server: `gunicorn config.wsgi:application --bind 0.0.0.0:8000`
- Remove exposed ports for db and redis — they should not be reachable from the host
- Use secrets management (env vars injected by the platform) not `.env` files

## Common mistakes

- `DB_HOST=localhost` — this points to the container's own loopback, not the DB container. Use the service name.
- No volume for the DB — data is lost every time the container restarts
- Bind mount in production — exposes local files and defeats the purpose of a built image
- Forgetting `depends_on` — backend starts before the DB is ready, connection fails
- `depends_on` alone isn't enough — it waits for the container to start, not for the DB to be ready. Use a health check or retry logic in the app.
