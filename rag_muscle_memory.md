# RAG Muscle Memory — 40+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `python filename.py`
> - Install: `pip install langchain langchain-openai langchain-community
>             chromadb faiss-cpu sentence-transformers tiktoken
>             pypdf python-dotenv`
> - Set `OPENAI_API_KEY` in `.env`
> - L1–L2 → embeddings, vector stores (20–30 min)
> - L3–L4 → document loading, chunking (25–35 min)
> - L5–L6 → retrieval, basic RAG chain (30–40 min)
> - L7–L8 → advanced retrieval, RAG with LangGraph (40–50 min)
> - L9–L10 → evaluation, full production pipeline (50–60 min)

---

## The Core Mental Model — Before You Write a Line

```
RAG = Retrieval-Augmented Generation

Problem it solves:
  LLMs have a knowledge cutoff and can't know YOUR private data.
  RAG lets the LLM answer questions using YOUR documents.

The pipeline — 2 phases:

  INDEXING (done once, offline):
    Documents → Split into chunks → Embed each chunk → Store in vector DB

  RETRIEVAL + GENERATION (done per query, online):
    User query → Embed query → Find similar chunks → Feed to LLM → Answer

The key insight:
  The LLM doesn't "know" your docs — it reads the retrieved chunks
  as context and generates an answer from them.
  Garbage retrieval = garbage answer. Retrieval quality is everything.
```

Burn this pipeline in before touching code.

---

## L1 — Embeddings (The Foundation)

### 1. What is an Embedding
```python
# 01_embeddings_intro.py
from langchain_openai import OpenAIEmbeddings
from dotenv import load_dotenv
import numpy as np

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

# An embedding = a list of floats representing meaning
text = "LangGraph is a library for building stateful AI agents."
embedding = embedder.embed_query(text)

print(f"Text: {text}")
print(f"Embedding dimensions: {len(embedding)}")
print(f"First 5 values: {embedding[:5]}")
print(f"Type: {type(embedding[0])}")
```

---

### 2. Cosine Similarity — The Core Math
```python
# 02_cosine_similarity.py
from langchain_openai import OpenAIEmbeddings
from dotenv import load_dotenv
import numpy as np

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

def cosine_similarity(a: list, b: list) -> float:
    """Measure similarity between two embeddings. 1.0 = identical, 0.0 = unrelated."""
    a = np.array(a)
    b = np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

sentences = [
    "Python is a programming language.",
    "Python is used for data science and AI.",
    "The snake python lives in tropical forests.",
    "FastAPI is a web framework for Python.",
    "I love eating pizza on weekends.",
]

query = "What is Python used for?"
query_embedding = embedder.embed_query(query)
embeddings = embedder.embed_documents(sentences)

print(f"Query: '{query}'\n")
results = []
for sent, emb in zip(sentences, embeddings):
    score = cosine_similarity(query_embedding, emb)
    results.append((score, sent))

results.sort(reverse=True)
for score, sent in results:
    print(f"  {score:.4f} | {sent}")
```

---

### 3. Embedding Multiple Documents
```python
# 03_batch_embeddings.py
from langchain_openai import OpenAIEmbeddings
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

# Embed a batch — more efficient than one by one
documents = [
    "FastAPI is a modern Python web framework.",
    "LangChain provides tools for building LLM applications.",
    "Neo4j is a graph database used for knowledge graphs.",
    "Vector databases store embeddings for similarity search.",
    "RAG combines retrieval with language model generation.",
]

embeddings = embedder.embed_documents(documents)

print(f"Embedded {len(documents)} documents")
print(f"Each embedding has {len(embeddings[0])} dimensions")

# Show similarity between first doc and all others
from numpy import dot
from numpy.linalg import norm

def sim(a, b):
    return dot(a, b) / (norm(a) * norm(b))

base = embeddings[0]
print(f"\nSimilarity to '{documents[0]}':")
for i, (doc, emb) in enumerate(zip(documents[1:], embeddings[1:]), 1):
    print(f"  {sim(base, emb):.4f} | {doc}")
```

---

## L2 — Vector Stores

### 4. FAISS — In-Memory Vector Store
```python
# 04_faiss_basic.py
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

# Create documents — each has content and metadata
docs = [
    Document(page_content="FastAPI uses Python type hints for automatic validation.",
             metadata={"source": "fastapi_docs", "topic": "web"}),
    Document(page_content="LangGraph builds stateful agent workflows using nodes and edges.",
             metadata={"source": "langgraph_docs", "topic": "ai"}),
    Document(page_content="Neo4j is a native graph database with Cypher query language.",
             metadata={"source": "neo4j_docs", "topic": "database"}),
    Document(page_content="RAG retrieves relevant documents and feeds them to an LLM.",
             metadata={"source": "rag_guide", "topic": "ai"}),
    Document(page_content="Vector databases index embeddings for fast similarity search.",
             metadata={"source": "vectordb_guide", "topic": "database"}),
    Document(page_content="CrewAI orchestrates multiple AI agents with roles and tasks.",
             metadata={"source": "crewai_docs", "topic": "ai"}),
]

# Build FAISS index from documents
vectorstore = FAISS.from_documents(docs, embedder)

# Similarity search — returns top k most similar documents
query = "How do AI agents work?"
results = vectorstore.similarity_search(query, k=3)

print(f"Query: '{query}'\n")
for i, doc in enumerate(results, 1):
    print(f"Result {i}: [{doc.metadata['topic']}] {doc.page_content}")

# Search with scores — lower distance = more similar
print("\nWith similarity scores:")
results_with_scores = vectorstore.similarity_search_with_score(query, k=3)
for doc, score in results_with_scores:
    print(f"  Score: {score:.4f} | {doc.page_content[:60]}...")
```

