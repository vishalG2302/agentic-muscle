# Chatbot Memory Layers — Muscle Memory Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `python filename.py`
> - Install: `pip install langchain langchain-openai langchain-community
>             langgraph chromadb faiss-cpu python-dotenv`
> - Set `OPENAI_API_KEY` in `.env`

---

## The Mental Model — Read This First

```
Humans have multiple types of memory. So should your chatbot.

┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY TYPES                                 │
├──────────────────┬──────────────────────────────────────────────┤
│ SHORT-TERM       │ The last few messages in this conversation   │
│                  │ Lives in the context window                  │
│                  │ Forgotten when session ends                  │
├──────────────────┼──────────────────────────────────────────────┤
│ WORKING          │ Current task, active reasoning scratch pad   │
│                  │ "What am I trying to do right now?"          │
│                  │ Cleared after task completes                 │
├──────────────────┼──────────────────────────────────────────────┤
│ SEMANTIC         │ Facts about the world / about the user       │
│                  │ "Vishal uses Python. He lives in Pune."      │
│                  │ Stored as embeddings in vector DB            │
├──────────────────┼──────────────────────────────────────────────┤
│ EPISODIC         │ Specific past events and conversations       │
│                  │ "On Monday Vishal asked about LangGraph"     │
│                  │ Stored with timestamps, retrievable          │
├──────────────────┼──────────────────────────────────────────────┤
│ LONG-TERM        │ Summarised knowledge that persists forever   │
│                  │ Old conversations compressed into summaries  │
│                  │ Stored in files / DB, survives restarts      │
└──────────────────┴──────────────────────────────────────────────┘

The problem most beginners hit:
  They only build short-term memory.
  The bot forgets everything on restart.
  It can't remember the user told it their name yesterday.
  It can't connect what the user asked Monday to what they asked Friday.

The goal of this sheet:
  Build each layer separately first.
  Then wire them together into one chatbot that has all layers.
```

---

## L1 — Short-Term Memory

### 1. Raw Message Buffer — The Simplest Memory
```python
# 01_raw_buffer.py
# Short-term memory = just keep the last N messages in a list
# This is what every beginner chatbot does

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

class ShortTermMemory:
    def __init__(self, system_prompt: str, max_messages: int = 10):
        self.system = SystemMessage(content=system_prompt)
        self.history = []
        self.max_messages = max_messages

    def add_human(self, text: str):
        self.history.append(HumanMessage(content=text))
        self._trim()

    def add_ai(self, text: str):
        self.history.append(AIMessage(content=text))
        self._trim()

    def _trim(self):
        # Keep only the last N messages to avoid context overflow
        if len(self.history) > self.max_messages:
            self.history = self.history[-self.max_messages:]

    def get_messages(self):
        return [self.system] + self.history

    def clear(self):
        self.history = []


def chat(memory: ShortTermMemory, user_input: str) -> str:
    memory.add_human(user_input)
    response = llm.invoke(memory.get_messages())
    memory.add_ai(response.content)
    return response.content


# --- Demo ---
memory = ShortTermMemory(
    system_prompt="You are a helpful assistant. Be concise.",
    max_messages=6
)

turns = [
    "My name is Vishal and I'm learning LangGraph.",
    "What am I learning?",          # should remember
    "What is my name?",             # should remember
]

for turn in turns:
    print(f"\n👤 {turn}")
    reply = chat(memory, turn)
    print(f"🤖 {reply}")

print(f"\nMessages in buffer: {len(memory.history)}")
```

---

### 2. Short-Term Memory with LangGraph Checkpointer
```python
# 02_langgraph_shortterm.py
# LangGraph's MemorySaver = short-term memory tied to a thread_id
# Same thread = same conversation history

from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, BaseMessage, SystemMessage
from typing import TypedDict, List
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)
SYSTEM = SystemMessage(content="You are a helpful assistant. Be concise.")

class State(TypedDict):
    messages: List[BaseMessage]

def chat_node(state: State) -> State:
    msgs = [SYSTEM] + state["messages"]
    response = llm.invoke(msgs)
    return {"messages": state["messages"] + [response]}

builder = StateGraph(State)
builder.add_node("chat", chat_node)
builder.set_entry_point("chat")
builder.add_edge("chat", END)

# MemorySaver stores state per thread_id — this IS short-term memory
memory = MemorySaver()
graph = builder.compile(checkpointer=memory)

def chat(user_input: str, thread_id: str) -> str:
    config = {"configurable": {"thread_id": thread_id}}
    result = graph.invoke(
        {"messages": [HumanMessage(content=user_input)]},
        config=config
    )
    return result["messages"][-1].content

# Same thread = shared memory
session = "vishal-session-01"

turns = [
    "My name is Vishal.",
    "I am learning LangGraph and FastAPI.",
    "What's my name and what am I learning?",  # tests memory
]

for turn in turns:
    print(f"\n👤 {turn}")
    print(f"🤖 {chat(turn, session)}")

# Different thread = fresh memory
print("\n--- New session (fresh memory) ---")
print(f"🤖 {chat('What is my name?', 'new-session-99')}")
```

---

