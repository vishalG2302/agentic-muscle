# Rust Muscle Memory — 40+ Drills (L1 → L10)

> **Rules — same as all other sheets**
> - No AI. No autocomplete. Type every line.
> - Compile after every exercise: `rustc filename.rs && ./filename`
> - Or use Cargo: `cargo new project && cargo run`
> - Rust is intentionally harder than Python/C++ — the compiler is your teacher, not your enemy
> - L1–L3 → variables, types, ownership basics (30–40 min)
> - L4–L5 → ownership, borrowing, lifetimes (40–50 min)
> - L6–L7 → structs, enums, pattern matching (40–50 min)
> - L8–L9 → error handling, traits, generics (40–50 min)
> - L10 → mini CRUD app from scratch (60 min)
> - Install: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

---

## The Core Mental Model — Before You Write a Line

```
Rust's three rules — burn these in before touching code:

1. OWNERSHIP   → every value has exactly ONE owner
                 when the owner goes out of scope, value is dropped (freed)

2. BORROWING   → you can lend references (&T) without giving up ownership
                 rules: many readers OR one writer, never both at once

3. LIFETIMES   → references must not outlive the data they point to
                 the compiler checks this at compile time — no runtime crashes

Why does this matter?
  No garbage collector. No manual malloc/free. No null pointer crashes.
  The compiler guarantees memory safety. That's the whole deal.
```

Rust will feel slow at first because the compiler rejects things other languages allow.
That's not Rust being difficult — that's Rust catching bugs before they reach production.

---

## L1 — Variables, Types, Mutability

### 1. Hello Rust
```rust
fn main() {
    println!("Hello, Rust!");
    println!("Hello, {}!", "Vishal");
    println!("{} + {} = {}", 10, 20, 10 + 20);
}
```

---

### 2. Variables and Mutability
```rust
fn main() {
    // Variables are immutable by default
    let x = 5;
    println!("x = {}", x);

    // Must use `mut` to allow changes
    let mut y = 10;
    println!("y before = {}", y);
    y = 20;
    println!("y after  = {}", y);

    // Shadowing — reuse variable name with new value or type
    let z = 5;
    let z = z * 2;       // shadows previous z
    let z = z + 3;       // shadows again
    println!("z = {}", z); // 13

    // Constants — always immutable, must have type annotation
    const MAX_SCORE: u32 = 100;
    println!("Max score: {}", MAX_SCORE);
}
```

---

### 3. Basic Types
```rust
fn main() {
    // Integers
    let a: i32 = -42;
    let b: u32 = 42;
    let c: i64 = 1_000_000; // underscores for readability
    let d: u8  = 255;

    // Floats
    let f1: f64 = 3.14159;
    let f2: f32 = 2.71828_f32;

    // Bool
    let is_active: bool = true;
    let is_done = false; // inferred

    // Char — Unicode, single quotes
    let letter: char = 'A';
    let emoji: char = '🦀';

    println!("i32: {}, u32: {}, i64: {}, u8: {}", a, b, c, d);
    println!("f64: {:.2}, f32: {:.2}", f1, f2);
    println!("bool: {}, {}", is_active, is_done);
    println!("char: {}, emoji: {}", letter, emoji);

    // Numeric operations
    println!("10 / 3 = {}", 10_i32 / 3);     // integer division: 3
    println!("10 % 3 = {}", 10_i32 % 3);     // remainder: 1
    println!("2^10  = {}", 2_i32.pow(10));   // 1024
}
```

---

### 4. Tuples and Arrays
```rust
fn main() {
    // Tuple — fixed length, mixed types
    let person: (String, u32, f64) = (String::from("Vishal"), 22, 9.1);
    println!("Name: {}, Age: {}, CGPA: {}", person.0, person.1, person.2);

    // Destructuring
    let (name, age, cgpa) = &person;
    println!("Destructured: {} is {} with CGPA {}", name, age, cgpa);

    // Array — fixed length, same type
    let scores: [i32; 5] = [88, 95, 72, 61, 90];
    println!("First: {}, Last: {}", scores[0], scores[4]);
    println!("Length: {}", scores.len());

    // All same value
    let zeros = [0; 10];
    println!("Zeros: {:?}", zeros);

    // Iterate
    let mut total = 0;
    for score in &scores {
        total += score;
    }
    println!("Sum: {}, Avg: {:.1}", total, total as f64 / scores.len() as f64);
}
```

