# 04 — Dependency Injection

> **Goal**: Understand FastAPI's built-in dependency injection system — `Depends()` — which is the backbone for database sessions, authentication, shared configuration, and reusable logic.

---

## 🧠 What is Dependency Injection (DI)?

**Dependency Injection** means: instead of a function creating the things it needs (its "dependencies"), those things are _provided_ to it from the outside.

```python
# WITHOUT Dependency Injection (bad)
def get_items():
    db = DatabaseConnection()  # The function creates its own dependency
    items = db.query("SELECT * FROM items")
    db.close()
    return items

# WITH Dependency Injection (good)
def get_items(db: DatabaseConnection):  # The dependency is "injected" from outside
    items = db.query("SELECT * FROM items")
    return items
```

**Why is DI good?**

- **Testability**: Easy to swap real DB for a test DB
- **Reusability**: The same dependency can be used across many endpoints
- **Separation of concerns**: Your endpoint doesn't know _how_ to create a DB connection

FastAPI has **built-in DI** via `Depends()`. No external library needed.

---

## 🔧 Basic Depends()

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# ─────────────────────────────────────────────
# Step 1: Define a dependency (it's just a function!)
# ─────────────────────────────────────────────
def get_common_params(skip: int = 0, limit: int = 10):
    """
    This is a "dependency function".
    It's just a regular Python function that returns something.

    FastAPI will:
    1. Call this function BEFORE your endpoint
    2. Resolve its parameters (skip, limit) from the query string
    3. Pass the return value to your endpoint
    """
    return {"skip": skip, "limit": limit}


# ─────────────────────────────────────────────
# Step 2: Use it in endpoints with Depends()
# ─────────────────────────────────────────────
@app.get("/items/")
def read_items(commons: dict = Depends(get_common_params)):
    """
    Depends(get_common_params) tells FastAPI:
      "Before running this endpoint, call get_common_params(),
       and inject its return value as 'commons'."

    The query parameters 'skip' and 'limit' are automatically
    extracted by FastAPI because they're in get_common_params's signature.

    Example: GET /items/?skip=5&limit=20
    → get_common_params(skip=5, limit=20) is called first
    → returns {"skip": 5, "limit": 20}
    → commons = {"skip": 5, "limit": 20}
    """
    return {"params": commons}


@app.get("/users/")
def read_users(commons: dict = Depends(get_common_params)):
    """
    Same dependency, reused in a different endpoint!
    Both /items/ and /users/ accept 'skip' and 'limit' query parameters.
    """
    return {"params": commons}
```

### What Happens Under the Hood

```
Request: GET /items/?skip=5&limit=20

1. FastAPI receives the request
2. It sees: read_items has a parameter commons = Depends(get_common_params)
3. FastAPI inspects get_common_params's signature:
     skip: int = 0  → query parameter
     limit: int = 10 → query parameter
4. FastAPI extracts skip=5 and limit=20 from the query string
5. FastAPI calls: get_common_params(skip=5, limit=20)
6. The return value {"skip": 5, "limit": 20} is passed as 'commons'
7. FastAPI calls: read_items(commons={"skip": 5, "limit": 20})
8. Your endpoint runs and returns the response
```

---

## 🏛️ Class-Based Dependencies

You can use classes as dependencies. This is useful when your dependency has configuration or state.

```python
from fastapi import FastAPI, Depends

app = FastAPI()

class Pagination:
    """
    A class-based dependency for pagination.

    When used with Depends(), FastAPI calls Pagination(skip=..., limit=...)
    which triggers __init__. The resulting instance is passed to your endpoint.
    """
    def __init__(self, skip: int = 0, limit: int = 10):
        self.skip = skip
        self.limit = limit

# Two equivalent ways to use it:

# Way 1: Depends(ClassName)
@app.get("/items/")
def read_items(pagination: Pagination = Depends(Pagination)):
    return {"skip": pagination.skip, "limit": pagination.limit}

# Way 2: Depends() shorthand (uses the type hint)
@app.get("/users/")
def read_users(pagination: Pagination = Depends()):
    """
    When you write Depends() with no argument, FastAPI looks at the
    type hint (Pagination) and uses that as the dependency.

    This is equivalent to Depends(Pagination).
    """
    return {"skip": pagination.skip, "limit": pagination.limit}
```

---

## 🪆 Nested Dependencies

Dependencies can depend on other dependencies. FastAPI resolves the entire tree.

```python
from fastapi import FastAPI, Depends, Header, HTTPException

