# FastAPI Muscle Memory — 50+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Every exercise is a runnable FastAPI app.
> - Run: `uvicorn filename:app --reload`
> - Test with: browser → `http://127.0.0.1:8000/docs` (Swagger UI auto-generated)
> - Or use curl: `curl http://127.0.0.1:8000/endpoint`
> - L1–L3 → routes, methods, path/query params (20–30 min)
> - L4–L5 → Pydantic models, request body, validation (30–40 min)
> - L6–L7 → responses, status codes, error handling (30–40 min)
> - L8–L9 → dependencies, middleware, background tasks (40–50 min)
> - L10 → full CRUD REST API from scratch (45–60 min)
> - Install: `pip install fastapi uvicorn`

---

## L1 — First App, Routes, Methods

### 1. Hello FastAPI
```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello, FastAPI!"}

@app.get("/health")
def health():
    return {"status": "ok"}
```
> Run: `uvicorn main:app --reload`
> Visit: `http://127.0.0.1:8000/` and `http://127.0.0.1:8000/docs`

---

### 2. Multiple HTTP Methods
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items")
def get_items():
    return {"method": "GET", "items": ["apple", "banana", "cherry"]}

@app.post("/items")
def create_item():
    return {"method": "POST", "message": "Item created"}

@app.put("/items/1")
def update_item():
    return {"method": "PUT", "message": "Item updated"}

@app.delete("/items/1")
def delete_item():
    return {"method": "DELETE", "message": "Item deleted"}

@app.patch("/items/1")
def patch_item():
    return {"method": "PATCH", "message": "Item partially updated"}
```

---

### 3. Path Parameters
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id, "name": f"User_{user_id}"}

@app.get("/files/{file_path:path}")
def get_file(file_path: str):
    return {"file_path": file_path}

@app.get("/items/{item_id}/reviews/{review_id}")
def get_review(item_id: int, review_id: int):
    return {"item_id": item_id, "review_id": review_id}
```
> Test: `http://127.0.0.1:8000/users/42`
> Test: `http://127.0.0.1:8000/files/home/vishal/notes.txt`

---

### 4. Query Parameters
```python
from fastapi import FastAPI
from typing import Optional

app = FastAPI()

@app.get("/items")
def list_items(
    skip: int = 0,
    limit: int = 10,
    search: Optional[str] = None,
    active: bool = True
):
    return {
        "skip": skip,
        "limit": limit,
        "search": search,
        "active": active
    }
```
> Test: `http://127.0.0.1:8000/items?skip=5&limit=20&search=phone&active=true`

---

### 5. Path + Query Combined
```python
from fastapi import FastAPI
from typing import Optional

app = FastAPI()

fake_db = {
    1: {"name": "Alice", "role": "admin"},
    2: {"name": "Bob", "role": "viewer"},
    3: {"name": "Vishal", "role": "editor"},
}

@app.get("/users/{user_id}")
def get_user(user_id: int, include_role: bool = False):
    user = fake_db.get(user_id)
    if not user:
        return {"error": "User not found"}
    if not include_role:
        return {"user_id": user_id, "name": user["name"]}
    return {"user_id": user_id, **user}
```
> Test: `http://127.0.0.1:8000/users/1`
> Test: `http://127.0.0.1:8000/users/1?include_role=true`

---

## L2 — Pydantic Models and Request Body

### 6. Basic Pydantic Model
```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float
    in_stock: bool = True

@app.post("/items")
def create_item(item: Item):
    return {
        "message": "Item created",
        "item": item
    }

@app.post("/items/total")
def price_with_tax(item: Item, tax: float = 0.18):
    total = item.price * (1 + tax)
    return {
        "name": item.name,
        "price": item.price,
        "tax": tax,
        "total": round(total, 2)
    }
```
> Test via Swagger: `http://127.0.0.1:8000/docs`
> Body: `{"name": "Laptop", "price": 999.99}`

---

