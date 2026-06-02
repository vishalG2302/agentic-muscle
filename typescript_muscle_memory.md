# TypeScript Muscle Memory — 50+ Drills (L1 → L10)

> **Rules — same as Python and C++ sheets**
> - No AI. No autocomplete. Type every line.
> - Run after every exercise: `npx ts-node filename.ts`
> - Or compile + run: `tsc filename.ts && node filename.js`
> - L1–L3 → types and syntax warm-up (20–30 min)
> - L4–L6 → interfaces, generics, functions (30–40 min)
> - L7–L8 → classes + advanced types (40–50 min)
> - L9–L10 → async/await + mini CRUD app (45–60 min)
> - Target: 3 days. Day 3 = blank file, build CRUD from memory.

---

## L1 — Variables, Types, Type Annotations

### 1. Basic Type Annotations
```typescript
let name: string = "Vishal";
let age: number = 22;
let height: number = 5.9;
let isActive: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;

console.log(`Name: ${name}`);
console.log(`Age: ${age}`);
console.log(`Height: ${height}`);
console.log(`Active: ${isActive}`);
console.log(`Null: ${nothing}`);
console.log(`Undefined: ${notDefined}`);
```

### 2. Type Inference
```typescript
// TypeScript infers the type — you don't always need to annotate
let city = "Pune";           // inferred: string
let year = 2026;             // inferred: number
let isPremium = false;       // inferred: boolean

// city = 42;                // Error: Type 'number' is not assignable to type 'string'

console.log(typeof city, typeof year, typeof isPremium);
```

### 3. Union Types
```typescript
let id: number | string;

id = 101;
console.log("ID as number:", id);

id = "USR-101";
console.log("ID as string:", id);

function printId(id: number | string): void {
    if (typeof id === "string") {
        console.log("String ID:", id.toUpperCase());
    } else {
        console.log("Number ID:", id.toFixed(0));
    }
}

printId(101);
printId("usr-202");
```

### 4. Literal Types
```typescript
type Direction = "north" | "south" | "east" | "west";
type Status = "active" | "inactive" | "pending";
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

function move(direction: Direction): void {
    console.log(`Moving ${direction}`);
}

function setStatus(status: Status): void {
    console.log(`Status set to: ${status}`);
}

move("north");
move("east");
setStatus("active");

let roll: DiceRoll = 4;
console.log("Rolled:", roll);
```

### 5. Type Aliases
```typescript
type UserID = string;
type Score = number;
type Pair = [string, number];

const userId: UserID = "USR-001";
const score: Score = 98.5;
const entry: Pair = ["Vishal", 95];

console.log(userId, score, entry);

// Tuple — fixed length, fixed types
const rgb: [number, number, number] = [255, 128, 0];
const [r, g, b] = rgb;
console.log(`RGB: r=${r} g=${g} b=${b}`);
```

---

## L2 — Functions and Type Safety

### 6. Typed Functions
```typescript
function add(a: number, b: number): number {
    return a + b;
}

function greet(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}!`;
}

function logMessage(message: string): void {
    console.log(`[LOG] ${message}`);
}

console.log(add(3, 4));
console.log(greet("Vishal"));
console.log(greet("Vishal", "Namaste"));
logMessage("App started");
```

### 7. Optional and Default Parameters
```typescript
function createUser(
    name: string,
    age: number,
    role: string = "viewer",
    email?: string
): string {
    const emailPart = email ? ` | ${email}` : "";
    return `${name} | Age: ${age} | Role: ${role}${emailPart}`;
}

console.log(createUser("Vishal", 22));
console.log(createUser("Alice", 30, "admin"));
console.log(createUser("Bob", 25, "editor", "bob@example.com"));
```

### 8. Rest Parameters
```typescript
function sum(...numbers: number[]): number {
    return numbers.reduce((acc, n) => acc + n, 0);
}

function stats(...numbers: number[]): object {
    if (numbers.length === 0) return {};
    return {
        count: numbers.length,
        sum: numbers.reduce((a, b) => a + b, 0),
        min: Math.min(...numbers),
        max: Math.max(...numbers),
        avg: numbers.reduce((a, b) => a + b, 0) / numbers.length,
    };
}

