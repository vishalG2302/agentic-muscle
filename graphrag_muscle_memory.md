# GraphRAG Muscle Memory — 40+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `python filename.py`
> - Install: `pip install langchain langchain-openai langchain-community
>             langchain-neo4j neo4j faiss-cpu python-dotenv`
> - Set `OPENAI_API_KEY` in `.env`
> - Neo4j: download Desktop from neo4j.com OR run with Docker:
>   `docker run -p 7474:7474 -p 7687:7687 -e NEO4J_AUTH=neo4j/password neo4j`
> - Neo4j browser: `http://localhost:7474`
> - L1–L2 → Neo4j basics, Cypher (30–40 min)
> - L3–L4 → building knowledge graphs from text (30–40 min)
> - L5–L6 → graph retrieval, Cypher QA (30–40 min)
> - L7–L8 → hybrid graph + vector retrieval (40–50 min)
> - L9–L10 → full GraphRAG pipeline (50–60 min)

---

## The Mental Model — Before You Write a Line

```
Regular RAG:                    GraphRAG:
                                
Documents                       Documents
    ↓ chunk                         ↓ extract entities + relationships
Chunks                          Knowledge Graph (nodes + edges)
    ↓ embed                         ↓ traverse + embed
Vector DB                       Graph DB + Vector DB
    ↓ similarity search             ↓ graph traversal + similarity
Top K chunks                    Connected subgraph
    ↓                               ↓
LLM → Answer                    LLM → Answer

Why GraphRAG beats regular RAG:

Regular RAG finds chunks SIMILAR to the query.
GraphRAG finds chunks CONNECTED to the query entities.

Example:
  Query: "How does Vishal's image memory system use Neo4j?"
  
  Regular RAG: finds chunks containing "image memory" OR "Neo4j"
  GraphRAG:    finds the entity Vishal → follows WORKS_ON edge →
               finds ImageMemorySystem → follows USES edge →
               finds Neo4j → returns ALL connected context

The difference is RELATIONSHIPS.
Regular RAG misses connections between facts.
GraphRAG follows them.

Key terms:
  Entity       → a node in the graph (person, tool, concept, project)
  Relationship → an edge between nodes (USES, WORKS_ON, PART_OF)
  Subgraph     → a connected cluster of nodes relevant to a query
  Cypher       → Neo4j's query language (like SQL for graphs)
  Community    → a group of densely connected nodes
```

---

## L1 — Neo4j and Cypher Basics

### 1. Connect to Neo4j
```python
# 01_connect_neo4j.py
from neo4j import GraphDatabase
from dotenv import load_dotenv
import os

load_dotenv()

# Neo4j connection settings
URI = os.getenv("NEO4J_URI", "bolt://localhost:7687")
USER = os.getenv("NEO4J_USER", "neo4j")
PASSWORD = os.getenv("NEO4J_PASSWORD", "password")

driver = GraphDatabase.driver(URI, auth=(USER, PASSWORD))

def test_connection():
    with driver.session() as session:
        result = session.run("RETURN 'Connected to Neo4j!' AS message")
        record = result.single()
        print(record["message"])

def clear_database():
    """Delete all nodes and relationships — use for fresh starts."""
    with driver.session() as session:
        session.run("MATCH (n) DETACH DELETE n")
        print("Database cleared.")

test_connection()
clear_database()
driver.close()
```
> Add to `.env`:
> ```
> NEO4J_URI=bolt://localhost:7687
> NEO4J_USER=neo4j
> NEO4J_PASSWORD=password
> ```

---

### 2. Create Nodes and Relationships
```python
# 02_create_nodes.py
from neo4j import GraphDatabase
from dotenv import load_dotenv
import os

load_dotenv()

driver = GraphDatabase.driver(
    os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    auth=(os.getenv("NEO4J_USER", "neo4j"), os.getenv("NEO4J_PASSWORD", "password"))
)

def run(query: str, params: dict = {}):
    with driver.session() as session:
        return list(session.run(query, params))

# Clear first
run("MATCH (n) DETACH DELETE n")

# CREATE nodes
run("""
CREATE (v:Person {name: 'Vishal', age: 22, role: 'AI Engineer'})
CREATE (l:Tool {name: 'LangGraph', type: 'Framework'})
CREATE (n:Tool {name: 'Neo4j', type: 'Database'})
CREATE (f:Tool {name: 'FastAPI', type: 'Framework'})
CREATE (p:Project {name: 'ImageMemorySystem', status: 'Building'})
""")

# CREATE relationships
run("""
MATCH (v:Person {name: 'Vishal'})
MATCH (l:Tool {name: 'LangGraph'})
MATCH (n:Tool {name: 'Neo4j'})
MATCH (f:Tool {name: 'FastAPI'})
MATCH (p:Project {name: 'ImageMemorySystem'})
CREATE (v)-[:USES]->(l)
CREATE (v)-[:USES]->(n)
CREATE (v)-[:USES]->(f)
CREATE (v)-[:WORKS_ON]->(p)
CREATE (p)-[:USES]->(l)
CREATE (p)-[:USES]->(n)
""")

# READ all nodes
print("=== All Nodes ===")
nodes = run("MATCH (n) RETURN labels(n) AS label, n.name AS name")
for record in nodes:
    print(f"  [{record['label'][0]}] {record['name']}")

# READ relationships
print("\n=== All Relationships ===")
rels = run("MATCH (a)-[r]->(b) RETURN a.name, type(r), b.name")
for r in rels:
    print(f"  {r['a.name']} --[{r['type(r)']}]--> {r['b.name']}")

driver.close()
```

---

