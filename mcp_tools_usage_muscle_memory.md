# MCP Tools (Using MCP) — Muscle Memory Drills (L1 → L10)

> **What this sheet covers**
> Using MCP (Model Context Protocol) as a CLIENT — connecting to MCP servers,
> calling their tools, and wiring them into LangChain/LangGraph agents.
>
> **Install:**
> ```
> pip install mcp langchain-mcp-adapters langgraph langchain-openai python-dotenv
> ```
> **Mental model:** MCP server = a process that exposes tools over a standard protocol.
> You connect to it, discover its tools, and give them to your LLM agent.

---

## THE CORE MENTAL MODEL — Read This First

```
MCP Server  → a running process (stdio or HTTP/SSE) that exposes tools
MCP Client  → your Python code that connects to the server
Tool        → a named function with a JSON schema (name, description, parameters)
Adapter     → langchain-mcp-adapters converts MCP tools → LangChain tools

Flow:
  Your agent
    → MCP Client connects to MCP Server
    → discovers available tools
    → LLM decides which tool to call
    → MCP Client sends tool call to server
    → server executes and returns result
    → LLM gets the result
```

---

## L1 — Understanding MCP Protocol

### 1. What an MCP Tool Looks Like (Schema)
```python
# You don't write this manually — servers expose it.
# But you must READ and understand it.

example_tool_schema = {
    "name": "read_file",
    "description": "Read the contents of a file at the given path",
    "inputSchema": {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "The absolute path to the file"
            }
        },
        "required": ["path"]
    }
}

example_tool_schema_2 = {
    "name": "search_web",
    "description": "Search the web for a query",
    "inputSchema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "The search query"
            },
            "max_results": {
                "type": "integer",
                "description": "Maximum number of results",
                "default": 5
            }
        },
        "required": ["query"]
    }
}

print("Tool:", example_tool_schema["name"])
print("Description:", example_tool_schema["description"])
print("Required params:", example_tool_schema["inputSchema"]["required"])
```

---

### 2. MCP Communication Types
```python
# MCP supports two transport types.
# You need to know which one a server uses before connecting.

# 1. STDIO transport — server runs as a local process
# You launch it as a subprocess and communicate via stdin/stdout.
stdio_server_config = {
    "command": "python",              # or "npx", "node", "uvx", etc.
    "args": ["my_mcp_server.py"],    # script to run
    "env": {"API_KEY": "abc123"}     # optional env vars
}

# 2. SSE (Server-Sent Events) / HTTP transport
# Server is already running at a URL.
sse_server_config = {
    "url": "http://localhost:8080/sse"
}

# In langchain-mcp-adapters you use:
# from mcp import ClientSession, StdioServerParameters
# from mcp.client.stdio import stdio_client

print("STDIO: launch process, communicate via stdin/stdout")
print("SSE: connect to running HTTP server via SSE stream")
```

---

## L2 — Connecting to an MCP Server (STDIO)

### 3. Connect to a Local MCP Server
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def connect_and_list_tools():
    # Define the server — this is a Python MCP server script
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"],  # replace with your server script
        env=None
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize the session
            await session.initialize()

            # List all available tools
            tools_response = await session.list_tools()
            tools = tools_response.tools

            print(f"Found {len(tools)} tools:")
            for tool in tools:
                print(f"  - {tool.name}: {tool.description}")

            return tools

# asyncio.run(connect_and_list_tools())
```

---

### 4. Call a Tool Directly via MCP Session
```python
import asyncio
import json
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def call_tool_directly():
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Call a specific tool with arguments
            result = await session.call_tool(
                name="add",                    # tool name
                arguments={"a": 15, "b": 27}  # tool arguments as dict
            )

            # Result is a list of content blocks
            print("Raw result:", result)
            print("Content:", result.content[0].text)

# asyncio.run(call_tool_directly())
```

---

## L3 — MCP + LangChain Integration

### 5. Convert MCP Tools to LangChain Tools
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain_mcp_adapters.tools import load_mcp_tools

async def get_langchain_tools():
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # This converts MCP tools → LangChain BaseTool objects
            tools = await load_mcp_tools(session)

            print(f"LangChain tools loaded: {len(tools)}")
            for t in tools:
                print(f"  Tool: {t.name}")
                print(f"  Desc: {t.description}")
                print(f"  Args: {t.args}")
                print()

            return tools

# asyncio.run(get_langchain_tools())
```

