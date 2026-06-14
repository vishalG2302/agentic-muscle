# Build Agents From Scratch — Muscle Memory Drills (L1 → L10)

> **What this sheet is about**
> LangGraph and CrewAI are training wheels.
> Companies like Anthropic (Claude Code), Cursor, Cognition (Devin), and
> OpenCode build their own agent loops from raw API calls.
> This sheet teaches you to do the same.
>
> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `python filename.py`
> - Install: `pip install anthropic openai python-dotenv httpx rich`
> - Set `ANTHROPIC_API_KEY` and/or `OPENAI_API_KEY` in `.env`
> - L1–L2 → raw tool calling, the agent loop (30–40 min)
> - L3–L4 → streaming, code execution (35–45 min)
> - L5–L6 → file system agent, multi-turn state (35–45 min)
> - L7–L8 → planning agents, multi-agent coordination (40–50 min)
> - L9–L10 → full coding agent from scratch (60–90 min)

---

## The Mental Model — Read This Before Writing a Line

```
What a framework like LangGraph does under the hood:

  while True:
      response = llm.call(messages + tools)

      if response.has_tool_calls:
          for tool_call in response.tool_calls:
              result = execute_tool(tool_call)
              messages.append(tool_result(result))
      else:
          return response.text   # agent is done

That's it. The entire agent loop is a while loop.

What Claude Code / Cursor / Devin adds on top:
  - Streaming so you see output as it generates
  - Code execution in a sandbox
  - File system read/write
  - Terminal command execution
  - Context window management (what to keep, what to drop)
  - Multi-turn conversation state
  - Error handling and retries
  - Cost tracking

None of this requires a framework.
It requires understanding the primitives.

The primitives:
  1. Tool definition   → tell the LLM what it can do
  2. Tool calling      → LLM returns tool_use blocks
  3. Tool execution    → you run the actual function
  4. Tool result       → you send the result back to LLM
  5. Loop              → repeat until no more tool calls
```

---

## L1 — Raw Tool Calling (No Frameworks)

### 1. Define and Call a Tool with Anthropic API
```python
# 01_raw_tool_call.py
import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

# Step 1: Define tools — tell the LLM what it can do
tools = [
    {
        "name": "calculator",
        "description": "Evaluate a mathematical expression and return the result.",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "Math expression to evaluate, e.g. '2 + 2', '10 * 5'"
                }
            },
            "required": ["expression"]
        }
    }
]

# Step 2: Call the LLM with tools
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[
        {"role": "user", "content": "What is 125 * 48 + 300?"}
    ]
)

print(f"Stop reason: {response.stop_reason}")
print(f"Content blocks: {len(response.content)}")

# Step 3: Check if LLM wants to use a tool
for block in response.content:
    print(f"Block type: {block.type}")
    if block.type == "tool_use":
        print(f"Tool name: {block.name}")
        print(f"Tool input: {block.input}")

        # Step 4: Execute the tool
        expression = block.input["expression"]
        try:
            result = eval(expression)
            tool_result = str(result)
        except Exception as e:
            tool_result = f"Error: {e}"

        print(f"Tool result: {tool_result}")

        # Step 5: Send result back to LLM
        final_response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=[
                {"role": "user", "content": "What is 125 * 48 + 300?"},
                {"role": "assistant", "content": response.content},
                {
                    "role": "user",
                    "content": [{
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": tool_result
                    }]
                }
            ]
        )
        print(f"\nFinal answer: {final_response.content[0].text}")
```

---

### 2. Multiple Tools
```python
# 02_multiple_tools.py
import anthropic
import math
import json
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

# Define multiple tools
tools = [
    {
        "name": "calculate",
        "description": "Perform mathematical calculations.",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {"type": "string", "description": "Math expression"}
            },
            "required": ["expression"]
        }
    },
    {
        "name": "get_word_count",
        "description": "Count words in a piece of text.",
        "input_schema": {
            "type": "object",
            "properties": {
                "text": {"type": "string", "description": "Text to count words in"}
            },
            "required": ["text"]
        }
    },
    {
        "name": "search_knowledge",
        "description": "Search the knowledge base for information about a topic.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "What to search for"}
            },
            "required": ["query"]
        }
    }
]

# Tool implementations
def execute_tool(name: str, inputs: dict) -> str:
    if name == "calculate":
        try:
            safe = {"sqrt": math.sqrt, "pi": math.pi, "abs": abs}
            return str(eval(inputs["expression"], {"__builtins__": {}}, safe))
        except Exception as e:
            return f"Error: {e}"

    elif name == "get_word_count":
        count = len(inputs["text"].split())
        return f"{count} words"

    elif name == "search_knowledge":
        kb = {
            "python": "Python is a high-level, interpreted programming language.",
            "fastapi": "FastAPI is a modern Python web framework with automatic docs.",
            "langchain": "LangChain is a framework for building LLM applications.",
        }
        query = inputs["query"].lower()
        for key, val in kb.items():
            if key in query:
                return val
        return "No information found in knowledge base."

    return "Unknown tool"

# Run one turn
def run_one_turn(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    # Handle tool calls
    if response.stop_reason == "tool_use":
        messages.append({"role": "assistant", "content": response.content})

        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                print(f"  🔧 {block.name}({block.input}) → {result}")
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                })

        messages.append({"role": "user", "content": tool_results})
        final = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )
        return final.content[0].text

    return response.content[0].text

# Test
questions = [
    "What is sqrt(225) + 100?",
    "Count the words in: 'The quick brown fox jumps over the lazy dog'",
    "What is FastAPI?",
]

for q in questions:
    print(f"\n❓ {q}")
    answer = run_one_turn(q)
    print(f"💬 {answer}")
```

