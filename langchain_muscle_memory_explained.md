# LangChain Muscle Memory — 40+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Run: `python filename.py`
> - Install: `pip install langchain langchain-openai langchain-community
>             langchain-core faiss-cpu tiktoken python-dotenv`
> - Set `OPENAI_API_KEY` in `.env`
> - L1–L2 → LLMs, prompt templates (20–30 min)
> - L3–L4 → output parsers, LCEL chains (30–40 min)
> - L5–L6 → memory, document loaders (30–40 min)
> - L7–L8 → retrievers, tools, agents (40–50 min)
> - L9–L10 → callbacks, full pipeline (50–60 min)

---

## The Mental Model — Before You Write a Line

```
LangChain is the glue layer between your code and the LLM.

Three things it gives you:

1. LCEL (LangChain Expression Language)
   chain = prompt | llm | output_parser
   Read left to right. Output of one feeds the next.
   This is the modern way to write everything in LangChain.

2. Components
   Prompts    → structure the input  (what you send to the LLM)
   LLMs       → generate the output  (the model itself)
   Parsers    → structure the output (what you get back)
   Retrievers → fetch relevant docs  (for RAG)
   Tools      → let the LLM act      (search, calculate, etc.)

3. Integrations
   100+ connectors to models, vector DBs, tools, document loaders.
   You don't implement them — you import them.

LangChain vs LangGraph:
  LangChain → linear chains, one-shot pipelines
  LangGraph → stateful graphs, loops, branching, agents
  In practice: LangChain chains INSIDE LangGraph nodes
```

---

## L1 — LLMs and Basic Invocation

### 1. Your First LLM Call

> **📖 What's happening here?**
>
> Think of this as picking up the phone and calling the AI for the first time.
>
> **Key concepts:**
>
> - `load_dotenv()` — reads your `.env` file and loads `OPENAI_API_KEY` into memory. Without this, the code can't authenticate with OpenAI.
> - `ChatOpenAI(...)` — creates a "connection" to the GPT model. `temperature=0` means "be as consistent and predictable as possible" (no randomness). Think of it like setting a dial from "robot-precise" (0) to "creative-wild" (1).
> - `HumanMessage(content="...")` — wraps your text in a special object that tells the LLM "this is what the user said." Chat models expect structured message objects, not raw strings.
> - `llm.invoke([...])` — sends the message list to the LLM and waits for a reply. The `[...]` is a list because a real conversation has multiple messages.
> - `response.content` — the actual text answer from the LLM. The response also contains metadata like which model was used and how many tokens were consumed.
>
> **Data flow:**
> ```
> Your text
>   → wrapped in HumanMessage
>   → sent to OpenAI via invoke()
>   → OpenAI returns an AIMessage object
>   → you read .content to get the plain text answer
> ```

```python
# 01_first_llm_call.py
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Single message
response = llm.invoke([HumanMessage(content="What is LangChain in one sentence?")])
print(response.content)
print(f"\nModel: {response.response_metadata.get('model_name')}")
print(f"Tokens used: {response.usage_metadata}")
```

---

### 2. Message Types

> **📖 What's happening here?**
>
> Chat models don't just take "a question" — they take a **conversation history** as a list of messages. There are three message types, each playing a different role:
>
> | Message Type | Who "speaks" it | Purpose |
> |---|---|---|
> | `SystemMessage` | You (the developer) | Set personality, rules, and behaviour for the AI |
> | `HumanMessage` | The user | What the user types |
> | `AIMessage` | The AI | What the AI previously said (used to simulate history) |
>
> **Why does this matter?**
> The LLM has no memory between calls. If you want it to remember that it already answered a question, you must manually include the previous exchange as `HumanMessage` + `AIMessage` pairs in the list.
>
> **Data flow:**
> ```
> [SystemMessage: "You are a Python expert"]
>   ↓
> [HumanMessage: "What is a decorator?"]
>   ↓
> [AIMessage: "A decorator is..."]   ← you include this to simulate history
>   ↓
> [HumanMessage: "Give me a simple example."]
>   ↓
> LLM reads ALL of this and gives a contextual answer
> ```

```python
# 02_message_types.py
from langchain_openai import ChatOpenAI
from langchain_core.messages import (
    HumanMessage,
    AIMessage,
    SystemMessage,
)
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# System → sets the behaviour
# Human  → what the user says
# AI     → what the model previously said (for history)

messages = [
    SystemMessage(content="You are a Python expert. Be concise and direct."),
    HumanMessage(content="What is a decorator?"),
    AIMessage(content="A decorator is a function that wraps another function to extend its behaviour without modifying it."),
    HumanMessage(content="Give me a simple example."),
]

response = llm.invoke(messages)
print(response.content)
```

---

### 3. Model Parameters

> **📖 What's happening here?**
>
> This drill shows how to control the LLM's "personality" and output size using parameters.
>
> **Key parameters:**
>
> - **`temperature`** — Controls how random/creative the output is.
>   - `0` = always gives the same answer (deterministic, good for code/facts)
>   - `0.9` = gives different answers each time (creative, good for brainstorming)
>   - Think of it like a coffee machine: `0` is plain black coffee every time, `0.9` surprises you.
> - **`max_tokens`** — Hard limit on how long the response can be. Useful to save money or force short answers.
> - **`model`** — Which AI model to use. `gpt-4o-mini` is fast and cheap; `gpt-4o` is smarter but more expensive.
>
> **Data flow:**
> ```
> Same question sent to two different LLM configs
>   → precise_llm (temp=0)   → same answer every run
>   → creative_llm (temp=0.9) → different answer every run
> ```
>
> The loop runs each twice so you can see: precise returns identical results, creative returns varied ones.

```python
# 03_model_params.py
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv

load_dotenv()

# temperature: 0 = deterministic, 1 = creative
# max_tokens: limit output length
# model: which model to use

precise_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
creative_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.9)

prompt = [HumanMessage(content="Write a one-line tagline for a Python learning app.")]

print("Precise (temp=0):")
for _ in range(2):
    print(f"  {precise_llm.invoke(prompt).content}")

print("\nCreative (temp=0.9):")
for _ in range(2):
    print(f"  {creative_llm.invoke(prompt).content}")

# Limit output tokens
short_llm = ChatOpenAI(model="gpt-4o-mini", max_tokens=20)
response = short_llm.invoke([HumanMessage(content="Explain Python in detail.")])
print(f"\nWith max_tokens=20: {response.content}")
```

---

### 4. Batch and Streaming

> **📖 What's happening here?**
>
> Two ways to call the LLM when you need more than one response:
>
> **Batch** — Send multiple inputs all at once in a single API call (more efficient than calling one by one). Like ordering 3 pizzas in one call vs. 3 separate calls.
> - `llm.batch([list_of_message_lists])` — each item in the outer list is one separate conversation.
> - Returns a list of responses in the same order.
>
> **Streaming** — Instead of waiting for the full response, get words (tokens) as they arrive, just like you see in ChatGPT's typing effect.
> - `streaming=True` enables this on the LLM.
> - `llm.stream([...])` returns a generator — each `chunk` is a tiny piece of the final answer.
> - `end=""` and `flush=True` in the print call prevent newlines between chunks, creating the smooth typewriter effect.
>
> **Data flow (Batch):**
> ```
> [question1, question2, question3]
>   → sent to OpenAI in one batch call
>   → [answer1, answer2, answer3] returned together
> ```
>
> **Data flow (Streaming):**
> ```
> question
>   → OpenAI starts generating
>   → sends chunk1 → you print it immediately
>   → sends chunk2 → you print it immediately
>   → ... until full answer is done
> ```

```python
# 04_batch_stream.py
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Batch — send multiple inputs at once (more efficient)
questions = [
    [HumanMessage(content="What is FastAPI?")],
    [HumanMessage(content="What is LangGraph?")],
    [HumanMessage(content="What is CrewAI?")],
]

print("=== Batch ===")
responses = llm.batch(questions)
for q, r in zip(questions, responses):
    print(f"Q: {q[0].content}")
    print(f"A: {r.content}\n")

# Streaming — get tokens as they arrive
print("=== Streaming ===")
stream_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0, streaming=True)
print("Response: ", end="", flush=True)
for chunk in stream_llm.stream([HumanMessage(content="Count to 5 slowly.")]):
    print(chunk.content, end="", flush=True)
print()
```

