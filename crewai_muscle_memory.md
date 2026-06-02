# CrewAI Muscle Memory — 40+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `python filename.py`
> - Install: `pip install crewai crewai-tools langchain-openai python-dotenv`
> - Set `OPENAI_API_KEY` in `.env`
> - Mental model: CrewAI = Agents (who) + Tasks (what) + Crew (orchestrator)
> - L1–L2 → agents and tasks, first crew (20–30 min)
> - L3–L4 → tools, custom tools (30–40 min)
> - L5–L6 → process types, task dependencies (30–40 min)
> - L7–L8 → memory, output models (30–40 min)
> - L9–L10 → full multi-agent pipelines (50–60 min)

---

## The Core Mental Model — Before You Write a Line

```
Agent  = a specialist with a role, goal, and backstory
Task   = a job given to an agent with a description and expected output
Crew   = the team that runs tasks in a process (sequential or hierarchical)

Sequential  → Task 1 → Task 2 → Task 3 (output of one feeds the next)
Hierarchical → a manager agent delegates tasks to worker agents
```

Burn this in before touching code.

---

## L1 — First Agent, First Task, First Crew

### 1. Simplest Possible Crew
```python
# 01_hello_crew.py
from crewai import Agent, Task, Crew
from dotenv import load_dotenv

load_dotenv()

# Step 1: Define an agent — who does the work
writer = Agent(
    role="Content Writer",
    goal="Write clear, concise explanations about technology topics",
    backstory="""You are an experienced technical writer who specialises
    in making complex topics easy to understand for beginners.""",
    verbose=True
)

# Step 2: Define a task — what needs to be done
write_task = Task(
    description="Write a 3-sentence explanation of what an API is.",
    expected_output="A clear 3-sentence explanation suitable for a beginner.",
    agent=writer
)

# Step 3: Assemble the crew — who runs what
crew = Crew(
    agents=[writer],
    tasks=[write_task],
    verbose=True
)

# Step 4: Kick it off
result = crew.kickoff()
print("\n=== RESULT ===")
print(result)
```

---

### 2. Two Agents, Two Tasks — Sequential
```python
# 02_two_agents.py
from crewai import Agent, Task, Crew, Process
from dotenv import load_dotenv

load_dotenv()

researcher = Agent(
    role="Research Analyst",
    goal="Find key facts and data about a given topic",
    backstory="""You are a meticulous researcher who finds accurate,
    relevant information from your knowledge base.""",
    verbose=True
)

writer = Agent(
    role="Content Writer",
    goal="Transform research findings into readable content",
    backstory="""You are a skilled writer who turns raw research
    into engaging, well-structured articles.""",
    verbose=True
)

research_task = Task(
    description="Research the top 3 benefits of using FastAPI over Flask for building APIs.",
    expected_output="A bullet list of 3 key benefits with one sentence explanation each.",
    agent=researcher
)

write_task = Task(
    description="""Using the research provided, write a short 2-paragraph blog intro
    about why developers should use FastAPI.""",
    expected_output="Two well-written paragraphs suitable for a developer blog.",
    agent=writer
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,   # default — tasks run in order
    verbose=True
)

result = crew.kickoff()
print("\n=== FINAL OUTPUT ===")
print(result)
```

---

### 3. Agent Configuration Options
```python
# 03_agent_config.py
from crewai import Agent
from dotenv import load_dotenv

load_dotenv()

# All the knobs you can turn on an agent
analyst = Agent(
    role="Data Analyst",
    goal="Analyse data and provide actionable insights",
    backstory="""You are a senior data analyst with 10 years of experience
    in Python, SQL, and data visualisation. You are detail-oriented
    and always back your findings with numbers.""",

    verbose=True,           # print agent thoughts
    allow_delegation=False, # cannot pass tasks to other agents
    max_iter=5,             # max reasoning iterations before stopping
    memory=True,            # agent remembers previous interactions

    # You can also set a specific LLM
    # llm="gpt-4o-mini"     # override default LLM for this agent
)

print("Agent created:")
print(f"  Role: {analyst.role}")
print(f"  Goal: {analyst.goal}")
```

---

## L2 — Tasks in Depth

