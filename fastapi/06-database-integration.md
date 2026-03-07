# 06 — Database Integration

> **Goal**: Connect FastAPI to PostgreSQL (running in Docker), use SQLAlchemy for both sync and async operations, run migrations with Alembic, and build a complete CRUD API.

---

## 🐳 Setting Up PostgreSQL with Docker

```bash
# Pull and run PostgreSQL in a Docker container
docker run -d \
  --name fastapi-postgres \
  -e POSTGRES_USER=fastapi_user \
  -e POSTGRES_PASSWORD=fastapi_pass \
  -e POSTGRES_DB=fastapi_db \
  -p 5432:5432 \
  postgres:16

# Verify it's running
docker ps

# Connect to it (optional, to check manually)
docker exec -it fastapi-postgres psql -U fastapi_user -d fastapi_db
```

**What this does:**
| Flag | Meaning |
|------|---------|
| `-d` | Run in the background (detached) |
| `--name fastapi-postgres` | Name the container for easy reference |
| `-e POSTGRES_USER=...` | Set the database username |
| `-e POSTGRES_PASSWORD=...` | Set the database password |
| `-e POSTGRES_DB=...` | Create this database on startup |
| `-p 5432:5432` | Map container port 5432 to host port 5432 |

Your **connection URL** is:

```
postgresql://fastapi_user:fastapi_pass@localhost:5432/fastapi_db
```

---

## 📦 Install Dependencies

```bash
# For SYNCHRONOUS database access
pip install sqlalchemy psycopg2-binary

# For ASYNCHRONOUS database access (recommended)
pip install sqlalchemy[asyncio] asyncpg

# For migrations
pip install alembic
```

| Package           | Purpose                                                  |
| ----------------- | -------------------------------------------------------- |
| `sqlalchemy`      | Python ORM — maps Python classes to database tables      |
| `psycopg2-binary` | Synchronous PostgreSQL driver                            |
| `asyncpg`         | Asynchronous PostgreSQL driver (much faster)             |
| `alembic`         | Database migration tool (alter tables without data loss) |

---

## 🏗️ Part 1: Synchronous SQLAlchemy (Classic)

### Project Structure

```
app/
├── main.py          # FastAPI app
├── database.py      # Database engine and session setup
├── models.py        # SQLAlchemy models (database tables)
├── schemas.py       # Pydantic models (request/response shapes)
└── crud.py          # CRUD operations
```

### Step 1: Database Setup (`database.py`)

```python
# app/database.py

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

# Connection URL for our Docker PostgreSQL
# Format: postgresql://USER:PASSWORD@HOST:PORT/DATABASE
DATABASE_URL = "postgresql://fastapi_user:fastapi_pass@localhost:5432/fastapi_db"

# ─── Create the Engine ───
# The engine is the starting point for SQLAlchemy.
# It manages a POOL of database connections.
#
# Think of it as: "the thing that knows HOW to talk to the database"
#
# pool_pre_ping=True: Before using a connection, check if it's still alive.
#   This prevents errors from stale connections.
engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True,
    # pool_size=5,       # Number of persistent connections (default: 5)
    # max_overflow=10,   # Extra connections allowed above pool_size
    # echo=True,         # Print all SQL queries to console (debug only!)
)

# ─── Create a Session Factory ───
# A session is a "workspace" for your database operations.
# sessionmaker creates a factory that produces new sessions.
#
# autocommit=False: Changes are NOT saved until you call commit()
# autoflush=False:  Don't auto-sync Python objects with DB
# bind=engine:      Use this engine for connections
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

# ─── Base class for all models ───
# All your database models will inherit from this.
# SQLAlchemy uses this to keep track of all your tables.
class Base(DeclarativeBase):
    pass
```

### Step 2: Database Models (`models.py`)