---

## L2 — The Agent Loop

### 3. Agent Loop from Scratch
```python
# 03_agent_loop.py
# This is the core of every agent — a while loop
import anthropic
import math
import json
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

tools = [
    {
        "name": "calculate",
        "description": "Evaluate math expressions.",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {"type": "string"}
            },
            "required": ["expression"]
        }
    },
    {
        "name": "get_fact",
        "description": "Get a fact about a topic.",
        "input_schema": {
            "type": "object",
            "properties": {
                "topic": {"type": "string"}
            },
            "required": ["topic"]
        }
    }
]

def execute_tool(name: str, inputs: dict) -> str:
    if name == "calculate":
        try:
            return str(eval(inputs["expression"], {"__builtins__": {}, "sqrt": math.sqrt}))
        except Exception as e:
            return f"Error: {e}"
    elif name == "get_fact":
        facts = {
            "python": "Python was created by Guido van Rossum in 1991.",
            "neo4j":  "Neo4j was founded in 2007 in Sweden.",
            "rust":   "Rust was originally designed by Graydon Hoare at Mozilla in 2010.",
        }
        return facts.get(inputs["topic"].lower(), "No fact found.")
    return "Unknown tool"

def run_agent(user_message: str, max_iterations: int = 10) -> str:
    """
    The agent loop:
    1. Send messages to LLM
    2. If LLM calls tools → execute them → add results → repeat
    3. If LLM gives text response → return it
    """
    messages = [{"role": "user", "content": user_message}]
    iteration = 0

    print(f"\n🤖 Starting agent loop for: '{user_message}'")

    while iteration < max_iterations:
        iteration += 1
        print(f"\n  [Iteration {iteration}]")

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        print(f"  Stop reason: {response.stop_reason}")

        # Agent is done — return the answer
        if response.stop_reason == "end_turn":
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return "No text response."

        # Agent wants to use tools
        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    print(f"  🔧 {block.name}({block.input}) → {result}")
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            messages.append({"role": "user", "content": tool_results})
            continue

        break

    return "Max iterations reached."

# Test multi-step reasoning
questions = [
    "What is sqrt(144) * 5? Then add 100 to that result.",
    "Get a fact about Python, then tell me how many words are in that fact. Use calculate for the math.",
]

for q in questions:
    answer = run_agent(q)
    print(f"\n✅ Final: {answer}")
```

---

### 4. Agent with System Prompt and Conversation History
```python
# 04_agent_with_history.py
import anthropic
from typing import Optional
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

SYSTEM = """You are a helpful assistant with access to tools.
When you need to perform calculations or look up information, use your tools.
Always show your reasoning before using a tool."""

tools = [
    {
        "name": "python_eval",
        "description": "Execute a Python expression and return the result. Safe subset only.",
        "input_schema": {
            "type": "object",
            "properties": {
                "code": {"type": "string", "description": "Python expression to evaluate"}
            },
            "required": ["code"]
        }
    }
]

def safe_eval(code: str) -> str:
    import math
    safe_globals = {
        "__builtins__": {},
        "math": math,
        "len": len,
        "range": range,
        "sum": sum,
        "min": min,
        "max": max,
        "sorted": sorted,
        "abs": abs,
        "round": round,
    }
    try:
        result = eval(code, safe_globals)
        return str(result)
    except Exception as e:
        return f"Error: {type(e).__name__}: {e}"

class Agent:
    def __init__(self, system: str, tools: list, max_iter: int = 10):
        self.system = system
        self.tools = tools
        self.max_iter = max_iter
        self.history = []   # persists across turns
        self.total_tokens = 0

    def _execute_tool(self, name: str, inputs: dict) -> str:
        if name == "python_eval":
            return safe_eval(inputs["code"])
        return "Unknown tool"

    def chat(self, user_message: str) -> str:
        self.history.append({"role": "user", "content": user_message})

        for iteration in range(self.max_iter):
            response = client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=1024,
                system=self.system,
                tools=self.tools,
                messages=self.history
            )

            self.total_tokens += response.usage.input_tokens + response.usage.output_tokens

            if response.stop_reason == "end_turn":
                text = next((b.text for b in response.content if hasattr(b, "text")), "")
                self.history.append({"role": "assistant", "content": response.content})
                return text

            if response.stop_reason == "tool_use":
                self.history.append({"role": "assistant", "content": response.content})

                results = []
                for block in response.content:
                    if block.type == "tool_use":
                        result = self._execute_tool(block.name, block.input)
                        print(f"  🔧 {block.name}: {block.input.get('code', '')} → {result}")
                        results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result
                        })

                self.history.append({"role": "user", "content": results})

        return "Max iterations reached."

    def stats(self) -> dict:
        return {
            "turns": len([m for m in self.history if m["role"] == "user"]),
            "total_tokens": self.total_tokens,
            "history_length": len(self.history)
        }

# Multi-turn conversation
agent = Agent(SYSTEM, tools)

turns = [
    "What is 2 ** 10?",
    "Now multiply that by 3 and subtract 100.",   # remembers previous result
    "What is the sum of [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]?",
]

for turn in turns:
    print(f"\n👤 {turn}")
    reply = agent.chat(turn)
    print(f"🤖 {reply}")

print(f"\n📊 Stats: {agent.stats()}")
```

