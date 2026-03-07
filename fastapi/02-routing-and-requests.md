# 02 — Routing and Requests

> **Goal**: Master how FastAPI handles incoming data — path parameters, query parameters, request bodies, headers, cookies, file uploads — and understand how it figures out where to get each value.

---

## 🧠 The Big Idea: FastAPI Reads Your Function Signature

This is the most important concept to understand. FastAPI **inspects your function's parameters and type hints** to decide:

1. **Where** to get each value (path? query? body? header?)
2. **What type** to convert it to
3. **Whether** it's required or optional
4. **How** to validate it

```python
@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    ...

# FastAPI sees:
#   item_id: int         → Name matches {item_id} in the path → PATH PARAMETER
#   q: str = None        → Not in the path, has a default → QUERY PARAMETER (optional)
```

**The decision tree FastAPI uses internally:**

```
For each function parameter:
│
├─ Is the parameter name in the URL path template?
│  └─ YES → It's a PATH PARAMETER
│
├─ Is the type a Pydantic model (BaseModel)?
│  └─ YES → It's a REQUEST BODY (from JSON)
│
├─ Is it annotated with Header(), Cookie(), etc.?
│  └─ YES → It comes from the corresponding HTTP header/cookie
│
├─ Is it annotated with File() or UploadFile?
│  └─ YES → It's a FILE UPLOAD
│
├─ Is it annotated with Form()?
│  └─ YES → It's FORM DATA
│
├─ Is it annotated with Depends()?
│  └─ YES → It's a DEPENDENCY (resolved by DI system)
│
└─ Otherwise → It's a QUERY PARAMETER
```

---

## 📍 Path Parameters

Path parameters are parts of the URL path that are variables.

```python
from fastapi import FastAPI

app = FastAPI()

# ─────────────────────────────────────────────
# Basic path parameter
# ─────────────────────────────────────────────
@app.get("/users/{user_id}")
def get_user(user_id: int):
    """
    {user_id} in the path becomes a function parameter.

    The ': int' type hint tells FastAPI to:
      1. Parse the URL segment as an integer
      2. Return 422 error if it can't be parsed as int
      3. Document it as type "integer" in the OpenAPI schema

    Examples:
      GET /users/42    → user_id = 42 (int)  ✓
      GET /users/abc   → 422 Validation Error ✗
      GET /users/3.14  → 422 Validation Error ✗
    """
    return {"user_id": user_id}


# ─────────────────────────────────────────────
# Multiple path parameters
# ─────────────────────────────────────────────
@app.get("/users/{user_id}/posts/{post_id}")
def get_user_post(user_id: int, post_id: int):
    """
    You can have multiple path parameters in a single route.
    Each {placeholder} must have a matching function parameter.
    """
    return {"user_id": user_id, "post_id": post_id}


# ─────────────────────────────────────────────
# Path parameter with predefined values (Enum)
# ─────────────────────────────────────────────
from enum import Enum

class ModelName(str, Enum):
    """
    By inheriting from both str and Enum, we get:
    - String behavior (for JSON serialization)
    - Enum validation (only these values are allowed)
    """
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
def get_model(model_name: ModelName):
    """
    model_name can ONLY be "alexnet", "resnet", or "lenet".
    Any other value → 422 error.
    The docs will show a dropdown with these three options.

    Examples:
      GET /models/alexnet  → ✓
      GET /models/vgg      → 422 Error ✗
    """
    return {"model": model_name, "value": model_name.value}
```

### ⚠️ Route Order Matters!

```python
# WRONG ORDER — /users/me will never match because {user_id} catches it first!
@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}

@app.get("/users/me")      # ← This will never be reached!
def get_current_user():
    return {"user": "current"}

# CORRECT ORDER — Static routes before dynamic routes
@app.get("/users/me")      # ← This is checked first
def get_current_user():
    return {"user": "current"}

@app.get("/users/{user_id}")  # ← This catches everything else
def get_user(user_id: int):
    return {"user_id": user_id}
```

**Why?** Starlette's router checks routes in the order they were registered. `"/users/{user_id}"` matches _any_ string, including "me". So put specific routes before generic ones.

---

## ❓ Query Parameters

Any function parameter that is **NOT** in the path template is automatically treated as a **query parameter**.

