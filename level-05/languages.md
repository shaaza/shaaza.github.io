---
layout: page
title: "Languages: Shared Constructs, Different Paradigms"
description: "One problem, six languages. Learn that syntax is noise — paradigm is signal."
permalink: /level-05/languages/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-05/">Level 0.5</a> / Languages'
---

## The Core Insight

Every programming language is a set of trade-offs frozen in syntax. Under the surface, they all manipulate the same machine you studied in Level 0. The differences are in **what they make easy, what they make hard, and what they make impossible**.

> **Key insight: Syntax is noise. Paradigm is signal.**

Two languages can look completely different and express the same computation. Two languages can look similar and enforce radically different guarantees. What matters is not the curly braces or the indentation — it is the *paradigm*: the fundamental model of how computation should be organized.

---

## One Problem, Six Languages

Take a single, simple problem:

> **Given a list of integers, keep only the even numbers, double each one, and sum the result.**

Input: `[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]`
Expected output: `60` (2+4+6+8+10 doubled = 4+8+12+16+20 = 60)

### C — Imperative / Systems

```c
#include <stdio.h>

int main() {
    int nums[] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    int len = sizeof(nums) / sizeof(nums[0]);
    int sum = 0;

    for (int i = 0; i < len; i++) {
        if (nums[i] % 2 == 0) {
            sum += nums[i] * 2;
        }
    }

    printf("%d\n", sum);
    return 0;
}
```

**What C makes explicit:** Memory layout, loop control, indexing. You manage everything. The `for` loop is a direct instruction to the machine: start here, check this, increment that.

### Python — Dynamic / Scripting

```python
nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
result = sum(x * 2 for x in nums if x % 2 == 0)
print(result)
```

**What Python makes easy:** Expressiveness. A generator expression replaces the explicit loop, condition, and accumulator in a single readable line. No types declared, no memory managed.

### Java — OOP / Static

```java
import java.util.Arrays;
import java.util.List;

public class Main {
    public static void main(String[] args) {
        List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        int result = nums.stream()
            .filter(x -> x % 2 == 0)
            .mapToInt(x -> x * 2)
            .sum();

        System.out.println(result);
    }
}
```

**What Java enforces:** Types, classes, explicit structure. Even a simple computation lives inside a class with a `public static void main`. The Stream API brings functional style into the OOP world.

### Haskell — Functional / Pure

```haskell
main :: IO ()
main = print $ sum $ map (*2) $ filter even [1..10]
```

**What Haskell expresses:** A pipeline of pure transformations. No variables mutated, no loop counter, no state. Read right to left: generate list, filter evens, double each, sum, print. The `$` operator is just function application — no parentheses needed.

### Rust — Systems / Ownership

```rust
fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let result: i32 = nums.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * 2)
        .sum();

    println!("{}", result);
}
```

**What Rust guarantees:** Memory safety without garbage collection. The iterator chain looks like Java's streams, but Rust's ownership system ensures no data races, no use-after-free, no null pointer dereferences — all at compile time.

### JavaScript — Multi-Paradigm

```javascript
const nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

const result = nums
    .filter(x => x % 2 === 0)
    .map(x => x * 2)
    .reduce((acc, x) => acc + x, 0);

console.log(result);
```

**What JavaScript allows:** Mix and match. Functional-style chaining with `filter`, `map`, `reduce` coexists with imperative mutation, prototype-based OOP, and async callbacks. Freedom and footguns in equal measure.

---

## Language Comparison Table

| Feature | C | Python | Java | Haskell | Rust | JavaScript |
|---------|---|--------|------|---------|------|------------|
| **Paradigm** | Imperative | Multi-paradigm | OOP + Functional | Pure Functional | Multi-paradigm | Multi-paradigm |
| **Typing** | Static, weak | Dynamic, strong | Static, strong | Static, strong | Static, strong | Dynamic, weak |
| **Memory** | Manual | GC | GC | GC | Ownership | GC |
| **Null** | NULL pointer | None | null | Maybe type | Option type | null / undefined |
| **Error handling** | Return codes | Exceptions | Exceptions | Maybe / Either | Result type | Exceptions / Promises |
| **Concurrency** | pthreads | GIL + async | Threads + virtual threads | STM + sparks | Ownership + async | Event loop + async |
| **Compilation** | Compiled (native) | Interpreted | Compiled (bytecode) | Compiled (native) | Compiled (native) | JIT compiled |
| **First appeared** | 1972 | 1991 | 1995 | 1990 | 2010 | 1995 |