### 7. Pydantic Validation
```python
from fastapi import FastAPI
from pydantic import BaseModel, Field, EmailStr
from typing import Optional

app = FastAPI()

class UserCreate(BaseModel):
    name: str = Field(..., min_length=2, max_length=50)
    email: str = Field(..., pattern=r"^[\w.-]+@[\w.-]+\.\w+$")
    age: int = Field(..., ge=1, le=120)
    bio: Optional[str] = Field(None, max_length=500)
    score: float = Field(0.0, ge=0.0, le=100.0)

@app.post("/users")
def create_user(user: UserCreate):
    return {"message": "User created", "user": user}
```
> Try sending invalid data — FastAPI returns detailed errors automatically.

---

### 8. Nested Models
```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

class Address(BaseModel):
    street: str
    city: str
    state: str
    pincode: str

class Tag(BaseModel):
    name: str
    color: str = "blue"

class Product(BaseModel):
    id: int
    name: str
    price: float
    tags: List[Tag] = []
    warehouse: Optional[Address] = None

@app.post("/products")
def create_product(product: Product):
    return {
        "received": product,
        "tag_count": len(product.tags)
    }
```

---

### 9. Multiple Body Parameters
```python
from fastapi import FastAPI, Body
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

class Buyer(BaseModel):
    name: str
    email: str

@app.post("/purchase")
def purchase(
    item: Item,
    buyer: Buyer,
    quantity: int = Body(..., ge=1, le=100)
):
    return {
        "buyer": buyer.name,
        "item": item.name,
        "quantity": quantity,
        "total": item.price * quantity
    }
```

---

### 10. Response Model
```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

class UserCreate(BaseModel):
    name: str
    email: str
    password: str  # we accept this but never return it

class UserOut(BaseModel):
    id: int
    name: str
    email: str
    # password is intentionally excluded

fake_users = []

@app.post("/users", response_model=UserOut)
def create_user(user: UserCreate):
    new_user = {"id": len(fake_users) + 1, "name": user.name, "email": user.email}
    fake_users.append(new_user)
    return new_user  # password is stripped by response_model

@app.get("/users", response_model=List[UserOut])
def list_users():
    return fake_users
```
> Key insight: `response_model` filters and validates what goes OUT.

---

## L3 — CRUD with In-Memory Store

### 11. Basic In-Memory CRUD
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, Optional

app = FastAPI()

class Task(BaseModel):
    title: str
    done: bool = False

class TaskOut(Task):
    id: int

tasks: Dict[int, dict] = {}
counter = 0

@app.post("/tasks", response_model=TaskOut)
def create_task(task: Task):
    global counter
    counter += 1
    tasks[counter] = {"id": counter, "title": task.title, "done": task.done}
    return tasks[counter]

@app.get("/tasks", response_model=list[TaskOut])
def list_tasks():
    return list(tasks.values())

@app.get("/tasks/{task_id}", response_model=TaskOut)
def get_task(task_id: int):
    if task_id not in tasks:
        raise HTTPException(status_code=404, detail="Task not found")
    return tasks[task_id]

@app.put("/tasks/{task_id}", response_model=TaskOut)
def update_task(task_id: int, task: Task):
    if task_id not in tasks:
        raise HTTPException(status_code=404, detail="Task not found")
    tasks[task_id].update({"title": task.title, "done": task.done})
    return tasks[task_id]

@app.delete("/tasks/{task_id}")
def delete_task(task_id: int):
    if task_id not in tasks:
        raise HTTPException(status_code=404, detail="Task not found")
    deleted = tasks.pop(task_id)
    return {"message": "Deleted", "task": deleted}