### 3. Window Buffer — Sliding Window
```python
# 03_window_buffer.py
# Keep only the last K turns (not all messages)
# Prevents context window overflow on long conversations

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from dotenv import load_dotenv
from collections import deque

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class WindowBuffer:
    """Keeps last `window` conversation turns (1 turn = 1 human + 1 AI)."""
    def __init__(self, system_prompt: str, window: int = 3):
        self.system = SystemMessage(content=system_prompt)
        self.window = window
        # deque auto-discards oldest when maxlen is exceeded
        self.turns = deque(maxlen=window)

    def add_turn(self, human: str, ai: str):
        self.turns.append((human, ai))

    def get_messages(self, current_human: str) -> list:
        msgs = [self.system]
        for h, a in self.turns:
            msgs.append(HumanMessage(content=h))
            msgs.append(AIMessage(content=a))
        msgs.append(HumanMessage(content=current_human))
        return msgs

    def chat(self, user_input: str) -> str:
        msgs = self.get_messages(user_input)
        response = llm.invoke(msgs)
        self.add_turn(user_input, response.content)
        return response.content

    def show_buffer(self):
        print(f"\n[Buffer has {len(self.turns)}/{self.window} turns]")
        for i, (h, a) in enumerate(self.turns, 1):
            print(f"  Turn {i}: '{h[:40]}...' → '{a[:40]}...'")


bot = WindowBuffer("You are a helpful assistant.", window=3)

for msg in [
    "I love Python.",
    "I also enjoy building AI agents.",
    "My name is Vishal.",
    "I live in Pune, India.",
    "What do I love?",       # Turn 1 (Python) may be forgotten with window=3
    "What is my name?",      # Turn 3 (name) should still be in window
]:
    print(f"\n👤 {msg}")
    print(f"🤖 {bot.chat(msg)}")

bot.show_buffer()
```

---

## L2 — Working Memory

### 4. Working Memory — The Scratchpad
```python
# 04_working_memory.py
# Working memory = what the bot is actively thinking about right now
# It tracks: current task, sub-goals, intermediate results
# Cleared when task completes

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from dotenv import load_dotenv
from typing import Optional
import json

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class WorkingMemory:
    """Tracks the current task, sub-goals, and intermediate results."""
    def __init__(self):
        self.current_task: Optional[str] = None
        self.sub_goals: list = []
        self.intermediate_results: list = []
        self.scratchpad: str = ""

    def set_task(self, task: str):
        self.current_task = task
        self.sub_goals = []
        self.intermediate_results = []
        self.scratchpad = ""

    def add_sub_goal(self, goal: str):
        self.sub_goals.append(goal)

    def add_result(self, result: str):
        self.intermediate_results.append(result)

    def update_scratchpad(self, notes: str):
        self.scratchpad = notes

    def clear(self):
        self.current_task = None
        self.sub_goals = []
        self.intermediate_results = []
        self.scratchpad = ""

    def to_context(self) -> str:
        if not self.current_task:
            return "No active task."
        return f"""Current Task: {self.current_task}
Sub-goals: {self.sub_goals}
Results so far: {self.intermediate_results}
Scratchpad: {self.scratchpad}"""


class ThinkingBot:
    def __init__(self):
        self.llm = llm
        self.working_memory = WorkingMemory()

    def start_task(self, task: str) -> str:
        self.working_memory.set_task(task)

        # Ask LLM to break it into sub-goals
        prompt = f"""Break this task into 2-3 sub-goals.
Return ONLY a JSON array of strings.
Task: {task}"""
        response = self.llm.invoke([HumanMessage(content=prompt)])
        try:
            sub_goals = json.loads(response.content)
            for g in sub_goals:
                self.working_memory.add_sub_goal(g)
        except:
            self.working_memory.add_sub_goal("Complete the task")

        return f"Task set. Sub-goals: {self.working_memory.sub_goals}"

    def work_on_goal(self, goal_index: int) -> str:
        goals = self.working_memory.sub_goals
        if goal_index >= len(goals):
            return "No such sub-goal."

        context = self.working_memory.to_context()
        goal = goals[goal_index]

        prompt = f"""Working memory context:
{context}

Work on this sub-goal: {goal}
Provide a brief result."""
        response = self.llm.invoke([HumanMessage(content=prompt)])
        self.working_memory.add_result(f"[{goal}]: {response.content[:100]}")
        return response.content

    def finish(self) -> str:
        context = self.working_memory.to_context()
        prompt = f"""Given this working memory:\n{context}\n
Synthesise a final answer."""
        response = self.llm.invoke([HumanMessage(content=prompt)])
        self.working_memory.clear()
        return response.content


# --- Demo ---
bot = ThinkingBot()

print(bot.start_task("Explain the benefits of using RAG in a chatbot"))
print("\n--- Working on sub-goals ---")
print(bot.work_on_goal(0))
print(bot.work_on_goal(1))
print("\n--- Final Answer ---")
print(bot.finish())
print("\nWorking memory after finish:", bot.working_memory.current_task)
```

---

## L3 — Semantic Memory

### 5. Semantic Memory — Facts About the User
```python
# 05_semantic_memory.py
# Semantic memory = facts, preferences, knowledge about the user
# Stored as embeddings so you can retrieve by meaning, not exact text

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import time

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class SemanticMemory:
    """Stores and retrieves facts by meaning using vector search."""
    def __init__(self):
        self.store = None
        self.facts = []

    def remember(self, fact: str, category: str = "general"):
        """Store a new fact."""
        doc = Document(
            page_content=fact,
            metadata={
                "category": category,
                "timestamp": time.strftime("%Y-%m-%d %H:%M"),
            }
        )
        self.facts.append(doc)
        # Rebuild store (in production: use add_documents)
        self.store = FAISS.from_documents(self.facts, embedder)
        return f"Remembered: {fact}"

    def recall(self, query: str, k: int = 3) -> list[str]:
        """Retrieve relevant facts for a query."""
        if not self.store:
            return []
        results = self.store.similarity_search(query, k=k)
        return [doc.page_content for doc in results]

    def recall_as_context(self, query: str, k: int = 3) -> str:
        facts = self.recall(query, k=k)
        if not facts:
            return "No relevant memories found."
        return "What I know:\n" + "\n".join(f"- {f}" for f in facts)


# --- Demo ---
mem = SemanticMemory()

# Store facts about Vishal
facts = [
    ("Vishal's name is Vishal Gupta.", "identity"),
    ("Vishal lives in Pune, India.", "location"),
    ("Vishal is 22 years old.", "identity"),
    ("Vishal is learning LangGraph and FastAPI.", "learning"),
    ("Vishal's goal is to become an LLM engineer.", "goal"),
    ("Vishal prefers Python over other languages.", "preference"),
    ("Vishal is building an image memory system using Neo4j.", "project"),
    ("Vishal practices coding without AI assistance.", "habit"),
]

for fact, category in facts:
    mem.remember(fact, category)

print("=== Semantic Memory Demo ===\n")

queries = [
    "Where does the user live?",
    "What is the user building?",
    "What technologies is the user learning?",
    "What are the user's career goals?",
]

for q in queries:
    print(f"❓ {q}")
    context = mem.recall_as_context(q)
    print(f"📚 {context}\n")
```

