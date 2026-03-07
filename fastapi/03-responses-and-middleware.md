# 03 — Responses and Middleware

> **Goal**: Control what your API sends back — response models, status codes, custom responses, error handling, and the middleware chain.

---

## 📤 Response Models

Response models let you control **what data gets sent back** to the client. They filter out fields you don't want to expose (like passwords) and document the response structure in the auto-generated docs.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional, List

app = FastAPI()

# ─────────────────────────────────────────────
# Input model (what the client sends)
# ─────────────────────────────────────────────
class UserCreate(BaseModel):
    username: str
    email: str
    password: str  # Client sends this, but we should NEVER return it

# ─────────────────────────────────────────────
# Output model (what the API returns)
# ─────────────────────────────────────────────
class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    # Note: 'password' is NOT here — it gets filtered out

# Fake database for this example
fake_db = {}
next_id = 1

@app.post(
    "/users/",
    response_model=UserResponse,  # ← This is the key!
    status_code=201               # ← HTTP 201 Created
)
def create_user(user: UserCreate):
    """
    What 'response_model=UserResponse' does internally:

    1. Your function returns a dict/object with ALL fields (including password)
    2. FastAPI takes the returned data
    3. It passes it through UserResponse(**data), which:
       a. Keeps only fields defined in UserResponse (id, username, email)
       b. Removes any extra fields (password gets dropped!)
       c. Validates the output matches the response model
    4. The filtered, validated data is serialized to JSON

    This is called "response filtering" and it's a security feature.
    Even if you accidentally include sensitive data, the response model strips it.
    """
    global next_id
    user_data = {
        "id": next_id,
        "username": user.username,
        "email": user.email,
        "password": user.password,  # This exists in our "DB"
    }
    fake_db[next_id] = user_data
    next_id += 1

    # Even though we return the password here, response_model filters it out!
    return user_data

# Client receives only:
# {
#     "id": 1,
#     "username": "john",
#     "email": "john@example.com"
# }
# 🔒 password is NOT included!
```

### Response Model Configuration

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: float = 0.0

# ─────────────────────────────────────────────
# Exclude unset fields (don't include defaults)
# ─────────────────────────────────────────────
@app.get(
    "/items/{item_id}",
    response_model=Item,
    response_model_exclude_unset=True
    # Only include fields that were explicitly set.
    # If 'description' was never set (still None), it won't appear in response.
    # Useful for PATCH endpoints where you only return changed fields.
)
def get_item(item_id: int):
    return {"name": "Widget", "price": 29.99}
    # Response: {"name": "Widget", "price": 29.99}
    # 'description' and 'tax' are omitted because they weren't set


# ─────────────────────────────────────────────
# Include/exclude specific fields
# ─────────────────────────────────────────────
@app.get(
    "/items/{item_id}/summary",
    response_model=Item,
    response_model_include={"name", "price"},  # Only these fields
)
def get_item_summary(item_id: int):
    return {"name": "Widget", "description": "A widget", "price": 29.99, "tax": 2.0}
    # Response: {"name": "Widget", "price": 29.99}

@app.get(
    "/items/{item_id}/public",
    response_model=Item,
    response_model_exclude={"tax"},  # Exclude these fields
)
def get_item_public(item_id: int):
    return {"name": "Widget", "description": "A widget", "price": 29.99, "tax": 2.0}
    # Response: {"name": "Widget", "description": "A widget", "price": 29.99}
```

---

## 📊 Status Codes

HTTP status codes tell the client what happened. FastAPI lets you set them easily:

```python
from fastapi import FastAPI, status

app = FastAPI()

# Using the 'status' module for readable constants:
# status.HTTP_200_OK           = 200
# status.HTTP_201_CREATED      = 201
# status.HTTP_204_NO_CONTENT   = 204
# status.HTTP_400_BAD_REQUEST  = 400
# status.HTTP_404_NOT_FOUND    = 404
# status.HTTP_422_UNPROCESSABLE_ENTITY = 422

@app.post("/items/", status_code=status.HTTP_201_CREATED)
def create_item(name: str):
    """Returns 201 instead of default 200."""
    return {"name": name}

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: int):
    """Returns 204 with NO response body."""
    # When status is 204, FastAPI automatically returns no body
    return None
```

