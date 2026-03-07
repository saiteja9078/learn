# 08 — Testing and Deployment

> **Goal**: Write tests for your FastAPI app with pytest, containerize it with Docker (alongside PostgreSQL), and deploy with Uvicorn/Gunicorn.

---

## 🧪 Part 1: Testing with pytest

### Install Test Dependencies

```bash
pip install pytest httpx
# pytest:  The test runner
# httpx:   Async HTTP client (needed for TestClient and async tests)
```

### TestClient (Synchronous Tests)

```python
# tests/test_main.py

from fastapi.testclient import TestClient
from app.main import app

# TestClient wraps your FastAPI app so you can send requests
# WITHOUT starting a real server. It's all in-memory.
client = TestClient(app)


def test_read_root():
    """
    client.get("/") simulates a GET request to your app.
    No network is involved — the request goes directly to your app.
    """
    response = client.get("/")

    assert response.status_code == 200
    assert response.json() == {"message": "Welcome to my API"}


def test_create_user():
    """Test creating a user via POST."""
    response = client.post(
        "/users/",
        json={  # 'json=' automatically sets Content-Type to application/json
            "username": "testuser",
            "email": "test@example.com",
            "password": "secret123"
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["username"] == "testuser"
    assert data["email"] == "test@example.com"
    assert "id" in data
    assert "password" not in data  # Password should NOT be in response


def test_read_nonexistent_user():
    """Test that missing users return 404."""
    response = client.get("/users/99999")
    assert response.status_code == 404
    assert response.json()["detail"] == "User not found"


def test_create_item_for_user():
    """Test creating an item linked to a user."""
    # First create a user
    user_resp = client.post("/users/", json={
        "username": "itemowner",
        "email": "owner@example.com",
        "password": "pass"
    })
    user_id = user_resp.json()["id"]

    # Then create an item for that user
    item_resp = client.post(
        f"/users/{user_id}/items/",
        json={"title": "Laptop", "price": 999.99}
    )
    assert item_resp.status_code == 201
    assert item_resp.json()["title"] == "Laptop"
    assert item_resp.json()["owner_id"] == user_id
```

### Using a Test Database

**Critical**: Never test against your production database! Use a separate test DB.

```python
# tests/conftest.py
# conftest.py is a special pytest file — fixtures defined here are
# available to ALL test files automatically.

import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.database import Base
from app.main import app, get_db

# Use a SEPARATE database for tests
TEST_DATABASE_URL = "postgresql://fastapi_user:fastapi_pass@localhost:5432/fastapi_test_db"
# Create this DB first:
#   docker exec -it fastapi-postgres psql -U fastapi_user -d fastapi_db \
#     -c "CREATE DATABASE fastapi_test_db;"

test_engine = create_engine(TEST_DATABASE_URL)
TestSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=test_engine)


def override_get_db():
    """Replacement dependency that uses the test database."""
    db = TestSessionLocal()
    try:
        yield db
    finally:
        db.close()


@pytest.fixture(scope="function")
def test_client():
    """
    This fixture:
    1. Creates all tables in the TEST database
    2. Overrides the get_db dependency to use the test DB
    3. Yields a TestClient for making requests
    4. Drops all tables after the test (clean state)

    'scope="function"' means each test gets a fresh database.
    """
    # Create tables
    Base.metadata.create_all(bind=test_engine)

    # Override the dependency
    app.dependency_overrides[get_db] = override_get_db

    # Create and yield the client
    with TestClient(app) as client:
        yield client

    # Cleanup: drop all tables
    Base.metadata.drop_all(bind=test_engine)

    # Remove the override
    app.dependency_overrides.clear()


# Now use the fixture in tests:
# tests/test_users.py

def test_create_user(test_client):
    response = test_client.post("/users/", json={
        "username": "alice",
        "email": "alice@example.com",
        "password": "secret"
    })
    assert response.status_code == 201

def test_duplicate_email(test_client):
    # Create first user
    test_client.post("/users/", json={
        "username": "bob",
        "email": "bob@example.com",
        "password": "secret"
    })
    # Try same email again
    response = test_client.post("/users/", json={
        "username": "bob2",
        "email": "bob@example.com",
        "password": "secret"
    })
    assert response.status_code == 400
    assert "already registered" in response.json()["detail"]
```

### Async Testing

```python
# tests/test_async.py

import pytest
from httpx import AsyncClient, ASGITransport
from app.main_async import app

# Mark the entire test as async
@pytest.mark.anyio
async def test_async_root():
    """
    For async endpoints, use httpx.AsyncClient instead of TestClient.
    ASGITransport connects the client directly to your app (no network).
    """
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get("/")
        assert response.status_code == 200

# You need the anyio pytest plugin:
# pip install anyio pytest-anyio
# Or add to pyproject.toml:
# [tool.pytest.ini_options]
# anyio_backend = "asyncio"
```

