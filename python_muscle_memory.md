# Python Muscle Memory — 50+ Drills (L1 → L10)

> **How to use this file**
> - No AI. No autocomplete. Just you and a blank `.py` file.
> - Type every line. Do not copy-paste.
> - L1–L3 → warm up fingers (15–20 min)
> - L4–L6 → get back into logic (20–30 min)
> - L7–L9 → data structures + OOP (30–45 min)
> - L10 → mini CRUD app from scratch (45–60 min)
> - Target: complete all levels in 2 days. Repeat L7–L10 on day 3.

---

## L1 — Variables, Print, Input, Types

### 1. Hello Typed World
```python
name = input("Enter your name: ")
age = int(input("Enter your age: "))
height = float(input("Enter your height in cm: "))
print(f"Name: {name}, Age: {age}, Height: {height:.1f}cm")
print(f"Type of age: {type(age).__name__}")
```

### 2. Swap Without Temp
```python
a = 10
b = 20
a, b = b, a
print(f"a={a}, b={b}")
```

### 3. String Operations
```python
s = "  Hello, Vishal!  "
print(s.strip())
print(s.strip().lower())
print(s.strip().upper())
print(s.strip().replace("Vishal", "World"))
print(len(s.strip()))
print(s.strip().split(", "))
```

### 4. F-String Formatting
```python
price = 1999.5
qty = 3
discount = 0.10
total = price * qty
final = total - (total * discount)
print(f"Price: ₹{price:.2f}")
print(f"Qty: {qty}")
print(f"Total before discount: ₹{total:.2f}")
print(f"After {int(discount*100)}% discount: ₹{final:.2f}")
```

### 5. Type Casting Chain
```python
user_input = "42"
num = int(user_input)
num_float = float(num)
back_to_str = str(num_float)
print(type(user_input), type(num), type(num_float), type(back_to_str))
print(back_to_str)
```

---

## L2 — Conditionals, Loops, Range

### 6. FizzBuzz (classic warm-up)
```python
for i in range(1, 51):
    if i % 15 == 0:
        print("FizzBuzz")
    elif i % 3 == 0:
        print("Fizz")
    elif i % 5 == 0:
        print("Buzz")
    else:
        print(i)
```

### 7. While Loop — Countdown
```python
n = int(input("Enter a number: "))
while n > 0:
    print(n)
    n -= 1
print("Blastoff!")
```

### 8. Even/Odd Collector
```python
evens = []
odds = []
for i in range(1, 21):
    if i % 2 == 0:
        evens.append(i)
    else:
        odds.append(i)
print("Evens:", evens)
print("Odds:", odds)
print("Sum of evens:", sum(evens))
print("Sum of odds:", sum(odds))
```

### 9. Number Grade
```python
score = int(input("Enter score (0-100): "))
if score >= 90:
    grade = "A"
elif score >= 75:
    grade = "B"
elif score >= 60:
    grade = "C"
elif score >= 40:
    grade = "D"
else:
    grade = "F"
print(f"Grade: {grade}")
```

### 10. Multiplication Table
```python
n = int(input("Table for: "))
for i in range(1, 11):
    print(f"{n} x {i:2} = {n * i:3}")
```

### 11. Pattern Print
```python
rows = 5
for i in range(1, rows + 1):
    print("* " * i)

print()

for i in range(rows, 0, -1):
    print("* " * i)
```

---

## L3 — Functions Basics

### 12. Basic Functions
```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

print(greet("Vishal"))
print(greet("Vishal", "Namaste"))
```

### 13. Factorial (Iterative)
```python
def factorial(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result

for n in range(0, 8):
    print(f"{n}! = {factorial(n)}")
```

### 14. Factorial (Recursive)
```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

print(factorial(6))
```

### 15. Is Prime
```python
def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0:
            return False
    return True

primes = [n for n in range(2, 50) if is_prime(n)]
print(primes)
```

### 16. *args and **kwargs
```python
def stats(*numbers):
    if not numbers:
        return None
    return {
        "count": len(numbers),
        "sum": sum(numbers),
        "min": min(numbers),
        "max": max(numbers),
        "avg": sum(numbers) / len(numbers),
    }

result = stats(10, 20, 5, 40, 15)
for key, val in result.items():
    print(f"{key}: {val}")
```

### 17. Function Returning Multiple Values
```python
def divide(a, b):
    if b == 0:
        return None, "Division by zero"
    return a / b, None

result, error = divide(10, 3)
if error:
    print("Error:", error)
else:
    print(f"Result: {result:.4f}")
```

---