console.log(sum(1, 2, 3, 4, 5));
console.log(stats(10, 20, 5, 40, 15));
```

### 9. Function Types
```typescript
// Define a function type
type MathOp = (a: number, b: number) => number;
type Predicate<T> = (value: T) => boolean;
type Transformer<T, U> = (value: T) => U;

const multiply: MathOp = (a, b) => a * b;
const isEven: Predicate<number> = (n) => n % 2 === 0;
const toString: Transformer<number, string> = (n) => n.toString();

console.log(multiply(4, 5));
console.log(isEven(8), isEven(7));
console.log(toString(42));

// Higher order function
function applyTwice(fn: MathOp, a: number, b: number): number {
    return fn(fn(a, b), b);
}

console.log(applyTwice(multiply, 2, 3)); // (2*3)*3 = 18
```

### 10. Arrow Functions and Callbacks
```typescript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

const evens = numbers.filter(n => n % 2 === 0);
const squares = numbers.map(n => n * n);
const total = numbers.reduce((acc, n) => acc + n, 0);

console.log("Evens:", evens);
console.log("Squares:", squares);
console.log("Total:", total);

// Chaining
const result = numbers
    .filter(n => n % 2 === 0)
    .map(n => n * n)
    .reduce((acc, n) => acc + n, 0);

console.log("Sum of squares of evens:", result);
```

---

## L3 — Arrays and Objects

### 11. Typed Arrays
```typescript
const scores: number[] = [88, 95, 72, 61, 90];
const names: string[] = ["Alice", "Vishal", "Bob", "Charlie"];
const flags: Array<boolean> = [true, false, true, true];

// Array methods
console.log("Sorted:", [...scores].sort((a, b) => b - a));
console.log("Max:", Math.max(...scores));
console.log("Min:", Math.min(...scores));
console.log("Avg:", scores.reduce((a, b) => a + b, 0) / scores.length);

// Includes and indexOf
console.log("Has 95:", scores.includes(95));
console.log("Index of 72:", scores.indexOf(72));
```

### 12. Readonly Arrays and Tuples
```typescript
const immutable: readonly number[] = [1, 2, 3, 4, 5];
// immutable.push(6);  // Error: Property 'push' does not exist on type 'readonly number[]'

console.log("Readonly array:", immutable);

// Labeled tuples
type Point = [x: number, y: number];
type RGB = [red: number, green: number, blue: number];

const origin: Point = [0, 0];
const color: RGB = [255, 128, 0];

const [x, y] = origin;
const [red, green, blue] = color;
console.log(`Point: (${x}, ${y})`);
console.log(`Color: rgb(${red}, ${green}, ${blue})`);
```

### 13. Object Types
```typescript
type Person = {
    name: string;
    age: number;
    email?: string;       // optional
    readonly id: string;  // cannot be reassigned
};

const user: Person = {
    id: "USR-001",
    name: "Vishal",
    age: 22,
};

user.name = "Vishal G";   // ok
// user.id = "USR-002";   // Error: Cannot assign to 'id' because it is a read-only property

const user2: Person = { id: "USR-002", name: "Alice", age: 30, email: "alice@example.com" };

console.log(user);
console.log(user2);
```

### 14. Spread and Destructuring
```typescript
// Object destructuring
const { name, age, email = "N/A" } = { name: "Vishal", age: 22 } as any;
console.log(name, age, email);

// Rename while destructuring
const config = { host: "localhost", port: 5432, db: "mydb" };
const { host: dbHost, port: dbPort, db: dbName } = config;
console.log(dbHost, dbPort, dbName);

// Spread
const defaults = { theme: "dark", lang: "en", fontSize: 14 };
const userPrefs = { lang: "hi", fontSize: 16 };
const merged = { ...defaults, ...userPrefs };
console.log(merged);