---

### 5. Functions
```rust
fn add(a: i32, b: i32) -> i32 {
    a + b  // no semicolon = expression = return value
}

fn greet(name: &str, greeting: &str) -> String {
    format!("{}, {}!", greeting, name)
}

fn divide(a: f64, b: f64) -> f64 {
    if b == 0.0 {
        panic!("Cannot divide by zero");  // like an exception
    }
    a / b
}

// Multiple return values via tuple
fn min_max(nums: &[i32]) -> (i32, i32) {
    let mut min = nums[0];
    let mut max = nums[0];
    for &n in nums {
        if n < min { min = n; }
        if n > max { max = n; }
    }
    (min, max)
}

fn main() {
    println!("{}", add(3, 4));
    println!("{}", greet("Vishal", "Namaste"));
    println!("{:.4}", divide(10.0, 3.0));

    let nums = [3, 1, 4, 1, 5, 9, 2, 6];
    let (min, max) = min_max(&nums);
    println!("Min: {}, Max: {}", min, max);
}
```

---

## L2 — Control Flow

### 6. If, Loop, While, For
```rust
fn main() {
    let score = 78;

    // if-else — also an expression
    let grade = if score >= 90 { "A" }
                else if score >= 75 { "B" }
                else if score >= 60 { "C" }
                else { "F" };
    println!("Grade: {}", grade);

    // loop with break value
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 5 { break counter * 2; }
    };
    println!("Loop result: {}", result);  // 10

    // while
    let mut n = 1;
    while n < 32 {
        print!("{} ", n);
        n *= 2;
    }
    println!();

    // for range
    for i in 1..=5 {
        print!("{} ", i);
    }
    println!();

    // for with enumerate
    let fruits = ["apple", "banana", "cherry"];
    for (i, fruit) in fruits.iter().enumerate() {
        println!("{}: {}", i + 1, fruit);
    }
}
```

---

### 7. FizzBuzz
```rust
fn main() {
    for i in 1..=50 {
        let output = if i % 15 == 0 { String::from("FizzBuzz") }
                     else if i % 3 == 0 { String::from("Fizz") }
                     else if i % 5 == 0 { String::from("Buzz") }
                     else { i.to_string() };
        println!("{}", output);
    }
}
```

---

## L3 — Ownership (The Most Important Chapter)

### 8. Move Semantics
```rust
fn main() {
    // String is heap-allocated — ownership can be moved
    let s1 = String::from("hello");
    let s2 = s1;             // s1 is MOVED into s2
    // println!("{}", s1);   // ERROR: s1 no longer valid
    println!("{}", s2);      // ok

    // i32 is stack-allocated — it's COPIED, not moved
    let x = 5;
    let y = x;               // x is copied
    println!("{} {}", x, y); // both valid — integers implement Copy

    // Clone makes a deep copy of heap data
    let a = String::from("world");
    let b = a.clone();       // explicit deep copy
    println!("{} {}", a, b); // both valid
}
```

---

### 9. Ownership with Functions
```rust
fn takes_ownership(s: String) {
    println!("Got: {}", s);
}   // s is dropped here

fn makes_copy(n: i32) {
    println!("Got: {}", n);
}   // n goes out of scope, nothing special

fn gives_ownership() -> String {
    String::from("I'm yours now")
}

fn takes_and_gives_back(s: String) -> String {
    s   // return ownership back to caller
}

fn main() {
    let s1 = gives_ownership();
    println!("{}", s1);

    let s2 = String::from("hello");
    let s3 = takes_and_gives_back(s2);
    // s2 is no longer valid — it was moved into the function
    println!("{}", s3);

    let n = 10;
    makes_copy(n);
    println!("n still valid: {}", n);  // ok — copied
}
```

