# 💪 Code Muscle Memory — Practice Sheets

> **Stop reading tutorials. Start typing code.**

A collection of structured practice sheets for developers who want to build **real muscle memory** — the ability to open a blank file and code without waiting for AI suggestions, autocomplete, or Stack Overflow.

Each sheet takes you from L1 (basics) to L10 (build a full app from scratch), with 40–55 exercises per topic.

---

## The One Rule

```
No AI. No autocomplete. No copy-paste.
Type every single line yourself.
```

That's it. That's the whole method.

The moment you turn off autocomplete and type through the discomfort — your hands start to remember. After 2–3 days per sheet, you'll notice you're not thinking about syntax anymore. You're thinking about the problem. That's the goal.

---

## Sheets Available

| # | Sheet | What you build by L10 | Time to complete |
|---|-------|----------------------|------------------|
| 01 | [Python](./python_muscle_memory.md) | CLI Task Manager (CRUD) | 2–3 days |
| 02 | [C++](./cpp_muscle_memory.md) | CLI Task Manager with OOP | 3–4 days |
| 03 | [TypeScript](./typescript_muscle_memory.md) | Fully typed CRUD app | 2–3 days |
| 04 | [FastAPI](./fastapi_muscle_memory.md) | Full REST API with all HTTP methods | 2–3 days |
| 05 | [LangGraph](./langgraph_muscle_memory.md) | Research + Write agentic pipeline | 3–4 days |
| 06 | [CrewAI](./crewai_muscle_memory.md) | Multi-agent startup analyser | 3–4 days |
| 07 | [FastMCP](./fastmcp_muscle_memory.md) | Knowledge base MCP server | 2–3 days |
| 08 | [RAG](./rag_muscle_memory.md) | Full production RAG pipeline | 3–4 days |
| 09 | [Chatbot Memory](./chatbot_memory_layers.md) | Memory-powered chatbot (all 5 layers) | 3–4 days |
| 10 | [Rust](./rust_muscle_memory.md) | CLI Task Manager (go slow here) | 5–7 days |

---

## How to Use These Sheets

### Step 1 — Pick one sheet
Start with the language or tool you use most. If you're unsure, start with Python.

### Step 2 — Set up your environment
```bash
# Create a folder for the sheet
mkdir python-practice && cd python-practice

# Turn off AI suggestions in VS Code
# Settings → Editor → Suggest → uncheck "Editor: Suggest On Trigger Characters"
# Or press Ctrl+Shift+P → "Disable GitHub Copilot"
```

### Step 3 — Open the .md file
Read the exercise. Close the .md file (or scroll past it). Open a blank `.py` / `.ts` / `.cpp` file. Type the code from memory.

### Step 4 — Run it
```bash
# Python
python filename.py

# TypeScript
npx ts-node filename.ts

# C++
g++ -std=c++17 -Wall filename.cpp -o out && ./out

# FastAPI
uvicorn filename:app --reload

# LangGraph / CrewAI / FastMCP / RAG / Memory
python filename.py
```

### Step 5 — Hit the final challenge
Every sheet ends with one final exercise — build the L10 project from a completely blank file with no peeking. When it runs cleanly, you're done with that sheet.

---

## Recommended Order

### If you're learning AI/LLM Engineering:
```
Python → FastAPI → FastMCP → RAG → LangGraph → CrewAI → Chatbot Memory
```

### If you're learning full-stack development:
```
Python → TypeScript → FastAPI → (then pick AI tools)
```

### If you want systems programming:
```
Python → C++ → Rust (go slow on Rust — it's different)
```

---

## What Each Sheet Covers

### 🐍 Python
Variables → Functions → OOP → Decorators → Generators → File I/O → JSON → CRUD App

### ⚙️ C++
Types → Pointers → Arrays → OOP → Templates → STL → Exception Handling → CRUD App

### 🔷 TypeScript
Types → Interfaces → Generics → Utility Types → Classes → Async/Await → CRUD App

### ⚡ FastAPI
Routes → Pydantic → CRUD → Status Codes → Dependencies → Middleware → Full REST API

