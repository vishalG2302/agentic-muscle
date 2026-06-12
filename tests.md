markdown

# Testing Muscle Memory — 40+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `pytest filename.py -v`
> - Install: `pip install pytest pytest-asyncio httpx fastapi python-dotenv`
> - L1–L2 → pytest basics, assertions, fixtures (25–35 min)
> - L3–L4 → parametrize, mocking (30–40 min)
> - L5–L6 → testing FastAPI, async code (35–45 min)
> - L7–L8 → testing classes, LLM code (35–45 min)
> - L9–L10 → full test suite for a real app (50–60 min)

---

## The Mental Model — Before You Write a Line

```
Testing in 4 concepts:

1. WHAT to test
   Not implementation details — test BEHAVIOUR.
   "Given this input, I expect this output."
   Don't test HOW it does it. Test WHAT it does.

2. AAA Pattern — every test follows this
   Arrange → set up the data and state
   Act     → call the function under test
   Assert  → verify the result is correct

3. FAST tests
   Unit tests    → test one function in isolation (milliseconds)
   Integration   → test components working together (seconds)
   End-to-end    → test the whole system (minutes)
   Write mostly unit tests. They run fast and fail clearly.

4. MOCK external dependencies
   Your test should not call real APIs, real databases, real filesystems.
   Replace them with fakes that return predictable values.
   This makes tests fast, deterministic, and free.

The one rule:
   If your test can fail due to something outside your code
   (internet down, API rate limit, clock time) → you need a mock.
```

---

## L1 — pytest Basics

### 1. Your First Tests
```python
# test_01_basics.py

# Functions to test
def add(a: int, b: int) -> int:
    return a + b

def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def is_even(n: int) -> bool:
    return n % 2 == 0

def reverse_string(s: str) -> str:
    return s[::-1]

# Tests — pytest finds any function starting with test_
def test_add_positive_numbers():
    assert add(2, 3) == 5

def test_add_negative_numbers():
    assert add(-1, -1) == -2

def test_add_zero():
    assert add(5, 0) == 5

def test_divide_normal():
    assert divide(10, 2) == 5.0

def test_divide_float():
    assert abs(divide(10, 3) - 3.3333) < 0.001

def test_divide_by_zero_raises():
    import pytest
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)

def test_is_even_true():
    assert is_even(4) is True

def test_is_even_false():
    assert is_even(7) is False

def test_reverse_string():
    assert reverse_string("hello") == "olleh"

def test_reverse_empty_string():
    assert reverse_string("") == ""

def test_reverse_single_char():
    assert reverse_string("a") == "a"
```
> Run: `pytest test_01_basics.py -v`

---

### 2. Assertions Deep Dive
```python
# test_02_assertions.py
import pytest

def get_user(user_id: int) -> dict:
    users = {
        1: {"id": 1, "name": "Vishal", "role": "admin", "skills": ["Python", "LangGraph"]},
        2: {"id": 2, "name": "Alice",  "role": "viewer", "skills": ["Python"]},
    }
    if user_id not in users:
        return None
    return users[user_id]

def get_scores() -> list[int]:
    return [88, 95, 72, 61, 90]

# Equality
def test_user_name():
    user = get_user(1)
    assert user["name"] == "Vishal"

# None checks
def test_user_not_found_returns_none():
    assert get_user(99) is None

def test_user_found_is_not_none():
    assert get_user(1) is not None

# Membership
def test_user_has_python_skill():
    user = get_user(1)
    assert "Python" in user["skills"]

def test_langgraph_in_admin_skills():
    user = get_user(1)
    assert "LangGraph" in user["skills"]

# Collections
def test_user_has_two_skills():
    user = get_user(1)
    assert len(user["skills"]) == 2

def test_scores_count():
    assert len(get_scores()) == 5

# Numeric comparisons
def test_max_score():
    assert max(get_scores()) == 95

def test_min_score():
    assert min(get_scores()) >= 60

def test_average_score():
    scores = get_scores()
    avg = sum(scores) / len(scores)
    assert 70 < avg < 100

# Boolean
def test_admin_is_admin():
    user = get_user(1)
    assert user["role"] == "admin"

# Multiple assertions — use one per test, but sometimes this is ok
def test_user_1_complete():
    user = get_user(1)
    assert user is not None
    assert user["id"] == 1
    assert user["name"] == "Vishal"
    assert user["role"] == "admin"
```

---

## L2 — Fixtures