```python
# app/models.py

from sqlalchemy import Column, Integer, String, Float, Boolean, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from app.database import Base

class User(Base):
    """
    This class maps to a 'users' table in PostgreSQL.

    Each attribute with Column() becomes a column in the table.
    SQLAlchemy generates SQL like:
        CREATE TABLE users (
            id SERIAL PRIMARY KEY,
            username VARCHAR NOT NULL UNIQUE,
            email VARCHAR NOT NULL UNIQUE,
            hashed_password VARCHAR NOT NULL,
            is_active BOOLEAN DEFAULT TRUE,
            created_at TIMESTAMP DEFAULT NOW()
        );
    """
    __tablename__ = "users"  # Name of the table in the database

    id = Column(Integer, primary_key=True, index=True)
    # primary_key=True: Auto-incrementing ID
    # index=True: Create a database index for faster lookups

    username = Column(String, unique=True, nullable=False, index=True)
    # unique=True: No two users can have the same username
    # nullable=False: This field CANNOT be NULL (required)

    email = Column(String, unique=True, nullable=False, index=True)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)

    created_at = Column(DateTime(timezone=True), server_default=func.now())
    # server_default=func.now(): The DEFAULT is set by PostgreSQL, not Python
    # This means the timestamp is generated by the DB server's clock

    # ─── Relationship ───
    # This creates a virtual attribute (not a real column).
    # user.items will return all Item objects that belong to this user.
    items = relationship("Item", back_populates="owner")
    # back_populates="owner": Item.owner will point back to this User


class Item(Base):
    __tablename__ = "items"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, nullable=False, index=True)
    description = Column(String, nullable=True)
    price = Column(Float, nullable=False)
    is_available = Column(Boolean, default=True)

    # ─── Foreign Key ───
    owner_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    # ForeignKey("users.id"): This column references the 'id' column
    # in the 'users' table. PostgreSQL enforces this constraint.

    owner = relationship("User", back_populates="items")
    # item.owner returns the User object that owns this item
```

### Step 3: Pydantic Schemas (`schemas.py`)

```python
# app/schemas.py
#
# WHY separate schemas from models?
# - Models (models.py) = DATABASE structure (SQLAlchemy)
# - Schemas (schemas.py) = API structure (Pydantic)
#
# They look similar but serve different purposes:
# - Schemas validate INPUT from clients and shape OUTPUT to clients
# - Models define how data is STORED in PostgreSQL

from pydantic import BaseModel, ConfigDict
from typing import Optional, List
from datetime import datetime

# ────── Item Schemas ──────

class ItemBase(BaseModel):
    """Fields shared between create and read."""
    title: str
    description: Optional[str] = None
    price: float
    is_available: bool = True

class ItemCreate(ItemBase):
    """Fields needed to CREATE an item (no id, no owner_id)."""
    pass

class ItemResponse(ItemBase):
    """Fields returned when READING an item."""
    id: int
    owner_id: int

    # This tells Pydantic to read data from SQLAlchemy model attributes
    # Without this, Pydantic can't convert SQLAlchemy objects to JSON
    model_config = ConfigDict(from_attributes=True)
    # What from_attributes=True does:
    #   Instead of expecting a dict like {"id": 1, "title": "..."}
    #   Pydantic can read from object attributes: item.id, item.title
    #   This is needed because SQLAlchemy returns objects, not dicts

# ────── User Schemas ──────

class UserBase(BaseModel):
    username: str
    email: str

class UserCreate(UserBase):
    password: str  # Plain text password (we'll hash it before storing)

class UserResponse(UserBase):
    id: int
    is_active: bool
    created_at: datetime
    items: List[ItemResponse] = []  # Include user's items in response

    model_config = ConfigDict(from_attributes=True)
```

### Step 4: CRUD Operations (`crud.py`)

```python
# app/crud.py

from sqlalchemy.orm import Session
from app import models, schemas

# ────── Users ──────

def get_user(db: Session, user_id: int):
    """
    db.query(models.User): Start a query on the 'users' table
    .filter(...):          Add a WHERE clause
    .first():              Return the first result (or None)

    Generated SQL: SELECT * FROM users WHERE id = :user_id LIMIT 1
    """
    return db.query(models.User).filter(models.User.id == user_id).first()

def get_user_by_email(db: Session, email: str):
    return db.query(models.User).filter(models.User.email == email).first()

def get_users(db: Session, skip: int = 0, limit: int = 100):
    """
    .offset(skip): Skip the first N rows (for pagination)
    .limit(limit): Return at most N rows

    SQL: SELECT * FROM users OFFSET :skip LIMIT :limit
    """
    return db.query(models.User).offset(skip).limit(limit).all()

def create_user(db: Session, user: schemas.UserCreate):
    """
    1. Create a SQLAlchemy model instance
    2. Add it to the session (schedules an INSERT)
    3. Commit the transaction (executes the INSERT)
    4. Refresh the object (reload from DB to get auto-generated fields like id)
    """
    # In production, hash the password! (see chapter 07)
    fake_hashed_password = user.password + "_hashed"

    db_user = models.User(
        username=user.username,
        email=user.email,
        hashed_password=fake_hashed_password,
    )
    db.add(db_user)        # Stage the INSERT (not yet executed)
    db.commit()            # Execute: INSERT INTO users VALUES (...)
    db.refresh(db_user)    # Reload: SELECT * FROM users WHERE id = :new_id
    return db_user

# ────── Items ──────

def get_items(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.Item).offset(skip).limit(limit).all()

def create_item(db: Session, item: schemas.ItemCreate, user_id: int):
    db_item = models.Item(**item.model_dump(), owner_id=user_id)
    # item.model_dump() converts Pydantic model to dict:
    #   {"title": "Widget", "description": "...", "price": 29.99, "is_available": True}
    # **dict unpacks it as keyword arguments
    # owner_id=user_id sets the foreign key
    db.add(db_item)
    db.commit()
    db.refresh(db_item)
    return db_item
```