## L4 — Lists, Tuples, Comprehensions

### 18. List Operations Drill
```python
fruits = ["apple", "banana", "cherry", "date", "elderberry"]

# slice
print(fruits[1:4])

# reverse
print(fruits[::-1])

# sort
sorted_fruits = sorted(fruits)
print(sorted_fruits)

# filter by length
long_fruits = [f for f in fruits if len(f) > 5]
print(long_fruits)

# uppercase all
upper_fruits = [f.upper() for f in fruits]
print(upper_fruits)
```

### 19. List Comprehension — Squares and Filters
```python
squares = [x**2 for x in range(1, 11)]
even_squares = [x**2 for x in range(1, 11) if x % 2 == 0]
nested = [(x, y) for x in range(1, 4) for y in range(1, 4) if x != y]

print(squares)
print(even_squares)
print(nested)
```

### 20. Tuple Unpacking
```python
student = ("Vishal", 22, "IIT Madras", 8.5)
name, age, college, cgpa = student
print(f"{name} | Age: {age} | {college} | CGPA: {cgpa}")

# Named tuple feel using regular tuple
point = (3, 4)
distance = (point[0]**2 + point[1]**2) ** 0.5
print(f"Distance from origin: {distance}")
```

### 21. Stack Using List
```python
stack = []
stack.append(1)
stack.append(2)
stack.append(3)
print("Stack:", stack)
print("Pop:", stack.pop())
print("Stack after pop:", stack)
print("Peek:", stack[-1])
```

### 22. Queue Using List
```python
from collections import deque

queue = deque()
queue.append("first")
queue.append("second")
queue.append("third")
print("Queue:", queue)
print("Dequeue:", queue.popleft())
print("Queue after dequeue:", queue)
```

---

## L5 — Dictionaries and Sets

### 23. Dict CRUD Drill
```python
contacts = {}

# Create
contacts["Alice"] = {"phone": "9876543210", "email": "alice@example.com"}
contacts["Bob"] = {"phone": "9123456789", "email": "bob@example.com"}

# Read
print(contacts["Alice"])
print(contacts.get("Charlie", "Not found"))

# Update
contacts["Alice"]["phone"] = "9999999999"
print(contacts["Alice"])

# Delete
del contacts["Bob"]
print(contacts)

# Iterate
for name, info in contacts.items():
    print(f"{name}: {info['phone']}")
```

### 24. Word Frequency Counter
```python
text = "the quick brown fox jumps over the lazy dog the fox"
words = text.split()

freq = {}
for word in words:
    freq[word] = freq.get(word, 0) + 1

# Sort by frequency
sorted_freq = sorted(freq.items(), key=lambda x: x[1], reverse=True)
for word, count in sorted_freq:
    print(f"{word}: {count}")
```

### 25. Dict Comprehension
```python
numbers = range(1, 11)
squares_dict = {n: n**2 for n in numbers}
even_squares = {k: v for k, v in squares_dict.items() if k % 2 == 0}
print(squares_dict)
print(even_squares)
```

### 26. Set Operations
```python
python_devs = {"Alice", "Bob", "Charlie", "Dave"}
js_devs = {"Charlie", "Dave", "Eve", "Frank"}

print("Both:", python_devs & js_devs)          # intersection
print("Either:", python_devs | js_devs)        # union
print("Only Python:", python_devs - js_devs)   # difference
print("Not both:", python_devs ^ js_devs)      # symmetric difference
```

---

## L6 — File I/O and Error Handling

### 27. Write and Read a File
```python
filename = "notes.txt"

with open(filename, "w") as f:
    f.write("Line 1: Python muscle memory\n")
    f.write("Line 2: Keep coding daily\n")
    f.write("Line 3: Ship something!\n")

with open(filename, "r") as f:
    for i, line in enumerate(f, 1):
        print(f"{i}: {line.strip()}")
```

### 28. Append and Count Lines
```python
filename = "log.txt"

entries = ["Login: Vishal", "Action: viewed dashboard", "Logout: Vishal"]

with open(filename, "w") as f:
    for entry in entries:
        f.write(entry + "\n")

with open(filename, "a") as f:
    f.write("Login: Admin\n")

with open(filename, "r") as f:
    lines = f.readlines()

print(f"Total log entries: {len(lines)}")
for line in lines:
    print(line.strip())
```

### 29. Try/Except/Finally
```python
def safe_divide(a, b):
    try:
        result = a / b
        return result
    except ZeroDivisionError:
        print("Error: Cannot divide by zero")
        return None
    except TypeError:
        print("Error: Both arguments must be numbers")
        return None
    finally:
        print("safe_divide called")

print(safe_divide(10, 2))
print(safe_divide(10, 0))
print(safe_divide(10, "x"))
```