### 3. Basic Fixtures
```python
# test_03_fixtures.py
import pytest

class TaskManager:
    def __init__(self):
        self.tasks = {}
        self._counter = 0

    def create(self, title: str, priority: str = "medium") -> dict:
        self._counter += 1
        task = {"id": self._counter, "title": title, "priority": priority, "done": False}
        self.tasks[self._counter] = task
        return task

    def get(self, task_id: int) -> dict | None:
        return self.tasks.get(task_id)

    def complete(self, task_id: int) -> bool:
        if task_id not in self.tasks:
            return False
        self.tasks[task_id]["done"] = True
        return True

    def delete(self, task_id: int) -> bool:
        return self.tasks.pop(task_id, None) is not None

    def list_all(self) -> list[dict]:
        return list(self.tasks.values())

    def list_pending(self) -> list[dict]:
        return [t for t in self.tasks.values() if not t["done"]]


# Fixture — shared setup code, runs before each test that uses it
@pytest.fixture
def manager():
    """Fresh TaskManager for each test."""
    return TaskManager()

@pytest.fixture
def manager_with_tasks(manager):
    """TaskManager pre-loaded with 3 tasks."""
    manager.create("Task A", "high")
    manager.create("Task B", "medium")
    manager.create("Task C", "low")
    return manager


# Tests using fixtures
def test_create_task(manager):
    task = manager.create("Learn pytest")
    assert task["id"] == 1
    assert task["title"] == "Learn pytest"
    assert task["done"] is False

def test_get_existing_task(manager):
    manager.create("My task")
    task = manager.get(1)
    assert task is not None
    assert task["title"] == "My task"

def test_get_missing_task(manager):
    assert manager.get(999) is None

def test_complete_task(manager_with_tasks):
    result = manager_with_tasks.complete(1)
    assert result is True
    assert manager_with_tasks.get(1)["done"] is True

def test_complete_missing_task(manager):
    assert manager.complete(999) is False

def test_delete_task(manager_with_tasks):
    assert manager_with_tasks.delete(1) is True
    assert manager_with_tasks.get(1) is None

def test_list_all(manager_with_tasks):
    tasks = manager_with_tasks.list_all()
    assert len(tasks) == 3

def test_list_pending(manager_with_tasks):
    manager_with_tasks.complete(1)
    pending = manager_with_tasks.list_pending()
    assert len(pending) == 2
    assert all(not t["done"] for t in pending)

# Fixtures can be scoped
@pytest.fixture(scope="module")
def shared_data():
    """Created once per module — not per test."""
    return {"expensive": "setup data"}

def test_uses_shared(shared_data):
    assert "expensive" in shared_data
```

---

### 4. Setup and Teardown with Fixtures
```python
# test_04_setup_teardown.py
import pytest
import os
import json
import tempfile

@pytest.fixture
def temp_file():
    """Create a temp file, yield its path, delete after test."""
    with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False) as f:
        json.dump({"key": "value", "count": 0}, f)
        path = f.name

    yield path   # test runs here

    # Teardown — always runs, even if test fails
    if os.path.exists(path):
        os.remove(path)


@pytest.fixture
def temp_db():
    """In-memory dict acting as a fake DB."""
    db = {"users": {}, "tasks": {}}
    yield db
    db.clear()  # cleanup


def test_file_exists(temp_file):
    assert os.path.exists(temp_file)

def test_file_has_correct_content(temp_file):
    with open(temp_file) as f:
        data = json.load(f)
    assert data["key"] == "value"
    assert data["count"] == 0

def test_file_can_be_updated(temp_file):
    with open(temp_file) as f:
        data = json.load(f)
    data["count"] = 5
    with open(temp_file, "w") as f:
        json.dump(data, f)

    with open(temp_file) as f:
        updated = json.load(f)
    assert updated["count"] == 5

def test_db_starts_empty(temp_db):
    assert len(temp_db["users"]) == 0

def test_db_insert(temp_db):
    temp_db["users"][1] = {"name": "Vishal"}
    assert 1 in temp_db["users"]

def test_db_isolation(temp_db):
    # Previous test's insert is NOT here — fixture is fresh
    assert len(temp_db["users"]) == 0
```

---

## L3 — Parametrize

### 5. Parametrize — Test Many Inputs at Once
```python
# test_05_parametrize.py
import pytest

def classify_score(score: int) -> str:
    if score >= 90: return "A"
    if score >= 75: return "B"
    if score >= 60: return "C"
    if score >= 40: return "D"
    return "F"

def is_valid_email(email: str) -> bool:
    return "@" in email and "." in email.split("@")[-1]

def fizzbuzz(n: int) -> str:
    if n % 15 == 0: return "FizzBuzz"
    if n % 3 == 0:  return "Fizz"
    if n % 5 == 0:  return "Buzz"
    return str(n)

# Instead of writing 5 separate test functions:
@pytest.mark.parametrize("score, expected", [
    (95,  "A"),
    (90,  "A"),
    (80,  "B"),
    (75,  "B"),
    (65,  "C"),
    (60,  "C"),
    (50,  "D"),
    (40,  "D"),
    (39,  "F"),
    (0,   "F"),
])
def test_classify_score(score, expected):
    assert classify_score(score) == expected

@pytest.mark.parametrize("email, valid", [
    ("vishal@example.com",  True),
    ("user@domain.org",     True),
    ("notanemail",          False),
    ("missing@dot",         False),
    ("@nodomain.com",       True),  # technically passes our simple check
    ("",                    False),
])
def test_email_validation(email, valid):
    assert is_valid_email(email) == valid

@pytest.mark.parametrize("n, expected", [
    (1,  "1"),
    (3,  "Fizz"),
    (5,  "Buzz"),
    (15, "FizzBuzz"),
    (30, "FizzBuzz"),
    (9,  "Fizz"),
    (10, "Buzz"),
    (7,  "7"),
])
def test_fizzbuzz(n, expected):
    assert fizzbuzz(n) == expected
```

---

## L4 — Mocking