```

---

### 12. Partial Update with PATCH
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, Dict

app = FastAPI()

class Task(BaseModel):
    title: str
    priority: str = "medium"
    done: bool = False

class TaskUpdate(BaseModel):
    title: Optional[str] = None
    priority: Optional[str] = None
    done: Optional[bool] = None

tasks: Dict[int, dict] = {
    1: {"id": 1, "title": "Buy groceries", "priority": "low", "done": False},
    2: {"id": 2, "title": "Finish FastAPI drills", "priority": "high", "done": False},
}

@app.patch("/tasks/{task_id}")
def patch_task(task_id: int, updates: TaskUpdate):
    if task_id not in tasks:
        raise HTTPException(status_code=404, detail="Task not found")

    # Only update fields that were sent
    update_data = updates.model_dump(exclude_unset=True)
    tasks[task_id].update(update_data)
    return tasks[task_id]
```
> Key: `exclude_unset=True` — only updates fields the client actually sent.

---

## L4 — Status Codes and HTTPException

### 13. Status Codes
```python
from fastapi import FastAPI
from fastapi import status

app = FastAPI()

@app.post("/items", status_code=status.HTTP_201_CREATED)
def create_item():
    return {"message": "Created"}

@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: int):
    return  # 204 = no content, return nothing

@app.get("/redirect", status_code=status.HTTP_302_FOUND)
def redirect():
    return {"location": "http://example.com"}
```

---

### 14. HTTPException — Proper Error Handling
```python
from fastapi import FastAPI, HTTPException, status
from typing import Dict

app = FastAPI()

fake_db: Dict[int, dict] = {
    1: {"id": 1, "name": "Vishal", "role": "admin"},
    2: {"id": 2, "name": "Alice", "role": "viewer"},
}

@app.get("/users/{user_id}")
def get_user(user_id: int):
    if user_id not in fake_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User with id {user_id} not found"
        )
    return fake_db[user_id]

@app.delete("/users/{user_id}")
def delete_user(user_id: int, requester_role: str = "viewer"):
    if requester_role != "admin":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Only admins can delete users"
        )
    if user_id not in fake_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    del fake_db[user_id]
    return {"message": "User deleted"}
```

---

### 15. Custom Exception Handler
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

class AppException(Exception):
    def __init__(self, status_code: int, message: str, error_code: str):
        self.status_code = status_code
        self.message = message
        self.error_code = error_code

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error_code,
            "message": exc.message,
            "path": str(request.url)
        }
    )

@app.get("/users/{user_id}")
def get_user(user_id: int):
    if user_id <= 0:
        raise AppException(
            status_code=400,
            message="User ID must be positive",
            error_code="INVALID_USER_ID"
        )
    if user_id > 100:
        raise AppException(
            status_code=404,
            message=f"User {user_id} does not exist",
            error_code="USER_NOT_FOUND"
        )
    return {"id": user_id, "name": f"User_{user_id}"}
```

---

## L5 — Query Params Validation and Headers

### 16. Query Params with Query()
```python
from fastapi import FastAPI, Query
from typing import Optional, List

app = FastAPI()

@app.get("/items")
def search_items(
    q: Optional[str] = Query(None, min_length=3, max_length=50),
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    tags: List[str] = Query([])
):
    return {
        "query": q,
        "skip": skip,
        "limit": limit,
        "tags": tags
    }
```
> Test: `http://127.0.0.1:8000/items?q=phone&tags=new&tags=sale&limit=5`

---

### 17. Path Params with Path()
```python
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/users/{user_id}/orders/{order_id}")
def get_order(
    user_id: int = Path(..., ge=1, description="The ID of the user"),
    order_id: int = Path(..., ge=1, description="The ID of the order"),
):
    return {
        "user_id": user_id,
        "order_id": order_id,
        "status": "shipped"
    }
```

---

### 18. Request Headers
```python
from fastapi import FastAPI, Header
from typing import Optional

app = FastAPI()

@app.get("/protected")
def protected_route(
    x_api_key: Optional[str] = Header(None),
    user_agent: Optional[str] = Header(None)
):
    if x_api_key != "secret-key-123":
        return {"error": "Invalid or missing API key"}
    return {
        "message": "Access granted",
        "user_agent": user_agent
    }
```
> Test with curl: `curl -H "X-API-Key: secret-key-123" http://127.0.0.1:8000/protected`

