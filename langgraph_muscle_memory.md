# LangGraph Muscle Memory — 50+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `python filename.py`
> - Install: `pip install langgraph langchain langchain-openai python-dotenv`
> - Set env: create `.env` with `OPENAI_API_KEY=your-key`
> - L1–L2 → State and graph basics
> - L3–L4 → Nodes, edges, conditions
> - L5–L6 → Tools and tool nodes
> - L7–L8 → Memory, checkpointing
> - L9–L10 → Full agent from scratch
> - Mental model: LangGraph = a state machine where nodes are functions and edges are transitions.

---

## THE CORE MENTAL MODEL — Read This First

```
State       → a TypedDict that flows through the graph
Node        → a Python function that receives State, returns updated State
Edge        → a connection from one node to another
Conditional → a function that reads State and decides WHICH node to go to next
Graph       → StateGraph(State) with nodes + edges compiled into a runnable
```

Every LangGraph program is:
1. Define your State
2. Write node functions
3. Add nodes to graph
4. Connect with edges
5. Compile and invoke

---

## L1 — State and Graph Basics

### 1. Simplest Possible Graph
```python
from typing import TypedDict
from langgraph.graph import StateGraph, END

# Step 1: Define State
class State(TypedDict):
    message: str

# Step 2: Define Nodes (functions that take State, return partial State)
def greet(state: State) -> State:
    return {"message": f"Hello! You said: {state['message']}"}

def shout(state: State) -> State:
    return {"message": state["message"].upper()}

# Step 3: Build Graph
graph = StateGraph(State)
graph.add_node("greet", greet)
graph.add_node("shout", shout)

# Step 4: Connect edges
graph.set_entry_point("greet")
graph.add_edge("greet", "shout")
graph.add_edge("shout", END)

# Step 5: Compile and run
app = graph.compile()
result = app.invoke({"message": "langgraph is awesome"})
print(result["message"])
```

---

### 2. State with Multiple Fields
```python
from typing import TypedDict
from langgraph.graph import StateGraph, END

class CounterState(TypedDict):
    count: int
    history: list
    done: bool

def increment(state: CounterState) -> CounterState:
    new_count = state["count"] + 1
    new_history = state["history"] + [new_count]
    return {"count": new_count, "history": new_history, "done": new_count >= 5}

def report(state: CounterState) -> CounterState:
    print(f"Final count: {state['count']}")
    print(f"History: {state['history']}")
    return state

graph = StateGraph(CounterState)
graph.add_node("increment", increment)
graph.add_node("report", report)
graph.set_entry_point("increment")
graph.add_edge("increment", "report")
graph.add_edge("report", END)

app = graph.compile()
result = app.invoke({"count": 0, "history": [], "done": False})
print(result)
```

---

### 3. Annotated State — List Accumulation
```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
import operator

# Annotated with operator.add means: when nodes return this field,
# the values are APPENDED not replaced
class LogState(TypedDict):
    input: str
    steps: Annotated[list, operator.add]  # accumulates across nodes
    result: str

def step_one(state: LogState) -> dict:
    return {"steps": ["step_one: started"]}

def step_two(state: LogState) -> dict:
    return {"steps": ["step_two: processed"]}

def step_three(state: LogState) -> dict:
    return {
        "steps": ["step_three: done"],
        "result": f"Processed: {state['input']}"
    }

graph = StateGraph(LogState)
graph.add_node("step_one", step_one)
graph.add_node("step_two", step_two)
graph.add_node("step_three", step_three)
graph.set_entry_point("step_one")
graph.add_edge("step_one", "step_two")
graph.add_edge("step_two", "step_three")
graph.add_edge("step_three", END)

app = graph.compile()
result = app.invoke({"input": "hello world", "steps": [], "result": ""})
print("Steps:", result["steps"])
print("Result:", result["result"])
```

---

## L2 — Conditional Edges (Routing)