### 6. unittest.mock — Replace External Dependencies
```python
# test_06_mocking.py
import pytest
from unittest.mock import Mock, patch, MagicMock
from datetime import datetime

# ── Code under test ────────────────────────────────────

def get_current_hour() -> int:
    return datetime.now().hour

def greet_by_time() -> str:
    hour = get_current_hour()
    if hour < 12:   return "Good morning!"
    if hour < 18:   return "Good afternoon!"
    return "Good evening!"

def fetch_user_from_api(user_id: int) -> dict:
    """In real code this would call requests.get(...)"""
    import requests
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()

def send_email(to: str, subject: str, body: str) -> bool:
    """In real code this sends a real email."""
    import smtplib
    # ... real SMTP code ...
    return True

def process_user(user_id: int, email_service) -> dict:
    """Business logic that uses injected dependencies."""
    user = {"id": user_id, "name": f"User_{user_id}", "email": f"user{user_id}@test.com"}
    email_service.send(user["email"], "Welcome!", f"Hello {user['name']}")
    return {"processed": True, "user": user}


# ── Tests ──────────────────────────────────────────────

# Mock a function the code calls
def test_greet_morning():
    with patch("test_06_mocking.get_current_hour", return_value=9):
        assert greet_by_time() == "Good morning!"

def test_greet_afternoon():
    with patch("test_06_mocking.get_current_hour", return_value=14):
        assert greet_by_time() == "Good afternoon!"

def test_greet_evening():
    with patch("test_06_mocking.get_current_hour", return_value=20):
        assert greet_by_time() == "Good evening!"

# Mock an HTTP call
def test_fetch_user():
    mock_response = Mock()
    mock_response.json.return_value = {"id": 1, "name": "Vishal"}

    with patch("requests.get", return_value=mock_response):
        result = fetch_user_from_api(1)

    assert result["name"] == "Vishal"
    assert result["id"] == 1

# Use Mock as an injected dependency
def test_process_user_sends_email():
    mock_email = Mock()

    result = process_user(42, mock_email)

    # Assert the result is correct
    assert result["processed"] is True
    assert result["user"]["id"] == 42

    # Assert the mock was called correctly
    mock_email.send.assert_called_once()
    call_args = mock_email.send.call_args
    assert "user42@test.com" in str(call_args)
    assert "Welcome!" in str(call_args)

def test_process_user_email_args():
    mock_email = Mock()
    process_user(1, mock_email)

    # Verify exact arguments
    mock_email.send.assert_called_once_with(
        "user1@test.com",
        "Welcome!",
        "Hello User_1"
    )
```

---

### 7. Patch Decorators and MagicMock
```python
# test_07_patch_decorator.py
import pytest
from unittest.mock import patch, MagicMock, call
import json
import os

# ── Code under test ────────────────────────────────────

def save_user(user: dict, filepath: str) -> bool:
    with open(filepath, "w") as f:
        json.dump(user, f)
    return True

def load_and_process(filepath: str) -> dict:
    if not os.path.exists(filepath):
        raise FileNotFoundError(f"File {filepath} not found")
    with open(filepath) as f:
        data = json.load(f)
    data["processed"] = True
    return data

class NotificationService:
    def __init__(self, api_key: str):
        self.api_key = api_key

    def notify(self, user_id: int, message: str) -> dict:
        import requests
        response = requests.post(
            "https://notify.example.com/send",
            json={"user_id": user_id, "message": message},
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        return response.json()


# ── Tests ──────────────────────────────────────────────

# patch as decorator — cleaner for multiple tests
@patch("builtins.open", create=True)
@patch("json.dump")
def test_save_user(mock_dump, mock_open):
    mock_file = MagicMock()
    mock_open.return_value.__enter__.return_value = mock_file

    result = save_user({"id": 1, "name": "Vishal"}, "user.json")

    assert result is True
    mock_dump.assert_called_once()

@patch("os.path.exists", return_value=False)
def test_load_missing_file(mock_exists):
    with pytest.raises(FileNotFoundError):
        load_and_process("nonexistent.json")

@patch("requests.post")
def test_notification_service(mock_post):
    mock_response = MagicMock()
    mock_response.json.return_value = {"status": "sent", "id": "abc123"}
    mock_post.return_value = mock_response

    service = NotificationService(api_key="test-key")
    result = service.notify(user_id=1, message="Hello!")

    assert result["status"] == "sent"
    mock_post.assert_called_once()

    # Verify the call was made with correct args
    call_kwargs = mock_post.call_args.kwargs
    assert call_kwargs["json"]["user_id"] == 1
    assert "Bearer test-key" in call_kwargs["headers"]["Authorization"]
```

---

## L5 — Testing FastAPI