---

## L2 — Prompt Templates

### 5. PromptTemplate — String Prompts

> **📖 What's happening here?**
>
> Instead of writing a new string every time you call the LLM, a **PromptTemplate** is like a reusable form with blank fields you fill in.
>
> **Why use templates?**
> - Reuse the same structure with different values
> - Keep prompts clean and separate from logic
> - Easy to test with different inputs
>
> **Key concepts:**
>
> - `input_variables=["topic", "audience"]` — declares the "blanks" in the template
> - `template="Explain {topic} to a {audience}..."` — `{topic}` and `{audience}` are placeholders, like Python f-strings
> - `template.format(topic="decorators", audience="beginner")` — fills the blanks manually and returns a plain string
> - `chain = template | llm | StrOutputParser()` — this is **LCEL** (the pipe `|` operator). It means: fill the template → send to LLM → parse output to plain string. Think of it as an assembly line.
>
> **Data flow:**
> ```
> {"topic": "async/await", "audience": "junior developer"}
>   → PromptTemplate fills blanks → "Explain async/await to a junior developer in exactly 2 sentences."
>   → LLM generates answer → AIMessage object
>   → StrOutputParser extracts .content → plain string
> ```

```python
# 05_prompt_template.py
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Basic template
template = PromptTemplate(
    input_variables=["topic", "audience"],
    template="Explain {topic} to a {audience} in exactly 2 sentences."
)

# Format manually
prompt_text = template.format(topic="decorators", audience="beginner")
print("Formatted prompt:", prompt_text)

# Or pipe directly
chain = template | llm | StrOutputParser()
result = chain.invoke({"topic": "async/await", "audience": "junior developer"})
print("\nResult:", result)

# Multiple invocations with different inputs
topics = [
    {"topic": "RAG", "audience": "product manager"},
    {"topic": "vector databases", "audience": "CEO"},
]
for inputs in topics:
    print(f"\n{inputs['topic']} for {inputs['audience']}:")
    print(chain.invoke(inputs))
```

---

### 6. ChatPromptTemplate — Structured Conversations

> **📖 What's happening here?**
>
> While `PromptTemplate` produces a plain string, `ChatPromptTemplate` produces a proper **list of messages** (System + Human + AI), which is exactly what chat models expect.
>
> **Key concepts:**
>
> - `ChatPromptTemplate.from_messages([...])` — builds a template from a list of `(role, text)` tuples. Roles: `"system"`, `"human"`, `"ai"`.
> - `{role}`, `{style}`, `{question}` — placeholders inside message text, filled when you `.invoke(...)`.
> - `MessagesPlaceholder(variable_name="history")` — a special slot where you inject a dynamic list of past messages. This is how you plug conversation history into a template.
>
> **Why `MessagesPlaceholder`?**
> Because the history list can have any number of messages — you can't hardcode it in the template. The placeholder says "insert whatever list I pass here."
>
> **Data flow:**
> ```
> Template with placeholders
>   → invoke({"role": "Python expert", "style": "concise", "question": "What is a generator?"})
>   → produces: [SystemMessage(...), HumanMessage(...)]
>   → LLM generates: AIMessage(...)
>   → StrOutputParser: plain string
>
> With history:
>   → history = [HumanMessage("My name is Vishal"), AIMessage("Hello Vishal!")]
>   → invoke({"history": [...], "input": "What is my name?"})
>   → LLM sees full context and answers: "Your name is Vishal."
> ```

```python
# 06_chat_prompt_template.py
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.messages import HumanMessage, AIMessage
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Basic chat template
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role}. Be {style}."),
    ("human", "{question}")
])

chain = prompt | llm | StrOutputParser()

result = chain.invoke({
    "role": "Python expert",
    "style": "concise and direct",
    "question": "What is a generator?"
})
print(result)

# With conversation history placeholder
history_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

history_chain = history_prompt | llm | StrOutputParser()

history = [
    HumanMessage(content="My name is Vishal."),
    AIMessage(content="Hello Vishal! How can I help you today?"),
]

result = history_chain.invoke({
    "history": history,
    "input": "What is my name?"
})
print(f"\nWith history: {result}")
```

---

### 7. Few-Shot Prompting

> **📖 What's happening here?**
>
> **Few-shot prompting** means teaching the LLM the format you want by showing it examples *inside the prompt itself*. Instead of describing what you want in words, you show it.
>
> It's like training an intern by example: "When I say X, you say Y. When I say A, you say B. Got it? Now do C."
>
> **Key concepts:**
>
> - `examples = [{"input": "happy", "output": "sad"}, ...]` — a list of input/output pairs that demonstrate the pattern
> - `example_prompt = ChatPromptTemplate.from_messages([("human", "{input}"), ("ai", "{output}")])` — the template for each individual example (how to format each pair)
> - `FewShotChatMessagePromptTemplate` — takes your examples + example_prompt and compiles them into a series of Human/AI message pairs
> - `final_prompt` — wraps everything: system instruction + all examples + the actual new question
>
> **Data flow:**
> ```
> examples = [happy→sad, tall→short, fast→slow, bright→dark]
>   → formatted as: Human:"happy" / AI:"sad" / Human:"tall" / AI:"short" / ...
>   → combined with system + new question: "loud"
>   → LLM sees the pattern and returns: "quiet"
> ```
>
> The LLM infers "oh, you want antonyms" just from the examples — no explicit instruction needed (though we add one in the system message too, to be safe).

```python
# 07_few_shot.py
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Examples that teach the LLM the format you want
examples = [
    {"input": "happy",   "output": "sad"},
    {"input": "tall",    "output": "short"},
    {"input": "fast",    "output": "slow"},
    {"input": "bright",  "output": "dark"},
]

example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

few_shot_prompt = FewShotChatMessagePromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
)

final_prompt = ChatPromptTemplate.from_messages([
    ("system", "Give the antonym of the word provided. One word only."),
    few_shot_prompt,
    ("human", "{word}")
])

chain = final_prompt | llm | StrOutputParser()

words = ["loud", "cold", "heavy", "sharp"]
for word in words:
    result = chain.invoke({"word": word})
    print(f"{word} → {result}")
```

---

## L3 — Output Parsers

### 8. StrOutputParser

> **📖 What's happening here?**
>
> When you call an LLM, it returns an **`AIMessage` object** — not a plain string. It's like getting a wrapped gift. `StrOutputParser` is the unwrapper — it extracts just the text from inside.
>
> **Why does this matter?**
> Without a parser, you'd always have to type `response.content` to get the text. With `StrOutputParser` in your chain, it automatically unwraps for you.
>
> **Comparison:**
>
> | Without Parser | With StrOutputParser |
> |---|---|
> | Returns `AIMessage(content="...")` object | Returns `"..."` plain string |
> | You do `result.content` to read it | You use `result` directly |
> | Type: `AIMessage` | Type: `str` |
>
> **Data flow:**
> ```
> Without parser:
>   prompt → LLM → AIMessage(content="FastAPI is a...")
>   result.content → "FastAPI is a..."
>
> With parser:
>   prompt → LLM → AIMessage → StrOutputParser → "FastAPI is a..."
>   result → "FastAPI is a..."   (already unwrapped!)
> ```

```python
# 08_str_parser.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini")

# Without parser — returns AIMessage object
raw_chain = ChatPromptTemplate.from_template("What is {topic}?") | llm
raw_result = raw_chain.invoke({"topic": "FastAPI"})
print(type(raw_result))       # <class 'AIMessage'>
print(raw_result.content[:50])

# With StrOutputParser — returns plain string
str_chain = (
    ChatPromptTemplate.from_template("What is {topic}?")
    | llm
    | StrOutputParser()
)
str_result = str_chain.invoke({"topic": "FastAPI"})
print(type(str_result))       # <class 'str'>
print(str_result[:50])
```