### 4. Task Configuration Options
```python
# 04_task_config.py
from crewai import Agent, Task, Crew
from pydantic import BaseModel
from typing import List
from dotenv import load_dotenv

load_dotenv()

class ResearchOutput(BaseModel):
    topic: str
    key_points: List[str]
    summary: str

agent = Agent(
    role="Research Specialist",
    goal="Produce structured research output",
    backstory="You are a thorough researcher who always structures findings clearly."
)

task = Task(
    description="""Research Python decorators. Cover:
    1. What they are
    2. How they work
    3. Two real-world use cases""",

    expected_output="""A structured research output with:
    - topic name
    - 3-5 key points as bullet list
    - one paragraph summary""",

    agent=agent,
    output_pydantic=ResearchOutput,  # structured output
    # output_file="research.md",     # save to file
    # context=[other_task],          # use output of another task as context
)

crew = Crew(agents=[agent], tasks=[task], verbose=True)
result = crew.kickoff()

# Access structured output
if hasattr(result, 'pydantic'):
    data = result.pydantic
    print(f"\nTopic: {data.topic}")
    print(f"Key Points: {data.key_points}")
    print(f"Summary: {data.summary}")
```

---

### 5. Task Context — Passing Output Between Tasks
```python
# 05_task_context.py
from crewai import Agent, Task, Crew, Process
from dotenv import load_dotenv

load_dotenv()

researcher = Agent(
    role="Researcher",
    goal="Research topics thoroughly",
    backstory="Expert at finding and summarising information."
)

critic = Agent(
    role="Critic",
    goal="Review and improve content quality",
    backstory="You have a sharp eye for clarity, accuracy, and completeness."
)

writer = Agent(
    role="Writer",
    goal="Write polished final content",
    backstory="You write clean, engaging technical content."
)

research_task = Task(
    description="Research the concept of 'agent memory' in AI systems. List key types and how they work.",
    expected_output="A structured list of agent memory types with explanations.",
    agent=researcher
)

review_task = Task(
    description="""Review the research provided and identify:
    1. Any missing information
    2. Any unclear explanations
    3. Suggestions for improvement""",
    expected_output="A review with specific improvement suggestions.",
    agent=critic,
    context=[research_task]   # this task sees research_task's output
)

write_task = Task(
    description="""Using the research and the review feedback, write a clean
    400-word explanation of agent memory for a technical audience.""",
    expected_output="A polished 400-word technical explanation.",
    agent=writer,
    context=[research_task, review_task]  # sees both previous outputs
)

crew = Crew(
    agents=[researcher, critic, writer],
    tasks=[research_task, review_task, write_task],
    process=Process.sequential,
    verbose=True
)

result = crew.kickoff()
print("\n=== FINAL ARTICLE ===")
print(result)
```

---

## L3 — Tools

### 6. Built-in Tools
```python
# 06_builtin_tools.py
from crewai import Agent, Task, Crew
from crewai_tools import (
    SerperDevTool,       # web search (needs SERPER_API_KEY)
    FileReadTool,        # read local files
    DirectoryReadTool,   # read directory contents
    # WebsiteSearchTool, # RAG over a website
)
from dotenv import load_dotenv
import os

load_dotenv()

# File tool — no API key needed
file_tool = FileReadTool()

agent = Agent(
    role="File Analyst",
    goal="Read and summarise file contents",
    backstory="You are skilled at reading files and extracting key information.",
    tools=[file_tool],
    verbose=True
)

# First create a test file
with open("sample.txt", "w") as f:
    f.write("""LangGraph is a library for building stateful agents.
    It uses nodes and edges to define agent workflows.
    Memory can be persisted using checkpointers.
    It supports human-in-the-loop patterns.""")

task = Task(
    description="Read the file 'sample.txt' and write a one-paragraph summary.",
    expected_output="A single paragraph summarising the file content.",
    agent=agent
)

crew = Crew(agents=[agent], tasks=[task], verbose=True)
result = crew.kickoff()
print("\n=== SUMMARY ===")
print(result)
```

---