// Array spread
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const combined = [...arr1, ...arr2];
console.log(combined);
```

---

## L4 — Interfaces

### 15. Basic Interface
```typescript
interface User {
    id: number;
    name: string;
    email: string;
    role: "admin" | "editor" | "viewer";
    createdAt: Date;
}

function printUser(user: User): void {
    console.log(`[${user.role.toUpperCase()}] ${user.name} <${user.email}>`);
}

const user: User = {
    id: 1,
    name: "Vishal",
    email: "vishal@example.com",
    role: "admin",
    createdAt: new Date(),
};

printUser(user);
```

### 16. Interface Extension
```typescript
interface Animal {
    name: string;
    sound: string;
    speak(): string;
}

interface Pet extends Animal {
    owner: string;
    isIndoor: boolean;
}

interface ServiceAnimal extends Pet {
    serviceType: string;
    certificationId: string;
}

const pet: Pet = {
    name: "Bruno",
    sound: "Woof",
    owner: "Vishal",
    isIndoor: true,
    speak() {
        return `${this.name} says ${this.sound}`;
    }
};

console.log(pet.speak());
```

### 17. Interface for Functions and Indexable Types
```typescript
// Function interface
interface MathOperation {
    (a: number, b: number): number;
    description: string;
}

const multiply: MathOperation = (a, b) => a * b;
multiply.description = "Multiplies two numbers";

console.log(multiply(4, 5));
console.log(multiply.description);

// Index signature — dict-like object
interface StringMap {
    [key: string]: string;
}

interface ScoreMap {
    [studentName: string]: number;
}

const translations: StringMap = { hello: "namaste", bye: "alvida" };
const scores: ScoreMap = { Vishal: 95, Alice: 88, Bob: 72 };

console.log(translations["hello"]);
console.log(scores["Vishal"]);
```

### 18. Interface vs Type — when to use what
```typescript
// Interface: prefer for object shapes, especially when extending
interface Vehicle {
    brand: string;
    speed: number;
    drive(): void;
}

interface ElectricVehicle extends Vehicle {
    batteryLevel: number;
    charge(): void;
}

// Type: prefer for unions, intersections, primitives, tuples
type StringOrNumber = string | number;
type AdminUser = User & { permissions: string[] };
type Coordinates = [number, number, number?]; // optional z

interface User {
    id: number;
    name: string;
}

const adminUser: AdminUser = {
    id: 1,
    name: "Vishal",
    permissions: ["read", "write", "delete"]
};

console.log(adminUser);
```

---

## L5 — Generics

### 19. Generic Functions
```typescript
function identity<T>(value: T): T {
    return value;
}

function firstElement<T>(arr: T[]): T | undefined {
    return arr[0];
}

function pair<T, U>(first: T, second: U): [T, U] {
    return [first, second];
}

console.log(identity(42));
console.log(identity("hello"));
console.log(identity(true));

console.log(firstElement([1, 2, 3]));
console.log(firstElement(["a", "b", "c"]));

console.log(pair("Vishal", 95));
console.log(pair(true, 3.14));
```

### 20. Generic Constraints
```typescript
interface HasLength {
    length: number;
}

function logLength<T extends HasLength>(value: T): T {
    console.log(`Length: ${value.length}`);
    return value;
}

logLength("Hello");
logLength([1, 2, 3, 4]);
logLength({ length: 10, name: "custom" });

// Constraint with keyof
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

const user = { id: 1, name: "Vishal", age: 22 };
console.log(getProperty(user, "name"));
console.log(getProperty(user, "age"));
// getProperty(user, "email");  // Error: Argument of type '"email"' is not assignable
```

### 21. Generic Interfaces and Classes
```typescript
interface Repository<T> {
    findById(id: number): T | undefined;
    findAll(): T[];
    save(item: T): void;
    delete(id: number): boolean;
}

interface Entity {
    id: number;
}

class InMemoryRepo<T extends Entity> implements Repository<T> {
    private items: Map<number, T> = new Map();

    findById(id: number): T | undefined {
        return this.items.get(id);
    }

    findAll(): T[] {
        return Array.from(this.items.values());
    }