---

## L3 — Streaming

### 5. Streaming Agent Response
```python
# 05_streaming.py
# Streaming is how Claude Code shows output as it generates
# The user sees tokens as they arrive — not all at once
import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

tools = [
    {
        "name": "calculate",
        "description": "Evaluate a math expression.",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {"type": "string"}
            },
            "required": ["expression"]
        }
    }
]

def run_streaming_agent(user_message: str):
    """Stream the agent response — show tokens as they arrive."""
    messages = [{"role": "user", "content": user_message}]

    print(f"\n👤 {user_message}")
    print("🤖 ", end="", flush=True)

    while True:
        # Use streaming context manager
        with client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages
        ) as stream:
            tool_calls = []
            text_buffer = ""

            # Stream events as they arrive
            for event in stream:
                # Text delta — print as it arrives
                if hasattr(event, "type"):
                    if event.type == "content_block_delta":
                        if hasattr(event.delta, "text"):
                            print(event.delta.text, end="", flush=True)
                            text_buffer += event.delta.text
                        elif hasattr(event.delta, "partial_json"):
                            pass  # tool input accumulating

            # Get the final message after streaming
            final = stream.get_final_message()

        # Check stop reason
        if final.stop_reason == "end_turn":
            print()  # newline after streaming
            return

        if final.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": final.content})

            results = []
            for block in final.content:
                if block.type == "tool_use":
                    import math
                    try:
                        result = str(eval(block.input["expression"],
                                         {"__builtins__": {}, "sqrt": math.sqrt}))
                    except Exception as e:
                        result = f"Error: {e}"

                    print(f"\n  🔧 calculate({block.input['expression']}) → {result}")
                    results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            messages.append({"role": "user", "content": results})
            print("🤖 ", end="", flush=True)
            continue

        break

# Test streaming
run_streaming_agent("Explain what RAG is in 3 sentences.")
run_streaming_agent("What is sqrt(1764) + 200?")
```

---

## L4 — Code Execution

### 6. Safe Code Execution — The Core of Coding Agents
```python
# 06_code_execution.py
# Every coding agent (Devin, Claude Code, Cursor) needs to execute code
# This is how you do it safely
import subprocess
import tempfile
import os
import sys
from typing import tuple

def execute_python(code: str, timeout: int = 10) -> tuple[str, str, int]:
    """
    Execute Python code in a subprocess.
    Returns: (stdout, stderr, return_code)
    """
    with tempfile.NamedTemporaryFile(
        mode="w",
        suffix=".py",
        delete=False,
        encoding="utf-8"
    ) as f:
        f.write(code)
        tmp_path = f.name

    try:
        result = subprocess.run(
            [sys.executable, tmp_path],
            capture_output=True,
            text=True,
            timeout=timeout,
            # Safety: no network access via env manipulation etc.
        )
        return result.stdout, result.stderr, result.returncode
    except subprocess.TimeoutExpired:
        return "", "Execution timed out", 1
    except Exception as e:
        return "", str(e), 1
    finally:
        os.unlink(tmp_path)

def execute_shell(command: str, timeout: int = 10) -> tuple[str, str, int]:
    """Execute a shell command safely."""
    # Basic safety checks
    dangerous = ["rm -rf", "sudo", "chmod 777", "mkfs", "> /dev/", "dd if="]
    for d in dangerous:
        if d in command:
            return "", f"Blocked: dangerous command pattern '{d}'", 1

    try:
        result = subprocess.run(
            command,
            shell=True,
            capture_output=True,
            text=True,
            timeout=timeout,
            cwd=tempfile.mkdtemp()  # run in temp dir
        )
        return result.stdout, result.stderr, result.returncode
    except subprocess.TimeoutExpired:
        return "", "Command timed out", 1
    except Exception as e:
        return "", str(e), 1

# --- Demo ---
print("=== Python Execution ===\n")

code_examples = [
    # Simple output
    "print('Hello from subprocess!')",

    # Math
    "import math\nresult = math.factorial(10)\nprint(f'10! = {result}')",

    # Multi-line with data structures
    """
data = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]
unique_sorted = sorted(set(data))
print(f"Unique sorted: {unique_sorted}")
print(f"Sum: {sum(data)}")
print(f"Mean: {sum(data)/len(data):.2f}")
""",
    # Error case
    "print(1/0)",
]

for code in code_examples:
    stdout, stderr, code_return = execute_python(code.strip())
    print(f"Code: {code.strip()[:50]}...")
    if stdout: print(f"  ✅ Output: {stdout.strip()}")
    if stderr: print(f"  ❌ Error: {stderr.strip()}")
    print()
```

---