### 4. Basic Conditional Edge
```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, END

class State(TypedDict):
    number: int
    result: str

def classify(state: State) -> Literal["positive", "negative", "zero"]:
    if state["number"] > 0:
        return "positive"
    elif state["number"] < 0:
        return "negative"
    else:
        return "zero"

def handle_positive(state: State) -> dict:
    return {"result": f"{state['number']} is positive"}

def handle_negative(state: State) -> dict:
    return {"result": f"{state['number']} is negative"}

def handle_zero(state: State) -> dict:
    return {"result": "It's zero!"}

graph = StateGraph(State)
graph.add_node("handle_positive", handle_positive)
graph.add_node("handle_negative", handle_negative)
graph.add_node("handle_zero", handle_zero)

graph.set_entry_point("classify")  # wait — classify is the routing function

# Actually, we need a router node first:
def router_node(state: State) -> dict:
    return state  # pass through

graph2 = StateGraph(State)
graph2.add_node("router", router_node)
graph2.add_node("handle_positive", handle_positive)
graph2.add_node("handle_negative", handle_negative)
graph2.add_node("handle_zero", handle_zero)

graph2.set_entry_point("router")
graph2.add_conditional_edges(
    "router",
    classify,
    {
        "positive": "handle_positive",
        "negative": "handle_negative",
        "zero": "handle_zero",
    }
)
graph2.add_edge("handle_positive", END)
graph2.add_edge("handle_negative", END)
graph2.add_edge("handle_zero", END)

app = graph2.compile()
for n in [5, -3, 0]:
    print(app.invoke({"number": n, "result": ""})["result"])
```

---

### 5. Loop with Conditional — Retry Pattern
```python
from typing import TypedDict, Literal, Annotated
from langgraph.graph import StateGraph, END
import operator
import random

class RetryState(TypedDict):
    task: str
    attempts: int
    success: bool
    logs: Annotated[list, operator.add]

def attempt_task(state: RetryState) -> dict:
    attempts = state["attempts"] + 1
    success = random.random() > 0.6  # 40% fail rate
    log = f"Attempt {attempts}: {'SUCCESS' if success else 'FAILED'}"
    print(log)
    return {"attempts": attempts, "success": success, "logs": [log]}

def should_retry(state: RetryState) -> Literal["retry", "done", "give_up"]:
    if state["success"]:
        return "done"
    elif state["attempts"] >= 4:
        return "give_up"
    else:
        return "retry"

def finish_success(state: RetryState) -> dict:
    return {"logs": [f"Completed after {state['attempts']} attempts"]}

def finish_fail(state: RetryState) -> dict:
    return {"logs": ["Gave up after max retries"]}

graph = StateGraph(RetryState)
graph.add_node("attempt", attempt_task)
graph.add_node("success", finish_success)
graph.add_node("fail", finish_fail)

graph.set_entry_point("attempt")
graph.add_conditional_edges(
    "attempt",
    should_retry,
    {"retry": "attempt", "done": "success", "give_up": "fail"}
)
graph.add_edge("success", END)
graph.add_edge("fail", END)

app = graph.compile()
result = app.invoke({"task": "deploy service", "attempts": 0, "success": False, "logs": []})
print("\nAll logs:", result["logs"])
```

---

## L3 — LLM Nodes

### 6. Single LLM Node
```python
from typing import TypedDict
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage
from dotenv import load_dotenv

load_dotenv()

class State(TypedDict):
    question: str
    answer: str

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def ask_llm(state: State) -> dict:
    response = llm.invoke([HumanMessage(content=state["question"])])
    return {"answer": response.content}

graph = StateGraph(State)
graph.add_node("ask", ask_llm)
graph.set_entry_point("ask")
graph.add_edge("ask", END)

app = graph.compile()
result = app.invoke({"question": "What is LangGraph in one sentence?", "answer": ""})
print("Q:", result["question"])
print("A:", result["answer"])
```

---