app = FastAPI()

# ─────── Level 1: Lowest-level dependency ───────
def get_api_key(x_api_key: str = Header(...)):
    """
    Extract the API key from the X-API-Key header.
    This is a dependency used by other dependencies.
    """
    return x_api_key


# ─────── Level 2: Depends on Level 1 ───────
def verify_api_key(api_key: str = Depends(get_api_key)):
    """
    Verify the API key is valid.
    This dependency DEPENDS ON get_api_key (nested!).
    """
    valid_keys = {"secret-key-1", "secret-key-2"}
    if api_key not in valid_keys:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key


# ─────── Level 3: Depends on Level 2 ───────
def get_current_user(api_key: str = Depends(verify_api_key)):
    """
    Get the user associated with the API key.
    This dependency DEPENDS ON verify_api_key, which depends on get_api_key.
    """
    # In a real app, you'd look up the user in a database
    users = {"secret-key-1": "Alice", "secret-key-2": "Bob"}
    return {"username": users[api_key], "api_key": api_key}


# ─────── Your endpoint ───────
@app.get("/protected/")
def protected_route(user: dict = Depends(get_current_user)):
    """
    The dependency chain that FastAPI resolves:

    1. get_api_key()         → reads X-API-Key header
    2. verify_api_key()      → validates the key
    3. get_current_user()    → looks up the user
    4. protected_route()     → runs your endpoint with the user

    If any step fails (missing header, invalid key), the chain stops
    and an error response is returned immediately.
    """
    return {"message": f"Hello, {user['username']}!"}
```

### Dependency Resolution Tree

```
protected_route
└── get_current_user
    └── verify_api_key
        └── get_api_key
            └── Header("X-API-Key")  ← reads from request

FastAPI resolves this bottom-up:
  1. Read X-API-Key header
  2. Pass to verify_api_key
  3. Pass to get_current_user
  4. Pass to protected_route
```

---

## 🔄 Yield Dependencies (Setup + Cleanup)

Some dependencies need cleanup after the request is done (e.g., closing a database session). Use `yield` for this:

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# ─────────────────────────────────────────────
# Simulated database session
# ─────────────────────────────────────────────
class FakeDBSession:
    """Simulates a database session."""
    def __init__(self):
        print("  📂 Opening DB session")

    def query(self, sql: str):
        print(f"  🔍 Executing: {sql}")
        return [{"id": 1, "name": "Item 1"}]

    def close(self):
        print("  📁 Closing DB session")


def get_db():
    """
    A dependency that YIELDS a database session.

    Code before 'yield': SETUP (runs before the endpoint)
    The yielded value: INJECTED into the endpoint
    Code after 'yield': CLEANUP (runs after the endpoint, even if it errors!)

    This is the exact same pattern as Python's context manager:
        with open("file.txt") as f:
            # use f
        # f is automatically closed

    FastAPI uses this pattern to ensure resources are always cleaned up.
    """
    db = FakeDBSession()       # SETUP: Create the session
    try:
        yield db               # INJECT: Pass session to the endpoint
    finally:
        db.close()             # CLEANUP: Always close, even if endpoint crashed


@app.get("/items/")
def read_items(db: FakeDBSession = Depends(get_db)):
    """
    FastAPI:
    1. Calls get_db() → creates FakeDBSession
    2. Reaches 'yield db' → pauses get_db()
    3. Passes db to read_items()
    4. Runs read_items()
    5. After read_items() completes (or crashes), resumes get_db()
    6. Executes 'finally: db.close()'

    Timeline:
      📂 Opening DB session
      🔍 Executing: SELECT * FROM items
      📁 Closing DB session  ← Always happens!
    """
    items = db.query("SELECT * FROM items")
    return {"items": items}
```

### Why `yield` and not just `return`?

```python
# This would NOT clean up if the endpoint crashes:
def get_db_bad():
    db = DatabaseSession()
    return db
    # If the endpoint raises an exception, who closes db? Nobody!

# yield + finally GUARANTEES cleanup:
def get_db_good():
    db = DatabaseSession()
    try:
        yield db
    finally:
        db.close()  # Runs no matter what!
```

---

## 🌐 Global Dependencies (Apply to All Routes)