    save(item: T): void {
        this.items.set(item.id, item);
    }

    delete(id: number): boolean {
        return this.items.delete(id);
    }
}

interface Task extends Entity {
    title: string;
    done: boolean;
}

const repo = new InMemoryRepo<Task>();
repo.save({ id: 1, title: "Complete TS drills", done: false });
repo.save({ id: 2, title: "Push to GitHub", done: false });

console.log(repo.findAll());
console.log(repo.findById(1));
repo.delete(2);
console.log(repo.findAll());
```

---

## L6 — Utility Types

### 22. Partial, Required, Readonly
```typescript
interface User {
    id: number;
    name: string;
    email: string;
    age: number;
}

// Partial — all fields optional (useful for update functions)
function updateUser(id: number, updates: Partial<User>): void {
    console.log(`Updating user ${id} with:`, updates);
}

updateUser(1, { name: "Vishal G" });
updateUser(2, { email: "new@email.com", age: 23 });

// Required — all fields required
type RequiredUser = Required<User>;

// Readonly — no field can be changed
type FrozenUser = Readonly<User>;
const frozen: FrozenUser = { id: 1, name: "Alice", email: "a@b.com", age: 30 };
// frozen.name = "Bob";  // Error

console.log(frozen);
```

### 23. Pick, Omit, Record
```typescript
interface Product {
    id: number;
    name: string;
    price: number;
    category: string;
    description: string;
    stock: number;
}

// Pick — take only some fields
type ProductCard = Pick<Product, "id" | "name" | "price">;

// Omit — exclude some fields
type ProductPreview = Omit<Product, "stock" | "description">;

// Record — key-value map type
type CategoryCount = Record<string, number>;
type UserRoleMap = Record<"admin" | "editor" | "viewer", string[]>;

const counts: CategoryCount = { electronics: 5, books: 12, clothing: 8 };
const roles: UserRoleMap = {
    admin: ["Vishal"],
    editor: ["Alice", "Bob"],
    viewer: ["Charlie", "Dave"],
};

console.log(counts);
console.log(roles);
```

### 24. ReturnType, Parameters, Exclude, Extract
```typescript
function fetchUser(id: number, includeEmail: boolean): { id: number; name: string } {
    return { id, name: "Vishal" };
}

// Extract return type of a function
type FetchUserReturn = ReturnType<typeof fetchUser>;
// type FetchUserReturn = { id: number; name: string }

// Extract parameter types
type FetchUserParams = Parameters<typeof fetchUser>;
// type FetchUserParams = [id: number, includeEmail: boolean]

// Exclude from union
type AllRoles = "admin" | "editor" | "viewer" | "banned";
type ActiveRoles = Exclude<AllRoles, "banned">;
// type ActiveRoles = "admin" | "editor" | "viewer"

// Extract from union
type StringOrNumber = string | number | boolean | null;
type OnlyStringOrNumber = Extract<StringOrNumber, string | number>;
// type OnlyStringOrNumber = string | number

const role: ActiveRoles = "admin";
console.log(role);
```

---

## L7 — Classes

### 25. Basic Class with Access Modifiers
```typescript
class BankAccount {
    private _balance: number;
    private _transactions: string[] = [];
    readonly id: string;
    protected owner: string;

    constructor(owner: string, initialBalance: number = 0) {
        this.owner = owner;
        this._balance = initialBalance;
        this.id = `ACC-${Date.now()}`;
    }

    get balance(): number {
        return this._balance;
    }

    deposit(amount: number): this {
        if (amount <= 0) throw new Error("Amount must be positive");
        this._balance += amount;
        this._transactions.push(`+${amount}`);
        return this;
    }

    withdraw(amount: number): this {
        if (amount > this._balance) throw new Error("Insufficient funds");
        this._balance -= amount;
        this._transactions.push(`-${amount}`);
        return this;
    }

    statement(): void {
        console.log(`\n--- ${this.owner}'s Account ---`);
        this._transactions.forEach(t => console.log(" ", t));
        console.log(`  Balance: ${this._balance}`);
    }
}

