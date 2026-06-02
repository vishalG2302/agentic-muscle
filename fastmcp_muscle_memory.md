# FastMCP Muscle Memory — 40+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Install: `pip install fastmcp python-dotenv httpx`
> - Run a server: `python server.py` (stdio mode, for Claude Desktop)
> - Run with HTTP: `python server.py --transport sse --port 8000`
> - Test tools directly: `fastmcp dev server.py` (opens inspector UI)
> - L1–L2 → first server, tools (20–30 min)
> - L3–L4 → resources, prompts (25–30 min)
> - L5–L6 → context, dependencies, auth (30–40 min)
> - L7–L8 → composition, clients (30–40 min)
> - L9–L10 → full MCP servers from scratch (50–60 min)

---

## The Core Mental Model — Before You Write a Line

```
MCP (Model Context Protocol) = a standard way for AI models to talk to external tools

FastMCP Server exposes 3 things to the LLM:
  Tools     → functions the LLM can CALL     (@mcp.tool)
  Resources → data the LLM can READ          (@mcp.resource)
  Prompts   → reusable prompt templates      (@mcp.prompt)

Transport modes:
  stdio     → Claude Desktop reads/writes via stdin/stdout (local tools)
  SSE       → HTTP server the LLM hits over the network
  in-memory → testing without a real server
```

Burn this in. Everything else is just filling in the blanks.

---

## L1 — First Server and First Tool

### 1. Simplest Possible MCP Server
```python
# 01_hello_mcp.py
from fastmcp import FastMCP

# Create the server — give it a name
mcp = FastMCP("Hello MCP")

# Register a tool — the LLM can call this
@mcp.tool()
def greet(name: str) -> str:
    """Greet a person by name."""
    return f"Hello, {name}! Welcome to FastMCP."

# Run it
if __name__ == "__main__":
    mcp.run()  # default: stdio transport
```
> Test: `fastmcp dev 01_hello_mcp.py`
> This opens the MCP Inspector — you can call `greet` manually.

---

### 2. Multiple Tools
```python
# 02_multiple_tools.py
from fastmcp import FastMCP

mcp = FastMCP("Calculator MCP")

@mcp.tool()
def add(a: float, b: float) -> float:
    """Add two numbers together."""
    return a + b

@mcp.tool()
def subtract(a: float, b: float) -> float:
    """Subtract b from a."""
    return a - b

@mcp.tool()
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

@mcp.tool()
def divide(a: float, b: float) -> float:
    """Divide a by b. Raises error if b is zero."""
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return round(a / b, 6)

@mcp.tool()
def power(base: float, exponent: float) -> float:
    """Raise base to the power of exponent."""
    return base ** exponent

if __name__ == "__main__":
    mcp.run()
```

---

### 3. Tool Return Types
```python
# 03_return_types.py
from fastmcp import FastMCP
from typing import List, Dict, Any

mcp = FastMCP("Return Types MCP")

@mcp.tool()
def get_string() -> str:
    """Returns a plain string."""
    return "Hello from MCP!"

@mcp.tool()
def get_number() -> float:
    """Returns a number."""
    return 3.14159

@mcp.tool()
def get_list() -> List[str]:
    """Returns a list of items."""
    return ["apple", "banana", "cherry"]

@mcp.tool()
def get_dict() -> Dict[str, Any]:
    """Returns a dictionary with mixed types."""
    return {
        "name": "Vishal",
        "age": 22,
        "skills": ["Python", "LangGraph", "Neo4j"],
        "active": True
    }

@mcp.tool()
def get_status() -> Dict[str, Any]:
    """Returns a status object."""
    return {
        "status": "ok",
        "version": "1.0.0",
        "tools_count": 5
    }

if __name__ == "__main__":
    mcp.run()
```

---

## L2 — Tools with Validation and Error Handling

### 4. Input Validation with Pydantic
```python
# 04_validation.py
from fastmcp import FastMCP
from pydantic import Field
from typing import Annotated

mcp = FastMCP("Validated MCP")

@mcp.tool()
def create_user(
    name: Annotated[str, Field(min_length=2, max_length=50, description="User's full name")],
    age: Annotated[int, Field(ge=1, le=120, description="User's age in years")],
    email: Annotated[str, Field(pattern=r"^[\w.-]+@[\w.-]+\.\w+$", description="Valid email address")],
    role: Annotated[str, Field(description="User role")] = "viewer"
) -> Dict:
    """Create a new user with validated inputs."""
    return {
        "id": 1,
        "name": name,
        "age": age,
        "email": email,
        "role": role,
        "created": True
    }

@mcp.tool()
def set_priority(
    task_id: Annotated[int, Field(ge=1, description="Task ID")],
    priority: Annotated[str, Field(description="Priority level: low, medium, or high")]
) -> str:
    """Set priority for a task."""
    valid = {"low", "medium", "high"}
    if priority not in valid:
        raise ValueError(f"Priority must be one of: {', '.join(valid)}")
    return f"Task {task_id} priority set to {priority}"

from typing import Dict

if __name__ == "__main__":
    mcp.run()
```

