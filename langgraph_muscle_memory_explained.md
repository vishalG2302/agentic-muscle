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

> **📖 What's happening here?**
>
> This is the "Hello World" of LangGraph. Before writing a single line, understand the **big idea**: LangGraph lets you build programs that look like flowcharts — boxes (nodes) connected by arrows (edges), and a shared notepad (State) that every box can read and write to.
>
> Think of it like an assembly line in a factory:
> - The **State** is the product on the conveyor belt — it carries all the information
> - Each **Node** is a workstation that does something to the product and passes it on
> - **Edges** are the conveyor belt connections between workstations
>
> **Breaking down each piece:**
>
> - `class State(TypedDict)` — defines your "notepad." `TypedDict` is just a Python dictionary with fixed keys and types. Every node receives this and returns a (partial) updated version of it.
> - `def greet(state: State) -> State` — a node is just a regular Python function. It takes the current State, does something, and returns a dict with only the fields it wants to update.
> - `StateGraph(State)` — creates a new graph that uses `State` as its shared memory.
> - `graph.add_node("greet", greet)` — registers the function as a node with a name.
> - `graph.set_entry_point("greet")` — tells the graph which node to start at.
> - `graph.add_edge("greet", "shout")` — draws an arrow: after `greet` finishes, go to `shout`.
> - `graph.add_edge("shout", END)` — `END` is a special constant meaning "stop here."
> - `graph.compile()` — "locks in" the graph and creates a runnable `app`.
> - `app.invoke({"message": "..."})` — starts the graph with an initial State.
>
> **Data flow:**
> ```
> Initial State: {"message": "langgraph is awesome"}
>   → greet node runs → returns {"message": "Hello! You said: langgraph is awesome"}
>   → State updated → {"message": "Hello! You said: langgraph is awesome"}
>   → shout node runs → returns {"message": "HELLO! YOU SAID: LANGGRAPH IS AWESOME"}
>   → State updated → reaches END
>   → app.invoke() returns the final State
> ```

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