---

### 19. Cookies
```python
from fastapi import FastAPI, Cookie, Response
from typing import Optional

app = FastAPI()

@app.post("/login")
def login(response: Response, username: str = "vishal"):
    response.set_cookie(key="session_user", value=username, httponly=True)
    return {"message": f"Logged in as {username}"}

@app.get("/me")
def get_me(session_user: Optional[str] = Cookie(None)):
    if not session_user:
        return {"error": "Not logged in"}
    return {"user": session_user}

@app.post("/logout")
def logout(response: Response):
    response.delete_cookie("session_user")
    return {"message": "Logged out"}
```

---

## L6 — Dependency Injection

### 20. Basic Dependency
```python
from fastapi import FastAPI, Depends
from typing import Optional

app = FastAPI()

def common_params(skip: int = 0, limit: int = 10, q: Optional[str] = None):
    return {"skip": skip, "limit": limit, "q": q}

@app.get("/users")
def list_users(params: dict = Depends(common_params)):
    return {"params": params, "data": "users list"}

@app.get("/items")
def list_items(params: dict = Depends(common_params)):
    return {"params": params, "data": "items list"}
```
> Key insight: one dependency, many routes. Change once, applies everywhere.

---

### 21. Dependency as Class
```python
from fastapi import FastAPI, Depends
from typing import Optional

app = FastAPI()

class PaginationParams:
    def __init__(self, skip: int = 0, limit: int = 10):
        self.skip = skip
        self.limit = limit

class SearchParams:
    def __init__(self, q: Optional[str] = None, active: bool = True):
        self.q = q
        self.active = active

@app.get("/products")
def list_products(
    pagination: PaginationParams = Depends(),
    search: SearchParams = Depends()
):
    return {
        "skip": pagination.skip,
        "limit": pagination.limit,
        "query": search.q,
        "active_only": search.active
    }
```

---

### 22. Dependency with Yield (like DB session)
```python
from fastapi import FastAPI, Depends
from typing import Generator

app = FastAPI()

# Simulating a DB connection
class FakeDB:
    def __init__(self):
        self.connected = True
        print("DB: connected")

    def query(self, sql: str):
        return f"Result of: {sql}"

    def close(self):
        self.connected = False
        print("DB: disconnected")

def get_db() -> Generator:
    db = FakeDB()
    try:
        yield db        # give DB to the route
    finally:
        db.close()      # always runs after the route, even on error

@app.get("/users")
def get_users(db: FakeDB = Depends(get_db)):
    result = db.query("SELECT * FROM users")
    return {"result": result}

@app.get("/items")
def get_items(db: FakeDB = Depends(get_db)):
    result = db.query("SELECT * FROM items")
    return {"result": result}
```

---

### 23. Chained Dependencies
```python
from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Optional

app = FastAPI()

# Dependency 1: Extract API key from header
def get_api_key(x_api_key: Optional[str] = Header(None)) -> str:
    if not x_api_key:
        raise HTTPException(status_code=401, detail="API key missing")
    return x_api_key

# Dependency 2: Validate API key (depends on #1)
def validate_api_key(api_key: str = Depends(get_api_key)) -> dict:
    valid_keys = {"admin-key": "admin", "user-key": "user"}
    if api_key not in valid_keys:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return {"key": api_key, "role": valid_keys[api_key]}

# Dependency 3: Check admin role (depends on #2)
def require_admin(user: dict = Depends(validate_api_key)) -> dict:
    if user["role"] != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return user

@app.get("/public")
def public_route(user: dict = Depends(validate_api_key)):
    return {"message": "Hello", "role": user["role"]}

@app.delete("/admin/nuke")
def admin_only(user: dict = Depends(require_admin)):
    return {"message": "Admin action executed", "by": user["key"]}
```

---

## L7 — Routers and App Structure