---

### 5. Error Handling in Tools
```python
# 05_error_handling.py
from fastmcp import FastMCP
from typing import Optional
import json

mcp = FastMCP("Error Handling MCP")

# In-memory store
tasks: dict = {}
counter = {"value": 0}

@mcp.tool()
def create_task(title: str, priority: str = "medium") -> dict:
    """Create a new task. Priority must be low, medium, or high."""
    if not title.strip():
        raise ValueError("Task title cannot be empty")
    if priority not in ("low", "medium", "high"):
        raise ValueError(f"Invalid priority '{priority}'. Choose: low, medium, high")

    counter["value"] += 1
    task_id = counter["value"]
    tasks[task_id] = {"id": task_id, "title": title, "priority": priority, "done": False}
    return tasks[task_id]

@mcp.tool()
def get_task(task_id: int) -> dict:
    """Get a task by ID. Raises error if not found."""
    if task_id not in tasks:
        raise KeyError(f"Task {task_id} not found. Available IDs: {list(tasks.keys())}")
    return tasks[task_id]

@mcp.tool()
def parse_json(json_string: str) -> dict:
    """Parse a JSON string and return it as a dict."""
    try:
        return json.loads(json_string)
    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON: {e}")

@mcp.tool()
def safe_divide(numerator: float, denominator: float) -> dict:
    """Safely divide two numbers, returning result or error message."""
    if denominator == 0:
        return {"success": False, "error": "Division by zero", "result": None}
    return {"success": True, "error": None, "result": round(numerator / denominator, 6)}

if __name__ == "__main__":
    mcp.run()
```

---

### 6. Tools that Call External Services
```python
# 06_external_calls.py
from fastmcp import FastMCP
import httpx
import json

mcp = FastMCP("External MCP")

@mcp.tool()
def get_github_user(username: str) -> dict:
    """Fetch a GitHub user's public profile information."""
    url = f"https://api.github.com/users/{username}"
    response = httpx.get(url, headers={"User-Agent": "FastMCP-drill"})
    if response.status_code == 404:
        raise ValueError(f"GitHub user '{username}' not found")
    if response.status_code != 200:
        raise RuntimeError(f"GitHub API error: {response.status_code}")
    data = response.json()
    return {
        "login": data["login"],
        "name": data.get("name"),
        "bio": data.get("bio"),
        "public_repos": data["public_repos"],
        "followers": data["followers"],
        "following": data["following"],
        "created_at": data["created_at"]
    }

@mcp.tool()
def get_random_fact() -> str:
    """Get a random fact from the useless facts API."""
    response = httpx.get("https://uselessfacts.jsph.pl/api/v2/facts/random")
    if response.status_code != 200:
        raise RuntimeError("Failed to fetch fact")
    return response.json()["text"]

if __name__ == "__main__":
    mcp.run()
```

---

## L3 — Resources

### 7. Static Resources
```python
# 07_static_resources.py
from fastmcp import FastMCP

mcp = FastMCP("Resources MCP")

# Resources = data the LLM can READ (not call like a function)
# Think of them like files or database records the LLM can fetch

@mcp.resource("config://app")
def get_app_config() -> dict:
    """Application configuration."""
    return {
        "name": "My App",
        "version": "1.0.0",
        "debug": False,
        "max_connections": 100,
        "allowed_origins": ["http://localhost:3000"]
    }

@mcp.resource("data://readme")
def get_readme() -> str:
    """Application README content."""
    return """# My FastMCP App

This server provides tools for task management and data analysis.

## Available Tools
- create_task: Create a new task
- get_task: Retrieve a task by ID
- list_tasks: List all tasks

## Usage
Connect this server to Claude Desktop or any MCP-compatible client.
"""

@mcp.resource("data://sample-users")
def get_sample_users() -> list:
    """Sample user data for testing."""
    return [
        {"id": 1, "name": "Vishal", "role": "admin"},
        {"id": 2, "name": "Alice", "role": "editor"},
        {"id": 3, "name": "Bob", "role": "viewer"},
    ]

if __name__ == "__main__":
    mcp.run()
```

---