```python
from fastapi import FastAPI, Query
from typing import List, Optional

app = FastAPI()

# ─────────────────────────────────────────────
# Basic query parameters
# ─────────────────────────────────────────────
@app.get("/items/")
def list_items(skip: int = 0, limit: int = 10):
    """
    Query parameters appear after the '?' in a URL.

    Example: GET /items/?skip=20&limit=5

    - skip: int = 0   → Optional, defaults to 0
    - limit: int = 10  → Optional, defaults to 10
    """
    return {"skip": skip, "limit": limit}


# ─────────────────────────────────────────────
# Required query parameters (no default value)
# ─────────────────────────────────────────────
@app.get("/items/search")
def search_items(q: str):
    """
    Because 'q' has no default value, it's REQUIRED.
    If the client doesn't provide it, FastAPI returns 422.

    Example:
      GET /items/search?q=phone  → ✓
      GET /items/search          → 422 Error ✗ (q is required)
    """
    return {"query": q}


# ─────────────────────────────────────────────
# Optional query parameters
# ─────────────────────────────────────────────
@app.get("/items/filter")
def filter_items(
    q: str = None,          # Optional — None if not provided
    category: str = "all",  # Optional — defaults to "all"
    in_stock: bool = True   # Optional — defaults to True
):
    """
    Example: GET /items/filter?q=phone&in_stock=false

    FastAPI handles boolean conversion:
      ?in_stock=true  → True
      ?in_stock=false → False
      ?in_stock=1     → True
      ?in_stock=yes   → True
      ?in_stock=on    → True
    """
    return {"q": q, "category": category, "in_stock": in_stock}


# ─────────────────────────────────────────────
# Query parameter validation with Query()
# ─────────────────────────────────────────────
@app.get("/items/validated")
def validated_search(
    q: str = Query(
        default=None,           # Default value (None = optional)
        min_length=3,           # Minimum string length
        max_length=50,          # Maximum string length
        pattern="^[a-zA-Z]+$",  # Regex pattern (letters only)
        title="Search Query",   # Shows in docs
        description="Search query string, letters only, 3-50 chars",
        examples=["phone", "laptop"],  # Example values in docs
    )
):
    """
    Query() gives you fine-grained control over validation.

    Internally, FastAPI creates a Pydantic field from these constraints
    and validates the incoming value against it.
    """
    return {"q": q}


# ─────────────────────────────────────────────
# List query parameters (multiple values)
# ─────────────────────────────────────────────
@app.get("/items/multi")
def multi_query(
    tags: List[str] = Query(default=[])
):
    """
    Accept multiple values for the same parameter.

    Example: GET /items/multi?tags=python&tags=fastapi&tags=api

    Result: tags = ["python", "fastapi", "api"]
    """
    return {"tags": tags}
```

---

## 📦 Request Body (Pydantic Models)

For `POST`, `PUT`, and `PATCH` requests, you typically send data in the request body as JSON. FastAPI uses **Pydantic models** to define, validate, and document this data.

### What is Pydantic?

Pydantic is a data validation library that uses Python type hints. When you define a class that inherits from `BaseModel`, Pydantic:

