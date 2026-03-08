# FastAPI Internals: How FastAPI Works Under the Hood

This document explains the internal architecture of FastAPI and how a request travels through the system. The goal is to understand FastAPI from a **low-level perspective**, including how the ASGI server, event loop, Starlette, and FastAPI layers interact.

---

# Architecture Overview

FastAPI is not a standalone web server. It is a layer built on top of other components. The full request pipeline looks like this:

```
Client
  ↓
Operating System (TCP Socket)
  ↓
Uvicorn (ASGI Server)
  ↓
ASGI Interface
  ↓
Starlette (Web Framework Core)
  ↓
FastAPI (Validation, Dependency Injection, Documentation)
  ↓
Application Endpoint
  ↓
Response Serialization
  ↓
Back through Starlette → Uvicorn → Client
```

Each layer has a well-defined responsibility.

---

# 1. Operating System and TCP Layer

When a client sends a request:

```
GET /items/42?q=hello HTTP/1.1
```

The request first arrives at the operating system networking stack.

Responsibilities at this layer:

* Accept TCP connections
* Handle socket buffers
* Deliver raw HTTP bytes to the server process

At this stage the request is simply a stream of bytes.

---

# 2. Uvicorn (ASGI Server)

Uvicorn is responsible for running the web application. It performs several critical tasks:

* Managing the asynchronous event loop
* Accepting incoming connections
* Parsing HTTP requests
* Creating ASGI messages
* Sending responses back to clients

Uvicorn typically uses the following libraries:

* **uvloop** – a high-performance event loop
* **httptools** – a fast HTTP parser written in C
* **websockets** – for WebSocket support

When an HTTP request arrives, Uvicorn converts it into an **ASGI scope**.

Example:

```
{
  "type": "http",
  "method": "GET",
  "path": "/items/42",
  "query_string": b"q=hello",
  "headers": [...]
}
```

Then Uvicorn calls the application using the ASGI interface:

```
await app(scope, receive, send)
```

---

# 3. ASGI Interface

ASGI (Asynchronous Server Gateway Interface) is the standard communication protocol between Python web servers and web frameworks.

The application callable follows this structure:

```
async def app(scope, receive, send)
```

Parameters:

| Parameter | Purpose                                    |
| --------- | ------------------------------------------ |
| scope     | Metadata about the request                 |
| receive   | Function used to receive request body data |
| send      | Function used to send response messages    |

ASGI enables asynchronous web applications and supports protocols such as HTTP and WebSockets.

Different servers can run the same application because they all speak ASGI.

Common ASGI servers include:

* Uvicorn
* Hypercorn
* Daphne

---

# 4. Starlette (Framework Core)

FastAPI is built on top of Starlette. Starlette provides the core web framework features.

Responsibilities of Starlette include:

## Routing

Starlette matches incoming URLs with registered routes.

Example route:

```
/items/{item_id}
```

Request:

```
/items/42
```

Extracted path parameters:

```
item_id = 42
```

---

## Middleware System

Middleware allows processing requests before and after they reach the endpoint.

Typical middleware pipeline:

```
Request
  ↓
Middleware 1
  ↓
Middleware 2
  ↓
Router
  ↓
Endpoint
```

Common middleware examples:

* CORS handling
* Authentication
* Logging
* Rate limiting

---

## Request and Response Objects

Starlette constructs request and response abstractions:

Request objects provide access to:

* headers
* query parameters
* cookies
* body

Response types include:

* JSONResponse
* HTMLResponse
* StreamingResponse
* FileResponse

---

## Background Tasks

Starlette supports tasks that run after the response is sent.

Example use cases:

* sending emails
* logging
* asynchronous processing

---

## WebSocket Support

Starlette also implements WebSocket handling using the ASGI protocol.

---

# 5. FastAPI Layer

FastAPI sits on top of Starlette and adds features specifically designed for building APIs.

Key responsibilities include:

---

## Parameter Extraction

FastAPI analyzes endpoint function signatures to determine where parameters come from.

Example endpoint:

```
def read_item(item_id: int, q: str | None = None)
```

FastAPI determines:

```
item_id → path parameter
q → query parameter
```

---

## Data Validation with Pydantic

All incoming data is validated against Python type hints using Pydantic models.

Example request:

```
GET /items/abc
```

If `item_id` is defined as an integer, FastAPI automatically returns:

```
422 Unprocessable Entity
```

Validation ensures data consistency and prevents invalid inputs.

---

## Dependency Injection

FastAPI has a powerful dependency injection system.

Example:

```
def get_db():
    ...

@app.get("/items")
def read_items(db = Depends(get_db)):
```

FastAPI constructs a **dependency graph** and resolves all dependencies before executing the endpoint.

Dependencies can be used for:

* database sessions
* authentication
* configuration
* shared services

---

## Automatic API Documentation

FastAPI automatically generates OpenAPI documentation.

Available endpoints:

```
/docs
/redoc
/openapi.json
```

The documentation is generated from:

* type hints
* Pydantic models
* route metadata

---

## Response Serialization

Endpoint return values are automatically converted into HTTP responses.

Example endpoint:

```
return {"id": 42}
```

FastAPI converts this into:

```
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 42}
```

Serialization uses FastAPI utilities such as:

```
jsonable_encoder()
```

---

# 6. Endpoint Execution

Finally, the user-defined endpoint function executes.

Example:

```
@app.get("/items/{id}")
async def read_item(id: int):
    return {"id": id}
```

FastAPI determines how to execute the function based on its type.

Execution models:

| Endpoint Type | Execution Method                |
| ------------- | ------------------------------- |
| async def     | Runs as coroutine on event loop |
| def           | Executed in thread pool         |

If a synchronous function is used, FastAPI runs it using a threadpool so the event loop remains free.

---

# 7. Response Pipeline

Once the endpoint returns a value, the response travels back through the stack.

```
Endpoint
  ↓
FastAPI serialization
  ↓
Starlette Response object
  ↓
Middleware (outbound)
  ↓
ASGI send()
  ↓
Uvicorn
  ↓
HTTP response to client
```

The final response is transmitted through the TCP connection back to the client.

---

# Complete FastAPI Request Lifecycle

```
Client Request
     ↓
Operating System Socket
     ↓
Uvicorn (HTTP parsing + event loop)
     ↓
ASGI Interface
     ↓
Starlette Middleware
     ↓
Starlette Router
     ↓
FastAPI Parameter Resolution
     ↓
Pydantic Validation
     ↓
Dependency Injection
     ↓
Endpoint Execution
     ↓
Response Serialization
     ↓
Starlette Response Handling
     ↓
ASGI send()
     ↓
Uvicorn sends HTTP response
     ↓
Client receives response
```

---

# Summary

FastAPI is composed of multiple layers working together:

| Layer            | Responsibility                                 |
| ---------------- | ---------------------------------------------- |
| Operating System | Network sockets and TCP communication          |
| Uvicorn          | ASGI server and event loop                     |
| ASGI             | Interface between server and framework         |
| Starlette        | Routing, middleware, request/response handling |
| FastAPI          | Validation, dependency injection, OpenAPI docs |
| Application Code | Business logic                                 |

Understanding this architecture helps explain why FastAPI is both **high performance and highly maintainable**.

The combination of asynchronous execution, ASGI standards, and modern Python typing allows FastAPI to build scalable API services efficiently.