### 7. Coding Agent — Execute Python from LLM
```python
# 07_coding_agent.py
import anthropic
import subprocess
import tempfile
import os
import sys
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

def run_python(code: str) -> str:
    """Execute Python code, return combined output."""
    with tempfile.NamedTemporaryFile(mode="w", suffix=".py", delete=False) as f:
        f.write(code)
        path = f.name
    try:
        result = subprocess.run(
            [sys.executable, path],
            capture_output=True, text=True, timeout=15
        )
        output = ""
        if result.stdout: output += result.stdout
        if result.stderr: output += f"\nSTDERR: {result.stderr}"
        return output.strip() or "No output produced."
    except subprocess.TimeoutExpired:
        return "ERROR: Code execution timed out (15s limit)"
    except Exception as e:
        return f"ERROR: {e}"
    finally:
        os.unlink(path)

SYSTEM = """You are an expert Python programmer and coding assistant.
When asked to solve a problem, write Python code and use the run_python tool to execute it.
Always test your code. If it fails, debug it and try again.
Show the code you're running and explain what it does."""

tools = [{
    "name": "run_python",
    "description": "Execute Python code and return the output. Use this to test solutions.",
    "input_schema": {
        "type": "object",
        "properties": {
            "code": {
                "type": "string",
                "description": "Python code to execute"
            }
        },
        "required": ["code"]
    }
}]

def coding_agent(task: str, max_iter: int = 8) -> str:
    messages = [{"role": "user", "content": task}]
    print(f"\n🤖 Working on: {task}\n")

    for i in range(max_iter):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system=SYSTEM,
            tools=tools,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            return next(
                (b.text for b in response.content if hasattr(b, "text")), ""
            )

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})
            results = []

            for block in response.content:
                if hasattr(block, "text"):
                    print(f"💭 {block.text[:200]}")

                if block.type == "tool_use":
                    print(f"\n```python\n{block.input['code']}\n```")
                    output = run_python(block.input["code"])
                    print(f"Output: {output}\n")

                    results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": output
                    })

            messages.append({"role": "user", "content": results})

    return "Max iterations reached."

# Tasks for the coding agent
tasks = [
    "Write a function to check if a number is prime. Test it with 17, 20, and 97.",
    "Create a function that returns the Fibonacci sequence up to n terms. Show first 10.",
]

for task in tasks:
    result = coding_agent(task)
    print(f"\n✅ Done: {result[:200]}")
    print("="*60)
```

---

## L5 — File System Agent

### 8. Read and Write Files
```python
# 08_file_agent.py
import anthropic
import os
import json
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

# Work in a safe sandbox directory
WORKSPACE = Path("./agent_workspace")
WORKSPACE.mkdir(exist_ok=True)

def read_file(path: str) -> str:
    """Read a file from the workspace."""
    full_path = WORKSPACE / path
    if not full_path.exists():
        return f"Error: File '{path}' not found"
    if not str(full_path.resolve()).startswith(str(WORKSPACE.resolve())):
        return "Error: Access denied — path outside workspace"
    try:
        return full_path.read_text(encoding="utf-8")
    except Exception as e:
        return f"Error reading file: {e}"

def write_file(path: str, content: str) -> str:
    """Write content to a file in the workspace."""
    full_path = WORKSPACE / path
    if not str(full_path.resolve()).startswith(str(WORKSPACE.resolve())):
        return "Error: Access denied — path outside workspace"
    full_path.parent.mkdir(parents=True, exist_ok=True)
    full_path.write_text(content, encoding="utf-8")
    return f"Written {len(content)} characters to {path}"

def list_files(directory: str = ".") -> str:
    """List files in a workspace directory."""
    full_path = WORKSPACE / directory
    if not full_path.exists():
        return f"Directory '{directory}' not found"
    files = []
    for item in full_path.iterdir():
        size = item.stat().st_size if item.is_file() else 0
        files.append(f"{'📁' if item.is_dir() else '📄'} {item.name} ({size} bytes)")
    return "\n".join(files) if files else "Empty directory"

def delete_file(path: str) -> str:
    """Delete a file from the workspace."""
    full_path = WORKSPACE / path
    if not full_path.exists():
        return f"Error: File '{path}' not found"
    if not str(full_path.resolve()).startswith(str(WORKSPACE.resolve())):
        return "Error: Access denied"
    full_path.unlink()
    return f"Deleted {path}"

tools = [
    {
        "name": "read_file",
        "description": "Read the contents of a file from the workspace.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path relative to workspace"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to a file in the workspace.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path"},
                "content": {"type": "string", "description": "Content to write"}
            },
            "required": ["path", "content"]
        }
    },
    {
        "name": "list_files",
        "description": "List files in a workspace directory.",
        "input_schema": {
            "type": "object",
            "properties": {
                "directory": {"type": "string", "default": "."}
            }
        }
    }
]

TOOL_MAP = {"read_file": read_file, "write_file": write_file, "list_files": list_files}

def file_agent(task: str) -> str:
    messages = [{"role": "user", "content": task}]

    for _ in range(10):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system="You are a file management assistant. Use tools to read, write, and organise files in the workspace.",
            tools=tools,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            return next((b.text for b in response.content if hasattr(b, "text")), "")

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})
            results = []
            for block in response.content:
                if block.type == "tool_use":
                    fn = TOOL_MAP.get(block.name, lambda **k: "Unknown tool")
                    result = fn(**block.input)
                    print(f"  🔧 {block.name}({block.input}) → {result[:80]}")
                    results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })
            messages.append({"role": "user", "content": results})

    return "Max iterations reached."

# Test
print(file_agent("Create a file called 'notes.txt' with 3 bullet points about Python."))
print(file_agent("Read the notes.txt file and summarise it."))
print(file_agent("List all files in the workspace."))
```

---

## L6 — Context Window Management

