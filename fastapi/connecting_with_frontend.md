[fastapi_react_ts_connection_guide.md](https://github.com/user-attachments/files/31259032/fastapi_react_ts_connection_guide.md)
# Connecting React + TypeScript Frontend to FastAPI Backend

This guide shows the three common ways a React + TypeScript frontend communicates with a FastAPI backend:

1. Normal HTTP request/response
2. WebSocket communication
3. HTTP async streaming

The examples use small snippets rather than a complete application.

---

## 1. First: What is Axios?

**Axios is an HTTP client for JavaScript/TypeScript.**

It gives the frontend a convenient API for sending HTTP requests such as:

- `GET`
- `POST`
- `PUT`
- `PATCH`
- `DELETE`

For example:

```ts
import axios from "axios";

const response = await axios.get("http://localhost:8000/users");

console.log(response.data);
```

Conceptually:

```text
React component
      |
      | axios.get(...)
      v
Browser HTTP client
      |
      | HTTP request
      v
FastAPI
      |
      | HTTP response
      v
Axios Promise resolves
      |
      v
response.data
```

Axios is **not a replacement for HTTP**. HTTP is the protocol. Axios is a JavaScript/TypeScript library that makes working with HTTP requests and responses easier.

Axios also provides useful features such as request/response interceptors, JSON handling, headers, timeouts, error handling, and configuration defaults.

---

# 2. Basic Project Setup

A typical project can look like this:

```text
project/
├── backend/
│   └── main.py
└── frontend/
    └── src/
        ├── api.ts
        ├── websocket.ts
        └── components/
```

Assume:

```text
FastAPI: http://localhost:8000
React/Vite: http://localhost:5173
```

Because these are different origins, the FastAPI application normally needs CORS configuration during development.

## FastAPI CORS

```py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

CORS is a **browser security mechanism**. It is not something Axios invented, and it is not required merely because the backend is FastAPI.

---

# 3. Normal HTTP Request

This is the simplest communication pattern.

```text
Frontend                       Backend
   |                              |
   | -------- HTTP request ------>|
   |                              |
   | <------- HTTP response ------|
   |                              |
```

The request is sent, the server processes it, and eventually one HTTP response is returned.

---

## 3.1 FastAPI backend

```py
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {
        "id": user_id,
        "name": "Sai"
    }
```

The endpoint is:

```text
GET http://localhost:8000/users/10
```

---

## 3.2 React + TypeScript + Axios

```ts
import axios from "axios";

interface User {
  id: number;
  name: string;
}

async function getUser(): Promise<User> {
  const response = await axios.get<User>(
    "http://localhost:8000/users/10"
  );

  return response.data;
}
```

Then in a React component:

```tsx
import { useEffect, useState } from "react";

function App() {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    getUser().then(setUser);
  }, []);

  return <div>{user?.name}</div>;
}
```

---

# 4. The Important TypeScript Idea: `async` + `Promise`

This is fundamental to frontend/backend communication.

Consider:

```ts
async function getUser(): Promise<User> {
  const response = await axios.get<User>(url);
  return response.data;
}
```

The function does **not** immediately return a `User`.

It returns:

```text
Promise<User>
```

Think of it as:

```text
getUser()
   |
   v
Promise<User>
   |
   | eventually resolves
   v
User
```

Therefore this is wrong:

```ts
const user: User = getUser();
```

because:

```text
getUser() -> Promise<User>
```

not:

```text
User
```

You need:

```ts
const user: User = await getUser();
```

inside an `async` function.

---

# 5. What Exactly Does `await` Do?

Consider:

```ts
const response = await axios.get<User>(url);
```

`axios.get()` immediately gives you a Promise.

`await` tells JavaScript:

> Pause this async function until this Promise settles, then continue with the resolved value.

It does **not** freeze the entire browser.

For example:

```ts
async function load() {
  console.log("A");

  const response = await axios.get<User>(url);

  console.log("B");
}