### 30. Custom Exception
```python
class ValidationError(Exception):
    def __init__(self, field, message):
        self.field = field
        self.message = message
        super().__init__(f"[{field}] {message}")

def validate_age(age):
    if not isinstance(age, int):
        raise ValidationError("age", "Must be an integer")
    if age < 0 or age > 120:
        raise ValidationError("age", "Must be between 0 and 120")
    return True

for test in [25, -1, 200, "old"]:
    try:
        validate_age(test)
        print(f"Age {test} is valid")
    except ValidationError as e:
        print(f"Invalid: {e}")
```

---

## L7 — OOP: Classes and Objects

### 31. Basic Class
```python
class Dog:
    species = "Canis lupus familiaris"

    def __init__(self, name, breed, age):
        self.name = name
        self.breed = breed
        self.age = age

    def bark(self):
        return f"{self.name} says: Woof!"

    def info(self):
        return f"{self.name} ({self.breed}), {self.age} years old"

    def __str__(self):
        return self.info()

    def __repr__(self):
        return f"Dog(name={self.name!r}, breed={self.breed!r}, age={self.age})"

d1 = Dog("Bruno", "Labrador", 3)
d2 = Dog("Milo", "Beagle", 5)
print(d1)
print(d2.bark())
print(repr(d1))
print(Dog.species)
```

### 32. Class Methods and Static Methods
```python
class Circle:
    PI = 3.14159

    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return Circle.PI * self.radius ** 2

    def perimeter(self):
        return 2 * Circle.PI * self.radius

    @classmethod
    def from_diameter(cls, diameter):
        return cls(diameter / 2)

    @staticmethod
    def is_valid_radius(r):
        return r > 0

    def __str__(self):
        return f"Circle(r={self.radius})"

c1 = Circle(5)
c2 = Circle.from_diameter(10)
print(c1, f"Area={c1.area():.2f}", f"Perimeter={c1.perimeter():.2f}")
print(c2)
print(Circle.is_valid_radius(-3))
```

### 33. Inheritance
```python
class Animal:
    def __init__(self, name, sound):
        self.name = name
        self.sound = sound

    def speak(self):
        return f"{self.name} says {self.sound}"

    def __str__(self):
        return f"{self.__class__.__name__}({self.name})"

class Dog(Animal):
    def __init__(self, name):
        super().__init__(name, "Woof")

    def fetch(self):
        return f"{self.name} fetches the ball!"

class Cat(Animal):
    def __init__(self, name):
        super().__init__(name, "Meow")

    def purr(self):
        return f"{self.name} purrs..."

animals = [Dog("Bruno"), Cat("Whiskers"), Dog("Max")]
for animal in animals:
    print(animal.speak())
    if isinstance(animal, Dog):
        print(animal.fetch())
```

### 34. Properties and Encapsulation
```python
class BankAccount:
    def __init__(self, owner, initial_balance=0):
        self.owner = owner
        self._balance = initial_balance
        self._transactions = []

    @property
    def balance(self):
        return self._balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self._balance += amount
        self._transactions.append(("deposit", amount))
        return self

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount
        self._transactions.append(("withdraw", amount))
        return self

    def statement(self):
        print(f"\n--- Statement: {self.owner} ---")
        for ttype, amount in self._transactions:
            print(f"  {ttype:10}: ₹{amount:.2f}")
        print(f"  Balance   : ₹{self._balance:.2f}")

acc = BankAccount("Vishal", 1000)
acc.deposit(500).deposit(200).withdraw(300)
acc.statement()
```

---

## L8 — Iterators, Generators, Decorators

### 35. Custom Iterator
```python
class Counter:
    def __init__(self, start, stop, step=1):
        self.current = start
        self.stop = stop
        self.step = step

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.stop:
            raise StopIteration
        val = self.current
        self.current += self.step
        return val

for num in Counter(0, 10, 2):
    print(num, end=" ")
print()
```

### 36. Generator Function
```python
def fibonacci(limit):
    a, b = 0, 1
    while a <= limit:
        yield a
        a, b = b, a + b

print(list(fibonacci(100)))

# Generator expression
squares_gen = (x**2 for x in range(10))
print(list(squares_gen))
```

### 37. Decorator — Timer
```python
import time
import functools

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {(end - start) * 1000:.3f}ms")
        return result
    return wrapper

@timer
def slow_sum(n):
    return sum(range(n))

print(slow_sum(1_000_000))
```

