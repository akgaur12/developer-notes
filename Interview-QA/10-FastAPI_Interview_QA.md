# ⚡ FastAPI Interview Q&A

## 🔹 Fundamentals

### 1. What is FastAPI?
FastAPI is a modern, high-performance **Python web framework** for building APIs, built on top of **Starlette** (for the web/ASGI layer) and **Pydantic** (for data validation), with native support for async I/O and automatic interactive API documentation.

---

### 2. Why is FastAPI fast?
- Built on **ASGI** (via Starlette/Uvicorn) instead of WSGI, enabling true async, non-blocking request handling
- Pydantic (v2 is Rust-based) provides highly optimized data validation/serialization
- Minimal overhead in routing and dependency resolution compared to older sync frameworks
- Performance is comparable to Node.js/Go web frameworks in benchmarks, well above Flask/Django (sync mode)

---

### 3. FastAPI vs Flask vs Django — how do they compare?
| FastAPI | Flask | Django |
|----|----|----|
| ASGI (async-native) | WSGI (sync, async partial) | WSGI (sync, ASGI partial support) |
| Built-in validation via Pydantic | No built-in validation | Uses Django Forms/Serializers (DRF) |
| Auto-generated OpenAPI docs | Requires extensions | Requires DRF for OpenAPI |
| Type-hint driven | Untyped by default | Untyped by default |
| Minimal, unopinionated | Minimal, unopinionated | Full-featured (ORM, admin, auth built-in) |
| Best for: APIs, microservices | Best for: small apps, simple APIs | Best for: full monolithic web apps |

---

### 4. What is ASGI, and how is it different from WSGI?
**ASGI** (Asynchronous Server Gateway Interface) is the successor to **WSGI**, supporting asynchronous applications, WebSockets, and long-lived connections — not just simple synchronous request/response cycles. FastAPI is built entirely around ASGI, which is what enables its native `async def` endpoint support.

---

### 5. What is Uvicorn, and why does FastAPI need an ASGI server?
Uvicorn is a lightning-fast **ASGI server** implementation (built on `uvloop` and `httptools`) that actually runs the FastAPI application and handles incoming HTTP connections — FastAPI itself is just the framework/application code; it needs an ASGI server like Uvicorn (or Hypercorn/Daphne) to serve traffic.

---

### 6. What is Starlette's relationship to FastAPI?
FastAPI is built **on top of Starlette** — it inherits Starlette's routing, middleware, WebSocket support, and background task systems, while adding Pydantic-based request/response validation, dependency injection, and automatic OpenAPI schema generation on top.

---

## 🔹 Path Operations & Request Handling

### 7. How do you define a basic route in FastAPI?
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

---

### 8. What's the difference between Path Parameters and Query Parameters?
- **Path parameters** – part of the URL path itself, declared in `{}` (e.g. `/items/{item_id}`), always required
- **Query parameters** – function arguments not in the path, appended as `?key=value` in the URL, optional by default unless given no default value

---

### 9. How does FastAPI use Python type hints for validation?
Every path/query parameter and request body field is declared with a **Python type hint**; FastAPI (via Pydantic) automatically validates incoming data against that type, converts it, and returns a structured **422 Unprocessable Entity** error with details if validation fails — no manual parsing/validation code needed.

---

### 10. What is a Pydantic model, and how is it used for the Request Body?
A Pydantic `BaseModel` subclass defines the expected shape/types of a JSON request body. FastAPI automatically parses, validates, and instantiates it as a Python object:
```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    is_offer: bool | None = None

@app.post("/items/")
async def create_item(item: Item):
    return item
```

---

### 11. How do you set default values, optional fields, and validation constraints on request data?
- Optional fields: type hint with `| None = None` (or `Optional[str] = None`)
- Extra validation/metadata: use `Field()` from Pydantic, e.g. `price: float = Field(gt=0)`
- Query parameter constraints: use `Query()`, e.g. `q: str = Query(default=None, max_length=50)`
- Path parameter constraints: use `Path()`, e.g. `item_id: int = Path(gt=0)`

---

### 12. What's the difference between `response_model` and just returning a Pydantic object?
`response_model` explicitly declares the **shape of the output** in the route decorator (`@app.get("/items/", response_model=Item)`), which FastAPI uses to **filter/validate the response** (e.g. stripping fields not in the model, like hiding a hashed password), independent of what the actual return object contains, and to generate accurate OpenAPI docs.

---

### 13. How do you handle file uploads in FastAPI?
Using `UploadFile` (from `fastapi`), which wraps the file in a spooled temporary file object supporting async read, and is more memory-efficient than `bytes` for large files:
```python
from fastapi import UploadFile

@app.post("/upload/")
async def upload_file(file: UploadFile):
    contents = await file.read()
    return {"filename": file.filename}
```