### 8. Dynamic Resources with Parameters
```python
# 08_dynamic_resources.py
from fastmcp import FastMCP
from typing import Optional

mcp = FastMCP("Dynamic Resources MCP")

# Fake database
users_db = {
    1: {"id": 1, "name": "Vishal", "role": "admin", "email": "v@example.com"},
    2: {"id": 2, "name": "Alice", "role": "editor", "email": "a@example.com"},
    3: {"id": 3, "name": "Bob", "role": "viewer", "email": "b@example.com"},
}

tasks_db = {
    1: {"id": 1, "title": "Build MCP server", "owner_id": 1, "done": False},
    2: {"id": 2, "title": "Write docs", "owner_id": 2, "done": True},
    3: {"id": 3, "title": "Deploy to prod", "owner_id": 1, "done": False},
}

# Dynamic resource — URI template with {user_id}
@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: int) -> dict:
    """Get a user's profile by ID."""
    if user_id not in users_db:
        raise ValueError(f"User {user_id} not found")
    return users_db[user_id]

@mcp.resource("users://{user_id}/tasks")
def get_user_tasks(user_id: int) -> list:
    """Get all tasks belonging to a specific user."""
    if user_id not in users_db:
        raise ValueError(f"User {user_id} not found")
    return [t for t in tasks_db.values() if t["owner_id"] == user_id]

@mcp.resource("tasks://{task_id}")
def get_task_resource(task_id: int) -> dict:
    """Get a task by ID as a resource."""
    if task_id not in tasks_db:
        raise ValueError(f"Task {task_id} not found")
    return tasks_db[task_id]

@mcp.resource("data://stats")
def get_stats() -> dict:
    """Overall statistics."""
    return {
        "total_users": len(users_db),
        "total_tasks": len(tasks_db),
        "completed_tasks": sum(1 for t in tasks_db.values() if t["done"]),
        "pending_tasks": sum(1 for t in tasks_db.values() if not t["done"]),
    }

if __name__ == "__main__":
    mcp.run()
```

---

### 9. File-Based Resources
```python
# 09_file_resources.py
from fastmcp import FastMCP
import os
import json

mcp = FastMCP("File Resources MCP")

# Create some test files first
os.makedirs("data", exist_ok=True)

with open("data/config.json", "w") as f:
    json.dump({"env": "development", "debug": True, "port": 8000}, f, indent=2)

with open("data/notes.txt", "w") as f:
    f.write("LangGraph uses nodes and edges.\nFastMCP exposes tools and resources.\nCrewAI uses agents and tasks.")

@mcp.resource("file://config")
def read_config() -> dict:
    """Read the application config JSON file."""
    with open("data/config.json") as f:
        return json.load(f)

@mcp.resource("file://notes")
def read_notes() -> str:
    """Read the notes text file."""
    with open("data/notes.txt") as f:
        return f.read()

@mcp.resource("file://logs/{filename}")
def read_log(filename: str) -> str:
    """Read a log file by name. Only .log files allowed."""
    if not filename.endswith(".log"):
        raise ValueError("Only .log files can be read")
    path = os.path.join("data", filename)
    if not os.path.exists(path):
        raise FileNotFoundError(f"Log file '{filename}' not found")
    with open(path) as f:
        return f.read()

if __name__ == "__main__":
    mcp.run()
```

---

## L4 — Prompts

### 10. Reusable Prompt Templates
```python
# 10_prompts.py
from fastmcp import FastMCP
from fastmcp.prompts import Message

mcp = FastMCP("Prompts MCP")

# Prompts = reusable conversation starters / templates
# The LLM uses these as pre-built instructions or conversation seeds

@mcp.prompt()
def code_review_prompt(language: str, code: str) -> str:
    """Generate a code review prompt for any language."""
    return f"""Please review the following {language} code:

{language}
{code}


Focus on:
1. Code quality and readability
2. Potential bugs or edge cases
3. Performance improvements
4. Best practices for {language}

Provide specific, actionable feedback."""

@mcp.prompt()
def explain_error_prompt(error_message: str, context: str = "") -> str:
    """Generate a prompt to explain an error message."""
    ctx = f"\nContext:\n{context}" if context else ""
    return f"""I encountered this error:
{error_message}
{ctx}

Please:
1. Explain what this error means in plain English
2. Identify the most likely cause
3. Provide a step-by-step fix
4. Show a corrected code example if applicable"""


@mcp.prompt()
def debug_conversation(problem: str, language: str = "Python") -> list[Message]:
    """Start a debugging conversation with system context."""
    return [
        Message(
            role="user",
            content=f"I'm having a problem with my {language} code: {problem}"
        )
    ]

if __name__ == "__main__":
    mcp.run()
```

---

## L5 — Context and Logging