### 24. APIRouter — Split Routes
```python
# users.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter()

class User(BaseModel):
    name: str
    email: str

fake_users = {1: {"id": 1, "name": "Vishal", "email": "v@example.com"}}

@router.get("/")
def list_users():
    return list(fake_users.values())

@router.get("/{user_id}")
def get_user(user_id: int):
    if user_id not in fake_users:
        raise HTTPException(status_code=404, detail="User not found")
    return fake_users[user_id]

@router.post("/")
def create_user(user: User):
    uid = max(fake_users.keys()) + 1
    fake_users[uid] = {"id": uid, **user.model_dump()}
    return fake_users[uid]
```

```python
# items.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
def list_items():
    return [{"id": 1, "name": "Laptop"}, {"id": 2, "name": "Phone"}]

@router.get("/{item_id}")
def get_item(item_id: int):
    return {"id": item_id, "name": f"Item_{item_id}"}
```

```python
# main.py
from fastapi import FastAPI
import users, items

app = FastAPI()

app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(items.router, prefix="/items", tags=["items"])

@app.get("/")
def root():
    return {"message": "Welcome to the API"}
```

---

## L8 — Middleware and Background Tasks

### 25. Custom Middleware
```python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_process_time(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    response.headers["X-Process-Time"] = str(round(duration * 1000, 2)) + "ms"
    return response

@app.middleware("http")
async def log_requests(request: Request, call_next):
    print(f"[{request.method}] {request.url.path}")
    response = await call_next(request)
    print(f"[{request.method}] {request.url.path} → {response.status_code}")
    return response

@app.get("/users")
def get_users():
    return [{"id": 1, "name": "Vishal"}]
```

---

### 26. CORS Middleware
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/data")
def get_data():
    return {"data": "This is accessible from allowed origins"}
```

---

### 27. Background Tasks
```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
import time

app = FastAPI()

def send_email(email: str, message: str):
    """Simulates a slow email send — runs in background."""
    time.sleep(2)  # simulate delay
    print(f"Email sent to {email}: {message}")

def write_log(action: str, user: str):
    """Write to log file in background."""
    with open("actions.log", "a") as f:
        f.write(f"{action} by {user}\n")

class UserCreate(BaseModel):
    name: str
    email: str

@app.post("/users")
def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    # Response returns immediately
    # Email and log happen in background
    background_tasks.add_task(send_email, user.email, "Welcome to our platform!")
    background_tasks.add_task(write_log, "user_created", user.name)

    return {"message": "User created. Welcome email is being sent.", "name": user.name}
```

---

## L9 — Async Routes and Response Types

### 28. Sync vs Async Routes
```python
import asyncio
import httpx
from fastapi import FastAPI

app = FastAPI()

# Use async when doing I/O: DB calls, HTTP calls, file reads
@app.get("/async-data")
async def get_async():
    await asyncio.sleep(0.1)  # simulate async I/O
    return {"source": "async", "data": "fetched"}

# Use sync when doing CPU work or using sync libraries
@app.get("/sync-data")
def get_sync():
    result = sum(i * i for i in range(1000))  # CPU work
    return {"source": "sync", "result": result}

# Async HTTP call to external API
@app.get("/github/{username}")
async def get_github_user(username: str):
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"https://api.github.com/users/{username}")
        if resp.status_code == 200:
            data = resp.json()
            return {"login": data["login"], "repos": data["public_repos"]}
        return {"error": "User not found"}
```
> Install: `pip install httpx`

---

### 29. Response Types
```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse, PlainTextResponse, HTMLResponse, RedirectResponse
from pydantic import BaseModel

app = FastAPI()

@app.get("/json")
def json_response():
    return JSONResponse(
        content={"message": "Hello"},
        status_code=200,
        headers={"X-Custom-Header": "my-value"}
    )

@app.get("/text", response_class=PlainTextResponse)
def text_response():
    return "Hello, this is plain text"

@app.get("/html", response_class=HTMLResponse)
def html_response():
    return """
    <html>
        <body>
            <h1>Hello from FastAPI</h1>
            <p>This is HTML response</p>
        </body>
    </html>
    """