---

### 6. Extract and Store Facts Automatically
```python
# 06_auto_extract_facts.py
# Instead of manually writing facts, extract them from conversation
# LLM reads the message and decides what facts are worth remembering

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import json
import time

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def extract_facts(message: str) -> list[str]:
    """Use LLM to extract memorable facts from a message."""
    prompt = f"""Extract facts worth remembering about the user from this message.
Return ONLY a JSON array of strings. Empty array [] if nothing to remember.
Only extract concrete facts: name, location, age, job, preferences, goals, projects.
Do NOT extract questions or generic statements.

Message: "{message}"

Facts (JSON array):"""
    response = llm.invoke([HumanMessage(content=prompt)])
    try:
        facts = json.loads(response.content.strip())
        return facts if isinstance(facts, list) else []
    except:
        return []


class AutoSemanticMemory:
    def __init__(self):
        self.docs = []
        self.store = None

    def process_message(self, message: str) -> list[str]:
        facts = extract_facts(message)
        for fact in facts:
            doc = Document(
                page_content=fact,
                metadata={"timestamp": time.strftime("%Y-%m-%d %H:%M")}
            )
            self.docs.append(doc)
        if self.docs:
            self.store = FAISS.from_documents(self.docs, embedder)
        return facts

    def recall(self, query: str, k: int = 3) -> str:
        if not self.store:
            return "No memories yet."
        results = self.store.similarity_search(query, k=k)
        facts = [doc.page_content for doc in results]
        return "\n".join(f"• {f}" for f in facts)


# --- Demo ---
mem = AutoSemanticMemory()

messages = [
    "Hey! I'm Vishal, 22 years old, from Pune.",
    "I've been building an image memory system for AI agents.",
    "My goal is to land a job as an LLM engineer within 60 days.",
    "What's the weather like today?",     # no facts to extract
    "I prefer working with Python and Neo4j over other tools.",
]

print("=== Auto Fact Extraction ===\n")
for msg in messages:
    facts = mem.process_message(msg)
    print(f"Message: '{msg[:50]}...'")
    print(f"Extracted: {facts}\n")

print("=== Recall ===")
print(mem.recall("What is the user building?"))
```

---

## L4 — Episodic Memory

### 7. Episodic Memory — Recording Past Events
```python
# 07_episodic_memory.py
# Episodic memory = specific events with WHEN they happened
# "On Monday, Vishal asked about LangGraph nodes"
# Lets the bot remember the arc of a relationship over time

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
from datetime import datetime
import json

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class EpisodicMemory:
    """Stores conversation episodes — what happened, when, what was said."""
    def __init__(self):
        self.episodes = []
        self.store = None

    def record_episode(self, user_msg: str, bot_reply: str, tags: list = None):
        """Record a conversation turn as an episode."""
        now = datetime.now()
        episode_text = f"""Date: {now.strftime("%Y-%m-%d %H:%M")}
User said: {user_msg}
Bot replied: {bot_reply[:150]}"""

        doc = Document(
            page_content=episode_text,
            metadata={
                "timestamp": now.isoformat(),
                "date": now.strftime("%Y-%m-%d"),
                "user_message": user_msg[:100],
                "tags": tags or [],
            }
        )
        self.episodes.append(doc)
        self.store = FAISS.from_documents(self.episodes, embedder)

    def recall_episodes(self, query: str, k: int = 3) -> list[dict]:
        """Find past episodes relevant to current query."""
        if not self.store:
            return []
        results = self.store.similarity_search(query, k=k)
        return [
            {
                "date": doc.metadata["date"],
                "content": doc.page_content,
                "tags": doc.metadata["tags"]
            }
            for doc in results
        ]

    def recall_as_context(self, query: str) -> str:
        episodes = self.recall_episodes(query)
        if not episodes:
            return "No relevant past conversations."
        lines = ["Past conversations:"]
        for ep in episodes:
            lines.append(f"\n[{ep['date']}]")
            for line in ep['content'].split('\n'):
                lines.append(f"  {line}")
        return "\n".join(lines)


# --- Demo ---
em = EpisodicMemory()

# Simulate past sessions
past_turns = [
    ("What are LangGraph nodes?",
     "LangGraph nodes are plain Python functions that take and return state.",
     ["langgraph", "learning"]),
    ("How do I add memory to my chatbot?",
     "You can use LangGraph's MemorySaver checkpointer for short-term memory.",
     ["memory", "chatbot"]),
    ("I finished the FastAPI muscle memory sheet today!",
     "That's great progress! FastAPI is a key skill for LLM engineering.",
     ["progress", "fastapi"]),
    ("Can you explain RAG to me?",
     "RAG stands for Retrieval-Augmented Generation. It retrieves relevant docs and passes them to an LLM.",
     ["rag", "learning"]),
]

for user_msg, bot_reply, tags in past_turns:
    em.record_episode(user_msg, bot_reply, tags)

print("=== Episodic Memory Recall ===\n")
queries = [
    "What have we discussed about LangGraph?",
    "What progress has the user made?",
]

for q in queries:
    print(f"❓ {q}")
    print(em.recall_as_context(q))
    print()
```

