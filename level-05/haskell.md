---
layout: page
title: "Deep Dive: Functional Programming with Haskell"
description: "A focused exercise in fundamentally different computation thinking. Pure functions, monads, type classes, and lazy evaluation."
permalink: /level-05/haskell/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-05/">Level 0.5</a> / Haskell'
---

## Why Haskell?

You have already seen Haskell alongside five other languages. Now we go deep. Haskell is not here because you will use it every day — it is here because it will **rewire how you think about computation**.

Haskell enforces constraints that other languages leave optional:

- **Every function is pure** — same input, same output, always
- **Side effects are tracked in the type system** — you can *see* what a function can do by reading its type
- **Data is immutable by default** — no mutation, no surprises
- **Evaluation is lazy** — nothing is computed until it is needed

These constraints sound limiting. They are liberating. They force you to think about *what* to compute rather than *how* to compute it.

> **Why it matters for the AI era:** FP's emphasis on pure, composable, side-effect-free functions maps directly to structuring work for AI agents. A pure function is easy to test, easy to verify, easy to replace, and easy to reason about — whether the reasoner is a human or an LLM.

---

## Pure Functions

A pure function has two properties:

1. **Deterministic:** Given the same inputs, it always returns the same output
2. **No side effects:** It does not modify any state, write to disk, send network requests, or do anything besides compute its return value

```haskell
-- Pure: always returns the same result for the same input
double :: Int -> Int
double x = x * 2

-- Pure: no matter how many times you call it, same result
add :: Int -> Int -> Int
add x y = x + y

-- Pure: works on any list, no side effects
sumList :: [Int] -> Int
sumList []     = 0
sumList (x:xs) = x + sumList xs
```

What about functions that *need* side effects — reading files, printing output, getting user input? Haskell handles those with the type system.

```haskell
-- This type signature tells you everything:
readFile :: FilePath -> IO String
--                      ^^ This function performs IO!

-- Compare with:
length :: [a] -> Int
--               ^^ No IO. Pure computation. Guaranteed.
```

The `IO` in the type is not decoration. The compiler enforces it. You cannot call an `IO` function from a pure function. Side effects are **quarantined**.

---

## The Type System

Haskell's type system is one of the most expressive in any mainstream language. Types are not just labels — they are a design tool.

### Basic Types and Type Signatures

```haskell
-- Every value has a type
x :: Int
x = 42

name :: String
name = "Haskell"

-- Functions have types too
-- Read -> as "takes ... and returns ..."
increment :: Int -> Int
increment n = n + 1

-- Multiple arguments are curried
add :: Int -> Int -> Int
add x y = x + y
-- Really means: Int -> (Int -> Int)
-- A function that takes an Int and returns a function!

-- Partial application falls out naturally
addFive :: Int -> Int
addFive = add 5
-- addFive 3 == 8
```

### Parametric Polymorphism

```haskell
-- 'a' is a type variable — this works for ANY type
identity :: a -> a
identity x = x

-- Works for any list element type
head :: [a] -> a
head (x:_) = x

-- The type tells you what the function CAN'T do:
-- identity :: a -> a
-- Since 'a' can be anything, the function can't inspect it,
-- can't add to it, can't compare it. It can ONLY return it.
-- There is literally one possible implementation!
```

### Type Classes

Type classes are like interfaces, but more powerful:

```haskell
-- Eq: types that can be compared for equality
class Eq a where
    (==) :: a -> a -> Bool
    (/=) :: a -> a -> Bool

-- Ord: types that can be ordered (requires Eq)
class Eq a => Ord a where
    compare :: a -> a -> Ordering
    (<)     :: a -> a -> Bool
    -- ...

-- Show: types that can be converted to strings
class Show a where
    show :: a -> String

-- You can require type class constraints in function signatures
sortAndShow :: (Ord a, Show a) => [a] -> String
sortAndShow xs = show (sort xs)
-- This works for ANY type that is both orderable and showable
```

### Algebraic Data Types in Depth

```haskell
-- Sum type: a value is one of several variants
data Color = Red | Green | Blue

-- Sum type with associated data
data Shape
    = Circle Double              -- radius
    | Rectangle Double Double    -- width, height
    | Triangle Double Double Double  -- three sides

-- Parameterized types
data Maybe a = Nothing | Just a
data Either a b = Left a | Right b

-- Recursive types
data List a = Empty | Cons a (List a)
-- [1, 2, 3] is Cons 1 (Cons 2 (Cons 3 Empty))

data Tree a = Leaf | Node (Tree a) a (Tree a)
```