### Step 5: Main Application (`main.py`)

```python
# app/main.py

from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List
from app import crud, models, schemas
from app.database import SessionLocal, engine

# Create all tables in the database
# This runs: CREATE TABLE IF NOT EXISTS ... for each model
models.Base.metadata.create_all(bind=engine)

app = FastAPI(title="FastAPI + PostgreSQL")

# ─── Database Dependency ───
def get_db():
    """
    Creates a new database session for each request.
    The 'yield' pattern ensures the session is always closed.
    """
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# ────── User Endpoints ──────

@app.post("/users/", response_model=schemas.UserResponse, status_code=201)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    # Check if email already exists
    db_user = crud.get_user_by_email(db, email=user.email)
    if db_user:
        raise HTTPException(status_code=400, detail="Email already registered")
    return crud.create_user(db=db, user=user)

@app.get("/users/", response_model=List[schemas.UserResponse])
def read_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_users(db, skip=skip, limit=limit)

@app.get("/users/{user_id}", response_model=schemas.UserResponse)
def read_user(user_id: int, db: Session = Depends(get_db)):
    db_user = crud.get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user

# ────── Item Endpoints ──────

@app.post("/users/{user_id}/items/", response_model=schemas.ItemResponse, status_code=201)
def create_item_for_user(
    user_id: int,
    item: schemas.ItemCreate,
    db: Session = Depends(get_db)
):
    # Verify user exists
    db_user = crud.get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return crud.create_item(db=db, item=item, user_id=user_id)

@app.get("/items/", response_model=List[schemas.ItemResponse])
def read_items(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud.get_items(db, skip=skip, limit=limit)
```

---

## ⚡ Part 2: Async SQLAlchemy (Recommended)

For better performance, use **async** database operations. This lets FastAPI handle many concurrent requests without blocking.

### Async Database Setup

```python
# app/database_async.py

from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase

# Note the different URL scheme: postgresql+asyncpg://
# 'asyncpg' is the async PostgreSQL driver
DATABASE_URL = "postgresql+asyncpg://fastapi_user:fastapi_pass@localhost:5432/fastapi_db"

# Async engine — same concept as sync engine, but non-blocking
async_engine = create_async_engine(
    DATABASE_URL,
    echo=False,          # Set True to log all SQL queries
    pool_size=5,
    max_overflow=10,
)

# Async session factory
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,    # Use async session class
    expire_on_commit=False  # Don't expire objects after commit
    # expire_on_commit=False is important for async:
    # Without it, accessing attributes after commit would trigger
    # a lazy load, which requires a synchronous call — and that
    # would fail in an async context.
)

class Base(DeclarativeBase):
    pass
```

### Async Dependency

```python
# app/dependencies.py

from app.database_async import AsyncSessionLocal

async def get_async_db():
    """
    Same pattern as sync, but with 'async with'.
    This ensures the session is properly closed even if an error occurs.
    """
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

### Async CRUD Operations

```python
# app/crud_async.py

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app import models, schemas

async def get_user(db: AsyncSession, user_id: int):
    """
    In async SQLAlchemy, you use 'select()' instead of 'db.query()'.
    This is the "2.0 style" query API.

    select(models.User) → SELECT * FROM users
    .where(...)         → WHERE id = :user_id

    'await db.execute(stmt)' sends the query to PostgreSQL
    '.scalars().first()' extracts the first result object
    """
    stmt = select(models.User).where(models.User.id == user_id)
    result = await db.execute(stmt)
    return result.scalars().first()