---

## L5 — Long-Term Memory

### 8. Long-Term Memory — Persistent Storage
```python
# 08_longterm_persistent.py
# Long-term memory = survives restarts
# Most basic form: write to a JSON file
# Production form: write to a database

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import json
import os
from datetime import datetime

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
MEMORY_FILE = "user_memory.json"

class LongTermMemory:
    """Persists user facts and summaries to disk. Survives restarts."""

    def __init__(self, user_id: str):
        self.user_id = user_id
        self.file = f"memory_{user_id}.json"
        self.data = self._load()

    def _load(self) -> dict:
        if os.path.exists(self.file):
            with open(self.file) as f:
                return json.load(f)
        return {
            "user_id": self.user_id,
            "facts": [],
            "summaries": [],
            "preferences": {},
            "created_at": datetime.now().isoformat(),
            "last_updated": None,
        }

    def _save(self):
        self.data["last_updated"] = datetime.now().isoformat()
        with open(self.file, "w") as f:
            json.dump(self.data, f, indent=2)

    def add_fact(self, fact: str, category: str = "general"):
        self.data["facts"].append({
            "fact": fact,
            "category": category,
            "timestamp": datetime.now().isoformat()
        })
        self._save()

    def add_summary(self, summary: str, session_date: str = None):
        self.data["summaries"].append({
            "summary": summary,
            "date": session_date or datetime.now().strftime("%Y-%m-%d"),
        })
        self._save()

    def set_preference(self, key: str, value: str):
        self.data["preferences"][key] = value
        self._save()

    def get_context(self) -> str:
        lines = []
        if self.data["facts"]:
            lines.append("Known facts:")
            for f in self.data["facts"][-10:]:  # last 10 facts
                lines.append(f"  - [{f['category']}] {f['fact']}")
        if self.data["preferences"]:
            lines.append("Preferences:")
            for k, v in self.data["preferences"].items():
                lines.append(f"  - {k}: {v}")
        if self.data["summaries"]:
            lines.append("Past sessions:")
            for s in self.data["summaries"][-3:]:  # last 3 summaries
                lines.append(f"  [{s['date']}] {s['summary']}")
        return "\n".join(lines) if lines else "No long-term memory yet."

    def clear(self):
        os.remove(self.file) if os.path.exists(self.file) else None
        self.data = self._load()


# --- Demo ---
mem = LongTermMemory("vishal")
mem.clear()  # start fresh for demo

# Simulate storing things over time
mem.add_fact("User's name is Vishal", "identity")
mem.add_fact("User lives in Pune, India", "location")
mem.add_fact("User is learning LangGraph and FastAPI", "learning")
mem.set_preference("language", "Python")
mem.set_preference("response_style", "concise")
mem.add_summary("Vishal asked about LangGraph nodes and memory patterns.")
mem.add_summary("Vishal completed FastAPI and FastMCP practice sheets.")

print("=== Long-Term Memory Context ===\n")
print(mem.get_context())

# Simulate a restart — reload from disk
print("\n--- Simulating restart ---")
mem2 = LongTermMemory("vishal")  # loads from file
print("\n=== Memory After Restart ===")
print(mem2.get_context())
```

---

### 9. Summarisation Memory — Compress Old Conversations
```python
# 09_summarise_memory.py
# When conversation gets too long:
# Summarise old messages → store the summary → drop old messages
# This is how you give the bot "memory" beyond the context window

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage, BaseMessage
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
SUMMARISE_THRESHOLD = 6  # summarise when buffer exceeds this many messages

def summarise_conversation(messages: list[BaseMessage]) -> str:
    """Use LLM to compress a list of messages into a summary."""
    transcript = "\n".join(
        f"{'User' if isinstance(m, HumanMessage) else 'Bot'}: {m.content}"
        for m in messages
        if not isinstance(m, SystemMessage)
    )
    prompt = f"""Summarise this conversation in 3-4 sentences.
Focus on: key facts shared, topics discussed, decisions made.
Be specific and factual.

Conversation:
{transcript}

Summary:"""
    return llm.invoke([HumanMessage(content=prompt)]).content


class SummarisationMemory:
    """Keeps recent messages in buffer. Summarises when buffer gets too long."""
    def __init__(self, system_prompt: str, threshold: int = 6):
        self.system_prompt = system_prompt
        self.threshold = threshold
        self.summary = ""          # compressed history
        self.recent = []           # recent messages (kept in full)

    def add_exchange(self, human: str, ai: str):
        self.recent.append(HumanMessage(content=human))
        self.recent.append(AIMessage(content=ai))
        # Compress if too long
        if len(self.recent) >= self.threshold:
            self._compress()

    def _compress(self):
        # Summarise the older half, keep the recent half
        half = len(self.recent) // 2
        to_summarise = self.recent[:half]
        new_summary = summarise_conversation(to_summarise)
        if self.summary:
            self.summary = f"{self.summary}\n{new_summary}"
        else:
            self.summary = new_summary
        self.recent = self.recent[half:]
        print(f"  [Memory compressed. Summary updated. Buffer: {len(self.recent)} msgs]")

    def build_messages(self, user_input: str) -> list[BaseMessage]:
        msgs = [SystemMessage(content=self.system_prompt)]
        if self.summary:
            msgs.append(SystemMessage(
                content=f"Summary of earlier conversation:\n{self.summary}"
            ))
        msgs.extend(self.recent)
        msgs.append(HumanMessage(content=user_input))
        return msgs

    def chat(self, user_input: str) -> str:
        msgs = self.build_messages(user_input)
        response = llm.invoke(msgs)
        self.add_exchange(user_input, response.content)
        return response.content


# --- Demo ---
bot = SummarisationMemory("You are a helpful assistant. Be concise.", threshold=6)

turns = [
    "My name is Vishal.",
    "I'm 22 and live in Pune.",
    "I'm building an image memory system.",
    "I use Neo4j and LangGraph for it.",
    "My goal is an LLM engineer role.",   # buffer full → compresses here
    "I also practice Python every day.",
    "What is my name and what am I building?",  # tests if summary preserved facts
    "Where do I live?",
]

for turn in turns:
    print(f"\n👤 {turn}")
    print(f"🤖 {bot.chat(turn)}")

print(f"\n=== Current Summary ===\n{bot.summary}")
print(f"=== Recent Buffer ===\n{len(bot.recent)} messages")
```