Pattern matching makes working with ADTs natural:

```haskell
area :: Shape -> Double
area (Circle r)        = pi * r * r
area (Rectangle w h)   = w * h
area (Triangle a b c)  = let s = (a + b + c) / 2
                          in sqrt (s * (s-a) * (s-b) * (s-c))

-- The compiler warns if you miss a case!

-- Maybe: safe handling of absent values
safeDivide :: Double -> Double -> Maybe Double
safeDivide _ 0 = Nothing
safeDivide x y = Just (x / y)

-- Using Maybe — you must handle both cases
describeDivision :: Double -> Double -> String
describeDivision x y =
    case safeDivide x y of
        Nothing     -> "Cannot divide by zero"
        Just result -> "Result: " ++ show result
```

---

## Monads

Monads have a reputation for being difficult. They are not. A monad is a design pattern for **sequencing computations that have some extra context**.

### The Pattern

Every monad provides two operations:

```haskell
-- Wrap a value in the context
return :: a -> m a

-- Chain two computations: take the result of the first,
-- feed it into a function that produces the second
(>>=) :: m a -> (a -> m b) -> m b
-- Pronounced "bind"
```

That is it. The rest is applying this pattern to different contexts.

### Maybe Monad — Computations That Might Fail

```haskell
-- Without monads: nested case analysis
lookupUser :: UserId -> Maybe User
lookupAddress :: User -> Maybe Address
lookupCity :: Address -> Maybe City

-- This is painful:
getCityForUser :: UserId -> Maybe City
getCityForUser uid =
    case lookupUser uid of
        Nothing   -> Nothing
        Just user ->
            case lookupAddress user of
                Nothing      -> Nothing
                Just address -> lookupCity address

-- With the Maybe monad: clean chaining
getCityForUser :: UserId -> Maybe City
getCityForUser uid =
    lookupUser uid >>= lookupAddress >>= lookupCity

-- With do-notation (syntactic sugar for >>=):
getCityForUser :: UserId -> Maybe City
getCityForUser uid = do
    user    <- lookupUser uid
    address <- lookupAddress user
    lookupCity address
-- If any step returns Nothing, the whole thing short-circuits to Nothing
```

### Either Monad — Computations That Might Fail With an Error

```haskell
data AppError = NotFound String | PermissionDenied | ParseError String

validateAge :: String -> Either AppError Int
validateAge s =
    case reads s of
        [(n, "")] -> if n > 0 && n < 150
                     then Right n
                     else Left (ParseError "Age out of range")
        _         -> Left (ParseError "Not a number")

validateName :: String -> Either AppError String
validateName "" = Left (ParseError "Name cannot be empty")
validateName n  = Right n

-- Chain with do-notation — stops at first Left
validatePerson :: String -> String -> Either AppError (String, Int)
validatePerson nameStr ageStr = do
    name <- validateName nameStr
    age  <- validateAge ageStr
    Right (name, age)
```

### IO Monad — Computations That Interact With the World

```haskell
-- IO actions are values that describe side effects
main :: IO ()
main = do
    putStrLn "What is your name?"   -- IO ()
    name <- getLine                  -- IO String, bind result to 'name'
    putStrLn ("Hello, " ++ name)    -- IO ()

-- Key insight: until main is executed by the runtime,
-- these are just descriptions of what to do.
-- The program is a pure value that describes an impure computation.
```

### Monad Summary

| Monad | Context | `>>=` Behavior |
|-------|---------|---------------|
| `Maybe` | Might be absent | Short-circuit on `Nothing` |
| `Either e` | Might fail with error | Short-circuit on `Left` |
| `IO` | Interacts with the world | Sequence side effects |
| `[]` (List) | Multiple possibilities | Try all combinations |
| `State s` | Carries mutable state | Thread state through |
| `Reader r` | Has read-only environment | Pass environment implicitly |

---

## Lazy Evaluation

Haskell does not evaluate an expression until its value is needed. This is *lazy evaluation* (or *call-by-need*).