console.log("C");
load();
console.log("D");
```

A typical ordering is:

```text
A
C
D
B
```

The browser can continue doing other work while the HTTP operation is waiting.

This is one reason asynchronous JavaScript is so important for frontend applications.

---

# 6. Axios Response Object

When you do:

```ts
const response = await axios.get<User>(url);
```

`response` is an Axios response object, not directly the user.

For example, conceptually:

```ts
response.status
response.headers
response.data
```

The actual backend payload is usually:

```ts
response.data
```

So:

```ts
const response = await axios.get<User>(url);
const user = response.data;
```

---

# 7. POST Request

### FastAPI

```py
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str

@app.post("/users")
async def create_user(user: UserCreate):
    return {
        "id": 1,
        "name": user.name
    }
```

### TypeScript + Axios

```ts
interface User {
  id: number;
  name: string;
}

interface UserCreate {
  name: string;
}

async function createUser(data: UserCreate): Promise<User> {
  const response = await axios.post<User>(
    "http://localhost:8000/users",
    data
  );

  return response.data;
}
```

The flow is:

```text
React object
    |
    | axios.post(...)
    v
HTTP request body
    |
    v
FastAPI / Pydantic
    |
    v
Python object
    |
    v
JSON response
    |
    v
Axios
    |
    v
TypeScript object
```

---

# 8. Axios Instance

Once an application gets bigger, you usually do not want to repeat the backend URL everywhere.

Create one Axios instance:

```ts
import axios from "axios";

export const api = axios.create({
  baseURL: "http://localhost:8000",
  timeout: 10000,
});
```

Now:

```ts
const response = await api.get<User>("/users/10");
```

instead of:

```ts
axios.get("http://localhost:8000/users/10");
```

This becomes especially useful for authentication headers and interceptors.

---

# 9. Axios Interceptors

An interceptor lets you run logic before a request or after a response.

For example, adding an access token:

```ts
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("access_token");

  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }

  return config;
});
```

This avoids repeating:

```ts
headers: {
  Authorization: `Bearer ${token}`
}
```

in every request.

---

# 10. HTTP Streaming Is Different

Now we move to an important distinction.

Normal HTTP usually looks like:

```text
request
   |
   v
[wait]
   |
   v
complete response
```

Streaming HTTP looks more like:

```text
request
   |
   v
chunk 1
   |
chunk 2
   |
chunk 3
   |
chunk 4
   |
   v
end
```

The connection remains open while the server produces pieces of the response.

This is useful for things such as:

- LLM token streaming
- progress updates
- large generated content
- logs
- incremental results

FastAPI can stream an HTTP response using `StreamingResponse` and an iterator/generator. citeturn740986search2turn740986search4

---

# 11. FastAPI Async Streaming

A simple example:

```py
import asyncio
from collections.abc import AsyncGenerator
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def generate() -> AsyncGenerator[str, None]:
    for i in range(5):
        yield f"chunk-{i}\n"
        await asyncio.sleep(1)

@app.get("/stream")
async def stream():
    return StreamingResponse(
        generate(),
        media_type="text/plain"
    )
```

The important part is:

```py
yield f"chunk-{i}\n"
```

`yield` produces one piece instead of creating the entire response first.

---

# 12. Why Normal `axios.get()` Is Not the Main Tool for Browser Streaming

For an ordinary request, this is perfect:

```ts
const response = await api.get<User>("/users/10");
```

You wait for the response and then read:

```ts
response.data
```

For browser-side HTTP streaming, it is usually clearer to use the browser's native `fetch()` API because `fetch()` exposes the response body as a `ReadableStream`.

The important distinction is:

```text
Axios normal request:

Promise -> complete response -> response.data

Fetch streaming:

Promise<Response>
       |
       v
response.body
       |
       v
ReadableStream
       |
       v