---

## 🔹 Async & Concurrency

### 14. When should a path operation be `async def` vs regular `def`?
- Use `async def` when the function performs **async I/O** (e.g. `await` on an async DB driver, async HTTP client) so it doesn't block the event loop
- Use regular `def` when the function does **blocking/sync work** (e.g. a sync DB library, CPU-bound code) — FastAPI automatically runs sync path operations in a **separate thread pool**, so they don't block the event loop either

---

### 15. What happens if you call blocking (sync) code inside an `async def` route without awaiting it properly?
It **blocks the entire event loop**, stalling every other concurrent request being handled by that worker process — this is one of the most common FastAPI performance bugs (e.g. calling a sync `requests.get()` inside an `async def` route instead of an async HTTP client like `httpx`).

---

### 16. How does FastAPI achieve concurrency with a single-threaded event loop?
Through Python's `asyncio` event loop: when an `async def` route hits an `await` (e.g. waiting on a DB query or HTTP call), it **yields control** back to the event loop, which can process other requests in the meantime — enabling high throughput for I/O-bound workloads without needing one thread per request.

---

### 17. How would you run CPU-bound work in a FastAPI app without blocking the event loop?
Offload it to a separate process pool (e.g. `concurrent.futures.ProcessPoolExecutor` via `loop.run_in_executor`) or a background task queue (Celery, ARQ, RQ) — async/await only helps with I/O-bound waiting, not CPU-bound computation, which still needs true parallelism (multiprocessing).

---

## 🔹 Dependency Injection

### 18. What is FastAPI's Dependency Injection system, and why use it?
A system (`Depends()`) for declaring **reusable, composable pieces of logic** (DB session creation, authentication, pagination params) that FastAPI automatically resolves and injects into path operations — promoting code reuse and separation of concerns instead of repeating logic in every route.
```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
async def read_items(db: Session = Depends(get_db)):
    ...
```

---

### 19. What is the purpose of `yield` inside a dependency function?
It splits the dependency into a **setup** phase (before `yield`) and a **teardown/cleanup** phase (after `yield`, run once the response has been sent) — the standard pattern for managing resources like DB sessions or connections that need guaranteed cleanup.

---

### 20. Can dependencies depend on other dependencies?
Yes — dependencies can themselves declare `Depends()` on other dependencies, and FastAPI resolves the whole chain, **caching** the result of each dependency **per request** by default so a shared dependency (e.g. `get_current_user`) isn't recomputed multiple times within the same request.

---

### 21. What are `Depends` use cases beyond dependency injection for shared logic?
- **Authentication/authorization** – e.g. `get_current_user` extracting/validating a token
- **Database session management**
- **Shared query parameter validation** (e.g. pagination: `skip`, `limit`)
- **Rate limiting / permission checks**
- Global dependencies applied to an entire router or app via `dependencies=[Depends(...)]`

---

## 🔹 Validation, Errors & Middleware

### 22. How does FastAPI handle validation errors?
Automatically — if incoming data doesn't match the declared Pydantic types/constraints, FastAPI returns a `422 Unprocessable Entity` response with a structured JSON body detailing exactly which field(s) failed and why, without any manual error-handling code.

---

### 23. How do you raise a custom HTTP error in FastAPI?
Using `HTTPException`:
```python
from fastapi import HTTPException

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
```

---

### 24. How do you handle custom exceptions globally?
Using `@app.exception_handler(CustomException)` to register a handler that converts a specific exception type into a structured JSON response application-wide, instead of catching it in every route individually.

---

### 25. What is Middleware in FastAPI, and how do you add it?
Code that runs **on every request/response**, before the route handler and after it returns — used for logging, timing, adding headers, or global error handling.
```python
@app.middleware("http")
async def add_process_time_header(request, call_next):
    response = await call_next(request)
    response.headers["X-Process-Time"] = "..."
    return response
```

---

### 26. What is CORS, and how do you enable it in FastAPI?
Cross-Origin Resource Sharing — a browser security mechanism restricting which origins can call your API from client-side JS. Enabled via `CORSMiddleware`:
```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(CORSMiddleware, allow_origins=["https://example.com"], allow_methods=["*"])
```

---

## 🔹 Background Tasks, WebSockets & Streaming