---

### 6. Use MCP Tools in a LangChain Agent
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from langchain_mcp_adapters.tools import load_mcp_tools
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from dotenv import load_dotenv

load_dotenv()

async def run_agent_with_mcp():
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Step 1: Load MCP tools as LangChain tools
            tools = await load_mcp_tools(session)

            # Step 2: Create agent with those tools
            llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
            agent = create_react_agent(llm, tools)

            # Step 3: Run the agent
            result = await agent.ainvoke({
                "messages": [("user", "Add 42 and 58, then multiply the result by 3")]
            })

            print(result["messages"][-1].content)

# asyncio.run(run_agent_with_mcp())
```

---

## L4 — MultiServerMCPClient

### 7. Connect to Multiple MCP Servers at Once
```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from dotenv import load_dotenv

load_dotenv()

async def multi_server_agent():
    # Connect to multiple MCP servers simultaneously
    async with MultiServerMCPClient(
        {
            "math_server": {
                "command": "python",
                "args": ["math_server.py"],
                "transport": "stdio",
            },
            "file_server": {
                "command": "python",
                "args": ["file_server.py"],
                "transport": "stdio",
            },
            # SSE server example:
            # "web_server": {
            #     "url": "http://localhost:8080/sse",
            #     "transport": "sse",
            # }
        }
    ) as client:
        # Get ALL tools from ALL servers merged together
        tools = client.get_tools()

        print(f"Total tools from all servers: {len(tools)}")
        for t in tools:
            print(f"  {t.name}")

        llm = ChatOpenAI(model="gpt-4o-mini")
        agent = create_react_agent(llm, tools)

        result = await agent.ainvoke({
            "messages": [("user", "Calculate 100 / 4 and save the result to a file called result.txt")]
        })
        print(result["messages"][-1].content)

# asyncio.run(multi_server_agent())
```

---

### 8. MultiServerMCPClient with SSE Transport
```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from dotenv import load_dotenv

load_dotenv()

async def sse_agent():
    # SSE transport — server is already running at a URL
    async with MultiServerMCPClient(
        {
            "my_api_server": {
                "url": "http://localhost:8000/mcp",
                "transport": "sse",
                # optionally pass headers for auth:
                # "headers": {"Authorization": "Bearer my-token"}
            }
        }
    ) as client:
        tools = client.get_tools()
        print("Tools from SSE server:", [t.name for t in tools])

        llm = ChatOpenAI(model="gpt-4o-mini")
        agent = create_react_agent(llm, tools)

        result = await agent.ainvoke({
            "messages": [("user", "Use the available tools to answer: what can you do?")]
        })
        print(result["messages"][-1].content)

# asyncio.run(sse_agent())
```

---

## L5 — Wiring MCP into LangGraph

### 9. MCP Tools Inside a LangGraph Graph
```python
import asyncio
from typing import TypedDict, Annotated
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, BaseMessage
import operator
from dotenv import load_dotenv

load_dotenv()

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]

async def build_and_run_graph():
    async with MultiServerMCPClient(
        {
            "tools_server": {
                "command": "python",
                "args": ["my_server.py"],
                "transport": "stdio",
            }
        }
    ) as client:
        tools = client.get_tools()
        tool_node = ToolNode(tools)

        llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

        def agent_node(state: AgentState) -> dict:
            response = llm.invoke(state["messages"])
            return {"messages": [response]}

        def route(state: AgentState) -> str:
            last = state["messages"][-1]
            if hasattr(last, "tool_calls") and last.tool_calls:
                return "tools"
            return "__end__"

        graph = StateGraph(AgentState)
        graph.add_node("agent", agent_node)
        graph.add_node("tools", tool_node)
        graph.set_entry_point("agent")
        graph.add_conditional_edges("agent", route)
        graph.add_edge("tools", "agent")

        app = graph.compile()

        result = await app.ainvoke({
            "messages": [HumanMessage(content="What tools do you have available?")]
        })
        print(result["messages"][-1].content)