---

### 9. JsonOutputParser

> **📖 What's happening here?**
>
> Sometimes you don't want free-form text — you want the LLM to return **structured data** (like a dictionary) that your code can actually use. `JsonOutputParser` makes this happen.
>
> **How it works:**
> 1. You define a **Pydantic model** — a Python class that describes the exact shape of data you want (field names, types, descriptions).
> 2. `JsonOutputParser` generates "format instructions" — special text that tells the LLM "output your answer as JSON with these exact fields."
> 3. You inject those instructions into the prompt.
> 4. The parser takes the LLM's JSON string and converts it to a Python dict.
>
> **Key concepts:**
> - `BaseModel` — Pydantic's base class for defining data structures
> - `Field(description="...")` — adds a description that the parser includes in the format instructions sent to the LLM
> - `parser.get_format_instructions()` — returns a string like "Return JSON with fields: name (str), pros (list)..."
> - `{format_instructions}` in the prompt — placeholder where the instructions get injected
>
> **Data flow:**
> ```
> Pydantic model (TechAnalysis)
>   → parser.get_format_instructions() → "Return JSON with {name, pros, cons, score}"
>   → injected into system prompt
>   → LLM returns: '{"name": "FastAPI", "pros": [...], "score": 9}'
>   → JsonOutputParser parses it → Python dict you can use normally
> ```

```python
# 09_json_parser.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field
from typing import List
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Define the schema you want back
class TechAnalysis(BaseModel):
    name: str = Field(description="Name of the technology")
    pros: List[str] = Field(description="List of 3 advantages")
    cons: List[str] = Field(description="List of 2 disadvantages")
    score: int = Field(description="Rating from 1 to 10")

parser = JsonOutputParser(pydantic_object=TechAnalysis)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a technology analyst. {format_instructions}"),
    ("human", "Analyse {technology}")
])

chain = prompt | llm | parser

result = chain.invoke({
    "technology": "FastAPI",
    "format_instructions": parser.get_format_instructions()
})

# Now result is a proper Python dict
print(f"Name: {result['name']}")
print(f"Score: {result['score']}/10")
print("Pros:")
for p in result['pros']:
    print(f"  ✓ {p}")
print("Cons:")
for c in result['cons']:
    print(f"  ✗ {c}")
```

---

### 10. PydanticOutputParser

> **📖 What's happening here?**
>
> Similar to `JsonOutputParser`, but instead of returning a Python `dict`, it returns a fully-typed **Pydantic object**. This means you get auto-complete, type checking, and proper attribute access with a dot (`.`) instead of `["key"]`.
>
> **JsonOutputParser vs PydanticOutputParser:**
>
> | Feature | JsonOutputParser | PydanticOutputParser |
> |---|---|---|
> | Returns | Python `dict` | Pydantic model instance |
> | Access fields | `result["name"]` | `result.name` |
> | Type safety | No (just a dict) | Yes (typed attributes) |
> | Use when | Quick data extraction | Production code, type safety needed |
>
> **Key concepts:**
> - `RecipeCard(BaseModel)` — defines the exact shape of the data. Every field has a type (`str`, `int`, `List[str]`) and a description.
> - `recipe.name`, `recipe.serves` — accessed like normal Python object attributes (not dictionary keys)
> - The LLM is told to return JSON matching this schema; the parser converts it to a real `RecipeCard` Python object
>
> **Data flow:**
> ```
> "Give me a recipe for scrambled eggs"
>   → format instructions injected into prompt
>   → LLM returns JSON string matching RecipeCard schema
>   → PydanticOutputParser converts to RecipeCard object
>   → recipe.name → "Classic Scrambled Eggs"
>   → recipe.ingredients → ["3 eggs", "1 tbsp butter", ...]
> ```

```python
# 10_pydantic_parser.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field
from typing import List, Optional
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

class RecipeCard(BaseModel):
    name: str = Field(description="Name of the recipe")
    ingredients: List[str] = Field(description="List of ingredients with quantities")
    steps: List[str] = Field(description="Numbered cooking steps")
    prep_time_minutes: int = Field(description="Prep time in minutes")
    difficulty: str = Field(description="Easy, Medium, or Hard")
    serves: int = Field(description="Number of servings")

parser = PydanticOutputParser(pydantic_object=RecipeCard)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a chef. Output recipe data in the requested format.\n{format_instructions}"),
    ("human", "Give me a recipe for {dish}")
])

chain = prompt | llm | parser

recipe = chain.invoke({
    "dish": "scrambled eggs",
    "format_instructions": parser.get_format_instructions()
})

# recipe is a RecipeCard Pydantic object — fully typed
print(f"🍳 {recipe.name}")
print(f"Serves {recipe.serves} | {recipe.prep_time_minutes} min | {recipe.difficulty}")
print("\nIngredients:")
for ing in recipe.ingredients:
    print(f"  - {ing}")
print("\nSteps:")
for i, step in enumerate(recipe.steps, 1):
    print(f"  {i}. {step}")
```

---

## L4 — LCEL Chains

### 11. Basic LCEL Chain

> **📖 What's happening here?**
>
> **LCEL (LangChain Expression Language)** is the `|` (pipe) operator applied to LangChain components. It's the heart of modern LangChain. The idea is borrowed from Unix pipes: output of one thing flows into the next.
>
> ```
> prompt | llm | parser
> ```
> This reads left to right: "take the prompt, send it to the LLM, then parse the output."
>
> **New components introduced:**
>
> - **`RunnableLambda`** — wraps any Python function so it can be used inside a chain with `|`. Here it's used to make `make_uppercase` work as a chain step.
> - **`RunnablePassthrough`** — passes the input unchanged. Used when you want to preserve the original input while also transforming it.
> - **`RunnableParallel`** — runs multiple chains at the same time on the same input, and returns a dict with all results. Like forking a road — same input goes down multiple paths simultaneously.
>
> **Data flow (transform_chain):**
> ```
> {"animal": "octopus"}
>   → ChatPromptTemplate → "Tell me a fact about octopus"
>   → LLM → "Octopus has three hearts..."
>   → StrOutputParser → plain string
>   → RunnableLambda(make_uppercase) → "OCTOPUS HAS THREE HEARTS..."
> ```
>
> **Data flow (parallel_chain):**
> ```
> {"text": "hello world"}
>   → RunnableParallel runs two paths simultaneously:
>       original: RunnablePassthrough → {"text": "hello world"} (unchanged)
>       upper: lambda → "HELLO WORLD"
>   → returns {"original": {"text": "hello world"}, "upper": "HELLO WORLD"}
> ```

```python
# 11_lcel_basic.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableLambda
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = StrOutputParser()

# LCEL = pipe operator chains components
# Each component's output becomes the next component's input

# Simple chain
simple_chain = (
    ChatPromptTemplate.from_template("Translate to French: {text}")
    | llm
    | parser
)
print(simple_chain.invoke({"text": "Hello, how are you?"}))

# Chain with transformation
def make_uppercase(text: str) -> str:
    return text.upper()

transform_chain = (
    ChatPromptTemplate.from_template("Tell me a fact about {animal}")
    | llm
    | parser
    | RunnableLambda(make_uppercase)
)
print(transform_chain.invoke({"animal": "octopus"}))

# Passthrough — pass input unchanged alongside transformed version
from langchain_core.runnables import RunnableParallel

parallel_chain = RunnableParallel(
    original=RunnablePassthrough(),
    upper=RunnableLambda(lambda x: x["text"].upper())
)
print(parallel_chain.invoke({"text": "hello world"}))
```

---

### 12. Sequential Chains