### 3. Cypher CRUD
```python
# 03_cypher_crud.py
from neo4j import GraphDatabase
from dotenv import load_dotenv
import os

load_dotenv()

driver = GraphDatabase.driver(
    os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    auth=(os.getenv("NEO4J_USER", "neo4j"), os.getenv("NEO4J_PASSWORD", "password"))
)

def run(query: str, params: dict = {}):
    with driver.session() as session:
        return list(session.run(query, params))

run("MATCH (n) DETACH DELETE n")  # fresh start

# CREATE with MERGE (create if not exists)
run("""
MERGE (v:Person {name: 'Vishal'})
SET v.age = 22, v.city = 'Pune'
MERGE (a:Person {name: 'Alice'})
SET a.age = 30, a.city = 'Mumbai'
MERGE (l:Tool {name: 'LangGraph'})
MERGE (n:Tool {name: 'Neo4j'})
MERGE (v)-[:KNOWS]->(a)
MERGE (v)-[:USES]->(l)
MERGE (v)-[:USES]->(n)
MERGE (a)-[:USES]->(n)
""")

# READ with WHERE filter
print("=== People in Pune ===")
result = run("MATCH (p:Person) WHERE p.city = 'Pune' RETURN p.name, p.age")
for r in result:
    print(f"  {r['p.name']}, age {r['p.age']}")

# READ with pattern matching
print("\n=== Who uses Neo4j? ===")
result = run("MATCH (p:Person)-[:USES]->(t:Tool {name: 'Neo4j'}) RETURN p.name")
for r in result:
    print(f"  {r['p.name']}")

# UPDATE a node property
run("MATCH (v:Person {name: 'Vishal'}) SET v.role = 'LLM Engineer'")
result = run("MATCH (v:Person {name: 'Vishal'}) RETURN v.role")
print(f"\n=== Vishal's role: {result[0]['v.role']}")

# DELETE a relationship
run("MATCH (:Person {name: 'Vishal'})-[r:KNOWS]->(:Person {name: 'Alice'}) DELETE r")
result = run("MATCH (:Person {name: 'Vishal'})-[r:KNOWS]->() RETURN count(r) AS count")
print(f"KNOWS relationships after delete: {result[0]['count']}")

# DELETE a node (must detach first)
run("MATCH (a:Person {name: 'Alice'}) DETACH DELETE a")
result = run("MATCH (p:Person) RETURN count(p) AS count")
print(f"People left: {result[0]['count']}")

driver.close()
```

---

### 4. Cypher Patterns — The Ones You'll Use Every Day
```python
# 04_cypher_patterns.py
from neo4j import GraphDatabase
from dotenv import load_dotenv
import os

load_dotenv()

driver = GraphDatabase.driver(
    os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    auth=(os.getenv("NEO4J_USER", "neo4j"), os.getenv("NEO4J_PASSWORD", "password"))
)

def run(query: str, params: dict = {}) -> list:
    with driver.session() as session:
        return list(session.run(query, params))

run("MATCH (n) DETACH DELETE n")

# Seed a knowledge graph
run("""
CREATE (py:Language {name: 'Python', paradigm: 'Multi-paradigm'})
CREATE (rs:Language {name: 'Rust', paradigm: 'Systems'})
CREATE (ts:Language {name: 'TypeScript', paradigm: 'OOP'})
CREATE (fa:Tool {name: 'FastAPI', category: 'Web Framework'})
CREATE (lg:Tool {name: 'LangGraph', category: 'AI Framework'})
CREATE (n4:Tool {name: 'Neo4j', category: 'Database'})
CREATE (lc:Tool {name: 'LangChain', category: 'AI Framework'})
CREATE (rag:Concept {name: 'RAG', description: 'Retrieval Augmented Generation'})
CREATE (graph:Concept {name: 'GraphRAG', description: 'Graph-based RAG'})
CREATE (fa)-[:BUILT_WITH]->(py)
CREATE (lg)-[:BUILT_WITH]->(py)
CREATE (lc)-[:BUILT_WITH]->(py)
CREATE (graph)-[:EXTENDS]->(rag)
CREATE (graph)-[:USES]->(n4)
CREATE (lg)-[:INTEGRATES_WITH]->(n4)
CREATE (lc)-[:INTEGRATES_WITH]->(n4)
""")

# Pattern 1: Path traversal — find 2-hop connections
print("=== Tools that integrate with Neo4j ===")
result = run("MATCH (t:Tool)-[:INTEGRATES_WITH]->(n:Tool {name: 'Neo4j'}) RETURN t.name")
for r in result: print(f"  {r['t.name']}")

# Pattern 2: Variable length path — find all connected nodes
print("\n=== Everything connected to RAG (up to 3 hops) ===")
result = run("MATCH (rag:Concept {name: 'RAG'})-[*1..3]-(connected) RETURN DISTINCT connected.name")
for r in result:
    if r['connected.name']: print(f"  {r['connected.name']}")

# Pattern 3: Optional match — don't fail if relationship missing
print("\n=== All tools with their language (if known) ===")
result = run("""
MATCH (t:Tool)
OPTIONAL MATCH (t)-[:BUILT_WITH]->(l:Language)
RETURN t.name AS tool, l.name AS language
ORDER BY tool
""")
for r in result:
    lang = r['language'] or 'Unknown'
    print(f"  {r['tool']}: {lang}")

# Pattern 4: Aggregation
print("\n=== Count relationships per node ===")
result = run("""
MATCH (n)-[r]->()
RETURN n.name AS name, count(r) AS out_degree
ORDER BY out_degree DESC
""")
for r in result:
    if r['name']: print(f"  {r['name']}: {r['out_degree']} outgoing")

# Pattern 5: Shortest path
print("\n=== Shortest path: Python → GraphRAG ===")
result = run("""
MATCH path = shortestPath(
    (py:Language {name: 'Python'})-[*]-(g:Concept {name: 'GraphRAG'})
)
RETURN [node in nodes(path) | node.name] AS path_names
""")
for r in result:
    print(f"  {' → '.join(filter(None, r['path_names']))}")

driver.close()
```

---

## L2 — LangChain + Neo4j Integration