### 8. FastAPI TestClient
```python
# test_08_fastapi.py
import pytest
from fastapi import FastAPI, HTTPException
from fastapi.testclient import TestClient
from pydantic import BaseModel
from typing import Dict, Optional

# ── App under test ─────────────────────────────────────

app = FastAPI()

tasks: Dict[int, dict] = {}
_counter = {"n": 0}

class TaskCreate(BaseModel):
    title: str
    priority: str = "medium"

class TaskUpdate(BaseModel):
    title: Optional[str] = None
    done: Optional[bool] = None

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/tasks", status_code=201)
def create_task(task: TaskCreate):
    _counter["n"] += 1
    new_task = {"id": _counter["n"], "title": task.title,
                "priority": task.priority, "done": False}
    tasks[_counter["n"]] = new_task
    return new_task

@app.get("/tasks")
def list_tasks():
    return list(tasks.values())

@app.get("/tasks/{task_id}")
def get_task(task_id: int):
    if task_id not in tasks:
        raise HTTPException(status_code=404, detail="Task not found")
    return tasks[task_id]

@app.patch("/tasks/{task_id}")
def patch_task(task_id: int, updates: TaskUpdate):
    if task_id not in tasks:
        raise HTTPException(status_code=404, detail="Task not found")
    data = updates.model_dump(exclude_unset=True)
    tasks[task_id].update(data)
    return tasks[task_id]

@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int):
    if task_id not in tasks:
        raise HTTPException(status_code=404, detail="Task not found")
    del tasks[task_id]


# ── Fixtures ───────────────────────────────────────────

@pytest.fixture(autouse=True)
def clear_tasks():
    """Clear tasks before every test — ensures isolation."""
    tasks.clear()
    _counter["n"] = 0
    yield
    tasks.clear()
    _counter["n"] = 0

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def client_with_task(client):
    client.post("/tasks", json={"title": "Test task", "priority": "high"})
    return client


# ── Tests ──────────────────────────────────────────────

def test_health_check(client):
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}

def test_create_task(client):
    response = client.post("/tasks", json={"title": "My task"})
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "My task"
    assert data["done"] is False
    assert "id" in data

def test_create_task_with_priority(client):
    response = client.post("/tasks", json={"title": "Urgent", "priority": "high"})
    assert response.status_code == 201
    assert response.json()["priority"] == "high"

def test_list_tasks_empty(client):
    response = client.get("/tasks")
    assert response.status_code == 200
    assert response.json() == []

def test_list_tasks_with_data(client_with_task):
    response = client_with_task.get("/tasks")
    assert response.status_code == 200
    assert len(response.json()) == 1

def test_get_task_exists(client_with_task):
    response = client_with_task.get("/tasks/1")
    assert response.status_code == 200
    assert response.json()["title"] == "Test task"

def test_get_task_not_found(client):
    response = client.get("/tasks/999")
    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()

def test_patch_task_title(client_with_task):
    response = client_with_task.patch("/tasks/1", json={"title": "Updated title"})
    assert response.status_code == 200
    assert response.json()["title"] == "Updated title"

def test_patch_task_complete(client_with_task):
    response = client_with_task.patch("/tasks/1", json={"done": True})
    assert response.status_code == 200
    assert response.json()["done"] is True

def test_patch_missing_task(client):
    response = client.patch("/tasks/999", json={"done": True})
    assert response.status_code == 404

def test_delete_task(client_with_task):
    response = client_with_task.delete("/tasks/1")
    assert response.status_code == 204

def test_delete_then_get_returns_404(client_with_task):
    client_with_task.delete("/tasks/1")
    response = client_with_task.get("/tasks/1")
    assert response.status_code == 404

def test_delete_missing_task(client):
    response = client.delete("/tasks/999")
    assert response.status_code == 404
```

---

## L6 — Testing Async Code

### 9. pytest-asyncio
```python
# test_09_async.py
import pytest
import asyncio
from unittest.mock import AsyncMock, patch

# ── Async code under test ──────────────────────────────

async def fetch_data(url: str) -> dict:
    """Simulates an async HTTP call."""
    await asyncio.sleep(0.01)  # simulate I/O
    return {"url": url, "data": "some data"}

async def process_items(items: list[str]) -> list[dict]:
    """Process items concurrently."""
    tasks = [fetch_data(f"https://api.example.com/{item}") for item in items]
    return await asyncio.gather(*tasks)

async def retry_operation(fn, max_retries: int = 3) -> dict:
    """Retry an async operation up to max_retries times."""
    for attempt in range(max_retries):
        try:
            return await fn()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            await asyncio.sleep(0.01)

class AsyncTaskService:
    async def create(self, title: str) -> dict:
        await asyncio.sleep(0.01)  # simulate DB write
        return {"id": 1, "title": title, "done": False}

    async def get(self, task_id: int) -> dict | None:
        await asyncio.sleep(0.01)  # simulate DB read
        if task_id != 1:
            return None
        return {"id": 1, "title": "Test task", "done": False}


# ── Tests ──────────────────────────────────────────────

@pytest.mark.asyncio
async def test_fetch_data():
    result = await fetch_data("https://api.example.com/items")
    assert "url" in result
    assert "data" in result

@pytest.mark.asyncio
async def test_process_items():
    items = ["item1", "item2", "item3"]
    results = await process_items(items)
    assert len(results) == 3
    assert all("url" in r for r in results)

@pytest.mark.asyncio
async def test_async_service_create():
    service = AsyncTaskService()
    task = await service.create("Learn pytest async")
    assert task["title"] == "Learn pytest async"
    assert task["done"] is False

@pytest.mark.asyncio
async def test_async_service_get_existing():
    service = AsyncTaskService()
    task = await service.get(1)
    assert task is not None

@pytest.mark.asyncio
async def test_async_service_get_missing():
    service = AsyncTaskService()
    task = await service.get(999)
    assert task is None

@pytest.mark.asyncio
async def test_retry_succeeds_on_first_try():
    mock_fn = AsyncMock(return_value={"status": "ok"})
    result = await retry_operation(mock_fn)
    assert result["status"] == "ok"
    assert mock_fn.call_count == 1

@pytest.mark.asyncio
async def test_retry_succeeds_after_failures():
    call_count = {"n": 0}

    async def flaky():
        call_count["n"] += 1
        if call_count["n"] < 3:
            raise Exception("Temporary error")
        return {"status": "ok"}

    result = await retry_operation(flaky, max_retries=3)
    assert result["status"] == "ok"
    assert call_count["n"] == 3

@pytest.mark.asyncio
async def test_retry_raises_after_max():
    async def always_fails():
        raise ValueError("Always fails")

    with pytest.raises(ValueError, match="Always fails"):
        await retry_operation(always_fails, max_retries=3)
```
> Add `asyncio_mode = "auto"` to `pytest.ini` or `pyproject.toml`
> OR use `@pytest.mark.asyncio` on each async test.