### Common HTTP Status Codes Reference

| Code  | Name                  | When to Use                                        |
| ----- | --------------------- | -------------------------------------------------- |
| `200` | OK                    | Successful GET, PUT, PATCH                         |
| `201` | Created               | Successful POST (new resource created)             |
| `204` | No Content            | Successful DELETE (nothing to return)              |
| `400` | Bad Request           | Client sent invalid data                           |
| `401` | Unauthorized          | Authentication required but not provided           |
| `403` | Forbidden             | Authenticated but not authorized                   |
| `404` | Not Found             | Resource doesn't exist                             |
| `422` | Unprocessable Entity  | Validation error (FastAPI's default for bad input) |
| `500` | Internal Server Error | Something broke on the server                      |

---

## 📬 Custom Responses

By default, FastAPI returns `JSONResponse`. You can use other response types:

```python
from fastapi import FastAPI
from fastapi.responses import (
    JSONResponse,       # Default — returns JSON
    HTMLResponse,       # Returns HTML
    PlainTextResponse,  # Returns plain text
    RedirectResponse,   # HTTP redirect
    StreamingResponse,  # Stream data (for large files/real-time)
    FileResponse,       # Serve a file from disk
)

app = FastAPI()

# ─────────────────────────────────────────────
# Custom JSON response with extra headers
# ─────────────────────────────────────────────
@app.get("/custom-json/")
def custom_json():
    """
    JSONResponse gives you full control over:
    - The content (dict → JSON)
    - Status code
    - HTTP headers
    """
    return JSONResponse(
        content={"message": "hello"},
        status_code=200,
        headers={"X-Custom-Header": "my-value"}
    )


# ─────────────────────────────────────────────
# HTML Response
# ─────────────────────────────────────────────
@app.get("/page/", response_class=HTMLResponse)
def get_page():
    """
    response_class=HTMLResponse tells FastAPI to:
    1. Set Content-Type to text/html
    2. Document the response as HTML in the OpenAPI schema
    """
    return """
    <html>
        <body>
            <h1>Hello from FastAPI!</h1>
            <p>This is an HTML response.</p>
        </body>
    </html>
    """


# ─────────────────────────────────────────────
# Redirect
# ─────────────────────────────────────────────
@app.get("/old-page")
def redirect_old_page():
    """Redirects the client to a different URL."""
    return RedirectResponse(url="/page/")


# ─────────────────────────────────────────────
# Streaming Response (for large data)
# ─────────────────────────────────────────────
import asyncio

@app.get("/stream/")
async def stream_data():
    """
    StreamingResponse sends data in chunks.
    Useful for: large files, real-time data, server-sent events.

    The client receives data as it's generated, without waiting
    for the entire response to be ready.
    """
    async def generate():
        for i in range(10):
            yield f"data: chunk {i}\n\n"
            await asyncio.sleep(0.5)  # Simulate slow data generation

    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )


# ─────────────────────────────────────────────
# File Response
# ─────────────────────────────────────────────
@app.get("/download/")
def download_file():
    """Serve a file from the filesystem."""
    return FileResponse(
        path="/path/to/file.pdf",
        filename="report.pdf",           # Name the browser sees
        media_type="application/pdf"
    )
```

---

## ❌ Error Handling

### HTTPException — The Standard Way

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

items_db = {"1": "Laptop", "2": "Phone"}

@app.get("/items/{item_id}")
def get_item(item_id: str):
    """
    HTTPException is how you return error responses in FastAPI.

    When you raise an HTTPException:
    1. FastAPI catches it (it doesn't crash your app)
    2. It creates a JSON response with the status code and detail message
    3. The response is sent to the client
    """
    if item_id not in items_db:
        # This raises an HTTP 404 error
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Item with id '{item_id}' not found",
            # You can also add custom headers:
            headers={"X-Error-Code": "ITEM_NOT_FOUND"}
        )
    return {"item_id": item_id, "name": items_db[item_id]}