### 7. Custom Tool — Function Style
```python
# 07_custom_tool_function.py
from crewai import Agent, Task, Crew
from crewai.tools import tool
from dotenv import load_dotenv
import json

load_dotenv()

# Define custom tools using the @tool decorator
@tool("Word Counter")
def word_counter(text: str) -> int:
    """Count the number of words in a given text."""
    return len(text.split())

@tool("JSON Formatter")
def json_formatter(data: str) -> str:
    """Format a JSON string to be pretty-printed."""
    try:
        parsed = json.loads(data)
        return json.dumps(parsed, indent=2)
    except json.JSONDecodeError:
        return "Invalid JSON"

@tool("Keyword Extractor")
def keyword_extractor(text: str) -> str:
    """Extract key words from text (words longer than 5 characters, unique)."""
    words = text.lower().split()
    keywords = list(set(w.strip(".,!?") for w in words if len(w) > 5))
    return ", ".join(sorted(keywords[:10]))

agent = Agent(
    role="Text Analyst",
    goal="Analyse text using available tools and provide insights",
    backstory="You are a text processing specialist who uses tools to extract insights.",
    tools=[word_counter, keyword_extractor],
    verbose=True
)

task = Task(
    description="""Analyse this text:
    'CrewAI is a framework for orchestrating role-playing autonomous AI agents.
    It enables agents to collaborate on complex tasks using tools and memory.'

    Use tools to: count words, extract keywords, then write a brief analysis.""",
    expected_output="Word count, keywords, and a 2-sentence analysis.",
    agent=agent
)

crew = Crew(agents=[agent], tasks=[task], verbose=True)
result = crew.kickoff()
print("\n=== ANALYSIS ===")
print(result)
```

---

### 8. Custom Tool — Class Style (Full Control)
```python
# 08_custom_tool_class.py
from crewai import Agent, Task, Crew
from crewai.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Type
from dotenv import load_dotenv
import time

load_dotenv()

# Class-style tool — use when you need input validation or state
class DatabaseInput(BaseModel):
    query: str = Field(..., description="The query to run against the fake database")

class FakeDatabaseTool(BaseTool):
    name: str = "Fake Database"
    description: str = "Query a fake in-memory database of users and products."

    # Define input schema
    args_schema: Type[BaseModel] = DatabaseInput

    # This is where the tool logic lives
    def _run(self, query: str) -> str:
        db = {
            "users": [
                {"id": 1, "name": "Vishal", "role": "admin"},
                {"id": 2, "name": "Alice", "role": "viewer"},
            ],
            "products": [
                {"id": 1, "name": "Laptop", "price": 999},
                {"id": 2, "name": "Phone", "price": 499},
            ]
        }
        query_lower = query.lower()
        if "user" in query_lower:
            return str(db["users"])
        elif "product" in query_lower:
            return str(db["products"])
        return "No matching data found."


class TimestampTool(BaseTool):
    name: str = "Timestamp Tool"
    description: str = "Returns the current timestamp."

    def _run(self, *args, **kwargs) -> str:
        return time.strftime("%Y-%m-%d %H:%M:%S")


db_tool = FakeDatabaseTool()
ts_tool = TimestampTool()

agent = Agent(
    role="Database Reporter",
    goal="Query database and generate timestamped reports",
    backstory="You are a database analyst who creates clear data reports.",
    tools=[db_tool, ts_tool],
    verbose=True
)

task = Task(
    description="Get all users from the database and create a timestamped report listing them.",
    expected_output="A timestamped report listing all users with their roles.",
    agent=agent
)

crew = Crew(agents=[agent], tasks=[task], verbose=True)
result = crew.kickoff()
print("\n=== REPORT ===")
print(result)
```

---

## L4 — Multiple Tools Per Agent