1. **Validates** all fields against their type hints
2. **Converts** types when possible (e.g., string "42" → integer 42)
3. **Generates** a JSON Schema (for FastAPI's docs)
4. **Provides** helpful error messages

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field, field_validator
from typing import Optional, List
from datetime import datetime

app = FastAPI()

# ─────────────────────────────────────────────
# Basic Pydantic model
# ─────────────────────────────────────────────
class ItemCreate(BaseModel):
    """
    This model defines what data the client must send to create an item.

    Each field has:
      - A name (e.g., 'name')
      - A type hint (e.g., str)
      - Optionally, a default value or Field() with constraints
    """
    name: str                           # Required — must be a string
    description: Optional[str] = None   # Optional — can be None
    price: float                        # Required — must be a number
    tax: float = 0.0                    # Optional — defaults to 0.0
    tags: List[str] = []                # Optional — defaults to empty list


@app.post("/items/")
def create_item(item: ItemCreate):
    """
    How FastAPI processes this:
      1. Reads the request body (raw bytes)
      2. Parses it as JSON → Python dict
      3. Passes the dict to ItemCreate(**data)
      4. Pydantic validates every field
      5. If valid → 'item' is an ItemCreate instance
      6. If invalid → 422 response with error details
    """
    # Calculate total with tax
    total = item.price + item.tax

    # item.model_dump() converts the Pydantic model back to a dict
    item_dict = item.model_dump()
    item_dict["price_with_tax"] = total

    return item_dict

# Valid request body (all fields):
# {
#     "name": "Widget",
#     "description": "A useful widget",
#     "price": 29.99,
#     "tax": 2.50,
#     "tags": ["tool", "hardware"]
# }

# Valid request body (only required fields):
# {
#     "name": "Widget",
#     "price": 29.99
# }

# Invalid request body (missing required field):
# {
#     "description": "No name or price"
# }
# → 422 Error: name is required, price is required


# ─────────────────────────────────────────────
# Field validation with Field()
# ─────────────────────────────────────────────
class Product(BaseModel):
    name: str = Field(
        ...,                     # ... means REQUIRED (no default)
        min_length=1,
        max_length=100,
        examples=["Laptop"],
        description="The product name"
    )
    price: float = Field(
        ...,
        gt=0,                    # Greater than 0
        le=1_000_000,           # Less than or equal to 1,000,000
        description="Price must be positive"
    )
    quantity: int = Field(
        default=0,
        ge=0,                    # Greater than or equal to 0
        description="Stock quantity"
    )

    # Field constraints reference:
    #   gt  = greater than
    #   ge  = greater than or equal
    #   lt  = less than
    #   le  = less than or equal
    #   min_length / max_length = string length
    #   pattern = regex pattern


# ─────────────────────────────────────────────
# Custom validators
# ─────────────────────────────────────────────
class User(BaseModel):
    username: str
    email: str
    age: int

    # @field_validator runs custom validation logic on a specific field
    @field_validator("email")
    @classmethod
    def email_must_contain_at(cls, v: str) -> str:
        """
        'v' is the value of the 'email' field.
        This runs AFTER type validation (so we know v is a string).
        If validation fails, raise ValueError with a message.
        """
        if "@" not in v:
            raise ValueError("email must contain @")
        return v  # Always return the value (you can also transform it)

    @field_validator("age")
    @classmethod
    def age_must_be_valid(cls, v: int) -> int:
        if v < 0 or v > 150:
            raise ValueError("age must be between 0 and 150")
        return v


# ─────────────────────────────────────────────
# Nested models
# ─────────────────────────────────────────────
class Address(BaseModel):
    street: str
    city: str
    country: str
    zip_code: str

class Company(BaseModel):
    name: str
    address: Address          # Nested model — expects a JSON object within
    employees: List[str] = []

@app.post("/companies/")
def create_company(company: Company):
    """
    Expected JSON body:
    {
        "name": "Acme Corp",
        "address": {
            "street": "123 Main St",
            "city": "Springfield",
            "country": "US",
            "zip_code": "62701"
        },
        "employees": ["Alice", "Bob"]
    }

    Pydantic validates the ENTIRE nested structure recursively.
    """
    return company
```

### 🔬 How Pydantic Validation Works Internally

```
Input JSON → Python dict → Pydantic Model

Step 1: JSON bytes → json.loads() → Python dict
        '{"name": "X", "price": 10}' → {"name": "X", "price": 10}

Step 2: Pydantic.__init__(**dict) is called
        ItemCreate(name="X", price=10)

Step 3: For EACH field, pydantic-core (Rust) runs validation:
        name: str  → "X" is a string ✓
        price: float → 10 is numeric, coerce to 10.0 ✓
        description: Optional[str] = None → not provided, use None ✓
        tax: float = 0.0 → not provided, use 0.0 ✓
        tags: List[str] = [] → not provided, use [] ✓

Step 4: If ALL fields pass → create the model instance
        If ANY field fails → collect ALL errors → return 422

The 422 response looks like:
{
    "detail": [
        {
            "type": "missing",
            "loc": ["body", "name"],
            "msg": "Field required",
            "input": {...}
        }
    ]
}
```

---

## 🔀 Combining Path, Query, and Body

You can mix parameter types in a single endpoint:

```python
from fastapi import FastAPI, Query, Path, Body
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.put("/items/{item_id}")
def update_item(
    # PATH parameter (because it's in the URL path)
    item_id: int = Path(
        ...,                  # Required
        title="Item ID",
        ge=1                  # Must be >= 1
    ),
    # QUERY parameter (not in path, not a Pydantic model)
    q: str = None,
    # BODY parameter (Pydantic model → automatically from body)
    item: Item = None,
    # Singular BODY value (Body() forces a simple type to come from body)
    importance: int = Body(default=0)
):
    """
    Example request:
      PUT /items/42?q=updated

      Body:
      {
          "item": {"name": "Widget", "price": 29.99},
          "importance": 5
      }

    Notice: when you have multiple body parameters (item + importance),
    FastAPI expects them as keys in a JSON object.
    """
    result = {"item_id": item_id, "q": q}
    if item:
        result["item"] = item.model_dump()
    result["importance"] = importance
    return result
```

---

## 📨 Headers and Cookies

```python
from fastapi import FastAPI, Header, Cookie

app = FastAPI()

# ─────────────────────────────────────────────
# Reading headers
# ─────────────────────────────────────────────
@app.get("/headers/")
def read_headers(
    user_agent: str = Header(default=None),
    x_token: str = Header(default=None),
    accept_language: str = Header(default=None),
):
    """
    Header() tells FastAPI to read these from HTTP headers.

    Important: HTTP headers use hyphens (X-Token), but Python
    variables can't have hyphens. FastAPI automatically converts:
      x_token (Python underscore) → X-Token (HTTP header)

    Example HTTP headers:
      User-Agent: Mozilla/5.0
      X-Token: abc123
      Accept-Language: en-US
    """
    return {
        "User-Agent": user_agent,
        "X-Token": x_token,
        "Accept-Language": accept_language,
    }


# ─────────────────────────────────────────────
# Reading cookies
# ─────────────────────────────────────────────
@app.get("/cookies/")
def read_cookies(
    session_id: str = Cookie(default=None),
):
    """
    Cookie() reads from the Cookie header.

    If the request has: Cookie: session_id=abc123
    Then session_id = "abc123"
    """
    return {"session_id": session_id}
```

---

## 📁 File Uploads

```python
from fastapi import FastAPI, File, UploadFile
from typing import List

app = FastAPI()

# ─────────────────────────────────────────────
# Single file upload (simple)
# ─────────────────────────────────────────────
@app.post("/upload/bytes/")
async def upload_file_bytes(
    file: bytes = File(...)
):
    """
    File(...) reads the ENTIRE file into memory as bytes.

    ⚠️ Warning: This loads the whole file into RAM.
    Only use for small files. For large files, use UploadFile.
    """
    return {"file_size": len(file)}


# ─────────────────────────────────────────────
# Single file upload (recommended — UploadFile)
# ─────────────────────────────────────────────
@app.post("/upload/")
async def upload_file(
    file: UploadFile
):
    """
    UploadFile is the recommended way to handle file uploads.

    Advantages over bytes:
      - Uses a spooled temporary file (not all in RAM)
      - Provides file metadata (filename, content_type)
      - Has async methods for reading in chunks
      - Efficient for large files

    UploadFile attributes:
      file.filename     → Original filename ("photo.jpg")
      file.content_type → MIME type ("image/jpeg")
      file.size         → File size in bytes
      file.file         → The underlying SpooledTemporaryFile

    UploadFile methods:
      await file.read()    → Read all content
      await file.read(N)   → Read N bytes
      await file.seek(0)   → Go back to start of file
      await file.write(b)  → Write bytes to file
      await file.close()   → Close the file
    """
    contents = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents)
    }