---

## L6 — Entity Memory

### 10. Entity Memory — Track People, Places, Things
```python
# 10_entity_memory.py
# Entity memory = extract and remember entities mentioned in conversation
# "Vishal" → {name, location, age, skills, project}
# "Neo4j" → {type: database, use: knowledge graphs}

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import json

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class EntityMemory:
    """Tracks entities (people, tools, projects) mentioned in conversation."""
    def __init__(self):
        self.entities: dict = {}

    def extract_entities(self, message: str) -> dict:
        """Extract entities and their properties from a message."""
        prompt = f"""Extract named entities from this message.
For each entity return its name and a brief description/property.
Return ONLY a JSON object like: {{"EntityName": "what we learned about it"}}
Return {{}} if nothing relevant.

Message: "{message}"

Entities:"""
        response = llm.invoke([HumanMessage(content=prompt)])
        try:
            return json.loads(response.content.strip())
        except:
            return {}

    def update(self, message: str) -> dict:
        """Extract entities and merge into memory."""
        new = self.extract_entities(message)
        for name, description in new.items():
            if name in self.entities:
                # Append new info to existing entity
                self.entities[name] += f" | {description}"
            else:
                self.entities[name] = description
        return new

    def get_entity(self, name: str) -> str:
        return self.entities.get(name, f"Nothing known about {name}")

    def get_context_for(self, message: str) -> str:
        """Return entity context relevant to the current message."""
        relevant = []
        message_lower = message.lower()
        for name, desc in self.entities.items():
            if name.lower() in message_lower:
                relevant.append(f"{name}: {desc}")
        if not relevant:
            return ""
        return "Known entities:\n" + "\n".join(f"  - {r}" for r in relevant)

    def show_all(self):
        print("\n=== Entity Memory ===")
        for name, desc in self.entities.items():
            print(f"  {name}: {desc}")


# --- Demo ---
em = EntityMemory()

messages = [
    "I'm Vishal, a data science student at IIT Madras.",
    "I'm building an image memory system using Neo4j and LangGraph.",
    "My friend Alice is helping me test the Neo4j integration.",
    "FastMCP is the library I use for MCP server building.",
    "What do you know about Neo4j?",
    "Tell me what you know about Vishal.",
]

print("=== Entity Memory Demo ===\n")
for msg in messages:
    new_entities = em.update(msg)
    context = em.get_context_for(msg)
    print(f"Message: '{msg[:60]}'")
    if new_entities:
        print(f"  Extracted: {new_entities}")
    if context:
        print(f"  Context: {context}")
    print()

em.show_all()
```

---

## L7 — Combining Memory Layers