### 9. Agent with a Toolbelt
```python
# 09_toolbelt.py
from crewai import Agent, Task, Crew
from crewai.tools import tool
from dotenv import load_dotenv
import math

load_dotenv()

@tool("Calculator")
def calculator(expression: str) -> str:
    """Evaluate a safe mathematical expression. E.g. '2 + 2', '10 * 5.5'"""
    try:
        allowed = set("0123456789+-*/.() ")
        if all(c in allowed for c in expression):
            return str(round(eval(expression), 4))
        return "Invalid expression"
    except Exception as e:
        return f"Error: {e}"

@tool("Unit Converter")
def unit_converter(value: float, from_unit: str, to_unit: str) -> str:
    """Convert between common units. Supports: km/miles, kg/lbs, celsius/fahrenheit"""
    conversions = {
        ("km", "miles"): lambda v: v * 0.621371,
        ("miles", "km"): lambda v: v * 1.60934,
        ("kg", "lbs"): lambda v: v * 2.20462,
        ("lbs", "kg"): lambda v: v * 0.453592,
        ("celsius", "fahrenheit"): lambda v: v * 9/5 + 32,
        ("fahrenheit", "celsius"): lambda v: (v - 32) * 5/9,
    }
    key = (from_unit.lower(), to_unit.lower())
    if key in conversions:
        result = conversions[key](value)
        return f"{value} {from_unit} = {round(result, 4)} {to_unit}"
    return f"Conversion from {from_unit} to {to_unit} not supported"

@tool("Statistics Calculator")
def stats_calculator(numbers: str) -> str:
    """Calculate statistics for a comma-separated list of numbers."""
    try:
        nums = [float(n.strip()) for n in numbers.split(",")]
        mean = sum(nums) / len(nums)
        variance = sum((x - mean)**2 for x in nums) / len(nums)
        return (f"Count: {len(nums)}, Sum: {sum(nums):.2f}, "
                f"Mean: {mean:.2f}, Min: {min(nums)}, Max: {max(nums)}, "
                f"Std Dev: {math.sqrt(variance):.2f}")
    except Exception as e:
        return f"Error: {e}"

scientist = Agent(
    role="Data Scientist",
    goal="Perform calculations and analysis using available tools",
    backstory="You are a quantitative analyst who uses tools to solve numerical problems.",
    tools=[calculator, unit_converter, stats_calculator],
    verbose=True
)

task = Task(
    description="""Solve these problems:
    1. Calculate (144 * 3.14159) / 2
    2. Convert 75 kg to lbs
    3. Find statistics for: 12, 45, 67, 23, 89, 34, 56
    Write a summary of all results.""",
    expected_output="Results for all 3 calculations with a brief summary.",
    agent=scientist
)

crew = Crew(agents=[scientist], tasks=[task], verbose=True)
result = crew.kickoff()
print("\n=== RESULTS ===")
print(result)
```

---

## L5 — Process Types

### 10. Sequential Process (Default)
```python
# 10_sequential.py
from crewai import Agent, Task, Crew, Process
from dotenv import load_dotenv

load_dotenv()

# Think of sequential like an assembly line
# Each agent passes their output as context to the next

planner = Agent(
    role="Project Planner",
    goal="Create structured project plans",
    backstory="You break complex projects into clear, actionable steps."
)

developer = Agent(
    role="Senior Developer",
    goal="Write clean, well-structured code solutions",
    backstory="You write Python code that is readable, efficient, and well-commented."
)

reviewer = Agent(
    role="Code Reviewer",
    goal="Review code for quality, bugs, and improvements",
    backstory="You are a meticulous reviewer who catches issues and suggests improvements."
)

plan_task = Task(
    description="Create a step-by-step plan for building a simple REST API with FastAPI that has CRUD for a 'Task' resource.",
    expected_output="A numbered list of implementation steps.",
    agent=planner
)

code_task = Task(
    description="Based on the plan, write the Python/FastAPI code for the Task CRUD API.",
    expected_output="Clean, runnable FastAPI code with all CRUD endpoints.",
    agent=developer
)

review_task = Task(
    description="Review the code written. List bugs, improvements, and a rating out of 10.",
    expected_output="A code review with issues, suggestions, and a rating.",
    agent=reviewer
)

crew = Crew(
    agents=[planner, developer, reviewer],
    tasks=[plan_task, code_task, review_task],
    process=Process.sequential,
    verbose=True
)

result = crew.kickoff()
print("\n=== CODE REVIEW ===")
print(result)
```

---

### 11. Hierarchical Process — Manager Delegates
```python
# 11_hierarchical.py
from crewai import Agent, Task, Crew, Process
from dotenv import load_dotenv

load_dotenv()

# In hierarchical process, a manager LLM decides which agent does what

researcher = Agent(
    role="Researcher",
    goal="Find accurate information on any topic",
    backstory="You are a thorough researcher with broad knowledge.",
    allow_delegation=False
)

writer = Agent(
    role="Writer",
    goal="Write clear and engaging content",
    backstory="You write professional content for technical audiences.",
    allow_delegation=False
)

editor = Agent(
    role="Editor",
    goal="Polish and improve content quality",
    backstory="You refine content for clarity, grammar, and flow.",
    allow_delegation=False
)

# In hierarchical, you don't assign agents to tasks — the manager decides
task = Task(
    description="""Produce a polished 3-paragraph article about
    'Why every Python developer should learn FastAPI in 2025'.
    It should be researched, well-written, and edited.""",
    expected_output="A polished 3-paragraph article, ready to publish."
)

crew = Crew(
    agents=[researcher, writer, editor],
    tasks=[task],
    process=Process.hierarchical,
    manager_llm="gpt-4o-mini",   # the manager that delegates
    verbose=True
)

result = crew.kickoff()
print("\n=== ARTICLE ===")
print(result)
```