> **📖 What's happening here?**
>
> Now the State has **multiple fields** — not just one string, but a counter, a history list, and a boolean flag. This shows how State works as the single source of truth for everything happening in your graph.
>
> **Key concepts:**
>
> - `count: int`, `history: list`, `done: bool` — three different types of data all living in the same State dict. Every node can read any of them.
> - **Nodes only return what they change** — `increment` returns all three fields because it updates all three. But you could return just `{"count": new_count}` if only that changed — the rest stay as-is.
> - `new_history = state["history"] + [new_count]` — this creates a **new list** (doesn't mutate the old one). LangGraph works best when you treat State as immutable — always create new values instead of modifying in-place.
> - `done: new_count >= 5` — you can store computed flags in State to be used for routing decisions later (like in conditional edges).
>
> **Data flow:**
> ```
> Initial: {"count": 0, "history": [], "done": False}
>   → increment: count=1, history=[1], done=False
>   → report: prints "Final count: 1", "History: [1]"
>   → END
> ```
>
> Notice this graph only runs `increment` once. To loop (run it 5 times until `done=True`), you'd need a conditional edge — that's the next drill.

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

> **📖 What's happening here?**
>
> This introduces one of the most important LangGraph patterns: **Annotated state fields that accumulate instead of overwrite.**
>
> **The problem without `Annotated`:**
> If three nodes all return `{"steps": ["something"]}`, the last one wins — earlier values are overwritten. That's bad when you want to collect logs/history from every node.
>
> **The solution — `Annotated[list, operator.add]`:**
> This tells LangGraph: "when a node returns a value for `steps`, don't replace the existing list — **append** (add) to it."
>
> - `operator.add` for lists = list concatenation (like `[1,2] + [3,4]` = `[1,2,3,4]`)
> - Each node returns `{"steps": ["step_one: started"]}` — a list with one item
> - LangGraph automatically merges these using `+` under the hood
>
> **Why this matters:**
> This is how you build message history (drill #7), audit logs, collected results, and anything where multiple nodes contribute to a growing list.
>
> **Data flow:**
> ```
> Initial: {"input": "hello world", "steps": [], "result": ""}
>   → step_one: returns {"steps": ["step_one: started"]}
>     → State.steps = [] + ["step_one: started"] = ["step_one: started"]
>   → step_two: returns {"steps": ["step_two: processed"]}
>     → State.steps = ["step_one: started"] + ["step_two: processed"]
>   → step_three: returns {"steps": ["step_three: done"], "result": "Processed: hello world"}
>     → State.steps = ["step_one: started", "step_two: processed", "step_three: done"]
> ```

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

> **📖 What's happening here?**
>
> Until now, graphs only went in one direction (A → B → C). **Conditional edges** let the graph choose different paths based on what's in the State — like an if/else written as a flowchart.
>
> **Two-part pattern — you always need both:**
>
> 1. **A router node** — a regular node that just passes state through (its only job is to be the "decision point" in the graph)
> 2. **A routing function** — reads State and returns a string label indicating which path to take
>
> **Key concept: `add_conditional_edges`**
> ```python
> graph.add_conditional_edges(
>     "router",          # from this node...
>     classify,          # ...call this function to decide...
>     {                  # ...and use this map to go to the right node
>         "positive": "handle_positive",
>         "negative": "handle_negative",
>         "zero":     "handle_zero",
>     }
> )
> ```
> The routing function `classify` reads `state["number"]` and returns `"positive"`, `"negative"`, or `"zero"`. LangGraph uses that string as the key in the map to find which node to go to next.
>
> **`Literal["positive", "negative", "zero"]`** — a type hint that says "this function returns exactly one of these three strings." Helps with auto-complete and catching typos.
>
> **Data flow:**
> ```
> {"number": 5}  → router (pass-through) → classify returns "positive"
>                                         → handle_positive runs → "5 is positive"
> {"number": -3} → router               → classify returns "negative"
>                                         → handle_negative runs → "-3 is negative"
> {"number": 0}  → router               → classify returns "zero"
>                                         → handle_zero runs → "It's zero!"
> ```

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

> **📖 What's happening here?**
>
> This is the first time the graph loops back on itself — a node can point back to an earlier node. This is what makes LangGraph fundamentally different from LangChain (which only goes forward). With loops, you can build **retry logic, validation loops, multi-step reasoning,** and agents.
>
> **The retry pattern:**
> - Run a task → check if it succeeded → if not, run it again → repeat until success or max retries
>
> **Key concepts:**
>
> - `random.random() > 0.6` — simulates a task that fails 40% of the time. In real life this could be an API call, a code execution, or an LLM response validation.
> - `should_retry` routing function — reads State and returns one of three strings:
>   - `"done"` → task succeeded, go to `success` node
>   - `"give_up"` → too many attempts, go to `fail` node
>   - `"retry"` → failed but still have attempts left → go back to `"attempt"` ← **this creates the loop!**
> - `"retry": "attempt"` in the edges map — pointing back to the same node is how you make a loop in LangGraph
> - `Annotated[list, operator.add]` on `logs` — each attempt appends its log entry without overwriting previous ones
>
> **Data flow (if it fails twice then succeeds):**
> ```
> → attempt (fail, attempts=1) → should_retry → "retry"
>   → attempt (fail, attempts=2) → should_retry → "retry"
>     → attempt (success, attempts=3) → should_retry → "done"
>       → success node → END
> ```

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

> **📖 What's happening here?**
>
> Now we bring an actual LLM into the graph. A node that calls an LLM is just a regular node function — the only difference is it calls `llm.invoke(...)` inside.
>
> **This is the bridge between LangChain and LangGraph:**
> - In LangChain, you call `llm.invoke(...)` directly or inside a chain
> - In LangGraph, you wrap that call inside a node function, and LangGraph controls when and how it runs
>
> **Key concepts:**
>
> - `ChatOpenAI(model="gpt-4o-mini", temperature=0)` — creates the LLM connection (same as in LangChain)
> - `llm.invoke([HumanMessage(content=state["question"])])` — calls the LLM. Note it takes a **list** of messages (even for a single message)
> - `response.content` — extracts the text from the AIMessage object
> - The node returns `{"answer": response.content}` — only updating the `answer` field in State
>
> **Data flow:**
> ```
> Initial State: {"question": "What is LangGraph in one sentence?", "answer": ""}
>   → ask_llm node:
>       reads state["question"]
>       calls LLM with HumanMessage("What is LangGraph...")
>       LLM returns AIMessage("LangGraph is a framework...")
>       returns {"answer": "LangGraph is a framework..."}
>   → State updated: {"question": "...", "answer": "LangGraph is a framework..."}
>   → END
> ```

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

> **📖 What's happening here?**
>
> This drill shows **the standard LangGraph pattern for building chatbots with memory** — storing the entire conversation as a list of message objects in State.
>
> **Why `Annotated[list[BaseMessage], operator.add]`?**
> Because `operator.add` on lists = append. Every time the `chat_node` returns `{"messages": [new_ai_response]}`, LangGraph appends that to the existing list instead of replacing it. This is how the conversation history grows turn by turn.
>
> **`BaseMessage`** — the parent class for all message types (`HumanMessage`, `AIMessage`, `SystemMessage`). Using it as the list type means the list can hold any message type.
>
> **Multi-turn conversation without a checkpointer:**
> Notice how Turn 2 works — you manually build the new state by taking `state["messages"]` (which includes the LLM's reply from Turn 1) and adding a new `HumanMessage`. You're passing the full history every time.
>
> **Data flow:**
> ```
> Turn 1:
>   invoke({"messages": [HumanMessage("My name is Vishal.")]})
>   → chat_node: llm sees [HumanMessage("My name is Vishal.")]
>   → returns {"messages": [AIMessage("Nice to meet you, Vishal!")]}
>   → State.messages = [HumanMessage("..."), AIMessage("...")]  (accumulated!)
>
> Turn 2:
>   state["messages"] now has 2 messages
>   invoke({"messages": state["messages"] + [HumanMessage("What is my name?")]})
>   → chat_node: llm sees all 3 messages → knows the name!
>   → returns {"messages": [AIMessage("Your name is Vishal.")]}
> ```

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

> **📖 What's happening here?**
>
> Three LLM nodes chained together — each one reads from State, calls the LLM, and writes a new field to State. The graph flows linearly: `outline → draft → refine`.
>
> **This is LangGraph's equivalent of LangChain's sequential chain** — but instead of `chain1 | chain2 | chain3`, you define nodes and connect them with edges. The advantage: each step's output is stored in State, so you can inspect intermediate results, add branching, or resume from any point.
>
> **Key concepts:**
>
> - Each node reads from a different State field: `create_outline` reads `topic`, `write_draft` reads `outline`, `refine` reads `draft`
> - Each node writes to a different field: `outline`, `draft`, `refined`
> - The initial State must have **all fields defined**, even the empty ones (`"outline": "", "draft": "", "refined": ""`) — TypedDict requires all keys to exist
>
> **Data flow:**
> ```
> {"topic": "Why AI agents need memory", "outline": "", "draft": "", "refined": ""}
>   → create_outline: reads "topic" → LLM generates → writes "outline"
>   → write_draft: reads "outline" → LLM generates → writes "draft"
>   → refine: reads "draft" → LLM improves it → writes "refined"
>   → END: result has all 4 fields filled in
> ```
>
> You can print `result["outline"]`, `result["draft"]`, and `result["refined"]` to see the transformation at each stage.

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

> **📖 What's happening here?**
>
> This drill introduces the **standard tool-calling loop** in LangGraph — the pattern used in almost every real agent. It combines three things: LangChain's `@tool` decorator, LangGraph's `ToolNode`, and a conditional edge that creates the loop.
>
> **Key concepts:**
>
> - **`@tool` decorator** — turns a regular Python function into a tool the LLM can call. The function's **docstring** is the tool's description — the LLM reads it to decide when to use the tool.
> - **`ToolNode(tools)`** — a pre-built LangGraph node that automatically executes whatever tool the LLM requested. You don't write a loop yourself — ToolNode handles it.
> - **`llm.bind_tools(tools)`** — attaches the tool definitions to the LLM so it knows they exist.
>
> **The agent loop (the key pattern):**
> ```
> [llm node] → if tool_calls exist → [tools node] → back to [llm node]
>            → if no tool_calls → END
> ```
>
> - `call_llm` node: sends messages to LLM. LLM may respond with tool call requests OR a final answer.
> - `should_use_tool`: routing function — checks if the last message has `tool_calls`. If yes → run tools. If no → end.
> - `ToolNode` executes the requested tools and adds `ToolMessage` results to the messages list.
> - The loop continues until LLM gives a final answer with no tool calls.
>
> **Data flow:**
> ```
> "What is 15 + 27? Also what's the weather in Pune?"
>   → call_llm: LLM sees question + knows about add/get_weather tools
>   → LLM returns: tool_calls=[add(15,27), get_weather("Pune")]
>   → should_use_tool: "tools"
>   → ToolNode: executes add(15,27)=42, get_weather("Pune")="28°C, partly cloudy"
>   → adds ToolMessage results to messages
>   → call_llm again: LLM sees results → no more tool calls → "15 + 27 = 42. Weather in Pune: 28°C, partly cloudy."
>   → should_use_tool: END
> ```

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

> **📖 What's happening here?**
>
> This drill shows how to define tools with **explicit, structured input schemas** using Pydantic — useful when your tool takes multiple parameters with types, defaults, and descriptions.
>
> **`@tool` without a schema (simple):**
> ```python
> @tool
> def add(a: int, b: int) -> int:
>     """Add two numbers."""
>     return a + b
> ```
> LangGraph infers the schema from the function signature.
>
> **`@tool(args_schema=...)` with explicit schema (detailed):**
> - You define a `BaseModel` class where each field is a `Field(description="...")` — the descriptions go into the prompt that tells the LLM *how* to use each parameter
> - `Optional[str]` means the field is not required (can be `None`)
> - `default=5` sets a fallback value
> - Pass the schema class to `@tool(args_schema=SearchInput)`
>
> **Why use explicit schemas?**
> For tools with complex inputs where the LLM needs guidance on what each parameter means and when it's optional. The descriptions appear in the tool's prompt sent to the LLM.
>
> **Direct tool testing with `.invoke()`:**
> You can test tools directly without an LLM — just call `tool_fn.invoke({"arg": value})`. Good for debugging your tools in isolation before plugging them into an agent.
>
> **Data flow:**
> ```
> search_knowledge_base.invoke({"query": "AI memory systems", "max_results": 3})
>   → Pydantic validates the input against SearchInput schema
>   → function runs with validated args
>   → returns [{"id": 1, ...}, {"id": 2, ...}, {"id": 3, ...}]
> ```

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

> **📖 What's happening here?**
>
> **ReAct = Reason + Act.** This is the fundamental agent pattern — the LLM alternates between reasoning about what to do and acting by calling tools, until it reaches a final answer.
>
> This drill builds the full ReAct loop **manually**, so you understand every piece before using the prebuilt version in drill #12.
>
> **The loop, step by step:**
> 1. User message enters → `agent` node
> 2. `agent` calls LLM with System + all messages so far
> 3. LLM **reasons**: "I need to calculate (123 * 456) + 789. I'll use the calculator tool."
> 4. LLM **acts**: returns a response with `tool_calls=[calculator("123 * 456")]`
> 5. `route` function sees `tool_calls` → goes to `"tools"` node
> 6. `ToolNode` **executes** `calculator("123 * 456")` → returns `"56088"`
> 7. `ToolMessage("56088")` added to messages
> 8. Back to `agent` node — LLM sees the result and reasons again
> 9. LLM calls `calculator("56088 + 789")` → gets `"56877"`
> 10. LLM sees no more tool calls needed → returns final text answer
> 11. `route` sees no `tool_calls` → returns `"__end__"` → graph stops
>
> **Key difference from drill #9:**
> Here we add a `SystemMessage` with instructions at the start of every LLM call. This is the "System Prompt" that gives the agent its persona and reasoning style.
>
> **`"__end__"` vs `END`:** They're the same thing — `END` is just a constant equal to `"__end__"`. Both work in the routing map.

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

> **📖 What's happening here?**
>
> Now that you understand how the ReAct loop works internally (drill #11), LangGraph gives you a shortcut: **`create_react_agent`** builds the entire loop for you in one line.
>
> **`create_react_agent(llm, tools)`** automatically creates:
> - An `agent` node that calls the LLM with your system prompt
> - A `tools` node (ToolNode) that executes tool calls
> - The conditional routing between them
> - The loop back from tools to agent
>
> **When to use the prebuilt vs building from scratch:**
>
> | Use `create_react_agent` | Build from scratch (drill #11) |
> |---|---|
> | Standard tool-calling agent | Need custom logic in the agent node |
> | Prototyping fast | Need to add extra nodes or custom routing |
> | No special state fields needed | Need to track iterations, intermediate results, etc. |
>
> **Data flow:**
> ```
> "What is the stock price of AAPL and MSFT?"
>   → agent: LLM decides to call get_stock_price("AAPL") and get_stock_price("MSFT")
>   → tools: executes both → "AAPL: $182.50", "MSFT: $378.90"
>   → agent: LLM sees results → "AAPL is $182.50 and MSFT is $378.90."
>   → no more tool calls → END
> ```

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

> **📖 What's happening here?**
>
> In drills #7 and #11, memory was managed manually — you carried the messages list yourself. **`MemorySaver`** automates this: it saves the entire State after each graph run and restores it on the next run with the same `thread_id`.
>
> Think of it like a save file in a video game — when you come back to the same session (same `thread_id`), you resume exactly where you left off.
>
> **Key concepts:**
>
> - **`MemorySaver()`** — an in-memory storage backend. State is saved to Python memory (lost when the program ends). For production, you'd use a database-backed checkpointer.
> - **`graph.compile(checkpointer=checkpointer)`** — attaches the checkpointer to the graph. Now every `invoke` reads and writes state automatically.
> - **`config = {"configurable": {"thread_id": "session-vishal-001"}}`** — the session identifier. Think of it like a conversation ID. Same ID = same memory. Different ID = fresh start.
> - You do **NOT** pass the full message history yourself anymore — the checkpointer loads it from the previous run automatically. You just pass the new message.
>
> **Data flow (with MemorySaver):**
> ```
> Turn 1: invoke({"messages": [HumanMessage("My favorite language is Python.")]}, config)
>   → graph runs → MemorySaver saves state (includes both messages) under thread_id
>
> Turn 2: invoke({"messages": [HumanMessage("What is my favorite language?")]}, config)
>   → MemorySaver loads saved state for this thread_id (has the Python message!)
>   → new message appended (operator.add)
>   → LLM sees full history → answers "Python"
> ```
>
> **Different `thread_id` = completely separate memory** — Alice and Vishal never see each other's conversations.

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

> **📖 What's happening here?**
>
> Once you have a `MemorySaver` checkpointer, you can **inspect the saved state** at any point — see what's currently in the State, what node runs next, and the full history of all previous checkpoints.
>
> This is extremely useful for **debugging**, **auditing**, and **resuming interrupted workflows**.
>
> **Key methods:**
>
> - **`app.get_state(config)`** — returns a `StateSnapshot` with:
>   - `snapshot.values` — the current State dictionary (all fields)
>   - `snapshot.next` — which node(s) would run next (empty tuple if graph is done)
>   - `snapshot.config` — the config used to retrieve it
>
> - **`app.get_state_history(config)`** — returns an iterator of all past `StateSnapshot` objects for this thread, most recent first. Each checkpoint represents a moment in time when LangGraph saved the state.
>
> **Why this matters:**
> - You can see exactly what the agent was thinking at each step
> - You can rewind to an earlier checkpoint and re-run from there (time travel!)
> - In human-in-the-loop workflows, you can inspect the state before approving the next step
>
> **Data flow:**
> ```
> After Turn 1 and Turn 2 from drill #13:
>   get_state(config) → snapshot.values["messages"] has 4 messages (2 Human + 2 AI)
>   get_state_history(config) → list of snapshots, one per graph run
>     history[0] → most recent (after Turn 2)
>     history[1] → after Turn 1
>     history[2] → initial state
> ```

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

> **📖 What's happening here?**
>
> A **subgraph** is a compiled LangGraph graph used as a node inside another (outer) graph. This enables **modularity** — you build small, focused graphs and compose them into larger ones, just like writing functions that call other functions.
>
> **Why use subgraphs?**
> - Break complex graphs into manageable, testable pieces
> - Reuse the same sub-logic in multiple outer graphs
> - Each subgraph can have its own State type and internal routing
>
> **How it works:**
>
> 1. **Inner graph** (`inner`) — has its own `SummaryState` with `text`, `summary`, `word_count`. It does two things: count words and create a summary. It's compiled into `inner_app`.
>
> 2. **Outer graph** (`outer`) — has `OuterState` with `documents` and `summaries`. Its `process_doc` node calls `inner_app.invoke(...)` directly, just like calling any Python function. It passes values from the outer state into the inner state format, runs the inner graph, and extracts the results.
>
> **Key pattern — calling a subgraph inside a node:**
> ```python
> def process_doc(state: OuterState) -> dict:
>     result = inner_app.invoke({...inner state...})  # run the subgraph
>     return {...outer state update using result...}
> ```
>
> **Data flow:**
> ```
> Outer state: {"documents": ["LangGraph is a framework..."], "summaries": []}
>   → process_doc node:
>       calls inner_app.invoke({"text": "LangGraph is a framework...", ...})
>       inner graph runs: count_words → 9 words → summarize → "LangGraph is a framework..."
>       returns {"summaries": [{"text": "LangGraph is a framework...", "words": 9}]}
>   → END
> ```

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

> **📖 What's happening here?**
>
> **Human-in-the-loop** means pausing the graph mid-execution and waiting for a real human to review, approve, or modify something before continuing. This is critical for production AI systems where you can't let an agent take irreversible actions without oversight.
>
> **`interrupt(value)`** — when the graph hits this call, it immediately pauses and sends `value` back to the caller. The graph is frozen in place (thanks to the `MemorySaver` checkpointer). The program doesn't crash — it just stops and waits.
>
> **The two-phase execution pattern:**
>
> **Phase 1 — Run until the interrupt:**
> ```python
> app.invoke(initial_state, config=config)
> # Graph pauses at interrupt() inside human_review node
> # You receive the interrupt value to display to the human
> ```
>
> **Phase 2 — Resume with the human's decision:**
> ```python
> app.invoke(Command(resume="approve"), config=config)
> # Graph picks up from where it paused
> # The interrupt() call returns "approve"
> # Graph continues to execute node
> ```
>
> **Why `MemorySaver` is required:** The checkpointer is what allows the graph to be paused and resumed. It saves the exact state (including which node was interrupted) so the graph knows where to continue from.
>
> **Real-world use cases:**
> - "Approve this email before sending"
> - "Review this database query before executing"
> - "Confirm this financial transaction"
> - "Review the AI's plan before it takes action"

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

> **📖 What's happening here?**
>
> The **Supervisor Pattern** is a multi-agent architecture where one "manager" agent (the supervisor) reads the task and routes it to the most appropriate specialist agent. Think of it like a team lead who reads incoming requests and assigns them to the right team member.
>
> **Why multi-agent?**
> No single LLM prompt does everything well. A coder agent has a different system prompt than a researcher agent. Specialization produces better results.
>
> **How the supervisor works here:**
> - **Simple keyword-based routing** (no LLM call needed) — checks if the task contains words like "code", "function" → coder; "research", "what is" → researcher; anything else → generalist
> - In production, you'd use an LLM to classify the task for more nuanced routing
>
> **Key concepts:**
>
> - `next_agent: str` in State — the supervisor writes the chosen agent name here. The routing function `route_to_agent` reads this and uses it as the edge direction.
> - Each specialist agent has its own `SystemMessage` that shapes its behavior and tone
> - All agents write to `final_answer` — the output field read at the end
>
> **Data flow:**
> ```
> task: "Write a Python function to reverse a linked list"
>   → supervisor: contains "function" → sets next_agent="coder"
>   → route_to_agent: reads "coder" → goes to coder node
>   → coder_agent: LLM with "You are an expert programmer" system prompt → writes code
>   → END
>
> task: "What is the history of AI?"
>   → supervisor: contains "what is" → sets next_agent="researcher"
>   → researcher_agent: LLM with "You are a research expert" prompt → gives detailed answer
> ```

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

> **📖 What's happening here?**
>
> This is the **capstone exercise** — it combines every concept into a complete, production-style research agent with tools, memory, iteration tracking, and a safety limit. Let's walk through every section:
>
> **🔧 Tools (4 total):**
> - `search_web` — simulates a web search (in production: use a real search API)
> - `read_article` — simulates reading a URL's content
> - `save_to_notes` — saves findings (in production: write to a database)
> - `calculator` — evaluates math expressions
>
> **🤖 LLM setup:**
> - `llm.bind_tools(tools)` — LLM knows about all 4 tools
> - `SYSTEM_PROMPT` — gives the agent a multi-step research workflow to follow
>
> **📊 State — two fields:**
> - `messages: Annotated[list[BaseMessage], operator.add]` — the full conversation + tool results (accumulated)
> - `iteration: int` — counts how many times the agent node has run (for the safety limit)
>
> **🔄 The agent loop:**
> - `agent_node`: prepends `SystemMessage` + calls LLM + increments iteration counter
> - `should_continue`: the exit conditions — stop if iteration > 10 (safety) OR no tool calls (done reasoning)
> - `ToolNode` executes whatever tools the LLM requested
> - Loop: `agent → tools → agent → tools → ... → agent (no tool calls) → END`
>
> **💾 MemorySaver + `thread_id`:**
> - State is checkpointed after every node
> - You can continue the conversation in a new `invoke` call with the same `thread_id`
>
> **Full data flow for "Research LangGraph benefits...":**
> ```
> Iteration 1: agent → tool_calls=[search_web("LangGraph benefits")]
> → tools: executes search → ToolMessage("Found 3 articles...")
> Iteration 2: agent → tool_calls=[read_article(url), save_to_notes("LangGraph", "...")]
> → tools: executes both → ToolMessages added
> Iteration 3: agent → no tool_calls → "LangGraph provides stateful workflows..."
> → should_continue: no tool_calls → "__end__"
> → Final answer printed
> ```

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