---

## L7 — Testing Classes and State

### 10. Testing Stateful Classes
```python
# test_10_stateful.py
import pytest
from unittest.mock import Mock, patch

# ── Code under test ────────────────────────────────────

class ShoppingCart:
    def __init__(self):
        self.items: list[dict] = []
        self.discount: float = 0.0

    def add_item(self, name: str, price: float, qty: int = 1):
        if price <= 0:
            raise ValueError("Price must be positive")
        if qty <= 0:
            raise ValueError("Quantity must be positive")
        self.items.append({"name": name, "price": price, "qty": qty})

    def remove_item(self, name: str) -> bool:
        for i, item in enumerate(self.items):
            if item["name"] == name:
                self.items.pop(i)
                return True
        return False

    def apply_discount(self, percent: float):
        if not 0 <= percent <= 100:
            raise ValueError("Discount must be 0–100")
        self.discount = percent / 100

    def subtotal(self) -> float:
        return sum(item["price"] * item["qty"] for item in self.items)

    def total(self) -> float:
        return round(self.subtotal() * (1 - self.discount), 2)

    def item_count(self) -> int:
        return sum(item["qty"] for item in self.items)

    def clear(self):
        self.items = []
        self.discount = 0.0


# ── Fixtures ───────────────────────────────────────────

@pytest.fixture
def empty_cart():
    return ShoppingCart()

@pytest.fixture
def cart_with_items():
    cart = ShoppingCart()
    cart.add_item("Python Book", 29.99)
    cart.add_item("Keyboard", 89.99)
    cart.add_item("Mouse", 39.99, qty=2)
    return cart


# ── Tests ──────────────────────────────────────────────

def test_empty_cart_total(empty_cart):
    assert empty_cart.total() == 0.0

def test_empty_cart_item_count(empty_cart):
    assert empty_cart.item_count() == 0

def test_add_single_item(empty_cart):
    empty_cart.add_item("Laptop", 999.99)
    assert empty_cart.item_count() == 1

def test_add_item_with_qty(empty_cart):
    empty_cart.add_item("Pen", 1.99, qty=5)
    assert empty_cart.item_count() == 5

def test_total_calculation(cart_with_items):
    expected = 29.99 + 89.99 + (39.99 * 2)
    assert abs(cart_with_items.total() - expected) < 0.01

def test_discount_applied(cart_with_items):
    subtotal = cart_with_items.subtotal()
    cart_with_items.apply_discount(10)
    assert cart_with_items.total() == round(subtotal * 0.9, 2)

def test_remove_existing_item(cart_with_items):
    result = cart_with_items.remove_item("Keyboard")
    assert result is True
    assert cart_with_items.item_count() == 3   # 1 + 2

def test_remove_missing_item(cart_with_items):
    result = cart_with_items.remove_item("NonExistent")
    assert result is False

def test_invalid_price_raises(empty_cart):
    with pytest.raises(ValueError, match="Price must be positive"):
        empty_cart.add_item("Free item", -5.0)

def test_invalid_discount_raises(empty_cart):
    with pytest.raises(ValueError, match="Discount must be 0–100"):
        empty_cart.apply_discount(150)

def test_clear_resets_cart(cart_with_items):
    cart_with_items.clear()
    assert cart_with_items.item_count() == 0
    assert cart_with_items.total() == 0.0
    assert cart_with_items.discount == 0.0
```

---

## L8 — Testing LLM Code