> **📖 What's happening here?**
>
> This is a **multi-step pipeline** where the output of Step 1 becomes the input of Step 2, which feeds Step 3. Think of it as an assembly line where each station adds value before passing the work to the next.
>
> **The pipeline here has 3 stages:**
> 1. **outline_chain** — takes a topic, generates a 3-point blog outline
> 2. **intro_chain** — takes the outline, writes a 2-sentence introduction
> 3. **cta_chain** — takes the intro, adds a call-to-action sentence
>
> **Key concept: `RunnablePassthrough.assign(key=chain)`**
> This is the glue. It says: "keep everything in the current dict, and *also* add a new key by running this chain." So the dict grows at each step:
> - After step 1: `{"topic": "...", "outline": "..."}`
> - After step 2: `{"topic": "...", "outline": "...", "intro": "..."}`
> - After step 3: `{"topic": "...", "outline": "...", "intro": "...", "final": "..."}`
>
> **Data flow:**
> ```
> {"topic": "why every developer should learn FastAPI"}
>   → assign(outline=outline_chain) → adds "outline" key
>   → assign(intro=lambda...) → adds "intro" key (uses outline)
>   → assign(final=lambda...) → adds "final" key (uses intro)
>   → result dict has all three outputs accessible
> ```

```python
# 12_sequential_chains.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = StrOutputParser()

# Chain 1: Generate a blog post outline
outline_chain = (
    ChatPromptTemplate.from_template(
        "Write a 3-point outline for a blog post about {topic}. Number each point."
    )
    | llm
    | parser
)

# Chain 2: Write intro based on outline
intro_chain = (
    ChatPromptTemplate.from_template(
        "Given this outline:\n{outline}\n\nWrite a 2-sentence blog intro."
    )
    | llm
    | parser
)

# Chain 3: Add a CTA
cta_chain = (
    ChatPromptTemplate.from_template(
        "Given this intro:\n{intro}\n\nAdd one compelling call-to-action sentence."
    )
    | llm
    | parser
)

# Wire them together — output of each feeds the next
full_pipeline = (
    {"topic": RunnablePassthrough()}
    | RunnablePassthrough.assign(outline=outline_chain)
    | RunnablePassthrough.assign(
        intro=lambda x: intro_chain.invoke({"outline": x["outline"]})
    )
    | RunnablePassthrough.assign(
        final=lambda x: cta_chain.invoke({"intro": x["intro"]})
    )
)

result = full_pipeline.invoke({"topic": "why every developer should learn FastAPI"})
print("=== Outline ===")
print(result["outline"])
print("\n=== Intro ===")
print(result["intro"])
print("\n=== With CTA ===")
print(result["final"])
```

---

### 13. Branching Chains with RunnableBranch

> **📖 What's happening here?**
>
> Instead of one linear path, this is a **routing system** — the input is first classified, and then sent down the correct chain based on what it is. Like a call center that routes you to Sales, Support, or Billing based on why you're calling.
>
> **The two-step process:**
> 1. **Classify** — `classify_chain` reads the message and returns one word: `greeting`, `question`, `complaint`, or `other`
> 2. **Branch** — `RunnableBranch` checks the intent and runs the appropriate chain
>
> **Key concepts:**
>
> - `RunnableLambda(str.strip)` and `RunnableLambda(str.lower)` — chained right after the parser to clean up the output (trim whitespace, lowercase). This prevents mismatches like `"Question"` vs `"question"`.
> - `RunnableBranch((condition_fn, chain), ..., default_chain)` — evaluates conditions in order and runs the first matching chain. The last argument (no condition) is the default/fallback.
>
> **Data flow:**
> ```
> "How does RAG work?"
>   → classify_chain → "question"
>   → RunnableBranch checks: is intent "greeting"? No. Is it "question"? YES!
>   → question_chain.invoke({"message": "How does RAG work?"})
>   → "RAG (Retrieval Augmented Generation) works by..."
> ```

```python
# 13_branching_chain.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableBranch, RunnableLambda
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = StrOutputParser()

# Classify intent first
classify_chain = (
    ChatPromptTemplate.from_template(
        """Classify this message as one of: greeting, question, complaint, other.
Return ONLY the single word.
Message: {message}"""
    )
    | llm
    | parser
    | RunnableLambda(str.strip)
    | RunnableLambda(str.lower)
)

# Different chains for different intents
greeting_chain = (
    ChatPromptTemplate.from_template("Respond warmly to this greeting: {message}")
    | llm | parser
)

question_chain = (
    ChatPromptTemplate.from_template("Answer this question helpfully: {message}")
    | llm | parser
)

complaint_chain = (
    ChatPromptTemplate.from_template("Respond empathetically to this complaint: {message}")
    | llm | parser
)

default_chain = (
    ChatPromptTemplate.from_template("Respond helpfully to: {message}")
    | llm | parser
)

# Router
def route(inputs: dict) -> str:
    msg = inputs["message"]
    intent = classify_chain.invoke({"message": msg})
    inputs["intent"] = intent
    return intent

branch = RunnableBranch(
    (lambda x: x["intent"] == "greeting", greeting_chain),
    (lambda x: x["intent"] == "question", question_chain),
    (lambda x: x["intent"] == "complaint", complaint_chain),
    default_chain
)

def full_router(message: str) -> str:
    intent = classify_chain.invoke({"message": message})
    print(f"  [Intent: {intent}]")
    return branch.invoke({"message": message, "intent": intent})

messages = [
    "Hello! Nice to meet you.",
    "How does RAG work?",
    "Your app crashed and I lost all my data!",
    "The weather is nice today.",
]

for msg in messages:
    print(f"\n📨 {msg}")
    print(f"💬 {full_router(msg)}")
```

---

## L5 — Memory in Chains

### 14. Conversation Buffer Memory

> **📖 What's happening here?**
>
> LLMs are **stateless** — they remember nothing between calls. Every time you call `.invoke()`, the LLM starts fresh with no memory of what was said before. To build a chatbot that "remembers," you must manually carry the history.
>
> This drill shows the **simplest and most transparent** approach: maintain a Python list of messages yourself.
>
> **Key concept — the `ConversationChain` class:**
> - `self.history = []` — the memory. Starts empty.
> - Each time `.chat(user_input)` is called:
>   1. It passes the current `history` + new `user_input` to the LLM
>   2. Gets a response
>   3. Appends both `HumanMessage(user_input)` and `AIMessage(response)` to `history`
>   4. Next call will include these, giving the LLM context
>
> **Data flow for 2nd message ("What am I building?"):**
> ```
> history = [
>   HumanMessage("My name is Vishal and I'm building an AI memory system."),
>   AIMessage("Nice to meet you Vishal! An AI memory system sounds fascinating...")
> ]
> + new HumanMessage("What am I building?")
>   → LLM sees the full conversation → "You're building an AI memory system."
> ```
>
> After 4 turns, `history` has 8 messages (4 Human + 4 AI). The LLM "remembers" everything because you're sending everything every time.

```python
# 14_conversation_memory.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.messages import HumanMessage, AIMessage
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.5)

# Manual conversation history — most transparent approach
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant. Be concise."),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

chain = prompt | llm | StrOutputParser()

class ConversationChain:
    def __init__(self):
        self.history = []

    def chat(self, user_input: str) -> str:
        response = chain.invoke({
            "history": self.history,
            "input": user_input
        })
        # Add this turn to history
        self.history.append(HumanMessage(content=user_input))
        self.history.append(AIMessage(content=response))
        return response

    def clear(self):
        self.history = []

# Demo
bot = ConversationChain()

turns = [
    "My name is Vishal and I'm building an AI memory system.",
    "What am I building?",
    "What is my name?",
    "Summarise what you know about me.",
]

for turn in turns:
    print(f"\n👤 {turn}")
    print(f"🤖 {bot.chat(turn)}")

print(f"\nHistory length: {len(bot.history)} messages")
```

---

### 15. RunnableWithMessageHistory