---

### 10. References and Borrowing
```rust
// Borrowing = lending a reference without giving up ownership
fn calculate_length(s: &String) -> usize {
    s.len()
}   // s goes out of scope but we don't own it, nothing dropped

fn change(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let s1 = String::from("hello");

    // Immutable borrow — many readers allowed
    let len = calculate_length(&s1);
    println!("'{}' has {} characters", s1, len); // s1 still valid

    // Mutable borrow — only ONE at a time
    let mut s2 = String::from("hello");
    change(&mut s2);
    println!("{}", s2); // "hello, world"

    // Cannot have mutable and immutable borrows at same time
    let mut s3 = String::from("hello");
    let r1 = &s3;           // immutable borrow
    let r2 = &s3;           // another immutable borrow — ok
    println!("{} {}", r1, r2); // r1 and r2 used here, borrows end
    let r3 = &mut s3;       // mutable borrow — ok now, r1/r2 no longer in use
    println!("{}", r3);
}
```

---

### 11. Slices
```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }
    &s[..]
}

fn sum_slice(nums: &[i32]) -> i32 {
    nums.iter().sum()
}

fn main() {
    let sentence = String::from("hello world");
    let word = first_word(&sentence);
    println!("First word: {}", word);

    // String slices
    let s = String::from("hello world");
    let hello = &s[0..5];
    let world = &s[6..11];
    println!("{} {}", hello, world);

    // Array slices
    let arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let middle = &arr[2..7];
    println!("Middle slice: {:?}", middle);
    println!("Sum of middle: {}", sum_slice(middle));
}
```

---

## L4 — Structs

### 12. Basic Struct
```rust
#[derive(Debug, Clone)]
struct Student {
    name: String,
    age: u32,
    cgpa: f64,
}

impl Student {
    // Associated function (like static method) — constructor
    fn new(name: &str, age: u32, cgpa: f64) -> Student {
        Student {
            name: String::from(name),
            age,
            cgpa,
        }
    }

    // Method — takes &self (immutable borrow of self)
    fn is_honours(&self) -> bool {
        self.cgpa >= 8.5
    }

    fn introduce(&self) -> String {
        format!("Hi, I'm {} (age {}), CGPA: {:.1}", self.name, self.age, self.cgpa)
    }

    // Mutable method — takes &mut self
    fn update_cgpa(&mut self, new_cgpa: f64) {
        self.cgpa = new_cgpa;
    }
}

fn main() {
    let mut s1 = Student::new("Vishal", 22, 9.1);
    let s2 = Student::new("Alice", 21, 7.8);

    println!("{}", s1.introduce());
    println!("Honours: {}", s1.is_honours());
    println!("{:?}", s2);

    s1.update_cgpa(9.3);
    println!("Updated CGPA: {}", s1.cgpa);

    // Struct update syntax
    let s3 = Student {
        name: String::from("Bob"),
        ..s2.clone()   // take remaining fields from s2
    };
    println!("{}", s3.introduce());
}
```

---

### 13. Struct with Methods — Rectangle
```rust
#[derive(Debug)]
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    fn new(width: f64, height: f64) -> Self {
        Rectangle { width, height }
    }

    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }

    fn is_square(&self) -> bool {
        (self.width - self.height).abs() < f64::EPSILON
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }

    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }
}

fn main() {
    let mut r1 = Rectangle::new(10.0, 5.0);
    let r2 = Rectangle::new(3.0, 2.0);
    let r3 = Rectangle::new(4.0, 4.0);

    println!("r1: {:?}", r1);
    println!("Area: {}", r1.area());
    println!("Perimeter: {}", r1.perimeter());
    println!("Is square: {}", r1.is_square());
    println!("r3 is square: {}", r3.is_square());
    println!("r1 can hold r2: {}", r1.can_hold(&r2));

    r1.scale(2.0);
    println!("After scale: {:?}", r1);
}
```