### 9. Managing Long Conversations
```python
# 09_context_management.py
# Long conversations hit the context window limit
# Real agents (Claude Code, Cursor) compress or summarise old history
import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

MAX_HISTORY_TOKENS = 4000    # keep under this
SUMMARY_THRESHOLD  = 3000    # summarise when we exceed this

def count_tokens_approx(messages: list) -> int:
    """Rough token count: 1 token ≈ 4 characters."""
    total = sum(
        len(str(m.get("content", ""))) // 4
        for m in messages
    )
    return total

def summarise_old_messages(messages: list, keep_last: int = 4) -> list:
    """Compress old messages into a summary, keep recent ones."""
    if len(messages) <= keep_last:
        return messages

    old = messages[:-keep_last]
    recent = messages[-keep_last:]

    # Build transcript of old messages
    transcript = []
    for m in old:
        role = m["role"].upper()
        content = str(m.get("content", ""))[:500]
        transcript.append(f"{role}: {content}")

    summary_response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": f"""Summarise this conversation history in 3-4 bullet points.
Focus on: key facts established, decisions made, problems solved.

{chr(10).join(transcript)}

Summary:"""
        }]
    )
    summary = summary_response.content[0].text

    # Return: summary as system context + recent messages
    summary_message = {
        "role": "user",
        "content": f"[Earlier conversation summary: {summary}]"
    }
    summary_ack = {
        "role": "assistant",
        "content": "Understood. I have the context from our earlier discussion."
    }

    return [summary_message, summary_ack] + recent

class ContextAwareAgent:
    def __init__(self, system: str):
        self.system = system
        self.history = []

    def _maybe_compress(self):
        tokens = count_tokens_approx(self.history)
        if tokens > SUMMARY_THRESHOLD:
            print(f"  [Compressing history: {tokens} tokens → summarising...]")
            self.history = summarise_old_messages(self.history)
            new_tokens = count_tokens_approx(self.history)
            print(f"  [Compressed to ~{new_tokens} tokens]")

    def chat(self, user_message: str) -> str:
        self.history.append({"role": "user", "content": user_message})
        self._maybe_compress()

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=512,
            system=self.system,
            messages=self.history
        )

        reply = response.content[0].text
        self.history.append({"role": "assistant", "content": reply})

        tokens = count_tokens_approx(self.history)
        print(f"  [History: {len(self.history)} messages, ~{tokens} tokens]")

        return reply

# Demo
agent = ContextAwareAgent("You are a helpful coding tutor. Remember everything discussed.")

turns = [
    "My name is Vishal and I'm learning Python.",
    "I'm working on a project using Neo4j and LangGraph.",
    "My biggest challenge right now is understanding graph traversal.",
    "Can you give me a simple Cypher query example?",
    "What was my name again?",
    "What project am I working on?",
]

for turn in turns:
    print(f"\n👤 {turn}")
    reply = agent.chat(turn)
    print(f"🤖 {reply[:200]}")
```

---

## L7 — Planning Agent

### 10. Plan → Execute → Reflect
```python
# 10_planning_agent.py
# Real agents don't just react — they plan, execute, then reflect
import anthropic
import json
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

def llm(messages: list, system: str = None, max_tokens: int = 1024) -> str:
    kwargs = {"model": "claude-sonnet-4-6", "max_tokens": max_tokens, "messages": messages}
    if system:
        kwargs["system"] = system
    response = client.messages.create(**kwargs)
    return response.content[0].text

def plan(task: str) -> list[dict]:
    """Break a task into steps."""
    prompt = f"""Break this task into 3-5 concrete steps.
Return ONLY a JSON array like:
[{{"step": 1, "action": "what to do", "tool": "python_eval or none"}}]

Task: {task}"""
    response = llm([{"role": "user", "content": prompt}])
    try:
        return json.loads(response.strip())
    except:
        return [{"step": 1, "action": task, "tool": "none"}]

def execute_step(step: dict) -> str:
    """Execute a single planned step."""
    if step.get("tool") == "python_eval":
        code_prompt = f"Write Python code to: {step['action']}\nReturn ONLY the code, no explanation."
        code = llm([{"role": "user", "content": code_prompt}])
        code = code.replace("```python", "").replace("```", "").strip()
        try:
            exec_globals = {}
            exec(code, exec_globals)
            return f"Executed: {code[:100]}..."
        except Exception as e:
            return f"Error: {e}"
    else:
        response = llm([{"role": "user", "content": f"Complete this step: {step['action']}"}])
        return response[:300]

def reflect(task: str, steps: list, results: list) -> str:
    """Reflect on what was done and summarise."""
    steps_summary = "\n".join(
        f"Step {s['step']}: {s['action']} → {r[:100]}"
        for s, r in zip(steps, results)
    )
    prompt = f"""Task: {task}

Execution log:
{steps_summary}

Write a brief summary of what was accomplished and any key findings."""

    return llm([{"role": "user", "content": prompt}])

def planning_agent(task: str) -> str:
    print(f"\n📋 Task: {task}")

    # Phase 1: Plan
    print("\n[PLANNING]")
    steps = plan(task)
    for s in steps:
        print(f"  Step {s['step']}: {s['action']}")

    # Phase 2: Execute
    print("\n[EXECUTING]")
    results = []
    for step in steps:
        print(f"\n  ▶ Step {step['step']}: {step['action']}")
        result = execute_step(step)
        print(f"    ✓ {result[:120]}")
        results.append(result)

    # Phase 3: Reflect
    print("\n[REFLECTING]")
    summary = reflect(task, steps, results)
    print(f"  {summary[:300]}")

    return summary

# Test
planning_agent("Research what RAG is, explain it simply, and give a one-line summary")
```

---

## L8 — Multi-Agent Without Frameworks