### 11. Using Context in Tools
```python
# 11_context.py
from fastmcp import FastMCP, Context
from typing import Optional

mcp = FastMCP("Context MCP")

@mcp.tool()
async def process_with_logging(
    data: str,
    steps: int = 3,
    ctx: Context = None
) -> dict:
    """Process data with progress logging via context."""

    await ctx.info(f"Starting processing of '{data}' in {steps} steps")

    results = []
    for i in range(1, steps + 1):
        # Report progress to the client
        await ctx.report_progress(i, steps)
        await ctx.debug(f"Step {i}/{steps}: processing...")

        # Simulate work
        result = f"step_{i}_result_of_{data}"
        results.append(result)

    await ctx.info("Processing complete")
    return {"input": data, "steps_completed": steps, "results": results}

@mcp.tool()
async def validate_with_context(value: int, ctx: Context = None) -> str:
    """Validate a value and log warnings for edge cases."""
    if value < 0:
        await ctx.warning(f"Negative value received: {value}")
        return f"Warning: negative value {value} accepted but unusual"
    if value > 1000:
        await ctx.warning(f"Large value received: {value}")
        return f"Warning: large value {value} may cause performance issues"

    await ctx.info(f"Value {value} is valid")
    return f"Value {value} is valid"

@mcp.tool()
async def read_resource_in_tool(resource_uri: str, ctx: Context = None) -> str:
    """Read a resource from within a tool using context."""
    await ctx.info(f"Fetching resource: {resource_uri}")
    # Tools can read resources using ctx.read_resource
    resource = await ctx.read_resource(resource_uri)
    return f"Resource content: {str(resource)[:200]}"

if __name__ == "__main__":
    mcp.run()
```

---

## L6 — Server Configuration and Transport

### 12. Server with SSE Transport (HTTP)
```python
# 12_sse_server.py
from fastmcp import FastMCP
from typing import Dict

mcp = FastMCP(
    name="HTTP MCP Server",
    # Server-level settings
)

tasks: Dict[int, dict] = {}
_id = {"n": 0}

@mcp.tool()
def create_task(title: str, priority: str = "medium") -> dict:
    """Create a task."""
    _id["n"] += 1
    tasks[_id["n"]] = {"id": _id["n"], "title": title, "priority": priority, "done": False}
    return tasks[_id["n"]]

@mcp.tool()
def list_tasks() -> list:
    """List all tasks."""
    return list(tasks.values())

@mcp.tool()
def complete_task(task_id: int) -> dict:
    """Mark a task as done."""
    if task_id not in tasks:
        raise ValueError(f"Task {task_id} not found")
    tasks[task_id]["done"] = True
    return tasks[task_id]

@mcp.resource("data://tasks")
def get_all_tasks() -> list:
    """Resource: all tasks in the store."""
    return list(tasks.values())

if __name__ == "__main__":
    import sys
    transport = sys.argv[1] if len(sys.argv) > 1 else "stdio"
    if transport == "sse":
        mcp.run(transport="sse", host="0.0.0.0", port=8000)
    else:
        mcp.run()

# Run with: python 12_sse_server.py sse
# MCP endpoint: http://localhost:8000/sse
```

---

### 13. Server with Dependencies (Lifespan)
```python
# 13_lifespan.py
from fastmcp import FastMCP
from contextlib import asynccontextmanager
from typing import AsyncIterator
import asyncio

# Simulate a DB connection that needs setup/teardown
class FakeDatabase:
    def __init__(self):
        self.connected = False
        self.data = {}

    async def connect(self):
        await asyncio.sleep(0.1)  # simulate connection time
        self.connected = True
        print("✓ Database connected")

    async def disconnect(self):
        await asyncio.sleep(0.05)
        self.connected = False
        print("✗ Database disconnected")

    def set(self, key: str, value) -> None:
        if not self.connected:
            raise RuntimeError("Database not connected")
        self.data[key] = value

    def get(self, key: str):
        if not self.connected:
            raise RuntimeError("Database not connected")
        return self.data.get(key)

    def all(self) -> dict:
        return dict(self.data)


# Lifespan manages startup and shutdown
@asynccontextmanager
async def lifespan(server: FastMCP) -> AsyncIterator[dict]:
    # Startup
    db = FakeDatabase()
    await db.connect()

    # Seed some data
    db.set("user:1", {"id": 1, "name": "Vishal"})
    db.set("user:2", {"id": 2, "name": "Alice"})

    # Yield context — available to all tools
    yield {"db": db}

    # Shutdown (always runs)
    await db.disconnect()


mcp = FastMCP("Lifespan MCP", lifespan=lifespan)


@mcp.tool()
def store_value(key: str, value: str, ctx=None) -> str:
    """Store a key-value pair in the database."""
    db: FakeDatabase = ctx.request_context.lifespan_context["db"]
    db.set(key, value)
    return f"Stored: {key} = {value}"

@mcp.tool()
def get_value(key: str, ctx=None) -> str:
    """Retrieve a value from the database by key."""
    db: FakeDatabase = ctx.request_context.lifespan_context["db"]
    value = db.get(key)
    if value is None:
        raise KeyError(f"Key '{key}' not found")
    return str(value)

@mcp.tool()
def list_all(ctx=None) -> dict:
    """List all stored key-value pairs."""
    db: FakeDatabase = ctx.request_context.lifespan_context["db"]
    return db.all()

if __name__ == "__main__":
    mcp.run()
```