async def get_users(db: AsyncSession, skip: int = 0, limit: int = 100):
    stmt = select(models.User).offset(skip).limit(limit)
    result = await db.execute(stmt)
    return result.scalars().all()

async def create_user(db: AsyncSession, user: schemas.UserCreate):
    db_user = models.User(
        username=user.username,
        email=user.email,
        hashed_password=user.password + "_hashed",
    )
    db.add(db_user)
    await db.commit()        # async commit
    await db.refresh(db_user)  # async reload from DB
    return db_user
```

### Async Endpoints

```python
# app/main_async.py

from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List
from app import models, schemas
from app.crud_async import get_user, get_users, create_user
from app.dependencies import get_async_db
from app.database_async import async_engine, Base

from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Create tables on startup
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
        # run_sync() runs a synchronous function in the async context
        # Base.metadata.create_all is synchronous, so we wrap it
    yield
    # Cleanup on shutdown
    await async_engine.dispose()

app = FastAPI(title="FastAPI + Async PostgreSQL", lifespan=lifespan)

@app.post("/users/", response_model=schemas.UserResponse, status_code=201)
async def create_new_user(
    user: schemas.UserCreate,
    db: AsyncSession = Depends(get_async_db)  # Async dependency!
):
    return await create_user(db=db, user=user)

@app.get("/users/", response_model=List[schemas.UserResponse])
async def read_users(
    skip: int = 0,
    limit: int = 100,
    db: AsyncSession = Depends(get_async_db)
):
    return await get_users(db, skip=skip, limit=limit)

@app.get("/users/{user_id}", response_model=schemas.UserResponse)
async def read_user(user_id: int, db: AsyncSession = Depends(get_async_db)):
    db_user = await get_user(db, user_id=user_id)
    if db_user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return db_user
```

---

## 🔄 Alembic Migrations

Instead of `create_all()`, production apps use **Alembic** to manage database changes (add columns, rename tables, etc.) without losing data.

```bash
# Initialize Alembic in your project
alembic init alembic

# This creates:
# alembic/
#   env.py          ← Configuration (edit this)
#   versions/       ← Migration scripts go here
# alembic.ini       ← Connection URL (edit this)
```

```ini
# alembic.ini — Set the database URL
sqlalchemy.url = postgresql://fastapi_user:fastapi_pass@localhost:5432/fastapi_db
```

```python
# alembic/env.py — Point to your models
# Add this near the top:
from app.database import Base
from app import models  # Import all models so Alembic sees them

target_metadata = Base.metadata  # Replace the default None
```

```bash
# Generate a migration (Alembic compares models to DB and creates a script)
alembic revision --autogenerate -m "create users and items tables"

# Apply the migration (run the SQL against the database)
alembic upgrade head

# Other useful commands:
alembic downgrade -1    # Undo the last migration
alembic history         # Show all migrations
alembic current         # Show current database version
```

---

## 🔬 Internals: Connection Pooling

```
When your FastAPI app starts, SQLAlchemy creates a CONNECTION POOL:

┌──── SQLAlchemy Engine ─────────────────────────┐
│                                                 │
│  Connection Pool (pool_size=5, max_overflow=10) │
│  ┌─────┬─────┬─────┬─────┬─────┐               │
│  │Conn1│Conn2│Conn3│Conn4│Conn5│  ← 5 ready    │
│  └─────┴─────┴─────┴─────┴─────┘               │
│  + up to 10 overflow connections                │
│                                                 │
│  Request comes in:                              │
│    1. "Borrow" a connection from the pool       │
│    2. Use it for queries                        │
│    3. "Return" it to the pool (don't close it!) │
│                                                 │
│  Why? Opening a new PostgreSQL connection       │
│  takes ~50-100ms. Reusing one takes ~0.1ms.     │
└─────────────────────────────────────────────────┘

The get_db() dependency borrows a connection, yields it,
then returns it to the pool in the 'finally' block.
```

---

## ➡️ Next Steps

Head to **[07 — Authentication and Security](./07-authentication-and-security.md)** to add OAuth2, JWT tokens, and password hashing to protect your API endpoints.