### 7. Message History in State
```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage
import operator
from dotenv import load_dotenv

load_dotenv()

class ChatState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]

llm = ChatOpenAI(model="gpt-4o-mini")

def chat_node(state: ChatState) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

graph = StateGraph(ChatState)
graph.add_node("chat", chat_node)
graph.set_entry_point("chat")
graph.add_edge("chat", END)

app = graph.compile()

# Turn 1
state = app.invoke({"messages": [HumanMessage(content="My name is Vishal.")]})
print("Bot:", state["messages"][-1].content)

# Turn 2 — history is preserved in messages
state = app.invoke({
    "messages": state["messages"] + [HumanMessage(content="What is my name?")]
})
print("Bot:", state["messages"][-1].content)
```

---

### 8. Multi-Node LLM Pipeline
```python
from typing import TypedDict
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv

load_dotenv()

class PipelineState(TypedDict):
    topic: str
    outline: str
    draft: str
    refined: str

llm = ChatOpenAI(model="gpt-4o-mini")

def create_outline(state: PipelineState) -> dict:
    response = llm.invoke([
        HumanMessage(content=f"Create a 3-point outline for a blog post about: {state['topic']}")
    ])
    return {"outline": response.content}

def write_draft(state: PipelineState) -> dict:
    response = llm.invoke([
        HumanMessage(content=f"Write a short blog post based on this outline:\n{state['outline']}")
    ])
    return {"draft": response.content}

def refine(state: PipelineState) -> dict:
    response = llm.invoke([
        HumanMessage(content=f"Improve this blog post. Make it more engaging:\n{state['draft']}")
    ])
    return {"refined": response.content}

graph = StateGraph(PipelineState)
graph.add_node("outline", create_outline)
graph.add_node("draft", write_draft)
graph.add_node("refine", refine)

graph.set_entry_point("outline")
graph.add_edge("outline", "draft")
graph.add_edge("draft", "refine")
graph.add_edge("refine", END)

app = graph.compile()
result = app.invoke({"topic": "Why AI agents need memory", "outline": "", "draft": "", "refined": ""})
print(result["refined"])
```

---

## L4 — Tools and ToolNode

### 9. Defining and Binding Tools
```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, BaseMessage
from langchain_core.tools import tool
import operator
from dotenv import load_dotenv

load_dotenv()

# Define tools with @tool decorator
@tool
def add(a: int, b: int) -> int:
    """Add two numbers together."""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    # Fake data for drill
    weather_data = {
        "pune": "28°C, partly cloudy",
        "mumbai": "31°C, humid",
        "delhi": "35°C, sunny"
    }
    return weather_data.get(city.lower(), f"Weather for {city}: 25°C, clear")

tools = [add, multiply, get_weather]
tool_node = ToolNode(tools)

llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

class State(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]

def call_llm(state: State) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

def should_use_tool(state: State) -> str:
    last = state["messages"][-1]
    if hasattr(last, "tool_calls") and last.tool_calls:
        return "tools"
    return END

graph = StateGraph(State)
graph.add_node("llm", call_llm)
graph.add_node("tools", tool_node)

graph.set_entry_point("llm")
graph.add_conditional_edges("llm", should_use_tool, {"tools": "tools", END: END})
graph.add_edge("tools", "llm")

app = graph.compile()
result = app.invoke({"messages": [HumanMessage(content="What is 15 + 27? Also what's the weather in Pune?")]})
print(result["messages"][-1].content)
```

---

### 10. Custom Tool with Schema
```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field
from typing import Optional

# Tool with explicit schema
class SearchInput(BaseModel):
    query: str = Field(description="The search query")
    max_results: int = Field(default=5, description="Max number of results")
    category: Optional[str] = Field(None, description="Filter by category")

@tool(args_schema=SearchInput)
def search_knowledge_base(query: str, max_results: int = 5, category: Optional[str] = None) -> list:
    """Search the internal knowledge base for relevant documents."""
    # Simulated results
    results = [
        {"id": i, "title": f"Doc {i}: {query}", "category": category or "general"}
        for i in range(1, max_results + 1)
    ]
    return results

@tool
def save_note(title: str, content: str) -> dict:
    """Save a note to the database."""
    return {"saved": True, "title": title, "id": hash(title) % 1000}

# Test tools directly
print(search_knowledge_base.invoke({"query": "AI memory systems", "max_results": 3}))
print(save_note.invoke({"title": "LangGraph notes", "content": "State machine for LLMs"}))
print("\nTool name:", search_knowledge_base.name)
print("Tool description:", search_knowledge_base.description)
```