### 5. Neo4jGraph — LangChain's Neo4j Wrapper
```python
# 05_neo4j_graph_wrapper.py
from langchain_neo4j import Neo4jGraph
from dotenv import load_dotenv
import os

load_dotenv()

# LangChain's Neo4j wrapper — handles connection and schema
graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)

# Clear and seed
graph.query("MATCH (n) DETACH DELETE n")

graph.query("""
CREATE (v:Person {name: 'Vishal', role: 'AI Engineer', city: 'Pune'})
CREATE (a:Person {name: 'Alice', role: 'Data Scientist', city: 'Mumbai'})
CREATE (lg:Tool {name: 'LangGraph', type: 'Framework'})
CREATE (n4:Tool {name: 'Neo4j', type: 'Database'})
CREATE (p:Project {name: 'ImageMemorySystem', status: 'In Progress'})
CREATE (v)-[:WORKS_ON]->(p)
CREATE (v)-[:USES]->(lg)
CREATE (v)-[:USES]->(n4)
CREATE (p)-[:USES]->(lg)
CREATE (p)-[:USES]->(n4)
CREATE (v)-[:COLLABORATES_WITH]->(a)
""")

# Refresh schema so LangChain knows the graph structure
graph.refresh_schema()

# Schema shows what nodes, relationships, and properties exist
print("=== Graph Schema ===")
print(graph.schema)
print()

# Run Cypher queries through the wrapper
result = graph.query("MATCH (p:Person)-[:USES]->(t:Tool) RETURN p.name, t.name")
print("=== Person → Tool relationships ===")
for r in result:
    print(f"  {r['p.name']} USES {r['t.name']}")
```

---

### 6. GraphCypherQAChain — Natural Language to Cypher
```python
# 06_cypher_qa_chain.py
from langchain_neo4j import Neo4jGraph, GraphCypherQAChain
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv
import os

load_dotenv()

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)

graph.query("MATCH (n) DETACH DELETE n")
graph.query("""
CREATE (v:Person {name: 'Vishal', age: 22, city: 'Pune', role: 'AI Engineer'})
CREATE (a:Person {name: 'Alice', age: 30, city: 'Mumbai', role: 'Data Scientist'})
CREATE (b:Person {name: 'Bob', age: 25, city: 'Pune', role: 'Backend Developer'})
CREATE (lg:Tool {name: 'LangGraph', category: 'AI'})
CREATE (n4:Tool {name: 'Neo4j', category: 'Database'})
CREATE (fa:Tool {name: 'FastAPI', category: 'Web'})
CREATE (p1:Project {name: 'ImageMemorySystem', status: 'Active'})
CREATE (p2:Project {name: 'ProcureAI', status: 'Planning'})
CREATE (v)-[:USES]->(lg)
CREATE (v)-[:USES]->(n4)
CREATE (v)-[:USES]->(fa)
CREATE (v)-[:WORKS_ON]->(p1)
CREATE (a)-[:USES]->(n4)
CREATE (b)-[:USES]->(fa)
CREATE (b)-[:WORKS_ON]->(p2)
CREATE (v)-[:MENTORS]->(b)
""")
graph.refresh_schema()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# GraphCypherQAChain converts natural language → Cypher → runs it → answers
chain = GraphCypherQAChain.from_llm(
    llm=llm,
    graph=graph,
    verbose=True,        # shows generated Cypher
    allow_dangerous_requests=True
)

questions = [
    "Who works on the ImageMemorySystem project?",
    "What tools does Vishal use?",
    "Which people live in Pune?",
    "Who does Vishal mentor?",
    "What projects exist and what is their status?",
]

for q in questions:
    print(f"\n{'='*50}")
    print(f"❓ {q}")
    result = chain.invoke({"query": q})
    print(f"💬 {result['result']}")
```

---

## L3 — Extracting Knowledge Graphs from Text

### 7. LLMGraphTransformer — Text to Graph
```python
# 07_llm_graph_transformer.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import ChatOpenAI
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_core.documents import Document
from dotenv import load_dotenv
import os

load_dotenv()

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)
graph.query("MATCH (n) DETACH DELETE n")

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# LLMGraphTransformer reads text and extracts entities + relationships
transformer = LLMGraphTransformer(llm=llm)

# Raw text — the transformer will extract a graph from this
texts = [
    """Vishal is a 22-year-old AI engineer from Pune who is building
    an image memory system. The system uses LangGraph for the agent pipeline
    and Neo4j for storing the knowledge graph. Vishal also uses FastAPI
    to expose the system as a REST API.""",

    """LangGraph is built on top of LangChain. It adds stateful workflows
    and supports cycles and branching. LangGraph integrates with Neo4j
    through the Neo4j LangChain integration package.""",

    """The image memory system ingests screenshots. PaddleOCR extracts text
    from the images. YOLOv8 detects objects. CLIP generates visual embeddings.
    All extracted entities are stored in Neo4j as a knowledge graph.""",
]

documents = [Document(page_content=t) for t in texts]

# Convert text → graph documents
print("Extracting knowledge graph from text...")
graph_docs = transformer.convert_to_graph_documents(documents)

# Show what was extracted
for i, gd in enumerate(graph_docs, 1):
    print(f"\n=== Document {i} ===")
    print(f"  Nodes ({len(gd.nodes)}):")
    for node in gd.nodes:
        print(f"    [{node.type}] {node.id}")
    print(f"  Relationships ({len(gd.relationships)}):")
    for rel in gd.relationships:
        print(f"    {rel.source.id} --[{rel.type}]--> {rel.target.id}")

# Store in Neo4j
graph.add_graph_documents(graph_docs, baseEntityLabel=True, include_source=True)
graph.refresh_schema()

print("\n=== Schema after extraction ===")
print(graph.schema)
```
> Install: `pip install langchain-experimental`

---