### 11. Orchestrator → Worker Pattern
```python
# 11_multi_agent_raw.py
# How real multi-agent systems work — no CrewAI, no LangGraph
# Just function calls and message passing
import anthropic
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

def call_agent(system: str, task: str, max_tokens: int = 512) -> str:
    """Call a specialised agent with a specific system prompt."""
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=max_tokens,
        system=system,
        messages=[{"role": "user", "content": task}]
    )
    return response.content[0].text

# Specialised agents — each is just a system prompt
RESEARCHER = """You are a research specialist. Your job is to find and
summarise key facts on any topic. Be accurate, concise, and structured.
Output bullet points of key findings."""

WRITER = """You are a technical writer. You take research notes and turn
them into clean, readable content for developers. Write in a clear,
direct style. No fluff."""

CRITIC = """You are a senior editor and critic. Review content for clarity,
accuracy, and completeness. Be specific about what's good and what needs
improvement. Give a rating from 1-10."""

SUMMARISER = """You are a summarisation expert. Condense content into its
most essential points. Be brutally brief. Max 3 sentences."""

def run_pipeline(topic: str) -> dict:
    """
    Orchestrate multiple agents:
    Research → Write → Critique → Summarise
    """
    print(f"\n🔄 Pipeline starting for: '{topic}'\n")

    # Step 1: Research
    print("1️⃣  Researcher working...")
    research = call_agent(
        RESEARCHER,
        f"Research this topic and find 5 key facts: {topic}"
    )
    print(f"   Done ({len(research)} chars)\n")

    # Step 2: Write — pass research output
    print("2️⃣  Writer working...")
    article = call_agent(
        WRITER,
        f"Turn these research notes into a 3-paragraph article:\n\n{research}"
    )
    print(f"   Done ({len(article)} chars)\n")

    # Step 3: Critique — pass article
    print("3️⃣  Critic reviewing...")
    critique = call_agent(
        CRITIC,
        f"Review this article:\n\n{article}"
    )
    print(f"   Done\n")

    # Step 4: Summarise — pass final article
    print("4️⃣  Summariser condensing...")
    summary = call_agent(
        SUMMARISER,
        f"Summarise in max 3 sentences:\n\n{article}"
    )
    print(f"   Done\n")

    return {
        "research": research,
        "article": article,
        "critique": critique,
        "summary": summary
    }

results = run_pipeline("GraphRAG and how it extends traditional RAG")

print("\n" + "="*60)
print("FINAL SUMMARY:")
print(results["summary"])
print("\nCRITIQUE HIGHLIGHTS:")
print(results["critique"][:300])
```

---

## L9 — CLI Agent (Like Claude Code)

### 12. Interactive CLI Agent
```python
# 12_cli_agent.py
# This is the pattern behind Claude Code and similar tools
# An interactive loop where the user types and the agent responds with tool use
import anthropic
import subprocess
import sys
import os
import tempfile
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

WORKSPACE = Path("./cli_workspace")
WORKSPACE.mkdir(exist_ok=True)

SYSTEM = """You are an expert AI coding assistant running in a terminal.
You have tools to:
- Run Python code
- Read and write files
- Execute shell commands

When given a coding task:
1. Think through the approach
2. Write and test code using run_python
3. Save important results to files
4. Explain what you did

Be concise. Show code. Test everything."""

tools = [
    {
        "name": "run_python",
        "description": "Execute Python code. Returns stdout and stderr.",
        "input_schema": {
            "type": "object",
            "properties": {"code": {"type": "string"}},
            "required": ["code"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to a file in the workspace.",
        "input_schema": {
            "type": "object",
            "properties": {
                "filename": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["filename", "content"]
        }
    },
    {
        "name": "read_file",
        "description": "Read a file from the workspace.",
        "input_schema": {
            "type": "object",
            "properties": {"filename": {"type": "string"}},
            "required": ["filename"]
        }
    },
    {
        "name": "list_files",
        "description": "List files in the workspace.",
        "input_schema": {
            "type": "object",
            "properties": {}
        }
    }
]

def run_python(code: str) -> str:
    with tempfile.NamedTemporaryFile(mode="w", suffix=".py", delete=False) as f:
        f.write(code)
        path = f.name
    try:
        r = subprocess.run([sys.executable, path], capture_output=True, text=True, timeout=15)
        out = r.stdout
        if r.stderr: out += f"\n[stderr]: {r.stderr}"
        return out.strip() or "(no output)"
    except subprocess.TimeoutExpired:
        return "ERROR: Timed out"
    finally:
        os.unlink(path)

def execute_tool(name: str, inputs: dict) -> str:
    if name == "run_python":
        return run_python(inputs["code"])
    elif name == "write_file":
        path = WORKSPACE / inputs["filename"]
        path.write_text(inputs["content"])
        return f"Written to {inputs['filename']}"
    elif name == "read_file":
        path = WORKSPACE / inputs["filename"]
        return path.read_text() if path.exists() else "File not found"
    elif name == "list_files":
        files = list(WORKSPACE.iterdir())
        return "\n".join(f.name for f in files) if files else "(empty)"
    return "Unknown tool"

def agent_turn(messages: list) -> str:
    """Run one agent turn — may call multiple tools."""
    for _ in range(10):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system=SYSTEM,
            tools=tools,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            return next((b.text for b in response.content if hasattr(b, "text")), "")

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})
            results = []
            for block in response.content:
                if hasattr(block, "text") and block.text:
                    print(f"\n  💭 {block.text[:200]}")
                if block.type == "tool_use":
                    print(f"\n  🔧 {block.name}(", end="")
                    if "code" in block.input:
                        print(f"\n```python\n{block.input['code']}\n```\n  )", end="")
                    else:
                        print(f"{block.input})", end="")
                    result = execute_tool(block.name, block.input)
                    print(f"\n  → {result[:200]}")
                    results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })
            messages.append({"role": "user", "content": results})

    return "Max iterations reached."

def main():
    """Interactive CLI loop."""
    messages = []
    print("\n🤖 CLI Agent (like Claude Code) — type 'exit' to quit\n")

    while True:
        try:
            user_input = input("You: ").strip()
        except (EOFError, KeyboardInterrupt):
            print("\nBye!")
            break

        if not user_input:
            continue
        if user_input.lower() in ("exit", "quit", "bye"):
            print("Goodbye!")
            break

        messages.append({"role": "user", "content": user_input})

        print("\nAgent:", end="", flush=True)
        reply = agent_turn(messages)
        messages.append({"role": "assistant", "content": reply})
        print(f"\n{reply}\n")

if __name__ == "__main__":
    main()
```

