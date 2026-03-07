# 🚀 FastAPI — The Complete Guide (Zero to Production)

> **FastAPI** is a modern, high-performance Python web framework for building APIs. It's built on top of **Starlette** (for the web layer) and **Pydantic** (for data validation). It's one of the fastest Python frameworks available — on par with Node.js and Go.

---

## 📖 What You'll Learn

This documentation suite takes you from **absolute zero** to building production-ready APIs with FastAPI. Every code snippet is explained line by line.

---

## 🗂️ Table of Contents

| #   | File                                                                         | What You'll Learn                                        |
| --- | ---------------------------------------------------------------------------- | -------------------------------------------------------- |
| 0   | **[README.md](./README.md)** ← you are here                                  | What FastAPI is, ASGI internals, Hello World             |
| 1   | **[01-getting-started.md](./01-getting-started.md)**                         | Installation, first app, request lifecycle, auto-docs    |
| 2   | **[02-routing-and-requests.md](./02-routing-and-requests.md)**               | Path/query params, Pydantic, request body, file uploads  |
| 3   | **[03-responses-and-middleware.md](./03-responses-and-middleware.md)**       | Responses, error handling, middleware, CORS              |
| 4   | **[04-dependency-injection.md](./04-dependency-injection.md)**               | `Depends()`, nested deps, scopes, DB session deps        |
| 5   | **[05-async-and-concurrency.md](./05-async-and-concurrency.md)**             | async/await, event loop, background tasks, sync vs async |
| 6   | **[06-database-integration.md](./06-database-integration.md)**               | SQLAlchemy, async DB, Alembic, full CRUD                 |
| 7   | **[07-authentication-and-security.md](./07-authentication-and-security.md)** | OAuth2, JWT, password hashing, protected routes          |
| 8   | **[08-testing-and-deployment.md](./08-testing-and-deployment.md)**           | pytest, TestClient, Docker, Uvicorn, production          |

---

## 🧠 Before We Start: Key Concepts

### What Problem Does FastAPI Solve?

Traditional Python web frameworks (Flask, Django) were built on **WSGI** — a synchronous protocol. This means each request blocks a thread until it finishes. If your API calls a slow database or external service, the thread just _waits_, doing nothing.

FastAPI is built on **ASGI** (Asynchronous Server Gateway Interface), which supports `async`/`await`. This means your server can handle _thousands_ of concurrent connections without blocking.

### The Architecture Stack

```
┌─────────────────────────────────────────┐
│              YOUR CODE                  │  ← You write endpoint functions
├─────────────────────────────────────────┤
│              FastAPI                    │  ← Adds validation, docs, DI
├─────────────────────────────────────────┤
│             Starlette                   │  ← Handles routing, middleware, requests/responses
├─────────────────────────────────────────┤
│              ASGI                       │  ← The protocol (async interface)
├─────────────────────────────────────────┤
│        Uvicorn (ASGI Server)            │  ← Actually receives HTTP connections
└─────────────────────────────────────────┘
```

**Let's break this down:**

| Layer         | What It Does                                                                   | Analogy                                                       |
| ------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------- |
| **Uvicorn**   | Listens on a port, receives raw HTTP bytes, converts them into ASGI events     | The **mailroom** — receives all incoming mail                 |
| **ASGI**      | A standard interface — a Python callable that takes `scope`, `receive`, `send` | The **envelope format** everyone agrees on                    |
| **Starlette** | Provides routing, middleware, request/response classes                         | The **sorting machine** — routes mail to the right desk       |
| **FastAPI**   | Adds type validation (Pydantic), automatic docs, dependency injection          | The **smart assistant** — validates, documents, and organizes |
| **Your Code** | The actual business logic                                                      | The **person** who reads the mail and writes a reply          |

### WSGI vs ASGI — Why It Matters