---

## L6 — Structured Outputs

### 12. Pydantic Output Models
```python
# 12_pydantic_output.py
from crewai import Agent, Task, Crew
from pydantic import BaseModel, Field
from typing import List, Optional
from dotenv import load_dotenv

load_dotenv()

# Define what you want the output to look like
class TechAnalysis(BaseModel):
    technology: str = Field(description="Name of the technology")
    category: str = Field(description="Category: framework, library, tool, language")
    pros: List[str] = Field(description="List of advantages")
    cons: List[str] = Field(description="List of disadvantages")
    best_for: str = Field(description="Best use case in one sentence")
    difficulty: str = Field(description="Learning difficulty: beginner, intermediate, advanced")
    rating: float = Field(description="Overall rating from 1.0 to 10.0")

analyst = Agent(
    role="Technology Analyst",
    goal="Analyse technologies and provide structured evaluations",
    backstory="You evaluate developer tools with expertise and structure your findings clearly."
)

task = Task(
    description="Analyse FastAPI as a technology for building backend APIs.",
    expected_output="A structured technology analysis with pros, cons, rating, and use cases.",
    agent=analyst,
    output_pydantic=TechAnalysis
)

crew = Crew(agents=[analyst], tasks=[task], verbose=True)
result = crew.kickoff()

# Access structured fields
if hasattr(result, 'pydantic') and result.pydantic:
    data = result.pydantic
    print(f"\n{'='*40}")
    print(f"Technology : {data.technology}")
    print(f"Category   : {data.category}")
    print(f"Difficulty : {data.difficulty}")
    print(f"Rating     : {data.rating}/10")
    print(f"Best For   : {data.best_for}")
    print(f"\nPros:")
    for p in data.pros:
        print(f"  ✓ {p}")
    print(f"\nCons:")
    for c in data.cons:
        print(f"  ✗ {c}")
```

---

### 13. JSON Output and File Output
```python
# 13_json_file_output.py
from crewai import Agent, Task, Crew
from pydantic import BaseModel
from typing import List
from dotenv import load_dotenv

load_dotenv()

class ComparisonResult(BaseModel):
    winner: str
    summary: str
    scores: dict

agent = Agent(
    role="Tech Comparator",
    goal="Compare technologies objectively and provide structured results",
    backstory="You are an expert at analysing and comparing developer tools."
)

task = Task(
    description="Compare Flask vs FastAPI for building REST APIs. Evaluate speed, ease of use, ecosystem, and documentation.",
    expected_output="A structured comparison with a winner and scores for each category.",
    agent=agent,
    output_json=ComparisonResult,
    output_file="comparison_result.json"   # also saves to file
)

crew = Crew(agents=[agent], tasks=[task], verbose=True)
result = crew.kickoff()
print("\n=== JSON OUTPUT ===")
print(result.json_dict if hasattr(result, 'json_dict') else result)
```

---

## L7 — Memory

### 14. Enable Agent Memory
```python
# 14_memory.py
from crewai import Agent, Task, Crew
from dotenv import load_dotenv

load_dotenv()

# memory=True gives the agent short-term memory within a session
agent = Agent(
    role="Personal Assistant",
    goal="Remember context and provide personalised help",
    backstory="You are a helpful assistant that remembers what you've been told.",
    memory=True,     # agent remembers across tasks in this crew run
    verbose=True
)

task1 = Task(
    description="My name is Vishal and I am building an AI image memory system using Neo4j and LangGraph.",
    expected_output="Acknowledge you've noted Vishal's project details.",
    agent=agent
)

task2 = Task(
    description="What project am I building and what technologies am I using?",
    expected_output="A summary of the user's project based on what was told earlier.",
    agent=agent
)

task3 = Task(
    description="Suggest two next steps I should take for my project.",
    expected_output="Two concrete, personalised next steps based on the project context.",
    agent=agent
)

crew = Crew(
    agents=[agent],
    tasks=[task1, task2, task3],
    memory=True,        # crew-level memory
    verbose=True
)

result = crew.kickoff()
print("\n=== SUGGESTIONS ===")
print(result)
```

---

## L8 — Kickoff with Inputs