---

## Shared Constructs Across Languages

Despite their differences, all these languages share fundamental constructs. Understanding these shared constructs lets you learn any new language quickly.

### Loops: `for` and `while`

Every language has a way to repeat computation. The syntax varies; the concept is universal.

```c
// C: explicit index
for (int i = 0; i < n; i++) { ... }

// Python: iteration over collection
for x in collection: ...

// Haskell: recursion replaces loops
sumList [] = 0
sumList (x:xs) = x + sumList xs

// Rust: iterator
for x in collection.iter() { ... }
```

In imperative languages, loops mutate state. In functional languages, recursion and higher-order functions replace loops entirely. **Same computation, different model.**

### Type Systems

Type systems are arguably the most important difference between languages. They determine what errors you catch at compile time vs. runtime.

```python
# Python: types checked at runtime
def add(a, b):
    return a + b   # Works with ints, strings, lists — or crashes

# Java: types checked at compile time
int add(int a, int b) {
    return a + b;   // Only ints. Compiler rejects anything else.
}

# Haskell: types inferred and checked at compile time
add :: Num a => a -> a -> a
add a b = a + b     -- Works with any Num type, but only Num types
```

### Error Handling

How a language handles errors reveals its philosophy:

```c
// C: error codes — easy to ignore
int result = open("file.txt", O_RDONLY);
if (result == -1) { /* handle error */ }
// Nothing stops you from ignoring the -1

// Java: exceptions — can be caught or crash
try {
    FileReader f = new FileReader("file.txt");
} catch (FileNotFoundException e) {
    // handle error
}

// Rust: Result type — compiler forces you to handle it
let result = File::open("file.txt");
match result {
    Ok(file) => { /* use file */ },
    Err(e) => { /* handle error */ },
}
// You literally cannot use the file without handling the error case

// Haskell: Maybe / Either — absence of value is in the type
readFile "file.txt" >>= \content -> ...
-- IO errors are tracked in the type system
```

### Strings

Strings seem simple. They are not. Every language handles them differently, and the differences matter:

```c
// C: array of bytes, null-terminated
char *s = "hello";   // s[5] == '\0'
// Buffer overflows, encoding confusion, manual memory

// Python: immutable unicode sequences
s = "hello 🌍"      # Unicode by default, immutable
# Easy but hides complexity of encoding

// Rust: UTF-8 encoded, owned or borrowed
let s: String = String::from("hello");  // Owned, heap-allocated
let r: &str = "hello";                  // Borrowed, string slice
// Cannot index by byte position — forces correct Unicode handling
```

---

## Deeper Language Features

Beyond the shared constructs, languages diverge into features that fundamentally change how you think about code.

### Recursion and Higher-Order Functions

**Recursion** replaces loops in functional languages. **Higher-order functions** — functions that take or return other functions — are the primary tool for abstraction.

```haskell
-- Haskell: map is a higher-order function
-- It takes a function and a list, applies the function to each element
map (*2) [1, 2, 3]         -- [2, 4, 6]

-- filter takes a predicate (function returning Bool) and a list
filter even [1, 2, 3, 4]   -- [2, 4]

-- foldr is the general pattern: reduce a list to a single value
foldr (+) 0 [1, 2, 3, 4]   -- 10

-- Function composition: (.) combines two functions
doubleEvens = map (*2) . filter even
doubleEvens [1..10]         -- [4, 8, 12, 16, 20]
```

```python
# Python supports the same patterns
list(map(lambda x: x * 2, filter(lambda x: x % 2 == 0, range(1, 11))))
# But the Pythonic way is comprehensions:
[x * 2 for x in range(1, 11) if x % 2 == 0]
```

### Pattern Matching

Pattern matching lets you destructure data and branch on its shape in a single expression:

```haskell
-- Haskell: pattern matching on algebraic data types
data Shape = Circle Double | Rectangle Double Double

area :: Shape -> Double
area (Circle r)      = pi * r * r
area (Rectangle w h) = w * h
```

```rust
// Rust: pattern matching with exhaustiveness checking
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}

fn area(s: &Shape) -> f64 {
    match s {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rectangle(w, h) => w * h,
        // Compiler error if you forget a variant!
    }
}
```

### Algebraic Data Types

Algebraic data types (ADTs) let you model your domain precisely. They come in two forms:

- **Sum types** (this OR that): `data Maybe a = Nothing | Just a`
- **Product types** (this AND that): `data Point = Point Double Double`