---

## L7 — Composing Servers

### 14. Mounting Sub-Servers
```python
# 14_composition.py
from fastmcp import FastMCP

# Sub-server 1: Math tools
math_mcp = FastMCP("Math Tools")

@math_mcp.tool()
def add(a: float, b: float) -> float:
    """Add two numbers."""
    return a + b

@math_mcp.tool()
def square_root(n: float) -> float:
    """Calculate the square root of a number."""
    if n < 0:
        raise ValueError("Cannot take square root of a negative number")
    return n ** 0.5

# Sub-server 2: Text tools
text_mcp = FastMCP("Text Tools")

@text_mcp.tool()
def word_count(text: str) -> int:
    """Count words in text."""
    return len(text.split())

@text_mcp.tool()
def reverse_text(text: str) -> str:
    """Reverse a string."""
    return text[::-1]

@text_mcp.tool()
def to_title_case(text: str) -> str:
    """Convert text to title case."""
    return text.title()

# Sub-server 3: Data tools
data_mcp = FastMCP("Data Tools")

@data_mcp.tool()
def flatten_list(items: list) -> list:
    """Flatten one level of nesting in a list."""
    result = []
    for item in items:
        if isinstance(item, list):
            result.extend(item)
        else:
            result.append(item)
    return result

@data_mcp.tool()
def unique_items(items: list) -> list:
    """Return unique items preserving order."""
    seen = set()
    result = []
    for item in items:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

# Main server that mounts all sub-servers
main_mcp = FastMCP("Main Server")

# Mount with prefixes — tool names become: math_add, text_word_count, etc.
main_mcp.mount(math_mcp, prefix="math")
main_mcp.mount(text_mcp, prefix="text")
main_mcp.mount(data_mcp, prefix="data")

if __name__ == "__main__":
    main_mcp.run()
```

---

## L8 — MCP Client

### 15. In-Process Client (Testing)
```python
# 15_client_inprocess.py
import asyncio
from fastmcp import FastMCP, Client

# Build a server
mcp = FastMCP("Test Server")

@mcp.tool()
def add(a: float, b: float) -> float:
    """Add two numbers."""
    return a + b

@mcp.tool()
def get_info() -> dict:
    """Get server info."""
    return {"name": "Test Server", "tools": ["add", "get_info"]}

@mcp.resource("data://greeting")
def greeting() -> str:
    """A friendly greeting."""
    return "Hello from the MCP server!"


async def main():
    # Client connects to the server in-process (great for testing)
    async with Client(mcp) as client:

        # List available tools
        tools = await client.list_tools()
        print("Available tools:")
        for tool in tools:
            print(f"  - {tool.name}: {tool.description}")

        # Call a tool
        result = await client.call_tool("add", {"a": 10, "b": 32})
        print(f"\nadd(10, 32) = {result[0].text}")

        # Call another tool
        info = await client.call_tool("get_info", {})
        print(f"\nget_info() = {info[0].text}")

        # List resources
        resources = await client.list_resources()
        print(f"\nAvailable resources:")
        for r in resources:
            print(f"  - {r.uri}")

        # Read a resource
        content = await client.read_resource("data://greeting")
        print(f"\ngreeting resource = {content[0].text}")


asyncio.run(main())
```

---

### 16. Client connecting to SSE Server
```python
# 16_client_sse.py
# First run: python 12_sse_server.py sse
# Then run this client

import asyncio
from fastmcp import Client

async def main():
    # Connect to a running SSE server
    async with Client("http://localhost:8000/sse") as client:

        # List tools
        tools = await client.list_tools()
        print("Tools on server:")
        for t in tools:
            print(f"  {t.name}")

        # Create tasks
        t1 = await client.call_tool("create_task", {
            "title": "Learn FastMCP",
            "priority": "high"
        })
        print(f"\nCreated: {t1[0].text}")

        t2 = await client.call_tool("create_task", {
            "title": "Build MCP server",
            "priority": "medium"
        })
        print(f"Created: {t2[0].text}")

        # List all tasks
        all_tasks = await client.call_tool("list_tasks", {})
        print(f"\nAll tasks: {all_tasks[0].text}")

        # Complete task 1
        done = await client.call_tool("complete_task", {"task_id": 1})
        print(f"\nCompleted: {done[0].text}")


asyncio.run(main())
```