---

## L5 — Enums and Pattern Matching

### 14. Basic Enum
```rust
#[derive(Debug)]
enum Direction {
    North,
    South,
    East,
    West,
}

#[derive(Debug)]
enum Priority {
    Low,
    Medium,
    High,
    Critical,
}

impl Priority {
    fn score(&self) -> u8 {
        match self {
            Priority::Low => 1,
            Priority::Medium => 2,
            Priority::High => 3,
            Priority::Critical => 4,
        }
    }
}

fn move_player(dir: &Direction) -> (i32, i32) {
    match dir {
        Direction::North => (0, 1),
        Direction::South => (0, -1),
        Direction::East  => (1, 0),
        Direction::West  => (-1, 0),
    }
}

fn main() {
    let dir = Direction::North;
    let delta = move_player(&dir);
    println!("Moving {:?}: dx={}, dy={}", dir, delta.0, delta.1);

    let p = Priority::High;
    println!("Priority {:?} has score {}", p, p.score());
}
```

---

### 15. Enums with Data
```rust
#[derive(Debug)]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Triangle { base, height } => 0.5 * base * height,
        }
    }

    fn name(&self) -> &str {
        match self {
            Shape::Circle { .. } => "Circle",
            Shape::Rectangle { .. } => "Rectangle",
            Shape::Triangle { .. } => "Triangle",
        }
    }
}

fn main() {
    let shapes = vec![
        Shape::Circle { radius: 5.0 },
        Shape::Rectangle { width: 4.0, height: 6.0 },
        Shape::Triangle { base: 3.0, height: 8.0 },
    ];

    for shape in &shapes {
        println!("{}: area = {:.2}", shape.name(), shape.area());
    }
}
```

---

### 16. Option<T> — Rust's Answer to Null
```rust
fn find_user(id: u32) -> Option<String> {
    let users = vec![
        (1, "Vishal"),
        (2, "Alice"),
        (3, "Bob"),
    ];
    for (uid, name) in &users {
        if *uid == id {
            return Some(String::from(*name));
        }
    }
    None
}

fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

fn main() {
    // Pattern matching on Option
    match find_user(1) {
        Some(name) => println!("Found: {}", name),
        None => println!("Not found"),
    }

    // if let — cleaner when you only care about Some
    if let Some(name) = find_user(2) {
        println!("User: {}", name);
    }

    // unwrap_or — default if None
    let name = find_user(99).unwrap_or(String::from("Anonymous"));
    println!("Name: {}", name);

    // map — transform the inner value if Some
    let doubled = divide(10.0, 4.0).map(|x| x * 2.0);
    println!("Doubled: {:?}", doubled);

    // chaining with and_then (flatMap)
    let result = find_user(1)
        .and_then(|name| if name.len() > 3 { Some(name) } else { None });
    println!("Result: {:?}", result);
}
```

---

## L6 — Error Handling

### 17. Result<T, E>
```rust
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    ParseError(ParseIntError),
    NegativeNumber(i32),
    TooBig(i32),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            AppError::ParseError(e) => write!(f, "Parse error: {}", e),
            AppError::NegativeNumber(n) => write!(f, "Negative number: {}", n),
            AppError::TooBig(n) => write!(f, "Number too big: {}", n),
        }
    }
}

fn parse_and_validate(s: &str) -> Result<i32, AppError> {
    let n: i32 = s.parse().map_err(AppError::ParseError)?;  // ? propagates error
    if n < 0 {
        return Err(AppError::NegativeNumber(n));
    }
    if n > 1000 {
        return Err(AppError::TooBig(n));
    }
    Ok(n)
}

fn main() {
    let inputs = ["42", "-5", "2000", "abc", "100"];

    for input in &inputs {
        match parse_and_validate(input) {
            Ok(n) => println!("'{}' → valid: {}", input, n),
            Err(e) => println!("'{}' → error: {}", input, e),
        }
    }

    // unwrap_or_else
    let value = parse_and_validate("xyz").unwrap_or_else(|_| -1);
    println!("Default on error: {}", value);
}
```