### 11. Memory Manager — All Layers in One
```python
# 11_memory_manager.py
# Wire all layers together:
# Short-term  → recent messages (buffer)
# Semantic    → user facts (vector store)
# Long-term   → persisted summaries (disk)
# Entity      → named entities (dict)

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from dotenv import load_dotenv
from datetime import datetime
import json, os

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.5)


class MemoryManager:
    """Combines short-term, semantic, long-term, and entity memory."""

    def __init__(self, user_id: str, system_prompt: str):
        self.user_id = user_id
        self.system_prompt = system_prompt

        # Short-term: recent messages
        self.short_term: list = []
        self.SHORT_TERM_LIMIT = 6

        # Semantic: user facts
        self.facts: list[Document] = []
        self.semantic_store = None

        # Long-term: persisted to disk
        self.lt_file = f"lt_{user_id}.json"
        self.long_term = self._load_lt()

        # Entity: named entity tracking
        self.entities: dict = {}

    # ── Short-term ──────────────────────────────────────
    def add_to_short_term(self, human: str, ai: str):
        self.short_term.append(HumanMessage(content=human))
        self.short_term.append(AIMessage(content=ai))
        if len(self.short_term) > self.SHORT_TERM_LIMIT:
            self.short_term = self.short_term[-self.SHORT_TERM_LIMIT:]

    # ── Semantic ────────────────────────────────────────
    def remember_fact(self, fact: str):
        doc = Document(
            page_content=fact,
            metadata={"user": self.user_id, "ts": datetime.now().isoformat()}
        )
        self.facts.append(doc)
        self.semantic_store = FAISS.from_documents(self.facts, embedder)

    def recall_facts(self, query: str, k: int = 3) -> str:
        if not self.semantic_store:
            return ""
        results = self.semantic_store.similarity_search(query, k=k)
        return "\n".join(f"• {d.page_content}" for d in results)

    # ── Long-term ───────────────────────────────────────
    def _load_lt(self) -> dict:
        if os.path.exists(self.lt_file):
            with open(self.lt_file) as f:
                return json.load(f)
        return {"summaries": [], "key_facts": []}

    def _save_lt(self):
        with open(self.lt_file, "w") as f:
            json.dump(self.long_term, f, indent=2)

    def save_session_summary(self, summary: str):
        self.long_term["summaries"].append({
            "date": datetime.now().strftime("%Y-%m-%d"),
            "summary": summary
        })
        self._save_lt()

    def get_lt_context(self) -> str:
        summaries = self.long_term.get("summaries", [])[-3:]
        if not summaries:
            return ""
        return "Past sessions:\n" + "\n".join(
            f"  [{s['date']}] {s['summary']}" for s in summaries
        )

    # ── Entity ──────────────────────────────────────────
    def update_entity(self, name: str, info: str):
        if name in self.entities:
            self.entities[name] += f" | {info}"
        else:
            self.entities[name] = info

    def entity_context(self, message: str) -> str:
        relevant = [
            f"{n}: {d}" for n, d in self.entities.items()
            if n.lower() in message.lower()
        ]
        return "Entities: " + "; ".join(relevant) if relevant else ""

    # ── Compose context ─────────────────────────────────
    def build_messages(self, user_input: str) -> list:
        semantic = self.recall_facts(user_input)
        lt_ctx = self.get_lt_context()
        entity_ctx = self.entity_context(user_input)

        extra_context = "\n".join(filter(None, [
            "User facts:\n" + semantic if semantic else "",
            lt_ctx,
            entity_ctx
        ]))

        system_content = self.system_prompt
        if extra_context:
            system_content += f"\n\nMemory context:\n{extra_context}"

        return (
            [SystemMessage(content=system_content)]
            + self.short_term
            + [HumanMessage(content=user_input)]
        )

    # ── Chat ────────────────────────────────────────────
    def chat(self, user_input: str) -> str:
        msgs = self.build_messages(user_input)
        response = llm.invoke(msgs)
        self.add_to_short_term(user_input, response.content)
        return response.content


# --- Demo ---
bot = MemoryManager(
    user_id="vishal",
    system_prompt="You are a helpful AI assistant. Use memory context to personalise responses."
)

# Seed semantic memory
bot.remember_fact("User's name is Vishal")
bot.remember_fact("User lives in Pune, India")
bot.remember_fact("User is building an image memory system with Neo4j and LangGraph")
bot.remember_fact("User's goal is to become an LLM engineer in 60 days")

# Seed long-term
bot.save_session_summary(
    "Vishal completed FastAPI and FastMCP practice sheets. Discussed memory layers for chatbots."
)

# Entity tracking
bot.update_entity("Neo4j", "graph database used for knowledge graphs")
bot.update_entity("LangGraph", "agentic workflow framework")

print("=== Memory-Powered Chatbot ===\n")
turns = [
    "What am I building?",
    "Remind me what my goal is.",
    "What did I complete recently?",
    "Tell me about Neo4j.",
]

for turn in turns:
    print(f"👤 {turn}")
    print(f"🤖 {bot.chat(turn)}\n")
```

---

## L8 — Memory with LangGraph State

### 12. Full Memory State in LangGraph
```python
# 12_langgraph_memory_state.py
# Put ALL memory types inside LangGraph State
# Every node can read and write to shared memory

from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage, BaseMessage
from typing import TypedDict, List, Dict, Optional
from dotenv import load_dotenv
import json

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class MemoryState(TypedDict):
    messages: List[BaseMessage]         # short-term
    user_facts: List[str]               # semantic
    entities: Dict[str, str]            # entity
    session_summary: str                # long-term
    current_input: str

# ── Nodes ──────────────────────────────────────────────
def extract_facts_node(state: MemoryState) -> MemoryState:
    """Extract facts from the latest user message and store them."""
    last_human = next(
        (m.content for m in reversed(state["messages"]) if isinstance(m, HumanMessage)),
        ""
    )
    if not last_human:
        return {}

    prompt = f"""Extract memorable user facts from: "{last_human}"
Return ONLY a JSON array of strings, or [] if nothing to remember.
Only extract: name, location, age, goals, projects, preferences."""
    response = llm.invoke([HumanMessage(content=prompt)])
    try:
        new_facts = json.loads(response.content.strip())
        if isinstance(new_facts, list) and new_facts:
            updated = list(set(state["user_facts"] + new_facts))
            return {"user_facts": updated}
    except:
        pass
    return {}

def build_context_node(state: MemoryState) -> MemoryState:
    """Build the system context from all memory layers."""
    # This node doesn't change state, just prepares context
    # (in practice you might store the built context in state)
    return {}

def respond_node(state: MemoryState) -> MemoryState:
    """Generate a response using all memory context."""
    context_parts = []

    if state["user_facts"]:
        context_parts.append("Known facts:\n" + "\n".join(f"• {f}" for f in state["user_facts"]))

    if state["entities"]:
        entity_str = "\n".join(f"• {k}: {v}" for k, v in state["entities"].items())
        context_parts.append(f"Known entities:\n{entity_str}")

    if state["session_summary"]:
        context_parts.append(f"Previous sessions:\n{state['session_summary']}")

    system_content = "You are a helpful assistant with memory.\n"
    if context_parts:
        system_content += "\nMemory:\n" + "\n\n".join(context_parts)

    msgs = [SystemMessage(content=system_content)] + state["messages"]
    response = llm.invoke(msgs)

    return {"messages": state["messages"] + [response]}

# ── Graph ──────────────────────────────────────────────
builder = StateGraph(MemoryState)
builder.add_node("extract", extract_facts_node)
builder.add_node("context", build_context_node)
builder.add_node("respond", respond_node)

builder.set_entry_point("extract")
builder.add_edge("extract", "context")
builder.add_edge("context", "respond")
builder.add_edge("respond", END)

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

def chat(user_input: str, thread_id: str, session_state: dict) -> str:
    config = {"configurable": {"thread_id": thread_id}}
    current_msgs = session_state.get("messages", [])
    result = graph.invoke(
        {
            "messages": current_msgs + [HumanMessage(content=user_input)],
            "user_facts": session_state.get("user_facts", []),
            "entities": session_state.get("entities", {}),
            "session_summary": session_state.get("session_summary", ""),
            "current_input": user_input,
        },
        config=config
    )
    session_state.update(result)
    return result["messages"][-1].content

# --- Demo ---
state = {
    "messages": [],
    "user_facts": [],
    "entities": {"LangGraph": "stateful agent framework"},
    "session_summary": "Previously: user asked about RAG and FastAPI.",
}

turns = [
    "My name is Vishal and I'm 22.",
    "I'm building a knowledge graph system.",
    "What's my name?",
    "What am I building?",
    "What did we discuss before?",
]

for turn in turns:
    print(f"\n👤 {turn}")
    reply = chat(turn, "vishal-thread-01", state)
    print(f"🤖 {reply}")

print(f"\nFacts stored: {state['user_facts']}")
```

