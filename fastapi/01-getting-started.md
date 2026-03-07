# 01 — Getting Started with FastAPI

> **Goal**: Install FastAPI, understand every piece of a basic app, and learn how a request flows through the system internally.

---

## 📦 Installation

```bash
# Create a project directory
mkdir my-fastapi-app && cd my-fastapi-app

# (Recommended) Create a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install FastAPI with all optional dependencies
# "fastapi[standard]" includes:
#   - uvicorn (ASGI server)
#   - pydantic (data validation)
#   - starlette (web framework layer)
#   - python-multipart (for form/file uploads)
#   - httpx (for async testing)
#   - jinja2 (for template rendering)
pip install "fastapi[standard]"

# Alternatively, install just the essentials
pip install fastapi uvicorn
```

### What Gets Installed (The Dependency Tree)

```
fastapi
├── starlette          → Web framework layer (routing, middleware, requests)
│   └── anyio          → Async compatibility layer (works with asyncio & trio)
├── pydantic           → Data validation using Python type hints
│   └── pydantic-core  → Rust-based validation engine (this is why it's fast)
└── typing-extensions  → Backport of newer typing features

uvicorn                → ASGI server (runs your app)
├── httptools          → Fast HTTP parser (written in C)
├── websockets         → WebSocket support
└── uvloop             → Fast event loop (replaces asyncio's default loop on Linux/Mac)
```

---

## 🏗️ Anatomy of a FastAPI Application

Let's build a slightly more complete app and understand every single line:

```python
# main.py

# ──────────────────────────────────────────────
# IMPORTS
# ──────────────────────────────────────────────

from fastapi import FastAPI       # The main application class
from pydantic import BaseModel    # Base class for request/response data models

# ──────────────────────────────────────────────
# APP INSTANCE
# ──────────────────────────────────────────────

# FastAPI() creates your application.
# Think of it as the "control center" that manages everything.
#
# Parameters you can pass:
#   title       → Name shown in the docs UI
#   description → Description in the docs
#   version     → API version string
#   docs_url    → URL for Swagger docs (default: "/docs")
#   redoc_url   → URL for ReDoc (default: "/redoc")
app = FastAPI(
    title="My Learning API",
    description="An API built while learning FastAPI",
    version="1.0.0"
)

# ──────────────────────────────────────────────
# DATA MODELS (Pydantic)
# ──────────────────────────────────────────────

# Pydantic models define the SHAPE of your data.
# When you use a Pydantic model as a function parameter,
# FastAPI will:
#   1. Read the request body as JSON
#   2. Validate the data against this model
#   3. Convert types if possible (e.g., "123" → 123)
#   4. Return a 422 error if validation fails
#   5. Generate documentation from the model's fields
class Item(BaseModel):
    name: str               # Required field, must be a string
    price: float            # Required field, must be a number
    is_offer: bool = False  # Optional field with a default value

# ──────────────────────────────────────────────
# ROUTES (Endpoints)
# ──────────────────────────────────────────────

# GET request to the root URL
@app.get("/")
def read_root():
    """
    This docstring appears in the auto-generated documentation!
    FastAPI reads it and uses it as the description for this endpoint.
    """
    return {"message": "Welcome to my API"}


# GET request with a PATH PARAMETER
# {item_id} in the URL becomes a function parameter
@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    """
    Get an item by its ID.

    - **item_id**: The ID of the item (from the URL path)
    - **q**: An optional query parameter

    Example: GET /items/42?q=searchterm
    """
    # item_id is automatically parsed as an integer because of the type hint `: int`
    # If someone sends GET /items/abc, FastAPI returns a 422 error automatically
    #
    # q is a QUERY PARAMETER because it's not in the path template
    # It's optional because it has a default value of None
    return {"item_id": item_id, "query": q}


# POST request with a REQUEST BODY
@app.post("/items/")
def create_item(item: Item):
    """
    Create a new item.

    FastAPI knows 'item' is a request body because:
    1. It's not in the URL path
    2. Its type (Item) is a Pydantic model
    3. Therefore, it must come from the request body as JSON
    """
    # 'item' is now a validated Pydantic model object
    # You can access its fields like regular Python attributes
    return {
        "item_name": item.name,
        "item_price": item.price,
        "is_offer": item.is_offer
    }
```

### Running This App

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

**Flag explanations:**

| Flag             | Meaning                                                    |
| ---------------- | ---------------------------------------------------------- |
| `main:app`       | File `main.py`, object `app`                               |
| `--reload`       | Watch files and auto-restart on changes (development only) |
| `--host 0.0.0.0` | Listen on all network interfaces (not just localhost)      |
| `--port 8000`    | Listen on port 8000 (default)                              |

---

## 🔬 Internals: How a Request Flows Through the System

When someone sends `GET /items/42?q=hello` to your FastAPI app, here's **exactly** what happens, step by step:

```
Client sends: GET /items/42?q=hello HTTP/1.1


Step 1: UVICORN (The ASGI Server)
─────────────────────────────────
  → Uvicorn's event loop picks up the incoming TCP connection
  → httptools (C-based parser) parses the raw HTTP bytes into:
      method: "GET"
      path: "/items/42"
      query_string: "q=hello"
      headers: {...}
  → Uvicorn creates an ASGI 'scope' dictionary:
      {
        "type": "http",
        "method": "GET",
        "path": "/items/42",
        "query_string": b"q=hello",
        "headers": [...],
        ...
      }
  → Uvicorn calls: await app(scope, receive, send)


Step 2: STARLETTE (The Web Framework Layer)
───────────────────────────────────────────
  → FastAPI inherits from Starlette, so app() is actually
    Starlette.__call__(scope, receive, send)

  → The middleware stack runs first (if any middleware is registered):
      Request → Middleware 1 → Middleware 2 → ... → Router

  → Starlette's Router matches the path "/items/42" against registered routes:
      "/items/{item_id}" ← MATCH!
      It extracts: path_params = {"item_id": "42"}


Step 3: FASTAPI (The Smart Layer)
─────────────────────────────────
  → FastAPI's route handler kicks in. It does:

  a) PARAMETER RESOLUTION
     FastAPI inspects the function signature:
       def read_item(item_id: int, q: str = None)
     It figures out:
       - item_id: int → comes from path params → "42" → int(42) ✓
       - q: str = None → comes from query params → "hello" ✓

  b) VALIDATION (via Pydantic)
     Every parameter is validated against its type hint.
     If item_id was "abc", Pydantic would reject it → 422 response.

  c) DEPENDENCY RESOLUTION
     If the function has Depends() parameters, they are resolved here.
     (We'll cover this in chapter 04.)

  d) FUNCTION CALL
     FastAPI calls: read_item(item_id=42, q="hello")

  e) RESPONSE SERIALIZATION
     The returned dict {"item_id": 42, "query": "hello"} is:
       → Serialized to JSON using Python's json module
       → Wrapped in a Starlette JSONResponse
       → Content-Type header is set to "application/json"


Step 4: RESPONSE TRAVELS BACK
──────────────────────────────
  → The JSONResponse is sent back through the middleware stack
    (middleware can modify the response too)
  → Starlette calls send() to write the response bytes
  → Uvicorn sends the HTTP response back to the client


Client receives:
  HTTP/1.1 200 OK
  Content-Type: application/json

  {"item_id": 42, "query": "hello"}
```

---

## 📚 Auto-Generated Documentation (OpenAPI)

One of FastAPI's killer features is **automatic API documentation**. Here's how it works internally:

### How It Works Under the Hood

```python
# When you define a route like this:
@app.get("/items/{item_id}", response_model=dict)
def read_item(item_id: int, q: str = None):
    """Get an item by ID."""
    return {"item_id": item_id, "query": q}

# FastAPI internally builds an OpenAPI schema object.
# You can see the raw schema at: http://127.0.0.1:8000/openapi.json
#
# For the route above, it generates something like:
# {
#   "/items/{item_id}": {
#     "get": {
#       "summary": "Read Item",
#       "description": "Get an item by ID.",
#       "parameters": [
#         {
#           "name": "item_id",
#           "in": "path",
#           "required": true,
#           "schema": {"type": "integer"}
#         },
#         {
#           "name": "q",
#           "in": "query",
#           "required": false,
#           "schema": {"type": "string"}
#         }
#       ]
#     }
#   }
# }
```

### The Three Documentation URLs

```python
# You can customize or disable docs:
app = FastAPI(
    docs_url="/docs",        # Swagger UI (default: "/docs", set None to disable)
    redoc_url="/redoc",      # ReDoc (default: "/redoc", set None to disable)
    openapi_url="/openapi.json"  # Raw OpenAPI schema (default: "/openapi.json")
)
```

| URL             | UI         | Best For                                                               |
| --------------- | ---------- | ---------------------------------------------------------------------- |
| `/docs`         | Swagger UI | **Interactive testing** — you can send requests right from the browser |
| `/redoc`        | ReDoc      | **Reading docs** — cleaner, more readable layout                       |
| `/openapi.json` | Raw JSON   | **Code generation** — tools can read this to generate client SDKs      |

---

## 🔀 HTTP Methods (CRUD Operations)

FastAPI provides decorators for all standard HTTP methods:

```python
from fastapi import FastAPI

app = FastAPI()

# CREATE — POST
# Used when creating new resources
# The request body contains the data for the new resource
@app.post("/items/")
def create_item():
    return {"action": "created"}

# READ — GET
# Used when retrieving data
# Should NEVER modify data on the server
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"action": "read", "item_id": item_id}

# UPDATE (Full) — PUT
# Used when replacing an entire resource
# Client sends ALL fields, even unchanged ones
@app.put("/items/{item_id}")
def update_item(item_id: int):
    return {"action": "full update", "item_id": item_id}

# UPDATE (Partial) — PATCH
# Used when updating specific fields only
# Client sends ONLY the fields that changed
@app.patch("/items/{item_id}")
def partial_update_item(item_id: int):
    return {"action": "partial update", "item_id": item_id}

# DELETE — DELETE
# Used when removing a resource
@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    return {"action": "deleted", "item_id": item_id}
```

### How Decorators Work (The `@` syntax)

If you're new to Python decorators, here's what's happening:

```python
# This decorator syntax:
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}

# Is exactly equivalent to:
def read_item(item_id: int):
    return {"item_id": item_id}

read_item = app.get("/items/{item_id}")(read_item)

# What app.get() does internally:
# 1. Creates a Route object with path="/items/{item_id}" and method="GET"
# 2. Stores a reference to your read_item function in that Route
# 3. Adds the Route to Starlette's Router
# 4. Returns the original function (so read_item is still callable)
```

---

## 🏃 Startup and Shutdown Events (Lifespan)

Sometimes you need to run code when the app starts (e.g., connect to a database) or stops (e.g., close connections). FastAPI uses a **lifespan** context manager for this:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

# This is the modern way (FastAPI 0.93+)
# The @asynccontextmanager decorator turns this into an async context manager
@asynccontextmanager
async def lifespan(app: FastAPI):
    # ──── STARTUP CODE ────
    # Everything BEFORE 'yield' runs when the app starts
    print("🚀 App is starting up...")
    # Example: connect to database, load ML models, etc.

    yield  # The app runs and handles requests here

    # ──── SHUTDOWN CODE ────
    # Everything AFTER 'yield' runs when the app shuts down
    print("🛑 App is shutting down...")
    # Example: close DB connections, cleanup resources, etc.

# Pass the lifespan to your app
app = FastAPI(lifespan=lifespan)

@app.get("/")
def read_root():
    return {"status": "running"}
```

**Why `yield`?**
The `yield` keyword pauses the function. Everything before it is "startup", everything after is "shutdown". The app "lives" during the `yield`. This pattern is called a **context manager** — the same concept as `with open("file.txt") as f:`.

---

## 📁 Project Structure (Best Practices)

For small projects, a single `main.py` is fine. As your project grows, organize it like this:

```
my-fastapi-app/
├── app/
│   ├── __init__.py          # Makes 'app' a Python package
│   ├── main.py              # FastAPI app instance + lifespan
│   ├── config.py            # Settings and configuration
│   ├── models/              # Pydantic models (request/response schemas)
│   │   ├── __init__.py
│   │   └── item.py
│   ├── routers/             # Route definitions (endpoints)
│   │   ├── __init__.py
│   │   ├── items.py
│   │   └── users.py
│   ├── services/            # Business logic
│   │   ├── __init__.py
│   │   └── item_service.py
│   ├── db/                  # Database setup
│   │   ├── __init__.py
│   │   ├── database.py
│   │   └── models.py        # SQLAlchemy models (DB tables)
│   └── dependencies.py      # Shared dependencies (Depends)
├── tests/
│   ├── __init__.py
│   └── test_items.py
├── requirements.txt
└── .env
```

### Using APIRouter (Splitting Routes into Files)

```python
# app/routers/items.py

from fastapi import APIRouter

# APIRouter works exactly like FastAPI() but for grouping routes.
# Think of it as a "mini-app" that you plug into the main app.
router = APIRouter(
    prefix="/items",    # All routes in this file start with /items
    tags=["Items"],     # Group these routes under "Items" in the docs
)

@router.get("/")
def list_items():
    return [{"name": "Item 1"}, {"name": "Item 2"}]

@router.get("/{item_id}")
def get_item(item_id: int):
    return {"item_id": item_id, "name": "Item"}

@router.post("/")
def create_item():
    return {"name": "New Item"}
```

```python
# app/main.py

from fastapi import FastAPI
from app.routers import items  # Import the router module

app = FastAPI()

# "Include" the router — this adds all routes from items.py to the main app
# After this, the routes are:
#   GET  /items/
#   GET  /items/{item_id}
#   POST /items/
app.include_router(items.router)

@app.get("/")
def read_root():
    return {"message": "Welcome! Check /items/ for items."}
```

**How `include_router` works internally:**

1. It iterates over all routes registered in the `APIRouter`
2. For each route, it prepends the prefix (`/items`) to the path
3. It adds the route to the main app's Starlette router
4. Tags are merged with any existing tags for documentation grouping

---

## ➡️ Next Steps

Now that you understand the basics, head to **[02 — Routing and Requests](./02-routing-and-requests.md)** to learn about path parameters, query parameters, request bodies, Pydantic validation, and how FastAPI magically knows where each parameter comes from.