### Testing Protected Endpoints

```python
# tests/test_auth.py

def test_login_and_access_protected(test_client):
    # 1. Register a user
    test_client.post("/register", json={
        "username": "secureuser",
        "email": "secure@example.com",
        "password": "mypassword"
    })

    # 2. Login (OAuth2 uses form data, not JSON!)
    login_response = test_client.post("/login", data={
        "username": "secureuser",
        "password": "mypassword"
    })
    assert login_response.status_code == 200
    token = login_response.json()["access_token"]

    # 3. Access protected endpoint with the token
    response = test_client.get(
        "/me",
        headers={"Authorization": f"Bearer {token}"}
    )
    assert response.status_code == 200
    assert response.json()["username"] == "secureuser"


def test_protected_without_token(test_client):
    """Accessing a protected route without a token → 401."""
    response = test_client.get("/me")
    assert response.status_code == 401


def test_protected_with_invalid_token(test_client):
    """Using an invalid token → 401."""
    response = test_client.get(
        "/me",
        headers={"Authorization": "Bearer invalidtoken123"}
    )
    assert response.status_code == 401
```

### Running Tests

```bash
# Run all tests
pytest

# Run with verbose output
pytest -v

# Run a specific file
pytest tests/test_users.py

# Run a specific test
pytest tests/test_users.py::test_create_user

# Run with print statements visible
pytest -s

# Run with coverage report
pip install pytest-cov
pytest --cov=app --cov-report=term-missing
```

---

## 🐳 Part 2: Docker Deployment

### Project Structure

```
my-fastapi-app/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   ├── schemas.py
│   ├── crud.py
│   └── security.py
├── tests/
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .env
└── .dockerignore
```

### Requirements File

```txt
# requirements.txt
fastapi[standard]
uvicorn[standard]
sqlalchemy[asyncio]
asyncpg
psycopg2-binary
alembic
python-jose[cryptography]
passlib[bcrypt]
python-multipart
```

### Dockerfile

```dockerfile
# Dockerfile

# ─── Stage 1: Use a Python base image ───
FROM python:3.12-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
# PYTHONDONTWRITEBYTECODE: Don't create .pyc files (smaller image)
# PYTHONUNBUFFERED: Print output immediately (don't buffer)

# Set working directory inside the container
WORKDIR /app

# ─── Install dependencies first (cached layer) ───
# Docker caches each layer. By copying requirements.txt FIRST,
# Docker only re-installs dependencies when requirements.txt changes.
# Your code changes won't trigger a reinstall.
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade -r requirements.txt

# ─── Copy application code ───
COPY ./app ./app

# ─── Expose the port ───
EXPOSE 8000

# ─── Run the application ───
# --host 0.0.0.0: Listen on all interfaces (required for Docker)
# --workers 4: Spawn 4 worker processes (adjust based on CPU cores)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### Docker Compose (App + PostgreSQL)

```yaml
# docker-compose.yml

version: "3.9"

services:
  # ─── PostgreSQL Database ───
  db:
    image: postgres:16
    container_name: fastapi-db
    environment:
      POSTGRES_USER: fastapi_user
      POSTGRES_PASSWORD: fastapi_pass
      POSTGRES_DB: fastapi_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      # Named volume: data persists even if the container is destroyed

  # ─── FastAPI Application ───
  web:
    build: .
    container_name: fastapi-app
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://fastapi_user:fastapi_pass@db:5432/fastapi_db
      # Note: host is "db" (the service name), NOT "localhost"!
      # Docker Compose creates a network where services can reach
      # each other by their service name.
    depends_on:
      - db
      # Start the 'db' service before 'web'
      # Note: this doesn't wait for PostgreSQL to be READY,
      # just for the container to START. For production, use healthchecks.

volumes:
  postgres_data:
    # This creates a named Docker volume that persists your database data
```

### Environment Variables with pydantic-settings

```python
# app/config.py

from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    """
    Reads configuration from environment variables.
    If an env var is not set, it uses the default value.

    Pydantic-settings automatically:
    1. Reads environment variables (case-insensitive)
    2. Reads from .env file (if python-dotenv is installed)
    3. Validates types (e.g., ensures DATABASE_URL is a string)
    """
    database_url: str = "postgresql+asyncpg://fastapi_user:fastapi_pass@localhost:5432/fastapi_db"
    secret_key: str = "dev-secret-key-change-in-production"
    access_token_expire_minutes: int = 30
    debug: bool = False

    class Config:
        env_file = ".env"  # Load from .env file if it exists