---

### 18. The ? Operator
```rust
use std::fs;
use std::io;

fn read_and_process(path: &str) -> Result<usize, io::Error> {
    let content = fs::read_to_string(path)?;   // ? returns early on error
    let word_count = content.split_whitespace().count();
    Ok(word_count)
}

fn main() {
    // Create a test file
    fs::write("test.txt", "Hello world this is Rust").unwrap();

    match read_and_process("test.txt") {
        Ok(count) => println!("Word count: {}", count),
        Err(e) => println!("Error: {}", e),
    }

    match read_and_process("nonexistent.txt") {
        Ok(count) => println!("Word count: {}", count),
        Err(e) => println!("Error: {}", e),  // file not found
    }

    // Clean up
    fs::remove_file("test.txt").unwrap();
}
```

---

## L7 — Collections

### 19. Vec<T>
```rust
fn main() {
    // Create
    let mut v: Vec<i32> = Vec::new();
    v.push(1);
    v.push(2);
    v.push(3);

    let v2 = vec![10, 20, 30, 40, 50]; // macro shorthand

    // Access
    println!("First: {}", v2[0]);
    println!("Safe get: {:?}", v2.get(10)); // returns Option — no panic

    // Iterate
    for n in &v2 {
        print!("{} ", n);
    }
    println!();

    // Mutate while iterating
    let mut v3 = vec![1, 2, 3, 4, 5];
    for n in &mut v3 {
        *n *= 2;   // dereference to modify
    }
    println!("Doubled: {:?}", v3);

    // Useful methods
    let nums = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3];
    println!("Len: {}", nums.len());
    println!("Contains 9: {}", nums.contains(&9));

    let mut sorted = nums.clone();
    sorted.sort();
    sorted.dedup();
    println!("Sorted unique: {:?}", sorted);

    // Functional style
    let evens: Vec<i32> = nums.iter().filter(|&&x| x % 2 == 0).copied().collect();
    let squares: Vec<i32> = nums.iter().map(|&x| x * x).collect();
    let total: i32 = nums.iter().sum();

    println!("Evens: {:?}", evens);
    println!("Squares: {:?}", squares);
    println!("Total: {}", total);
}
```

---

### 20. HashMap
```rust
use std::collections::HashMap;

fn main() {
    // Create and insert
    let mut scores: HashMap<String, i32> = HashMap::new();
    scores.insert(String::from("Vishal"), 95);
    scores.insert(String::from("Alice"), 88);
    scores.insert(String::from("Bob"), 72);

    // Access
    if let Some(score) = scores.get("Vishal") {
        println!("Vishal's score: {}", score);
    }

    // entry API — insert only if not present
    scores.entry(String::from("Alice")).or_insert(100);  // won't overwrite
    scores.entry(String::from("Charlie")).or_insert(65); // inserts new

    println!("{:?}", scores);

    // Iterate
    for (name, score) in &scores {
        println!("{}: {}", name, score);
    }

    // Word frequency — classic pattern
    let text = "hello world hello rust world hello";
    let mut freq: HashMap<&str, u32> = HashMap::new();

    for word in text.split_whitespace() {
        let count = freq.entry(word).or_insert(0);
        *count += 1;
    }

    println!("\nWord frequencies:");
    let mut freq_vec: Vec<(&&str, &u32)> = freq.iter().collect();
    freq_vec.sort_by(|a, b| b.1.cmp(a.1));
    for (word, count) in freq_vec {
        println!("  {}: {}", word, count);
    }
}
```

---

## L8 — Traits

### 21. Defining and Implementing Traits
```rust
use std::fmt;

trait Area {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;

    // Default method
    fn describe(&self) -> String {
        format!("Area: {:.2}, Perimeter: {:.2}", self.area(), self.perimeter())
    }
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Area for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn perimeter(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }
}

impl Area for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }
}

impl fmt::Display for Circle {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Circle(r={})", self.radius)
    }
}

// Trait as parameter — accepts any type that implements Area
fn print_shape_info(shape: &impl Area) {
    println!("{}", shape.describe());
}

fn main() {
    let c = Circle { radius: 5.0 };
    let r = Rectangle { width: 4.0, height: 6.0 };

    print_shape_info(&c);
    print_shape_info(&r);
    println!("{}", c);
}
```