### 27. What are Background Tasks in FastAPI?
A lightweight mechanism (`BackgroundTasks`) to run a function **after the response has already been sent** to the client (e.g. sending a confirmation email) — for heavier/long-running async work, a proper task queue (Celery, ARQ) is preferred since background tasks run in the same process and aren't persisted/retried.
```python
@app.post("/send-notification/")
async def notify(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(send_email, email)
    return {"message": "Notification sent in the background"}
```

---

### 28. How do you implement WebSockets in FastAPI?
Using `@app.websocket()`, which gives access to a `WebSocket` object for bidirectional message exchange, common in chat apps or real-time/streaming LLM token responses.
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Echo: {data}")
```

---

### 29. How do you stream a response in FastAPI (e.g. for streaming an LLM's output)?
Using `StreamingResponse`, which takes a generator/async generator and streams chunks to the client incrementally instead of waiting for the full response body:
```python
from fastapi.responses import StreamingResponse

async def token_generator():
    for token in llm_stream():
        yield token

@app.get("/generate")
async def generate():
    return StreamingResponse(token_generator(), media_type="text/plain")
```

---

## 🔹 Security & Authentication

### 30. What built-in security utilities does FastAPI provide?
The `fastapi.security` module provides OAuth2 (`OAuth2PasswordBearer`), HTTP Basic/Bearer auth, and API key schemes — these integrate with FastAPI's dependency injection and automatically document the auth scheme in the OpenAPI UI.

---

### 31. How do you implement JWT-based authentication in FastAPI?
Typically: a login endpoint issues a signed JWT (via a library like `python-jose` or `pyjwt`); a `get_current_user` dependency decodes/validates the token from the `Authorization` header (via `OAuth2PasswordBearer`) on protected routes, raising `HTTPException(401)` on failure.

---

### 32. How do you restrict a route to specific user roles/permissions?
By adding a dependency (e.g. `Depends(require_role("admin"))`) that checks the authenticated user's role/claims and raises `HTTPException(403)` if unauthorized — composed on top of the base authentication dependency.

---

## 🔹 Documentation, Testing & Deployment

### 33. How does FastAPI auto-generate API documentation?
It generates an **OpenAPI (Swagger) schema** automatically from your route definitions, type hints, and Pydantic models, and serves interactive docs UIs at `/docs` (Swagger UI) and `/redoc` (ReDoc) out of the box — no separate documentation writing needed.

---

### 34. How do you test a FastAPI application?
Using `TestClient` (from `fastapi.testclient`, backed by `httpx`), which lets you call your API endpoints directly in tests without running a live server:
```python
from fastapi.testclient import TestClient
client = TestClient(app)

def test_read_item():
    response = client.get("/items/1")
    assert response.status_code == 200
```

---

### 35. What is `APIRouter`, and why use it?
A way to **modularize routes** into separate files/modules (e.g. `users.py`, `items.py`), each declaring its own `APIRouter()`, which are then included into the main app via `app.include_router(router, prefix="/users")` — keeping large applications organized instead of one giant `main.py`.

---

### 36. How do you manage configuration/environment variables in FastAPI apps?
Commonly via Pydantic's `BaseSettings` (in `pydantic-settings` for Pydantic v2), which reads and validates environment variables/`.env` files into a typed settings object, injected as a dependency where needed.

---

### 37. How do you deploy a FastAPI application in production?
Run it behind an ASGI server like **Uvicorn** with multiple worker processes (often managed by **Gunicorn** using `uvicorn.workers.UvicornWorker`), typically behind a reverse proxy (Nginx) for TLS termination and load balancing, and containerized with Docker for consistent deployment.

---

### 38. How do you version a FastAPI API?
Common approaches:
- **URL path versioning** – `/v1/items`, `/v2/items` via separate `APIRouter`s with different prefixes
- **Header-based versioning** – client specifies version in a custom header
- Maintaining separate router modules per version keeps backward compatibility manageable

---

### 39. What are common performance pitfalls in FastAPI applications?
- Blocking the event loop with sync calls inside `async def` routes
- Not using connection pooling for databases/HTTP clients
- N+1 query patterns when using an ORM without eager loading
- Doing heavy CPU-bound work in-process instead of offloading to a worker/queue
- Returning huge payloads without pagination or streaming

---

### 40. Why is FastAPI a popular choice for serving LLM/AI applications?
- Native async support suits I/O-bound LLM API calls and streaming token responses
- Pydantic validation cleanly enforces structured request/response schemas (important for tool-calling/agent APIs)
- `StreamingResponse`/WebSockets map naturally to streaming LLM output to a frontend
- Auto-generated OpenAPI docs make it easy to expose an API as a callable tool (including for MCP-style tool servers)
- Lightweight and fast enough to avoid becoming the bottleneck in an LLM-backed pipeline

---
