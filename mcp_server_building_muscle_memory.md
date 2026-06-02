# Building Your Own MCP Server — Muscle Memory Drills (L1 → L10)

> **What this sheet covers**
> Building MCP servers from scratch — exposing your own tools, prompts,
> and resources so any MCP-compatible agent (Claude Desktop, LangGraph, etc.)
> can use them.
>
> **Install:**
> ```
> pip install mcp fastapi uvicorn python-dotenv
> ```
> **Mental model:** You are building the server side. Your job is to define
> tools (functions with schemas), prompts (templates), and resources (readable data),
> then expose them over MCP protocol so agents can discover and call them.

---

## THE CORE MENTAL MODEL — Read This First

```
Your MCP Server exposes:
  TOOLS     → functions agents can CALL (add, search, send_email, query_db)
  PROMPTS   → reusable prompt TEMPLATES agents can retrieve
  RESOURCES → readable DATA agents can access (files, DB records, configs)

Two ways to run your server:
  STDIO     → subprocess. stdin/stdout. Used by Claude Desktop, local agents.
  HTTP/SSE  → web server. Used by remote clients, FastAPI integration.

Every tool needs:
  1. A name (snake_case)
  2. A description (what the LLM reads to decide whether to use it)
  3. An input schema (what arguments it takes)
  4. A handler function (the actual Python code)
```

---

## L1 — Simplest MCP Server

### 1. Hello MCP Server (STDIO)
```python
# server_hello.py
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

# Step 1: Create the server
app = Server("hello-server")

# Step 2: Register a tool
@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="greet",
            description="Greet a person by name",
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {
                        "type": "string",
                        "description": "The name of the person to greet"
                    }
                },
                "required": ["name"]
            }
        )
    ]

# Step 3: Handle tool calls
@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "greet":
        person = arguments.get("name", "World")
        return [types.TextContent(type="text", text=f"Hello, {person}! 👋")]
    raise ValueError(f"Unknown tool: {name}")

# Step 4: Run the server
async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```
> Test: `python server_hello.py` (will wait for MCP protocol messages on stdin)

---

### 2. Server with Multiple Tools
```python
# server_math.py
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("math-server")

TOOLS = [
    types.Tool(
        name="add",
        description="Add two numbers",
        inputSchema={
            "type": "object",
            "properties": {
                "a": {"type": "number", "description": "First number"},
                "b": {"type": "number", "description": "Second number"}
            },
            "required": ["a", "b"]
        }
    ),
    types.Tool(
        name="multiply",
        description="Multiply two numbers",
        inputSchema={
            "type": "object",
            "properties": {
                "a": {"type": "number", "description": "First number"},
                "b": {"type": "number", "description": "Second number"}
            },
            "required": ["a", "b"]
        }
    ),
    types.Tool(
        name="power",
        description="Raise a number to a power",
        inputSchema={
            "type": "object",
            "properties": {
                "base": {"type": "number", "description": "The base number"},
                "exponent": {"type": "number", "description": "The exponent"}
            },
            "required": ["base", "exponent"]
        }
    ),
]

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return TOOLS

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name == "add":
        result = arguments["a"] + arguments["b"]
    elif name == "multiply":
        result = arguments["a"] * arguments["b"]
    elif name == "power":
        result = arguments["base"] ** arguments["exponent"]
    else:
        raise ValueError(f"Unknown tool: {name}")

    return [types.TextContent(type="text", text=str(result))]

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

---

## L2 — Tool Design Patterns

### 3. Tool with Optional Parameters
```python
# server_search.py
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("search-server")