# ─────────────────────────────────────────────
# Multiple file uploads
# ─────────────────────────────────────────────
@app.post("/upload/multiple/")
async def upload_multiple_files(
    files: List[UploadFile]
):
    """
    Accept multiple files in a single request.

    The client sends a multipart form with multiple 'files' fields.
    """
    result = []
    for file in files:
        contents = await file.read()
        result.append({
            "filename": file.filename,
            "size": len(contents)
        })
    return {"files": result}


# ─────────────────────────────────────────────
# File upload with additional form data
# ─────────────────────────────────────────────
from fastapi import Form

@app.post("/upload/with-form/")
async def upload_with_form(
    file: UploadFile,
    description: str = Form(...),  # Form data (not JSON!)
    category: str = Form(default="general"),
):
    """
    You can combine file uploads with form fields.

    ⚠️ Important: When using File/UploadFile, you CANNOT also have
    a JSON body. Use Form() for additional fields instead.

    This is because the request Content-Type becomes
    'multipart/form-data' (not 'application/json').
    """
    return {
        "filename": file.filename,
        "description": description,
        "category": category
    }
```

---

## 🔬 Internals: How FastAPI Resolves Parameters

Here's a deeper look at what FastAPI does when it processes your endpoint function:

```python
# When you write:
@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None, verbose: bool = False):
    return {"item_id": item_id}

# At startup, FastAPI does this (simplified):

# 1. INSPECT the function signature using Python's 'inspect' module
import inspect
sig = inspect.signature(read_item)
# sig.parameters = {
#   'item_id': Parameter(name='item_id', annotation=int, default=EMPTY),
#   'q':       Parameter(name='q', annotation=str, default=None),
#   'verbose': Parameter(name='verbose', annotation=bool, default=False),
# }

# 2. ANALYZE each parameter against the path template "/items/{item_id}"
# path_params = {"item_id"}

# 3. BUILD a dependency for each parameter:
# item_id → PathParam(name="item_id", type=int, required=True)
# q       → QueryParam(name="q", type=str, required=False, default=None)
# verbose → QueryParam(name="verbose", type=bool, required=False, default=False)

# 4. CREATE a Pydantic model from these parameters for validation:
# class RequestModel(BaseModel):
#     item_id: int
#     q: Optional[str] = None
#     verbose: bool = False

# 5. At request time, RESOLVE each parameter:
# item_id → scope["path_params"]["item_id"] → validate as int
# q → request.query_params.get("q") → validate as str or use None
# verbose → request.query_params.get("verbose") → coerce to bool

# 6. CALL the function with validated values:
# read_item(item_id=42, q=None, verbose=False)
```

---

## ➡️ Next Steps

Now that you can handle all types of incoming data, head to **[03 — Responses and Middleware](./03-responses-and-middleware.md)** to learn how to control what your API sends back — response models, status codes, error handling, and middleware.