chunk -> chunk -> chunk -> chunk
```

The browser Streams API supports asynchronous consumption of incoming chunks. citeturn740986search8

---

# 13. React + TypeScript HTTP Streaming

```ts
async function streamData() {
  const response = await fetch("http://localhost:8000/stream");

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  if (!response.body) {
    throw new Error("Response body is not available");
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { value, done } = await reader.read();

    if (done) {
      break;
    }

    const text = decoder.decode(value, { stream: true });

    console.log(text);
  }
}
```

This is the key TypeScript pattern for low-level HTTP streaming in the browser.

---

# 14. Understanding `reader.read()`

This line is very important:

```ts
const { value, done } = await reader.read();
```

`reader.read()` returns a Promise.

Eventually it resolves to something conceptually like:

```ts
{
  value: Uint8Array,
  done: false
}
```

for an incoming chunk.

When the stream is finished:

```ts
{
  value: undefined,
  done: true
}
```

So this:

```ts
while (true) {
  const { value, done } = await reader.read();

  if (done) {
    break;
  }

  // process value
}
```

means:

```text
read chunk
   |
   v
wait asynchronously
   |
   v
chunk received
   |
   +---- done? ---- yes ---> exit
   |
   no
   v
process chunk
   |
   v
read next chunk
```

`await` is therefore happening **for every chunk**.

The function is not blocking the browser between chunks; it suspends the async function and resumes it when the next piece becomes available.

---

# 15. Why `TextDecoder` Is Needed

Network streams commonly give JavaScript binary chunks (`Uint8Array`), not necessarily JavaScript strings.

So:

```ts
const { value } = await reader.read();
```

may give you bytes.

You convert them to text:

```ts
const text = decoder.decode(value, { stream: true });
```

The `{ stream: true }` option matters when a character can be split across byte chunks.

Conceptually:

```text
network
   |
   v
bytes
   |
   v
TextDecoder
   |
   v
string
```

---

# 16. A Better React Streaming Example

Suppose we want to display an LLM-like response as it arrives.

```tsx
import { useState } from "react";

function App() {
  const [output, setOutput] = useState("");

  async function startStream() {
    const response = await fetch("http://localhost:8000/stream");

    if (!response.body) {
      throw new Error("No response body");
    }

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { value, done } = await reader.read();

      if (done) break;

      const chunk = decoder.decode(value, { stream: true });

      setOutput((previous) => previous + chunk);
    }
  }

  return (
    <>
      <button onClick={startStream}>Start</button>
      <pre>{output}</pre>
    </>
  );
}
```

Notice this:

```ts
setOutput((previous) => previous + chunk);
```

instead of:

```ts
setOutput(output + chunk);
```

The callback form is safer when multiple asynchronous updates may happen over time because React can give the updater the latest state value.

---

# 17. Streaming JSON: JSON Lines / NDJSON

Sending arbitrary text chunks works, but applications often want structured messages.

For example:

```text
{"type":"token","value":"Hello"}\n
{"type":"token","value":" world"}\n
{"type":"done"}\n
```

This pattern is commonly called **JSON Lines** or **NDJSON**.

FastAPI's streaming guidance specifically discusses JSON Lines when the streamed data should be structured as JSON. citeturn740986search2

Backend:

```py
import json
from collections.abc import AsyncGenerator

async def generate() -> AsyncGenerator[str, None]:
    yield json.dumps({"type": "token", "value": "Hello"}) + "\n"
    yield json.dumps({"type": "token", "value": " world"}) + "\n"
    yield json.dumps({"type": "done"}) + "\n"
```

Frontend parsing becomes:

```ts
const text = decoder.decode(value, { stream: true });

for (const line of text.split("\n")) {
  if (!line.trim()) continue;

  const message = JSON.parse(line);
  console.log(message);
}
```

In production, you normally also maintain a **buffer** because one network chunk is not guaranteed to contain exactly one complete JSON line.

---

# 18. The Important Streaming Concept: Chunk Boundaries Are Not Message Boundaries

Do **not** assume:

```text
one yield() == one fetch() chunk
```

or:

```text
one JSON object == one network chunk
```

The network and HTTP stack are free to split or combine bytes.

For example, the server may conceptually produce:

```text
{"token":"hello"}\n
{"token":"world"}\n
```

but the browser could receive:

```text
chunk 1: {"token":"hel"
chunk 2: "lo"}\n{"token":"wor"
chunk 3: "ld"}\n
```

Therefore streaming protocols need **framing**.

For JSON Lines, the newline is the frame delimiter.

---

# 19. WebSocket

WebSocket solves a different problem from normal request/response HTTP.

Normal HTTP:

```text
Client  ---- request ----> Server
Client  <--- response ---- Server
```

WebSocket:

```text
Client  <==== persistent connection ====>  Server
          messages in both directions