const acc = new BankAccount("Vishal", 1000);
acc.deposit(500).deposit(200).withdraw(300);
acc.statement();
console.log("Balance:", acc.balance);
```

### 26. Inheritance and Polymorphism
```typescript
abstract class Shape {
    abstract area(): number;
    abstract perimeter(): number;
    abstract name(): string;

    describe(): void {
        console.log(`${this.name()} | Area: ${this.area().toFixed(2)} | Perimeter: ${this.perimeter().toFixed(2)}`);
    }
}

class Circle extends Shape {
    constructor(private radius: number) { super(); }

    area(): number { return Math.PI * this.radius ** 2; }
    perimeter(): number { return 2 * Math.PI * this.radius; }
    name(): string { return "Circle"; }
}

class Rectangle extends Shape {
    constructor(private w: number, private h: number) { super(); }

    area(): number { return this.w * this.h; }
    perimeter(): number { return 2 * (this.w + this.h); }
    name(): string { return "Rectangle"; }
}

const shapes: Shape[] = [new Circle(5), new Rectangle(4, 6)];
shapes.forEach(s => s.describe());
```

### 27. Implementing Interfaces
```typescript
interface Printable {
    print(): void;
}

interface Serializable {
    serialize(): string;
    deserialize(data: string): void;
}

class Task implements Printable, Serializable {
    constructor(
        public id: number,
        public title: string,
        public done: boolean = false
    ) {}

    print(): void {
        const mark = this.done ? "✓" : "○";
        console.log(`[${mark}] #${this.id} ${this.title}`);
    }

    serialize(): string {
        return JSON.stringify({ id: this.id, title: this.title, done: this.done });
    }

    deserialize(data: string): void {
        const parsed = JSON.parse(data);
        this.id = parsed.id;
        this.title = parsed.title;
        this.done = parsed.done;
    }
}

const task = new Task(1, "Complete TypeScript drills");
task.print();
const json = task.serialize();
console.log("Serialized:", json);

const task2 = new Task(0, "");
task2.deserialize(json);
task2.print();
```

### 28. Static Members and Singleton Pattern
```typescript
class Config {
    private static instance: Config;
    private settings: Record<string, string> = {};

    private constructor() {
        this.settings["theme"] = "dark";
        this.settings["lang"] = "en";
    }

    static getInstance(): Config {
        if (!Config.instance) {
            Config.instance = new Config();
        }
        return Config.instance;
    }

    get(key: string): string | undefined {
        return this.settings[key];
    }

    set(key: string, value: string): void {
        this.settings[key] = value;
    }

    getAll(): Record<string, string> {
        return { ...this.settings };
    }
}

const config1 = Config.getInstance();
const config2 = Config.getInstance();

console.log(config1 === config2);  // true — same instance

config1.set("lang", "hi");
console.log(config2.get("lang")); // "hi" — same object

console.log(config1.getAll());
```

---

## L8 — Advanced Types

### 29. Discriminated Unions
```typescript
type Circle = { kind: "circle"; radius: number };
type Rectangle = { kind: "rectangle"; width: number; height: number };
type Triangle = { kind: "triangle"; base: number; height: number };

type Shape = Circle | Rectangle | Triangle;

function getArea(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2;
        case "rectangle":
            return shape.width * shape.height;
        case "triangle":
            return 0.5 * shape.base * shape.height;
        default:
            const _exhaustive: never = shape;
            throw new Error("Unknown shape");
    }
}

const shapes: Shape[] = [
    { kind: "circle", radius: 5 },
    { kind: "rectangle", width: 4, height: 6 },
    { kind: "triangle", base: 3, height: 8 },
];

shapes.forEach(s => console.log(`${s.kind}: area = ${getArea(s).toFixed(2)}`));
```

### 30. Type Guards
```typescript
interface Cat { type: "cat"; meow(): void; }
interface Dog { type: "dog"; bark(): void; }

type Pet = Cat | Dog;

// Custom type guard
function isCat(pet: Pet): pet is Cat {
    return pet.type === "cat";
}