### 38. Decorator — Logger
```python
import functools

def logger(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper

@logger
def add(a, b):
    return a + b

@logger
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

add(3, 4)
greet("Vishal", greeting="Namaste")
```

---

## L9 — Working with JSON and Data

### 39. JSON Serialize / Deserialize
```python
import json

data = {
    "name": "Vishal",
    "skills": ["Python", "LangChain", "Neo4j"],
    "age": 22,
    "active": True,
    "score": 9.1
}

# Serialize
json_str = json.dumps(data, indent=2)
print(json_str)

# Deserialize
parsed = json.loads(json_str)
print(type(parsed))
print(parsed["skills"][1])
```

### 40. JSON File I/O
```python
import json

filename = "user_data.json"

users = [
    {"id": 1, "name": "Alice", "role": "admin"},
    {"id": 2, "name": "Bob", "role": "viewer"},
    {"id": 3, "name": "Charlie", "role": "editor"},
]

with open(filename, "w") as f:
    json.dump(users, f, indent=2)

with open(filename, "r") as f:
    loaded = json.load(f)

for user in loaded:
    print(f"[{user['id']}] {user['name']} — {user['role']}")
```

### 41. Sorting Dicts by Key
```python
students = [
    {"name": "Alice", "marks": 88},
    {"name": "Vishal", "marks": 95},
    {"name": "Bob", "marks": 72},
    {"name": "Charlie", "marks": 88},
]

# Sort by marks descending, then name ascending
ranked = sorted(students, key=lambda s: (-s["marks"], s["name"]))

for rank, student in enumerate(ranked, 1):
    print(f"{rank}. {student['name']} — {student['marks']}")
```

### 42. defaultdict and Counter
```python
from collections import defaultdict, Counter

# defaultdict
word_positions = defaultdict(list)
sentence = "the cat sat on the mat near the cat"
for i, word in enumerate(sentence.split()):
    word_positions[word].append(i)

for word, positions in sorted(word_positions.items()):
    print(f"{word}: {positions}")

print()

# Counter
counts = Counter(sentence.split())
print(counts.most_common(3))
```

---

## L10 — Mini CRUD App (No AI, No Libraries)

### 43. In-Memory Task Manager (CRUD)
```python
# Task Manager — pure Python, no frameworks

tasks = {}
next_id = 1

def create_task(title, priority="medium"):
    global next_id
    task = {
        "id": next_id,
        "title": title,
        "priority": priority,
        "done": False,
    }
    tasks[next_id] = task
    next_id += 1
    return task

def read_task(task_id):
    return tasks.get(task_id, None)

def list_tasks(filter_done=None):
    result = tasks.values()
    if filter_done is not None:
        result = [t for t in result if t["done"] == filter_done]
    return list(result)

def update_task(task_id, **kwargs):
    if task_id not in tasks:
        return None
    allowed = {"title", "priority", "done"}
    for key, val in kwargs.items():
        if key in allowed:
            tasks[task_id][key] = val
    return tasks[task_id]

def delete_task(task_id):
    return tasks.pop(task_id, None)

def print_task(task):
    status = "✓" if task["done"] else "○"
    print(f"  [{status}] #{task['id']} | {task['priority'].upper():6} | {task['title']}")

# --- Demo ---
create_task("Set up Python environment", "high")
create_task("Complete L1-L9 drills", "high")
create_task("Build CRUD app", "medium")
create_task("Push to GitHub", "low")

print("\n=== All Tasks ===")
for t in list_tasks():
    print_task(t)

update_task(1, done=True)
update_task(2, done=True)
update_task(3, priority="high")

print("\n=== After Updates ===")
for t in list_tasks():
    print_task(t)

delete_task(4)

print("\n=== Pending Only ===")
for t in list_tasks(filter_done=False):
    print_task(t)

print("\n=== Completed Only ===")
for t in list_tasks(filter_done=True):
    print_task(t)
```