### 8. Constrained Entity Extraction
```python
# 08_constrained_extraction.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import ChatOpenAI
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_core.documents import Document
from dotenv import load_dotenv
import os

load_dotenv()

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)
graph.query("MATCH (n) DETACH DELETE n")

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Constrain what node types and relationship types get extracted
# This keeps your graph clean and consistent
transformer = LLMGraphTransformer(
    llm=llm,
    allowed_nodes=["Person", "Tool", "Project", "Concept", "Organization"],
    allowed_relationships=["USES", "WORKS_ON", "BUILT_WITH", "PART_OF", "KNOWS", "EXTENDS"],
    node_properties=["name", "description", "category"],
    relationship_properties=["since", "role"],
)

docs = [
    Document(page_content="""
    Vishal is an AI engineer who works on the ImageMemorySystem project.
    He uses LangGraph, Neo4j, and FastAPI in his work. LangGraph is part
    of the LangChain ecosystem. The project is built with Python.
    Vishal is a student at IIT Madras studying Data Science.
    """),
    Document(page_content="""
    GraphRAG extends standard RAG by using knowledge graphs.
    It uses Neo4j as the graph database. GraphRAG is part of the
    broader field of Retrieval Augmented Generation. Microsoft Research
    introduced GraphRAG as a concept in 2024.
    """),
]

graph_docs = transformer.convert_to_graph_documents(docs)

for gd in graph_docs:
    print(f"Nodes: {[n.id for n in gd.nodes]}")
    print(f"Edges: {[(r.source.id, r.type, r.target.id) for r in gd.relationships]}")
    print()

graph.add_graph_documents(graph_docs, baseEntityLabel=True, include_source=True)

# Verify what's in the graph
result = graph.query("MATCH (n) RETURN labels(n) AS label, n.name AS name LIMIT 20")
print("=== Nodes in graph ===")
for r in result:
    print(f"  [{r['label']}] {r['name']}")
```

---

## L4 — Graph-Based Retrieval

### 9. Entity-Based Graph Retrieval
```python
# 09_entity_retrieval.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import os
import json

load_dotenv()

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)

graph.query("MATCH (n) DETACH DELETE n")
graph.query("""
MERGE (v:Person {name: 'Vishal', role: 'AI Engineer', city: 'Pune'})
MERGE (lg:Tool {name: 'LangGraph', category: 'AI Framework', language: 'Python'})
MERGE (n4:Tool {name: 'Neo4j', category: 'Graph Database'})
MERGE (fa:Tool {name: 'FastAPI', category: 'Web Framework'})
MERGE (rag:Concept {name: 'RAG', description: 'Retrieval Augmented Generation'})
MERGE (grag:Concept {name: 'GraphRAG', description: 'Graph-based RAG using knowledge graphs'})
MERGE (p:Project {name: 'ImageMemorySystem', status: 'Active', goal: 'Store and retrieve image memories'})
MERGE (v)-[:USES]->(lg)
MERGE (v)-[:USES]->(n4)
MERGE (v)-[:USES]->(fa)
MERGE (v)-[:WORKS_ON]->(p)
MERGE (p)-[:USES]->(lg)
MERGE (p)-[:USES]->(n4)
MERGE (grag)-[:EXTENDS]->(rag)
MERGE (grag)-[:USES]->(n4)
""")

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def extract_entities(question: str) -> list[str]:
    """Extract entity names from a question."""
    prompt = f"""Extract named entities from this question.
Return ONLY a JSON array of entity names (people, tools, projects, concepts).
Return [] if none found.

Question: "{question}"
Entities:"""
    response = llm.invoke([HumanMessage(content=prompt)])
    try:
        return json.loads(response.content.strip())
    except:
        return []

def retrieve_from_graph(entities: list[str], hops: int = 2) -> str:
    """Fetch graph context around the given entities."""
    if not entities:
        return "No entities found."

    results = []
    for entity in entities:
        # Find the entity and its neighbourhood
        records = graph.query("""
MATCH (n)
WHERE toLower(n.name) CONTAINS toLower($entity)
OPTIONAL MATCH (n)-[r1]-(neighbour)
OPTIONAL MATCH (neighbour)-[r2]-(neighbour2)
RETURN n.name AS node,
       collect(DISTINCT {rel: type(r1), neighbour: neighbour.name}) AS connections
LIMIT 5
""", params={"entity": entity})

        for r in records:
            if r["node"]:
                results.append(f"Entity: {r['node']}")
                for conn in r["connections"]:
                    if conn["neighbour"]:
                        results.append(f"  --[{conn['rel']}]--> {conn['neighbour']}")

    return "\n".join(results) if results else "No graph context found."

def graph_rag(question: str) -> str:
    """Graph-based retrieval + LLM generation."""
    # Step 1: Extract entities from question
    entities = extract_entities(question)
    print(f"  Entities: {entities}")

    # Step 2: Retrieve graph context
    context = retrieve_from_graph(entities)
    print(f"  Context:\n{context}")

    # Step 3: Generate answer
    prompt = f"""Answer the question using the graph context below.
Be specific and use the relationships shown.

Graph Context:
{context}

Question: {question}
Answer:"""
    response = llm.invoke([HumanMessage(content=prompt)])
    return response.content

print("=== Graph RAG Demo ===\n")
questions = [
    "What does Vishal use for his project?",
    "What is GraphRAG and how does it relate to RAG?",
    "What tools are used in the ImageMemorySystem?",
]

for q in questions:
    print(f"❓ {q}")
    print(f"💬 {graph_rag(q)}\n")
```

---

## L5 — Hybrid: Graph + Vector Retrieval