### 15. Dynamic Inputs with kickoff()
```python
# 15_dynamic_inputs.py
from crewai import Agent, Task, Crew
from dotenv import load_dotenv

load_dotenv()

researcher = Agent(
    role="Research Analyst",
    goal="Research topics and provide clear summaries",
    backstory="You find accurate, concise information on any topic."
)

writer = Agent(
    role="Content Writer",
    goal="Write engaging content from research",
    backstory="You write clear technical content for developer audiences."
)

# Use {placeholders} in descriptions — filled at kickoff time
research_task = Task(
    description="Research {topic} and find its top {num_points} use cases in {industry}.",
    expected_output="A list of {num_points} use cases with brief explanations.",
    agent=researcher
)

write_task = Task(
    description="Write a {format} about the research findings on {topic}.",
    expected_output="A polished {format} ready to share with a technical audience.",
    agent=writer
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    verbose=True
)

# Pass inputs at runtime — they fill the placeholders
result = crew.kickoff(inputs={
    "topic": "vector databases",
    "num_points": "3",
    "industry": "AI applications",
    "format": "short blog post"
})

print("\n=== OUTPUT ===")
print(result)
```

---

### 16. kickoff_for_each — Run Crew for Multiple Inputs
```python
# 16_kickoff_for_each.py
from crewai import Agent, Task, Crew
from dotenv import load_dotenv

load_dotenv()

agent = Agent(
    role="Tech Explainer",
    goal="Explain technical concepts clearly in one paragraph",
    backstory="You make complex technical topics accessible to beginners."
)

task = Task(
    description="Explain {concept} in simple terms in exactly 3 sentences.",
    expected_output="A 3-sentence beginner-friendly explanation of {concept}.",
    agent=agent
)

crew = Crew(agents=[agent], tasks=[task], verbose=False)

# Run the same crew for multiple inputs
topics = [
    {"concept": "vector embeddings"},
    {"concept": "graph databases"},
    {"concept": "async programming"},
]

results = crew.kickoff_for_each(inputs=topics)

for topic, result in zip(topics, results):
    print(f"\n📚 {topic['concept'].upper()}")
    print(result)
```

---

## L9 — Full Pipelines

### 17. Content Production Pipeline
```python
# 17_content_pipeline.py
from crewai import Agent, Task, Crew, Process
from crewai.tools import tool
from pydantic import BaseModel
from typing import List
from dotenv import load_dotenv

load_dotenv()

# ── Tools ───────────────────────────────────────────────
@tool("Word Counter")
def word_counter(text: str) -> int:
    """Count words in text."""
    return len(text.split())

@tool("Readability Checker")
def readability_checker(text: str) -> str:
    """Check text readability: avg sentence length and long word ratio."""
    sentences = [s.strip() for s in text.split(".") if s.strip()]
    words = text.split()
    avg_len = len(words) / max(len(sentences), 1)
    long_words = sum(1 for w in words if len(w) > 8)
    ratio = long_words / max(len(words), 1)
    score = "Easy" if avg_len < 15 and ratio < 0.2 else "Medium" if avg_len < 25 else "Hard"
    return f"Avg sentence length: {avg_len:.1f} words | Long word ratio: {ratio:.2f} | Score: {score}"

# ── Agents ──────────────────────────────────────────────
researcher = Agent(
    role="Senior Researcher",
    goal="Research topics deeply and find key insights",
    backstory="You find accurate, relevant information and organise it logically.",
    allow_delegation=False
)

writer = Agent(
    role="Technical Writer",
    goal="Write clear, engaging technical content",
    backstory="You specialise in making technical topics accessible to developers.",
    tools=[word_counter],
    allow_delegation=False
)

editor = Agent(
    role="Senior Editor",
    goal="Polish content to professional publication standards",
    backstory="You have 15 years editing technical content. You improve clarity, fix grammar, and ensure flow.",
    tools=[readability_checker, word_counter],
    allow_delegation=False
)

# ── Tasks ───────────────────────────────────────────────
research_task = Task(
    description="""Research 'agent memory systems in AI' covering:
    1. Types of memory (short-term, long-term, episodic, semantic)
    2. How they are implemented technically
    3. Key frameworks that use them (LangGraph, CrewAI, mem0)""",
    expected_output="Structured research notes with headings and bullet points.",
    agent=researcher
)

write_task = Task(
    description="""Using the research, write a 400-500 word technical article
    about agent memory systems. Include a brief intro, body covering the 3 research areas,
    and a conclusion. Use the word counter tool to verify length.""",
    expected_output="A 400-500 word technical article with proper structure.",
    agent=writer,
    context=[research_task]
)

edit_task = Task(
    description="""Edit the article for:
    1. Clarity and flow
    2. Grammar and punctuation
    3. Use readability checker and ensure score is Easy or Medium
    4. Verify word count is between 400-500
    Return the final polished article.""",
    expected_output="A polished, publication-ready article with readability score noted.",
    agent=editor,
    context=[write_task]
)

# ── Crew ────────────────────────────────────────────────
crew = Crew(
    agents=[researcher, writer, editor],
    tasks=[research_task, write_task, edit_task],
    process=Process.sequential,
    memory=True,
    verbose=True
)

result = crew.kickoff()
print("\n" + "="*60)
print("FINAL PUBLISHED ARTICLE")
print("="*60)
print(result)
```