---

## L9 — Patterns and Real Servers

### 17. CRUD MCP Server Pattern
```python
# 17_crud_server.py
from fastmcp import FastMCP
from pydantic import BaseModel, Field
from typing import Annotated, Optional, Dict, List
import time

mcp = FastMCP("Task CRUD MCP")

# ── In-memory store ──────────────────────────────────────
store: Dict[int, dict] = {}
_counter = 0

# ── Tools ────────────────────────────────────────────────
@mcp.tool()
def create_task(
    title: Annotated[str, Field(min_length=1, max_length=200)],
    priority: Annotated[str, Field(description="low | medium | high")] = "medium",
    tags: Optional[List[str]] = None
) -> dict:
    """Create a new task. Returns the created task with its ID."""
    global _counter
    if priority not in ("low", "medium", "high"):
        raise ValueError("Priority must be: low, medium, or high")
    _counter += 1
    task = {
        "id": _counter,
        "title": title,
        "priority": priority,
        "tags": tags or [],
        "done": False,
        "created_at": time.strftime("%Y-%m-%d %H:%M:%S")
    }
    store[_counter] = task
    return task

@mcp.tool()
def get_task(task_id: int) -> dict:
    """Get a task by its ID."""
    if task_id not in store:
        raise KeyError(f"Task {task_id} not found. IDs available: {list(store.keys())}")
    return store[task_id]

@mcp.tool()
def list_tasks(
    done: Optional[bool] = None,
    priority: Optional[str] = None,
    tag: Optional[str] = None
) -> List[dict]:
    """List tasks with optional filters for status, priority, or tag."""
    result = list(store.values())
    if done is not None:
        result = [t for t in result if t["done"] == done]
    if priority:
        result = [t for t in result if t["priority"] == priority]
    if tag:
        result = [t for t in result if tag in t.get("tags", [])]
    return result

@mcp.tool()
def update_task(
    task_id: int,
    title: Optional[str] = None,
    priority: Optional[str] = None,
    tags: Optional[List[str]] = None
) -> dict:
    """Update task fields. Only provided fields are changed."""
    if task_id not in store:
        raise KeyError(f"Task {task_id} not found")
    if title is not None:
        store[task_id]["title"] = title
    if priority is not None:
        if priority not in ("low", "medium", "high"):
            raise ValueError("Priority must be: low, medium, or high")
        store[task_id]["priority"] = priority
    if tags is not None:
        store[task_id]["tags"] = tags
    return store[task_id]

@mcp.tool()
def complete_task(task_id: int) -> dict:
    """Mark a task as done."""
    if task_id not in store:
        raise KeyError(f"Task {task_id} not found")
    store[task_id]["done"] = True
    return store[task_id]

@mcp.tool()
def delete_task(task_id: int) -> str:
    """Delete a task permanently."""
    if task_id not in store:
        raise KeyError(f"Task {task_id} not found")
    title = store.pop(task_id)["title"]
    return f"Deleted task '{title}' (id={task_id})"

@mcp.tool()
def get_stats() -> dict:
    """Get overview statistics of all tasks."""
    all_tasks = list(store.values())
    return {
        "total": len(all_tasks),
        "done": sum(1 for t in all_tasks if t["done"]),
        "pending": sum(1 for t in all_tasks if not t["done"]),
        "by_priority": {
            p: sum(1 for t in all_tasks if t["priority"] == p)
            for p in ("low", "medium", "high")
        }
    }

# ── Resources ────────────────────────────────────────────
@mcp.resource("tasks://all")
def all_tasks_resource() -> List[dict]:
    """Resource: snapshot of all tasks."""
    return list(store.values())

@mcp.resource("tasks://{task_id}")
def task_resource(task_id: int) -> dict:
    """Resource: a specific task by ID."""
    if task_id not in store:
        raise KeyError(f"Task {task_id} not found")
    return store[task_id]

# ── Prompts ──────────────────────────────────────────────
@mcp.prompt()
def task_summary_prompt() -> str:
    """Generate a prompt to summarise current tasks."""
    stats = get_stats()
    return f"""I have {stats['total']} tasks total:
- {stats['done']} completed
- {stats['pending']} pending
- Priority breakdown: {stats['by_priority']}

Please give me a brief summary and suggest which pending task I should tackle first."""

if __name__ == "__main__":
    mcp.run()
```