### 10. Vector Store on Graph Nodes
```python
# 10_vector_on_graph.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import os

load_dotenv()

# The key insight:
# Graph gives you CONNECTIONS (who knows who, what uses what)
# Vector gives you SIMILARITY (semantically close content)
# Combine both = most relevant + most connected context

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)
graph.query("MATCH (n) DETACH DELETE n")

# Seed graph
graph.query("""
MERGE (v:Person {name: 'Vishal', desc: 'AI engineer building image memory systems in Pune'})
MERGE (lg:Tool {name: 'LangGraph', desc: 'Framework for stateful agentic workflows using graphs'})
MERGE (n4:Tool {name: 'Neo4j', desc: 'Native graph database with Cypher query language'})
MERGE (rag:Concept {name: 'RAG', desc: 'Combines retrieval with LLM generation for grounded answers'})
MERGE (grag:Concept {name: 'GraphRAG', desc: 'Extends RAG using knowledge graph traversal'})
MERGE (p:Project {name: 'ImageMemorySystem', desc: 'Stores and retrieves image memories using graph and vectors'})
MERGE (v)-[:USES]->(lg)
MERGE (v)-[:USES]->(n4)
MERGE (v)-[:WORKS_ON]->(p)
MERGE (grag)-[:EXTENDS]->(rag)
MERGE (grag)-[:USES]->(n4)
MERGE (p)-[:USES]->(n4)
""")

# Build vector store from graph node descriptions
node_records = graph.query("MATCH (n) WHERE n.desc IS NOT NULL RETURN n.name AS name, n.desc AS desc, labels(n)[0] AS label")

docs = [
    Document(
        page_content=r["desc"],
        metadata={"name": r["name"], "label": r["label"]}
    )
    for r in node_records
]

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(docs, embedder)
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def hybrid_retrieve(query: str, k: int = 3) -> str:
    # Vector retrieval — semantically similar descriptions
    vector_results = vectorstore.similarity_search(query, k=k)
    vector_names = [d.metadata["name"] for d in vector_results]

    # Graph retrieval — neighbours of found entities
    graph_context = []
    for name in vector_names:
        records = graph.query("""
MATCH (n {name: $name})-[r]-(neighbour)
RETURN n.name AS node, type(r) AS rel, neighbour.name AS neighbour
""", params={"name": name})
        for r in records:
            graph_context.append(f"{r['node']} --[{r['rel']}]--> {r['neighbour']}")

    # Combine both
    vector_context = "\n".join(f"- {d.page_content}" for d in vector_results)
    graph_ctx = "\n".join(graph_context) if graph_context else "No graph connections found."

    return f"Vector matches:\n{vector_context}\n\nGraph connections:\n{graph_ctx}"

def answer(query: str) -> str:
    context = hybrid_retrieve(query)
    prompt = f"""Answer using this context from both vector search and graph traversal:

{context}

Question: {query}
Answer:"""
    return llm.invoke([HumanMessage(content=prompt)]).content

questions = [
    "Tell me about graph-based retrieval systems",
    "What is Vishal building and what tools does he use?",
]

for q in questions:
    print(f"❓ {q}")
    print(f"💬 {answer(q)}\n")
```

---

## L6 — Neo4j Vector Index (Native)

### 11. Neo4j Native Vector Search
```python
# 11_neo4j_vector_index.py
from langchain_neo4j import Neo4jGraph, Neo4jVector
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.documents import Document
from dotenv import load_dotenv
import os

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Neo4jVector stores embeddings INSIDE Neo4j nodes
# No separate FAISS index needed — graph + vectors in one place
vectorstore = Neo4jVector.from_documents(
    documents=[
        Document(page_content="LangGraph builds stateful workflows with nodes and edges.",
                 metadata={"source": "langgraph_docs", "topic": "agents"}),
        Document(page_content="Neo4j stores data as nodes and relationships for graph traversal.",
                 metadata={"source": "neo4j_docs", "topic": "database"}),
        Document(page_content="GraphRAG uses graph traversal to find connected context.",
                 metadata={"source": "graphrag_paper", "topic": "rag"}),
        Document(page_content="FastAPI provides automatic OpenAPI docs at /docs endpoint.",
                 metadata={"source": "fastapi_docs", "topic": "web"}),
        Document(page_content="RAG grounds LLM responses in retrieved documents.",
                 metadata={"source": "rag_guide", "topic": "rag"}),
    ],
    embedding=embedder,
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
    index_name="doc_embeddings",
    node_label="KnowledgeChunk",
    text_node_property="content",
    embedding_node_property="embedding",
)

retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# Standard similarity search — all inside Neo4j
results = vectorstore.similarity_search("How does graph-based retrieval work?", k=3)
print("=== Vector Search in Neo4j ===")
for doc in results:
    print(f"  [{doc.metadata['topic']}] {doc.page_content[:80]}")
```

---

## L7 — Full GraphRAG Chain

### 12. GraphRAG Chain — Entity Extract → Graph Retrieve → Vector Retrieve → Answer
```python
# 12_graphrag_chain.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableLambda
from dotenv import load_dotenv
import os
import json

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)

# Seed knowledge graph
graph.query("MATCH (n) DETACH DELETE n")
graph.query("""
MERGE (v:Person {name: 'Vishal', role: 'AI Engineer'})
MERGE (lg:Tool {name: 'LangGraph', desc: 'Stateful agent workflow framework'})
MERGE (n4:Tool {name: 'Neo4j', desc: 'Graph database with Cypher'})
MERGE (fa:Tool {name: 'FastAPI', desc: 'Python web framework'})
MERGE (rag:Concept {name: 'RAG', desc: 'Retrieval Augmented Generation'})
MERGE (grag:Concept {name: 'GraphRAG', desc: 'RAG using knowledge graph traversal'})
MERGE (im:Project {name: 'ImageMemorySystem', desc: 'AI agent memory using images'})
MERGE (v)-[:USES]->(lg)
MERGE (v)-[:USES]->(n4)
MERGE (v)-[:USES]->(fa)
MERGE (v)-[:WORKS_ON]->(im)
MERGE (im)-[:USES]->(n4)
MERGE (im)-[:USES]->(lg)
MERGE (grag)-[:EXTENDS]->(rag)
MERGE (grag)-[:USES]->(n4)
""")

# Build vector store from same data
docs = [
    Document(page_content="LangGraph builds stateful agentic workflows using nodes and conditional edges.",
             metadata={"entity": "LangGraph"}),
    Document(page_content="Neo4j is a native graph database that stores data as nodes and relationships.",
             metadata={"entity": "Neo4j"}),
    Document(page_content="GraphRAG extends RAG by traversing a knowledge graph to find connected context.",
             metadata={"entity": "GraphRAG"}),
    Document(page_content="The ImageMemorySystem stores AI agent memories extracted from screenshots.",
             metadata={"entity": "ImageMemorySystem"}),
    Document(page_content="FastAPI is a Python web framework with automatic validation and docs.",
             metadata={"entity": "FastAPI"}),
]
vectorstore = FAISS.from_documents(docs, embedder)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# ── Step 1: Extract entities from question ─────────────
def extract_entities(question: str) -> list[str]:
    prompt = f"""Extract named entities from: "{question}"
Return ONLY a JSON array. Return [] if none."""
    try:
        r = llm.invoke([{"role": "user", "content": prompt}])
        return json.loads(r.content.strip())
    except:
        return []

# ── Step 2: Graph traversal ────────────────────────────
def graph_retrieve(entities: list[str]) -> str:
    if not entities:
        return ""
    parts = []
    for entity in entities:
        records = graph.query("""
MATCH (n) WHERE toLower(n.name) CONTAINS toLower($name)
OPTIONAL MATCH (n)-[r]-(m)
RETURN n.name AS node, collect(DISTINCT type(r) + ' ' + coalesce(m.name,'')) AS rels
LIMIT 3
""", params={"name": entity})
        for r in records:
            if r["node"]:
                parts.append(f"{r['node']}: {', '.join(filter(None, r['rels']))}")
    return "Graph context:\n" + "\n".join(parts) if parts else ""

# ── Step 3: Vector retrieval ───────────────────────────
def vector_retrieve(question: str) -> str:
    docs = retriever.invoke(question)
    return "Vector context:\n" + "\n".join(d.page_content for d in docs)

# ── Step 4: Answer ─────────────────────────────────────
answer_prompt = ChatPromptTemplate.from_template("""
Answer the question using the graph and vector context below.
Use relationships from the graph context to show connections.

{graph_context}

{vector_context}

Question: {question}
Answer:""")

answer_chain = answer_prompt | llm | StrOutputParser()

def graphrag(question: str) -> str:
    entities = extract_entities(question)
    graph_ctx = graph_retrieve(entities)
    vector_ctx = vector_retrieve(question)

    return answer_chain.invoke({
        "graph_context": graph_ctx,
        "vector_context": vector_ctx,
        "question": question,
    })

print("=== GraphRAG Chain Demo ===\n")
questions = [
    "What is Vishal building and what tools does it use?",
    "How does GraphRAG relate to regular RAG?",
    "What are the components of the ImageMemorySystem?",
]

for q in questions:
    print(f"❓ {q}")
    print(f"💬 {graphrag(q)}\n")
```