```

WebSocket provides a two-way interactive communication session where both sides can send messages over the established connection. citeturn740986search0turn740986search1

Typical use cases:

- chat applications
- multiplayer systems
- live dashboards
- collaborative applications
- real-time notifications

---

# 20. FastAPI WebSocket Endpoint

```py
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()

    while True:
        message = await websocket.receive_text()
        await websocket.send_text(f"Server received: {message}")
```

FastAPI exposes WebSocket support directly; the implementation is provided through Starlette. citeturn740986search0turn740986search5

---

# 21. React + TypeScript WebSocket

Unlike Axios, the browser already provides a WebSocket API.

```ts
const socket = new WebSocket("ws://localhost:8000/ws");
```

Then attach handlers:

```ts
socket.onopen = () => {
  console.log("connected");
  socket.send("hello");
};

socket.onmessage = (event) => {
  console.log("server:", event.data);
};

socket.onclose = () => {
  console.log("connection closed");
};

socket.onerror = (error) => {
  console.error(error);
};
```

The key difference is that a WebSocket is **event-driven**.

You do not normally write:

```ts
const response = await websocket.send(...);
```

Instead, you listen for events:

```ts
socket.onmessage = ...
```

---

# 22. WebSocket Does Not Behave Like Axios

This is a very important distinction.

### Axios

```ts
const response = await axios.get<User>(url);
```

The mental model is:

```text
one operation
    |
    v
one Promise
    |
    v
one response
```

### WebSocket

```ts
const socket = new WebSocket(url);
```

The mental model is:

```text
open connection
      |
      +---- send message ---->
      |
      +<---- receive message -+
      |
      +---- send message ---->
      |
      +<---- receive message -+
      |
      ...
      |
      +---- close ----------->
```

There can be many messages in both directions during one connection.

That is why WebSocket is better thought of as a **long-lived bidirectional channel**, not as a request function.

---

# 23. React WebSocket Hook Pattern

A small pattern:

```tsx
import { useEffect, useState } from "react";

function Chat() {
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    const socket = new WebSocket("ws://localhost:8000/ws");

    socket.onmessage = (event) => {
      setMessages((previous) => [...previous, event.data]);
    };

    return () => {
      socket.close();
    };
  }, []);

  return (
    <div>
      {messages.map((message, index) => (
        <p key={index}>{message}</p>
      ))}
    </div>
  );
}
```

The cleanup function is important because the React component may unmount.

Without cleanup, you can leave a socket connection running after the component disappears.

---

# 24. WebSocket Message Types

WebSockets can carry text, binary data, and JSON-style application messages. FastAPI's WebSocket interface supports text, binary, and JSON operations. citeturn740986search0

A common application pattern is to send JSON messages:

```json
{
  "type": "chat",
  "message": "hello"
}
```

Frontend:

```ts
socket.send(JSON.stringify({
  type: "chat",
  message: "hello"
}));
```

Receiving:

```ts
socket.onmessage = (event) => {
  const message = JSON.parse(event.data);

  console.log(message.type);
  console.log(message.message);
};
```

You can therefore define a small application-level protocol on top of WebSocket.

---

# 25. HTTP Streaming vs WebSocket

These two are often confused because both can deliver data incrementally.

| Property | HTTP streaming | WebSocket |
|---|---|---|
| Connection | HTTP connection | Persistent WebSocket connection |
| Communication | Usually request → streamed response | Bidirectional |
| Server sends multiple chunks | Yes | Yes |
| Client can continuously send messages during stream | Not naturally the same model | Yes |
| Typical API | `fetch()` + `ReadableStream` | `WebSocket` |
| Great for LLM output | Yes | Yes |
| Great for chat | Possible, but less natural | Yes |
| One request followed by one streamed result | Excellent | Often unnecessary |

A useful rule is:

```text
One operation producing incremental output
        -> HTTP streaming