---

## L10 — Full Coding Agent

### 13. Production-Style Coding Agent — Everything Together
```python
# 13_full_coding_agent.py
# This is closest to how Claude Code, Cursor, OpenCode work
import anthropic
import subprocess
import sys, os, json, time
import tempfile
from pathlib import Path
from dataclasses import dataclass, field
from typing import Optional
from dotenv import load_dotenv

load_dotenv()
client = anthropic.Anthropic()

WORKSPACE = Path("./coding_agent_workspace")
WORKSPACE.mkdir(exist_ok=True)

# ── Tool definitions ───────────────────────────────────
TOOLS = [
    {
        "name": "run_python",
        "description": "Execute Python code. Returns stdout + stderr.",
        "input_schema": {
            "type": "object",
            "properties": {"code": {"type": "string", "description": "Python code to run"}},
            "required": ["code"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to a file.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["path", "content"]
        }
    },
    {
        "name": "read_file",
        "description": "Read file contents.",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"]
        }
    },
    {
        "name": "list_directory",
        "description": "List directory contents.",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string", "default": "."}},
        }
    },
    {
        "name": "run_tests",
        "description": "Run pytest on a file or directory.",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"]
        }
    }
]

# ── Tool execution ─────────────────────────────────────
def _run_python(code: str) -> str:
    with tempfile.NamedTemporaryFile(mode="w", suffix=".py", delete=False) as f:
        f.write(code); path = f.name
    try:
        r = subprocess.run([sys.executable, path], capture_output=True, text=True, timeout=30)
        out = r.stdout + (f"\n[STDERR]: {r.stderr}" if r.stderr else "")
        return out.strip() or "(no output)"
    except subprocess.TimeoutExpired:
        return "ERROR: Timed out after 30s"
    except Exception as e:
        return f"ERROR: {e}"
    finally:
        os.unlink(path)

def execute_tool(name: str, inputs: dict) -> str:
    if name == "run_python":
        return _run_python(inputs["code"])

    elif name == "write_file":
        p = WORKSPACE / inputs["path"]
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(inputs["content"])
        return f"✓ Written {len(inputs['content'])} chars to {inputs['path']}"

    elif name == "read_file":
        p = WORKSPACE / inputs["path"]
        return p.read_text() if p.exists() else f"Error: {inputs['path']} not found"

    elif name == "list_directory":
        p = WORKSPACE / inputs.get("path", ".")
        if not p.exists(): return f"Error: {inputs.get('path')} not found"
        items = sorted(p.iterdir(), key=lambda x: (not x.is_dir(), x.name))
        return "\n".join(
            f"{'📁' if i.is_dir() else '📄'} {i.name}" for i in items
        ) or "(empty)"

    elif name == "run_tests":
        p = WORKSPACE / inputs["path"]
        r = subprocess.run(
            [sys.executable, "-m", "pytest", str(p), "-v", "--tb=short"],
            capture_output=True, text=True, cwd=str(WORKSPACE)
        )
        return (r.stdout + r.stderr)[:2000]

    return f"Unknown tool: {name}"

# ── Agent state ────────────────────────────────────────
@dataclass
class AgentState:
    task: str
    messages: list = field(default_factory=list)
    tool_calls_made: int = 0
    tokens_used: int = 0
    start_time: float = field(default_factory=time.time)
    errors: list = field(default_factory=list)

    def elapsed(self) -> str:
        return f"{time.time() - self.start_time:.1f}s"

# ── Agent loop ─────────────────────────────────────────
SYSTEM = """You are an expert AI coding assistant.

Your approach:
1. UNDERSTAND the task fully before writing code
2. PLAN the solution in steps
3. WRITE clean, well-commented code
4. TEST it using run_python or run_tests
5. ITERATE if tests fail — fix and retry
6. SAVE important files to the workspace

Rules:
- Always test code before saying it's done
- If code fails, debug it — don't give up
- Write tests for non-trivial functions
- Be concise in explanations, thorough in code"""

def run_coding_agent(task: str, max_iter: int = 15) -> AgentState:
    state = AgentState(task=task)
    state.messages.append({"role": "user", "content": task})

    print(f"\n{'='*60}")
    print(f"🎯 Task: {task}")
    print(f"{'='*60}\n")

    for iteration in range(max_iter):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2048,
            system=SYSTEM,
            tools=TOOLS,
            messages=state.messages
        )

        state.tokens_used += response.usage.input_tokens + response.usage.output_tokens

        # Done
        if response.stop_reason == "end_turn":
            text = next((b.text for b in response.content if hasattr(b, "text")), "")
            state.messages.append({"role": "assistant", "content": response.content})
            print(f"\n✅ COMPLETE [{state.elapsed()} | {state.tokens_used} tokens | {state.tool_calls_made} tool calls]")
            print(f"\n{text}")
            return state

        # Tool use
        if response.stop_reason == "tool_use":
            state.messages.append({"role": "assistant", "content": response.content})
            results = []

            for block in response.content:
                # Print reasoning
                if hasattr(block, "text") and block.text.strip():
                    print(f"💭 {block.text.strip()[:300]}\n")

                # Execute tool
                if block.type == "tool_use":
                    state.tool_calls_made += 1
                    print(f"🔧 [{state.tool_calls_made}] {block.name}")

                    if block.name == "run_python" and "code" in block.input:
                        code_preview = block.input["code"][:200]
                        print(f"```\n{code_preview}{'...' if len(block.input['code']) > 200 else ''}\n```")

                    result = execute_tool(block.name, block.input)
                    result_preview = result[:300] + ("..." if len(result) > 300 else "")
                    print(f"→ {result_preview}\n")

                    if "ERROR" in result:
                        state.errors.append(result)

                    results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            state.messages.append({"role": "user", "content": results})

    print(f"\n⚠️  Max iterations ({max_iter}) reached")
    return state

# ── Run tasks ──────────────────────────────────────────
if __name__ == "__main__":
    tasks = [
        """Write a Python module called 'data_utils.py' in the workspace with:
1. A function clean_text(text) that strips whitespace and lowercases
2. A function word_frequency(text) that returns a dict of word counts
3. A function top_n_words(text, n=5) that returns the top N words
Then write tests for all three functions and run them.""",

        """Create a simple CLI task manager as a Python script in the workspace.
It should support: add <task>, list, complete <id>, delete <id>
Save tasks to tasks.json. Test it by adding 3 tasks, completing one, then listing.""",
    ]

    for task in tasks[:1]:   # run first task
        state = run_coding_agent(task)
        print(f"\n📊 Summary: {state.tool_calls_made} tool calls | {state.tokens_used} tokens | {len(state.errors)} errors")
```