---

### 5. Chroma — Persistent Vector Store
```python
# 05_chroma_persistent.py
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_core.documents import Document
from dotenv import load_dotenv
import shutil
import os

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
PERSIST_DIR = "./chroma_db"

# Clean up from previous runs
if os.path.exists(PERSIST_DIR):
    shutil.rmtree(PERSIST_DIR)

docs = [
    Document(page_content="Python decorators wrap functions to add behaviour.",
             metadata={"category": "python", "difficulty": "intermediate"}),
    Document(page_content="Async/await in Python enables non-blocking I/O operations.",
             metadata={"category": "python", "difficulty": "intermediate"}),
    Document(page_content="Pydantic validates data using Python type annotations.",
             metadata={"category": "python", "difficulty": "beginner"}),
    Document(page_content="FastAPI automatically generates OpenAPI documentation.",
             metadata={"category": "fastapi", "difficulty": "beginner"}),
    Document(page_content="FastAPI dependency injection manages shared resources.",
             metadata={"category": "fastapi", "difficulty": "advanced"}),
]

# Create and persist
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embedder,
    persist_directory=PERSIST_DIR
)
print(f"Created Chroma store with {vectorstore._collection.count()} documents")

# Load from disk (simulating restart)
vectorstore2 = Chroma(
    persist_directory=PERSIST_DIR,
    embedding_function=embedder
)
print(f"Loaded from disk: {vectorstore2._collection.count()} documents")

# Filter by metadata
results = vectorstore2.similarity_search(
    "how to handle dependencies",
    k=2,
    filter={"category": "fastapi"}
)
print("\nFiltered to FastAPI only:")
for doc in results:
    print(f"  [{doc.metadata['difficulty']}] {doc.page_content}")
```

---

### 6. Add and Delete from Vector Store
```python
# 06_vectorstore_crud.py
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

# Start with some docs
initial_docs = [
    Document(page_content="LangChain is a framework for LLM applications.",
             metadata={"id": "1", "source": "langchain"}),
    Document(page_content="LangGraph adds statefulness to LangChain agents.",
             metadata={"id": "2", "source": "langgraph"}),
]

vectorstore = FAISS.from_documents(initial_docs, embedder)
print(f"Initial docs: {vectorstore.index.ntotal}")

# Add more documents
new_docs = [
    Document(page_content="FastMCP builds MCP servers for AI tool integration.",
             metadata={"id": "3", "source": "fastmcp"}),
    Document(page_content="CrewAI orchestrates multi-agent AI workflows.",
             metadata={"id": "4", "source": "crewai"}),
]
vectorstore.add_documents(new_docs)
print(f"After adding: {vectorstore.index.ntotal}")

# Search
results = vectorstore.similarity_search("multi-agent AI systems", k=2)
for doc in results:
    print(f"  [{doc.metadata['source']}] {doc.page_content}")

# Save and reload
vectorstore.save_local("faiss_index")
loaded = FAISS.load_local("faiss_index", embedder, allow_dangerous_deserialization=True)
print(f"\nReloaded store has: {loaded.index.ntotal} docs")
```

---

## L3 — Document Loading and Chunking

### 7. Text Splitters — The Most Important Skill in RAG
```python
# 07_text_splitters.py
from langchain.text_splitter import (
    RecursiveCharacterTextSplitter,
    CharacterTextSplitter,
)
from langchain_core.documents import Document

# Your chunk size is one of the most important decisions in RAG.
# Too small → chunks lack context
# Too large → chunks are noisy and exceed context window
# Overlap → prevents information from being cut at boundaries

long_text = """
LangGraph is a library for building stateful, multi-actor applications with LLMs.
It extends LangChain's expression language with the ability to coordinate multiple
chains across multiple steps in a cyclic manner.

The core abstraction in LangGraph is a StateGraph. A StateGraph is a graph where
each node represents a computation or an action. The edges represent the flow of
data between computations. The state is shared across all nodes and is updated
as the graph runs.

LangGraph supports three types of edges: normal edges, conditional edges, and
entry points. Normal edges go from one node to another unconditionally. Conditional
edges use a function to determine which node to go to next. Entry points define
where the graph starts.

Memory in LangGraph is handled through checkpointers. A checkpointer saves the
state after each node executes. This allows you to resume a graph from any point,
implement human-in-the-loop patterns, and maintain conversation history.
"""

# RecursiveCharacterTextSplitter — preferred, tries to split on paragraphs first
r_splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    chunk_overlap=40,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=len,
)

chunks = r_splitter.split_text(long_text)
print(f"RecursiveCharacter → {len(chunks)} chunks")
for i, chunk in enumerate(chunks, 1):
    print(f"\n--- Chunk {i} ({len(chunk)} chars) ---")
    print(chunk.strip())
```