# When item is missing, client receives:
# HTTP/1.1 404 Not Found
# Content-Type: application/json
# X-Error-Code: ITEM_NOT_FOUND
#
# {"detail": "Item with id '999' not found"}
```

### Custom Exception Handlers

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# ─────────────────────────────────────────────
# Define a custom exception class
# ─────────────────────────────────────────────
class ItemNotFoundException(Exception):
    """Custom exception for when an item is not found."""
    def __init__(self, item_id: str):
        self.item_id = item_id

# ─────────────────────────────────────────────
# Register a handler for this exception
# ─────────────────────────────────────────────
@app.exception_handler(ItemNotFoundException)
async def item_not_found_handler(request: Request, exc: ItemNotFoundException):
    """
    This function is called whenever ItemNotFoundException is raised
    ANYWHERE in your app. It converts the exception into a proper HTTP response.

    How it works internally:
    1. Your endpoint raises ItemNotFoundException("42")
    2. FastAPI catches it (because of this handler)
    3. FastAPI calls this function with the request and the exception
    4. You return a proper Response object
    """
    return JSONResponse(
        status_code=404,
        content={
            "error": "item_not_found",
            "message": f"Item '{exc.item_id}' does not exist",
            "documentation": "https://api.example.com/docs/errors"
        }
    )

@app.get("/items/{item_id}")
def get_item(item_id: str):
    items = {"1": "Laptop"}
    if item_id not in items:
        raise ItemNotFoundException(item_id)  # Our custom exception
    return {"item": items[item_id]}


# ─────────────────────────────────────────────
# Override the default validation error handler
# ─────────────────────────────────────────────
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    """
    Override FastAPI's default 422 validation error response.
    Useful for customizing the error format.
    """
    return JSONResponse(
        status_code=422,
        content={
            "error": "validation_failed",
            "details": [
                {
                    "field": ".".join(str(loc) for loc in err["loc"]),
                    "message": err["msg"],
                    "type": err["type"]
                }
                for err in exc.errors()
            ]
        }
    )
```

---

## 🔗 Middleware

Middleware is code that runs **before and after every request**. Think of it as a wrapper around your entire application.

```
Request → Middleware 1 → Middleware 2 → Router → Your Function
                                                      ↓
Response ← Middleware 1 ← Middleware 2 ← Router ← Your Function
```

### Writing Custom Middleware

```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

# ─────────────────────────────────────────────
# Method 1: Using @app.middleware decorator
# ─────────────────────────────────────────────
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    """
    This middleware measures how long each request takes to process.

    Parameters:
        request     → The incoming HTTP request
        call_next   → A function that passes the request to the next middleware
                       or to your endpoint function

    How it works:
    1. Code BEFORE 'await call_next(request)' runs BEFORE the endpoint
    2. 'await call_next(request)' calls the actual endpoint (or next middleware)
    3. Code AFTER 'await call_next(request)' runs AFTER the endpoint
    """
    # ── BEFORE the request is processed ──
    start_time = time.perf_counter()

    # ── Process the request (calls your endpoint) ──
    response = await call_next(request)

    # ── AFTER the request is processed ──
    process_time = time.perf_counter() - start_time
    response.headers["X-Process-Time"] = str(round(process_time, 4))

    return response


# ─────────────────────────────────────────────
# Method 2: Using a class (BaseHTTPMiddleware)
# ─────────────────────────────────────────────
from starlette.middleware.base import BaseHTTPMiddleware

class LoggingMiddleware(BaseHTTPMiddleware):
    """
    Class-based middleware. Gives you more structure for complex middleware.
    """
    async def dispatch(self, request: Request, call_next):
        # Log the incoming request
        print(f"📥 {request.method} {request.url.path}")

        # Process the request
        response = await call_next(request)

        # Log the response
        print(f"📤 {request.method} {request.url.path} → {response.status_code}")

        return response

# Register the class-based middleware
app.add_middleware(LoggingMiddleware)


# ─────────────────────────────────────────────
# Example endpoint
# ─────────────────────────────────────────────
@app.get("/")
def root():
    return {"message": "Hello!"}

# When you hit GET /:
# Console output:
#   📥 GET /
#   📤 GET / → 200
# Response headers include:
#   X-Process-Time: 0.0002
```