---

## L10 — Full MCP Server (Build from Memory)

### 18. Knowledge Base MCP Server
```python
# 18_knowledge_mcp.py
from fastmcp import FastMCP, Context
from pydantic import Field
from typing import Annotated, Optional, List, Dict
import time
import json

mcp = FastMCP("Knowledge Base MCP")

# ── Store ────────────────────────────────────────────────
notes: Dict[int, dict] = {}
_id = {"n": 0}

def _next_id() -> int:
    _id["n"] += 1
    return _id["n"]

# ── Tools ────────────────────────────────────────────────
@mcp.tool()
def add_note(
    title: Annotated[str, Field(min_length=1)],
    content: str,
    tags: Optional[List[str]] = None,
    category: str = "general"
) -> dict:
    """Add a new note to the knowledge base."""
    note_id = _next_id()
    note = {
        "id": note_id,
        "title": title,
        "content": content,
        "tags": tags or [],
        "category": category,
        "created_at": time.strftime("%Y-%m-%d %H:%M:%S"),
        "word_count": len(content.split())
    }
    notes[note_id] = note
    return note

@mcp.tool()
def search_notes(
    query: str,
    category: Optional[str] = None,
    tag: Optional[str] = None
) -> List[dict]:
    """Search notes by keyword in title or content, with optional filters."""
    query_lower = query.lower()
    results = []
    for note in notes.values():
        matches_query = (
            query_lower in note["title"].lower() or
            query_lower in note["content"].lower()
        )
        matches_category = (category is None or note["category"] == category)
        matches_tag = (tag is None or tag in note["tags"])
        if matches_query and matches_category and matches_tag:
            results.append(note)
    return results

@mcp.tool()
def get_note(note_id: int) -> dict:
    """Get a note by its ID."""
    if note_id not in notes:
        raise KeyError(f"Note {note_id} not found")
    return notes[note_id]

@mcp.tool()
def update_note(
    note_id: int,
    title: Optional[str] = None,
    content: Optional[str] = None,
    tags: Optional[List[str]] = None
) -> dict:
    """Update a note. Only provided fields are changed."""
    if note_id not in notes:
        raise KeyError(f"Note {note_id} not found")
    if title: notes[note_id]["title"] = title
    if content:
        notes[note_id]["content"] = content
        notes[note_id]["word_count"] = len(content.split())
    if tags is not None: notes[note_id]["tags"] = tags
    return notes[note_id]

@mcp.tool()
def delete_note(note_id: int) -> str:
    """Delete a note permanently."""
    if note_id not in notes:
        raise KeyError(f"Note {note_id} not found")
    title = notes.pop(note_id)["title"]
    return f"Deleted note: '{title}'"

@mcp.tool()
def list_categories() -> List[str]:
    """List all unique categories in the knowledge base."""
    return list(set(n["category"] for n in notes.values()))

@mcp.tool()
def list_tags() -> List[str]:
    """List all unique tags used across notes."""
    all_tags = []
    for note in notes.values():
        all_tags.extend(note["tags"])
    return list(set(all_tags))

@mcp.tool()
async def export_notes(format: str = "json", ctx: Context = None) -> str:
    """Export all notes as JSON or a plain text summary."""
    await ctx.info(f"Exporting {len(notes)} notes as {format}")
    if format == "json":
        return json.dumps(list(notes.values()), indent=2)
    elif format == "text":
        lines = []
        for note in notes.values():
            lines.append(f"[{note['id']}] {note['title']} ({note['category']})")
            lines.append(f"  Tags: {', '.join(note['tags']) or 'none'}")
            lines.append(f"  Words: {note['word_count']}")
            lines.append(f"  {note['content'][:100]}...")
            lines.append("")
        return "\n".join(lines)
    else:
        raise ValueError("Format must be 'json' or 'text'")

# ── Resources ────────────────────────────────────────────
@mcp.resource("kb://all")
def all_notes() -> List[dict]:
    """Resource: all notes in the knowledge base."""
    return list(notes.values())

@mcp.resource("kb://{note_id}")
def note_resource(note_id: int) -> dict:
    """Resource: a specific note."""
    if note_id not in notes:
        raise KeyError(f"Note {note_id} not found")
    return notes[note_id]

@mcp.resource("kb://stats")
def kb_stats() -> dict:
    """Resource: knowledge base statistics."""
    return {
        "total_notes": len(notes),
        "categories": list_categories(),
        "tags": list_tags(),
        "total_words": sum(n["word_count"] for n in notes.values())
    }

# ── Prompts ──────────────────────────────────────────────
@mcp.prompt()
def ask_about_topic(topic: str) -> str:
    """Generate a prompt to query the knowledge base about a topic."""
    return f"""Search the knowledge base for information about '{topic}'.
Summarise what you find, noting which notes are most relevant.
If nothing is found, say so clearly."""

@mcp.prompt()
def daily_review_prompt() -> str:
    """Prompt for a daily knowledge base review."""
    stats = kb_stats()
    return f"""I have {stats['total_notes']} notes in my knowledge base across
these categories: {', '.join(stats['categories']) or 'none yet'}.

Please help me review and organise my knowledge base:
1. What are the main themes?
2. Are there any gaps I should fill?
3. Which notes could be merged or connected?"""

if __name__ == "__main__":
    mcp.run()
```