```python
from fastapi import FastAPI, Depends, Header, HTTPException

# ─────────────────────────────────────────────
# A dependency that runs for EVERY request
# ─────────────────────────────────────────────
async def verify_token(x_token: str = Header(...)):
    if x_token != "expected-token":
        raise HTTPException(status_code=403, detail="Invalid token")

# Apply globally — every route must have a valid X-Token header
app = FastAPI(dependencies=[Depends(verify_token)])

@app.get("/items/")
def read_items():
    return {"items": ["item1", "item2"]}

@app.get("/users/")
def read_users():
    return {"users": ["user1", "user2"]}

# Both /items/ and /users/ require the X-Token header,
# even though neither endpoint explicitly uses Depends(verify_token).
```

You can also apply dependencies to a group of routes using `APIRouter`:

```python
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/admin",
    tags=["Admin"],
    dependencies=[Depends(verify_token)]  # All routes in this router require the token
)

@router.get("/dashboard")
def admin_dashboard():
    return {"page": "dashboard"}

@router.get("/settings")
def admin_settings():
    return {"page": "settings"}
```

---

## 🔬 Internals: How FastAPI Resolves Dependencies

```
When FastAPI starts up, for each endpoint it builds a "dependency tree":

@app.get("/items/")
def read_items(
    user: dict = Depends(get_current_user),
    db: Session = Depends(get_db),
    pagination: Pagination = Depends()
):
    ...

Dependency Tree:
┌─────────────────────────────────────────┐
│ read_items                              │
├─────────────────────────────────────────┤
│ ├── get_current_user                    │
│ │   └── verify_api_key                  │
│ │       └── get_api_key (Header)        │
│ ├── get_db (yield dependency)           │
│ └── Pagination (class dependency)       │
│     ├── skip (query param)              │
│     └── limit (query param)             │
└─────────────────────────────────────────┘

At request time, FastAPI:

1. TOPOLOGICAL SORT the dependency tree
   (dependencies without sub-dependencies are resolved first)

2. RESOLVE each dependency:
   - Functions → call and get return value
   - Classes → instantiate
   - Yield functions → call up to yield, save the generator

3. CACHE results within the request
   (if two endpoints depend on get_db, it's called only ONCE)

4. CALL the endpoint with all resolved dependencies

5. CLEANUP yield dependencies in reverse order
   (generators are resumed past yield for the finally block)

This is similar to how pytest fixtures work!
```

### Dependency Caching

```python
from fastapi import Depends

def get_config():
    print("Loading config...")  # This prints only ONCE per request
    return {"debug": True}

def dep_a(config: dict = Depends(get_config)):
    return {"a": True, **config}

def dep_b(config: dict = Depends(get_config)):
    return {"b": True, **config}

@app.get("/")
def root(a: dict = Depends(dep_a), b: dict = Depends(dep_b)):
    """
    Both dep_a and dep_b depend on get_config.
    FastAPI calls get_config ONCE and reuses the result.

    Output: "Loading config..." prints only 1 time, not 2.

    To disable caching (call the dependency every time):
    def dep_a(config: dict = Depends(get_config, use_cache=False)):
    """
    return {"a": a, "b": b}
```

---

## 🧩 Real-World Pattern: Database Session Dependency

This is the most common real-world use of `Depends()`:

```python
from fastapi import FastAPI, Depends
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

# Database setup
DATABASE_URL = "sqlite:///./app.db"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# The dependency
def get_db():
    """
    This pattern is used in almost every FastAPI project.

    1. Create a new database session for each request
    2. Yield it to the endpoint
    3. Close it after the endpoint finishes (even if it crashes)

    This ensures:
    - Each request gets its own isolated session
    - Sessions are never leaked (always closed)
    - Transactions can be properly committed or rolled back
    """
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

app = FastAPI()

@app.get("/users/")
def get_users(db: Session = Depends(get_db)):
    # 'db' is a SQLAlchemy session, ready to use
    users = db.query(User).all()
    return users

@app.post("/users/")
def create_user(name: str, db: Session = Depends(get_db)):
    user = User(name=name)
    db.add(user)
    db.commit()
    db.refresh(user)  # Reload from DB to get the auto-generated ID
    return user
```

> We'll explore this pattern in much more detail in **[06 — Database Integration](./06-database-integration.md)**.

---

## ➡️ Next Steps

Head to **[05 — Async and Concurrency](./05-async-and-concurrency.md)** to understand how `async`/`await` works in FastAPI, when to use it, and how FastAPI handles synchronous vs asynchronous endpoints internally.