# Create a single settings instance
settings = Settings()

# Usage in other files:
# from app.config import settings
# engine = create_async_engine(settings.database_url)
```

```bash
# .env file (NOT committed to git!)
DATABASE_URL=postgresql+asyncpg://fastapi_user:fastapi_pass@localhost:5432/fastapi_db
SECRET_KEY=my-super-secret-production-key-change-this
ACCESS_TOKEN_EXPIRE_MINUTES=60
DEBUG=false
```

### .dockerignore

```
# .dockerignore
__pycache__
*.pyc
.env
.git
.gitignore
venv/
.venv/
tests/
*.md
```

### Running with Docker Compose

```bash
# Build and start both services
docker compose up --build

# Run in the background
docker compose up --build -d

# View logs
docker compose logs -f web

# Stop everything
docker compose down

# Stop and remove volumes (deletes database data!)
docker compose down -v
```

---

## 🚀 Part 3: Production Tips

### Gunicorn + Uvicorn Workers

For production, use **Gunicorn** as a process manager with **Uvicorn workers**:

```bash
pip install gunicorn

# Run with Gunicorn managing Uvicorn workers
gunicorn app.main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --access-logfile - \
  --error-logfile -

# --workers N: Rule of thumb → 2 × CPU cores + 1
# --worker-class: Use Uvicorn's async worker (not Gunicorn's default sync)
# --bind: Listen address
# --access-logfile -: Log to stdout (for Docker/cloud)
```

Update your `Dockerfile` CMD:

```dockerfile
CMD ["gunicorn", "app.main:app", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000"]
```

### Production Checklist

```
✅ Security
  □ Use a strong, random SECRET_KEY (openssl rand -hex 32)
  □ Store secrets in environment variables, not in code
  □ Enable CORS only for your frontend domain
  □ Use HTTPS (e.g., behind Nginx/Caddy reverse proxy)
  □ Set token expiry (30-60 minutes)

✅ Database
  □ Use connection pooling (SQLAlchemy default pool_size=5)
  □ Run migrations with Alembic (not create_all in production)
  □ Use async DB driver (asyncpg) for best performance
  □ Set up database backups

✅ Performance
  □ Use multiple workers (2 × CPU cores + 1)
  □ Use async endpoints with async DB drivers
  □ Add caching (Redis) for frequently accessed data
  □ Use CDN for static files

✅ Observability
  □ Structured logging (json format for log aggregation)
  □ Health check endpoint: GET /health → {"status": "ok"}
  □ Monitor response times and error rates

✅ Docker
  □ Use multi-stage builds to reduce image size
  □ Don't run as root (USER nonroot in Dockerfile)
  □ Use .dockerignore to exclude unnecessary files
  □ Pin dependency versions in requirements.txt
```

### Health Check Endpoint

```python
from fastapi import FastAPI
from sqlalchemy import text

@app.get("/health")
async def health_check(db: AsyncSession = Depends(get_async_db)):
    """
    Health check endpoint. Used by Docker, Kubernetes, and load balancers
    to verify the app is running and can connect to the database.
    """
    try:
        await db.execute(text("SELECT 1"))
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return JSONResponse(
            status_code=503,
            content={"status": "unhealthy", "database": str(e)}
        )
```

---

## 🎉 Congratulations!

You've covered the complete FastAPI stack:

| Chapter                       | What You Learned                                |
| ----------------------------- | ----------------------------------------------- |
| **00 README**                 | What FastAPI is, ASGI, architecture             |
| **01 Getting Started**        | Installation, app anatomy, request lifecycle    |
| **02 Routing & Requests**     | Path/query params, Pydantic, file uploads       |
| **03 Responses & Middleware** | Response models, errors, CORS, middleware chain |
| **04 Dependency Injection**   | Depends(), nesting, yield, caching              |
| **05 Async & Concurrency**    | async/await, event loop, background tasks       |
| **06 Database Integration**   | PostgreSQL Docker, SQLAlchemy, Alembic, CRUD    |
| **07 Authentication**         | OAuth2, JWT, bcrypt, protected routes           |
| **08 Testing & Deployment**   | pytest, Docker Compose, Gunicorn, production    |

**Where to go next:**

- [FastAPI Official Docs](https://fastapi.tiangolo.com/)
- [SQLAlchemy Docs](https://docs.sqlalchemy.org/)
- [Pydantic Docs](https://docs.pydantic.dev/)
- Build a real project — that's the best way to learn! 🚀