# Simulated database
ITEMS = [
    {"id": 1, "name": "Laptop", "category": "electronics", "price": 999},
    {"id": 2, "name": "Python Book", "category": "books", "price": 45},
    {"id": 3, "name": "Mechanical Keyboard", "category": "electronics", "price": 120},
    {"id": 4, "name": "Desk Lamp", "category": "furniture", "price": 35},
    {"id": 5, "name": "FastAPI Book", "category": "books", "price": 40},
]

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="search_items",
            description="Search for items by name or filter by category",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Search query (searches in name)"
                    },
                    "category": {
                        "type": "string",
                        "description": "Filter by category: electronics, books, furniture"
                    },
                    "max_price": {
                        "type": "number",
                        "description": "Maximum price filter"
                    },
                    "limit": {
                        "type": "integer",
                        "description": "Maximum number of results (default: 5)",
                        "default": 5
                    }
                },
                "required": []  # all optional
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name != "search_items":
        raise ValueError(f"Unknown tool: {name}")

    results = list(ITEMS)

    query = arguments.get("query", "").lower()
    if query:
        results = [i for i in results if query in i["name"].lower()]

    category = arguments.get("category")
    if category:
        results = [i for i in results if i["category"] == category]

    max_price = arguments.get("max_price")
    if max_price:
        results = [i for i in results if i["price"] <= max_price]

    limit = arguments.get("limit", 5)
    results = results[:limit]

    return [types.TextContent(type="text", text=json.dumps(results, indent=2))]

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

---