---

## L9 — Memory Retrieval Patterns

### 13. Relevance-Gated Memory — Only Inject What's Needed
```python
# 13_relevance_gated.py
# Don't dump ALL memory into every prompt — it wastes tokens and confuses the LLM
# Only inject memory that is relevant to the CURRENT question

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage, SystemMessage
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# All user memories
all_memories = [
    Document(page_content="User's name is Vishal."),
    Document(page_content="User lives in Pune, India."),
    Document(page_content="User is building an image memory system."),
    Document(page_content="User prefers Python over JavaScript."),
    Document(page_content="User's goal is to become an LLM engineer."),
    Document(page_content="User practices coding 4 hours per day."),
    Document(page_content="User uses Neo4j for graph storage."),
    Document(page_content="User is studying Rust slowly on the side."),
]

store = FAISS.from_documents(all_memories, embedder)

def chat_with_relevant_memory(question: str, k: int = 3) -> str:
    # Step 1: Retrieve only relevant memories
    relevant = store.similarity_search(question, k=k)
    memory_context = "\n".join(f"- {doc.page_content}" for doc in relevant)

    # Step 2: Build prompt with only relevant memory
    prompt = f"""You are a helpful assistant with memory about the user.

Relevant memories:
{memory_context}

Answer the question using the memories above.
If the memory doesn't contain the answer, say so.

Question: {question}"""

    response = llm.invoke([HumanMessage(content=prompt)])
    return response.content

print("=== Relevance-Gated Memory ===\n")

questions = [
    "Where does the user live?",
    "What is the user building?",
    "What programming language does the user prefer?",
    "What are the user's hobbies?",  # not in memory
]

for q in questions:
    print(f"❓ {q}")
    print(f"🤖 {chat_with_relevant_memory(q)}\n")
```

---

## L10 — Full Memory-Powered Chatbot

### 14. Complete Chatbot — All Layers Wired Together
```python
# 14_full_memory_chatbot.py
# Everything in one file:
# Short-term buffer + Semantic (vector) + Long-term (disk) + Entity + Summarisation

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from dotenv import load_dotenv
from datetime import datetime
from collections import deque
import json, os

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.5)


class Chatbot:
    SYSTEM_PROMPT = """You are a helpful, personalised AI assistant.
Use the memory context provided to give relevant, personalised responses.
Be concise. If you know facts about the user, use them naturally."""

    def __init__(self, user_id: str):
        self.user_id = user_id
        self.lt_file = f"memory_{user_id}.json"

        # Short-term: sliding window of recent messages
        self.buffer = deque(maxlen=8)

        # Semantic: vector store of user facts
        self.fact_docs: list[Document] = []
        self.fact_store = None

        # Entity: named entity tracking
        self.entities: dict = {}

        # Long-term: loaded from and saved to disk
        self.long_term = self._load_lt()

        # Session stats
        self.turn_count = 0

    # ── Long-term persistence ───────────────────────────
    def _load_lt(self) -> dict:
        if os.path.exists(self.lt_file):
            with open(self.lt_file) as f:
                return json.load(f)
        return {"facts": [], "summaries": [], "entities": {}}

    def _save_lt(self):
        self.long_term["entities"] = self.entities
        with open(self.lt_file, "w") as f:
            json.dump(self.long_term, f, indent=2)

    # ── Fact extraction ─────────────────────────────────
    def _extract_and_store_facts(self, message: str):
        prompt = f"""Extract memorable facts about the user from this message.
Return ONLY a JSON array of short strings. Return [] if nothing to extract.
Facts to extract: name, age, location, job, goal, project, preference, skill.

Message: "{message}"
Facts:"""
        try:
            response = llm.invoke([HumanMessage(content=prompt)])
            facts = json.loads(response.content.strip())
            for fact in (facts or []):
                doc = Document(page_content=fact,
                               metadata={"ts": datetime.now().isoformat()})
                self.fact_docs.append(doc)
                self.long_term["facts"].append(fact)
            if self.fact_docs:
                self.fact_store = FAISS.from_documents(self.fact_docs, embedder)
            self._save_lt()
        except:
            pass

    # ── Entity extraction ───────────────────────────────
    def _extract_entities(self, message: str):
        prompt = f"""Extract named entities (people, tools, projects, places) from:
"{message}"
Return ONLY JSON: {{"EntityName": "one-line description"}} or {{}}"""
        try:
            response = llm.invoke([HumanMessage(content=prompt)])
            new = json.loads(response.content.strip())
            for name, desc in new.items():
                self.entities[name] = (
                    f"{self.entities[name]} | {desc}"
                    if name in self.entities else desc
                )
            self._save_lt()
        except:
            pass

    # ── Recall ──────────────────────────────────────────
    def _recall_facts(self, query: str, k: int = 4) -> str:
        if not self.fact_store:
            return ""
        docs = self.fact_store.similarity_search(query, k=k)
        return "\n".join(f"• {d.page_content}" for d in docs)

    def _recall_entities(self, message: str) -> str:
        hits = [f"{n}: {d}" for n, d in self.entities.items()
                if n.lower() in message.lower()]
        return "Entities: " + " | ".join(hits) if hits else ""

    def _recall_lt(self) -> str:
        summaries = self.long_term.get("summaries", [])[-2:]
        if not summaries:
            return ""
        return "Past sessions:\n" + "\n".join(
            f"  [{s['date']}] {s['summary']}" for s in summaries
        )

    # ── Summarise session ───────────────────────────────
    def summarise_session(self):
        if not self.buffer:
            return
        transcript = "\n".join(
            f"{'User' if isinstance(m, HumanMessage) else 'Bot'}: {m.content}"
            for m in self.buffer
        )
        prompt = f"Summarise in 2 sentences:\n{transcript}"
        summary = llm.invoke([HumanMessage(content=prompt)]).content
        self.long_term["summaries"].append({
            "date": datetime.now().strftime("%Y-%m-%d"),
            "summary": summary
        })
        self._save_lt()
        print(f"\n[Session summary saved: {summary[:80]}...]")

    # ── Build messages ──────────────────────────────────
    def _build_messages(self, user_input: str) -> list:
        facts = self._recall_facts(user_input)
        entities = self._recall_entities(user_input)
        lt_ctx = self._recall_lt()

        context = "\n\n".join(filter(None, [
            "User facts:\n" + facts if facts else "",
            entities,
            lt_ctx,
        ]))

        system_content = self.SYSTEM_PROMPT
        if context:
            system_content += f"\n\nMemory context:\n{context}"

        return (
            [SystemMessage(content=system_content)]
            + list(self.buffer)
            + [HumanMessage(content=user_input)]
        )

    # ── Chat ────────────────────────────────────────────
    def chat(self, user_input: str) -> str:
        # Extract and store memories in background
        self._extract_and_store_facts(user_input)
        self._extract_entities(user_input)

        # Build context-aware messages
        msgs = self._build_messages(user_input)
        response = llm.invoke(msgs)
        reply = response.content

        # Update short-term buffer
        self.buffer.append(HumanMessage(content=user_input))
        self.buffer.append(AIMessage(content=reply))

        self.turn_count += 1
        return reply


# ── Run ────────────────────────────────────────────────
if __name__ == "__main__":
    bot = Chatbot("vishal")

    print("=== Memory-Powered Chatbot ===")
    print("Type 'quit' to exit, 'memory' to see stored facts, 'save' to save session\n")

    while True:
        user_input = input("You: ").strip()
        if not user_input:
            continue
        if user_input.lower() == "quit":
            bot.summarise_session()
            print("Goodbye!")
            break
        if user_input.lower() == "memory":
            print(f"\nFacts: {bot.long_term.get('facts', [])}")
            print(f"Entities: {bot.entities}")
            print(f"Summaries: {bot.long_term.get('summaries', [])}\n")
            continue
        if user_input.lower() == "save":
            bot.summarise_session()
            continue

        reply = bot.chat(user_input)
        print(f"Bot: {reply}\n")
```