> **📖 What's happening here?**
>
> The previous drill had you manage history manually. `RunnableWithMessageHistory` **automates** this — it handles storing and retrieving conversation history for you, including multi-user, multi-session support.
>
> **Key concepts:**
>
> - **`InMemoryHistory`** — a custom class that stores messages in a Python list. It implements two required methods: `add_messages()` (append new messages) and `clear()` (wipe history).
> - **`store = {}`** — a dictionary where each key is a `session_id` (like a user ID) and each value is that user's `InMemoryHistory`.
> - **`get_history(session_id)`** — a function that returns the history for a given session. If it doesn't exist yet, creates a new empty one.
> - **`RunnableWithMessageHistory(chain, get_history, ...)`** — wraps your chain with automatic history management. Before each call it loads history; after each call it saves the new messages.
> - **`config={"configurable": {"session_id": "session-A"}}`** — tells the wrapper which session's history to use.
>
> **Data flow:**
> ```
> Session A: "My name is Vishal."
>   → store["session-A"] = [HumanMessage("My name is Vishal"), AIMessage("Hello Vishal!")]
>
> Session B: "My name is Alice."
>   → store["session-B"] = [HumanMessage("My name is Alice"), AIMessage("Hello Alice!")]
>
> Session A: "Do you remember my name?"
>   → loads store["session-A"] (has Vishal's history)
>   → LLM answers: "Yes, your name is Vishal."
>   (Alice's session is completely separate)
> ```

```python
# 15_runnable_with_history.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.chat_history import BaseChatMessageHistory
from langchain_core.messages import BaseMessage
from langchain_core.runnables.history import RunnableWithMessageHistory
from dotenv import load_dotenv
from typing import List

load_dotenv()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Simple in-memory history store
class InMemoryHistory(BaseChatMessageHistory):
    def __init__(self):
        self.messages: List[BaseMessage] = []

    def add_messages(self, messages: List[BaseMessage]) -> None:
        self.messages.extend(messages)

    def clear(self) -> None:
        self.messages = []

# Session store — keyed by session_id
store: dict[str, InMemoryHistory] = {}

def get_history(session_id: str) -> InMemoryHistory:
    if session_id not in store:
        store[session_id] = InMemoryHistory()
    return store[session_id]

# Chain
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

chain = prompt | llm | StrOutputParser()

# Wrap with history management
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_history,
    input_messages_key="input",
    history_messages_key="history",
)

def chat(message: str, session_id: str) -> str:
    return chain_with_history.invoke(
        {"input": message},
        config={"configurable": {"session_id": session_id}}
    )

# Two separate sessions
print("=== Session A ===")
print(chat("My name is Vishal.", "session-A"))
print(chat("What is my name?", "session-A"))

print("\n=== Session B (different user) ===")
print(chat("My name is Alice.", "session-B"))
print(chat("What is my name?", "session-B"))

print("\n=== Back to Session A ===")
print(chat("Do you remember my name?", "session-A"))
```

---

## L6 — Document Loaders and Text Splitters

### 16. Load, Split, and Embed

> **📖 What's happening here?**
>
> This is the foundation of **RAG (Retrieval-Augmented Generation)** — the technique of giving the LLM access to your own documents at query time. There are 4 steps:
>
> **Step 1 — Load:** `TextLoader` reads a file from disk and wraps its content in a `Document` object (which has `page_content` and `metadata`).
>
> **Step 2 — Split:** Documents can be huge. LLMs have a token limit, so you can't send an entire book as context. `RecursiveCharacterTextSplitter` breaks the document into smaller **chunks**.
> - `chunk_size=200` — each chunk is at most 200 characters
> - `chunk_overlap=30` — chunks overlap by 30 characters so context isn't lost at boundaries (like cutting a sentence in half)
> - `separators=["\n\n", "\n", ". ", " "]` — prefers splitting on paragraph breaks first, then newlines, then sentences, then spaces
>
> **Step 3 — Embed:** `OpenAIEmbeddings` converts each chunk of text into a **vector** (a list of numbers). These numbers represent the *meaning* of the text in mathematical space. Similar meanings → similar vectors.
>
> **Step 4 — Store & Search:** `FAISS` stores all the vectors in a fast search index. When you ask a question, your question is also turned into a vector, and FAISS finds the chunks whose vectors are closest (most similar in meaning).
>
> **Data flow:**
> ```
> text file
>   → TextLoader → 1 Document object
>   → RecursiveCharacterTextSplitter → 8 smaller chunk Documents
>   → OpenAIEmbeddings → each chunk becomes a [0.2, -0.5, ...] vector
>   → FAISS stores all vectors
>   → query "How do agents work?" → converted to vector → FAISS finds top 3 closest chunks
> ```

```python
# 16_load_split_embed.py
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from dotenv import load_dotenv
import os

load_dotenv()

# Create test document
os.makedirs("docs", exist_ok=True)
with open("docs/langchain_notes.txt", "w") as f:
    f.write("""LangChain Overview
LangChain is a framework for building applications with large language models.
It provides abstractions for prompts, chains, memory, tools, and agents.

LCEL - LangChain Expression Language
LCEL uses the pipe operator to chain components together.
Each component takes input and produces output for the next component.
This makes chains composable, readable, and easy to extend.

Memory in LangChain
LangChain supports multiple memory types for conversations.
ConversationBufferMemory stores all messages in a buffer.
ConversationSummaryMemory summarises old messages to save tokens.
VectorStoreRetrieverMemory stores messages in a vector database.

Agents and Tools
Agents use LLMs to decide which tools to use and when.
Tools are functions the LLM can call to get information or take actions.
The ReAct pattern (Reason + Act) is the most common agent pattern.
""")

# Step 1: Load
loader = TextLoader("docs/langchain_notes.txt")
docs = loader.load()
print(f"Loaded {len(docs)} document(s)")

# Step 2: Split
splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,
    chunk_overlap=30,
    separators=["\n\n", "\n", ". ", " "]
)
chunks = splitter.split_documents(docs)
print(f"Split into {len(chunks)} chunks")

# Step 3: Embed and store
embedder = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(chunks, embedder)
print(f"Indexed {vectorstore.index.ntotal} vectors")

# Step 4: Search
results = vectorstore.similarity_search("How do agents work in LangChain?", k=3)
print("\nTop matches:")
for doc in results:
    print(f"  - {doc.page_content[:80]}...")
```

---

## L7 — Retrieval Chains

### 17. RetrievalQA Chain

> **📖 What's happening here?**
>
> This is a full **RAG chain** — it retrieves relevant documents and uses them to answer a question. The LLM only answers based on what's in the provided context (not from its training data).
>
> **Key concepts:**
>
> - **`Document`** — the basic unit of text in LangChain. Has `page_content` (the text) and optional `metadata`.
> - **`vectorstore.as_retriever(search_kwargs={"k": 3})`** — converts the FAISS vector store into a Retriever object. A Retriever's job is simple: take a query string, return the top `k` relevant Documents.
> - **`format_docs(docs)`** — a helper function that takes a list of Documents and joins their text with newlines. This creates the "context" block that goes into the prompt.
> - **`{context}: retriever | format_docs`** — this is the key LCEL pattern for RAG. The `retriever` fetches docs, `format_docs` formats them, and the result fills the `{context}` placeholder.
> - **`{question}: RunnablePassthrough()`** — passes the original question through unchanged to fill the `{question}` placeholder.
>
> **Data flow:**
> ```
> "What does the pipe operator do in LCEL?"
>   → split into two parallel paths:
>       context path: question → retriever → [3 relevant docs] → format_docs → "- LCEL chains use..."
>       question path: RunnablePassthrough → "What does the pipe operator do..."
>   → both combined into the prompt
>   → LLM answers using ONLY the provided context
> ```
>
> Notice the last question ("How do I add routing logic?") is NOT in the docs — the LLM should say "I don't know" instead of making something up.