### 4. Tool That Returns Structured JSON
```python
# server_tasks.py
import asyncio
import json
from datetime import datetime
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("task-server")

# In-memory task store
tasks = {}
_counter = 0

def make_tool(name, description, schema):
    return types.Tool(name=name, description=description, inputSchema=schema)

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        make_tool("create_task", "Create a new task",
            {"type": "object",
             "properties": {
                 "title": {"type": "string", "description": "Task title"},
                 "priority": {"type": "string", "enum": ["low", "medium", "high"], "default": "medium"}
             },
             "required": ["title"]}),
        make_tool("list_tasks", "List all tasks, optionally filter by done status",
            {"type": "object",
             "properties": {
                 "done": {"type": "boolean", "description": "Filter by done status"}
             }}),
        make_tool("complete_task", "Mark a task as done",
            {"type": "object",
             "properties": {
                 "task_id": {"type": "integer", "description": "The task ID to complete"}
             },
             "required": ["task_id"]}),
        make_tool("delete_task", "Delete a task by ID",
            {"type": "object",
             "properties": {
                 "task_id": {"type": "integer", "description": "The task ID to delete"}
             },
             "required": ["task_id"]}),
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    global _counter

    if name == "create_task":
        _counter += 1
        task = {
            "id": _counter,
            "title": arguments["title"],
            "priority": arguments.get("priority", "medium"),
            "done": False,
            "created_at": datetime.now().isoformat()
        }
        tasks[_counter] = task
        return [types.TextContent(type="text", text=json.dumps(task, indent=2))]

    elif name == "list_tasks":
        result = list(tasks.values())
        if "done" in arguments:
            result = [t for t in result if t["done"] == arguments["done"]]
        return [types.TextContent(type="text", text=json.dumps(result, indent=2))]

    elif name == "complete_task":
        tid = arguments["task_id"]
        if tid not in tasks:
            return [types.TextContent(type="text", text=f"Error: Task {tid} not found")]
        tasks[tid]["done"] = True
        return [types.TextContent(type="text", text=json.dumps(tasks[tid], indent=2))]

    elif name == "delete_task":
        tid = arguments["task_id"]
        if tid not in tasks:
            return [types.TextContent(type="text", text=f"Error: Task {tid} not found")]
        deleted = tasks.pop(tid)
        return [types.TextContent(type="text", text=f"Deleted: {json.dumps(deleted)}")]

    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

---

## L3 — Error Handling in Tools

### 5. Proper Error Responses
```python
# server_safe.py
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("safe-server")

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="divide",
            description="Divide two numbers safely",
            inputSchema={
                "type": "object",
                "properties": {
                    "numerator": {"type": "number"},
                    "denominator": {"type": "number"}
                },
                "required": ["numerator", "denominator"]
            }
        ),
        types.Tool(
            name="parse_json",
            description="Parse a JSON string and return formatted output",
            inputSchema={
                "type": "object",
                "properties": {
                    "json_string": {"type": "string", "description": "The JSON string to parse"}
                },
                "required": ["json_string"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    try:
        if name == "divide":
            num = arguments["numerator"]
            den = arguments["denominator"]
            if den == 0:
                # Return error as text — don't raise (agent needs to see the error)
                return [types.TextContent(type="text", text="Error: Cannot divide by zero")]
            result = num / den
            return [types.TextContent(type="text", text=f"{result:.6f}")]

        elif name == "parse_json":
            raw = arguments["json_string"]
            try:
                parsed = json.loads(raw)
                return [types.TextContent(type="text", text=json.dumps(parsed, indent=2))]
            except json.JSONDecodeError as e:
                return [types.TextContent(type="text", text=f"Error: Invalid JSON — {e}")]

        else:
            return [types.TextContent(type="text", text=f"Error: Unknown tool '{name}'")]

    except KeyError as e:
        return [types.TextContent(type="text", text=f"Error: Missing required argument {e}")]
    except Exception as e:
        return [types.TextContent(type="text", text=f"Error: Unexpected error — {e}")]

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

---

## L4 — Resources

### 6. Exposing Resources
```python
# server_resources.py
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("resource-server")

# Simulated data store
CONFIG = {
    "app_name": "MyAIApp",
    "version": "1.0.0",
    "model": "gpt-4o-mini",
    "max_tokens": 1000
}

DOCS = {
    "readme": "# MyAIApp\n\nThis app uses LangGraph and MCP to build AI agents.\n\n## Setup\n1. Install deps\n2. Set API keys\n3. Run the server",
    "api": "# API Reference\n\n## Endpoints\n- POST /agent/query\n- GET /health\n- GET /tools"
}

@app.list_resources()
async def list_resources() -> list[types.Resource]:
    return [
        types.Resource(
            uri="config://app",
            name="App Configuration",
            description="Current application configuration",
            mimeType="application/json"
        ),
        types.Resource(
            uri="docs://readme",
            name="README",
            description="Application README",
            mimeType="text/markdown"
        ),
        types.Resource(
            uri="docs://api",
            name="API Reference",
            description="API documentation",
            mimeType="text/markdown"
        ),
    ]

@app.read_resource()
async def read_resource(uri: str) -> str:
    if uri == "config://app":
        return json.dumps(CONFIG, indent=2)
    elif uri == "docs://readme":
        return DOCS["readme"]
    elif uri == "docs://api":
        return DOCS["api"]
    else:
        raise ValueError(f"Unknown resource: {uri}")

@app.list_tools()
async def list_tools() -> list[types.Tool]:
    return []  # this server only exposes resources

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

---

### 7. Dynamic Resources
```python
# server_dynamic_resources.py
import asyncio
import json
import os
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("dynamic-resource-server")

@app.list_resources()
async def list_resources() -> list[types.Resource]:
    """List files in current directory as resources."""
    resources = []
    for filename in os.listdir("."):
        if filename.endswith((".py", ".txt", ".json", ".md")):
            resources.append(
                types.Resource(
                    uri=f"file://{filename}",
                    name=filename,
                    description=f"Contents of {filename}",
                    mimeType="text/plain"
                )
            )
    return resources

@app.read_resource()
async def read_resource(uri: str) -> str:
    if not uri.startswith("file://"):
        raise ValueError(f"Unknown URI scheme: {uri}")

    filename = uri[len("file://"):]

    # Security: only allow files in current dir
    if "/" in filename or ".." in filename:
        raise ValueError("Invalid filename")

    if not os.path.exists(filename):
        raise ValueError(f"File not found: {filename}")

    with open(filename, "r") as f:
        return f.read()

@app.list_tools()
async def list_tools():
    return []

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

---

## L5 — Prompts

### 8. Exposing Prompt Templates
```python
# server_prompts.py
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("prompt-server")

@app.list_prompts()
async def list_prompts() -> list[types.Prompt]:
    return [
        types.Prompt(
            name="summarize",
            description="Summarize a piece of text concisely",
            arguments=[
                types.PromptArgument(name="text", description="The text to summarize", required=True),
                types.PromptArgument(name="style", description="Summary style: bullet or paragraph", required=False)
            ]
        ),
        types.Prompt(
            name="code_review",
            description="Review code and suggest improvements",
            arguments=[
                types.PromptArgument(name="code", description="The code to review", required=True),
                types.PromptArgument(name="language", description="Programming language", required=True)
            ]
        ),
    ]

@app.get_prompt()
async def get_prompt(name: str, arguments: dict | None) -> types.GetPromptResult:
    args = arguments or {}

    if name == "summarize":
        text = args.get("text", "")
        style = args.get("style", "paragraph")
        style_instruction = "as bullet points" if style == "bullet" else "as a paragraph"
        content = f"Please summarize the following text {style_instruction}:\n\n{text}"
        return types.GetPromptResult(
            description="Summarization prompt",
            messages=[
                types.PromptMessage(
                    role="user",
                    content=types.TextContent(type="text", text=content)
                )
            ]
        )

    elif name == "code_review":
        code = args.get("code", "")
        language = args.get("language", "Python")
        content = f"""Please review this {language} code and provide:
1. Code quality assessment
2. Potential bugs or issues
3. Performance improvements
4. Best practice suggestions

Code:
```{language.lower()}
{code}
```"""
        return types.GetPromptResult(
            description="Code review prompt",
            messages=[
                types.PromptMessage(
                    role="user",
                    content=types.TextContent(type="text", text=content)
                )
            ]
        )

    raise ValueError(f"Unknown prompt: {name}")

@app.list_tools()
async def list_tools():
    return []

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```

---

## L6 — FastMCP (Simpler API)

### 9. FastMCP — Decorator-Based Server
```python
# server_fastmcp.py
# FastMCP is the simpler, more Pythonic way to build MCP servers
# pip install fastmcp

from fastmcp import FastMCP
from datetime import datetime
import json

# Create server in one line
mcp = FastMCP("My FastMCP Server")

# Register tools with just a decorator — FastMCP reads type hints for schema
@mcp.tool()
def add(a: float, b: float) -> float:
    """Add two numbers together."""
    return a + b

@mcp.tool()
def get_current_time() -> str:
    """Get the current date and time."""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

@mcp.tool()
def search_items(query: str, category: str = "all", limit: int = 5) -> list:
    """Search for items in the catalog.

    Args:
        query: The search term
        category: Filter by category (all, electronics, books)
        limit: Maximum results to return
    """
    items = [
        {"id": 1, "name": "Laptop", "category": "electronics"},
        {"id": 2, "name": "Python Book", "category": "books"},
        {"id": 3, "name": "Keyboard", "category": "electronics"},
    ]
    results = [i for i in items if query.lower() in i["name"].lower()]
    if category != "all":
        results = [i for i in results if i["category"] == category]
    return results[:limit]

# Register a resource
@mcp.resource("config://settings")
def get_settings() -> str:
    """Application settings."""
    return json.dumps({"model": "gpt-4o-mini", "temperature": 0}, indent=2)

# Register a prompt
@mcp.prompt()
def analyze_prompt(data: str) -> str:
    """Generate a data analysis prompt."""
    return f"Please analyze the following data and provide insights:\n\n{data}"

if __name__ == "__main__":
    mcp.run()  # runs as stdio server by default
```

---

### 10. FastMCP with Context and Logging
```python
# server_fastmcp_advanced.py
from fastmcp import FastMCP, Context
from typing import Optional
import json

mcp = FastMCP("Advanced FastMCP Server")

# In-memory store
notes: dict[int, dict] = {}
_id = 0

@mcp.tool()
async def create_note(title: str, content: str, tags: Optional[list[str]] = None, ctx: Context = None) -> dict:
    """Create and save a note.

    Args:
        title: Note title
        content: Note content
        tags: Optional list of tags
    """
    global _id
    _id += 1

    # ctx.info() logs to the MCP client
    if ctx:
        await ctx.info(f"Creating note: {title}")

    note = {
        "id": _id,
        "title": title,
        "content": content,
        "tags": tags or []
    }
    notes[_id] = note

    if ctx:
        await ctx.info(f"Note created with ID: {_id}")

    return note

@mcp.tool()
async def get_note(note_id: int, ctx: Context = None) -> dict:
    """Get a note by its ID.

    Args:
        note_id: The ID of the note to retrieve
    """
    if ctx:
        await ctx.info(f"Fetching note {note_id}")

    if note_id not in notes:
        if ctx:
            await ctx.warning(f"Note {note_id} not found")
        return {"error": f"Note {note_id} not found"}

    return notes[note_id]

@mcp.tool()
async def list_notes(tag_filter: Optional[str] = None, ctx: Context = None) -> list:
    """List all notes, optionally filtered by tag.

    Args:
        tag_filter: Optional tag to filter by
    """
    result = list(notes.values())
    if tag_filter:
        result = [n for n in result if tag_filter in n.get("tags", [])]

    if ctx:
        await ctx.info(f"Returning {len(result)} notes")

    return result

if __name__ == "__main__":
    mcp.run()
```

---

## L7 — HTTP/SSE Server

### 11. MCP Server with SSE Transport (Accessible over HTTP)
```python
# server_sse.py
# This makes your MCP server accessible as an HTTP endpoint
# Agents can connect via URL instead of launching a subprocess

from fastmcp import FastMCP
import json
from datetime import datetime

mcp = FastMCP("SSE MCP Server")

@mcp.tool()
def get_server_info() -> dict:
    """Get information about this MCP server."""
    return {
        "name": "SSE MCP Server",
        "version": "1.0.0",
        "transport": "sse",
        "started_at": datetime.now().isoformat()
    }

@mcp.tool()
def echo(message: str, uppercase: bool = False) -> str:
    """Echo a message back, optionally in uppercase.

    Args:
        message: The message to echo
        uppercase: Whether to return in uppercase
    """
    return message.upper() if uppercase else message

@mcp.tool()
def calculate(expression: str) -> str:
    """Safely evaluate a math expression.

    Args:
        expression: A math expression like '2 + 3 * 4'
    """
    try:
        # Only allow safe operations
        allowed = set("0123456789+-*/()., ")
        if not all(c in allowed for c in expression):
            return "Error: Invalid characters in expression"
        return str(eval(expression))
    except Exception as e:
        return f"Error: {e}"

if __name__ == "__main__":
    # Run as SSE server on port 8000
    # Agents connect to: http://localhost:8000/sse
    mcp.run(transport="sse", host="0.0.0.0", port=8000)
```
> Run: `python server_sse.py`
> Connect from agent: `{"url": "http://localhost:8000/sse", "transport": "sse"}`

---

## L8 — Real-World Server Patterns

### 12. Database-Connected MCP Server
```python
# server_db.py
import asyncio
import json
import sqlite3
from fastmcp import FastMCP

mcp = FastMCP("Database MCP Server")

# Initialize SQLite DB
DB_PATH = "app.db"

def init_db():
    conn = sqlite3.connect(DB_PATH)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            priority TEXT DEFAULT 'medium',
            done BOOLEAN DEFAULT FALSE,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    conn.close()

def get_conn():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row  # returns dict-like rows
    return conn

init_db()

@mcp.tool()
def create_task(title: str, priority: str = "medium") -> dict:
    """Create a new task in the database.

    Args:
        title: Task title
        priority: Task priority (low, medium, high)
    """
    conn = get_conn()
    cursor = conn.execute(
        "INSERT INTO tasks (title, priority) VALUES (?, ?) RETURNING *",
        (title, priority)
    )
    row = dict(cursor.fetchone())
    conn.commit()
    conn.close()
    return row

@mcp.tool()
def list_tasks(done: bool = None) -> list:
    """List all tasks from the database.

    Args:
        done: Filter by completion status (None = all)
    """
    conn = get_conn()
    if done is None:
        rows = conn.execute("SELECT * FROM tasks ORDER BY created_at DESC").fetchall()
    else:
        rows = conn.execute("SELECT * FROM tasks WHERE done=? ORDER BY created_at DESC", (done,)).fetchall()
    conn.close()
    return [dict(r) for r in rows]

@mcp.tool()
def complete_task(task_id: int) -> dict:
    """Mark a task as complete.

    Args:
        task_id: The ID of the task to complete
    """
    conn = get_conn()
    conn.execute("UPDATE tasks SET done=TRUE WHERE id=?", (task_id,))
    conn.commit()
    row = conn.execute("SELECT * FROM tasks WHERE id=?", (task_id,)).fetchone()
    conn.close()
    if not row:
        return {"error": f"Task {task_id} not found"}
    return dict(row)

@mcp.tool()
def delete_task(task_id: int) -> dict:
    """Delete a task permanently.

    Args:
        task_id: The ID of the task to delete
    """
    conn = get_conn()
    row = conn.execute("SELECT * FROM tasks WHERE id=?", (task_id,)).fetchone()
    if not row:
        conn.close()
        return {"error": f"Task {task_id} not found"}
    conn.execute("DELETE FROM tasks WHERE id=?", (task_id,))
    conn.commit()
    conn.close()
    return {"deleted": dict(row)}

if __name__ == "__main__":
    mcp.run()
```

---

### 13. MCP Server with External API Calls
```python
# server_weather.py
import asyncio
import httpx
from fastmcp import FastMCP

mcp = FastMCP("Weather MCP Server")

# Fake weather data (replace with real API in production)
WEATHER_DATA = {
    "pune": {"temp": 28, "condition": "partly cloudy", "humidity": 65},
    "mumbai": {"temp": 31, "condition": "humid", "humidity": 85},
    "delhi": {"temp": 35, "condition": "sunny", "humidity": 30},
    "bangalore": {"temp": 24, "condition": "pleasant", "humidity": 60},
}

@mcp.tool()
def get_weather(city: str) -> dict:
    """Get current weather for a city.

    Args:
        city: City name (pune, mumbai, delhi, bangalore)
    """
    city_lower = city.lower()
    if city_lower not in WEATHER_DATA:
        return {"error": f"Weather data not available for {city}"}

    data = WEATHER_DATA[city_lower]
    return {
        "city": city,
        "temperature_celsius": data["temp"],
        "condition": data["condition"],
        "humidity_percent": data["humidity"],
        "feels_like": data["temp"] - 2 if data["humidity"] < 50 else data["temp"] + 3
    }

@mcp.tool()
def compare_weather(city1: str, city2: str) -> dict:
    """Compare weather between two cities.

    Args:
        city1: First city name
        city2: Second city name
    """
    w1 = get_weather(city1)
    w2 = get_weather(city2)

    if "error" in w1:
        return w1
    if "error" in w2:
        return w2

    hotter = city1 if w1["temperature_celsius"] > w2["temperature_celsius"] else city2
    return {
        city1: w1,
        city2: w2,
        "hotter_city": hotter,
        "temp_difference": abs(w1["temperature_celsius"] - w2["temperature_celsius"])
    }

if __name__ == "__main__":
    mcp.run()
```

---

## L9 — claude_desktop_config

### 14. Configure Server for Claude Desktop
```json
// ~/.config/claude/claude_desktop_config.json  (Mac/Linux)
// %APPDATA%\Claude\claude_desktop_config.json  (Windows)

{
  "mcpServers": {
    "math": {
      "command": "python",
      "args": ["/absolute/path/to/server_math.py"]
    },
    "tasks": {
      "command": "python",
      "args": ["/absolute/path/to/server_tasks.py"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/vishal/projects"]
    }
  }
}
```

```python
# Verify your server works before adding to Claude Desktop
# Run this test client against your server:

import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def test_my_server():
    params = StdioServerParameters(
        command="python",
        args=["server_math.py"]  # your server
    )

    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # List tools
            tools = await session.list_tools()
            print("Tools:", [t.name for t in tools.tools])

            # Call a tool
            result = await session.call_tool("add", {"a": 10, "b": 20})
            print("10 + 20 =", result.content[0].text)

asyncio.run(test_my_server())
```

---

## L10 — Full Production MCP Server (Build from Memory)

### 15. Complete Image Memory MCP Server
```python
# image_memory_server.py
# This is your actual project — an MCP server for the image memory layer

import asyncio
import json
from datetime import datetime
from typing import Optional
from fastmcp import FastMCP, Context

mcp = FastMCP(
    "Image Memory MCP Server",
    instructions="This server provides tools to store, search, and retrieve image memories."
)

# ─── In-Memory Store (replace with Neo4j in production) ──
memories: dict[int, dict] = {}
_id = 0

# ─── Tools ───────────────────────────────────────────────

@mcp.tool()
async def store_image_memory(
    filename: str,
    description: str,
    people: Optional[list[str]] = None,
    location: Optional[str] = None,
    event: Optional[str] = None,
    date: Optional[str] = None,
    tags: Optional[list[str]] = None,
    intent: Optional[str] = None,
    ctx: Context = None
) -> dict:
    """Store an analyzed image as a memory.

    Args:
        filename: Image filename or path
        description: What is in the image
        people: List of people identified in the image
        location: Where the image was taken
        event: Event or occasion
        date: Date of the image (YYYY-MM-DD)
        tags: Descriptive tags
        intent: Why this was saved (memory, reference, todo, inspiration)
    """
    global _id
    _id += 1

    memory = {
        "id": _id,
        "filename": filename,
        "description": description,
        "people": people or [],
        "location": location,
        "event": event,
        "date": date,
        "tags": tags or [],
        "intent": intent or "memory",
        "stored_at": datetime.now().isoformat()
    }

    memories[_id] = memory

    if ctx:
        await ctx.info(f"Stored memory #{_id}: {filename}")

    return memory

@mcp.tool()
async def search_memories(
    query: str,
    person: Optional[str] = None,
    location: Optional[str] = None,
    event: Optional[str] = None,
    intent: Optional[str] = None,
    limit: int = 10
) -> list:
    """Search image memories by natural language or filters.

    Args:
        query: Natural language search query
        person: Filter by person name
        location: Filter by location
        event: Filter by event name
        intent: Filter by intent (memory, reference, todo, inspiration)
        limit: Maximum results
    """
    results = list(memories.values())

    # Text search in description and tags
    if query:
        q = query.lower()
        results = [
            m for m in results
            if q in m["description"].lower()
            or any(q in t.lower() for t in m["tags"])
            or (m["event"] and q in m["event"].lower())
        ]

    if person:
        results = [m for m in results if person.lower() in [p.lower() for p in m["people"]]]

    if location:
        results = [m for m in results if m["location"] and location.lower() in m["location"].lower()]

    if event:
        results = [m for m in results if m["event"] and event.lower() in m["event"].lower()]

    if intent:
        results = [m for m in results if m["intent"] == intent]

    return results[:limit]

@mcp.tool()
async def get_memory(memory_id: int) -> dict:
    """Get a specific memory by ID.

    Args:
        memory_id: The ID of the memory
    """
    if memory_id not in memories:
        return {"error": f"Memory {memory_id} not found"}
    return memories[memory_id]

@mcp.tool()
async def list_people() -> list:
    """List all unique people across all memories."""
    all_people = set()
    for m in memories.values():
        for p in m["people"]:
            all_people.add(p)
    return sorted(list(all_people))

@mcp.tool()
async def get_timeline(
    person: Optional[str] = None,
    location: Optional[str] = None
) -> list:
    """Get memories in chronological order.

    Args:
        person: Filter by person
        location: Filter by location
    """
    results = [m for m in memories.values() if m["date"]]

    if person:
        results = [m for m in results if person in m["people"]]
    if location:
        results = [m for m in results if m["location"] and location in m["location"]]

    results.sort(key=lambda m: m["date"])
    return results

@mcp.tool()
async def delete_memory(memory_id: int) -> dict:
    """Delete a memory by ID.

    Args:
        memory_id: The ID of the memory to delete
    """
    if memory_id not in memories:
        return {"error": f"Memory {memory_id} not found"}
    deleted = memories.pop(memory_id)
    return {"deleted": deleted}

# ─── Resources ───────────────────────────────────────────
@mcp.resource("memory://stats")
def get_stats() -> str:
    """Memory store statistics."""
    all_people = set()
    all_events = set()
    intents: dict[str, int] = {}

    for m in memories.values():
        all_people.update(m["people"])
        if m["event"]:
            all_events.add(m["event"])
        intents[m["intent"]] = intents.get(m["intent"], 0) + 1

    stats = {
        "total_memories": len(memories),
        "unique_people": len(all_people),
        "unique_events": len(all_events),
        "by_intent": intents
    }
    return json.dumps(stats, indent=2)

# ─── Prompts ─────────────────────────────────────────────
@mcp.prompt()
def analyze_image_prompt(image_description: str) -> str:
    """Generate a prompt to extract structured memory from image description."""
    return f"""Analyze this image and extract structured memory information.

Image: {image_description}

Extract and return a JSON object with:
- description: what you see in the image
- people: list of people (use names if known, else descriptors)
- location: where this was taken (if identifiable)
- event: what occasion or event this is
- date: estimated date if visible
- tags: descriptive keywords
- intent: one of [memory, reference, todo, inspiration, saved_for_later]
- mood: overall mood/emotion of the image"""

if __name__ == "__main__":
    mcp.run()
```

---

## Bonus Drills — Blank File Challenges

### B1. Build a math MCP server from scratch with 5 tools: add, subtract, multiply, divide, power. Handle division by zero.

### B2. Build a notes MCP server with tools: create_note, get_note, list_notes (filter by tag), delete_note. Use JSON file as storage.

### B3. Build a FastMCP server with one tool that calls a real public API (e.g., `https://api.coindesk.com/v1/bpi/currentprice.json` for Bitcoin price).

### B4. Build a server with a resource that reads a real file from disk and returns its contents. Handle file-not-found gracefully.

### B5. Write a test client that connects to your server, lists all tools, calls each one with sample arguments, and prints results.

### B6. Build the image memory server (exercise 15) completely from memory. All 6 tools, 1 resource, 1 prompt.

### B7. Add SQLite persistence to any server — replace in-memory dict with actual DB queries.

### B8. Build a server that exposes your LangGraph agent as a tool: `run_agent(query: str) -> str`. The server calls your LangGraph agent internally.

---

## Quick Reference

```python
# FastMCP (recommended)
from fastmcp import FastMCP
mcp = FastMCP("server-name")

@mcp.tool()
def my_tool(arg1: str, arg2: int = 5) -> str:
    """Docstring becomes the tool description."""
    return f"result"

@mcp.resource("config://app")
def my_resource() -> str:
    return json.dumps({"key": "value"})

@mcp.prompt()
def my_prompt(topic: str) -> str:
    return f"Tell me about {topic}"

mcp.run()                         # stdio (default)
mcp.run(transport="sse", port=8000)  # HTTP/SSE

# Low-level MCP
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

app = Server("name")

@app.list_tools()
async def list_tools() -> list[types.Tool]: ...

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]: ...

async with stdio_server() as (r, w):
    await app.run(r, w, app.create_initialization_options())
```

---

## Progress Tracker

| Level | Topic                                      | Done? | Time Taken |
|-------|--------------------------------------------|-------|------------|
| L1    | Hello Server, Multiple Tools               |  [ ]  |            |
| L2    | Optional params, Structured JSON output    |  [ ]  |            |
| L3    | Error handling in tools                    |  [ ]  |            |
| L4    | Resources — static and dynamic             |  [ ]  |            |
| L5    | Prompts — templates                        |  [ ]  |            |
| L6    | FastMCP — decorator API, Context, logging  |  [ ]  |            |
| L7    | SSE transport — HTTP server                |  [ ]  |            |
| L8    | Real-world: SQLite, external APIs          |  [ ]  |            |
| L9    | Claude Desktop config, testing client      |  [ ]  |            |
| L10   | Full image memory MCP server               |  [ ]  |            |
| Bonus | Blank File Challenges                      |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 15 (image memory server)
> completely from memory — all 6 tools, 1 resource, 1 prompt. Run it.
> Connect to it with the test client from exercise 14. Every tool must respond.
> That's your milestone.