Continuous two-way communication
        -> WebSocket
```

---

# 26. The Three TypeScript Methodologies

This is probably the most important part to remember.

## A. Normal HTTP: Promise-based

```ts
const response = await axios.get<User>(url);
```

Mental model:

```text
request
   |
   v
Promise
   |
   v
complete response
```

Use:

```ts
async/await
Promise<T>
```

---

## B. WebSocket: Event-based

```ts
const socket = new WebSocket(url);

socket.onmessage = (event) => {
  console.log(event.data);
};
```

Mental model:

```text
connection
   |
   +---- event ---> message received
   |
   +---- event ---> message received
   |
   +---- event ---> connection closed
```

Use:

```ts
WebSocket
onopen
onmessage
onerror
onclose
```

---

## C. HTTP streaming: Stream-based async iteration / reader

```ts
const response = await fetch(url);
const reader = response.body!.getReader();

while (true) {
  const { value, done } = await reader.read();

  if (done) break;

  // process chunk
}
```

Mental model:

```text
request
   |
   v
Promise<Response>
   |
   v
ReadableStream
   |
   +--> await chunk 1
   |
   +--> await chunk 2
   |
   +--> await chunk 3
   |
   +--> done
```

Use:

```ts
fetch()
ReadableStream
reader.read()
await
```

The browser Streams API also supports asynchronous iteration, so another style is possible:

```ts
const response = await fetch(url);

if (!response.body) throw new Error("No body");

for await (const chunk of response.body) {
  console.log(chunk);
}
```

Readable streams implement the async iterable protocol, allowing `for await...of`. citeturn740986search8

---

# 27. `async/await` Is Not the Same Thing as Streaming

This distinction is important.

You can have:

```ts
const response = await axios.get(url);
```

and that does **not** mean you are streaming.

Here `await` means:

> Wait for the Promise representing the operation to resolve.

Streaming means:

> The operation produces multiple pieces over time, and the application consumes those pieces before the entire operation is finished.

So:

```text
async/await
```

is a JavaScript control-flow mechanism.

Whereas:

```text
streaming
```

is a way data is delivered over time.

They often appear together:

```ts
const response = await fetch(url);

while (...) {
  const chunk = await reader.read();
}
```

but they are conceptually different things.

---

# 28. Axios vs Fetch vs WebSocket

| Requirement | Recommended browser API |
|---|---|
| Normal GET/POST/PUT/DELETE | Axios |
| Authentication headers/interceptors | Axios |
| Simple JSON API | Axios |
| HTTP streaming | `fetch()` + `ReadableStream` |
| Persistent bidirectional communication | `WebSocket` |
| React state updates | `useState` / `useReducer` |
| Lifecycle of socket/stream | `useEffect` or a custom hook |

You do **not** have to use one library for everything.

A perfectly normal stack is:

```text
Axios      -> ordinary REST/HTTP API
Fetch      -> HTTP streaming
WebSocket  -> real-time bidirectional communication
```

---

# 29. A Clean Frontend Architecture

Instead of putting all network code directly into React components, keep API logic separate.

For example:

```text
src/
├── api/
│   ├── client.ts
│   ├── users.ts
│   ├── stream.ts
│   └── socket.ts
├── components/
└── App.tsx
```

### `client.ts`

```ts
import axios from "axios";

export const api = axios.create({
  baseURL: "http://localhost:8000",
});
```

### `users.ts`

```ts
import { api } from "./client";

export interface User {
  id: number;
  name: string;
}

export async function getUser(id: number): Promise<User> {
  const response = await api.get<User>(`/users/${id}`);
  return response.data;
}
```

### React

```tsx
useEffect(() => {
  getUser(10).then(setUser);
}, []);
```

This keeps your component focused on UI instead of HTTP details.

---

# 30. Error Handling

For Axios:

```ts
try {
  const response = await api.get<User>("/users/10");
  console.log(response.data);
} catch (error) {
  console.error(error);
}
```

For Fetch, remember that `fetch()` does not reject merely because the server returned an HTTP error status such as 404 or 500.

Therefore check:

```ts
const response = await fetch(url);