### 🔗 LangGraph
Graphs → Nodes → Edges → State → Tools → Memory → Human-in-the-Loop → Agentic Pipeline

### 🤖 CrewAI
Agents → Tasks → Tools → Process Types → Memory → Pipelines → Multi-Agent System

### 🔌 FastMCP
Tools → Resources → Prompts → Context → SSE Transport → Composition → MCP Server

### 📚 RAG
Embeddings → Vector Stores → Chunking → LCEL Chains → Retrieval Strategies → Full RAG System

### 🧠 Chatbot Memory
Short-term → Working → Semantic → Episodic → Long-term → Entity → All Layers Together

### 🦀 Rust
Ownership → Borrowing → Structs → Enums → Option/Result → Traits → Generics → CRUD App

---

## The Progress Tracker

Every sheet has a built-in tracker at the bottom. Keep it honest.

```
| Level | Topic              | Done? | Time Taken |
|-------|--------------------| ------|------------|
| L1    | Variables, Types   |  [x]  | 25 min     |
| L2    | Control Flow       |  [x]  | 20 min     |
| L3    | Functions          |  [ ]  |            |
```

---

## Why This Works

Most developers learn by reading documentation, watching tutorials, or using AI to generate code. That builds knowledge but not muscle memory.

The difference shows up when you're in an interview, building a side project at 11pm, or debugging something with no internet. At that moment, you either know it in your hands or you don't.

These sheets are designed to build the second kind of knowing.

The method is simple: **struggle slightly, fix it yourself, move on**. The struggle is not a sign that something is wrong. The struggle is the learning.

---

## Frequently Asked Questions

**Q: What if I get stuck on an exercise?**
Try for 5 minutes. Really try. If still stuck, look at the answer in the sheet, understand why, then close it, delete what you wrote, and type it again from scratch. Looking at the answer is fine. Looking and then copy-pasting is not.

**Q: Should I do all 10 levels in one sitting?**
No. L1–L3 in one session. L4–L6 in another. L7–L10 on a third day. Sleep helps memory consolidate.

**Q: Do I need to do all sheets?**
No. Pick the ones relevant to what you're building. If you're doing AI work, Python + FastAPI + LangGraph + RAG is the core four.

**Q: The Rust sheet feels way harder than the others.**
That's correct and expected. Rust's compiler rejects things other languages silently allow. Every rejection is the compiler teaching you something. Read the error message fully — Rust errors are the most helpful of any language. Just go slower.

**Q: My friend uses GitHub Copilot for everything. Should I?**
For shipping production code: use every tool available. For learning: turn it off. You can't build muscle memory if the tool types for you. Think of it like learning to drive — you need to understand what the car is doing before you let it drive itself.

---

## Contributing

Found a bug in an exercise? Have a drill that helped you a lot? PRs welcome.

When adding exercises:
- Must be runnable (include all imports)
- Must build toward the L10 final project
- No frameworks that aren't already in the sheet's stack
- No exercises longer than ~50 lines

---

## Setup by Sheet

### Python
```bash
pip install langchain langchain-openai python-dotenv
```

### C++
```bash
# Needs g++ with C++17 support
g++ --version
```

### TypeScript
```bash
npm install -g typescript ts-node
npx ts-node --version
```

### FastAPI
```bash
pip install fastapi uvicorn pydantic python-dotenv
```

### LangGraph
```bash
pip install langgraph langchain langchain-openai python-dotenv
```

### CrewAI
```bash
pip install crewai crewai-tools langchain-openai python-dotenv
```

### FastMCP
```bash
pip install fastmcp httpx python-dotenv
```

### RAG
```bash
pip install langchain langchain-openai langchain-community \
            chromadb faiss-cpu tiktoken python-dotenv
```

### Chatbot Memory
```bash
pip install langchain langchain-openai langchain-community \
            langgraph faiss-cpu python-dotenv
```

### Rust
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustc --version
```

---

## Environment Setup (for AI sheets)

Create a `.env` file in your project folder:
```
OPENAI_API_KEY=your-key-here
```

All AI-powered sheets load this automatically via `python-dotenv`.

---

*Built by developers who got tired of forgetting syntax they already learned.*
*Share it. Practice it. Ship something.*