@app.get("/old-url")
def redirect():
    return RedirectResponse(url="/json", status_code=301)
```

---

### 30. Streaming Response
```python
import asyncio
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def generate_data():
    for i in range(1, 6):
        yield f"chunk {i}\n"
        await asyncio.sleep(0.5)

@app.get("/stream")
def stream():
    return StreamingResponse(generate_data(), media_type="text/plain")
```

---

## L10 — Full CRUD REST API (Build from Memory)

### 31. Complete Task API — Everything Together
```python
from fastapi import FastAPI, HTTPException, Depends, Query, status, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
from typing import Optional, List, Dict
import time

# ─── App ────────────────────────────────────────────────
app = FastAPI(title="Task API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# ─── Middleware ──────────────────────────────────────────
@app.middleware("http")
async def timer_middleware(request, call_next):
    start = time.time()
    response = await call_next(request)
    response.headers["X-Response-Time"] = f"{(time.time() - start)*1000:.2f}ms"
    return response

# ─── Schemas ─────────────────────────────────────────────
class TaskCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    priority: str = Field("medium", pattern="^(low|medium|high)$")
    done: bool = False

class TaskUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=1, max_length=200)
    priority: Optional[str] = Field(None, pattern="^(low|medium|high)$")
    done: Optional[bool] = None

class TaskOut(BaseModel):
    id: int
    title: str
    priority: str
    done: bool
    created_at: float

# ─── In-Memory DB ────────────────────────────────────────
tasks: Dict[int, dict] = {}
_counter = 0

# ─── Dependency ──────────────────────────────────────────
def get_task_or_404(task_id: int) -> dict:
    if task_id not in tasks:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Task {task_id} not found"
        )
    return tasks[task_id]

# ─── Background ──────────────────────────────────────────
def log_action(action: str, task_id: int):
    with open("task_log.txt", "a") as f:
        f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')} | {action} | task_id={task_id}\n")

# ─── Routes ──────────────────────────────────────────────
@app.get("/", tags=["health"])
def root():
    return {"status": "ok", "total_tasks": len(tasks)}

@app.post("/tasks", response_model=TaskOut, status_code=status.HTTP_201_CREATED, tags=["tasks"])
def create_task(task: TaskCreate, bg: BackgroundTasks):
    global _counter
    _counter += 1
    new_task = {
        "id": _counter,
        "title": task.title,
        "priority": task.priority,
        "done": task.done,
        "created_at": time.time()
    }
    tasks[_counter] = new_task
    bg.add_task(log_action, "CREATE", _counter)
    return new_task

@app.get("/tasks", response_model=List[TaskOut], tags=["tasks"])
def list_tasks(
    done: Optional[bool] = None,
    priority: Optional[str] = Query(None, pattern="^(low|medium|high)$"),
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
):
    result = list(tasks.values())
    if done is not None:
        result = [t for t in result if t["done"] == done]
    if priority:
        result = [t for t in result if t["priority"] == priority]
    return result[skip: skip + limit]

@app.get("/tasks/{task_id}", response_model=TaskOut, tags=["tasks"])
def get_task(task: dict = Depends(get_task_or_404)):
    return task

@app.put("/tasks/{task_id}", response_model=TaskOut, tags=["tasks"])
def replace_task(task_id: int, task: TaskCreate, bg: BackgroundTasks):
    existing = get_task_or_404(task_id)
    tasks[task_id].update({
        "title": task.title,
        "priority": task.priority,
        "done": task.done,
    })
    bg.add_task(log_action, "UPDATE", task_id)
    return tasks[task_id]

@app.patch("/tasks/{task_id}", response_model=TaskOut, tags=["tasks"])
def patch_task(task_id: int, updates: TaskUpdate, bg: BackgroundTasks):
    get_task_or_404(task_id)
    update_data = updates.model_dump(exclude_unset=True)
    tasks[task_id].update(update_data)
    bg.add_task(log_action, "PATCH", task_id)
    return tasks[task_id]