---

## L10 — Full Multi-Agent System (Build from Memory)

### 18. AI Startup Analyser Crew
```python
# 18_startup_analyser.py
from crewai import Agent, Task, Crew, Process
from crewai.tools import tool
from pydantic import BaseModel, Field
from typing import List
from dotenv import load_dotenv

load_dotenv()

# ── Tools ───────────────────────────────────────────────
@tool("Market Size Estimator")
def market_size(industry: str) -> str:
    """Estimate market size for a given industry."""
    sizes = {
        "ai": "$500B by 2027, growing at 38% CAGR",
        "saas": "$700B by 2028, growing at 18% CAGR",
        "fintech": "$320B by 2026, growing at 23% CAGR",
        "healthtech": "$280B by 2026, growing at 15% CAGR",
    }
    for key, val in sizes.items():
        if key in industry.lower():
            return val
    return "Market data not available. Estimated $50B+ for niche markets."

@tool("Competitor Finder")
def find_competitors(domain: str) -> str:
    """Find known competitors in a domain."""
    comps = {
        "agent memory": "mem0, Zep, LangMem, Graphiti",
        "ai coding": "GitHub Copilot, Cursor, Codeium, Tabnine",
        "ai search": "Perplexity, You.com, Kagi, Exa",
        "rag": "LlamaIndex, LangChain, Haystack, Weaviate",
    }
    for key, val in comps.items():
        if key in domain.lower():
            return val
    return "No direct competitor data found. Likely a blue ocean or niche."

@tool("Risk Scorer")
def score_risk(factors: str) -> str:
    """Score startup risk based on described factors."""
    high_risk = ["no revenue", "unproven", "crowded market", "no moat"]
    low_risk = ["revenue", "patent", "unique", "enterprise", "contract"]
    score = 5
    factors_lower = factors.lower()
    for f in high_risk:
        if f in factors_lower:
            score += 1
    for f in low_risk:
        if f in factors_lower:
            score -= 1
    score = max(1, min(10, score))
    label = "High" if score >= 7 else "Medium" if score >= 4 else "Low"
    return f"Risk Score: {score}/10 ({label} Risk)"

# ── Output Model ────────────────────────────────────────
class StartupReport(BaseModel):
    startup_name: str
    industry: str
    market_size: str
    competitors: List[str]
    strengths: List[str]
    weaknesses: List[str]
    risk_assessment: str
    investment_verdict: str
    overall_score: float = Field(ge=1.0, le=10.0)

# ── Agents ──────────────────────────────────────────────
market_analyst = Agent(
    role="Market Research Analyst",
    goal="Analyse market size, trends, and competitive landscape",
    backstory="You are a senior analyst at a top VC firm specialising in tech markets.",
    tools=[market_size, find_competitors],
    allow_delegation=False
)

due_diligence = Agent(
    role="Due Diligence Specialist",
    goal="Assess startup strengths, weaknesses, and risks",
    backstory="You have reviewed 500+ startups and have a sharp eye for red flags and diamonds.",
    tools=[score_risk],
    allow_delegation=False
)

investment_advisor = Agent(
    role="Investment Advisor",
    goal="Make clear, justified investment recommendations",
    backstory="You synthesise research into decisive investment verdicts backed by data.",
    allow_delegation=False
)

# ── Tasks ───────────────────────────────────────────────
market_task = Task(
    description="""Analyse the market for this startup:
    Name: {startup_name}
    Description: {description}

    Use tools to find market size and competitors.""",
    expected_output="Market size data and competitor list with brief analysis.",
    agent=market_analyst
)

dd_task = Task(
    description="""Perform due diligence on {startup_name}:
    Description: {description}

    Identify top 3 strengths and top 3 weaknesses.
    Use the risk scorer tool with identified factors.""",
    expected_output="Strengths, weaknesses, and risk score with explanation.",
    agent=due_diligence,
    context=[market_task]
)

verdict_task = Task(
    description="""Based on all research, give a final investment verdict for {startup_name}.
    Provide an overall score out of 10 and a clear recommendation.""",
    expected_output="Investment verdict with score and 2-sentence justification.",
    agent=investment_advisor,
    context=[market_task, dd_task],
    output_pydantic=StartupReport
)

# ── Crew ────────────────────────────────────────────────
crew = Crew(
    agents=[market_analyst, due_diligence, investment_advisor],
    tasks=[market_task, dd_task, verdict_task],
    process=Process.sequential,
    verbose=True
)

# ── Run ─────────────────────────────────────────────────
result = crew.kickoff(inputs={
    "startup_name": "MemoryLayer AI",
    "description": "A startup building persistent memory and context systems for AI agents using graph databases and vector embeddings."
})

print("\n" + "="*60)
print("INVESTMENT REPORT")
print("="*60)
if hasattr(result, 'pydantic') and result.pydantic:
    r = result.pydantic
    print(f"Startup       : {r.startup_name}")
    print(f"Industry      : {r.industry}")
    print(f"Market Size   : {r.market_size}")
    print(f"Competitors   : {', '.join(r.competitors)}")
    print(f"Risk          : {r.risk_assessment}")
    print(f"Score         : {r.overall_score}/10")
    print(f"\nVerdict: {r.investment_verdict}")
else:
    print(result)
```