---

### 8. Chunking Documents with Metadata
```python
# 08_chunking_with_metadata.py
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.documents import Document

splitter = RecursiveCharacterTextSplitter(
    chunk_size=150,
    chunk_overlap=30,
)

# Simulate multiple documents from different sources
raw_docs = [
    Document(
        page_content="""FastAPI is a modern, fast web framework for building APIs with Python.
        It is based on standard Python type hints. FastAPI automatically generates
        interactive API documentation. It supports async operations natively.""",
        metadata={"source": "fastapi_guide.md", "page": 1}
    ),
    Document(
        page_content="""Neo4j is a graph database management system. It stores data as nodes
        and relationships rather than tables and rows. Cypher is Neo4j's query language.
        It is particularly useful for connected data and knowledge graphs.""",
        metadata={"source": "neo4j_intro.md", "page": 1}
    ),
]

chunks = splitter.split_documents(raw_docs)

print(f"Original docs: {len(raw_docs)}")
print(f"After chunking: {len(chunks)} chunks\n")

for i, chunk in enumerate(chunks, 1):
    print(f"Chunk {i}: [{chunk.metadata['source']}] {chunk.page_content[:80]}...")
```

---

### 9. Token-Based Splitting
```python
# 09_token_splitter.py
from langchain.text_splitter import RecursiveCharacterTextSplitter
import tiktoken

# LLMs count tokens, not characters — always split by tokens for accuracy
def get_token_splitter(
    model: str = "gpt-4o-mini",
    chunk_size: int = 100,
    chunk_overlap: int = 20
) -> RecursiveCharacterTextSplitter:
    """Create a splitter that counts tokens, not characters."""
    enc = tiktoken.encoding_for_model(model)

    def token_len(text: str) -> int:
        return len(enc.encode(text))

    return RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        length_function=token_len,
        separators=["\n\n", "\n", ". ", " ", ""],
    )

text = """
Retrieval-Augmented Generation (RAG) is a technique that combines the strengths
of retrieval-based and generation-based approaches. Instead of relying solely on
the parametric knowledge of a large language model, RAG retrieves relevant
documents from an external knowledge base and uses them as context for generation.

This allows the model to access up-to-date information beyond its training cutoff,
cite sources, and produce more accurate, grounded responses. The quality of the
retrieval step is critical — poor retrieval leads to poor generation, regardless
of how capable the underlying language model is.
"""

splitter = get_token_splitter(chunk_size=60, chunk_overlap=10)
chunks = splitter.split_text(text)

enc = tiktoken.encoding_for_model("gpt-4o-mini")
print(f"Total chunks: {len(chunks)}")
for i, chunk in enumerate(chunks, 1):
    tokens = len(enc.encode(chunk))
    print(f"\nChunk {i} ({tokens} tokens): {chunk.strip()[:80]}...")
```

---

### 10. Loading Real Files
```python
# 10_document_loaders.py
from langchain_community.document_loaders import (
    TextLoader,
    # PyPDFLoader,        # needs: pip install pypdf
    # WebBaseLoader,      # needs: pip install beautifulsoup4
    # CSVLoader,
    # JSONLoader,
)
from langchain.text_splitter import RecursiveCharacterTextSplitter
import os

# Create test files
os.makedirs("docs", exist_ok=True)

with open("docs/notes.txt", "w") as f:
    f.write("""LangGraph Overview
LangGraph builds on LangChain to support stateful agentic workflows.
It uses a graph-based model with nodes and edges.

Key Concepts
- StateGraph: the main graph class
- Nodes: Python functions that transform state
- Edges: connections between nodes
- Checkpointer: saves state for memory and resumption

Use Cases
LangGraph is used for building chatbots with memory, multi-step
reasoning agents, and human-in-the-loop workflows.""")

# Load text file
loader = TextLoader("docs/notes.txt")
docs = loader.load()
print(f"Loaded {len(docs)} document(s)")
print(f"Content preview: {docs[0].page_content[:100]}...")
print(f"Metadata: {docs[0].metadata}")

# Split loaded docs
splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=30)
chunks = splitter.split_documents(docs)
print(f"\nChunked into {len(chunks)} pieces:")
for i, c in enumerate(chunks, 1):
    print(f"  [{i}] {c.page_content[:70]}...")
```

---

## L4 — Building a Basic RAG Chain