---

### 22. Generics with Trait Bounds
```rust
use std::fmt::Display;

fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn print_pair<T: Display, U: Display>(a: T, b: U) {
    println!("({}, {})", a, b);
}

#[derive(Debug)]
struct Pair<T> {
    first: T,
    second: T,
}

impl<T: Display + PartialOrd> Pair<T> {
    fn new(first: T, second: T) -> Self {
        Pair { first, second }
    }

    fn larger(&self) -> &T {
        if self.first >= self.second { &self.first } else { &self.second }
    }
}

fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    println!("Largest number: {}", largest(&numbers));

    let chars = vec!['y', 'm', 'a', 'q'];
    println!("Largest char: {}", largest(&chars));

    print_pair("hello", 42);
    print_pair(3.14, true);

    let p = Pair::new(5, 10);
    println!("Larger: {}", p.larger());
}
```

---

## L9 — Iterators and Closures

### 23. Closures
```rust
fn apply<F: Fn(i32) -> i32>(f: F, value: i32) -> i32 {
    f(value)
}

fn apply_twice<F: Fn(i32) -> i32>(f: F, value: i32) -> i32 {
    f(f(value))
}

fn main() {
    // Basic closure
    let double = |x| x * 2;
    let add_ten = |x| x + 10;
    let square = |x: i32| x * x;

    println!("{}", apply(double, 5));       // 10
    println!("{}", apply_twice(double, 3)); // 12
    println!("{}", apply(square, 4));       // 16

    // Capture from environment
    let threshold = 50;
    let above_threshold = |n: &i32| *n > threshold;

    let nums = vec![10, 60, 30, 80, 45, 70];
    let above: Vec<&i32> = nums.iter().filter(|n| above_threshold(n)).collect();
    println!("Above {}: {:?}", threshold, above);

    // move closure — takes ownership
    let name = String::from("Vishal");
    let greet = move || println!("Hello, {}!", name);
    greet();
    // name is no longer valid here — moved into closure
}
```

---

### 24. Iterator Methods
```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // map, filter, collect
    let even_squares: Vec<u32> = nums.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .collect();
    println!("Even squares: {:?}", even_squares);

    // fold — like reduce
    let product: u32 = nums.iter().fold(1, |acc, &x| acc * x);
    println!("Product: {}", product);

    // any and all
    let has_even = nums.iter().any(|&x| x % 2 == 0);
    let all_positive = nums.iter().all(|&x| x > 0);
    println!("Has even: {}, All positive: {}", has_even, all_positive);

    // enumerate, zip
    let letters = vec!['a', 'b', 'c', 'd'];
    let pairs: Vec<(usize, &char)> = letters.iter().enumerate().collect();
    println!("Pairs: {:?}", pairs);

    let zipped: Vec<(&u32, &char)> = nums.iter().take(4).zip(letters.iter()).collect();
    println!("Zipped: {:?}", zipped);

    // flatten
    let nested = vec![vec![1, 2, 3], vec![4, 5], vec![6, 7, 8]];
    let flat: Vec<i32> = nested.into_iter().flatten().collect();
    println!("Flattened: {:?}", flat);

    // chain
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    let chained: Vec<&i32> = a.iter().chain(b.iter()).collect();
    println!("Chained: {:?}", chained);
}
```

---

## L10 — Mini CRUD App (Build from Memory)