```python
# 17_retrieval_qa.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Build a small knowledge base
docs = [
    Document(page_content="LCEL chains use the | pipe operator to connect components."),
    Document(page_content="ChatPromptTemplate structures messages for chat models."),
    Document(page_content="StrOutputParser converts AIMessage to a plain string."),
    Document(page_content="RunnablePassthrough passes input through unchanged."),
    Document(page_content="RunnableParallel runs multiple chains simultaneously."),
    Document(page_content="RunnableLambda wraps a Python function as a Runnable."),
    Document(page_content="RunnableWithMessageHistory adds session-based memory to any chain."),
]

vectorstore = FAISS.from_documents(docs, embedder)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

prompt = ChatPromptTemplate.from_template("""
Answer using only the context provided.
Say "I don't know" if the answer isn't in the context.

Context:
{context}

Question: {question}

Answer:""")

def format_docs(docs: list) -> str:
    return "\n".join(f"- {d.page_content}" for d in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

questions = [
    "What does the pipe operator do in LCEL?",
    "How do I pass input through a chain unchanged?",
    "What is StrOutputParser used for?",
    "How do I add routing logic to a chain?",  # not in docs
]

for q in questions:
    print(f"\n❓ {q}")
    print(f"💬 {rag_chain.invoke(q)}")
```

---

### 18. Multi-Query Retrieval Chain

> **📖 What's happening here?**
>
> Standard retrieval has a problem: if your question is phrased differently from how the document was written, the vector similarity search might miss the right chunks. **Multi-Query Retrieval** solves this by generating multiple versions of your question automatically.
>
> **How `MultiQueryRetriever` works:**
> 1. Takes your question: "How does LangGraph manage flow and state?"
> 2. Uses the LLM to generate 3–5 alternative phrasings: "What controls data flow in LangGraph?", "How is state handled in LangGraph?", "Explain LangGraph's routing mechanism"
> 3. Runs each phrasing as a separate vector search
> 4. Combines and deduplicates all results
> 5. Returns a richer, more complete set of relevant documents
>
> **Key concept: `MultiQueryRetriever.from_llm(retriever=..., llm=...)`**
> - Wraps your existing retriever with LLM-powered query expansion
> - The LLM is used only for generating query variants, not for answering
>
> **Why this matters:**
> Imagine your doc says "LangGraph MemorySaver persists state" but you ask "How does LangGraph remember things?" — the words don't match well. Multi-query generates "persist state LangGraph" as a variant and finds it.
>
> **Data flow:**
> ```
> "How does LangGraph manage flow and state?"
>   → LLM generates: ["LangGraph flow control?", "LangGraph state management?", ...]
>   → each variant searched against vectorstore
>   → results merged and deduplicated
>   → combined context → LLM answer
> ```

```python
# 18_multi_query_chain.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain.retrievers.multi_query import MultiQueryRetriever
from dotenv import load_dotenv

load_dotenv()

embedder = OpenAIEmbeddings(model="text-embedding-3-small")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

docs = [
    Document(page_content="LangGraph builds stateful agent workflows with cycles."),
    Document(page_content="LangGraph nodes are Python functions that transform state."),
    Document(page_content="LangGraph edges define the flow between nodes."),
    Document(page_content="LangGraph supports conditional routing based on state."),
    Document(page_content="LangGraph MemorySaver persists state for multi-turn conversations."),
]

vectorstore = FAISS.from_documents(docs, embedder)

# MultiQueryRetriever generates multiple search queries from one input
# catches more relevant docs than a single query
mq_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(k=2),
    llm=llm
)

prompt = ChatPromptTemplate.from_template("""
Use the context to answer the question concisely.

Context:
{context}

Question: {question}""")

def format_docs(docs):
    return "\n".join(d.page_content for d in docs)

chain = (
    {"context": mq_retriever | format_docs, "question": RunnablePassthrough()}
    | prompt | llm | StrOutputParser()
)

result = chain.invoke("How does LangGraph manage flow and state?")
print(result)
```

---

## L8 — Tools and Agents

### 19. Tools with @tool

> **📖 What's happening here?**
>
> **Tools** are functions that you give the LLM permission to call. Instead of the LLM only generating text, it can now *do things* — calculate math, look up data, convert text, etc.
>
> **The `@tool` decorator** turns a regular Python function into a LangChain Tool. The LLM reads the function's **docstring** to understand what the tool does and when to use it. So the docstring is not just for humans — it's the tool's instruction manual for the AI.
>
> **`llm.bind_tools(tools)`** — attaches the tools to the LLM. Now when you invoke this LLM, it knows these tools exist and can decide to call them.
>
> **Important:** At this stage, the LLM only *decides* to call a tool — it doesn't actually execute it. It returns a response with `tool_calls` that tells you "call this function with these arguments." You (or an agent loop) execute the actual function and send the result back.
>
> **Data flow:**
> ```
> "What is sqrt(144) + 20?"
>   → LLM sees the question + knows about calculator tool
>   → LLM says: "I need to call calculator(expression='sqrt(144)')"
>   → response.tool_calls = [{"name": "calculator", "args": {"expression": "sqrt(144)"}}]
>   → YOU execute calculator("sqrt(144)") → "12.0"
>   → (next step would be sending 12.0 back and asking LLM to continue)
> ```

```python
# 19_tools.py
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from dotenv import load_dotenv
import math
import json

load_dotenv()

@tool
def calculator(expression: str) -> str:
    """Evaluate a mathematical expression. E.g. '2 + 2', '10 * 5', 'sqrt(16)'."""
    try:
        safe_dict = {"sqrt": math.sqrt, "pi": math.pi, "abs": abs}
        result = eval(expression, {"__builtins__": {}}, safe_dict)
        return str(round(result, 6))
    except Exception as e:
        return f"Error: {e}"

@tool
def get_word_count(text: str) -> int:
    """Count the number of words in a text."""
    return len(text.split())

@tool
def to_uppercase(text: str) -> str:
    """Convert text to uppercase."""
    return text.upper()

@tool
def lookup_tech(name: str) -> str:
    """Look up information about a technology from the knowledge base."""
    kb = {
        "fastapi": "FastAPI is a modern Python web framework with automatic docs generation.",
        "langgraph": "LangGraph builds stateful agent workflows using graphs.",
        "langchain": "LangChain is the glue layer between your code and LLMs.",
        "neo4j": "Neo4j is a native graph database with the Cypher query language.",
    }
    return kb.get(name.lower(), f"No information found for '{name}'.")

# Bind tools to the LLM
tools = [calculator, get_word_count, to_uppercase, lookup_tech]
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).bind_tools(tools)

# The LLM decides which tools to call
response = llm.invoke([HumanMessage(content="What is sqrt(144) + 20?")])
print("Response type:", type(response))
if hasattr(response, "tool_calls") and response.tool_calls:
    print("Tool calls:")
    for tc in response.tool_calls:
        print(f"  {tc['name']}({tc['args']})")
```

---

### 20. ReAct Agent with Tools

> **📖 What's happening here?**
>
> This is the full **ReAct agent loop** — the most common pattern for AI agents. ReAct stands for **Re**ason + **Act**.
>
> **The loop works like this:**
> 1. **Reason** — LLM looks at the question and thinks: "What do I need? Which tool should I call?"
> 2. **Act** — LLM issues a tool call (says "call multiply(12, 8)")
> 3. **Observe** — You execute the tool and send the result back as a `ToolMessage`
> 4. **Repeat** — LLM sees the result and either calls another tool or gives the final answer
>
> **Key concepts:**
>
> - **`tool_map = {t.name: t for t in tools}`** — a dictionary mapping tool names to the actual functions. Used to look up and call the right function when the LLM requests it.
> - **`ToolMessage(content=result, tool_call_id=tc["id"])`** — sends the tool's output back to the LLM. The `tool_call_id` links this result to the specific tool call that requested it (important when multiple tools are called).
> - **`if not response.tool_calls: return response.content`** — the exit condition. When the LLM stops calling tools, it's done reasoning and gives the final answer.
> - **`max_steps=5`** — safety limit to prevent infinite loops.
>
> **Data flow for "What is 25 multiplied by 4, then add 100?":**
> ```
> Step 1: LLM → tool_call: multiply(25, 4)
>   → execute: multiply(25, 4) = 100
>   → ToolMessage("100")
> Step 2: LLM sees 100 → tool_call: add(100, 100)
>   → execute: add(100, 100) = 200
>   → ToolMessage("200")
> Step 3: LLM sees 200 → no more tool calls → "The answer is 200."
> ```