### 11. Mock LLM Calls — Never Call Real APIs in Tests
```python
# test_11_llm_mocking.py
import pytest
from unittest.mock import Mock, patch, MagicMock

# ── Code under test ────────────────────────────────────

def classify_sentiment(text: str, llm) -> str:
    """Classify text sentiment using an LLM."""
    prompt = f"Classify as positive, negative, or neutral: '{text}'"
    response = llm.invoke(prompt)
    return response.content.strip().lower()

def extract_keywords(text: str, llm) -> list[str]:
    """Extract keywords from text using LLM."""
    import json
    prompt = f"Extract 3-5 keywords from: '{text}'. Return JSON array only."
    response = llm.invoke(prompt)
    try:
        return json.loads(response.content)
    except:
        return []

def rag_answer(question: str, retriever, llm) -> dict:
    """RAG pipeline: retrieve → generate → return with sources."""
    docs = retriever.invoke(question)
    context = "\n".join(d.page_content for d in docs)
    prompt = f"Context: {context}\n\nQuestion: {question}"
    response = llm.invoke(prompt)
    return {
        "answer": response.content,
        "sources": [d.metadata.get("source") for d in docs],
        "doc_count": len(docs)
    }


# ── Tests ──────────────────────────────────────────────

@pytest.fixture
def mock_llm():
    """A fake LLM that returns predictable responses."""
    llm = Mock()
    response = Mock()
    response.content = "positive"
    llm.invoke.return_value = response
    return llm

def test_classify_positive_sentiment(mock_llm):
    result = classify_sentiment("I love this product!", mock_llm)
    assert result == "positive"
    mock_llm.invoke.assert_called_once()

def test_classify_passes_text_to_llm(mock_llm):
    text = "This is great!"
    classify_sentiment(text, mock_llm)
    call_args = mock_llm.invoke.call_args[0][0]
    assert text in call_args

def test_extract_keywords():
    import json
    llm = Mock()
    llm.invoke.return_value = Mock(content='["python", "fastapi", "testing"]')

    result = extract_keywords("Testing FastAPI with Python", llm)

    assert "python" in result
    assert "fastapi" in result
    assert len(result) == 3

def test_extract_keywords_handles_invalid_json():
    llm = Mock()
    llm.invoke.return_value = Mock(content="not valid json")

    result = extract_keywords("Some text", llm)
    assert result == []

def test_rag_answer():
    from unittest.mock import MagicMock

    # Mock retriever
    doc1 = MagicMock()
    doc1.page_content = "FastAPI is a Python web framework."
    doc1.metadata = {"source": "fastapi_docs.md"}

    doc2 = MagicMock()
    doc2.page_content = "It has automatic validation."
    doc2.metadata = {"source": "fastapi_guide.md"}

    retriever = Mock()
    retriever.invoke.return_value = [doc1, doc2]

    # Mock LLM
    llm = Mock()
    llm.invoke.return_value = Mock(content="FastAPI is a web framework with auto validation.")

    result = rag_answer("What is FastAPI?", retriever, llm)

    assert result["answer"] == "FastAPI is a web framework with auto validation."
    assert result["doc_count"] == 2
    assert "fastapi_docs.md" in result["sources"]

def test_rag_answer_calls_retriever_with_question():
    retriever = Mock()
    retriever.invoke.return_value = []
    llm = Mock()
    llm.invoke.return_value = Mock(content="No context available.")

    rag_answer("What is LangGraph?", retriever, llm)

    retriever.invoke.assert_called_once_with("What is LangGraph?")
```

---

## L9 — Markers, Coverage, and Conftest

### 12. Markers and conftest.py
```python
# conftest.py (put this in the same directory)
import pytest

@pytest.fixture(scope="session")
def app_config():
    """Session-wide config — created once for all tests."""
    return {
        "db_url": "sqlite:///:memory:",
        "api_key": "test-key-12345",
        "debug": True
    }

@pytest.fixture
def sample_user():
    return {
        "id": 1,
        "name": "Vishal",
        "email": "vishal@test.com",
        "role": "admin"
    }

@pytest.fixture
def sample_tasks():
    return [
        {"id": 1, "title": "Task A", "priority": "high",   "done": False},
        {"id": 2, "title": "Task B", "priority": "medium", "done": True},
        {"id": 3, "title": "Task C", "priority": "low",    "done": False},
    ]
```

```python
# test_12_markers.py
import pytest

# Custom markers — register in pytest.ini:
# [pytest]
# markers =
#     slow: marks tests as slow
#     integration: marks integration tests

def filter_tasks(tasks: list, done: bool = None, priority: str = None) -> list:
    result = tasks
    if done is not None:
        result = [t for t in result if t["done"] == done]
    if priority:
        result = [t for t in result if t["priority"] == priority]
    return result

def summarise_user(user: dict) -> str:
    return f"{user['name']} ({user['role']}) "

# Use fixtures from conftest.py
def test_filter_pending(sample_tasks):
    pending = filter_tasks(sample_tasks, done=False)
    assert len(pending) == 2
    assert all(not t["done"] for t in pending)

def test_filter_high_priority(sample_tasks):
    high = filter_tasks(sample_tasks, priority="high")
    assert len(high) == 1
    assert high[0]["title"] == "Task A"

def test_filter_combined(sample_tasks):
    pending_low = filter_tasks(sample_tasks, done=False, priority="low")
    assert len(pending_low) == 1

def test_summarise_user(sample_user):
    result = summarise_user(sample_user)
    assert "Vishal" in result
    assert "admin" in result
    assert "vishal@test.com" in result

@pytest.mark.skip(reason="Feature not implemented yet")
def test_future_feature():
    assert False

@pytest.mark.xfail(reason="Known bug — fix in next sprint")
def test_known_bug():
    assert 1 == 2  # this will fail but test won't be counted as failure

@pytest.mark.skipif(
    condition=True,
    reason="Skipped in CI environment"
)
def test_local_only():
    pass
```