### 25. In-Memory Task Manager
```rust
use std::collections::HashMap;
use std::io::{self, BufRead, Write};

#[derive(Debug, Clone)]
struct Task {
    id: u32,
    title: String,
    priority: String,
    done: bool,
}

impl Task {
    fn new(id: u32, title: String, priority: String) -> Self {
        Task { id, title, priority, done: false }
    }

    fn display(&self) {
        let mark = if self.done { "✓" } else { "○" };
        println!(
            "  [{}] #{:<3} | {:<8} | {}",
            mark, self.id, self.priority, self.title
        );
    }
}

struct TaskManager {
    tasks: HashMap<u32, Task>,
    next_id: u32,
}

impl TaskManager {
    fn new() -> Self {
        TaskManager {
            tasks: HashMap::new(),
            next_id: 1,
        }
    }

    fn create(&mut self, title: String, priority: String) -> &Task {
        let id = self.next_id;
        self.tasks.insert(id, Task::new(id, title, priority));
        self.next_id += 1;
        self.tasks.get(&id).unwrap()
    }

    fn get(&self, id: u32) -> Option<&Task> {
        self.tasks.get(&id)
    }

    fn list(&self, filter_done: Option<bool>) -> Vec<&Task> {
        let mut tasks: Vec<&Task> = match filter_done {
            Some(done) => self.tasks.values().filter(|t| t.done == done).collect(),
            None => self.tasks.values().collect(),
        };
        tasks.sort_by_key(|t| t.id);
        tasks
    }

    fn complete(&mut self, id: u32) -> bool {
        if let Some(task) = self.tasks.get_mut(&id) {
            task.done = true;
            true
        } else {
            false
        }
    }

    fn delete(&mut self, id: u32) -> Option<Task> {
        self.tasks.remove(&id)
    }

    fn stats(&self) -> (usize, usize, usize) {
        let total = self.tasks.len();
        let done = self.tasks.values().filter(|t| t.done).count();
        (total, done, total - done)
    }
}

fn show_menu() {
    println!("\n=== Task Manager ===");
    println!("1. Add Task");
    println!("2. List All");
    println!("3. List Pending");
    println!("4. Complete Task");
    println!("5. Delete Task");
    println!("6. Stats");
    println!("7. Exit");
    print!("Choice: ");
    io::stdout().flush().unwrap();
}

fn read_line() -> String {
    let stdin = io::stdin();
    let mut line = String::new();
    stdin.lock().read_line(&mut line).unwrap();
    line.trim().to_string()
}

fn main() {
    let mut manager = TaskManager::new();

    // Seed some tasks
    manager.create(String::from("Learn Rust ownership"), String::from("high"));
    manager.create(String::from("Build MCP server"), String::from("high"));
    manager.create(String::from("Complete Rust drills"), String::from("medium"));
    manager.create(String::from("Push to GitHub"), String::from("low"));

    loop {
        show_menu();
        let choice = read_line();

        match choice.as_str() {
            "1" => {
                print!("Title: ");
                io::stdout().flush().unwrap();
                let title = read_line();
                if title.is_empty() {
                    println!("Title cannot be empty.");
                    continue;
                }
                print!("Priority (low/medium/high): ");
                io::stdout().flush().unwrap();
                let priority = read_line();
                let valid = ["low", "medium", "high"];
                if !valid.contains(&priority.as_str()) {
                    println!("Invalid priority.");
                    continue;
                }
                let task = manager.create(title, priority);
                println!("Created task #{}", task.id);
            }
            "2" => {
                let tasks = manager.list(None);
                if tasks.is_empty() {
                    println!("No tasks.");
                } else {
                    for t in tasks { t.display(); }
                }
            }
            "3" => {
                let tasks = manager.list(Some(false));
                if tasks.is_empty() {
                    println!("No pending tasks.");
                } else {
                    for t in tasks { t.display(); }
                }
            }
            "4" => {
                print!("Task ID: ");
                io::stdout().flush().unwrap();
                let id: u32 = match read_line().parse() {
                    Ok(n) => n,
                    Err(_) => { println!("Invalid ID."); continue; }
                };
                if manager.complete(id) {
                    println!("Task #{} marked complete.", id);
                } else {
                    println!("Task #{} not found.", id);
                }
            }
            "5" => {
                print!("Task ID: ");
                io::stdout().flush().unwrap();
                let id: u32 = match read_line().parse() {
                    Ok(n) => n,
                    Err(_) => { println!("Invalid ID."); continue; }
                };
                match manager.delete(id) {
                    Some(t) => println!("Deleted: '{}'", t.title),
                    None => println!("Task #{} not found.", id),
                }
            }
            "6" => {
                let (total, done, pending) = manager.stats();
                println!("Total: {} | Done: {} | Pending: {}", total, done, pending);
            }
            "7" => {
                println!("Bye!");
                break;
            }
            _ => println!("Invalid choice."),
        }
    }
}
```