---

## L5 — ReAct Agent Pattern

### 11. ReAct from Scratch
```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, BaseMessage, SystemMessage
from langchain_core.tools import tool
import operator
from dotenv import load_dotenv

load_dotenv()

@tool
def calculator(expression: str) -> str:
    """Evaluate a math expression. Example: '2 + 3 * 4'"""
    try:
        return str(eval(expression))
    except Exception as e:
        return f"Error: {e}"

@tool
def get_fact(topic: str) -> str:
    """Get a fun fact about a topic."""
    facts = {
        "python": "Python was named after Monty Python, not the snake.",
        "fastapi": "FastAPI is one of the fastest Python web frameworks.",
        "langgraph": "LangGraph models agent workflows as state machines."
    }
    return facts.get(topic.lower(), f"No fact found for {topic}")

tools = [calculator, get_fact]
tool_node = ToolNode(tools)
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).bind_tools(tools)

SYSTEM = """You are a helpful assistant with access to tools.
Think step by step. Use tools when needed."""

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]

def agent(state: AgentState) -> dict:
    messages = [SystemMessage(content=SYSTEM)] + state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}

def route(state: AgentState) -> Literal["tools", "__end__"]:
    last = state["messages"][-1]
    if hasattr(last, "tool_calls") and last.tool_calls:
        return "tools"
    return "__end__"

graph = StateGraph(AgentState)
graph.add_node("agent", agent)
graph.add_node("tools", tool_node)
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", route)
graph.add_edge("tools", "agent")

app = graph.compile()

result = app.invoke({
    "messages": [HumanMessage(content="What is (123 * 456) + 789? Also give me a fact about Python.")]
})
print(result["messages"][-1].content)
```

---

### 12. Prebuilt ReAct Agent
```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from dotenv import load_dotenv

load_dotenv()

@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Search results for '{query}': [result1, result2, result3]"

@tool
def get_stock_price(symbol: str) -> str:
    """Get the current stock price for a symbol."""
    prices = {"AAPL": "182.50", "GOOGL": "140.20", "MSFT": "378.90"}
    return f"{symbol}: ${prices.get(symbol.upper(), '100.00')}"

llm = ChatOpenAI(model="gpt-4o-mini")
tools = [search, get_stock_price]

# create_react_agent builds the entire ReAct loop for you
agent = create_react_agent(llm, tools)

result = agent.invoke({
    "messages": [("user", "What is the stock price of AAPL and MSFT?")]
})
print(result["messages"][-1].content)
```

---

## L6 — Memory and Checkpointing

### 13. In-Memory Checkpointer (Short-term Memory)
```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, BaseMessage
import operator
from dotenv import load_dotenv

load_dotenv()

class State(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]

llm = ChatOpenAI(model="gpt-4o-mini")

def chat(state: State) -> dict:
    return {"messages": [llm.invoke(state["messages"])]}

graph = StateGraph(State)
graph.add_node("chat", chat)
graph.set_entry_point("chat")
graph.add_edge("chat", END)

# MemorySaver persists state across invocations for the same thread_id
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# thread_id is like a conversation/session ID
config = {"configurable": {"thread_id": "session-vishal-001"}}

# Turn 1
app.invoke({"messages": [HumanMessage(content="My favorite language is Python.")]}, config=config)

# Turn 2 — it remembers!
result = app.invoke({"messages": [HumanMessage(content="What is my favorite language?")]}, config=config)
print(result["messages"][-1].content)

# Different thread_id = fresh memory
config2 = {"configurable": {"thread_id": "session-alice-001"}}
result2 = app.invoke({"messages": [HumanMessage(content="What is my favorite language?")]}, config=config2)
print(result2["messages"][-1].content)  # Won't know
```