---

## L10 — Full Test Suite

### 13. Complete Test Suite for a Real App
```python
# test_13_full_suite.py
import pytest
from fastapi import FastAPI, HTTPException, Depends
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
from typing import Optional, Dict, List
from unittest.mock import Mock, patch
import time

# ── Application ────────────────────────────────────────

app = FastAPI(title="Task API")

tasks: Dict[int, dict] = {}
_id = {"n": 0}

class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    priority: str = Field(default="medium", pattern="^(low|medium|high)$")

class TaskUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=1)
    priority: Optional[str] = Field(None, pattern="^(low|medium|high)$")
    done: Optional[bool] = None

def get_db():
    return tasks

@app.get("/")
def root():
    return {"status": "ok", "task_count": len(tasks)}

@app.post("/tasks", status_code=201)
def create(task: TaskCreate, db: dict = Depends(get_db)):
    _id["n"] += 1
    new_task = {
        "id": _id["n"],
        "title": task.title,
        "priority": task.priority,
        "done": False,
        "created_at": time.time()
    }
    db[_id["n"]] = new_task
    return new_task

@app.get("/tasks")
def list_tasks(
    done: Optional[bool] = None,
    priority: Optional[str] = None,
    db: dict = Depends(get_db)
):
    result = list(db.values())
    if done is not None:
        result = [t for t in result if t["done"] == done]
    if priority:
        result = [t for t in result if t["priority"] == priority]
    return result

@app.get("/tasks/{task_id}")
def get_task(task_id: int, db: dict = Depends(get_db)):
    if task_id not in db:
        raise HTTPException(404, f"Task {task_id} not found")
    return db[task_id]

@app.patch("/tasks/{task_id}")
def update_task(task_id: int, updates: TaskUpdate, db: dict = Depends(get_db)):
    if task_id not in db:
        raise HTTPException(404, f"Task {task_id} not found")
    update_data = updates.model_dump(exclude_unset=True)
    db[task_id].update(update_data)
    return db[task_id]

@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int, db: dict = Depends(get_db)):
    if task_id not in db:
        raise HTTPException(404, f"Task {task_id} not found")
    del db[task_id]

@app.post("/tasks/{task_id}/complete")
def complete_task(task_id: int, db: dict = Depends(get_db)):
    if task_id not in db:
        raise HTTPException(404, f"Task {task_id} not found")
    db[task_id]["done"] = True
    return db[task_id]


# ── Fixtures ───────────────────────────────────────────

@pytest.fixture(autouse=True)
def reset_state():
    tasks.clear()
    _id["n"] = 0
    yield
    tasks.clear()
    _id["n"] = 0

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def seeded_client(client):
    client.post("/tasks", json={"title": "High priority task", "priority": "high"})
    client.post("/tasks", json={"title": "Medium task", "priority": "medium"})
    client.post("/tasks", json={"title": "Low priority task", "priority": "low"})
    return client


# ── Health check ───────────────────────────────────────

class TestHealth:
    def test_root_returns_ok(self, client):
        assert client.get("/").status_code == 200

    def test_root_returns_task_count(self, client):
        assert client.get("/").json()["task_count"] == 0


# ── Create ─────────────────────────────────────────────

class TestCreate:
    def test_returns_201(self, client):
        r = client.post("/tasks", json={"title": "Test"})
        assert r.status_code == 201

    def test_returns_task_with_id(self, client):
        r = client.post("/tasks", json={"title": "Test"})
        assert "id" in r.json()
        assert r.json()["id"] == 1

    def test_default_priority_is_medium(self, client):
        r = client.post("/tasks", json={"title": "Test"})
        assert r.json()["priority"] == "medium"

    def test_invalid_priority_returns_422(self, client):
        r = client.post("/tasks", json={"title": "Test", "priority": "urgent"})
        assert r.status_code == 422

    def test_empty_title_returns_422(self, client):
        r = client.post("/tasks", json={"title": ""})
        assert r.status_code == 422

    def test_done_defaults_false(self, client):
        r = client.post("/tasks", json={"title": "Test"})
        assert r.json()["done"] is False


# ── Read ───────────────────────────────────────────────

class TestRead:
    def test_list_empty(self, client):
        assert client.get("/tasks").json() == []

    def test_list_returns_all(self, seeded_client):
        r = seeded_client.get("/tasks")
        assert len(r.json()) == 3

    def test_filter_by_priority(self, seeded_client):
        r = seeded_client.get("/tasks?priority=high")
        assert len(r.json()) == 1
        assert r.json()[0]["priority"] == "high"

    def test_filter_by_done(self, seeded_client):
        seeded_client.post("/tasks/1/complete")
        r = seeded_client.get("/tasks?done=true")
        assert len(r.json()) == 1

    def test_get_by_id(self, seeded_client):
        r = seeded_client.get("/tasks/1")
        assert r.status_code == 200
        assert r.json()["id"] == 1

    def test_get_missing_returns_404(self, client):
        assert client.get("/tasks/999").status_code == 404


# ── Update ─────────────────────────────────────────────

class TestUpdate:
    def test_patch_title(self, seeded_client):
        r = seeded_client.patch("/tasks/1", json={"title": "Updated"})
        assert r.json()["title"] == "Updated"

    def test_patch_only_updates_sent_fields(self, seeded_client):
        original = seeded_client.get("/tasks/1").json()
        seeded_client.patch("/tasks/1", json={"done": True})
        updated = seeded_client.get("/tasks/1").json()
        assert updated["done"] is True
        assert updated["title"] == original["title"]   # unchanged

    def test_patch_missing_returns_404(self, client):
        assert client.patch("/tasks/999", json={"done": True}).status_code == 404

    def test_complete_endpoint(self, seeded_client):
        r = seeded_client.post("/tasks/1/complete")
        assert r.status_code == 200
        assert r.json()["done"] is True


# ── Delete ─────────────────────────────────────────────

class TestDelete:
    def test_delete_returns_204(self, seeded_client):
        assert seeded_client.delete("/tasks/1").status_code == 204

    def test_deleted_task_not_retrievable(self, seeded_client):
        seeded_client.delete("/tasks/1")
        assert seeded_client.get("/tasks/1").status_code == 404

    def test_delete_missing_returns_404(self, client):
        assert client.delete("/tasks/999").status_code == 404

    def test_list_shrinks_after_delete(self, seeded_client):
        seeded_client.delete("/tasks/1")
        assert len(seeded_client.get("/tasks").json()) == 2
```