# asyncio.run(build_and_run_graph())
```

---

## L6 — Inspecting and Debugging MCP Tools

### 10. Inspect Tool Schemas Before Using
```python
import asyncio
import json
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def inspect_tools():
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            tools_resp = await session.list_tools()

            for tool in tools_resp.tools:
                print(f"\n{'='*40}")
                print(f"Name:        {tool.name}")
                print(f"Description: {tool.description}")
                print(f"Schema:")
                print(json.dumps(tool.inputSchema, indent=2))

# asyncio.run(inspect_tools())
```

---

### 11. Error Handling When Calling MCP Tools
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def safe_tool_call(session: ClientSession, tool_name: str, args: dict):
    try:
        result = await session.call_tool(name=tool_name, arguments=args)

        if result.isError:
            print(f"Tool returned error: {result.content}")
            return None

        # Extract text content from result
        text_contents = [c.text for c in result.content if hasattr(c, "text")]
        return "\n".join(text_contents)

    except Exception as e:
        print(f"Failed to call tool '{tool_name}': {e}")
        return None

async def main():
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Good call
            result = await safe_tool_call(session, "add", {"a": 10, "b": 20})
            print("Result:", result)

            # Bad call — wrong args
            result2 = await safe_tool_call(session, "add", {"x": 10})
            print("Result2:", result2)

            # Tool that doesn't exist
            result3 = await safe_tool_call(session, "nonexistent_tool", {})
            print("Result3:", result3)

# asyncio.run(main())
```

---

## L7 — Prompts and Resources (MCP Beyond Tools)

### 12. MCP Prompts
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def use_mcp_prompts():
    """
    MCP servers can also expose PROMPTS — reusable prompt templates.
    Less common than tools, but useful for standardized workflows.
    """
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # List available prompts
            prompts_resp = await session.list_prompts()
            print("Available prompts:")
            for p in prompts_resp.prompts:
                print(f"  - {p.name}: {p.description}")

            # Get a specific prompt with arguments
            if prompts_resp.prompts:
                prompt_name = prompts_resp.prompts[0].name
                prompt = await session.get_prompt(
                    name=prompt_name,
                    arguments={"topic": "LangGraph"}
                )
                print("\nPrompt content:")
                for msg in prompt.messages:
                    print(f"  [{msg.role}]: {msg.content.text}")

# asyncio.run(use_mcp_prompts())
```

---

### 13. MCP Resources
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def use_mcp_resources():
    """
    MCP servers can expose RESOURCES — readable data like files, DB records, configs.
    Resources have URIs, tools have schemas.
    """
    server_params = StdioServerParameters(
        command="python",
        args=["my_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # List available resources
            resources_resp = await session.list_resources()
            print("Available resources:")
            for r in resources_resp.resources:
                print(f"  URI:  {r.uri}")
                print(f"  Name: {r.name}")
                print(f"  Type: {r.mimeType}")
                print()

            # Read a specific resource by URI
            if resources_resp.resources:
                uri = resources_resp.resources[0].uri
                content = await session.read_resource(uri)
                print(f"Content of {uri}:")
                for c in content.contents:
                    print(c.text if hasattr(c, "text") else c)

# asyncio.run(use_mcp_resources())
```

---

## L8 — Real MCP Servers (Public)

### 14. Using the Filesystem MCP Server
```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from dotenv import load_dotenv

load_dotenv()

# Install: npm install -g @modelcontextprotocol/server-filesystem
# This is an OFFICIAL MCP server for file system operations

async def filesystem_agent():
    async with MultiServerMCPClient(
        {
            "filesystem": {
                "command": "npx",
                "args": [
                    "@modelcontextprotocol/server-filesystem",
                    "/tmp"  # root directory the server can access
                ],
                "transport": "stdio",
            }
        }
    ) as client:
        tools = client.get_tools()
        print("Filesystem tools:", [t.name for t in tools])

        llm = ChatOpenAI(model="gpt-4o-mini")
        agent = create_react_agent(llm, tools)

        result = await agent.ainvoke({
            "messages": [("user", "List all files in /tmp and tell me how many there are")]
        })
        print(result["messages"][-1].content)

# asyncio.run(filesystem_agent())
```

---