### 11. Manual RAG — No Framework
```python
# 11_manual_rag.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from dotenv import load_dotenv

load_dotenv()

# ── Index ──────────────────────────────────────────────
embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

docs = [
    Document(page_content="FastAPI automatically generates /docs with Swagger UI."),
    Document(page_content="FastAPI uses Pydantic for request body validation."),
    Document(page_content="FastAPI supports async endpoints with async def."),
    Document(page_content="LangGraph nodes are plain Python functions that take and return state."),
    Document(page_content="LangGraph uses MemorySaver as a checkpointer for conversation memory."),
    Document(page_content="LangGraph conditional edges route to different nodes based on state."),
]

vectorstore = FAISS.from_documents(docs, embedder)

# ── Manual RAG pipeline ────────────────────────────────
def rag(query: str, k: int = 3) -> str:
    # Step 1: Retrieve
    retrieved = vectorstore.similarity_search(query, k=k)
    context = "\n".join(f"- {doc.page_content}" for doc in retrieved)

    # Step 2: Build prompt
    prompt = f"""Answer the question using only the context below.
If the answer is not in the context, say "I don't have that information."

Context:
{context}

Question: {query}

Answer:"""

    # Step 3: Generate
    from langchain_core.messages import HumanMessage
    response = llm.invoke([HumanMessage(content=prompt)])
    return response.content

# Test it
queries = [
    "How does FastAPI validate request data?",
    "How does LangGraph handle memory?",
    "What is the capital of France?",   # not in docs
]

for q in queries:
    print(f"\n❓ {q}")
    print(f"💬 {rag(q)}")
```

---

### 12. RAG Chain with LangChain Expression Language (LCEL)
```python
# 12_rag_lcel.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_core.prompts import ChatPromptTemplate
from dotenv import load_dotenv

load_dotenv()

# ── Setup ──────────────────────────────────────────────
embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

docs = [
    Document(page_content="CrewAI agents have a role, goal, and backstory."),
    Document(page_content="CrewAI tasks have a description, expected_output, and an assigned agent."),
    Document(page_content="CrewAI supports sequential and hierarchical process types."),
    Document(page_content="CrewAI tools are decorated with @tool and have a docstring as description."),
    Document(page_content="CrewAI memory=True enables agents to remember context across tasks."),
]

vectorstore = FAISS.from_documents(docs, embedder)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# ── Prompt ─────────────────────────────────────────────
prompt = ChatPromptTemplate.from_template("""
Answer the question using only the context provided.
If the context doesn't contain the answer, say so clearly.

Context:
{context}

Question: {question}

Answer:""")

def format_docs(docs: list) -> str:
    return "\n\n".join(doc.page_content for doc in docs)

# ── LCEL Chain ─────────────────────────────────────────
chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# Run it
questions = [
    "What does a CrewAI agent need?",
    "How do you give a CrewAI agent memory?",
    "What is the process type for a manager delegating tasks?",
]

for q in questions:
    print(f"\n❓ {q}")
    print(f"💬 {chain.invoke(q)}")
```

---

### 13. RAG with Source Citations
```python
# 13_rag_with_sources.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

docs = [
    Document(page_content="FastMCP @mcp.tool() registers a Python function as an MCP tool.",
             metadata={"source": "fastmcp_docs.md", "section": "tools"}),
    Document(page_content="FastMCP @mcp.resource() exposes data the LLM can read.",
             metadata={"source": "fastmcp_docs.md", "section": "resources"}),
    Document(page_content="FastMCP @mcp.prompt() creates reusable prompt templates.",
             metadata={"source": "fastmcp_docs.md", "section": "prompts"}),
    Document(page_content="FastMCP can run in stdio mode for Claude Desktop integration.",
             metadata={"source": "fastmcp_guide.md", "section": "deployment"}),
]

vectorstore = FAISS.from_documents(docs, embedder)
retriever = vectorstore.as_retriever(k=3)

prompt = ChatPromptTemplate.from_template("""
Answer using only the context below. Be concise.

Context:
{context}

Question: {question}

Answer:""")

def format_docs(docs):
    return "\n\n".join(
        f"[{doc.metadata['source']}] {doc.page_content}" for doc in docs
    )

def get_sources(docs):
    return list(set(doc.metadata["source"] for doc in docs))

# Parallel chain — returns both answer and sources
rag_chain_with_sources = RunnableParallel(
    answer=(
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | prompt | llm | StrOutputParser()
    ),
    sources=(retriever | get_sources),
)

result = rag_chain_with_sources.invoke("What decorators does FastMCP provide?")
print(f"Answer: {result['answer']}")
print(f"Sources: {result['sources']}")
```

---

## L5 — Advanced Retrieval