function handlePet(pet: Pet): void {
    if (isCat(pet)) {
        pet.meow();
    } else {
        pet.bark();
    }
}

// instanceof type guard
class ApiError extends Error {
    constructor(public statusCode: number, message: string) {
        super(message);
        this.name = "ApiError";
    }
}

class NetworkError extends Error {
    constructor(public url: string, message: string) {
        super(message);
        this.name = "NetworkError";
    }
}

function handleError(err: unknown): void {
    if (err instanceof ApiError) {
        console.log(`API Error ${err.statusCode}: ${err.message}`);
    } else if (err instanceof NetworkError) {
        console.log(`Network Error at ${err.url}: ${err.message}`);
    } else if (err instanceof Error) {
        console.log(`Error: ${err.message}`);
    } else {
        console.log("Unknown error");
    }
}

handleError(new ApiError(404, "Not found"));
handleError(new NetworkError("https://api.example.com", "Timeout"));
```

### 31. Mapped Types and Conditional Types
```typescript
// Mapped type — transform every property
type Nullable<T> = { [K in keyof T]: T[K] | null };
type Optional<T> = { [K in keyof T]?: T[K] };
type Stringify<T> = { [K in keyof T]: string };

interface User {
    id: number;
    name: string;
    age: number;
}

type NullableUser = Nullable<User>;
const nullUser: NullableUser = { id: null, name: "Vishal", age: null };
console.log(nullUser);

// Conditional type
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<string>;   // "yes"
type B = IsString<number>;   // "no"

// NonNullable built-in
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>;  // string
```

---

## L9 — Async / Await and Promises

### 32. Promise Basics
```typescript
function delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
}

function fetchUser(id: number): Promise<{ id: number; name: string }> {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            if (id <= 0) {
                reject(new Error("Invalid ID"));
            } else {
                resolve({ id, name: `User_${id}` });
            }
        }, 100);
    });
}

// Promise chaining
fetchUser(1)
    .then(user => {
        console.log("Got user:", user);
        return fetchUser(2);
    })
    .then(user => console.log("Got user:", user))
    .catch(err => console.error("Error:", err.message));
```

### 33. Async / Await
```typescript
async function getUser(id: number): Promise<{ id: number; name: string }> {
    await new Promise(r => setTimeout(r, 50)); // simulate network
    if (id <= 0) throw new Error("Invalid user ID");
    return { id, name: `User_${id}` };
}

async function main(): Promise<void> {
    try {
        const user = await getUser(1);
        console.log("User:", user);

        const user2 = await getUser(-1);
        console.log("User2:", user2);
    } catch (err) {
        if (err instanceof Error) {
            console.error("Caught:", err.message);
        }
    }
}

main();
```

### 34. Promise.all and Promise.allSettled
```typescript
async function simulateFetch(id: number, shouldFail = false): Promise<string> {
    await new Promise(r => setTimeout(r, Math.random() * 100));
    if (shouldFail) throw new Error(`Failed to fetch ${id}`);
    return `Result_${id}`;
}

async function runAll(): Promise<void> {
    // Promise.all — fails fast if any rejects
    try {
        const results = await Promise.all([
            simulateFetch(1),
            simulateFetch(2),
            simulateFetch(3),
        ]);
        console.log("All resolved:", results);
    } catch (err) {
        console.error("One failed:", err);
    }

    // Promise.allSettled — waits for all, reports each outcome
    const settled = await Promise.allSettled([
        simulateFetch(1),
        simulateFetch(2, true),  // this one fails
        simulateFetch(3),
    ]);

    settled.forEach((result, i) => {
        if (result.status === "fulfilled") {
            console.log(`Task ${i + 1} ok:`, result.value);
        } else {
            console.log(`Task ${i + 1} failed:`, result.reason.message);
        }
    });
}

runAll();
```

---

## L10 — Mini CRUD App

### 35. In-Memory Task Manager
```typescript
interface Task {
    id: number;
    title: string;
    priority: "low" | "medium" | "high";
    done: boolean;
    createdAt: Date;
}