```haskell
-- Infinite lists are perfectly fine
ones :: [Int]
ones = 1 : ones           -- [1, 1, 1, 1, ...]

naturals :: [Int]
naturals = [1..]           -- [1, 2, 3, 4, ...]

-- Only compute what you use
take 5 naturals            -- [1, 2, 3, 4, 5]

-- Elegant algorithms become possible
primes :: [Int]
primes = sieve [2..]
  where
    sieve (p:xs) = p : sieve [x | x <- xs, x `mod` p /= 0]

-- First 10 primes
take 10 primes             -- [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]

-- Fibonacci via self-reference
fibs :: [Integer]
fibs = 0 : 1 : zipWith (+) fibs (tail fibs)

take 15 fibs
-- [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
```

Lazy evaluation lets you **separate the description of a computation from its execution**. You define what the answer *is*; the runtime figures out what it needs to compute.

---

## Exercise: Build a Small Interpreter

The best way to internalize Haskell is to build something that uses all its features. Implement a small arithmetic expression interpreter:

### Step 1: Define the Data Type

```haskell
data Expr
    = Lit Double               -- literal number
    | Add Expr Expr            -- addition
    | Mul Expr Expr            -- multiplication
    | Div Expr Expr            -- division (can fail!)
    | Neg Expr                 -- negation
    deriving (Show)
```

### Step 2: Write the Evaluator

```haskell
eval :: Expr -> Either String Double
eval (Lit n)   = Right n
eval (Neg e)   = do
    v <- eval e
    Right (negate v)
eval (Add a b) = do
    x <- eval a
    y <- eval b
    Right (x + y)
eval (Mul a b) = do
    x <- eval a
    y <- eval b
    Right (x * y)
eval (Div a b) = do
    x <- eval a
    y <- eval b
    if y == 0
        then Left "Division by zero"
        else Right (x / y)
```

### Step 3: Test It

```haskell
-- (3 + 4) * 2
expr1 :: Expr
expr1 = Mul (Add (Lit 3) (Lit 4)) (Lit 2)
-- eval expr1 == Right 14.0

-- 10 / (5 - 5) — should fail
expr2 :: Expr
expr2 = Div (Lit 10) (Add (Lit 5) (Neg (Lit 5)))
-- eval expr2 == Left "Division by zero"
```

### Step 4: Add a Pretty Printer

```haskell
prettyPrint :: Expr -> String
prettyPrint (Lit n)   = show n
prettyPrint (Neg e)   = "-(" ++ prettyPrint e ++ ")"
prettyPrint (Add a b) = "(" ++ prettyPrint a ++ " + " ++ prettyPrint b ++ ")"
prettyPrint (Mul a b) = "(" ++ prettyPrint a ++ " * " ++ prettyPrint b ++ ")"
prettyPrint (Div a b) = "(" ++ prettyPrint a ++ " / " ++ prettyPrint b ++ ")"
```

### Extensions (Challenge)

- Add variables: `Var String` and an environment `Map String Double`
- Add a parser: turn `"(3 + 4) * 2"` into `Mul (Add (Lit 3) (Lit 4)) (Lit 2)`
- Add let-bindings: `Let String Expr Expr` for `let x = 3 in x + 1`

---

## Why This Matters for AI-Era Development

| FP Principle | AI Agent Benefit |
|-------------|-----------------|
| **Pure functions** | Deterministic, easy to test, easy to verify agent output |
| **Immutability** | No hidden state for agents to corrupt |
| **Strong types** | Compiler catches errors before agent code runs |
| **Composability** | Small, verified pieces that agents can combine |
| **Explicit effects** | Clear boundaries — agents know what can have side effects |
| **Pattern matching + ADTs** | Exhaustive handling — compiler ensures no cases missed |

When you ask an AI agent to generate code, the functional style produces code that is:

- **Testable** — pure functions are trivially unit-testable
- **Verifiable** — types constrain what the code can do
- **Composable** — small functions combine into larger ones without hidden interactions
- **Replaceable** — swap one pure function for another without fear of breaking distant code

> "The same properties that make Haskell code correct make it agent-friendly."

---

<p style="font-size: 0.9rem;">
  <a href="/level-05/languages/">← Languages</a> · Next: <a href="/level-05/modularity/">Modularity: From Unix Pipes to AI Agents →</a>
</p>