### 14. MMR — Maximum Marginal Relevance
```python
# 14_mmr_retrieval.py
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

# Many similar docs — similarity search would return duplicates
docs = [
    Document(page_content="Python is used for web development."),
    Document(page_content="Python is widely used for web applications."),
    Document(page_content="Python web frameworks include FastAPI and Flask."),
    Document(page_content="Python is the top language for machine learning."),
    Document(page_content="Python dominates AI and data science projects."),
    Document(page_content="Rust is a systems programming language focused on safety."),
    Document(page_content="Rust has no garbage collector — it uses ownership."),
]

vectorstore = FAISS.from_documents(docs, embedder)

query = "Tell me about Python"

# Standard similarity search — returns redundant results
print("=== Similarity Search (may be redundant) ===")
sim_results = vectorstore.similarity_search(query, k=3)
for doc in sim_results:
    print(f"  - {doc.page_content}")

# MMR — balances relevance with diversity
print("\n=== MMR Search (diverse results) ===")
mmr_results = vectorstore.max_marginal_relevance_search(
    query, k=3,
    fetch_k=10,   # fetch more candidates, then pick diverse k
    lambda_mult=0.5  # 0=max diversity, 1=max relevance
)
for doc in mmr_results:
    print(f"  - {doc.page_content}")
```

---

### 15. Retriever Types
```python
# 15_retriever_types.py
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

docs = [
    Document(page_content="LangGraph StateGraph manages shared state across nodes.",
             metadata={"topic": "langgraph", "level": "beginner"}),
    Document(page_content="LangGraph conditional edges route flow based on state values.",
             metadata={"topic": "langgraph", "level": "intermediate"}),
    Document(page_content="LangGraph MemorySaver persists state between invocations.",
             metadata={"topic": "langgraph", "level": "intermediate"}),
    Document(page_content="FastAPI Depends() injects shared dependencies into routes.",
             metadata={"topic": "fastapi", "level": "intermediate"}),
    Document(page_content="FastAPI Pydantic models validate request body automatically.",
             metadata={"topic": "fastapi", "level": "beginner"}),
]

vectorstore = FAISS.from_documents(docs, embedder)

# 1. Basic retriever
retriever_basic = vectorstore.as_retriever(search_kwargs={"k": 2})

# 2. MMR retriever
retriever_mmr = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": 2, "fetch_k": 5}
)

# 3. Score threshold retriever — only return docs above similarity threshold
retriever_threshold = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={"score_threshold": 0.5, "k": 3}
)

query = "How does LangGraph handle memory?"

for name, retriever in [
    ("Basic", retriever_basic),
    ("MMR", retriever_mmr),
    ("Threshold", retriever_threshold),
]:
    results = retriever.invoke(query)
    print(f"\n=== {name} retriever ({len(results)} results) ===")
    for doc in results:
        print(f"  [{doc.metadata['topic']}] {doc.page_content[:70]}...")
```

---

### 16. Multi-Query Retrieval
```python
# 16_multi_query.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain.retrievers.multi_query import MultiQueryRetriever
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

docs = [
    Document(page_content="RAG improves LLM accuracy by grounding responses in retrieved documents."),
    Document(page_content="Vector databases store embeddings and support similarity search."),
    Document(page_content="Chunking splits documents into smaller pieces for embedding."),
    Document(page_content="Embedding models convert text into numerical vectors."),
    Document(page_content="Retrieval quality determines the quality of RAG output."),
]

vectorstore = FAISS.from_documents(docs, embedder)

# MultiQueryRetriever generates multiple query variations
# and merges results — catches more relevant docs
retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(k=2),
    llm=llm
)

# Single query, but the retriever generates multiple variants internally
query = "What makes RAG systems work well?"
results = retriever.invoke(query)

print(f"Query: '{query}'")
print(f"Retrieved {len(results)} unique documents:\n")
for doc in results:
    print(f"  - {doc.page_content}")
```

---

## L6 — Conversational RAG

### 17. RAG with Chat History
```python
# 17_conversational_rag.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

docs = [
    Document(page_content="LangGraph nodes are plain Python functions."),
    Document(page_content="LangGraph state is a TypedDict shared across nodes."),
    Document(page_content="LangGraph edges define the flow between nodes."),
    Document(page_content="LangGraph supports conditional edges for branching."),
    Document(page_content="LangGraph MemorySaver persists state across turns."),
]

vectorstore = FAISS.from_documents(docs, embedder)
retriever = vectorstore.as_retriever(k=2)

# Step 1: Rephrase question given history (so retrieval makes sense)
rephrase_prompt = ChatPromptTemplate.from_messages([
    ("system", """Given a chat history and a follow-up question, rephrase the
follow-up question to be standalone. Return ONLY the rephrased question."""),
    MessagesPlaceholder("chat_history"),
    ("human", "{question}")
])

rephrase_chain = rephrase_prompt | llm | StrOutputParser()

# Step 2: Answer using retrieved context
answer_prompt = ChatPromptTemplate.from_messages([
    ("system", """Answer using only the context below. Be concise.

Context:
{context}"""),
    MessagesPlaceholder("chat_history"),
    ("human", "{question}")
])

def format_docs(docs):
    return "\n".join(doc.page_content for doc in docs)

def rag_with_history(question: str, history: list) -> str:
    # Rephrase if there's history
    if history:
        standalone = rephrase_chain.invoke({"question": question, "chat_history": history})
    else:
        standalone = question

    # Retrieve
    context = format_docs(retriever.invoke(standalone))

    # Answer
    chain = answer_prompt | llm | StrOutputParser()
    return chain.invoke({"question": question, "context": context, "chat_history": history})

# Simulate a conversation
history = []
conversation = [
    "What is LangGraph?",
    "How does it handle state?",          # ambiguous — needs rephrase
    "And what about branching the flow?", # needs rephrase too
]

for question in conversation:
    print(f"\n👤 {question}")
    answer = rag_with_history(question, history)
    print(f"🤖 {answer}")
    history.append(HumanMessage(content=question))
    history.append(AIMessage(content=answer))
```