---

## Bonus Drills

### B1. Write tests for a Stack class
```
Stack with push, pop, peek, is_empty, size
Test: empty stack, push/pop order, peek doesn't remove, underflow raises
```

### B2. Write tests for a password validator
```
Rules: min 8 chars, at least 1 uppercase, 1 number, 1 special char
Use parametrize with (password, is_valid) pairs — at least 10 cases
```

### B3. Write tests for a rate limiter
```
RateLimiter(max_calls=3, window_seconds=60)
Test: within limit passes, over limit raises, window resets
Mock time.time() to control the clock
```

### B4. Write tests for an async FastAPI endpoint
```
Async endpoint that calls an async service
Mock the service with AsyncMock
Test success, not found, and service error cases
```

### B5. Write a test that checks a whole workflow
```
Create task → complete task → verify in list_pending it's gone →
verify in list_all it's still there with done=True
```

### B6. Full test suite from scratch in one blank file
```
Build test_13_full_suite.py from memory:
App + fixtures + reset state + TestCreate + TestRead + TestUpdate + TestDelete
All tests must pass. No peeking.
```

---

## Testing Cheat Sheet

```python
import pytest
from unittest.mock import Mock, AsyncMock, patch, MagicMock
from fastapi.testclient import TestClient

# Fixtures
@pytest.fixture
def thing():
    obj = MyClass()
    yield obj    # teardown code goes after yield

@pytest.fixture(autouse=True)  # runs for every test in scope
def reset():
    db.clear()
    yield
    db.clear()

# Parametrize
@pytest.mark.parametrize("input, expected", [
    (1, "one"),
    (2, "two"),
])
def test_fn(input, expected):
    assert fn(input) == expected

# Expect exception
def test_raises():
    with pytest.raises(ValueError, match="must be positive"):
        fn(-1)

# Mock
mock = Mock()
mock.method.return_value = {"key": "value"}
mock.method.assert_called_once_with(arg1, arg2)
mock.method.assert_called_once()

# Patch
with patch("module.function", return_value=42) as mock_fn:
    result = code_that_calls_function()
    mock_fn.assert_called_once()

@patch("module.function", return_value=42)
def test_with_patch(mock_fn):
    ...

# Async mock
async_mock = AsyncMock(return_value={"data": "value"})

# FastAPI testing
client = TestClient(app)
response = client.post("/endpoint", json={"key": "value"})
assert response.status_code == 201
assert response.json()["key"] == "value"
```

---

## Progress Tracker

|
 Level 
|
 Topic                                    
|
 Done? 
|
 Time Taken 
|
|
-------
|
------------------------------------------
|
-------
|
------------
|
|
 L1    
|
 First tests, assertions                  
|
  [ ]  
|
|
|
 L2    
|
 Assertions deep dive                     
|
  [ ]  
|
|
|
 L3    
|
 Fixtures, setup/teardown                 
|
  [ ]  
|
|
|
 L4    
|
 Parametrize                              
|
  [ ]  
|
|
|
 L5    
|
 Mocking with unittest.mock               
|
  [ ]  
|
|
|
 L6    
|
 Patch decorators, MagicMock              
|
  [ ]  
|
|
|
 L7    
|
 Testing FastAPI with TestClient          
|
  [ ]  
|
|
|
 L8    
|
 Testing async code with pytest-asyncio   
|
  [ ]  
|
|
|
 L9    
|
 Testing stateful classes, LLM mocking    
|
  [ ]  
|
|
|
 L10   
|
 Full test suite — classes + markers      
|
  [ ]  
|
|
|
 Bonus 
|
 Stack, rate limiter, workflow tests       
|
  [ ]  
|
|

---

> **Final challenge:** Open a blank file. Build test_13_full_suite.py from memory —
> the FastAPI app + reset fixture + TestCreate + TestRead + TestUpdate + TestDelete.
> Run `pytest test_13_full_suite.py -v`. Every test must be green.
> When you see 25+ passing tests in your terminal — you write tests properly.