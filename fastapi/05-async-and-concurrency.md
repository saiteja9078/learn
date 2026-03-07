# 05 — Async and Concurrency

> **Goal**: Understand Python's `async`/`await`, how FastAPI handles sync vs async endpoints, the event loop, and background tasks.

---

## 🧠 Why Async Matters

Most APIs spend time **waiting** — for DB queries, external APIs, file I/O. With synchronous code, the thread is **blocked** while waiting. With async, the server handles other requests during the wait.

```
SYNCHRONOUS: Each request blocks a thread
Thread 1: [Request A][────wait for DB────][Response A]
Thread 2:            [Request B][────wait for DB────][Response B]
→ 1000 concurrent requests = 1000 threads = ~2GB RAM

ASYNCHRONOUS: One thread handles many requests
Event Loop: [Req A][wait]→[Req B][wait]→[A done→Resp A][B done→Resp B]
→ 1000 concurrent requests = 1 thread = minimal overhead
```

---

## 📚 async/await Crash Course

```python
import asyncio

# Regular function — runs immediately, blocks the thread
def sync_function():
    return "done"

# Async function (coroutine) — can pause and resume
async def async_function():
    return "done"

# 'await' means: "pause here, let other tasks run, resume when done"
async def fetch_data():
    print("Starting...")
    await asyncio.sleep(1)   # Pauses — event loop handles other work
    print("Done!")
    return {"data": [1, 2, 3]}

# Running concurrently with asyncio.gather
async def main():
    # Both tasks run at the same time — total time ~2s, not 3s
    results = await asyncio.gather(
        slow_task_2_seconds(),
        slow_task_1_second(),
    )

asyncio.run(main())
```

### The Event Loop Visualized

```
The event loop is like a waiter serving multiple tables:

  ┌──────────── Event Loop ─────────────┐
  │                                     │
  │  Time 0s: Start task_a, task_b      │
  │    → task_a hits 'await' → pauses   │
  │    → task_b hits 'await' → pauses   │
  │                                     │
  │  Time 1s: task_b's wait is done     │
  │    → Resume task_b → finishes       │
  │                                     │
  │  Time 2s: task_a's wait is done     │
  │    → Resume task_a → finishes       │
  └─────────────────────────────────────┘

  The loop NEVER blocks. When a task waits, it moves on.
```

---

## ⚡ Async vs Sync Endpoints in FastAPI

This is the **most important section**. FastAPI handles `async def` and `def` completely differently:

```python
from fastapi import FastAPI
import asyncio, time

app = FastAPI()

# ────── ASYNC endpoint ──────
@app.get("/async")
async def async_endpoint():
    """
    Runs directly on the event loop (main thread).

    ✅ Use when calling async libraries (asyncpg, httpx, aiofiles)
    ❌ NEVER do blocking calls here (time.sleep, requests.get)
       — it freezes the ENTIRE server!
    """
    await asyncio.sleep(1)  # Non-blocking — correct!
    return {"type": "async"}

# ────── SYNC endpoint ──────
@app.get("/sync")
def sync_endpoint():
    """
    FastAPI runs this in a THREAD POOL automatically.
    Equivalent to: await asyncio.to_thread(sync_endpoint)

    ✅ Use for blocking/synchronous libraries (requests, psycopg2)
    ✅ Use when you're not sure — sync is the safer default
    """
    time.sleep(1)  # Blocking is fine here — it's in a worker thread
    return {"type": "sync"}
```

### ⚠️ The #1 Mistake

```python
# ❌ WRONG — blocks the event loop, freezes entire server!
@app.get("/bad")
async def bad_endpoint():
    import requests  # synchronous library!
    response = requests.get("https://api.example.com/data")
    return response.json()

# ✅ Option 1: Use async library
@app.get("/good-async")
async def good_endpoint():
    import httpx  # async library
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return response.json()

# ✅ Option 2: Use regular def (auto-threaded)
@app.get("/good-sync")
def good_endpoint():
    import requests
    response = requests.get("https://api.example.com/data")
    return response.json()
```

### Decision Flowchart

```
Should I use 'async def' or 'def'?

  Using any 'await' calls?
  ├── NO → Use 'def' (FastAPI auto-threads it)
  └── YES → Are ALL I/O operations async?
            ├── YES → Use 'async def' ✅
            └── NO  → Use 'def' (safer default)
```

---

## 🏃 Background Tasks

Run code **after** the response is sent (e.g., send emails, write logs):

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def send_email(email: str, message: str):
    """Runs AFTER the response is sent. Client doesn't wait."""
    import time
    print(f"📧 Sending email to {email}...")
    time.sleep(3)  # Slow operation
    print(f"✅ Email sent!")

def write_log(message: str):
    with open("app.log", "a") as f:
        f.write(f"{message}\n")

@app.post("/register/")
async def register_user(
    email: str,
    background_tasks: BackgroundTasks  # FastAPI injects this
):
    """
    1. Client sends POST /register/?email=user@example.com
    2. We queue background tasks (they DON'T run yet)
    3. Response is sent IMMEDIATELY
    4. Background tasks run AFTER response is sent
    """
    background_tasks.add_task(send_email, email, "Welcome!")
    background_tasks.add_task(write_log, f"New user: {email}")

    return {"message": f"Registered! Email will be sent shortly."}
    # Client gets this instantly — doesn't wait for email

# For critical/long tasks, use Celery instead of BackgroundTasks
```

---

## 🔬 Internals: Uvicorn's Architecture

```
uvicorn main:app --workers 4

┌────────────────── Process 1 ──────────────────┐
│  ┌──── Main Thread (Event Loop) ────┐         │
│  │  async def endpoints run here    │         │
│  └──────────────────────────────────┘         │
│  ┌──── Thread Pool (40 threads) ────┐         │
│  │  def endpoints run here          │         │
│  └──────────────────────────────────┘         │
├───────────────────────────────────────────────┤
│  Process 2 (same structure)                   │
├───────────────────────────────────────────────┤
│  Process 3 (same structure)                   │
├───────────────────────────────────────────────┤
│  Process 4 (same structure)                   │
└───────────────────────────────────────────────┘

'async def' → Event Loop (1 thread, non-blocking)
'def'       → Thread Pool (40 threads by default)
'--workers' → Multiple processes (for multi-core CPUs)
```

---

## ➡️ Next Steps

Head to **[06 — Database Integration](./06-database-integration.md)** to connect FastAPI to PostgreSQL — both sync and async — with full CRUD examples.