---

## L7 — Indexing Strategies

### 18. Parent-Child Chunking
```python
# 18_parent_child.py
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.documents import Document
from dotenv import load_dotenv

load_dotenv()

# Key idea:
# Index SMALL chunks (good for retrieval — precise matches)
# Return LARGE chunks (good for context — more information)

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

parent_doc = Document(
    page_content="""FastAPI Overview
    FastAPI is a modern web framework for building APIs with Python.
    It is based on standard Python type hints and provides automatic validation.

    Key Features
    FastAPI automatically generates interactive documentation at /docs.
    It supports asynchronous request handling with async/await.
    Dependency injection is built in via the Depends() function.
    Pydantic models handle request and response validation.

    Performance
    FastAPI is one of the fastest Python frameworks available.
    It is built on Starlette for the web parts and Pydantic for data parts.
    Benchmarks show it competing with Node.js and Go in throughput.""",
    metadata={"source": "fastapi_overview.md"}
)

# Small chunks for embedding/retrieval
child_splitter = RecursiveCharacterTextSplitter(chunk_size=100, chunk_overlap=20)
child_chunks = child_splitter.split_documents([parent_doc])

# Large chunks for context
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=400, chunk_overlap=50)
parent_chunks = parent_splitter.split_documents([parent_doc])

# Index the small chunks
vectorstore = FAISS.from_documents(child_chunks, embedder)

# On retrieval: find small chunk → return its parent chunk instead
def retrieve_with_parent(query: str, k: int = 2) -> list:
    child_results = vectorstore.similarity_search(query, k=k)
    # In production you'd look up the parent by ID
    # Here we simulate by returning the larger parent chunk
    return parent_chunks[:k]

results = retrieve_with_parent("How does FastAPI handle async requests?")
print(f"Retrieved {len(results)} parent chunks")
for doc in results:
    print(f"\n[{len(doc.page_content)} chars]: {doc.page_content[:120]}...")
```

---

## L8 — RAG Evaluation

### 19. Manual Evaluation — Faithfulness and Relevance
```python
# 19_rag_evaluation.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

docs = [
    Document(page_content="RAG stands for Retrieval-Augmented Generation."),
    Document(page_content="In RAG, documents are first chunked and embedded into a vector store."),
    Document(page_content="Retrieval quality directly impacts RAG answer quality."),
    Document(page_content="Chunking strategy affects what information gets retrieved."),
]

vectorstore = FAISS.from_documents(docs, embedder)
retriever = vectorstore.as_retriever(k=2)

def run_rag(question: str) -> tuple[str, list]:
    retrieved = retriever.invoke(question)
    context = "\n".join(doc.page_content for doc in retrieved)
    prompt = f"Answer using only the context:\n{context}\n\nQuestion: {question}"
    answer = llm.invoke([HumanMessage(content=prompt)]).content
    return answer, retrieved

def evaluate_faithfulness(answer: str, context_docs: list) -> dict:
    """Check if the answer is grounded in the retrieved context."""
    context = "\n".join(doc.page_content for doc in context_docs)
    prompt = f"""Given this context:
{context}

And this answer:
{answer}

Is the answer faithful to the context? Answer with:
- verdict: yes or no
- reason: one sentence explanation
- score: 1 (faithful) or 0 (hallucinated)"""
    response = llm.invoke([HumanMessage(content=prompt)]).content
    return {"evaluation": response}

# Run and evaluate
test_cases = [
    "What does RAG stand for?",
    "How does RAG work?",
]

for question in test_cases:
    print(f"\n{'='*50}")
    print(f"Q: {question}")
    answer, retrieved = run_rag(question)
    print(f"A: {answer}")
    print(f"Retrieved: {[d.page_content[:50] for d in retrieved]}")
    eval_result = evaluate_faithfulness(answer, retrieved)
    print(f"Evaluation:\n{eval_result['evaluation']}")
```

---

## L9 — Full RAG Pipeline Patterns