---

### 14. Inspecting Graph State
```python
from langgraph.checkpoint.memory import MemorySaver
# Continuing from exercise 13 setup...

# After some conversation, inspect the saved state
config = {"configurable": {"thread_id": "session-vishal-001"}}

# Get the current state snapshot
snapshot = app.get_state(config)
print("Current state messages:", len(snapshot.values["messages"]))
print("Next node:", snapshot.next)

# Get state history
history = list(app.get_state_history(config))
print(f"\nTotal checkpoints saved: {len(history)}")
for i, h in enumerate(history[:3]):
    print(f"  Checkpoint {i}: {len(h.values['messages'])} messages")
```

---

## L7 — Subgraphs and Parallel Nodes

### 15. Subgraph Pattern
```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
import operator

# Inner graph — does one specific job
class SummaryState(TypedDict):
    text: str
    summary: str
    word_count: int

def count_words(state: SummaryState) -> dict:
    return {"word_count": len(state["text"].split())}

def summarize(state: SummaryState) -> dict:
    words = state["text"].split()
    summary = " ".join(words[:10]) + "..." if len(words) > 10 else state["text"]
    return {"summary": summary}

inner = StateGraph(SummaryState)
inner.add_node("count", count_words)
inner.add_node("summarize", summarize)
inner.set_entry_point("count")
inner.add_edge("count", "summarize")
inner.add_edge("summarize", END)
inner_app = inner.compile()

# Outer graph uses inner graph as a node
class OuterState(TypedDict):
    documents: list
    summaries: Annotated[list, operator.add]

def process_doc(state: OuterState) -> dict:
    doc = state["documents"][0] if state["documents"] else ""
    result = inner_app.invoke({"text": doc, "summary": "", "word_count": 0})
    return {"summaries": [{"text": result["summary"], "words": result["word_count"]}]}

outer = StateGraph(OuterState)
outer.add_node("process", process_doc)
outer.set_entry_point("process")
outer.add_edge("process", END)

app = outer.compile()
result = app.invoke({
    "documents": ["LangGraph is a framework for building stateful multi-actor applications with LLMs."],
    "summaries": []
})
print(result["summaries"])
```

---

## L8 — Human-in-the-Loop

### 16. Interrupt for Human Approval
```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt
import operator

class WorkflowState(TypedDict):
    task: str
    plan: str
    approved: bool
    result: str
    logs: Annotated[list, operator.add]

def create_plan(state: WorkflowState) -> dict:
    plan = f"Plan for '{state['task']}': Step 1 → Step 2 → Step 3"
    return {"plan": plan, "logs": [f"Plan created: {plan}"]}

def human_review(state: WorkflowState) -> dict:
    # interrupt() pauses the graph and waits for human input
    decision = interrupt({
        "message": "Please review this plan",
        "plan": state["plan"],
        "options": ["approve", "reject"]
    })
    approved = decision == "approve"
    return {"approved": approved, "logs": [f"Human decision: {decision}"]}

def execute(state: WorkflowState) -> dict:
    if state["approved"]:
        return {"result": f"Executed: {state['plan']}", "logs": ["Execution successful"]}
    return {"result": "Cancelled by human", "logs": ["Execution cancelled"]}

graph = StateGraph(WorkflowState)
graph.add_node("plan", create_plan)
graph.add_node("review", human_review)
graph.add_node("execute", execute)

graph.set_entry_point("plan")
graph.add_edge("plan", "review")
graph.add_edge("review", "execute")
graph.add_edge("execute", END)

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "workflow-001"}}

# Phase 1: run until interrupt
try:
    result = app.invoke(
        {"task": "Deploy to production", "plan": "", "approved": False, "result": "", "logs": []},
        config=config
    )
except Exception as e:
    print("Paused for human review:", e)

# Phase 2: resume with human decision
result = app.invoke(Command(resume="approve"), config=config)
print("Final result:", result["result"])
print("Logs:", result["logs"])
```