### 15. Using Multiple Official MCP Servers
```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from dotenv import load_dotenv
import os

load_dotenv()

# Install servers:
# npm install -g @modelcontextprotocol/server-filesystem
# npm install -g @modelcontextprotocol/server-memory

async def multi_official_servers():
    async with MultiServerMCPClient(
        {
            "filesystem": {
                "command": "npx",
                "args": ["@modelcontextprotocol/server-filesystem", "/tmp"],
                "transport": "stdio",
            },
            "memory": {
                "command": "npx",
                "args": ["@modelcontextprotocol/server-memory"],
                "transport": "stdio",
            },
        }
    ) as client:
        tools = client.get_tools()
        print(f"Total tools: {len(tools)}")
        for t in tools:
            print(f"  {t.name}")

        llm = ChatOpenAI(model="gpt-4o-mini")
        agent = create_react_agent(llm, tools)

        result = await agent.ainvoke({
            "messages": [("user", "Remember that my name is Vishal and I am building an image memory system")]
        })
        print(result["messages"][-1].content)

# asyncio.run(multi_official_servers())
```

---

## L9 — Async Patterns for MCP

### 16. Reusable MCP Agent Context Manager
```python
import asyncio
from contextlib import asynccontextmanager
from typing import AsyncGenerator
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from dotenv import load_dotenv

load_dotenv()

@asynccontextmanager
async def mcp_agent(server_configs: dict) -> AsyncGenerator:
    """Reusable context manager for MCP-powered agents."""
    async with MultiServerMCPClient(server_configs) as client:
        tools = client.get_tools()
        llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
        agent = create_react_agent(llm, tools)
        yield agent

async def main():
    configs = {
        "math": {
            "command": "python",
            "args": ["math_server.py"],
            "transport": "stdio",
        }
    }

    async with mcp_agent(configs) as agent:
        # Use agent multiple times within the same context
        r1 = await agent.ainvoke({"messages": [("user", "What is 15 * 15?")]})
        print("Q1:", r1["messages"][-1].content)

        r2 = await agent.ainvoke({"messages": [("user", "What is 144 / 12?")]})
        print("Q2:", r2["messages"][-1].content)

# asyncio.run(main())
```

---

### 17. FastAPI + MCP Agent Endpoint
```python
from fastapi import FastAPI
from pydantic import BaseModel
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from dotenv import load_dotenv

load_dotenv()

app = FastAPI(title="MCP Agent API")

SERVER_CONFIGS = {
    "tools": {
        "command": "python",
        "args": ["my_server.py"],
        "transport": "stdio",
    }
}

class QueryRequest(BaseModel):
    message: str

class QueryResponse(BaseModel):
    response: str
    tool_calls_made: int

@app.post("/agent/query", response_model=QueryResponse)
async def query_agent(request: QueryRequest):
    async with MultiServerMCPClient(SERVER_CONFIGS) as client:
        tools = client.get_tools()
        llm = ChatOpenAI(model="gpt-4o-mini")
        agent = create_react_agent(llm, tools)

        result = await agent.ainvoke({
            "messages": [("user", request.message)]
        })

        messages = result["messages"]
        response_text = messages[-1].content

        # Count tool calls
        tool_calls = sum(
            1 for m in messages
            if hasattr(m, "tool_calls") and m.tool_calls
        )

        return QueryResponse(response=response_text, tool_calls_made=tool_calls)

# Run: uvicorn this_file:app --reload
```

---

## L10 — Full MCP Agent System (Build from Memory)