```python
# WSGI (Flask-style) — Synchronous
# Each request BLOCKS a worker thread
def wsgi_app(environ, start_response):
    # This function runs start-to-finish on ONE thread
    # If it waits for a DB query, the thread is STUCK
    status = '200 OK'
    response_headers = [('Content-Type', 'text/plain')]
    start_response(status, response_headers)
    return [b'Hello World']

# ASGI (FastAPI-style) — Asynchronous
# The event loop can switch to other tasks while waiting
async def asgi_app(scope, receive, send):
    # 'scope' contains connection info (like environ in WSGI)
    # 'receive' is an async callable to get request body
    # 'send' is an async callable to send response
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [[b'content-type', b'text/plain']],
    })
    await send({
        'type': 'http.response.body',
        'body': b'Hello World',
    })
```

> **Key insight**: You never write raw ASGI code. FastAPI and Starlette handle all of this. But understanding that this is what happens under the hood helps you debug and make better design decisions.

---

## ⚡ Your First FastAPI App (Hello World)

```python
# main.py

# 1. Import the FastAPI class
#    FastAPI is a Python class that provides all the functionality for your API.
#    Under the hood, it inherits from Starlette's application class.
from fastapi import FastAPI

# 2. Create an instance of FastAPI
#    This 'app' object is the main point of interaction for creating your API.
#    All your routes, middleware, and event handlers are registered on this object.
app = FastAPI()

# 3. Define a route using a "decorator"
#    @app.get("/") means: "When someone sends a GET request to the root URL (/),
#    run the function below."
#
#    The decorator registers this function in Starlette's routing table.
#    FastAPI also inspects the function's type hints and return type to:
#      - Validate input automatically
#      - Generate OpenAPI documentation automatically
@app.get("/")
def read_root():
    # 4. Return a Python dictionary
    #    FastAPI automatically converts this dict to a JSON response.
    #    It sets Content-Type: application/json and serializes using json.dumps().
    return {"message": "Hello, World!"}
```

### Running It

```bash
# Install FastAPI and Uvicorn
pip install fastapi uvicorn

# Run the app
# - "main:app" means: in the file "main.py", use the object named "app"
# - "--reload" means: auto-restart the server when you save code changes (dev only!)
uvicorn main:app --reload
```

**What happens when you run this command:**

```
1. Uvicorn starts and binds to http://127.0.0.1:8000
2. It creates an asyncio event loop
3. It loads your 'app' object from main.py
4. It starts listening for HTTP connections
5. When a request arrives at GET /, it:
   a. Creates an ASGI 'scope' dict with request info
   b. Starlette's router matches "/" to your read_root function
   c. FastAPI calls read_root()
   d. The returned dict {"message": "Hello, World!"} is serialized to JSON
   e. The JSON bytes are sent back through ASGI → Uvicorn → Client
```

### Visit the Auto-Generated Docs

Once the server is running, open your browser:

| URL                           | What You Get                                   |
| ----------------------------- | ---------------------------------------------- |
| `http://127.0.0.1:8000`       | Your API response (the JSON)                   |
| `http://127.0.0.1:8000/docs`  | **Swagger UI** — Interactive API documentation |
| `http://127.0.0.1:8000/redoc` | **ReDoc** — Alternative documentation style    |

> These docs are **automatically generated** from your code's type hints and docstrings. You don't write any documentation manually — FastAPI reads your Python code and creates the OpenAPI schema.

---

## 🔑 Key Features at a Glance

| Feature                  | How It Works                                                             |
| ------------------------ | ------------------------------------------------------------------------ |
| **Type Validation**      | Uses Python type hints + Pydantic to validate request data automatically |
| **Auto Documentation**   | Generates OpenAPI (Swagger) docs from your code — zero config            |
| **Async Support**        | Native `async`/`await` — handles thousands of concurrent requests        |
| **Dependency Injection** | Built-in DI system via `Depends()` — manage DB sessions, auth, etc.      |
| **High Performance**     | Comparable to Node.js/Go thanks to Starlette + Uvicorn                   |
| **Standards-Based**      | Built on OpenAPI and JSON Schema standards                               |

---

## ➡️ Next Steps

Start with **[01 — Getting Started](./01-getting-started.md)** to dive deeper into installation, app structure, and request lifecycle internals.