---

## L8 — Community Summaries

### 13. Community Detection + Summaries
```python
# 13_community_summaries.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import os

load_dotenv()

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

graph.query("MATCH (n) DETACH DELETE n")
graph.query("""
MERGE (v:Person {name: 'Vishal', community: 'AI_Engineering'})
MERGE (lg:Tool {name: 'LangGraph', community: 'AI_Engineering'})
MERGE (n4:Tool {name: 'Neo4j', community: 'Data_Infrastructure'})
MERGE (fa:Tool {name: 'FastAPI', community: 'Web_Development'})
MERGE (py:Language {name: 'Python', community: 'AI_Engineering'})
MERGE (rag:Concept {name: 'RAG', community: 'AI_Engineering'})
MERGE (grag:Concept {name: 'GraphRAG', community: 'AI_Engineering'})
MERGE (cypher:Language {name: 'Cypher', community: 'Data_Infrastructure'})
MERGE (v)-[:USES]->(lg)
MERGE (v)-[:USES]->(n4)
MERGE (v)-[:USES]->(fa)
MERGE (lg)-[:BUILT_WITH]->(py)
MERGE (n4)-[:USES]->(cypher)
MERGE (grag)-[:EXTENDS]->(rag)
MERGE (grag)-[:USES]->(n4)
""")

def get_community_nodes(community: str) -> list[dict]:
    """Get all nodes in a community and their relationships."""
    return graph.query("""
MATCH (n {community: $community})
OPTIONAL MATCH (n)-[r]-(m)
RETURN n.name AS name, labels(n)[0] AS type,
       collect(DISTINCT type(r) + ': ' + coalesce(m.name, '')) AS connections
""", params={"community": community})

def summarise_community(community: str) -> str:
    """Use LLM to summarise a community of nodes."""
    nodes = get_community_nodes(community)
    if not nodes:
        return f"Community {community} is empty."

    description = f"Community: {community}\n"
    for n in nodes:
        connections = [c for c in n["connections"] if c.strip(": ")]
        desc = f"  [{n['type']}] {n['name']}"
        if connections:
            desc += f" — {', '.join(connections[:3])}"
        description += desc + "\n"

    prompt = f"""Summarise this community of related technologies in 2-3 sentences.
Focus on how they relate to each other and what they collectively enable.

{description}

Summary:"""
    return llm.invoke([HumanMessage(content=prompt)]).content

# Get all communities
communities = graph.query("MATCH (n) WHERE n.community IS NOT NULL RETURN DISTINCT n.community AS c")
community_names = [r["c"] for r in communities]

print("=== Community Summaries ===\n")
community_summaries = {}
for community in community_names:
    summary = summarise_community(community)
    community_summaries[community] = summary
    print(f"📦 {community}")
    print(f"   {summary}\n")

# Use summaries for global questions
def answer_global(question: str) -> str:
    """Answer using community summaries — good for broad questions."""
    context = "\n\n".join(
        f"[{name}]\n{summary}"
        for name, summary in community_summaries.items()
    )
    prompt = f"""Using these community summaries, answer the question:

{context}

Question: {question}
Answer:"""
    return llm.invoke([HumanMessage(content=prompt)]).content

print("=== Global Question Answering ===\n")
print(f"❓ What are the main technology areas in this knowledge graph?")
print(f"💬 {answer_global('What are the main technology areas in this knowledge graph?')}")
```

---

## L9 — GraphRAG Evaluation