### 18. Complete Async MCP Agent with LangGraph
```python
import asyncio
from typing import TypedDict, Annotated, Literal
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage, SystemMessage
import operator
from dotenv import load_dotenv

load_dotenv()

# ─── State ───────────────────────────────────────────────
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    iteration: int

# ─── Config ──────────────────────────────────────────────
SERVER_CONFIGS = {
    "filesystem": {
        "command": "npx",
        "args": ["@modelcontextprotocol/server-filesystem", "/tmp"],
        "transport": "stdio",
    },
    # Add more servers here
}

SYSTEM = """You are a helpful assistant with access to tools.
Think step by step. Use tools efficiently. Be concise."""

async def run_mcp_agent(user_message: str, thread_id: str = "default"):
    async with MultiServerMCPClient(SERVER_CONFIGS) as client:
        # Get tools from all servers
        tools = client.get_tools()
        tool_node = ToolNode(tools)

        llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).bind_tools(tools)

        # ─── Nodes ───────────────────────────────────────
        def agent_node(state: AgentState) -> dict:
            messages = [SystemMessage(content=SYSTEM)] + state["messages"]
            response = llm.invoke(messages)
            return {"messages": [response], "iteration": state["iteration"] + 1}

        # ─── Routing ─────────────────────────────────────
        def route(state: AgentState) -> Literal["tools", "__end__"]:
            if state["iteration"] > 8:
                return "__end__"
            last = state["messages"][-1]
            if hasattr(last, "tool_calls") and last.tool_calls:
                return "tools"
            return "__end__"

        # ─── Graph ───────────────────────────────────────
        graph = StateGraph(AgentState)
        graph.add_node("agent", agent_node)
        graph.add_node("tools", tool_node)
        graph.set_entry_point("agent")
        graph.add_conditional_edges("agent", route)
        graph.add_edge("tools", "agent")

        checkpointer = MemorySaver()
        app = graph.compile(checkpointer=checkpointer)

        config = {"configurable": {"thread_id": thread_id}}

        result = await app.ainvoke(
            {"messages": [HumanMessage(content=user_message)], "iteration": 0},
            config=config
        )

        return result["messages"][-1].content

async def main():
    response = await run_mcp_agent("List files in /tmp and count them")
    print(response)

# asyncio.run(main())
```

---

## Bonus Drills — Blank File Challenges

### B1. Write a function `list_server_tools(server_script: str) -> list[str]` that connects to a stdio MCP server and returns all tool names.

### B2. Write `call_tool_safely(session, name, args)` with full error handling: missing tool, wrong args, server error.

### B3. Wire a MultiServerMCPClient with 2 local servers into a LangGraph ReAct agent. Add MemorySaver.

### B4. Build a FastAPI endpoint `/tools` that returns all available tool names and descriptions from a connected MCP server.

### B5. Write an async function that calls 3 different MCP tools in sequence and accumulates their results into a summary.

### B6. Build an agent that connects to the official filesystem MCP server and can: list files, read a file, write a file.

### B7. Add session-level caching: connect once, store the tool list, reuse for multiple queries without reconnecting each time.

### B8. Build a CLI: `python agent_cli.py "your question"` that connects to an MCP server and prints the agent's response.

---

## Quick Reference

```python
# STDIO connection
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server = StdioServerParameters(command="python", args=["server.py"])
async with stdio_client(server) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()
        tools = await session.list_tools()
        result = await session.call_tool("tool_name", {"arg": "value"})

# MultiServerMCPClient (recommended)
from langchain_mcp_adapters.client import MultiServerMCPClient

async with MultiServerMCPClient({"server": {"command": "python", "args": ["s.py"], "transport": "stdio"}}) as client:
    tools = client.get_tools()   # returns list of LangChain BaseTool

# Use with LangGraph
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(llm, tools)
result = await agent.ainvoke({"messages": [("user", "hello")]})
```

---

## Progress Tracker

| Level | Topic                                  | Done? | Time Taken |
|-------|----------------------------------------|-------|------------|
| L1    | MCP Protocol, Schemas, Transport types |  [ ]  |            |
| L2    | Connect to STDIO Server, call tools    |  [ ]  |            |
| L3    | MCP → LangChain tools conversion       |  [ ]  |            |
| L4    | MultiServerMCPClient, SSE              |  [ ]  |            |
| L5    | MCP Tools inside LangGraph             |  [ ]  |            |
| L6    | Inspect, debug, error handling         |  [ ]  |            |
| L7    | Prompts and Resources                  |  [ ]  |            |
| L8    | Official MCP servers (filesystem etc.) |  [ ]  |            |
| L9    | Async patterns, FastAPI integration    |  [ ]  |            |
| L10   | Full agent system from scratch         |  [ ]  |            |
| Bonus | Blank File Challenges                  |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 18 completely from memory.
> MultiServerMCPClient → tools → LangGraph graph → MemorySaver → compile → invoke.
> When the agent actually uses a tool from the MCP server — you're done.