---

## Bonus Drills — Blank File Challenges

Write these from scratch without peeking.

### B1. Weather MCP Server
```
3 tools: get_weather(city), get_forecast(city, days), list_cities()
Resources: weather://current/{city}, weather://cities
Prompt: weather_report_prompt(city)
Use fake data — no real API needed
```

### B2. File Manager MCP Server
```
Tools: list_files(directory), read_file(path), write_file(path, content),
       delete_file(path), search_files(directory, extension)
Resources: fs://{path}
Safety rule: only allow operations inside a /workspace directory
```

### B3. Calculator with History
```
Tools: calculate(expression), get_history(), clear_history()
Resource: calc://history
Each calculation stored with timestamp and result
```

### B4. In-Process Client Test
```
Build a server with 3 tools
Write an async test function using Client(mcp)
Assert all 3 tools work correctly
```

### B5. Composition Server
```
Build 2 sub-servers (notes_mcp, tasks_mcp) each with 2 tools
Mount both into main_mcp with prefixes
Verify all 4 tools are accessible from main
```

### B6. Full Server from Scratch
```
Build a contacts MCP:
Fields: id, name, phone, email, tags
Full CRUD: add, get, update, delete, search, list
Resources: contacts://all, contacts://{id}
Prompt: find_contact_prompt(name)
```

---

## Claude Desktop Integration

Once your server runs, add it to Claude Desktop's config:

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json  (macOS)
// %APPDATA%/Claude/claude_desktop_config.json                       (Windows)

{
  "mcpServers": {
    "my-task-server": {
      "command": "python",
      "args": ["/absolute/path/to/17_crud_server.py"],
      "env": {
        "OPENAI_API_KEY": "your-key-here"
      }
    }
  }
}
```
> After saving, restart Claude Desktop. Your tools appear automatically.

---

## FastMCP Cheat Sheet

```python
from fastmcp import FastMCP, Context, Client

mcp = FastMCP("name")

# Tool — LLM can call this
@mcp.tool()
def my_tool(param: str) -> str: ...

# Resource — LLM can read this
@mcp.resource("scheme://path/{param}")
def my_resource(param: str) -> dict: ...

# Prompt — reusable template
@mcp.prompt()
def my_prompt(topic: str) -> str: ...

# Context — logging and progress (async tools)
@mcp.tool()
async def async_tool(data: str, ctx: Context = None) -> str:
    await ctx.info("starting")
    await ctx.report_progress(1, 3)
    await ctx.warning("something odd")
    return "done"

# Run modes
mcp.run()                                        # stdio (Claude Desktop)
mcp.run(transport="sse", host="0.0.0.0", port=8000)  # HTTP

# Client
async with Client(mcp) as c:           # in-process
    tools = await c.list_tools()
    result = await c.call_tool("name", {"param": "value"})
    content = await c.read_resource("scheme://path")
```

---

## Progress Tracker

| Level | Topic                                    | Done? | Time Taken |
|-------|------------------------------------------|-------|------------|
| L1    | First server, tools, return types        |  [ ]  |            |
| L2    | Validation, error handling, HTTP calls   |  [ ]  |            |
| L3    | Static and dynamic resources             |  [ ]  |            |
| L4    | File resources, prompts                  |  [ ]  |            |
| L5    | Context, logging, progress               |  [ ]  |            |
| L6    | SSE transport, lifespan/dependencies     |  [ ]  |            |
| L7    | Server composition, mounting             |  [ ]  |            |
| L8    | In-process client, SSE client            |  [ ]  |            |
| L9    | CRUD MCP server pattern                  |  [ ]  |            |
| L10   | Full Knowledge Base MCP server           |  [ ]  |            |
| Bonus | Blank file challenges                    |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 18 (Knowledge Base MCP)
> from memory — tools, resources, prompts, context, export. Run it with
> `fastmcp dev`. Call every tool from the Inspector. Connect it to Claude Desktop.
> When Claude can add and search notes through conversation — you own FastMCP.