### CORS Middleware (Cross-Origin Resource Sharing)

CORS is critical when your frontend (e.g., React running on `localhost:3000`) calls your API (running on `localhost:8000`). Browsers block cross-origin requests by default.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# List of allowed origins (frontend URLs)
origins = [
    "http://localhost:3000",       # React dev server
    "http://localhost:5173",       # Vite dev server
    "https://myapp.example.com",   # Production frontend
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,         # Which origins can make requests
    # allow_origins=["*"]          # Allow ALL origins (not recommended for production)
    allow_credentials=True,        # Allow cookies to be included
    allow_methods=["*"],           # Allow all HTTP methods (GET, POST, etc.)
    allow_headers=["*"],           # Allow all request headers
)

# How CORS works internally:
#
# 1. Browser sends a "preflight" request:
#    OPTIONS /api/items HTTP/1.1
#    Origin: http://localhost:3000
#    Access-Control-Request-Method: POST
#
# 2. CORSMiddleware checks if the origin is in allow_origins
#
# 3. If allowed, it responds with:
#    Access-Control-Allow-Origin: http://localhost:3000
#    Access-Control-Allow-Methods: POST
#    Access-Control-Allow-Headers: Content-Type
#
# 4. Browser sees the "all clear" and sends the actual request
#
# 5. On the actual response, CORSMiddleware adds:
#    Access-Control-Allow-Origin: http://localhost:3000

@app.get("/api/data")
def get_data():
    return {"data": "accessible from allowed origins"}
```

### Other Built-in Middleware

```python
from fastapi import FastAPI
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()

# Reject requests from unexpected Host headers (prevents host header attacks)
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["example.com", "*.example.com"]
)

# Compress responses larger than 500 bytes using gzip
app.add_middleware(GZipMiddleware, minimum_size=500)

# Redirect all HTTP requests to HTTPS
app.add_middleware(HTTPSRedirectMiddleware)
```

---

## 🔬 Internals: Middleware Chain Execution

```
When a request comes in, middleware runs in the ORDER it was added.
But responses travel BACKWARDS through the same chain.

# If you have:
app.add_middleware(A)
app.add_middleware(B)
app.add_middleware(C)

# The execution order is:
#   Request:  C.before → B.before → A.before → Your Endpoint
#   Response: A.after  → B.after  → C.after  → Client

# Why reversed? Because add_middleware wraps the existing app:
#   After adding A: app = A(original_app)
#   After adding B: app = B(A(original_app))
#   After adding C: app = C(B(A(original_app)))
#
# So the last added middleware (C) is the OUTERMOST wrapper.
# When a request comes in, it hits C first.

# Think of it like nested boxes:
#   ┌─── C ────────────────────────┐
#   │  ┌─── B ──────────────────┐  │
#   │  │  ┌─── A ────────────┐  │  │
#   │  │  │  Your Endpoint   │  │  │
#   │  │  └──────────────────┘  │  │
#   │  └────────────────────────┘  │
#   └──────────────────────────────┘
#   Request enters from outside → goes inward
#   Response goes outward → exits to client
```

---

## ➡️ Next Steps

Now head to **[04 — Dependency Injection](./04-dependency-injection.md)** to learn about FastAPI's powerful `Depends()` system — the backbone for managing database sessions, authentication, shared logic, and more.