### 14. Evaluate Graph vs Vector vs GraphRAG
```python
# 14_compare_retrieval.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import os, json, time

load_dotenv()

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)
embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

graph.query("MATCH (n) DETACH DELETE n")
graph.query("""
MERGE (v:Person {name: 'Vishal', role: 'AI Engineer', city: 'Pune'})
MERGE (a:Person {name: 'Alice', role: 'Data Scientist', city: 'Mumbai'})
MERGE (lg:Tool {name: 'LangGraph', category: 'AI'})
MERGE (n4:Tool {name: 'Neo4j', category: 'Database'})
MERGE (im:Project {name: 'ImageMemorySystem', status: 'Active'})
MERGE (v)-[:WORKS_ON]->(im)
MERGE (v)-[:USES]->(lg)
MERGE (v)-[:USES]->(n4)
MERGE (v)-[:COLLABORATES_WITH]->(a)
MERGE (im)-[:USES]->(n4)
MERGE (im)-[:USES]->(lg)
""")

docs = [
    Document(page_content="Vishal is an AI Engineer from Pune building an image memory system."),
    Document(page_content="The ImageMemorySystem uses LangGraph for the agent pipeline."),
    Document(page_content="Neo4j is used as the knowledge graph store in the ImageMemorySystem."),
    Document(page_content="Alice is a Data Scientist from Mumbai collaborating on the project."),
]
vectorstore = FAISS.from_documents(docs, embedder)

def vector_rag(q: str) -> str:
    docs = vectorstore.similarity_search(q, k=3)
    ctx = "\n".join(d.page_content for d in docs)
    return llm.invoke([HumanMessage(content=f"Answer using context:\n{ctx}\n\nQ: {q}")]).content

def graph_only(q: str) -> str:
    entities_resp = llm.invoke([HumanMessage(content=f"Extract entities from: '{q}'. Return JSON array.")])
    try:
        entities = json.loads(entities_resp.content)
    except:
        entities = []
    ctx_parts = []
    for e in entities:
        records = graph.query("""
MATCH (n) WHERE toLower(n.name) CONTAINS toLower($name)
OPTIONAL MATCH (n)-[r]-(m)
RETURN n.name, type(r), m.name LIMIT 5
""", params={"name": e})
        for r in records:
            ctx_parts.append(f"{r['n.name']} --[{r['type(r)']}]--> {r['m.name']}")
    ctx = "\n".join(ctx_parts) or "No graph context."
    return llm.invoke([HumanMessage(content=f"Answer using graph:\n{ctx}\n\nQ: {q}")]).content

def hybrid_graphrag(q: str) -> str:
    v_docs = vectorstore.similarity_search(q, k=2)
    v_ctx = "\n".join(d.page_content for d in v_docs)
    entities_resp = llm.invoke([HumanMessage(content=f"Extract entities from: '{q}'. Return JSON array.")])
    try:
        entities = json.loads(entities_resp.content)
    except:
        entities = []
    g_parts = []
    for e in entities:
        records = graph.query("MATCH (n) WHERE toLower(n.name) CONTAINS toLower($name) OPTIONAL MATCH (n)-[r]-(m) RETURN n.name, type(r), m.name LIMIT 4", params={"name": e})
        for r in records:
            g_parts.append(f"{r['n.name']} --[{r['type(r)']}]--> {r['m.name']}")
    g_ctx = "\n".join(g_parts) or "No graph context."
    ctx = f"Vector:\n{v_ctx}\n\nGraph:\n{g_ctx}"
    return llm.invoke([HumanMessage(content=f"Answer using context:\n{ctx}\n\nQ: {q}")]).content

# Test question that requires following relationships
q = "What project is Vishal building and who is he collaborating with?"

print(f"❓ {q}\n")
print(f"Vector RAG:\n  {vector_rag(q)}\n")
print(f"Graph Only:\n  {graph_only(q)}\n")
print(f"Hybrid GraphRAG:\n  {hybrid_graphrag(q)}\n")
```

---

## L10 — Full GraphRAG System

### 15. Production GraphRAG — Everything Together
```python
# 15_full_graphrag_system.py
from langchain_neo4j import Neo4jGraph
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import os, json, time

load_dotenv()

# ── Config ─────────────────────────────────────────────
embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI", "bolt://localhost:7687"),
    username=os.getenv("NEO4J_USER", "neo4j"),
    password=os.getenv("NEO4J_PASSWORD", "password"),
)
graph.query("MATCH (n) DETACH DELETE n")

# ── Indexing: Text → Graph + Vectors ──────────────────
raw_texts = [
    """Vishal is a 22-year-old AI engineer from Pune. He is building the
    ImageMemorySystem — an AI agent that can remember and retrieve visual
    memories. The system uses LangGraph for orchestration, Neo4j for the
    knowledge graph, and FastAPI for the API layer.""",

    """The ImageMemorySystem ingests screenshots through a multi-step pipeline.
    PaddleOCR extracts text from images. YOLOv8 detects objects. CLIP generates
    visual embeddings. All extracted entities are stored in Neo4j. The system
    exposes a natural language query interface built on top of LangGraph.""",

    """LangGraph extends LangChain to support stateful, cyclical agent workflows.
    It uses a StateGraph with nodes and edges. MemorySaver provides persistence
    between sessions. LangGraph integrates natively with Neo4j for graph-based
    memory. It is built with Python.""",

    """GraphRAG is a retrieval technique that combines knowledge graph traversal
    with vector similarity search. It was introduced by Microsoft Research.
    Unlike standard RAG which retrieves similar chunks, GraphRAG follows entity
    relationships to find connected context. It uses Neo4j as the graph store.""",
]

raw_docs = [Document(page_content=t) for t in raw_texts]

# Build knowledge graph from text
print("Building knowledge graph...")
transformer = LLMGraphTransformer(
    llm=llm,
    allowed_nodes=["Person", "Tool", "Project", "Concept", "Organization"],
    allowed_relationships=["USES", "WORKS_ON", "BUILT_WITH", "EXTENDS", "INTEGRATES_WITH"],
)
graph_docs = transformer.convert_to_graph_documents(raw_docs)
graph.add_graph_documents(graph_docs, baseEntityLabel=True, include_source=True)
graph.refresh_schema()
print(f"Graph built. Schema:\n{graph.schema[:300]}...")

# Build vector store from same docs
vectorstore = FAISS.from_documents(raw_docs, embedder)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# ── Retrieval ──────────────────────────────────────────
def extract_entities(question: str) -> list[str]:
    r = llm.invoke([HumanMessage(
        content=f"Extract named entities from: '{question}'. JSON array only. Return []  if none."
    )])
    try: return json.loads(r.content.strip())
    except: return []

def graph_retrieve(entities: list[str]) -> str:
    if not entities: return ""
    parts = []
    for e in entities:
        records = graph.query("""
MATCH (n) WHERE toLower(n.id) CONTAINS toLower($name) OR toLower(coalesce(n.name,'')) CONTAINS toLower($name)
OPTIONAL MATCH (n)-[r]-(m)
RETURN n.id AS node, type(r) AS rel, m.id AS neighbour
LIMIT 8
""", params={"name": e})
        for r in records:
            if r["node"] and r["neighbour"]:
                parts.append(f"{r['node']} --[{r['rel']}]--> {r['neighbour']}")
    return "Graph connections:\n" + "\n".join(set(parts)) if parts else ""

def vector_retrieve(question: str) -> str:
    docs = retriever.invoke(question)
    return "Relevant passages:\n" + "\n\n".join(d.page_content for d in docs)

# ── Answer ─────────────────────────────────────────────
answer_prompt = ChatPromptTemplate.from_template("""
You are a knowledgeable assistant. Use BOTH the graph connections
and the relevant passages to give a comprehensive answer.
The graph connections show relationships between entities.
The passages contain detailed descriptions.

{graph_context}

{vector_context}

Question: {question}

Answer (use relationships from graph where relevant):""")

chain = answer_prompt | llm | StrOutputParser()

def graphrag(question: str) -> dict:
    start = time.time()
    entities = extract_entities(question)
    graph_ctx = graph_retrieve(entities)
    vector_ctx = vector_retrieve(question)
    answer = chain.invoke({
        "graph_context": graph_ctx or "No graph context found.",
        "vector_context": vector_ctx,
        "question": question
    })
    return {
        "answer": answer,
        "entities": entities,
        "time": round(time.time() - start, 2)
    }

# ── Run ────────────────────────────────────────────────
print("\n=== Full GraphRAG System ===\n")
questions = [
    "What is Vishal building and what tools does it use?",
    "How does GraphRAG differ from standard RAG?",
    "What is the ImageMemorySystem pipeline?",
    "What is the connection between LangGraph and Neo4j?",
]

for q in questions:
    print(f"{'='*55}")
    print(f"❓ {q}")
    result = graphrag(q)
    print(f"💬 {result['answer']}")
    print(f"🔍 Entities: {result['entities']}")
    print(f"⏱  {result['time']}s\n")
```