```python
# 20_react_agent.py
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, ToolMessage, BaseMessage
from dotenv import load_dotenv
from typing import List
import json

load_dotenv()

@tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

@tool
def add(a: float, b: float) -> float:
    """Add two numbers."""
    return a + b

@tool
def search_knowledge(query: str) -> str:
    """Search the knowledge base for information."""
    kb = {
        "python": "Python is an interpreted, high-level programming language.",
        "fastapi": "FastAPI is a modern Python web framework for building APIs.",
        "langchain": "LangChain is a framework for building LLM applications.",
    }
    for key, val in kb.items():
        if key in query.lower():
            return val
    return "No matching information found."

tools = [multiply, add, search_knowledge]
tool_map = {t.name: t for t in tools}

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0).bind_tools(tools)

def run_agent(user_input: str, max_steps: int = 5) -> str:
    """Simple ReAct loop: think → act → observe → repeat until done."""
    messages: List[BaseMessage] = [HumanMessage(content=user_input)]

    for step in range(max_steps):
        response = llm.invoke(messages)
        messages.append(response)

        # No tool calls = agent is done
        if not response.tool_calls:
            return response.content

        # Execute each tool call
        for tc in response.tool_calls:
            tool_fn = tool_map.get(tc["name"])
            if tool_fn:
                result = tool_fn.invoke(tc["args"])
                messages.append(ToolMessage(
                    content=str(result),
                    tool_call_id=tc["id"]
                ))
                print(f"  [Step {step+1}] {tc['name']}({tc['args']}) → {result}")

    return "Max steps reached."

print("=== Agent Demo ===\n")
questions = [
    "What is 25 multiplied by 4, then add 100?",
    "What is FastAPI?",
    "Multiply 12 by 8, then tell me what Python is.",
]

for q in questions:
    print(f"❓ {q}")
    print(f"💬 {run_agent(q)}\n")
```

---

## L9 — Callbacks and Observability

### 21. Custom Callbacks

> **📖 What's happening here?**
>
> **Callbacks** are hooks that fire automatically at different points in a chain's execution — like event listeners. They let you observe, log, time, or debug what's happening inside your chain without modifying the chain itself.
>
> Think of them like a sports commentator watching the game — they don't change the game, but they report everything that happens.
>
> **How to create a callback:**
> - Subclass `BaseCallbackHandler`
> - Override the methods for the events you care about
>
> **Key callback methods:**
>
> | Method | When it fires |
> |---|---|
> | `on_llm_start` | Just before LLM is called |
> | `on_llm_end` | After LLM returns a response |
> | `on_llm_error` | If the LLM call fails |
> | `on_chain_start` | When a chain begins |
> | `on_chain_end` | When a chain finishes |
>
> **`TimingCallback`** — measures how long each LLM call takes and tracks token usage. The `self.calls` list stores elapsed times for the `summary()` method.
>
> **`LoggingCallback`** — logs chain start/end events with input/output keys.
>
> **Two ways to attach callbacks:**
> 1. `ChatOpenAI(..., callbacks=[timer])` — attached permanently to the LLM
> 2. `chain.invoke(..., config={"callbacks": [timer]})` — attached for one specific call
>
> **Data flow:**
> ```
> chain.invoke({"topic": "Python"})
>   → [LoggingCallback fires: on_chain_start]
>   → [TimingCallback fires: on_llm_start → records start time]
>   → LLM generates response
>   → [TimingCallback fires: on_llm_end → calculates elapsed time, logs tokens]
>   → [LoggingCallback fires: on_chain_end]
>   → result returned to you
> ```

```python
# 21_callbacks.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.outputs import LLMResult
from dotenv import load_dotenv
import time

load_dotenv()

class TimingCallback(BaseCallbackHandler):
    """Track how long each LLM call takes."""
    def __init__(self):
        self.start_time = None
        self.calls = []

    def on_llm_start(self, serialized, prompts, **kwargs):
        self.start_time = time.time()
        print(f"[LLM] Starting call...")

    def on_llm_end(self, response: LLMResult, **kwargs):
        elapsed = time.time() - self.start_time
        tokens = response.llm_output.get("token_usage", {})
        self.calls.append(elapsed)
        print(f"[LLM] Completed in {elapsed:.2f}s | Tokens: {tokens}")

    def on_llm_error(self, error, **kwargs):
        print(f"[LLM] Error: {error}")

    def summary(self):
        if not self.calls:
            return "No calls made."
        return f"Total calls: {len(self.calls)}, Avg time: {sum(self.calls)/len(self.calls):.2f}s"


class LoggingCallback(BaseCallbackHandler):
    """Log all chain events."""
    def on_chain_start(self, serialized, inputs, **kwargs):
        print(f"[Chain] Starting with inputs: {list(inputs.keys())}")

    def on_chain_end(self, outputs, **kwargs):
        print(f"[Chain] Completed with keys: {list(outputs.keys())}")


timer = TimingCallback()
logger = LoggingCallback()

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0, callbacks=[timer])
chain = (
    ChatPromptTemplate.from_template("Tell me one fact about {topic}.")
    | llm
    | StrOutputParser()
)

topics = ["Python", "Rust", "LangChain"]
for topic in topics:
    result = chain.invoke({"topic": topic}, config={"callbacks": [timer]})
    print(f"Result: {result[:80]}...\n")

print(timer.summary())
```

---

## L10 — Full Pipeline

### 22. Full Document QA System — Everything Together

> **📖 What's happening here?**
>
> This is the **capstone exercise** — it combines every concept from L1 to L9 into a production-grade conversational document Q&A system. Let's break it into its building blocks:
>
> **🔧 Config block** — Constants at the top (`EMBED_MODEL`, `LLM_MODEL`, `CHUNK_SIZE`, etc.). Good practice: never hardcode magic numbers inside functions.
>
> **📄 Documents block** — Creates a knowledge base text file with info about 4 tools (LangChain, LangGraph, FastAPI, FastMCP). This is your "database" of knowledge.
>
> **📦 Indexing block (the RAG pipeline):**
> 1. `TextLoader` reads the file
> 2. `RecursiveCharacterTextSplitter` breaks it into 300-char chunks with 40-char overlap
> 3. `FAISS.from_documents` embeds all chunks and stores them
> 4. `MultiQueryRetriever` wraps the base retriever to generate multiple queries per search
>
> **🧠 Memory block** — `SimpleHistory` class + `store` dict, identical to drill #15. Each session ID gets its own isolated conversation history.
>
> **🔄 Rephrase chain** — A critical piece for multi-turn conversations. When you ask a follow-up like "How does it compare to the others?", "it" has no meaning without context. The rephrase chain reads your history and rewrites the follow-up as a standalone question: "How does FastMCP compare to LangChain, LangGraph, and FastAPI?"
>
> **💬 Answer chain** — Takes `{context}` (retrieved docs) + `{chat_history}` (past turns) + `{input}` (current question) and generates a grounded answer.
>
> **🔗 Full chain assembly:**
> ```
> retrieve_with_rephrase()
>   → if history exists: rephrase question first
>   → run retriever with (rephrased) question
>   → format docs as context string
>   ↓
> RunnablePassthrough.assign(context=...)
>   → adds context to the inputs dict
>   ↓
> answer_prompt | llm | StrOutputParser
>   → generates final answer
>   ↓
> RunnableWithMessageHistory
>   → auto-saves Human + AI messages to session store
> ```
>
> **Full data flow for the 5th question ("How does it compare to the other frameworks?"):**
> ```
> User: "How does it compare to the other frameworks?"
>   → rephrase_chain reads history (knows FastMCP was discussed)
>   → rephrases to: "How does FastMCP compare to LangChain, LangGraph, and FastAPI?"
>   → retriever finds relevant chunks about all 4 tools
>   → answer_prompt filled with context + history + rephrased question
>   → LLM answers: "FastMCP focuses on MCP server building, while LangChain..."
>   → saved to session history for future turns
> ```