---

## Bonus Drills

### B1. Build a diff-aware file editor tool
```
Tool: edit_file(path, old_string, new_string)
Find old_string in the file and replace with new_string
Error if old_string not found or appears multiple times
```

### B2. Add token budget tracking to the agent
```
Track tokens used per turn
If approaching 80% of model context → summarise old messages
Print token usage after every agent response
```

### B3. Build a self-correcting code agent
```
If run_python returns an error:
  - Send error back to LLM with "fix this"
  - Retry up to 3 times
  - Report how many attempts it took
```

### B4. Build a two-agent code review pipeline
```
Agent 1 (Coder): write a Python function for any task
Agent 2 (Reviewer): review the code, list issues
Agent 1: fix the issues
No frameworks — just function calls and message passing
```

### B5. Build a minimal agent framework (your own LangGraph)
```
Class Agent with: system, tools, history, max_iter
Method: run(task) → executes the agent loop
Method: add_tool(fn, description, schema) → register a tool
Run it on 3 different tasks
```

### B6. Full coding agent from blank file
```
Build exercise 13 from memory:
Tools (run_python, write_file, read_file, list_directory, run_tests) +
AgentState + execute_tool + agent loop + streaming tool output
Give it a real task: "build a word counter CLI tool with tests"
```

---

## The Core Primitives Cheat Sheet

```python
import anthropic
client = anthropic.Anthropic()

# 1. Define a tool
tool = {
    "name": "tool_name",
    "description": "What this tool does",
    "input_schema": {
        "type": "object",
        "properties": {
            "param": {"type": "string", "description": "param description"}
        },
        "required": ["param"]
    }
}

# 2. The agent loop
messages = [{"role": "user", "content": task}]
while True:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system=system_prompt,
        tools=[tool],
        messages=messages
    )

    if response.stop_reason == "end_turn":
        print(response.content[0].text)
        break

    if response.stop_reason == "tool_use":
        messages.append({"role": "assistant", "content": response.content})
        results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                })
        messages.append({"role": "user", "content": results})

# 3. Streaming
with client.messages.stream(...) as stream:
    for event in stream:
        if event.type == "content_block_delta":
            if hasattr(event.delta, "text"):
                print(event.delta.text, end="", flush=True)
    final = stream.get_final_message()
```

---

## Progress Tracker

| Level | Topic                                    | Done? | Time Taken |
|-------|------------------------------------------|-------|------------|
| L1    | Raw tool calling, Anthropic API directly |  [ ]  |            |
| L2    | Multiple tools, tool execution map       |  [ ]  |            |
| L3    | Agent loop from scratch                  |  [ ]  |            |
| L4    | Agent with history and stats             |  [ ]  |            |
| L5    | Streaming agent responses                |  [ ]  |            |
| L6    | Code execution in subprocess             |  [ ]  |            |
| L7    | Coding agent that writes + runs code     |  [ ]  |            |
| L8    | File system agent                        |  [ ]  |            |
| L9    | Context window management                |  [ ]  |            |
| L10   | Full production coding agent             |  [ ]  |            |
| Bonus | Diff editor, self-correcting, own framework |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 13 from memory —
> 5 tools, AgentState, execute_tool, the full agent loop with streaming output.
> Give it this task: "Write a Python function to check if a string is a palindrome,
> test it, and save it to a file."
> When it writes, tests, and saves the code without you touching anything —
> you understand how Claude Code works.