---

## Bonus Drills

### B1. Build a memory system that detects and corrects stale facts
```
If user says "I moved to Mumbai" — update the old "lives in Pune" fact
instead of storing a contradiction
```

### B2. Add a forgetting mechanism
```
Facts older than 7 days lose their weight in retrieval
Implement by boosting recency score during similarity search
```

### B3. Build a reflection node for LangGraph
```
After every 5 turns, run a "reflect" node that:
- Summarises what was discussed
- Extracts 3 key learnings
- Stores them in long-term memory
```

### B4. Multi-user memory isolation
```
Build the MemoryManager so that user_id="vishal" and user_id="alice"
have completely isolated fact stores, entity stores, and disk files
```

### B5. Build a "what do you know about me?" endpoint
```
FastAPI GET /memory/{user_id}
Returns all facts, entities, and session summaries for that user
```

---

## Cheat Sheet — Memory Types at a Glance

```
Short-term  → list of HumanMessage/AIMessage  → trimmed to last N
Working     → dict with current_task, sub_goals, scratchpad
Semantic    → FAISS vector store of facts      → recalled by similarity
Episodic    → FAISS vector store of events     → recalled by similarity + timestamp
Long-term   → JSON file on disk                → loaded on startup, saved on write
Entity      → plain dict {name: description}  → matched by string lookup
Summarise   → compress old messages via LLM   → store summary, drop old msgs

The question to ask for each layer:
  Short-term  → "What did we say in this conversation?"
  Working     → "What am I trying to do right now?"
  Semantic    → "What facts do I know about this user?"
  Episodic    → "What happened last Tuesday with this user?"
  Long-term   → "What do I know across all past sessions?"
  Entity      → "What do I know about Neo4j specifically?"
```

---

## Progress Tracker

| Level | Topic                                       | Done? | Time Taken |
|-------|---------------------------------------------|-------|------------|
| L1    | Short-term buffer, window buffer, LangGraph |  [ ]  |            |
| L2    | Working memory / scratchpad                 |  [ ]  |            |
| L3    | Semantic memory with vector store           |  [ ]  |            |
| L4    | Auto fact extraction                        |  [ ]  |            |
| L5    | Episodic memory with timestamps             |  [ ]  |            |
| L6    | Long-term persistence to disk               |  [ ]  |            |
| L7    | Summarisation memory                        |  [ ]  |            |
| L8    | Entity memory                               |  [ ]  |            |
| L9    | Memory manager — all layers combined        |  [ ]  |            |
| L10   | Full production chatbot from scratch        |  [ ]  |            |
| Bonus | Stale facts, forgetting, reflection node    |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 14 (the full chatbot)
> from memory. All 5 layers wired together. Start a conversation. Type "memory"
> and see your facts stored. Quit and restart. Type "What's my name?" — if it
> still knows, your long-term memory works.
> That's the milestone.