---

## L9 — Multi-Agent Pattern

### 17. Supervisor Pattern
```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage, SystemMessage
import operator
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class MultiAgentState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    task: str
    next_agent: str
    final_answer: str

def supervisor(state: MultiAgentState) -> dict:
    """Decides which agent handles the task."""
    task = state["task"].lower()
    if any(w in task for w in ["code", "program", "function", "debug"]):
        return {"next_agent": "coder"}
    elif any(w in task for w in ["research", "find", "search", "what is"]):
        return {"next_agent": "researcher"}
    else:
        return {"next_agent": "generalist"}

def coder_agent(state: MultiAgentState) -> dict:
    response = llm.invoke([
        SystemMessage(content="You are an expert programmer. Write clean, working code."),
        HumanMessage(content=state["task"])
    ])
    return {
        "messages": [AIMessage(content=f"[CODER]: {response.content}")],
        "final_answer": response.content
    }

def researcher_agent(state: MultiAgentState) -> dict:
    response = llm.invoke([
        SystemMessage(content="You are a research expert. Give thorough, cited answers."),
        HumanMessage(content=state["task"])
    ])
    return {
        "messages": [AIMessage(content=f"[RESEARCHER]: {response.content}")],
        "final_answer": response.content
    }

def generalist_agent(state: MultiAgentState) -> dict:
    response = llm.invoke([
        SystemMessage(content="You are a helpful generalist assistant."),
        HumanMessage(content=state["task"])
    ])
    return {
        "messages": [AIMessage(content=f"[GENERALIST]: {response.content}")],
        "final_answer": response.content
    }

def route_to_agent(state: MultiAgentState) -> str:
    return state["next_agent"]

graph = StateGraph(MultiAgentState)
graph.add_node("supervisor", supervisor)
graph.add_node("coder", coder_agent)
graph.add_node("researcher", researcher_agent)
graph.add_node("generalist", generalist_agent)

graph.set_entry_point("supervisor")
graph.add_conditional_edges(
    "supervisor",
    route_to_agent,
    {"coder": "coder", "researcher": "researcher", "generalist": "generalist"}
)
graph.add_edge("coder", END)
graph.add_edge("researcher", END)
graph.add_edge("generalist", END)

app = graph.compile()

for task in [
    "Write a Python function to reverse a linked list",
    "What is the history of artificial intelligence?",
    "Tell me a joke"
]:
    result = app.invoke({"messages": [], "task": task, "next_agent": "", "final_answer": ""})
    print(f"\nTask: {task}")
    print(f"Answer: {result['final_answer'][:200]}...")
```

---

## L10 — Full Agent from Scratch (Build This from Memory)

### 18. Complete Research Agent
```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage, SystemMessage
from langchain_core.tools import tool
import operator
from dotenv import load_dotenv

load_dotenv()

# ─── Tools ───────────────────────────────────────────────
@tool
def search_web(query: str) -> str:
    """Search the web for information."""
    return f"Web results for '{query}': Found 3 relevant articles about {query}."

@tool
def read_article(url: str) -> str:
    """Read the content of an article by URL."""
    return f"Article at {url}: This article discusses the topic in detail. Key points: ..."

@tool
def save_to_notes(title: str, content: str) -> dict:
    """Save research findings to notes."""
    return {"saved": True, "note_id": abs(hash(title)) % 10000, "title": title}

@tool
def calculator(expression: str) -> str:
    """Evaluate a math expression."""
    try:
        return str(eval(expression))
    except:
        return "Invalid expression"

tools = [search_web, read_article, save_to_notes, calculator]
tool_node = ToolNode(tools)

# ─── LLM ─────────────────────────────────────────────────
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).bind_tools(tools)

SYSTEM_PROMPT = """You are a research agent with access to tools.
When asked to research something:
1. Search the web for information
2. Read relevant articles
3. Save your findings to notes
4. Provide a clear summary

Always think step by step and use your tools efficiently."""

# ─── State ───────────────────────────────────────────────
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    iteration: int

# ─── Nodes ───────────────────────────────────────────────
def agent_node(state: AgentState) -> dict:
    messages = [SystemMessage(content=SYSTEM_PROMPT)] + state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response], "iteration": state["iteration"] + 1}

# ─── Routing ─────────────────────────────────────────────
def should_continue(state: AgentState) -> Literal["tools", "__end__"]:
    last_msg = state["messages"][-1]
    if state["iteration"] > 10:  # safety limit
        return "__end__"
    if hasattr(last_msg, "tool_calls") and last_msg.tool_calls:
        return "tools"
    return "__end__"

# ─── Graph ───────────────────────────────────────────────
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue)
graph.add_edge("tools", "agent")

checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# ─── Run ─────────────────────────────────────────────────
config = {"configurable": {"thread_id": "research-001"}}
result = app.invoke(
    {"messages": [HumanMessage(content="Research the benefits of using LangGraph for AI agents and save your findings.")], "iteration": 0},
    config=config
)
print(result["messages"][-1].content)
```