### 44. Persist CRUD to JSON
```python
import json
import os

DB_FILE = "tasks_db.json"

def load_db():
    if os.path.exists(DB_FILE):
        with open(DB_FILE, "r") as f:
            return json.load(f)
    return {"tasks": {}, "next_id": 1}

def save_db(db):
    with open(DB_FILE, "w") as f:
        json.dump(db, f, indent=2)

def create(db, title, priority="medium"):
    task_id = str(db["next_id"])
    db["tasks"][task_id] = {
        "id": task_id,
        "title": title,
        "priority": priority,
        "done": False,
    }
    db["next_id"] += 1
    save_db(db)
    return db["tasks"][task_id]

def list_all(db):
    return list(db["tasks"].values())

def complete(db, task_id):
    task_id = str(task_id)
    if task_id in db["tasks"]:
        db["tasks"][task_id]["done"] = True
        save_db(db)
        return db["tasks"][task_id]
    return None

def delete(db, task_id):
    task_id = str(task_id)
    task = db["tasks"].pop(task_id, None)
    if task:
        save_db(db)
    return task

# --- Demo ---
db = load_db()
create(db, "Write extraction prompt")
create(db, "Test Neo4j schema", "high")
create(db, "Push image memory demo", "high")

for t in list_all(db):
    print(t)

complete(db, 1)
print("\nAfter completing task 1:")
for t in list_all(db):
    status = "DONE" if t["done"] else "TODO"
    print(f"  [{status}] {t['title']}")
```

### 45. CLI Menu Loop
```python
def show_menu():
    print("\n=== Task Manager ===")
    print("1. Add Task")
    print("2. List All Tasks")
    print("3. Complete Task")
    print("4. Delete Task")
    print("5. Exit")

tasks = {}
next_id = [1]

def add(title):
    tid = next_id[0]
    tasks[tid] = {"id": tid, "title": title, "done": False}
    next_id[0] += 1
    print(f"Added task #{tid}")

def show_all():
    if not tasks:
        print("No tasks.")
        return
    for t in tasks.values():
        mark = "✓" if t["done"] else "○"
        print(f"  [{mark}] #{t['id']} {t['title']}")

def complete(tid):
    if tid in tasks:
        tasks[tid]["done"] = True
        print(f"Task #{tid} marked complete.")
    else:
        print("Task not found.")

def delete(tid):
    if tasks.pop(tid, None):
        print(f"Task #{tid} deleted.")
    else:
        print("Task not found.")

while True:
    show_menu()
    choice = input("Choice: ").strip()
    if choice == "1":
        title = input("Task title: ").strip()
        if title:
            add(title)
    elif choice == "2":
        show_all()
    elif choice == "3":
        try:
            tid = int(input("Task ID to complete: "))
            complete(tid)
        except ValueError:
            print("Invalid ID")
    elif choice == "4":
        try:
            tid = int(input("Task ID to delete: "))
            delete(tid)
        except ValueError:
            print("Invalid ID")
    elif choice == "5":
        print("Bye!")
        break
    else:
        print("Invalid choice, try again.")
```

---

## Bonus Drills — Fill in the Blanks Style

Practice these without looking at the answers above.

### B1. Write a function `flatten(nested_list)` that flattens one level of nesting.
```
Input:  [[1, 2], [3, 4], [5]]
Output: [1, 2, 3, 4, 5]
```

### B2. Write `most_frequent(lst)` — returns the most common element.
```
Input:  [1, 3, 3, 2, 1, 3]
Output: 3
```

### B3. Write a `Stack` class with `push`, `pop`, `peek`, `is_empty`, `size`.

### B4. Write a `memoize(func)` decorator that caches results.

### B5. Write a generator `read_chunks(filename, chunk_size)` that yields lines in chunks.

### B6. Write a class `Student` with name, marks list, and methods: `average()`, `highest()`, `lowest()`, `__str__`.

### B7. Write `rotate_list(lst, k)` — rotates list left by k positions.
```
Input:  [1, 2, 3, 4, 5], k=2
Output: [3, 4, 5, 1, 2]
```

### B8. Write a function `merge_dicts(*dicts)` — merges all dicts, later keys win.

---

## Progress Tracker

| Level | Topic                          | Done? | Time Taken |
|-------|-------------------------------|-------|------------|
| L1    | Variables, Print, Types        |  [ ]  |            |
| L2    | Conditionals, Loops            |  [ ]  |            |
| L3    | Functions                      |  [ ]  |            |
| L4    | Lists, Comprehensions          |  [ ]  |            |
| L5    | Dicts, Sets                    |  [ ]  |            |
| L6    | File I/O, Error Handling       |  [ ]  |            |
| L7    | OOP                            |  [ ]  |            |
| L8    | Generators, Decorators         |  [ ]  |            |
| L9    | JSON, Data                     |  [ ]  |            |
| L10   | Mini CRUD App                  |  [ ]  |            |
| Bonus | Fill in the Blanks             |  [ ]  |            |

---

> **Final challenge:** After finishing all levels, open a blank file and build the L10 CLI Task Manager (exercise 45) completely from memory. No peeking. That's your milestone — fingers back in shape.