---

## Bonus Drills — Blank File Challenges

Write these from scratch without looking at anything.

### B1. Single agent with 3 custom tools
```
Build an agent with tools: text_upper, text_reverse, char_counter
Task: transform a given sentence using all three tools and report results
```

### B2. Two-agent proofreading crew
```
Agent 1 (Writer): Write a 5-sentence paragraph about Python
Agent 2 (Proofreader): Proofread and fix any errors
Sequential process, task context passed correctly
```

### B3. Hierarchical crew with 3 specialists
```
Manager delegates to: DataAgent, CodeAgent, ExplainerAgent
Task: "Explain how to calculate standard deviation with code and example"
```

### B4. Pydantic output crew
```
Model: JobPosting(title, company, skills: List[str], salary_range, remote: bool)
Agent: HR Analyst
Task: Generate a job posting for a "Senior LangGraph Engineer"
```

### B5. Dynamic input crew (kickoff_for_each)
```
Run the same crew for 3 different programming languages
Task: "Explain the best use case for {language} in 2 sentences"
```

### B6. Full pipeline — research → code → test
```
3 agents: Researcher, Developer, QA Engineer
3 tasks with context chaining
Final output: working Python code with test cases
```

---

## CrewAI vs LangGraph — When to Use What

| Situation | Use CrewAI | Use LangGraph |
|---|---|---|
| Multiple specialist agents collaborating | ✓ | |
| Role-based task delegation | ✓ | |
| Content production pipelines | ✓ | |
| You think in "teams and roles" | ✓ | |
| Fine-grained control over flow | | ✓ |
| Loops, retries, cycles in logic | | ✓ |
| Human-in-the-loop at specific steps | | ✓ |
| Custom graph topology | | ✓ |
| ReAct / single agent with tools | Either | |

---

## Progress Tracker

| Level | Topic                                    | Done? | Time Taken |
|-------|------------------------------------------|-------|------------|
| L1    | First Agent, Task, Crew                  |  [ ]  |            |
| L2    | Task Config, Context, Multi-task         |  [ ]  |            |
| L3    | Built-in Tools, Custom @tool             |  [ ]  |            |
| L4    | Class Tools, Agent Toolbelt              |  [ ]  |            |
| L5    | Sequential vs Hierarchical Process       |  [ ]  |            |
| L6    | Pydantic Output, JSON, File Output       |  [ ]  |            |
| L7    | Memory                                   |  [ ]  |            |
| L8    | Dynamic Inputs, kickoff_for_each         |  [ ]  |            |
| L9    | Content Production Pipeline              |  [ ]  |            |
| L10   | Full Multi-Agent Startup Analyser        |  [ ]  |            |
| Bonus | Blank File Challenges                    |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 18 (the Startup Analyser)
> from memory. Three agents, three tools, three tasks with context chaining,
> Pydantic output, dynamic inputs. When it runs and prints a structured report —
> you own CrewAI.