```python
# 22_full_pipeline.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.chat_history import BaseChatMessageHistory
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage
from langchain.retrievers.multi_query import MultiQueryRetriever
from dotenv import load_dotenv
from typing import List
import os, time

load_dotenv()

# ── Config ─────────────────────────────────────────────
EMBED_MODEL = "text-embedding-3-small"
LLM_MODEL = "gpt-4o-mini"
CHUNK_SIZE = 300
CHUNK_OVERLAP = 40
TOP_K = 4

embedder = OpenAIEmbeddings(model=EMBED_MODEL)
llm = ChatOpenAI(model=LLM_MODEL, temperature=0)

# ── Documents ──────────────────────────────────────────
os.makedirs("kb", exist_ok=True)
with open("kb/ai_stack.txt", "w") as f:
    f.write("""LangChain
LangChain is a framework for building LLM applications.
LCEL uses the pipe operator to connect prompts, LLMs, and parsers.
RunnableWithMessageHistory adds session memory to any chain.
Tools let LLMs call external functions during reasoning.

LangGraph
LangGraph extends LangChain to support stateful agentic workflows.
Graphs have nodes (functions), edges (flow), and shared state.
MemorySaver persists conversation state between invocations.
Conditional edges route to different nodes based on state values.

FastAPI
FastAPI is a modern Python web framework built on Starlette.
Pydantic models validate request and response data automatically.
Depends() provides dependency injection for shared resources.
It generates interactive docs at /docs via OpenAPI automatically.

FastMCP
FastMCP builds MCP (Model Context Protocol) servers in Python.
Tools are registered with @mcp.tool() and exposed to AI clients.
Resources expose readable data via @mcp.resource() with URI templates.
Prompts are reusable templates registered with @mcp.prompt().
""")

# ── Indexing ───────────────────────────────────────────
loader = TextLoader("kb/ai_stack.txt")
raw_docs = loader.load()

splitter = RecursiveCharacterTextSplitter(
    chunk_size=CHUNK_SIZE,
    chunk_overlap=CHUNK_OVERLAP
)
chunks = splitter.split_documents(raw_docs)
vectorstore = FAISS.from_documents(chunks, embedder)

base_retriever = vectorstore.as_retriever(search_kwargs={"k": TOP_K})
retriever = MultiQueryRetriever.from_llm(retriever=base_retriever, llm=llm)

# ── Memory ─────────────────────────────────────────────
class SimpleHistory(BaseChatMessageHistory):
    def __init__(self):
        self.messages: List[BaseMessage] = []
    def add_messages(self, messages: List[BaseMessage]) -> None:
        self.messages.extend(messages)
    def clear(self) -> None:
        self.messages = []

store: dict[str, SimpleHistory] = {}

def get_session_history(session_id: str) -> SimpleHistory:
    if session_id not in store:
        store[session_id] = SimpleHistory()
    return store[session_id]

# ── Rephrase for history ────────────────────────────────
rephrase_prompt = ChatPromptTemplate.from_messages([
    ("system", "Rephrase the follow-up as a standalone question. Return only the question."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}")
])

rephrase_chain = rephrase_prompt | llm | StrOutputParser()

# ── Answer prompt ──────────────────────────────────────
answer_prompt = ChatPromptTemplate.from_messages([
    ("system", """You are a helpful AI engineering assistant.
Answer using ONLY the context below.
If the context doesn't contain the answer, say "I don't have that information."

Context:
{context}"""),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}")
])

def format_docs(docs: list) -> str:
    return "\n\n".join(d.page_content for d in docs)

# ── Full chain ─────────────────────────────────────────
def make_chain():
    def retrieve_with_rephrase(inputs: dict) -> str:
        history = inputs.get("chat_history", [])
        if history:
            standalone = rephrase_chain.invoke(inputs)
        else:
            standalone = inputs["input"]
        docs = retriever.invoke(standalone)
        return format_docs(docs)

    chain = (
        RunnablePassthrough.assign(context=retrieve_with_rephrase)
        | answer_prompt
        | llm
        | StrOutputParser()
    )

    return RunnableWithMessageHistory(
        chain,
        get_session_history,
        input_messages_key="input",
        history_messages_key="chat_history",
    )

qa_chain = make_chain()

# ── Run ────────────────────────────────────────────────
def ask(question: str, session_id: str = "default") -> str:
    start = time.time()
    result = qa_chain.invoke(
        {"input": question},
        config={"configurable": {"session_id": session_id}}
    )
    print(f"  ({time.time() - start:.2f}s)")
    return result

print("=== Full LangChain QA System ===\n")

session = "vishal-session"
questions = [
    "What is LangChain used for?",
    "How does FastAPI validate request data?",
    "What's the difference between LangChain and LangGraph?",
    "Tell me about FastMCP tools.",
    "How does it compare to the other frameworks?",  # tests follow-up with history
]

for q in questions:
    print(f"❓ {q}")
    print(f"💬 {ask(q, session)}\n")
```

---

## Bonus Drills

### B1. Build a classification chain
```
Classify any text as: positive, negative, or neutral
Use PydanticOutputParser to return {label, confidence, reason}
```

### B2. Build a translation chain
```
Detect language first, then translate to English
Two chained LCEL steps
```

### B3. Build a summarisation pipeline
```
Load a text file → split → embed → retrieve top 5 chunks → summarise
```

### B4. Build a multi-turn quiz bot
```
System: "You are a quiz master for {topic}"
Ask 3 questions one at a time
Remember what the user answered
Give a final score
```

### B5. Build a ReAct agent with 4 tools
```
search_knowledge, calculator, get_date, format_as_list
Run it on: "What are 3 benefits of FastAPI? Number them."
```

### B6. Full pipeline from scratch in one file
```
Load docs → split → embed → FAISS → MultiQueryRetriever → 
RephrasedHistory chain → RunnableWithMessageHistory → 
Ask 3 follow-up questions in the same session
No peeking at exercise 22.
```

---

## LangChain Cheat Sheet

```python
# Imports you use constantly
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableParallel, RunnableLambda
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
from langchain_core.tools import tool
from langchain_community.vectorstores import FAISS
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

# The three patterns you build everything from:

# 1. Basic chain
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"key": "value"})

# 2. RAG chain
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt | llm | StrOutputParser()
)

# 3. Chain with memory
chain_with_history = RunnableWithMessageHistory(
    chain, get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history",
)
chain_with_history.invoke(
    {"input": "hello"},
    config={"configurable": {"session_id": "abc"}}
)
```

---

## LangChain vs LangGraph — Pick the Right One

| Use LangChain when | Use LangGraph when |
|---|---|
| Linear pipeline: A → B → C | Cycles, loops, retries |
| One-shot question answering | Multi-step reasoning agents |
| RAG over documents | Human-in-the-loop workflows |
| Translation, summarisation, classification | Stateful multi-turn agents |
| Simple chatbot with history | Complex agent with tools + memory |
| You think in chains | You think in graphs |

In practice: **LangChain chains live inside LangGraph nodes.**

---

## Progress Tracker

| Level | Topic                                    | Done? | Time Taken |
|-------|------------------------------------------|-------|------------|
| L1    | LLMs, messages, batch, streaming         |  [ ]  |            |
| L2    | PromptTemplate, ChatPromptTemplate       |  [ ]  |            |
| L3    | Few-shot, StrOutputParser                |  [ ]  |            |
| L4    | JsonOutputParser, PydanticOutputParser   |  [ ]  |            |
| L5    | LCEL basics, sequential, branching       |  [ ]  |            |
| L6    | Conversation memory, RunnableWithHistory |  [ ]  |            |
| L7    | Load, split, embed, index                |  [ ]  |            |
| L8    | RetrievalQA, MultiQueryRetriever         |  [ ]  |            |
| L9    | Tools, ReAct agent, callbacks            |  [ ]  |            |
| L10   | Full document QA system                  |  [ ]  |            |
| Bonus | Classification, quiz bot, full pipeline  |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 22 (the full QA system)
> from memory — load docs, split, embed, FAISS, multi-query retriever, rephrase
> chain, answer chain, session history. Ask 5 questions in a single session
> including one follow-up that needs context from the previous answer.
> When the follow-up is answered correctly — you own LangChain.