class TaskManager {
    private tasks: Map<number, Task> = new Map();
    private nextId: number = 1;

    create(title: string, priority: Task["priority"] = "medium"): Task {
        const task: Task = {
            id: this.nextId++,
            title,
            priority,
            done: false,
            createdAt: new Date(),
        };
        this.tasks.set(task.id, task);
        return task;
    }

    findById(id: number): Task | undefined {
        return this.tasks.get(id);
    }

    list(filter?: Partial<Pick<Task, "done" | "priority">>): Task[] {
        let result = Array.from(this.tasks.values());
        if (filter?.done !== undefined)
            result = result.filter(t => t.done === filter.done);
        if (filter?.priority)
            result = result.filter(t => t.priority === filter.priority);
        return result;
    }

    update(id: number, updates: Partial<Omit<Task, "id" | "createdAt">>): Task | undefined {
        const task = this.tasks.get(id);
        if (!task) return undefined;
        Object.assign(task, updates);
        return task;
    }

    complete(id: number): boolean {
        const task = this.tasks.get(id);
        if (!task) return false;
        task.done = true;
        return true;
    }

    delete(id: number): boolean {
        return this.tasks.delete(id);
    }

    print(task: Task): void {
        const mark = task.done ? "✓" : "○";
        const p = task.priority.toUpperCase().padEnd(6);
        console.log(`  [${mark}] #${task.id} | ${p} | ${task.title}`);
    }

    printAll(tasks: Task[] = this.list()): void {
        if (tasks.length === 0) { console.log("  (none)"); return; }
        tasks.forEach(t => this.print(t));
    }
}

// --- Demo ---
const tm = new TaskManager();

tm.create("Set up TypeScript environment", "high");
tm.create("Complete L1-L9 drills", "high");
tm.create("Build image memory demo", "medium");
tm.create("Push to GitHub", "low");

console.log("\n=== All Tasks ===");
tm.printAll();

tm.complete(1);
tm.complete(2);
tm.update(3, { priority: "high" });
tm.delete(4);

console.log("\n=== Pending ===");
tm.printAll(tm.list({ done: false }));

console.log("\n=== Completed ===");
tm.printAll(tm.list({ done: true }));

console.log("\n=== High Priority Only ===");
tm.printAll(tm.list({ priority: "high" }));
```

### 36. Generic Repository Pattern (Real-World)
```typescript
interface Entity {
    id: number;
}

interface Repository<T extends Entity> {
    create(data: Omit<T, "id">): T;
    findById(id: number): T | undefined;
    findAll(): T[];
    update(id: number, data: Partial<T>): T | undefined;
    delete(id: number): boolean;
}

class InMemoryRepository<T extends Entity> implements Repository<T> {
    private store: Map<number, T> = new Map();
    private nextId = 1;

    create(data: Omit<T, "id">): T {
        const item = { ...data, id: this.nextId++ } as T;
        this.store.set(item.id, item);
        return item;
    }

    findById(id: number): T | undefined {
        return this.store.get(id);
    }

    findAll(): T[] {
        return Array.from(this.store.values());
    }

    update(id: number, data: Partial<T>): T | undefined {
        const item = this.store.get(id);
        if (!item) return undefined;
        const updated = { ...item, ...data };
        this.store.set(id, updated);
        return updated;
    }

    delete(id: number): boolean {
        return this.store.delete(id);
    }
}

// Usage with Task
interface Task extends Entity {
    title: string;
    done: boolean;
}

interface Note extends Entity {
    content: string;
    tag: string;
}

const taskRepo = new InMemoryRepository<Task>();
const noteRepo = new InMemoryRepository<Note>();

taskRepo.create({ title: "Build CRUD app", done: false });
taskRepo.create({ title: "Write tests", done: false });
taskRepo.update(1, { done: true });

noteRepo.create({ content: "Use discriminated unions", tag: "typescript" });
noteRepo.create({ content: "Prefer interfaces for objects", tag: "typescript" });