---

## Bonus Drills

### B1. Build a GraphRAG system for your own tech stack
```
Add 10 tools/concepts you use to the graph
Connect them with meaningful relationships
Ask: "How does X connect to Y?" — it should traverse the graph
```

### B2. Add entity resolution
```
When extracting "Neo4j" and "neo4j" from different docs
Merge them into the same node instead of creating duplicates
Use MERGE + toLower() in your Cypher
```

### B3. Build a graph from a PDF
```
Load a PDF → extract text → LLMGraphTransformer → store in Neo4j
Query the graph with 3 natural language questions
```

### B4. Multi-hop reasoning
```
Q: "What language is used by the tool that Vishal's project uses?"
This requires: Vishal → WORKS_ON → Project → USES → Tool → BUILT_WITH → Language
Write the Cypher for this manually, then via GraphCypherQAChain
```

### B5. Compare: GraphRAG vs RAG on the same corpus
```
Same 5 documents indexed in both FAISS and Neo4j
Same 3 questions
Show where graph retrieval finds connections vector search misses
```

### B6. Full system from scratch in one blank file
```
Build exercise 15 from memory:
  Text → LLMGraphTransformer → Neo4j
  Text → FAISS
  Entity extraction → graph traversal → vector search → answer
No peeking.
```

---

## GraphRAG vs RAG Cheat Sheet

```
When to use RAG:
  ✓ Semantic similarity is enough
  ✓ Questions like: "What is X?" "Explain Y"
  ✓ Simple document QA

When to use GraphRAG:
  ✓ Questions require following relationships
  ✓ "How does X connect to Y?"
  ✓ "What are all the things related to Z?"
  ✓ Multi-hop reasoning across entities
  ✓ You have structured entity-relationship data

GraphRAG pipeline in one view:
  Text
    ↓ LLMGraphTransformer
  (Nodes + Relationships)
    ↓ graph.add_graph_documents()
  Neo4j Graph + FAISS Vectors
    ↓ query time
  extract_entities(question)
    ↓
  graph_retrieve(entities)     ← graph traversal
  vector_retrieve(question)    ← semantic similarity
    ↓ combine
  LLM → Answer

Key Cypher patterns:
  MERGE (n:Label {name: $name})                    # create or find
  MATCH (a)-[r]->(b) RETURN a.name, type(r), b.name  # all relationships
  MATCH (n)-[*1..3]-(m) RETURN DISTINCT m.name       # variable length path
  MATCH path = shortestPath((a)-[*]-(b))             # shortest path
```

---

## Progress Tracker

| Level | Topic                                       | Done? | Time Taken |
|-------|---------------------------------------------|-------|------------|
| L1    | Neo4j connect, create, Cypher CRUD          |  [ ]  |            |
| L2    | Cypher patterns, path traversal             |  [ ]  |            |
| L3    | Neo4jGraph wrapper, GraphCypherQAChain      |  [ ]  |            |
| L4    | LLMGraphTransformer, constrained extraction |  [ ]  |            |
| L5    | Entity-based graph retrieval                |  [ ]  |            |
| L6    | Hybrid: graph + vector retrieval            |  [ ]  |            |
| L7    | Neo4j native vector index                   |  [ ]  |            |
| L8    | Full GraphRAG chain (LCEL)                  |  [ ]  |            |
| L9    | Community summaries                         |  [ ]  |            |
| L10   | Full production GraphRAG system             |  [ ]  |            |
| Bonus | Blank file challenges                       |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 15 from memory.
> Raw text → LLMGraphTransformer → Neo4j → FAISS →
> entity extraction → graph traversal → vector retrieval → answer.
> Ask it: "What is the connection between LangGraph and Neo4j?"
> If it traverses the graph and finds the path — you own GraphRAG.
>
> **Note for your project:** Your ImageMemorySystem IS a GraphRAG system.
> Images → entity extraction → Neo4j knowledge graph → natural language retrieval.
> Every exercise in this sheet is directly applicable to what you're building.