---

## Bonus Drills — Blank File Challenges

### B1. Write a `Stack<T>` struct with `push`, `pop`, `peek`, `is_empty`, `size`

### B2. Write a function `fibonacci(n: u64) -> u64` both iteratively and recursively

### B3. Write a `Matrix` struct (2x2) with `add`, `multiply`, `transpose`, `Display` trait

### B4. Write `fn word_frequency(text: &str) -> HashMap<&str, usize>` and return top 3 words

### B5. Implement the `Display` and `From` traits for a custom `Color` struct with r, g, b fields

### B6. Build a `BankAccount` struct with `deposit`, `withdraw` returning `Result<f64, String>`, and a transaction history `Vec<String>`

---

## Rust vs Python/C++ — Quick Reference

| Python | C++ | Rust |
|---|---|---|
| No type annotations needed | Manual types | Types required, inferred where possible |
| GC handles memory | malloc/free manually | Ownership system — compile time |
| `None` | `nullptr` | `Option<T>` — no null |
| `try/except` | `try/catch` | `Result<T, E>` + `?` operator |
| `def` | `void/int fn()` | `fn` with explicit return type |
| List | `std::vector` | `Vec<T>` |
| Dict | `std::unordered_map` | `HashMap<K, V>` |
| Class methods | Member functions | `impl` blocks |
| Inheritance | Inheritance | Traits (composition over inheritance) |
| `print()` | `std::cout` | `println!()` macro |

---

## The Ownership Rules — Keep This Visible

```
Rule 1: Each value has one owner.
Rule 2: There can only be one owner at a time.
Rule 3: When the owner goes out of scope, the value is dropped.

Borrowing rules:
  - You can have any number of immutable references (&T)
  - OR exactly one mutable reference (&mut T)
  - But never both at the same time
  - References must always be valid (no dangling pointers)
```

When the compiler rejects your code — READ the error message fully.
Rust error messages are the best in any language. They tell you exactly what went wrong and often how to fix it.

---

## Progress Tracker

| Level | Topic                                    | Done? | Time Taken |
|-------|------------------------------------------|-------|------------|
| L1    | Variables, Types, Functions              |  [ ]  |            |
| L2    | Control Flow                             |  [ ]  |            |
| L3    | Ownership, Borrowing, Slices             |  [ ]  |            |
| L4    | Structs and impl blocks                  |  [ ]  |            |
| L5    | Enums, Pattern Matching, Option          |  [ ]  |            |
| L6    | Error Handling, Result, ?                |  [ ]  |            |
| L7    | Collections: Vec, HashMap                |  [ ]  |            |
| L8    | Traits, Generics                         |  [ ]  |            |
| L9    | Closures, Iterators                      |  [ ]  |            |
| L10   | Mini CRUD App                            |  [ ]  |            |
| Bonus | Blank File Challenges                    |  [ ]  |            |

---

> **One honest note about Rust:**
> It will feel slower than Python and C++ at first — not because you are slow,
> but because the compiler is catching things other languages let slide until runtime.
> Every time the compiler rejects your code, it is teaching you something real.
> Read the error. Fix it. Move on. That loop IS the learning.
>
> **Final challenge:** Open a blank file. Build exercise 25 (the Task Manager CLI)
> from memory — struct, impl block, HashMap, Result, match, CLI loop.
> When it compiles and runs — you own Rust basics.