console.log("Tasks:", taskRepo.findAll());
console.log("Notes:", noteRepo.findAll());
```

### 37. Async CRUD with Error Handling
```typescript
interface Task {
    id: number;
    title: string;
    done: boolean;
}

class TaskService {
    private tasks: Task[] = [];
    private nextId = 1;

    async create(title: string): Promise<Task> {
        await this.simulateDelay();
        if (!title.trim()) throw new Error("Title cannot be empty");
        const task: Task = { id: this.nextId++, title, done: false };
        this.tasks.push(task);
        return task;
    }

    async getAll(): Promise<Task[]> {
        await this.simulateDelay();
        return [...this.tasks];
    }

    async complete(id: number): Promise<Task> {
        await this.simulateDelay();
        const task = this.tasks.find(t => t.id === id);
        if (!task) throw new Error(`Task ${id} not found`);
        task.done = true;
        return task;
    }

    async delete(id: number): Promise<void> {
        await this.simulateDelay();
        const idx = this.tasks.findIndex(t => t.id === id);
        if (idx === -1) throw new Error(`Task ${id} not found`);
        this.tasks.splice(idx, 1);
    }

    private simulateDelay(): Promise<void> {
        return new Promise(r => setTimeout(r, 10));
    }
}

async function run(): Promise<void> {
    const service = new TaskService();

    const t1 = await service.create("Build TypeScript CRUD");
    const t2 = await service.create("Write README");
    console.log("Created:", t1, t2);

    await service.complete(1);
    await service.delete(2);

    const all = await service.getAll();
    console.log("Final tasks:", all);

    try {
        await service.complete(99);
    } catch (err) {
        if (err instanceof Error)
            console.error("Error:", err.message);
    }
}

run();
```

---

## Bonus Drills — Blank File Challenges

Write these completely from scratch.

### B1. Write a generic `Stack<T>` class with `push`, `pop`, `peek`, `isEmpty`, `size`

### B2. Write a `deepClone<T>(obj: T): T` function using JSON

### B3. Write a `groupBy<T>(arr: T[], key: keyof T): Record<string, T[]>` function
```
Input:  [{name:"Alice", role:"admin"}, {name:"Bob", role:"viewer"}, {name:"Eve", role:"admin"}]
Output: { admin: [...], viewer: [...] }
```

### B4. Write a `memoize<T extends any[], U>(fn: (...args: T) => U)` generic decorator

### B5. Write a `Result<T, E>` type (like Rust's Result) with `ok` and `err` variants and a `match` helper

### B6. Write a `pipe(...fns)` function that composes functions left to right
```
pipe(double, addOne, square)(3) → square(addOne(double(3))) = 49
```

### B7. Type a function `pluck<T, K extends keyof T>(arr: T[], key: K): T[K][]`
```
pluck([{name:"Vishal", age:22}], "name") → ["Vishal"]
```

### B8. Write a fully typed `EventEmitter<Events>` class where Events is a map of event names to payloads

---

## Progress Tracker

| Level | Topic                               | Done? | Time Taken |
|-------|-------------------------------------|-------|------------|
| L1    | Variables, Types, Unions            |  [ ]  |            |
| L2    | Functions, Callbacks, Arrow fns     |  [ ]  |            |
| L3    | Arrays, Objects, Destructuring      |  [ ]  |            |
| L4    | Interfaces, Extension               |  [ ]  |            |
| L5    | Generics, Constraints               |  [ ]  |            |
| L6    | Utility Types                       |  [ ]  |            |
| L7    | Classes, Inheritance, Abstract      |  [ ]  |            |
| L8    | Discriminated Unions, Type Guards   |  [ ]  |            |
| L9    | Async / Await, Promises             |  [ ]  |            |
| L10   | Mini CRUD App                       |  [ ]  |            |
| Bonus | Blank File Challenges               |  [ ]  |            |

---

> **Final challenge:** Open a blank file. Build exercise 35 (TypeScript Task Manager with full typing) completely from memory — no `any`, no peeking. Compile it clean. That's your milestone.