if (!response.ok) {
  throw new Error(`HTTP ${response.status}`);
}
```

Then read the stream.

---

# 31. Cancellation

A request may become unnecessary when a React component unmounts or when the user starts a newer request.

Fetch uses `AbortController`:

```ts
const controller = new AbortController();

const response = await fetch(url, {
  signal: controller.signal,
});
```

Later:

```ts
controller.abort();
```

This is especially useful for streaming requests because a stream may otherwise remain open while the UI no longer needs it.

---

# 32. Authentication: HTTP vs WebSocket

With normal HTTP, an Axios interceptor can add an authorization header:

```ts
api.interceptors.request.use((config) => {
  const token = getToken();

  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }

  return config;
});
```

With WebSocket, the browser's native `WebSocket` API does not expose the same arbitrary-header mechanism that Axios gives you for HTTP requests.

Applications commonly use one of these approaches:

```text
secure cookies
query parameters/tokens during connection setup
subprotocols
```

The exact authentication design should depend on your security model.

---

# 33. One Mental Model for the Entire System

Think of the browser/backend connection as three layers.

```text
                 React UI
                    |
                    v
           TypeScript application
                    |
        +-----------+-----------+
        |           |           |
      Axios       Fetch     WebSocket
        |           |           |
       HTTP      HTTP stream   WS
        |           |           |
        +-----------+-----------+
                    |
                 FastAPI
                    |
        +-----------+-----------+
        |           |           |
      REST      Streaming    WebSocket
      route      route         route
```

And the response-handling styles are:

```text
HTTP JSON
    -> Promise
    -> await
    -> complete response

HTTP streaming
    -> Promise<Response>
    -> ReadableStream
    -> await chunks
    -> update UI incrementally

WebSocket
    -> persistent connection
    -> events/messages
    -> update UI whenever a message arrives
```

---

# 34. Minimal Cheat Sheet

## Normal API call

```ts
const response = await api.get<User>("/users/1");
const user = response.data;
```

FastAPI:

```py
@app.get("/users/{id}")
async def get_user(id: int):
    return {"id": id, "name": "Sai"}
```

---

## POST

```ts
const response = await api.post<User>("/users", {
  name: "Sai"
});
```

FastAPI:

```py
@app.post("/users")
async def create_user(user: UserCreate):
    return {"id": 1, "name": user.name}
```

---

## HTTP streaming

Frontend:

```ts
const response = await fetch("http://localhost:8000/stream");
const reader = response.body!.getReader();

while (true) {
  const { value, done } = await reader.read();
  if (done) break;

  // process chunk
}
```

Backend:

```py
return StreamingResponse(generate())
```

---

## WebSocket

Frontend:

```ts
const socket = new WebSocket("ws://localhost:8000/ws");

socket.onmessage = (event) => {
  console.log(event.data);
};

socket.send("hello");
```

Backend:

```py
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()

    while True:
        message = await websocket.receive_text()
        await websocket.send_text(message)
```

---

# 35. The Core Takeaways

```text
Axios
  = HTTP client

HTTP request
  = one request + one eventual response

async/await
  = JavaScript mechanism for working with Promises

Promise<T>
  = future result of an asynchronous operation

HTTP streaming
  = one HTTP response whose body arrives incrementally

ReadableStream
  = browser abstraction for consuming streaming data

WebSocket
  = persistent bidirectional communication channel
```

For a React + FastAPI application, a strong default architecture is:

```text
REST CRUD / normal APIs  -> Axios
LLM/token/output stream  -> Fetch + ReadableStream
Chat / real-time events  -> WebSocket
```

The biggest conceptual distinction is this:

```text
Axios:
"Give me the result of this HTTP request."

Streaming:
"Give me the pieces of this response as they arrive."

WebSocket:
"Keep this communication channel open so both sides can
send messages whenever they need to."
```