```haskell
-- Model a web request result precisely
data Response
    = Success Int String       -- status code and body
    | Redirect String          -- new URL
    | ClientError Int String   -- error code and message
    | ServerError Int          -- error code
    | Timeout                  -- no additional data

-- The compiler ensures you handle every case
handleResponse :: Response -> String
handleResponse (Success _ body)     = "Got: " ++ body
handleResponse (Redirect url)       = "Go to: " ++ url
handleResponse (ClientError code _) = "Client error: " ++ show code
handleResponse (ServerError code)   = "Server error: " ++ show code
handleResponse Timeout              = "Request timed out"
```

### Ownership and Borrowing (Rust)

Rust's most distinctive feature: the compiler tracks who *owns* each piece of data and who is *borrowing* it.

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;        // s1 is MOVED to s2. s1 is no longer valid.
    // println!("{}", s1);  // Compiler error: value borrowed after move

    let s3 = String::from("world");
    let len = calculate_length(&s3);  // Borrow s3 (read-only reference)
    println!("{} has length {}", s3, len);  // s3 still valid — it was borrowed, not moved
}

fn calculate_length(s: &String) -> usize {
    s.len()
}   // s goes out of scope, but since it was a borrow, nothing happens
```

This eliminates entire categories of bugs (use-after-free, data races, dangling pointers) at compile time. No garbage collector needed.

### Lazy Evaluation (Haskell)

Haskell evaluates expressions only when their values are needed:

```haskell
-- This works! An infinite list, never fully computed
naturals = [1..]

-- Take only what you need
take 5 naturals             -- [1, 2, 3, 4, 5]

-- Elegant definition of Fibonacci sequence
fibs = 0 : 1 : zipWith (+) fibs (tail fibs)
take 10 fibs                -- [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

Lazy evaluation enables working with infinite data structures and separating the *description* of a computation from its *execution*.

### Macros (Rust, Lisp)

Macros let you write code that writes code — metaprogramming:

```rust
// Rust: declarative macros
macro_rules! vec_of_strings {
    ($($x:expr),*) => {
        vec![$($x.to_string()),*]
    };
}

let names = vec_of_strings!["Alice", "Bob", "Charlie"];
// Expands at compile time to:
// vec!["Alice".to_string(), "Bob".to_string(), "Charlie".to_string()]
```

---

## Feature Comparison Table

| Feature | C | Python | Java | Haskell | Rust | JavaScript |
|---------|---|--------|------|---------|------|------------|
| **Higher-order functions** | Function pointers | Yes | Yes (lambdas) | Core feature | Yes (closures) | Yes |
| **Pattern matching** | switch (limited) | match (3.10+) | switch + sealed | Core feature | Core feature | No |
| **Algebraic data types** | Tagged unions (manual) | dataclass + Enum | Sealed classes | Core feature | Core feature (enum) | No |
| **Generics / Parametric polymorphism** | Void pointers | Duck typing | Yes (type erasure) | Yes (type classes) | Yes (monomorphized) | No (dynamic) |
| **Lazy evaluation** | No | Generators | Streams (lazy) | Default | Iterators (lazy) | Generators |
| **Macros** | Preprocessor (#define) | Decorators (limited) | Annotations | Template Haskell | Procedural + declarative | No |
| **Ownership / Borrowing** | No | No | No | No | Core feature | No |
| **Tail call optimization** | Compiler-dependent | No | No | Yes | No (limited) | Spec yes, engines vary |
| **Immutability default** | No (const opt-in) | No | No (final opt-in) | Yes | Yes (mut opt-in) | No (const opt-in) |

---

## What to Take Away

1. **Learn paradigms, not just languages.** Once you understand imperative, functional, and ownership-based systems thinking, picking up a new language takes days, not months.

2. **Every language is a trade-off.** C gives you control but demands discipline. Haskell gives you safety but demands a new way of thinking. Rust gives you both but demands a steeper learning curve.

3. **The AI-era implication:** When an AI agent generates code, you need to evaluate whether it made *good* trade-offs for the context. That requires understanding the design space, not just one language.

> "A language that doesn't affect the way you think about programming is not worth knowing." — Alan Perlis

---

<p style="font-size: 0.9rem;">
  <a href="/level-05/">← Level 0.5 Overview</a> · Next: <a href="/level-05/haskell/">Deep Dive: Functional Programming with Haskell →</a>
</p>