### 20. Hybrid Search — Keyword + Semantic
```python
# 20_hybrid_search.py
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from dotenv import load_dotenv

load_dotenv()

# Hybrid search = semantic (vector) + keyword (BM25) combined
# Catches both meaning-based and exact-keyword matches

embedder = OpenAIEmbeddings(model="text-embedding-3-small")

docs = [
    Document(page_content="FastAPI dependency injection uses the Depends() function.",
             metadata={"id": 1}),
    Document(page_content="LangGraph StateGraph is the main graph class.",
             metadata={"id": 2}),
    Document(page_content="Neo4j Cypher uses MATCH and WHERE for queries.",
             metadata={"id": 3}),
    Document(page_content="FastMCP tools are decorated with @mcp.tool().",
             metadata={"id": 4}),
    Document(page_content="Pydantic validates data with Python type hints.",
             metadata={"id": 5}),
]

vectorstore = FAISS.from_documents(docs, embedder)

def keyword_search(query: str, docs: list) -> list:
    """Simple BM25-like keyword matching."""
    query_words = set(query.lower().split())
    scored = []
    for doc in docs:
        doc_words = set(doc.page_content.lower().split())
        overlap = len(query_words & doc_words)
        if overlap > 0:
            scored.append((overlap, doc))
    scored.sort(reverse=True)
    return [doc for _, doc in scored[:3]]

def hybrid_search(query: str, k: int = 3) -> list:
    """Combine semantic and keyword search results."""
    semantic_results = vectorstore.similarity_search(query, k=k)
    keyword_results = keyword_search(query, docs)

    # Deduplicate by content
    seen = set()
    combined = []
    for doc in semantic_results + keyword_results:
        if doc.page_content not in seen:
            seen.add(doc.page_content)
            combined.append(doc)
    return combined[:k]

query = "How does FastAPI handle Depends?"
print(f"Query: '{query}'\n")

sem = vectorstore.similarity_search(query, k=3)
print("Semantic only:")
for d in sem: print(f"  - {d.page_content}")

print("\nHybrid (semantic + keyword):")
hyb = hybrid_search(query)
for d in hyb: print(f"  - {d.page_content}")
```

---

## L10 — Full Production RAG System

### 21. Complete RAG System — Everything Together
```python
# 21_full_rag_system.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.retrievers.multi_query import MultiQueryRetriever
from dotenv import load_dotenv
from typing import Optional
import time

load_dotenv()

# ── Config ─────────────────────────────────────────────
EMBEDDING_MODEL = "text-embedding-3-small"
LLM_MODEL = "gpt-4o-mini"
CHUNK_SIZE = 300
CHUNK_OVERLAP = 50
RETRIEVAL_K = 4

# ── Models ─────────────────────────────────────────────
embedder = OpenAIEmbeddings(model=EMBEDDING_MODEL)
llm = ChatOpenAI(model=LLM_MODEL, temperature=0)

# ── Knowledge Base ─────────────────────────────────────
raw_docs = [
    Document(
        page_content="""LangGraph is a library for building stateful multi-actor applications.
        It uses nodes (Python functions), edges (flow connectors), and state (TypedDict).
        Conditional edges route to different nodes based on state values.
        MemorySaver persists state across invocations for conversation memory.
        Human-in-the-loop patterns use interrupt_before and interrupt_after.""",
        metadata={"source": "langgraph.md", "topic": "langgraph"}
    ),
    Document(
        page_content="""FastAPI is a modern Python web framework for building APIs.
        It uses Pydantic for automatic request validation via type hints.
        Routes are defined with @app.get, @app.post, @app.put, @app.delete.
        Dependency injection works via the Depends() function.
        It auto-generates Swagger docs at /docs and ReDoc at /redoc.""",
        metadata={"source": "fastapi.md", "topic": "fastapi"}
    ),
    Document(
        page_content="""FastMCP builds MCP servers for AI tool integration.
        Tools are registered with @mcp.tool() decorator.
        Resources expose data the LLM can read via @mcp.resource().
        Prompts are reusable templates via @mcp.prompt().
        Servers run in stdio mode for Claude Desktop or SSE for HTTP.""",
        metadata={"source": "fastmcp.md", "topic": "fastmcp"}
    ),
    Document(
        page_content="""RAG (Retrieval-Augmented Generation) improves LLM accuracy.
        The pipeline: chunk documents, embed chunks, store in vector DB.
        At query time: embed query, retrieve similar chunks, pass to LLM.
        Retrieval quality is the biggest factor in RAG system performance.
        Common strategies: MMR for diversity, multi-query for coverage.""",
        metadata={"source": "rag.md", "topic": "rag"}
    ),
]

# ── Indexing ───────────────────────────────────────────
splitter = RecursiveCharacterTextSplitter(
    chunk_size=CHUNK_SIZE,
    chunk_overlap=CHUNK_OVERLAP
)
chunks = splitter.split_documents(raw_docs)
vectorstore = FAISS.from_documents(chunks, embedder)

# ── Retriever with Multi-Query ─────────────────────────
base_retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"k": RETRIEVAL_K, "fetch_k": 10}
)

multi_query_retriever = MultiQueryRetriever.from_llm(
    retriever=base_retriever,
    llm=llm
)

# ── Prompt ─────────────────────────────────────────────
prompt = ChatPromptTemplate.from_template("""
You are a helpful AI assistant with expertise in Python and AI engineering.
Answer the question using ONLY the context provided below.
If the context does not contain the answer, say "I don't have that information."
Be concise and specific.

Context:
{context}

Question: {question}

Answer:""")

def format_docs(docs: list) -> str:
    return "\n\n".join(
        f"[{doc.metadata.get('source', 'unknown')}]\n{doc.page_content}"
        for doc in docs
    )

# ── Chain ──────────────────────────────────────────────
chain = RunnableParallel(
    answer=(
        {"context": multi_query_retriever | format_docs, "question": RunnablePassthrough()}
        | prompt | llm | StrOutputParser()
    ),
    sources=(
        multi_query_retriever | (lambda docs: list(set(
            doc.metadata.get("source", "unknown") for doc in docs
        )))
    ),
)

# ── Query ──────────────────────────────────────────────
def ask(question: str) -> None:
    print(f"\n{'='*55}")
    print(f"❓ {question}")
    start = time.time()
    result = chain.invoke(question)
    elapsed = time.time() - start
    print(f"💬 {result['answer']}")
    print(f"📚 Sources: {result['sources']}")
    print(f"⏱  {elapsed:.2f}s")

# ── Demo ───────────────────────────────────────────────
questions = [
    "How does LangGraph handle memory between conversations?",
    "What decorators does FastAPI use for routes?",
    "How do I build an MCP server with FastMCP?",
    "What is the RAG retrieval pipeline?",
    "What is the capital of France?",   # out of scope
]

for q in questions:
    ask(q)
```