---

## Bonus Drills — Blank File Challenges

### B1. Build a graph with 3 nodes: `parse` → `validate` → `store`. State has `raw_input`, `parsed`, `valid`, `stored`.

### B2. Build a retry loop graph that calls an LLM, checks if its output contains a number, and retries up to 3 times if not.

### B3. Build a conditional graph that routes based on language detected in input: English → `english_node`, Hindi → `hindi_node`, Other → `translate_node`.

### B4. Build a parallel summarizer: given a list of texts, build a graph that summarizes each one and merges results into a final summary.

### B5. Build a supervisor agent that routes between a `sql_agent` (for database questions) and an `api_agent` (for API questions).

### B6. Add MemorySaver to any previous graph. Run 3 turns of conversation and verify the bot remembers context from turn 1 in turn 3.

### B7. Build a human-in-the-loop workflow: `draft_email` → interrupt for review → `send_email` or `revise_email`.

### B8. Build the full ReAct agent from exercise 11 completely from memory — no peeking. All nodes, edges, routing, tools.

---

## Quick Reference

```python
# State basics
class MyState(TypedDict):
    field: str
    list_field: Annotated[list, operator.add]  # accumulates

# Graph basics
graph = StateGraph(MyState)
graph.add_node("name", function)
graph.set_entry_point("first_node")
graph.add_edge("node_a", "node_b")
graph.add_edge("last_node", END)

# Conditional routing
graph.add_conditional_edges("node", routing_fn, {"option_a": "node_a", "option_b": "node_b"})

# Tools
tool_node = ToolNode(tools)
llm_with_tools = llm.bind_tools(tools)

# Compile
app = graph.compile()                              # no memory
app = graph.compile(checkpointer=MemorySaver())   # with memory

# Invoke
app.invoke(initial_state)
app.invoke(state, config={"configurable": {"thread_id": "abc"}})
```

---

## Progress Tracker

| Level | Topic                                    | Done? | Time Taken |
|-------|------------------------------------------|-------|------------|
| L1    | State, Graph, Nodes, Edges               |  [ ]  |            |
| L2    | Conditional Edges, Loops, Retry          |  [ ]  |            |
| L3    | LLM Nodes, Message History               |  [ ]  |            |
| L4    | Tools, ToolNode, Custom Tools            |  [ ]  |            |
| L5    | ReAct Pattern, Prebuilt Agent            |  [ ]  |            |
| L6    | Memory, Checkpointing, State Inspection  |  [ ]  |            |
| L7    | Subgraphs                                |  [ ]  |            |
| L8    | Human-in-the-Loop, Interrupt             |  [ ]  |            |
| L9    | Multi-Agent, Supervisor Pattern          |  [ ]  |            |
| L10   | Full Agent from Scratch                  |  [ ]  |            |
| Bonus | Blank File Challenges                    |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 18 (Full Research Agent)
> from complete memory. State → Tools → Nodes → Routing → Graph → Compile → Run.
> When it runs and the agent actually uses tools — you're done.