@app.delete("/tasks/{task_id}", status_code=status.HTTP_204_NO_CONTENT, tags=["tasks"])
def delete_task(task_id: int, bg: BackgroundTasks):
    get_task_or_404(task_id)
    del tasks[task_id]
    bg.add_task(log_action, "DELETE", task_id)

@app.post("/tasks/{task_id}/complete", response_model=TaskOut, tags=["tasks"])
def complete_task(task_id: int, bg: BackgroundTasks):
    get_task_or_404(task_id)
    tasks[task_id]["done"] = True
    bg.add_task(log_action, "COMPLETE", task_id)
    return tasks[task_id]
```
> This one exercise contains everything: models, validation, dependencies,
> status codes, filtering, pagination, PATCH partial update, background tasks,
> middleware, CORS. Build this from scratch on Day 3.

---

## Bonus Drills — Blank File Challenges

Write these from scratch without looking at anything.

### B1. Simple auth dependency
```
Write a dependency `require_token(token: str = Header(None))`
that raises 401 if token is missing or not equal to "secret"
```

### B2. Rate limiter middleware
```
Write middleware that tracks requests per IP (use a dict)
and returns 429 Too Many Requests if > 5 requests in 60 seconds
```

### B3. Paginated response model
```
Write a generic PaginatedResponse[T] model with:
items: List[T], total: int, skip: int, limit: int, has_next: bool
```

### B4. File upload endpoint
```
from fastapi import UploadFile, File
Write a POST /upload endpoint that accepts a file and returns
its filename, size, and content_type
```

### B5. Request validation error handler
```
from fastapi.exceptions import RequestValidationError
Write a custom handler that returns errors in this shape:
{"errors": [{"field": "name", "message": "field required"}]}
```

### B6. Health check with app startup/shutdown
```
Use @app.on_event("startup") and @app.on_event("shutdown")
to simulate DB connect and disconnect, store state in app.state
```

### B7. Versioned API with two routers
```
/api/v1/users → returns users as list
/api/v2/users → returns users wrapped in {"data": [...], "version": "v2"}
```

### B8. Build a Notes API
```
Note: id, title, content, tags (list of str), created_at
Full CRUD: POST, GET all (filter by tag), GET one, PUT, PATCH, DELETE
```

---

## Flask vs FastAPI — Quick Reference

| Flask | FastAPI |
|---|---|
| `@app.route("/", methods=["GET"])` | `@app.get("/")` |
| `request.json` | `item: ItemModel` in function param |
| `jsonify({...})` | `return {...}` |
| Manual validation | Pydantic does it automatically |
| No type hints needed | Type hints are the framework |
| `return jsonify(...), 201` | `status_code=201` in decorator |
| No auto docs | `/docs` and `/redoc` auto-generated |
| `abort(404)` | `raise HTTPException(status_code=404)` |
| `g` object for request state | `Depends()` for dependency injection |

---

## Progress Tracker

| Level | Topic                                  | Done? | Time Taken |
|-------|----------------------------------------|-------|------------|
| L1    | Routes, Path + Query Params            |  [ ]  |            |
| L2    | Pydantic Models, Request Body          |  [ ]  |            |
| L3    | In-Memory CRUD, PATCH                  |  [ ]  |            |
| L4    | Status Codes, HTTPException            |  [ ]  |            |
| L5    | Query/Path validation, Headers, Cookies|  [ ]  |            |
| L6    | Dependency Injection, Yield            |  [ ]  |            |
| L7    | Routers, App Structure                 |  [ ]  |            |
| L8    | Middleware, Background Tasks           |  [ ]  |            |
| L9    | Async, Response Types                  |  [ ]  |            |
| L10   | Full CRUD API — all concepts together  |  [ ]  |            |
| Bonus | Blank File Challenges                  |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 31 (the full Task API)
> completely from memory. No `any` shortcuts, no peeking. Run it. Hit `/docs`.
> Every route must work. That's your milestone — you're ready to build real APIs.