---

## Bonus Drills — Blank File Challenges

Write these from scratch without peeking.

### B1. Build a RAG system for a topic of your choice
```
- At least 6 documents you write yourself
- FAISS vector store
- LCEL chain with formatted prompt
- Test with 3 questions (2 in-scope, 1 out-of-scope)
```

### B2. Add metadata filtering to an existing RAG system
```
Filter by topic before retrieval so only relevant category is searched
```

### B3. Build a conversational RAG with 3-turn history
```
Turn 1: "What is LangGraph?"
Turn 2: "How does its memory work?"   ← needs rephrase
Turn 3: "Can you give an example?"   ← needs rephrase
```

### B4. Compare retrieval strategies on the same query
```
Run similarity_search, MMR, and threshold retriever on same query
Print results side by side — explain which you'd use and why
```

### B5. Build a RAG evaluation loop
```
- 3 question-answer test cases with known ground truth
- Run RAG on each
- Score: 1 if answer contains key phrase, 0 if not
- Report accuracy
```

### B6. Full system from scratch in one file
```
Docs → Split → Embed → FAISS → MMR Retriever → Prompt → LLM → Answer + Sources
No peeking at exercise 21.
```

---

## RAG Cheat Sheet

```
Indexing:
  docs → splitter.split_documents(docs) → vectorstore.from_documents(chunks, embedder)

Retrieval:
  retriever = vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 4})
  docs = retriever.invoke(query)

LCEL Chain:
  chain = (
      {"context": retriever | format_docs, "question": RunnablePassthrough()}
      | prompt | llm | StrOutputParser()
  )
  answer = chain.invoke(question)

Key decisions to make in every RAG project:
  1. Chunk size      → 100-500 tokens (smaller=precise, larger=more context)
  2. Chunk overlap   → 10-20% of chunk size
  3. Embedding model → text-embedding-3-small (fast) or large (accurate)
  4. k (top-k)       → 3-5 for focused answers, 6-10 for broad questions
  5. Search type     → similarity (default), mmr (diverse), threshold (precise)
```

---

## Progress Tracker

| Level | Topic                                    | Done? | Time Taken |
|-------|------------------------------------------|-------|------------|
| L1    | Embeddings, cosine similarity            |  [ ]  |            |
| L2    | FAISS, Chroma, vector store CRUD         |  [ ]  |            |
| L3    | Text splitters, token-based chunking     |  [ ]  |            |
| L4    | Document loaders, manual RAG             |  [ ]  |            |
| L5    | LCEL chain, source citations             |  [ ]  |            |
| L6    | MMR, retriever types, multi-query        |  [ ]  |            |
| L7    | Conversational RAG, parent-child chunks  |  [ ]  |            |
| L8    | RAG evaluation                           |  [ ]  |            |
| L9    | Hybrid search                            |  [ ]  |            |
| L10   | Full production RAG pipeline             |  [ ]  |            |
| Bonus | Blank file challenges                    |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 21 (the full RAG system)
> from memory — docs, splitter, FAISS, MMR retriever, multi-query retriever,
> LCEL chain with sources, timing. Feed it 4 topics and ask 5 questions.
> When one answer correctly says "I don't have that information" for an
> out-of-scope question — your RAG pipeline is working right.
